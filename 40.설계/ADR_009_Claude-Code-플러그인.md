---
title: ADR - 009 Claude Code 플러그인 · 엔진 위치 · hook → 앱 전달
path:  euta-workspace / 40.설계 / ADR - 009 Claude-Code-플러그인
ref: ARCH 전문 모드(P2) · ARCH 보안·개인정보 · ADR-008
---

# ADR - 009 Claude Code 플러그인 · 엔진 위치 · hook → 앱 전달

## 상태

채택 (2026-09-29, 3단계 구현). 레포 `euta-claude-plugin/`: marketplace `euta-tools` + 플러그인 `euta`(`plugins/euta/`). Claude Code 2.1.283 에서 `claude plugin validate` 통과.

## 맥락

- ARCH P2: 플러그인은 엔진 MCP 서버 연결, 테스트 작성·결과 분석 skills, 앱으로 가는 hooks 를 담는다.
- 엔진은 **다른 레포**(`euta-engine`)에 있고 npm 배포 전이다. 플러그인은 엔진 위치를 알아내야 한다.
- 엔진 `loadConfig` 는 빈 문자열과 미설정을 다르게 다룬다: `EUTA_RUNS_DIR=""` → cwd 로 해석, `EUTA_STEP_IMAGE=""` → 예외. `.mcp.json` 의 `${VAR:-}` 나 비어 있는 `userConfig` 가 그대로 넘어가면 깨진다.
- Claude Code 문서(plugins-reference · hooks · mcp, 2026-09 기준)에서 확인한 제약:
  - `http` hook 의 `url` 은 환경변수 보간이 **없다**. 보간은 `headers` 에서 `allowedEnvVars` 로 허용한 변수만 된다. 앱 포트는 매번 바뀌므로 URL 을 고정할 수 없다.
  - `command` hook 은 exec form(`command` + `args`)이면 셸 없이 실행되고 `${CLAUDE_PLUGIN_ROOT}` 가 인자 하나로 치환된다. Windows 에서도 `node` + 스크립트 경로는 동작한다(`.cmd` shim 은 불가).
  - `async: true` command hook 은 백그라운드로 돌아 Claude 를 막지 않고 결정도 할 수 없다. `-p` 모드 종료 시에는 끊긴다.
  - `SessionEnd` hook 은 전체 1.5초 예산을 공유하고, **플러그인 hook 의 `timeout` 은 이 예산을 늘리지 못한다**.
  - `.mcp.json` 은 `${VAR}` · `${VAR:-default}` 확장, 플러그인 MCP 설정은 `${CLAUDE_PLUGIN_ROOT}` · `${user_config.KEY}` 치환을 지원한다. `claude plugin install` 은 `userConfig` 를 묻지 않는다(`--config KEY=VALUE` 또는 `/plugin configure`).
  - 로컬 폴더 marketplace 의 상대 경로 플러그인은 복사되지 않고 제자리에서 로드된다(`CLAUDE_PLUGIN_ROOT` = 소스 폴더).
  - 플러그인 MCP 도구 이름은 `mcp__plugin_<plugin>_<server>__<tool>` → `mcp__plugin_euta_euta__run_script`.

## 결정

### 1. 레포 구성

```
euta-claude-plugin/
  .claude-plugin/marketplace.json   name "euta-tools", plugins[0] = { name "euta", source "./plugins/euta" }
  plugins/euta/
    .claude-plugin/plugin.json      매니페스트 + userConfig(engine_dir · runs_dir · secrets_file, 모두 기본값 "")
    .mcp.json                       서버 "euta" → node scripts/mcp-launch.mjs
    hooks/hooks.json                11개 이벤트 → node scripts/hook-forward.mjs
    scripts/                        mcp-launch.mjs · launch-plan.mjs · hook-forward.mjs (Node 내장 모듈만, 의존성 0)
    skills/write-test-script · skills/analyze-test-run
  test/                             node:test
```

- marketplace 이름을 플러그인과 다르게(`euta-tools`) 해서 설치 id 가 `euta@euta-tools` 가 되게 했다. 테스트·README 는 플러그인 폴더 밖에 둬 `CLAUDE_PLUGIN_ROOT` 에 섞이지 않게 했다.
- 설치: `claude plugin marketplace add <abs>/euta-claude-plugin` → `claude plugin install euta@euta-tools [--config engine_dir=<abs>] [--config secrets_file=<abs>]`.

