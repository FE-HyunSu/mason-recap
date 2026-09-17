---
description: 가장 최근에 완료된 사용자 턴(들)을 mason-recap가 수집한 관찰 증거(observed)만으로 재구성하여 Mason Recap Report를 생성합니다. 숫자 인자로 몇 턴을 볼지 지정할 수 있습니다(기본값 1).
argument-hint: "[n]"
allowed-tools: Bash, Read
---

# 목표

가장 최근에 완료된 사용자 턴(들)에 대해, `.mason-recap/events/`에 기록된 로그만을 근거로
실행 과정을 재구성한다. 인자를 주지 않으면 가장 최근 턴 1개, 숫자를 주면(`/mason-recap:latest 3`
처럼) 그 개수만큼의 최근 턴을 시간순으로 보여준다. 최근 턴이 아니라 과거의 특정 턴을
직접 골라 분석하고 싶으면 `/mason-recap:select`를 대신 사용한다.

**이 명령은 Claude의 비공개 chain-of-thought를 조회하거나 요구하지 않는다.** 오직 Hook과
transcript에서 관찰 가능한 사실(호출된 Tool, 읽거나 수정한 파일, 실행한 명령, Subagent 활동,
최종 답변)만을 사용한다.

# 절차

1. 아래 명령으로 최근 턴(들)의 원본 이벤트를 가져온다. `$ARGUMENTS`가 비어 있으면 1로
   취급된다(스크립트가 알아서 기본값 처리).

   ```bash
   node "${CLAUDE_PLUGIN_ROOT}/scripts/read-events.js" last-turns "$ARGUMENTS"
   ```

   반환된 JSON은 `requestedCount`(요청한 개수), `returnedCount`(실제로 로그에 있던 턴
   개수 — 요청보다 적을 수 있음), `turns`(시간순, 오래된 것부터 최신 순으로 정렬된 배열)로
   구성된다. 각 턴 항목은 `prompt`(해당 턴의 `UserPromptSubmit` 이벤트),
   `promptIdCorrelated`(같은 `promptId`로 명시적으로 연결된 이벤트 — observed 근거로
   취급 가능), `timeWindowCorrelated`(`promptId`가 없어 시간 구간으로만 연결된 이벤트 —
   반드시 inferred/약한 근거로 취급)를 담는다. `last-turns`는 프롬프트 텍스트가
   `/mason-recap:`로 시작하는 턴(이 플러그인 자신의 커맨드를 호출한 턴, 예: 지금 이
   커맨드를 실행시킨 `/mason-recap:latest 2` 그 자체)을 이미 제외하고 반환한다 — 리포트
   생성 요청 자체는 분석 대상 턴이 아니기 때문이다.

   `turns`가 빈 배열이면, 아직 수집된 로그가 없다는 사실을 그대로 보고하고 중단한다.
   `returnedCount`가 `requestedCount`보다 작으면, 요청한 개수만큼의 턴이 아직 기록되어
   있지 않다는 점을 리포트에 명시한다(추측으로 채우지 않는다).

2. `plugins/mason-recap/skills/decision-analysis/SKILL.md`에 정의된 분석 절차, Skill 활성화
   증거 등급(confirmed / strongly-inferred / weakly-inferred / not-observed), 등급→근사
   확신도(%) 변환, 프롬프트 문구→트리거 매핑 규칙을 각 턴에 그대로 적용한다. 이 Skill의
   절차를 skip하지 말고 각 단계를 실제로 수행한다.

3. 아래 출력 형식을 그대로 사용하여 보고서를 작성한다. `returnedCount`가 1이면 "단일 턴
   형식"을, 2 이상이면 "다중 턴 형식"을 사용한다. 가독성을 위해 실행 흐름은 긴 문단이 아니라
   **짧은 시간순 불릿 목록**으로 쓰고, 트리거 판정은 **표(table)**로 분리한다 — 문단 하나에
   근거를 전부 욱여넣지 않는다. 각 불릿에는 반드시 `(observed)` / `(inferred)` / `(unknown)`
   태그를 붙인다. "Claude가 이렇게 생각했다"처럼 단정하지 말고, "관찰된 행동을 보면 이렇게
   판단한 것으로 추정된다"는 식으로만 서술한다. 트리거 매핑 표에 확신도 %를 적을 때는 항상
   등급 이름과 함께 적고(예: "strongly-inferred (~70%, 근사)"), % 단독으로 쓰지 않는다.

# 출력 형식

### 단일 턴 형식 (`returnedCount` == 1)

```markdown
# Mason Recap Report

> "<사용자 프롬프트 원문 또는 핵심 요약>"

**실행 흐름** (시간순)
- <행동 1> (observed/inferred/unknown)
- <행동 2> (observed/inferred/unknown)
- <최종 답변이 위 행동과 일치하는지> (inferred/unknown)

**프롬프트 문구 → 트리거 매핑**

| 근거 문구 | 트리거 | 종류 | 등급 (근사 %) |
|---|---|---|---|
| "<프롬프트 중 해당 부분>" 또는 "특정 문구 없음" | <Skill/지침/Tool 이름> | Skill / 지침(Rule) / Tool | <confirmed/strongly-inferred/weakly-inferred/not-observed> (~<%>, 근사) |

<위 표에 후보가 전혀 없으면 표 대신 "이 턴에서 트리거로 판정할 후보 자체가 관찰되지
않음(not-observed)"이라고 적는다. 표의 % 값 위/아래에 다음 한 줄을 항상 덧붙인다:
"※ %는 실측 확률이 아니라 등급을 근사 시각화한 값 — Claude의 내부 판단 확률에는 접근할
수 없음.">

## 한계

이 리포트는 실행 증거를 기반으로 재구성한 분석이며
Claude의 비공개 내부 사고과정이 아니다. 표의 %는 등급의 근사 표현이며 실측값이 아니다.
```

### 다중 턴 형식 (`returnedCount` >= 2)

```markdown
# Mason Recap Report (최근 <returnedCount>턴)

<requestedCount > returnedCount인 경우: "요청한 <requestedCount>턴 중 <returnedCount>턴만
로그에 존재함"을 여기에 명시>

## 턴 1
> "<프롬프트 1 원문 또는 핵심 요약>"

**실행 흐름**
- <행동> (observed/inferred/unknown)

**프롬프트 문구 → 트리거 매핑**

| 근거 문구 | 트리거 | 종류 | 등급 (근사 %) |
|---|---|---|---|
| ... | ... | ... | ... |

## 턴 2
> "<프롬프트 2>"

**실행 흐름**
- ...

**프롬프트 문구 → 트리거 매핑**

| 근거 문구 | 트리거 | 종류 | 등급 (근사 %) |
|---|---|---|---|
| ... | ... | ... | ... |

<!-- returnedCount 만큼 "## 턴 N" 블록을 반복한다 -->

## 한계

이 리포트는 실행 증거를 기반으로 재구성한 분석이며
Claude의 비공개 내부 사고과정이 아니다. 표의 %는 등급의 근사 표현이며 실측값이 아니다.
```
