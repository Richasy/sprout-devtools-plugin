# Pitfalls — sprout-devtools

Terse DO / DON'T assertions from real use. Add one whenever a mistake costs real time.

## Safety

- **DON'T** run a synthetic-input step — `--type-text`, `--click`, `--drag-to-*`, or the script ops `typeText`,
  `keyDown`, `keyUp`, `click`, `pointerMove`, `pointerDown`, `pointerUp`, `drag`, `dragToEdge` — on the developer's own desktop.
  They are `SendInput`, which is **global**: it injects into whatever window is foreground at that instant, not into
  the app you named. It has destroyed a user's live terminal sessions, irreversibly. **DO** use the UIA pattern verbs
  (`--invoke` / `--toggle` / `--set-value` / `--expect-*`), which act through the accessibility API and synthesize
  nothing — and which are the deterministic assertion anyway. For a genuine desktop gesture, use `isolate` or
  `drive-shell`.
- **DON'T** run `gate` on the developer's own desktop. It injects a real touch gesture into the host desktop and belongs
  only in an isolated guest, for the same reason as every synthetic `drive` step.
- **DON'T** claim a UI looks right when you could not verify it. **DO** say so and ask the user to look, or run the
  check in an isolated guest. An unverifiable claim is cheaper than an unrecoverable action.

## Trusting a result

- **DO** inspect `preparation.diagnostic` before attributing an experiment's `uiaUnavailable` to slow startup.
  Strict drive waits for a window before creating its ordinary UIA client; experiment preparation shares one readiness
  budget across window discovery and a timeout-bounded UIA client. A later successful drive is not proof that either
  difference caused the failure. **DON'T** discard the last connection exception when a deadline or a later poll ends
  preparation; retain only type/HRESULT, never provider messages or UIA text.
- **DON'T** treat `debug` reporting `passed` as "the app works". Its only assertion is `windowCaptured` (`>0x0`) — a
  .NET crash dialog satisfies it. **DO** read `debug-uia-tree.txt` and open `debug-window.png` every time.
- **DON'T** report on a screenshot you have not opened. **DO** view the PNG; image files render directly.
- **DON'T** treat a screenshot or `recording.mp4` as a red/green gate — post-compositor capture varies with GPU,
  driver, DWM, and the wallpaper behind a transparent window. **DO** decide with `--expect-*` assertions, the
  `selftest` token, or snapshot data, and use the image to *see*.
- **DON'T** report `recording.json.actualFps` as the app's present rate when the recorder reports backpressure drops;
  the H.264/readback worker is then the limiting pipeline. **DO** use `experiment --frame-timing` for metadata-only WGC
  compositor timestamps, and keep `record` for pixels/video evidence.
- **DO** use the built-in experiment scroll preset instead of writing a one-off plan: `--scroll-mode page` measures a
  steady UIA scroll workload; `percent` deliberately stresses long normalized jumps similar to scrollbar movement.
- **DO** read `tests[].assertions[]` and `artifacts[]` from `result.json`. **DON'T** parse stdout prose.
- **DON'T** truncate GPU engines by identity before choosing the busiest rows: idle engines can hide the actual
  physical-adapter workload. **DO** retain the highest-utilization rows independently for the target and observed
  process, then order those rows by identity. Absence from the bounded list does not prove zero activity.
- **DO** collect `doctor --gpu-adapters --json` for each measurement group and match GPU engine LUIDs to its software
  flags. **DON'T** compare software-engine percentages with physical-adapter percentages or infer an app's adapter
  from the inventory order. Adapter inventory is not process attribution.

## Snapshots and layout

- **DON'T** divide `bounds` or `desiredSize` by `dpiScale`. They are **already epx**; `dpiScale` is an independent
  fact reported for when you need to derive a physical value. Dividing invents layout bugs that do not exist.
- **DON'T** trust the first snapshot after launch. A cold app returns `status: partial` with an almost-empty node list
  for roughly two seconds, while still reporting a correct `dpiScale` and `clientBounds` — so it reads as a real, empty
  UI. **DO** poll until `response.status` is `complete`.
