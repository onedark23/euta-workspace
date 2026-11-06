---
title: ADR - 008 MCP 서버 도구 설계 · secrets 경계 · 앱 이벤트 전달
path:  euta-workspace / 40.설계 / ADR - 008 MCP-서버-도구-설계
ref: ARCH 전문 모드(P1) · ARCH 이벤트 모델 · ARCH 보안·개인정보 · ADR-006 · ADR-007
---

# ADR - 008 MCP 서버 도구 설계 · secrets 경계 · 앱 이벤트 전달

## 상태

채택 (2026-09-29, 2단계 구현). `euta-engine/packages/mcp-server`(`@euta/mcp-server`, bin `euta-mcp`, CLI 별칭 `euta mcp`).

## 맥락

- ARCH P1: MCP 서버는 **실행형**(검증·실행·결과 조회) 도구와 **탐색형**(AI 가 한 스텝씩 조작 → YAML 로 내보내기) 도구를 둔다. 둘 다 ADR-007 의 core 공개 API(`runScript` · `openSession/Session` · `RunIndex` · `validateScript` · `getJsonSchema`) 위에 얹는다.
- AI 는 secret 값을 절대 보지 않아야 한다(ARCH 보안). 스크립트·스텝은 `${secrets.NAME}` 참조만 쓰고, 값은 서버 쪽 설정에서만 온다.
- 외부 Claude Code/Codex 세션이 MCP 로 실행한 경우, 떠 있는 Electron 앱(4단계)이 같은 이벤트를 받아 라이브 뷰를 그려야 한다(ARCH 이벤트 모델 마지막 줄).
- stdio 전송이므로 stdout 은 JSON-RPC 전용이다.

## 결정

### 1. SDK · 전송

- **공식 TypeScript SDK v2 분리 패키지** `@modelcontextprotocol/server` 2.1(런타임) · `@modelcontextprotocol/client` 2.1(테스트 전용 dev 의존). `McpServer.registerTool(name, { inputSchema: z.object(...) }, handler)` + zod 4(core 와 같은 메이저).
- 전송은 `McpServer.connect(new StdioServerTransport())`. 2025-era `initialize`(`protocolVersion: 2025-06-18`, Claude Code·Codex 가 보내는 형태)로 원시 JSON-RPC 스모크를 통과함을 확인했다.
- 진행 알림은 `notifications/progress`(요청에 `progressToken` 이 있을 때). MCP 로깅 알림(`ctx.mcpReq.log`)은 2026-07-28 개정에서 deprecated 라 쓰지 않고, 서버 로그는 **stderr 전용**(`[euta-mcp] ...`). `console.log/info/debug` 는 stderr 로 돌리고, node:sqlite `ExperimentalWarning` 은 CLI 와 같은 방식으로 숨긴다.

### 2. 도구 목록과 형태

- **이름은 접두사 없이 snake_case.** 서버 이름 `euta` 가 네임스페이스 역할을 한다(Claude Code 에서는 `mcp__euta__run_script`). `euta_` 접두사는 중복이다.
- **결과는 `content` 의 JSON 텍스트(2칸 들여쓰기) 하나 + 필요 시 이미지/본문 텍스트.** `structuredContent` 는 쓰지 않는다 — 클라이언트가 text 와 structured 를 모두 모델에 넘기면 토큰이 두 배가 되고, outputSchema 를 도구마다 유지하는 비용 대비 이득이 작다.
- 사용자 오류(경로 없음·검증 실패·세션 없음)는 throw 하지 않고 `isError: true` + `{ error, issues? }` 로 돌려준다(모델이 고쳐서 다시 부를 수 있게).
- runId 를 받는 곳은 전체 id 또는 **short id**(run id 끝 6자리 hex, `RunIndex.findRun` 규칙)를 받는다. 세션 id = 세션의 run id.

