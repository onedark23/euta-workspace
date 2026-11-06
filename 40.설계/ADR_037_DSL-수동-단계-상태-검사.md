---
title: ADR - 037 DSL 수동 단계 · 상태 검사 구현 (manual · enabled/disabled · attribute · style · expectDownload.zip)
path:  euta-workspace / 40.설계 / ADR - 037 DSL-수동-단계-상태-검사
ref: PLAN 단계 16 · SPEC_test-script-dsl(manual · expect · expectDownload) · SPEC_TC-문서-변환 "AI 변환" 변환 규칙 · SPEC_선택자-점검 · ADR-024 · ADR-030
---

# ADR - 037 DSL 수동 단계 · 상태 검사 구현

## 상태

채택 (2026-10-26, 단계 16 엔진 DSL 보완). euta-engine `packages/core`(+ `mcp-server` cheatsheet) — 커밋 전(사용자 검토 후 로컬 커밋). macOS(arm64)에서 core 타입체크·빌드, core 단위·E2E 전체, mcp-server 테스트로 검증했다.

## 맥락

- [SPEC_TC-문서-변환](SPEC_TC-문서-변환.md) 의 AI 변환 규칙은 자연어 TC 단계를 DSL 로 옮길 때 ① 사람 판단·자동화할 수 없는 동작·정보 부족은 `manual` 단계로 ② 활성/비활성·커서 모양 같은 결과는 새 `expect` 조건으로 ③ "이미지 여러 장 zip 다운로드" 는 `expectDownload.zip` 으로 쓰게 했다. [SPEC_test-script-dsl](SPEC_test-script-dsl.md) 은 문법만 정하고(2026-10-26 설계) 상태 표현·메시지·기본값은 구현 때 정하게 했다. 이 ADR 이 그 세부를 기록한다.
- 제약: 실행 결과를 읽는 쪽(데스크톱 앱 Live·결과 화면, 클라우드 웹 실행 상세, MCP `run_script` 요약, `index.sqlite`)은 이미 단계 상태 `passed`·`failed`·`error`·`skipped` 네 가지와 `summary { total, passed, failed, error, skipped }` 를 안다. 이번 변경으로 이들을 깨지 않아야 한다.

## 결정

### 1. `manual` 상태 = `skipped` + `manual` 표시 (새 상태 값 아님)

- 단계 결과(`StepResult`)·`step.finished` 이벤트(→ `steps.jsonl`)·`run.json` 에 `manual: { instruction, reason? }` 를 붙이고 상태는 `skipped`. 표시가 객체라 "수동 단계인가" 와 "무엇을 확인하나" 를 한 필드로 준다(소비자가 `step.started.step` 을 다시 뒤질 필요 없음).
- 도달한 `manual` 은 **`error` 가 없다.** 앞 단계 실패로 도달하지 못한 `manual` 은 기존 `skip()` 경로로 `error: "skipped after step N failed"` 가 붙고 `manual` 표시는 그대로 — 둘은 `error` 유무로 구분한다. MCP 요약의 실패 목록 필터(`skipped` 이면서 `error` 가 `skipped after step` 이 아닌 것)에 수동 단계가 섞이지 않는 것도 이 규칙 덕분이다(내용을 `error` 에 넣지 않음).
- 실행 상태: `worstStatus` 가 `skipped` 를 빼므로 손대지 않고 "전체 상태에 영향 없음" 이 된다. 그룹 상태도 같음.
- 요약: `RunSummary` 에 `manual` 개수를 **넣지 않았다.** `index-db` 열·`session.ts` 기본값·MCP 요약 타입·기존 테스트가 이 모양을 그대로 쓴다. 수동 단계는 `skipped` 수에 들어가고, 개수가 필요한 곳(리포트)은 단계의 `manual` 표시를 센다.
- 내용은 **스크립트에 쓴 그대로(raw)** 기록한다. `skip()` 은 raw 단계만 받으므로 실행·건너뜀 두 경로가 같은 값을 내려면 raw 가 맞고, `step.started.step` 도 raw 다. `${secrets.*}` 참조는 이름만 남아 값이 새지 않는다(로컬에서도). 앱 단계 문장(`describeStep`)도 raw 를 `{var}`·`🔒NAME` 표기로 보여 준다.
- 실행하지 않는다: 입력 드라이버·settle·DOM 스냅샷·`responseTime` 없음, 타임아웃만 평소처럼 설정. `optional` 경고 로그(`optional step skipped; continuing`)는 수동 단계에 내지 않는다.
- 스크린샷: `record.screenshots`·`screenshot` 규칙을 그대로 따르되 `on-failure` 에서는 실패로 보지 않는다. `each-step`(기본)이면 그 시점 화면이 남아 사람이 나중에 확인할 근거가 된다.
- `run.log`: `step.finished` 의 `skipped` 는 WARN 이지만 도달한 수동 단계는 INFO + `— manual check: <내용>`.
- report.html: 상태 배지 "수동 확인 필요"(보라색 `--man` 토큰, 다크 모드 값 포함), 요약 카드 "수동 확인 필요 N"(모든 `manual` 단계), 타임라인 앞에 "수동 확인 필요" 표(단계 id · 이름 · 내용 · 이유, 도달 못 한 단계는 "실행 안 됨: …"). 수동 단계가 없으면 카드·표 모두 없음.
- `--cloud`: 브라우저 동작이 없어 별도 규칙이 없다. secrets 위치 규칙은 그대로 적용돼 내용에 `${secrets.*}` 를 쓰면 `secret-placement` 오류.
- 대안:
  - 새 상태 `manual` — 앱·클라우드 웹·MCP·`index.sqlite` 의 상태 색·필터·개수 계산을 모두 바꿔야 하고, 모르는 상태를 만난 옛 앱은 깨진다. 버림.
  - 상태 `passed` + 표시 — "통과" 개수에 사람이 확인하지 않은 단계가 섞여 결과를 부풀린다. 버림.
  - 내용을 `error` 에 — 기존 소비자가 실패 이유로 읽는다(MCP 실패 목록). 버림.

