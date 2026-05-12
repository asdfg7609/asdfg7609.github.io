---
layout: post
title: "Pydantic AI"
date: 2026-05-12 16:00:00 +0900
categories:
  - LLM
  - Agent
tags: [PydanticAI, LLM, Agent, TypeSafety, DependencyInjection, FastAPI]
math: false
---

> {: .prompt-tip }
> **요약:** Pydantic AI는 Pydantic 팀이 만든 타입 안전 AI 에이전트 프레임워크입니다. FastAPI가 웹 개발에 가져온 것과 같은 개발자 경험을 AI 에이전트 개발에 적용하는 것이 목표입니다. 구조화된 출력, 의존성 주입, API 키 없이 동작하는 TestModel이 핵심 차별점입니다. 2025년 9월 V1 출시 후 2026년 5월 기준 16,500개 이상의 GitHub 스타를 기록하며 빠르게 성장 중입니다.

---

LLM을 활용해 에이전트를 개발하다 보면 어느 순간 이런 생각이 듭니다.

> "에이전트 출력이 문자열인데, 이게 올바른 형식인지 어떻게 보장하지?"
> "도구(tool)에 DB 연결을 주입하려면 전역 변수밖에 없나?"
> "이 에이전트를 테스트하려면 API 키를 써야 하나?"

기존 프레임워크들은 이 질문들에 명확한 답을 주지 않습니다. **Pydantic AI는 이 세 가지를 전부 해결하는 것을 출발점으로 설계된 프레임워크입니다.**

---

## 1. AI 에이전트 프레임워크의 공통 불편함

Pydantic AI를 이해하려면 기존 방식이 어떤 문제를 갖고 있는지 먼저 봐야 합니다.

### 문제 1: 출력이 문자열이다

LLM을 직접 호출하든, LangChain을 쓰든 대부분의 에이전트 출력은 문자열입니다. 개발자는 이 문자열을 직접 파싱해야 합니다.

```python
# 기존 방식 — 출력을 직접 파싱해야 함
response = llm.invoke("영화 Inception을 리뷰해줘. JSON 형식으로 title, rating, summary를 포함해서.")

# 이게 올바른 JSON인지? rating이 숫자인지 문자열인지?
# LLM이 "rating: eight"이라고 했다면? 파싱에서 터짐
import json
try:
    data = json.loads(response.content)
    rating = data["rating"]  # 런타임에 터질 수 있음
except:
    # 에러 처리를 직접 해야 함
    pass
```

문자열 파싱은 깨지기 쉽습니다. LLM이 조금이라도 다른 형식으로 답하면 런타임 에러가 납니다.

### 문제 2: 도구에 외부 의존성 주입이 어렵다

도구(tool)는 보통 DB 연결, API 클라이언트, 사용자 정보 등이 필요합니다. 기존 방식에서는 이것을 전달하기가 어렵습니다.

```python
# 기존 방식 — 전역 변수 또는 클로저에 의존
db_connection = get_database_connection()  # 전역 상태

@tool
def get_user_info(user_id: str) -> dict:
    # 전역 변수 사용 → 테스트하기 어렵고, 재사용하기 어렵다
    return db_connection.query(f"SELECT * FROM users WHERE id = '{user_id}'")
```

전역 상태는 테스트를 어렵게 만들고, 의존성이 코드 어디에 있는지 추적하기 어렵습니다.

### 문제 3: 에이전트 테스트에 API 키가 필요하다

에이전트를 테스트하려면 실제 LLM을 호출해야 합니다. 이것은 세 가지 문제를 만듭니다. 첫째, API 비용이 발생합니다. 둘째, 응답이 비결정론적이라 테스트가 불안정합니다. 셋째, CI/CD에서 API 키 관리가 필요합니다.

```python
# 기존 방식 — 테스트마다 실제 API 호출 발생
def test_agent():
    result = agent.invoke("질문")  # 매번 API 비용 발생, 응답이 매번 다름
    assert "정답" in result        # 불안정한 테스트
```

---

## 2. Pydantic AI란 무엇인가

