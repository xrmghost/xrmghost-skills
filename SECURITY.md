# Security Policy

## Capability Profile

- The skills in this repository guide agents to use the XrmGhost CLI through the `xg` command.
- The `authoring-dataverse-plugin-scenarios` skill writes local scenario JSON files for XrmGhost runs.
- The skills do not provide arbitrary shell execution.
- The skills do not perform direct network access on their own.
- The skills do not handle, store, or manage authentication material.

## Data Handling

- By default, these skills work with local scenario content and the standard `mock` data source.
- The `--data-source auto` and `--data-source live` modes are opt-in, require a Pro license, and are only used when the user explicitly chooses them.
- In `auto` and `live`, XrmGhost reads Dataverse data in read-only mode; the skills do not write data back to the environment.
- Live data retrieved for local runs can be cached locally to reduce repeated reads.
- Users can clear a stale local cache to force fresh reads.
- The skills do not transmit data to third-party services.

## Supported Versions

Security updates are provided for the `main` branch only.

## Reporting a Vulnerability

Please report suspected vulnerabilities privately to [filippo@agostinilab.it](mailto:filippo@agostinilab.it) and include the repository name, affected skill, impact, and reproduction details.

Please do not open a public GitHub issue for unpatched vulnerabilities. We aim to acknowledge reports within 5 business days and share a status update within 10 business days.
