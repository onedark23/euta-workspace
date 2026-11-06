---
title: FEATURE - euta 기능 목록
path:  euta-workspace / 43.기능정의 / FEATURE - euta-기능-목록
ref:
---

# FEATURE - euta 기능 목록

> **지금 쓸 수 있는 기능의 목록.** 사용 면(앱 화면 · CLI · MCP 도구 · 플러그인)별로 한 줄씩 적는다. 동작 상세는 각 SPEC · ADR 이 정본이다.
> 기준: 2026-10-02 단계 10 구현까지 (엔진 · 앱 · 플러그인 로컬 커밋 기준). euta cloud 는 계획 단계라 여기에 없다 → [PLAN_euta-cloud](../20.프로젝트관리/PLAN_euta-cloud.md).

상태 표기 — **✓** 구현·자동 검증 통과 · **수동** 구현, 사람이 하는 검증 대기 · **⏸** 뒤로 미룸. 단계 0~10 전체가 아직 실사용 확인 전이다.

## 데스크톱 앱 (euta-app)

앱은 프로젝트(테스트 대상 사이트) 단위로 동작한다. 왼쪽 사이드바에서 프로젝트를 고르고 프로젝트 안에서 탭을 쓴다. → [ARCH 앱 구성](../40.설계/ARCH_end-user-테스트툴.md#앱-구성)

| 화면 | 기능 | 상태 | 정본 |
|---|---|---|---|
| 사이드바 | 프로젝트 목록(고정 → 최근 순) · 새 프로젝트 · 실행 중 표시 · 앱 설정 진입 | ✓ | [SPEC_프로젝트-관리](../40.설계/SPEC_프로젝트-관리.md) |
| 스크립트 | 목록(이름 · 경로 · 최근 실행), 새로 만들기 · 가져오기 · 이름 변경 · 삭제(복원) · 내보내기 · 검증 · 실행 · 기본 편집기로 열기, 바깥 변경 감지 | ✓ | SPEC_프로젝트-관리 |
| 스크립트 › 시연으로 만들기 | 실제 사이트에서 시연 녹화 → 초안 저장(AI 없이) 또는 AI로 정리 | ✓ | [SPEC_스크립트-작성-시연녹화](../40.설계/SPEC_스크립트-작성-시연녹화.md) |
| 스크립트 › TC 문서 | 자연어 TC 문서(xlsx · csv · docx · md) 가져오기 → 미리보기·고치기 → TC 목록(검색 + 체크 순서 = 변환 대기열 · 변환 상태 · 수동 단계 수) → AI로 변환 / 이어서 변환(자동 진행 — TC 마다 메시지 하나, 다음 TC 자동, 한도에 닿으면 한 번 더, 진행 개수, "n / N 진행 중" + [중지]) · 시연으로 변환(TC 단계 안내) · 테스트 세트 만들기(문서 전체 · 분류별 · 체크) · 내보내기(xlsx 블록형 · md). 실행 화면에 "수동 확인 필요" 표시 | ✓ (가짜 에이전트 E2E). 실제 모델 변환 확인 대기 | [SPEC_TC-문서-변환](../40.설계/SPEC_TC-문서-변환.md) · [ADR-039](../40.설계/ADR_039_TC-문서-변환-앱-구현.md) · [ADR-046](../40.설계/ADR_046_TC-변환-자동-진행.md) |
| 녹화 패널 | 실시간 동작 목록 · 텍스트 메모 · 체크포인트(확인) · 일시정지/재개 · 마지막 동작 취소 · 줄별 민감/메모/삭제 · 민감 값 페이지 안 폐기 | ✓ | SPEC_스크립트-작성-시연녹화 · [ADR-017](../40.설계/ADR_017_시연-녹화기-구현.md) |
| 스크립트 상세 | 한국어 단계 문장 보기 + 최근 실행 결과 겹치기(스크린샷) · 경고(검증 오류 · 없는 secret · CSS 대상) | ✓ | [SPEC_스크립트-보기-편집](../40.설계/SPEC_스크립트-보기-편집.md) |
| 스크립트 상세 › 편집 | 단계 단위 폼 편집(추가 · 삭제 · 복제 · 순서 · 묶음 넣기/빼기 · 머리 정보), 실시간 검증, 저장 버튼(주석 보존) | ✓ | SPEC_스크립트-보기-편집 · [ADR-018](../40.설계/ADR_018_스크립트-편집기-이력-구현.md) |
| 스크립트 상세 › 이력 | 버전 목록(저장 · AI 수정 · 녹화 · 외부 변경) · 단계 문장 비교 · 되돌리기 · 편집 중 바깥 변경 알림 | ✓ | SPEC_스크립트-보기-편집 |
| 스크립트 상세 › AI에게 수정 요청 | 스크립트 · 단계를 새 채팅으로 넘겨 수정, AI 저장 시 배너 | ✓ | SPEC_스크립트-보기-편집 |
| Run | 스크립트 + 환경 선택 → 검증 → 실행(vars · env 덮어쓰기, headless, 브라우저) · 스위트 실행 · 취소 | ✓ | SPEC_프로젝트-관리 |
| Live | 실행별 스텝 타임라인 · 스크린샷 · expect · 지표 · 콘솔/네트워크 실패. 앱 · 스위트 · AI · 외부 MCP 실행을 한 화면에 | ✓ | [ADR-012](../40.설계/ADR_012_앱-로컬-서버-라이브-스토어.md) |
| History | 프로젝트 실행 이력(환경 열 · 스위트 묶음) → 열기(타임라인 재생) · 리포트 | ✓ | SPEC_프로젝트-관리 |
| AI | 프로젝트별 채팅(본인 Claude 구독, Agent SDK) — euta 도구로 엔진 조작, 실행 환경 선택, "말로 시키기", 로그인 상태(`claude auth`) | ✓ (가짜 에이전트 E2E). 실제 모델 호출 0회 | [ADR-015](../40.설계/ADR_015_앱-AI-채팅-Agent-SDK.md) |
| AI › 고급: 터미널 | 사용자가 설치한 `claude` · `codex` 를 pty 로 그대로 실행, hook 활동 표시 | ✓ | [ADR-011](../40.설계/ADR_011_터미널-패널-AI-CLI-호스팅.md) |
| 작업 로그 | 실행 · 스위트 · AI 대화 · 스크립트 · 설정 변경의 시간 역순 타임라인, 종류 · 행위자 필터 | ✓ | SPEC_프로젝트-관리 |
| 프로젝트 설정 | 기본 정보 · 환경 여러 개(주소 · 변수 · secrets: vault 또는 외부 파일) · 기본 브라우저 · 스위트 · 삭제 | ✓ | SPEC_프로젝트-관리 · [ADR-016](../40.설계/ADR_016_프로젝트-저장소-구현.md) |
| 앱 설정 | AI 기본 모델 · vault 상태 · 데이터 위치 | ✓ | SPEC_프로젝트-관리 |
| 패키징 | 개인 테스트용 macOS `.app` (`pnpm package`, ad-hoc 서명, 이 Mac 전용) | ✓ | [ADR-014](../40.설계/ADR_014_개인-테스트용-앱-패키징.md) |

## 엔진 (euta-engine)

| 영역 | 기능 | 상태 | 정본 |
|---|---|---|---|
| 스크립트 DSL | YAML 스크립트 파싱 · JSON Schema 검증, 변수 `${env.*}` · `${secrets.*}` · `${vars.*}` | ✓ | [SPEC_test-script-dsl](../40.설계/SPEC_test-script-dsl.md) |
| 스텝 | `goto` · `click` · `dblclick` · `hover` · `tap` · `fill` · `type` · `press` · `select` · `check`/`uncheck` · `upload` · `scroll` · `drag` · `wait` · `group` | ✓ | SPEC_test-script-dsl |
| 검증 · 측정 | `expect` · `measure`(스크린샷 · Web Vitals · 타이밍) · `visual`(기준 이미지 픽셀 비교) · `expectDownload`(결과물 파일) | ✓ | SPEC_test-script-dsl · [ADR-005](../40.설계/ADR_005_시각-비교-픽셀-diff.md) · [ADR-004](../40.설계/ADR_004_PDF-텍스트-추출.md) |
| 수동 단계 · 상태 검사 | `manual`(실행 안 함, "수동 확인 필요") · `expect` 의 `enabled`/`disabled`/`attribute`/`style` · `expectDownload.zip` | ✓ | SPEC_test-script-dsl · [ADR-037](../40.설계/ADR_037_DSL-수동-단계-상태-검사.md) |
| TC 문서 (`@euta/tc-doc`) | 자연어 TC 문서 읽기(블록형 · 행형 감지) · 내보내기(xlsx · md) · 스크립트 → TC | ✓ | [ADR-038](../40.설계/ADR_038_TC-문서-읽기-내보내기.md) |
| AI 스텝 | `ai` — 전문 모드 2단계 | ⏸ (예약, 지금은 "지원 안 함" 오류) | SPEC_test-script-dsl |
| 입력 | 브라우저 입력(Playwright, 기본) + 스텝 단위 OS 입력(`input: os`, 선택 어댑터) | 브라우저 ✓ · OS 수동 | [ADR-013](../40.설계/ADR_013_OS-입력-어댑터.md) · [TEST_OS입력-어댑터_Plan](../60.테스트/TEST_OS입력-어댑터_Plan.md) |
| 진단 | 콘솔 로그 · JS 에러 · HAR · 실패/느린 요청 | ✓ | ARCH 측정·분석 |
| 결과 저장 | 실행별 폴더(`run.json` · `steps.jsonl` · `run.log` · 스크린샷 · 영상 · trace · HAR · 결과물 · `report.html`) + `index.sqlite` 색인 | ✓ | ARCH 결과 저장 · [ADR-003](../40.설계/ADR_003_SQLite-바인딩.md) |
| secrets 마스킹 | 로그 · 리포트 · HAR · 스크린샷에서 secrets 가림 | ✓. 영상 · trace 는 가리지 못함 | [ADR-006](../40.설계/ADR_006_secrets-마스킹.md) |
| 시연 녹화기 | headed 녹화 · 대상 후보 · 체크포인트 · 가린 스크린샷 → `recording.json`, 결정적 초안 변환 | ✓ | ADR-017 |
| 단계 문장 · 다시 쓰기 | 한국어 단계 문장 생성 · 주석 보존 다시 쓰기 · 실행 기록 `scriptHash` | ✓ | ADR-018 |

## CLI (`euta`)

| 명령 | 기능 | 상태 |
|---|---|---|
| `euta run` | 스크립트 실행 (기본 headless, `--runs-dir` · `--secrets`) | ✓ |
| `euta validate` · `euta schema` | 스크립트 검증 · JSON Schema 출력 | ✓ |
| `euta runs list` · `runs show` · `report` | 실행 이력 · 상세 · 리포트 | ✓ |
| `euta mcp` | stdio MCP 서버 기동 | ✓ |
| `euta install-browsers` | Playwright 브라우저 설치(사용자 캐시) | ✓ |

## MCP 도구 (외부 Claude Code · Codex 용, 앱 AI 채팅과 같은 핸들러)

→ [ADR-008](../40.설계/ADR_008_MCP-서버-도구-설계.md)

| 부류 | 도구 | 상태 |
|---|---|---|
| 실행형 | `validate_script` · `run_script` · `list_runs` · `get_run` · `get_screenshot` · `get_artifact` · `get_schema` | ✓ |
| 탐색형 | `session_open` · `session_step` · `session_snapshot` · `session_screenshot` · `session_export`(YAML 로 내보내기) · `session_close` | ✓ |
| 앱 AI 채팅 전용 | `list_files` · `read_file` · `save_script` · `get_recording` · TC 문서 `get_tc_doc` · `get_tc` · `set_tc_conversion` · `get_tc_import` · `save_tc_extraction` | ✓ |

## Claude Code 플러그인 (euta-claude-plugin)

→ [ADR-009](../40.설계/ADR_009_Claude-Code-플러그인.md)

| 구성 | 기능 | 상태 |
|---|---|---|
| `.mcp.json` | 엔진 MCP 서버 연결 | ✓ |
| skills | `write-test-script`(스크립트 작성) · `analyze-test-run`(실패 분석 · 결과 해석) | ✓ |
| hooks | 프롬프트 · 도구 호출 · 종료 이벤트를 앱으로 전달(HTTP) | ✓ |
| marketplace | 로컬 marketplace 설치 | ✓ |

- Codex 는 `codex mcp add` 로 같은 MCP 서버에 연결한다. Codex 용 hook 은 ⏸.

## 관련 문서

- 요구사항: [REQ_euta-요구사항](../30.요구사항/REQ_euta-요구사항.md)
- 개발 현황: [WR_20261020](../23.보고/WR_20261020.md)
- 아키텍처: [ARCH_end-user-테스트툴](../40.설계/ARCH_end-user-테스트툴.md)

## Change Log

### 2026-11-05 — TC 변환 자동 진행 · AI 저장 전 검증 반영 (단계 16-4, ADR-046)

### 2026-10-26 — 단계 16 TC 문서 변환 · 수동 단계 · 상태 검사 · `@euta/tc-doc` · AI 도구 5종 추가 (단계 11~15 항목은 아직 미반영)

### 2026-10-20 — 작성 (단계 10 구현 기준)
