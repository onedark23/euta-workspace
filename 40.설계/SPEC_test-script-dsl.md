---
title: SPEC - 테스트 스크립트 DSL (YAML)
path:  euta-workspace / 40.설계 / SPEC - test-script-dsl
ref:
---

# SPEC - 테스트 스크립트 DSL (YAML)

> 형식은 YAML, 검증은 JSON Schema(엔진이 생성·배포). 사람과 AI 가 같은 형식을 쓴다 — MCP 탐색 세션에서 AI 가 한 조작도 이 형식으로 내보낸다.
> 세부 문법은 구현 중 조정될 수 있다(위임 범위). 변경 시 이 문서와 스키마를 함께 갱신한다.

## 파일 구조

```yaml
version: 1
name: 로그인 후 보고서 내보내기
description: 선택
target:
  baseUrl: ${env.BASE_URL}
browser:                 # 선택 — 기본 chromium, 1280x800
  engine: chromium       # chromium | firefox | webkit
  viewport: { width: 1280, height: 800 }
  device: iPhone 15      # 선택 — Playwright 디바이스 프리셋 (터치·UA·뷰포트)
  headless: true
  locale: ko-KR
  hasTouch: false        # 선택 — 터치 에뮬레이션 (tap 에 필요. 터치 디바이스 프리셋은 자동으로 켬)
vars:
  reportName: 월간 보고서
defaults:
  timeout: 10s
  input: browser         # browser | os
  settle: network-idle   # 동작 후 화면 안정 판정: network-idle | dom-stable | none
record:                  # 선택 — 기본 모두 켬
  video: true           # true | false | on-failure (실패·오류 실행만 남김)
  trace: true           # true | false | on-failure
  har: true
  console: true
  screenshots: each-step # each-step | on-failure | none
  dom: true             # true | false | on-failure — 단계별 DOM 스냅샷 (선택자 점검·화면 지도용, SPEC_선택자-점검)
steps:
  - goto: /login
  - fill: { target: { label: 이메일 }, value: "${secrets.USER_EMAIL}" }
  - fill: { target: { label: 비밀번호 }, value: "${secrets.USER_PASSWORD}" }
  - click: { target: { role: button, name: 로그인 } }
  - expect: { url: /dashboard }
  - measure: [web-vitals, screenshot]
  - click: { target: { text: 내보내기 } }
  - expectDownload: { name: "*.pdf", minSize: 1kb, contains: "${vars.reportName}" }
```

## 변수 해석

| 문법 | 출처 | 기록 |
|---|---|---|
| `${env.NAME}` | 실행 환경변수 | 그대로 |
| `${secrets.NAME}` | 환경변수 또는 로컬 secrets 파일(레포 밖, `--secrets <file>`) | **항상 마스킹** (`***`) |
| `${vars.NAME}` | 스크립트 `vars` · CLI `--var k=v` | 그대로 |
| `${run.id}` · `${run.startedAt}` | 실행 컨텍스트 | 그대로 |

- 정의되지 않은 참조는 **실행 전 검증 오류**다.

## 대상 지정 (target)

우선순위: 사용자 관점 속성 → 텍스트 → CSS. 사람이 보는 방식으로 찾는 것을 권장한다.

```yaml
target: { role: button, name: 저장 }      # 접근성 role + 이름
target: { label: 이메일 }                  # 폼 라벨
target: { text: 내보내기, exact: false }   # 표시 텍스트
target: { placeholder: 검색 }
target: { testId: submit-btn }
target: { css: "#email" }                  # 최후 수단
target: "#email"                           # 문자열 = css 축약
target: { role: row, name: 홍길동, nth: 0 } # 여러 개 중 선택
```

## 스텝 어휘

공통 옵션: `timeout`, `input`(browser|os), `settle`, `name`(표시명), `optional`(실패해도 계속), `screenshot`(bool), `mask`(스크린샷 가릴 target 목록).

공통 옵션은 **액션 키와 같은 층(형제 키)** 에 둔다. 액션 인자 객체 안에 넣으면 검증 오류다(오타·위치 실수를 바로 잡기 위해 모든 객체는 정의된 키만 허용).

```yaml
- click: { target: { role: button, name: 로그인 } }
  name: 로그인 버튼
  timeout: 5s
  optional: true
- goto: /login          # 인자가 문자열인 스텝도 같은 방식
  settle: dom-stable
```

