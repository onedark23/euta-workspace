---
title: SPEC - 앱 배포 · 업데이트
path:  euta-workspace / 40.설계 / SPEC - 앱-배포-업데이트
ref:
---

# SPEC - 앱 배포 · 업데이트

> 상태: **배포 · 첫 릴리스** — `0.1.0-alpha.1` GitHub 게시(2026-11-09) · 서버 미러 자동 게시(2026-11-10) · 릴리스 자동화(태그 → GitHub Actions, 빌드만 시험 통과 2026-11-10, [ADR-048](ADR_048_앱-릴리스-자동화.md)) · 업데이트 원본 선택([ADR-047](ADR_047_앱-업데이트-원본-선택.md)). 남은 것: 태그로 게시까지 1회 · 실제 업데이트 1회(alpha.1 → alpha.2) · 맥 · Windows · Linux 실기. 이전: 설계 · 구현 2026-10-30(클라우드 [ADR-041](ADR_041_앱-업데이트-배포-클라우드-구현.md), 앱 [ADR-042](ADR_042_앱-배포-업데이트-앱-구현.md)). 결정 정본은 [PLAN 결정 사항](../20.프로젝트관리/PLAN_end-user-테스트툴-종합계획.md#결정-사항) "앱 배포".
> 선행: [ARCH 앱 구성](ARCH_end-user-테스트툴.md#앱-구성) · [ARCH_euta-cloud 배포 · 운영](ARCH_euta-cloud.md#배포--운영) · [SPEC_클라우드-앱-연동](SPEC_클라우드-앱-연동.md) · [RESEARCH_AI-구독-연동-방식](../35.분석/RESEARCH_AI-구독-연동-방식.md).

## 목적

- 지금 앱은 이 Mac 에서만 도는 개인 테스트용 `.app`(ad-hoc 서명)이다. **사내 전체**가 설치하고, 새 버전을 앱 안에서 받게 한다(사용자 결정, 2026-10-30).
- 앱 AI 채팅은 지금 방식 그대로 배포한다 — 각 사용자가 자기 PC 에서 공식 `claude auth login` 으로 자기 Claude 구독에 로그인해 자기가 쓴다(사용자 결정, 2026-10-30. 상위 `CLAUDE.md` AI 연동 금지선 갱신).
- 클라우드 API 가 바뀌어도 옛 앱이 클라우드 데이터를 망가뜨리지 않게 **최소 버전**을 둔다.

## 범위

| 포함 | 범위 밖 (후속) |
|---|---|
| 버전 · 릴리스 빌드(macOS arm64 · x64, Windows x64, **Linux x64**) · 릴리스 노트 | Linux arm64 · deb/rpm · 앱 스토어 |
| 업데이트 원본 두 곳 — GitHub Releases(공개 기본) · euta cloud 서버(그 서버를 운영하는 조직의 미러) — 와 고르는 규칙 · 웹 "앱 내려받기" 화면 | CDN · 사용자가 원본을 직접 고르는 설정 |
| 앱 업데이트 확인 · 내려받기 · 설치(서명 상태에 따라 단계적) | 업데이트 강제 재시작 |
| 클라우드 최소 앱 버전 · 미만이면 클라우드 기능 잠금 | 로컬 기능 잠금 |
| 채널 — 지금은 알파(alpha)만, 베타·정식은 나중에 추가 | 사용자별 단계 출시(%) |
| 런타임 의존성 정확한 버전 고정 해제 | 앱 용량 줄이기(번들 CLI 228MB 유지) |

## 버전 · 릴리스

- 앱 버전 = `euta-app/package.json` `version`, SemVer. **첫 배포는 `0.1.0-alpha.1`**(사용자 결정, 2026-10-30) — 알파 동안은 `0.1.0-alpha.N` · `0.2.0-alpha.1` 처럼 올린다. prerelease 이름이 곧 채널(`alpha` → `beta` → 표기 없음 = 정식).
- 릴리스 = 버전 올리기 커밋 + git 태그 `app-v<버전>` + 릴리스 노트(`euta-app/RELEASES.md`, 버전별 "바뀐 것" 몇 줄 — 업데이트 안내에 그대로 보인다).
- 빌드 산출물(electron-builder):

| 플랫폼 | 설치용 | 업데이트용 |
|---|---|---|
| macOS arm64 · x64 | `.dmg` | `.zip` + `latest-mac.yml` |
| Windows x64 | NSIS `.exe` | 같은 `.exe` + `latest.yml` (차등 블록맵) |
| Linux x64 | `.AppImage` | 같은 `.AppImage` + `latest-linux.yml` (AppImage 는 electron-updater 자동 업데이트 지원, 서명 불필요) |

- **릴리스는 태그 push 로 자동**(2026-11-10, [ADR-048](ADR_048_앱-릴리스-자동화.md)): 앱 레포에 `app-v<버전>` 태그를 올리면 GitHub Actions(`.github/workflows/release.yml`)가 빌드 · 게시한다 — macOS 러너가 맥 arm64 · x64 와 Windows x64(교차, 아래와 같음), Ubuntu 러너가 Linux x64(그 자리에서 node-pty 컴파일). 엔진은 같은 위치에 `onedark23/euta-engine` 을 받는다(기본 `main`, 실행 입력으로 바꿈). 모든 빌드가 끝나면 한 작업이 `release:github --publish` 로 게시. 버전 · `RELEASES.md` 절이 없거나 태그와 버전이 다르면 빌드 전에 멈춘다. 아래 "이 맥에서 빌드" 는 예비 경로로 남는다.
- **모든 플랫폼을 이 맥에서 빌드**한다. 네이티브 모듈 중 `robotjs` 는 darwin/win32/linux prebuild 를 포함하고, `node-pty` 는 darwin/win32 prebuild 만 있어 **Linux 는 Docker(`electronuserland/builder`, linux/amd64)에서 Electron 용으로 컴파일**한다. SQLite 는 Node 내장. 번들 Claude CLI 는 대상 플랫폼 패키지(`claude-agent-sdk-<os>-<cpu>`)를 stage 에 따로 설치한다. Windows 설치 파일(NSIS)은 이 맥(macOS 10.15+)에서 electron-builder 26 이 Wine 없이 만들도록 설정했다(언인스톨러 추출 · 리소스 편집이 JS — 코드 읽기 근거, **첫 실제 빌드에서 확인**, 안 되면 `electronuserland/builder:wine`, [ADR-042](ADR_042_앱-배포-업데이트-앱-구현.md)). 엔진은 지금처럼 tarball.
- 명령: `pnpm release [--mac] [--win] [--linux]` → `release/<버전>/<mac|win|linux>/`, `EUTA_ADMIN_TOKEN=… pnpm release:upload [--mac|--win|--linux] [--publish]`(플랫폼을 나눠 같은 draft 에 올리고 마지막에 게시). 파일 이름 `euta-<버전>-<mac|win|linux>-<arm64|x64>.<ext>`(서버가 이름으로 분류). 맥 두 arch 는 한 번의 electron-builder 실행으로 만들어 `latest-mac.yml` 하나에 둘 다 적힌다. `latest*.yml` 의 `releaseNotes` = RELEASES.md 그 버전 절.
- 교차 빌드는 실행 확인을 대신하지 못한다 — **Windows · Linux 는 실제 기기에서 설치·실행·업데이트 1회 확인**해야 배포한다(둘 다 지금까지 미검증). Windows PC 는 있음(사용자). Linux 는 Ubuntu 계열 데스크톱(X11) 기준 — OS 입력(`robotjs`)은 X11 에서만, Wayland 는 범위 밖.
- 버전 표기: `package.json` 의존성은 **범위**로 적는다(정확한 버전 고정 금지 — 재현성은 lockfile). 정확히 고정돼 있던 것(`@anthropic-ai/claude-agent-sdk` · `electron-builder` · `node-pty` · `tar-stream` · `lucide-react` · `pretendard` · `@types/tar-stream`)은 `^` 범위로 바꿨다(lockfile 의 해석 버전은 그대로). 빌드는 설치된 정확한 버전을 스테이징에 적고, SDK ↔ 플랫폼 CLI 패키지는 설치된 SDK 버전으로 맞춘다(둘이 다르면 빌드 실패).

## 서명 · 공증

| 플랫폼 | 서명 | 그동안의 동작 |
|---|---|---|
| macOS | Developer ID 서명 + 공증 — **회사 Apple Developer 계정 확인 중(`미결`)** | 계정 전: ad-hoc 서명 dmg. 처음 설치 때 "확인되지 않은 개발자" 경고 → 시스템 설정 › 개인정보 보호 및 보안 › [그래도 열기] 안내(웹 내려받기 화면에 그림으로). **자동 설치 불가** — 업데이트는 "새 버전 알림 + 내려받기" |
| Windows | 코드 서명 인증서 — `미결`(없어도 동작) | 서명 없으면 설치 때 SmartScreen 경고 → [추가 정보] › [실행] 안내. 자동 업데이트는 동작 |
| Linux | 없음(AppImage 는 서명 불필요) | 내려받은 뒤 실행 권한(`chmod +x`) 안내. 자동 업데이트는 동작 |

- 서명 계정이 생기면 빌드 설정만 바꾼다(앱 동작은 아래 "업데이트 방식" 이 서명 여부를 보고 고른다).

## 업데이트 원본 (GitHub · 클라우드)

2026-11-09 — 공개 전환(ADR-045)으로 공개 배포는 GitHub Releases 가 맡고, euta cloud 서버의 배포 경로(아래 절)는 그 서버를 운영하는 조직의 **미러**로 남긴다. 상세 · 대안: [ADR-047](ADR_047_앱-업데이트-원본-선택.md).

| 원본 | 기준 주소 | 언제 |
|---|---|---|
| euta cloud 서버 | `https://<서버>/updates/<채널>/` | 앱에 클라우드 서버 주소가 있고, 그 서버의 `GET /api/config` 가 `appUpdates: true`(게시된 릴리스가 하나 이상)일 때 |
| GitHub Releases | `https://github.com/onedark23/euta-app/releases/latest/download/` | 그 밖 — 서버 주소가 없거나, 서버가 앱 배포를 하지 않을 때 |

- **확인할 때마다 다시 고른다** — 실행 중에 클라우드에 연결 · 해제하면 다음 확인부터 원본이 바뀐다. 앱 설정의 업데이트 카드에 지금 원본("GitHub" 또는 서버 주소)을 보인다.
- 클라우드를 골랐는데 서버에 닿지 않으면(사무실 밖 등) GitHub 로 넘어가지 않고 그 확인을 실패로 둔다 — 조직이 올린 버전 · 최소 앱 버전(아래)과 어긋나지 않게 원본을 섞지 않는다. 서버 주소는 있는데 `/api/config` 를 못 읽어도 같다.
- 두 원본 모두 **electron-updater generic provider 형식**(기준 주소 + `latest*.yml` + 파일)으로 읽는다 — 앱 코드는 기준 주소만 다르다. GitHub 은 `releases/latest/download/<이름>` 이 가장 최근 정식 릴리스의 파일로 넘겨 주므로 GitHub API(비로그인 IP 당 시간당 60회 — 사무실은 IP 하나로 나감)를 쓰지 않는다.
- **GitHub 릴리스 규칙**: 태그 `app-v<버전>`, 노트 = `RELEASES.md` 그 버전 절, 파일 = 빌드 산출물 전부(설치 · 업데이트 · `latest*.yml` · blockmap). **prerelease 표시를 하지 않는다** — `releases/latest` 는 prerelease 를 건너뛴다. 채널은 버전 이름(`-alpha.N`)이 담고, GitHub 쪽 채널은 "가장 최근 게시본" 하나다(베타 · 정식이 함께 생기면 다시 정한다). 올리기 = 앱 레포 `pnpm release:github`(GitHub CLI).
- 올린 직후 `latest*.yml` 과 파일 사이에 다음 릴리스가 끼면 sha512 검증이 실패한다 — 조용히 다음 확인 때 다시(아래 "업데이트 방식").

## 업데이트 배포 (euta cloud 서버 — 미러)

- 파일 위치: 서버 결과 파일 볼륨(`euta-server-data`, 14Gi)의 `/data/updates/<채널>/`. 결과 파일 상한(`RESULT_MAX_TOTAL` 10Gi)과 따로 세고, **채널별 최근 3개 버전만** 남긴다(한 버전 ≈ 1.5GB, 4 빌드(맥 2 · 윈도 · 리눅스)). 네임스페이스 저장소 한도 증설 불필요(실제 사용 ≈ 50MB, 2026-10-30 확인).
- 공개 경로(로그인 없이 — 앱이 로그인 전에도 업데이트를 받아야 함, 사내망 전용):
  - `GET /updates/<채널>/latest-mac.yml` · `latest.yml` · 파일 — electron-updater generic provider 형식 그대로
  - `GET /api/app/releases?channel=` — 버전 · 날짜 · 릴리스 노트 · 플랫폼별 설치 파일 주소(웹 화면 · 서명 없는 맥 앱이 씀)
- **GitHub 에서 자동으로 가져오기**(미러, 2026-11-10, [ADR-048](ADR_048_앱-릴리스-자동화.md)): 서버 설정 `APP_MIRROR_GITHUB=<owner/repo>`(비면 끔)이면 서버가 시작 1분 뒤 · 10분마다(`APP_MIRROR_INTERVAL`) GitHub 최신 릴리스를 본다.
  - 최신 태그: `https://github.com/<repo>/releases/latest` 가 넘겨 주는 주소에서 태그를 읽는다(GitHub API 아님). 그 태그의 `latest-mac.yml` · `latest.yml` · `latest-linux.yml`(없는 플랫폼은 건너뜀)의 버전이 서버에 없으면(초안 · 게시 모두) 가져온다.
  - 파일: 각 yml 의 `files` 와 그 `.blockmap`, yml 자체를 `releases/download/<태그>/<이름>` 에서 받아 아래 "올리기" 와 같은 저장 · 검사(이름 규칙 · 1GiB · sha512 — yml 에 적힌 값과 다르면 실패)를 거쳐 초안에 넣고, 다 받으면 게시한다. 노트 = yml `releaseNotes`. 하나라도 실패하면 초안을 지우고 다음 주기에 다시.
  - 관리자 웹 "릴리스 관리" 에 마지막 가져오기(시각 · 결과 · 버전)와 [지금 가져오기](`POST /api/admin/app-releases/mirror`)를 보인다. 한 번에 하나만 돈다.
  - GitHub 에 공개된 것만 받는다(사내에만 먼저 배포는 수동 올리기로).
  - 그 채널의 게시본보다 낮은 버전은 게시하지 않는다. 가져오는 도중 서버가 꺼져 남은 미러 초안은 45분 뒤 다음 실행이 지우고 다시 가져온다(관리자가 올린 초안은 그대로).
- 올리기: 관리자만(관리자 계정의 **읽기·쓰기** 개인 토큰, 또는 관리자 웹 세션) — 앱 레포의 `pnpm release:upload` 가 빌드 산출물을 올린다. 같은 버전 다시 올리기는 거절(버전을 올린다).
  - 순서: `POST /api/admin/app-releases {version, notes}`(초안) → 파일마다 `PUT …/<버전>/files/<이름>`(본문 = 파일, 1개 최대 1GiB, 스트리밍 저장 · sha512) → `POST …/<버전>/publish`(`latest*.yml` 이 하나 이상 있어야 함). 초안은 `DELETE …/<버전>` 으로 지운다. 목록은 `GET /api/admin/app-releases`(초안 포함).
  - 게시 = DB 행 하나를 바꾸는 것 — `/updates/<채널>/<이름>` 은 "그 이름의 파일을 가진 가장 최근 게시본" 에서 나가므로 모든 `latest*.yml` 이 한 번에 바뀐다. 한 버전에 빠진 플랫폼은 직전 게시본 것이 계속 나간다. 게시하면서 채널별 4번째 이후 게시본은 행 · 파일을 지운다.
  - 파일 이름: `[A-Za-z0-9._-]` 만, 확장자 `.dmg` · `.zip` · `.exe` · `.AppImage` · `.blockmap` · `.yml`. 플랫폼 · arch 는 이름에서 정하므로 앱 빌드 이름에 공백 없이 arch 를 넣는다(예 `euta-0.1.0-alpha.1-mac-arm64.dmg`).
  - `*.yml` 은 캐시하지 않음(`no-store`). 설치 · 업데이트 파일은 단일 Range(206) 지원, 여러 구간 Range 는 미지원 → 앱 generic provider `useMultipleRangeRequest: false`.
  - 큰 본문은 이 경로만 받는 두 번째 Ingress(`proxy-body-size: 1g`, 요청 버퍼링 끔, 같은 IP 허용 목록). 상세: [ADR-041](ADR_041_앱-업데이트-배포-클라우드-구현.md).
- 웹 "앱 내려받기" 화면(`/download`, 로그인 사용자, 머리 메뉴): 내 OS 에 맞는 설치 파일을 먼저 크게 보이고, 다른 플랫폼 · 지난 버전(남은 3개) · 릴리스 노트 · 첫 설치 경고 넘기는 법(내 OS 먼저, 맥 · 윈도 · 리눅스 바꿔 보기). 브라우저는 맥의 Apple 칩/Intel 을 대개 알 수 없어 맥이면 두 설치 파일을 함께 보이고 "이 Mac에 관하여" 로 확인하는 법을 적는다. 관리자에게는 "릴리스 관리"(초안 포함 목록 · 게시 · 삭제)가 더 보인다 — 올리기는 CLI 로만. 경고 안내는 그림 대신 단계 글로 시작한다(실제 화면 캡처는 첫 배포 때 추가).
- 이름 해석 · 접속 범위: 정식 이름 `https://<euta cloud>` 는 **외부 DNS** 에 공용 ingress 공인 IP 로 등록(2026-10-30). 서버 ingress 가 사무실 공인 IP 만 허용하면(ARCH 접속 제한) 앱 업데이트 · 내려받기도 **사무실(또는 사무실 IP 로 나가는 VPN)에서만** 된다. 예전 hosts 줄(ingress 내부 주소)이 남은 PC 는 403 이므로 배포 공지에 "hosts 줄 지우기" 를 넣는다.

## 업데이트 방식 (앱)

- 확인 시점: 앱 시작 30초 뒤 + 6시간마다 + 설정 › [업데이트 확인] 버튼. 채널은 앱 버전의 prerelease 이름을 따른다(지금은 알파 하나 — 채널 고르기 화면은 베타가 생길 때).
- 서명 여부에 따라:

| 앱 상태 | 동작 |
|---|---|
| Windows · Linux(AppImage), 또는 Developer ID 서명된 macOS | electron-updater 로 **뒤에서 내려받기** → 끝나면 머리줄에 "새 버전 x.y.z 준비됨 — [다시 시작해서 업데이트]" + [바뀐 것]. 누르기 전엔 설치 안 함. 앱을 끌 때 자동 설치(다음 실행부터 새 버전) |
| ad-hoc 서명 macOS | 원본의 `latest-mac.yml` 로 새 버전 확인 → "새 버전 x.y.z — [내려받기]"(브라우저로 그 원본의 dmg — `latest-mac.yml` 파일 목록에서 이 기기 arch, Apple 칩에 arm64 가 없으면 Intel) + [바뀐 것]. 설치는 사람이 |

- 서명 여부는 빌드가 앱 `package.json` 의 `eutaBuild.macSignature`(`adhoc` | `developer-id`)에 적고 앱이 읽는다. Linux 를 AppImage 가 아닌 방식으로 실행하면 ad-hoc 맥과 같은 "알림 + 내려받기". 개발 실행(패키징 안 된 앱)은 확인하지 않는다.
- 알림은 모든 화면 위 머리줄(닫기 = 이번 실행 동안 그 버전 숨김). 내려받는 동안은 알리지 않고 준비되면 알린다. [바뀐 것] = 그 버전의 릴리스 노트(두 경로 모두 원본의 `latest*.yml` `releaseNotes`).
- **실행 중 보호**: 이 PC 실행 · 녹화 · AI 대화가 진행 중이면 [다시 시작해서 업데이트] 를 누를 때 "진행 중인 작업이 끊깁니다"(실행 n개 · 시연 녹화 · AI 대화 n개) 확인 — [끊고 업데이트] · [끝나면 업데이트] · [취소]. **[끝나면 업데이트] 는 작업이 다 끝나면 알림을 다시 띄울 뿐 저절로 다시 시작하지 않는다**(앱을 끌 때는 설치됨).
- 업데이트 중 실패(내려받기 · 검증) → 조용히 다음 확인 때 다시. 설정 화면에 마지막 확인 시각 · 결과.
- 사용자 데이터(userData · 프로젝트 · 금고)는 업데이트로 바뀌지 않는다. 데이터 형식을 바꾸는 버전은 첫 실행 때 이관하고, 이관 전 사본을 남긴다.

## 최소 앱 버전 (클라우드 호환)

- 서버 설정 `MIN_APP_VERSION`(ConfigMap, 비면 검사 안 함). 서버는 앱 요청 헤더 `X-Euta-App-Version` 을 보고, 없거나 미만이면 `426` + `{ code: 'APP_TOO_OLD', minVersion }` 로 거절한다(웹 · MCP · 러너는 해당 없음). 비교는 SemVer(`0.1.0-alpha.2` < `0.1.0-alpha.10` < `0.1.0`).
  - 대상 = 앱 장치 토큰(`Authorization: Bearer` 로 보낸 `euta_` 아닌 세션 토큰) 요청. SSE 는 연결할 때 검사. 로그인(`/api/auth/*`)과 `/api/app/*`(릴리스 목록 · `GET /api/app/compat` → `{ minVersion }`)은 검사하지 않는다 — 옛 앱도 새 버전을 찾아야 하므로.
- 앱: `APP_TOO_OLD` 를 받으면 클라우드 프로젝트를 "업데이트 필요" 로 잠근다 — 보내기 대기분은 지우지 않고 보관, 로컬 프로젝트 · 이 PC 실행은 그대로. 잠금 화면에 [업데이트] 버튼(위 방식 — 준비됐으면 다시 시작, 알림 경로면 내려받기, 아직 모르면 지금 확인). 연결 상태 `app_too_old`(사이드바 칩·프로젝트 배지 "업데이트 필요"), 재시도하지 않고 다음 실행(새 버전)까지 유지. 요청 · SSE 연결 · 보내기 · 알림 스트림 어디서 426 을 받아도 같다.
- 서버 API 가 옛 앱과 맞지 않게 바뀌는 배포는 **같은 배포에서 `MIN_APP_VERSION` 을 올린다**(배포 절차 체크 항목).

## 보안

- 업데이트 파일은 electron-updater 가 `latest*.yml` 의 sha512 로 검증한다. 서명된 맥 · 윈도는 OS 서명 검사도 거친다.
- 올리기는 관리자 토큰만. 공개 경로는 읽기 전용, 사내망에서만 열린다(ingress 그대로).
- AI 채팅: 앱은 지금처럼 공식 `claude auth login/status/logout` 만 실행하고 자격증명을 읽지 않는다. 배포판에 다른 사람의 구독이나 공용 계정을 넣지 않는다.

## 완료 기준 (검증)

- 단위: 버전 비교 · 채널 선택 · 서명 여부 판정 · **원본 고르기(서버 없음 → GitHub · `appUpdates` 참 → 클라우드 · 거짓 → GitHub · 서버에 닿지 않음 → 실패, 넘어가지 않음)** · 알림 경로의 `latest-mac.yml` 해석 · `APP_TOO_OLD` 처리 · 서버 올리기(관리자만, 같은 버전 거절, 최근 3개 유지) · `/api/config` `appUpdates`.
- 서버 · 웹 E2E: 릴리스 올리기 → `/updates/<채널>/latest*.yml` · 파일 · 내려받기 화면 · 지난 버전 정리.
- 앱 E2E(가짜 업데이트 서버): 새 버전 감지 → 알림 → (서명 없는 맥) 내려받기 링크 / (업데이터 경로) 준비됨 → 실행 중 확인 대화상자. 최소 버전 미만 → 클라우드 잠금 · 로컬 정상.
- 실제 1회: `0.1.0-alpha.1` 설치 → `0.1.0-alpha.2` 올리기 → 앱이 감지 → 업데이트 후 데이터 유지(맥 arm64 이 PC). Windows · Linux 는 실제 기기에서 설치·실행·업데이트 1회(설치 파일은 이쪽이 만들고, 실기 확인은 사용자 PC).
- 화면 캡처 확인(UI 그라운드룰).

## Change Log

### 2026-11-10 — 릴리스 자동화: 태그 `app-v*` → GitHub Actions 빌드(macOS 러너 = 맥 · Windows, Ubuntu 러너 = Linux) · 게시, 서버 미러는 `APP_MIRROR_GITHUB` 로 GitHub 최신 릴리스를 10분마다 가져와 게시(+ 관리자 [지금 가져오기]) — [ADR-048](ADR_048_앱-릴리스-자동화.md)

### 2026-11-09 — 업데이트 원본 두 곳(GitHub Releases 공개 기본 · euta cloud 서버 미러)과 고르는 규칙(`/api/config` `appUpdates`, 확인마다, 넘어가지 않음) · 두 원본 모두 generic 형식(GitHub `releases/latest/download/`, prerelease 표시 안 함) · 서명 없는 맥 알림도 `latest-mac.yml` 로 — [ADR-047](ADR_047_앱-업데이트-원본-선택.md)

### 2026-10-30 — 앱 구현 반영: 머리줄 알림 · 실행 중 확인(끊고/끝나면 — 끝나면은 다시 알림만) · 서명 여부는 빌드가 `eutaBuild` 에 기록 · 426 → `app_too_old` 잠금 · 의존성 `^` 범위 · 파일 이름 규칙 · 윈도 NSIS 는 Wine 없이(맥 호스트) — [ADR-042](ADR_042_앱-배포-업데이트-앱-구현.md)

### 2026-10-30 — 클라우드 구현 반영: 게시 = DB 전환(가장 최근 게시본에서 파일) · 관리자 읽기·쓰기 토큰 · 파일 이름 규칙 · 여러 구간 Range 미지원 · 426 대상(장치 토큰) · 웹 `/download` 맥 두 칩 표시 — [ADR-041](ADR_041_앱-업데이트-배포-클라우드-구현.md)

### 2026-10-30 — 대상에 Linux x64(AppImage) 추가(사용자) · node-pty 는 Linux 용으로 Docker 에서 컴파일 · Windows PC 있음

### 2026-10-30 — DNS 는 외부 DNS(공인 IP) · 사무실 IP 에서만 업데이트

### 2026-10-30 — 첫 버전 `0.1.0-alpha.1` · 알파 채널만 · DNS 는 IT 정식 요청 · 세 플랫폼 이 맥에서 교차 빌드(Windows 실기 확인 1회 필수)

### 2026-10-30 — 초안 (사내 전체 배포 · AI 채팅 그대로 · euta cloud 업데이트 배포 · 최소 앱 버전)