- **DON'T** expect a node's own `displayName` to identify the control: a **site** descriptor's name is just the site
  (`"Build"`). **DO** qualify it with `ownerDescriptorId`'s display name to get `MyControl.Build`.
- **DON'T** look for `sourceDescriptors` inside a root. It is a sibling of `roots` on `response.snapshot`, interned
  once per capture.
- **DON'T** look for `parentId` / `children` on a runtime node. There are none — linkage is the separate `treeEdges`
  array.
- **DO** name a node by walking `runtimeNodes[].logicalElementId` → `logicalElements[].sourceDescriptorId` →
  `sourceDescriptors[].displayName`. There is no name on the node itself.
- **DO** treat `bounds < desiredSize` as the real defect signal (the node got less room than it needs).
  `bounds > desiredSize` is usually an intentional stretch — confirm against authored intent before reporting it.
- **DON'T** report a node with `bounds` and `desiredSize` both `0x0` as an empty container. A `Visibility.Collapsed`
  node is removed from layout entirely and still appears in the snapshot. That is normal.
- **DON'T** trust a node's plausible-looking bounds without checking its ancestors: a collapsed **ancestor**
  short-circuits the layout walk, so its descendants keep **stale, last non-zero** bounds while off screen.
- **DO** check `response.status` before concluding a control is missing — a `partial` capture hit a budget.
- **DON'T** poll forever when every capture is `partial` with `timeBudget` truncation. The default capture budget is
  25 ms and can be too small for a large tree; **DO** raise `RuntimeDiagnosticsLocalOptions.CaptureTimeBudget` (or use
  `Timeout.InfiniteTimeSpan` while inspecting your own app) and accept the longer UI-thread stall.
- **DO** prefer a snapshot over grepping layout source when the question involves a number. A code scan produces
  candidates and false positives; the snapshot produces the defect.

## Running apps

- **DON'T** expect `debug` to leave the app running — it launches, captures, and exits. **DO** use
  `drive --keep-open` (or the matching `capture` / `record` / `gate` option) when you need a resident process to
  `inspect snapshot`. Use `deploy` for resident files/package registration; only its reusable loose-package
  `--keep-open` route parks the exact process it just activated. **DON'T** assume a RID-specific `build` output is
  self-contained; direct-launch it only when the project declares that deployment model, or launch it through its
  matching `dotnet` runtime.
- **DO** give reusable loose-package development a separate stable `--package-dir`, close its prior process, and let
  `deploy` perform the identity-scoped same-version upsert. **DON'T** build into the registered directory, pre-remove
  the package, or use `drive --packaged` as an updater: the resident updater deliberately refuses active or ambiguous
  ownership and never terminates a foreign process.
- **DON'T** hand-roll a `PrintWindow` / `System.Drawing` / `Add-Type` screenshot script or an ad-hoc UIA dump. Sprout
  windows are `WS_EX_NOREDIRECTIONBITMAP` pure DirectComposition surfaces that ordinary GDI capture returns blank for,
  and pwsh forwards `System.Drawing.Common` so `Bitmap`/`Graphics` do not even resolve. **DO** use this CLI.
- **DO** use `capture --pid <pid>` when the already-navigated window state must survive inspection. It verifies exact
  process/HWND identity around capture and never launches, activates, foregrounds, resizes, injects input, or terminates.
  **DON'T** add an app argument, `--app-arg`, `--packaged`, or `--keep-open`; attach mode rejects them before opening
  the target.
- **DO** explicitly select an owned popup with `capture --pid --window-title <exact-title>` or `--hwnd <handle>`;
  **DON'T** assume the main capture's `secondaryWindows: excluded` image contains separate popup pixels.
  Selectors are mutually exclusive, immediate, and limited to the held process's visible top-level windows.
  **DON'T** treat them as permission to call attached-experiment `SetFocus`, activate, or synthesize input.
