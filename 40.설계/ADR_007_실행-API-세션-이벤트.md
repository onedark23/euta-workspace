---
title: ADR - 007 실행 API · Session · 이벤트 버스 · 입력 드라이버
path:  euta-workspace / 40.설계 / ADR - 007 실행-API-세션-이벤트
ref: ARCH 이벤트 모델 · ARCH 전문 모드(P1 탐색형 도구) · ARCH 입력 드라이버
---

# ADR - 007 실행 API · Session · 이벤트 버스 · 입력 드라이버

## 상태

채택 (2026-09-29). 2단계(MCP) · 4단계(Electron) · 5단계(OS 입력)가 이 API 위에 올라간다.

## 맥락

- MCP 탐색형 도구는 "세션 열기 → 한 스텝씩 실행 → 스냅샷·스크린샷 → YAML 로 내보내기 → 닫기" 가 필요하다(ARCH P1).
- Electron 앱은 엔진을 같은 프로세스 계열에 내장해 이벤트를 실시간으로 구독한다.
- 스크립트 실행과 탐색 세션이 서로 다른 실행기를 가지면 동작이 어긋난다.

## 결정

**하나의 스텝 실행기(`StepExecutor`) 위에 `Session` 을 두고, `runScript` 는 Session 을 이용해 만든다.** 스텝 파이프라인은 `before(드라이버 선택) → act → settle → measure(스크린샷·응답 시간) → expect`.

### 공개 API (`@euta/core`)

```ts
// 스크립트 실행 — 즉시 반환, 결과는 Promise
function runScript(input: string | { path?: string; source?: string }, opts?: BaseRunOptions): RunHandle;
interface RunHandle { runId: string; runDir: string; events: EventBus; result: Promise<RunResult> }

interface BaseRunOptions {                // ScriptInputs 포함: env, vars, secrets, run
  env?: Record<string, string | undefined>; vars?: Record<string, string | number | boolean>;
  secrets?: Record<string, string>;
  runsDir?: string;                       // 기본 ./runs (index.sqlite 위치)
  baselineDir?: string; headless?: boolean; browserEngine?: 'chromium' | 'firefox' | 'webkit';
  inputDrivers?: { os?: InputDriverFactory };
  sqlite?: SqliteOpener;                  // 기본 node:sqlite (ADR-003)
}

// 탐색 세션
function openSession(opts?: SessionOptions): Promise<Session>;   // = Session.open
interface SessionOptions extends BaseRunOptions {
  name?: string; description?: string; baseUrl?: string; browser?: BrowserConfig; record?: RecordConfig;
  defaults?: { timeout?: number | string; input?: 'browser' | 'os'; settle?: SettleMode }; scriptDir?: string;
}
class Session {
  readonly runId: string; readonly runDir: string; readonly events: EventBus;
  readonly page: Page;                    // 라이브 뷰 등 호스트용
  readonly steps: readonly ExecutedStep[]; readonly isClosed: boolean;
  step(step: unknown): Promise<StepResult>;          // DSL 스텝 1개. 검증 실패 시 ScriptValidationError (실행 안 함). 호출은 순서대로 직렬화
  screenshot(o?: { name?: string; fullPage?: boolean }): Promise<{ path: string; relativePath: string }>;
  snapshot(): Promise<string>;                        // ARIA 스냅샷(YAML 텍스트, 마스킹)
  exportScript(o?: { name?: string; includeFailed?: boolean }): Script;
  exportYaml(o?: { name?: string; includeFailed?: boolean }): string;  // 통과한 스텝만, ${secrets.*} 참조 유지
  close(): Promise<RunResult>;                        // 멱등. trace/HAR 정리·run.json·report·index 기록
}

// 검증·스키마
function validateScript(source: string | object, inputs?: ScriptInputs): ValidationIssue[];
function prepareScript(source, inputs): PrepareResult;       // raw(참조 유지) + resolved(해석) + secretValues
function getJsonSchema(): Record<string, unknown>;

// 저장소 질의
class RunIndex {
  static open(runsRoot: string, opener?: SqliteOpener): RunIndex;
  static with<T>(runsRoot: string, fn: (idx: RunIndex) => T, opener?: SqliteOpener): T;
  listRuns(q?: { limit?: number; status?: string; name?: string }): RunRecord[];   // 최신순
  findRun(idOrPart: string): RunRecord | undefined;       // 전체 id 또는 유일한 앞/뒤 일부(short id)
  getRun(idOrPart: string): { run: RunRecord; steps: StepRecord[]; metrics: MetricRecord[] } | undefined;
  metricHistory(name: string, limit?: number): (MetricRecord & { runName: string })[];
  close(): void;
}
function regenerateReport(runDir: string): string;
function installBrowsers(browsers?: string[], o?: { withDeps?: boolean }): Promise<number>;
```

