---
title: SPEC - 선택자 점검 · 화면 지도 (AI 도구)
path:  euta-workspace / 40.설계 / SPEC - 선택자-점검
ref:
---

# SPEC - 선택자 점검 · 화면 지도 (AI 도구)

> 상태: **구현 완료** (2026-10-23, 단계 12 — 실사용 대상 서비스 수동 점검 1회 남음). 구현 세부: [ADR-030](ADR_030_선택자-점검-구현.md). 결정 정본은 [PLAN 결정 사항](../20.프로젝트관리/PLAN_end-user-테스트툴-종합계획.md#결정-사항) "선택자 점검".
> 선행: [SPEC_test-script-dsl](SPEC_test-script-dsl.md)(대상 지정·`record`) · [ADR-008](ADR_008_MCP-서버-도구-설계.md)(MCP 도구) · [ADR-006](ADR_006_secrets-마스킹.md)(마스킹).

## 목적

- AI 가 스크립트를 **실제 화면 구조를 보고** 쓰고 고치게 한다. 스크린샷만 보고 역할·이름을 짐작하면(버튼으로 보이는 라디오, 원래 잠긴 입력란 등) 실행할 때마다 실패를 하나씩 고치게 된다(2026-10-23 실사용 대상 서비스 TC 작성에서 확인).
- 제공하는 것(사용자 결정):
  - **단계별 DOM 스냅샷** — 실행할 때 단계마다 화면 구조를 결과 폴더에 남긴다(성공 실행 포함).
  - **선택자 점검** — 스크립트의 모든 대상(target)을 실행 기록의 스냅샷에 대 보고, 단계마다 "정확히 1개 / 없음 / 여러 개 / 숨김" 을 알려 준다.
  - **화면 지도** — 실행 중 지나간 화면마다 접근성 트리(역할·이름·상태)와 주소를 돌려준다.
- 쓰는 곳: **AI 도구** — 앱 AI 채팅과 엔진 MCP 서버(공용 도구 핸들러). 앱 화면·CLI 는 범위 밖(후속).
- 범위 밖: 클라우드 원격 MCP 도구(후속 — 같은 엔진 API 를 쓰면 됨), 자동 수정(점검은 알려 주기만, 고치는 것은 AI·사람).

## 사용 흐름 (AI)

```
사용자: "로그인 후 메뉴 TC 만들어줘"
AI:  run_script(탐색용 초안) ──► 실행 (단계마다 DOM 스냅샷 저장)
     get_page_map(runId)      ──► 화면별 역할·이름 지도 (예: /studio/clips 의 radiogroup "클립 상태값" · radio "초안")
     스크립트 작성 · validate_script
     check_script_targets(script, runId) ──► 단계별 ✓ / ✗ 없음 / ⚠ 여러 개 (후보 제시)
     고친 뒤 run_script ──► 결과 확인
```

- 사용자가 실행한 결과(앱 실행 버튼·스위트)도 같은 도구로 점검한다 — 실패한 실행의 "어느 단계 대상이 왜 안 맞았나" 를 AI 가 화면 구조로 설명.

## 단계별 DOM 스냅샷 (엔진)

| 항목 | 규칙 |
|---|---|
| 켜기 | DSL `record.dom` — `true`(기본) · `false` · `on-failure`. `--cloud` 실행 기본값은 `on-failure`(녹화 규칙과 같음, 스크립트 명시값 우선) |
| 시점 | 동작 단계(goto·입력 스텝)는 **동작 전**과 **동작 후(settle 뒤)**, `expect`·`wait` 는 **판정이 끝난 시점** 1장. 그룹은 하위 단계 기준 |
| 범위 | 메인 프레임 DOM(같은 출처 iframe 은 포함, 다른 출처는 자리만). `<script>` 제거. 입력란 값·체크 상태는 속성으로 보존 |
| 화면 모양 | 점검 때 보이는지(visible) 판정이 맞도록 **스타일시트를 함께 저장**한다(같은 출처 CSS 규칙 텍스트, 실행 안에서 내용 해시로 중복 제거). 다른 출처 CSS 는 주소만 |
| 민감 값 | **password 입력란 값은 저장하지 않는다.** 나머지는 실행의 secrets 마스킹(ADR-006)을 거친다 — `${secrets.*}` 로 입력한 값·그 인코딩은 `***` |
| 저장 | 결과 폴더 `dom/` — 단계별 스냅샷 + `dom/styles/` + `dom/index.json`(단계 id · 시점 before/after · 주소 · 제목 · 파일). `run.json` 에 `dom` 요약(개수·용량) |
| 용량 | 한 장 수십~수백 KB. 압축 저장(구현 세부, ADR) — 수 시간짜리 실행은 `on-failure` 권장 |
| 실패 | 스냅샷을 못 떠도 단계 결과에 영향 없음(경고 로그 1줄) |

## 선택자 점검 (엔진 API)

`checkTargets(source, runDir, { vars?, env? }) → { runId, scriptMatchesRun, steps: [...] }`

- **어느 스냅샷에 대 보나**: 동작 단계 = 그 단계의 **동작 전** 스냅샷, `expect`·`wait` = **판정 시점** 스냅샷. 단계 id(`3`, `4.2` …)로 실행 기록과 짝짓는다.
- **스크립트가 실행 후 바뀌었으면**(`run.json.scriptHash` 와 다름) `scriptMatchesRun: false` 와 함께 단계 id 기준으로 최선 점검하고, 짝이 없는 단계는 `no-snapshot`.
- **판정**(엔진 `resolveTarget` 과 같은 규칙, 보이는 요소만 셈):

| 판정 | 뜻 | 대상 |
|---|---|---|
| `ok` | 정확히 1개(동작) · 1개 이상(확인) | |
| `missing` | 0개 | 동작 · `visible` · `text` · `count` · `enabled` · `disabled` · `attribute` · `style` · `wait visible` |
| `ambiguous` | 2개 이상 — 동작은 실행하면 오류 | 동작 (`nth` 없을 때) · `text` · `enabled` · `disabled` · `attribute` · `style`(실행기가 strict 로 읽어 오류) |
| `present` | `hidden` 기대인데 보임 | `hidden` · `wait hidden` |
| `dynamic` | 판정 시점 스냅샷에서 아직 조건 전(예: 클릭 뒤 상태) — 그 단계가 실행에서 통과했으면 `ok` 로 봄 | 확인 · 동작(동작 전 화면에 0개인데 실행에서 통과 — 기다리는 동안 나타남) |
| `no-snapshot` | 그 단계가 실행되지 않음(건너뜀·앞 단계 실패) · 스냅샷 없음 | |

- 각 결과: 단계 id · 단계 문장(`describeStep`) · 대상 · 개수 · 판정 · **후보**(최대 3개 — 같은 화면에서 이름·텍스트가 비슷한 요소를 시연 녹화기의 대상 후보 규칙(role+이름 → 라벨 → placeholder → 텍스트 → testId → CSS)으로 만든 target 과 그 개수).
- `${vars.*}`·`${env.*}` 는 값을 넣어 점검, `${secrets.*}` 가 든 대상은 `skipped`(정의 안 된 참조·형식이 틀린 대상도 `skipped` + 이유).
- "실행에서 통과" 는 같은 단계 id 의 결과가 `passed` 이고 스크립트가 같거나(`scriptHash`) 그 단계 원문이 실행한 스크립트의 같은 단계와 같을 때만 본다 — 다른 스크립트를 점검하면 `dynamic` 이 나오지 않는다.
- `count` 는 실행기처럼 숨은 요소를 포함해 세고, 0개인데 조건이 0을 허용하지 않을 때만 `missing`.

## 화면 지도 (엔진 API)

`pageMap(runDir, { url?, maxScreens? }) → { screens: [{ url, title, h1, stepIds, aria }] }`

- 실행 중 지나간 화면을 **주소(쿼리 제외) + 화면 구조 해시**로 묶어 1개씩. `stepIds` = 그 화면이 보인 단계들.
- `aria` = Playwright 접근성 스냅샷 텍스트(역할·이름·상태 — `[checked]` `[disabled]` `[expanded]` `[selected]` `[pressed]`). 마스킹 적용. 공통 머리말·꼬리말(같은 내용이 모든 화면에 반복)은 첫 화면에만 두고 나머지는 생략 표시.
- 대화상자가 열린 상태는 별도 화면(`url` 같음 + `dialog` 이름)으로.

## AI 도구 (공용 핸들러 — 앱 AI 채팅 · 엔진 MCP)

| 도구 | 입력 | 출력 | 권한 |
|---|---|---|---|
| `get_page_map` | `runId`, `url?`(부분 일치), `maxScreens?`(기본 20) | 화면 목록 + 접근성 트리 | 읽기 |
| `check_script_targets` | `runId`, `script`(본문) 또는 `path`(프로젝트 스크립트), `vars?`·`env?` | 단계별 판정 · 후보 · `scriptMatchesRun` · 요약 · `problems` | 읽기 |

- 둘 다 **읽기 전용** — 앱 AI 채팅 게이트에서 확인 없이 허용(기존 `get_run` 과 같은 등급).
- 결과는 길이 상한(기본 화면 20개(최대 50) · 트리 화면당 400줄 · 점검 결과 150건)을 두고 잘렸으면 표시한다.
- 도구 설명(cheatsheet)에 "스크립트를 쓰기 전에 `get_page_map`, 실행 전에 `check_script_targets`" 순서를 넣는다.
- 결과 폴더 `dom/` 이 없는 옛 실행이면: trace 가 남아 있으면 **trace 의 DOM 스냅샷에서 복원**해 같은 결과를 내고(보조 경로), 없으면 "DOM 스냅샷 없음 — `record.dom` 을 켜고 다시 실행" 을 돌려준다.

## 보안 · 개인정보

- 스냅샷은 실행 결과 폴더 안에만 있고 보관·삭제는 실행과 같다.
- password 값은 저장 단계에서 버린다. secrets 마스킹은 스냅샷·지도·점검 결과 모두에 적용.
- 화면에 보이는 일반 데이터(이름·이메일 등)는 스크린샷과 같은 수준으로 남는다 — AI 도구 결과로 모델에 전달될 수 있다(스크린샷 도구와 같은 전제, 본인 구독·본인 사용).

## 완료 기준 (검증)

- 엔진 단위: 스냅샷 직렬화(스크립트 제거·password 값 제거·마스킹) · 스타일 중복 제거 · 판정 6종 · 후보 생성 · 화면 묶기 · trace 복원.
- 엔진 E2E(픽스처 사이트): 실행 → `dom/` 생성(성공 실행 포함) → 일부러 틀린 스크립트(라디오를 버튼으로·중복 이름·없는 텍스트)에 `ambiguous`·`missing`·후보가 나오고, 후보로 고친 스크립트가 실행 통과. `record.dom: false` 면 `dom/` 없음, `on-failure` 성공이면 버림.
- MCP 스모크: `get_page_map` · `check_script_targets` 호출 → 결과 형식·길이 상한·마스킹(스냅샷 파일 평문 secret 0건).
- 앱: AI 채팅 도구 목록에 두 도구(가짜 에이전트로 호출 1회), 게이트 확인 없이 허용.
- 수동 1회: 실사용 대상 서비스 실제 실행으로 메뉴 TC 를 점검 → 판정이 실제 실행 결과와 일치.

## Change Log

### 2026-10-26 — `expect` 의 `enabled` · `disabled` · `attribute` · `style` 대상도 점검 (DSL 단계 16, ADR-037) — `text` 와 같은 판정(정확히 1개가 `ok`, 0개 `missing`/`dynamic`, 2개 이상 `ambiguous`)

### 2026-10-23 — 구현 완료 (ADR-030) — 엔진 `record.dom`·`checkTargets`·`pageMap`·trace 복원, 공용 도구 2종, 앱 AI 채팅 노출, 플러그인 스킬 안내. 설계와 다른 점: `dynamic` 을 동작 대상에도 적용 · `expect.text` 2개 이상은 `ambiguous` · `count` 는 숨은 요소 포함(실행기와 같음) · "통과" 판단은 같은 단계 원문일 때만 · 점검 결과 상한 150건 · 후보 순서는 이 SPEC 순서(녹화기 자체 순서는 그대로). 실사용 대상 서비스 수동 점검은 남음

### 2026-10-23 — 초안 (단계 12) — 단계별 DOM 스냅샷(`record.dom`) · 선택자 점검 · 화면 지도 · AI 도구 2종. 근거: 실사용 대상 서비스 TC 작성 중 trace 복원으로 한 수작업 점검