### 탄생 배경

Pydantic AI는 **Pydantic 팀**이 직접 만든 Python AI 에이전트 프레임워크입니다. 공식 문서에 이 동기가 명확하게 나와 있습니다.

> "FastAPI는 Pydantic 기반의 타입 힌트를 활용해 웹 개발에 혁신을 가져왔습니다. 그런데 OpenAI SDK, Anthropic SDK, LangChain, CrewAI, LlamaIndex 등 사실상 모든 Python AI 프레임워크가 내부적으로 Pydantic을 사용하고 있음에도, Pydantic 팀이 LLM을 직접 사용해보니 FastAPI와 같은 느낌을 주는 프레임워크가 없었습니다. 우리는 그것을 만들기로 했습니다."

FastAPI가 웹 API 개발에 한 것을, AI 에이전트 개발에 적용하는 것이 목표입니다.

### 주요 지표

| 항목 | 현황 (2026년 5월 기준) |
| :--- | :--- |
| GitHub 스타 | 16,500+ |
| 출시 | 2024년 말 얼리 액세스 |
| V1 정식 출시 | 2025년 9월 4일 |
| 최신 버전 | v1.88.0 (2026년 4월 29일) |
| 지원 모델 | 25개+ (OpenAI, Anthropic, Gemini, Ollama, Bedrock 등) |
| 라이선스 | MIT |

### 핵심 철학: "에이전트가 돌려주는 값이 항상 내가 기대한 형태임을 보장한다."

Pydantic AI에서 에이전트는 FastAPI의 앱 객체처럼 선언적으로 정의됩니다. 입력 타입, 출력 타입, 의존성 타입이 모두 제네릭으로 명시됩니다.

```python
from pydantic_ai import Agent

# 이 에이전트의 타입은 Agent[SupportDependencies, SupportOutput]
# 의존성과 출력이 모두 타입으로 표현됨
support_agent = Agent(
    'anthropic:claude-opus-4-5',
    deps_type=SupportDependencies,
    output_type=SupportOutput,
    instructions='고객 지원 에이전트입니다.',
)
```

mypy나 pyright 같은 정적 타입 체커가 에이전트 코드의 버그를 배포 전에 잡아줍니다.

---

## 3. 4가지 핵심 개념

### 개념 1: 구조화된 출력 (Structured Output)

`output_type`에 Pydantic 모델을 지정하면 LLM 출력이 자동으로 검증됩니다. 형식이 맞지 않으면 에이전트가 자동으로 재시도합니다.

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent

# ── Before: 문자열 파싱 (깨지기 쉬움) ──────────────────────────────
response = llm.invoke("영화를 리뷰해줘. JSON으로.")
data = json.loads(response)          # LLM이 다른 형식으로 답하면 터짐
rating = int(data.get("rating", 0))  # 수동 변환, 검증 없음

# ── After: Pydantic AI (타입 안전) ──────────────────────────────────
class MovieReview(BaseModel):
    title:  str
    rating: int   = Field(ge=1, le=10, description="1~10점 사이의 평점")
    summary: str  = Field(max_length=200, description="200자 이내의 요약")

agent = Agent(
    'anthropic:claude-opus-4-5',
    output_type=MovieReview,  # ← 이것만 추가하면 끝
)

result = agent.run_sync("영화 Inception을 리뷰해줘.")

# result.output은 MovieReview 인스턴스 (검증 완료)
print(result.output.title)   # "Inception"   (str이 보장됨)
print(result.output.rating)  # 8             (int, 1~10 사이 보장됨)
print(result.output.summary) # "..."         (200자 이내 보장됨)

# LLM이 rating: "eight"으로 답해도 → Pydantic이 잡아서 재시도
# LLM이 rating: 15로 답해도 → Field(le=10) 위반으로 재시도
```

> {: .prompt-info }
> **재시도는 자동입니다.** 검증에 실패하면 Pydantic AI가 에러 메시지를 LLM에 다시 보내 수정을 요청합니다. 개발자가 재시도 로직을 직접 구현할 필요가 없습니다.

### 개념 2: 타입 힌트 = 도구 스키마

`@agent.tool` 데코레이터가 함수의 타입 힌트와 docstring을 읽어 LLM에게 전달할 JSON Schema를 자동으로 생성합니다. 수동 스키마 작성이 필요 없습니다.

```python
import random
from pydantic_ai import Agent, RunContext

