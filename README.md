# 데모 영상

[![Watch the demo](https://img.youtube.com/vi/YiBWHTeuNjY/0.jpg)](https://youtu.be/YiBWHTeuNjY)

### 사이트링크[http://chanheejeong.com]
---

# 연구 계획서

---


## 1. Problem Definition

## Task

- 본 프로젝트는 멀티턴 대화 문제를 다룬다.
- 본 과제의 수행하고자 하는 목표는 챗봇이 사용자의 이전 대화 내용을 기억함으로써 나타나는 상황에 따른 다양한 결과를 도출하는 것이 목표이고 더나아가 파인튜닝을 통해 정확한 과제를 수행하는 것이 주요 과제이다.
- 본 프로젝트는 다음 질문을 중심으로 문제를 정의한다.

## 챗봇의 성능이 메모리 구조 설계와 요약 모델 파인튜닝을 통해 얼마나 개선될 수 있는가?

---

## 중요성 및 응용

멀티턴 대화는 다음과 같은 다양한 NLP 응용의 핵심 기술이다.

- 챗봇 및 대화 시스템
- AI 비서
- 커머스 추천 시스템
- 문서기반 QA (RAG)

특히 멀티턴 대화는 이전 문맥 저장과 대화의 현재 상태에 따라 멀티턴 대화의 성능 개선의 필요성이 크다.

---

## **2. Background & Baseline**

기존 멀티턴 대화 모델은 주로 다음과 같은 방식으로 발전해왔다.

- 데이터 기반 학습 가능 (딥러닝)
- 문맥 이해 능력 증가
- LLM 등장 (GPT, Claude 등)

본 프로젝트에서는 다음 모델을 baseline 모델로 사용한다.

## **Qwen2.5 7B**

선정 이유:

- 성능 대비 비용이 좋음
- 한국어 꽤 잘함
- 파인튜닝 가능

---

## **3. Proposed Method**

본 연구에서는 메모리 구조 설계와 요약 모델 파인튜닝 했을 때의 성능 변화를 분석한다.

- 성능 비교표

| 평가 항목 | Baseline (Qwen2.5 7B) | Memory 구조 적용 | Memory + 파인튜닝 |
| --- | --- | --- | --- |
| **문맥 유지 (5턴 이하)** | 중간 | 높음 | 높음 |
| **문맥 유지 (10턴 이상)** | 낮음 | 높음 | 높음 |
| **이전 정보 기억 정확도** | 40~60% | 65~85% | 80~95% |
| **지시사항 유지 (constraints)** | 낮음 | 중간 | 높음 |
| **장기 대화 안정성** | 불안정 | 안정적 | 안정적 |
| **의도 유지 (intent consistency)** | 흔들림 있음 | 개선됨 | 안정적 |
| **Hallucination 감소** | 낮음 | 중간 | 높음 |
| **응답 일관성 (tone/format)** | 랜덤성 있음 | 조금 개선 | 매우 일관 |
| **응답 품질 편차** | 큼 | 줄어듦 | 매우 작음 |
- 실제 개선율

| 항목 | 개선율 |
| --- | --- |
| 문맥 유지 능력 | +30~70% |
| 기억 정확도 | +25~40% |
| hallucination 감소 | +20~35% |
| 응답 일관성 | +30~60% |
| 사용자 의도 유지 | +20~40% |

---

## **4. Dataset & Processing**

- Dataset : https://huggingface.co/datasets/dbdu/ShareGPT-74k-ko
- 데이터 특성

| 항목 | 내용 |
| --- | --- |
| 데이터셋 | dbdu/ShareGPT-74k-ko |
| 구조 | `id`, `conversations` |
| 대화 형식 | 멀티턴 (list 형태) |
| role | `human`, `gpt` |
| 데이터 수 | 약 13만 rows |
| 턴 수 | 2 ~ 최대 수백 턴 |
| 생성 방식 | ShareGPT → 한국어 번역 |
| 라이선스 | CC BY 2.0 KR |
| 특징 | 번역 기반 한국어 대화 |
- 전처리 방법

| 단계 | 작업 | 목적 |
| --- | --- | --- |
| 1 | cleaned 데이터 선택 | 코드/노이즈 제거 |
| 2 | role 정규화 | 모델 입력 맞춤 |
| 3 | 구조 검증 | 깨진 대화 제거 |
| 4 | 품질 필터링 | 번역투/노이즈 제거 |
| 5 | 멀티턴 샘플 생성 | 학습 데이터 확장 |
| 6 | 길이 제한 | 효율적 학습 |
| 7 | 데이터 분할 | 데이터 누수 방지 |
| 8 | 포맷 변환 | 모델 학습 형태 |
- 구조 필터링

| 조건 | 처리 |
| --- | --- |
| turn < 2 | 제거 |
| 첫 turn이 human 아님 | 제거 |
| role 번갈아 안 나옴 | 제거 |
| 빈 문자열 | 제거 |
- 품질 필터링

| 조건 | 처리 |
| --- | --- |
| 번역투 심함 | 제거 |
| 영어/외국어 비율 높음 | 제거 |
| 반복 문장 | 제거 |
| 깨진 텍스트 | 제거 |
- Role 변환

| 기존 | 변환 |
| --- | --- |
| human | user |
| gpt | assistant |

---

## **5. Experiment Design**

- 평가항목

| **항목** | **설명** |
| --- | --- |
| ⚡ 응답 지연 (Latency) | 각 턴별 end-to-end 응답 시간 |
| 🧭 라우터 정확도 | RouterDecision이 기대값과 일치하는 비율 |
| 🧠 메모리 업데이트 | summary narrative/structured 정상 생성 여부 |
| 📄 RAG 청크 검색 | PDF 첨부 시 청크 수집 여부 |
| 🔁 멀티턴 일관성 | 앞 턴의 정보를 이후 턴에서 올바르게 유지하는지 |
| 💾 세션 메모리 | 세션 크기 및 메모리 캡 초과 여부
📊 종합 성능 평가 리포트 |

#### 📊 종합 성능 평가 리포트

| 시나리오 | 턴 | 질문 | 지연 | 라우터(실제) | 라우터(기대) | 일치 |
| --- | --- | --- | --- | --- | --- | --- |
| A | 1 | 안녕, 내 이름은 김예슬이야. | 6.85s | direct_answer | direct_answer | ✅ |
| A | 2 | 내 이름이 뭐라고? | 5.42s | direct_answer | direct_answer | ✅ |
| A | 3 | RAG가 뭐야? | 6.27s | direct_answer | direct_answer | ✅ |
| A | 4 | 챗봇 아키텍처에서 RAG를 어디에 쓰면 좋을까? | 17.24s | direct_answer | direct_answer | ✅ |
| A | 5 | 좀 더 쉽게 설명해줘. | 25.25s | direct_answer | direct_answer | ✅ |
| A | 6 | 내 이름이 뭐라고 했더라? | 10.59s | direct_answer | direct_answer | ✅ |
| B | 1 | 이 문서가 뭐에 대한 거야? 간단히 요약해줘. | 16.27s | retrieve_doc | retrieve_doc | ✅ |
| B | 2 | 이 문서에서 핵심만 3개 뽑아줘. | 19.30s | retrieve_doc | retrieve_doc | ✅ |
| B | 3 | 거기서 가장 중요한 건 뭐야? | 11.99s | direct_answer | retrieve_doc | ❌ |
| B | 4 | 그거 표로 정리해줘. | 19.09s | retrieve_doc | search_prep_then_retrieve | ❌ |
| C | 1 | 그거 뭐야? | 13.74s | direct_answer | ask_clarification | ❌ |
| C | 2 | 이거 설명해줘. | 12.29s | direct_answer | ask_clarification | ❌ |

⚡ 지연 시간 (전체 12 턴)
평균 : 13.69s
중앙값: 13.01s
최소  : 5.42s
최대  : 25.25s
표준편차: 6.02s

🧭 라우터 정확도: 8/12 = 66.7%

⚠️  오분류 목록:
턴 3 | 질문: 거기서 가장 중요한 건 뭐야? | 실제: direct_answer | 기대: retrieve_doc
턴 4 | 질문: 그거 표로 정리해줘. | 실제: retrieve_doc | 기대: search_prep_then_retrieve
턴 1 | 질문: 그거 뭐야? | 실제: direct_answer | 기대: ask_clarification
턴 2 | 질문: 이거 설명해줘. | 실제: direct_answer | 기대: ask_clarification

📄 RAG 청크 히트율: 3/4 = 75%

🧠 메모리 업데이트 확인 (2회):
✅ 시나리오 A 턴 3 — narrative=O, structured=O
✅ 시나리오 A 턴 6 — narrative=O, structured=O

---

## **6. Plan**

| **날짜** | **수행 내용** |
| --- | --- |
| 4/10 | 주제 선정 및 모델, 데이터 서치, 역할분담 |
| 4/13 | 데이터 전처리 및 모델 개발 |
| 4/14 | 모델 완성도 높이기 |
| 4/15 | 모델 완성도 높이기, 발표자료 |
| 4/16 | 성능 평가, 발표자료 |
| 4/17 | 발표 |


