# Multiturn Memory Chat

> **이전 대화를 기억하고 PDF 문서를 근거로 답하는 멀티턴(multi-turn) 대화 챗봇.**
> 메모리 구조 설계와 요약 모델 파인튜닝으로 멀티턴 성능이 얼마나 개선되는지를 연구한 NLP 팀 프로젝트입니다. (2026-NLP-03 · 2026.04)

이 저장소(`docs`)는 프로젝트의 **대표 저장소**로, 시스템 전체 개요·아키텍처·평가와 각 구성 저장소로의 링크를 모읍니다. 멀티레포(multi-repo) 구조라, 전체 그림은 여기서 보고 세부 구현은 각 저장소에서 확인하면 됩니다.

- 연구 계획 전문: [RESEARCH_PLAN.md](RESEARCH_PLAN.md)
- GitHub Organization: [`2026-Pretrained-Conversational-Model`](https://github.com/2026-Pretrained-Conversational-Model)

---

## 무엇을 푸는가

챗봇은 단발 질문에는 잘 답하지만 **대화가 길어지면** 앞서 들은 이름·제약을 잊고, "그거/아까" 같은 지시어를 놓치며, 답의 일관성이 흔들립니다. 이 프로젝트는 한 가지 질문에서 출발합니다.

> **챗봇 성능은 메모리 구조 설계와 요약 모델 파인튜닝으로 얼마나 개선될 수 있는가?**

큰 모델에 의존하는 대신, 대화를 **기억 상태(memory state)**로 압축·갱신하는 구조를 직접 설계하고, 그 요약을 담당하는 모델을 파인튜닝해 효과를 측정했습니다.

---

## 시스템 아키텍처

핵심 아이디어는 **"한 턴 = 하나의 기억 상태를 읽고, 답하고, 갱신하는 사이클"**입니다.

```
[사용자/브라우저]
   │  WebSocket
   ▼
[Node.js 게이트웨이]  ── WS↔HTTP 번역, 파일 업로드 중계
   │  HTTP POST /chat, /upload
   ▼
[FastAPI 오케스트레이터]  ── 턴 파이프라인
   │   세션 로드 → (PDF 백그라운드 ingest)
   │   → 지시어해소·의도·주제·라우터 (병렬)
   │   → 라우터 분기(DIRECT / RETRIEVE / SEARCH_PREP / ASK)
   │   → 프롬프트(메모리+최근대화+검색) → 모델 호출
   │   → finalize(3턴마다 메모리 갱신 + 메모리 캡 검사)
   │  HTTP
   ▼
[Qwen2.5 LLM / VLM]  ── 로컬(Colab·RunPod) 또는 SageMaker
```

---

## 저장소 구성 (멀티레포)

| 저장소 | 역할 | 주 담당 |
| --- | --- | --- |
| [ai-engine](https://github.com/2026-Pretrained-Conversational-Model/ai-engine) | FastAPI 오케스트레이터 — 턴 파이프라인·세션 메모리·RAG | 김예슬 / 진주용(라우터) / 이지선(검색) |
| [backend](https://github.com/2026-Pretrained-Conversational-Model/backend) | Node.js WebSocket 게이트웨이 (WS↔HTTP, 업로드) | 팀 공동 (설계·통합: 김예슬) |
| frontend | 정적 채팅 UI (HTML/JS, nginx) | 팀 공동 |
| [ai-pod-v1](https://github.com/2026-Pretrained-Conversational-Model/ai-pod-v1) | RunPod GPU 런처 (오케스트레이터 기동) | 정찬희 |
| [model](https://github.com/2026-Pretrained-Conversational-Model/model) | Qwen2.5-3B 모델 자원 | 정찬희 / 김예슬(메모리 파인튜닝) |
| infra | SageMaker 모델 배포 | 정찬희 |
| [eval](https://github.com/2026-Pretrained-Conversational-Model/eval) | 성능 평가 스크립트·실측 결과 | 팀 공동 |
| [docs](https://github.com/2026-Pretrained-Conversational-Model/docs) | 연구계획·평가·전체 개요 (현재 저장소) | 김예슬 |

---

## 핵심 설계 원칙

1. **학습 대상은 메모리 모델 하나로 고정** — 라우터·검색·답변 모델은 비학습 baseline으로 두어, 성능 변화의 변인을 메모리 모델로 좁혔습니다.
2. **멀티턴 우선, 문서는 보조** — 모든 요청은 세션 맥락으로 먼저 해석하고, PDF/이미지는 필요할 때만 답변을 보강합니다.
3. **판단과 추론 분리** — "검색이 필요한가" 같은 판단은 오케스트레이터가, 추론은 모델 서버가 맡아 모델 교체가 쉽습니다.

---

## 사용 모델

| 역할 | 모델 |
| --- | --- |
| 답변(Answer) | Qwen2.5-7B-Instruct (4bit NF4) |
| 라우터(Router) | Qwen2.5-3B-Instruct |
| 메모리 요약(Memory) | [`yeseul0-0/qwen2.5-3b-memory-summary-default_v0.3`](https://huggingface.co/yeseul0-0/qwen2.5-3b-memory-summary-default_v0.3) (Qwen2.5-3B LoRA 파인튜닝) |
| 임베딩(Embedding) | jhgan/ko-sroberta-multitask (768d) |
| 벡터 검색 | FAISS(IndexFlatIP), numpy 폴백 |

---

## 정량 평가 (실측)

자동 평가 기준이며, 잘된 부분과 한계를 함께 적습니다. 상세는 [eval](https://github.com/2026-Pretrained-Conversational-Model/eval) 저장소의 리포트를 참고하세요.

| 지표 | 값 | 비고 |
| --- | --- | --- |
| 멀티턴 정보 보존 | 75% | 단기 100%, 5턴 이후 장기 보존 실패 |
| 라우터 정확도 | 66.7%(시나리오) / 50%(자동) | 자동 평가는 측정 방식 한계 포함 |
| E2E 지연(중앙값) | 약 8.1초 | 평균 28.3초(긴 답변이 만든 꼬리) |
| E2E 지연(P95) | 약 72.8초 | 장문 답변에서 폭주 |
| RAG Faithfulness | 0.63 | RAGAS 기반 간이 측정 |

> 완성형이 아니라 **변인을 좁힌 baseline**입니다. 장기 보존과 라우터 후속 질의가 약점이며, 이는 메모리 모델 학습 데이터 보강이라는 다음 과제로 이어집니다.

---

## 팀 구성 및 역할

| 이름 | 역할 |
| --- | --- |
| **김예슬 (팀장)** | 전체 시스템 아키텍처·턴 파이프라인 설계, FastAPI 오케스트레이터, 세션 메모리 구조, 메모리 요약 모델 파인튜닝 |
| 진주용 | 라우터 — RAG 필요 판단(NEED_RAG/NO_RAG) 및 경로 결정 |
| 이지선 | 임베딩·FAISS 검색 — RAG retrieval baseline |
| 정찬희 | 인프라(SageMaker 배포)·모델 학습 |

---

## 더 읽기

- [연구 계획서](RESEARCH_PLAN.md) — 문제 정의, 데이터셋, 실험 설계
- 각 저장소 README — 구성요소별 설계·실행법