agent = Agent('anthropic:claude-opus-4-5', instructions="주사위 게임 진행자입니다.")

# ── Before: 수동 스키마 작성 필요 ─────────────────────────────────
tools = [{
    "name": "roll_dice",
    "description": "주사위를 굴립니다.",
    "parameters": {
        "type": "object",
        "properties": {
            "sides": {"type": "integer", "description": "주사위 면 수"}
        }
    }
}]

# ── After: 타입 힌트가 곧 스키마 ──────────────────────────────────
@agent.tool_plain
def roll_dice(sides: int = 6) -> int:
    """주사위를 굴립니다.

    Args:
        sides: 주사위 면 수 (기본값: 6)
    """
    return random.randint(1, sides)

# Pydantic AI가 타입 힌트와 docstring을 읽어 자동으로 스키마 생성
# JSON Schema, 파라미터 설명, 반환 타입이 모두 자동 처리됨
```

### 개념 3: 의존성 주입 (Dependency Injection)

FastAPI의 `Depends()`와 동일한 철학입니다. `RunContext`를 통해 DB 연결, API 클라이언트, 사용자 정보 등을 전역 변수 없이 도구에 전달합니다.

```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext

# 의존성을 dataclass로 선언
@dataclass
class SupportDeps:
    customer_id: int
    db: DatabaseConnection   # DB 연결을 전역 변수 없이 주입

agent = Agent(
    'anthropic:claude-opus-4-5',
    deps_type=SupportDeps,   # 에이전트에 의존성 타입 등록
)

@agent.tool
async def get_customer_balance(ctx: RunContext[SupportDeps]) -> float:
    """고객의 계좌 잔액을 조회합니다."""
    # ctx.deps로 주입된 의존성에 접근
    return await ctx.deps.db.get_balance(ctx.deps.customer_id)

@agent.tool
async def get_customer_name(ctx: RunContext[SupportDeps]) -> str:
    """고객 이름을 조회합니다."""
    return await ctx.deps.db.get_name(ctx.deps.customer_id)

# 실행 시 의존성을 주입
result = await agent.run(
    "내 잔액을 확인하고 싶어요.",
    deps=SupportDeps(customer_id=123, db=real_db_connection),
)

# 테스트 시: 가짜 DB로 교체
result = await agent.run(
    "내 잔액을 확인하고 싶어요.",
    deps=SupportDeps(customer_id=123, db=mock_db_connection),  # Mock 주입
)
```

> {: .prompt-tip }
> **mypy가 잡아줍니다.** `ctx: RunContext[SupportDeps]`의 타입 어노테이션이 잘못되면 정적 타입 체커가 에러를 표시합니다. 런타임이 아니라 IDE에서 버그를 발견할 수 있습니다.

### 개념 4: TestModel — API 키 없이 에이전트 테스트

Pydantic AI 가장 독창적인 기능입니다. `TestModel`은 실제 LLM을 대체하는 Mock 모델로, API 호출 없이 에이전트의 구조와 도구 호출 로직을 테스트할 수 있습니다.

```python
import pytest
from pydantic_ai import Agent
from pydantic_ai.models.test import TestModel
from pydantic_ai.models.function import FunctionModel, ModelContext
from pydantic_ai.messages import ModelResponse, TextPart

# 에이전트 정의
agent = Agent(
    'anthropic:claude-opus-4-5',
    output_type=MovieReview,
)

# ── TestModel: API 키 없이 구조 검증 ─────────────────────────────
def test_agent_structure():
    """에이전트가 올바른 출력 구조를 반환하는지 확인. API 호출 없음."""
    with agent.override(model=TestModel()):
        result = agent.run_sync("영화 리뷰해줘.")
        # TestModel은 output_type 스키마에 맞는 Mock 데이터를 자동 생성
        assert isinstance(result.output, MovieReview)
        assert isinstance(result.output.rating, int)
        # 밀리초 단위로 실행됨 (API 호출 없음)

