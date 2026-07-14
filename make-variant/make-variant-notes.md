# Make.com Scenario — Module-by-Module Notes

`blueprint.json` is the final, working version of the scenario, built with Make's own Copilot (fed the assignment spec directly) and then iterated on to fix the retry approach. It replaces an earlier hand-authored attempt that had two broken Data Store module IDs; those are now the correct ones (`datastore:AddRecord`, `datastore:GetRecord`).

## Import

1. Make.com → Scenarios → **Create a new scenario** → menu (⋮) → **Import Blueprint** → upload `blueprint.json`.
2. The blueprint references a Data Store by internal ID (`118237`), specific to the account it was built in. On import to a different account, create a data store named `PaymentRecords` with fields `referenceId` (text, key), `payment_link` (text), `ID` (text), `status` (text), `errorReason` (text), `timestamp` (date), then re-point each of the 5 Data Store modules at it if Make doesn't resolve the reference automatically.
3. **Enable Store incomplete executions**: Scenario editor → gear icon → Scenario settings. This is the retry mechanism described below. It's a scenario-level setting, not part of the blueprint JSON, so it has to be turned on manually after every import.
4. Bind the webhook: module 1 → **Create a webhook** (or select an existing one). This becomes your trigger URL.

## Module walkthrough

| # | Module | Purpose |
|---|---|---|
| 1 | Custom Webhook | Entry point. Receives Tekion's JSON payload. |
| 2 | Router | Two routes, each with a module-level `filter` rather than a route-level one — Copilot attached filters directly to modules 4 and 6. |
| 4 | Data Store: Add Record | Fires when the filter finds any required field missing. Logs `status: failed`, `errorReason: "Missing required field in Tekion payload"`. No retry, since validation failures aren't transient. |
| 6 | Data Store: Get Record | Fires when the filter confirms all required fields exist. Idempotency lookup by `referenceId`. |
| 7 | Router | Splits on whether module 6's `payment_link` output exists. |
| 10 | Set Variables | Filter: only when `payment_link` doesn't exist. Computes `expireBy` (now + 24h, Unix seconds via `formatDate(addHours(now;24);"X")`) and normalizes `customerEmail`. |
| 11 | HTTP: Call Payment Link API | POST to the Beeceptor Payment Link URL. On failure, after Make's automatic retries (see below), falls to error handler module 21. |
| 21 | (onerror of 11) Data Store: Add Record | Logs `status: failed`, `errorReason: "Payment Link API unreachable after 3 retries"`. |
| 13 | Parse JSON | Uses a dedicated `PaymentLinkResponse` data structure, created by Copilot, to extract `payment_link`, `ID`, `referenceId` cleanly instead of a freeform parse. |
| 14 | HTTP: Call Update API | GET-with-body to the Beeceptor Update URL (see gotcha below). Same retry behavior as module 11. |
| 15 | (onerror of 14) Data Store: Add Record | Logs `status: failed`, `errorReason: "Update API unreachable after 3 retries"`. |
| 16 | Data Store: Add Record | Success path. Upserts the final record with `status: "updated"`, `payment_link`, `ID`. |

The idempotent-replay route (router 7, the branch that fires when `payment_link` already exists) ends at module 17 ("Idempotent End") and runs nothing further — the short-circuit behavior described in `docs/ASSUMPTIONS.md`.

## Verified test results

Full details and exact values are in `docs/README.md`'s "Tested results" section. Short version: all four paths (happy path, missing-field validation, idempotency replay, API-failure error handling) were run against the live scenario and passed.

One real bug came up during testing: HTTP module 11 initially had **Parse response** off, so a real Payment Link API failure got caught as a generic error with no visible status code or body. Turning it on surfaced the actual response, and the following happy-path run succeeded cleanly.

## Retry policy: Store incomplete executions, not a fixed attempt count

Make's HTTP module has no "retry exactly N times, only on 5xx/timeout" setting in normal configuration — confirmed directly by Make's Copilot when asked to build one. The scenario instead uses Make's native mechanism:

- **Store incomplete executions** (scenario setting, step 3 above) makes Make automatically retry transient failures (timeouts, connection errors, 5xx-type responses) with exponential backoff. The attempt count is platform-controlled — Make documents up to roughly 8 automatic retries for retryable incomplete executions — rather than something the scenario pins to a specific number.
- Permanent failures (4xx client errors, mapping/validation errors) aren't auto-retried. They go straight to the error-handler Data Store logging on modules 21 and 15.
- A custom Repeater + Sleep + conditional-branch loop could force an exact 3-attempt cap. Copilot flagged that as real added complexity, and repurposing the existing error-handler wiring, for a mock API endpoint that isn't flaky in a way a hard attempt cap would meaningfully improve.

If a strict "3 attempts, then log" guarantee turns out to matter later, that means swapping modules 11/14's error handling for an explicit loop. Worth flagging as a possible next iteration; this build doesn't implement it.

## Configuring the GET-with-body gotcha (module 14)

Make's HTTP module defaults to no body on GET. To mirror the assignment's literal spec:
1. In module 14, set **Method** to `GET`.
2. Expand **Show advanced settings**.
3. Set **Body type** to `Raw`, **Content type** to `JSON (application/json)`, and paste the body in the **Request content** field.

Make allows this, unlike some stricter HTTP clients, but it's non-standard. `docs/ASSUMPTIONS.md` covers why this is intentional rather than an oversight. If someone rebuilding this scenario later tries to "fix" it to a POST, that's a regression, not a correction — the spec's method choice is the point.

## Known risk, accepted rather than engineered around

Both Beeceptor endpoints are unauthenticated, ephemeral mock servers per the assignment's own warning, and could be unreachable at review time. No fallback endpoint was built for the Make-only submission (see `docs/ASSUMPTIONS.md`). If Beeceptor is down during testing, the error-handler path still logs a `failed` record correctly either way.
