---
layout: post
title: "AI 워크플로 입문 — 단순 챗봇을 넘어 일하는 AI 시스템 설계하기"
date: 2026-05-07 18:00:00 +0900
categories:
  - LLM
  - Architecture
tags: [AIWorkflow, LLM, Agent, PromptChaining, Orchestrator, n8n, LangChain, LangGraph, 자동화]
math: false
---

> {: .prompt-tip }
> **요약:** AI 워크플로는 LLM 호출을 단순한 한 번의 질문-답변을 넘어 **여러 단계로 연결된 자동화 파이프라인**으로 만드는 설계 방식입니다. Anthropic이 정리한 5가지 핵심 패턴(Prompt Chaining, Routing, Parallelization, Orchestrator-Workers, Evaluator-Optimizer)을 중심으로, AI 워크플로가 무엇인지, AI 에이전트와 어떻게 다른지, 어떤 도구를 써야 하는지, 그리고 프로덕션에서 무엇을 주의해야 하는지 처음 접하는 분도 이해할 수 있게 정리합니다.

---

"GPT API를 연결했는데, 이걸로 어떻게 실제로 유용한 걸 만들 수 있을까?"

LLM을 처음 써본 많은 분들이 겪는 막막함입니다. API 호출 하나로 단순한 Q&A는 되지만, 진짜 업무를 처리하는 시스템을 만들려면 뭔가 더 필요하다는 느낌이 듭니다. 그 "뭔가"가 바로 **AI 워크플로**입니다.

---

## 1. AI 워크플로란 무엇인가?

### 단순 LLM 호출의 한계

가장 기본적인 LLM 활용은 이렇게 생겼습니다.

```python
response = llm("이 계약서를 요약해줘: " + contract_text)
print(response)
```

이것은 LLM 호출이지, 워크플로가 아닙니다. 단일 호출로는 다음과 같은 한계가 있습니다.

- **컨텍스트 제약:** 한 번의 프롬프트에 담을 수 있는 정보가 한정적입니다
- **품질 검증 없음:** 출력이 맞는지 틀린지 검증할 방법이 없습니다
- **복잡한 태스크 불가:** "조사하고 → 분석하고 → 보고서 쓰기"처럼 여러 단계로 이루어진 작업을 처리할 수 없습니다
- **재사용 불가:** 비슷한 작업을 반복할 때마다 처음부터 다시 해야 합니다

### AI 워크플로의 정의

**AI 워크플로(AI Workflow)**는 LLM 호출과 도구(tool)들을 **미리 정의된 순서와 로직으로 연결**해 복잡한 태스크를 자동으로 처리하는 파이프라인입니다.

핵심은 이것입니다.

> **개발자가 실행 흐름을 설계한다. LLM은 각 단계 안에서만 추론한다.**

```
[단순 LLM 호출]
사용자 입력 ──→ LLM ──→ 출력
                (모든 것을 LLM에 맡김)

[AI 워크플로]
사용자 입력
    ↓
[단계 1: 문서 파싱]   ← 개발자가 설계한 흐름
    ↓
[단계 2: LLM 분석]    ← LLM은 이 단계 안에서만 추론
    ↓
[단계 3: 결과 검증]
    ↓
[단계 4: 보고서 생성]
    ↓
최종 출력
```

### 워크플로를 구성하는 세 가지 요소

**① LLM (추론 엔진)**
각 단계에서 텍스트를 이해하고 생성합니다. GPT-4o, Claude, Gemini 등 어떤 모델이든 사용할 수 있습니다.

**② 도구 (Tools)**
LLM이 외부 세계와 상호작용하기 위한 기능들입니다.

| 도구 종류 | 예시 |
| :--- | :--- |
| 검색 | 웹 검색, 벡터 DB 검색, SQL 쿼리 |
| 파일 처리 | 파일 읽기/쓰기, PDF 파싱, 이미지 처리 |
| 외부 API | 슬랙 메시지, 이메일 발송, 캘린더 등록 |
| 코드 실행 | Python 인터프리터, 데이터 분석 |

