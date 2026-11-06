---
title: ADR - 038 TC 문서 읽기 · 내보내기 구현 (@euta/tc-doc · 격자 · 양식 감지 · xlsx 쓰기)
path:  euta-workspace / 40.설계 / ADR - 038 TC-문서-읽기-내보내기
ref: PLAN 단계 16 · SPEC_TC-문서-변환("가져오기" · "저장" · "내보내기") · SPEC_스크립트-보기-편집(단계 문장) · ADR-001 · ADR-002 · ADR-037
---

# ADR - 038 TC 문서 읽기 · 내보내기 구현

## 상태

채택 (2026-10-26, 단계 16 엔진 TC 문서). euta-engine 새 패키지 `packages/tc-doc` — 커밋 전(사용자 검토 후 로컬 커밋). macOS(arm64)에서 tc-doc 타입체크·빌드·단위 20건으로 검증했고, 받은 고객 문서(블록형 xlsx)를 로컬에서 읽어 TC 13개를 확인했다(레포에는 넣지 않음).

## 맥락

- [SPEC_TC-문서-변환](SPEC_TC-문서-변환.md) 이 가져오기(xlsx · csv · docx · md → 격자 → 양식 감지 → 정리된 TC), 경고 5종, `unknown` 격자 텍스트, 내보내기(xlsx 블록형 · md)와 왕복을 정했다. 이 ADR 은 SPEC 이 정하지 않은 **구현 방식**(패키지 위치, 라이브러리, 감지 규칙 세부)을 기록한다.
- 받은 고객 문서에서 확인한 배치: 표지 · 개정 이력 · 결과 집계 · `TC List`(3행 머리, 분류 세로 병합) · 분류별 시트(블록: B 라벨(B:C 병합) · D 값, E `TC Description`·`Pre-condition` 은 **값이 한 줄 아래**, F `Result`, 단계 머리 `No. │ Procedure(C:D) │ Expected Result │ Remarks`, 끝에 절차가 빈 번호 행). 분류별 시트의 TC ID · 제목 · 분류는 `='TC List'!D4` 같은 **수식**이라 저장된 계산값을 읽어야 한다.

## 결정

### 1. 위치 — core 가 아닌 새 패키지 `@euta/tc-doc`

- SPEC 은 "엔진 core 에 `parseTcDocument`" 라고 했지만, 문서 읽기·쓰기는 실행 엔진과 무관하고 앱(나중에 CLI)만 쓴다. core 를 가볍게 두려고 `packages/tc-doc` 로 뗐다. 의존: `@euta/core`(워크스페이스 — `@euta/core/describe` 서브패스만 씀), `fflate`, `zod`, `yaml`(셋 다 core 가 이미 쓰는 판).
- src 는 `node:` 모듈을 쓰지 않는다(`TextDecoder`·`fflate` 만) — 앱 렌더러에서 번들해도 된다.

### 2. 라이브러리 — 자체 OOXML 최소 구현 (fflate + 작은 XML 읽기)

| 용도 | 고른 것 | 대안 · 버린 이유 |
|---|---|---|
| xlsx 읽기·쓰기 | fflate(zip) + 자체 XML 토크나이저(`xml.ts`, 요소·속성·텍스트만) | **exceljs** — 읽기·쓰기·스타일이 다 되지만 의존이 큼(archiver · unzipper · jszip · saxes · fast-csv · tmp …), 마지막 릴리스 2023 · 설치 경고. **SheetJS** — npm `xlsx` 는 0.18.5 에서 멈췄고 최신은 cdn.sheetjs.com 타르볼로만 받음(잠금 파일·보안 갱신 관리가 번거로움), 커뮤니티판은 스타일(굵게·테두리·배경) 쓰기 불가 |
| docx 읽기 | fflate 로 `word/document.xml` 만 꺼내 같은 XML 읽기 | **mammoth** — HTML 로 바꿔 주는 라이브러리라 표 병합(gridSpan · vMerge) 정보가 사라지고, 다시 HTML 을 파싱해야 함 |
| csv | 자체 RFC 4180 파서(따옴표 · 칸 안 줄바꿈 · `""`), 구분자 `,` `\t` `;` 중 첫 줄에 많은 것 | csv-parse 등 — 30줄 분량이라 의존을 늘릴 이유가 없음 |
| 인코딩 | `TextDecoder('utf-8', { fatal: true })`(BOM 자동 제거) → 실패하면 `TextDecoder('euc-kr')` | iconv-lite — Node 22 · Electron 은 full ICU 라 WHATWG `euc-kr`(= windows-949, CP949 상위집합)을 기본 지원 |

