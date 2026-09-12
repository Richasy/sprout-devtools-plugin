# Troubleshooting

Work down this list before concluding a tool or an app is broken.

## `sprout-devtools` is not recognized

The tool is not installed, or its shim directory is not on this shell's PATH.

```powershell
dotnet tool install -g Sprout.DevTools --prerelease     # install
dotnet tool list -g | Select-String sprout              # confirm
$env:PATH = "$env:USERPROFILE\.dotnet\tools;$env:PATH"  # this shell only
```

A shell opened before the install will not see the shim. Open a new one.

## `You must install .NET to run this application` / a runtime error on launch

First identify which executable failed:

- The installed `sprout-devtools` tool and the in-repository `Sprout.DevTools` host are framework-dependent and need a
  **.NET 10** runtime. Check `dotnet --list-runtimes`. From source, run the host through the pinned runtime:
  `dotnet run --project tools\Sprout.DevTools\Sprout.DevTools.csproj -- <verb>`. Do not make the built apphost the
  normal invocation.
- A target produced by `sprout-devtools build` is RID-specific, but is self-contained only when its project declares
  `SelfContained` or `PublishAot`. If it is framework-dependent, launch it through the matching `dotnet` runtime or
  install that runtime; do not assume the `.exe` can run alone.
- The self-contained DevTools publish delivered to Sandbox/Hyper-V, and a self-contained/AOT target app, may be
  launched directly without `DOTNET_ROOT`.

If several .NET installations exist, point `DOTNET_ROOT` at the one carrying .NET 10.

## `inspect list` returns no targets

**First, give it a moment.** The endpoint is published a little after process start — an immediate call returns
`targetNotFound` even for a correctly configured app. Retry for a few seconds before concluding anything.

If it stays empty, the app has not opted into the diagnostics endpoint. Both halves are required, and the first
without the second produces exactly this symptom:

1. `<SproutDiagnosticsEnabled>true</SproutDiagnosticsEnabled>` in the app's `.csproj`, or a `Sprout.Diagnostics`
   package reference (which sets it).
2. A call to `RuntimeDiagnostics.StartLocalAsync(...)` at startup. `RuntimeDiagnostics.Start(...)` alone opens **no**
   endpoint.

Then confirm the app is actually still running — `debug` exits it. Also check the app is running as the **same user**
(the rendezvous directory `%LocalAppData%\Sprout\DevTools\sessions` is ACL-restricted to the current user), and pass
`--sessions <dir>` if the app overrode it.

## The snapshot says the window is empty

You captured too early. A cold app reports `status: partial` with an almost-empty node list for roughly the first two
seconds, while still reporting a correct `dpiScale` and `clientBounds` — which makes it look like a real, empty UI.

**Check `response.status` and re-capture until it is `complete`.** Use the `Get-SproutSnapshot` helper in
[layout-inspection.md](layout-inspection.md); raise its timeout for an app with heavy startup.

## `targetAmbiguous`

One PID hosts more than one diagnostics session. Pass the `instanceId` GUID from `inspect list` instead of the PID.

## A blank or all-one-colour screenshot

1. `sprout-devtools doctor` — check DPI awareness and isolation.
2. **Is this an RDP or disconnected session?** `Windows.Graphics.Capture` needs an active interactive desktop; a
   disconnected session keeps the HWNDs but produces no compositor frame. Move to a local session, Windows Sandbox, or
   an auto-logon Hyper-V guest.
3. Raise `--wait-ms` and `--settle-ms` — the app may not have presented its first real frame yet.
4. Read `debug-window-candidates.txt`, written exactly for this case; it lists every visible top-level window so you
   can see whether a splash, launcher or dialog was grabbed instead.

## `debug` says `passed` but the screenshot is wrong

This is a current limitation: the only assertion is `windowCaptured` (`>0x0`). Read `debug-uia-tree.txt` to find out
what was really captured. A .NET crash dialog passes this assertion, so `debug` remains triage rather than proof that
the intended application surface launched.

## A `find` fails

- The control may genuinely not be exposed to UI Automation — that *is* the finding. Dump the tree
  ([accessibility.md](accessibility.md)) and look.
