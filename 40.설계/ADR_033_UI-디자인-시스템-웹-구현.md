---
title: ADR - 033 UI 디자인 시스템 웹 구현
path:  euta-workspace / 40.설계 / ADR - 033 UI-디자인-시스템-웹-구현
ref: SPEC_UI-디자인-시스템 · PLAN 결정 "UI 디자인 시스템"(2026-10-23) · ADR-027(앱 아이콘) · ADR-029(클라우드 웹 화면)
---

# ADR - 033 UI 디자인 시스템 웹 구현

## 상태

채택 (2026-10-23). SPEC "적용 순서" 1단계 — euta cloud 웹(`euta-cloud/packages/web`)에 토큰·글꼴·아이콘·공통 컴포넌트를 넣고 기존 화면을 모두 교체했다. 앱(2단계)은 아직이다.

## 맥락

- SPEC_UI-디자인-시스템 이 토큰 값·테마 구조·컴포넌트 목록의 정본이다(시안 B "부드러운 카드형").
- 웹은 그동안 `styles.css` 155줄에 자체 색(`#fff`, `--card`, `--line`, `--error` …)과 전역 `button` 스타일을 썼고, 버튼은 글자만, 동작 일부는 글자 링크(`button.link`)였다.
- 조건: 화면 구조·탭 구성·동작·문구 의미는 바꾸지 않는다(모양만). E2E 선택자(`data-testid`·역할·라벨·글자)를 지킨다. 다른 세션이 같은 레포 서버·`suite-report.tsx` 를 동시에 고치고 있었다.

## 결정

- **패키지(정확한 버전 고정)**: `pretendard` **1.3.9**(OFL-1.1, 2023-11-05 배포) · `lucide-react` **1.53.0**(ISC, 2026-10-08 배포 — 1.54.0 은 2주가 안 돼 제외).
- **글꼴**: `main.tsx` 에서 `pretendard/dist/web/variable/pretendardvariable-dynamic-subset.css` 를 import → vite 가 woff2 조각 92개를 `dist/assets` 에 넣는다(외부 CDN 없음). 단일 파일(`pretendardvariable.css`, 2MB) 대신 unicode-range 조각을 써서 쓰는 글자 조각만 받는다. 글꼴 이름 `Pretendard Variable` 은 SPEC `--font` 그대로.
- **파일 배치** (`packages/web/src/ui/`):
  - `tokens.css` — SPEC 표의 사본. 밝은 값 `:root`(+`color-scheme: light`) → `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) }` → `:root[data-theme="dark"]`, 다크 블록에 `color-scheme: dark`. **src 안에서 색 값이 있는 유일한 파일.**
  - `ui.css` — 기본 요소(글꼴·제목·입력·표) + 컴포넌트 스타일.
  - 컴포넌트: `Button`(+`buttonClass` — 파일 열기 링크 등 `<a>` 에 같은 모양) · `IconButton` · `Tooltip` · `TextField`/`TextArea`/`Select`(`Field.tsx`) · `Checkbox`/`Switch` · `Card` · `Badge`(+`runTone` 상태 매핑) · `Tabs` · `Dialog`/`DialogActions` · `EmptyState` · `PageHeader` · `theme.tsx`(`ThemeSelect`) · `index.ts`.
  - `src/styles.css` — 화면별 배치(헤더·단계 카드·설정·테스트 세트 리포트 등)만, 토큰으로 다시 썼다. 쓰지 않게 된 규칙(`button.secondary/.danger/.small/.link`, `.modal*`, `.badge.st-*`, `a.button-link`, `nav.tabs`, `.danger-text`, `.stat`의 테두리 등)은 지웠다.