- **DO** read `selectedWindow` identity separately from `secondaryWindows`; **DON'T** mark a direct popup capture as
  secondary-window compositing or describe a synthetic bitmap as a real capture.
- **DON'T** run a capture over RDP or in a disconnected session — the HWNDs survive but no compositor frame is
  produced, so frames come back blank. **DO** use a local session, Windows Sandbox, or an auto-logon Hyper-V guest.
- **DO** attach a `--`-prefixed option value with `=` (`--app-arg=--some-flag`), or it parses as another option.
- **DON'T** truncate `--help` output when checking options; the verb list and option lists are long and slicing them
  causes guess-and-retry loops.
- **DON'T** call `host stop` as cleanup against the default shared target host — another worktree may own active work.
  Ordinary stop refuses while busy; only stop a private state root you created, or use `--force` as an intentional
  shared-work cancellation.
- **DON'T** infer the live guest resolution from host VM metadata. A basic session can still be 1024x768. **DO** query
  and, when needed, change it through the in-guest `drive-shell` `display` operation before resizing a large app.
- **DON'T** call a resize successful merely because the request returned. `resizeClient` must be `ok:true` and its
  measured physical client dimensions must equal the target; a window constrained by the desktop or
  `WM_GETMINMAXINFO` is a failed operation.

## Diagnostics opt-in

- **DON'T** expect `inspect` to see an ordinary app. It requires **both** `SproutDiagnosticsEnabled=true` at build time
  (a linker feature switch — it cannot be enabled at runtime) **and** a `RuntimeDiagnostics.StartLocalAsync(...)` call.
  `RuntimeDiagnostics.Start(...)` alone opens no endpoint.
- **DO** check that the app runs as the same user when `inspect list` is empty — the rendezvous directory is
  ACL-restricted to the current user SID.

## Touch injection

- **DON'T** put `POINTER_FLAG_NEW` in an injected `POINTER_TOUCH_INFO`. It is a **delivery** flag the system sets on
  a pointer message to mark a new contact; `InjectTouchInput` rejects a packet carrying it with
  `ERROR_INVALID_PARAMETER` (87), so the gesture fails at its first contact and `gate` cannot drive touch at all --
  on any machine. Measured 3/3 both ways with every other field held constant. **DO** send
  `INRANGE | INCONTACT | DOWN`. A unit test that asserts the flag *is* present pins the defect instead of catching it.
# Native IME guest controls

- **DO** establish a native Windows EDIT positive control when both local and remote custom editors fail; **DON'T**
  infer a remote-protocol defect from Latin-only input in an English-only guest.
- **DO** use guest-bound `isolate --input-language zh-CN`, check registration/restoration receipts and pristine
  rollback, and send native `keyDown` / `keyUp`; **DON'T** count a successful profile-activation HRESULT,
  a keyboard-layout handle, UIA `SetValue`, or Unicode `typeText` as composition evidence.
- **DO** wait for composing/preedit and committed-value witnesses; **DON'T** repair an early UIA sample with a sleep or
  accept retry-only native IME success.
- **DO** observe and explicitly decline Pinyin's OS-owned first-run Bing-suggestions card before continuing after the
  first N; **DON'T** assume the app's logical focus means that Space will not activate the card's focused Enable button.
  Use guest-bound `findDesktop` and stable AutomationIds, then inspect the actual candidate panel. Do not modify
  Sprout's IME implementation until a control with that prompt handled still reproduces the failure.
- **DO** derive guest-only permission from the authenticated active managed-worker session and OS-observed partition
  identity; **DON'T** trust writable authorization files, caller-computable MachineGuid hashes, flags, or environment
  markers. A diagnostic receipt is not a grant.
- **DO** keep language/contrast mutation inside the authenticated guest worker and revalidate its lease at native
  boundaries; **DON'T** return a boolean to an arbitrary host process and let that process perform the operation.
- **DO** use one reader for a live encrypted stream; **DON'T** run a concurrent `Poll`/`Available` EOF detector beside
  it, because the real reader can drain the data between those observations and spuriously revoke the session.
