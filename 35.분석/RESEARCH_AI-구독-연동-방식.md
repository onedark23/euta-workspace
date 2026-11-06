---
title: RESEARCH - AI 구독 기반 연동 방식
path:  euta-workspace / 35.분석 / RESEARCH - AI-구독-연동-방식
ref:
---

# RESEARCH - AI 구독 기반 연동 방식

> 질문: API 키가 아니라 사용자의 **구독**(Claude Pro/Max/Team/Enterprise · ChatGPT · Google)으로 AI 를 툴에 더 밀접하게 붙일 수 있는가?
> 조사일: 2026-09-28. 정책 문서는 자주 바뀐다 — 결정 전 재확인 필요.
> 이 문서는 조사 결과와 **제안**까지만 담는다. 방식 결정은 사용자와 상의 후 PLAN 에 기록한다.

## 결론

- **가능하다. 조건은 하나 — 로그인과 추론을 벤더의 공식 클라이언트(수정하지 않은 CLI·앱)가 하게 두는 것.** 우리 툴은 그 클라이언트를 품거나(호스팅) 그 클라이언트에 꽂히는(MCP·플러그인) 쪽에 선다.
- **금지선**: 우리 앱이 "Claude 로 로그인" 을 직접 제공하거나, 구독 토큰을 수집·저장·중계하거나, 토큰으로 벤더 백엔드를 직접 호출하는 것. Anthropic·Google 모두 원문으로 금지.
- **원래 원하던 "앱 안에서 대화하며 과정·결과를 본다" 는 구독으로 구현할 수 있다** — Electron 앱 안 터미널 패널에서 수정하지 않은 `claude`(또는 `codex`) CLI 를 그대로 띄우고, 우리 MCP 서버·hooks 로 툴과 앱 화면을 묶는 방식. Anthropic 법무 문서의 제품 탑재 조항("preinstalling or running Claude Code in your products or services")이 이 형태를 포괄한다고 읽힌다. 다만 조항의 예시는 "hosted sandboxes or other agent infrastructure"(서버 호스팅)라 데스크톱 앱 사례는 명시돼 있지 않다 — Anthropic 확인 권장. 조건은 Commercial Terms 동의 · 바이너리 무수정 · 토큰 무접촉.
- Agent SDK 로 우리만의 채팅 UI 를 만들면서 구독 로그인을 쓰는 것은 "사전승인" 없이는 불가 — 이 형태는 API 키로만.

## 판단 기준 — 정책 원문

### Anthropic (직접 대조)

