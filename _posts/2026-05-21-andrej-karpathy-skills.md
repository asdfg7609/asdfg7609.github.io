---
layout: post
title: "Karpathy가 말한 AI 코딩의 4가지 병폐"
date: 2026-05-21 21:00:00 +0900
categories:
  - AI
  - ClaudeCode
tags: [Claude, ClaudeCode, CLAUDE.md, KarpathySkills, AgentCoding, Anthropic, LLM]
---

> {: .prompt-tip }
> **요약:** Andrej Karpathy의 2026년 1월 X 포스트에서 촉발된 `forrestchang/andrej-karpathy-skills` 레포는 70줄짜리 `CLAUDE.md` 파일 하나로 LLM 코딩 에이전트의 고질적인 4가지 병폐를 교정합니다. 출시 3개월 만에 누적 스타 220,000개를 기록하며 GitHub 역사상 가장 빠르게 성장한 레포 중 하나가 됐습니다.

---

이 블로그에서 지난 두 편에 걸쳐 `CLAUDE.md`와 Claude Skills를 살펴봤습니다. `CLAUDE.md`는 Claude Code에게 프로젝트의 맥락을 전달하고, Skills는 도메인별 절차를 동적으로 탑재합니다. 두 도구 모두 "Claude에게 무엇을 알려줄 것인가"에 대한 답이었습니다.

이번 글은 한 발 더 거슬러 올라갑니다. **"왜 이 도구들이 필요한가"**, 즉 AI 코딩 에이전트가 근본적으로 어떤 방식으로 실패하는지에 대한 질문입니다.

---

## 1. 개요

Karpathy는 OpenAI 공동 창업자이자 Tesla AI 총괄을 역임한 인물입니다. "vibe coding"이라는 용어를 2025년 초에 처음 만든 사람이기도 합니다. 그런 그가 2026년 1월 26일 자신의 코딩 워크플로에 대한 메모를 X에 올렸습니다.

> "지난 몇 주간 Claude Code로 꽤 많이 코딩한 메모들. 코딩 워크플로. 최근 LLM 코딩 능력의 향상으로, 많은 사람들처럼 나도 11월에 80% 수동+자동완성, 20% 에이전트 방식에서 80% 에이전트 코딩, 20% 편집+마무리로 빠르게 전환했다."

이어서 그는 이 변화의 규모를 이렇게 표현했습니다.

> "Easily the biggest change to my basic coding workflow in 2 decades of programming, and it happened over the course of a few weeks."
> (20년 프로그래밍 인생에서 기본 코딩 워크플로의 가장 큰 변화이며, 불과 몇 주 만에 일어났다.)

이 변화는 갑작스러웠습니다. 2025년 11월까지도 코딩 에이전트는 사실상 쓸 만하지 않았습니다. 그런데 12월에 무언가가 바뀌었고, Karpathy의 표현대로 "에이전트가 기본적으로 작동하기 시작했습니다." 에이전트는 이제 한 줄이 아니라 전체 태스크를 완수할 수 있게 됐습니다.

그러나 Karpathy는 그 기쁨을 담담하게 서술한 뒤, 곧 냉정한 관찰로 전환했습니다.

> "It feels like I'm cheating. Which is a very weird feeling to have."
> (치팅하는 것 같은 느낌이다. 정말 이상한 기분이다.)

그리고 이 글의 핵심이 되는 관찰이 이어졌습니다.

---

## 2. LLM 코딩 에이전트의 4가지 구조적 실패

Karpathy는 에이전트 코딩의 효용을 인정하면서도, 오늘날의 모델이 반복적으로 저지르는 실패 패턴 세 가지를 명확하게 짚었습니다.

### 실패 1: 조용한 가정 (Silent Assumptions)

> "The models make wrong assumptions on your behalf and just run along with them without checking. They don't manage their confusion, don't seek clarifications, don't surface inconsistencies, don't present tradeoffs, don't push back when they should."

모델은 요청이 모호할 때 멈추지 않습니다. 스스로 해석을 골라 진행합니다. "유저 데이터를 export해줘"라는 요청에 어떤 형식인지, 어떤 필드인지, 브라우저 다운로드인지 API 응답인지 묻지 않고, 자체적으로 결정하고 코드를 씁니다. 문제는 그 결정이 틀렸을 때 수백 줄의 코드를 다시 짜야 한다는 점입니다.

### 실패 2: 과잉 복잡화 (Over-engineering)

