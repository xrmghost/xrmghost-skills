# xrmghost-skills

[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/xrmghost/xrmghost-skills/badge)](https://securityscorecards.dev/viewer/?uri=github.com/xrmghost/xrmghost-skills)

Agent Skills for [XrmGhost](https://docs.xrmghost.tech) — running and debugging Dataverse / Dynamics 365 / Power Platform server-side C# locally with the `xg` CLI, instead of deploying to an online instance and reading trace logs. Start with the product docs at **https://docs.xrmghost.tech** for setup, CLI reference, and licensing details.

If you landed here to understand how these two skills fit into the wider product rather than to read their source, the dedicated docs section is a better starting point than this README: **[docs.xrmghost.tech/skills/](https://docs.xrmghost.tech/skills/)** covers what the skills are for and when they activate, **[installing-skills/](https://docs.xrmghost.tech/skills/installing-skills/)** walks through setup per agent, and **[using-skills/](https://docs.xrmghost.tech/skills/using-skills/)** shows the day-to-day authoring/debugging loop end to end.

These skills are written to the [Agent Skills](https://agentskills.io) open standard (a `SKILL.md` with `name` + `description` frontmatter, plus optional supporting files). They are consumed by AI coding agents — the primary way XrmGhost is used in practice is through an agent writing and verifying plugin / Custom API code, not by hand.

## Skills in this repository

| Skill | What it does | Engage when |
|-------|--------------|-------------|
| [`debugging-dataverse-plugins-locally`](skills/debugging-dataverse-plugins-locally/) | Runs and debugs plugins, Custom APIs, and other C# pipeline objects locally with `xg run`; interprets results and drives the fix loop. | A plugin/Custom API just reached a working state, a bug is being investigated, or an `xg` run result needs interpreting. |
| [`authoring-dataverse-plugin-scenarios`](skills/authoring-dataverse-plugin-scenarios/) | Writes the JSON scenario files that drive a local run: the message, input data, backing data, and expected outcome. | Preparing the input for a local run — defining the operation, supplying data the code reads, describing what should happen. |

The two are designed to activate **independently** (each is self-sufficient on its own triggers) so they work across agents without relying on cross-skill references. In the common flow — write code, write scenarios, run — both activate naturally.

## Installation

Each agent looks for skills in its own directory. Copy the desired folder(s) from `skills/` into the right place:

| Agent | Project-level (this workspace) | User-level (all projects) |
|-------|-------------------------------|---------------------------|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| GitHub Copilot CLI | `.github/skills/` | `~/.copilot/skills/` or `~/.agents/skills/` |
| Codex CLI | `.agents/skills/` | `~/.agents/skills/` |
| OpenCode | `.claude/skills/` (Claude Code fallback) | `~/.claude/skills/` |
| Pi | `.pi/skills/` or `.agents/skills/` | `~/.pi/agent/skills/` or `~/.agents/skills/` |

Example (Claude Code, user-level):

```bash
cp -r skills/debugging-dataverse-plugins-locally ~/.claude/skills/
cp -r skills/authoring-dataverse-plugin-scenarios ~/.claude/skills/
```

Copy the whole skill folder, including its `references/` subdirectory — supporting files are loaded relative to `SKILL.md`. Do not nest a skill one level too deep: the layout must be `…/skills/<skill-name>/SKILL.md`.

A guided multi-agent installer (auto-detect installed agents, choose project vs user scope) is planned.

## Prerequisites

The skills assume the `xg` CLI is installed and set up. First-time setup — installing `xg`, `xg setup host`, and license activation — is covered by the [official Getting Started](https://docs.xrmghost.tech/getting-started/). XrmGhost is distributed for evaluation; runs stop working as expected once the trial period lapses.

## Documentation

Authoritative reference: **https://docs.xrmghost.tech**. The docs are the source of truth for installation, the full [`xg` command reference](https://docs.xrmghost.tech/cli/reference/), environment/licensing specifics, and the broader product documentation; these skills cover the day-to-day local development loop.

This repository's own architectural rationale — why the linter and the advisory judge were chosen for a public, Markdown-only, single-maintainer repo — lives in [`docs/adr/`](docs/adr/); the product-wide architecture these decisions have to stay consistent with is documented separately at [docs.xrmghost.tech/architecture/](https://docs.xrmghost.tech/architecture/).

Want to change a skill or add a new one? See [CONTRIBUTING.md](CONTRIBUTING.md) for the linting, signing, and PR requirements enforced on this repo.

Security disclosures and repository security scope are documented in the [Security Policy](SECURITY.md).

## License

See [LICENSE](LICENSE).
