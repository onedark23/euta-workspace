---
title: ADR - 011 터미널 패널 · AI CLI 호스팅(P3) · pty 환경 정책
path:  euta-workspace / 40.설계 / ADR - 011 터미널-패널-AI-CLI-호스팅
ref: ARCH 전문 모드(P3) · RESEARCH_AI-구독-연동-방식 · CLAUDE.md AI 연동 금지선 · ADR-008 · ADR-009
---

# ADR - 011 터미널 패널 · AI CLI 호스팅(P3) · pty 환경 정책

## 상태

채택 (2026-09-29, 4단계 구현). macOS 에서 셸 실행을 E2E 로 검증. `claude`/`codex` 실행은 정책상 테스트에서 띄우지 않았다(탐지·실행 경로만 단위 테스트).

**2026-10-02 갱신**: PLAN 결정 "전문 모드 연동" 변경에 따라 AI 탭의 기본은 앱 자체 채팅([ADR-015](ADR_015_앱-AI-채팅-Agent-SDK.md))이 되고, 이 터미널 패널(+ 연결 안내 · hook 활동 패널)은 AI 탭의 **"고급: 터미널"** 옵션으로 옮겼다. 동작·env 계약(§3)은 바뀌지 않았다.

## 맥락

- P3: 앱 터미널 패널이 사용자가 설치한 **수정하지 않은** `claude`/`codex` 를 pty 로 그대로 띄운다. 로그인은 CLI 안에서 사용자가 한다(RESEARCH: Anthropic 법무 문서 — 바이너리 무수정, 자격증명 수집·저장·중계 금지).
- CLI 안에서 도는 euta 플러그인(ADR-009)·MCP 서버(ADR-008)가 앱으로 이벤트를 보내려면 앱 엔드포인트와 토큰을 알아야 한다. 경로는 **환경변수 상속**뿐이다.
- Finder/Dock 에서 띄운 macOS 앱은 최소 PATH(`/usr/bin:/bin:…`)만 받아 `~/.local/bin/claude`, Homebrew·nvm 의 `node` 를 못 찾는다. 플러그인의 MCP 서버·hook 은 `node` 로 실행된다.
- node-pty 1.1.0: N-API prebuild(darwin/win32 × x64/arm64), Windows 는 ConPTY. 자식 env 에 `TERM`(= `name` 옵션)과 `PWD`(= `cwd`)를 직접 넣는다(`lib/unixTerminal.js`).

## 결정

### 1. 라이브러리

- **node-pty 1.1.0**(Microsoft 공식, N-API) — Electron ABI 와 무관해 `@electron/rebuild` 불필요. `spawn-helper` 실행 비트는 postinstall 로 복구(ADR-010).
- **@xterm/xterm 6 + @xterm/addon-fit** — DOM 렌더러(기본). 탭을 오가도 터미널이 유지되도록 패널을 언마운트하지 않는다.
- 데이터 흐름: 렌더러 `term.onData` → IPC `term:write` → `pty.write`; `pty.onData` → IPC `term:data` → `term.write`. resize 는 FitAddon + ResizeObserver → `pty.resize`(범위 clamp).

### 2. 무엇을 띄우나

| 버튼 | 실행 |
|---|---|
| Shell | `$SHELL -l`(없으면 macOS `/bin/zsh`, Linux `/bin/bash`), Windows 는 `pwsh` → `%ComSpec%`/`powershell.exe` |
| claude / codex | PATH 에서 찾은 **절대 경로 그대로, 인자 없이**. Windows 에서 npm shim(`.cmd`/`.bat`)이면 `cmd.exe /d /s /c <경로>` 로만 감싼다(ConPTY 가 `.cmd` 를 직접 못 띄움 — CLI 자체는 무수정) |

- 탐지: `findExecutable(name)` — PATH 의 절대 디렉터리만, 실행 권한 확인(Windows 는 `PATHEXT`). 없으면 버튼을 끄고 설치 안내를 보인다(`curl -fsSL https://claude.ai/install.sh | bash` / `npm i -g @anthropic-ai/claude-code`, `npm i -g @openai/codex` / `brew install --cask codex`).
- 작업 폴더: 홈 디렉터리.

### 3. 환경변수 정책 (orchestrator 계약)

- 자식 env = **앱 프로세스 env 의 사본 + 아래 5개만 설정**(같은 이름이 이미 있으면 덮어씀). 그 밖에는 추가·삭제·변경하지 않는다.

| 변수 | 값 | 소비자 |
|---|---|---|
| `EUTA_APP_URL` | `http://127.0.0.1:<port>` | 플러그인 hook → `/hooks/claude` |
| `EUTA_APP_TOKEN` | 실행마다 새 토큰 | 〃 |
| `EUTA_APP_EVENTS_URL` | `${EUTA_APP_URL}/events` (명시적으로 설정 — 플러그인은 하나에서 다른 하나를 유도하지 않음) | MCP 서버 이벤트 전달(ADR-008 §5) |
| `EUTA_APP_EVENTS_TOKEN` | 같은 토큰 | 〃 |
| `EUTA_RUNS_DIR` | 앱의 runs 폴더 | MCP 서버 — 실행 결과가 앱 History 에 쌓이도록 |