> "They really like to overcomplicate code and APIs, bloat abstractions, don't clean up dead code... implement a bloated construction over 1000 lines when 100 would do."

LLM은 '좋은 코드'의 패턴을 학습했습니다. 팩토리 패턴, 전략 패턴, 의존성 주입... 문제는 그것이 현재 요청에 필요한지와 무관하게 적용된다는 점입니다. 단순한 할인 계산 함수 하나를 요청했더니 미래의 확장성을 위한 Strategy 패턴 계층 구조가 돌아옵니다. 코드는 맞지만 100배 복잡합니다.

### 실패 3: 직교적 수정 (Orthogonal Edits)

> "They still sometimes change/remove comments and code they don't sufficiently understand as side effects, even if orthogonal to the task."

버그 하나를 고치러 들어갔다가, 관련 없는 파일의 따옴표 스타일을 바꾸거나 타입 힌트를 추가하거나 오래된 주석을 지우고 나옵니다. 각각은 사소해 보이지만, 코드 리뷰어 입장에서는 실제 변경과 부수 효과를 분리하는 작업이 추가됩니다. diff가 오염됩니다.

### 실패 4: 검증 기준의 부재 (No Success Criteria)

이 항목은 Karpathy가 직접 언급한 실패보다는, 그가 LLM의 강점으로 언급한 특성에서 역으로 도출됩니다.

> "LLMs are exceptionally good at looping until they meet specific goals... Don't tell it what to do, give it success criteria and watch it go."

에이전트는 명확한 목표가 주어지면 스스로 반복하며 달성할 수 있습니다. 그런데 대부분의 요청은 "이 버그 고쳐줘", "코드 개선해줘"처럼 검증 기준이 없습니다. 에이전트는 '개선된 것 같은' 상태에서 멈추고, 개발자는 그게 실제로 올바른지 전부 검증해야 합니다.

---

## 3. Forrest Chang의 응답: 70줄 파일

2026년 1월 26일 Karpathy의 포스트가 올라온 다음날, 개발자 Forrest Chang(Jiayuan Zhang)은 특이한 방식으로 이 문제에 응답했습니다.

Claude Code에게 Karpathy의 포스트를 읽히고, 에이전트 Skills 파일들을 생성하게 했습니다. 처음에는 800줄짜리 문서가 나왔습니다. 그 다음, Claude Code 자신이 생성한 Skills로 자기 자신을 리뷰하게 했습니다.

결과: 70줄.

Chang은 이렇게 말했습니다:

> "I let Claude Code turn @karpathy's post into agent skills. It first generated a bunch of skill files and around 800 lines of descriptions. Then I let it use these agent skills to review itself. Boom, it cut itself down to 70 lines of clean, solid instructions."

이 70줄이 `forrestchang/andrej-karpathy-skills`의 `CLAUDE.md`입니다. 4가지 병폐 각각에 대응하는 4원칙을 담고 있습니다.

---

## 4. 4원칙 상세 해부

### 원칙 1: Think Before Coding
**"Don't assume. Don't hide confusion. Surface tradeoffs."**

코드를 쓰기 전에 먼저 생각하라는 원칙입니다. 요청이 모호하면 가정을 명시적으로 나열하고, 여러 해석이 가능하면 조용히 하나를 선택하지 말고 선택지를 제시하고 확인을 받아야 합니다.

**실패 패턴 → 올바른 동작:**

```
# ❌ 조용한 가정
"유저 export 해줘"
→ (묻지 않고) JSON 형식으로 모든 필드를 파일로 저장하는 코드 작성

# ✅ Think Before Coding
"유저 export 해줘"
→ "구현 전에 확인이 필요합니다:
   1. 범위: 전체 유저인가요, 필터링된 서브셋인가요?
   2. 형식: 브라우저 다운로드 / 백그라운드 작업 / API 응답?
   3. 필드: 어떤 필드를 포함할까요? (민감한 정보 포함 여부)
   어떤 방식이 필요하신가요?"
```

### 원칙 2: Simplicity First
**"Minimum code that solves the problem. Nothing speculative."**

요청된 것만 구현합니다. 요청되지 않은 유연성, 확장성, 추상화는 추가하지 않습니다. "200줄을 썼는데 50줄로 쓸 수 있다면, 다시 쓰세요. 시니어 엔지니어가 이 코드를 보고 과도하게 복잡하다고 할 것 같으면, 단순화하세요."

**실패 패턴 → 올바른 동작:**