# ── FunctionModel: 원하는 응답 직접 지정 ─────────────────────────
def test_agent_response_logic():
    """특정 응답에 대한 에이전트 동작을 테스트."""

    def mock_model(ctx: ModelContext, _) -> ModelResponse:
        # LLM 응답을 직접 지정
        return ModelResponse(parts=[TextPart(
            '{"title": "Inception", "rating": 9, "summary": "훌륭한 영화"}'
        )])

    with agent.override(model=FunctionModel(mock_model)):
        result = agent.run_sync("영화 리뷰해줘.")
        assert result.output.title == "Inception"
        assert result.output.rating == 9

# ── pytest fixture로 재사용 ───────────────────────────────────────
@pytest.fixture
def override_agent():
    with agent.override(model=TestModel()):
        yield

def test_with_fixture(override_agent):
    result = agent.run_sync("리뷰해줘.")
    assert result.output is not None
```

TestModel의 작동 방식이 영리합니다. 실제 ML 모델이 아닙니다. 등록된 도구와 `output_type`의 JSON Schema를 읽어, 타입 검증을 통과하는 Mock 데이터를 순수 Python 코드로 생성합니다. API 없이도 에이전트 전체 흐름을 검증할 수 있습니다.

---

## 4. 다른 프레임워크와 비교

각 프레임워크의 철학을 한 줄로 정리하면 이렇습니다.

| 프레임워크 | 에이전트는... | 2026 포지션 |
| :--- | :--- | :--- |
| **LangGraph** | 상태 기계(그래프)다 | 복잡한 워크플로, 정밀 제어, 엔터프라이즈 |
| **CrewAI** | 역할 기반 팀원이다 | 빠른 멀티에이전트 프로토타이핑 |
| **AutoGen/AG2** | 대화 참여자다 | 연구·실험용, 프로덕션 사용 감소 추세 |
| **Pydantic AI** | 타입 안전한 Python 객체다 | 코드 품질·테스트 중시, 멀티 LLM |

### LangGraph와의 비교

LangGraph의 에이전트는 `TypedDict`로 정의된 상태(State)와 `StateGraph`의 노드·엣지로 표현됩니다.

```python
# LangGraph: 에이전트 = 명시적 상태 그래프
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]

graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue)
graph.add_edge("tools", "agent")
compiled = graph.compile()

# Pydantic AI: 에이전트 = Python 클래스 객체
from pydantic_ai import Agent

agent = Agent(
    'anthropic:claude-opus-4-5',
    output_type=MyOutput,
    deps_type=MyDeps,
)
# 그래프 없이도 도구 호출 루프가 자동으로 처리됨
```

LangGraph는 복잡한 조건 분기, 사이클, Human-in-the-Loop를 명시적 그래프로 제어해야 할 때 강점이 있습니다. Pydantic AI는 에이전트 내부의 도구 호출 루프를 프레임워크가 처리하고, 개발자는 비즈니스 로직에 집중합니다.

### CrewAI와의 비교

```python
# CrewAI: 에이전트 = 역할과 목표를 가진 팀원
from crewai import Agent, Task, Crew

researcher = Agent(
    role='Research Analyst',
    goal='최신 AI 동향을 조사한다',
    backstory='10년 경력의 AI 연구자',
    tools=[search_tool],
)

# Pydantic AI: 에이전트 = 타입 시스템으로 정의된 Python 객체
from pydantic_ai import Agent

research_agent = Agent(
    'anthropic:claude-opus-4-5',
    output_type=ResearchResult,  # 역할이 아닌 타입으로 정의
    instructions='최신 AI 동향을 조사합니다.',
)
```

CrewAI는 "리서처, 분석가, 작성자"처럼 역할 기반으로 멀티에이전트를 빠르게 구성할 때 직관적입니다. Pydantic AI는 각 에이전트의 출력 타입이 명확히 정의되어야 하는 프로덕션 시스템에 강합니다.

### 결정 트리

```
에이전트를 만들어야 하는데...

