---
title: ADR - 004 결과물 PDF 텍스트 추출 (unpdf)
path:  euta-workspace / 40.설계 / ADR - 004 PDF-텍스트-추출
ref: ARCH 측정·분석(결과물 파일) · SPEC expectDownload
---

# ADR - 004 결과물 PDF 텍스트 추출 (unpdf)

## 상태

채택 (2026-09-29)

## 맥락

- `expectDownload.contains` 는 내려받은 파일 안의 텍스트를 확인한다. SaaS 의 "보고서 내보내기" 결과물은 PDF 가 많다.
- 엔진은 macOS · Windows 와 Electron 안에서 추가 설치 없이 돌아야 한다 — 네이티브 바이너리·시스템 도구 의존은 피한다.

## 결정

- PDF 는 **unpdf**(`getDocumentProxy` + `extractText({ mergePages: true })`)로 추출한다. PDF 여부는 확장자가 아니라 시그니처(`%PDF-`)로 판단한다.
- 텍스트 형식(csv · tsv · txt · json · xml · html · md · yaml · ics 등, 또는 앞 8KB 에 NUL 이 없는 파일)은 UTF-8 로 읽는다.
- 비교는 양쪽 공백을 정규화한 뒤 부분 문자열 포함 여부로 한다.
- CSV 행 수는 따옴표(쉼표·줄바꿈 포함 필드)를 처리하는 자체 카운터로 센다(헤더 제외).
- 모듈은 필요할 때만 동적 import 한다.

## 이유

- unpdf 는 PDF.js 의 서버리스 빌드를 포함한 순수 JS 라 워커·캔버스·네이티브 설정이 필요 없고, 현재도 유지보수된다.
- CSV 는 행 수만 필요해 파서 의존성을 추가할 이유가 없다.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| pdf-parse | 오래된 PDF.js 에 묶여 있고 유지보수가 뜸함 |
| pdfjs-dist 직접 사용 | Node 에서 워커·폰트 경로 설정이 필요 — unpdf 가 이를 감싼 것 |
| poppler `pdftotext` | 시스템 바이너리 설치 필요(Windows 배포 부담) |
| csv-parse | 행 수 세기에는 과함 |

## 결과

- 표준 폰트(Type1)·텍스트 레이어가 있는 PDF 는 추출된다. **스캔 이미지 PDF(OCR 필요)와 폰트 매핑이 없는 일부 CJK 임베딩 PDF 는 텍스트가 안 나올 수 있다** — 이 경우 `download.text` 검증 실패로 드러난다. OCR 은 범위 밖.
- 오피스 문서(xlsx · docx) 텍스트 추출은 미지원 — 필요해지면 별도 ADR.
