---
title: ARCH - End-User 테스트 툴 아키텍처
path:  euta-workspace / 40.설계 / ARCH - end-user-테스트툴
ref:
---

# ARCH - End-User 테스트 툴 아키텍처

> 확정 결정은 [PLAN 결정 사항](../20.프로젝트관리/PLAN_end-user-테스트툴-종합계획.md#결정-사항) 이 정본. 이 문서는 그 결정을 구성 요소·프로세스·이벤트로 풀어낸다.
> 라이브러리 선택 등 구현 세부는 `40.설계/ADR_*` 로 기록한다.

## 한눈에 보기

```
 ┌──────────────────────────────── euta-app (Electron) ────────────────────────────────┐
 │ 탭: Run · Live · History · AI · Settings                                             │
 │                                                                                      │
 │  ┌─ AI 채팅 (기본) ───────────────┐   ┌─ Live ─────────────┐  ┌─ History ──────────┐ │
 │  │ 사용자 요청 → Agent SDK query() │   │ 스텝 타임라인      │  │ runs + index.sqlite│ │
 │  │ (본인 Claude 구독, SDK 번들 CLI)│   │ 스크린샷 · 측정값  │  │ report.html        │ │
 │  │ 도구: euta 핸들러(in-process)   │   └─────────▲──────────┘  └─────────▲──────────┘ │
 │  └───────────────┬────────────────┘             │ 이벤트(EventBus)      │            │
 │                  │ 도구 호출(게이트 통과분만)    │                       │            │
 │  ┌───────────────▼───────────────────────────────┴───────────────────────┴─────────┐  │
 │  │                     euta-engine core (메인 프로세스에 내장)                      │  │
 │  └─────────────────────────────────────────────────────────────────────────────────┘  │
 │  고급: 터미널(pty, 사용자 claude/codex) · hook 활동 ◀── 127.0.0.1 로컬 서버(/events, /hooks/claude) │
 └──────────────────────────────────────────────────────────────────────────────────────┘

 euta-engine (GUI 없음)                                   보조 경로 (외부 Claude Code / Codex)
 ┌────────────────────────────────────────────────┐       ┌──────────────────────────────────┐
 │ core      DSL · 실행 엔진 · 입력 드라이버 ·     │       │ euta-claude-plugin               │
 │           측정/분석 · 저장 · 이벤트 버스        │◀──────│  .mcp.json → euta-mcp            │
 │ input-os  OS 입력 어댑터(선택, robotjs)         │  MCP  │  skills · hooks → 앱 로컬 서버   │
 │ cli       euta run / validate / runs / report   │       │ codex mcp add (수동 등록)        │
 │ mcp-server stdio MCP 서버 + 공용 도구 핸들러    │       └──────────────────────────────────┘
 └────────────────────────────────────────────────┘
```

## 레포와 책임

| 레포 | 패키지 | 책임 |
|---|---|---|
| `euta-engine` | `core` | YAML DSL 파싱·검증, 실행 엔진, 입력 드라이버 인터페이스, 측정·분석, 결과 저장, 이벤트 버스, 리포트 생성 ([ADR-007](ADR_007_실행-API-세션-이벤트.md)). **시연 녹화기**(`openRecorder` — headed 브라우저 + 페이지 수집 스크립트, 대상 후보·유일성, 민감 값 페이지 안 폐기, 체크포인트, 가린 스크린샷 → `recording.json`)와 **결정적 초안 변환**(`recordingToScript`) ([ADR-017](ADR_017_시연-녹화기-구현.md)). TC 시연: `tcRef` · `markStep` → `stepMarks`, `recordingToScript({ tcSteps })` 가 TC 단계마다 `group`, 체크포인트 없는 기대 결과는 `manual` ([ADR-039](ADR_039_TC-문서-변환-앱-구현.md), 단계 16 구현). **단계 문장 생성기**(`describeStep`, 한국어 — 의존성 없는 `@euta/core/describe` 하위 경로로 앱 렌더러도 사용), **주석 보존 다시 쓰기**(`rewriteScript` — 바뀌지 않은 키·단계는 원문 그대로), 실행 기록 `run.json.scriptHash`(실행한 원문의 sha256) ([ADR-018](ADR_018_스크립트-편집기-이력-구현.md)). **단계별 DOM 스냅샷**(`record.dom` → 결과 폴더 `dom/`)과 **선택자 점검**(`checkTargets`)·**화면 지도**(`pageMap`) — AI 도구 `check_script_targets`·`get_page_map` 의 바탕, `dom/` 이 없는 옛 실행은 trace.zip 에서 복원 ([SPEC_선택자-점검](SPEC_선택자-점검.md) · [ADR-030](ADR_030_선택자-점검-구현.md), 단계 12 구현). DSL `manual` 단계·`expect` 상태 검사(`enabled`·`disabled`·`attribute`·`style`)·`expectDownload.zip` ([ADR-037](ADR_037_DSL-수동-단계-상태-검사.md), 단계 16 구현) |
| `euta-engine` | `tc-doc` | **TC 문서 읽기·내보내기** — `parseTcDocument`(xlsx·csv·docx·md → 격자 → 블록형·행형 감지 → 정리된 TC, 못 읽으면 `gridText`) · `exportTcDocument`(xlsx 블록형·md) · `scriptsToTcs` · `groupPreconditions` · `validateTcs`. 외부 의존 없이 fflate + 작은 XML 읽기 ([SPEC_TC-문서-변환](SPEC_TC-문서-변환.md) · [ADR-038](ADR_038_TC-문서-읽기-내보내기.md), 단계 16 구현) |
| | `input-os` | OS 수준 입력 어댑터(선택). `input: os` 스텝을 실제 커서·키보드로 실행 ([ADR-013](ADR_013_OS-입력-어댑터.md)) |
| | `cli` | `euta run` · `validate` · `runs list/show` · `report` · `schema` · `mcp` — CI 와 사람이 GUI 없이 쓰는 진입점 |
| | `mcp-server` | stdio MCP 서버(외부 Claude Code·Codex 용) + **transport 중립 도구 핸들러** `@euta/mcp-server/tools` — 앱 AI 채팅과 같은 코드를 쓴다 ([ADR-008](ADR_008_MCP-서버-도구-설계.md)) |
| `euta-app` | — | Electron. 엔진을 메인 프로세스에 내장해 실행·표시. AI 채팅(Agent SDK) · Live · History · 설정 · 고급 터미널 · 로컬 서버 ([ADR-010](ADR_010_앱-구성-빌드.md) · [011](ADR_011_터미널-패널-AI-CLI-호스팅.md) · [012](ADR_012_앱-로컬-서버-라이브-스토어.md) · [015](ADR_015_앱-AI-채팅-Agent-SDK.md)) |
| `euta-claude-plugin` | — | Claude Code 플러그인(보조 경로). MCP 서버 연결, 테스트 작성·분석 skills, 앱으로 보내는 hooks ([ADR-009](ADR_009_Claude-Code-플러그인.md)) |

- 의존 방향: `app → engine`, `plugin → engine(MCP 서버 실행)`. engine 은 아무것도 모른다.
- 레포 간 연결: 개발 중에는 `link:` 의존([ADR-001](ADR_001_엔진-레포-구성-빌드.md)), 앱 패키징 때는 엔진 패키지를 tarball 로 묶어 스테이징([ADR-014](ADR_014_개인-테스트용-앱-패키징.md)).

## 앱 구성

앱은 **프로젝트(테스트 대상 사이트) 단위**로 동작한다(단계 8, [SPEC_프로젝트-관리](SPEC_프로젝트-관리.md) · [ADR-016](ADR_016_프로젝트-저장소-구현.md)). 왼쪽 사이드바에서 프로젝트를 고르고(고정 → 최근 순, 새 프로젝트, 앱 설정), 프로젝트 안에서 탭을 쓴다.

프로젝트는 **로컬**(동기화 없음)과 **클라우드**(euta cloud 공유 프로젝트의 작업 사본, 사이드바 ☁) 두 종류다([SPEC_클라우드-앱-연동](SPEC_클라우드-앱-연동.md) · [ADR-025](ADR_025_앱-클라우드-동기화-구현.md)). 클라우드 프로젝트도 탭·실행·채팅·녹화는 같고, 스크립트 저장은 서버(기준 버전, 충돌 3선택, 오프라인이면 보내기 대기), 이력은 서버 이력, 설정·환경·테스트 계정·테스트 세트·구성원은 서버에 저장한다(오프라인·보기 역할이면 읽기만). 사이드바 하단에 연결 상태 칩, "클라우드 프로젝트 찾기". 클라우드 프로젝트는 Run 탭에서 **실행 위치 이 PC / 클라우드**(클라우드 = 서버 실행 요청, Live 에 "클라우드" 배지, 앱을 꺼도 계속), History 는 로컬 + 클라우드 실행 병합(클라우드 실행은 열 때 `cloud-runs/<id>/` 로 내려받아 재생, 로컬 실행 "클라우드에 올리기"), **예약** 탭, 앱 알림(`/api/me/events` → OS 알림)이 더해진다([ADR-026](ADR_026_앱-클라우드-실행-위임-구현.md)).

| 탭 | 하는 일 |
|---|---|
| 스크립트 | 프로젝트 `scripts/` 의 YAML 목록(이름·경로·최근 실행) — **시연으로 만들기**·새로 만들기·가져오기·이름 변경·삭제·내보내기·검증·실행·기본 편집기로 열기. 바깥 변경은 감지해 갱신. 아래에 삭제된 스크립트(복원)·시연 녹화 목록(초안 저장·AI로 정리·이름 변경·삭제). **스크립트 상세**(보기): 한국어 단계 문장 + 최근 실행 겹치기(내용 해시가 같을 때만, 스크린샷) · 경고(검증 오류·없는 secret 이름·CSS 대상) · 단계 단위 폼 편집(추가·삭제·복제·순서·묶음 넣기/빼기·머리 정보, 실시간 검증, 저장 버튼) · 이력(버전 목록·단계 문장 비교·되돌리기) · AI에게 수정 요청(스크립트/단계 → 새 채팅, AI 저장 시 배너) ([SPEC_스크립트-보기-편집](SPEC_스크립트-보기-편집.md)). 위쪽 전환 **스크립트 \| 테스트 세트**(내부 이름 suite): 카드마다 실행(실행 위치는 Run 탭 선택 — `App` 상태) · 최근 결과 · 항목 순서와 "항상 실행" · 바로 저장되는 편집 · 이 PC 실행의 항목별 진행·취소, 목록에서 체크 → "테스트 세트에 넣기" ([ADR-031](ADR_031_테스트-세트-구현.md)). **TC 문서로 만들기**(시연으로 만들기 옆 → 파일 → 미리보기: TC 목록 · 단계 표 고치기 · 사전 조건 묶음 → 확정; 양식을 못 알아보면 [AI로 읽기]) · 위쪽 전환 세 번째 **TC 문서**: 목록 · 상세(검색 + 체크 = 대기열 순번 · 변환 상태 · 스크립트 · 수동 단계 · 최근 결과 · 단계 대응 펼치기 · AI로 변환/이어서 변환/시연으로 변환/다시 변환 — AI 변환은 **자동 진행**(앱이 항목마다 메시지 하나, 다음 항목 자동, 한도에 닿으면 한 번 더, 진행 개수, "n / N 진행 중" 띠 + [중지] — 상세 · 채팅 위, ADR-046) · 테스트 세트 만들기(전체·분류별·체크) · 내보내기 xlsx·md), 테스트 세트 카드·스크립트 체크 막대의 [TC 문서로 내보내기] ([SPEC_TC-문서-변환](SPEC_TC-문서-변환.md) · [ADR-039](ADR_039_TC-문서-변환-앱-구현.md), 단계 16 구현). 결과 화면(실시간 · 실행 이력 · 스크립트 상세 · 세트 리포트)의 도달한 `manual` 단계 = **"수동 확인 필요"** 배지와 개수 |
| Run | 스크립트 + **환경** 선택 → 검증 → 실행(vars·env 덮어쓰기, headless, 브라우저). 실행 위치(클라우드 프로젝트)는 테스트 세트 실행도 따른다 |
| Live | 실행별 스텝 타임라인·스크린샷·expect·지표·콘솔/네트워크 실패. 앱·테스트 세트·AI·외부 MCP 실행을 같은 화면에. 테스트 세트 실행은 목록에 한 줄로 묶이고, 고르면 세트 머리(이름 · 환경 · 완료/전체 · 경과 · 취소/리포트) + 항목 목록(실행 중 항목 펼침, 나머지는 눌러 펼침 — 이 PC·클라우드 같은 모양, ADR-032). 프로젝트를 바꿔도 실행은 계속(사이드바에 진행 표시) |
| History | 프로젝트 `runs/index.sqlite` 목록 + 환경 열 + "테스트 세트: 이름" 묶음 → 열기(steps.jsonl 재생) · 리포트 · 세트 묶음 줄 [리포트](세트 리포트, 클라우드 세트는 웹 실행 상세) ([ADR-032](ADR_032_테스트-세트-리포트-Live-구현.md)) |
| AI | 프로젝트별 채팅(작업 폴더 = 프로젝트 `scripts/`, 상단에서 실행 환경 선택, "시연으로 만들기" 버튼, 입력창 위 "말로 시키기" 예시) + "고급: 터미널"(pty · hook 활동, `EUTA_RUNS_DIR` = 프로젝트 runs) |
| 녹화 패널 | 시연 녹화 중 프로젝트 화면 오른쪽에 붙는 패널: 실시간 동작 목록 · 메모(Enter) · 확인(체크포인트, 종류 선택) · 일시정지/재개 · 마지막 동작 취소 · 줄별 민감/메모/삭제 · 종료 → "AI 없이 저장"(초안 → 라이브러리, 등록 필요 secret 표시) / "AI로 정리"(새 채팅에 녹화 첨부) ([SPEC_스크립트-작성-시연녹화](SPEC_스크립트-작성-시연녹화.md)). TC 문서에서 시작하면(시연으로 변환) 위에 TC 단계 안내(지금 단계 · 기대 결과 · [다음 단계] · 체크포인트 없으면 한 번 묻기) → 저장 시 TC 단계별 묶음 초안 + TC 완료 기록 (단계 16 구현, ADR-039) |
| 작업 로그 | 실행·스위트·AI 대화·스크립트·설정 변경의 시간 역순 타임라인, 종류·행위자 필터, 클릭 → run/채팅/스크립트 |
| 프로젝트 설정 | 기본 정보 · 환경(주소·변수·허용 도메인·secrets: vault 또는 외부 파일) · 기본 브라우저 · 삭제 · "클라우드로 올리기"(로컬). 클라우드: + 테스트 규칙 · 허용 도메인 · 테스트 계정 "클라우드/이 PC" 이중 표시 · 구성원 · 작업 사본 지우기 |
| 앱 설정(전역) | AI 기본 모델 · vault 상태 · 클라우드(서버 주소 · 장치 로그인 · 로그아웃) · 데이터 위치 |

- 메인 프로세스: `ProjectService`(프로젝트·환경·스위트 저장소, vault, 작업 로그, 스크립트 라이브러리·감시, 환경별 실행 조립, 스위트, 프로젝트별 채팅 엔진, 시연 녹화기 1개·녹화 저장소, AI 도구 `get_recording`, TC 문서 `TcDocs`(저장소 `tc-docs/<docId>/tc-doc.json` — 로컬에만, 가져오기 세션 메모리 보관, 변환 채팅 · 대기열 · 자동 진행 루프(메시지 하나 = 항목 하나, 문서당 하나 잠금, 진행 상태 메모리 — ADR-046), 세트 계획, 내보내기)·AI 도구 `get_tc_doc`·`get_tc`·`set_tc_conversion`·`get_tc_import`·`save_tc_extraction`(단계 16 구현, ADR-039), **스크립트 쓰기 한 경로 `writeScript`**(편집기·AI `save_script`·녹화 저장·되돌리기·생성/가져오기 → 충돌 검사·쓰기 전 동기화·원자적 교체·버전 기록·작업 로그, 감시자의 바깥 변경은 `external` 버전), 스크립트 이력 `ScriptHistory`), LiveStore(이벤트 → 화면 상태, run 에 projectId), 로컬 서버(`127.0.0.1`, 토큰 인증: `POST /events` · `POST /hooks/claude`), pty.
- 메인 프로세스(클라우드): `CloudService`(연결 상태·재연결·장치 로그인, 토큰 safeStorage 암호문, 클라우드 프로젝트 작업 사본, 연 프로젝트의 SSE 변경 수신, 서버 이력, 클라우드 쪽 설정, 올리기) + 프로젝트별 `ProjectSync`(보내기 대기열·충돌·피드 반영·전체 다시 받기). `ProjectService.writeScript` 는 클라우드 프로젝트면 작업 사본에 쓰고 대기열로 보낸다(`CloudBridge`). 토큰·클라우드 테스트 계정 값은 렌더러로 가지 않는다.
- 렌더러: React. preload 의 좁은 IPC 만 사용(contextIsolation, sandbox, CSP). 프로젝트별 호출은 projectId 를 명시한다. secret 값은 쓰기 전용(이름만 돌아옴).
- 실행 조립: `env` = 사용자 환경 + `BASE_URL`(환경 주소) + 환경 변수, `secrets` = 환경의 vault 값(또는 외부 파일), `runsDir` = 프로젝트 `runs/`, `baselineDir` = `baselines/<envId>/`, 브라우저 = 프로젝트 기본값, secrets 규칙 = `restrictSecrets` + 환경 허용 도메인(아래 보안). run.json `meta` 에 프로젝트·환경(·스위트/AI) 기록. DSL 은 그대로(`${env.BASE_URL}`).
- 로컬 저장(영구):

```
<userData>/app.json                최근 프로젝트 순서 · 마지막 프로젝트 · AI 기본 모델
<userData>/vault.json              secret 암호문만 (Electron safeStorage, 키 = projectId/envId/NAME)
<userData>/projects/<id>/          project.json · scripts/ · baselines/<envId>/ · runs/(+index.sqlite)
                                   designs/<name>.png + .json (Figma 에서 가져온 디자인) · design-reviews/<실행 id>.json (AI 검수 · 판정)
                                   suite-runs/*.json (+ suite-runs/<id>/report.html — 세트 리포트, ADR-032) · chats/*.json · activity/<YYYY-MM>.jsonl
                                   recordings/<recordingId>/recording.json + shots/<seq>.png (가림 적용)
                                   script-history/keys.json(경로 → scriptKey, 삭제 목록) · <scriptKey>/index.json + <version>.yaml (스크립트당 최근 100개)
<userData>/cloud/account.json      클라우드 서버 주소 · 계정 · 장치 토큰 암호문
<userData>/cloud/<serverId>/projects/<cloud projectId>/
                                   state.json(마지막 변경 순번 · 스크립트별 기준 버전 · 역할) · snapshot.json(서버 사본) · pending/(보내기 대기)
                                   project.json(파생) · scripts/ · runs/ · chats/ · recordings/ · baselines/ … (로컬 프로젝트와 같은 구조, script-history 없음)
```

- 단계 7까지의 전역 데이터(`settings.json` · `runs/` · `chats/`)는 읽지 않고 지우지도 않는다(이관 없음).

## 실행 흐름 (기본 모드)

입력 → 진행 → 결과 저장. 모든 단계가 이벤트와 로그를 남긴다.

1. **입력** — YAML 스크립트 로드 → 스키마 검증 → 변수 해석(`${env.*}` · `${secrets.*}` · `${vars.*}`) → 실행 계획. 실패하면 실행 전에 중단하고 검증 오류를 기록한다.
2. **진행** — 브라우저 컨텍스트 생성(뷰포트·디바이스·터치 에뮬레이션·녹화·HAR) → 스텝 순차 실행. 스텝마다 `before → act → wait-settle → measure → expect` 순서를 거친다.
3. **결과 저장** — `runs/<run-id>/` 에 증거 파일, `index.sqlite` 에 실행·스텝·지표 색인, `report.html` 생성.

### 입력 드라이버 (혼합)

| 드라이버 | 방식 | 기본 | 비고 |
|---|---|---|---|
| `browser` | Playwright/CDP 입력 (마우스·키보드·터치 에뮬레이션) | ✅ | 헤드리스·병렬 가능 |
| `os` | 요소 좌표 계산 → OS 커서·키보드로 실제 입력 | 스텝 옵션 `input: os` | **선택 어댑터** — 네이티브 모듈 + macOS 접근성 권한 필요. 없으면 명확한 오류로 실패. 헤드풀·직렬 실행 강제 |

- OS 드라이버는 인터페이스 뒤의 교체 가능한 어댑터로 둔다. 구현은 하되 **검증은 수동**(권한 대화상자를 사람이 허용해야 함).
- 터치는 두 드라이버 모두 데스크톱에서는 에뮬레이션이다(`hasTouch` 컨텍스트 + tap).

### 측정·분석 (1단계 4종)

| 범주 | 수집 | 판정 |
|---|---|---|
| 화면 증거·시각 비교 | 스텝별 스크린샷, 실행 영상, trace | 기준 이미지 대비 픽셀 diff(임계치), diff 이미지 저장 |
| 디자인 대조 (단계 11) | `design` 단계 — 실제 화면 · Figma 디자인 · 차이 이미지 | 판정 없음(경고·참고 수치). 사람 비교 화면 + 앱의 AI 검수 → [SPEC_디자인-검수](SPEC_디자인-검수.md) |
| 체감 성능 | 동작→화면 안정까지 시간, Web Vitals(LCP·CLS·INP), 로드 타이밍 | 스텝별 임계치 `expect` |
| 콘솔·네트워크 | console 로그·JS 에러, 요청/응답(HAR), 실패(4xx/5xx)·느린 요청 | 에러 0건·실패 요청 0건 등 `expect` |
| 결과물 파일 | 다운로드 파일 캡처 → `artifacts/` | 존재·형식·크기·내용(텍스트 포함, CSV 행 수 등) |

## 이벤트 모델 (모든 소비자의 접점)

엔진은 실행 중 이벤트를 내보내고, 앱 라이브 뷰 · MCP 도구 결과 · 로그 파일 · 리포트가 같은 이벤트를 소비한다. 스키마는 core 가 소유한다.

| 이벤트 | 주요 필드 | 소비자 |
|---|---|---|
| `run.started` / `run.finished` | runId, script, 환경, 상태, 요약 | 앱 · MCP · SQLite |
| `step.started` / `step.finished` | index, 종류, 대상, 상태, 소요 시간, 오류 | 앱 타임라인 · `steps.jsonl` · MCP |
| `artifact.saved` | 종류(screenshot·diff·video·trace·download), 경로 | 앱 뷰 · 리포트 |
| `metric.recorded` | 이름, 값, 단위, 스텝 | 앱 · `metrics.json` · SQLite |
| `console.message` / `network.failed` | 레벨·텍스트 / URL·상태·시간 | 앱 · `console.jsonl` · HAR |
| `expect.evaluated` | 조건, 기대값, 실제값, 통과 여부 | 앱 · 리포트 · MCP |
| `log` | 단계(입력·진행·저장), 레벨, 메시지 | `run.log` |

- 이벤트는 `steps.jsonl` 등에 그대로 기록되므로, 실행이 끝난 뒤에도 같은 타임라인을 재생할 수 있다.
- 앱은 엔진을 같은 프로세스 계열에서 내장해 이벤트를 직접 구독한다. 외부 Claude Code 세션이 MCP 로 실행한 경우는 MCP 서버가 앱의 로컬 엔드포인트로 이벤트를 전달한다(앱이 떠 있을 때만).

## 결과 저장

```
runs/
  <YYYYMMDD-HHmmss>_<script-slug>_<short-id>/
    run.json        메타 · 환경 · 결과 요약 · scriptHash(실행한 원문 sha256)
    steps.jsonl     이벤트 로그 (스텝 단위)
    run.log         단계별 텍스트 로그
    screenshots/    diffs/    video/    trace.zip
    dom/            단계별 DOM 스냅샷 (record.dom) — <seq>-<stepId>-<before|after>.html.gz · styles/<hash>.css.gz · index.json
    designs/<name>/ 디자인 비교 (actual · reference · diff · reference.json)
    metrics.json    network.har    console.jsonl
    artifacts/      다운로드 결과물
    report.html     자체 완결 리포트
index.sqlite        runs · steps · metrics 색인 (이력 검색·추이)
```

## 전문 모드 (AI 연동 — 앱 자체 AI 채팅 + 보조 경로)

근거와 정책 원문: [RESEARCH_AI-구독-연동-방식](../35.분석/RESEARCH_AI-구독-연동-방식.md) · PLAN 결정 "전문 모드 연동"(2026-10-02) · 설계 [ADR-015](ADR_015_앱-AI-채팅-Agent-SDK.md)

- **앱 AI 채팅(기본)** — AI 탭에서 사용자가 요청을 입력하면, Electron 메인의 Claude Agent SDK `query()` 가 SDK 번들 Claude Code CLI 를 띄워 **이 PC 의 본인 Claude 구독 로그인**으로 대화한다.
  - AI 는 euta 도구만 쓴다 — 엔진 MCP 서버와 같은 핸들러(`@euta/mcp-server/tools`)를 SDK in-process MCP 서버로 넘기고, 작업 폴더 안 파일 도구(`list_files`·`read_file`·`save_script`)를 더한다. 내장 셸·편집·웹 도구는 끄고, 모든 호출은 deny-by-default 게이트(PreToolUse hook + canUseTool)를 거친다. 사용자 `~/.claude` 설정·skills·MCP 는 섞지 않는다(`settingSources: []` · `skills: []` · `strictMcpConfig`).
  - AI 가 시작한 실행·세션은 같은 프로세스의 이벤트 버스를 직접 구독해 Live/History 에 `AI` 로 바로 보인다. 세션은 기본 headed.
  - 로그인·상태는 공식 `claude auth login/status/logout` 실행으로만. 앱은 자격증명·토큰을 읽거나 저장·중계하지 않고 인증 환경변수를 설정하지 않는다. API 키 등 과금 관련 환경변수는 이름만 경고한다.
  - **본인 PC · 본인 구독 · 본인 사용 전제**(배포 범위 밖).
- **P1 · MCP 서버(보조 경로)** — 외부 Claude Code / Codex 용. 도구 두 부류.
  - 실행형: 스크립트 검증·실행, 실행 목록·상세, 스크린샷·diff·지표·아티팩트 조회, 리포트 경로.
  - 탐색형(AI 가 한 스텝씩 조작): 브라우저 세션 열기/닫기, 단일 스텝 실행(DSL 스텝과 같은 어휘), 접근성 스냅샷·스크린샷, 지금까지의 조작을 YAML 스크립트로 내보내기.
- **P2 · Claude Code 플러그인(보조 경로)** — `.mcp.json`(엔진 MCP 서버), skills(스크립트 작성·실패 분석·결과 해석), `hooks.json`(HTTP hook → 앱 엔드포인트: 프롬프트·도구 호출·종료 이벤트).
- **P3 · 앱 내 CLI 호스팅(고급 옵션)** — AI 탭 "고급: 터미널" 이 사용자가 설치한 `claude`/`codex` 를 pty 로 **그대로** 실행한다. 로그인은 CLI 안에서 사용자가 직접 한다.
  - 앱은 CLI 바이너리·자격증명·토큰에 손대지 않는다. 인증 관련 환경변수를 주입하지 않는다.
  - Codex 는 `codex mcp add` 로 같은 MCP 서버에 연결한다(hook 대응은 추후).
- **사용량 전제** — 구독 경로는 대화형·보조용(채팅 턴당 최대 60 라운드). 무인 대량 AI 판정은 API 키 경로로 별도 설계(2단계).

## 보안·개인정보

- **secrets 는 결과물에 남기지 않는다.** 로그·`steps.jsonl`·리포트에는 마스킹 값만 기록한다. HAR 은 요청 헤더·본문·쿠키에서 secrets 값과 인증 헤더를 마스킹한 뒤 저장한다. secrets 가 입력된 필드는 스크린샷에서 가린다(마스킹 옵션).
- secrets 공급원은 환경변수·로컬 파일(레포 밖)로 한정하고, 스크립트 본문에는 참조만 둔다.
- **secrets 규칙(모든 로컬 실행, 2026-10-21)** — `${secrets.*}` 는 입력 스텝(`fill`·`type`) 값에만 쓸 수 있고(그 밖이면 실행 전 검증 오류), secrets 입력 스텝은 페이지 호스트가 환경의 허용 도메인(기본 = 환경 주소 호스트)일 때만 실행된다. 앱의 실행 버튼·스위트·AI 채팅·고급 터미널 MCP 모두 엔진 옵션 `restrictSecrets` + `allowedDomains` 로 같은 규칙을 쓴다 — AI(프롬프트 인젝션 포함)가 스크립트로 테스트 계정을 바깥 주소에 보내는 경로를 막는다. 클라우드 전용인 `input: os` 거절·녹화 기본값은 로컬에 켜지 않는다. 상세: [SPEC_프로젝트-관리 secrets 규칙](SPEC_프로젝트-관리.md#secrets-규칙-모든-로컬-실행) · [ADR-024](ADR_024_엔진-클라우드-보완.md)
- 앱 밖에서 단독으로 띄운 엔진 MCP 서버·CLI 는 기본값이 바뀌지 않는다(`EUTA_RESTRICT_SECRETS`·`EUTA_ALLOWED_DOMAINS` / `--restrict-secrets`·`--allowed-domains` 로 켬).
- 앱 로컬 엔드포인트(hook·이벤트 수신)는 `127.0.0.1` 에만 바인딩하고 세션 토큰으로 요청을 검증한다.

## 플랫폼

- macOS · Windows · Linux(x64 데스크톱, 2026-10-30 추가) 대상. 검증은 macOS(arm64)만 — Windows · Linux 는 단계 17 에서 실기 확인.
- Playwright 브라우저는 앱에 포함하지 않고 사용자 캐시를 쓴다(`euta install-browsers`).
- OS 입력 어댑터: macOS 는 접근성·화면 기록 권한 필요(수동 검증 — [TEST_OS입력-어댑터_Plan](../60.테스트/TEST_OS입력-어댑터_Plan.md)), Windows 는 권한 불필요.
- 앱 패키징: **개인 테스트용 `.app`**(ad-hoc 서명, 이 Mac 전용, `pnpm package`) — [ADR-014](ADR_014_개인-테스트용-앱-패키징.md). **사내 배포판**(단계 17): 릴리스 빌드(맥 arm64·x64 dmg/zip, Windows NSIS, Linux AppImage) · 업데이트는 euta cloud 서버에서 받고(서명 있으면 electron-updater 로 뒤에서, 서명 없는 맥은 알림 + 내려받기) · 클라우드 최소 앱 버전 미만이면 클라우드 기능만 잠금 — [SPEC_앱-배포-업데이트](SPEC_앱-배포-업데이트.md). 구현: `pnpm release`(OS 별 스테이징 — 대상 플랫폼 CLI 패키지 · 리눅스 node-pty 는 Docker 에서 Electron 헤더로 컴파일 · 맥 두 arch 한 번 빌드) · `pnpm release:upload`(draft → 파일 → 게시) · main 의 `AppUpdater`(30초 뒤 · 6시간마다 · 설정 버튼, 서명 여부는 빌드가 적은 `eutaBuild`) · 클라우드 요청마다 `X-Euta-App-Version`, `426` → 연결 상태 `app_too_old` — [ADR-042](ADR_042_앱-배포-업데이트-앱-구현.md).

## 관련 문서

- 스크립트 보기·편집 명세(단계 10 구현 완료): [SPEC_스크립트-보기-편집](SPEC_스크립트-보기-편집.md) · [ADR-018](ADR_018_스크립트-편집기-이력-구현.md)

- 계획서: [PLAN_end-user-테스트툴-종합계획](../20.프로젝트관리/PLAN_end-user-테스트툴-종합계획.md)
- DSL 명세: [SPEC_test-script-dsl](SPEC_test-script-dsl.md)
- 프로젝트 관리 명세: [SPEC_프로젝트-관리](SPEC_프로젝트-관리.md)
- 스크립트 작성 명세(시연 녹화 → AI 정리, 단계 9 구현 완료): [SPEC_스크립트-작성-시연녹화](SPEC_스크립트-작성-시연녹화.md) · [ADR-017](ADR_017_시연-녹화기-구현.md)
- TC 문서 변환 명세(자연어 TC ↔ 스크립트, 단계 16 설계): [SPEC_TC-문서-변환](SPEC_TC-문서-변환.md)
- AI 연동 조사: [RESEARCH_AI-구독-연동-방식](../35.분석/RESEARCH_AI-구독-연동-방식.md)
- 구현 세부 결정: `ADR_001` ~ `ADR_018` (이 폴더)

## Change Log

### 2026-11-05 — 단계 16-4 앱 구현 반영: TC 변환 자동 진행(`TcDocs` 루프 · 진행 띠 · [중지]) · AI 저장 전 검증 · 거절된 저장 버리기 보정 (ADR-046)

### 2026-10-30 — 플랫폼: 앱 배포 · 업데이트 구현 반영 (릴리스 빌드 · 올리기 · 업데이터 · 426, ADR-042)

### 2026-10-30 — 플랫폼: 앱 사내 배포판 · 업데이트 경로 · 최소 앱 버전 (단계 17 설계, SPEC_앱-배포-업데이트)

### 2026-10-26 — 단계 16-3 앱 구현 반영: TC 문서 화면(스크립트 탭 세 번째 전환) · `TcDocs` · AI 도구 5종(`get_tc_import` 추가) · 녹화기 `markStep`/묶음 초안 · "수동 확인 필요" 표시 (ADR-039)

### 2026-10-26 — core 에 TC 문서 읽기·내보내기 · DSL `manual`·상태 검사, 앱에 TC 문서 화면·저장소·AI 도구 4종, 녹화 패널 TC 단계 안내 (단계 16 설계, SPEC_TC-문서-변환)

### 2026-10-23 — 단계 14: Live 테스트 세트 묶음(세트 한 줄 · 머리 + 항목 펼침, `RunView.suiteRunId` · 클라우드 `suiteItems`), 세트 리포트 `suite-runs/<id>/report.html`(main `suite-report.ts`, 리포트 창 공용 `showReport`), 클라우드 세트 [리포트] = 웹 실행 상세 (ADR-032)

### 2026-10-23 — 단계 13: 테스트 세트를 스크립트 탭 안으로(실행 위치는 App 상태로 올려 Run 탭과 공유), 프로젝트 설정·Run 탭의 스위트 칸 삭제, 항상 실행 (ADR-031)

### 2026-10-23 — 단계 12 구현 반영: 결과 폴더 `dom/` 구성 · trace 복원 보조 경로 (ADR-030)

### 2026-10-23 — core 에 단계별 DOM 스냅샷·선택자 점검·화면 지도(단계 12 설계, SPEC_선택자-점검)

### 2026-10-23 — 디자인 대조(단계 11) — 측정 표 · 결과 폴더 `designs/` · 프로젝트 `designs/` · `design-reviews/` (SPEC_디자인-검수). Figma 토큰은 앱 전역 설정(safeStorage)

### 2026-10-21 — 앱 구성에 클라우드 실행 위임(C3: 실행 위치 · Live · History 병합 · 올리기 · 예약 탭 · 앱 알림) 반영 (ADR-026)

### 2026-10-21 — 보안: secrets 규칙(사용 위치 제한 + 허용 도메인)을 모든 로컬 실행에 적용, 환경 허용 도메인 (PLAN_euta-cloud 결정 "secrets 규칙 적용 범위")

### 2026-10-21 — 앱 구성에 클라우드 프로젝트 종류 반영 (C2 구현, ADR-025)

### 2026-10-02 — 단계 10 구현 반영: core `describeStep`·`rewriteScript`·run.json `scriptHash`, 앱 스크립트 상세·편집·이력·쓰기 경로 (ADR-018)
### 2026-10-02 — 스크립트 보기·편집 명세 연결 — 단계 10 설계 확정
### 2026-10-02 — 단계 9 구현 반영: core 녹화기·초안 변환, 앱 녹화 패널·녹화 저장소·`get_recording` (ADR-017)
### 2026-10-02 — 스크립트 작성(시연 녹화) 명세 연결 — 단계 9 설계 확정
### 2026-10-02 — 앱 구성 절을 구현된 프로젝트 모델로 갱신 (단계 8 구현 완료, ADR-016)
### 2026-10-02 — 앱 구성에 프로젝트 단위 전환(단계 8) 반영
### 2026-10-02 — 구현 상태 역반영: 구성도·레포 책임(input-os·공용 도구 핸들러)·앱 구성 절·플랫폼/패키징
### 2026-10-02 — 전문 모드를 앱 자체 AI 채팅(Agent SDK) 중심으로 갱신, 터미널은 고급 옵션
### 2026-09-29 — 결정 반영, 본문 작성
### 2026-09-28 — 골격 작성
