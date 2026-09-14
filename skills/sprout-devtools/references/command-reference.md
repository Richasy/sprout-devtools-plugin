# Command reference

`sprout-devtools <command> [args] [options]`. The installed build's `--help` remains the parser authority. This file
lists every supported **public** command and long option in the current source; guest/elevated worker commands are
identified separately and are not public automation contracts.

## Global options

The root parser accepts these recursively. A command consumes only the ones that apply to its output:

| Option | Effect |
|---|---|
| `--json` | Emit machine-readable output where that command supports a JSON form. |
| `--artifacts <dir>` | Evidence root for application check commands; default `.sprout-devtools/<runId>`. |
| `--verbose`, `-v` | Diagnostic logging on stderr. |

Pass an option value beginning with `--` by using `=`: `--app-arg=--some-flag`.

## Start with the workflow, not a manual build-and-launch

| Goal | Command |
|---|---|
| Build Debug and inspect one window | `debug <project> --from-source` |
| Build and run the app's checks | `run <project>` (project input infers `--from-source`) |
| Keep a launched process alive | `drive|capture|record|gate … --keep-open` |
| Keep files or package registration resident | `deploy <app|project>` |
| Run the framework-dependent DevTools host from source | `dotnet run --project tools\Sprout.DevTools\Sprout.DevTools.csproj -- <verb>` |

`build` creates a RID-specific publish. It is self-contained/AOT only when the project declares that deployment model.
`--self-contained true|false` explicitly overrides that choice; omitted keeps the project's model.
Do not assume the resulting `.exe` can run without its matching runtime.

### `test <app>`

Runs a self-contained Microsoft.Testing.Platform executable with TRX reporting
inside an explicitly selected VM. Required: `--target vm` and
`--target-tool-dir <self-contained-tools>`. Optional: `--filter`, `--timeout`,
`--queue-wait-seconds`, `--pool`, `--target-state-dir`, `--pool-state-dir`,
`--existing-vm-profile`, `--artifacts`, and `--json`.

There is no host/local/direct/auto route. Busy guests queue. A positive executed
and passed test count, zero adverse outcomes, exit zero, held-process cleanup,
and guest rollback are required. Stdout, stderr, TRX, and the published-payload
manifest are hash-bound. Empty or all-skipped selections cannot pass.
`--target-state-dir` selects the localhost broker pipe, journal, and artifacts.
`--pool-state-dir` independently selects an existing managed Hyper-V pool
manifest root; omission preserves the prior behavior and reads the pool from
the target-host root. The separate root requires explicit VM routing, cannot be
combined with `--existing-vm-profile`, and is echoed in the receipt and final
`run.host.targetPoolStateDirectory`.

For an exact local DevTools payload that must not replace a foreign resident
target host, use the additive direct-isolation form:

```powershell
sprout-devtools isolate <tests.exe> --portable-test --backend vm `
  --devtools <self-contained-tools> --pool default `
  --filter 'FullyQualifiedName~WindowTests'
```

It queues on the global per-VM lease and retains the same strict test,
process-cleanup, service-cleanup, and checkpoint-rollback requirements.
The requested `--timeout` remains the exact test-process limit; separate
bounded worker connection, cleanup/report, and host-receive reserves preserve
the terminal evidence.

Use `selftest` only for the Sprout PASS/leak token protocol, never as a wrapper
that fabricates tokens for another test runner.

### `process-test <app>`

Runs an arbitrary native or console test executable in an explicitly selected
Hyper-V guest. Required: `--target vm`, `--target-tool-dir
<self-contained-tools>`, and `--success-token <literal>`. Optional:
`--test-arg`, `--timeout`, `--queue-wait-seconds`, `--pool`,
`--target-state-dir`, `--pool-state-dir`, `--existing-vm-profile`,
`--artifacts`, and `--json`.

There is no auto, Sandbox, local, direct, or host fallback. The token is a
1–512-character literal matched against complete stdout lines with ordinal
comparison; it cannot contain leading/trailing whitespace or control
characters.
Pass requires exactly one stdout match, zero stderr matches, exit code zero
before the exact requested timeout, stdout/stderr within 1,048,576 characters
each, a locked full-payload manifest, retained PID/creation
FILETIME/executable SHA-256 exit proof, service cleanup, checkpoint restore,
and final VM power-off. A token printed before a crash, nonzero exit, timeout,
or cancellation does not pass. `--timeout` is the exact process deadline and
accepts 1–1800 seconds.

`--test-arg` is repeatable, capped at 64 values / 4096 characters each /
16384 characters combined, and uses `ProcessStartInfo.ArgumentList` rather
than shell interpolation. Use `--test-arg=--name` for option-shaped values.
The managed-pool and existing-VM routing rules match explicit VM `selftest` and
`test`: a private `--target-state-dir` may use a separate authorized
`--pool-state-dir`, while `--existing-vm-profile` is mutually exclusive with
an explicit pool.

The single check id is `process-test`. `failed` means the executable completed
with a wrong exit/token/output verdict; `errored` means launch, capture,
payload identity, result I/O, cancellation, cleanup, or rollback could not be
proven. Hash-bound evidence includes stdout, stderr, and
`process-test-payload.json`. Never accept the token without the
`processCleanup` and target cleanup receipts. Raw guest schema and every
run/check/step/assertion status must be present string values; missing, null,
numeric, or unknown values fail before typed deserialization. The guest run
ID, artifact root, completed job ID/status, and result path must bind to the
exact inner job receipt.

## Application checks

### `doctor`

Reports OS/build, DPI awareness, isolation, Developer Mode, and launch guidance. `doctor --json` writes a `HostInfo`
object to stdout. It does not write an orchestration `result.json`.

Add `--gpu-adapters` to include `gpuAdapters`: a fresh DXGI inventory with `status`, optional failure `diagnostic`,
and adapter description, canonical LUID, software flag, vendor ID, and device ID. Inventory failure returns exit 2.
Match a measured engine's LUID to this inventory; enumeration order is not a PDH physical-adapter index and does not
prove which adapter the app uses. The command does not create a rendering device or change GPU preference.

### `experiment`

`--pid` For a no-application desktop control, `experiment --pid <dwm-pid> --metrics-only` reuses the same
process identity, sampler, interval weighting, and result contract without opening UIA or requiring a window.
An explicit empty `--plan` remains accepted for compatibility; launching with `--app` or declaring an action is rejected.
The report mode is `metrics-only`. This read-only attachment never terminates the process. Use it to distinguish
desktop compositor load from an idle application's raster counters rather than attributing all DWM work to that app.

Owned experiments also retain `windowConditions`, `lastWindowConditions`, and `windowConditionChecks`. Every process
sample checks the window before and after collection: HWND/root identity, physical client position/size, DPI,
visibility, minimization, and whether foreground belongs to the app. Hidden/minimized windows or a changed condition
abort the run rather than presenting reduced activity as a benefit. Foreground ownership includes the app's own
popups. Occlusion remains explicitly unknown; this is not proof that no other window overlaps the target.

Attaches to an existing positive PID without the tool launching, activating or foregrounding, resizing, pixel/window
capturing, closing, terminating, or otherwise taking lifecycle ownership of the target, and without requiring the
Sprout diagnostics endpoint. Plans reject every synthetic `SendInput` operation. Allowlisted trusted UIA pattern actions
remain available: application-defined `Invoke`, `Select`, `Expand`, and related patterns may intentionally navigate,
open or close an application window, or request application lifecycle behavior. The contract constrains the tool's own
orchestration, not arbitrary behavior implemented by the target's accessibility patterns. Plans also reject `snapshot`:
experiment artifacts are sanitized process metrics only and never persist UIA titles, values, or URLs.

```powershell
sprout-devtools experiment --pid 1234 --plan experiment-plan.json `
  --sample-ms 200 --warmup-ms 5000 --baseline-ms 10000 `
  --action-ms 10000 --cooldown-ms 5000 --require-gpu `
  --artifacts out\experiment --json