출처: [Legal and compliance](https://code.claude.com/docs/en/legal-and-compliance)

- 금지: "Anthropic does not permit third-party developers to offer Claude.ai login into their own applications, or to route requests through Free, Pro, or Max plan credentials on behalf of their users. Moreover, developers may not collect, store, or intermediate Claude.ai credentials or session tokens — sign-in to a Claude account must complete through Anthropic's own flow."
- 허용 범위: "Nor does it prevent an end user from signing in to the unmodified Claude Code binary with their own Claude subscription, including where a platform hosts Claude Code as described under *Can customers offer Claude Code in their products?*"
- 제품 탑재 조건 ("Can customers offer Claude Code in their products?"): "preinstalling or running Claude Code in your products or services … requires agreeing to our Commercial Terms of Service and complying with the conditions below"
  - "The Claude Code binary must not be modified." — 내장 인증 방식을 제거·제한해서도 안 된다.
  - "Customers may not pay for, resell, or intermediate Claude usage on their end users' behalf. Each end user must authenticate with their own … Claude subscription plan credentials …"
- 사용량 전제: "Advertised usage limits for Pro and Max plans assume ordinary, individual usage of Claude Code and the Agent SDK."
- Agent SDK 기반 제품: "Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK." ([Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview))
- 과금 현황: Agent SDK·`claude -p`·서드파티 앱 사용량은 현재 구독 한도에서 차감 — 개편안은 2026-06-15 보류 ([support](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan), 조사 에이전트 인용)

### Google (직접 대조)

- "Directly accessing the services powering Gemini CLI (for example, the Gemini Code Assist service) using third-party software, tools, or services (for example, using OpenClaw with Gemini CLI OAuth) is a violation of applicable terms and policies. Such actions may be grounds for suspension or termination of your account." ([tos-privacy](https://github.com/google-gemini/gemini-cli/blob/main/docs/resources/tos-privacy.md))
- Gemini CLI 자체를 프로그램으로 구동하는 것(`--acp`, 헤드리스)은 문서화된 용도.

### OpenAI

- 제3자 앱의 ChatGPT 구독 사용에 대한 명시 정책 **없음** (허용·금지 모두 문구 없음). 이용약관 원문은 접근 실패(403) — 미확인.
- 대신 [Codex app-server](https://learn.chatgpt.com/docs/app-server) 가 "deep integration inside your own product: authentication, conversation history, approvals, and streamed agent events" 용도로 안내되고, 로그인 모드 `chatgpt`("Codex owns the ChatGPT OAuth flow") 를 제공 — 로그인을 Codex 가 소유하므로 Anthropic 기준의 "공식 흐름으로 로그인" 과 같은 구조.
- `codex mcp-server` 는 폐기 — "Use the Codex app server for existing integrations." ([codex-sdk](https://learn.chatgpt.com/docs/codex-sdk), 직접 대조)
- 엔터프라이즈 대상 신규 통합은 "contact OpenAI to get it added to a known clients list" (컴플라이언스 로그용).

## 연동 패턴 — 구독으로 가능한 것

통합 깊이: ★ 약함 ~ ★★★ 강함

| 패턴 | 동작 | 깊이 | Claude | Codex | Gemini |
|---|---|---|---|---|---|
| P1. MCP 서버 제공 | 사용자의 CLI·데스크톱 앱이 우리 MCP 서버에 연결해 툴을 조작 | ★ | 허용 | 허용 | 허용 |
| P2. 플러그인 + hooks | MCP·skills·명령·hooks 를 플러그인으로 묶음. hooks 가 대화·도구 진행을 앱 localhost 로 전송 → 앱이 실시간 표시 | ★★ | 허용 | (설정 공유 구조 있음, hooks 는 미조사) | (미조사) |
| P3. 앱 내 CLI 호스팅 | Electron 터미널 패널에서 수정하지 않은 CLI 를 그대로 실행. 사용자가 CLI 안에서 직접 로그인. P1·P2 와 결합해 대화·화면·결과를 한 창에 | ★★★ | 허용으로 읽힘 — 일반 조항이 포괄, 예시는 서버 호스팅이라 데스크톱 앱은 확인 권장 (Commercial Terms 동의 조건) | 허용 쪽 (명시 문구 없음) | 허용 |
| P4. 공식 프로토콜 호스팅 | 앱이 자체 채팅 UI 를 만들고 벤더 CLI 를 프로토콜로 구동 — Codex app-server(JSON-RPC) · Gemini `--acp` | ★★★ | 해당 없음 (Claude 는 Agent SDK 경로 → P6) | 불명확 (문서화된 용도, 정책 문구 없음) | 허용 |
| P5. 채팅 표면 UI (MCP Apps) | 테스트 결과 대시보드를 claude.ai·Claude Desktop·ChatGPT 대화 안에 렌더링 | ★★ | 허용 (전 플랜) | ChatGPT 는 공개 HTTPS 필요 | 미지원 |
| P6. Agent SDK 로 자체 채팅 UI + 구독 로그인 | 앱이 SDK 로 대화를 주도 | ★★★ | **금지** (사전승인 없이는) | — | — |
| P7. 헤드리스 자동 실행 | 앱이 `claude -p` · `codex exec` · `gemini -p` 를 배치 실행 (저장된 로그인 재사용) | ★★ | 불명확 — 추가로 `--bare` 가 `-p` 기본값이 될 예정이며 bare 는 OAuth 를 읽지 않음 | 불명확 (자동화는 API 키 권장) | 허용 |

공통 금지: CLI 자격증명 파일·키체인 읽기, `setup-token` 값 붙여넣기 받기, 토큰으로 백엔드 직접 호출.

### 보조 기능 (Claude)

- **Channels** (research preview) — MCP 서버가 실행 중인 세션에 이벤트를 밀어 넣음 (예: "테스트 실패"). 개인 플랜은 개발 플래그 필요, Team/Enterprise 는 관리자 허용목록으로 사용 가능. ([channels](https://code.claude.com/docs/en/channels), 조사 에이전트 인용)
- **Claude in Chrome** — 유료 플랜, 사용자의 브라우저 로그인 상태 공유. 외부 앱이 제어하는 API 는 없음 → 탐색 테스트 보조 수단 정도.

## 제안 (결정 아님)

- **1단계**: P1 + P2 + P3 조합.
  - 코어는 MCP 서버. Claude Code 용 플러그인(MCP + 테스트 작성·분석 skills + HTTP hooks)을 함께 배포.
  - Electron 앱에 터미널 패널을 두고 사용자가 설치·로그인한 `claude` / `codex` 를 그대로 띄운다. 앱의 나머지 영역은 브라우저 화면·스텝 진행·측정 결과를 hooks·MCP 이벤트로 실시간 표시.
  - 결과: 구독으로 과금, 앱 안에서 대화 + 과정·결과 확인. 로그인·토큰을 우리가 만지지 않음.
- **2단계 후보**: P4(Codex app-server · Gemini ACP 로 자체 채팅 UI), P5(MCP Apps 결과 위젯), API 키 사용자용 Agent SDK 채팅.
- **구독 경로의 한계**: "Advertised usage limits for Pro and Max plans assume ordinary, individual usage" — 구독은 **대화형·보조용**(사람이 대화하며 AI 가 돕는 흐름)에 맞다. 무인으로 모든 테스트의 모든 스텝을 AI 가 판정하는 대량 실행은 API 키 경로로 설계한다.
- **피할 것**: P6, 그리고 정책·기술 모두 불안정한 P7 을 핵심 경로로 삼는 것.

## 사례 — naby (leonardo204/naby, 2026-09-28 커밋 ddb4296)

로컬 퍼스트 개인 에이전트 데스크톱(Electron). 앱 자체 채팅 UI 에서 Claude 구독으로 대화한다. 코드 직접 확인(2026-10-02).

- **Claude 구독 = Agent SDK + 이 PC 의 Claude Code 로그인** (`src/engines/claude-agent-sdk-engine.ts`, `@anthropic-ai/claude-agent-sdk` 0.3.283)
  - Electron 메인에서 SDK `query()` 로 대화. API 키 없이, SDK 가 띄우는 CLI 가 기존 로그인을 그대로 쓴다.
  - 로그인·로그아웃·상태는 앱이 **실제 `claude auth login/status/logout` 을 실행**해 처리(`claude-login.ts`). 앱은 자격증명 파일·키체인을 열지 않는다고 명시.
  - 멀티 계정: 계정마다 빈 디렉터리를 만들어 `CLAUDE_CONFIG_DIR` 로 지정 → CLI 가 키체인·자격증명을 분리 저장(`claude-accounts.ts`, `specs/claude-multi-account.md`).
  - 격리: `settingSources: []` · `skills: []` · `strictMcpConfig: true` — 사용자의 `~/.claude` 설정·hook·MCP 가 섞이지 않게. 도구는 SDK 내장 MCP(in-process)로만 주고, `PreToolUse` hook 으로 모든 도구 호출을 게이트.
  - 사용 한도 표시: Claude Code 상태줄 캐시(`~/.claude/.hud_cache`)를 읽음(토큰 없음).
- **ChatGPT 구독 = 비공식 백엔드** (`src/providers/chatgpt-oauth.ts`): Codex 의 공개 client id 로 OAuth 를 앱이 직접 수행하고 `chatgpt.com/backend-api/codex` 를 호출. 코드 주석 스스로 "ToS grey zone — a dev convenience, never a product feature" 로 표기.
- **Gemini 구독 OAuth 는 하지 않음** — 약관이 정확히 그 재사용을 금지하기 때문이라고 명시(본 문서 Google 원문과 일치).
- **naby 가 스스로 둔 전제**: "single-user, in-house development tool".

### 정책 대조 (이 문서의 기존 판정 보정)

- 기존 판정 P6 "Agent SDK 자체 채팅 + 구독 로그인 = 금지" 는 **배포 시** 기준이었다. 원문 금지 대상은 "third-party developers ... offer Claude.ai login into their own applications, or ... route requests through Free, Pro, or Max plan credentials **on behalf of their users**".
- 반면 같은 문서는 "Advertised usage limits for Pro and Max plans assume ordinary, individual usage of Claude Code **and the Agent SDK**" 라고 하고, 지원 문서는 "Claude Agent SDK, `claude -p`, and third-party app usage still draw from your subscription's usage limits" (2026-06-15 개편 보류 공지)라고 한다 — **본인이 본인 구독으로 Agent SDK 를 쓰는 것은 예정된 사용 형태**로 읽힌다.
- 정리(해석): **본인 PC · 본인 구독 · 본인 사용** → naby 방식 가능. **다른 사람에게 배포해 그들의 구독 로그인을 앱에서 받게 하는 것** → 금지 대상. 동료·사내 배포는 Team/Enterprise 포함 Anthropic 확인 필요(기존 "벤더에 물어야 할 것" 유지).
- ChatGPT 비공식 백엔드 방식은 naby 도 회색지대로 표기 — euta 에는 채택하지 않는다(Codex 는 공식 app-server 경로가 있음).

## 벤더에 물어야 할 것

- **Anthropic**: 사내 도구가 Electron 앱 안에서 수정하지 않은 `claude` 를 띄우는 것도 "platform hosts Claude Code" 조건(Commercial Terms 동의)에 해당하는가 / 앱이 `claude -p` 를 자동 실행하는 것은 "offer claude.ai login" 인가 / 금지 문장이 Free·Pro·Max 만 명시한 것이 Team·Enterprise 좌석 사용을 허용한다는 뜻인가 / "previously approved" 신청 절차 — 창구: [contact sales](https://www.anthropic.com/contact-sales)
- **OpenAI**: 서드파티 앱이 app-server `chatgpt` 로그인 모드로 사용자 구독을 쓰는 것이 약관에 부합하는가 / 비엔터프라이즈도 known clients 등록이 필요한가
- **Google**: ACP 클라이언트 안에서 Google 로그인 흐름을 시작해도 되는가 / 상용 앱 번들 가능 여부

## 미확인

- OpenAI 이용약관 원문, ChatGPT 개발자 모드 요금제 범위 (403)
- Codex TypeScript SDK 자체 인증 방식, ChatGPT 데스크톱·Codex 의 MCP Apps 렌더링 여부
- Claude Code 데스크톱의 MCP Apps 렌더링 여부
- 2026-04 서드파티 하니스 과금 변경 보도 — Anthropic 1차 문서 없음

## 조사 방법

- 조사 에이전트 2개(Claude / OpenAI·Google) 병렬 수행 후 결정적 원문을 직접 대조했다 — Anthropic 법무 페이지 · Agent SDK overview · Gemini 약관 · codex-sdk 페이지.
- hooks 의 `http` 핸들러(이벤트 JSON 을 URL 로 POST)도 [hooks](https://code.claude.com/docs/en/hooks) 원문으로 확인했다.
- 그 외 인용(channels · MCP Apps · 커넥터 · app-server 세부 등)은 조사 에이전트가 수집한 것으로 직접 대조하지 않았다.

## Change Log

### 2026-10-30 — 결정 기록: 사용자 판단으로 앱 AI 채팅을 지금 방식 그대로 사내 배포(각자 자기 구독 로그인). 위 "벤더에 물어야 할 것" 은 확인하지 않은 채 남음 → PLAN "앱 배포"

### 2026-10-02 — naby 사례 추가, P6 판정을 배포/개인 사용으로 구분

### 2026-09-28 — 초기 조사