### 이벤트 버스

- `EventBus`: `on(type, fn)` · `onAny(fn, { replay })` · `events()` · `for await (const e of bus)`(처음부터 재생, `close()` 시 종료) · `isClosed`.
- 봉투: `{ type, runId, seq, ts, ...payload }`. 모든 이벤트는 발행 시 마스킹(ADR-006).
- 타입: `run.started`(kind · script · env · runDir) · `run.finished`(status · summary · durationMs) · `step.started`(stepId · index · action · name · target · step(raw)) · `step.finished`(status · durationMs · error · optional) · `artifact.saved`(kind: screenshot|diff|baseline|video|trace|har|download|report, path(실행 폴더 기준 상대)) · `metric.recorded`(name · value · unit · stepId) · `console.message` · `network.failed` · `expect.evaluated`(condition · expected · actual · passed) · `log`(phase: input|progress|save → run.log 에서 입력·진행·저장).
- `stepId` 는 1부터 시작하는 계층 id(`"3"`, `"4.2"`, trigger 는 `"N.1"`), `index` 는 실행 순번.
- `runScript` 는 실제 작업을 다음 tick 에 시작하므로 반환 직후 구독하면 누락이 없다. 늦게 붙어도 `onAny(fn, { replay: true })`.

### 입력 드라이버

```ts
interface InputDriver {
  readonly kind: 'browser' | 'os';
  click(t: Locator, o?: ClickOptions): Promise<void>; dblclick(t): …; hover(t): …; tap(t): …;
  fill(t, value: string): …; type(t, text: string, delayMs?: number): …; press(key: string, t?: Locator): …;
  select(t, o: { values?: string[]; labels?: string[] }): …; setChecked(t, checked: boolean): …;
  upload(t, files: string[]): …; scroll(o: { target?: Locator; to?: 'top' | 'bottom'; by?: { x?: number; y?: number } }): …;
  drag(from: Locator, to: Locator): …; dispose?(): Promise<void>;
}
type InputDriverFactory = (ctx: { page: Page; headless: boolean }) => InputDriver | Promise<InputDriver>;
```

- 대상은 Playwright `Locator` 로 넘긴다 — OS 어댑터가 `boundingBox()` 로 좌표를 계산할 수 있다.
- `input: os` 스텝은 `inputDrivers.os` 로 라우팅되고, 없으면 `OsInputUnavailableError`("OS input adapter not installed") 로 스텝 `error`. 5단계는 이 팩토리를 구현해 주입하면 된다(헤드풀·직렬 강제는 그때 추가). → 구현: ADR-013 (`@euta/input-os`).

## 이유

- 실행기를 하나로 두면 MCP 에서 AI 가 한 조작과 그 조작을 내보낸 YAML 의 재실행 결과가 같다(E2E 테스트로 왕복 확인).
- 이벤트를 파일(steps.jsonl)과 실시간 구독이 같은 스키마로 공유해, 실행 뒤에도 같은 타임라인을 재생·리포트 재생성할 수 있다.
- 세션도 실행 폴더·색인을 남겨 AI 탐색의 증거가 사람이 보는 이력과 같은 곳에 쌓인다(`kind: session`).

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| Node `EventEmitter` 그대로 노출 | 타입 안전성·재생·마스킹 관문이 없음 |
| 스크립트 실행기와 세션 실행기 분리 | 동작 불일치 위험 |
| RxJS Observable | 의존 추가 대비 이득 작음. AsyncIterable 로 충분 |
| 세션은 실행 폴더를 남기지 않음 | AI 탐색 증거가 사라지고 리포트를 못 봄 |

## 결과

- MCP 서버는 `openSession/step/snapshot/screenshot/exportYaml/close` 와 `runScript/validateScript/RunIndex` 를 도구로 감싸면 된다.
- Electron 앱은 메인 프로세스에서 core 를 불러 `events` 를 IPC 로 렌더러에 전달하고, `RunIndex` 로 이력을 보여 준다. `runsDir` 는 앱 데이터 폴더로 지정.
- 한 Session 은 한 페이지만 다룬다 — 팝업·새 탭 전환은 1단계 범위 밖.
