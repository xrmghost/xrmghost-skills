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
- **OptionSet / Picklist**: `{ "value": X }` — never the bare number. The discriminator may also be written explicitly as `{ "type": "OptionSetValue", "value": X }`; both forms mean OptionSetValue, and the untagged `{ "value": X }` stays OptionSetValue.
- **Money / currency**: `{ "type": "Money", "value": X }` — the `type` tag is what makes it Money; without it the object is read as an OptionSetValue.
- **EntityReference / lookup**: `{ "logicalName": "...", "id": "...", "name": "..." }` (`name` optional).
- **GUIDs must be valid hex.** IDs with non-hex characters fail to parse inside EntityReference objects. Use real GUIDs for test records.

> **Requires XrmGhost 1.7.6 or later.** The tagged Money form, the explicit OptionSetValue tag, and the Retrieve behaviors described under *Supplying data the code reads* (Retrieve reading the `entities` seed, the mock-mode precedence of in-run mutations, ColumnSet projection, the Dataverse fault instead of `KeyNotFoundException` on a Retrieve, a malformed seed failing the run) apply from **1.7.6**. Check the installed version with `xg --version`, or bring the host up to date with `xg setup host --update`. On earlier versions `{ "type": "Money", "value": X }` is read as an OptionSetValue, and Retrieve does not see `entities`. This note does **not** cover the `Update`/`Delete` faults nor the CRUD routing of `Execute`: those carry their own marker, under *Writes the code makes* and *The `Execute` boundary*.

## Describing the expected outcome

What you expect when the operation runs:

- **Expect success**: omit `expectedException`. A run that completes without throwing is the success.
- **Expect rejection**: declare `expectedException` with a `typeName`. The run is the success only if an exception satisfying it is thrown — and an `expectedException` that is declared but never raised during the run is a **RED** outcome, not a green one: a rejection scenario fails when the rejection does not happen.

### How `expectedException` is matched

- **`typeName` is required.** It is the **fully qualified** type name — namespace included — and it is compared **exactly** (case-sensitive), so copy it verbatim — see *When the record is not there* for the full name of the Dataverse fault. A simple name such as `InvalidPluginExecutionException` does not match, and neither does a derived type that would be assignable to the declared one. An empty `typeName` is not a wildcard: it matches nothing, so a scenario that leaves it empty can never pass.
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

When the code reads other records while it runs, describe that data. There are two distinct mechanisms, with **different attribute shapes** — mixing up the shapes is the second most common mistake:

- `organizationServiceMock.retrieveResponses` — the explicit answer to a `Retrieve`; attributes **nested** under `attributes`.
- Root-level `entities` — the seed of records the run starts with; attributes **flat** on the entity object. `RetrieveMultiple` queries it, and from 1.7.6 `Retrieve` reads it too.

### `Retrieve` (single record) → `organizationServiceMock.retrieveResponses` or the `entities` seed

Each `retrieveResponses` entry matches one `Service.Retrieve(entityName, id, ...)` call. Attributes are **nested** under an `attributes` object.

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

A record that is not in `retrieveResponses` can be supplied through the root-level `entities` seed instead (shape below): `Retrieve` finds it there by logical name and id. The two structures can coexist in the same scenario — `retrieveResponses` stays the explicit answer for a given Retrieve, `entities` the data both Retrieve and RetrieveMultiple can read.

The asymmetry is deliberate: **`retrieveResponses` does not feed `RetrieveMultiple`.** A record described only there is invisible to a query until a mutation of the run — an `Update` of it — puts it into the run state; from then on the query sees it. A record a query must find from the start goes in `entities`.

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
- OptionSet → `{ "value": X }` (or `{ "type": "OptionSetValue", "value": X }`); Money → `{ "type": "Money", "value": X }`; EntityReference → `{ "logicalName": "...", "id": "..." }`.
- The CLI applies the QueryExpression itself (logical name, `Equal`/`In` filters, column set, ordering) and returns only matches.
- `Retrieve` reads the same records (1.7.6+), by logical name and id.
- A malformed seed entity **fails the run**: the run stops with a message naming `EntitiesJson`, the entity's logical name, its index in the list and, when known, the attribute that could not be converted — it does not continue with zero entities. Fix the named entry rather than looking for the problem in the code.

> Do **not** use the legacy `retrieveMultipleResponses` format (attributes nested inside a `response.entities` array). Use the root-level `entities` structure above. A single scenario may combine `entities` and `retrieveResponses`.

### Which source a `Retrieve` sees (precedence)

With `--data-source mock`, when more than one source holds the same record (logical name + id), `Retrieve` answers from the first one that has it, in this order:

1. **In-run mutations** — what the code itself did earlier in the same run (`Create`, `Update`, `Delete`; see *Writes the code makes*). After an `Update`, a later `Retrieve` of that record sees the updated values; after a `Delete`, a later `Retrieve` of that record is not-found (the fault below), even if `retrieveResponses` or `entities` still describe it.
2. **`retrieveResponses`** — the explicit answer.
3. **`entities`** seed.

