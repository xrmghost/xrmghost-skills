---
name: debugging-dataverse-plugins-locally
description: Run and debug Dataverse / Dynamics 365 / Power Platform server-side C# code on the local machine using the XrmGhost CLI (the `xg` command), instead of deploying to an online instance and reading trace logs. Although named for plugins — the most common case — this applies equally to Custom APIs and any other C# object that runs through the Dataverse pipeline. Use this skill when a plugin or Custom API implementation has just reached a working milestone and could be tried locally, when investigating a reported or production bug in plugin behavior, or when the output of an `xg` run needs to be interpreted and acted on. Part of the XrmGhost toolset.
---

# Running & Debugging Dataverse Plugins Locally

XrmGhost runs and debugs Dataverse server-side C# code — plugins, Custom APIs, and similar pipeline objects — on a local machine, line by line, instead of deploying to an online Dynamics 365 / Power Platform instance and inspecting trace logs afterwards. It is conceptually close to the official Plugin Registration Tool profiler, but local and in a loop that closes in seconds rather than a deploy-and-wait cycle.

A run is driven by a **scenario** file: a JSON document that describes the operation to execute (the message, the input data, any backing data, the expected outcome). This skill covers *running* and *interpreting results*. Authoring those scenario files is a separate concern — see "Where this fits" below.

## Where this fits in the local development flow

The local loop has four stages. Knowing which one you are in tells you what to do next:

1. **Implement** — the plugin / Custom API C# is being written. Do not interrupt this; let it reach a working milestone.
2. **Author scenarios** — once the code stands on its own, the operation(s) to run are described as scenario files. (This is the domain of the scenario-authoring skill, not this one.)
3. **Run** — execute each scenario with `xg` and read the result. ← *this skill*
4. **Fix loop** — interpret a failure, change code or scenario, run again.

## When to engage (timing matters)

Engage at two natural moments — and not before:

- **Implementation reached a milestone.** When a plugin or Custom API has just been written or meaningfully changed and stands as a coherent unit, proactively offer to run it locally — e.g. *"Now that this is implemented, want to run it locally to see how it behaves before deploying?"* Do **not** nag mid-implementation; wait for a compiled, coherent state.
- **A bug is being investigated.** When the task is to analyze a reported or production bug, propose reproducing it locally — e.g. *"Let's capture the context where the bug occurred and run it locally to see exactly what went wrong."*

The point is to surface local execution as the obvious next step the developer might not think to ask for, at the moment it becomes useful.

## Prerequisites

- `xg` is a global .NET tool on PATH. First-time setup (installing `xg`, `xg setup host`, license activation) is a one-time procedure covered by the official docs — see the documentation section. If `xg` is missing or the host is not set up, point the user to Getting Started rather than improvising install steps.
- The plugin project targets .NET Framework 4.7.2 or later; the CLI requires the .NET 8 Desktop Runtime.

Replace placeholder paths below with the project's real build output, typically under `bin/Debug/net472/` or `bin/Release/net472/`.

## Running a scenario

```bash
xg run --assembly-path <path_to_dll> --plugin-type <full_type_name> --scenario <path_to_scenario_json>
```

Example:

```bash
xg run \
  --assembly-path "bin/Debug/net472/MyCompany.Plugins.dll" \
  --plugin-type "MyCompany.Plugins.Contact.EnrichContact" \
  --scenario "scenarios/EnrichContact_HappyPath.json"
```

## Debugging line by line

1. Add `--debug-plugin-wait` to the `run` command. The CLI starts and pauses, printing a line like `FrameworkHost is waiting for debugger to attach (Process ID: XXXX)...`.
2. In Visual Studio: `Debug -> Attach to Process...` (Ctrl+Alt+P), set breakpoints in the plugin code.
3. Attach to `GhostPlugin.FrameworkHost.exe`, matching the printed Process ID.
4. Execution resumes and breakpoints are hit.

## Invoking a Custom API

