---
title: ADR - 020 클라우드 DB·대기열·예약
path:  euta-workspace / 40.설계 / ADR - 020 클라우드-DB-대기열-예약
ref: PLAN_euta-cloud 결정 "실행 모델"·"예약"·"실행 운영 규칙"·"동기화·충돌"(2026-10-20) · ARCH_euta-cloud · ADR-019
---

# ADR - 020 클라우드 DB·대기열·예약

## 상태

채택 (2026-10-20, 설계). 구현(C1·C3)에서 검증한다.

## 맥락

- PLAN 은 "상주 러너 + DB 대기열", "서버 내장 스케줄러(DB 잠금으로 한 곳만 등록)", 저장 시 기준 버전 충돌 검사, 실시간 변경 수신을 정했다. 별도 메시지 브로커를 두지 않는 것이 전제다(dev 메모리 한도 4Gi).
- 로컬 앱의 실행 색인은 엔진 SQLite(`runs/index.sqlite`)지만, 클라우드는 여러 프로세스(서버·러너)가 동시에 쓴다.

## 결정

### 1. DB — PostgreSQL 17, 접근은 Drizzle ORM

- PostgreSQL 하나로 메타데이터·버전·실행 색인·대기열·세션·토큰을 모두 담는다.
- 대기열·예약·실시간에 필요한 기능이 DB 안에 있다: `FOR UPDATE SKIP LOCKED`(임대), advisory lock(스케줄러 단일 실행), `LISTEN/NOTIFY`(변경 알림).
- Drizzle: TypeScript 스키마 = 타입, SQL 에 가까운 질의(잠금 절·advisory lock 을 그대로 씀), 마이그레이션 도구(drizzle-kit) 포함, 런타임 엔진 바이너리 없음. 인증 라이브러리(ADR-023)도 Drizzle 어댑터를 지원.
- dev: 클러스터 안 단일 인스턴스 + PVC 5Gi(ceph-rbd). 운영 DB 는 정식화 때.

### 2. 대기열 — 실행 테이블 자체가 대기열

```
실행 상태: 대기 → 임대됨(러너 id, 임대 만료 시각) → 실행 중 → 통과 | 실패 | 시간 초과 | 중단 | 건너뜀
```

- 러너는 `SELECT … WHERE 상태=대기 AND 계정 잠금 없음 ORDER BY 등록 시각 FOR UPDATE SKIP LOCKED LIMIT 1` 로 가져가며 임대를 건다.
- 실행 중 주기적으로(예: 30초) 임대를 연장한다. 서버의 정리 작업이 만료된 임대를 "중단" 으로 바꾼다.
- **같은 테스트 계정 1개** 규칙은 "실행 중인 실행이 쓰는 계정 집합" 과 겹치지 않는 대기 실행만 고르는 조건으로 구현한다(계정 이름은 사전 검사 때 스크립트에서 뽑아 실행 행에 저장).
- 새 실행 등록 시 `NOTIFY` 로 러너를 깨우고, 놓쳐도 러너가 주기적으로 다시 확인한다.

### 3. 예약 — 서버 안 1분 주기 + advisory lock

- 서버가 1분마다 advisory lock 획득을 시도하고, 잡은 한 곳만 "다음 실행 시각 ≤ 지금" 인 예약을 찾아 실행을 등록한다.
- 다음 실행 시각은 cron 식 + Asia/Seoul 로 계산해 예약 행에 저장. 놓친 회차는 몰아 등록하지 않고 "건너뜀" 실행 행을 1건 남긴 뒤 다음 시각으로 넘긴다. 이전 회차가 아직 돌면 "건너뜀".
- cron 계산 라이브러리는 구현 시 선택(시간대 지원 필수).

### 4. 동기화 지원

- 스크립트 버전 테이블: 원문 · 내용 해시 · 이전 버전 · 작성자 · 출처. 저장 요청의 기준 버전 ≠ 현재 버전이면 409 충돌.
- 변경 피드: 프로젝트별 증가 번호(변경 순번)를 붙인 변경 기록 테이블 — SSE 재연결 시 "마지막 순번 이후" 를 돌려준다(ADR-022). 보관 기간은 SPEC 에서.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| Redis + BullMQ 대기열 | 프로세스·메모리 1개 추가, 정본이 둘(DB·Redis)로 갈림. 규모상 DB 대기열로 충분 |
| pg-boss 등 Postgres 대기열 라이브러리 | 가능하나 "같은 계정 1개" 같은 선택 조건을 넣기 어렵고, 실행 테이블과 대기열 테이블이 이중화 |
| SQLite | 서버·러너 다중 프로세스 동시 쓰기, 클러스터 배포에 부적합 |
| Prisma | 런타임 엔진·생성 단계 부담, 잠금 절을 raw SQL 로 우회해야 함 |
| 외부 cron(k8s CronJob) | 클러스터 범위 밖 자원은 아니나 등록 로직이 서버와 갈림. 서버 내장으로 완결 |

