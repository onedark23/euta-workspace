---
title: TEST - OS 입력 어댑터 수동 검증 계획
path:  euta-workspace / 60.테스트 / TEST - OS입력-어댑터_Plan
ref: ADR-013 OS 입력 어댑터 · ADR-007 §입력 드라이버 · SPEC input: os · ARCH 입력 드라이버
---

# TEST - OS 입력 어댑터 수동 검증 계획

> `input: os` 스텝이 실제 OS 커서·키보드로 동작하는지 사람이 확인한다. 권한 허용과 화면 관찰이 필요해 자동 테스트로 대신할 수 없다(ARCH: 검증은 수동).
> 자동 테스트가 이미 보장하는 것: 좌표 변환·키 매핑·모의 백엔드 호출 순서·헤드리스/권한/미설치 오류·배선(실제 헤드풀 창 + 기록용 백엔드). 여기서는 **실제 입력이 화면의 올바른 위치에 닿는지**만 본다.

## 준비

### 공통

- Node 22.13+, pnpm 9. `euta-engine` 에서 `pnpm install && pnpm -r build`.
  - `@euta/input-os` 와 백엔드 `robotjs`(prebuild 포함)가 함께 설치된다. 컴파일러는 필요 없다.
- 브라우저: `pnpm euta install-browsers chromium` (이미 설치돼 있으면 생략).
- 실행 중에는 **마우스·키보드를 건드리지 않는다**. 다른 창이 브라우저를 가리지 않게 한다.
- 입력 소스는 **영문(ABC / US)** 으로 둔다. 한글 입력기가 켜져 있으면 영문 키 입력이 한글로 조합될 수 있다.

### macOS

1. 권한 상태 확인(대화상자 없음):
   ```bash
   cd euta-engine
   pnpm --filter @euta/input-os exec node -e "console.log(require('robotjs').getAccessibilityPermission())"
   ```
   `true` 가 나와야 한다.
2. `false` 면: 시스템 설정 → 개인정보 보호 및 보안 → **손쉬운 사용(Accessibility)** → euta 를 실행하는 앱(터미널, iTerm, VS Code, euta 앱)을 켠다 → 그 앱을 **완전히 종료 후 다시 실행** → 1 을 다시 확인.
3. 화면 기록(Screen Recording) 권한은 **필요 없다** — 어댑터는 화면을 읽지 않는다(스크린샷은 Playwright 가 브라우저 안에서 찍음).

### Windows

- 권한 설정은 없다. 단, 관리자 권한으로 실행된 창에는 일반 권한 프로세스가 입력을 보낼 수 없다(UIPI) — 터미널과 브라우저를 같은 권한으로 실행한다.
- 디스플레이 배율(125%·150%)에서 좌표가 어긋나면 `EUTA_OS_INPUT_COORDS=dip` 또는 `physical` 로 강제해 비교한다(기본은 자동 감지).
- 휠 방향·양은 근사다(100px ≈ 1 노치). 3-6 에서 방향이 반대면 기록한다.

## 실행

터미널 2개.

```bash
# 터미널 A — 픽스처 사이트 (종료: Ctrl+C)
cd euta-engine
pnpm fixture                     # fixture site listening on http://127.0.0.1:4173

# 터미널 B
cd euta-engine
BASE_URL=http://127.0.0.1:4173 pnpm euta run examples/os-input.yaml
# Windows PowerShell:  $env:BASE_URL="http://127.0.0.1:4173"; pnpm euta run examples/os-input.yaml
```

- 스크립트는 `browser.headless: false`, `defaults.input: os` 다. `--headed` 는 필요 없다.
- 결과: 터미널 출력 + `runs/<run-id>/report.html`(스텝별 스크린샷·영상).

## 확인 항목

| # | 스텝 | 눈으로 볼 것 | 기대 결과 (터미널) |
|---|---|---|---|
| 3-1 | goto /widgets | 크롬 창이 뜨고 콘텐츠 영역이 1100×720 (`runs/<run-id>/run.log` 에 `OS input mode: real browser window, content 1100x720 …`) | PASS 1 |
| 3-2 | check 약관 동의 | **실제 커서**가 체크박스로 이동해 클릭, 체크됨 | PASS 2 |
| 3-3 | type 메모 | 커서가 메모 칸 클릭 → `hello 안녕` 이 한 글자씩 입력 | PASS 3 |
| 3-4 | press Enter (단축키) | 단축키 칸 클릭 → Enter | PASS 4 |
| 3-5 | dblclick · hover · drag | 더블클릭 버튼, 마우스 올리기 버튼, 노란 "끌기" 상자를 점선 "놓기" 영역으로 끌어다 놓기 | PASS 5–7, expect 8–12 PASS |
| 3-6 | scroll by 1200 · to bottom | 커서가 뷰포트 중앙으로 가서 휠로 아래로 스크롤, 바닥까지 | PASS 13–15 |
| 3-7 | select (input: browser) | 브라우저 입력으로 요금제 선택(커서 움직임 없음) | PASS 16 |
| 3-8 | 전체 | — | `PASS passed: 17/17 passed` · 종료 코드 0 |