```python
# ❌ 과잉 복잡화: 단순 할인 계산에 Strategy 패턴 적용
class DiscountStrategy(ABC):
    @abstractmethod
    def apply(self, price): ...

class PercentageDiscount(DiscountStrategy):
    def __init__(self, pct): self.pct = pct
    def apply(self, price): return price * (1 - self.pct)

class DiscountCalculator:
    def __init__(self, strategy: DiscountStrategy): ...

# ✅ Simplicity First: 요청된 것만
def apply_discount(price: float, discount_pct: float) -> float:
    return price * (1 - discount_pct)
```

### 원칙 3: Surgical Changes
**"Touch only what you must. Clean up only your own mess."**

요청된 작업에 직접적으로 관련된 코드만 수정합니다. 관련 없는 파일의 포매팅, 타입 힌트, 주석은 건드리지 않습니다. 3개 파일 이상을 수정해야 할 것 같으면 먼저 범위를 확인합니다.

**실패 패턴 → 올바른 동작:**

```python
# 요청: "calculate_total의 버그 수정해줘"

# ❌ Orthogonal Edit: 버그 수정 + 부수 효과
- def calculate_total(items):  # old function
+ def calculate_total(items: list[dict]) -> float:  # 타입 힌트 추가 (요청 없음)
      total = 0
-     for item in items:
-         total += item['price']  # 따옴표 스타일 변경 (요청 없음)
+     for item in items:
+         total += item["price"]
+     return round(total, 2)  # 반올림 추가 (요청 없음)

# ✅ Surgical Change: 버그만 수정
  def calculate_total(items):
      total = 0
      for item in items:
-         total += item['price'] * item['qty']  # 버그: qty 누락
+         total += item['price'] * item.get('qty', 1)
      return total
```

### 원칙 4: Goal-Driven Execution
**"Define success criteria. Loop until verified."**

모호한 명령("버그 고쳐줘") 대신 검증 가능한 성공 기준("이 버그를 재현하는 테스트를 작성하고, 그 테스트가 통과할 때까지 수정하라")으로 변환합니다. LLM은 명확한 목표가 있을 때 스스로 반복하며 달성하는 데 탁월합니다.

| ❌ 모호한 명령 | ✅ 검증 가능한 목표 |
|:---|:---|
| "이 코드 개선해줘" | "함수 실행 시간을 현재의 절반 이하로 줄이고, 기존 테스트가 모두 통과해야 함" |
| "버그 고쳐줘" | "버그를 재현하는 테스트 케이스를 먼저 작성하고, 그 테스트가 통과하면 완료" |
| "리팩토링해줘" | "각 함수가 단일 책임을 가지도록 분리하되, 공개 API 시그니처는 변경 없음" |

---

## 5. 레포 구조와 설치 방법

`forrestchang/andrej-karpathy-skills`는 동일한 4원칙을 세 가지 형식으로 제공합니다.

```
andrej-karpathy-skills/
├── CLAUDE.md                          # Claude Code용 (핵심 파일)
├── EXAMPLES.md                        # 원칙별 Before/After 예시
├── CURSOR.md                          # Cursor IDE 설정 가이드
├── skills/
│   └── karpathy-guidelines/
│       └── SKILL.md                   # Claude Code Skills 플러그인용
└── .claude-plugin/
    ├── plugin.json                    # 플러그인 메타데이터
    └── marketplace.json               # 마켓플레이스 등록 정보
```

동일한 4원칙이 `CLAUDE.md`(프로젝트 직접 적용), `SKILL.md`(Claude Code Skills 플러그인), `.cursorrules`(Cursor) 세 가지 경로로 배포됩니다.

### 방법 A: Claude Code Plugin (전체 프로젝트에 글로벌 적용, 권장)

Claude Code 터미널 안에서 두 줄이면 됩니다.

```bash
/plugin marketplace add forrestchang/andrej-karpathy-skills
/plugin install andrej-karpathy-skills@karpathy-skills
```

한 번 설치하면 이후 모든 프로젝트에서 자동으로 적용됩니다.

### 방법 B: 프로젝트별 CLAUDE.md 적용

새 프로젝트에 처음 적용하는 경우:

```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md
```

기존 `CLAUDE.md`가 있는 프로젝트에 추가하는 경우:

```bash
echo "" >> CLAUDE.md
curl https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md >> CLAUDE.md
```

프로젝트 고유 규칙은 파일 끝에 추가합니다. 나중에 작성된 규칙이 앞의 규칙보다 우선 적용되므로, 프로젝트별 설정이 Karpathy 가이드라인을 덮어씁니다.