## 결과

### C1 서버 1부 — 계정·프로젝트 (2026-10-20)

DB·동기화 지원(§1·§4)을 구현했다. 대기열(§2)·예약(§3)은 다음 작업(실행·러너)에서.

| 항목 | 실제 |
|---|---|
| DB | PostgreSQL 17(`postgres:17-alpine`) · `pg` 8.23 · drizzle-orm 0.45.4 · drizzle-kit 0.31.11. 마이그레이션 `packages/server/drizzle/0000_init.sql` — 서버 시작 시 전용 연결 1개로 적용 |
| 테이블 | Better Auth: `users`(+`status` · `is_admin`) · `sessions` · `accounts` · `verifications` · `apikeys` · `device_codes` / euta: `org_settings` · `login_attempts` · `projects` · `project_members` · `environments` · `credentials` · `scripts` · `script_versions` · `suites` · `activity` · `change_feed` (SQL 이름 snake_case, id 는 text) |
| 스크립트 버전 | `scripts.current_version`(정수) + `script_versions(script_id, version)` 유일. 저장 트랜잭션에서 스크립트 행 `FOR UPDATE` → 기준 버전 비교 → 새 버전 → 200개 넘는 오래된 버전 삭제. 같은 기준 버전 동시 저장 4건 중 1건만 성공(테스트) |
| 살아 있는 경로 유일 | `scripts(project_id, path) WHERE deleted_at IS NULL` 부분 유일 인덱스 — 삭제된 스크립트는 경로를 쥔 채 복원 대기 |
| 변경 피드 | 프로젝트 행의 `feed_seq` 를 같은 트랜잭션에서 `+1` 해 순번 발급(프로젝트 행 잠금이라 커밋 순서 = 순번 순서) → `change_feed(project_id, seq)` 기록 → `pg_notify('euta_feed', '{p,s}')`(커밋 시 전달, 8KB 한도 때문에 순번만). 서버마다 LISTEN 전용 연결 1개, 알림을 받으면 DB 에서 `seq > 마지막 전송` 을 다시 읽어 보냄 |
| 피드 보관 | **7일**(`FEED_RETENTION_DAYS`) — SPEC 에 값이 없어 구현에서 정함. 요청 위치 이후가 지워졌으면 SSE `resync` 이벤트(ADR-022) |
| 작업 로그 | `activity` — 행위자 종류(user·ai·schedule·system)·표시 이름·종류(`script.saved` 등)·대상·요약·details(값 없음). 1년 뒤 삭제 |
| 정리 작업 | 서버 내 1시간 주기 `purgeExpired` — 휴지통 30일 지난 프로젝트(FK cascade 로 전부)·스크립트, 1년 지난 작업 로그, 보관 기간 지난 피드. 시계 주입으로 테스트 |

- 다음 작업을 막지 않게: 실행·예약 테이블은 추가 마이그레이션으로 붙이고, 작업 로그·피드 종류는 문자열이라 `run.*` · `schedule.*` 를 그대로 쓸 수 있다.
- 변경: §3 의 advisory lock·§2 의 `SKIP LOCKED` 는 아직 쓰지 않았다(대기열·예약 구현 때).

### C1 서버 2부 — 실행·러너 대기열 §2 (2026-10-20)

§2 를 구현했다. 레포 `euta-cloud` 커밋 `574df12`~`85d6e1c`. 예약(§3)은 C3.

