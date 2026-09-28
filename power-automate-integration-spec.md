# Power Automate → App Integration Spec

Generic contract for connecting this app to Power Automate flows that run SQL Server stored procedures and return results. This spec is not specific to any one flow — per-flow details are supplied separately using the template in Section 6.

An app can have many flows. All flows follow this same contract.

**Source of truth:** https://raw.githubusercontent.com/paul-linksunlimited/power-automate-integration/main/power-automate-integration-spec.md — this spec is maintained at that URL and may change. Do not save a copy into the project. Re-fetch it from that URL whenever setting up a new flow (Section 6) or working on this integration.

## 1. Secrets (per application, not per flow)

| Secret | Direction | Purpose |
|---|---|---|
| `TriggerKey` | App → Flow | Sent in every trigger request. Flow rejects the request if wrong/missing. Ask the user for this — do not generate it. It comes from the Power Automate side. |
| `CallbackKey` | Flow → App | Sent by Power Automate in the `x-ingest-secret` header of every callback. App must verify it. Ask the user for this — do not generate it. It comes from the Power Automate side. |
| `CallbackURL` | App → User | The base URL Power Automate calls back to (see Section 2). The app generates this itself, then gives it to the user, who configures it in Power Automate. |

`TriggerKey` and `CallbackKey` are secrets — store server-side only, never in client-side code. `CallbackURL` is not a secret the app needs to store; it's a value the app generates once and hands back to the user (see Section 2), who configures it in Power Automate.

## 2. What to build

