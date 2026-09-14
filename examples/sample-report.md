# Sample Mason Recap Report (worked example)

이 문서는 실제 사용자 세션이 아니라, `plugins/mason-recap/scripts/capture-event.js`와
`read-events.js`에 실제 샘플 Hook 이벤트를 흘려보내 얻은 결과를 근거로 손으로 작성한
예시다. `/mason-recap:chat`가 실행되면 Claude가 이와 유사한 형태의 리포트를
생성한다.

입력으로 사용된 이벤트 시퀀스(요약): `SessionStart` → `UserPromptSubmit`
("utils.js에 있는 off-by-one 버그를 고쳐줘") → `InstructionsLoaded` (`CLAUDE.md`) →
`PreToolUse`/`PostToolUse` (`Read src/utils.js`) → `PreToolUse`/`PostToolUse`
(`Edit src/utils.js`) → `PreToolUse`/`PostToolUse` (`Bash: npm test`) → `Stop`.

---

# Mason Recap Report

> "utils.js에 있는 off-by-one 버그를 고쳐줘"

**실행 흐름** (시간순)
- `CLAUDE.md`가 이 턴 시작 시점에 로드됨 (observed, `InstructionsLoaded`)
- `Read`로 `src/utils.js`를 조회함 (observed)
- `Edit`로 같은 파일을 수정함 (observed) — 다만 정확히 어떤 부분을 어떻게 바꿨는지는
  diff 원문을 저장하지 않는 정책상 확인 불가 (unknown)
- `Bash`로 `npm test`를 실행해 "5 passing" 결과를 얻음 (observed)
- 최종 답변("off-by-one 버그를 수정했고 테스트 5개가 모두 통과합니다")은 관찰된 행동
  (Edit 발생 + 테스트 통과)과 대체로 일치함 (inferred) — 다만 수정한 내용이 실제로 이
  5개 테스트가 검증하는 대상인지는 로그만으로 확인할 수 없음 (unknown)

**프롬프트 문구 → 트리거 매핑**

| 근거 문구 | 트리거 | 종류 | 등급 (근사 %) |
|---|---|---|---|
| 특정 문구 없음 | (해당 없음) | Skill | not-observed (~0%, 근사) — 이 턴에서 Skill 파일 접근이나 명시적 `/plugin:skill` 호출 문자열이 전혀 관찰되지 않음 |
| "utils.js에 있는 off-by-one 버그를 고쳐줘" | `CLAUDE.md` | 지침(Rule) | weakly-inferred (~40%, 근사) — 로드된 사실은 observed이나, 지침 내용을 저장하지 않으므로 이 문구가 실제로 어떤 규칙을 촉발했는지는 로그로 확인 불가 |

※ %는 실측 확률이 아니라 등급을 근사 시각화한 값 — Claude의 내부 판단 확률에는 접근할
수 없음.

## 한계

이 리포트는 실행 증거를 기반으로 재구성한 분석이며
Claude의 비공개 내부 사고과정이 아니다. 표의 %는 등급의 근사 표현이며 실측값이 아니다.