### 2. 엔진 위치 — 얇은 런처 + userConfig

`.mcp.json`:

```json
"euta": {
  "command": "node",
  "args": ["${CLAUDE_PLUGIN_ROOT}/scripts/mcp-launch.mjs",
           "--engine-dir", "${user_config.engine_dir}",
           "--runs-dir", "${user_config.runs_dir}",
           "--secrets", "${user_config.secrets_file}"],
  "env": { "EUTA_APP_EVENTS_URL": "${EUTA_APP_EVENTS_URL:-}",
           "EUTA_APP_EVENTS_TOKEN": "${EUTA_APP_EVENTS_TOKEN:-}" }
}
```

런처 규칙(빈 값과 치환 안 된 `${…}` 는 "없음"으로 본다):

| 항목 | 우선순위 |
|---|---|
| 엔진 폴더 | `engine_dir` 옵션 > `EUTA_ENGINE_DIR` > 플러그인 레포 옆 `../euta-engine` (개발 배치) — `packages/mcp-server/dist/main.js` 가 있는 첫 후보 |
| 실행 폴더 | `EUTA_RUNS_DIR` 환경변수(앱이 지정) > `runs_dir` 옵션 > `~/.euta/runs` |
| secrets | `EUTA_SECRETS_FILE` > `secrets_file` 옵션 > 없음 |

- 비어 있는 `EUTA_*` 환경변수는 지운 뒤 엔진 `main([--runs-dir, --secrets])` 를 **같은 프로세스에서** import 해 실행한다(신호·stdin 처리를 엔진이 그대로 가짐, 자식 프로세스 없음).
- 엔진을 못 찾으면 stderr 에 설정 방법을 한 줄 안내하고 종료 코드 1. stdout 에는 아무것도 쓰지 않는다(JSON-RPC 전용).
- 앱 이벤트 변수만 `env` 에 명시했다. 나머지 `EUTA_*` 는 부모 환경에서 그대로 상속된다 — `${X:-}` 로 명시하면 상속값을 빈 문자열로 덮을 수 있어서다.

### 3. hook → 앱 전달 — exec form command hook + 번들 Node 스크립트

- 계약(오케스트레이터 고정): 앱이 `claude` 자식 프로세스에 `EUTA_APP_URL`(`http://127.0.0.1:<port>`)과 `EUTA_APP_TOKEN` 을 준다. 플러그인은 `POST ${EUTA_APP_URL}/hooks/claude`, `Authorization: Bearer ${EUTA_APP_TOKEN}`, 본문 = hook 입력 JSON **바이트 그대로**.
- 이벤트: `SessionStart` · `UserPromptSubmit` · `PreToolUse` · `PostToolUse` · `PostToolUseFailure` · `Notification` · `SubagentStart` · `SubagentStop` · `Stop` · `StopFailure` · `SessionEnd`. matcher 생략 = 전부. `StopFailure` 는 API 오류로 끝난 턴을 앱이 "실행 중" 으로 남기지 않게, `SubagentStart` 는 `SubagentStop` 과 짝을 맞추려고 넣었다.
- 핸들러: `{ "type": "command", "command": "node", "args": ["${CLAUDE_PLUGIN_ROOT}/scripts/hook-forward.mjs"], "async": true }`. `SessionEnd` 만 동기(`timeout: 2`) — 종료 중인 부모에서 백그라운드 프로세스가 끊길 수 있어서다.
- `hook-forward.mjs` 동작
  1. `EUTA_APP_URL` 이 없거나, URL 이 아니거나, `http(s)` + loopback(`127.0.0.1` · `localhost` · `::1`)이 아니면 **stdin 도 읽지 않고** 즉시 종료 0.
  2. 시작 시각 기준 **1초 마감**(`SessionEnd` 1.5초 예산 안) — stdin 이 안 닫히거나 앱이 응답하지 않아도 마감에 종료 0.
  3. stdin 을 모아 그대로 POST(`Content-Type: application/json`). 토큰이 없으면 Authorization 헤더를 생략한다.
  4. 응답·오류와 무관하게 종료 코드 0, stdout·stderr 출력 없음. 따라서 Claude 의 동작(결정·컨텍스트)을 바꾸지 않는다.