**③ 실행 흐름 (Orchestration)**
단계들을 어떤 순서로, 어떤 조건에서 실행할지 정의합니다. 이것이 워크플로의 뼈대입니다.

---

## 2. AI 워크플로 vs AI 에이전트 — 무엇이 다른가?

이 두 개념은 자주 혼용되지만 근본적으로 다릅니다. 이 차이를 모르면 잘못된 도구를 선택하게 됩니다.

### 핵심 차이: 누가 흐름을 결정하는가?

```
[AI 워크플로 — 개발자가 흐름 결정]

개발자: "1단계는 검색, 2단계는 분석, 3단계는 요약"
         ↓ 미리 설계
실행 시: 항상 이 순서대로 실행
         조건 분기도 모두 미리 정의됨
         LLM은 각 단계 안에서만 역할 수행

[AI 에이전트 — LLM이 흐름 결정]

개발자: "목표만 알려줌"
LLM: "이 목표를 달성하려면 어떤 도구를 어떤 순서로 써야 할까?"
      ↓ 런타임에 판단
실행 시: LLM이 매번 다음 행동을 스스로 결정
         Goal → Perception → Reasoning → Action → Observation → 반복
```

### 비교표

| 항목 | AI 워크플로 | AI 에이전트 |
| :--- | :--- | :--- |
| **실행 흐름 결정** | 개발자 (사전 설계) | LLM (런타임 판단) |
| **예측 가능성** | 높음 (항상 같은 경로) | 낮음 (매번 달라질 수 있음) |
| **디버깅 용이성** | ✅ 쉬움 | ⚠️ 어려움 |
| **비용 예측** | ✅ 예측 가능 | ⚠️ 변동성 큼 |
| **처리량(Throughput)** | ✅ 높음 | ⚠️ 낮음 |
| **유연성** | ⚠️ 사전 정의 범위 내 | ✅ 열린 태스크 처리 |
| **적합한 태스크** | 반복적, 예측 가능한 작업 | 동적, 개방형 판단 필요 작업 |

### 언제 무엇을 쓸까?

```
태스크가 예측 가능한 단계로 분해되는가?
    └── YES → 워크플로
    └── NO  →

입력에 따라 어떤 도구를 쓸지 런타임에 결정해야 하는가?
    └── YES → 에이전트
    └── NO  → 워크플로

고처리량, 실시간, 비용 통제가 중요한가?
    └── YES → 워크플로
    └── NO  →

사용자와 멀티턴 상호작용하며 맥락 기반 판단이 필요한가?
    └── YES → 에이전트
    └── NO  → 워크플로
```

> {: .prompt-tip }
> **황금률:** 워크플로로 시작하고, 진짜 에이전트가 필요한 곳에만 에이전트를 추가하세요. 대부분의 프로덕션 시스템은 워크플로가 전체 구조를 담당하고, 에이전트는 일부 동적 판단이 필요한 단계에서만 사용됩니다.

### 오류 누적의 수학

에이전트 방식의 실질적인 위험을 수치로 보면 이렇습니다.

```
각 단계 신뢰도 99%로 가정:

단계  1개: 99.0% 성공
단계  5개: 95.1% 성공
단계 10개: 90.4% 성공
단계 20개: 81.8% 성공
단계 50개: 60.5% 성공

→ 에이전트가 자율적으로 10단계를 거치면
  10번 중 1번은 잘못된 결과를 냅니다.
  각 단계의 신뢰도가 95%라면 더 빨리 악화됩니다.
```

워크플로는 각 단계를 명시적으로 검증하거나 에러 핸들링을 추가할 수 있어, 이 누적 오류 문제를 훨씬 효과적으로 관리할 수 있습니다.

---

## 3. 5가지 핵심 워크플로 패턴

Anthropic의 엔지니어링 팀이 프로덕션에서 작동하는 패턴을 분석해 정리한 5가지 핵심 패턴입니다. 이 패턴들은 단독으로 또는 조합해서 사용합니다.

---

### 패턴 1: Prompt Chaining (프롬프트 체이닝)

