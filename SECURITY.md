# Security Policy

This policy covers the `xrmghost-skills` repository only — the Markdown skill content and its CI. It is not the place to report issues with the `xg` CLI, the AKMS licensing backend, or the Dataverse plugins these skills drive; those live in their own repos and are out of scope here. Product-level license tiers referenced below (`mock` / `auto` / `live` data sources, Pro license) are defined authoritatively at [docs.xrmghost.tech/editions/](https://docs.xrmghost.tech/editions/), not by this file.

## Capability Profile

- The skills in this repository guide agents to use the XrmGhost CLI through the `xg` command.
- The `authoring-dataverse-plugin-scenarios` skill writes local scenario JSON files for XrmGhost runs.
- The skills do not provide arbitrary shell execution.
- The skills do not perform direct network access on their own.
- The skills do not handle, store, or manage authentication material.

## Data Handling

- The default `--data-source` is `auto`. On a standard license, or with no active environment, `auto` answers from local scenario content only, exactly as `mock` does — Dataverse is never contacted.
- On a Pro license with an active environment, `auto` is live-first: a `Retrieve` falls back to local scenario content only when the record is missing in Dataverse or live is unavailable or fails, with a warning; any other Dataverse fault reaches the plugin unchanged. `RetrieveMultiple`, `RetrieveMultipleRequest` and `QueryByAttribute` never fall back to scenario content after a live attempt — when live is unavailable or fails they return an empty collection with a warning, and a fault Dataverse actually returns reaches the plugin.
- `--data-source mock` never contacts Dataverse, on any license. `--data-source live` requires a Pro license and reads only Dataverse, never local scenario content: without a Pro license, without an active environment, or when live is unavailable or fails, `Retrieve` raises an exception with a warning; otherwise any Dataverse fault — including a missing record — reaches the plugin. `RetrieveMultiple`, `RetrieveMultipleRequest` and `QueryByAttribute` return an empty collection with a warning in those same conditions, and a Dataverse fault reaches the plugin.
- In `auto` and `live`, any Dataverse access is read-only; the skills do not write data back to the environment.
- Live data retrieved for local runs can be cached locally to reduce repeated reads.
- Users can clear a stale local cache to force fresh reads.
- The skills do not transmit data to third-party services.

## Supported Versions

Security updates are provided for the `main` branch only.

## Reporting a Vulnerability

Please report suspected vulnerabilities privately to [filippo@agostinilab.it](mailto:filippo@agostinilab.it) and include the repository name, affected skill, impact, and reproduction details.

Please do not open a public GitHub issue for unpatched vulnerabilities. We aim to acknowledge reports within 5 business days and share a status update within 10 business days.