### 방법 C: Cursor 사용자

레포를 클론하면 `.cursor/rules/karpathy-guidelines.mdc`가 이미 포함되어 있습니다. 프로젝트를 Cursor로 열기만 하면 자동으로 적용됩니다.

---

## 6. 실제로 동작하는지 확인하는 법

원칙이 적용되고 있다면 다음과 같은 변화가 관찰됩니다.

- **diff가 깨끗해집니다** — 요청한 것만 변경되고, 관련 없는 포매팅 변경이 사라집니다.
- **구현 전에 질문이 옵니다** — 모호한 요청에 대해 먼저 명시적 확인이 들어옵니다.
- **코드가 처음부터 단순합니다** — 과잉 추상화 없이, 요청에 딱 맞는 코드가 나옵니다.
- **PR에 drive-by 리팩토링이 없습니다** — 코드 리뷰 시 관련 없는 변경을 걸러낼 필요가 없습니다.

파일 자체도 이 기준을 제시합니다:

> "These guidelines are working if: fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes."

---

## 7. 주의할 점: 트레이드오프를 솔직하게

이 레포가 주목받는 이유 중 하나는 자신의 한계를 명시적으로 인정한다는 점입니다. `CLAUDE.md` 파일 상단에는 이런 문구가 있습니다.

> "These guidelines bias toward caution over speed. For trivial tasks (simple typo fixes, obvious one-liners), use judgment — not every change needs the full rigor."

원칙들은 의도적으로 보수적입니다. 단순한 오타 수정이나 명확한 한 줄짜리 변경에도 가정을 나열하고 성공 기준을 정의하라고 하면, 오히려 속도가 느려집니다. 복잡한 멀티파일 변경, 아키텍처 결정, 잘못된 가정이 대규모 재작업으로 이어질 수 있는 작업에서 진가를 발휘합니다.

또한 하나 더 짚고 넘어갈 사실이 있습니다. **Karpathy는 이 레포를 공식 승인하지 않았습니다.** 레포는 Chang이 Karpathy의 관찰을 바탕으로 만든 것이며, Karpathy의 이름을 달고 있는 이유는 그의 아이디어에서 직접 파생됐기 때문입니다.

---

## 마치며: 시리즈를 통해 본 에이전트 코딩의 설계 원리

이 블로그의 세 편을 돌아보면 하나의 층위 구조가 완성됩니다.

- **`CLAUDE.md`**: Claude Code에게 프로젝트의 맥락을 줍니다. "이 프로젝트는 무엇인가."
- **Claude Skills**: Claude Code에게 도메인 절차를 줍니다. "이 작업은 어떻게 해야 하는가."
- **`andrej-karpathy-skills`**: Claude Code의 행동 방식을 교정합니다. "AI 에이전트는 어떻게 실패하며, 어떻게 그 실패를 방지하는가."

세 도구는 서로를 대체하지 않습니다. 조합해서 사용할 때 에이전트 코딩의 효율이 극대화됩니다. `CLAUDE.md`에는 프로젝트 규칙과 Karpathy 가이드라인을 함께 넣고, Skills에는 도메인별 SOP를 담고, 에이전트에게는 검증 가능한 목표를 줍니다.

Karpathy가 "20년 만의 가장 큰 변화"라고 부른 전환점 위에서, 우리는 어떻게 에이전트에게 더 잘 지시할 것인가를 배워가고 있습니다. 그 학습의 산물 중 하나가 이 70줄짜리 파일입니다.

---

**참고 자료**
- [forrestchang/andrej-karpathy-skills — GitHub](https://github.com/forrestchang/andrej-karpathy-skills)
- [Andrej Karpathy 원본 X 포스트 (2026.01.26)](https://x.com/karpathy/status/2015883857489522876)
- [DeepWiki: forrestchang/andrej-karpathy-skills](https://deepwiki.com/forrestchang/andrej-karpathy-skills)
- [Karpathy CLAUDE.md: The #1 GitHub Trending File — pasqualepillitteri.it](https://pasqualepillitteri.it/en/news/1872/karpathy-claude-md-trending-github-llm-coding)
- [Karpathy-Inspired CLAUDE.md Passes 220,000 Combined GitHub Stars — TechTimes](http://www.techtimes.com/articles/316798/20260518/karpathy-inspired-claudemd-passes-220000-combined-github-stars-four-rules-that-stop-ai-breaking.htm)