| 항목 | 실제 |
|---|---|
| 테이블 | 마이그레이션 `0001_runs.sql` — `runs` 1개가 실행 기록이자 대기열. 상태 `queued · leased · running · passed · failed · error · timeout · aborted · cancelled · skipped`, `flaky`(C3 재실행용 자리), 대상(스크립트 id·경로·**고정 버전·내용 해시·원문 사본** — 200개 넘는 옛 버전이 지워져도 실행 가능), 환경 id·이름, `vars` 덮어쓰기, 출처(`web·app·ai_mcp·schedule·local_upload`)·요청자·행위자 종류, `credential_names text[]`, 시간 제한(초), 녹화 값, 임대 러너 id·만료, 실행 위치, 정지 요청(`cancel·timeout`), 단계·지표·expect 결과·요약(jsonb), 결과 파일 상태(`none·present·expired`)·크기. 프로젝트에 `run_time_limit_sec`(기본 21600) 추가 |
| 임대 | 러너는 DB 에 직접 붙지 않는다 — `POST /internal/runner/claim` 을 서버가 처리. 트랜잭션 안에서 `pg_advisory_xact_lock(hashtext('euta_run_claim'))` 로 **임대를 직렬화**한 뒤 `… ORDER BY created_at, order_index FOR UPDATE OF r SKIP LOCKED LIMIT 1`. 이유: "같은 (환경, 테스트 계정) 1개" 는 행을 넘는 조건이라 SKIP LOCKED 만으로는 두 러너가 같은 계정을 쓰는 서로 다른 대기 행을 동시에 집을 수 있다. 임대는 드물어 직렬화 비용이 없다(동시 임대 4건 → 1건만 성공, 테스트) |
| 같은 계정 1개 | 조건 = 대기 행의 `credential_names` 와 실행 중(`leased·running`, 스크립트 실행만) 행의 집합이 같은 환경에서 겹치지 않음(`&&`). 계정을 안 쓰는 실행은 제약 없음. 막힌 실행이 뒤 실행을 막지 않는다(머리 막힘 없음) |
| 스위트 | 부모 행(`kind=suite`, 임대 대상 아님) + 순서(`order_index`)가 있는 자식 스크립트 실행. 앞 형제가 모두 끝나야 다음 자식이 임대된다(실패해도 다음 진행). 부모는 첫 자식 임대 때 `running`, 마지막 자식이 끝나면 정산 — 스위트 취소면 `cancelled`, 모두 통과면 `passed`, 아니면 `failed`(요약 = 자식 상태 개수). 부모의 계정 집합은 표시용이며 동시 실행 판정에서 뺀다 |
| 깨우기 | 실행 등록·종료 트랜잭션에서 `pg_notify('euta_runs')`. 서버의 기존 LISTEN 연결(변경 피드)이 이 채널도 듣고, 임대 요청을 **긴 폴링**(`waitMs` ≤ 25초)으로 붙잡고 있다가 알림이 오면 다시 임대를 시도한다. 놓쳐도 러너가 다시 요청(기본 20초). 러너가 기다리다 끊으면(정지) 그 사이 임대한 실행은 대기로 되돌린다 |
| 임대 연장 | 러너의 `events` 호출마다 만료 = 지금 + `RUN_LEASE_TTL_MS`(2분). 하트비트 5초(SPEC 의 30초에서 취소 응답성 때문에 줄임 — SPEC Change Log) |
| 정리 작업 | 서버 15초 주기 `sweepRuns` — 임대 만료 → `aborted`(남은 부분 업로드 삭제), 시간 제한 초과 → 정지 요청 `timeout`(러너가 다음 응답으로 받아 엔진 종료), 그 뒤 5분 안에 러너가 끝내지 않으면 서버가 `timeout` 으로 닫음. 시계 주입으로 테스트 |
| 임대 때 예외 | 대기 중 환경이 지워진 실행은 임대 시 `error`("environment was deleted"). 휴지통 프로젝트의 대기 실행은 임대하지 않고 남긴다(복원하면 진행, 30일 뒤 영구 삭제와 함께 사라짐) |
| 보관 | 실행 행(요약·단계·지표·expect)은 종료 1년 뒤 `purgeExpired` 가 삭제(스위트 자식은 부모와 함께) |

- 대안 검토: 러너가 DB 에 직접 `LISTEN`·`SKIP LOCKED` — 러너 이미지에 DB 자격증명이 들어가고 SPEC 러너 프로토콜(HTTP)과 갈려 채택하지 않음. 긴 폴링으로 같은 깨우기 효과.
- 남은 한계: 임대를 되돌릴 때(러너가 끊긴 긴 폴링) 이미 `running` 으로 바뀐 스위트 부모는 되돌리지 않는다(다음 임대 때 그대로 진행되므로 표시만 일찍 바뀜).

### C3 서버 — 예약 §3 · 재실행 · 알림 outbox · 하루 실행 상한 (2026-10-21)

§3 을 구현했다. 레포 `euta-cloud` 커밋 `b0bca84`·`75bb36d`. 테스트 172 → 207건.