| 도구 | 입력 | 출력 |
|---|---|---|
| `validate_script` | `path` 또는 `yaml`, `vars?`, `env?` | `{ valid, issues[{path,message}] }` |
| `run_script` | `path` 또는 `yaml`, `vars?`, `env?`, `headed?`(기본 headless), `browser?` | RunOverview: `{ runId, shortId, kind, name, status, summary, durationMs, startedAt, runDir, reportPath, steps[{stepId,action,name?,status,durationMs,error?,screenshot?}], failures[{stepId,action,status,error,failedExpects[]}], metrics[{name,value,unit,stepId?}] (이름·스텝별 마지막 값), validationIssues?, error? }` + 스텝마다 progress 알림(`"passed 3 click 로그인 버튼"`) |
| `list_runs` | `limit?`(20), `status?`, `name?` | `[{ id, shortId, kind, name, status, startedAt, durationMs, total, passed, failed, errors, skipped, dir }]` 최신순 |
| `get_run` | `runId` | RunOverview + `expects[{stepId,condition,expected,actual,passed}]`, `consoleErrors[]`, `networkFailures[]`(각 30개 상한), `artifacts[{path,size}]`(실행 폴더 기준 상대 경로). 열린 세션이면 `status: running` 과 현재까지의 스텝 |
| `get_screenshot` | `runId` + (`stepId` 또는 `file`) | 텍스트 `{ path, relativePath }` + **image**(png) |
| `get_artifact` | `runId`, `file`, `maxBytes?`(64,000) | 텍스트 `{ file, size, kind: text|pdf-text, truncated }` + 본문. PDF 는 텍스트 추출, 그 밖의 바이너리는 메타데이터만 |
| `get_schema` | — | JSON Schema(draft-07) |
| `session_open` | `baseUrl?`, `name?`, `description?`, `browser?{engine,device,viewport,hasTouch,locale,headless}`, `defaults?{timeout,settle}`, `record?{video,trace,har,screenshots}`, `vars?`, `env?` | `{ sessionId, shortId, runDir, headless, openSessions }`. **기본 headed**(사용자가 지켜볼 수 있게) |
| `session_step` | `sessionId`, `step`(DSL 스텝 객체 1개), `image?`(none·on-failure·always, 기본 on-failure) | `{ stepId, action, status, durationMs, error?, expects[], screenshot?, page{url,title} }` + 정책에 따라 image. 잘못된 스텝은 실행하지 않고 `isError` + issues |
| `session_snapshot` | `sessionId` | `url:` · `title:` 머리 + ARIA 스냅샷(YAML 텍스트, 마스킹) |
| `session_screenshot` | `sessionId`, `fullPage?`, `name?` | `{ path, relativePath }` + image |
| `session_export` | `sessionId`, `name?`, `includeFailed?`, `savePath?`, `overwrite?` | `{ steps, issues, savedTo? }` + YAML 본문. 통과한 스텝만, `${secrets.*}` 참조 유지, 기존 파일은 `overwrite: true` 없이는 덮어쓰지 않음 |
| `session_close` | `sessionId` | RunOverview(`run_script` 와 같은 형태) |

- **리소스**: `euta://schema/script.json`(JSON Schema) · `euta://docs/dsl-cheatsheet`(SPEC 요약, 영어 — AI 소비용) · `euta://examples/<file>.yaml`(워크스페이스의 `examples/` 가 있을 때).
- **동시 세션**: 세션은 서버 프로세스 안 `Map<runId, Session>` 으로 여러 개를 연다. 각 세션의 스텝은 core 가 직렬화한다.
- **종료**: stdin 종료(클라이언트 종료) · SIGINT · SIGTERM 에 모든 세션을 `close()`(trace·HAR 정리, run.json·report·index 기록) → 이벤트 전달 flush → `process.exit(0)`. 30초 안에 끝나지 않으면 강제 종료.

### 3. 설정 (서버 쪽에서만)

| 변수 / 인자 | 의미 | 기본 |
|---|---|---|
| `EUTA_RUNS_DIR` / `--runs-dir` | 실행 폴더·`index.sqlite` 위치 | `<cwd>/runs` |
| `EUTA_BASELINE_DIR` / `--baseline-dir` | 시각 비교 기준 이미지 폴더 | 스크립트 폴더 기준(SPEC) |
| `EUTA_SECRETS_FILE` / `--secrets` | secrets 파일(JSON 또는 .env, 레포 밖) | 없음 |
| `EUTA_RUN_HEADLESS` | `run_script` 기본 headless | `true` |
| `EUTA_SESSION_HEADLESS` | `session_open` 기본 headless | `false`(headed) |
| `EUTA_STEP_IMAGE` | `session_step` 이미지 정책 | `on-failure` |
| `EUTA_APP_EVENTS_URL` · `EUTA_APP_EVENTS_TOKEN` | 앱 이벤트 전달(아래) | 없음 = 끔 |

### 4. secrets 경계 (AI 는 값을 보지 않는다)