**한 줄 정의:** 이전 단계의 출력이 다음 단계의 입력이 되는 순차적 파이프라인

```
입력 → [LLM 1] → 중간 결과 → [LLM 2] → 중간 결과 → [LLM 3] → 최종 출력
```

**언제 쓰는가:**
- 태스크를 여러 단계로 명확하게 분해할 수 있을 때
- 각 단계가 독립적으로 검증 가능할 때
- 긴 문서 처리(파싱 → 분석 → 요약 → 리포트)

**실전 예시: 계약서 리뷰 파이프라인**

```python
import anthropic

client = anthropic.Anthropic()

def contract_review_pipeline(contract_text: str) -> dict:
    """3단계 계약서 리뷰 워크플로."""

    # 단계 1: 핵심 조항 추출
    step1 = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": f"다음 계약서에서 핵심 조항(의무, 기간, 위약금)만 추출하세요:\n\n{contract_text}"
        }]
    )
    key_clauses = step1.content[0].text

    # 단계 2: 리스크 분석 (1단계 결과를 입력으로 사용)
    step2 = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1500,
        messages=[{
            "role": "user",
            "content": f"다음 계약 조항의 법적 리스크를 분석하세요:\n\n{key_clauses}"
        }]
    )
    risk_analysis = step2.content[0].text

    # 단계 3: 최종 요약 리포트 생성 (1, 2단계 결과를 모두 활용)
    step3 = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1000,
        messages=[{
            "role": "user",
            "content": f"""
            계약 조항: {key_clauses}
            리스크 분석: {risk_analysis}

            위 내용을 바탕으로 경영진용 1페이지 요약 리포트를 작성하세요.
            """
        }]
    )

    return {
        "key_clauses": key_clauses,
        "risk_analysis": risk_analysis,
        "executive_summary": step3.content[0].text
    }
```

> {: .prompt-info }
> **체크포인트 추가:** 각 단계 사이에 출력 검증 로직을 추가하면 오류가 다음 단계로 전파되는 것을 막을 수 있습니다. 예를 들어 1단계 출력이 비어있으면 2단계로 넘어가지 않게 합니다.

---

### 패턴 2: Routing (라우팅)

**한 줄 정의:** 입력의 종류를 분류해 각각에 맞는 전문화된 처리 경로로 보내는 패턴

```
                    ┌─→ [전문가 A: 기술 질문 처리]
입력 → [분류기] ───┼─→ [전문가 B: 환불 처리]
                    └─→ [전문가 C: 일반 문의 처리]
```

**언제 쓰는가:**
- 입력 유형에 따라 최적의 처리 방식이 다를 때
- 각 카테고리에 맞춤화된 시스템 프롬프트가 필요할 때
- 비용 최적화 (단순 질문은 저렴한 모델, 복잡한 질문은 고성능 모델로 라우팅)

**실전 예시: 고객 문의 라우팅**

```python
from typing import Literal

def route_customer_inquiry(inquiry: str) -> str:
    """고객 문의를 분류해 적절한 처리기로 라우팅."""

    # 분류기: 빠르고 저렴한 모델 사용
    classifier = client.messages.create(
        model="claude-haiku-4-5",   # 분류는 저렴한 모델로
        max_tokens=20,
        messages=[{
            "role": "user",
            "content": f"""
            다음 문의를 분류하세요. 반드시 하나만 답하세요:
            technical / refund / general

            문의: {inquiry}
            """
        }]
    )
    category = classifier.content[0].text.strip().lower()

    # 카테고리에 맞는 전문 처리기로 라우팅
    system_prompts = {
        "technical": "당신은 소프트웨어 기술 지원 전문가입니다. 단계별로 문제를 해결해주세요.",
        "refund":    "당신은 환불 정책 전문가입니다. 정중하고 명확하게 절차를 안내해주세요.",
        "general":   "당신은 친절한 고객 서비스 담당자입니다.",
    }
    system = system_prompts.get(category, system_prompts["general"])

    # 전문 처리기: 필요에 따라 더 강력한 모델 사용
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=500,
        system=system,
        messages=[{"role": "user", "content": inquiry}]
    )
    return response.content[0].text
```