복잡한 조건 분기, 사이클, 재귀, 명시적 Human-in-the-Loop가 필요한가?
  └── YES → LangGraph

"리서처, 분석가, 작성자" 같은 역할 기반 팀 구성을 빠르게 하고 싶은가?
  └── YES → CrewAI

타입 안전성, 테스트 가능성, 구조화된 출력이 최우선인가?
  └── YES → Pydantic AI

다수의 LLM 공급자를 교체하며 쓰거나 FastAPI와 통합해야 하는가?
  └── YES → Pydantic AI

무엇을 선택해야 할지 모르겠다. 처음 만들어보는 에이전트다.
  └── Pydantic AI (단순하고 직관적, FastAPI 경험자라면 특히)
```

---

## 5. 실전 예시: 은행 고객 지원 에이전트

Pydantic AI 공식 문서의 대표 예제입니다. 4가지 핵심 개념이 모두 자연스럽게 담겨 있습니다.

```python
from dataclasses import dataclass
from pydantic import BaseModel, Field
from pydantic_ai import Agent, RunContext

# ── 의존성 정의 ───────────────────────────────────────────────────
@dataclass
class SupportDependencies:
    customer_id: int
    db: DatabaseConn   # DB 연결을 전역 변수 없이 주입 (개념 3)

# ── 출력 스키마 정의 ──────────────────────────────────────────────
class SupportOutput(BaseModel):             # (개념 1: 구조화된 출력)
    support_advice: str = Field(description="고객에게 제공할 조언")
    block_card:     bool = Field(description="카드를 차단해야 하는가")
    risk:           int  = Field(ge=0, le=10, description="리스크 수준 0~10")

# ── 에이전트 정의 ─────────────────────────────────────────────────
support_agent = Agent(
    'anthropic:claude-opus-4-5',
    deps_type=SupportDependencies,     # 의존성 타입 등록
    output_type=SupportOutput,         # 출력 타입 등록
    instructions='은행 고객 지원 에이전트입니다.',
)

# ── 동적 지시문 (의존성 활용) ────────────────────────────────────
@support_agent.instructions
async def add_customer_name(ctx: RunContext[SupportDependencies]) -> str:
    """DB에서 고객 이름을 조회해 지시문에 추가."""
    name = await ctx.deps.db.customer_name(id=ctx.deps.customer_id)
    return f"고객 이름: {name!r}"

# ── 도구 정의 (타입 힌트 = 스키마) ──────────────────────────────
@support_agent.tool        # (개념 2: 타입 힌트가 곧 스키마)
async def get_balance(ctx: RunContext[SupportDependencies]) -> float:
    """고객의 현재 계좌 잔액을 조회합니다."""
    return await ctx.deps.db.customer_balance(id=ctx.deps.customer_id)

@support_agent.tool
async def get_recent_transactions(
    ctx: RunContext[SupportDependencies],
    limit: int = 5,         # ← 이 타입 힌트가 LLM에게 전달되는 파라미터 스키마
) -> list[dict]:
    """최근 거래 내역을 조회합니다."""
    return await ctx.deps.db.recent_transactions(
        id=ctx.deps.customer_id,
        limit=limit,
    )

# ── 실행 ─────────────────────────────────────────────────────────
async def main():
    deps = SupportDependencies(customer_id=42, db=real_db)
    result = await support_agent.run(
        "카드가 도용된 것 같아요.",
        deps=deps,
    )

    print(result.output.support_advice)  # "카드를 즉시 차단하겠습니다..."
    print(result.output.block_card)      # True    (bool이 보장됨)
    print(result.output.risk)            # 9       (0~10 사이 보장됨)

# ── 테스트 (API 키 없이) ─────────────────────────────────────────
from pydantic_ai.models.test import TestModel   # (개념 4: TestModel)

