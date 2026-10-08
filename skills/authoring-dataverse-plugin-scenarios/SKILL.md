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

| Type | Write it as | Notes |
|------|-------------|-------|
| Plain value (string, number, bool) | `"name": "Contoso Ltd"`, `"numberofemployees": 250`, `"donotemail": true` | A bare number arrives as a plain number, never as `Money` (`100` in `Target.attributes` was observed as `System.Int32`). |
| OptionSet / Picklist | `{ "value": X }` | Never the bare number. |
| EntityReference / lookup | `{ "logicalName": "...", "id": "...", "name": "..." }` | `name` optional. GUIDs must be valid hex — IDs with non-hex characters fail to parse. Use real GUIDs for test records. |
| Money | **No working form — known limit** | Probed with released `xg` (CLI 1.7.5, host 1.0.2) in `Target.attributes`, `retrieveResponses`, root-level `entities` and `preEntityImages`: no context delivered a `Microsoft.Xrm.Sdk.Money`. `{ "value": 100 }` arrives as an `OptionSetValue` in all four; `{ "value": 100.50 }` and `{ "value": 100.0 }` are rejected while the scenario is parsed (`Target.attributes`, `retrieveResponses`), fail the run with a `System.FormatException` (`preEntityImages`), or silently drop the record (root-level `entities`). |

**Money is not supported by scenarios today.** Do not write `{ "value": N }` — or any other shape — expecting the plugin to read a `Money`: the probe above found none, in any of the four contexts. A plugin path that reads a Money attribute cannot be driven green by a scenario yet; say so to the user instead of inventing a form, and keep the Money-reading logic covered by a pure-logic unit test until the follow-up bug on XrmGhost Core is fixed.

## Describing the expected outcome

What you expect when the operation runs:

- **Expect success**: omit `expectedException`. A run that completes without throwing is the success.
- **Expect rejection**: declare `expectedException` with a `typeName`. The run is the success only if an exception satisfying it is thrown — and an `expectedException` that is declared but never raised during the run is a **RED** outcome, not a green one: a rejection scenario fails when the rejection does not happen.

### How `expectedException` is matched

- **`typeName` is required.** It is the **fully qualified** type name — namespace included — and it is compared **exactly**: a simple name such as `InvalidPluginExecutionException` does not match, and neither does a derived type that would be assignable to the declared one. An empty `typeName` is not a wildcard: it matches nothing, so a scenario that leaves it empty can never pass.
- **The comparison is applied to every link of the exception chain**, walked from the outermost exception inward through each `InnerException`. One link satisfying the expectation is enough.
- **`messageContains` is optional.** When present it is a substring of the exception message, and it is checked on the **same link that satisfied the type** — not on another link, and not on the chain as a whole. Omit it when the type alone is what you mean to assert.

When the plugin wraps an internal failure — `throw new InvalidPluginExecutionException("...", inner)` — declare the type **Dataverse shows the caller**, which is the wrapper (`Microsoft.Xrm.Sdk.InvalidPluginExecutionException`), not the inner type. The inner type would also be found, since the chain is walked, but asserting it pins an implementation detail instead of the contract the caller sees.

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

Dropping `messageContains` asserts the rejection type and nothing about its wording — useful when the message is not part of what you are pinning down:

```json
"expectedException": {
  "typeName": "Microsoft.Xrm.Sdk.InvalidPluginExecutionException"
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

## New Custom API or webhook wiring: a pair of xg scenarios

When the code adds or changes **wiring** — a Custom API, or a plugin that receives a webhook and calls another API through `IOrganizationService.Execute` — author **a pair of xg scenarios** for it, even when xUnit + Moq tests already cover the logic:

- **The call path** — the input that makes the code call out.
- **The skip path** — the input that makes the code decide *not* to call out, ending clean.

The pair is what proves the wiring: message name, parameters, branching. The rule against mocks is narrow: do not write a **new mock of `IOrganizationService`** to stand in for these scenarios. xUnit + Moq tests of **pure logic** (parsing, mapping, decisions that never touch the service) stay legitimate and are not replaced by the pair.

### One plugin per run, and the `Execute` boundary

`xg` runs **exactly one plugin type per run** — the one named by `--plugin-type`. There is no pipeline cascade: when that plugin calls another Custom API through `Execute`, the other API's plugin does not run. The call stops at the host's `Execute` boundary instead, which handles only:

- `RetrieveRequest` — on every license;
- the metadata requests `RetrieveEntityRequest`, `RetrieveAllEntitiesRequest`, `RetrieveAttributeRequest` and `RetrieveRelationshipRequest` — **Pro**: without a Pro license they throw `ProFeatureNotLicensedException`, which `expectedException` can never catch.

**Every other request** throws `System.NotImplementedException` with this message, where `{RequestName}` is the request's name:

```text
FrameworkOrganizationServiceAdapter.Execute for request type {RequestName} is not yet implemented.
```

So the call path of a wiring scenario ends at that boundary. Deduce it **before the first run**, from the code: the request is not in the list above, so the scenario declares the boundary exception up front. Declare it exactly:

- `typeName`: `System.NotImplementedException`, fully qualified.
- `messageContains`: the full sentence above with the real `{RequestName}`. The `{RequestName}` is the string **the plugin code passes** — the argument of `new OrganizationRequest("...")`, or the `RequestName` of the early-bound request class — **without** the `custom_` prefix. The `custom_` prefix belongs only to a scenario's `messageName`.

Two consequences to state to the user:

- **The code after `Execute` is not covered.** Whatever the plugin does with the response — tracing, output parameters, follow-up writes — never runs in the call path. Cover it with a pure-logic test or on a real environment.
- **The test depends on the host's message text.** If a future host release rewords the message, `messageContains` stops matching and the call path turns red. Re-check the wording against the released host rather than loosening the assertion.

### Example: `custom_contoso_ProcessWebhook` calling `contoso_SendNotification`

The plugin behind the `custom_contoso_ProcessWebhook` Custom API reads `EventType` and `Payload`. For `EventType = "notification"` it calls the `contoso_SendNotification` Custom API with `service.Execute(new OrganizationRequest("contoso_SendNotification") { ["Message"] = payload })`; for any other event it traces and returns, setting `Handled = false`.

Call path — the run is green only because the declared boundary exception is raised, with the exact `RequestName`:

```json
{
  "scenarioName": "Custom API - webhook notification reaches contoso_SendNotification",
  "pluginAssemblyName": "Contoso.Plugins.dll",
  "pluginTypeName": "Contoso.Plugins.Webhooks.ProcessWebhook",
  "executionContext": {
    "messageName": "custom_contoso_ProcessWebhook",
    "primaryEntityName": "",
    "stage": 30,
    "mode": 0,
    "inputParameters": {
      "EventType": "notification",
      "Payload": "Order 1001 shipped"
    }
  },
  "expectedException": {
    "typeName": "System.NotImplementedException",
    "messageContains": "FrameworkOrganizationServiceAdapter.Execute for request type contoso_SendNotification is not yet implemented."
  }
}
```

Skip path — no `expectedException`; the run must end clean:

```json
{
  "scenarioName": "Custom API - webhook event without notification is skipped",
  "pluginAssemblyName": "Contoso.Plugins.dll",
  "pluginTypeName": "Contoso.Plugins.Webhooks.ProcessWebhook",
  "executionContext": {
    "messageName": "custom_contoso_ProcessWebhook",
    "primaryEntityName": "",
    "stage": 30,
    "mode": 0,
    "inputParameters": {
      "EventType": "heartbeat",
      "Payload": "ping"
    }
  }
}
```

Keep the two scenarios distinct. A call path that turns green on a **different** `RequestName`, or a skip path that hits `Execute` at all, means the code calls something you did not declare — that is a finding, not a pass.

## Custom API scenarios: the parameters

A Custom API scenario differs from an entity plugin in how its **input parameters** are written:

- **Message name**: `custom_<publisherprefix>_<ApiName>`, e.g. `custom_contoso_EvaluateMembership` — the `custom_` segment and the publisher prefix are both required.
- **Request name is not the message name.** When the code calls another Custom API through `Execute`, the `{RequestName}` asserted in `messageContains` is the name the code passes (`contoso_SendNotification`), never the scenario form `custom_contoso_SendNotification` — see the `Execute` boundary above.
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

## Official documentation

Authoritative, always-current reference: **https://docs.xrmghost.tech**. Link the user there for installation, CLI behavior outside day-to-day scenario authoring, and environment/licensing specifics.

- **[Getting Started](https://docs.xrmghost.tech/getting-started/)** — install, one-time `xg setup host`, license activation.
- **[CLI Reference](https://docs.xrmghost.tech/cli/reference/)** — every `xg` command and its flags.
