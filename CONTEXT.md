# CONTEXT — euta (End-User 테스트 자동화 툴)

**Version**: 1.3
**Last Updated**: 2026-10-20

> 추가 설명 없이도 다음 세션·다른 AI 가 즉시 이어갈 수 있는 최소 상태 문서.
> 얇은 색인 — 내용은 정본 문서에, 여기는 1줄 + 링크.
> 진행·상태·이력은 `.ctx/lanes/root/` (STATE.md · JOURNAL.md) 가 전담 — §3·§8·§9 는 비워 둔다.

---

## 1. Project Overview

- **목적**: 개발 단계 E2E 테스트로는 부족한 상용화 검증을 메우기 위해, 실제 사용자처럼 웹앱/웹페이지를 조작·측정·분석하는 **End-User 테스트 툴**을 만든다.
- **대상**: SaaS 로 서비스 중이거나 서비스될 웹 서비스. 특정 서비스 전용이 아니라 범용으로 여러 서비스를 테스트한다.
- **모드**: 기본 모드(스크립트 기반 자동화) + 전문 모드(기본 모드에 AI 가 붙어 인텔리전트 테스트 보조).
- **형태**: 로컬 앱(만들기) + 클라우드 웹 euta cloud(돌리기·공유). 엔진 공용.
- **현재 단계**: 구현 — 엔진·MCP·플러그인·앱(AI 채팅 · 프로젝트 관리 · 시연 녹화 · 스크립트 보기/편집)·개인 테스트용 패키징까지 구현·자동 검증(단계 0~10). 실사용 확인 단계. 클라우드 웹(euta cloud)도 구현 단계 — 진행은 [PLAN_euta-cloud](20.프로젝트관리/PLAN_euta-cloud.md) 단계 표.
- **레포**: 상위 폴더 아래 `euta-workspace`(문서) · `euta-engine` · `euta-app` · `euta-claude-plugin` · `euta-cloud` — 지도는 [`../CLAUDE.md`](../CLAUDE.md)

## 2. 프로젝트 영역

이 프로젝트는 웹 서비스 대상 end-user 테스트의 입력 → 진행 → 결과 저장 → 분석 전 과정을 다룬다.

### [A] 기본 모드 실행 파이프라인
- **무엇**: 테스트 스크립트를 받아 입력 → 진행 → 결과 저장까지 자동 실행하고 단계별 로그를 남긴다.
- **왜**: 툴의 기본 동작이자 전문 모드의 토대.
- **정본 문서**: [ARCH_end-user-테스트툴](40.설계/ARCH_end-user-테스트툴.md)

### [B] 사용자 동작 · 측정 · 분석
- **무엇**: 마우스·키보드·터치 동작을 실제 사용자처럼 수행하고, 화면 변화를 측정하며, 결과물을 분석한다.
- **왜**: 개발 단계 E2E 가 놓치는 실사용 품질을 검증하는 핵심 가치.
- **정본 문서**: [ARCH_end-user-테스트툴](40.설계/ARCH_end-user-테스트툴.md) *(세부 SPEC 신설 대기)*

### [C] 테스트 스크립트 규격
- **무엇**: 여러 서비스에 공통으로 쓰는 테스트 스크립트 입력 형식.
- **왜**: 범용 툴의 전제 — 서비스별 코드 수정 없이 스크립트만 바꿔 테스트한다.
- **정본 문서**: [SPEC_test-script-dsl](40.설계/SPEC_test-script-dsl.md)

### [D] 전문 모드 (AI 연동)
- **무엇**: 앱 자체 AI 채팅(Agent SDK + 본인 Claude 구독)이 euta 도구로 엔진을 조작한다. 외부 Claude Code·Codex 는 MCP 서버·플러그인으로 보조 연결.
- **왜**: 스크립트만으로 어려운 판단(탐색·이상 판정·결과 해석)을 AI 가 돕는다.
- **정본 문서**: [ARCH 전문 모드](40.설계/ARCH_end-user-테스트툴.md) · [ADR-015](40.설계/ADR_015_앱-AI-채팅-Agent-SDK.md) · [RESEARCH](35.분석/RESEARCH_AI-구독-연동-방식.md)

### [E] 클라우드 실행·공유 (euta cloud)
- **무엇**: PC 가 꺼져 있어도 스크립트를 대신 실행하고, 프로젝트를 구성원이 함께 본다.
- **왜**: 로컬 앱은 PC 가 켜져 있어야 돌고 공유가 안 된다.
- **정본 문서**: [PLAN_euta-cloud](20.프로젝트관리/PLAN_euta-cloud.md) · [ARCH_euta-cloud](40.설계/ARCH_euta-cloud.md)

## 3. Active Tasks

> lane 활성 — 단기 체크리스트는 `.ctx/lanes/root/STATE.md` 의 "다음 할 일" 이 전담한다.

## 4. Decisions & Rationale

