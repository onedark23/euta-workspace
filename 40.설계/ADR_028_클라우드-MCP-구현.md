---
title: ADR - 028 클라우드 MCP 구현
path:  euta-workspace / 40.설계 / ADR - 028 클라우드-MCP-구현
ref: PLAN_euta-cloud 결정 "클라우드 AI"·"MCP 연결"·"권한"(2026-10-20) · SPEC_클라우드-MCP · SPEC_클라우드-계정-프로젝트 개인 토큰 · SPEC_클라우드-실행-예약 · ARCH_euta-cloud 원격 MCP 커넥터 · ADR-008 · ADR-009 · ADR-023
---

# ADR - 028 클라우드 MCP 구현

## 상태

채택 (2026-10-21, C5 구현). 실제 서버(PostgreSQL) 대상 SDK 클라이언트 스모크 11건 + 실제 러너·엔진(Chromium)·픽스처 사이트 E2E 1건, 실제 Claude Code 2.1.305 의 `claude mcp add` 연결(격리 설정 폴더)로 검증했다. 결과는 아래 "결과" 절.

## 맥락

- [SPEC_클라우드-MCP](SPEC_클라우드-MCP.md) 의 원격 MCP(`/mcp`, Streamable HTTP, 개인 토큰) 도구 14개 · 권한 판정 · 호출 상한 · 작업 로그 표기를 euta-cloud 서버에 붙이고, euta-claude-plugin 에 연결 방법을 넣는다.
- 서버 쪽 사실(euta-cloud `8853f52` 코드로 확인): 개인 토큰 인증(`Bearer euta_…` → Better Auth `verifyApiKey`, 토큰당 분당 120회 rate limit 이 api-key 플러그인에 이미 켜져 있음, 넘으면 `AppError(429)`), 권한 판정 `decide()`(토큰 권한 ∧ 역할, 비활성 계정 제외), 행위자 `actorOf()` = 토큰이면 `AI — <이름> (<토큰 이름>)`, 실행 요청은 토큰이면 출처 `ai_mcp`. 스크립트 저장 로직은 라우트 안에 있었고, 실행 요청·취소는 서비스(`planRun`·`createRun`·`cancelRun`)였다.
- 엔진의 공용 도구 핸들러(`@euta/mcp-server/tools` `createEutaToolHandlers`)는 로컬 실행 폴더·`SecretGuard`·세션 관리자·`ServerConfig` 에 묶여 있어 클라우드 프로젝트를 대상으로 쓸 수 없다.

## 결정

### 1. 전송 — MCP SDK v2 `createMcpHandler`, 무상태

- `@modelcontextprotocol/server` **2.1.0**(엔진과 같은 버전)의 `createMcpHandler(factory)` 를 Fastify 라우트 `GET·POST·DELETE /mcp` 에 붙인다. 라우트가 본문을 버퍼로 받아 웹 표준 `Request` 를 만들고 `handler.fetch()` 의 `Response` 를 그대로 흘린다(`/api/auth/*` 와 같은 방식 — `@modelcontextprotocol/node` 의존 추가 없음).
- **무상태(세션 없음)**: 요청마다 `McpServer` 를 새로 만들고, 그 요청에서 인증된 `Principal` 을 클로저로 묶는다. 토큰 폐기·만료·계정 비활성화·역할 변경이 다음 호출에 바로 반영되고, 서버 여러 대로 늘려도 세션 고정이 필요 없다. principal 은 `authInfo` 객체를 키로 한 `WeakMap` 으로 팩터리에 넘긴다(`authInfo` 는 팩터리까지 같은 객체로 통과 — SDK 문서). `authInfo.token` 에는 원문 대신 `[redacted]`.
- 프로토콜: SDK 기본 `legacy: 'stateless'` — 2025 계열(Claude Code · SDK 클라이언트가 지금 쓰는 `2025-11-25`)은 요청마다 Streamable HTTP 로 답하고(POST 응답은 요청 단위 SSE 프레임 — Streamable HTTP 의 일부이며 폐기 예정인 HTTP+SSE 전송이 아님), 2026-07-28 계열은 같은 팩터리로 JSON 응답. **`GET /mcp` 는 405**(서버 → 클라이언트 독립 스트림 없음, 사양상 허용). 응답 모드는 기본(`auto`) — 도구가 결과 전에 알림을 보내지 않으므로 사실상 JSON.
- 본문 상한 1 MiB(스크립트 상한 256 KiB 를 넉넉히 덮음).

