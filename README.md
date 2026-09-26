# xrmghost-skills

**XrmGhost lets your AI coding agent build Dataverse plugins that actually work before they ship.**

The agent writes the plugin or Custom API, runs it locally with the `xg` CLI, reads the real execution result, and fixes it — in seconds, with no deploy to an online instance and no digging through trace logs. That loop, not hand-authoring, is how XrmGhost is used in practice. This repository holds the two Agent Skills that teach an agent to drive it.

## See the loop

The agent writes a plugin that validates VAT numbers for business accounts. To pick the branch it reads the account type through a helper that returns `null` when the field is empty — then reads `.Value` off it without a null check. Plausible code; the happy path was tested in the agent's head. On an account created with no type set — a legitimate case that should just save — it runs the scenario locally:

```bash
xg run -a bin/Debug/net462/Contoso_Dataverse_Project.dll \
       -t Contoso_Dataverse_Project.AccountTypeValidationPlugin \
       -s scenarios/AccountVatValidation_BugRepro_NoAccountType.json
```

and the code blows up — with the exact type, message, and method:

```
Execution Failed!
Error Type:    System.NullReferenceException
Error Message: Object reference not set to an instance of an object.
   at Contoso_Dataverse_Project.AccountTypeValidationPlugin.Execute(IServiceProvider serviceProvider)
      in AccountTypeValidationPlugin.cs:line 99
```

(exit code `1`) — a real defect, surfaced in about a second, that no amount of reading the code would have caught. (Stack-trace line numbers appear when plugins are built with full debug symbols.) The agent adds the missing guard, in C#:

```diff
  var accountType = GetOptionSetValue(target, preImage, AccountTypeField);
+ if (accountType == null)
+ {
+     Trace(tracingService, "No account type set; no VAT validation applies. Allowing save.");
+     return;
+ }
  if (accountType.Value != BusinessValue)
```

rebuilds, and re-runs the same command:

```
Execution Succeeded!
```

(exit code `0`) — green. Write → run → read the real failure → fix the code → re-run, entirely on the developer's machine.

And it scales past one plugin. Point the agent at the whole project — *"test all the plugins"* — and it writes the scenarios, runs them all, and comes back with a grid of results and the bug you weren't looking for: not the loud crash you'd have caught on day one, but the plugin that saves without complaint and quietly writes the wrong value, the case that survives every manual test and surfaces three months later. That is the real shift: not a faster version of what a careful developer already does by hand, but exhausting the case space no one has the time to enumerate.

## Why an agent can't do this without XrmGhost

An AI agent can already *write* Dataverse plugin code. What it cannot do — on its own — is *verify* it. The only feedback path for server-side Dataverse code is to build it, deploy it to an online instance, trigger it, and read the trace logs. That round-trip is minutes long and lives outside the agent's reach, so the agent is writing blind: it produces plausible code it has no way to run.

XrmGhost closes that loop by executing the plugin locally, out of process, against a scenario you describe — and handing back the real result the agent can act on. This is also what separates XrmGhost from the unit-testing libraries it is sometimes mistaken for: it is not a mock of the pipeline, it runs your compiled plugin and reports what actually happened.

## Skills in this repository

| Skill | What it does | Engage when |
|-------|--------------|-------------|
| [`debugging-dataverse-plugins-locally`](skills/debugging-dataverse-plugins-locally/) | Runs and debugs plugins, Custom APIs, and other C# pipeline objects locally with `xg run`; interprets results and drives the fix loop. | A plugin/Custom API just reached a working state, a bug is being investigated, or an `xg` run result needs interpreting. |
| [`authoring-dataverse-plugin-scenarios`](skills/authoring-dataverse-plugin-scenarios/) | Writes the JSON scenario files that drive a local run: the message, input data, backing data, and expected outcome. | Preparing the input for a local run — defining the operation, supplying data the code reads, describing what should happen. |

The two are designed to activate **independently** (each is self-sufficient on its own triggers) so they work across agents without relying on cross-skill references. In the common flow — write code, write scenarios, run — both activate naturally.

## Installation

