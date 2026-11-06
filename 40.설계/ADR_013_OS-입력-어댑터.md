---
title: ADR - 013 OS 입력 어댑터 (@euta/input-os)
path:  euta-workspace / 40.설계 / ADR - 013 OS-입력-어댑터
ref: ARCH 입력 드라이버 · ADR-007 §입력 드라이버 · SPEC input: os · TEST_OS입력-어댑터_Plan
---

# ADR - 013 OS 입력 어댑터 (@euta/input-os)

## 상태

채택 (2026-09-29, 5단계 구현). 자동 검증은 좌표 변환·키 매핑·모의 백엔드 드라이버·배선까지다. 실제 OS 커서·키보드 입력은 **수동 검증**(권한 허용이 사람 몫) — `60.테스트/TEST_OS입력-어댑터_Plan.md`.

## 맥락

- ARCH: `os` 드라이버는 선택 어댑터. 없으면 명확한 오류, 헤드풀·직렬 강제, 검증은 수동.
- ADR-007: `InputDriverFactory = (ctx: { page, headless }) => InputDriver` 를 `inputDrivers.os` 로 주입. 대상은 Playwright `Locator`.
- 요구: 무료 라이선스, 사용자 PC 에 컴파일러 없이 설치, macOS·Windows 지원, 권한이 없으면 멈추지 않고 즉시 실패.

## 결정

### 1. 백엔드: `robotjs` 0.9.x (MIT)

2026-09-29 npm·GitHub 확인 결과로 고른다(기억이 아니라 현재 상태 기준).

| 후보 | 상태 (2026-09 확인) | 판단 |
|---|---|---|
| **robotjs** (octalmage) | 2026-03 부터 재개발. 0.7.0(03-11) → 0.9.1(08-07). **N-API prebuild 동봉**(darwin-arm64/x64, win32-x64/arm64, linux). MIT. 0.9.0 에 macOS 권한 조회 API(`getAccessibilityPermission` 등, 조회는 대화상자 없음). 유니코드 `typeString`(macOS `CGEventKeyboardSetUnicodeString`, Windows `KEYEVENTF_UNICODE`) | **채택** |
| @jitsi/robotjs | 0.6.24(2026-07), prebuild 동봉, MIT | 대안 1순위. 권한 조회 API 없음 |
| nut.js (`@nut-tree/nut-js`) | 공개 npm 에서 내려감. prebuild 는 **유료 구독**(Core $20/월~). 소스 빌드만 무료 | 제외 — 라이선스 비용·빌드 도구 필요 |
| `@nut-tree-fork/nut-js` | 4.2.6 이후 갱신 없음(2025-03). Apache-2.0, prebuild 포함 | 제외 — 유지보수 멈춤, jimp 등 무거운 의존 |
| 네이티브 없는 헬퍼(Swift/JXA CGEvent, PowerShell SendInput) | 직접 작성 필요. Swift 바이너리 서명·배포, PowerShell 실행 정책, 한 동작마다 프로세스 기동 지연 | 제외 — 유지 부담이 가장 큼. 백엔드 인터페이스 뒤라 나중에 추가 가능 |

- `node-gyp-build` 가 prebuild 를 고르므로 지원 플랫폼에서는 컴파일러가 필요 없다.
- 백엔드는 `OsBackend` 인터페이스(이동·버튼·클릭·휠·키·문자열·권한 조회·화면 크기) 뒤에 둔다. 다른 백엔드로 바꾸려면 이 인터페이스만 구현해 `createOsInputDriverFactory({ backend })` 에 넣는다.

### 2. 패키징·활성화

- 새 패키지 `packages/input-os` = `@euta/input-os`. 이 패키지만 `robotjs` 에 의존하고, `robotjs` 는 **처음 OS 입력이 필요할 때** `require` 한다(평소 실행은 네이티브 모듈을 읽지 않음).
- CLI·MCP 서버는 `@euta/input-os` 를 `optionalDependencies` 로 두고 **동적 import 로 자동 감지**한다. 스크립트(또는 세션 기본값)가 `input: os` 를 쓸 때만 불러온다. `EUTA_OS_INPUT=0` 이면 끈다.
- 라이브러리 사용자(앱 포함)는 `runScript(..., { inputDrivers: { os: createOsInputDriverFactory() } })` 로 직접 주입한다.
- 패키지가 없으면 core 의 `OsInputUnavailableError`(설치 안내 포함), 백엔드 로드 실패·권한 없음은 `OsInputBackendError` 로 스텝 `error`. 어떤 경우에도 권한 대화상자를 띄우지 않는다(`request*` API 를 부르지 않음).