- 필요한 범위만 읽는다: 통합 문서 시트 순서(`workbook.xml` → rels → 대상 경로, 절대·상대 경로 모두), 숨은 시트 제외, 셀 종류 `s`(공유 문자열 — 서식 있는 글자는 `<t>` 를 이어 붙이고 후리가나 `<rPh>` 는 버림) · `inlineStr` · `str`(수식 문자열 결과) · `b` · 숫자, **수식은 `<v>`(저장된 계산값)만** 읽고 `<f>` 는 보지 않음, `_xHHHH_` 이스케이프, `mergeCells`. 날짜는 일련번호 그대로(TC 문서에서 날짜 칸은 읽지 않는 시트에만 있음).
- 쓰기는 공유 문자열 표 + 최소 `styles.xml`(라벨: 굵게·연한 배경·테두리·가운데, 값: 테두리·위 정렬·줄바꿈, 제목: 14pt 굵게) + `sheetFormatPr`(없으면 macOS Quick Look 이 행 높이 0 으로 그려 빈 화면이 됨 — 확인함). openpyxl 로 다시 열어 시트 · 병합 · 굵게가 읽히는 것과 Quick Look 렌더를 확인했다.

### 3. 격자 — 병합 셀은 범위 전체에 왼쪽 위 값

- 모든 형식을 `Grid { name, rows: string[][], merges }` 로 바꾼다. 병합 범위의 **모든 칸에 왼쪽 위 값**을 채운다(목록 시트의 세로 병합 분류를 따로 이어 붙일 필요가 없음). 라벨 찾기·머리 열 찾기는 `merges` 로 "병합 안의 왼쪽 위가 아닌 칸"을 건너뛰어 같은 라벨을 두 번 보지 않는다.
- 이름: xlsx = 시트 이름, csv = 파일 이름, docx · md = `Table n`. 경고 위치는 `<이름>!<셀>`(예: `Editor!E18`).
- 셀 값은 `\r\n` → `\n`, 앞뒤 공백 제거.
- docx: `w:tbl > w:tr > w:tc`(내용 컨트롤 `sdt` 등은 투명하게 통과), 칸 = 문단을 `\n` 으로, `w:br` → `\n`, `w:tab` → 탭, 삭제 글자 제외. `gridSpan` = 가로 병합, `vMerge`(restart 아닌 것) = 위 칸과 세로 병합. 표 밖 문단은 `unknown` 때만 쓴다.
- md: 머리 줄 다음에 `|---|` 구분 줄이 있는 파이프 표만. `\|` → `|`, `<br>` → 줄바꿈.

### 4. 양식 감지 — 시트(표)마다, 표 → 블록 → 목록 순

- 머리·라벨 글자는 소문자 + 공백 · `.:-_()[]` 제거 후 동의어 표로 필드를 정한다(`parse.ts` `SYNONYMS`): ID(`TC ID`·`ID`·`테스트 ID`·`TC 번호`…) · 제목(`TC Title`·`제목`·`테스트명`…) · 분류(`Category`·`분류`·`구분`…) · 설명(`TC Description`·`설명`·`목적`…) · 사전 조건(`Pre-condition`·`사전 조건`·`선행 조건`…) · 화면(`화면 정보`·`Screen`) · 번호(`No.`·`번호`·`순번`) · 절차(`Procedure`·`절차`·`수행 절차`·`테스트 절차`·`Step`·`Action`·`동작`) · 기대 결과(`Expected Result`·`Expected`·`기대 결과`·`예상 결과`) · 무시(`Result`·`Remarks`·`비고`·`Author`·`작성자`·`Item`…). 40자 넘는 칸은 라벨이 아니다.
- **표형**: 한 행에 ID(또는 제목) 열과 절차 열이 함께 있으면 그 행이 머리. 아래 행: ID 가 바뀌면 새 TC, 같은 ID 또는 빈 ID 는 앞 TC 의 단계(ID 열이 없으면 제목으로). 단계 번호는 **차례대로 매긴다** — 표형의 `No.` 열은 행 번호인 경우가 많아 쓰지 않는다(그래서 표형은 번호 경고가 나지 않음). 한 칸에 `1. … 2. …`(`1)` 도) 번호 줄이 2개 이상이면 단계로 나누고(첫 번호 앞 글은 첫 단계에 붙임), 기대 결과 칸도 번호 줄이면 같은 번호끼리 짝짓고 남는 것은 마지막 단계에, 번호가 없으면 통째로 **마지막 단계**에 붙인다. 머리에 `Step` 과 `Action` 이 함께 있고 번호 열이 없으면 `Step` 을 번호 열로 본다.
- **블록형**: ID 열이 없는 단계 머리 행(절차 + 번호 또는 기대 결과)이 있고, 시트 어딘가에 ID 또는 제목 라벨이 있으면. 위에서 아래로 읽으며
  - 라벨 칸(ID · 제목 · 분류 · 설명 · 사전 조건 · 화면)의 값 = 오른쪽으로 라벨 자신의 병합을 건너뛰고 빈 칸은 2개까지 건너뛴 첫 값. 그 칸이 다른 라벨(예: `Pre-condition` 오른쪽의 `Result`)이면 멈추고 **바로 아래 칸**(라벨의 병합 아래)을 값으로 본다. 고객 문서의 `TC Description`(E3 → 값 E4) · `Pre-condition`(E5 → `Result` 에서 멈춤 → 값 E6) 이 이 규칙 하나로 읽힌다.
  - 단계 머리 다음 행 중 번호 칸이 숫자(`1`·`1.`·`1)`)인 행 = 단계. 절차가 빈 번호 행은 버린다. 번호 칸이 비고 절차/기대 결과 칸에 글이 있으면 앞 단계에 줄을 덧붙인다.
  - 단계를 읽은 뒤 라벨 행이 나오거나, 이미 채운 필드의 라벨이 다시 나오면 새 블록.
