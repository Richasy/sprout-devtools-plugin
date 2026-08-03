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
| `--show-context-menu` | |
| `--expand`, `--collapse`, `--select` | `--click` |
| `--scroll-*`, `--set-scroll-*` | `--drag-to-*` |
| every `--expect-*` | script steps `typeText`, `keyDown`, `keyUp`, `click`, `pointerMove`, `pointerDown`, `pointerUp`, `drag`, `dragToEdge` |

The safe column talks to the element **through the accessibility API** and synthesizes nothing. The dangerous column
calls `SendInput`, which is **global** — it injects into whatever window is foreground at that instant, not into the
app you launched. If the app takes focus late, loses it, or never gets it, the keystrokes and clicks land in the
developer's shell. This has already destroyed a user's live terminal sessions, and there is no undo.

**Never run a synthetic-input step on the developer's own desktop.** If a real desktop gesture is genuinely what you
are testing, run it in an isolated guest — `isolate --drive-script`, or `drive-shell` for a live Hyper-V session. If
neither is available, verify headlessly instead, or ask the user to run it and report what they see. An unverifiable
appearance claim is cheaper than an unrecoverable action.

**Never run `gate` on the developer's own desktop either.** It injects a real touch gesture into the host desktop, so
it is isolated-guest-only even though it is not a `drive` step.

The UIA pattern verbs are the deterministic assertion anyway, so the safe column is almost always what you actually
wanted.

## Dragging a row that only moves after a hold

When a real drag *is* the subject — inside a guest — a plain `drag` is not always enough. A row whose own content owns
the press (a rail button, a whole-row control) is made draggable by a **press-and-hold** instead of a drag threshold,
and such a row **disarms itself the moment it sees the pointer travel before its hold elapses**. `drag` starts moving
90 ms after the press, so it can never arm one.

Add the dwell explicitly — `--drag-hold-ms` on the flags, or `"holdMs"` in a script step — with a value above the app's
hold delay (Sprout's is 500 ms):

```json
{ "op": "find", "automationId": "sidebar.item.plex" },
{ "op": "drag", "automationId": "sidebar.item.emby", "holdMs": 700 }
```

The dwell is spent between the button-down and the first move, which is the only place a hold recognizer counts it; the
move cadence afterwards is unchanged.

## Reaching a context command

`--show-context-menu` opens the found element's context menu through the accessibility API
(`IRawElementProviderSimple2::ShowContextMenu`) — no synthetic input. It is the only safe way to reach anything behind a
right-click, and it matters more than it sounds: **a reorder has no UIA pattern at all**, so a list that can be
rearranged exposes that ability as ordinary commands, and those commands almost always live in a context menu.

```powershell
sprout-devtools drive <app> `
  --find-automation-id "sidebar.item.plex" --show-context-menu --artifacts out
sprout-devtools drive <app> --script move-up.json --artifacts out
```

```json
[
  { "op": "find", "automationId": "sidebar.item.plex" },
  { "op": "showContextMenu" },
  { "op": "find", "name": "Move up", "controlType": "MenuItem" },
  { "op": "invoke" },
  { "op": "snapshot" }
]
```

An element with no menu of its own defers to its UI Automation parent, per the provider contract; an element with none
anywhere up its chain reports a failed step rather than silently doing nothing.

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
