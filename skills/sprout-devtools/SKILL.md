---
name: sprout-devtools
description: >-
  Look at a running Sprout app and check its UI: take a real screenshot, verify layout/spacing/sizes, confirm a
  change's visual effect, dump the accessibility tree, or act on a control and assert the result. Use whenever asked
  whether the UI looks or behaves right. CLI: `sprout-devtools`.
---

# Look at a Sprout app and check its UI

`sprout-devtools` is the first-party CLI for **seeing and checking a real Sprout application**. It launches the app,
screenshots it after the compositor, dumps its accessibility tree, reads the framework's own layout truth out of the
running process, and drives controls through UI Automation — all emitting structured JSON instead of prose.

Use it whenever the request is some form of *"is the UI right?"*:

- "show me what this screen looks like" / "did my change actually work visually"
- "check the sidebar spacing / this element's size / whether something is clipped"
- "is this reachable by a screen reader" / "dump the accessibility tree"
- "click that button and tell me what happened"

**Never hand-roll this.** Do not write `PrintWindow` / `System.Drawing` / `Add-Type` screenshot scripts, or ad-hoc UIA
dumps. They dead-end on Sprout's `WS_EX_NOREDIRECTIONBITMAP` windows (a pure DirectComposition surface that ordinary
GDI capture returns blank for) and they cost far more time than this tool.

## Install (once per machine)

```powershell
dotnet tool install -g Sprout.DevTools --prerelease
sprout-devtools doctor
```

Requires a **.NET 10** runtime on PATH. The package is self-contained — nothing else needs installing. Upgrade with
`dotnet tool update -g Sprout.DevTools --prerelease`. If `sprout-devtools` is not found after installing, the shim
directory (`%USERPROFILE%\.dotnet\tools`) is not on PATH for this shell; open a new shell or prepend it.

`doctor` reports the host (OS build, DPI awareness, Developer Mode, isolation) and is the right first move whenever a
capture looks wrong.

## Pick the workflow

| What you were asked | Do this | Detail |
|---|---|---|
| "What does it look like?" / "did my change land visually?" | `debug <app> --from-source` — build Debug, launch, screenshot **and** dump the UIA tree in one shot | [screenshots.md](references/screenshots.md) |
| "Is the layout / spacing / size right?" | Launch the app resident, then `inspect snapshot <pid>` and compare each node's `bounds` against its `desiredSize` | [layout-inspection.md](references/layout-inspection.md) |
| "Is it accessible / what does a screen reader see?" | `debug` and read `debug-uia-tree.txt`, or a `drive` script with a `snapshot` step | [accessibility.md](references/accessibility.md) |
| "Click it and check what happened" | `drive <app> --find-name … --invoke --expect-…` (UIA patterns only) | [interaction.md](references/interaction.md) |
| "Does it still pass its own checks?" | `selftest <app>` — the deterministic token gate | [command-reference.md](references/command-reference.md) |
| Something failed and you don't know why | [troubleshooting.md](references/troubleshooting.md) | |

Full verb list, global options, result schema and exit codes: [command-reference.md](references/command-reference.md).

## Three things that will mislead you

Read these before trusting any result. The long-tail list is in [pitfalls.md](pitfalls.md).

### 1. `debug` reporting PASSED does **not** mean your UI is fine

`debug` asserts exactly one thing — `windowCaptured`, "the screenshot came back bigger than 0x0". It does not check the
window's title, class, or content. A **.NET unhandled-exception dialog is a perfectly good window**, so a crashed app
reports `passed` with a screenshot of its own crash box.

**Always confirm what you actually got** by reading `debug-uia-tree.txt` (and looking at `debug-window.png`) before
reporting anything. If the tree's root is a dialog or an error box, or holds a handful of generic controls where your
real UI should be, the run failed no matter what the status says.

### 2. `debug` is one-shot forensics, not a resident session

`debug` launches the app, captures evidence, and **exits it**. There is no live process left to inspect afterwards.

When you need a *resident* app — to snapshot it, to iterate, or to let a human look at it — start it yourself and then
attach:

```powershell
sprout-devtools build <project> --artifacts out       # or your repo's own build script
Start-Process out\publish\<YourApp>.exe               # resident; keep it running
sprout-devtools inspect list                          # find its pid
sprout-devtools inspect snapshot <pid> > snapshot.json
```

`drive … --keep-open` does the same thing while also putting the app into a particular state first.

Give the app a couple of seconds before snapshotting: a cold app briefly reports `status: partial` with an almost-empty
tree that reads as a real, empty UI. Poll until `response.status` is `complete` — see
[layout-inspection.md](references/layout-inspection.md).

### 3. Never synthesize input on the developer's own desktop

The `drive` steps `typeText` / `keyDown` / `keyUp` / `click` / `pointerMove` / `pointerDown` / `pointerUp` / `drag` are
implemented with **`SendInput`, which is global**: it injects into whatever window is foreground at that instant, not
into the app you named. This has already destroyed a user's live terminal sessions, irreversibly, and there is no undo.

The **UIA pattern** steps — `find`, `setFocus`, `invoke`, `toggle`, `setValue`, `expand`, `select`, and every `expect*`
— are safe: they act on the element through the accessibility API and synthesize nothing. They are also the
deterministic assertion anyway. If a real desktop gesture is genuinely the subject, run it in an isolated guest
(`isolate --drive-script`, `drive-shell`). See [interaction.md](references/interaction.md).

## Reading the result

Every command writes one `result.json` under `--artifacts <dir>` (default `.sprout-devtools/<runId>`) and returns:

| Exit code | Meaning |
|---|---|
| `0` | all assertions passed |
| `1` | an assertion failed (the app or the expectation is wrong) |
| `2` | the harness itself errored (bad arguments, app never launched, environment missing) |

**Read `tests[].assertions[]` for the verdict and `artifacts[]` for the evidence files — not stdout prose.** Add
`--json` to get the machine-readable form on stdout. Status values are lowercase `passed` / `failed` / `errored`, and
the overall status is worst-wins.

A screenshot is **triage evidence, not a gate**: post-compositor capture depends on the GPU, driver, DWM and even the
wallpaper behind a transparent window. Use it to *see*; use assertions and snapshot data to *decide*.

## Reference index

| File | Read it when |
|---|---|
| [references/screenshots.md](references/screenshots.md) | Capturing a window, judging the image, blank or wrong-window frames |
| [references/layout-inspection.md](references/layout-inspection.md) | Verifying layout, spacing and sizes from framework truth; the snapshot JSON shape |
| [references/accessibility.md](references/accessibility.md) | Reading and checking the UI Automation tree |
| [references/interaction.md](references/interaction.md) | Acting on controls and asserting state; drive scripts |
| [references/command-reference.md](references/command-reference.md) | Every verb, its options, the result schema |
| [references/troubleshooting.md](references/troubleshooting.md) | A command failed, produced nothing, or found no target |
| [pitfalls.md](pitfalls.md) | The full DO / DON'T list |