---

### 패턴 3: Parallelization (병렬화)

**한 줄 정의:** 독립적인 서브태스크를 동시에 실행해 전체 처리 시간을 단축하는 패턴

병렬화에는 두 가지 변형이 있습니다.

**변형 A: Sectioning — 큰 작업을 병렬 처리**

```
                 ┌─→ [LLM: 1~50페이지 분석]  ─┐
긴 문서 → 분할 ──┼─→ [LLM: 51~100페이지 분석] ─┼→ [결과 통합]
                 └─→ [LLM: 101~150페이지 분석]─┘
```

**변형 B: Voting — 여러 관점으로 검증**

```
                 ┌─→ [LLM 1: 답변 생성]  ─┐
동일 입력 → ─────┼─→ [LLM 2: 답변 생성]  ─┼→ [다수결 또는 종합]
                 └─→ [LLM 3: 답변 생성]  ─┘
```

**언제 쓰는가:**
- 긴 문서를 섹션별로 처리할 때
- 여러 독립적 검색을 동시에 수행할 때
- 다수결로 정확도를 높여야 하는 중요한 판단에

**실전 예시: 병렬 문서 분석**

```python
import asyncio

async def analyze_section(section: str, section_num: int) -> dict:
    """문서 섹션 하나를 비동기로 분석."""
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": f"다음 텍스트의 핵심 인사이트를 3가지로 요약하세요:\n\n{section}"
        }]
    )
    return {"section": section_num, "insights": response.content[0].text}

async def parallel_document_analysis(document: str) -> list:
    """긴 문서를 섹션으로 나눠 병렬 분석."""
    # 문서를 500단어 단위로 분할
    words = document.split()
    sections = [
        " ".join(words[i:i+500])
        for i in range(0, len(words), 500)
    ]

    # 모든 섹션을 동시에 분석 (순차 처리 대비 N배 빠름)
    tasks = [
        analyze_section(section, i+1)
        for i, section in enumerate(sections)
    ]
    results = await asyncio.gather(*tasks)
    return results

# 실행
results = asyncio.run(parallel_document_analysis(long_document))
```

---

### 패턴 4: Orchestrator-Workers (오케스트레이터-워커)

**한 줄 정의:** 중앙 오케스트레이터가 태스크를 분석해 전문 워커에게 동적으로 위임하고 결과를 종합하는 패턴

```
                      ┌─→ [워커: 코드 작성]    ─┐
사용자 요청 → [오케스트레이터] ──┼─→ [워커: 테스트 작성]  ─┼→ [최종 통합]
              (태스크 분해/위임)  └─→ [워커: 문서 작성]    ─┘
```

Parallelization과의 차이가 중요합니다. Parallelization은 **미리 정의된** 서브태스크를 병렬 실행하지만, Orchestrator-Workers는 **오케스트레이터가 런타임에 동적으로** 어떤 워커에게 무엇을 맡길지 결정합니다.

**언제 쓰는가:**
- 필요한 서브태스크를 미리 알 수 없는 복잡한 태스크
- 멀티 에이전트 리서치 시스템 (Anthropic의 Deep Research 방식)
- 여러 파일을 수정해야 하는 코딩 태스크

**실전 예시: 리서치 오케스트레이터**

```python
def research_orchestrator(query: str, workers: dict) -> str:
    """
    오케스트레이터가 리서치 태스크를 분해해 워커에게 위임.
    workers: {"web_search": fn, "pdf_reader": fn, "data_analyst": fn}
    """

    # 오케스트레이터: 태스크 분해 및 계획 수립
    plan = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1000,
        system="""당신은 리서치 매니저입니다.
        사용 가능한 워커: web_search, pdf_reader, data_analyst
        각 워커에게 맡길 서브태스크를 JSON으로 계획하세요.""",
        messages=[{"role": "user", "content": f"다음을 조사하세요: {query}"}]
    )

    # 계획 파싱 (실제로는 JSON 파싱)
    subtasks = parse_plan(plan.content[0].text)

    # 각 서브태스크를 해당 워커에게 위임
    results = {}
    for task in subtasks:
        worker_fn = workers.get(task["worker"])
        if worker_fn:
            results[task["id"]] = worker_fn(task["instruction"])

    # 오케스트레이터가 결과 종합
    final = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": f"다음 리서치 결과를 종합해 최종 보고서를 작성하세요:\n{results}"
        }]
    )
    return final.content[0].text
```

