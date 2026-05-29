---
layout: post
title: "CLAUDE.md의 진화: Claude Skills로 AI에게 '업무 매뉴얼'을 쥐어주다"
date: 2026-05-20 12:00:00 +0900
categories:
  - AI
  - ClaudeCode
tags: [Claude, SKILL.md, CLAUDE.md, Anthropic, ClaudeCode]
---

> {: .prompt-tip }
> **요약:** Claude Skills는 CLAUDE.md의 정적인 한계를 극복한 **동적 절차서**입니다. SKILL.md 파일 하나로 Claude에게 도메인 전문 지식과 반복 워크플로를 탑재할 수 있으며, 필요한 순간에만 로드되어 컨텍스트 창을 효율적으로 사용합니다.

---

`CLAUDE.md`는 "이 프로젝트는 Python 3.11을 쓴다", "커밋 메시지는 한국어로 작성한다"처럼, 항상 적용되어야 하는 **배경 지식**을 전달하는 데 탁월합니다.

그런데 실무를 하다 보면 이런 상황이 생깁니다.

> "PR 리뷰할 때만 보안 취약점 체크리스트를 따르게 하고 싶다."
> "Excel 파일 가공할 때만 사내 데이터 처리 SOP를 적용하고 싶다."
> "보고서 쓸 때만 회사 문체 가이드를 참고하게 하고 싶다."

이 모든 것을 `CLAUDE.md` 하나에 쑤셔 넣으면 어떻게 될까요? 파일은 비대해지고, 매 대화마다 불필요한 내용까지 컨텍스트 창을 잠식합니다. Claude에게 코드 한 줄 수정을 부탁했을 뿐인데, 보고서 문체 가이드까지 읽어야 한다면 비효율적입니다.

Anthropic이 2025년 10월 이 문제의 해법으로 내놓은 것이 바로 **Claude Skills**입니다.

---

## 1. CLAUDE.md의 한계: '항상 켜진 라디오' 문제

`CLAUDE.md`는 **정적(static)** 이고 **전역(global)** 입니다. Claude Code가 실행되는 순간부터 항상 컨텍스트에 로드됩니다. 프로젝트 전반에 적용되는 규칙—코드 스타일, 폴더 구조, 금지 명령어 목록—을 정의하는 데는 이 특성이 강점입니다.

하지만 바로 그 특성이 약점이기도 합니다.

- **컨텍스트 낭비:** 지금 하는 작업과 무관한 절차도 항상 로드됩니다.
- **유지보수 부담:** 도메인별 지식이 하나의 파일에 뒤섞이면 관리가 어렵습니다.
- **재사용 불가:** 다른 프로젝트에 같은 SOP를 적용하려면 파일을 복사해야 합니다.

`CLAUDE.md`가 "사원증"이라면—출근하는 순간부터 퇴근할 때까지 항상 목에 걸려 있는—, Skills는 "업무 매뉴얼 바인더"에 가깝습니다. 코드 리뷰를 할 때는 코드 리뷰 매뉴얼을, 보고서를 쓸 때는 문서 작성 매뉴얼을 꺼내 드는 방식입니다.

---

## 2. Claude Skills란 무엇인가

**Claude Skills**는 지침(instructions), 스크립트(scripts), 리소스(resources)를 하나의 폴더로 패키징하여, Claude가 관련 작업을 수행할 때 자동으로 로드하는 **모듈형 역량 패키지**입니다.

핵심 작동 원리는 **Progressive Disclosure(점진적 공개)** 입니다. Claude는 등록된 모든 Skill의 메타데이터(이름과 설명, 수십 토큰)를 항상 인지하고 있다가, 현재 요청이 특정 Skill과 관련이 있다고 판단될 때 비로소 해당 Skill의 전체 내용을 로드합니다. 불필요한 내용이 컨텍스트 창을 점유하지 않습니다.

### CLAUDE.md vs Skills: 무엇이 다른가

