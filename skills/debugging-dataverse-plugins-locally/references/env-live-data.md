# Live data & environment management (Pro)

Reference for using XrmGhost against a **real Dataverse environment** in read-only mode, instead of (or alongside) mocked data. This is a **Pro-licensed** capability; on a standard license, live is never contacted and the default `auto` data source answers from the scenario mocks, exactly as `mock` does. Load this material only when the user is on Pro and explicitly wants live or `auto` data.

## What this enables

Without an active environment, the default `auto` (like `mock`) resolves `Retrieve`, `RetrieveMultiple`, `RetrieveMultipleRequest` and `QueryByAttribute` only against data mocked inside the scenario; `live` never reads mocks (see below). With a Pro license and an active environment selected, the `--data-source` flag (omitted means `auto`) reads real data:

- `auto` (default) — live-first. `Retrieve` reads live and falls back to the scenario mock only when the record does not exist in Dataverse or live is unavailable, with a warning; any other Dataverse fault reaches the plugin unchanged. `RetrieveMultiple`, `RetrieveMultipleRequest` and `QueryByAttribute` always read live and never fall back to mocks: if the live attempt is unavailable the result is an empty collection with a warning, and a fault Dataverse actually returns reaches the plugin.
- `live` — ignore mocks entirely. Without a Pro license, without an active environment, or when live is unavailable, `Retrieve` raises an exception (with a warning); otherwise any Dataverse fault — including a missing record — reaches the plugin. `RetrieveMultiple`, `RetrieveMultipleRequest` and `QueryByAttribute` return an empty collection with a warning when live is unavailable, and reach the plugin on a Dataverse fault.

## Live operations are read-only

Access to the live organization is **read-only**: XrmGhost reads from the environment to satisfy queries during a local run and never writes to it.

- **Approved live operations.** At the request level only `Retrieve`, `RetrieveMultiple`, and `WhoAmI` reach live Dataverse. `RetrieveMultipleRequest` and `QueryByAttribute` are the documented query forms routed through the `RetrieveMultiple` live path.
- **Everything else is rejected.** Any non-read request is rejected before any network access and fails the run with `Live connection is read-only: '<operation>' is not allowed (only Retrieve, RetrieveMultiple and WhoAmI reach Dataverse).`, where `<operation>` names the refused operation (for example `Create`, or `Execute:<request>`).
- **Plugin writes stay simulated.** Plugin writes always go to the simulated run service inside the local run, and the run sends only `Retrieve`, `RetrieveMultiple` and `WhoAmI` to the live connection. They are not writes to the organization, and scenarios that exercise them remain valid under `auto` and `live`. The read-only block is a second guarantee on top of that.
- **Availability.** The block is available from the next stable XrmGhost release onwards.

When a run fails with `Live connection is read-only:`, treat it as a possible XrmGhost bug, not a defect in the user's plugin: a plugin cannot reach the live connection with a write. Report the full error to the user as a possible XrmGhost bug (to be filed at https://docs.xrmghost.tech/community/report-a-bug/) and stop. Do not try to work around the block, and do not change the plugin to avoid it. Never bypass the block by writing to the live organization through the Web API, `pac`, a separate SDK client, or scripts. The prohibition covers writes to the live organization only, not simulated plugin writes.

## The `xg env` command group

Environments are managed through `xg env`. The typical lifecycle:

- **Register** an environment — give XrmGhost the connection details for a Dataverse instance and a friendly name to refer to it by.
- **List** registered environments to see what is available.
- **Select** the target environment that `auto` / `live` runs will resolve against.
- **Inspect / remove** environments as they change.

For the exact subcommands, flags, and authentication options, consult the CLI Reference for `xg env` at https://docs.xrmghost.tech/cli/reference/ — these are environment- and version-specific and the docs are the source of truth.

## Delegated authentication and the reusable session

Live access runs as the **delegated identity of the environment's owner**: the person who registered the environment signs in once, and the CLI reuses that session on later runs. The session is kept **per tenant and organisation**: environments for different organisations never share it, while two aliases of the same organisation in the same tenant can.

> Delegated authentication is available from the next stable XrmGhost release onwards.