- **수신부(4단계 앱)에 대한 요구**: 2xx 를 빨리 돌려준다. async hook 은 이벤트마다 별도 프로세스라 **도착 순서가 보장되지 않는다** — 정렬·짝 맞춤은 `session_id` + `tool_use_id`(Pre/PostToolUse), `agent_id`(Subagent*), 수신 시각으로 한다.

### 4. skills

- `write-test-script`: `session_open` → (`session_step` 1스텝씩 · `session_snapshot` 으로 role/name 확인 · `expect` 추가) → `session_export`(savePath) → `session_close` → `validate_script`. 자격증명은 `${secrets.NAME}` 만, 값은 채팅으로 받지 않고 secrets 파일에 넣게 안내, `${env.X}` 로 자격증명 금지. DSL 은 `euta://docs/dsl-cheatsheet` 와 `get_schema` 를 가리키고 본문에 복제하지 않는다.
- `analyze-test-run`: `validate_script` → `run_script`(또는 `list_runs`) → `get_run` → 실패 스텝마다 `get_screenshot`, 시각 비교 `diffs/*.diff.png`, 다운로드 `get_artifact` → 원인 분류(스크립트 drift · 타이밍 · 제품 버그 · 환경/데이터)와 성능 기준(LCP 2.5s · INP 200ms · CLS 0.1 · FCP 1.8s · TTFB 0.8s)으로 요약 보고.
- `description` 은 영어(핵심 용도 먼저), `when_to_use` 에 한국어·영어 트리거 문구. `allowed-tools` 로 읽기 전용 도구만 턴 단위 사전 승인(`get_schema` · `validate_script` · `session_snapshot` · `session_screenshot` / `list_runs` · `get_run` · `get_screenshot` · `get_artifact`). 브라우저를 움직이는 도구는 사용자 권한 설정에 맡긴다.
- MCP 서버가 다른 cwd 에서 돌 수 있으므로 파일 경로는 절대 경로로 넘기라고 명시했다.
- slash command 는 두지 않았다 — skill 이 `/euta:write-test-script` 로도 호출되므로 추가 가치가 없다.

## 이유

- 런처는 엔진의 빈 문자열 처리 차이, `userConfig` 미설정 시 치환 결과(문서에 없음), 개발 배치의 무설정 기동을 한곳에서 해결한다. 순수 선언형 `.mcp.json` 은 `runs_dir` 이 비면 cwd 에 결과를 쓰고, `engine_dir` 없이는 기동하지 않는다.
- hook URL 은 동적 포트라 `http` hook 을 쓸 수 없다. command hook 의 비용(이벤트마다 `node` 기동 수십 ms)은 `async` 로 Claude 경로에서 빠진다. 앱 밖 사용자는 이 비용조차 체감하지 않는다.
- 1초 마감은 가장 빡빡한 `SessionEnd` 예산(1.5초, 플러그인이 못 늘림)에 맞춘 값이다.
- loopback 제한은 ADR-008 의 이벤트 전달과 같은 규칙이다 — 토큰이 원격으로 새는 설정 실수를 막는다.
- 실행 폴더 기본값을 `~/.euta/runs` 로 둔 것은 프로젝트 폴더를 더럽히지 않고, `CLAUDE_PLUGIN_DATA`(제거 시 삭제됨)에 증거를 두지 않기 위해서다.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| `http` hook 으로 직접 POST | `url` 에 환경변수 보간이 없어 동적 포트를 담을 수 없다 |
| 셸 form hook(`[ -n "$EUTA_APP_URL" ] && curl …`) | Windows(Git Bash 없음 → PowerShell)에서 깨지고 curl 존재를 가정한다 |
| 모든 hook 동기 실행 + 짧은 timeout | 도착 순서는 좋아지지만 앱 밖 사용자도 도구 호출마다 `node` 기동을 두 번 기다린다 |
| `.mcp.json` 에서 엔진 `dist/index.js` 직접 실행 | `userConfig` 가 비면 기동 실패·cwd 에 결과 기록·`EUTA_STEP_IMAGE=""` 예외 |
| PATH 의 `euta-mcp` 바이너리 | private pnpm 워크스페이스라 전역 link 가 번거롭고 Windows shim(`.cmd`) 문제 |
| 엔진을 플러그인에 번들(.mcpb 등) | 엔진이 Playwright·네이티브 의존을 가져 크기가 크고, 배포 방식은 아직 미정(ARCH: 추후 ADR) |
| 실행 폴더 기본값 `<cwd>/runs`(엔진 기본) | 사용자가 `claude` 를 띄운 프로젝트마다 `runs/` 가 생긴다 |
| slash command 추가 | skill 이 이미 `/euta:<skill>` 로 호출 가능 |