- **목록 시트**: 절차 열 없이 ID + (제목 · 분류 · 화면) 머리가 있는 표. 같은 ID 의 TC 에 비어 있는 제목 · 분류 · 설명 · 화면을 채우고(블록에 이미 있는 값이 우선), 빈 분류 칸은 위 값을 이어 받는다. 화면 정보는 줄마다 하나(`screens[]`). TC 순서는 **목록 순서**(목록에 없는 TC 는 뒤에 원래 순서로) — 세트 만들기(④)가 문서 순서를 쓴다.
- 그 밖의 시트(표지 · 개정 이력 · 결과 집계)는 시트 이름을 보지 않고 위 조건에 안 맞아 저절로 빠진다.
- 문서 `layout` = 블록 시트에서 TC 가 나왔으면 `block`, 아니면 표에서 나왔으면 `table`, 하나도 없으면 `unknown` + `gridText`. ID 가 없는 TC 는 `TC-001` 식으로 번호를 붙이고, 제목이 없으면 ID 를 제목으로.

### 5. 경고 · 격자 텍스트

- 코드 5종: `no-steps` · `step-number-missing`(1 부터 최대 번호 사이 빠진 번호) · `step-number-duplicate` · `duplicate-id` · `credential`. 메시지는 한국어(미리보기 표시용), 위치는 셀 주소. 비밀번호처럼 보이는 값: `비밀번호`·`패스워드`·`비번`·`암호`·`password`·`pwd`·`pw`·`token`·`api key`·`secret` 뒤에 `:` `=` `은/는` 과 4자 이상 영숫자·기호 값(`${…}` 참조는 제외). **경고 메시지에 값을 넣지 않는다**(필드 이름 · 위치만).
- `gridText`: 표마다 `## <이름>` 다음 `A1=값`(병합은 왼쪽 위 1번, 칸 안 줄바꿈은 `\n` 글자), md · docx 는 `## Text` 아래 본문. 상한 100,000자(`GRID_TEXT_LIMIT`), 넘으면 자르고 `… (truncated, N more characters)`.

### 6. 정리된 TC · 검증

- 타입 `Tc { id, title, category?, description?, precondition?, screens?, steps[{ no, action, expected? }] }`, 빈 선택 필드는 아예 뺀다(`normalizeTc`) — 형식마다 같은 모양이고 왕복 비교가 단순해진다.
- `validateTcs(input)`(AI 추출 `save_tc_extraction` 용): zod 스키마(문자열 trim, `no` 는 숫자 문자열도 받음, id · title · action 은 비면 안 됨) + 같은 ID 중복 오류. 실패하면 `issues[{ path: '[0].steps[1].action', message }]`.
- `groupPreconditions(tcs)`: 공백을 하나로 줄인 문장이 같으면 한 묶음, 처음 나온 순서대로 `pre-1`…, 묶음 글은 처음 나온 원문.

### 7. 내보내기

