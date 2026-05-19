---
layout: post
title: "Claude Code가 내 프로젝트 규칙을 기억하게 만드는 법"
date: 2026-05-19 22:00:00 +0900
categories:
  - AI
  - ClaudeCode
tags: [ClaudeCode, CLAUDE.md, ContextEngineering, Karpathy, AICoading]
math: false
---

> {: .prompt-tip }
> **요약:** Claude Code는 세션이 시작될 때마다 컨텍스트가 초기화됩니다. CLAUDE.md는 이 문제를 해결하는 파일로, 프로젝트 규칙과 작업 방식을 Claude에게 영구적으로 주입합니다. 이 글에서는 CLAUDE.md를 제대로 작성하는 방법, Andrej Karpathy의 4가지 원칙, 그리고 실제로 Claude Code를 더 잘 쓸 수 있는 실용적인 팁들을 정리합니다.

---

Claude Code를 처음 쓰다 보면 어느 순간 이런 상황이 옵니다.

> "이 프로젝트는 TypeScript 씁니다."  
> "테스트는 항상 pytest로 실행해주세요."  
> "절대로 `console.log`를 production 코드에 남기지 마세요."

새 세션을 열 때마다 같은 말을 반복합니다. Claude Code는 대화가 끝나면 그 맥락을 전혀 기억하지 못합니다. 매번 신입 직원에게 팀 규칙을 처음부터 설명하는 것과 같습니다.

이 문제를 해결하는 것이 **CLAUDE.md**입니다.

---

## 1. CLAUDE.md가 무엇인가

CLAUDE.md는 Claude Code가 세션을 시작할 때 **자동으로 읽는 마크다운 파일**입니다. 프로젝트 루트에 놓으면 그 프로젝트에서, 홈 디렉터리(`~/.claude/`)에 놓으면 모든 프로젝트에서 적용됩니다.

```
프로젝트 루트/
├── CLAUDE.md          ← 프로젝트별 규칙
├── src/
└── tests/

~/.claude/
└── CLAUDE.md          ← 전역 규칙 (모든 프로젝트에 적용)
```

두 파일 모두 있으면 전역 → 프로젝트 순서로 둘 다 읽습니다.

2026년 현재 개발자 커뮤니티의 공통된 평가는 이렇습니다.

> "CLAUDE.md는 이제 .gitignore만큼 필수 인프라다."

처음 만드는 가장 빠른 방법은 `/init` 명령어입니다. Claude Code가 현재 프로젝트 구조를 분석해 빌드 시스템, 테스트 프레임워크, 코드 패턴을 자동으로 감지하고 초안을 만들어줍니다. 이후 필요에 따라 수정하면 됩니다.

```bash
> /init
```

---

## 2. 무엇을 넣고, 무엇을 넣지 말아야 하는가

CLAUDE.md를 처음 만들면 "최대한 많이 넣어야겠다"는 생각이 드는데, 이것이 가장 흔한 실수입니다.

### 왜 짧아야 하는가: 토큰 예산의 현실

Claude Code는 세션이 시작될 때 시스템 프롬프트, 도구 정의, CLAUDE.md를 로딩하는 데만 약 **2만 토큰**을 소비합니다. Claude의 총 컨텍스트 윈도우는 20만 토큰이지만 실제로 작업에 쓸 수 있는 공간은 그보다 훨씬 적습니다.

더 중요한 것은 Claude가 모든 지시를 동등하게 따르지 않는다는 점입니다. 프론티어 모델이 합리적으로 따를 수 있는 지시는 약 **150~200개** 수준이며, Claude Code 시스템 프롬프트가 이 중 50개를 차지합니다. 실제로 여러분이 쓸 수 있는 슬롯은 **100~150개** 정도입니다.

HumanLayer가 권장하는 CLAUDE.md 길이는 **60줄 이하**, 커뮤니티 일반 합의는 **300줄 이하**입니다.

### 넣어야 하는 것

```markdown
# CLAUDE.md

## 빌드 및 실행 명령어
- 개발 서버: `npm run dev`
- 테스트: `pytest tests/ -v`
- 린트: `ruff check . && mypy .`
- 빌드: `npm run build`

## 아키텍처 규칙
- 비즈니스 로직은 domain/ 레이어에만 위치
- 외부 API 호출은 반드시 adapters/ 레이어를 통해서만
- LLM 관련 코드에 특정 프레임워크 import 금지 (domain/ 내부)

## Claude가 자주 틀리는 것
- `__init__.py`를 루트에 만들지 않기
- 테스트 파일명은 반드시 `test_`로 시작
- 환경변수는 코드에 하드코딩하지 않기

## 금지 사항
- `rm -rf` 사용 금지
- 프로덕션 환경변수 직접 수정 금지
- 되돌릴 수 없는 행동 전 반드시 확인
```