| 항목 | 실제 |
|---|---|
| cron 라이브러리 | **cron-parser 5.10**(`CronExpressionParser.parse(expr, { tz: 'Asia/Seoul', currentDate })`) — 시간대·DST 처리 내장, 런타임 의존성 1개(luxon). `next()` 는 기준 시각과 같은 회차를 건너뛰고 **엄격히 뒤** 회차를 준다(확인함). 입력은 5필드만: 라이브러리가 받는 6필드(초)·`H`(지터)·`L`·`W`·`#`·`?` 는 서버가 먼저 거절 |
| 최소 간격 | 표본을 몇 개 뽑아 보는 대신 **필드로 정확히 계산**: 하루 안 회차 시각 = 시 × 분 조합의 간격, 날을 넘는 간격 = (첫 시각 + 1440 × 연속 허용 날 최소 차 − 마지막 시각). "연속 허용 날" 은 `0 0 <일> <월> <요일>` 을 4년+1일 훑어 구한다(2월 28일 → 3월 1일 같은 경우 포함, 1~50ms). 2만 회차를 직접 훑는 방식은 1.6초라 기각 |
| 테이블 | 마이그레이션 `0002_schedules.sql` — `schedules`(대상 종류·스크립트/스위트·환경 id, 입력값 `cadence` + 변환된 `cron`, `timezone`, `interval_sec`, 시간 제한, 재실행, 켜짐, `disabled_reason`, `next_run_at`, 만든 사람 id + **이름 사본**), `notifications`(Teams outbox), `user_events`(앱 알림). `runs` 에 `schedule_id · scheduled_for · attempt · retry_of_run_id(부분 유일) · skip_reason · local_modified · device`, `projects` 에 `notify_settings · daily_run_cap` |
| 틱 | 서버마다 60초 `setInterval` → `pg_try_advisory_lock(hashtext('euta_scheduler'))`(세션 잠금, 못 잡으면 이번 틱 건너뜀) → 기한 지난 예약을 **예약마다 별도 트랜잭션**에서 `FOR UPDATE` 로 다시 읽고 `next_run_at ≤ 지금` 을 재확인한 뒤 처리. 두 장치(advisory lock + 행 잠금 재확인)로 서버 2대가 동시에 돌아도 회차당 1건(테스트: 두 인스턴스 6틱 동시 → 1건). 한 트랜잭션에 몰지 않은 이유: 실행 등록·알림이 프로젝트 행 잠금을 잡으므로 예약 여러 건을 한 트랜잭션에 두면 잠금이 길어지고 앱 알림 잠금 순서가 깨진다 |
| 회차 처리 | 다음 시각은 항상 `next(cron, 지금)` 으로 저장(따라잡기 없음). 마지막 기한 회차 = `prev(cron, 지금)`. 지연 허용 `SCHEDULE_MISSED_GRACE_MS` 5분 — 넘으면 놓침 1건만, 안이면 그 회차 실행 + 그보다 앞선 회차가 있었으면 놓침 1건. 겹침 = 같은 예약의 최상위 실행이 `queued·leased·running`(재실행 포함). 회차 때 사전 검사 실패는 `error` 실행으로 남김(대안: 조용히 건너뛰기 — 예약이 왜 안 도는지 안 보여 기각). 예약의 환경이 지워졌으면(FK `set null`) 기본 환경으로 바꿔 돌리지 않고 `error`(다른 대상에 실제 부작용이 생기지 않게). 다시 켜거나 주기를 바꾸면 지금부터 다시 계산 |
| 실행 등록 공용화 | `createRun`(사람 요청)에 묶여 있던 등록 코드를 `insertPlannedRun(ctx, tx, project, plan, origin)` 으로 빼서 예약·재실행이 같은 경로를 쓴다(행위자·요청자·출처·예약 id·회차·시도 번호를 `origin` 으로) |
| 종료 후 처리 | `afterTerminal` 끝에서 최상위 실행만 `onTopLevelTerminal` — 재실행 등록 → 불안정 표시 → 알림 판정(outbox 행) → 앱 알림 행. 모두 종료 트랜잭션 안(HTTP 없음). 재실행은 원래 행(스위트는 자식까지)을 복사(고정 원문 그대로) — 다시 계획하면 그 사이 저장된 새 버전이 섞여 "같은 회차 재시도" 가 아니게 된다 |
| Teams outbox | `sendPendingNotifications` — `status='pending' and next_attempt_at ≤ now` 를 1건씩 `FOR UPDATE SKIP LOCKED` 로 잡고 보낸 뒤 상태 갱신(행 잠금을 쥔 채 최대 10초 HTTP). 깨우기 = `NOTIFY euta_notify`(종료 트랜잭션) + 15초 주기. 서버 재시작에도 남는다 |
| 하루 실행 상한 | 등록 트랜잭션에서 `select … from projects for update` 로 프로젝트 행을 잠근 뒤 오늘(Asia/Seoul 고정 UTC+9 — 1988년 이후 DST 없음) 등록된 스크립트 실행 수를 센다(출처 `local_upload`·상태 `skipped` 제외). 동시 요청이 마지막 1칸을 같이 통과하지 않는다 |

- 고려한 대안: cron 라이브러리 `croner`(의존성 0, 시간대 지원)도 가능했으나 필드 값(`fields.minute.values` 등)을 바로 꺼내 최소 간격을 계산할 수 있는 cron-parser 를 택함. 스케줄러를 트랜잭션 advisory lock 하나로 감싸는 방식은 위 잠금 순서 문제로 기각.
- 남은 한계: 겹침 판정은 같은 예약만 본다(다른 예약이 같은 대상을 돌리는 중이어도 등록 — 같은 계정 1개 규칙이 직렬화). 놓침 판정의 5분은 설정값이 아니다(상수).