```

For the common scroll-performance case, omit the external plan entirely:

```powershell
sprout-devtools experiment --pid 1234 `
  --scroll-automation-id catalog.scroll `
  --scroll-axis vertical --scroll-mode page --scroll-steps 10 --scroll-cycles 2 `
  --action-ms 15000 --runtime-metrics --require-gpu `
  --artifacts out\scroll-experiment --json
```

The preset builds one in-memory allowlisted sweep. Both modes first position the target at `--scroll-start`.
`--scroll-mode page` then uses UIA `LargeIncrement`/`LargeDecrement` in the direction of `--scroll-end`, which is the
normal steady-scroll profile; `percent` uses normalized positions through `--scroll-end` and back, which intentionally
stresses long scrollbar-like jumps. Waits use absolute deadlines from the measurement boundary, so provider-call time
shortens the following wait instead of pushing tail transitions outside the sampled action window. The preset reserves
one provider-call interval for closure and fails if the complete sweep still exceeds `--action-ms`; completion grace is
only timeout headroom, never an accepted unmeasured tail. The AutomationId is used only by the live UIA query and is not
copied into the sanitized experiment result.

| Option | Meaning |
|---|---|
| `--pid <positive>` | Existing target process id. |
| `--app <exe>` | Mutually exclusive with `--pid`: launch one unpackaged self-contained executable and require strict cleanup. Projects, directories, package layouts, installed package images, and framework-dependent apphosts are refused. |
| `--app-arg <value>` | Repeat once per argument in owned mode; use `--app-arg=--option`. |
| `--redact-app-args` | Replace owned-mode recorded arguments with `<redacted>` markers. |
| `--allow-background` | Owned-mode diagnostic that ignores foreground ownership changes only; HWND/root, physical geometry, DPI, visibility, and minimization remain strict, and `timing.allowBackground=true` records the weaker condition. |
| `--timeout-ms <1..3600000>` | Owned worker deadline, including readiness, setup, all phases and teardown; default 120000. Independent worker/process cleanup reserves are additional. |
| `--setup-script <path>` | Owned-mode script before warmup; adds only `resizeClient` to the action allowlist and may begin with that global operation instead of `find`. |
| `--setup-focus-automation-id <id>` | Owned-mode in-memory `find` + `setFocus` + `expectFocused` before warmup; mutually exclusive with `--setup-script`. |
| `--setup-focus-occurrence <0..10000>` | Zero-based match for the inline focus target; default `0`. |
| `--setup-invoke-automation-id <id>` | Owned-mode in-memory `find` + `invoke` before warmup; mutually exclusive with `--setup-script` and the focus preset. |
| `--setup-invoke-occurrence <0..10000>` | Zero-based match for the inline invoke target; default `0`. |
| `--setup-timeout-ms <n>` | Setup action deadline, default 10000; must fit the owned total deadline. Explicit find/wait budgets are checked before launch. |
| `--find-timeout-ms <1..60000>` | Readiness/per-find deadline, default 5000; applies consistently to plan validation, setup budgets, UIA preparation, and worker execution. |
| `--runtime-metrics` | Secure Local runtime counters enabled before warmup, with raw full before/after envelopes for warmup, baseline, every action, and cooldown. Each read has a 5000 ms deadline. |
| `--frame-timing` | After preparation/setup, bind WGC compositor timestamps to the stable final HWND without surface readback or video encoding; fail on later HWND replacement and summarize actual frames/gaps per phase. |
| `--frame-target-fps <1..240>` | Reference cadence used for gap and estimated-missed-frame statistics; default `60`. |
| `--teardown-script <path>` | Owned-mode UIA script after measurement, before process cleanup. No implicit close or flush acknowledgement. |
| `--plan <path>` | `sprout.devtools.experiment.plan.v1` JSON plan. Mutually exclusive with `--scroll-automation-id`; optional with `--metrics-only`. |
| `--scroll-automation-id <id>` | Build a single in-memory scroll-sweep action instead of reading `--plan`. |
| `--scroll-axis vertical\|horizontal` | Sweep axis; default `vertical`. |
| `--scroll-mode percent\|page` | `percent` jumps through normalized positions; `page` uses large UIA increments/decrements for steady scrolling. Default `percent`. |
| `--scroll-start <0..100>`, `--scroll-end <0..100>` | Sweep endpoints; defaults `0` and `100`, and they must differ. Page mode positions at start and uses the endpoint ordering to choose its forward direction. |
| `--scroll-steps <1..100>` | Equal increments in each direction; default `10`. |
| `--scroll-cycles <1..20>` | Forward-and-back repetitions; default `1`. |
| `--scroll-occurrence <0..10000>` | Zero-based match when the AutomationId is repeated; default `0`. |
| `--sample-ms <10..60000>` | Absolute monotonic sample cadence; default 200. |
| `--warmup-ms <n>` | Warm-up duration excluded from statistics; default 5000. |
| `--baseline-ms <n>` | Baseline duration; default 10000. |
| `--action-ms <n>` | Default duration of each action window; default 10000. |
| `--cooldown-ms <n>` | Cooldown duration; default 5000. |
| `--require-gpu` | Error when baseline, any action, or cooldown lacks a fully valid target GPU sample. Off by default for portable diagnostics. |
| `--observe-gpu-pid <positive>` | Repeat once per read-only secondary PID, at most eight distinct observers. Share the target's PDH collection; never action targets or lifecycle-owned processes. |

Owned mode emits JSON even without `--json`; target stdout/stderr are discarded rather than recorded. It locks the
executable before launch and reuses strict drive's held process-generation/image identity and cleanup receipt. Only
measurement and UIA execute in the bounded worker, so a hung provider does not strand the parent waiting for COM.
`processCleanup` preserves typed PID, Int64 creation FILETIME, and the verified executable SHA-256, including cleanup
failure. Before identity binding succeeds the hash is unknown, not fabricated. `processTermination` distinguishes
`alreadyExited` from `terminationRequested`; neither proves a flush.

The optional setup script uses `--setup-timeout-ms`; teardown keeps its 10-second action budget within the worker deadline.
Setup may explicitly `setFocus` through UIA before warmup, for a user-authorized foreground-controlled measurement.
Focus remains forbidden in measured actions and teardown. Choose a focusable element outside pause-on-focus animation
regions, and require `windowConditions.foregroundOwned` plus unchanged cadence; a successful focus request alone is
not proof of an equivalent workload. No synthetic keyboard or pointer input is admitted.
Both reject declared find/wait budgets that exceed their deadline, rather than silently shortening those finds.
Setup-only `resizeClient` uses the existing exact physical-client verification and retains its `window` assertions in
`setup.windowAssertions` and the result check; it adds no synthetic input and records no UIA title/value.
After successful teardown the parent waits up to five additional seconds for natural exit, then uses strict cleanup
if the app is still alive. Teardown is attempted after a failed setup when the UIA connection was prepared; a hard worker timeout/cancellation
can prevent teardown entirely. The parent still runs strict process cleanup. No descendant, redirected single-instance
target, or replacement PID is adopted. A completed worker report is persisted only after cleanup; worker loss yields
an errored report, not synthetic samples. `applicationFlush` is always `unknown`: no generic terminal/flush protocol
exists here, and passing an Invoke-based teardown does not prove durable application data. Attach mode stays non-owning.
`--gpu-trace` also supports owned mode. The parent starts its separately named WPR session after binding the
application identity and before starting the measurement worker, then stops that session before process cleanup.
WPR setup and teardown have their own existing bounded tool deadlines, outside `--timeout-ms`. A trace failure
fails the experiment while preserving strict held-process cleanup. Trace runs are a separate diagnostic workload,
not interchangeable with non-trace samples; the machine-wide ETL is local-only and is never PID-filtered.

With `--runtime-metrics`, the target must expose the secure Local `performance.metrics.v1` capability. Discovery waits
within the first read's deadline; ambiguity, generation/session replacement, missing capability, malformed response, or
timeout errors the experiment. The first successful `performance.metrics.read` enables runtime collection before
warmup; it does not reset counters already enabled by another observer. Every phase gets full before/after protocol
envelopes in `experiment.json`'s `runtimeMetrics`, bound by that artifact's SHA-256. Each receipt carries typed PID,
Int64 creation FILETIME, diagnostics instance GUID, UTC and monotonic read-start/read-completion timestamps and clock
frequency. Identity is checked around every secure Local read. Counter scopes and unknown/future body fields are
preserved verbatim: process counters are not a sum of window counters, and unknown backing coverage stays unknown.
No derived runtime aggregate is emitted. Collection overhead is outside the PDH/CPU phase window; a new prime follows
each before-read so inspection latency does not contaminate that phase's scheduled samples. Partial read failures
retain earlier receipts and a structured error; a killed worker still cannot supply an incomplete result.

Before the first discarded PDH prime and warm-up sample, the command verifies the pinned process and main HWND, creates
one attached UIA client/reconnecting root, and performs a read-only provider initialization. `experiment.json` records
the `preparation` outcome and `result.json` records a `prepare` step. A preparation failure still writes partial errored
artifacts and runs no action. Provider/session allocations therefore happen before warm-up sampling and are excluded
from baseline statistics.

`preparation.diagnostic` records the UIA preparation stage, elapsed milliseconds, exception type/HRESULT,
last connection exception type/HRESULT, whether a nonzero window was observed, window replacement count, and connection
attempt count. It never records exception messages, window handles, titles, names, URLs, or help text. A later deadline
preserves the last connection failure separately. Setup retains this UIA diagnostic; its own result remains under `setup`.
This is diagnostic evidence, not a readiness workaround: an errored preparation with no samples remains errored.

Each action has a unique sanitized `label`, optional `windowMs`, optional nonnegative `completionGraceMs`, optional
boolean `measureAfterInitialFind`, optional boolean `paceWaitsToWindow`, and `steps`. The grace extends only the action
execution deadline after the fixed measurement window; it is not sampled. When the measurement gate is true, the
initial required `find` finishes before the action phase starts and the second step waits for that phase boundary.
`paceWaitsToWindow` requires that gate, rebases the provider deadline there, and treats successive waits as cumulative
deadlines from the boundary; the action fails if it has not completed when the window closes. An action must begin with
its own `find`; state
does not carry between actions. Allowed operations are `find`, `findWithin`, `invoke`, `toggle`, `setValue`,
`setRangeValue`, `expand`, `collapse`, `select`, `showContextMenu`, `scroll`, `setScrollPercent`, every `expect*`,
`waitUntilEnabled`, and `wait`. Snapshot, synthetic input, focus changes, drag, and resize in performance actions are rejected before
process or UI Automation access. The prepared client and root remain command-owned and are reused across all actions;
`DriveExecutor.Run` still resets the logical current element for each action. Every action refreshes the UIA transaction
deadline, cancellation, and pinned process/window checks. A replaced main HWND reconnects only after the same-PID guard,
disposing the stale local root wrapper. The command releases all remaining local UIA/client wrappers after the final
cooldown/verification boundary or on error; attached mode never closes or terminates the target. Descendant queries include the
PID condition and re-read `CurrentProcessId` before accepting a match;
each mutating pattern re-reads it again immediately before the provider call. A mismatch is never invoked. Ordinary
`drive` remains unconfined so hosted cross-process UIA surfaces such as WebView continue to work.

JSON `find` and `findWithin` steps accept an optional zero-based non-negative integer `occurrence` (default `0`).
Selection uses deterministic UIA Control View preorder after all named criteria are applied. The finder polls until at
least `occurrence + 1` same-PID matches exist or the action deadline expires. Foreign hosted-process subtrees are
skipped before their names or automation ids are inspected. Find result details contain only the query shape, observed
match count, and selected/requested index; attached experiment artifacts never persist matched element names or
automation ids.

```json
{
  "schema": "sprout.devtools.experiment.plan.v1",
  "actions": [{
    "label": "select-connector-2",
    "steps": [
      { "op": "find", "automationId": "sidebar.connector-list" },
      { "op": "findWithin", "controlType": "ListItem", "occurrence": 2 },
      { "op": "select" }
    ]
  }]
}
```

The command writes the ordinary `result.json` plus `experiment.json` and invariant-culture `experiment.csv`. Each tick
labels process and GPU readings with one monotonic sample timestamp and records collection duration plus start/end
lateness. Process fields are Working Set, Private Bytes, Peak Working Set, handle count, thread count, cumulative CPU
time, and interval CPU utilization. The cumulative endpoint remains available in each raw sample; phase summaries use
only adjacent readings wholly inside the same phase, normalize their CPU-time delta by wall duration and logical
processor count, and weight the mean by that interval duration.
Direct PDH sampling reads the English `GPU Engine(*)\Utilization Percentage`,
`GPU Process Memory(*)\Dedicated Usage`, and `GPU Process Memory(*)\Shared Usage` wildcard arrays, then retains only
instances whose invariant `pid_<decimal>` token exactly equals `--pid`. GPU busy is the maximum unique target engine;
engine sum may exceed 100 percent. Dedicated and shared bytes are summed once per unique target adapter instance. A
bounded engine list uses canonical LUID/physical-adapter/engine-index/type identities and never includes another PID.
When more than 16 engines are present, the list retains the 16 highest-utilization engines, with canonical identity
breaking ties. The returned rows remain identity-ordered. This applies independently to the target and every observed PID;
aggregate counters still use every unique engine. An absent row in this bounded list is not evidence of zero activity.

#### Read-only GPU observers

Use `--observe-gpu-pid 2345 --observe-gpu-pid 3456` to measure, for example, an application's helper and its
same-session compositor together. The option works with `--pid`, `--metrics-only` and `--app`. Selection order is
preserved. The command rejects more than eight PIDs, duplicates (not coalesced), nonpositive/non-Int32 values,
the target PID and the executing DevTools process PID. Owned mode rechecks the launched PID in the parent and worker.
One integer is required per occurrence; comma-separated lists and unflagged additional values are not accepted.
These checks happen before observer attachment; an owned target-PID collision is detected after launch but before
measurement, with normal exact-process cleanup.

Each accessible observer retains the existing limited-query/synchronize process handle, native creation FILETIME and
image identity. The held identity is verified before and after **one shared PDH collection**; all target/observer
readings use those same wildcard arrays, query generation, native timestamps and monotonic phase endpoints. No
observer is a UIA target, launched, adopted for cleanup or terminated. Disposing observer leases only closes handles.
The target application's own behavior is outside that guarantee: it may independently stop one of its helpers.

The `sprout.devtools.experiment.v1` artifact remains additive and null-omitting:

| Scope | New shape |
|---|---|
| Report | `observedGpuPids: [pid, ...]`; `observedGpus: [{ pid, creationFileTime?, gpu: { availability, reason?, availableSamples, partialSamples, unavailableSamples, queryRebuilds } }, ...]`. |
| Sample | `observedGpus: [{ pid, creationFileTime?, gpuBusyPct?, dedicatedGpuBytes?, sharedGpuBytes?, gpuIntervalStartTimestamp100Ns?, gpuIntervalEndTimestamp100Ns?, gpuIntervalDurationMs?, availability, diagnostic?, queryGeneration, engines: [...], engineReadings: [...] }, ...]`. |
| Phase summary | `observedGpus: [{ pid, creationFileTime?, availability, reason?, metrics: [...] }, ...]`. Each entry uses the existing three `observedGpuBusyPct`, `observedDedicatedGpuBytes`, `observedSharedGpuBytes` metric names and ordinary metric-summary shape. |
| Engine reading | `{ id, luid, physicalAdapter, engineIndex, engineType, availability: "measured"|"unavailable", utilizationPct?, diagnostic? }`. Existing `engines` still contains only valid measured rows. |

`creationFileTime` is the held process's native Int64 creation FILETIME, not a sampling timestamp; it remains after
exit and is absent when no identity could be acquired. Report and phase `availability` retain
`available`/`partial`/`unavailable` metric-coverage semantics. Phase GPU-busy means and baseline deltas retain the shared
interval-duration weighting. Every phase masks unprimed/out-of-phase observer values, including engine readings.
The new engine list keeps at most 16 canonical identities per observer, preferring measured/high-utilization rows
and using identity to break ties; returned rows are identity-ordered. Named invalid/nonfinite/negative counter
values are `unavailable`, not zero. An engine with no returned instance has no row; missing or truncated rows do not
prove inactivity or permit inference of an adapter total. Busy remains the maximum valid unique engine, not a sum.

Legacy `observedGpuPid`, report/sample `observedGpu`, phase `observedGpuAvailability`/`observedGpuReason`, and the three
flat `observed*` metrics continue to describe only the **first requested observer**, with unchanged field meanings.
The new ordered lists are emitted even for one observer. The single-observer CSV keeps its existing columns; with
two or more, one JSON-valued `observedGpus` column is appended after all existing columns (including optional
`deviceTelemetry`). With no observers, all observer fields, metrics and CSV columns remain absent.

An observer that cannot be opened starts with `observerOpenFailed` (or another explicit identity diagnostic), empty
engine lists and omitted numeric values. Exit or identity loss before/during a collection permanently latches
`observerExited`/`observerIdentityMismatch`/`observerIdentityUnavailable`: that collection and all later samples omit
its GPU values, even if the PID is reused. Missing counter instances produce `observerGpuInstancesUnavailable` or
the corresponding PDH diagnostic; counter availability may recover while the same identity remains alive. Query
open/collect failure is reported independently for all observers. Identity failures take precedence over a shared
PDH/phase diagnostic. Unlike the former fail-fast single-observer behavior, observer identity loss does **not** abort
the target or remaining observers. `--require-gpu` still gates only the target. Availability is evidence at collection
boundaries, not a claim that an observer stayed alive after its final sample.

None of these readings is a group/adapter total, present rate, frame timing or energy measurement. Never sum
independent engine or process percentages, and never attribute a whole-device NVML reading to a selected PID.

Warm-up primes PDH and has no summary. Baseline, each action, and cooldown report valid-value mean/p50/p95/peak plus
delta from baseline mean for process metrics, GPU busy, engine sum, dedicated bytes, and shared bytes. CPU utilization
means are weighted by `cpuIntervalDurationMs`; GPU busy and engine-sum means, including their baseline deltas, are
weighted by `gpuIntervalDurationMs`. Their p50/p95 values retain the ordinary sample nearest-rank contract.
Dedicated/shared bytes are endpoint gauges and retain ordinary unweighted sample statistics. PDH rate
counters commonly make the first collection invalid; adapters and engine instances can also appear or disappear. The
sampler accepts `PDH_CSTATUS_VALID_DATA` and `PDH_CSTATUS_NEW_DATA`, reports unavailable/partial data with diagnostics,
and rebuilds a persistently invalid query after a bounded threshold. Missing GPU values stay null/empty, never zero,
and do not invalidate otherwise valid process metrics unless `--require-gpu` is set. `result.json` identifies the app
as `mode: "attached"` with its PID and does not publish the executable path or plan element queries.

Baseline, every action label, and cooldown require at least one valid sample; otherwise the run is errored while
retaining partial samples and artifacts. Required phase durations, including an action's `windowMs`, must be at least
`sample-ms`. With `--require-gpu`, each required phase also needs at least one sample where all four GPU metrics are
valid; a partial GPU sample is diagnostic, not acceptance evidence.

PDH GPU counters depend on the Windows display-driver stack and may be absent in remote, virtualized, software-only, or
restricted sessions. The command never falls back to machine-wide GPU totals and never reports unavailable GPU data as
0 percent. Collection remains on the absolute cadence; if process/PDH collection starts before a terminal deadline,
that endpoint must finish before the nominal phase end. Phase completion itself is different: one terminal collection
starts at or after the nominal end while the ending phase/action remains active. Its wake/start lateness tolerance is
`min(sample-ms / 4, 250 ms)` with integer milliseconds and a 1 ms floor.
`ExperimentTiming.boundaryLatenessToleranceMs` records the tolerance; the terminal sample records
`nominalPhaseEndMs`, the PDH collection timestamp as `actualPhaseEndMs`, `boundaryStartLatenessMs`, and
`phaseEndLatenessMs`. A start beyond tolerance is retained as `sampleOutOfPhase`, counts full missed cadences, and
cannot satisfy coverage; the runner does not wait another cadence or otherwise self-extend. Process endpoint gauges and
the GPU interval come from that same collection. An action is canceled only after its terminal collection, then fully
awaited; cooldown next performs a fresh discarded PDH prime and starts at that collection timestamp, so no
action-execution interval can enter cooldown GPU statistics.

Each synchronous UIA provider call sets the OS transaction timeout from the current remaining action deadline and checks
cancellation/deadline again after return. A mutation may have occurred when a non-cooperative provider returns late, but
the action is reported `actionTimedOut` and no later plan step executes. After the cooldown delay, the command performs
one non-recording process liveness/creation-time/image verification before success; an exit or identity change in the
final cadence gap returns the partial artifacts with `targetExited` or `targetIdentityMismatch`.

### `selftest <app>`

Runs the app's headless token convention (`=> PASS`, `leaked=0`, exit 0).

| Option | Meaning |
|---|---|
| `--selftest-arg <value>` | Trigger argument; default `--selftest`. |
| `--expect-token <name>` | Require a named feature token to report `name=ok`; repeatable. |
| `--timeout <seconds>` | Child self-test timeout; default 120. |
| `--target auto|vm|sandbox|local|direct` | Execution target; `auto` is default and `direct` bypasses the shared host. |
| `--target-state-dir <dir>` | Local target-host pipe, journal, job artifact, and cached-payload root. |
| `--pool-state-dir <dir>` | Existing managed Hyper-V pool manifest root, independent of the broker root; requires explicit `--target vm` and rejects `--existing-vm-profile`. Omission uses `--target-state-dir`. |
| `--queue-wait-seconds <seconds>` | Bound target selection, queueing, and execution; default 600. |
| `--target-tool-dir <dir>` | Self-contained DevTools payload for an isolated target. |
| `--pool <name>` | Managed Hyper-V pool for auto/vm routing; default `default`. |
| `--existing-vm-profile <json>` | Explicit caller-owned Hyper-V targets with immutable VM/checkpoint IDs and credential references; requires `--target vm` and rejects explicit `--pool`. |

Feature names start with an ASCII letter and may contain letters, digits, underscores, and hyphens. Required tokens
match the complete name: `window-hide=ok` satisfies `--expect-token window-hide`, never `--expect-token hide`.
A missing, bad, or skipped required token still fails the gate.

Target-routed result metadata includes the requested target, pool, durable job id, child timeout, and total queue-wait
bound. Managed-pool jobs also retain `targetPoolStateDirectory`. `--target vm` failures retain those fields and the
terminal error without implying that localhost ran.
After submission timeout/cancellation, the client requests job cancellation and waits separately for cleanup.
`run.host.targetJobTerminal` and `targetJobCleanupVerified` must both be true before run-owned credential/profile
cleanup. `targetJobId`, `targetJobStatus`, `targetCancellationRequested`, and `targetCleanupState` retain the recovery
context when cleanup is unproven. A passed identity/provenance check cannot promote a skipped operational run.

### `capture [<app|publishDir|layoutDir>] [--pid <positive>]`

With an app argument, launches the selected form and writes a post-compositor PNG exactly as before. With `--pid`,
attaches to the existing process and captures its current main HWND without relaunching, registering a package,
activating/foregrounding the window, injecting input, resizing, or terminating it. Attach mode pins the process handle,
creation time, and image path; verifies the selected HWND still belongs to that exact process before and after capture;
and never falls back to the desktop or another process's window.
An explicit `--window-title` or `--hwnd` selects a visible owned popup/tool window too, without changing the default
main-window heuristic or adding action authority.

| Option | Meaning |
|---|---|
| `--pid <positive>` | Existing process to capture. Omit the app argument. |
| `--window-title <exact-title>` | With `--pid`, select the unique visible top-level window with this exact case-sensitive title. Mutually exclusive with `--hwnd`. |
| `--hwnd <decimal-or-0x-hex>` | With `--pid`, select this positive visible top-level HWND owned by the held process. Mutually exclusive with `--window-title`. |
| `--backend Auto|WindowsGraphicsCapture|PrintWindow` | Select capture backend; default `Auto`. Explicit `WindowsGraphicsCapture` fails instead of falling back. |
| `--app-arg <value>` | Repeatable app argument. |
| `--redact-app-args` | Launch with the arguments but omit their values from result metadata and emitted errors. |
| `--require-artifact <id=relative-path>` | Require and SHA-256-bind a bounded file already written under the artifact root. |
| `--crash-metadata` | On an early process exit, write bounded sanitized Windows crash metadata as `crash-metadata.json`; no dump, raw event XML/message, path, or value-bearing stack is retained. |
| `--wait-ms <n>` | Bound default main-window discovery; explicit selectors resolve immediately. |
| `--settle-ms <n>` | Delay after the window appears. |
| `--keep-open` | Leave the launched process (and development registration when applicable) alive. |
| `--packaged[:true|false]` | Force packaged/unpackaged; omission auto-detects. |

Attach mode rejects an app argument, `--app-arg`, `--packaged`, and `--keep-open` before opening the target process.
Its `result.json` reports `run.app.mode: "attached"` and `run.app.pid`; successful evidence includes `window.png` and
`window-capture.json` (dimensions, uniform-image status, backend, and secondary-window status). Use a fresh artifacts
directory: attached capture publishes only after both files are staged and identity is verified, and it never
overwrites existing final evidence.
Explicit selection is checked before/after acquisition and again before publishing the staged evidence. A changed
handle/owner, lost process generation, hidden/missing match, or ambiguity fails closed; it never retargets a retry.
Explicit captures add `selectedWindow` identity metadata. The existing `secondaryWindows` flag still describes
secondary-window compositing, not whether the capture target itself is an owned popup.

### `record <app|publishDir>`

Records bounded post-compositor H.264/MP4 evidence.

| Option | Meaning |
|---|---|
| `--form exe|msix` | Launch form; default `exe`. |
| `--app-arg <value>` | Repeatable app argument. |
| `--exe <name>` | Executable within a publish directory. |
| `--duration-ms <500..60000>` | Recording duration; default 5000. |
| `--fps <1..60>` | Sampling ceiling; default 30. |
| `--first-frame-timeout-ms <n>` | Bound first non-uniform frame acquisition. |
| `--wait-ms <n>` | Bound main-window discovery. |
| `--settle-ms <n>` | Delay before recording. |
| `--allow-static` | Permit intentional non-uniform but visually static output. |
| `--keep-open` | Leave an unpackaged app running. |
| `--identity-name <name>` | Synthetic package identity name. |
| `--publisher <dn>` | Synthetic package publisher distinguished name. |
| `--install register|signed` | Packaged install mode. |
| `--msix <path>` | Signed package input. |
| `--cert <path>` | Signing certificate input. |
| `--drive-script <path>` | Run a drive plan after the first valid frame. |
| `--drive-find-timeout-ms <n>` | Bound each drive-script find. |
| `--drive-settle-ms <n>` | Additional UIA settle delay. |
| `--drive-snapshot-depth <n>` | Bound post-drive UIA evidence depth. |
| `--drive-no-capture` | Skip drive screenshot/tree evidence. |

Existing signed inputs always use the name, publisher, version, and application ID from their bounded MSIX manifest,
including when `--msix` is explicit. Synthetic identity options do not rename those bytes. Invalid application
manifests or a declared executable that differs from the requested app are rejected before installation.

### `gate <app>`

Captures a frame sequence while injecting a real touch gesture. Run only in an interactive isolated guest.

| Option | Meaning |
|---|---|
| `--app-arg <value>` | Repeatable app argument. |
| `--wait-ms <n>` | Bound main-window discovery. |
| `--settle-ms <n>` | Delay before capture. |
| `--duration-ms <n>` | Frame-sequence duration. |
| `--min-frame-interval-ms <n>` | Minimum accepted frame interval. |
| `--gesture drag|drag-hold|repeated-interrupt` | Gesture program. |
| `--axis vertical|horizontal` | Gesture axis. |
| `--gesture-from-x <percent>`, `--gesture-from-y <percent>` | Start point. |
| `--gesture-to-x <percent>`, `--gesture-to-y <percent>` | End point. |
| `--hold-ms <n>` | Hold before motion. |
| `--max-static-frames <n>` | Frozen-run threshold. |
| `--max-timestamp-gap-ms <n>` | Maximum accepted compositor gap. |
| `--blank-color <RRGGBB>` | Uncovered-band sentinel. |
| `--blank-color-tolerance <n>` | Sentinel channel tolerance. |
| `--min-blank-band-px <n>` | Minimum uncovered component width. |
| `--roi-inset-x-percent <n>` | Horizontal analysis inset. |
| `--roi-inset-top-percent <n>` | Top analysis inset. |
| `--roi-inset-bottom-percent <n>` | Bottom analysis inset. |
| `--target-events` | Require target-reported gesture/stall events. |
| `--require-target-stall` | Require an observed target stall. |
| `--stall-expect none|freeze|continue` | Expected presentation during the stall. |
| `--expect-presentation none|ui-thread|off-thread` | Required presentation tier. |
| `--keep-open` | Leave the app running after the gate. |

### `drive <app|publishDir|layoutDir>`

Finds UI Automation elements, acts, and asserts. Script mode supports the broader stateful input and `resizeClient`
surface described in [interaction.md](interaction.md).

**Input/form/evidence**

| Option | Meaning |
|---|---|
| `--app-arg <value>` | Repeatable app argument. |
| `--script <path>` | JSON drive plan instead of inline single-step flags. |
| `--wait-ms <n>` | Bound main-window discovery. |
| `--find-timeout-ms <n>` | Bound each find/readiness poll. |
| `--settle-ms <n>` | Delay before UIA connection. |
| `--no-capture` | Skip post-action screenshot/tree evidence. |
| `--capture-target main|owned-popup` | Select screenshot target. |
| `--snapshot-depth <n>` | Bound evidence depth only. |
| `--keep-open` | Leave app/eligible development registration alive. |
| `--require-cleanup` | Require held-process exit evidence and signed exact-package/certificate cleanup before emitting the result. Executable/signed MSIX only; rejects `--keep-open`, loose registration, and synthetic-input plans. |
| `--timeout-ms <positive>` | Whole strict interaction-worker deadline, default 60000; requires `--require-cleanup`. Worker/app cleanup have separate five-second reserves; signed setup (snapshot/install/image lookup) shares 90 seconds, and rollback has its own 90 seconds, each plus five seconds for termination/output drain. |
| `--packaged[:true|false]` | Force form; omission auto-detects. |
| `--install register|signed` | Packaged session mode. |
| `--msix <path>`, `--cert <path>` | Signed package inputs. |
| `--dependency-package <path>` | Explicit signed framework `.msix`/`.appx` (repeatable, maximum 8, 512 MiB each/1 GiB total). Requires `--packaged --install signed --require-cleanup`; no discovery or framework-certificate import. |
| `--preserve-app-data` | Preserve data for the eligible loose-register path. |

**Find and nested find**

`--find-name`, `--find-automation-id`, `--find-type`, `--find-within-name`,
`--find-within-automation-id`, `--find-within-type`.

In JSON script mode, `find` and `findWithin` also accept `occurrence`, a zero-based non-negative integer that defaults
to `0`. Matches are ordered by deterministic UIA Control View preorder, and the find poll is not limited by snapshot
depth/node evidence caps:

```json
[
  { "op": "find", "automationId": "sidebar.connector-list" },
  { "op": "findWithin", "controlType": "ListItem", "occurrence": 2 },
  { "op": "select" }
]
```

**UIA actions**

`--invoke`, `--toggle`, `--set-value`, `--set-range-value`, `--expand`, `--collapse`, `--select`,
`--show-context-menu`, `--scroll-horizontal`, `--scroll-vertical`, `--set-scroll-horizontal`,
`--set-scroll-vertical`.

**Assertions**

`--expect-toggle`, `--expect-value`, `--expect-range-value`, `--expect-enabled`,
`--expect-expand-collapse`, `--expect-selected`, `--expect-scroll-horizontal`, `--expect-scroll-vertical`,
`--expect-text`, `--expect-selected-text`.

**Synthetic pointer/keyboard actions**

`--type-text`, `--click`, `--hover`, `--hover-x`, `--hover-y`, `--drag-to-name`,
`--drag-to-automation-id`, `--drag-to-type`, `--drag-hold-ms`.

These synthetic actions use global desktop input. Never run them on the developer desktop; use `isolate` or
`drive-shell`.

### `build <project>`

Publishes one project to a RID-specific output. Project properties decide whether that output is framework-dependent,
self-contained, or NativeAOT.

| Option | Meaning |
|---|---|
| `--configuration <name>`, `-c` | Configuration; default `Release`. |
| `--runtime <rid>`, `-r` | RID; default `win-x64`. |
| `--output <dir>`, `-o` | Publish directory; default `<artifacts>\publish`. |
| `--dotnet <path>` | Explicit dotnet executable; otherwise resolved from the running/pinned runtime. |
| `--timeout-ms <n>` | Publish timeout; default 600000. |

### `produce <publishDir>`

Reuses the app's package when present or produces a synthetic MSIX.

`--exe`, `--identity-name`, `--display-name`, `--publisher`, `--version`, `--no-sign`,
`--synthetic-msix`, `--timeout-ms`.

### `run <app|publishDir|project>`

Builds when the input is a project, then runs self-test/capture/package/parity checks.

`--form auto|exe|msix|both`, `--demo-arg`, `--selftest-arg`, `--exe`, `--identity-name`, `--publisher`,
`--install register|signed`, `--msix`, `--cert`, `--timeout`, `--wait-ms`, `--settle-ms`, `--no-selftest`,
`--from-source`, `--configuration`/`-c`, `--runtime`/`-r`, `--dotnet`, `--build-timeout-ms`.

### `deploy <app|publishDir|project>`

Leaves files or package registration resident. Its ordinary verification launch exits; a reusable Developer-Mode
loose package may explicitly retain the exact launched process with `--keep-open`.

`--form auto|exe|msix|both`, `--from-source`, `--configuration`/`-c`, `--runtime`/`-r`, `--dotnet`,
`--build-timeout-ms`, `--deploy-dir`, `--package-dir`, `--exe`, `--identity-name`, `--display-name`, `--publisher`,
`--version`, `--synthetic-msix`, `--app-arg`, `--no-launch`, `--keep-open`, `--wait-ms`, `--settle-ms`,
`--timeout-ms`.

For a build-produced loose AppX layout, `--package-dir <stable-dir>` selects the resident layout used across repeated
same-version updates. The source and resident directories must be separate. The update holds the package identity
lease, refuses non-development or ambiguous registrations and active package/layout processes, stages and hashes the
new files before swapping the stable layout, and never uninstalls the prior package. Post-swap failures restore and
re-register the exact previous layout or remove only a proven newly created development registration; incomplete
recovery reports retained paths. `--keep-open` leaves only this command's PID/generation/package/image-verified
AUMID-launched process running and records its executable SHA-256; `--no-launch` and `--keep-open` are mutually
exclusive. Both options are rejected for signed MSIX deployment. Omit `--form` for a real loose layout;
`--form msix` requests the synthetic package route. See
[`devtools-package-updates.md`](../../../../docs/guide/devtools-package-updates.md).

### `debug <app|publishDir|project>`

Builds Debug only when `--from-source` is supplied, launches, captures a PNG and UIA tree, then exits. Its current
`windowCaptured` assertion proves only that non-zero pixels were captured; inspect the tree/image because an error
dialog can satisfy it.

`--from-source`, `--configuration`/`-c`, `--runtime`/`-r`, `--dotnet`, `--build-timeout-ms`, `--exe`,
`--app-arg`, `--wait-ms`, `--settle-ms`.

### `generate assets <image>`

`--out`/`-o`, `--ico`, `--full`, `--no-ico`.

### `isolate <app|publishDir>`

Runs selected checks inside Sandbox (default) or a prepared Hyper-V guest.

`--selftest-context` is an exclusive plain-selftest mode requiring managed `--backend vm`. It selects the authenticated
guest worker/lease without unrelated permissions and adds the hash-bound `selftestContext` artifact for the launcher
and direct child: token/authentication/logon/session IDs and types, elevation/app-container/profile booleans,
read-only credential-persistence capabilities, and native errors/HRESULTs. No usernames, paths, credential contents,
or environment markers are emitted. Missing context never overrides strict app guard failures or permits host
fallback. See [the context guide](../../../../docs/guide/devtools-selftest-context.md).

`--devtools`, `--devtools-exe`, `--exe`, `--demo-arg`, `--no-selftest`, `--portable-test`, `--filter`,
`--contrast-theme`, `--record`, `--record-app-arg`,
`--record-duration-ms`, `--record-fps`, `--record-wait-ms`, `--record-allow-static`, `--drive-script`,
`--drive-app-arg`, `--private-input-manifest`, `--drive-packaged`,
`--drive-preserve-app-data`, `--drive-wait-ms`,
`--drive-find-timeout-ms`, `--drive-settle-ms`, `--drive-no-capture`, `--drive-require-cleanup`, `--drive-crash-metadata`,
`--drive-snapshot-depth`,
`--backend sandbox|vm`, `--pool`, `--target-state-dir`, `--legacy-vm-env`, `--timeout`, `--vm-wait-seconds`,
`--packaged`, `--msix`, `--cert`, `--dependency-package`, `--app-fixture`.

`--portable-test` is an exclusive direct managed-guest mode. Pass the exact
self-contained Microsoft.Testing.Platform executable, `--backend vm`, and a
self-contained `--devtools` directory. The optional `--filter` is forwarded as
one argument. `--devtools-exe` accepts one existing file name under that
canonical directory; rooted paths, separators, traversal, alternate data
streams, and missing files are rejected, and the selected name is bound to the
verified tool payload. Sandbox, host, automatic, legacy-environment, selftest,
capture/record/drive, package, theme, language, and fixture combinations are
rejected before guest effects. Managed and explicitly authorized existing VMs
use the ordinary global lease and immutable identity checks.

The authenticated worker invokes the existing `DotNetTestCheck` directly; it
does not recursively call the public `test` command or contact the resident
target host. App and tool trees use transient store-only archives for the
PowerShell Direct transfer. A host-locked file manifest is verified after
fail-stop cleanup and extraction into canonical `C:\sprout` children.
Success still requires a non-empty passing TRX, exit zero, locked complete
payload, held-process exit, guest-service cleanup, and verified rollback.
A claimed passing guest result is accepted only when its unique check,
aggregate, assertion set, TRX/payload artifacts, executable/payload hashes,
process generation, service ID, and lease ID are mutually consistent.
The `testTimeout` assertion records the exact requested child-process deadline;
worker and host reserves are separate and cannot silently lengthen that test.
The outer `payloadDelivery`, `testPayloadDelivery`, and
`toolPayloadDelivery` assertions bind the guest-verified extracted trees to
the host-selected manifests.

`--app-fixture <manifest.json>` declares **2..4 independent signed applications** for one authenticated managed-VM
drive session. Use `--backend vm --no-selftest --drive-no-capture --drive-script <steps>`; no legacy/Sandbox/host
fallback, other launch/package mode, private-input manifest, or implicit settle delay. The manifest has
`schema: "sprout.devtools.app-fixture.v1"`, `primary: "<role>"`, and `apps[]` entries with `role`, relative `msix`,
relative `certificate`, exact `aumid`, and optional `arguments`. Roles are unique bounded lowercase names and must
have different real package identities. Inputs and script are hash-bound into the authenticated plan before execution.
Each lifecycle step accepts only `op` and `role`: `installApp`, `startApp`, `selectApp`, `stopApp`, `uninstallApp`.
Install the primary first; late secondary installation while the primary runs is supported. `startApp` selects the
new owned generation, `selectApp` preserves other processes, and every lifecycle boundary clears the found element
and releases held input. Use a new `find` before the next element action. Each role installs once and may start at
most 16 generations. A secondary is an application role, never a framework dependency.
Explicit `snapshot` works; recording and `findDesktop` / `snapshotDesktop` remain outside this narrow route.
Desktop steps fail preflight before guest launch or package preparation, regardless of an ambient guest capability.
Role queries keep their owned-PID scope. Native IME uses existing guest-only key operations and the authorized
`--input-language` transaction; it does not require extra Desktop UIA authority.
The timeout is 1..900 seconds; scripts are at most 1 MiB / 512 steps. Role-prefixed `appFixture.<role>.*`
assertions retain separate MSIX/certificate/executable hashes, exact package/AUMID, owned process identities, and
cleanup. Original and cleanup failures are both retained. Shared framework/certificate restoration occurs after all
actors receive cleanup; require `checkpointRollback` as well as the app assertions. See
[the full app-fixture guide](../../../../docs/guide/devtools-app-fixtures.md).

`--drive-require-cleanup` opts the separate in-guest drive into `drive --require-cleanup`, inheriting its
60000 ms interaction deadline and independent cleanup reserves. It requires `--drive-script` and rejects `--record`
before backend or payload access. Both guest execution paths forward the option without changing the requested
deployment form; strict drive still rejects loose registration and synthetic input. Use
`--packaged --drive-packaged` for strict signed-MSIX driving. Omission preserves existing behavior; signed
dependency-package drives continue to require cleanup, with no duplicate flag. Read the hash-bound
`guestDriveResult` and require `processCleanup` with expected/actual `exited`, positive `processId` and
`processCreationFileTime`, and the approved `processExecutableSha256`. The outer `--timeout` must allow
launch/cleanup reserves; guest rollback or timeout alone is not an exit receipt.

`--contrast-theme Aquatic|Desert|Dusk|"Night sky"` requires `--backend vm` and applies that installed built-in Windows contrast theme inside the
guest before any app process starts. DevTools reads the installed `.theme` data, enables high contrast through
`SystemParametersInfo(SPI_SETHIGHCONTRAST)`, applies its system colors through `SetSysColors`, and verifies both the
mode and resulting `GetSysColor` values. The lifted `contrastTheme` check reports the verified palette. Missing theme
data, an unsupported OS, or a rejected Win32 operation errors explicitly; DevTools does not use Settings automation,
private theme APIs, or activation workarounds. Omitting the option preserves the existing light/dark/system path.

`--private-input-manifest` accepts a caller-owned JSON file with up to eight guest-only files (256 MiB each,
512 MiB total) and sixteen private drive arguments. Arguments can contain `{guest-file:<id>}` placeholders. Inputs
are delivered under opaque names outside the app/package payload, argument values are redacted from result metadata,
and transient staging stays outside the artifact tree and is deleted after the isolated run. The manifest and source
files remain caller-owned.
The manifest may also declare up to four small required outputs:
`"outputs":[{"id":"receipt","fileName":"startup-receipt.json"}]`. Use `{guest-output:receipt}` in a private app
argument. It resolves only under the owned guest drive-artifact directory. The guest forwards a matching
`--require-artifact receipt=declared\startup-receipt.json`; the existing artifact store enforces a 1 MiB file ceiling
and emits `tests[id=drive].artifacts[kind=guestOutput:receipt].{path,sha256}` in `guestDriveResult`.
`--drive-crash-metadata` forwards `drive --crash-metadata` for the separate
in-guest drive path. An early exit adds
`artifacts[kind=crashMetadata,path=crash-metadata.json]`; the document contains
the exact PID/launch interval, observed exit/exception code or HResult, faulting
module basename, fault RVA, closed category, and at most eight validated method
names. No dump, raw event XML/message, full path, user string, or value-bearing
stack is retained. Missing/unavailable matching events produce typed
`notObserved` rather than an inferred cause.
`--packaged` first reuses an app-owned MSIX and sibling/derivable certificate from the publish directory.
Use `--msix <signed.msix> --cert <certificate.cer>` to select an external existing artifact explicitly. DevTools
synthesizes only when the app ships no package.

`--dependency-package <signed-framework.msix|appx>` is repeatable and requires the signed drive-only combination
`--no-selftest --packaged --drive-packaged --drive-script`, or the managed `--app-fixture` path above. It validates at most eight explicit files (512 MiB each,
1 GiB total), copies them with SHA-256 verification, and forwards them to the guest's
`drive --packaged --install signed --require-cleanup`. That one installation uses `Add-AppxPackage -DependencyPath`
before AUMID activation. There is no host installation, download, recursive discovery, or dependency-certificate
import. Inputs must be regular local non-reparse framework packages without traversal, duplicate identities,
applications, or resource identities; a package signature is required. Windows, not structural ZIP inspection,
verifies cryptographic trust and compatibility. Preexisting exact signed frameworks are retained; conflicting
registrations are refused; newly registered frameworks join parent-owned rollback.

The outer `dependencyPackageSha256` assertions compare supplied and consumed hashes, keyed by full name in `message`.
The nested drive receipts use the full name in `expected` and the hash in `actual`, with a separate
`dependencyPackageCleanup` receipt after verified rollback. Keep `processCleanup`, `packageUninstall`, and
`checkpointRollback` as independent gates. Dependency failure never changes the requested deployment form.

For VM evidence, read the outer `run.host.isolation`, `isolationTarget`, and `isolationWaitMs`, then resolve
`tests[id=isolatedRun].artifacts[kind=guestDriveResult].path` and verify its `sha256`. The nested result owns
`run.host.{os,build,dpiAwareness}`,
`run.app.{form,formSelection,identitySource,packageFamilyName,packageFullName,packageArtifactSha256,aumid}`,
and `tests[id=drive].assertions[kind=signatureKind|isDevelopmentMode|packageUninstall]`. The outer
`tests[id=isolatedRun].assertions[kind=packageArtifactSha256]` proves the guest consumed the exact source MSIX;
`tests[id=isolatedRun].assertions[kind=checkpointRollback]` proves verified checkpoint restore and final VM Off.

`--backend vm` resolves the named managed pool (`--pool default`) from `--target-state-dir` and reads each member's
credential from its configured Windows Credential Manager entry before lease or launch. It uses the same
`HyperVPoolExecutor`/global per-VM lease as the other first-party VM paths. Missing pool state, no Ready member, or
missing authentication fails closed. The legacy environment path remains available only through
`--legacy-vm-env`.

## Management commands

Management commands do not promise `sprout.devtools.orchestration.result.v1`.

### `host`

The recursive `--state-dir <dir>` option selects the complete host root.

| Command | Options |
|---|---|
| `host serve` | `--background` (normally used only by automatic startup) |
| `host status` | no leaf options |
| `host selftest <app>` | `--selftest-arg`, `--expect-token`, `--timeout`, `--queue-wait-seconds` |
| `host cancel <job-id>` | no leaf options |
| `host stop` | `--force`; strict owned-host stop additionally uses `--expected-process-id`, `--expected-process-creation-file-time`, `--expected-server-payload-sha256`, `--require-exit`, and optional `--exit-timeout-seconds` |

`host status` includes the exact server PID, process creation FILETIME,
orchestration payload SHA-256, contract version, and the `processTest`
capability used before VM submission. Unqualified `host stop` preserves the existing
behavior. Automation that owns a private broker root can bind shutdown to that
observed generation:

```powershell
sprout-devtools host stop --state-dir <owned-root> `
  --expected-process-id <pid> `
  --expected-process-creation-file-time <filetime> `
  --expected-server-payload-sha256 <sha256> `
  --require-exit --exit-timeout-seconds 15
