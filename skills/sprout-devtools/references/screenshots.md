# Screenshots and visual evidence

## The one-shot loop: `debug`

The fastest way to answer "what does it look like / did my change land":

```powershell
sprout-devtools debug <project-or-exe> --from-source --artifacts out
```

It publishes the project in **Debug** (symbols), launches it, waits for a window, screenshots it, dumps the UI
Automation tree, and exits the app. Use `--from-source` when passing a `.csproj` or a project directory; omit it when
passing an already-published `.exe` or publish directory.

Useful options: `--app-arg=--some-flag` (repeatable — note the `=`, or a `--`-prefixed value parses as an option),
`--wait-ms` (window poll, default 10000), `--settle-ms` (delay before capture, default 500),
`--build-timeout-ms` (default 600000 — NativeAOT publishes take minutes), `--exe <name>` to disambiguate inside a
publish directory.

### What it writes

| File | Written | Contents |
|---|---|---|
| `debug-window.png` | when a window was found | the post-compositor screenshot |
| `debug-uia-tree.txt` | best-effort | indented `ControlType "Name"` accessibility tree |
| `debug-window-candidates.txt` | only when no window was found, or the frame was blank/uniform | every visible top-level window with `hwnd`, `pid`, `size`, `owned`, `tool`, `title`, marking which belong to the target process |
| `result.json` | always | the canonical run result |

### Always read the tree, not just the status

`debug`'s only assertion is `windowCaptured` — "the screenshot is bigger than `0x0`". A .NET crash dialog satisfies
that, so **a crashed app reports `passed`**. Open `debug-uia-tree.txt` and confirm the root and its children are your
real UI before reporting success. See [SKILL.md, trap 1](../SKILL.md).

### Actually look at the image

After capturing, **view `debug-window.png`** — image files render directly. Do not describe a screenshot you have not
opened, and do not infer appearance from source code when an image is sitting in the artifacts directory.

## Screenshot only: `capture`

```powershell
sprout-devtools capture <app> --app-arg=--some-flag --artifacts out
```

Same launch/capture path without the Debug build or the tree dump; writes `window.png`.

To preserve the exact state of an app that is already running and navigated, omit the app argument and attach by PID:

```powershell
sprout-devtools capture --pid 2976 --artifacts out
```

This mode does not relaunch or re-register the app, activate or foreground its window, inject input, resize it, or
terminate it. It pins the process creation time and image identity, resolves only that process's main HWND, verifies
the HWND owner before and after capture, and publishes no screenshot if the process exits or either identity changes.
Do not combine `--pid` with an app argument, `--app-arg`, `--packaged`, or `--keep-open`. Read `window-capture.json`
beside `window.png` for the capture dimensions and backend. Use a fresh `--artifacts` directory; attached capture does
not overwrite existing final screenshot/metadata files.

An owned popup is a separate top-level window, not necessarily part of the main-window image. Select it explicitly:

```powershell
sprout-devtools capture --pid 2976 --window-title "Details" --artifacts popup-title
sprout-devtools capture --pid 2976 --hwnd 0x1234 --artifacts popup-handle
```

Use exactly one selector. Titles are exact and case-sensitive; HWNDs accept positive decimal or `0x`-prefixed hex.
Selection is immediate and restricted to a unique visible, positively sized top-level window of the held process,
including owned popups and tool windows. `--wait-ms` remains a default-main-window option. Missing/hidden/ambiguous,
foreign, or changed selections fail rather than switching to another window.
The process and selected window are rechecked around capture and before final evidence publication.

Read the additive `selectedWindow` identity in `window-capture.json` (mode, HWND, PID, process creation FILETIME,
resolved title, owner HWND). `secondaryWindows` still describes compositing relative to that selected window;
`excluded` does not mean an explicitly selected popup was omitted. No window activation, experiment `SetFocus`,
desktop input, or synthetic combined bitmap is added.

To require a real Windows.Graphics.Capture frame with no PrintWindow fallback:

```powershell
sprout-devtools capture --pid 2976 --backend WindowsGraphicsCapture --artifacts out
```

Explicit backend evidence records both `requestedBackend` and actual `backend`; `Auto` preserves the prior fallback
behavior and metadata shape.

## Video: `record`

```powershell
sprout-devtools record <app> --app-arg=--some-flag --duration-ms 3000 --fps 15 --artifacts out
```

Produces `recording.mp4` + a typed `recording.json` (measured timestamps, `encodedFrameCount`, `actualFps`, drop
counts, SHA-256). Requested FPS is a **ceiling**, not a promise. It fails closed on blank, frozen, truncated or
partial output. Use it for animations, transitions and gesture motion — anything a single frame cannot show. Adding
`--drive-script <p>` starts the script only after the first valid frame, so the interaction and the video share one
launch.

## How capture actually works (and why it sometimes looks wrong)

The capture path is capability-routed:

- **`Windows.Graphics.Capture` (WGC)** for ordinary windows, including WinUI 3 / WebView2 child-composition surfaces.
- **`PrintWindow(PW_RENDERFULLCONTENT)`** for Sprout's own `WS_EX_NOREDIRECTIONBITMAP` windows — a pure
  DirectComposition surface for which WGC produces no frame at all.

Consequences worth knowing:

- **WGC needs an active interactive desktop.** A disconnected RDP session keeps the HWNDs alive but produces no
  compositor frame, so captures come back blank. Use a local session, Windows Sandbox, or an auto-logon Hyper-V guest.
- **A screenshot is triage, not a gate.** It varies with GPU, driver, DWM, and the wallpaper behind a transparent
  window. Decide with assertions and snapshot data; use the image to see.
- The window picked is the target process's foreground window, else its largest visible, non-owned, positively-sized
  top-level window. If that guessed wrong, `debug-window-candidates.txt` shows you what else was on screen.

## Blank or wrong frame checklist

1. Run `sprout-devtools doctor` — check DPI awareness, Developer Mode, and whether the session is isolated.
2. Read `debug-window-candidates.txt` if present: was a splash, a launcher, or a dialog captured instead?
3. Increase `--wait-ms` and `--settle-ms` for an app that takes a while to present its first real frame.
4. If the shell is running over RDP or a disconnected session, move to a local session or an isolated guest.

More failure modes: [troubleshooting.md](troubleshooting.md).