---

### 패턴 5: Evaluator-Optimizer (평가자-최적화자)

**한 줄 정의:** 생성기가 초안을 만들고 평가자가 피드백을 주며, 기준을 만족할 때까지 반복 개선하는 패턴

```
           ┌─────────────────────────────────────┐
           ↓                                     │ (기준 미달)
입력 → [생성기: 초안 작성] → [평가자: 품질 검사] ─┤
                                                 │ (기준 충족)
                                                 ↓
                                             최종 출력
```

**언제 쓰는가:**
- 명확한 품질 기준이 있고 반복 개선이 가능한 태스크
- 번역 품질 검증
- 코드 리뷰 및 버그 수정 루프
- 마케팅 카피 정제

**실전 예시: 자기 교정 번역 파이프라인**

```python
def translation_with_quality_loop(
    text: str,
    target_lang: str = "영어",
    max_iterations: int = 3
) -> dict:
    """품질 기준을 충족할 때까지 번역을 반복 개선."""

    iteration = 0
    current_translation = None

    while iteration < max_iterations:
        iteration += 1

        # 생성기: 번역 (첫 번째는 초역, 이후는 개선)
        if current_translation is None:
            prompt = f"다음을 {target_lang}로 번역하세요:\n\n{text}"
        else:
            prompt = f"""
            원문: {text}
            현재 번역: {current_translation}
            평가 피드백: {feedback}

            피드백을 반영해 번역을 개선하세요.
            """

        gen_response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=2000,
            messages=[{"role": "user", "content": prompt}]
        )
        current_translation = gen_response.content[0].text

        # 평가자: 번역 품질 검사
        eval_response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=500,
            messages=[{
                "role": "user",
                "content": f"""
                원문: {text}
                번역: {current_translation}

                다음 기준으로 번역을 평가하세요:
                1. 정확성 (원문 의미 보존)
                2. 자연스러움 (목표 언어 표현)
                3. 전문 용어 적절성

                판정: PASS 또는 FAIL
                피드백: (FAIL인 경우만)
                """
            }]
        )

        eval_result = eval_response.content[0].text
        if "PASS" in eval_result:
            return {
                "translation": current_translation,
                "iterations": iteration,
                "status": "PASS"
            }

        feedback = eval_result

    # 최대 반복 도달
    return {
        "translation": current_translation,
        "iterations": iteration,
        "status": "MAX_ITERATIONS_REACHED"
    }
```

---

### 패턴 조합: 실제 프로덕션은 여러 패턴이 섞인다

실제 시스템은 단일 패턴이 아니라 여러 패턴을 조합해서 구성됩니다. 컴플라이언스 모니터링 도구를 예로 들면 이렇습니다.

```
[문서 수신]
    ↓
[Routing: 문서 유형 분류]
  계약서 / 규정 / 내부 정책
    ↓
[Parallelization: 문서 섹션 병렬 처리]
    ↓
[Orchestrator-Workers: 관련 규정 동적 조회 및 검토]
    ↓
[Prompt Chaining: 추출 → 분석 → 플래그 → 리포트 순서로]
    ↓
[Evaluator-Optimizer: QA 검토 후 최종 승인]
    ↓
[담당자에게 알림 발송]
```

---

## 4. 실제 적용 사례

### 금융: Capital One의 카드 온보딩 자동화

Capital One은 법인카드 온보딩 프로세스에 AI 워크플로를 적용했습니다. 기존에는 문서 검토, 정책 준수 확인, 부서 간 결재 라우팅을 모두 수동으로 처리했습니다.

AI 워크플로 적용 후:
- AI가 문서 스캔 및 데이터 검증 처리
- RPA가 미리 정의된 결재 단계를 자동 라우팅
- 전체 구조가 정해진 순서를 따르는 순수 워크플로 방식