| 스텝 | 인자 | 설명 |
|---|---|---|
| `goto` | 경로 또는 URL | baseUrl 기준 이동 |
| `click` | `target`, `button`, `clickCount`, `position` | 클릭 |
| `dblclick` · `hover` | `target` | |
| `tap` | `target` | 터치 탭 (터치 에뮬레이션 컨텍스트 필요) |
| `fill` | `target`, `value` | 입력란 값 설정 |
| `type` | `target`, `text`, `delay` | 키 하나씩 입력 (사람 타이핑 흉내) |
| `press` | `key` (예: `Enter`, `Meta+K`), `target` 선택 | 키 입력 |
| `select` | `target`, `value` 또는 `label` | 셀렉트 박스 |
| `check` · `uncheck` | `target` | |
| `upload` | `target`, `files` | 파일 업로드 |
| `scroll` | `target` 또는 `to: top|bottom`, `by: {x,y}` | 스크롤 |
| `drag` | `from`, `to` | 드래그 앤 드롭 |
| `wait` | `for: {visible|hidden|url|network-idle}` 또는 `ms`, 선택 `reload`(간격) | 대기. `reload` 를 주면 조건이 안 맞는 동안 그 간격마다 새로고침하며 기다린다 |
| `expect` | 아래 표 | 검증 (실패 시 스텝 실패) |
| `measure` | 목록: `screenshot`, `web-vitals`, `timing`, `visual` | 측정 기록 |
| `expectDownload` | `trigger`(선택 스텝), `name`, `minSize`, `maxSize`, `type`, `contains`, `csv: {minRows}`, `zip: {entries, min, max}` | 다운로드 결과물 검증 |
| `visual` | `name`, `target`(선택), `threshold` | 기준 이미지 대비 비교. 기준이 없으면 생성 |
| `design` | `name`, `target`(선택), `fullPage` | Figma 디자인 대비 비교 — **실패 없음**(경고·검수용). 기준이 없어도 만들지 않음 → [SPEC_디자인-검수](SPEC_디자인-검수.md) |
| `group` | `name`, `steps` | 묶음 (리포트 표시용) |
| `ai` | `instruction` | **예약** — 전문 모드 2단계. 1단계 엔진은 "지원 안 함" 오류 |
| `manual` | 확인할 내용(문자열) 또는 `{ instruction, reason? }` | 자동화하지 않는 단계(사람 판단·OS 바꾸기·정보 부족). 실행하지 않고 "수동 확인 필요"로 기록(상태 `skipped` + `manual` 표시), 전체 상태에 영향 없음. 리포트에 따로 모아 보여 준다(Live·웹 실행 상세는 같은 표시를 읽는다) → 아래 세부 규칙 · [SPEC_TC-문서-변환](SPEC_TC-문서-변환.md) |

## 검증 (expect)

```yaml
- expect: { url: /dashboard }                          # 포함 또는 정규식 /.../
- expect: { title: 대시보드 }
- expect: { visible: { text: 환영합니다 } }
- expect: { hidden: { role: dialog } }
- expect: { text: { target: { testId: total }, equals: "1,200" } }
- expect: { count: { target: { role: row }, min: 1 } }
- expect: { console: { errors: 0 } }
- expect: { network: { failed: 0, slowerThan: 3s, maxCount: 0 } }
- expect: { metric: { name: LCP, max: 2.5s } }
- expect: { responseTime: { max: 1s } }                # 직전 동작 → 화면 안정
- expect: { enabled: { role: button, name: 원본으로 되돌리기 } }
- expect: { disabled: { role: button, name: 제거 } }
- expect: { attribute: { target: { testId: canvas }, name: aria-busy, equals: "false" } }
- expect: { style: { target: { testId: canvas }, name: cursor, equals: crosshair, hover: true } }   # 마우스를 올린 뒤 계산된 스타일
```