### 2. 인증 — 개인 토큰만

| 경우 | 응답 |
|---|---|
| `Authorization` 없음 · `Bearer euta_…` 형식 아님(쿠키 세션 · 장치 토큰 · Basic 포함) | 401 + `WWW-Authenticate: Bearer realm="euta-cloud"` |
| 모르는 · 만료 · 폐기 토큰, 비활성 계정의 토큰 | 401 + 같은 헤더(기존 `authenticate()` 의 401 을 그대로) |
| 토큰당 분당 120회 초과 | **HTTP 429** + `Retry-After: 60` |

- 호출 상한은 api-key 플러그인 rate limit 을 그대로 쓴다 → **HTTP 요청 단위**(initialize · tools/list · notifications 포함, 도구 호출 1회 = 요청 1회). 같은 토큰으로 REST 를 부른 것도 같이 센다(토큰당 상한이므로).
- 도구 안에서 429 가 날 수는 없다(인증 단계에서 막힘). 도구 오류 문구에도 상한 안내를 둔다.

### 3. 도구 — 서버 서비스 위에 클라우드용으로 다시 정의

- 엔진 도구 핸들러를 import 하지 않는다(맥락 참고). 이름·입력 모양은 ADR-008 에 맞추고, 엔진에서는 `@euta/core` 의 `getJsonSchema` · `describe`(`describeStep`/`stepText`) · `cloudPrecheck`(사전 검사 안) · `parseYaml`/`flattenSteps`/`extractText` 만 쓴다. ARCH 의 "공용 도구 핸들러를 노출" 문구와 다른 점이다(ARCH Change Log 반영).
- 모든 도구가 REST 와 같은 서비스를 탄다 → 권한·피드·작업 로그·알림·하루 상한이 웹·앱·MCP 에서 같다. 이를 위해 서버를 조금 정리했다(동작 변경 없음, 기존 221건 통과로 확인):
  - `projectAccessFor(ctx, principal, …)` — `projectAccess(req)` 가 이것을 부른다.
  - `services/scripts.ts` `saveScript()` — PUT 라우트의 저장(기준 버전 충돌 409 + 현재 버전 · 검증 422 · 256 KiB · 200개 보관 · `script.created/saved` 기록)을 옮김. MCP 는 `origin: 'ai_mcp'` 고정.
  - `services/runs.ts` — `planRun` 의 스크립트별 사전 검사를 `checkScriptForRun` 으로 떼고 `precheckSource()`(저장 안 한 원문 + 환경)를 추가, 실행 목록 조회 `listRunRows()`.
  - `environmentList()` · `suiteList()` — 라우트 안 DTO 도우미를 모듈 수준으로(`projectDto`·`scheduleDtos` 와 같은 방식).