- **테마**: 상단 계정 영역의 "화면 테마" 선택(시스템 / 밝게 / 어둡게, `data-testid="theme-select"`). `localStorage` 키 `euta-theme`(읽기·쓰기 모두 try/catch), 밝게·어둡게 = `<html data-theme>`, 시스템 = 속성과 저장값 삭제. `index.html` `<head>` 의 인라인 스크립트가 첫 그림 전에 같은 키를 읽어 건다(서버는 SPA 에 CSP 를 걸지 않음 — 리포트 경로만).
- **헤더**: 로고(`/favicon.svg`) + `euta cloud` · 프로젝트/토큰/관리자 링크(아이콘 + 글자) · 오른쪽 계정 영역(이름 · 화면 테마 · 로그아웃 버튼). 로그아웃 상태에서도 테마 선택을 보인다.
- **프로젝트 화면**: 머리(`PageHeader` — 이름 h1 + 역할·보기 전용 배지)와 `Tabs`(알약형, `role=tablist/tab` 유지)는 바탕 위, 각 탭 내용은 `Card`. 현황은 수치 카드 3개 + 카드(최근 실패·예약 2열, 최근 실행). 설정은 섹션마다 카드.
- **버튼 규칙**: 모든 동작 버튼을 `Button`(아이콘 + 글자, 테두리)으로 바꿨다. 동작인데 글자 링크였던 것(값 바꾸기·삭제·닫기·빼기·대상 지정·스크립트 정보 펼치기·접기/펼치기)도 버튼으로. 아이콘만인 버튼(단계·세트 항목 위로/아래로, 단계 스크린샷 보기)은 `IconButton` + `Tooltip`(400ms, CSS `::after` — 접근 이름은 `aria-label`). 화면당 강조(primary)는 주 동작 하나(새 프로젝트·새 스크립트·실행…·저장·클라우드에서 실행·다시 실행 등).
- **상태 배지**: `StatusBadge` → `Badge tone={runTone(status, flaky)}` — passed=ok · failed/error/timeout/aborted=bad · queued/leased/running=run · 불안정 통과=warn · cancelled/skipped=회색(idle). 글자는 기존 `RUN_STATUS_KO` 그대로.
- **폼**: 이름 붙은 입력은 `TextField`/`Select`/`TextArea`(네이티브 컨트롤을 `<label>` 안에 둠 → `getByLabel`·`name`·`data-testid` 그대로). 괄호 안 안내 문구는 라벨 아래 도움말(`--fs-xs`)로 옮겼다(글자는 같음). 필드가 있는 가로 줄(`.inline:has(> .field)`)은 입력 높이가 맞도록 버튼·체크를 라벨 높이만큼 내린다.
- **켜기/끄기**: 예약 폼 "켜기"와 알림 종류 토글은 `Switch`(`role="switch"` 체크박스), 나머지 체크는 `Checkbox`(네이티브 + `accent-color: var(--accent)`).
- **대화상자**: 충돌 창·스크린샷 보기를 `Dialog`로. 스크린샷 보기는 전처럼 아무 데나 누르면 닫힌다(`onBackdropClick`).
- **좁은 화면**: 헤더 줄바꿈, 표는 카드 안에서 가로 스크롤(`.table-wrap`), 단계 카드는 결과·도구 줄을 문장 아래로.

## 이유

- 토큰·컴포넌트를 `ui/` 한 곳에 두면 앱(2단계)이 같은 이름·속성으로 옮겨 갈 수 있고, 색 값이 화면 코드로 새지 않는다(검색으로 확인 가능).
- 네이티브 `<select>`·체크박스·`window.confirm` 을 유지한 것은 E2E(`selectOption`·`check`·`dialog` 이벤트)와 동작을 그대로 두기 위해서다.
- dynamic-subset 글꼴은 한글 위주 화면에서 받는 양이 단일 2MB 파일보다 훨씬 작다.

## 고려한 대안

- **기존 `styles.css` 변수 이름만 바꾸기** — 버튼 아이콘·배지·탭 모양을 컴포넌트로 묶지 못해 앱과 공유할 기준이 안 생긴다. 제외.
- **`pretendardvariable.css`(단일 woff2)** — 오프라인이 중요한 앱에는 맞지만 웹은 조각이 유리. 앱 단계에서 다시 고른다.
- **Toast 로 저장 결과 표시** — 지금 저장 결과는 각 섹션 안의 한 줄(`role=status`, E2E 가 섹션 안에서 찾음)이라 떠다니는 Toast 로 바꾸면 위치·사라짐 동작이 바뀐다. 이번에는 만들지 않음.
- **PickList** — 웹의 여러 개 고르기(테스트 세트 스크립트 추가)는 `select` + "추가" 구조이고 E2E 가 `selectOption` 을 쓴다. 바꾸면 동작 변경이라 이번에는 만들지 않음(순번 원만 붙임).
- **Tooltip 을 실제 DOM 요소로** — 부모의 `textContent`(E2E `toHaveText`)에 툴팁 글자가 섞인다. CSS `::after` 로 결정.

