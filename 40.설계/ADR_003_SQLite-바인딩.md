---
title: ADR - 003 SQLite 바인딩 (node:sqlite)
path:  euta-workspace / 40.설계 / ADR - 003 SQLite-바인딩
ref: PLAN 결정 사항(결과 저장: 실행별 폴더 + SQLite 색인) · PLAN 리스크(네이티브 모듈 Electron 재빌드)
---

# ADR - 003 SQLite 바인딩 (node:sqlite)

## 상태

채택 (2026-09-29). 4단계(Electron) 착수 시 실제 앱에서 재확인.

## 맥락

- `runs/index.sqlite`(runs · steps · metrics)를 CLI 와 **같은 core 를 내장한 Electron 앱**이 함께 쓴다.
- PLAN 리스크: 네이티브 모듈(pty · SQLite · OS 입력)은 Electron 버전마다 재빌드가 필요해 앱 단계를 늦춘다.
- 확인한 사실(2026-09-29):
  - `node:sqlite` — Node 22.5 추가, **22.13 · 23.4 에서 플래그 없이 사용 가능**(아직 실험 경고), **25.7 에서 Release Candidate(1.2)**. 출처: nodejs.org/api/sqlite.html 이력 표.
  - Electron 최신 안정판 44.4.5 는 **Node 24.21.0** 포함(43.x 도 24.21, 42.x 는 24.19). 출처: releases.electronjs.org.
  - Electron 36.7.3 릴리스 노트: "Fixed an issue where `require('node:sqlite')` didn't work" (electron/electron#47755) — 이후 버전의 메인 프로세스에서 `node:sqlite` 사용 가능.
  - 로컬 Node 22.18 에서 동작 확인(ExperimentalWarning 출력).

## 결정

- 기본 바인딩은 **Node 내장 `node:sqlite`(`DatabaseSync`)**.
- core 는 최소 인터페이스(`SqliteDatabase`: `exec` · `prepare().run/get/all` · `close`)와 `SqliteOpener` 로 감싸고, 실행 옵션 `sqlite` 로 다른 구현(예: `better-sqlite3`)을 주입할 수 있게 둔다.
- Node 22 · 24 공통 API 만 쓴다(생성자 `timeout` 옵션(24.14+), `statement.close()`(26.8+) 등은 쓰지 않음). 잠금 대기는 `PRAGMA busy_timeout`, 저널은 WAL.
- `node:sqlite` 는 **지연 로딩**(`createRequire`)한다 — 색인을 쓰지 않는 사용처에서 실험 경고가 나지 않게. CLI 는 이 경고 하나만 걸러 낸다.
- DB 는 실행 시작·종료 때 열고 바로 닫는다(여러 프로세스 — CLI · 앱 · MCP — 가 같은 파일을 쓰므로 장시간 점유하지 않음).
- 엔진 요구 Node ≥ 22.13.

## 이유

- 네이티브 애드온이 없으므로 Electron 업그레이드 때 `@electron/rebuild` 가 필요 없고, macOS · Windows 배포 파이프라인이 단순해진다.
- 필요한 기능(동기 쿼리 · 트랜잭션)이 단순해 실험 단계 API 변동의 영향 범위가 작고, 어댑터로 격리돼 있다.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| better-sqlite3 | 성숙·빠르지만 네이티브 모듈 — Electron ABI 마다 재빌드, Windows 빌드 도구 의존. 어댑터로 교체 가능하게만 둠 |
| sqlite3(node-sqlite3) | 네이티브 + 비동기 콜백 API, 유지보수 활발도 낮음 |
| sql.js(WASM) | 재빌드 없음. 하지만 파일 DB 를 메모리로 읽고 통째로 다시 써야 해 여러 프로세스 동시 사용에 부적합 |
| JSON 파일 색인 | 이력 검색·추이 질의가 느림(PLAN 에서 이미 제외) |

## 결과

- `RunIndex.open/with`, `listRuns` · `findRun`(전체 id 또는 앞/뒤 일부) · `getRun` · `metricHistory`.
- 위험: `node:sqlite` 가 아직 실험(Node 22/24)이라 경고가 출력되고 API 가 바뀔 수 있다 → 어댑터 뒤에 두었고, 4단계에서 Electron 44+ 메인 프로세스 스모크 테스트로 확인한다. 문제가 있으면 앱 쪽에서 `better-sqlite3` opener 를 주입한다.