With `auto` and `live` the order is different, because Dataverse comes first:

- **`auto`** — `Retrieve` and `RetrieveMultiple` ask Dataverse first. A `Retrieve` falls back to the run state and the mocks (in the order above) only when Dataverse answers that the record does not exist or cannot be reached. A `RetrieveMultiple` reads the mocks only when no live attempt is made (no Pro license or no active environment); a live attempt that fails returns an empty collection with a warning, not mock rows. So a record Dataverse holds is answered **by Dataverse**, not by the run state: an `Update` or `Delete` the code made on it earlier in the run is not reflected in that answer.
- **`live`** — `Retrieve` and `RetrieveMultiple` never read the mocks nor the run state.

The in-run consistency "a mutation is visible to the next `Retrieve`/`RetrieveMultiple`" therefore holds in `mock`, and in `auto` only for records Dataverse does not have.

### Which columns a `Retrieve` returns (ColumnSet)

`Retrieve` projects the record onto the `ColumnSet` the code passes, whichever source the record came from:

- **Columns required** (`new ColumnSet("name", "telephone1")`) — only those attributes, the ones the source has.
- **Empty** (`new ColumnSet()`) — no attributes besides the primary key.
- **AllColumns** (`new ColumnSet(true)`) — every attribute the source has.

The primary key is always returned. Mock every column the code requests, or it will be absent from the returned entity.

### When the record is not there

A `Retrieve` of a record no source has — or with `Guid.Empty` as id, or after a `Delete` in the same run — raises the Dataverse fault, as Dataverse does: `FaultException<OrganizationServiceFault>` with ErrorCode `0x80040217` (object does not exist). It is **not** a `KeyNotFoundException`. To author a failure path on it, declare exactly:

