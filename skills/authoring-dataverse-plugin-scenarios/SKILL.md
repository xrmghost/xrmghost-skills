---
name: authoring-dataverse-plugin-scenarios
description: "Write the JSON scenario files that XrmGhost uses to run Dataverse / Dynamics 365 / Power Platform server-side C# locally — describing the operation to execute: the message, the input data, the backing data a query or retrieve will see, and the expected outcome. Although named for plugins — the most common case — this covers Custom APIs and other C# pipeline objects too. Use this skill when preparing the input for a local run: defining what operation to execute, supplying the data the code will read, or describing what should happen when it runs. Part of the XrmGhost toolset."
---

# Authoring Dataverse Plugin Scenarios

A **scenario** is a JSON document that describes one operation for XrmGhost to execute locally: which message fires, on what data, what the code will read while it runs, and what you expect to happen. Think of it as *defining the execution context you are sending the code into* — not as writing a test suite. The same scenario can later be run against mocked data or, on Pro, against a live environment.

This skill covers how to *write* scenarios. Running them and interpreting results is the domain of the plugin-debugging skill.

## Where this fits in the local development flow

1. **Implement** the plugin / Custom API C#.
2. **Author scenarios** — describe the operation(s) to run. ← *this skill*
3. **Run** them with `xg` and read the result. (debugging skill)
4. **Fix loop** — adjust code or scenario, run again.

## Exploring behavior: the happy path and the failure path

The real value of authoring scenarios is not "covering" the code — it is **probing how it behaves** and surfacing gaps you had not considered. Write both kinds deliberately:

- **Happy path** — the operation as it is meant to run, with valid data, expecting success.
- **Failure path** — the operation with something deliberately wrong or missing, expecting the code to reject it.

Authoring failure scenarios is a *discovery* tool: forcing yourself to ask "what if this field is missing? what if this status is unexpected? what if the related record isn't there?" is how edge cases and gaps in the implementation come to light. Treat it as interrogating the design, not as box-ticking.

## Minimum scenario shape

```json
{
  "scenarioName": "Short description of the operation",
  "pluginAssemblyName": "MyCompany.Plugins.dll",
  "pluginTypeName": "MyCompany.Plugins.Namespace.PluginClass",
  "executionContext": {
    "messageName": "Create",
    "primaryEntityName": "account",
    "primaryEntityId": "00000000-0000-0000-0000-000000000001",
    "stage": 20,
    "mode": 0,
    "inputParameters": {
      "Target": {
        "logicalName": "account",
        "id": "00000000-0000-0000-0000-000000000001",
        "attributes": { }
      }
    }
  }
}
```

## Attribute value formats (get these right first)

Inside any `attributes` object, value typing must be explicit or the framework misreads it. This is the single most common source of authoring errors:

- **Plain values** (string, number, bool): written directly — `"name": "Contoso Ltd"`, `"revenue": 1000000`.
- **OptionSet / Picklist**: `{ "value": X }` — never the bare number.
- **EntityReference / lookup**: `{ "logicalName": "...", "id": "...", "name": "..." }` (`name` optional).
- **GUIDs must be valid hex.** IDs with non-hex characters fail to parse inside EntityReference objects. Use real GUIDs for test records.

## Describing the expected outcome

What you expect when the operation runs:

- **Expect success**: omit `expectedException`. A run that does not throw is the success.
- **Expect rejection**: declare `expectedException` with a type and a substring of the message. The run matches only if that exception is thrown.

```json
{
  "scenarioName": "Rejects contact without parent account",
  "pluginAssemblyName": "MyCompany.Plugins.dll",
  "pluginTypeName": "MyCompany.Plugins.Contact.ValidateContact",
  "executionContext": {
    "messageName": "Create",
    "primaryEntityName": "contact",
    "primaryEntityId": "1a8f6b0e-9e8a-4b6d-9f0c-7e3a5d2b1c0f",
    "stage": 20,
    "inputParameters": {
      "Target": {
        "logicalName": "contact",
        "id": "1a8f6b0e-9e8a-4b6d-9f0c-7e3a5d2b1c0f",
        "attributes": { "fullname": "John Doe" }
      }
    }
  },
  "expectedException": {
    "typeName": "Microsoft.Xrm.Sdk.InvalidPluginExecutionException",
    "messageContains": "Parent account is required."
  }
}
```

## Supplying data the code reads

When the code reads other records while it runs, describe that data. There are two distinct mechanisms — using the wrong one is the second most common mistake.

### `Retrieve` (single record) → `organizationServiceMock.retrieveResponses`

Each entry matches one `Service.Retrieve(entityName, id, ...)` call. Attributes are **nested** under an `attributes` object.

```json
"organizationServiceMock": {
  "retrieveResponses": [
    {
      "request": { "entityName": "account", "id": "3c9f7c1f-0d9b-4c7e-af1d-8f4b6c3d2e1b" },
      "response": {
        "logicalName": "account",
        "id": "3c9f7c1f-0d9b-4c7e-af1d-8f4b6c3d2e1b",
        "attributes": { "name": "Contoso Ltd", "telephone1": "+1 555 0100" }
      }
    }
  ]
}
```

