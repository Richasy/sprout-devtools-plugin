# Pitfalls — sprout-devtools

Terse DO / DON'T assertions from real use. Add one whenever a mistake costs real time.

## Safety

- **DON'T** run a synthetic-input step — `--type-text`, `--click`, `--drag-to-*`, or the script ops `typeText`,
  `keyDown`, `keyUp`, `click`, `pointerMove`, `pointerDown`, `pointerUp`, `drag` — on the developer's own desktop.
  They are `SendInput`, which is **global**: it injects into whatever window is foreground at that instant, not into
  the app you named. It has destroyed a user's live terminal sessions, irreversibly. **DO** use the UIA pattern verbs
  (`--invoke` / `--toggle` / `--set-value` / `--expect-*`), which act through the accessibility API and synthesize
  nothing — and which are the deterministic assertion anyway. For a genuine desktop gesture, use `isolate` or
  `drive-shell`.
- **DON'T** claim a UI looks right when you could not verify it. **DO** say so and ask the user to look, or run the
  check in an isolated guest. An unverifiable claim is cheaper than an unrecoverable action.

## Trusting a result

- **DON'T** treat `debug` reporting `passed` as "the app works". Its only assertion is `windowCaptured` (`>0x0`) — a
  .NET crash dialog satisfies it. **DO** read `debug-uia-tree.txt` and open `debug-window.png` every time.
- **DON'T** report on a screenshot you have not opened. **DO** view the PNG; image files render directly.
- **DON'T** treat a screenshot or `recording.mp4` as a red/green gate — post-compositor capture varies with GPU,
  driver, DWM, and the wallpaper behind a transparent window. **DO** decide with `--expect-*` assertions, the
  `selftest` token, or snapshot data, and use the image to *see*.
- **DO** read `tests[].assertions[]` and `artifacts[]` from `result.json`. **DON'T** parse stdout prose.

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
- **DO** prefer a snapshot over grepping layout source when the question involves a number. A code scan produces
  candidates and false positives; the snapshot produces the defect.

## Running apps

- **DON'T** expect `debug` to leave the app running — it launches, captures, and exits. **DO** use `drive --keep-open`,
  or start the published exe yourself, when you need a resident process to `inspect snapshot`.
- **DON'T** hand-roll a `PrintWindow` / `System.Drawing` / `Add-Type` screenshot script or an ad-hoc UIA dump. Sprout
  windows are `WS_EX_NOREDIRECTIONBITMAP` pure DirectComposition surfaces that ordinary GDI capture returns blank for,
  and pwsh forwards `System.Drawing.Common` so `Bitmap`/`Graphics` do not even resolve. **DO** use this CLI.
- **DON'T** run a capture over RDP or in a disconnected session — the HWNDs survive but no compositor frame is
  produced, so frames come back blank. **DO** use a local session, Windows Sandbox, or an auto-logon Hyper-V guest.
- **DO** attach a `--`-prefixed option value with `=` (`--app-arg=--some-flag`), or it parses as another option.
- **DON'T** truncate `--help` output when checking options; the verb list and option lists are long and slicing them
  causes guess-and-retry loops.

## Diagnostics opt-in

- **DON'T** expect `inspect` to see an ordinary app. It requires **both** `SproutDiagnosticsEnabled=true` at build time
  (a linker feature switch — it cannot be enabled at runtime) **and** a `RuntimeDiagnostics.StartLocalAsync(...)` call.
  `RuntimeDiagnostics.Start(...)` alone opens no endpoint.
- **DO** check that the app runs as the same user when `inspect list` is empty — the rendezvous directory is
  ACL-restricted to the current user SID.
