# Live data & environment management (Pro)

Reference for using XrmGhost against a **real Dataverse environment** in read-only mode, instead of (or alongside) mocked data. This is a **Pro-licensed** capability; on a standard license live is never contacted, and the default `auto` data source answers from the scenario mocks, as `mock` does. Load this material only when the user is on Pro and explicitly wants live or `auto` data.

## What this enables

Without an active environment, `xg run` resolves `Retrieve` and `RetrieveMultiple` only against data mocked inside the scenario. With an environment configured, the `--data-source` flag reads real data:

- `auto` (default) — live first. `Retrieve` falls back to the scenario mocks only when the record does not exist in Dataverse or live is unavailable; `RetrieveMultiple` reads live only, and a failed live attempt returns an empty result with a warning, not mock rows.
- `live` — ignore mocks entirely and query the live environment.

Access is **read-only**: XrmGhost reads from the environment to satisfy queries during a local run; it does not write back.

## The `xg env` command group

Environments are managed through `xg env`. The typical lifecycle:

- **Register** an environment — give XrmGhost the connection details for a Dataverse instance and a friendly name to refer to it by.
- **List** registered environments to see what is available.
- **Select** the target environment that `auto` / `live` runs will resolve against.
- **Inspect / remove** environments as they change.

For the exact subcommands, flags, and authentication options, consult the CLI Reference for `xg env` at https://docs.xrmghost.tech/cli/reference/ — these are environment- and version-specific and the docs are the source of truth.

## Data caching

To avoid hitting the live environment repeatedly for the same reads across runs, retrieved data can be cached locally. This speeds up the iterate-and-rerun loop and reduces load on the environment. Cache behavior and controls are documented in the CLI Reference.

## Interaction with scenario authoring

The data source chosen here changes how scenarios should be written:

- Under `mock`, every record a query needs must be present in the scenario.
- Under `auto` with an active environment, live wins: mock only `Retrieve` records that do not exist in Dataverse (for example, created by the run); query mocks are not read.
- Under `live`, query-backing data in the scenario is ignored — keep the scenario focused on the `Target`, input parameters, and expected outcome.

Decide the data source **before** authoring the backing data of a scenario, so effort is not spent mocking data that will be read live.

## When results look wrong

If `auto` / `live` returns unexpected data:

- Confirm the correct environment is selected (`xg env`).
- Confirm the records actually exist and are visible to the authenticated identity (access is read-only and identity-scoped).
- Remember that a stale local cache can mask recent changes in the environment; clearing it forces fresh reads.