| 구분 | CLAUDE.md | Claude Skills |
| :--- | :--- | :--- |
| **적용 방식** | 항상 로드 (정적) | 필요시 로드 (동적) |
| **적용 범위** | 프로젝트 전체 | 특정 작업·도메인 |
| **재사용성** | 프로젝트 단위 | 프로젝트·계정 간 공유 가능 |
| **용도** | 배경 지식, 환경 설정 | 절차·워크플로, 도메인 SOP |
| **토큰 비용** | 항상 소비 | 관련 작업 시에만 소비 |

정리하면 **"무엇을 알아야 하는가"는 CLAUDE.md에, "어떻게 해야 하는가"는 Skills에** 담는 것이 올바른 설계입니다.

---

## 3. SKILL.md 파일 해부

모든 Skill의 핵심은 `SKILL.md` 파일입니다. 구조는 간단합니다: **YAML 프론트매터(Frontmatter)** + **Markdown 본문**.

### 기본 구조

```
my-skill/
├── SKILL.md          # 필수: Skill의 핵심 파일
├── guide.md          # 선택: 추가 참고 문서
├── scripts/          # 선택: 실행 가능한 코드
│   └── process.py
└── resources/        # 선택: 템플릿, 데이터 파일
    └── template.xlsx
```

`SKILL.md`는 다음 두 레이어로 구성됩니다.

**레이어 1 — YAML 프론트매터 (메타데이터)**

Claude가 항상 읽는 부분입니다. Skill을 언제 사용해야 하는지 판단하는 기준이 됩니다. `name`과 `description`은 필수입니다.

```yaml
---
name: "PR Code Review"
description: "Pull Request의 보안 취약점, 성능 이슈, 코딩 컨벤션을 검토할 때 사용합니다."
version: "1.0"
author: "Engineering Team"
---
```

**레이어 2 — Markdown 본문 (절차와 지식)**

Claude가 Skill을 사용하기로 결정했을 때 로드하는 부분입니다. 단계별 절차, 체크리스트, 예시, 금지 사항 등을 자유롭게 담을 수 있습니다.

```markdown
## PR 리뷰 프로토콜

사용자가 PR 리뷰를 요청하면 다음 순서로 수행합니다.

### 1단계: 보안 검토 (High Priority)
- SQL Injection, XSS, CSRF 취약점 여부를 먼저 확인합니다.
- `src/auth/` 경로의 변경사항은 반드시 별도 코멘트를 남깁니다.
- 하드코딩된 API 키나 시크릿이 없는지 확인합니다.

### 2단계: 성능 검토
- N+1 쿼리 패턴이 발생하는지 확인합니다.
- 루프 내 불필요한 I/O 작업이 없는지 검토합니다.

### 3단계: 컨벤션 검토
- Python: `snake_case`, TypeScript: `camelCase` 준수 여부를 확인합니다.
- 함수 길이가 50줄을 초과하면 리팩토링을 제안합니다.

### 리뷰 결과 형식
리뷰는 다음 형식으로 출력합니다.
- 🔴 **Critical**: 즉시 수정 필요
- 🟡 **Warning**: 수정 권장
- 🟢 **Suggestion**: 개선 아이디어
```

`description`을 정밀하게 작성할수록 Claude가 Skill을 정확한 타이밍에 활성화합니다. "코드 관련 작업"처럼 넓게 쓰는 것보다 "Pull Request의 보안·성능·컨벤션을 검토할 때"처럼 구체적으로 기술하는 것이 좋습니다.

---

## 4. 직접 만들어보기: 기술 블로그 작성 Skill

가장 빠르게 Skill의 가치를 체감하는 방법은 직접 만드는 것입니다. 기술 블로그를 일관된 품질로 작성하는 Skill을 예시로 들겠습니다.

### 폴더 구조 생성

```bash
mkdir -p my-skills/tech-blog-writer
```

### SKILL.md 작성