- `exportTcDocument(doc, format)` 는 `TcDocument { name, tcs, preconditions? }` 를 받아 파일 바이트(`Uint8Array`)를 돌려준다(SPEC 의 `(tcs, format)` 대신 문서 이름을 제목에 쓰려고).
- xlsx: `TC List`(B1 문서 이름 병합 · 3행 머리 `No. │ Category │ TC ID │ TC Title │ 화면 정보 │ Result │ Remarks` · 연속된 같은 분류 세로 병합) + 분류별 시트(분류 없는 TC 는 `TC` 시트, 이름은 31자 · `[]:*?/\` 제거 · 중복 시 번호). 블록은 고객 양식 배치 그대로(Category / TC ID / TC Title 라벨 B:C 병합 · 값 D · `TC Description` E:F 병합과 값 아래 줄 · `Pre-condition` 과 값 아래 줄 · `Result` · 단계 머리 · 절차 C:D 병합), 블록 사이 빈 줄 2개. 작성자(Author) 줄은 값이 없어 라벨 없이 둔다. Result · Remarks 칸은 비운다.
- md: `# 문서 이름` → `## 분류` → `### ID · 제목`, 화면 · 설명 · 사전 조건 줄, `| No. | 절차 | 기대 결과 |` 표(`|` 이스케이프, 줄바꿈 `<br>`). md 는 읽기용이라 다시 가져오면 `unknown` 이다(SPEC 의 왕복 조건은 xlsx 만).
- 왕복: 블록형 픽스처 → 내보내기 → 다시 읽기가 같은 TC(목록 순서 · 분류 · 화면 · 여러 줄 포함), 특수 문자 · 분류 없음 · 빈 필드도 같다(단위 테스트). 고객 문서도 로컬에서 왕복 동일.

### 8. 스크립트 → TC (`scriptsToTcs`)

- 입력 `{ path, source }[]`(YAML 원문). ID = `description` 의 `TC ID: X` 또는 첫 `영문/한글-숫자` 꼴(`NOTE-02`), 없으면 파일 이름(확장자 뺌). 제목 = `name`. `description` 이 ID 와 다르면 설명으로.
- `group` 1개 = TC 단계 1개: 절차 = 묶음 이름에서 앞 번호(`3. `)를 뗀 것, 이름이 없으면 묶음 안 동작 단계 문장들. 기대 결과 = 묶음 안 `expect` · `manual` 단계 문장(줄마다). 묶음 사이에 있는 묶음 밖 단계들은 함께 TC 단계 1개로.
- 묶음이 없으면 동작 단계 1개 = TC 단계 1개, `expect`/`manual` 은 앞 단계의 기대 결과에 붙인다(앞 단계가 없으면 절차 "화면 확인").
- 문장: 단계 `name` 이 있으면 그것, 아니면 `describeStep` 문장(`expect` 는 조건마다 한 줄, "확인: " 머리 뗌). `manual` 은 core 의 문장 생성기 지원과 무관하게 직접 읽는다(문자열 또는 `instruction`).

## 결과

- 단위 `packages/tc-doc/test/tc-doc.test.ts` 20건: 블록형 xlsx(목록 합치기 · 표지/이력/집계 제외 · 여러 줄 · 경고 4종 위치) · 표형 csv(UTF-8 BOM) · csv(CP949) · xlsx(세로 병합 ID) · docx(vMerge) · md(`\|` · `<br>`) · 중복 ID · 비밀번호 · xlsx 셀 종류(수식 계산값 · 서식 글자 · 후리가나 · 절대 경로 · 숨은 시트) · `unknown`(md 본문 · 격자 텍스트 · 상한) · 지원 안 하는 형식 · `validateTcs` · `groupPreconditions` · 왕복 2종 · md · `scriptsToTcs` 2종.
- 합성 픽스처(가상의 메모장 웹 앱) `test/fixtures/`: `block.xlsx` · `table.csv` · `table-cp949.csv` · `table.xlsx` · `table.docx` · `table.md` · `unknown.md`. 바이너리는 openpyxl · python-docx 로 만든다(`make-fixtures.py` — 다시 만들 때만 실행, 우리 쓰기 코드와 다른 도구가 만든 파일로 읽기를 검증).
- 고객 문서(로컬 확인, 레포 미포함): `layout: block`, TC 13개(Policy 2 · Home 8 · Variation 3), 단계 91개, 경고 0건. openpyxl 로 직접 읽은 값과 비교해 12개는 제목 · 설명 · 사전 조건 · 단계(번호 · 절차 · 기대 결과)가 모두 같고, Home-01 은 원본의 `Pre-condition` 라벨 칸이 `\` 로 잘못 적혀 있어 사전 조건을 못 읽었다(미리보기에서 고치는 경우). 사전 조건 묶음 3개. 내보내기 → 다시 읽기 동일.
- 알려진 한계: 구형 `.xls` · `.doc` · 암호 걸린 파일은 읽지 않는다(오류). 날짜 칸은 일련번호로 보인다. 셀 서식 숫자(`1.0`)는 저장된 값 그대로. 블록형에서 라벨 글자가 틀리면(위 `\`) 그 값을 놓친다.
