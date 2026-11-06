---
title: ADR - 002 DSL 스키마 검증 라이브러리
path:  euta-workspace / 40.설계 / ADR - 002 DSL-스키마-검증-라이브러리
ref: SPEC_test-script-dsl · PLAN 결정 사항(YAML DSL + JSON Schema)
---

# ADR - 002 DSL 스키마 검증 라이브러리

## 상태

채택 (2026-09-29)

## 맥락

- 스크립트는 YAML, 검증은 JSON Schema 로 하기로 했다(PLAN). 편집기(yaml-language-server)와 AI(MCP)가 쓸 **JSON Schema 파일**을 엔진이 내보내야 한다.
- 실행 전 검증 오류는 사람이 바로 고칠 수 있게 **정확한 위치**(`steps[3].click.target`)와 이유를 보여야 한다.
- 코드에서는 검증된 값을 TS 타입으로 다루고 싶다.

## 결정

- **zod 4** 로 스키마를 정의하고, **`z.toJSONSchema()`**(draft-07, `io: input`)로 JSON Schema 를 생성한다. 주요 정의는 `.meta({ id })` 로 이름을 붙인다(`Step`, `Step_click`, `Target`, `Duration`, `Size` …).
- YAML 파싱은 **`yaml`**(eemeli/yaml) 의 `parseDocument` — 문법 오류를 위치와 함께 받는다.
- 스텝 검증은 **액션 키를 먼저 고른 뒤 그 액션의 스키마로만 검사**한다. 거대한 union 에 넣으면 "invalid input" 만 나오기 때문. `group.steps` · `expectDownload.trigger` 는 검증기가 직접 재귀해 하위 경로를 정확히 보고한다.
- 모든 객체는 strict(정의된 키만 허용) — 공통 옵션을 인자 안에 넣는 실수·오타를 잡는다.
- 변수 해석(`${...}`)은 **검증 전에** 적용한다(`timeout: ${vars.t}` 허용). 기록·내보내기는 참조가 남은 원본(raw)을 쓴다.
- YAML 파일에서 스키마 연결: `# yaml-language-server: $schema=<경로>/euta-script.schema.json`.

## 이유

- zod 4 는 JSON Schema 생성을 공식 지원(외부 변환기 불필요)하고, 같은 정의로 런타임 검증·오류 경로·TS 타입을 얻는다 — 스키마 정의가 한 곳.
- draft-07 은 yaml-language-server · 대부분 편집기·AI 도구가 가장 넓게 지원한다.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| 손으로 쓴 JSON Schema + ajv | 스키마와 TS 타입을 따로 관리해야 하고, ajv 오류 메시지를 사람이 읽기 좋게 다듬는 비용 |
| TypeBox | JSON Schema 가 1급이지만 커스텀 규칙(대상 키는 정확히 하나 등)과 오류 메시지 다듬기가 zod 보다 번거로움 |
| valibot | 가볍지만 JSON Schema 생성이 별도 패키지·기능 범위 확인 필요 |
| zod 3 + zod-to-json-schema | zod 4 가 내장 기능으로 대체 |

## 결과

- `getJsonSchema()` · `euta schema` · `@euta/core/schema.json`(빌드 시 생성, 커밋). 단위 테스트가 커밋된 파일과 생성 결과가 같은지 확인한다.
- `superRefine`/`refine` 규칙(대상 키 1개, `wait` 는 ms/for 중 하나 등)은 JSON Schema 에 표현되지 않는다 — 편집기는 구조만, 최종 판정은 엔진(`euta validate`).
- 스키마를 바꾸면 SPEC 과 함께 갱신한다(SPEC Change Log).