### `RetrieveMultiple` (query) → root-level `entities`

For `Service.RetrieveMultiple()`, provide candidate records under a **root-level** `entities` key (a sibling of `executionContext`), grouped by logical name. Here attributes sit **directly on the entity object**, NOT nested under `attributes` — the opposite of `retrieveResponses`.

```json
"entities": {
  "account": [
    { "logicalName": "account", "id": "00000000-0000-0000-0000-000000000001", "name": "Contoso Ltd", "revenue": 1000000, "industrycode": { "value": 1 } },
    { "logicalName": "account", "id": "00000000-0000-0000-0000-000000000002", "name": "Fabrikam Inc", "revenue": 2000000, "industrycode": { "value": 2 } }
  ],
  "contoso_membership": [
    { "logicalName": "contoso_membership", "id": "99999999-9999-9999-9999-999999999999", "contoso_accountid": { "logicalName": "account", "id": "00000000-0000-0000-0000-000000000001" }, "contoso_status": { "value": 2 } }
  ]
}
```

Rules for `entities`:
- Root-level — sibling of `executionContext`, never nested inside it.
- `logicalName` and `id` at the root of each entity object; attributes flat on the object.
- Include every field the query's filter references, or the record gets filtered out.
- OptionSet → `{ "value": X }`; EntityReference → `{ "logicalName": "...", "id": "..." }`.
- The CLI applies the QueryExpression itself (logical name, `Equal`/`In` filters, column set, ordering) and returns only matches.

> Do **not** use the legacy `retrieveMultipleResponses` format (attributes nested inside a `response.entities` array). Use the root-level `entities` structure above. A single scenario may combine `entities` (for RetrieveMultiple) and `retrieveResponses` (for Retrieve).

## Pre-entity images (PreImage)

For a plugin registered with a pre-image, provide a **root-level** `preEntityImages` key — a sibling of `executionContext`, never inside it (nesting it there causes a deserialization error).

```json
"preEntityImages": {
  "PreImage": {
    "logicalName": "account",
    "id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
    "attributes": {
      "name": "Original Name",
      "primarycontactid": { "logicalName": "contact", "id": "b1111111-1111-1111-1111-111111111111", "name": "Jane Smith" }
    }
  }
}
```

- The key (`"PreImage"`) must match the image name registered on the step and read via `Context.PreEntityImages["PreImage"]`.
- Attribute format matches `inputParameters.Target.attributes`: `{ "value": X }` for OptionSets, `{ "logicalName": "...", "id": "..." }` for EntityReferences.

## Custom API scenarios: the parameters

A Custom API scenario differs from an entity plugin in how its **input parameters** are written:

- **Message name**: `custom_<publisherprefix>_<ApiName>`, e.g. `custom_contoso_EvaluateMembership` — the `custom_` segment and the publisher prefix are both required.
- **No `Target`**: each declared request parameter is a **direct key** under `inputParameters`, not wrapped in a `Target`.
- **Primary entity**: empty string `""` or omitted — Custom APIs are global, not bound to a record.
- **Supported input types**: String, Integer (`42`), Boolean (`true`), Decimal (`12.5`), Guid (as a hex string), DateTime (ISO 8601 string).

```json
{
  "scenarioName": "Custom API - evaluate membership",
  "pluginAssemblyName": "MyCompany.Plugins.dll",
  "pluginTypeName": "MyCompany.Plugins.General.EvaluateMembership_CustomAPI",
  "executionContext": {
    "messageName": "custom_contoso_EvaluateMembership",
    "primaryEntityName": "",
    "stage": 30,
    "mode": 0,
    "inputParameters": {
      "UserId": "12345678-1234-1234-1234-123456789012",
      "EffectiveDate": "2026-01-01T00:00:00Z"
    }
  }
}
```

Backing data (`retrieveResponses`, `entities`) works exactly as above — a Custom API reads data the same way a plugin does.

## How the data source affects what you mock

Before mocking backing data, know which data source the run will use (set by `--data-source` at run time):

- `mock` (default): every record a query or retrieve needs must be in the scenario.
- `auto` (Pro): mock only what you want to pin or override; the rest is read live.
- `live` (Pro): backing data in the scenario is ignored — keep the scenario focused on `Target`, input parameters, and expected outcome.

If the run will be live or auto, **do not mock data that will come from the real environment** — those mocks are dead weight and can mislead. Decide this before writing the backing data.

## Best practices

- One scenario per operation; name files by intent (`EnrichContact_HappyPath`, `ValidateContact_MissingParent`).
- Mock only what the code actually reads for that scenario.
- Valid hex GUIDs everywhere.
- Right mechanism: `retrieveResponses` (attributes nested) for Retrieve; root-level `entities` (attributes flat) for RetrieveMultiple.