```

Strict identity flags are all-or-none and reject `--force`. The server compares
them atomically with the idle/admission decision. A replacement process,
different creation time or payload, or newly admitted job refuses without
stopping. Strict requests use the v6+ strict-stop
`target.stop-strict.v1` method; `target.stop` remains the legacy unqualified
method, so an older replacement server rejects the unknown strict method before
performing any stop. Success emits one
`sprout.devtools.target-host-stop.v1` JSON receipt containing the expected and
accepted identity, process-executable SHA-256, server generation, stop
acceptance, and held-handle exit result. Timeout, cancellation, rejection, or
ambiguous identity returns an errored receipt with `exited:false`; PID absence
alone is never success.

### `vm`

The recursive `--state-dir <dir>` option selects image/pool/provisioning state.

| Command | Options |
|---|---|
| `vm image import <iso>` | `--sha256`, `--source officialEvaluation|licensedByol`, `--architecture x64|arm64` |
| `vm pool ensure <name>` | `--image`, `--image-index`, `--members`, `--devtools`, `--guest-user`, `--checkpoint`, `--processors`, `--startup-memory-gb`, `--maximum-memory-gb`, `--disk-size-gb`, `--activation none|productKey`, `--timeout-minutes`, `--plan-only` |
| `vm pool status <name>` | `--public` emits nonsecret routing/member/checkpoint identity plus `fingerprintSha256` |
| `vm pool repair <name>` | `--devtools <current-tool-dir>`, `--wait-seconds 0..600`, `--timeout-minutes`, `--plan-only` |
| `vm diagnose <pool>` | `--member <id>`; bounded read-only typed checks, best consumed with `--json` |
| `vm quarantine explain <pool>` | required `--member <id>` |
| `vm quarantine recover <pool>` | required `--member <id>`, `--wait-seconds 0..600` |

### Caller-owned existing VM profiles

Credential operations are `vm credential set <id> --user <user> --secret-stdin`, `vm credential status <id> --json`,
and `vm credential remove <id> --confirm`. Set refuses an existing reference unless `--replace` is explicit;
set/status/remove serialize on that reference. Never store a password in a profile or command line.

Use an untracked profile with schema `sprout.devtools.existing-hyperv-targets.v1` and a `targets` array of 1..16 entries:

```json
{
  "schema": "sprout.devtools.existing-hyperv-targets.v1",
  "targets": [{
    "id": "existing-a",
    "vmName": "<exact-vm-name>",
    "vmId": "11111111-1111-1111-1111-111111111111",
    "checkpointName": "<exact-checkpoint-name>",
    "checkpointId": "22222222-2222-2222-2222-222222222222",
    "guestUserName": ".\\Admin",
    "credentialName": "Sprout.DevTools/ExistingHyperV/existing-a/GuestPassword",
    "authority": "restoreCheckpointAndPowerOff"
  }]
}
```

The caller authorizes restoration and final power-off of that exact checkpoint. The VM must support PowerShell
Direct, an interactive console, and the app/tool runtime; its creator is irrelevant.

The same `--existing-vm-profile` works with `isolate --backend vm` and `drive-shell`. It rejects `--pool` and the
name-only `--legacy-vm-env` route. Immutable identities are checked while the per-VM lease is held. Managed owner
markers cannot be bypassed through this route, and no VHD/pool ownership, provisioning, or repair is acquired.
Operational failure and failed abandoned-owner recovery are terminal. Results expose `targetVmId`,
`targetCheckpointId`, and `targetOwnership=callerOwned`; exact rollback/off remains mandatory.
Managed-only authenticated-worker operations such as app fixtures and OS theme/input-language mutation remain
managed-pool-only.

## Interactive command

### `drive-shell <app>`

Runs a live Hyper-V guest channel. Without `--script` it is an interactive JSONL REPL; script mode writes a check
result.

`--devtools`, `--devtools-exe`, `--exe`, `--app-arg`, `--script`, `--service-guid`, `--deadline`,
`--connect-timeout`, `--call-timeout`, `--find-timeout`,
`--capture-backend Auto|WindowsGraphicsCapture|PrintWindow`, `--no-evidence`, `--vm-wait-seconds`, `--pool`,
`--target-state-dir`, `--existing-vm-profile`.

`--capture-backend` defaults to `Auto` and supplies the default for interactive `capture` requests plus automatic
failure screenshots. A request may override it with
`{"op":"capture","args":{"backend":"WindowsGraphicsCapture"}}`. Unknown names fail before capture, and explicit WGC
never falls back to PrintWindow. A successful response records `result.artifact.requestedBackend` and the actual
`result.artifact.backend`; the shell writes the announced `savedMetadata` JSON beside `saved` PNG evidence. When a
capture request itself fails, automatic evidence does not issue a second capture through another backend.

The guest-only global operation
`{"op":"setContrastTheme","args":{"scheme":"Aquatic"}}` keeps the same app process alive while switching among
`Aquatic`, `Desert`, `Dusk`, and `Night sky`. Success returns `result.contrastTheme` with the canonical scheme,
installed theme file name, resolved display name, high-contrast state, and verified system colors. Normal host
`drive` has no system-theme mutation surface; an agent not explicitly launched as an isolated guest rejects the op.

This success proves Windows native state, not an application palette-cache refresh. Wait for a Sprout app-visible
palette witness before capture. Use a fresh process per preset for Windows App SDK 1.8 WinUI references: its resident
cache retained the first contrast scheme in the controlled experiment, even after later native readbacks succeeded.

## Runtime inspection

All `inspect` leaves inherit `--sessions <dir>` and `--connect-timeout-ms <n>`.

| Command | Options/output |
|---|---|
| `inspect list` | `sprout.devtools.cli.result.v1` target list |
| `inspect connect <target>` | negotiated identity/capabilities |
| `inspect snapshot <target>` | `--cached`, `--expected-version`, `--deadline-ms` |
| `inspect export <target>` | snapshot options plus required `--output`/`-o` and optional `--force`; writes `sprout.devtools.snapshot.v1` |

`--deadline-ms` bounds the target-side wait for a UI safe point. It does not change
`RuntimeDiagnosticsLocalOptions.CaptureTimeBudget`.

## Internal commands

`agent`, `host agent`, `host mailbox-agent`, `vm pool provision-worker`, and hidden `record-worker` are
orchestration-owned guest/elevated workers. They are not public operator contracts; do not call them directly or copy
their worker flags into application automation.

## Application check result contract

Application check commands write `sprout.devtools.orchestration.result.v1`:

```
run { id, status, startedUtc, durationMs, toolVersion,
      host { os, build, dpiAwareness, isolation, isolationTarget?, isolationWaitMs?,
             target?, targetSelection?, targetReason?, developerMode, gpu? },
      app? { path?, form?, formSelection?, identitySource?, packageFamilyName?, aumid?, arguments? } }
tests[] { id, title?, status, durationMs, steps[],
          assertions[] { kind, status, expected?, actual?, message? },
          artifacts[] { kind, path, sha256? },
          error? { message, diagnostic? } }
artifactsRoot?
```

Status is lowercase `passed`, `failed`, `skipped`, or `errored`; overall status is worst-wins.

| Exit code | Meaning |
|---|---|
| `0` | Passed |
| `1` | Assertion failed |
| `2` | Skipped, errored, or another non-verdict status |

Read `run.status` and every `tests[].status` first. A skipped/errored check may carry `error` without an assertion.
Read `assertions[]` for verdict details and `artifacts[]` for evidence, not stdout prose.

`app.form` is `unpackaged` or `packaged`; `app.formSelection` is `auto` or `explicit`;
`app.identitySource` is `projectPackage`, `looseLayout`, or `synthetic`. Read these fields instead of inferring identity
from the checks that happened to run.

## What is a gate?

| Evidence | Deterministic gate? |
|---|---|
| `selftest` token | Yes |
| `process-test` exact token + exit/cleanup receipts | Yes |
| UIA `--expect-*` assertion | Yes |
| complete/valid `inspect snapshot` data | Yes |
| screenshot or `recording.mp4` | No; post-compositor triage evidence |
