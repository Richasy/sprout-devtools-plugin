# Verify layout, spacing and sizes

> Read the framework's **own** layout truth out of the running process. This beats reading source code: a static scan
> of layout code finds candidates, a snapshot finds the actual defect.

## When to use this instead of a screenshot

A screenshot tells you something looks off. A **snapshot** tells you which node, by how many pixels, and whether the
node was clipped or merely positioned oddly. Reach for it whenever the question involves a number — spacing, width,
padding, alignment, "is this clipped", "why is there a gap".

## Prerequisite: the app must expose the diagnostics endpoint

`inspect` only reaches apps that **explicitly opted in**. Two separate things are required, and the first without the
second silently produces no target:

1. **Build-time** — the instrumentation must be linked in. Either reference the `Sprout.Diagnostics` package (which
   sets it automatically) or set the MSBuild property in the app's `.csproj`:

   ```xml
   <PropertyGroup>
     <SproutDiagnosticsEnabled>true</SproutDiagnosticsEnabled>
   </PropertyGroup>
   ```

   This is a **linker feature switch**, evaluated once for the process. It defaults to `false` so a shipping build
   carries no diagnostics code at all — you cannot turn it on at runtime.

2. **Run-time** — the app must open the local endpoint:

   ```csharp
   _ = RuntimeDiagnostics.StartLocalAsync(application, new RuntimeDiagnosticsLocalOptions { /* … */ });
   ```

   `RuntimeDiagnostics.Start(...)` alone starts a session but opens **no endpoint** — `inspect` will not see it.

If either is missing, `inspect list` returns an empty `targets` array. That is the diagnosis, not an error.

## Find the target

```powershell
sprout-devtools inspect list
```

Each entry is a live target: `instanceId` (a GUID), `processId`, `name`, `version`, `capabilities`. Targets publish
themselves as one JSON descriptor per session under `%LocalAppData%\Sprout\DevTools\sessions` (override with
`--sessions <dir>`); the directory is ACL-restricted to the current user.

A `<target>` argument accepts **either** the instance GUID **or** the PID. If one PID hosts more than one session the
command fails with `targetAmbiguous` and you must pass the GUID.

Remember [SKILL.md trap 2](../SKILL.md): `debug` exits the app, so it leaves nothing to inspect. Start the app
yourself and keep it running.

## Capture

```powershell
sprout-devtools inspect snapshot <pid> > snapshot.json
# or write it atomically as a schema-tagged artifact:
sprout-devtools inspect export <pid> --output snapshot.json
```

`snapshot` prints the whole capture to stdout; `export` writes a `sprout.devtools.snapshot.v1` artifact file and prints
only a summary. `connect` performs the handshake only and returns no tree — use it to prove reachability.

`inspect` uses its own envelope: `{ "schema": "sprout.devtools.cli.result.v1", "success": …, … }`, and on failure
`{ "success": false, "error": { "code": "targetNotFound", "message": "…" } }`. **Branch on `success` and `error.code`**,
not on stdout text.

### Wait for `status: complete` — a freshly launched app looks empty

A cold app does not become inspectable instantly, and the intermediate state is **not** an error — it is a valid
snapshot with almost nothing in it. Measured on a trivial app:

| Time since launch | Result |
|---|---|
| ~0.3 s | `error.code: targetNotFound` — the endpoint is not published yet |
| ~1.5 s | `status: partial`, **1 node, 0 surfaces, 0 with `desiredSize`** |
| ~2.6 s onward | `status: complete`, full tree |

**If you snapshot too early you will see a root with a correct `dpiScale` and `clientBounds` but an empty node list,
and conclude the UI is empty. It is not.** Always check `response.status` first and re-capture until it is `complete`:

```powershell
function Get-SproutSnapshot([int]$TargetPid, [int]$TimeoutSec = 30) {
    $deadline = (Get-Date).AddSeconds($TimeoutSec)
    while ((Get-Date) -lt $deadline) {
        $raw = (sprout-devtools inspect snapshot $TargetPid 2>&1 | Out-String).Trim()
        $s = $raw | ConvertFrom-Json
        if ($s.success -and $s.response.status -eq 'complete') { return $s }
        Start-Sleep -Milliseconds 400
    }
    throw "no complete snapshot from pid $TargetPid within $TimeoutSec s"
}
```

A real application with heavier startup takes longer; raise the timeout rather than accepting a `partial`.

## The snapshot shape

One compact camelCase JSON line. The parts that matter:

```
response.status                          "complete" | "partial" | "notReady" | "resyncRequired"
response.snapshot.roots[]                one per Window
  .dpiScale                              e.g. 2.5
  .clientBounds  {x,y,width,height}      the window client area, in epx
  .runtimeNodes[]                        the layout facts  ← what you came for
  .logicalElements[]                     the authored element graph
  .treeEdges[]                           parent/child linkage
  .surfaces[]                            composition surfaces
response.snapshot.sourceDescriptors[]    control type + source location  ← NOTE: a sibling of roots, not per-root
```

> **`sourceDescriptors` lives on `response.snapshot`, not inside a root.** It is interned once per capture and shared
> by every root.

### A runtime node

```json
{ "handle": { "kind": "widget", "value": 9 }, "state": "bound", "role": "render",
  "logicalElementId": 1, "bindingEpoch": 1,
  "bounds": { "x": 184, "y": 38, "width": 112, "height": 18.8 },
  "desiredSize": { "width": 112, "height": 18.8 },
  "flags": 0, "dirtyFlags": 3 }
```

- **`handle` is the identity** — `{kind, value}`. There is no separate `id`. `kind` is one of `control`,
  `projectedSlot`, `widget`, `interactionNode`, `semanticsNode`, `window`, `popupWindow`, `layer`, `surface`,
  `nativeBoundary`.
