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
running process, and drives controls through UI Automation. Check commands emit structured result JSON; management
commands and interactive/server commands have their own documented output contracts.

Use it whenever the request is some form of *"is the UI right?"*:

- "show me what this screen looks like" / "did my change actually work visually"
- "check the sidebar spacing / this element's size / whether something is clipped"
- "is this reachable by a screen reader" / "dump the accessibility tree"
- "click that button and tell me what happened"
- "run this safely while other worktrees are testing" / "drive this app step-by-step in a managed VM"

**Never hand-roll this.** Do not write `PrintWindow` / `System.Drawing` / `Add-Type` screenshot scripts, or ad-hoc UIA
dumps. They dead-end on Sprout's `WS_EX_NOREDIRECTIONBITMAP` windows (a pure DirectComposition surface that ordinary
GDI capture returns blank for) and they cost far more time than this tool.

## Install (once per machine)

For VM tests submitted by ordinary desktop clients, a self-contained DevTools payload can install the optional
elevated relay with `host relay install`. Review its manifest with `host relay install --plan-only` first. Installation
requires one Windows UAC approval and verifies an ordinary client connection before enabling routing. The current
user's logon Scheduled Task runs the fixed administrator-protected payload with highest privileges; subsequent
explicit `selftest --target vm` and managed `isolate --backend vm` requests use it without elevating the desktop client.

The installed task uses the published `Sprout.DevTools.Relay.exe` GUI-subsystem apphost for the same CLI payload.
Logon and on-demand startup allocate no console window. Keep that apphost with the complete published payload;
`--background` alone does not hide a console executable.