### 2. `enabled` · `disabled`

- Playwright `locator.isEnabled()` 를 `poll`(100ms 간격, 스텝 `timeout` 까지)로 반복, 한 번 읽기 제한 `min(timeout, 2s)`. 기존 `text` 와 같은 틀(strict — 여러 개면 오류가 실제 값으로 남음).
- 판정 = Playwright 정의: `disabled` 속성이 있는 폼 요소·비활성 `fieldset` 안·자신 또는 조상의 `aria-disabled="true"`. 사용자에게 보이는 "비활성" 과 가장 가깝고 직접 구현할 필요가 없다.
- 결과 값은 불리언 대신 `"enabled"`/`"disabled"` 문자열 — `disabled …: expected true, got false` 처럼 뜻이 뒤집혀 읽히는 메시지를 피하려고. 예: `enabled role=button name="제거": expected "enabled", got "disabled"`.

### 3. `attribute`

- `locator.getAttribute(name)` 를 같은 `poll` 로. 속성이 없으면 실제 값 `null`.
- 비교: `equals` 그대로 일치(속성 값은 공백을 정규화하지 않음 — `text` 와 다름), `contains` 포함, `matches` 정규식(잘못된 정규식은 바로 스텝 `error` — `text` 는 시간 초과까지 기다리지만 새 조건은 미리 컴파일). 여럿이면 모두 만족.
- `equals`·`contains` 에 문자열·숫자·불리언 허용, 문자열로 바꿔 비교. `aria-busy`·`aria-expanded`·`aria-pressed` 처럼 `true`/`false` 값이 흔한데 YAML 에서 따옴표를 빼면 불리언이 되어 검증 오류가 나는 실수를 막는다.
- 비교 조건이 하나도 없으면 "속성이 있음" 으로 판정(기대값 표시 `"present"`). `href`·`download` 같은 존재 자체가 의미인 속성용.

### 4. `style`

- `locator.evaluate(el => getComputedStyle(el).getPropertyValue(name).trim())` 를 같은 `poll` 로. `name` 이 `backgroundColor` 처럼 camelCase 면 `background-color` 로 바꾸고 `--사용자 속성` 은 그대로(`getPropertyValue` 는 CSS 이름만 받음).
- `hover: true` 면 **읽을 때마다** `locator.hover()`(Playwright 브라우저 마우스) 후 읽는다. 처음 한 번만 올리면 요소가 늦게 나타날 때 실패하고, 반복 hover 는 같은 자리라 부작용이 없다. `expect` 는 입력 단계가 아니라 `input: os` 드라이버를 쓰지 않는다. 조건 이름에 `(hover)` 를 붙인다(`style cursor (hover) testId=canvas`).
- 비교 조건(`equals`·`contains`·`matches`) 중 하나 이상 필수(스키마 refine) — 계산된 스타일은 늘 값이 있어 "있음" 판정이 의미 없다. `equals`·`contains` 는 문자열·숫자(`opacity: 1`).
- 값은 브라우저 정규화 형태(색 `rgb(…)`, 기본 커서 `auto`)라고 SPEC 에 적었다.

### 5. `expectDownload.zip`

- zip 판별 = 파일 시그니처 `PK\x03\x04`(빈 압축 `PK\x05\x06`). 확장자는 보지 않는다(확장자는 기존 `type`). 시그니처가 아니면 `got "not a zip archive"`, 목록을 못 읽으면 `"corrupt zip archive: …"`.
- 목록 읽기 = fflate `unzipSync(bytes, { filter })` 에서 filter 가 항상 `false` 를 돌려 **압축을 풀지 않고** 이름만 모은다. 폴더 항목(`/` 로 끝남)은 뺀다. fflate 는 이미 core 의존성(trace 처리) — 새 의존성 없음. 순수 함수 `zipEntryNames` 로 `downloads.ts` 에 두고 단위 테스트한다.
- 개수 = `entries` 글롭(기존 `globToRegExp` — `*`·`?`, 대소문자 무시)에 맞는 파일 수, 대상은 압축 안 전체 경로라 `*` 가 `/` 도 넘는다(`*.png` 가 하위 폴더 것도 셈 — 사용자가 "png 몇 장" 을 기대하는 대로). 기본 `entries` = `*`.
- `min`·`max` 둘 다 없으면 `min: 1`(조건이 아무것도 확인하지 않는 일을 막음). `max` 만 있으면 `min` 은 0(`{ entries: "*.exe", max: 0 }` = 실행 파일 없음).
- 결과 값 = `{ count, files }`(맞은 개수 · 전체 파일 수) — 글롭이 틀렸는지 바로 보인다. 예: `download.zip: expected {"entries":"*.png","min":5}, got {"count":3,"files":4}`.