- **`bounds`** is the arranged rectangle: where the node actually ended up.
- **`desiredSize`** is what measure asked for. It is present **only on `widget` nodes** — the layout participants.
- **There is no `parentId`/`children`.** Linkage is the separate `treeEdges` array:
  `{ tree, parent: {kind,value}, child: {kind,value}, index }`, where `tree` is `control` / `widget` / `interaction` /
  `semantics` / `composition`.
- **`logicalElementId`** is the only bridge to naming: node → `logicalElements[].id` →
  `logicalElements[].sourceDescriptorId` → `sourceDescriptors[].displayName`.

### Units: already epx — do **not** divide by `dpiScale`

`bounds` and `desiredSize` are in **effective pixels (epx)**, the same unit you author layout in. The conversion from
physical pixels happens inside the app *before* the value is put into the snapshot.

`dpiScale` is reported alongside as an **independent fact**, for when you need to derive a physical value yourself. It
is not a correction to apply. The proof is in the same capture: a window authored `480x320` reports
`clientBounds` `480x320` and `surfaces[].physicalSize` `1200x800` at `dpiScale: 2.5` — `physicalSize` is
`bounds x dpiScale`, so `bounds` is already the epx side.

**Dividing `bounds` by `dpiScale` is the single most common way to invent a layout bug that does not exist.**

## The defect test: `bounds` vs `desiredSize`

Comparing a widget's arranged `bounds` against its measured `desiredSize` is the reliable screen for layout defects,
and it is far more trustworthy than grepping layout source. Screen with `!=`, then triage:

| Observation | Meaning |
|---|---|
| `bounds.width < desiredSize.width` (or height) | **Defect.** The node was given less room than it needs — clipped, squeezed, or overflowing its parent. This is what you are hunting. |
| `bounds > desiredSize` | Usually **intentional** — a stretch alignment or a fill. Confirm against the authored intent before reporting it. |
| `bounds` is `0x0` while `desiredSize` is not | The node was collapsed or never arranged. Check whether that is deliberate. |
| both `0x0` | Normal for a collapsed node — see below. |

### Screening a whole window

```powershell
$snap = Get-SproutSnapshot <pid>          # the readiness helper above
$root = $snap.response.snapshot.roots[0]

$desc = @{}; foreach ($d in $snap.response.snapshot.sourceDescriptors) { $desc[[int]$d.id] = $d }
$elem = @{}; foreach ($e in $root.logicalElements) { $elem[[int]$e.id] = $e }

function Get-NodeLabel($node) {
    if ($null -eq $node.logicalElementId) { return '(unnamed)' }
    $el = $elem[[int]$node.logicalElementId]
    if (-not $el -or $null -eq $el.sourceDescriptorId) { return '(unnamed)' }
    $d = $desc[[int]$el.sourceDescriptorId]
    # A site descriptor's own displayName is just the site ("Build"); qualify it with the owning type.
    if ($null -ne $d.ownerDescriptorId -and $desc.ContainsKey([int]$d.ownerDescriptorId)) {
        return "$($desc[[int]$d.ownerDescriptorId].displayName).$($d.displayName)"
    }
    return $d.displayName
}

$root.runtimeNodes |
  Where-Object { $_.bounds -and $_.desiredSize } |
  ForEach-Object {
      [pscustomobject]@{
          Name    = Get-NodeLabel $_
          X       = $_.bounds.x;          Y     = $_.bounds.y
          W       = $_.bounds.width;      H     = $_.bounds.height
          WantW   = $_.desiredSize.width; WantH = $_.desiredSize.height
          Starved = ($_.bounds.width -lt $_.desiredSize.width) -or ($_.bounds.height -lt $_.desiredSize.height)
      }
  } |
  Sort-Object -Property Starved -Descending | Format-Table -AutoSize
```

Every row with `Starved = True` is a real candidate. Then use `treeEdges` to walk up to the parent and see *which*
container did the starving, and the descriptor's `source` (`documentUri`, `line`, `column` — project-relative,
zero-based) to jump straight to the authored code.

> Only `widget` nodes carry `desiredSize`, so this table covers the layout participants — typically a small fraction of
> `runtimeNodes`. That is expected; `window`, `projectedSlot`, `interactionNode` and `semanticsNode` entries have no
> measure step to compare against.

### Checking a specific measurement

For "is the sidebar 280 epx wide?" or "is the gap 12 epx?", you do not need the whole tree: find the node by its
descriptor `displayName`, read `bounds.width`, and for a gap subtract one sibling's `bounds.x + bounds.width` from the
next sibling's `bounds.x`. Both operands are epx, so the difference is directly comparable to the authored value.

## Collapsed nodes: `height: 0` is usually fine

A node with `Visibility.Collapsed` (`.Visible(false)`) is removed from layout entirely — CSS `display: none`. It
**still appears** in `runtimeNodes`, with `bounds` and `desiredSize` both zeroed. **That is normal, not an empty
container, and not a defect.** Do not report it.

One nuance that will otherwise fool you: a collapsed **ancestor** short-circuits the layout walk and does not re-arrange
its children, so a *descendant* of a collapsed subtree keeps its **stale, last non-zero** bounds rather than zero. So a
node with plausible-looking bounds may not be on screen at all. Before trusting a node's numbers, confirm no ancestor
(via `treeEdges`) is collapsed.

## Partial captures

`response.status` can be `partial` when the capture hit a budget, and each root carries a `completeness` block. A
`partial` snapshot is still valid for the nodes it contains — but do not conclude "the node is missing, therefore the
control is missing" from one. Check `status` before reasoning about absence.