- **Initial sign-in (owner only).** The owner registers the environment with `xg env add` (the wizard stores the method, it does not sign in), then signs in once in their own interactive terminal: `xg env activate <name>`, then `xg env refresh` (no name: it acts on the active environment).
- **Reuse.** Later runs, including headless ones, reuse the cached session silently for that environment. Under `auto` or `live` no prompt is needed while the session is valid.
- **Sign-out.** `xg env logout [FRIENDLY_NAME]` removes the cached session of the named environment (the active one when no name is given). If the sign-in cache is shared with another environment it is not deleted, and the command says so.
- **Supported methods** (chosen in the `xg env add` / `xg env edit` wizard):
  - interactive browser sign-in with MFA (on Windows through WAM, the Windows account broker);
  - device code;
  - certificate (app registration, ClientId required);
  - client secret (app registration, ClientId required);
  - Azure CLI (reuses `az login`).
- **Persistence limit on Linux.** Without `libsecret` the cache is not persistent: the session stays in memory and no refresh token is written to disk, so every new process needs a non-interactive method (certificate, client secret, Azure CLI) or a fresh interactive sign-in.
- **Unverified XrmGhost app.** If the tenant blocks consent for the multitenant XrmGhost app (its publisher is not verified), the owner registers the environment with the **ClientId of the customer's own app registration** (the "my own app registration" choices for browser and device code). Never write a ClientId, token, password or secret into skills, scenarios or chat.

### Rules for agents (headless)

An agent never gets a sign-in prompt, even when it runs `xg` in a terminal of its own (an editor's integrated terminal): the CLI recognises agent and CI runs and fails fast instead of opening a browser, WAM or device code.

- **Never** start a sign-in or prompt, and **never** run `xg env logout`, `xg env edit` or `xg env add`: they are the owner's actions. Do not configure the environment from the agent's process either.
- When the session is missing or expired, `xg run` fails with a re-authentication message. Report that message **as the CLI printed it**; do not paraphrase it or invent one. At the time of writing the CLI says: `Environment '<name>' needs an interactive sign-in, which is not allowed here (no terminal, --no-interactive, or a command started by an agent or CI). Run 'xg env activate <name>' and then 'xg env refresh' in your own terminal to sign in, then run this command again.` or, for other token failures, `Could not get a token for environment '<name>'. Run 'xg env activate <name>' and then 'xg env refresh' in your own terminal to sign in again, then run this command again.` For Azure CLI the message names the `az login --tenant <tenant>` the owner has to run.
- Then stop and ask the owner to **re-authenticate** in their own interactive terminal: run `xg env activate <name>`, then `xg env refresh` (for Azure CLI, the `az login` the message names), then tell the agent to retry. `xg env edit` does not sign in. Meanwhile `--data-source mock` keeps the run deterministic and needs no session.
- The read-only guarantee above is unchanged by delegated identity: only `Retrieve`, `RetrieveMultiple` and `WhoAmI` reach Dataverse.

## Data caching

To avoid hitting the live environment repeatedly for the same reads across runs, retrieved data can be cached locally. This speeds up the iterate-and-rerun loop and reduces load on the environment. Cache behavior and controls are documented in the CLI Reference.

## Interaction with scenario authoring

The data source chosen here changes how scenarios should be written:

- Under `mock`, every record a query or retrieve needs must be present in the scenario; Dataverse is never contacted, regardless of license or environment.
- Under `auto` on Standard, or with no active environment, this behaves exactly like `mock`: every record a query or retrieve needs must be present in the scenario.
- Under `auto` with a Pro license and an active environment, live wins: query mocks (`RetrieveMultiple`, `RetrieveMultipleRequest`, `QueryByAttribute`) are never read, and a `Retrieve` mock matters only as the fallback for a record that does not exist in Dataverse (for example, one the run itself creates) or when live is unavailable.
- Under `live`, query-backing data in the scenario is ignored — keep the scenario focused on the `Target`, input parameters, and expected outcome.

Decide the data source **before** authoring the backing data of a scenario, so effort is not spent mocking data that will be read live. For reproducible runs — CI, or any scenario shared between people — use `--data-source mock` explicitly rather than relying on the environment state behind `auto`.

## When results look wrong

If `auto` / `live` returns unexpected data:

- Confirm the correct environment is selected (`xg env`).
- Confirm the records actually exist and are visible to the authenticated identity (access is read-only and identity-scoped).
- Remember that a stale local cache can mask recent changes in the environment; clearing it forces fresh reads.