**넣어야 하는 것들:**
- 빌드, 테스트, 린트 실행 명령어
- 아키텍처 결정 사항 (어디에 무엇을 두는지)
- Claude가 반복적으로 틀리는 패턴
- 절대 하면 안 되는 행동

### 넣지 말아야 하는 것

**① 린터가 할 일을 Claude에게 시키지 않기**

```markdown
# ❌ 이런 내용은 불필요
- 항상 세미콜론으로 문장을 끝내세요
- 들여쓰기는 2칸을 사용하세요
- 변수명은 camelCase를 사용하세요
```

코드 스타일은 ESLint, ruff, prettier 같은 도구에 맡겨야 합니다. CLAUDE.md에 스타일 규칙을 쓰면 토큰만 낭비하고 Claude는 어차피 실제 코드 패턴을 따라갑니다.

**② 파일 전체를 @import로 끌어오지 않기**

```markdown
# ❌ 매 세션마다 전체 파일을 context에 올림
@api-conventions.md
@database-schema.md
@architecture-guide.md
```

파일 전체를 임베드하면 매 세션마다 그 내용이 전부 컨텍스트에 올라갑니다. 대신 참조 방식을 씁니다.

```markdown
# ✅ 필요할 때 찾아보도록 안내
복잡한 API 사용법이나 FooBarError 처리는 docs/api-conventions.md 참조
```

**③ 불필요하게 많은 규칙 넣지 않기**

각 줄을 넣기 전에 스스로 물어보세요: **"이걸 빼면 Claude가 실수를 하는가?"** 아니라면 빼세요.

---

## 3. /karpathy-guidelines — 60,000개의 별을 받은 단 하나의 파일

2026년 1월, Andrej Karpathy(전 Tesla AI 디렉터, 전 OpenAI)가 AI 코딩 에이전트의 실패 패턴을 공개적으로 정리했습니다.

> "모델이 여러분 대신 잘못된 가정을 하고 그냥 달려나갑니다. 혼란을 관리하지 않고, 명확한 설명을 구하지 않고, 불일치를 드러내지 않고, 트레이드오프를 제시하지 않습니다."

> "코드와 API를 지나치게 복잡하게 만들고, 추상화를 부풀리고, 죽은 코드를 정리하지 않습니다. 100줄이면 될 것을 1000줄로 구현합니다."

> "자신이 충분히 이해하지 못하는 코드나 주석을 부작용으로 변경하거나 제거합니다."

개발자 Forrest Chang이 이 관찰을 하나의 CLAUDE.md 파일로 만들었고, 이 저장소는 **60,000개 이상의 GitHub 스타**를 받았습니다. 프레임워크도, 라이브러리도, 앱도 아닌 단 하나의 마크다운 파일이 그 정도 반응을 받았다는 것은 이 문제가 얼마나 보편적인지를 보여줍니다.

이 파일은 Claude Code 플러그인으로 설치하거나, 프로젝트 CLAUDE.md에 직접 추가할 수 있습니다.

```bash
# 플러그인으로 설치 (모든 프로젝트에 적용)
/plugin marketplace add forrestchang/andrej-karpathy-skills
/plugin install andrej-karpathy-skills@karpathy-skills

# 프로젝트에 직접 추가
curl -o CLAUDE.md https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md

# 기존 CLAUDE.md에 추가
curl https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md >> CLAUDE.md
```

### Karpathy 4원칙

**원칙 1: Think Before Coding (코딩 전에 생각하라)**

가정을 명시적으로 밝히고, 프롬프트가 모호하면 여러 해석을 제시하고, 더 단순한 방법이 있으면 반론을 제기하고, 불명확한 것이 있으면 추측하지 말고 물어보도록 강제합니다.

```
# ❌ Claude가 기본적으로 하는 것
모호한 요청 → 가정 → 바로 구현

# ✅ Karpathy 원칙 적용 후
모호한 요청 → "이렇게 이해했는데 맞나요?"
            → 해석 A, B, C 중 어떤 것인가요?
            → 확인 후 구현
```

**원칙 2: Simplicity First (단순함 우선)**

요청을 해결하는 최소한의 코드만 작성합니다. 추가 기능, 불필요한 추상화, 요청하지 않은 설정 옵션을 만들지 않습니다. 시니어 엔지니어가 "이거 왜 이렇게 복잡해?"라고 할 만한 코드는 쓰지 않습니다.

**원칙 3: Surgical Changes (외과적 변경)**

요청받은 것만 변경합니다. 관련 없는 코드, 주석, 함수에 손대지 않습니다. 변경 범위를 최소화합니다. 코드를 완전히 이해하지 못한 상태에서 수정하지 않습니다.

**원칙 4: Goal-Driven Execution (목표 기반 실행)**