**Recommended — one command, via `npx skills`.** [vercel-labs/skills](https://github.com/vercel-labs/skills) is the de-facto installer for the Agent Skills ecosystem; it reads this repo straight from GitHub and installs both skills into the right place for the coding agents it detects on your machine:

```bash
npx skills@latest add xrmghost/xrmghost-skills --all -g -y
```

Pass `-a <agent>` (see below) to target a specific agent instead of every detected one.

| Flag | Effect |
|------|--------|
| `--list` | Show the skills in this repo without installing anything |
| `--all` | Install both skills (omit to choose interactively) |
| `-a <agent>` | Target a specific agent, e.g. `claude-code` |
| `-g` | Install user-level (all projects); omit for project-level (this workspace) |
| `-y` | Skip confirmation prompts |

A community alternative, [`gh-upskill`](https://github.com/trieloff/gh-upskill), also installs from this public repo and needs no `gh` login.

**Manual fallback.** If you don't use a skills CLI, copy the folders yourself. Each agent looks for skills in its own directory:

| Agent | Project-level (this workspace) | User-level (all projects) |
|-------|-------------------------------|---------------------------|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| GitHub Copilot CLI | `.github/skills/` | `~/.copilot/skills/` or `~/.agents/skills/` |
| Codex CLI | `.agents/skills/` | `~/.agents/skills/` |
| OpenCode | `.claude/skills/` (Claude Code fallback) | `~/.claude/skills/` |
| Pi | `.pi/skills/` or `.agents/skills/` | `~/.pi/agent/skills/` or `~/.agents/skills/` |

```bash
cp -r skills/debugging-dataverse-plugins-locally ~/.claude/skills/
cp -r skills/authoring-dataverse-plugin-scenarios ~/.claude/skills/
```

Copy the whole skill folder, including its `references/` subdirectory — supporting files are loaded relative to `SKILL.md`. Do not nest a skill one level too deep: the layout must be `…/skills/<skill-name>/SKILL.md`.

## Prerequisites

The skills assume the `xg` CLI is installed and set up. First-time setup — installing `xg`, `xg setup host`, and license activation — is covered by the [official Getting Started](https://docs.xrmghost.tech/getting-started/).

XrmGhost Standard is free for 60 days — no account, no card, no sign-up.

## Documentation

Authoritative reference: **https://docs.xrmghost.tech**. The docs are the source of truth for installation, the full [`xg` command reference](https://docs.xrmghost.tech/cli/reference/), environment/licensing specifics, and the broader product documentation. If you want the guided tour of what these two skills do and when they activate, [docs.xrmghost.tech/skills/](https://docs.xrmghost.tech/skills/) covers exactly that, [installing-skills/](https://docs.xrmghost.tech/skills/installing-skills/) walks through setup per agent, and [using-skills/](https://docs.xrmghost.tech/skills/using-skills/) shows the day-to-day authoring/debugging loop end to end.

These skills are written to the [Agent Skills](https://agentskills.io) open standard (a `SKILL.md` with `name` + `description` frontmatter, plus optional supporting files) and are consumed by AI coding agents.

## For contributors and maintainers

[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/xrmghost/xrmghost-skills/badge)](https://securityscorecards.dev/viewer/?uri=github.com/xrmghost/xrmghost-skills)

- **Change or add a skill?** See [CONTRIBUTING.md](CONTRIBUTING.md) for the linting, signing, and PR requirements enforced on this repo.
- **Architectural rationale** — why the linter and the advisory judge were chosen for a public, Markdown-only, single-maintainer repo — lives in [`docs/adr/`](docs/adr/); the product-wide architecture these decisions stay consistent with is documented at [docs.xrmghost.tech/architecture/](https://docs.xrmghost.tech/architecture/).
- **Security disclosures** and repository security scope are documented in the [Security Policy](SECURITY.md).

## Related resources

- **[www.xrmghost.tech](https://www.xrmghost.tech)** — the product marketing site: what XrmGhost is, who it's for, and how to get started.
- **[github.com/xrmghost/xrmghost](https://github.com/xrmghost/xrmghost)** — the base repository and index for the wider XrmGhost suite.
- **[github.com/xrmghost/xrmghost-docs](https://github.com/xrmghost/xrmghost-docs)** — source of the docs site referenced throughout this README.
- **[github.com/xrmghost/xrmghost-attributes](https://github.com/xrmghost/xrmghost-attributes)** — a sibling library with strongly-typed attribute helpers for Dataverse plugin code, often written alongside the scenarios and runs these skills drive.
- **[XrmGhost.Attributes on NuGet](https://www.nuget.org/packages/XrmGhost.Attributes)** — the packaged distribution of that library.

## License

See [LICENSE](LICENSE).
