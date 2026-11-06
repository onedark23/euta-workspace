---
title: ADR - 034 UI 디자인 시스템 앱 구현
path:  euta-workspace / 40.설계 / ADR - 034 UI-디자인-시스템-앱-구현
ref: PLAN 결정 "UI 디자인 시스템"(2026-10-23) · SPEC_UI-디자인-시스템 · ADR-033(웹 구현) · ADR-027(앱 아이콘)
---

# ADR - 034 UI 디자인 시스템 앱 구현

## 상태

채택 (2026-10-23). euta-app 커밋 `c4b495e`(토큰·글꼴·아이콘·공통 컴포넌트) · `2958dac`(화면 테마 설정) · `a5c1c23`(전 화면 적용).

## 맥락

- SPEC_UI-디자인-시스템(시안 B "부드러운 카드형")을 앱에 적용한다. 토큰 값·이름은 SPEC 이 정본이고, 웹(ADR-033)이 먼저 `packages/web/src/ui/` 를 만들었다.
- 앱 화면은 `styles.css` 342줄에 토큰 몇 개(`--panel` 등)와 색 값이 흩어져 있었고, 버튼은 글자·기호(▶ ☁ ✕ 📄 ⚙)만, 동작 다수가 작은 텍스트 링크였다.
- 제약: 화면 구조·동작·글자 뜻은 바꾸지 않는다(모양만). E2E 5개가 `data-testid`·일부 글자(`☁ Rules`, `passed`)·`.step-main`·네이티브 `<select>`/체크박스(`selectOption`·`check`)에 기대고 있다.

## 결정

### 의존성 (정확한 버전, devDependencies)

- `pretendard` **1.3.9** · `lucide-react` **1.53.0** (웹과 같은 버전, 2026-10-08 공개). 렌더러는 Vite 가 번들하므로 `devDependencies` 에 둔다 — `scripts/package.mjs` 는 `dependencies` 만 스테이징하므로 런타임 node_modules 에 중복되지 않는다.
- 글꼴: `pretendard/dist/web/variable/pretendardvariable.css` 를 `main.tsx` 에서 import → `PretendardVariable.woff2`(2.0MB) 한 파일이 `out/renderer/assets/` 로 번들되고 패키지 `app.asar` 의 `/out/renderer/assets/PretendardVariable-*.woff2` 로 들어간다. 오프라인에서 같은 글꼴. (웹은 네트워크 효율 때문에 dynamic-subset CSS — 앱은 로컬 파일이라 단일 가변 파일이 단순하다.)

### 파일 구성

- `src/renderer/src/ui/tokens.css` — SPEC 토큰 사본(웹 `tokens.css` 와 머리 주석만 다름). 밝은 값 `:root`, 다크 `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) }` + `:root[data-theme="dark"]`, 다크 블록에 `color-scheme: dark`. **렌더러에서 색 값이 있는 유일한 파일.**
- `ui/ui.css` — 기본 요소(글꼴·입력·체크박스 `accent-color`·표·`[hidden]`) + 공통 컴포넌트 모양. 웹 `ui.css` 를 바탕으로 앱에 맞게 조정(아래 "웹과 다른 점").
- `ui/*.tsx` — 앱이 실제로 쓰는 것만: `Button`·`IconButton`·`Badge`(+`runTone`)·`Tooltip`·`Dialog`/`DialogActions`·`EmptyState`·`PageHeader` 는 웹 파일과 **같은 코드**, `Tabs`(버튼판)·`Switch`·`PickList` 는 앱판. `index.ts` 로 내보낸다.
- `styles.css` — 앱 골격과 화면별 모양만(토큰만 사용). 옛 변수(`--panel` → `--surface` 등)와 색 값·안 쓰게 된 규칙(`.seg-toggle`·`.item-btn`·`.order-badge`·`.picker-list`·`.modal*`·`.auth-chip`·`button.primary/danger/small-btn` 등)은 지웠다.

### 화면 골격