> "LLM은 구체적인 목표를 향해 루프를 돌 때 특히 뛰어납니다. 무엇을 할지 알려주는 대신 성공 기준을 주고 지켜보세요."

단계별 지시가 아닌 검증 가능한 목표를 줍니다. 테스트를 먼저 작성하고 그것이 통과하도록 만들게 합니다.

```
# ❌ 지시 방식
"1단계: 파일을 열어서, 2단계: 함수를 찾아서, 3단계: 수정해서..."

# ✅ 목표 방식
"이 함수가 빈 리스트 입력 시 빈 리스트를 반환하는 테스트를 통과시켜줘."
```

---

## 4. Claude Code를 더 잘 쓰는 실용적인 팁들

### 팁 1: Plan Mode를 먼저 쓰고 구현하라

Plan Mode는 Claude가 파일을 읽고 분석하되, **어떤 변경도 하지 않는** 읽기 전용 모드입니다. 복잡한 작업 전에 항상 계획을 먼저 세우게 하면 잘못된 방향으로 달려나가는 것을 방지합니다.

```bash
# Shift+Tab 또는 명령어로 Plan Mode 전환
> /plan

# Plan Mode에서 Claude가 하는 것:
# - 코드베이스 전체를 분석
# - 어떤 파일을 어떻게 수정할지 계획 수립
# - 사용자가 승인하기 전까지 실행 안 함

# 계획을 확인하고 승인하면 구현으로 진행
```

판단 기준으로 삼기 좋은 규칙이 있습니다: **"변경 내용을 한 문장으로 설명할 수 있으면 그냥 진행, 없으면 먼저 Plan Mode"**

### 팁 2: 컨텍스트가 60%를 넘기 전에 /clear 하라

Claude Code의 컨텍스트 품질은 **20~40% 채워지는 시점부터 떨어지기 시작**합니다. 83.5%에서 자동 압축이 실행되는데, 이 자동 압축은 원래 내용의 20~30%만 보존하는 손실 압축입니다.

```bash
# 세션이 길어지면 현재 상태를 파일에 저장 후 초기화
> 지금까지의 진행 상황과 다음 해야 할 일을 progress.md에 저장해줘
> /clear
> progress.md를 읽고 이어서 진행해줘
```

실용적인 규칙: **한 작업이 끝나면 /clear로 초기화**합니다. 세션 초기화 비용은 약 2만 토큰입니다. 하지만 컨텍스트가 오염된 세션을 계속 쓰는 비용은 그보다 훨씬 큽니다.

### 팁 3: Hooks로 규칙을 강제하라

CLAUDE.md의 지시는 "권고"입니다. Claude가 따르지 않을 수 있습니다. Hooks는 "강제"입니다. Claude가 반드시 따라야 합니다.

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint -- --fix"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "npm test -- --passWithNoTests"
          }
        ]
      }
    ]
  }
}
```

자주 쓰이는 Hook 패턴:

| 이벤트 | 용도 |
| :--- | :--- |
| `PostToolUse` (파일 수정 후) | 자동 린트, 포맷팅 실행 |
| `Stop` (응답 완료 후) | 자동 테스트 실행 |
| `PreToolUse` (실행 전) | 위험한 명령어 차단 |

### 팁 4: 서브에이전트로 컨텍스트를 격리하라

큰 작업을 하나의 세션에서 처리하면 컨텍스트가 빠르게 오염됩니다. 서브에이전트는 독립적인 컨텍스트 윈도우에서 서브태스크를 처리하고 결과만 돌려줍니다.

```bash
# 메인 세션에서 탐색을 위임
> 서브에이전트를 사용해서 이 코드베이스에서 인증 관련 로직이 어디에 있는지 찾아줘. 
  메인 컨텍스트는 건드리지 말고.
```

`.claude/agents/` 폴더에 정의하면 더 구체적으로 제어할 수 있습니다:

```yaml
# .claude/agents/code-reviewer.yml
name: code-reviewer
description: 코드 품질과 보안을 검토합니다. 코드 변경 후 자동으로 호출하세요.
tools: Read, Glob, Grep
model: claude-sonnet-4-6
---
당신은 시니어 코드 리뷰어입니다. 
코드 품질, 보안, 성능에 집중해서 구체적이고 실행 가능한 피드백을 제공하세요.
```

### 팁 5: .claudeignore로 불필요한 파일을 제외하라

Claude가 파일을 읽을 때마다 토큰이 소비됩니다. 빌드 결과물, 의존성, 생성된 파일은 제외합니다.

``` gitignore
# .claudeignore
node_modules/
.venv/
dist/
build/
.next/
__pycache__/
*.lock
*.min.js
coverage/
.git/
```

`.gitignore`와 비슷한 역할이지만 Claude Code의 파일 접근을 제어합니다.

### 팁 6: /btw로 컨텍스트를 오염시키지 않고 질문하라

작업 중에 빠른 확인이 필요할 때 일반 프롬프트를 쓰면 대화 이력에 남아 컨텍스트를 차지합니다. `/btw`는 답변이 대화 이력에 추가되지 않습니다.

```bash
# 리팩토링 진행 중에 빠른 확인이 필요할 때
> /btw 지금 건드리는 함수가 몇 번 파일에 있어?

