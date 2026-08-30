# Command reference

`sprout-devtools <command> [args] [options]`. Live `--help` is authoritative — `sprout-devtools <command> --help`
matches the installed build exactly. Do not truncate its output; guessing option names causes retry loops.

## Global options

| Option | Effect |
|---|---|
| `--json` | machine-readable JSON on stdout instead of a human table |
| `--artifacts <dir>` | where evidence goes (default `.sprout-devtools/<runId>`) |
| `-v`, `--verbose` | diagnostic logging on stderr |

**Pass a `--`-prefixed option *value* with `=`** — `--app-arg=--some-flag`, not `--app-arg --some-flag`, or it parses
as another option.

## Everyday verbs

| Verb | What it does |
|---|---|
| `doctor` | report the host (OS build, DPI awareness, isolation, Developer Mode) and steer to the right launch form. Run first when anything looks wrong. |
| `debug <app>` | build Debug, launch, screenshot + UIA tree dump, exit. The dump is triage evidence capped by default at depth 8 and 250 nodes. [screenshots.md](screenshots.md) |
| `capture <app>` | launch and screenshot only. An app that ships its own MSIX is launched under that identity automatically. |
| `record <app>` | bounded H.264/MP4 of a real window (`--duration-ms`, `--fps`, optional `--drive-script`). |
| `drive <app>` | find a control, act on it, assert its state; `--capture-target owned-popup` captures a separate popup opened by the action. [interaction.md](interaction.md) |
| `selftest <app>` | submit the app's own `--selftest` to the shared target host and assert its token line (`=> PASS`, `leaked=0`, exit 0). `--target auto` prefers managed VM, then Sandbox, then localhost; `--target vm|sandbox|local` is explicit and `direct` bypasses coordination. |
| `host serve|status|selftest|cancel|stop` | inspect or control the per-user queue. Ordinary `stop` refuses while work is active; `--force` is the explicit destructive form. |
| `build <project>` | publish a project into a self-contained deployment directory (`--artifacts out` → `out\publish\`). |
| `run <app\|dir\|project>` | the capstone: `selftest` + capture. `--form auto` is the default — an app that ships MSIX runs packaged under its real identity, anything else runs unpackaged. `--form exe\|msix\|both` names the form instead; `both` adds a parity verdict. |
| `inspect list \| connect \| snapshot \| export` | discover and read runtime snapshots from an opted-in app. [layout-inspection.md](layout-inspection.md) |

## Packaging and isolation verbs

| Verb | What it does |
|---|---|
| `produce <publishDir>` | pack a published directory into a signed MSIX (+ cert). |
| `deploy <app>` | make an app **resident** (lay down the publish, or install/register the package) and leave it there. `--form auto` is the default, same detection as `run`. Its launch is a transient verify — the directory stays, not a live window. |
| `generate assets <image>` | turn one image into the full Windows/MSIX visual-asset set plus a multi-size `app.ico`. |
| `isolate <app> --devtools <dir>` | run the checks inside a clean Windows Sandbox or Hyper-V guest and lift the verdict. The right place for synthetic input. |
| `vm image import` / `vm pool ensure|status|repair` | import verified Windows media and provision/resume owned Hyper-V members with resident target servers and exact checkpoint teardown. |
| `drive-shell <app> --devtools <dir> [--pool <name>]` | live multi-step host↔guest drive channel (Hyper-V only). `--pool` consumes a managed pool; the legacy environment-configured VM path remains available. |
| `agent <app> --serve` | the guest-internal resident agent-server; not a host command. |
| `gate <app>` | gate a real post-compositor touch-scroll gesture. |

The synthetic package paths used by `produce` and packaged `run` / `deploy` need MakeAppx and, when signing, SignTool
from an installed Windows SDK or the `Microsoft.Windows.SDK.BuildTools` package cache. Loose packaged deployment also
needs Developer Mode. `isolate` needs the Windows Sandbox optional feature or a configured Hyper-V VM; `drive-shell`
is Hyper-V-only. Its interactive JSONL operations include `find`, `act`, `read`, `snapshot`, `capture`, `query`,
`display`, `resizeClient`, stateful key/pointer input, and `shutdown`. `display` queries or changes the guest's active
desktop mode; `resizeClient` succeeds only after the measured client size exactly matches.

`gate` injects real touch into the host desktop. Run it only inside an isolated guest, never on the developer's desktop.

## Orchestration result contract

Every orchestration command (all verbs except `inspect`) writes one `result.json` (even on harness error), schema
`sprout.devtools.orchestration.result.v1`:

```
run { id, status, startedUtc, durationMs, toolVersion,
      host { os, build, dpiAwareness, isolation, isolationTarget,
             target, targetSelection, targetReason, gpu },
      app  { path, form, formSelection, identitySource, packageFamilyName, aumid, arguments } }
tests[] { id, title, status, durationMs, steps[],
          assertions[] { kind, status, expected, actual, message },
          artifacts[]  { kind, path, sha256 } }
artifactsRoot
```

`app.form` is `unpackaged` or `packaged` — which identity form the app actually ran in. `app.formSelection` is `auto`
(detected from the app) or `explicit` (you named it), and for a packaged run `app.identitySource`
(`projectPackage` / `looseLayout` / `synthetic`) and `app.aumid` say whose identity was used. Read those instead of
guessing from which checks appear; they are filled in on harness errors too, so a packaged attempt that failed still
reports itself as packaged. Keys with no value are omitted.

`inspect` commands use the separate `sprout.devtools.cli.result.v1` envelope, and `inspect export` writes a
`sprout.devtools.snapshot.v1` artifact.

Status is lowercase `passed` / `failed` / `skipped` / `errored`; the overall status is worst-wins.

| Exit code | Meaning |
|---|---|
| `0` | pass |
| `1` | assertion failed |
| `2` | harness error |

**Read `tests[].assertions[]` and `artifacts[]`, not stdout prose.**

The `inspect` envelope returns 0 on success. Its failures use 2 for invalid input, 3 for discovery/timeout/I/O, 4 for
access denial, and 5 for protocol or state errors.

## What is a gate and what is not

| Evidence | Deterministic gate? |
|---|---|
| `selftest` token line | **yes** |
| UIA `--expect-*` assertions | **yes** |
| `inspect snapshot` data | **yes** (it is framework truth, not a rendering) |
| screenshot / `recording.mp4` | **no** — post-compositor, varies with GPU/driver/DWM/wallpaper. Triage only. |