```markdown
---
name: "Tech Blog Writer"
description: "기술 블로그 포스트를 작성하거나 초안을 검토할 때 사용합니다. Jekyll, GitHub Pages 기반 블로그에 최적화되어 있습니다."
version: "1.0"
---

## 블로그 포스트 작성 원칙

### 구조
모든 포스트는 다음 구조를 따릅니다.
1. **도입부**: 독자가 겪는 문제 상황으로 시작합니다. 기술 개요로 시작하지 않습니다.
2. **본론**: 문제 → 메커니즘 설명 → 실습 코드 → 결과 순서를 지킵니다.
3. **결론**: 기술의 실용적 의미와 한계를 언급합니다.

### 코드 예시 규칙
- 모든 코드 블록에는 언어를 명시합니다 (```python, ```bash 등).
- 코드에 한국어 주석을 포함합니다.
- 실제로 실행 가능한 예시를 우선합니다.

### 문체
- 독자에게 직접 말하는 2인칭("여러분")을 쓰지 않습니다.
- 비유와 구체적인 예시를 통해 추상적인 개념을 설명합니다.
- 소제목은 독자의 질문 형식("왜 X가 문제일까?")을 활용합니다.

### Jekyll 프론트매터 형식
반드시 다음 형식을 사용합니다.
\`\`\`yaml
---
layout: post
title: "제목"
date: YYYY-MM-DD HH:MM:SS +0900
categories:
  - 카테고리1
  - 카테고리2
tags: [태그1, 태그2]
---
\`\`\`
```

### Claude Code에서 사용하기

Claude Code를 사용한다면 프로젝트 루트에 `skills/` 디렉터리를 만들고 Skill 폴더를 넣으면 됩니다. Claude가 시작 시 자동으로 감지합니다.

```bash
my-project/
├── CLAUDE.md
├── skills/
│   ├── tech-blog-writer/
│   │   └── SKILL.md
│   └── pr-code-review/
│       └── SKILL.md
└── _posts/
```

이제 "이 주제로 블로그 포스트 초안 써줘"라고 요청하면, Claude는 `Tech Blog Writer` Skill을 자동으로 활성화하여 정해진 구조와 문체 가이드를 따릅니다. 매번 같은 지침을 프롬프트에 반복해서 쓸 필요가 없습니다.

---

## 5. Skills vs MCP vs 프롬프트: 언제 무엇을 쓸까

Skills가 등장한 이후, "MCP랑 뭐가 다르냐"는 질문이 많습니다. 세 가지 도구는 역할이 명확히 구분됩니다.

**MCP(Model Context Protocol)** 는 **인프라 계층**입니다. Claude가 외부 시스템—GitHub, Jira, Slack, 데이터베이스—에 연결하여 실시간 데이터를 읽고 쓸 수 있도록 API 브릿지를 제공합니다. "무엇에 접근할 수 있는가"를 정의합니다.

**Skills** 는 **절차 계층**입니다. 특정 작업을 어떻게 수행해야 하는지, 어떤 기준으로 판단해야 하는지를 담은 SOP입니다. "어떻게 행동해야 하는가"를 정의합니다.

**프롬프트** 는 **실행 계층**입니다. 지금 이 순간 특정 요청에만 적용되는 일회성 지시입니다. 매 대화가 끝나면 사라집니다.

| 도구 | 계층 | 수명 | 주요 용도 |
| :--- | :--- | :--- | :--- |
| **프롬프트** | 실행 | 대화 단위 | 일회성 지시, 즉각적인 문맥 전달 |
| **CLAUDE.md** | 설정 | 프로젝트 내 영구 | 환경 설정, 배경 지식 |
| **Skills** | 절차 | 계정·프로젝트 간 영구 | 도메인 SOP, 반복 워크플로 |
| **MCP** | 인프라 | 시스템 수준 | 외부 서비스 연결, 실시간 데이터 |

실무에서 이 네 가지는 조합하여 사용합니다. 예를 들어, GitHub MCP로 PR diff를 가져오고(MCP), PR 리뷰 Skill의 보안 체크리스트를 적용하고(Skills), 현재 PR의 특수한 맥락을 프롬프트로 보완하는(프롬프트) 방식입니다.

---

## 6. Claude.ai와 API에서 Skills 사용하기

