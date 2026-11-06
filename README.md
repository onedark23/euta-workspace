# euta-workspace

euta(End-User Test Automation)의 문서 저장소 — 계획 · 요구사항 · 설계(ARCH · SPEC · ADR) · 테스트 계획. 실제 사용자처럼 웹 서비스를 조작 · 측정 · 분석하는 테스트 툴이다(로컬 앱으로 만들고, 클라우드 웹으로 돌리고 공유).

## 레포 구성

euta 는 레포 다섯 개로 나뉜다. **같은 상위 폴더에 나란히 클론**한다 — 레포끼리 `link:../euta-engine/...` 상대 경로로 의존한다.

```sh
mkdir euta && cd euta
for r in euta-workspace euta-engine euta-app euta-cloud euta-claude-plugin; do
  git clone https://github.com/onedark23/$r.git
done
```

| 레포 | 내용 |
|---|---|
| `euta-workspace` | 문서(이 레포) |
| `euta-engine` | GUI 없는 Node 패키지 묶음 — YAML DSL 파서 · 실행 엔진 · 측정/분석 · 결과 저장 · CLI · MCP 서버 |
| `euta-app` | Electron 데스크톱 앱 — 실행 · 결과 화면 + 터미널 패널(사용자의 `claude`/`codex` 호스팅) |
| `euta-claude-plugin` | Claude Code 플러그인 — MCP 연결 · skills · HTTP hooks · marketplace |
| `euta-cloud` | 클라우드 웹(euta cloud) — 실행 위임 · 예약 · 공유 프로젝트 · 원격 MCP 커넥터(서버 · 웹 · 러너). 서버는 각자 구축한다 — 예시 `k8s/example` |

빌드 순서: `euta-engine` 을 먼저 빌드(`pnpm install && pnpm build`)한 뒤 앱 · 클라우드를 빌드한다. 레포별 준비 · 실행은 각 레포 README.

## 문서 지도

| 폴더 | 내용 |
|---|---|
| `20.프로젝트관리/` | 종합 계획(결정 사항 표가 정본) · 클라우드 계획 |
| `30.요구사항/` | 요구사항 |
| `35.분석/` | 조사 · 분석 |
| `40.설계/` | ARCH(구조) · SPEC(동작 명세) · ADR(결정 기록) · DSL 규격 |
| `43.기능정의/` | 기능 목록 |
| `60.테스트/` | 테스트 계획 |

처음 읽을 곳: [CONTEXT.md](CONTEXT.md) → [종합 계획](20.프로젝트관리/PLAN_end-user-테스트툴-종합계획.md) → [ARCH](40.설계/ARCH_end-user-테스트툴.md).
