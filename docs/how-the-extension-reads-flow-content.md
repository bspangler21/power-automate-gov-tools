# How the Extension Reads a Flow's Content / Actions / Steps

**Last Verified:** 2026-06-03
**Source:** Source code review — `src/background.ts`, `src/common/providers/ApiProvider.ts`, `src/features/flow-editor/useFlowEditor.ts`, `src/features/flow-editor/FlowEditorPage.tsx`, `public/manifest.json`
**Branch:** `add-gov-compatibility`

## Overview

This document explains the mechanism by which the Power Automate Gov Tools extension obtains and displays a flow's definition. The central point: the extension **does not parse individual actions or steps**, and it does **not** scrape the designer's DOM. A flow's entire structure — its trigger and every action, including nesting and ordering — is encoded in a single JSON object (`properties.definition`, in Azure Logic Apps' Workflow Definition Language). The extension captures the user's own API bearer token from in-flight network requests, calls Microsoft's API to download that JSON blob, and renders it in a Monaco JSON editor. Microsoft's servers perform all parsing and validation.

## End-to-End Data Flow

| Step | What happens | Where |
|------|--------------|-------|
| 1 | Background worker sniffs outgoing requests to Power Automate API hosts and captures the `Authorization` bearer token per tab | [background.ts:84-96](../src/background.ts#L84-L96), [background.ts:148-150](../src/background.ts#L148-L150) |
| 2 | Regex extracts `envId` + `flowId` from the request URL (and tab URL) | [background.ts:257-296](../src/background.ts#L257-L296), [background.ts:222-231](../src/background.ts#L222-L231) |
| 3 | Toolbar click opens `app.html?envId=...&flowId=...` | [background.ts:53-75](../src/background.ts#L53-L75) |
| 4 | Editor app GETs the flow resource using the captured token | [useFlowEditor.ts:64](../src/features/flow-editor/useFlowEditor.ts#L64), [ApiProvider.ts:34-55](../src/common/providers/ApiProvider.ts#L34-L55) |
| 5 | App cherry-picks `properties.definition` + `connectionReferences`, stringifies, and loads into Monaco | [useFlowEditor.ts:65-78](../src/features/flow-editor/useFlowEditor.ts#L65-L78), [FlowEditorPage.tsx:81-85](../src/features/flow-editor/FlowEditorPage.tsx#L81-L85) |

## 1. Token Capture (No Own Credentials)

The extension holds no credentials. The background service worker registers `chrome.webRequest.onBeforeSendHeaders` against the flow API hosts, and on each authenticated call made by the maker portal it lifts the `Authorization` header out of the request and caches it on a per-tab basis.

```ts
// background.ts:148-150
const token = details.requestHeaders?.find(
  (x) => x.name.toLowerCase() === "authorization"
)?.value;
```

**Consequence:** the extension must "overhear" a real API call before it can act. This is why validation surfaces the message instructing the user to *open a flow in the designer first* ([useFlowEditor.ts:154](../src/features/flow-editor/useFlowEditor.ts#L154)) — it is waiting for a request from which to capture the legacy token.

### Monitored API Hosts

From the `onBeforeSendHeaders` URL filter ([background.ts:87-93](../src/background.ts#L87-L93)):

```
https://*.api.flow.microsoft.com/*
https://*.api.powerplatform.com/*
https://*.api.gov.powerplatform.microsoft.us/*
https://*.api.high.powerplatform.microsoft.us/*
https://*.flow.microsoft.us/*
```

## 2. Identifying the Flow (envId + flowId)

`extractFlowDataFromUrl` ([background.ts:257-296](../src/background.ts#L257-L296)) matches two URL shapes:

| Format | Pattern | envId source | flowId source |
|--------|---------|--------------|---------------|
| Legacy | `/providers/Microsoft.ProcessSimple/environments/{envId}/flows/{flowId}` | URL path | URL path |
| New | `/powerautomate/flows/{flowId}` | **Not in path** — recovered from browser tab URL via `extractEnvIdFromTabUrl` | URL path |

In the new format the environment ID is encoded in the request subdomain (not directly usable), so `envId` is returned as `null` and resolved separately from the tab URL ([background.ts:204-214](../src/background.ts#L204-L214)).

## 3. Fetching the Flow Definition

The editor hook issues a single GET:

```ts
// useFlowEditor.ts:64
const flow = await api.get(`powerautomate/flows/${flowId}`);
```

`api.get` → `makeRequest` ([ApiProvider.ts:34-55](../src/common/providers/ApiProvider.ts#L34-L55)) performs a plain `fetch`, attaching the captured token in the `authorization` header and appending the API version as a query parameter.

## 4. The Actions/Steps Are One Property

The hook extracts exactly three fields from the API response ([useFlowEditor.ts:65-78](../src/features/flow-editor/useFlowEditor.ts#L65-L78)):

```ts
{
  $schema: editorSchema,                              // "https://power-automate-tools.local/flow-editor.json#"
  connectionReferences: flow.properties.connectionReferences,
  definition: flow.properties.definition,             // ← trigger + all actions/steps
}
```

`flow.properties.definition` **is** the steps. In Workflow Definition Language, structure, ordering, and nesting are declarative:

```json
{
  "triggers": { "When_a_row_is_added": { } },
  "actions": {
    "Condition": {
      "type": "If",
      "actions": { "Send_email": { "runAfter": { } } }
    }
  }
}
```

- Action **ordering** is expressed via each action's `runAfter`.
- Action **nesting** is expressed by nested `actions` objects.
- There is **no DOM scraping and no per-step parser** in the extension.

The result is stringified with 2-space indentation and rendered in a Monaco editor with `language="json"` ([FlowEditorPage.tsx:81-85](../src/features/flow-editor/FlowEditorPage.tsx#L81-L85)). The fields are loaded as the editor's `defaultValue`.

## 5. Write-Back and Validation

| Operation | Method / Endpoint | API version | Notes |
|-----------|-------------------|-------------|-------|
| Save | `PATCH powerautomate/flows/{flowId}` | `1` (new API) | Sends `displayName`, `environment`, `definition`, `connectionReferences` ([useFlowEditor.ts:122-130](../src/features/flow-editor/useFlowEditor.ts#L122-L130)) |
| Validate (errors) | `POST {legacyFlowUrl}/checkFlowErrors` | `2016-11-01` (legacy API) | Server-side lint ([useFlowEditor.ts:163-171](../src/features/flow-editor/useFlowEditor.ts#L163-L171)) |
| Validate (warnings) | `POST {legacyFlowUrl}/checkFlowWarnings` | `2016-11-01` (legacy API) | Server-side lint ([useFlowEditor.ts:173-180](../src/features/flow-editor/useFlowEditor.ts#L173-L180)) |

`legacyFlowUrl` = `providers/Microsoft.ProcessSimple/environments/{envId}/flows/{flowId}` ([useFlowEditor.ts:199-201](../src/features/flow-editor/useFlowEditor.ts#L199-L201)).

Before saving, the hook requires both `definition` and `connectionReferences` to be present in the edited JSON, erroring otherwise ([useFlowEditor.ts:101-112](../src/features/flow-editor/useFlowEditor.ts#L101-L112)). The `parameters` property handling is currently commented out throughout the hook.

## Two APIs In Play

The extension tracks two distinct API base URLs and tokens per tab ([background.ts:158-175](../src/background.ts#L158-L175)):

| API | Host classifier | API version | Used for |
|-----|-----------------|-------------|----------|
| New Power Platform API | `isNewApiHost` ([background.ts:34-40](../src/background.ts#L34-L40)) — `api.powerplatform.com`, `api.gov.powerplatform.microsoft.us`, `api.high.powerplatform.microsoft.us` | `1` | Read (GET) and write (PATCH) of the flow definition |
| Legacy Flow API | `isLegacyApiHost` ([background.ts:44-49](../src/background.ts#L44-L49)) — `api.flow.microsoft.com`, `flow.microsoft.us` (api subdomain) | `2016-11-01` | Validation only (`checkFlowErrors` / `checkFlowWarnings`) |

The legacy API is retained because its validation endpoints were not ported to the new Power Platform API. If only a legacy host has been seen and no new-API URL is captured yet, the legacy URL is also used as the primary `apiUrl` as a fallback ([background.ts:168-174](../src/background.ts#L168-L174)).

### Cloud Coverage (Gov Compatibility)

Both host classifiers explicitly enumerate three clouds, which is the focus of the `add-gov-compatibility` branch:

- **Commercial** — `api.powerplatform.com` / `api.flow.microsoft.com`
- **GCC** — `api.gov.powerplatform.microsoft.us` / `flow.microsoft.us`
- **GCC High** — `api.high.powerplatform.microsoft.us` / `flow.microsoft.us`

The new-format flow-ID regex is deliberately host-agnostic ([background.ts:281-282](../src/background.ts#L281-L282)); the `webRequest` listener's URL filter already restricts which hosts reach the handler, so the path match works across all clouds without hardcoding the API hostname.

## Background ↔ App Communication

The captured token and API URLs live in the background worker's per-tab state and are pushed to the React app via `chrome.runtime` messaging:

- App, on mount, sends `app-loaded` ([ApiProvider.ts:84](../src/common/providers/ApiProvider.ts#L84)); background responds with `sendTokenChanged` ([background.ts:103-105](../src/background.ts#L103-L105), [background.ts:114-126](../src/background.ts#L114-L126)).
- Background pushes a `token-changed` message carrying `token`, `apiUrl`, `legacyApiUrl`, `legacyToken` ([background.ts:119-125](../src/background.ts#L119-L125)); the app stores these and marks the API ready when both `apiUrl` and `token` are present ([ApiProvider.ts:70-78](../src/common/providers/ApiProvider.ts#L70-L78)).

Message action types are defined in [backgroundActions.ts](../src/common/types/backgroundActions.ts): `refresh`, `app-loaded`, `token-changed`.

## Summary Table

| Goal | Mechanism | Endpoint / Field |
|------|-----------|------------------|
| Get an auth token | Sniff `Authorization` header on outgoing API requests | `chrome.webRequest.onBeforeSendHeaders` |
| Identify the flow | Regex on request URL + tab URL | `envId`, `flowId` |
| Read the flow's actions/steps | GET flow resource, read one property | `GET powerautomate/flows/{flowId}` → `properties.definition` |
| Display it | Stringify and load into JSON editor | Monaco editor, `language="json"` |
| Save edits | PATCH the definition back | `PATCH powerautomate/flows/{flowId}` (api-version `1`) |
| Validate | POST to legacy lint endpoints | `checkFlowErrors` / `checkFlowWarnings` (api-version `2016-11-01`) |

## Notes

- All field names, endpoints, and values above are confirmed from the current source code on the `add-gov-compatibility` branch.
- The `parameters` flow property is referenced only in commented-out code and is not currently read or written.
- "Steps" is not a first-class concept in the code; it is shorthand for the trigger + actions encoded in the Workflow Definition Language `definition` object.