- 사이드바(폭 210, `--surface`): 로고(`build/icon.svg` 를 상대 경로 import — 사본을 렌더러에 두지 않음) + `euta` · "새 프로젝트"(테두리 버튼, 주 동작이 아니므로 강조 면 아님) · 프로젝트 목록(폴더/구름 아이콘) · 아래 클라우드 칩 + 앱 설정. 현재 항목 `--accent-soft` 바탕 + `--accent` 글자.
- 프로젝트 머리(`--surface` 띠): 프로젝트 이름 + 상태 배지 + 알약형 `Tabs`(role=tablist 유지). 본문 `--bg`, 여백 `--sp-6`.
- 탭마다 `PageHeader`(제목 `--fs-xl` 700 + 오른쪽 동작 버튼). 카드는 테두리 없이 `--surface` + `--shadow-1`, 표는 `.table-card`(카드 안 표, 넓으면 가로 스크롤).
- 버튼: 아이콘 + 글자, 테두리. 표 행 안은 `sm`. 화면당 강조 면 하나(스크립트 "시연으로 만들기", Run "실행", 상세 "실행"/"저장", 채팅 "Send" 등). 삭제·빼기는 `danger`. 아이콘만인 버튼(순서 ↑↓, 스크린샷 크게 보기, 리포트 브라우저로 열기)은 `IconButton` + 툴팁. 화면 이동만 하는 것("Live 에서 보기", "변경 보기", "시연으로 보여주기 →")은 텍스트 링크로 남겼다.
- 상태 배지: SPEC "상태 매핑"의 색(`runTone` — passed→ok, failed/error/timeout/aborted→bad, running/queued/leased→run, flaky→warn, 나머지 회색) + 점 + 글자.
- `PickList`: 테스트 세트 편집의 "검색 + 체크 + 순번"을 그대로 옮겼다(체크 순서대로 위, 순번 원은 오른쪽 끝). 상태(`picked`·`query`)와 "선택한 N개 추가" 버튼은 `TestSets` 가 갖고, `testId` 로 `testset-picker`·`-search`·`-item`(data-path·data-order) 을 그대로 낸다.
- 대화상자: 시연 시작·충돌·스크린샷 보기를 `Dialog` 로. 확인(`window.confirm`) 흐름은 네이티브 그대로.

### 테마

- 앱 설정 "화면 테마" — 시스템 / 밝게 / 어둡게(`Tabs`, testid `app-theme-<key>`). `AppSettings.theme` 로 기존 `app.json`(AppStateStore)에 저장(기본 `system`, 잘못된 값은 `system`). **새 IPC 채널 없이** 기존 `appSettings.get/set` 을 쓴다.
- main: `applyTheme(nativeTheme, theme)`(app-state.ts) 를 서비스 생성 직후·첫 창 전에 한 번, `appSettingsSet` 마다 한 번. Electron 이 `prefers-color-scheme` 을 그 값으로 바꾸므로 렌더러에 `data-theme` 을 쓰지 않는다. 작업 로그에 "화면 테마=어둡게" 로 남는다.

### xterm 색

- xterm 은 CSS 변수를 못 읽으므로 `terminalTheme()`(Terminal.tsx) 가 `getComputedStyle(document.documentElement)` 에서 토큰을 읽어 `ITheme` 을 만든다: 바탕 `--surface`, 글자 `--text`, 커서 `--accent`, 선택 `--accent-soft`, ANSI red/green/yellow/blue/magenta/cyan = `--bad`/`--ok`/`--warn`/`--run`/`--accent`/`--focus`, black/white 는 밝은·다크에 따라 `--text`·`--muted` / `--surface-2`·`--text`.
- `matchMedia('(prefers-color-scheme: dark)')` 의 `change` 에서 `t.options.theme` 를 다시 건다(테마 설정 → nativeTheme → change 이벤트).

### 웹과 다른 점 (같은 이름·속성, 속 구현만 다름)

- `label` 전역 규칙: 웹은 모든 `label` 을 세로 배치·굵게 한다. 앱은 `<label><input type=checkbox/> 글자</label>` 가 수십 곳이라 `label:has(> input[type=checkbox|radio])` 만 가로 정렬하고, 굵은 라벨은 `.form-row > label:first-child` 에만.
- `Tabs`: 라우터 링크 대신 `<button>`(asTablist 면 role=tab + aria-selected, 아니면 aria-pressed).
- `Switch`: 웹은 입력을 숨기고 옆 span 을 트랙으로 그린다. 앱은 **체크박스 자체를 트랙으로**(`appearance: none` + `::before`) 그려 E2E `check()`·`setChecked()` 가 입력을 직접 누를 수 있게 했다.
- `Tooltip`: 숨김 상태에서 `scale(0)` — 웹판(opacity 0 만)은 보이지 않는 말풍선이 표·사이드바 같은 스크롤 영역을 넓혀 가로 스크롤바가 생겼다(History 표에서 확인). 웹에도 같은 수정을 권한다.
- 목록 그리드에 `grid-template-columns: minmax(0, 1fr)` — 줄임표 이름이 목록을 넓히지 않게.

## 이유

- 토큰·컴포넌트를 웹과 같은 이름·코드로 두면 두 제품이 같아 보이고, 한쪽에서 고친 것을 옮기기 쉽다.
- 네이티브 입력·선택·체크박스를 그대로 두고 CSS 로만 모양을 맞추면 E2E·키보드·접근성 동작이 바뀌지 않는다(모양만 바꾸는 이번 범위).
- 테마를 nativeTheme 로 걸면 OS 대화상자·스크롤바·`color-scheme` 까지 한 번에 따라오고, 렌더러 코드는 미디어 쿼리 하나만 안다.

## 고려한 대안