결과: 처리 속도 개선과 정확도 향상. 관리자들은 반복 업무 대신 고부가가치 분석에 집중.

### 고객 서비스: Klarna의 에이전트 도입

스웨덴 핀테크 기업 Klarna는 자율 AI 에이전트를 도입했습니다. 이 에이전트는 고객 문의를 분석하고, 주문 정보를 조회하며, 환불을 처리하는 작업을 자율적으로 수행합니다.

결과: 700명 고객서비스 직원의 업무량을 처리하는 수준. 이 사례는 에이전트가 빛나는 상황을 보여줍니다. 고객 문의는 매번 다르고 동적인 판단이 필요하기 때문입니다.

### 의료 교육: 시뮬레이션 시나리오 자동 생성

의료 시뮬레이션 시나리오를 생성하는 멀티에이전트 워크플로가 구축됐습니다. Prompt Chaining, Parallelization, RAG, 반복 개선이 결합된 파이프라인이 사용됐습니다.

결과: 시나리오 개발 시간 **70~80% 단축**. INACSL, ASPiH 등 표준 지침을 일관되게 준수. 의료 전문가들이 별도 AI 전문 지식 없이도 복잡한 워크플로를 운영 가능.

### 콘텐츠 자동화: Copy.ai의 콘텐츠 머신

Copy.ai는 기획 → 작성 → 검토 → 게시까지 이어지는 콘텐츠 생성 워크플로를 구축했습니다. 기본 에이전트 기능(계획, 개선)을 활용하며, 일부 단계에서는 여전히 인간의 검토가 포함됩니다.

이 사례가 의미하는 것: 완전 자동화가 목표가 아닐 수 있습니다. **사람이 중요한 단계에 참여하는 Human-in-the-Loop 설계**가 현실적으로 더 안정적인 경우가 많습니다.

---

## 5. 도구 생태계 선택 가이드

2026년 현재 AI 워크플로 도구는 세 가지 계층으로 나뉩니다.

### 계층 1: 노코드/로우코드 자동화 도구

| 도구 | 특징 | 적합한 상황 |
| :--- | :--- | :--- |
| **Zapier** | 8,000+ 앱 통합, 가장 쉬움, 비쌈 | 비개발자, 단순 통합, SaaS 연결 |
| **Make (Integromat)** | 시각적 캔버스, 가격 대비 기능 우수 | 중간 복잡도 워크플로, 비주얼 설계 선호 |
| **n8n** | 오픈소스, 자체 호스팅 가능, 70+ AI 노드 | 개발자 포함 팀, 데이터 프라이버시 중요, 고량 처리 |

**2026년 시장 동향:** n8n이 2025년 중소기업 시장에서 10배 이상 성장하며 빠르게 점유율을 늘리고 있습니다. Zapier 고객의 80%가 n8n을 추가로 채택하는 것으로 나타났습니다.

**비용 비교 핵심:**
- Zapier: 액션 1개 = 태스크 1개 과금 (고량에서 매우 비쌈)
- Make: 모듈 실행 횟수로 과금 (중간)
- n8n: 워크플로 실행 횟수로 과금, 자체 호스팅 시 무료 (복잡한 워크플로일수록 유리)

### 계층 2: 개발자용 AI 에이전트 프레임워크

| 프레임워크 | 특징 | 적합한 상황 |
| :--- | :--- | :--- |
| **LangChain** | 가장 큰 생태계, 방대한 통합 | 빠른 프로토타이핑, 서드파티 연동 |
| **LangGraph** | 그래프 기반 상태 관리, 멀티에이전트 | 복잡한 멀티스텝, 상태 관리 필요 |
| **CrewAI** | 역할 기반 멀티에이전트, 선언적 | 팀 단위 에이전트, 빠른 구성 |
| **AutoGen** | 대화형, 에이전트 간 토론 | 품질 중요 오프라인 작업, Azure 연동 |
| **Google ADK** | 계층적 에이전트 트리, A2A 지원 | Google Cloud/Vertex AI 기반 |