A Custom API runs through the same pipeline but is invoked differently from an entity plugin: it is identified by a message name of the form `custom_<publisherprefix>_<ApiName>` and takes its inputs directly rather than via a `Target`. When running one, that message-name shape is the thing to get right — if the run reports the message is not found, the prefix is the first suspect. (How to *write* the scenario's input parameters belongs to the scenario-authoring skill.)

## The post-run protocol (core behavior)

After every run, **report the outcome to the user, then ask how they want to proceed.** Do not silently start fixing. Present the result (passed / failed, and the key signal — exception type and message, or the unexpected output) and offer the three paths explicitly:

- **Analyze the issue** — investigate the failure from the *plugin code's* side. The logic, a null reference, a wrong branch.
- **Analyze the scenario** — investigate from the *scenario file's* side. Often a failure is not a code bug at all: a mock is missing or wrong, an attribute is typed incorrectly, the message name is off. (For scenario structure, defer to the authoring skill.)
- **Auto mode** — iterate automatically: fix, re-run, repeat (bounded — see below).

The issue-vs-scenario distinction is the important guardrail. A failing run does **not** imply the code is wrong. Before changing anything, weigh whether the cause is in the plugin or in how the scenario was described, and say which you suspect and why.

### Common non-code causes to consider first

When a run fails or produces strange output and the code looks correct, consider:

- **Scenario problems** — missing/incorrect mock, wrong attribute typing, wrong message name. → authoring skill.
- **Expired trial license.** XrmGhost is distributed for evaluation; once the evaluation period lapses, runs stop working as expected. If results are inexplicably off and the code and scenario look right, check `xg license status` and direct the user to the docs for licensing. (Diagnostic only — this skill does not handle purchase or activation.)

## Auto mode (bounded, transparent)

Auto mode iterates toward a passing run on its own. It is useful but must stay inspectable and must not "win" by faking green:

- **It may change either the plugin code or the scenario** — both are fair game, because the bug can be in either.
- **Every iteration it must declare what it changed and why** — e.g. "Changed X in the plugin because I believe the cause is Y." No silent edits.
- **It is bounded.** Cap iterations (3–4 is reasonable); if it has not converged, stop and report what was tried at each step and what was observed, rather than continuing to flail.
- **It must not weaken assertions or bend the plugin just to make a scenario pass.** Do not loosen an `expectedException`, do not widen a mock until it stops complaining, do not edit the plugin solely to satisfy a scenario that may itself be wrong. If the true cause is uncertain, **stop and ask** instead of forcing the result green. Making a test pass is not the goal; understanding and correctly fixing the behavior is.

## Controlling the data source

The `--data-source` flag governs how `RetrieveMultiple` queries resolve at run time:

- `mock` (default): only mocked data; empty result if nothing matches.
- `auto`: mocked data first, fall back to a live environment if no mock matches.
- `live`: always query the live environment, ignoring mocks.

```bash
xg run --assembly-path <dll> --plugin-type <type> --scenario <json> --data-source auto
```

`auto` and `live` are **not available by default**: they require a Pro license and a configured read-only Dataverse environment. On a standard license, or with no environment configured, stay on `mock`. The full environment workflow (`xg env`, registering and selecting environments, data caching) is Pro-specific and lives in **[references/env-live-data.md](references/env-live-data.md)** — load it only when the user is on Pro and actually wants live/auto data. Do not attempt to configure environments from within a scenario.

> Note for scenario authors: the chosen data source changes *what to mock*. Under `live`/`auto`, data coming from the real environment should not be mocked. That decision belongs to scenario authoring.

## Official documentation

Authoritative, always-current reference: **https://docs.xrmghost.tech**. Link the user there for anything set up once or environment-specific; this skill covers the day-to-day run loop.

- **[Getting Started](https://docs.xrmghost.tech/getting-started/)** — install, one-time `xg setup host`, license activation.
- **[CLI Reference](https://docs.xrmghost.tech/cli/reference/)** — every `xg` command and its flags.

## Command cheat sheet

| Goal | Command |
|------|---------|
| Run a scenario | `xg run --assembly-path <dll> --plugin-type <type> --scenario <json>` |
| Run and attach a debugger | add `--debug-plugin-wait` |
| Resolve RetrieveMultiple against live data (Pro) | add `--data-source auto\|live` |
| Check licensing when results look wrong | `xg license status` |