| 도구 | 권한 | 입력 · 동작 메모 |
|---|---|---|
| `list_projects` | 보기 | 휴지통 제외 전체(조직 구성원은 누구나 보기) + `myRole` · `effectiveRole`(읽기 토큰 = viewer) |
| `get_project` | 보기 | 설정 · `testRules` · 환경(이름·주소·변수·허용 도메인·기본·**테스트 계정 이름만**) · 스위트 · 오늘 실행 사용량 · `can`. **Teams webhook 주소는 역할과 관계없이 빼고** `teamsWebhookConfigured` 만 |
| `list_scripts` | 보기 | 경로 · 현재 버전 · 검증 · 마지막 변경 · **최근 실행**(스크립트별 `DISTINCT ON` 1건, 스위트 자식 포함) |
| `read_script` | 보기 | `{projectId, path}` → 원문 · `version`(저장 기준) · 검증 · 단계 문장(`"3.1. …"` 번호, 묶음 재귀) |
| `get_schema` | — | DSL JSON Schema |
| `validate_script` | 보기 | `source` 또는 `path` + `environment?`(이름·id) + `vars?` → `{valid, issues, cloudPrecheck{ok, problems, environment, credentialNames}, runnable}`. 저장 안 함 |
| `save_script` | 쓰기(`script.edit`) | `{path, source, baseVersion?, memo?}`. 충돌이면 오류 + `current`(버전·작성자·출처·**원문**), 검증 실패면 오류 + `issues`, 같은 내용이면 `unchanged` |
| `run_script` | 쓰기(`run.execute`) | `scriptPath` 또는 `suite`(이름·id) + `environment?` + `vars?` → 실행 id 즉시. 사전 검사 문제·하루 상한은 도구 오류 |
| `get_run` | 보기 | `{runId}` 만(프로젝트는 실행에서 찾음, 보이지 않는 프로젝트의 실행은 "찾을 수 없음"). 상태 · 현재 단계 · 단계별 결과 · 실패 · 실패한 expect · 지표 · 스위트 자식 · 결과 파일 목록(최대 300) |
| `list_runs` | 보기 | 대상 · 환경(이름·id) · 상태 · 출처 · `limit`(기본 20) |
| `get_screenshot` | 보기 | `stepId`(결과 폴더 `run.json` 의 단계 `screenshot`) 또는 `file` → MCP 이미지(8 MiB 상한) |
| `get_artifact` | 보기 | 텍스트 계열은 텍스트(기본 64,000자, `maxChars`), PDF 는 엔진 `extractText`, 이미지·바이너리는 메타만 |
| `cancel_run` | 쓰기(`run.execute`) | `cancelRun()` 그대로(대기 → 즉시 취소, 실행 중 → 정지 요청) |
| `list_schedules` | 보기 | `scheduleDtos()` 그대로 |

- 결과는 JSON 텍스트 1블록(+ 이미지·본문 블록). `readOnlyHint` 는 쓰기 3개만 false.
- 실행 id 의 엔진식 짧은 id(접미사) 찾기는 하지 않는다 — 서버 실행 id(UUID)만.
- 결과 파일: `resultFiles = expired` → **"파일 만료 — 보관 기간이 지나…"**(`code: FILES_EXPIRED`), `none` → "결과 파일이 아직 없습니다". 경로는 `safeRelPath` 로 검사(탈출 거절).

### 4. 오류 — 도구 오류(isError) + 한국어 문구

- 서버 `AppError` 를 도구 오류 `{error: <한국어 문구>, code, …}` 로 바꾼다. 403 은 읽기 토큰이면 "읽기 전용 토큰…", 역할이면 "보기 역할이라…" 로 나눠 말한다. 409 충돌은 "read_script 로 다시 읽고 baseVersion=N 으로" 를 안내. 사전 검사 문제는 코드별 한국어 접두(`SECRET_PLACEMENT` → "${secrets.*} 는 fill/type 의 값에만…") + 원래 문구. 하루 상한은 `사용/상한 (날짜, 서울 기준)`.
- 예상하지 못한 예외는 서버 로그에 남기고 "서버 오류" 만 돌려준다(내부 메시지 노출 없음).

### 5. 테스트 계정 값 · 비밀값

- 도구 결과의 재료는 이름만 담긴 DTO(환경 `credentialNames`, 실행 `credentialNames`)와 스크립트 원문(`${secrets.NAME}` 참조만)뿐이다. 결과 파일은 엔진 마스킹을 거친 것이고 서버는 이미 "결과 파일에 평문 0건"을 러너 E2E 로 검사한다.
- Teams webhook 주소(SAS `sig`)는 편집자에게도 MCP 로는 주지 않는다(설정 변경 도구가 없으니 필요 없음).
- 개인 토큰 원문은 로그(`authorization` 헤더 가림)·`authInfo`·도구 결과 어디에도 없다.