### 계층 3: 엔터프라이즈 내구성 워크플로 플랫폼

| 플랫폼 | 특징 | 적합한 상황 |
| :--- | :--- | :--- |
| **Temporal** | 워크플로 내구성, 장기 실행 | 대규모 비즈니스 프로세스, 금융 |
| **AWS Step Functions** | AWS 네이티브, 시각적 상태 머신 | AWS 인프라 기반 엔터프라이즈 |
| **Inngest** | 이벤트 드리븐, 에이전트 친화적 | 에이전트 오케스트레이션, Human-in-the-Loop |

### 선택 가이드

```
내 상황에 맞는 도구는?

코딩 없이 바로 쓰고 싶다
    └── Zapier (가장 쉬움) 또는 Make (더 저렴)

데이터 프라이버시 / 자체 호스팅이 필요하다
    └── n8n (자체 호스팅 무료)

개발팀이 있고 AI 파이프라인을 직접 구축한다
    └── LangChain + LangGraph (가장 많이 쓰임)

멀티에이전트 팀 구성이 필요하다
    └── CrewAI 또는 AutoGen

Google Cloud / Vertex AI를 쓴다
    └── Google ADK

엔터프라이즈 규모의 내구성 있는 장기 실행 워크플로
    └── Temporal 또는 AWS Step Functions
```

---

## 6. 프로덕션에서의 주의사항

### Human-in-the-Loop (HITL) 설계

모든 단계를 자동화할 필요는 없습니다. 오히려 **중요한 단계에 사람을 포함**시키는 것이 전체 시스템의 신뢰성을 높입니다.

```
[좋은 HITL 설계 예시]

자동화됨:
  데이터 수집 → 전처리 → AI 분석 → 초안 생성

사람이 검토:
  ─→ [담당자 승인 게이트] ←─ 이메일/Slack 알림
              ↓ 승인
  자동화됨: 최종 배포 → 알림 발송
```

HITL이 특히 중요한 상황은 이렇습니다. 되돌리기 어려운 행동(메일 발송, DB 수정, 결제 처리), 고위험 판단(의료, 법률, 금융), 처음 배포하는 새 파이프라인처럼 아직 충분한 신뢰가 쌓이지 않은 경우입니다.

### 관측 가능성 (Observability)

워크플로/에이전트의 "블랙박스" 문제를 해결하는 것이 프로덕션의 핵심입니다.

**추천 도구:**
- **LangSmith** (LangChain 공식): LLM 호출 트레이싱, 비용 추적
- **Langfuse** (오픈소스): 비결정론적 에이전트 동작 디버깅
- **Arize Phoenix**: 에이전트 평가 및 모니터링

**최소한 로깅해야 할 것들:**

```python
# 각 LLM 호출마다 기록해야 하는 정보
log_entry = {
    "timestamp": "2026-05-07T10:30:00Z",
    "step": "risk_analysis",           # 워크플로 단계 이름
    "model": "claude-opus-4-5",        # 사용 모델
    "input_tokens": 1523,             # 입력 토큰 수
    "output_tokens": 487,             # 출력 토큰 수
    "latency_ms": 2340,               # 응답 시간
    "success": True,                  # 성공 여부
    "run_id": "wf_20260507_abc123",   # 워크플로 실행 ID (추적용)
}
```

### 비용 관리

에이전트 시스템은 LLM 호출이 많아질수록 비용이 빠르게 증가합니다. 실용적인 비용 관리 방법들을 알아봅니다.

**① 단계별 모델 차별화**

```python
# 분류 같은 간단한 작업 → 저렴한 모델
classifier = dspy.LM("claude-haiku-4-5")

# 복잡한 분석 → 고성능 모델
analyzer   = dspy.LM("claude-opus-4-5")
```

**② 캐싱 활용**

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_llm_call(prompt_hash: str, prompt: str) -> str:
    """동일한 프롬프트에 대한 반복 호출을 캐싱."""
    return llm(prompt)