추가 확인(각 1회):

| # | 방법 | 기대 결과 |
|---|---|---|
| 4-1 | 권한을 끈 상태(또는 권한 없는 앱)에서 "실행" 절 명령 실행 | 2번 스텝이 **즉시**(1초 이내) `ERR` + "OS input permission missing (Accessibility) … System Settings > Privacy & Security > Accessibility …". 대화상자가 뜨지 않고 멈추지 않음. 종료 코드 2 |
| 4-2 | `examples/os-input.yaml` 의 `headless: false` 를 `true` 로 바꿔 실행 | 브라우저를 띄우지 않고 `input: os requires a headed browser …` 로 종료 코드 2 |
| 4-3 | `EUTA_OS_INPUT=0` 을 붙여 실행 | 2번 스텝 `OS input adapter not installed …` |
| 4-4 | 4-5 의 MCP 세션에서 스텝 사이에 브라우저 창을 다른 위치(보조 모니터 포함)로 옮긴 뒤 click 스텝을 다시 실행 | 창 위치와 무관하게 요소를 정확히 클릭(좌표는 매 동작 다시 계산). 배율이 다른 모니터에서는 어긋날 수 있음(알려진 한계) |
| 4-5 | MCP: `session_open { defaults: { input: os } }` → `session_step { step: { goto: "/widgets" } }` → `session_step { step: { click: { target: { role: button, name: 더블클릭 } } } }` | 실제 커서로 클릭. 헤드리스(`browser.headless: true`)로 열면 open 이 오류 |

## 결과 기록

| 환경 | OS/배율 | 브라우저 | 3-1~3-8 | 4-1~4-5 | 비고 | 확인자·날짜 |
|---|---|---|---|---|---|---|
| macOS (Apple Silicon, Retina) | | chromium | | | | |
| Windows 11 (100%) | | chromium | | | | |
| Windows 11 (150%) | | chromium | | | | |

- 2026-09-29 개발 환경(macOS 26.5.1, Retina): 터미널에 손쉬운 사용 권한이 없어 실제 입력은 실행하지 않음. 4-1 에 해당하는 동작(첫 OS 스텝이 108ms 만에 권한 안내 오류, 나머지 skipped)만 확인.

## 문제 해결

| 증상 | 원인·조치 |
|---|---|
| `OS input permission missing (Accessibility)` | 준비 › macOS 절차. 권한을 준 뒤 **앱 재시작** 필수. VS Code 통합 터미널이면 VS Code 에 권한을 준다 |
| `could not be loaded … robotjs` | 지원 prebuild 가 없는 플랫폼이거나 설치가 깨짐 → `pnpm install --force`. pnpm 10 이상은 `pnpm approve-builds` 로 robotjs 설치 스크립트를 허용 |
| `OS input adapter not installed` | `@euta/input-os` 가 설치되지 않았거나 `EUTA_OS_INPUT=0`. 코어를 직접 쓰면 `inputDrivers: { os: createOsInputDriverFactory() }` 를 넘긴다 |
| `input: os requires a headed browser` | `browser.headless: false` 또는 `--headed` |
| `cannot be combined with browser.device` | 디바이스 프리셋 제거(OS 입력은 실제 창 모드) |
| 클릭이 요소보다 위/아래로 일정하게 어긋남 | 브라우저 확대/축소가 100% 가 아님(Ctrl/⌘+0) 또는 창이 화면보다 커서 잘림(로그의 `content WxH` 가 요청과 다름 → 더 작은 `viewport`) |
| Windows 에서 좌표가 배율만큼 어긋남 | `EUTA_OS_INPUT_COORDS=physical` 또는 `dip` 로 다시 실행해 맞는 쪽을 기록 |
| 영문이 한글로 입력됨 | 입력 소스를 영문으로 전환 |
| 커서가 다른 창을 클릭 | 실행 중 창을 옮기거나 다른 창이 앞을 가림. 매 동작 전에 `bringToFront` 하지만 macOS 는 앱 활성화를 보장하지 않음 — 첫 클릭 전에 브라우저 창을 한 번 눌러 두면 안정적 |
| `"select" is not supported with input: os` | 네이티브 팝업·파일 대화상자·터치는 미지원. 해당 스텝에 `input: browser` |
| 스크린샷·시각 비교 기준이 달라짐 | OS 입력 실행은 실제 `devicePixelRatio`(Retina 2) — 기준 이미지를 따로 둔다 |
