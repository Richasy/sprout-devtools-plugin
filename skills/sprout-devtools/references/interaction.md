# Act on a control and assert the result

```powershell
sprout-devtools drive <app> --app-arg=--some-flag `
  --find-name "Accept the terms" --find-type CheckBox `
  --toggle --expect-toggle On --artifacts out
```

`drive` launches the app, finds a control through UI Automation, acts on it, asserts its state, and writes a `drive`
check with `uia` assertions plus `window.png`, `uia-tree.txt` and `uia-snapshot.json`.

## Safe vs dangerous steps

> **This is the most consequential rule in this skill.**

| Safe — UIA patterns | Dangerous — synthetic `SendInput` |
|---|---|
| `--invoke`, `--toggle`, `--set-value`, `--set-range-value` | `--type-text` |
| `--expand`, `--collapse`, `--select` | `--click` |
| `--scroll-*`, `--set-scroll-*` | `--drag-to-*` |
| every `--expect-*` | script steps `typeText`, `keyDown`, `keyUp`, `click`, `pointerMove`, `pointerDown`, `pointerUp`, `drag` |

The safe column talks to the element **through the accessibility API** and synthesizes nothing. The dangerous column
calls `SendInput`, which is **global** — it injects into whatever window is foreground at that instant, not into the
app you launched. If the app takes focus late, loses it, or never gets it, the keystrokes and clicks land in the
developer's shell. This has already destroyed a user's live terminal sessions, and there is no undo.

**Never run a synthetic-input step on the developer's own desktop.** If a real desktop gesture is genuinely what you
are testing, run it in an isolated guest — `isolate --drive-script`, or `drive-shell` for a live Hyper-V session. If
neither is available, verify headlessly instead, or ask the user to run it and report what they see. An unverifiable
appearance claim is cheaper than an unrecoverable action.

The UIA pattern verbs are the deterministic assertion anyway, so the safe column is almost always what you actually
wanted.

## Finding a control

- `--find-name` (accessible Name), `--find-automation-id`, `--find-type` (`Button`, `CheckBox`, `Slider`, `Edit`, …).
  Combine them to disambiguate.
- `--find-within-name` / `--find-within-automation-id` / `--find-within-type` narrow to a **descendant** of the first
  match — the way to reach "the Save button inside the Card group".
- `--find-timeout-ms` (default 5000) is the provider-ready gate for each find; `--wait-ms` (default 10000) is the main
  window poll. An app that starts slowly needs a bigger `--wait-ms`, not a sleep.

A top-level `find` follows the process if it replaces its main window (launcher → workspace). `findWithin` stays
scoped to the current element.

## Scripts

For more than one step, use `--script steps.json`:

```json
[
  { "op": "find", "name": "Card", "controlType": "Group" },
  { "op": "findWithin", "name": "Save", "controlType": "Button" },
  { "op": "expectEnabled", "value": true },
  { "op": "invoke" },
  { "op": "snapshot" }
]
```

A script is also the only way to reach the JSON-only steps, notably `resizeClient` (set an exact client size in
physical px or epx and emit a `window` assertion with DPI evidence) and `snapshot` (capture a structured Control View
subtree as evidence).

## Leaving the app running

`--keep-open` leaves the app alive after the drive instead of terminating it. This is the clean way to get a
**resident** app for [`inspect snapshot`](layout-inspection.md): drive it into the state you care about, keep it open,
then snapshot the layout of that exact state.

```powershell
sprout-devtools drive <app> --find-name "Expand" --invoke --keep-open --artifacts out
sprout-devtools inspect list
sprout-devtools inspect snapshot <pid> > snapshot.json
```

Remember to close the process when you are done.

## Evidence

Every drive writes `uia-tree.txt` and `uia-snapshot.json` bounded by `--snapshot-depth` (default 8, max 250 nodes).
That bound applies to the **evidence only** — it never weakens the `--find-*` gate, which searches the full tree.
`--no-capture` skips the screenshot if you only want the assertions.

Read the verdict from `tests[].assertions[]` in `result.json`, not from stdout.

## Packaged apps

`--packaged` registers a build-produced loose layout's real `AppxManifest.xml` in Developer Mode, AUMID-launches it,
and drives that exact packaged process — no admin, no VM. `--install signed --msix <m> --cert <c>` drives the real
signed install instead. Use these when identity-sensitive startup is the subject; omit for the ordinary executable.
