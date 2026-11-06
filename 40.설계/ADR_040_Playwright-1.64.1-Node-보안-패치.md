---
title: ADR - 040 Playwright 1.64.1 업그레이드 · Node 보안 패치 (CVE-2026-111497)
path:  euta-workspace / 40.설계 / ADR - 040 Playwright-1.64.1-Node-보안-패치
ref: ADR-019(클라우드 이미지) · ADR-001(엔진 구성) · ADR-010(앱 구성) · Node.js October 2026 security releases
---

# ADR - 040 Playwright 1.64.1 업그레이드 · Node 보안 패치

## 상태

채택 (2026-10-27, 설계). 엔진 · 앱 · 클라우드에서 구현·검증한다.

## 맥락

- Node.js 2026-10-21 보안 릴리스가 CVE-2026-111497(내장 Brotli 디코더 범위 밖 쓰기 — 공격자 Brotli 데이터를 `node:zlib` 로 풀면 크래시 · 메모리 노출 · 원격 코드 실행, Node 등급 High) 외 9건을 고쳤다. 고친 버전 22.23.4 · 24.22.1 · 26.12.1, 우회 방법 없음. 출처: [nodejs.org — October 2026 security releases](https://nodejs.org/en/blog/vulnerability/october-2026-security-releases).
- dev 클러스터(배포 `1b4f448-e809bb26`)에서 실제 확인(2026-10-27): 서버 Node **22.23.3**(`node:22-bookworm-slim`, 2026-10-21 빌드) · 러너 Node **24.20.0**(`mcr.microsoft.com/playwright:v1.63.0-noble`, 이미지 빌드 2026-09-04). 둘 다 영향 범위.
- 노출 경로: 서버는 Teams 웹훅 호출에 Node 내장 `fetch` 를 쓰고 웹훅 URL 은 사용자가 넣는다. 러너는 임의 사이트를 대상으로 엔진을 돌린다.
- 러너 이미지는 엔진 Playwright npm 버전과 짝이 맞아야 한다(이미지의 `/ms-playwright` 브라우저 빌드 = npm 버전이 기대하는 빌드).

## 결정

### 1. 서버 — Dockerfile 그대로, 재빌드

- `node:22-bookworm-slim` 메이저 태그를 유지한다. 2026-10-27 기준 이 태그 digest 가 `node:22.23.4-bookworm-slim` 과 같아 재빌드만으로 22.23.4 가 들어간다.
- 정확한 패치 버전으로 고정하지 않는다(사용자 결정 2026-10-27) — 다음 보안 릴리스도 재빌드로 받는다.
- 그래서 이미지 빌드는 항상 `--pull` 로 한다(`k8s/dev/README.md` 빌드 명령 · `stage-image.sh --build`). 없으면 로컬에 캐시된 옛 태그가 쓰인다 — 2026-10-28 확인: `--pull` 없이 빌드하니 22.23.3, 붙이니 22.23.4.

### 2. 러너 — Playwright 1.63.0 → 1.64.1 (엔진 · 앱 · 클라우드 함께)

- `mcr.microsoft.com/playwright:v1.64.1-noble`(이미지 빌드 2026-10-23)의 Node 가 **24.22.1** 임을 이미지 실행으로 확인했다. 브라우저 Chromium 1248 · Firefox 1555 · WebKit 2370.
- Playwright 이미지 태그는 npm 버전과 짝이라 원래 정확한 버전으로 적어 왔다(이번에도 1.63.0 → 1.64.1 교체이지 새 고정이 아니다).
- 같은 브라우저 빌드를 쓰도록 엔진(`core` · `input-os`) · 앱(`playwright` · `@playwright/test`) · 클라우드 웹 E2E(`@playwright/test`)를 함께 1.64.1 로 올린다.
- 1.64.1 은 버그 수정만. 1.64.0 의 동작 변경 중 엔진에 닿는 것: 기기 설정(`devices[…]`)이 `screen` 크기도 넘김(엔진 `--device`) · 숨겨진 iframe 안 요소를 숨김으로 판정(동작 전 가시성 확인). 엔진·앱 테스트로 확인한다.

### 3. 로컬 앱 — Electron 44.4.5 → 44.10.0 (2026-10-28)

- Electron 44.4.5 는 Node 24.21.0 을 품어 같은 CVE 영향 범위. Electron 이 2026-10-22 `44-x-y` 에 Node 24.22.1 을 머지했고(electron/electron #55162) **44.10.0(2026-10-27)** 이 그것을 담은 첫 44 안정판이다(Chromium 152.0.7977.169).
- 44 줄 안에서 올린다 — 같은 메이저라 네이티브 모듈 ABI(node-pty N-API prebuild · robotjs)가 그대로고, 44.5~44.10 은 기능 추가·버그 수정만이다. 45 로 올릴 이유는 없다.
- `electron` 은 정확한 버전으로 고정하지 않고 범위 `^44.10.0` 으로 적는다(사용자 결정 2026-10-28) — 44 줄 안 패치·마이너는 따라가고, 설치 재현은 `pnpm-lock.yaml` 이 맡는다. 패키징(`scripts/package.mjs`)은 설치된 `node_modules/electron` 버전을 읽어 그대로 쓴다.
- 기다리는 동안의 노출 판정(2026-10-27): Electron Node 에서 외부 Brotli 를 푸는 경로가 없었다 — 메인 프로세스 `fetch` 는 euta cloud 서버로만(`cloud/client.ts`), 엔진의 대상 사이트 통신은 Playwright 브라우저가 디코딩(HAR 본문 포함), AI 채팅은 Agent SDK 의 네이티브 `claude` 바이너리.

### 4. Playwright 도 범위로 — 이미지 태그는 빌드 때 설치 버전에서 (2026-10-28)

- 사용자 지시(버전 정확히 고정하지 않는다)를 Playwright 에도 적용한다. `package.json` 은 범위 `^1.64.1`(엔진 `core` · `input-os` peer·dev, 앱 `playwright` · `@playwright/test`, 클라우드 웹 `@playwright/test`), 실제 설치는 각 레포 `pnpm-lock.yaml`.
- 러너 이미지 태그와 브라우저 빌드가 짝이라는 제약은 소스에 버전을 적지 않고 빌드 때 푼다: `packages/runner/scripts/stage-image.sh` 가 엔진에 **설치된** `playwright` 버전을 읽어 ① `.docker/playwright-version` 에 쓰고 ② 이미지 안 `npm install` 에 `overrides.playwright` 로 같은 버전을 준다. Dockerfile 은 `ARG PLAYWRIGHT_VERSION` → `FROM mcr.microsoft.com/playwright:v${PLAYWRIGHT_VERSION}-noble`, 빌드 명령이 `--build-arg PLAYWRIGHT_VERSION=$(cat …/playwright-version)` 을 넘긴다. 서버 staging 도 같은 `overrides`(브라우저는 받지 않지만 라이브러리를 엔진과 맞춤).
- 앱 패키징(`scripts/package.mjs`)도 staging 의 `playwright` 를 `package.json` 범위가 아니라 앱에 **설치된** 버전으로 넣고 `overrides` 로 엔진 tarball 의 의존까지 맞춘다 — 사용자 캐시의 브라우저 빌드와 어긋나지 않게(Electron 버전을 읽는 방식과 같음).

### 고려한 대안

- 러너 이미지는 1.63.0 그대로 두고 `node:24-bookworm-slim` 의 node 바이너리를 덮어쓰기 — 공식 이미지 구성을 임의로 바꾸는 우회라 버림(사용자 2026-10-27).
- 서버 이미지를 `node:22.23.4-…` 로 고정 — 다음 패치를 놓치기 쉬워 버림(사용자 2026-10-27).
- 러너를 `node` 이미지 + `playwright install --with-deps` 로 직접 구성 — 이미지 구성·크기·빌드 시간이 바뀌는 큰 변경. 공식 이미지가 이미 고친 Node 를 담아 불필요.

## 결과

구현 (2026-10-27). 1.64.0 동작 변경으로 고친 코드는 없다 — 기존 테스트가 그대로 통과했다.

| 레포 | 변경 | 검증 | 커밋 |
|---|---|---|---|
| euta-engine | `core` · `input-os` playwright 1.64.1 | 타입체크 · 전체 테스트 251건 | `6d7d9c1` |
| euta-app | `playwright` · `@playwright/test` 1.64.1 | 타입체크 · 단위 196건 · Electron E2E 8건 | `00d6d24` |
| euta-app (2026-10-28) | `electron` 44.4.5 → `^44.10.0`(설치 44.10.0) — `process.versions` electron 44.10.0 · node 24.22.1 · chrome 152.0.7977.169 | 타입체크 · 단위 196건 · Electron E2E 8건(클라우드 연동 2건 포함) | `77e2e05` |
| euta-cloud | 러너 `FROM …playwright:v1.64.1-noble` · 웹 `@playwright/test` 1.64.1 · README. 서버 Dockerfile 변경 없음 | 타입체크 · 테스트 322건 · 웹 E2E 20건 | `f3dafe6` |

- **Playwright 범위화 (2026-10-28)**: 엔진 `34da342` · 앱 `3d2ad3b` · 클라우드 `887d866`. 엔진 단위 108건 · 앱 타입체크·단위 196건 · 클라우드 타입체크·테스트 322건. 앱 패키징 staging 의 playwright · playwright-core 모두 1.64.1. 러너 이미지를 `--build-arg PLAYWRIGHT_VERSION=$(cat .docker/playwright-version)`(=1.64.1)로 빌드 → Node v24.22.1 · 엔진 playwright 1.64.1 · `/ms-playwright` chromium-1248, 스모크 통과. 배포 이미지 내용은 같아 재배포하지 않았다.
- **앱 설치 (2026-10-28)**: `^44.10.0` 범위로 바꿈(euta-app `93a3100`) → `pnpm package` → 번들의 Electron Framework 44.10.0 · Node v24.22.1 확인 → 기존 `/Applications/euta.app`(44.4.5)을 옮겨 두고 새 빌드 설치, `codesign --verify` 통과, 실행 시 창(1440×920) 표시.
- 이미지(linux/amd64, 태그 `f3dafe6`): 서버 `node -v` = **v22.23.4**, 러너 = **v24.22.1** · 엔진 playwright 1.64.1.
- **dev 배포 (2026-10-27)**: 사내 이미지 저장소 push → `k8s/dev` `newTag` `f3dafe6`(euta-cloud 커밋 `cf5222a`) → `kubectl apply -k`(바뀐 것은 두 Deployment 이미지뿐, DB 변경 없음). 실행 중 pod 에서 서버 `node -v` v22.23.4 · 러너 v24.22.1, 러너 `runner started`, 공개 주소 `/healthz` · `/readyz` · `/api/config` · 웹 200.
- 러너 이미지에서 엔진 CLI 로 공개 사이트 스모크(`goto` + 텍스트 `expect`, `--cloud`) 통과. 서버 이미지를 compose PostgreSQL 의 빈 DB 에 붙여 마이그레이션 · `/healthz` · `/readyz` · `/api/config` · 웹 200.

## 남은 일

- 없음.