### 6. 주변 반영

- 스키마(zod) → JSON Schema(`schema/euta-script.schema.json`, 빌드가 다시 씀) · TS 타입 · `ACTIONS` 끝에 `manual`.
- 단계 문장(`describeStep`): `수동 확인: <내용>`(+ 줄 `이유: …`, 아이콘 🙋), `'X' 버튼 사용 가능(활성)` · `사용 불가(비활성)`, `[canvas] 속성 aria-busy 값이 'false' 와 같음` · `속성 href 있음`, `[canvas] 스타일 cursor 값이 'crosshair' 와 같음 (마우스 올린 상태)`, 다운로드 줄 `압축 안 *.png 파일 3개 이상`(`min~max개`·`N개 이하`). 불안정(CSS) 대상 세기에 `enabled`·`disabled` 키 추가.
- 선택자 점검(`collectTargets`·`verdictOf`): 새 조건 4종의 대상도 모은다(`expect.enabled` 등). 실행기가 strict 로 1개를 읽으므로 `expect.text` 와 같은 판정 — 1개 `ok`, 0개 `missing`(실행에서 통과했으면 `dynamic`), 2개 이상 `ambiguous`. SPEC_선택자-점검 표·Change Log 반영.
- `rewrite.ts`(그룹만 특별 취급)·`cloud.ts`·`precheck.ts` 는 변경 없음 — 단계 종류를 나열하지 않는다.
- MCP 서버 `cheatsheet.ts`(단계 표·`expect` 예시·상태 설명) 갱신. MCP 요약(`summary.ts` `stepSummary`)의 단계 줄에 `manual: { instruction, reason?, note }` 를 덧붙인다(기존 필드는 그대로 — 추가만). `note` = 도달했으면 `"수동 확인 필요"`, 앞 단계 실패로 도달 못 했으면 `"실행 안 됨"`. 실패 목록(`failures`)에는 넣지 않는다.

## 결과

- 엔진 단위(core) 101 → 106건(13파일 통과): 스키마(새 형식 허용 · `manual` 빈 문자열/모르는 키 · `style` 비교 조건 없음 · `attribute.name` 없음 · `zip.min` 음수 · `zip` 모르는 키), 단계 문장(수동·활성·속성·스타일·zip 4형태·불안정 대상 3), `zipEntryNames`(폴더 제외 · 하위 폴더 글롭 · 빈 압축 · zip 아님 · 깨진 압축), 선택자 점검 수집·판정, `cloudPrecheck`(수동·새 조건 0건, 수동 안 secrets 는 `secret-placement`), 리포트(배지 · 개수 · 표 · 이스케이프 · 도달 못 한 단계는 `skipped` 배지).
- 픽스처 `/states`(활성·`disabled`·`aria-disabled` 버튼, 300ms 뒤 `aria-busy=false` 로 바뀌는 캔버스, `:hover { cursor: crosshair }`, `images.zip`(png 3 · txt 1 · 폴더) · 확장자만 zip 인 텍스트 다운로드) + E2E `manual-checks.test.ts` 4건: 수동 단계(최상위·그룹 안)가 있어도 `passed`·요약 `{9,7,0,0,2}`·`steps.jsonl`/`run.json`/`run.log`/report 표시 · 실패 메시지 6종(활성 · 속성 값 · 속성 없음 · hover 없는 커서 `auto` · zip 개수 · zip 아님) · 실패 뒤 수동 단계는 `error` + `manual` · `--cloud` 실행 허용. core 전체(단위 + E2E) 24파일 156건, mcp-server 20건 통과. 리포트 화면 캡처로 배지·카드·표 확인.
- MCP `run_script`·`get_run`·`session_close` 의 단계 요약에 `manual` 표시(`note: "수동 확인 필요"` / `"실행 안 됨"`)가 붙어, AI 가 수동 단계를 그냥 건너뛴 단계와 구분한다(mcp-server 단위 1건).
- 남은 일(이 ADR 범위 밖): 앱 Live·결과 화면, 클라우드 웹 실행 상세에 `manual` 표시를 읽어 "수동 확인 필요" 로 보여 주기 — 지금은 `skipped`(오류 없음)로 보인다. 탐색 세션 `exportScript` 는 `passed` 단계만 내보내므로 세션에서 실행한 `manual` 단계는 빠진다.