- The name may differ from the visible text. Search the tree dump for the control type first.
- Raise `--find-timeout-ms` (provider-ready gate) and `--wait-ms` (main window poll) for a slow app.
- Use `--find-within-*` when the control is a descendant of a container that also matches.

## Acrylic looks like a solid white or gray panel

This is not established by `debug`/`drive` saying `passed`: a captured window and `BrushKind.Acrylic` do not prove that
the requested backdrop was sampled. First distinguish desktop/window Acrylic, current-target D2D Acrylic, and
cross-surface in-app `CompositionAcrylicSurface`.

For a drawer over promoted or transformed content, prefer a retained `CompositionAcrylicSurface` behind the
interactive pane, with an explicitly transparent SplitView pane background. The ordinary `AcrylicSurface` remains
inline D2D. Read the material observation's `RealizedRoute`, `PolicyReasons`, and `BackendFailureReason`;
`RenderTransform` / `LayerClip` are rendering-scope limitations, not instructions to lower the fallback color's alpha.
Respect real accessibility, OS-policy, and capability fallback.

Changing only a uniform panel's white/gray shade may just be changing `FallbackColor`. Confirm blurred transmission
over a live contrasting backdrop as well as the actual route; an opaque tint/foreground layer can still conceal a
working material. A dark light-dismiss scrim is a separate choice: `LightDismissOverlayMode.Off` removes dimming, not
outside-click dismissal. See the
[framework's Acrylic drawer recipe](https://github.com/Richasy/Sprout/blob/main/docs/guide/material.md#common-overlay-drawer-recipe).

## A snapshot node's numbers look impossible

- **Do not divide `bounds` by `dpiScale`.** They are already epx. See [layout-inspection.md](layout-inspection.md).
- A node under a **collapsed ancestor** keeps stale, last non-zero bounds — it is not on screen at all. Walk
  `treeEdges` up and check for a collapsed ancestor before trusting the numbers.
- Check `response.status`: a `partial` capture hit a budget, so absence does not prove the control is missing.

## The build step times out

NativeAOT publishes take minutes. Raise `--build-timeout-ms` (default 600000). For the inner loop prefer Debug —
`debug` already defaults to it.

## `--selftest` fails only under MSIX

Expected: the framework's self-test hard-asserts it is *not* packaged, so a packaged launch reports
`packaging => bad`. The packaged form is gated on register + AUMID launch + window presented instead, and `run`
excludes the self-test from parity. This is documented behaviour, not a regression. Under an automatically detected
packaged run the self-test is still executed against the **executable**, so it keeps passing.

## The app ran unpackaged when you expected packaged

Read `run.app.form` and `run.app.formSelection` first — the answer is in the result, not in the log. Detection looks
for the app's own `<AssemblyName>.msix` beside the publish output, or a build-produced loose AppX layout with a real
`AppxManifest.xml`. A publish directory with neither is genuinely unpackaged.

For a Sprout app the usual cause is that MSIX was never opted into or never built: packaging runs at **publish** time
and only when `<SproutPackageFormat>Msix</SproutPackageFormat>` is set, alongside the `Sprout.Packaging` and
`Microsoft.Windows.SDK.BuildTools` package references. A plain `dotnet build` produces no package, so an inner-loop
build directory is always unpackaged. If the project *does* declare MSIX and the publish still produced none, `run
<project>` reports that as a harness error rather than running unpackaged.

## The packaged run failed instead of falling back

That is deliberate — a manifest asks for identity guarantees a raw `.exe` cannot honour, so degrading quietly would
hide the real problem. Read the error and the register/install log in the artifacts directory. The usual causes:
Developer Mode is off (`0x80073CFF`), the development certificate is not yet trusted (first-time trust needs one
elevated session; afterwards the same thumbprint installs unelevated), or the same package identity is already
installed as a non-development package (`0x80073CFB`) and must be removed manually. Pass `--form exe` if you
deliberately want the unpackaged run.

## An option value starting with `--` is misparsed

Attach it with `=`: `--app-arg=--some-flag`, `--record-app-arg=--some-flag`.