- 값의 출처는 **서버 설정뿐**: secrets 파일 → 같은 이름의 환경변수(SPEC 규칙 그대로). 도구 입력에는 secret 값을 넣는 칸이 없다.
- 마스킹은 두 겹이다.
  1. core(ADR-006): 실행·세션마다 Redactor, 이벤트 버스가 단일 관문. 스냅샷·RunResult·run.json·HAR·trace·스크린샷 입력란 마스킹.
  2. 서버 `SecretGuard`: **모든 도구 결과의 텍스트**를 한 번 더 통과시킨다. 설정된 secrets 전부 + 도구 입력(스텝·YAML·파일 내용)에서 `${secrets.X}` 로 참조된 환경변수 값을 등록한다. 이미지 `data`(base64)는 건드리지 않는다(우연한 base64 변형 일치로 PNG 가 깨지는 것을 막음 — 픽셀 마스킹은 core 담당).
- **`${env.X}` 차단 규칙**: SPEC 상 `${env.*}` 는 "그대로 기록" 이라 마스킹되지 않는다. 그래서 MCP 서버는 이름이 자격증명처럼 보이는 키(`SECRET|TOKEN|PASSWORD|PASSWD|PASSPHRASE|API_KEY|ACCESS_KEY|PRIVATE_KEY|CREDENTIAL|COOKIE|AUTH`) 또는 secrets 파일에 있는 키를 `${env.X}` 로 참조하면 **실행 전에 거부**한다("use `${secrets.X}` instead"). `validate_script` 도 같은 issue 를 낸다. `EUTA_APP_EVENTS_TOKEN` 은 스크립트 env 에서 아예 뺀다.
- 도구 설명(`description`)과 서버 `instructions` 에 이 정책을 적어 AI 가 secret 값을 요청하거나 스크립트에 넣지 않도록 안내한다.

### 5. 앱 이벤트 전달 계약 (4단계 수신부 구현용)

`EUTA_APP_EVENTS_URL` 이 있으면 서버가 연 모든 실행·세션의 core 이벤트를 앱으로 POST 한다. 없으면 아무것도 하지 않는다.

- **대상 제한**: `http(s)` + loopback 호스트(`127.0.0.1` · `localhost` · `::1`)만. 그 밖이면 경고 한 줄 후 전달을 끈다(ARCH: 앱 엔드포인트는 127.0.0.1 바인딩). 토큰이 원격으로 새는 설정 실수를 막는다.
- **요청**

```
POST <EUTA_APP_EVENTS_URL>
Content-Type: application/json
Authorization: Bearer <EUTA_APP_EVENTS_TOKEN>

{
  "source": "euta-mcp",
  "protocol": 1,
  "pid": 12345,                       // MCP 서버 프로세스 (여러 서버 구분용)
  "sentAt": "2026-09-29T01:02:03.456Z",
  "events": [ EutaEvent, ... ]        // ADR-007 봉투 그대로
}
```

- `EutaEvent` = `{ type, runId, seq, ts, ...payload }`(ADR-007 · `packages/core/src/events.ts` `EventPayloads`). 타입: `run.started`(kind: script|session · script · env · runDir) · `run.finished` · `step.started`(step 은 참조가 남은 원본) · `step.finished` · `artifact.saved`(path 는 runDir 기준 상대) · `metric.recorded` · `console.message` · `network.failed` · `expect.evaluated` · `log`.
- **마스킹**: core 이벤트 버스에서 이미 마스킹된 이벤트를 서버 SecretGuard 로 한 번 더 통과시켜 보낸다. 수신부는 secret 을 받지 않는다.
- **배치·순서**: 200ms 마다 또는 50개가 모이면 보낸다. `run.finished` 는 즉시 flush. 요청은 한 번에 하나만 보내므로 **서버 하나 안에서는 순서가 보장**된다(같은 runId 의 `seq` 는 증가). 한 배치에 여러 runId 가 섞일 수 있다. 세션·실행을 늦게 구독해도 처음부터 재생(replay)해 보낸다.
- **전달 보장**: at-most-once, 재시도 없음, 요청 타임아웃 2초. 실패는 치명적이지 않다 — 첫 실패에 stderr 경고 한 줄, 이후 실패는 조용히 버리고 다음 배치는 계속 시도한다(앱이 늦게 떠도 이후 이벤트부터 받음). 누락 복구는 수신부가 `runDir/steps.jsonl`(같은 스키마)을 읽어 하면 된다.
- **수신부 요구**: 2xx 를 빠르게 돌려준다(본문 무시). 토큰이 다르면 401. 종료 시 서버는 최대 3초 동안 남은 배치를 flush 한다.