- **TextField·TextArea·Select·Checkbox·Card·Table 컴포넌트로 모든 폼 교체** — 수백 곳 마크업을 바꿔야 하고 얻는 모양은 전역 CSS 와 같다. 이번엔 전역 스타일 + 클래스(`.form-row`·`.card`·`.table-card`)로. 새 화면부터 컴포넌트를 쓰려면 웹 `Field.tsx` 를 옮겨 오면 된다.
- **Toast 로 저장 결과 표시** — 지금 인라인 안내(`scripts-note`·`detail-note`)를 4초 후 사라지게 하면 동작이 바뀐다(E2E 도 그 글자를 본다). 넣지 않았다.
- **`data-theme` 을 렌더러에서 직접 걸기(웹 방식)** — SPEC 이 앱은 nativeTheme 로 정했다. OS 요소까지 맞추려면 nativeTheme 가 맞다.
- **새 IPC `app:theme-set`** — 기존 앱 설정 IPC 가 같은 일을 하므로 늘리지 않았다.
- **Pretendard dynamic-subset(웹과 같은 CSS)** — 92개 woff2 로 나뉘어 asar 목록·검증이 번거롭다. 로컬 앱은 2MB 한 파일이 단순하다.

## 결과

- 검증(2026-10-23, macOS arm64): `pnpm typecheck` 통과 · `pnpm test` 19파일 175개 통과(테마 저장·적용 2개 추가, `settings()` 기대값에 theme 추가) · `pnpm test:e2e` 5개 통과(3.8분, app.spec 에 "어둡게 → 재시작 후 themeSource=dark" 확인 추가) · `EUTA_PACKAGE_OUT=release/next pnpm package` 성공, `app.asar` 에 `/out/renderer/assets/PretendardVariable-CJuje-Rk.woff2` 포함(asar 의 node_modules 에 pretendard·lucide 없음).
- 색 값 검색: `grep -rnE '#[0-9a-fA-F]{3,8}\b|rgba?\(' src/renderer` 결과는 `ui/tokens.css` 뿐(69줄). `suite-report.ts` 는 범위 밖(손대지 않음).
- 캡처: 스크래치 드라이버(Playwright `_electron`, 임시 userData·픽스처 사이트·가짜 AI)로 사이드바+스크립트·상세·편집·테스트 세트·Run·Live·History·AI 채팅·터미널·작업 로그·프로젝트 설정·앱 설정·시연 대화상자를 밝게/어둡게 두 벌. 1차 캡처에서 PickList 행 폭·Switch 라벨 굵기·History 가로 스크롤(툴팁)·Live 목록 폭을 고쳐 다시 찍었다. 주의: Playwright `electron.launch` 는 기본 `colorScheme: 'light'` 로 미디어를 고정하므로 다크 캡처는 `colorScheme: null` 로 띄워야 nativeTheme 가 반영된다(E2E 캡처는 늘 밝은 모드).
- SPEC 과 다른 점(보고):
  1. 상태 배지 글자는 영어 상태값(`passed`·`failed` …) 그대로 — E2E 가 그 글자를 확인하고 기존 표시 글자라 바꾸지 않았다. 색은 상태 매핑대로. SPEC "배지 글자 통과/실패…" 는 미적용.
  2. 공통 컴포넌트 중 TextField·TextArea·Select·Checkbox·Card·Table·Toast 는 앱에 컴포넌트로 만들지 않았다(전역 CSS·클래스로 같은 모양, Toast 는 미사용).
  3. 토큰 `--r-pill`(999px) — SPEC 표에 이름 없이 "알약 999px" 로만 있어 웹과 같은 이름을 붙였다.
  4. 간격 토큰 밖의 고정 크기: 배지 높이 22·스위치 32×18·순번 원 20·체크박스 16·일부 최소 폭(단계 번호 28, 폼 라벨 96, 실행 목록 260, 채팅 목록 240) — 간격이 아닌 크기라 남겼다.
  5. 프로젝트 이름의 `☁ ` 글자는 그대로(E2E `toHaveText('☁ Rules')`). 다른 버튼의 ▶ ☁ ● ✓ ✕ 📄 ⚙ 🖼 🤖 ⠿ 기호는 lucide 아이콘으로 바꿨다. "묶음↘/묶음↖" 는 화살표가 아이콘이 되면서 "묶음 넣기/묶음 빼기"(툴팁과 같은 뜻), 빈 표 머리 칸에 "동작".
  6. 역할이 없는 색: 작업 로그 행위자 AI = `--accent(-soft)`, MCP = warn, 테스트 세트 = ok, 클라우드 태그 = run.
  7. `[hidden] { display: none !important }` 전역 — 예전엔 `.inline` 의 `display: flex` 가 `hidden` 을 이겨, Run 탭 클라우드 모드에서도 Headless·Browser 옵션이 보였다. 이제 의도대로 숨는다.
- 남은 일: 다크 사이드바에서 남색 로고 대비가 낮다(아이콘 자체 색 — ADR-027 범위). 탭 이름 등 영어 글자(Run·Live·History·Open·Report·Send)는 모양만 바꾸는 이번 범위에서 그대로 — 글쓰기 규칙 적용은 별도 작업.