- 한 `expect` 에 조건을 여러 개 둘 수 있다 — 모두 통과해야 스텝 통과.
- `url` · `title`: 부분 문자열, 또는 `/.../flags` 정규식. **정규식으로 보는 것은 끝 `/` 뒤가 비었거나 JavaScript 정규식 플래그(`d g i m s u v y`, 중복 없이)뿐일 때만**이다. 그 밖(`/library/originals` 처럼 경로에 `/` 가 두 번 이상)은 부분 문자열이다. 플래그 형태가 맞는데 본문이 잘못된 정규식이면 스텝 `error`. DOM 조건(url·title·visible·hidden·text·count·enabled·disabled·attribute·style)은 스텝 `timeout` 까지 재시도한다.
- `text`: `equals`(공백 정규화 후 일치) · `contains` · `matches`(정규식) 중 하나 이상. `count`: `equals` · `min` · `max`.
- `enabled` · `disabled`: 대상 1개(여러 개면 strict 오류 → 실패)가 사용 가능 / 사용 불가인지. 판정은 Playwright 규칙 — `disabled` 속성(버튼·입력란 등)·비활성 `fieldset` 안·`aria-disabled="true"`(자신 또는 조상)면 사용 불가. 메시지 예: `enabled role=button name="제거": expected "enabled", got "disabled"`.
- `attribute`: `{ target, name, equals?, contains?, matches? }` — 대상 1개의 속성 값. `equals` 는 그대로 일치(공백 정규화 없음), `contains` 는 포함, `matches` 는 정규식. `equals`·`contains` 에 숫자·불리언을 써도 문자열로 비교(`equals: false` = `"false"`). 셋 다 없으면 **속성이 있기만 하면** 통과. 속성이 없으면 실제 값 `null`(예: `attribute data-mode testId=canvas: expected {"equals":"edit"}, got "view"`).
- `style`: `{ target, name, equals?, contains?, matches?, hover? }` — 대상 1개의 **계산된 스타일**(`getComputedStyle(el).getPropertyValue(name)`, 앞뒤 공백 제거). `equals`·`contains`·`matches` 중 하나 이상 필요(없으면 검증 오류). `name` 은 CSS 이름(`background-color`) — `backgroundColor` 처럼 쓰면 바꿔 읽고, `--변수` 는 그대로. 값은 브라우저가 정규화한 형태(색 = `rgb(…)`, 기본 커서 = `auto`). `hover: true` 면 읽기 전에 그 요소 위에 마우스를 올린다(Playwright 브라우저 hover — `input: os` 와 무관). `:hover` 로 바뀌는 `cursor` 등은 `hover: true` 가 있어야 한다(예: 없으면 `style cursor testId=canvas: expected {"equals":"crosshair"}, got "auto"`).
- `console.errors` · `console.warnings` · `network.failed` 는 **실행 시작부터 누적한 정확한 개수**다. `errors` 에는 uncaught 예외(pageerror)가 포함된다. `failed` = 4xx/5xx 응답 + 중단된 요청(다운로드로 바뀐 요청 제외).
- `network.slowerThan` + `maxCount`(기본 0): 지정 시간보다 오래 걸린 요청 수가 `maxCount` 이하.
- `metric.name`: `LCP` `CLS` `INP` `FCP`(`measure: web-vitals`), `TTFB` `DOMInteractive` `DOMContentLoaded` `Load`(`measure: timing`), `responseTime`(동작 스텝마다 자동). 가장 최근 값으로 판정. `min`·`max` 는 숫자 또는 시간 단위.
- `INP` 는 상호작용 이벤트의 최대 지속 시간으로 근사한다. `CLS` 는 입력 직후를 뺀 layout-shift 합계다.

## 세부 규칙 (1단계 엔진)