## 이유

- core 가 이미 실행기·세션·이벤트·마스킹을 가지고 있어, MCP 서버는 얇은 어댑터로 두는 것이 동작 일치(탐색 → 내보내기 → 재실행 green)와 유지보수에 유리하다.
- v2 분리 패키지는 서버 런타임 의존이 `zod` + SDK core 뿐이라 가볍다(v1 `@modelcontextprotocol/sdk` 는 express·hono·cors 등 HTTP 의존을 끌고 온다).
- 결과를 JSON 텍스트 하나로 두면 모든 클라이언트(Claude Code·Codex)에서 같게 보이고 토큰이 예측 가능하다.
- `${env.*}` 차단은 "env 는 기록한다"는 SPEC 을 바꾸지 않으면서 AI 경로에서만 새는 틈을 막는다. 환경변수 전체를 Redactor 에 넣는 방식은 짧은 값(`localhost` 등)이 모든 출력에서 `***` 로 바뀌는 부작용이 커서 택하지 않았다.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| v1 `@modelcontextprotocol/sdk` 1.30 | 동작은 하지만 HTTP 서버 의존이 많고 v2 가 현재 권장 경로 |
| `serveStdio(factory)`(v2 era 자동 선택) | 1:1 stdio 에서는 `connect(StdioServerTransport)` 로 충분하고 2025-era 호환을 원시 스모크로 확인함. 2026-07-28 전용 클라이언트가 늘면 재검토 |
| `structuredContent` + outputSchema | 토큰 중복·스키마 유지 비용. 필요해지면 도구별로 추가 |
| 도구명 `euta_` 접두사 | 서버 이름이 이미 범위를 정함 |
| 민감 이름 환경변수를 모두 secrets 로 옮겨 마스킹 | 짧거나 흔한 값이 출력 전반에서 가려지는 부작용 |
| 이벤트 전달을 WebSocket/SSE 로 | 앱이 서버를 모르는 상태에서도 붙어야 하므로 서버 → 앱 단방향 POST 가 단순. 앱은 hook 수신과 같은 HTTP 엔드포인트를 재사용 가능 |
| 전달 실패 시 재시도·디스크 큐 | 라이브 뷰 용도라 늦은 이벤트 가치가 낮고, 정본은 `steps.jsonl` 에 있음 |

## 결과

- 등록: `claude mcp add euta -e EUTA_RUNS_DIR=<abs> -e EUTA_SECRETS_FILE=<abs> -- node <abs>/packages/mcp-server/dist/index.js`, Codex 는 `codex mcp add` 또는 `config.toml`(`packages/mcp-server/README.md`).
- 테스트: SDK 클라이언트로 빌드된 서버를 stdio 로 띄워 도구 목록 · 검증 issue · 예제 실행/조회 · 스크린샷 이미지 · 세션 왕복(export → validate → run_script green) · 모든 응답/진행 알림에 secret 0건 · 로컬 HTTP 수신기로 이벤트 전달(Bearer · 순서 · secret 0건) · stdin 종료 시 열린 세션 마무리와 프로세스 종료 · `euta mcp` 별칭을 확인한다.
- **남은 공백(알려진 한계)**
  - AI 는 secret 값을 보지 못하지만, `goto: https://외부/?q=${secrets.X}` 처럼 값을 **브라우저로 외부에 보내게** 할 수는 있다(프롬프트 인젝션 위험). secret 참조 위치 제한(예: `fill`/`type` 값만)은 core DSL 변경이라 보류.
  - 클라이언트가 stdin 을 닫은 뒤 약 4초 만에 SIGKILL 하면(SDK stdio 클라이언트의 기본 동작) 영상·trace 정리가 긴 세션은 마무리되지 못하고 run.json 이 `running` 으로 남는다. SIGTERM 까지는 정상 처리한다.
  - `get_artifact` 는 다운로드 원본(디스크에는 증거로서 마스킹 없이 저장)을 **설정된 secrets 와 이 서버 프로세스에서 참조된 secrets** 로만 가린다.
  - 도구 호출 취소(`notifications/cancelled`)가 실행을 중단하지 않는다 — core 에 중단 API 가 없다.
  - 스크린샷을 축소하지 않는다(1280x800 PNG 그대로). progress 알림에 `total` 이 없다(그룹·trigger 하위 스텝 수를 미리 세지 않음).
  - 세션은 한 페이지만 다룬다(ADR-007, 팝업·새 탭 미지원).