- **One ingestion endpoint**: `POST <CallbackURL>/:flow` — a single route parameterized by `flow`, not one endpoint per flow. Only used by async flows (Section 4); sync flows (Section 5) never call it.
- **Request table** (async, app-triggered flows only): tracks the lifecycle of each `requestId` this app started. Required fields — dictated by the wire contract: `requestId`, `flow`, `parameters`, `status` (`pending` / `succeeded` / `failed` / `timed_out`), `runId`, `errorCode`, `error`, from the trigger request (Section 3) and callback (Section 4). Also required: `triggerHttpStatus` — the HTTP status Power Automate returned to the trigger call itself (Section 3), recorded when the trigger is sent. This is the only way to tell "the flow never ran" (e.g. a 400 schema rejection) apart from "the flow ran and failed," since a rejected trigger produces no run and therefore no callback. Recommended, not required: `created` and `completed` timestamps for tracking run duration and spotting stuck/slow flows — `created` set when the trigger is sent, `completed` set when a matching callback is processed (can simply reuse the callback's own `generatedAt` rather than tracking a separate receipt time). Row is created `pending` on trigger, then updated with `status`/`runId`/`errorCode`/`error`/`completed` when its callback arrives. Sync flows (Section 5) don't use this table — there's no pending state, since the result comes back on the same call.
- **Results storage** (async only): the actual data from every accepted callback — app-triggered, scheduled, and failures alike — storing the full envelope and `rows`. This is separate from the request table: a scheduled callback has no request to update, but its result still needs to be stored (see Section 4). Sync flow results are transient by default — used once by whatever called them — unless the per-flow instructions (Section 6) say to persist them.
- **Trigger action** (server-side, needed for any flow the app calls — async or sync, see Section 6): for async, generate a `requestId`, save it `pending` in the request table, POST to the flow's trigger URL, handle the response per Section 3. For sync, POST the same way but read the result straight from that call's response — see Section 5.
- **Timeout job** (async only): mark `pending` requests `timed_out` after 15 minutes with no callback.

Once built, give the user the `CallbackURL` — this is the only value the app produces that must go back to Power Automate.

## 3. Trigger request (App → Flow)

This request shape is the same whether the flow is async (Section 4) or sync (Section 5) — the per-flow instructions (Section 6) say which mode a given flow uses. Not all flows are app-triggered at all — a scheduled/recurrence flow (Section 4) has no trigger request.

```json
{
  "requestId": "6f1c2e0a-9b7d-4c1e-8a55-2f3d9c0b1a77",
  "flow": "example-flow",
  "triggerKey": "<TRIGGER_KEY>",
  "ProcedureParameters": { "startDate": "2026-01-01", "endDate": "2026-01-31" }
}
```

- `flow`: the flow's registered id. Determines where its callback is sent.
- `ProcedureParameters`: omit if the procedure takes no parameters.
- Use the trigger URL exactly as given for that flow.

The table below applies to **async flows**. For sync flows, see Section 5 — the result comes back as the body of this same call instead.

| Trigger HTTP response | App action (async) |
|---|---|
| 202 | Keep request `pending`. |
| 400 | Body didn't match the flow's schema; no run was created. Mark `failed`. |
| 401 / 403 | Trigger URL wrong/incomplete. Mark `failed`. |
| 429 | Throttled. Mark `failed` (user can retry). |
| Other / network error | Mark `failed`. |

## 4. Callback (Flow → App) — async flows

`POST <CallbackURL>/<flow>`, headers `Content-Type: application/json` and `x-ingest-secret: <CALLBACK_KEY>`.

Every outcome — success, failure, and scheduled runs — uses the same body shape:

```json
{
  "requestId": "6f1c2e0a-9b7d-4c1e-8a55-2f3d9c0b1a77",
  "flow": "example-flow",
  "status": "succeeded",
  "runId": "08584...",
  "generatedAt": "2026-01-31T19:12:03.123Z",
  "parameters": { "startDate": "2026-01-01", "endDate": "2026-01-31" },
  "errorCode": null,
  "error": null,
  "rows": [ { "id": 1, "name": "Example", "quantity": 42 } ]
}
```

Failure examples — there are two failure modes, both use `status: "failed"` with a different `errorCode`:

```json
{
  "requestId": "6f1c2e0a-9b7d-4c1e-8a55-2f3d9c0b1a77",
  "flow": "example-flow",
  "status": "failed",
  "runId": "08584...",
  "generatedAt": "2026-01-31T19:12:03.123Z",
  "parameters": { "startDate": "2026-01-01", "endDate": "2026-01-31" },
  "errorCode": "invalid_secret",
  "error": "Flow rejected the request: missing or wrong secret key",
  "rows": []
}
```

```json
{
  "requestId": "6f1c2e0a-9b7d-4c1e-8a55-2f3d9c0b1a77",
  "flow": "example-flow",
  "status": "failed",
  "runId": "08584...",
  "generatedAt": "2026-01-31T19:12:03.123Z",
  "parameters": { "startDate": "2026-01-01", "endDate": "2026-01-31" },
  "errorCode": "procedure_failed",
  "error": "<message from the stored procedure's own error>",
  "rows": []
}
```

| Field | Notes |
|---|---|
| `requestId` | Echoed from the trigger request. **Empty string `""` for scheduled flows** — there is no app-initiated request to echo. |
| `flow` | The flow id. Always equals the URL's `:flow` segment. |
| `status` | `succeeded` or `failed`. |
| `runId` | Power Automate run id. Unique per run. Used for duplicate detection. |
| `generatedAt` | UTC time the callback was sent. |
| `parameters` | Echoed `ProcedureParameters`, or `null`. |
| `errorCode` | `null` on success; e.g. `invalid_secret`, `procedure_failed` on failure. |
| `error` | Readable message, or `null`. |
| `rows` | Result set, field names exactly as the procedure returns. `[]` on failure. The SQL connector's on-premises gateway caps the response at 8 MB — keep procedures scoped to result sets well under that. |

**Handling `requestId`:**
- Non-empty → look up the matching pending request, update its status/fields.
- Empty (scheduled flow) → no request to update. Store the result the same as any other accepted callback (Section 2's results storage, deduped by `runId` as below) — there's just no request-table row to update alongside it.

**App's reply:**

| Status | When |
|---|---|
| 202 | Stored. |
| 200 `{ "duplicate": true }` | This `runId` was already stored — return this instead of storing again. |
| 400 | Bad JSON; unknown `flow`; body `flow` ≠ URL `flow`; invalid `status`/`errorCode`; or a `succeeded` callback missing the flow's expected row fields. |
| 401 | Missing or wrong `x-ingest-secret`. |

> **Reply as soon as the result is stored — before any heavier processing.** Power Automate is waiting on this HTTP response; a slow reply risks a timeout on the flow's side, which can trigger a retry of the same callback. Do validation and the storage write first, send the response, then do anything else (transforms, notifications, etc.) afterward.

Because retries can happen, a duplicate `runId` must not be stored twice.

## 5. Synchronous flows

An alternative to Section 4, for flows where the result comes back on the same HTTP call — no callback, no `/ingest` call, no request-table row.

The trigger request (Section 3) is identical. The response comes back on that same call, with the same envelope body used in async callbacks (Section 4): `requestId`, `flow`, `status`, `runId`, `generatedAt`, `parameters`, `errorCode`, `error`, `rows`. The HTTP status code reflects which path the flow took:

| HTTP status | When |
|---|---|
| 200 | Succeeded. |
| 401 | Invalid/missing `triggerKey`. |
| 500 | Procedure failed. |

The app reads the response body the same way it reads an async callback body (Section 4's field table) — there's just one call and one response, with no request-table or duplicate-detection logic needed.

**What to build:** no ingestion endpoint work needed for sync flows — the trigger action (Section 2) reads the result off its own HTTP response instead of waiting on a callback. Whether that result gets persisted anywhere is a per-flow decision (Section 6), not something this contract requires.

## 6. Adding a new flow

No new endpoint or route is needed — the existing `/:flow` route handles async flows, and sync flows don't need one at all. To add a flow, the user will give the app builder:

- **Flow name** — exact string, lowercase-hyphen, e.g. `hourly-picks`. This is the value the app will match on `flow` in callbacks (async) or send in the trigger request (both modes).
- **Response mode** — `sync` or `async`. Determines whether the app reads the result off the trigger's own response (Section 5) or waits for a callback (Section 4).
- **Trigger URL** — required for `sync` (there's no other way to get a result). For `async`, blank means the flow is scheduled/recurrence and the app never triggers it. If provided, it is a secret (it contains a `sig=` value): store it server-side, never in client code.
- **Parameters** — names and types the procedure takes, or "none". The app sends these as `ProcedureParameters` in the trigger request (Section 3).
- **Row fields** — exact field names and types that will appear in `rows` for this flow, ideally with one example row. The app uses these to validate `succeeded` results (Section 4 or 5) and to render the data.

> Note for the person filling this out: if this flow is `async` and scheduled/recurrence (not app-triggered), the flow name above must already be hardcoded on the Power Automate side, in the flow's callback step. The app cannot detect a mismatch — a wrong or missing hardcoded name will just fail as "unknown flow" with no other symptom.

## 7. Security

- `TriggerKey`, `CallbackKey`, and all trigger URLs are secrets: store server-side only, never in client code or logs.
- Compare `x-ingest-secret` in constant time.