# Claude가 답변을 오버레이로 보여주고
# 대화 이력에는 추가되지 않음 → 컨텍스트 절약
```

### 팁 7: 글로벌 CLAUDE.md와 프로젝트 CLAUDE.md를 분리하라

모든 프로젝트에 공통으로 적용할 규칙과, 특정 프로젝트에만 적용할 규칙을 분리합니다.

```markdown
# ~/.claude/CLAUDE.md (전역 — 모든 프로젝트)
## 공통 원칙
- 되돌릴 수 없는 행동 전 반드시 확인
- 환경변수를 코드에 하드코딩하지 않기
- 테스트 없이 production 코드 수정하지 않기

## 내 스타일
- 주석은 "왜"를 설명하고 "무엇"은 코드로
- 함수는 한 가지 일만
```

```markdown
# 프로젝트 루트/CLAUDE.md (프로젝트별)
## 이 프로젝트 특화 규칙
- 빌드: `make build`
- 테스트: `pytest tests/ -v --cov=src`
- 이 프로젝트는 Python 3.11+ 사용
- 도메인 레이어에 외부 프레임워크 import 금지
```

---

## 5. 실제 프로젝트에서 쓰는 CLAUDE.md 예시

이 블로그의 scaffold-agents 프로젝트에서 사용하는 CLAUDE.md입니다. AI 에이전트 개발 프로젝트에 맞게 설계했습니다.

```markdown
# AGENTS.md / CLAUDE.md

## 역할 원칙
- 할당된 단일 역할만 수행 (SRP)
- 역할 범위를 벗어나는 요청은 적절한 에이전트에게 위임

## 아키텍처 규칙
- domain/ 폴더에는 프레임워크 import 금지 (import anthropic, import langgraph 없음)
- 외부 서비스 호출은 반드시 tools/ 레이어를 통해서만
- 환경변수를 코드에 하드코딩 금지

## 도구 설계 원칙
- 도구 하나 = 기능 하나 (원자성)
- 모든 도구 파라미터는 Pydantic 스키마로 정의
- 에러 메시지는 Claude가 읽고 교정할 수 있도록 구체적으로

## 행동 분류
- GREEN (자동 실행): 읽기, 조회, 분석
- YELLOW (감사 로그 필수): 파일 생성, 데이터 수정
- RED (Human 승인 필수): 삭제, 이메일 발송, 결제

## 절대 금지
- rm -rf, DROP TABLE, DELETE FROM (WHERE 없이) 사용 금지
- 승인 없이 외부로 데이터 전송 금지

## 빌드 및 테스트
- 테스트: pytest tests/unit/ -v
- 유닛 테스트는 API 키 없이 실행 가능해야 함
- 새 기능 추가 시 해당 레이어 테스트 필수
```

---

## 정리

Claude Code 효율의 차이는 프롬프트 실력이 아니라 **컨텍스트 설계**에 있습니다.

| 설정 | 역할 | 권장 크기 |
| :--- | :--- | :--- |
| **CLAUDE.md** | 세션마다 자동 주입되는 프로젝트 규칙 | 60~300줄 |
| **Karpathy 4원칙** | Claude의 반복적인 실수 패턴 방지 | `/plugin install`로 한 번에 |
| **Hooks** | 권고가 아닌 강제 실행 | 린트, 테스트 자동화 |
| **/clear** | 컨텍스트 오염 방지 | 작업 단위마다 |
| **.claudeignore** | 불필요한 파일 제외 | node_modules 등 |
| **서브에이전트** | 탐색·검증 작업을 별도 컨텍스트로 | 복잡한 태스크에 |

처음에는 CLAUDE.md 하나만 제대로 만드는 것부터 시작하면 충분합니다. 세션을 열 때마다 반복하던 설명이 사라지는 것만으로도 Claude Code 경험이 완전히 달라집니다.

> {: .prompt-info }
> **참고 자료**
> - Claude Code 공식 Best Practices: [https://code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)
> - Karpathy CLAUDE.md: [https://github.com/forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills)
> - HumanLayer — Writing a good CLAUDE.md: [https://www.humanlayer.dev/blog/writing-a-good-claude-md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
> - DataCamp — Claude Code Best Practices: [https://www.datacamp.com/tutorial/claude-code-best-practices](https://www.datacamp.com/tutorial/claude-code-best-practices)
