# Check the accessibility tree

Sprout draws its own controls, so nothing is accessible unless the framework publishes it. Checking the UI Automation
tree is how you verify that a screen reader, Narrator, or any assistive technology can actually reach the UI.

## Get a tree

Two routes, both writing an indented `ControlType "Name"` dump:

```powershell
# one-shot, alongside a screenshot
sprout-devtools debug <app> --from-source --artifacts out      # -> out\debug-uia-tree.txt

# as part of an interaction, with a structured JSON form too
sprout-devtools drive <app> --find-name "…" --invoke --artifacts out
                                                                # -> out\uia-tree.txt + out\uia-snapshot.json
```

`drive` bounds its evidence with `--snapshot-depth` (default 8, 250 nodes max). Raise it when the tree you care about
is deeper; the bound never affects the `--find-*` gate itself.

## What to look for

| Check | Why it matters |
|---|---|
| Every interactive control appears | An element absent from the tree is invisible to assistive technology, no matter how it looks. |
| Control types are correct | A button exposed as `Text` cannot be invoked; `Group`/`List`/`ListItem` structure is what conveys grouping. |
| Names are meaningful | `Button ""` is unusable. A name must say what the control does, not what it is. |
| Structure is not flat | Related controls should nest under a container, not sit as siblings of everything else. |
| No duplicate names in one scope | Ambiguous names break both navigation and `--find-name`. |

Assert a specific expectation with `drive` rather than eyeballing:

```powershell
sprout-devtools drive <app> --find-name "Volume" --find-type Slider --expect-enabled true
```

A `find` that fails is itself the finding: the control is not exposed, or not exposed under that name/type.

## Also use the tree to validate a run

`debug` reports `passed` whenever *any* window was captured, including a .NET crash dialog. `debug-uia-tree.txt` is how
you tell the difference: if the root is a dialog or an error box, or the tree holds a handful of generic controls where
your real UI should be, the run failed. Read it every time before reporting success. See
[SKILL.md, trap 1](../SKILL.md).

## Not the same thing as the layout snapshot

The UIA tree is what the **outside world** can see. [`inspect snapshot`](layout-inspection.md) is the framework's
**internal** truth — its `semanticsNode` runtime nodes are the source the UIA tree is projected from. Use the tree for
"can a screen reader reach this"; use the snapshot for "is this the right size and position".
