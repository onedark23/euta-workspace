---
title: ADR - 006 secrets 마스킹 구조
path:  euta-workspace / 40.설계 / ADR - 006 secrets-마스킹
ref: ARCH 보안·개인정보 · PLAN 리스크(결과물에 secrets 유출)
---

# ADR - 006 secrets 마스킹 구조

## 상태

채택 (2026-09-29)

## 맥락

- ARCH: secrets 는 결과물에 남기지 않는다(로그 · steps.jsonl · 리포트 · HAR, 스크린샷의 입력란). PLAN 리스크 표의 대응도 "마스킹을 엔진 필수 기능으로".
- secrets 는 여러 형태로 새어 나간다: 원문, URL 인코딩(쿼리), 폼 인코딩(POST 본문 — `!` 까지 `%21`), JSON 이스케이프, HTML 엔티티, base64(바이너리 응답 본문 · Basic 인증), 그리고 Playwright trace 의 DOM 스냅샷(입력값)·네트워크 본문.

## 결정

- **실행마다 `Redactor` 하나**. 값 등록 시 변형(원문 · encodeURIComponent · `+` 공백 · WHATWG form-urlencoded · encodeURI · JSON 이스케이프(1·2중) · HTML 이스케이프 · base64)을 모두 패턴으로 만들고 `***` 로 바꾼다. 등록 대상: `--secrets` 파일의 모든 값 + 스크립트·세션 스텝이 참조한 `${secrets.*}` 값.
- **단일 관문 = 이벤트 버스.** 모든 이벤트는 발행 시 Redactor 를 통과한다. steps.jsonl · run.log · console.jsonl · report.html · index.sqlite · 앱/MCP 구독자는 마스킹된 이벤트만 받는다. run.json · metrics.json · RunResult 도 저장 전에 한 번 더 통과시킨다.
- 기록되는 스크립트는 참조가 남은 원본(`${secrets.USER_PASSWORD}`)이다. 해석된 값은 실행에만 쓴다.
- **HAR 후처리**: 원본을 임시 파일(`.network.raw.har`)로 받은 뒤 → `Authorization` · `Proxy-Authorization` · `Cookie` · `Set-Cookie` 헤더와 `cookies[]` 값을 `***` 로 → base64 응답 본문은 디코드해 바이트 치환 후 재인코딩 → 전체 텍스트 치환 → `network.har` 저장, 원본은 `finally` 에서 삭제.
- **trace.zip 후처리**: fflate 로 풀어 `.trace` · `.network` 줄 단위 JSON 의 민감 헤더를 가리고, 모든 엔트리에 바이트 치환 후 다시 압축. 원본은 삭제.
- **스크린샷**: `fill`/`type` 값에 `${secrets.*}` 가 들어가면 그 대상을 기억해 이후 모든 스크린샷(스텝 · measure · visual · 세션)에 Playwright `mask` 로 가린다. 스텝 `mask` 도 지원.
- 3자 미만 secret 은 무관한 문자열을 대량으로 가릴 수 있어 마스킹하지 않고 `run.json.redactionWarnings` 에 경고를 남긴다.
- `vars` 안에서 `${secrets.*}` 참조를 금지한다(vars 는 "그대로 기록" 대상).

## 이유

- 소비자가 늘어도(앱 · MCP · 리포트) 한 곳만 지키면 된다.
- trace 는 압축돼 있어 단순 grep 검사로는 새는 것을 못 찾는다 — 엔트리 단위 후처리와 테스트가 필요했다.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| 각 기록 지점에서 개별 마스킹 | 새 소비자가 생길 때마다 누락 위험 |
| HAR `content: 'omit'` | 응답 본문 진단 가치를 잃음. 대신 base64 까지 후처리 |
| trace 비활성화(secrets 사용 시) | 실패 분석 핵심 증거를 잃음 |
| 녹화 중 입력란 스타일 변경으로 가림 | 테스트 대상 화면을 바꿔 시각 비교·동작에 영향 |

## 결과

- E2E 테스트가 실행 폴더 전체(trace.zip 엔트리 포함)와 `index.sqlite` 를 원문 · URL · 폼 인코딩 형태로 grep 해 0건을 확인한다.
- **남은 공백(알려진 한계)**
  - **영상(video.webm)** 은 픽셀이라 가릴 수 없다. 비밀번호 입력란은 브라우저가 가리지만, 텍스트 입력란에 넣은 secret(예: 이메일 계정)과 화면에 다시 표시된 값은 영상·trace 스크린캐스트(JPEG)에 보인다. 민감 계정이면 `record.video: false` 를 권장.
  - 스크린샷 마스킹은 "secret 을 입력한 입력란" 만 가린다. 앱이 그 값을 다른 곳에 표시하면(예: "환영합니다, 이메일") 그 영역은 스텝 `mask` 로 직접 지정해야 한다.
  - 서버가 secret 을 변형(해시 · 다른 인코딩)해 돌려주는 경우는 찾지 못한다.