async def test_support_agent():
    mock_deps = SupportDependencies(
        customer_id=42,
        db=MockDatabaseConn(),   # 가짜 DB로 교체
    )
    with support_agent.override(model=TestModel()):
        result = await support_agent.run("잔액이 얼마인가요?", deps=mock_deps)
        assert isinstance(result.output.risk, int)
        assert 0 <= result.output.risk <= 10
        # API 키 없이, 비용 없이, 밀리초 만에 완료
```

---

## 6. 한계와 언제 다른 프레임워크를 쓸까

Pydantic AI를 솔직하게 평가하면 다음 한계가 있습니다.

**Pydantic v2 필수**

Pydantic v1 레거시 코드베이스에서는 마이그레이션 비용이 발생합니다.

**스트리밍 구조화 출력이 베타**

`output_type`을 지정한 상태에서 스트리밍 응답을 받는 기능은 2026년 5월 기준 여전히 베타입니다. 스트리밍이 중요한 실시간 애플리케이션에서는 주의가 필요합니다.

**공식 관측 도구(Logfire)는 유료**

Pydantic 팀의 관측 플랫폼 Logfire는 유료입니다. OpenTelemetry를 통한 대안은 가능하지만 공식 문서화가 부족합니다.

**복잡한 상태 기계에는 LangGraph가 낫다**

에이전트가 여러 단계를 거치며 이전 상태를 참조하고, 조건 분기가 복잡하고, 인간이 중간에 개입하는 구조라면 LangGraph의 명시적 그래프가 더 적합합니다.

**새로운 프레임워크의 빠른 변화**

V1이 출시됐지만 여전히 빠르게 변화하고 있습니다. `result_type`이 `output_type`으로 바뀐 것처럼 마이너 버전에서도 API가 변경될 수 있습니다. 프로덕션에서는 버전을 고정(`pydantic-ai==1.88.0`)하는 것을 권장합니다.

---

## 정리

| 개념 | 핵심 내용 |
| :--- | :--- |
| **구조화된 출력** | `output_type=MyModel`로 LLM 출력을 Pydantic으로 검증. 실패 시 자동 재시도 |
| **타입 힌트 = 스키마** | `@agent.tool` 데코레이터가 함수 타입 힌트를 읽어 JSON Schema 자동 생성 |
| **의존성 주입** | `RunContext[Deps]`로 DB·API 클라이언트를 전역 변수 없이 주입 |
| **TestModel** | API 키 없이 에이전트 구조·도구 로직 테스트. 밀리초 단위, 결정론적 |
| **모델 무관성** | 모델 문자열 하나만 바꾸면 Claude → GPT → Gemini → Ollama 교체 가능 |

Pydantic AI가 내놓는 핵심 주장은 이것입니다.

> **"에이전트 코드가 일반 Python 코드처럼 읽혀야 한다. 타입 체커가 버그를 잡아야 한다. 테스트가 API 키 없이 돌아야 한다."**

FastAPI를 처음 써봤을 때의 "이렇게 깔끔할 수 있구나"라는 느낌을 기억한다면, Pydantic AI는 그 느낌을 AI 에이전트 개발에서 재현하려는 시도입니다. 처음 에이전트를 만들거나, LangChain의 복잡한 추상화에 지쳤다면 시작점으로 좋은 선택입니다.

> {: .prompt-info }
> **참고 자료**
> - 공식 문서: [https://ai.pydantic.dev](https://ai.pydantic.dev)
> - GitHub: [https://github.com/pydantic/pydantic-ai](https://github.com/pydantic/pydantic-ai)
> - V1 출시 블로그: [https://pydantic.dev/articles/pydantic-ai-v1](https://pydantic.dev/articles/pydantic-ai-v1)
> - Testing 가이드: [https://ai.pydantic.dev/testing](https://ai.pydantic.dev/testing)
> - 프레임워크 비교 (DEV Community): [The 2026 AI Agent Framework Decision Guide](https://dev.to/linou518/the-2026-ai-agent-framework-decision-guide-langgraph-vs-crewai-vs-pydantic-ai-b2h)