## SPEC 과 다른 점

- **Toast · PickList 미구현** — 위 대안 참고. 웹이 아직 쓰지 않는 구조라 "쓰는 것만 만든다".
- **배지 글자**: SPEC 상태 매핑의 "대기 중"(queued) 대신 기존 문구 "대기"를 유지(문구는 바꾸지 않는다는 조건). "시작 중"(leased)·"시간 초과"·"중단" 도 기존 문구.
- **알약 모서리 토큰 이름** `--r-pill`(999px) — SPEC 표에 이름이 없어 붙였다.
- **카드 안쪽 여백**: 기본 `--sp-4`(SPEC), 단 로그인·가입·장치 승인 같은 좁은 단독 카드(`.card.narrow`)는 `--sp-6`.
- **충돌 창 버튼 순서**: SPEC "확인 버튼은 오른쪽 끝" 대신 기존 ①②③ 순서 + "닫기" 마지막(오른쪽 정렬). 번호가 거꾸로 읽히지 않게.
- **StepForm(폼 편집기)**: 촘촘한 줄이라 `TextField` 로 바꾸지 않고 네이티브 입력에 같은 토큰 스타일(높이 `--h-sm`)만 입혔다. 버튼 5개만 `Button` 으로. 앱의 같은 파일(ADR-029 "같이 맞춤")과는 버튼 import 만 달라졌다 — 앱 2단계에서 맞춘다.

- **버튼·표시 글자 소폭 변경**(의미는 같음, 사용자 확인 대상): `묶음↘`→`묶음 넣기`, `묶음↖`→`묶음 빼기`, `+ 대상 지정`→`대상 지정`, `▸/▾`·`↑↓`·`🖼` 글리프는 아이콘으로, 스크립트 상세 "수정 이후 아직 실행하지 않음" 앞 ` · ` 제거(배지로). 상태 글자(통과·켜짐·활성 등)는 배지로 감쌌다. ARIA 추가: 스크린샷 보기 `role="dialog"`·`aria-modal`, 오류 입력 `aria-invalid`, 테마 선택 `data-testid="theme-select"`. 테스트 세트 폼 항목에 순번 원(목록 번호 대신).

## 결과

- 검증(2026-10-23, macOS): `pnpm typecheck` · `pnpm build`(웹) 통과, 웹 단위 15 통과, 웹 E2E **14/14 통과**(실제 서버 + PostgreSQL + 러너). `src` 에서 `#[0-9a-fA-F]{3,8}` · `rgb(` 검색 = `ui/tokens.css` 외 0건. 빌드 CSS 에 외부 URL 0건, woff2 92개 포함.
- 캡처(밝게·어둡게 각각): 로그인 · 프로젝트 목록 · 현황 · 스크립트 목록 · 스크립트 상세 · 편집(단계 폼 + 스크립트 정보) · 실행 상세(통과·오류) · 테스트 세트 리포트 · 실행 이력 · 예약(+폼) · 테스트 세트(+폼) · 작업 로그 · 설정 · 토큰 · 관리자 · 스크린샷 창, 시스템 다크(OS 다크 + 저장값 없음 → 다크 토큰 확인), 폰 폭 390px(현황·실행 상세). 보고 고친 것: 검색칸 아이콘 겹침, 알림 토글 간격, 폰 폭 단계 카드·표.
- 한계: 스크립트 목록 API(`ScriptDto`)에 `name:` 이 없어 목록·실행 이력은 여전히 경로로 보인다(그라운드룰 "이름으로 보이기"는 별도 작업 — 서버 응답 변경 필요). 다크 모드 탭의 선택 탭(`--surface`)이 바탕 띠(`--surface-2`)보다 어둡게 보인다(시안과 같음, 토큰대로).
