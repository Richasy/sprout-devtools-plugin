# sprout-devtools — a Copilot CLI plugin

Teach GitHub Copilot CLI to **look at a running [Sprout](https://github.com/Richasy/Sprout) application and check
its UI** — in any repository, not just the framework's own.

Ask in plain language:

> "Take a look at this screen and tell me if it's right."
> "Is the sidebar spacing correct?"
> "Did my change actually land visually?"
> "Can a screen reader reach this button?"

…and Copilot will reach for the `sprout-devtools` CLI, capture real evidence, and answer from it instead of guessing
from source code.

## Why this exists

Skills in a repository's `.github/skills/` are only visible **inside that repository**. An application built *with*
Sprout lives somewhere else entirely, so it never sees them. A plugin is installed **per developer and applies across
every repository**, which is exactly the right shape for a tool you use *on* your app.

## Install

Add the marketplace, then install from it:

```shell
copilot plugin marketplace add Richasy/sprout-devtools-plugin
copilot plugin install sprout-devtools@sprout
```

> Use the marketplace route. Direct installs (`copilot plugin install owner/repo`, git URLs, local paths) still work
> today but are **deprecated** — Copilot CLI warns that only `plugin@marketplace` installs will be supported in a
> future release.

Verify:

```shell
copilot plugin list
```

or, in an interactive session, `/plugin list` and `/skills list`.

Update later with `copilot plugin marketplace update sprout` followed by `copilot plugin update sprout-devtools`.

### The CLI itself

The plugin is documentation; the work is done by the `sprout-devtools` command, distributed separately on nuget.org:

```powershell
dotnet tool install -g Sprout.DevTools --prerelease
sprout-devtools doctor
```

Requires a **.NET 10** runtime. The skill guides Copilot to install it when it is missing.

## What's inside

```
plugin.json
skills/sprout-devtools/
├── SKILL.md                        when to use it, install, workflow decision table, the three traps
├── pitfalls.md                     the full DO / DON'T list
└── references/
    ├── screenshots.md              capturing a window, judging the image, blank frames
    ├── layout-inspection.md        verifying layout from framework truth; the snapshot JSON shape
    ├── accessibility.md            reading and checking the UI Automation tree
    ├── interaction.md              acting on controls and asserting state
    ├── command-reference.md        every verb, the result schema, exit codes
    └── troubleshooting.md          a command failed, produced nothing, or found no target
```

`SKILL.md` stays small on purpose and the references load only when they are needed — a skill that dumps everything at
once teaches nothing.

## Safety

The skill is explicit that **synthetic input must never be run on a developer's own desktop**. `SendInput` is global:
it injects into whatever window is foreground at that instant, not into the app being tested, and it has destroyed
live terminal sessions. The safe UI Automation pattern verbs are the deterministic assertion anyway, and a genuine
desktop gesture belongs in an isolated guest.

## License

MIT. See [LICENSE](LICENSE).