### 6. 플러그인 — `euta-cloud` 서버는 선택(직접 `claude mcp add`)

- SPEC 은 "`.mcp.json` 에 `euta-cloud`(http) 추가, 주소·토큰은 `userConfig`(토큰은 민감 값), 비워 두면 연결하지 않음(C5 에서 확인)" 이었다. **Claude Code 2.1.305 로 확인한 실제 동작**: `${user_config.cloud_url}` 이 비어 있으면 서버를 건너뛰지 않고 `claude mcp list`·`/mcp` 에 `✘ Failed to connect — URL is unset or invalid — open /plugin manage and configure euta options` 로 **항상 실패 표시**한다(플러그인의 다른 서버·skill 은 정상). 클라우드를 안 쓰는 모든 사용자에게 오류처럼 보이므로, 과제 지시("noisy 하면 설정 단계를 문서화하고 opt-in")대로 `.mcp.json`·`userConfig` 에는 넣지 않았다.
- 대신 README "euta cloud 연결" 절(토큰 발급 → `claude mcp add --transport http -s user euta-cloud …/mcp --header "Authorization: Bearer euta_…"` → `claude mcp list` ✔ Connected, 401 일 때 문구와 재등록)과 skill `cloud-test-script`(read_script → save_script 기준 버전 · 테스트 규칙 · 충돌 · 권한 오류 · 폴링 간격 · 변경 불가 항목)를 더했다. 도구 이름은 `mcp__euta-cloud__*`.
- 헤더 방식에서 토큰이 틀리면 Claude Code 는 OAuth 로 넘어가지 않고(`headers.Authorization` 이 있으면 OAuth fallback 끔) 401 본문을 그대로 보여 준다 — 401 의 `WWW-Authenticate` 가 OAuth 탐색을 일으키지 않음을 확인.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| 엔진 `createEutaToolHandlers` 재사용 | 로컬 실행 폴더·secrets 파일·세션에 묶임. 클라우드 대상으로 쓰려면 엔진을 크게 바꿔야 함 |
| MCP 도구가 `app.inject()` 로 REST 를 내부 호출 | 코드 중복은 없지만 내부 호출마다 토큰 재검증(rate limit 이중 계산)·principal 우회 장치가 필요. 서비스 함수 공유가 더 단순 |
| 상태 있는 세션(`Mcp-Session-Id`) | 폐기·비활성화 반영을 세션마다 다시 확인해야 하고, 서버 여러 대에서 세션 고정이 필요. 도구가 모두 요청-응답이라 이득 없음 |
| `responseMode: 'json'` 명시 | 기동 때 경고를 찍고, 알림을 안 보내는 지금은 `auto` 와 결과가 같음 |
| 도구 호출만 세는 별도 상한 | 이미 있는 api-key rate limit(토큰당)과 중복 구현. SPEC "호출" 을 HTTP 요청으로 해석해 문서화 |
| 플러그인 `.mcp.json` 에 `euta-cloud` + `userConfig` | 비우면 실패 표시(위 6). 필요하면 마켓플레이스에 **별도 opt-in 플러그인 `euta-cloud`**(필수·민감 `userConfig`)로 낼 수 있음 — 제품 판단으로 남김 |
| `get_run` 등에 `projectId` 필수 | 실행 id 로 프로젝트를 찾을 수 있고 AI 의 입력 실수를 줄임(권한 판정은 그대로) |

## 결과

