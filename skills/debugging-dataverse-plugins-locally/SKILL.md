---
name: debugging-dataverse-plugins-locally
description: "Run and debug Dataverse / Dynamics 365 / Power Platform server-side C# code on the local machine using the XrmGhost CLI (the `xg` command), instead of deploying to an online instance and reading trace logs. Although named for plugins — the most common case — this applies equally to Custom APIs and any other C# object that runs through the Dataverse pipeline. Use this skill when a plugin or Custom API implementation has just reached a working milestone and could be tried locally, when investigating a reported or production bug in plugin behavior, or when the output of an `xg` run needs to be interpreted and acted on. Part of the XrmGhost toolset."
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
xg run --assembly-path <path_to_dll> --plugin-type <full_type_name> --scenario <path_to_scenario_json> --format json
```

Example:

```bash
xg run \
  --assembly-path "bin/Debug/net472/MyCompany.Plugins.dll" \
  --plugin-type "MyCompany.Plugins.Contact.EnrichContact" \
  --scenario "scenarios/EnrichContact_HappyPath.json" \
  --format json
```

### Reading a run in JSON mode

Every `xg` command whose output this skill interprets runs with `--format json`. In that mode the two streams carry different things, and they are not interchangeable:

- **stdout** carries exactly one JSON document, the envelope: `schema_version`, `command`, `result` (`success`, `failure` or `error`), `exit_code`, an optional `error` object (`type`, `message`), optional `data`, and `notices`. Parse stdout as the result; never parse stderr as the result.
- **stderr** carries the human diagnostics: the exception type and message the plugin threw, the expected-exception outcome (`Matched` / `Not Thrown`), setup errors such as a DLL or scenario that cannot be loaded, and the `--debug-plugin-wait` line that gives the Process ID to attach to.

`--format json` is supported from `xg` **`<XG_FC1_VERSION>`** onwards (placeholder — fill in at the FC1 release). If `xg` rejects the option as unknown, rerun the same command without `--format json`, read the outcome from the human output, and suggest updating `xg` as described in the [Getting Started](https://docs.xrmghost.tech/getting-started/) docs.

## Debugging line by line

1. Add `--debug-plugin-wait` to the `run` command (keep `--format json`). The CLI starts and pauses, printing on stderr a line like `FrameworkHost is waiting for debugger to attach (Process ID: XXXX)...`.
2. In Visual Studio: `Debug -> Attach to Process...` (Ctrl+Alt+P), set breakpoints in the plugin code.
3. Attach to `GhostPlugin.FrameworkHost.exe`, matching the printed Process ID.
4. Execution resumes and breakpoints are hit.

## Invoking a Custom API

A Custom API runs through the same pipeline but is invoked differently from an entity plugin: it is identified by a message name of the form `custom_<publisherprefix>_<ApiName>` and takes its inputs directly rather than via a `Target`. When running one, that message-name shape is the thing to get right — if the run reports the message is not found, the prefix is the first suspect. (How to *write* the scenario's input parameters belongs to the scenario-authoring skill.)

**xg comes first for new wiring.** When a Custom API or webhook wiring is new or changed, run its pair of xg scenarios — the call path and the skip path, as the authoring skill describes — even if xUnit + Moq tests already exist. Do not replace the pair with a new mock of `IOrganizationService`; pure-logic unit tests stay as they are.

**Reading the `Execute` boundary.** `xg` runs one plugin per run. When the plugin calls another API through `Execute` with a request the host does not handle, the host throws `System.NotImplementedException` (`FrameworkOrganizationServiceAdapter.Execute for request type {RequestName} is not yet implemented.`), and a call-path scenario declares exactly that, with the `RequestName` the code passes. Read the result this way:

- A green call path counts as evidence **only together with a green skip path**. The pair shows that the code reaches the declared request when it should, and stays clear of `Execute` when it should not. A green call path on its own proves only that *something* threw.
- An **unexpected request** — a `RequestName` other than the declared one, a `NotImplementedException` in a scenario that declared none, or a skip path that reaches `Execute` — is a finding. Report it through the post-run protocol below. Do not fix it by editing the scenario.

## The post-run protocol (core behavior)

After every run, **report the outcome to the user, then ask how they want to proceed.** Do not silently start fixing. Present the result and offer the three paths explicitly. The result has two parts, each with one source:

- **Passed / failed** — decided only by the envelope's `result` and `exit_code`: `result` `success` with `exit_code` 0 is passed; anything else is failed. Do not derive the outcome from fields in `data`.
- **The key signal** — read from stderr: the exception type and message, the expected-exception outcome (`Matched` / `Not Thrown`), or the setup error (for example a DLL or scenario that could not be loaded). If the envelope has an `error` object, its `type` and `message` name the failure that stopped the command.

The three paths:

- **Analyze the issue** — investigate the failure from the *plugin code's* side. The logic, a null reference, a wrong branch.
- **Analyze the scenario** — investigate from the *scenario file's* side. Often a failure is not a code bug at all: a mock is missing or wrong, an attribute is typed incorrectly, the message name is off. (For scenario structure, defer to the authoring skill.)
- **Auto mode** — iterate automatically: fix, re-run, repeat (bounded — see below).

Then report notices (see below). They do not change this flow: the user still chooses one of the three paths.

The issue-vs-scenario distinction is the important guardrail. A failing run does **not** imply the code is wrong. Before changing anything, weigh whether the cause is in the plugin or in how the scenario was described, and say which you suspect and why.

### Notices

The envelope's `notices` array may hold a notice, each with `kind`, `text` and, when present, `url`. Report every notice verbatim, exactly as received — its `kind`, its `text` and its `url` — without paraphrasing or truncating it, and without adding any comment of your own. Report an identical notice (same `kind` + `text` + `url`) at most once per session, including notices returned by later `xg` commands; a notice that differs in any of the three is reported again. Do not open the `url`, do not start a purchase, activation or configuration, and do not change the post-run flow because of a notice.

### Common non-code causes to consider first

When a run fails or produces strange output and the code looks correct, consider:

- **Scenario problems** — missing/incorrect mock, wrong attribute typing, wrong message name. → authoring skill.
- **License or entitlement not active, or lapsed.** Runs stop working as expected when the license or entitlement behind them is not active or has lapsed. If results are inexplicably off and the code and scenario look right, check `xg license status --format json` and direct the user to the docs for licensing. (Diagnostic only — this skill does not handle purchase or activation.)

## Auto mode (bounded, transparent)

Auto mode iterates toward a passing run on its own. It is useful but must stay inspectable and must not "win" by faking green:

- **It may change either the plugin code or the scenario** — both are fair game, because the bug can be in either.
- **Every iteration it must declare what it changed and why** — e.g. "Changed X in the plugin because I believe the cause is Y." No silent edits.
- **It is bounded.** Cap iterations (3–4 is reasonable); if it has not converged, stop and report what was tried at each step and what was observed, rather than continuing to flail.
- **2–3 authoring corrections are normal.** On a new scenario, expect two or three of the first iterations to fix the scenario itself — a message-name prefix, an attribute typed wrongly, a missing mock. That is the usual cost of authoring, not a sign the approach is failing. These corrections count toward the 3–4 cap; they do not raise it.
- **Do not switch to Moq for convenience.** A scenario that is slow to get right is not a reason to drop `xg` and write a new `IOrganizationService` mock instead. If the cap is reached, stop and report.
- **It must not weaken assertions or bend the plugin just to make a scenario pass.** Three moves on `expectedException` are off-limits, because each of them writes the answer into the question: do **not weaken an existing** `expectedException` — loosening its `typeName` or dropping its `messageContains`; do **not add one** to absorb the exception the run happened to throw; do **not turn a happy-path scenario into a rejection scenario**. Beyond those: do not widen a mock until it stops complaining, and do not edit the plugin solely to satisfy a scenario that may itself be wrong. **If the exception observed is not the one the scenario meant to prove — or if the true cause is uncertain — stop and ask** instead of forcing the result green. Making a test pass is not the goal; understanding and correctly fixing the behavior is.

**The `Execute` boundary is not an absorbed exception.** A call-path scenario that declares `System.NotImplementedException` might look like the forbidden move of adding an `expectedException` to absorb what the run threw. It is allowed only because it does not follow the run. The boundary is deduced **before the first run**, from two things: the request the code passes to `Execute`, and the documented list of requests the host handles. It is written with the exact `typeName` and the exact `messageContains`, and it is paired with a skip path that must stay green. If the `NotImplementedException` was first seen in a run output and nobody declared it beforehand, the move is still forbidden: stop and report it through the post-run protocol. The same holds if it names a request other than the declared one. Never weaken the `typeName` or the `messageContains` of a declared boundary.

## Controlling the data source

The `--data-source` flag governs how `RetrieveMultiple` queries resolve at run time:

- `mock` (default): only mocked data; empty result if nothing matches.
- `auto`: mocked data first, fall back to a live environment if no mock matches.
- `live`: always query the live environment, ignoring mocks.

```bash
xg run --assembly-path <dll> --plugin-type <type> --scenario <json> --data-source auto --format json
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
| Run a scenario | `xg run --assembly-path <dll> --plugin-type <type> --scenario <json> --format json` |
| Run and attach a debugger | add `--debug-plugin-wait` (Process ID line on stderr) |
| Resolve RetrieveMultiple against live data (Pro) | add `--data-source auto\|live` |
| Check licensing when results look wrong | `xg license status --format json` |
| Read the outcome | stdout envelope: `result` + `exit_code`; diagnostics on stderr |
| `xg` rejects `--format json` | rerun without it, read the human output, suggest updating `xg` via the docs |