| 항목 | 규칙 |
|---|---|
| 기본값 | `timeout` 10s · `input` browser · `settle` network-idle · 뷰포트 1280x800 · 녹화 모두 켬 |
| `settle` | `network-idle` = 진행 중 요청 0개가 400ms 유지 + load. `dom-stable` = DOM 변경 없음 300ms. `none`. 동작 스텝(goto·입력 스텝)에만 적용하며, 시간 안에 안정되지 않으면 **경고만** 남기고 진행한다 |
| `target` | `nth: -1` = 마지막. 동작 스텝에서 여러 요소가 잡히면 오류(→ `nth` 로 지정). `visible`/`hidden` 은 "보이는 요소가 1개 이상 / 0개" |
| `wait.for` | `network-idle` · `{ visible: target }` · `{ hidden: target }` · `{ url: 부분문자열 또는 /정규식/ }` |
| `wait.reload` | `for` 의 `visible`·`hidden`·`url` 과 함께만 쓴다(`network-idle`·`ms` 와 함께면 검증 오류). 최소 간격 10s(미만이면 검증 오류). 두 조건은 엔진 검증에서만 잡고 JSON Schema 에는 없다. 조건을 확인하다가 **마지막 페이지 로드(`load`) 후** `reload` 간격이 지나도 안 맞으면 현재 페이지를 새로고침하고 `settle`(스텝 → `defaults` → `network-idle`)을 적용한 뒤 다시 확인, 스텝 `timeout` 까지 반복. 페이지가 스스로 다시 로드되면 그때부터 다시 센다. 새로고침 1회와 그 뒤 `settle` 은 각각 `min(남은 시간, 간격)` 까지, 새로고침이 실패하면 경고만 남기고 계속 기다린다. 새로고침 횟수 `reloads` 를 스텝 결과·`step.finished` 이벤트·리포트에 남긴다. 시간 초과는 스텝 `error`(예: `target not visible after 12000ms (reloaded 1 time)`). 예: `- wait: { for: { visible: { text: 분석 완료 } }, reload: 5m }` + `timeout: 3h` |
| 클라우드 실행 옵션 | DSL 문법이 아니라 엔진 실행 옵션 — CLI `euta run <script> --cloud --allowed-domains a.com,b.com`(쉼표 구분·여러 번 가능), API `runScript(…, { cloud: true, allowedDomains: [...] })`(`openSession` 도 같음). `--cloud` 가 ①~④ 를 켠다. ① `input: os`(스텝·그룹·`defaults.input`·`expectDownload.trigger`, 보간 후 값 기준) → 실행 전 검증 오류 ② 녹화 기본값 `video`·`trace` = `on-failure`(스크립트 `record` 명시값 우선) ③ **secrets 사용 위치 제한** — `${secrets.*}` 는 `fill.value`·`type.text` 에만(그룹·`trigger` 안 포함). 그 밖(URL·`vars`·`expect`·`name`·target 등)은 위치마다 검증 오류 ④ **허용 도메인** — secrets 가 들어간 `fill`·`type` 스텝은 동작 직전 현재 페이지 호스트 이름(포트 무시, 대소문자 무관)이 허용 도메인과 같거나 점 경계의 하위 도메인일 때만 실행(`login.a.com` ⊂ `a.com`, `evil-a.com` ⊄ `a.com`), 아니면 스텝 `error`(`secret input blocked: page host "…" is not an allowed domain (…)`). 목록 없이 `--cloud` 면 어떤 호스트도 허용하지 않는다(fail closed — 러너가 항상 목록을 넘긴다). `--allowed-domains` 만 주면 ④ 만 켠다. **`--restrict-secrets`(API `restrictSecrets: true`)는 ③ 만 켠다** — 로컬 실행용(로컬 앱의 모든 실행이 ③ + ④ 를 켬, PLAN_euta-cloud 결정 "secrets 규칙 적용 범위" 2026-10-21). 엔진 MCP 서버는 `EUTA_RESTRICT_SECRETS=1` · `EUTA_ALLOWED_DOMAINS=a.com,b.com`(또는 `--restrict-secrets` · `--allowed-domains`)으로 같은 규칙을 `run_script`·`session_*`·`validate_script` 에 적용(기본 끔). ①③ 은 다른 검증 오류와 같이 브라우저 없이 실행 `error` + `validationIssues`(종료 코드 2). 브라우저 없이 같은 규칙을 검사하는 `cloudPrecheck` 는 아래 "사전 검사 API". 근거: [SPEC_클라우드-MCP 보안](SPEC_클라우드-MCP.md) · [ADR-024](ADR_024_엔진-클라우드-보완.md) |
| 사전 검사 API | `cloudPrecheck(source, { vars?, env? }) → { issues: [{ kind, path, message }], secretNames }` — `@euta/core`(또는 가벼운 하위 경로 `@euta/core/precheck`). `kind` = `validation`(스키마·참조) · `os-input` · `secret-placement`. secrets 값 없이 이름마다 자리값으로 검증하고, `secretNames` 는 참조한 이름(정렬·중복 제거). `env` 기본값 `{}`(호출한 프로세스 환경변수를 쓰지 않음 — `${env.*}` 를 쓰는 스크립트는 값을 넘겨야 함). `${run.*}` 는 자리값 |
| `record` `on-failure` | `video`·`trace` 에 `on-failure` — 실행 중에는 기록하고, 실행 상태가 `passed` 면 끝날 때 파일을 남기지 않는다(`failed`·`error` 면 남김). 상태를 먼저 판정한 뒤 저장하므로 지운 파일은 `artifact.saved` 이벤트·run.json·리포트 어디에도 나오지 않고, 버린 항목 이름을 담은 `log` 1줄(예: `trace and video discarded (record: on-failure, run passed)`)만 남는다. run.json `record` 에는 설정값이 그대로. 수 시간짜리 실행의 용량을 줄이기 위함 |
| `record.dom` | 단계별 DOM 스냅샷을 결과 폴더 `dom/` 에 남긴다(기본 `true`, `--cloud` 기본 `on-failure`). 동작 단계는 동작 전·후, `expect`·`wait` 는 판정 시점. `<script>` 제거, password 입력란 값 미저장, secrets 마스킹. 스냅샷 실패는 경고만. `on-failure` 이고 실행이 `passed` 면 `dom/` 을 지우고 `log` 1줄(`DOM snapshots discarded (record.dom: on-failure, run passed)`) — trace·video 문장과 별개. 상세·점검 규칙: [SPEC_선택자-점검](SPEC_선택자-점검.md) · [ADR-030](ADR_030_선택자-점검-구현.md) |
| `upload.files` | 상대 경로는 스크립트 파일 폴더 기준 |
| `expectDownload` | `trigger` 는 하위 스텝 `N.1` 로 실행하고, 그 뒤 시작된 다운로드를 검증한다. `trigger` 가 없으면 아직 검증하지 않은 가장 오래된 다운로드를 기다린다. 모든 다운로드는 시작 즉시 `artifacts/` 에 저장. `type` = 확장자(`pdf` 는 파일 시그니처도 확인). `contains` = 문자열 또는 목록, PDF·텍스트 형식에서 추출한 텍스트(공백 정규화)에 포함. `csv.minRows` = **헤더 제외** 데이터 행 수. `zip` = 파일이 zip 이어야 하고(시그니처 `PK\x03\x04`, 빈 압축은 `PK\x05\x06` — 확장자 무관), 안의 **파일**(폴더 항목 제외) 중 `entries` 글롭(`*`·`?`, 압축 안 전체 경로에 대고 `*` 는 `/` 도 넘음 — `*.png` 는 하위 폴더 것도 셈, 대소문자 무시, 기본 `*` = 모든 파일)에 맞는 개수가 `min` 이상 · `max` 이하. `min`·`max` 둘 다 없으면 `min: 1`. 압축을 풀지 않고 목록만 읽는다. 메시지 예: `download.zip: expected {"entries":"*.png","min":5}, got {"count":3,"files":4}` · zip 이 아니면 `got "not a zip archive"`, 깨진 압축이면 `"corrupt zip archive: …"` |
| `manual` | 실행하지 않는다(브라우저 동작·대기·DOM 스냅샷 없음). 단계 결과 = 상태 `skipped` + `manual: { instruction, reason? }`(문자열 형식이면 `instruction` 만), **`error` 없음**. 내용은 스크립트에 쓴 그대로(`${vars.*}` 등 참조를 풀지 않음 — secrets 값도 나오지 않음). 이 표시는 단계 결과·`step.finished` 이벤트·`steps.jsonl`·`run.json` 에 같이 남는다. 공통 옵션(`name` 등)은 다른 단계처럼 쓸 수 있고 `optional`·`timeout` 등은 효과 없음. 스크린샷은 `record.screenshots`·`screenshot` 규칙을 따르되 `on-failure` 에서는 실패로 보지 않는다(`each-step` 이면 그 시점 화면을 남겨 사람이 확인할 근거로 씀). 앞 단계 실패로 도달하지 못한 `manual` 은 다른 단계처럼 `skipped` + `error: "skipped after step N failed"` 이고 `manual` 표시는 그대로 — 둘을 `error` 유무로 구분한다. `run.log` 는 `INFO … manual skipped … — manual check: <내용>`. report.html: 상태 배지 "수동 확인 필요"(도달 못 한 것은 `skipped` 배지), 요약에 "수동 확인 필요" 개수(모든 `manual` 단계), "수동 확인 필요" 표(단계 id · 이름 · 내용 · 이유). `--cloud` 실행에서도 허용(내용에 `${secrets.*}` 는 secrets 위치 규칙으로 오류). 그룹 안에서도 같음 |
| `visual` | `threshold` = 허용하는 **다른 픽셀 비율**(0~1, 기본 0.01). `fullPage`(bool). 기준 이미지 = `<스크립트 폴더>/baselines/<스크립트 slug>/<name>.png`(CLI `--baseline-dir` 로 변경). 기준이 없으면 만들고 통과. 실행 폴더에 실제(`screenshots/visual-<name>.png`)·기준 사본·diff(`diffs/`)를 남긴다. `measure: [visual]` 은 스텝 `name`(없으면 `step-<id>`)을 이름으로 쓴다 |
| `design` | 디자인 = `<designsDir>/<name>.png` + `.json`(엔진 옵션 `designsDir`, CLI·MCP `--designs-dir`, 기본 `<스크립트 폴더>/designs/`). 단계 상태는 늘 `passed`(요소를 못 찾으면 `failed`), 결과는 `design.compared` · run.json `designChecks[]` · 실행 폴더 `designs/<name>/`. `target` 과 `fullPage` 동시 사용은 검증 오류. 크기 맞추기·기록 규칙은 [SPEC_디자인-검수](SPEC_디자인-검수.md#엔진--비교-규칙) |
| `input: os` | 입력 스텝을 선택 어댑터 `@euta/input-os`(ADR-013)로 보내 실제 OS 커서·키보드로 입력한다. **헤드풀 전용** — 스크립트(스텝·그룹·`expectDownload.trigger`·`defaults.input`)가 쓰면 헤드리스일 때 브라우저를 띄우기 전에 실행 `error`. `browser.device` 와 함께 쓸 수 없다. 이런 실행은 뷰포트 에뮬레이션 없이 실제 창을 `viewport` 크기로 맞춘다(`devicePixelRatio` 는 디스플레이 실제 값). 지원: click·dblclick·hover·fill·type·press·check/uncheck·scroll·drag. `select`·`upload`·`tap` 은 오류(해당 스텝은 `input: browser`). 어댑터가 없으면 "OS input adapter not installed", 권한(macOS 손쉬운 사용)이 없으면 즉시 스텝 `error`. 키 이름은 Playwright 표기(`Enter`, `Meta+K`, `ControlOrMeta+A`) |
| `tap` | 터치 에뮬레이션 컨텍스트(`browser.hasTouch` 또는 터치 디바이스 프리셋)가 아니면 오류 |
| `ai` | 스키마상 유효. 실행하면 "not supported in phase 1" 오류 |
| secrets | `--secrets <파일>`(JSON 또는 .env) → 같은 이름의 환경변수 순으로 찾는다. `vars` 안에서는 `${secrets.*}` · `${vars.*}` 참조 금지. 3자 미만 값은 마스킹하지 않고 경고. secrets 로 채운 입력란은 이후 모든 스크린샷에서 가린다 |
| 스크린샷 | `record.screenshots` 가 `each-step` 이면 스텝마다, `on-failure` 면 실패·오류 스텝만. 스텝 `screenshot: true/false` 가 우선 |
| 스키마 | JSON Schema(draft-07)는 `@euta/core/schema.json`(= `packages/core/schema/euta-script.schema.json`) · `euta schema` |

## 단위 표기

- 시간: `500ms`, `2s`, `1m`, `1h` 또는 숫자(ms). 소수 가능(`1.5h`).
- 크기: `512b`, `10kb`, `2mb`.

## 실행 결과 상태

`passed` · `failed`(expect 실패) · `error`(실행 불가·타임아웃) · `skipped`(optional 실패 후 등). 실행 전체 상태는 스텝 상태의 최악값이다.

- 심각도: `error` > `failed` > `passed`. `skipped` 는 전체 상태 계산에서 뺀다(optional 스텝 실패가 실행을 깨지 않도록).
- `optional: true` 스텝이 실패·오류면 `skipped` 로 기록하고(오류 메시지는 남김) 계속한다.
- `manual` 스텝은 `skipped` + `manual` 표시(오류 없음)로 기록한다 — 실행 상태 계산에서 빠지고, 요약 `skipped` 수에 들어간다(별도 개수는 단계의 `manual` 표시로 센다).
- optional 이 아닌 스텝이 실패·오류면 거기서 멈추고 나머지 스텝을 `skipped` 로 기록한다(그룹 안에서도 동일, 그룹 상태 = 하위 스텝 최악값).
- 검증 오류(스키마·정의되지 않은 참조)는 브라우저를 띄우지 않고 실행 상태 `error` 로 기록한다(`run.json` 의 `validationIssues`).
- CLI 종료 코드: `0` passed · `1` failed · `2` error(검증 오류·사용법 오류 포함).

## Change Log

### 2026-10-26 — 단계 16 보완 구현 (ADR-037) — `manual` 은 상태 `skipped` + `manual: { instruction, reason? }` 표시(오류 없음, 도달 못 하면 `error` 있음), 리포트 "수동 확인 필요" 배지·개수·표. `enabled`·`disabled`(Playwright 활성 판정)·`attribute`(셋 다 없으면 속성 존재, 숫자·불리언은 문자열로 비교)·`style`(계산된 스타일, 비교 조건 1개 이상 필수, `hover`) 은 다른 DOM 조건처럼 `timeout` 까지 재시도. `expectDownload.zip`(시그니처 확인, 폴더 항목 제외, `entries` 기본 `*`, `min`·`max` 없으면 1개 이상). 실행 요약(`summary`) 형식은 바꾸지 않음

### 2026-10-26 — 단계 16 보완 설계 (TC 문서 변환): `manual` 단계, `expect` 의 `enabled` · `disabled` · `attribute` · `style`(`hover` 선택), `expectDownload.zip`(압축 안 파일 이름 패턴 · 개수). 기존 문법은 바꾸지 않음. 세부는 구현 때 이 문서·스키마·ADR 로 확정

### 2026-10-23 — `record.dom` 추가 (단계별 DOM 스냅샷, 기본 켬) — 선택자 점검·화면 지도용 (SPEC_선택자-점검). 기존 문법 변경 없음

### 2026-10-23 — `design` 단계 추가(Figma 디자인 대조, 실패 없음) — 상세는 SPEC_디자인-검수. 기존 문법은 바꾸지 않음

### 2026-10-23 — `url`·`title` 패턴 해석 명확화: 끝 `/` 뒤가 정규식 플래그일 때만 정규식, 아니면 부분 문자열 (`/library/originals` 가 플래그 `originals` 정규식으로 오해석되던 버그). 문법 변경 없음

### 2026-10-21 — 실행 옵션 `restrictSecrets` / CLI `--restrict-secrets` / MCP `EUTA_RESTRICT_SECRETS`·`EUTA_ALLOWED_DOMAINS` (secrets 사용 위치 제한만 따로 켬 — 로컬 앱의 모든 실행용). 문법 변경 없음. [ADR-024](ADR_024_엔진-클라우드-보완.md) 결과

### 2026-10-20 — euta cloud 대비 보완 구현 반영 (C1 엔진)
- 위 보완 4건 구현. 실제 동작 명시: `wait.reload` 간격 기준(마지막 `load`)·새로고침 1회 시간·실패 시 계속·`reloads` 기록, `on-failure` 는 저장 전에 버림, CLI `--cloud` · `--allowed-domains`(목록 없으면 fail closed, 단독이면 도메인 검사만), 도메인 비교 규칙, 사전 검사 API `cloudPrecheck`. 문법 변경 없음. 구현 세부: [ADR-024](ADR_024_엔진-클라우드-보완.md)

### 2026-10-20 — euta cloud 대비 보완 설계 (C1)
- `wait.reload`(새로고침하며 기다리기), `record.video`·`record.trace` 의 `on-failure`, 시간 단위 `h` 추가, 클라우드 실행 옵션(`input: os` 거절 · 녹화 기본값 · secrets 사용 위치 · 허용 도메인). 기존 문법은 바꾸지 않음. 근거: [PLAN_euta-cloud](../20.프로젝트관리/PLAN_euta-cloud.md) 결정 "엔진 보완" · [ARCH_euta-cloud 엔진 보완](ARCH_euta-cloud.md)

### 2026-09-29 — 5단계 OS 입력 어댑터 반영
- `input: os` 규칙 구체화: 헤드풀 강제(실행 전 오류), `browser.device` 와 병용 불가, 실제 창 모드, 지원·미지원 동작, 키 이름. 문법 변경 없음.

### 2026-09-29 — 1단계 엔진 구현 반영
- 공통 옵션 위치를 형제 키로 명시, `browser.hasTouch` 추가, `expect` 다중 조건·세부 판정 규칙, `expectDownload`·`visual`·secrets·상태 계산 세부 규칙, CLI 종료 코드 추가. 기존 문법은 바꾸지 않음.

### 2026-09-29 — 초안