Skills는 Claude Code뿐 아니라 claude.ai 웹 인터페이스와 API 모두에서 사용할 수 있습니다.

### claude.ai에서 사용하기

`설정 > 기능 > Skills`에서 zip 파일로 압축한 Skill 폴더를 업로드합니다. Pro, Max, Team, Enterprise 플랜에서 Code Execution이 활성화되어 있어야 합니다. 업로드된 Skill은 해당 계정의 모든 대화에서 자동으로 사용 가능합니다.

### API에서 사용하기

API를 통해 Skill을 업로드하고 관리할 수 있습니다. `skills-2025-10-02` 베타 헤더가 필요합니다.

```python
import anthropic
from pathlib import Path

# Skills 베타 헤더로 클라이언트 초기화
client = anthropic.Anthropic(
    api_key="YOUR_API_KEY",
    default_headers={"anthropic-beta": "skills-2025-10-02"}
)

# Skill 업로드
with open("my-skills/tech-blog-writer/SKILL.md", "rb") as f:
    response = client.beta.skills.create(
        display_title="Tech Blog Writer",
        files=[("files[]", ("SKILL.md", f, "text/markdown"))]
    )

skill_id = response.id
print(f"Skill 업로드 완료: {skill_id}")
```

업로드된 Skill은 메시지 요청 시 `container` 파라미터로 지정합니다.

```python
# Skill을 포함한 메시지 전송
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2000,
    tools=[{"type": "code_execution_20250522", "name": "code_execution"}],  # Code Execution 필수
    container={
        "type": "persistent",
        "skills": [
            {
                "type": "custom",
                "skill_id": skill_id
            }
        ]
    },
    messages=[
        {
            "role": "user",
            "content": "KV 캐시 양자화 기술에 대한 블로그 포스트 초안을 작성해줘."
        }
    ]
)

print(response.content[0].text)
```

Anthropic에서 제공하는 사전 빌드 Skill(`pptx`, `xlsx`, `docx`, `pdf`)도 같은 방식으로 `skill_id`를 지정해 사용합니다. 별도 업로드 없이 즉시 활용 가능합니다.

---

## 7. 마치며: "조직의 지식"을 코드화하는 시대

Claude Skills가 열어가는 변화의 본질은 기술적 효율성보다 더 깊은 곳에 있습니다.

지금까지 조직의 노하우는 노련한 팀원의 머릿속에, 혹은 아무도 읽지 않는 Confluence 페이지에 갇혀 있었습니다. Skills는 그 지식을 **실행 가능한 형태**로 캡처합니다. `SKILL.md` 파일에 담긴 PR 리뷰 프로토콜은 팀원의 숙련도에 관계없이, 어떤 프로젝트에서든, 어떤 Claude 인스턴스에서든 일관되게 동작합니다.

더 주목할 점은 **오픈 표준화**의 방향입니다. Anthropic은 2025년 12월 Agent Skills를 `agentskills.io`를 통해 오픈 표준으로 공개했습니다. 원칙적으로 Claude에서 만든 Skill이 다른 AI 플랫폼에서도 동작할 수 있는 기반이 마련된 것입니다. 특정 AI 벤더에 묶이지 않고 조직의 절차 지식을 이식할 수 있게 되는 미래가 가까워지고 있습니다.

`CLAUDE.md`가 Claude에게 프로젝트의 **맥락**을 알려준다면, Skills는 **능력**을 부여합니다. 두 가지를 함께 활용하면, 단순히 지시를 따르는 AI가 아니라 팀의 맥락을 이해하고 조직의 SOP를 자율적으로 적용하는 진정한 AI 동료에 한 발 더 가까워집니다.

---

**참고 자료**
- [Agent Skills Overview — Anthropic Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [How to create custom Skills — Claude Help Center](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)
- [Skills explained — claude.com Blog](https://claude.com/blog/skills-explained)
- [Building custom Skills — Claude Cookbook](https://platform.claude.com/cookbook/skills-notebooks-03-skills-custom-development)