The relay forwards opaque messages to private VM-only workers. A persistent session under the broker state root,
partitioned by workspace, keeps query/cancel requests with the correct worker after reconnect. Connection sequences
correlate individual responses. Different workspaces or broker roots receive separate worker scopes. Do not reuse
another workspace's session descriptor. If submission disconnects, its outcome is unknown; do not automatically
resubmit. Before submission, an unavailable installed relay is started through its exact owned Scheduled Task.
Three failed starts exhaust a shared persistent budget; later clients use the conventional VM route until relay
readiness is observed, which resets the budget. This route still requires normal Hyper-V permissions and never
changes the target to Host or Sandbox. An accepted or uncertain submission is never replayed. Queued packaged isolate requires existing
explicit `--msix` and `--cert` inputs; do not rely on implicit host package creation. Cancellation retains evidence and
requires verified rollback and service cleanup. Interactive `drive-shell`, caller-owned VM profiles, legacy VM
configuration, and VM maintenance are separate routes. See
[elevated relay](https://github.com/Richasy/Sprout/blob/main/docs/guide/devtools-elevated-relay.md).

```powershell
dotnet tool install -g Sprout.DevTools --prerelease
sprout-devtools doctor
```

Requires a **.NET 10** runtime on PATH. The tool package brings its own managed dependencies; packaging and isolation
verbs still require the host capabilities listed in [command-reference.md](references/command-reference.md). Upgrade
with `dotnet tool update -g Sprout.DevTools --prerelease`. If `sprout-devtools` is not found after installing, the shim
directory (`%USERPROFILE%\.dotnet\tools`) is not on PATH for this shell; open a new shell or prepend it.

The self-contained payload delivered to managed guests keeps the CLI apphost compatible with the supported Windows 10
build 19041 floor by opting that apphost out of CET compatibility. Do not copy that tool-specific setting into a target
application; applications retain their own CET policy.

`doctor` reports the host (OS build, DPI awareness, Developer Mode, isolation) and is the right first move whenever a
capture looks wrong.

## Pick the workflow

| What you were asked | Do this | Detail |
|---|---|---|
| "What does it look like?" / "did my change land visually?" | `debug <project> --from-source` — build Debug, launch, screenshot **and** dump the UIA tree in one shot | [screenshots.md](references/screenshots.md) |
| "Capture the app exactly where it is now" | `capture --pid <pid>` — attach to the current process/window without launch, activation, foregrounding, input, resize, or termination | [screenshots.md](references/screenshots.md) |
| "Identify its windows, DPI, and screen coordinates" | `capture --pid <pid> --list-windows --json` — process-generation-bound inventory including negative physical `x/y` coordinates | [screenshots.md](references/screenshots.md) |
| "Inspect a tooltip in an existing process" | `drive --pid <pid> --script <json> --capture-target owned-popup` — find/hover/wait/read-only assertions; hover activates the target and moves the pointer, while lifecycle remains caller-owned | [interaction.md](references/interaction.md) |
| "Capture its separate popup/dialog" | `capture --pid <pid> --window-title <exact-title>` or `--hwnd <decimal-or-0x-hex>` — explicitly select one visible owned top-level window without changing default capture or experiment authority | [screenshots.md](references/screenshots.md) |
| "Is the layout / spacing / size right?" | Launch the app resident, then `inspect snapshot <pid>` and compare each node's `bounds` against its `desiredSize` | [layout-inspection.md](references/layout-inspection.md) |
| "Is it accessible / what does a screen reader see?" | `debug` and read `debug-uia-tree.txt`, or a `drive` script with a `snapshot` step | [accessibility.md](references/accessibility.md) |
| "Click it and check what happened" | `drive <app> --find-name … --invoke --expect-…` (UIA patterns only); add `--capture-target owned-popup` when the result is a separate Flyout/menu HWND | [interaction.md](references/interaction.md) |
| "Measure an existing app process unattended" | `experiment --pid <pid> --plan <json>` for arbitrary allowlisted UIA actions, `--scroll-automation-id <id>` for an in-memory scroll sweep, or `--metrics-only` for a passive control — prepare one command-owned attached session before warm-up, leave target lifecycle/window ownership with the caller, and sample exact-PID process plus optional PDH GPU/runtime metrics | [command-reference.md](references/command-reference.md) |
| "Does it still pass its own checks?" | `selftest <app>` — the deterministic token gate | [command-reference.md](references/command-reference.md) |
| "Run .NET tests on an approved target" | Publish an MTP test executable with TRX reporting. Prefer `test <tests.exe> --target vm --target-tool-dir <tools>` or exact-tool `isolate`; when host execution is separately authorized, use `test <tests.exe> --target host --allow-host`. Every route requires non-empty passing tests and exact-process cleanup; there is no fallback | [command-reference.md](references/command-reference.md) |
| "Run a native or console test safely" | `process-test <tests.exe> --target vm --target-tool-dir <tools> --success-token <literal>` — exact whole-line stdout token + exit zero + bounded output + payload/process/VM cleanup proof, with no host fallback | [command-reference.md](references/command-reference.md) |
| "My app is MSIX — run it that way" | Nothing extra: `run` / `deploy` / `drive` / `capture` detect it and launch under package identity | [command-reference.md](references/command-reference.md) |
| "Test-sign an original MSIX and update its resident identity" | Explicit `produce --sign-existing-msix`, then `deploy --allow-package-update --no-launch`; ordinary produce/deploy never infer that authority | [package updates](../../../docs/guide/devtools-package-updates.md) |
| "Run concurrent worktrees without desktop conflicts" | `selftest <app>` uses the shared target host; when that broker must be worktree-owned, combine a private `--target-state-dir` with the existing pool's `--pool-state-dir` | [command-reference.md](references/command-reference.md) |
| "Use an existing Hyper-V VM from another provisioner" | Supply `--existing-vm-profile` to explicit VM `selftest`, `test`, `process-test`, `isolate`, or `drive-shell`; bind VM/checkpoint IDs and credential references without adopting the VM | [command-reference.md](references/command-reference.md) |
| "Drive a real app adaptively inside a managed VM" | `drive-shell <publish-dir> --devtools <tool-dir> --pool default`; add `--capture-backend WindowsGraphicsCapture` when compositor pixels are required, and use guest-only `setContrastTheme` for live OS contrast changes | [interaction.md](references/interaction.md) |
| Something failed and you don't know why | [troubleshooting.md](references/troubleshooting.md) | |

Full verb list, global options, result schema and exit codes: [command-reference.md](references/command-reference.md).

## Parallel worktrees and managed targets

Ordinary `selftest` calls share one per-user host. `--target auto` prefers a Ready managed Hyper-V member, then a
configured Windows Sandbox, then localhost. A busy isolated target stays queued instead of silently spilling onto the
developer desktop:

```powershell
sprout-devtools selftest <app.exe>
sprout-devtools host status --json
```

When the shared broker cannot run the exact current CLI contract, isolate only
the broker while retaining the authorized managed pool:

```powershell
sprout-devtools test <tests.exe> --target vm --pool default `
  --target-state-dir <worktree-broker-state> `
  --pool-state-dir <existing-pool-state> `
  --target-tool-dir <exact-self-contained-tools>
```

If the current machine is explicitly approved for the test payload, select it
with both authority-bearing options:

```powershell
sprout-devtools test <tests.exe> --target host --allow-host `
  --timeout 120 --artifacts <host-test-artifacts> --json
```

This is a direct first-party MTP/TRX run with locked payload identity, the same
bounded observed-process runner, and retained PID/creation-FILETIME/image exit
proof. It rejects VM routing options and never activates because VM selection
failed.

The first root owns the secure pipe, host lock, journal, and job artifacts. The
second is read only as the selected pool manifest/payload/credential-reference
root. The request, receipt, journal, and result bind the normalized pool root;
an older host or unsupported backend fails before VM execution. Global per-VM
leases, immutable VM/checkpoint validation, cancellation, service cleanup, and
rollback remain unchanged. Omit `--pool-state-dir` for the compatible
single-root behavior.

Native and other console tests use the same explicit managed-target route:

```powershell
sprout-devtools process-test <tests.exe> --target vm --pool default `
  --target-state-dir <worktree-broker-state> `
  --pool-state-dir <existing-pool-state> `
  --target-tool-dir <exact-self-contained-tools> `
  --success-token NATIVE-CONSOLE-TESTS-PASSED `
  --test-arg=--mode --test-arg strict
```

The token is a bounded literal whole line, not a regex or substring. Exactly
one stdout match, no stderr match, exit zero, bounded output, the locked
payload, retained PID/creation-FILETIME/image identity, target-service cleanup,
and VM rollback are all required. A token printed before a crash or hang still
fails. This command has no auto, Sandbox, local, direct, or host fallback and
does not weaken `selftest` or Microsoft.Testing.Platform `test`.

Create a managed pool from official or licensed Windows media whose SHA-256 you already trust:

```powershell
sprout-devtools vm image import <windows.iso> --sha256 <hash> --source licensedByol
sprout-devtools vm pool ensure default --image <hash> --image-index <index> `
    --members 2 --devtools <self-contained-tool-dir>
```

The guest tool directory must be a complete self-contained Windows publish, including its .NET runtime; the
framework-dependent global tool installation is not a guest delivery directory. Before provisioning, optional
`vm image prepare <hash> --extractor <absolute-local-reader.exe>` can extract the trusted ISO with an explicitly
selected local reader such as 7-Zip. Run extraction unelevated; the immutable hash-verified cache lets later
provisioning avoid a virtual DVD mount. A reader or extraction failure stops preparation without a download fallback.
Provisioning and maintenance still require administrator authority; the installed relay forwards VM test jobs.

For an adaptive real-app session, `drive-shell --pool default` leases one member, copies the published app, and exposes
live JSON operations for UIA, screenshots, display mode, exact window resize, window enumeration, and shutdown. The
lease is held through artifact collection, exact checkpoint restore, and final power-off.
Live captures default to `Auto`. Pass `--capture-backend WindowsGraphicsCapture` to require the compositor path for
all shell-initiated and failure-evidence captures, or override one request with
`{"op":"capture","args":{"backend":"printWindow"}}`. Explicit WGC failure remains an operation failure and never
substitutes PrintWindow. Each successful capture reports both `requestedBackend` and actual `backend`, and writes a
JSON metadata file beside its numbered PNG.

Caller-owned existing VMs use `--existing-vm-profile`, not fabricated pool membership or name-only environment
variables. The profile pins immutable VM/checkpoint IDs, explicit restore-and-power-off authority, and namespaced
Credential Manager references. It is mutually exclusive with an explicitly selected `--pool`; `selftest` additionally
requires `--target vm`. Busy targets queue, and an executed failure never retries another VM or the host.
After a submitted selftest times out, inspect `targetJobId`, `targetJobTerminal`, and `targetJobCleanupVerified`.
Do not remove run-owned credentials or profiles until terminal cleanup is proven; retain them for recovery otherwise.
For a private broker owned by the current run, capture its status PID,
`processCreationFileTime`, and payload SHA-256, then use strict `host stop`
with all three expected values plus `--require-exit`. Trust only the returned
generation-bound JSON receipt; never stop by state-root reuse, process name,
or PID disappearance.

For a DevTools brick whose self-contained payload intentionally differs from a
foreign resident target host, use
`isolate <tests.exe> --portable-test --backend vm --devtools <local-tool-dir>`.
It bypasses only that resident controller: the authenticated guest worker,
global per-VM lease, strict TRX/payload/process checks, service cleanup, and
checkpoint rollback remain mandatory. Never stop or replace another session's
host to make a local build compatible. The exact requested test deadline is
separate from bounded connection, payload-validation, cleanup/report, and host
reply reserves; a passing result is revalidated against the host-selected
payload manifest and the authenticated service/lease receipt.

## Packaged apps run packaged, automatically

If your app ships as MSIX, it is launched **with package identity** by default — so notifications, background tasks,
file-type and protocol associations, `ApplicationData`, and single-instance activation all behave as they will for your
users. Nothing to pass: `run`, `deploy`, `drive` and `capture` detect it from the app itself (its own
`<AssemblyName>.msix` beside the publish output, or a build-produced loose AppX layout). An app that ships no package
runs as a plain executable, silently — that is the normal case and nothing complains about it.

Check what actually happened in `result.json`:

```
run.app.form            "packaged" | "unpackaged"
run.app.formSelection   "auto" (detected) | "explicit" (you named it)
run.app.identitySource  "projectPackage" | "looseLayout" | "synthetic"
run.app.aumid           the identity that was activated
```

Override it when you need to: `--form exe` (or `--packaged:false` for `drive`/`capture`) forces the plain executable,
`--form msix` forces the packaged form. **A packaged launch never silently degrades** — if registration, certificate
trust or activation fails, the run errors with the reason instead of quietly running the raw `.exe`.

An explicit signed `--msix` keeps the identity declared by its bounded manifest, including its application ID.
Do not repeat an executable-derived or synthetic name for a differently named package. Recording and the other signed
consumers reject malformed application manifests and executable mismatches before installation.

For a Sprout app, MSIX is opted into with `<SproutPackageFormat>Msix</SproutPackageFormat>` plus the `Sprout.Packaging`
and `Microsoft.Windows.SDK.BuildTools` package references; `Package.appxmanifest` is an optional customization on top.
Packaging happens at publish time, so `run <project>` (which publishes) produces and then uses the package in one step.
Use `deploy` rather than repeated `run` when the loop needs settings or granted capabilities to survive between
iterations — `run` removes its development registration each time.

For a reusable Developer-Mode package, deploy a build-produced loose layout into a separate stable resident directory:

```powershell
sprout-devtools deploy out\AppX --package-dir out\resident-package --keep-open --artifacts out\deploy --json
```

Leave `--form` omitted so the real loose manifest is auto-detected; explicit `--form msix` is the synthetic-package
route.

Close the prior app before the next update. The identity-scoped operation refuses Store/non-development packages,
publisher/version changes, ambiguous registrations, and any active process belonging to the existing package or
running from its layout. It stages and hashes the new payload before replacing the stable directory, re-registers the
same version without uninstalling, and preserves application data. Post-swap failures restore/re-register the exact
previous layout or remove only a proven newly created Development-Mode registration; an unsafe rollback fails with
retained recovery paths. `--keep-open` also reports the exact retained process image SHA-256. `--no-launch` performs
only the update; omit both launch switches for a verification launch that exits. See
[`docs/guide/devtools-package-updates.md`](../../../docs/guide/devtools-package-updates.md).

### Original MSIX signing and an authorized signed resident update

Ordinary `produce <publishDir>` reuses an existing MSIX byte-for-byte, even when unsigned. Signing that original is a
separate explicit operation, not a reason to synthesize a replacement manifest:

```powershell
sprout-devtools produce out\App.msix --sign-existing-msix --artifacts out\signed --json
sprout-devtools deploy out\signed\signed-package.msix --allow-package-update --no-launch --artifacts out\update --json
```

Signing holds/hashes the original, signs only its exact copy, and proves every ordinary archive entry unchanged;
only `AppxSignature.p7x` and `[Content_Types].xml` may change. Read original/signed SHA-256, manifest SHA-256,
`packagePayloadPreserved`, public `signing.cer`/thumbprint, and `package-payload-sha256.txt`. Already signed input,
synthetic/no-sign options, and identity overrides fail. No PFX/password is evidence.

The signed update authority must be separately approved, including certificate trust when needed. Direct MSIX input
and `--allow-package-update` require `--no-launch`; the opt-in also accepts an app-owned signed MSIX in a publish
directory. It refuses loose/synthetic inputs/options, identity overrides, ambiguous current-user registrations,
publisher/family/architecture/resourceId mismatch, same-version/downgrade revisions, and active or ambiguous package
processes. Existing signed production/Store registrations are admitted only through this explicit path. First-party
package/certificate leases, immutable input verification, subprocess prior-tuple rechecks, and exact installed
signature verification remain mandatory. Never uninstall, delete app data, or stop a pre-existing process to help it.

Read `resident-signed-package.json` and `residentSignedPackageRecovery` on failure: a possible successor or unknown
state stays resident, certificate trust is retained unless unused session-added entries can be safely proven and
removed, and the command still fails. This does not relax transient fixture or reusable loose-package guards.

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

When you need a *resident* app — to snapshot it, to iterate, or to let a human look at it — prefer a DevTools command
that deliberately keeps the launched process alive:

```powershell
sprout-devtools drive <app> --find-name "Ready" --expect-enabled true --keep-open
sprout-devtools inspect list
sprout-devtools inspect snapshot <pid-or-instance-id>
```

Launch-mode `capture`, `record`, and `gate` also expose `--keep-open` for their own workflows. Attached
`capture --pid` already leaves the process untouched and rejects `--keep-open`. Use `deploy <project>` when the
**files or package registration** must remain resident; its ordinary verification launch exits, while a reusable loose
package may use `deploy --package-dir ... --keep-open` to retain the exact AUMID-launched process. Launch a deployed
executable directly only when that project declares a self-contained/AOT deployment, or launch it through its matching
`dotnet` runtime. A RID-specific `sprout-devtools build` does not itself guarantee self-containment.

Give the app a couple of seconds before snapshotting: a cold app briefly reports `status: partial` with an almost-empty
tree that reads as a real, empty UI. Poll until `response.status` is `complete` — see
[layout-inspection.md](references/layout-inspection.md).

### 3. Never synthesize input on the developer's own desktop

The `drive` steps `typeText` / `keyDown` / `keyUp` / `click` / `hover` / `pointerMove` / `pointerDown` / `pointerUp` / `drag` /
`dragToEdge` are implemented with **`SendInput`, which is global**: it injects into whatever window is foreground at
that instant, not into the app you named. The `gate` command likewise injects real touch into the host desktop. These
actions have already destroyed a user's live terminal sessions, irreversibly, and there is no undo.

The **UIA pattern** steps — `find`, `setFocus`, `invoke`, `toggle`, `setValue`, `expand`, `select`, and every `expect*`
— are safe: they act on the element through the accessibility API and synthesize nothing. They are also the
deterministic assertion anyway. If a real desktop gesture is genuinely the subject, run it in an isolated guest
(`isolate --drive-script`, `drive-shell`). See [interaction.md](references/interaction.md).

## Reading command results

Application check commands such as `build`, `selftest`, `capture`, `record`, `gate`, `drive`, `produce`, `run`,
`experiment`, `deploy`, `debug`, and `isolate` write one `sprout.devtools.orchestration.result.v1` `result.json` under
`--artifacts <dir>` (default `.sprout-devtools/<runId>`) and return:

| Exit code | Meaning |
|---|---|
| `0` | all assertions passed |
| `1` | an assertion failed (the app or the expectation is wrong) |
| `2` | skipped, errored, or another non-verdict status; read `run.status` and `tests[].status` |

**Read `tests[].assertions[]` for the verdict and `artifacts[]` for the evidence files — not stdout prose.** Add
`--json` to get the machine-readable form on stdout. Status values are lowercase `passed` / `failed` / `skipped` /
`errored`, and the overall status is worst-wins.

`doctor`, `host`, and `vm` are management commands and do not promise an orchestration `result.json`. Interactive
`drive-shell` and the guest-internal `agent` server have separate lifecycle/output contracts. `inspect` emits the
`sprout.devtools.cli.result.v1` envelope on stdout; it returns 0 on success, then 2 for invalid input, 3 for
discovery/timeout/I/O, 4 for access denial, and 5 for protocol or state errors.

A screenshot is **triage evidence, not a gate**: post-compositor capture depends on the GPU, driver, DWM and even the
wallpaper behind a transparent window. Use it to *see*; use assertions and snapshot data to *decide*.

## Reference index

| File | Read it when |
|---|---|
| [references/screenshots.md](references/screenshots.md) | Capturing a window, judging the image, blank or wrong-window frames |
| [references/layout-inspection.md](references/layout-inspection.md) | Verifying layout, spacing and sizes from framework truth; the snapshot JSON shape |
| [references/accessibility.md](references/accessibility.md) | Reading and checking the UI Automation tree |
| [references/interaction.md](references/interaction.md) | Acting on controls and asserting state; drive scripts |
| [references/command-reference.md](references/command-reference.md) | Every public verb, its options, and each output/result contract |
| [references/troubleshooting.md](references/troubleshooting.md) | A command failed, produced nothing, or found no target |
| [pitfalls.md](pitfalls.md) | The full DO / DON'T list |