```json
"expectedException": {
  "typeName": "System.ServiceModel.FaultException`1[[Microsoft.Xrm.Sdk.OrganizationServiceFault, Microsoft.Xrm.Sdk, Version=9.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]"
}
```

With `--data-source live`, a Retrieve of a record Dataverse does not have raises Dataverse's own fault, unchanged — same type, same `typeName`.

## Writes the code makes: `Create`, `Update`, `Delete` (the run state)

> **Requires the first host release after 1.1.0 that includes ADO-3974.** The `Update`/`Delete` faults below and the CRUD routing of `Execute` are not in 1.7.6: on earlier hosts an `Update` or `Delete` of an unknown record does not fault. Check with `xg --version` and update with `xg setup host --update`.

`Create`, `Update` and `Delete` write into the **run state**: an in-memory record of what the code did during this run. The run state:

- **outranks the declared sources** — a record updated or deleted in the run reads as the run left it, whatever `retrieveResponses` or `entities` still say (in `mock`; see the precedence above for `auto`/`live`);
- **is shared by every service** of the run — a record written through one `IOrganizationService` reads the same through another;
- **never rewrites the scenario and never writes to Dataverse** — in every data source, `live` included. The next run starts again from the scenario as written.

The existence checks of these verbs read the records declared in the scenario **in every data source**: a `Create` duplicate, the base an `Update` merges onto, the record a `Delete` removes — in `auto` and `live` too.

### Verb → effect on the run state → fault

The fault messages below use `{entity}` and `{id}` for the logical name and id of the call. Every fault in this table is `FaultException<OrganizationServiceFault>`, declared with the same `typeName` as *When the record is not there*.

| Verb | Effect on the run state | Fault |
|---|---|---|
| `Create` | Adds the record, with the id it passed or a generated one; a later `Update`/`Delete` finds it. | Explicit id already known to any source of the run — `entities`, `organizationServiceMock.retrieveResponses`, a record created or updated in the run — raises ErrorCode `0x80040237` (DuplicateRecord), message `Cannot insert duplicate key. {entity} With Id = {id} already exists.` A record deleted in the run does not count as known. |
| `Retrieve` | None. | Record no source has, `Guid.Empty`, or deleted in the run: ErrorCode `0x80040217`, message `{entity} With Id = {id} Does Not Exist` (`mock`; `auto`/`live` per the precedence above). |
| `RetrieveMultiple` | None. | No fault for "no match": the collection is empty. |
| `Update` | Merges the attributes passed onto the record as `Retrieve` would read it from the run's sources; the attributes passed are kept. A record known only to Dataverse is kept in the run with just the attributes passed. Nothing is written to Dataverse. | Record unknown to every source, `Guid.Empty`, or deleted earlier in the run: ErrorCode `0x80040217`, message `{entity} With Id = {id} Does Not Exist`. |
| `Delete` | Leaves a **tombstone** for the record in the run. Nothing is deleted in Dataverse. | Same as `Update`: ErrorCode `0x80040217`, message `{entity} With Id = {id} Does Not Exist`. |
| `Execute(CreateRequest)` | As `Create`. | As `Create`. |
| `Execute(RetrieveRequest)` | As `Retrieve`. | As `Retrieve`. |
| `Execute(RetrieveMultipleRequest)` | As `RetrieveMultiple`. | As `RetrieveMultiple`. |
| `Execute(UpdateRequest)` | As `Update`. | As `Update`. |
| `Execute(DeleteRequest)` | As `Delete`. | As `Delete`. |

An `Execute` request whose `Target` (or, for `RetrieveMultipleRequest`, `Query`) is null throws `System.InvalidOperationException` before reaching the verb.

### How `Update` and `Delete` decide the record exists

In order:

1. **The run's tombstone wins.** A record deleted earlier in the run makes any later `Update` or `Delete` of it fault with `0x80040217`, in **every** data source, without asking Dataverse. The tombstone outranks Dataverse **only for this existence check**: a `Retrieve`/`RetrieveMultiple` in `auto` or `live` still asks Dataverse first (see the precedence above).
2. **The run's sources** — the run state, `retrieveResponses`, the `entities` seed — in every data source.
3. **Dataverse**, only in `auto` and `live` with a Pro license and an active environment. If Dataverse answers that the record does not exist, its own fault (`0x80040217`) is raised unchanged. If the live check fails for any other reason, the record is treated as not existing — the `0x80040217` fault, plus a warning in `auto`.

`Guid.Empty` never exists. In `mock` there is no step 3.

### Happy path: `Update` / `Delete` of an existing record

- **`mock`** — declare the record first, in `entities` or in `organizationServiceMock.retrieveResponses`. Without it the call faults.
- **`auto` / `live`** — a record that exists in Dataverse is **not** declared: the existence check finds it there, and the update or delete stays in the run state without writing to Dataverse. Declare it only when it is not in Dataverse.

A plugin that loads an account and updates its phone number, run with `--data-source mock`:

```json
"entities": {
  "account": [
    { "logicalName": "account", "id": "3c9f7c1f-0d9b-4c7e-af1d-8f4b6c3d2e1b", "name": "Contoso Ltd", "telephone1": "+1 555 0100" }
  ]
}
```

After `Update` with `telephone1 = "+1 555 0199"`, a later `Retrieve` in the same run returns `name = "Contoso Ltd"` and `telephone1 = "+1 555 0199"`. After `Delete`, a later `Retrieve`, `Update` or `Delete` of that id faults with `0x80040217`.

### Failure path: `Update` / `Delete` of a record that is not there

A record no source knows, `Guid.Empty` as id, or a record already deleted in the run: the call raises `FaultException<OrganizationServiceFault>` with ErrorCode `0x80040217`, as Dataverse does. When the plugin lets it propagate, declare exactly:

```json
"expectedException": {
  "typeName": "System.ServiceModel.FaultException`1[[Microsoft.Xrm.Sdk.OrganizationServiceFault, Microsoft.Xrm.Sdk, Version=9.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]",
  "messageContains": "Does Not Exist"
}
```

When the plugin catches it and wraps it in `InvalidPluginExecutionException`, declare the wrapper, as under *How `expectedException` is matched*. A duplicate `Create` takes the same `typeName`, with `messageContains: "already exists"`.

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

- the CRUD requests `CreateRequest`, `UpdateRequest`, `DeleteRequest`, `RetrieveRequest` and `RetrieveMultipleRequest` — on every license, each delegated to the matching method (`Create`, `Update`, `Delete`, `Retrieve`, `RetrieveMultiple`), with the same run-state effect and the same faults (see *Verb → effect on the run state → fault*). **Requires the first host release after 1.1.0 that includes ADO-3974** for `CreateRequest`, `UpdateRequest` and `DeleteRequest`; on earlier hosts they fall into the `NotImplementedException` below;
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
- `live` (Pro): `Retrieve` and `RetrieveMultiple` ignore the backing data in the scenario — keep the scenario focused on `Target`, input parameters, and expected outcome. `Create`, `Update` and `Delete` still read the declared records (duplicate check, merge base, existence), as in every data source.

If the run will be live or auto, **do not mock data that will come from the real environment** — those mocks are dead weight and can mislead. Decide this before writing the backing data. Whatever the data source, the code's writes stay in the run state: nothing is written to Dataverse.

## Best practices

- One scenario per operation; name files by intent (`EnrichContact_HappyPath`, `ValidateContact_MissingParent`).
- Mock only what the code actually reads for that scenario.
- Valid hex GUIDs everywhere.
- Right shape: `retrieveResponses` (attributes nested) is the explicit answer for Retrieve; root-level `entities` (attributes flat) feeds RetrieveMultiple and, from 1.7.6, Retrieve.

## Official documentation

Authoritative, always-current reference: **https://docs.xrmghost.tech**. Link the user there for installation, CLI behavior outside day-to-day scenario authoring, and environment/licensing specifics.

- **[Getting Started](https://docs.xrmghost.tech/getting-started/)** — install, one-time `xg setup host`, license activation.
- **[CLI Reference](https://docs.xrmghost.tech/cli/reference/)** — every `xg` command and its flags.