### 3. 헤드풀·직렬·창 준비 (core)

- `needsOsInput(script)` — `defaults.input: os` 이거나 어떤 스텝(그룹·`expectDownload.trigger` 포함)이 `input: os` 면 참.
- 헤드리스면 **브라우저를 띄우기 전에** 실행을 `error` 로 끝낸다(`input: os requires a headed browser …`). 세션은 `defaults.input: os` + 헤드리스로 열면 `open` 이 실패하고, 실행 중 `input: os` 스텝도 헤드리스면 스텝 `error`.
- OS 입력 실행은 **뷰포트 에뮬레이션을 끈다**(`viewport: null`). 헤드풀 Chromium 에서 에뮬레이션을 켜면 Playwright 가 창 크기를 추정 여백으로 맞추는데, 실측 결과 macOS 에서 툴바가 87px 인데 창은 80px 여백으로 잡혀 JS 로 읽은 `outerHeight - innerHeight` 가 실제 콘텐츠 위치와 7px 어긋났다. 에뮬레이션을 끄고 CDP `Browser.setWindowBounds` 로 **실제 콘텐츠 영역이 스크립트 뷰포트와 같아지게** 창 크기를 맞춘다(Chromium). Firefox/WebKit 은 창 크기를 맞추지 않고 경고만 남긴다.
- 결과: OS 입력 실행의 `devicePixelRatio` 는 디스플레이 실제 값(Retina 2)이다 — 스크린샷 해상도가 에뮬레이션 실행과 달라 시각 비교 기준 이미지는 따로 둔다. `browser.device` 프리셋과 `input: os` 는 함께 쓸 수 없다(브라우저를 띄우기 전 실행 `error`).
- 직렬: 한 실행 안의 스텝은 원래 직렬. 한 프로세스에 세션이 여럿이어도(MCP) OS 입력 동작은 **프로세스 전역 뮤텍스**로 한 번에 하나만 수행한다. 프로세스 간 동시 실행은 막지 않는다(문서로 안내).
- 동작마다 `page.bringToFront()` 후 입력한다.

### 4. 좌표 변환 (순수 함수, `geometry.ts`)

```
border  = max(0, (outerWidth - innerWidth·zoom) / 2)          # 좌우 창 테두리 (macOS 0, Windows 보이지 않는 테두리)
originX = screenX + border
originY = screenY + (outerHeight - innerHeight·zoom) - border # 툴바 높이 = 바깥 - 안쪽 - 아래 테두리
dipX    = originX + (box.x + box.width·ax)·zoom               # ax/ay = 0.5 (중앙) 또는 position 오프셋
screen  = space === 'physical' ? dip · (devicePixelRatio / zoom) : dip
```

- 입력: 창 지표(`screenX/Y`, `outer/innerWidth/Height`, `devicePixelRatio`), 요소 상자(뷰포트 CSS px, `scrollIntoViewIfNeeded` 뒤 `boundingBox()`), 페이지 줌(기본 1), 좌표 공간.
- 좌표 공간: macOS 백엔드는 포인트(=DIP) → `dip`. Windows 는 프로세스 DPI 인식 여부에 따라 백엔드 좌표가 논리(DIP) 또는 물리 픽셀이다. `detectCoordinateSpace(백엔드 화면 폭, CSS screen.width, dpr)` 가 둘 중 가까운 쪽을 고르고, `EUTA_OS_INPUT_COORDS=dip|physical` 로 강제할 수 있다.
- 결과 좌표가 창 콘텐츠 영역 밖이면 입력하지 않고 오류.

### 5. 동작 매핑