- 서버 테스트 221 → **233**(+12): `packages/server/test/mcp.test.ts` 11건(SDK `Client` + `StreamableHTTPClientTransport`, 실제 소켓) — 도구 목록 = SPEC 14개·쓰기 3개 `readOnlyHint=false`·`GET` 405 · 401(헤더 없음·Basic·euta_ 아님·모르는 토큰·쿠키 세션·폐기·만료·비활성 계정, 모두 `WWW-Authenticate: Bearer`) · 읽기 도구(테스트 규칙·계정 이름만·webhook 주소 없음·단계 문장) · `validate_script`(통과·URL secret·없는 계정·검증 실패·저장 경로·모르는 환경, 저장 안 됨) · `save_script`(새로 만들기·기준 버전 저장·충돌 → 현재 버전·원문·새 경로에 기준 버전 → 충돌·검증 실패·같은 내용, 이력 출처 `ai_mcp`·작성자 토큰 주인, 작업 로그 `AI — editor (mcp-editor)`) · 읽기 토큰/보기 역할의 쓰기 거절 · `run_script`·`get_run`·`list_runs`·`list_scripts` 최근 실행·`cancel_run`·사전 검사 거절·하루 상한·작업 로그 표기 · 결과 파일(단계 스크린샷 이미지·파일 이미지·경로 탈출 거절·텍스트·잘림·만료) · `list_schedules` · rate limit(120회 200 → 429 ×5, `Retry-After`, 다른 토큰 영향 없음) · 모든 도구 결과·서버 로그에 테스트 계정 값·webhook sig·토큰 원문 0건.
- `packages/runner/test/mcp-e2e.test.ts` 1건: MCP 클라이언트 `save_script` → `run_script` → `get_run` 폴링 → **실제 러너 + 엔진 CLI(Chromium) + 픽스처 사이트** 로그인 5단계 통과(출처 `ai_mcp`, 요청자 `AI — editor (claude)`) → `get_screenshot`(PNG) · `get_artifact`(run.log) · `list_runs` → 결과·로그에 계정 값 0건.
- `pnpm -r typecheck` · build · `pnpm test`(25 파일 233건) · `pnpm test:e2e`(웹 2건) 통과.
- **실제 Claude Code 연결**(2.1.305, `CLAUDE_CONFIG_DIR` 를 임시 폴더로 — 사용자 `~/.claude` 무변경 확인): 실제 서버(테스트 DB) 에 `claude mcp add --transport http -s user euta-cloud http://127.0.0.1:47811/mcp --header "Authorization: Bearer euta_…"` → `claude mcp list` = `✔ Connected`, 틀린 토큰은 `✘ Failed to connect — Server rejected the configured Authorization header (HTTP 401)…`. 모델 호출(대화로 "고치고 실행")은 사용자 구독을 쓰므로 하지 않았다 — **사용자 확인 1건 남음**.
- 플러그인: `npm test` 12건 · `claude plugin validate`(플러그인·marketplace·skills) 통과, `plugin details` 가 skill 3개 인식. 위 6 의 비설정 동작은 같은 임시 설정 폴더에 플러그인 복사본을 설치해 확인.
- 엔진 변경 없음(ADR-024 추가 없음).
- 커밋: euta-cloud `32e33de`(서비스 정리) · `c38de23`(엔드포인트·도구) · `c5f5cc6`(테스트) · `b6edd96`(응답 모드) · `b6ef24f`(README), euta-claude-plugin `229943d`.

## 한계·후속

- Claude Desktop 연결(로컬 설정 + 중계 도구)은 확인하지 않았다 — SPEC "Desktop" 항목은 열린 채.
- 대화형 Claude Code 세션에서 "스크립트 고치고 실행" 1회는 사용자 확인으로 남김.
- 호출 상한이 HTTP 요청 단위라 연결 직후 initialize·tools/list 도 센다(분당 120회에는 여유).
- `get_run` 은 결과 파일 목록을 300개까지만 싣는다(초과 시 `filesTruncated`).
- 플러그인에서 주소·토큰을 설정 화면으로 받고 싶다면 별도 opt-in 플러그인으로(위 대안) — 제품 판단.