```

**③ 루프에 최대 반복 횟수 설정**

```python
MAX_ITERATIONS = 5  # 반드시 상한선 설정
COST_LIMIT_USD = 2.0  # 워크플로 1회 실행당 비용 한도

current_cost = 0.0
for i in range(MAX_ITERATIONS):
    if current_cost >= COST_LIMIT_USD:
        break  # 비용 한도 초과 시 중단
    # ... LLM 호출
```

### 에러 처리와 복구

워크플로가 길어질수록 중간에 실패할 가능성이 높아집니다. 실용적인 에러 처리 패턴을 알아봅니다.

```python
import time
from typing import Optional

def llm_with_retry(
    prompt: str,
    max_retries: int = 3,
    fallback_response: Optional[str] = None
) -> str:
    """재시도 로직과 폴백이 포함된 안전한 LLM 호출."""
    for attempt in range(max_retries):
        try:
            response = client.messages.create(
                model="claude-opus-4-5",
                max_tokens=1000,
                messages=[{"role": "user", "content": prompt}]
            )
            return response.content[0].text

        except Exception as e:
            if attempt == max_retries - 1:
                # 마지막 시도 실패 → 폴백
                if fallback_response:
                    return fallback_response
                raise e
            # 지수 백오프로 재시도
            time.sleep(2 ** attempt)
```

---

## 7. 정리 및 시작 방법

### 핵심 개념 요약

| 개념 | 핵심 내용 |
| :--- | :--- |
| **AI 워크플로** | 개발자가 설계한 흐름대로 LLM + 도구를 연결하는 파이프라인 |
| **AI 에이전트** | LLM이 런타임에 스스로 다음 행동을 결정하는 자율 시스템 |
| **Prompt Chaining** | 이전 출력 → 다음 입력, 순차적 처리 |
| **Routing** | 입력 분류 → 전문 처리기로 라우팅 |
| **Parallelization** | 독립적 서브태스크를 동시에 실행 |
| **Orchestrator-Workers** | 오케스트레이터가 동적으로 태스크 위임 및 결과 종합 |
| **Evaluator-Optimizer** | 생성 → 평가 → 개선 반복 루프 |
| **HITL** | 중요한 단계에 사람이 검토·승인하는 설계 |

### 초보자를 위한 단계적 시작 방법

**Step 1: Prompt Chaining으로 시작** — 가장 단순하고 예측 가능한 패턴입니다. 기존에 수동으로 하던 3~5단계 작업을 그대로 코드로 옮겨보세요.

**Step 2: 체크포인트와 에러 처리 추가** — 각 단계 출력을 검증하고, 실패 시 재시도 또는 폴백 로직을 추가합니다.

**Step 3: 관측 가능성 구축** — LangSmith 또는 Langfuse를 연결해 호출 내역을 추적합니다. 나중에 반드시 필요한 인프라입니다.

**Step 4: 복잡도 필요한 곳에만 Routing 또는 Parallelization 추가** — 단순하게 시작해서 필요할 때만 패턴을 추가합니다.

**Step 5: 에이전트는 마지막에** — 워크플로로 해결할 수 없는 동적 판단이 필요한 부분에만 에이전트를 도입합니다.

---

AI 워크플로의 핵심 인사이트는 이것입니다. LLM은 강력하지만, **그 강력함이 제대로 발휘되려면 잘 설계된 구조가 필요합니다.** 프롬프트 한 줄로 모든 것을 처리하려는 것은 마치 CPU 하나로 운영체제 없이 소프트웨어를 실행하려는 것과 같습니다. AI 워크플로는 그 운영체제입니다.

> {: .prompt-info }
> **참고 자료**
> - Anthropic: [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)
> - Anthropic: [Multi-Agent Research System](https://www.anthropic.com/engineering/multi-agent-research-system)
> - Vellum: [Agentic Workflows in 2026](https://www.vellum.ai/blog/agentic-workflows-emerging-architectures-and-design-patterns)
> - Redis: [AI Agents vs Workflows](https://redis.io/blog/agents-vs-workflows/)
> - n8n: [Best AI Workflow Automation Tools](https://blog.n8n.io/best-ai-workflow-automation-tools/)