- **인증 관련 변수는 절대 설정하지 않는다**: `ANTHROPIC_*` · `CLAUDE_CODE_OAUTH_TOKEN` · `CLAUDE_CODE_USE_*` · `OPENAI_*` · `CODEX_API_KEY` 등. 사용자가 원래 가진 값은 **그대로 통과**한다("상속" — 제거도 변경도 하지 않음). 단위 테스트가 "추가된 키 = 정확히 5개", "인증 패턴 키 = 입력에 있던 것뿐, 값 동일" 을 확인한다.
- node-pty 가 스스로 넣는 `TERM=xterm-256color`(터미널 종류) · `PWD`(작업 폴더)는 인증과 무관한 터미널 규약이다 — "앱이 추가하는 변수" 목록에서는 이 둘을 node-pty 동작으로 분리해 기록한다.
- 앱은 환경변수 **값을 로그에 남기지 않는다**.

### 4. PATH 보정 (앱 자신의 프로세스)

- macOS/Linux 시작 시 한 번, 사용자의 로그인 셸에 PATH 만 묻는다: `$SHELL -l -c 'printf "%s" "__EUTA_PATH__$PATH__EUTA_PATH__"'`(실패 시 `-i -l -c`), 4초 타임아웃, 표식 사이 값만 사용, 로그 없음. 결과를 기존 PATH 앞에 병합해 `process.env.PATH` 를 바꾼다. 터미널에서 앱을 띄운 것과 같은 PATH 를 자식이 물려받게 하는 것이고, pty env 계약(§3)의 "base" 는 이 보정된 앱 env 다.
- CLI 탐지·spawn 은 보정이 끝난 뒤(`await pathFixed`) 한다. Windows 는 GUI 앱도 사용자 PATH 를 받으므로 하지 않는다.

### 5. 하지 않는 것 (체크리스트)

- CLI 바이너리 수정·래핑·패치·인자 주입 — 없음.
- `~/.claude*`, `~/.codex`, 키체인, `setup-token` 값, OAuth 토큰, CLI 설정/인증 파일 — 읽지도 쓰지도 않는다(코드에 해당 경로가 없음). CLI 탐지는 PATH 디렉터리의 실행 파일 존재만 본다.
- "Claude/ChatGPT 로 로그인" UI — 없음. 앱은 CLI 가 무엇을 출력하든 해석하지 않고 터미널에 그대로 그린다.
- hook 응답으로 결정 반환 — 없음(ADR-012, 항상 빈 204).

## 이유

- 환경변수 상속은 CLI 를 건드리지 않고 플러그인·MCP 서버에 연결 정보를 건네는 유일한 방법이고, 목록을 5개로 고정하면 정책 준수를 테스트로 증명할 수 있다.
- 절대 경로로 무인자 실행하면 "unmodified binary" 조건이 명확하고, 사용자가 셸에서 `claude` 를 치는 것과 같다.
- 로그인 셸 PATH 보정은 fix-path 류의 일반적인 Electron 관행이며, 없으면 Finder 실행에서 플러그인의 `node` 가 없어 MCP·hook 이 조용히 실패한다.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| `@homebridge/node-pty-prebuilt-multiarch` · `@lydell/node-pty` | 공식 node-pty 가 1.x 부터 N-API prebuild 를 제공해 포크가 필요 없음 |
| 셸을 띄우고 `claude` 를 타이핑해 넣기 | 사용자 rc 파일·별칭에 따라 동작이 달라지고, 앱이 입력을 대신 치는 것은 "CLI 무수정" 설명을 흐린다 |
| pty env 에서 사용자 인증 변수를 제거 | 사용자가 의도한 설정(API 키 경로 등)을 앱이 바꾸게 됨. 금지선은 "앱이 설정하지 않는다" |
| PATH 를 pty env 에만 덮어쓰기 | "추가만 한다" 계약이 흐려짐. 앱 프로세스 자체를 보정하면 CLI 탐지와 자식이 같은 PATH 를 본다 |
| Codex app-server / Agent SDK 로 자체 채팅 UI | (당시) RESEARCH P4/P6 — 1단계 범위 밖. 2026-10-02 RESEARCH 판정 보정(본인 사용) 후 Agent SDK 채팅을 ADR-015 로 채택 |

## 결과

- 코드: `euta-app/src/main/terminal-env.ts`(`EUTA_ENV_KEYS` · `buildPtyEnv` · `findExecutable` · `spawnSpecFor` · `fixProcessPath`) · `pty-manager.ts` · 렌더러 `tabs/Terminal.tsx` · `tabs/AiTab.tsx`(연결 안내: 플러그인 설치, `claude mcp add euta -e EUTA_RUNS_DIR=<runs> -- node <engine>/packages/mcp-server/dist/index.js`, `codex mcp add …`).
- 테스트: `test/unit/terminal-env.test.ts`(5개 추가·인증 변수 무설정·사용자 값 보존·오래된 EUTA_* 교체·PATH 탐지·Windows PATHEXT/`.cmd`) · E2E 에서 셸 `echo eu""ta-pty-ok` 출력과 `EUTA_APP_EVENTS_URL` 주입 확인(토큰은 출력하지 않음).
- 남은 일: Windows 실기(ConPTY 크기 변경, PowerShell 프로필), Codex hook 대응(ARCH: 추후), 터미널 여러 개(탭) — 지금은 한 번에 하나.