## 결과

- 검증: `claude plugin validate plugins/euta` · `claude plugin validate .`(marketplace) · `claude plugin validate plugins/euta/skills` 통과. `claude --plugin-dir plugins/euta plugin details euta` 가 skills 2 · hooks 11 · MCP 서버 1 을 인식.
- 테스트(`npm test`, node:test 12건): hook 전달기 — `EUTA_APP_URL` 없음(즉시·무출력, stdin 을 기다리지 않음) · loopback 수신기(본문 바이트 동일·Bearer·경로 `/hooks/claude`) · 끝 `/` 와 `localhost` · 닫힌 포트 · 응답 없는 서버(1초 마감) · 닫히지 않는 stdin · 비 loopback/비 http URL 미전송. MCP — `.mcp.json` 을 Claude Code 방식으로 확장해 실행 → `initialize`(2025-06-18) + `tools/list` = 13개, `runs_dir` 옵션/`EUTA_RUNS_DIR` 반영(`index.sqlite` 생성), stdout 은 JSON-RPC 만, `userConfig` 미치환 시 sibling 엔진 탐색, 엔진 없음 → 종료 1 + 안내.
- **남은 공백(알려진 한계)**
  - `${user_config.KEY}` 가 비설정일 때 실제로 무엇으로 치환되는지는 문서에 없다. 기본값 `""` 를 두고 런처가 빈 값·`${…}` 를 모두 "없음"으로 처리해 양쪽을 대비했지만, 실제 Claude Code 세션에서의 기동은 이 단계에서 확인하지 않았다(설치·대화형 세션 금지).
  - `${EUTA_APP_EVENTS_URL:-}` 의 빈 기본값 확장도 문서 문법(`${VAR:-default}`)에 따른 것이며 실제 세션으로는 확인하지 않았다. 확장되지 않아도 엔진이 "잘못된 URL → 전달 끔" 으로 안전하게 처리한다.
  - 앱 계약은 `EUTA_APP_URL`/`EUTA_APP_TOKEN`(hook)이고, MCP 서버 이벤트 전달은 `EUTA_APP_EVENTS_URL`/`EUTA_APP_EVENTS_TOKEN` 을 읽는다. 앱(4단계)이 두 쌍을 모두 설정해야 실행 이벤트도 앱에 간다(플러그인은 URL 경로를 추측해 만들지 않는다).
  - async hook 은 도착 순서가 보장되지 않고, `claude -p` 종료 시 끝나지 않은 전달은 끊긴다(라이브 뷰 용도라 허용).
  - hook 은 이벤트마다 `node` 를 띄운다(앱 밖에서는 즉시 종료). Codex 용 hook 은 범위 밖(ARCH: 추후).
  - 엔진 개선 제안(엔진 미수정): `loadConfig` 가 `""` 를 미설정으로 취급하면 런처의 환경변수 정리가 필요 없어진다.

### C5 원격 MCP 추가 (2026-10-21, [ADR-028](ADR_028_클라우드-MCP-구현.md) §6)

- skill `cloud-test-script` 추가(클라우드 프로젝트: `read_script` → `save_script` 기준 버전 · 테스트 규칙 · 충돌·권한 오류 · `get_run` 폴링). README 에 "euta cloud 연결" 절(`claude mcp add --transport http -s user euta-cloud …/mcp --header "Authorization: Bearer euta_…"`).
- **`.mcp.json`·`userConfig` 에는 `euta-cloud` 를 넣지 않았다.** Claude Code 2.1.305 확인: `${user_config.X}` 가 비어 있는 http 서버는 건너뛰지 않고 `/mcp` 에 "✘ Failed to connect — URL is unset or invalid" 로 남는다. 위 "남은 공백" 의 "비설정 `user_config` 치환" 질문에 대한 실측 답이기도 하다 — stdio 서버 `euta` 는 빈 인자로 기동된다(런처가 빈 값 = 없음으로 처리하는 지금 설계가 맞음).
- 검증: `npm test` 12건 · `claude plugin validate`(플러그인·marketplace·skills) 통과, `plugin details` skill 3개. 커밋 euta-claude-plugin `229943d`.
