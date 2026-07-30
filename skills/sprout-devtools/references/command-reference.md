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
| `doctor` | report the host (OS build, DPI awareness, GPU, isolation, Developer Mode) and steer to the right launch form. Run first when anything looks wrong. |
| `debug <app>` | build Debug, launch, screenshot + UIA tree dump, exit. The default "look at it" loop. [screenshots.md](screenshots.md) |
| `capture <app>` | launch and screenshot only. |
| `record <app>` | bounded H.264/MP4 of a real window (`--duration-ms`, `--fps`, optional `--drive-script`). |
| `drive <app>` | find a control, act on it, assert its state. [interaction.md](interaction.md) |
| `selftest <app>` | run the app's own `--selftest` and assert its token line (`=> PASS`, `leaked=0`, exit 0). Requires the app to opt into that convention. |
| `build <project>` | publish a project into a self-contained deployment directory (`--artifacts out` → `out\publish\`). |
| `run <app\|dir\|project>` | the capstone: `selftest` + capture, for the exe form, the MSIX form, or `--form both` with a parity verdict. |
| `inspect list \| connect \| snapshot \| export` | discover and read runtime snapshots from an opted-in app. [layout-inspection.md](layout-inspection.md) |

## Packaging and isolation verbs

| Verb | What it does |
|---|---|
| `produce <publishDir>` | pack a published directory into a signed MSIX (+ cert). |
| `deploy <app>` | make an app **resident** (lay down the publish, or install/register the package) and leave it there. Its launch is a transient verify — the directory stays, not a live window. |
| `generate assets <image>` | turn one image into the full Windows/MSIX visual-asset set plus a multi-size `app.ico`. |
| `isolate <app> --devtools <dir>` | run the checks inside a clean Windows Sandbox or Hyper-V guest and lift the verdict. The right place for synthetic input. |
| `drive-shell <app> --devtools <dir>` | live multi-step host↔guest drive channel (Hyper-V only). |
| `agent <app> --serve` | the guest-internal resident agent-server; not a host command. |
| `gate <app>` | gate a real post-compositor touch-scroll gesture. |

## Result contract

Every command writes one `result.json` (even on harness error), schema
`sprout.devtools.orchestration.result.v1`:

```
run { id, status, startedUtc, durationMs, toolVersion,
      host { os, build, dpiAwareness, isolation, gpu },
      app  { path, arguments } }
tests[] { id, title, status, durationMs, steps[],
          assertions[] { kind, status, expected, actual, message },
          artifacts[]  { kind, path, sha256 } }
artifactsRoot
```

`inspect` commands use the separate `sprout.devtools.cli.result.v1` envelope, and `inspect export` writes a
`sprout.devtools.snapshot.v1` artifact.

Status is lowercase `passed` / `failed` / `errored`; the overall status is worst-wins.

| Exit code | Meaning |
|---|---|
| `0` | pass |
| `1` | assertion failed |
| `2` | harness error |

**Read `tests[].assertions[]` and `artifacts[]`, not stdout prose.**

## What is a gate and what is not

| Evidence | Deterministic gate? |
|---|---|
| `selftest` token line | **yes** |
| UIA `--expect-*` assertions | **yes** |
| `inspect snapshot` data | **yes** (it is framework truth, not a rendering) |
| screenshot / `recording.mp4` | **no** — post-compositor, varies with GPU/driver/DWM/wallpaper. Triage only. |