> 정본 = [PLAN 결정 사항](20.프로젝트관리/PLAN_end-user-테스트툴-종합계획.md#결정-사항) (이유·제외 대안 포함) · 클라우드 = [PLAN_euta-cloud 결정 사항](20.프로젝트관리/PLAN_euta-cloud.md#결정-사항). 구현 세부 = `40.설계/ADR_*`.

| 결정 | 이유 |
|---|---|
| 입력 = 혼합(Playwright 기본 + 스텝 단위 OS 입력) | 헤드리스·병렬 유지 + 봇 탐지·OS 다이얼로그 대응 |
| TypeScript · Electron 앱 + GUI 없는 엔진/CLI | Playwright·MCP SDK 1차 언어, MCP·CI 는 GUI 없이 호출 |
| 스크립트 = YAML DSL (JSON Schema) | 사람과 AI 가 같은 형식을 읽고 씀 |
| 저장 = 실행별 폴더 + SQLite 색인 | 실행 단위 공유 + 이력 검색 |
| 전문 모드 = 앱 자체 AI 채팅(Agent SDK + 본인 구독), 터미널은 고급 옵션 | 앱에 AI 가 붙은 경험. 본인 PC·본인 구독·본인 사용 한정 |
| 레포 = 상위 폴더 + 모듈별 독립 레포 5개(workspace · engine · app · claude-plugin · cloud) | 모듈마다 수명주기가 다름 |
| 개인 제품 — 개인 GitHub `onedark23` 공개(모듈별 5개, 라이선스 없음), 회사는 같은 소스로 사내 운영(배포 설정 = 사내 배포 레포(비공개)), 제품 문서 = GitHub · 회사 내용 = 사내 문서(비공개) | 개인 제품 · 포트폴리오 (사용자 2026-11-05) → [ADR-045](40.설계/ADR_045_공개-저장소-전환.md) |
| 작업 방식 = 설계 우선(문서 → 구현 → 역반영) | 문서와 코드가 어긋나지 않게 (사용자 지시 2026-10-02) |
| 패키징 = 개인 테스트용 `.app`, 배포 범위 밖 | 개인 테스트에 필요한 최소 |
| 스크립트 보기·편집 = 단계 문장 보기 + 폼 편집 + AI 요청, 저장 버튼 + 변경 이력 | YAML 없이 읽고 고치고 되돌림 → [SPEC](40.설계/SPEC_스크립트-보기-편집.md) |
| 스크립트 작성 = 시연 녹화 + 텍스트 설명 → AI 정리 → 자동 확인 | 사람은 시연·설명, AI 는 스크립트화 → [SPEC](40.설계/SPEC_스크립트-작성-시연녹화.md) |
| 프로젝트(사이트) 단위 관리 · 앱 내부 저장 · 이관 없음 | 여러 서비스를 나눠 영구 보관 → [SPEC](40.설계/SPEC_프로젝트-관리.md) · [ADR-016](40.설계/ADR_016_프로젝트-저장소-구현.md) (구현 완료) |

## 5. Constraints

- **플랫폼**: macOS 와 Windows 양쪽에서 동작해야 한다.
- **대상 범위**: 이번 단계는 웹앱/웹페이지 테스트만. (네이티브 앱은 범위 밖)
- **범용성**: 특정 서비스에 종속되지 않는다. 테스트 스크립트는 공통 형식으로 입력받는다.
- **기록**: 자동화 각 단계(입력·진행·결과 저장)마다 로그를 남긴다.
- **AI 연동 다양성**: 전문 모드는 특정 AI 한 곳에 묶이지 않는다 (Claude Code · Codex · 각 AI API).

## 6. Important Context

- **계획서**: [PLAN_end-user-테스트툴-종합계획](20.프로젝트관리/PLAN_end-user-테스트툴-종합계획.md)
- **요구사항 · 기능 · 현황**: [REQ_euta-요구사항](30.요구사항/REQ_euta-요구사항.md) (요구별 반영·상태) · [FEATURE_euta-기능-목록](43.기능정의/FEATURE_euta-기능-목록.md) · 개발 현황 보고 `23.보고/WR_*`
- **아키텍처**: [ARCH_end-user-테스트툴](40.설계/ARCH_end-user-테스트툴.md) · DSL [SPEC_test-script-dsl](40.설계/SPEC_test-script-dsl.md)
- **레포 지도·구현 운영 규칙**: 상위 [`../CLAUDE.md`](../CLAUDE.md) — euta-workspace · euta-engine · euta-app · euta-claude-plugin
- **설계 결정 기록(ADR)**: `40.설계/ADR_*` — 엔진 구성·DSL 검증·SQLite·PDF·시각 비교·secrets 마스킹·실행 API·MCP·플러그인·앱
- **AI 구독 연동 조사**: [RESEARCH_AI-구독-연동-방식](35.분석/RESEARCH_AI-구독-연동-방식.md) — 벤더 정책 원문·연동 패턴
- **문서 트리 규약**: 번호 폴더 + pdoc prefix 규약
- **skill**: 맥락 = `~/.claude/skills/ctx/` · 문서 = `~/.claude/skills/pdoc/` (prefix 정본 `naming.md`)
- **상태·이력**: `.ctx/lanes/root/STATE.md` (지금 무엇부터) · `JOURNAL.md` (여기까지 뭘 했나)

## 7. Known Issues / TODO

- 영상·trace 는 secrets 마스킹이 안 된다 — 민감 계정은 녹화를 끈다 (ADR-006).
- AI 가 `${secrets.*}` 를 외부 URL 로 보내는 경로가 막혀 있지 않다 — 실사이트 테스트 전 DSL 차원 대책 필요.
- Windows 는 코드상 고려만, 실기 미검증. 정량 목표·성공 기준 미정.

## 8. Next Actions

> lane 활성 — 즉시 실행 항목은 `.ctx/lanes/root/STATE.md` 의 "다음 할 일" 이 전담한다.

## 9. Session Log

> lane 활성 — 날짜별 이력은 `.ctx/lanes/root/JOURNAL.md` 가 전담한다.

---

## 10. LLM Bootstrap Context

이 프로젝트는 SaaS 웹 서비스를 실제 사용자처럼 조작·측정·분석하는 범용 end-user 테스트 툴(기본 모드 + AI 전문 모드)을 만든다.
엔진·MCP·플러그인·앱(AI 채팅)·개인용 패키징까지 구현됐고 클라우드 웹(euta cloud)을 구현 중이며, 기능 변경은 설계 문서를 먼저 고친 뒤 구현한다.
다음 작업은 `.ctx/lanes/root/STATE.md` 의 다음 할 일 참조.