| 동작 | OS 입력 |
|---|---|
| click · dblclick · hover | 대상 중앙(또는 `position`)으로 사람처럼 이동(선형 보간 N단계) → 버튼 |
| fill | 클릭 → 전체 선택(macOS `command+a`, 그 외 `control+a`) → 삭제 → 문자열 입력 |
| type | 클릭 → 글자마다 입력 + `delay` |
| press | `target` 이 있으면 클릭으로 포커스 → 키 조합. `Meta`/`Control`/`Alt`/`Shift`/`ControlOrMeta` + Playwright 키 이름 → robotjs 키 이름 |
| check/uncheck | 현재 상태가 다를 때만 클릭 |
| scroll | `target` 은 화면 안으로 스크롤(프로그램) 후 커서를 올림, `by` 는 뷰포트 중앙에서 휠, `to` 는 휠을 끝까지(반복 상한) |
| drag | 누르기 → 단계 이동 → 놓기 (HTML5 DnD 포함) |
| select · upload · tap | **지원 안 함** — 네이티브 팝업·파일 대화상자·터치는 OS 입력으로 안정적으로 다룰 수 없음. 해당 스텝은 `input: browser` 로 쓰라는 오류 |

- 한글 등 비 ASCII 는 유니코드 문자 이벤트로 넣는다(IME 를 거치지 않음). **한글 입력기가 켜져 있으면 ASCII 가 조합될 수 있으니** 입력 소스를 영문(ABC)으로 두고 실행한다 — 오류가 아니라 문서화된 한계.

## 이유

- 2026 년 robotjs 가 prebuild·권한 조회 API 와 함께 되살아나, "무료 + 컴파일러 없음 + mac/Windows" 를 동시에 만족하는 유일한 후보였다.
- 백엔드 조회만으로 권한을 판단하므로 권한이 없을 때 대화상자에 걸려 멈추지 않는다.
- 좌표 변환을 순수 함수로 분리해 백엔드 없이 DPR·창 위치·툴바 높이 조합을 단위 테스트한다.
- 에뮬레이션을 끄는 쪽이 "JS 로 읽은 창 지표 = 실제 화면" 을 보장하는 가장 단순한 방법이다(실측 근거).

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| 에뮬레이션 유지 + 툴바 높이 보정값 | 브라우저 버전·OS 마다 다름. 실측 7px 오차 |
| 매 동작 마우스 보정(커서를 옮겨 `mousemove` 좌표로 역산) | 권한이 있어야 가능, 동작이 느려지고 페이지 이벤트를 오염 |
| 클립보드 붙여넣기로 문자열 입력 | 사용자 클립보드를 덮어씀. 유니코드 이벤트로 충분해 기본값으로 두지 않음 |
| core 에 OS 입력 내장 | ARCH "선택 어댑터" 원칙 위반, 모든 사용자가 네이티브 모듈 설치 |

## 결과

- 자동 테스트(vitest): 좌표 변환(DPR 1/2/1.5, 창 위치, 툴바·테두리, 줌, 물리 좌표, 범위 밖), 키 매핑, 모의 백엔드로 click/dblclick/hover/fill/type/press/check/scroll/drag 호출 기록, 미지원 동작 오류, 헤드리스·권한 없음·백엔드 로드 실패 오류, `needsOsInput`, runScript 배선(`inputDrivers.os` 로 모의 드라이버 주입 · 헤드리스면 브라우저 없이 error).
- 수동 검증: 실제 커서·키보드 입력은 `examples/os-input.yaml` + 검증 가이드. 이 Mac(개발 환경)에서는 Accessibility 권한이 없어(`getAccessibilityPermission() === false`) 실제 스모크를 돌리지 않았다.
- 실제 robotjs 어댑터로 `euta run examples/os-input.yaml`(헤드풀)을 돌려, 첫 OS 스텝이 108ms 만에 권한 안내 오류로 끝나고 대화상자 없이 나머지가 skipped 되는 것을 확인.
- MCP `session_open` 의 `defaults.input` 에 `os` 를 받도록 스키마를 넓혔다(헤드리스면 open 오류).
- euta-app 도 CLI·MCP 와 같은 지연 로딩으로 연결했다(`src/main/runner.ts` `osInputOption`, `EUTA_OS_INPUT=0` 으로 끔). Electron 44(Node 24.21)에서 어댑터와 robotjs 네이티브 모듈 로드를 확인했다(2026-09-29).
- 알려진 한계: Windows 는 코드만 작성(미검증) — 휠 단위·DPI 인식은 수동 확인 필요, 서로 다른 배율의 다중 모니터, 한글 입력기 상태, 프로세스 간 동시 실행, 네이티브 select/파일 대화상자.
