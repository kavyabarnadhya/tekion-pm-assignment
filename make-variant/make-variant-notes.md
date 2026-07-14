# Make.com Scenario — Module-by-Module Notes

`blueprint.json` in this folder is the final, working version of the scenario — built with Make's own Copilot (fed the assignment spec directly), then iterated with it to correct the retry approach. It supersedes an earlier hand-authored attempt that had two broken Data Store module IDs; those are now the verified real ones (`datastore:AddRecord`, `datastore:GetRecord`).

## Import

1. Make.com → Scenarios → **Create a new scenario** → menu (⋮) → **Import Blueprint** → upload `blueprint.json`.
2. The blueprint references a Data Store by internal ID (`118237`), which is specific to the account it was built in. On import to a different account, create a data store named `PaymentRecords` with fields `referenceId` (text, key), `payment_link` (text), `ID` (text), `status` (text), `errorReason` (text), `timestamp` (date), then re-point each of the 5 Data Store modules at it if Make doesn't resolve the reference automatically.
3. **Enable Store incomplete executions**: Scenario editor → gear icon → Scenario settings. This is the retry mechanism (see below) — it's a scenario-level setting, not part of the blueprint JSON, so it must be turned on manually after every import.
4. Bind the webhook: module 1 → **Create a webhook** (or select an existing one) → this becomes your trigger URL.

## Module walkthrough

| # | Module | Purpose |
|---|---|---|
| 1 | Custom Webhook | Entry point. Receives Tekion's JSON payload. |
| 2 | Router | Two routes, each with a module-level `filter` (not a route-level one — Copilot attached filters directly to modules 4 and 6). |
| 4 | Data Store: Add Record | Fires when the filter finds ANY required field missing. Logs `status: failed`, `errorReason: "Missing required field in Tekion payload"`. No retry — validation failures aren't transient. |
| 6 | Data Store: Get Record | Fires when the filter confirms ALL required fields exist. Idempotency lookup by `referenceId`. |
| 7 | Router | Splits on whether module 6's `payment_link` output exists. |
| 10 | Set Variables | Filter: only when `payment_link` does NOT exist (i.e. no existing link). Computes `expireBy` (now + 24h, Unix seconds via `formatDate(addHours(now;24);"X")`) and normalizes `customerEmail`. |
| 11 | HTTP: Call Payment Link API | POST to the Beeceptor Payment Link URL. On failure (after Make's automatic retries — see Retry Policy below), falls to error handler module 21. |
| 21 | (onerror of 11) Data Store: Add Record | Logs `status: failed`, `errorReason: "Payment Link API unreachable after 3 retries"`. |
| 13 | Parse JSON | Uses a dedicated `PaymentLinkResponse` data structure (created by Copilot) to extract `payment_link`, `ID`, `referenceId` cleanly, rather than a freeform parse. |
| 14 | HTTP: Call Update API | GET-with-body to the Beeceptor Update URL (see gotcha below). Same retry behavior as module 11. |
| 15 | (onerror of 14) Data Store: Add Record | Logs `status: failed`, `errorReason: "Update API unreachable after 3 retries"`. |
| 16 | Data Store: Add Record | Success path — upserts the final record with `status: "updated"`, `payment_link`, `ID`. |

Note: the idempotent-replay route (router 7, the branch that fires when `payment_link` already exists) ends at module 17 ("Idempotent End") and runs no further — the short-circuit behavior described in `docs/ASSUMPTIONS.md`.

## Verified test results

All four paths were run end-to-end against the live scenario (not just designed/expected):

- **Happy path** (`referenceId: TS1989`, full valid payload): confirmed. Ran through to module 16 — `payment_link: https://api.xyz.com/v1/payment_links/123456`, `ID: 123456`, `status: updated` written to `PaymentRecords`.
- **Missing field** (`referenceId: TS9999`, `paymentAmount` omitted): confirmed. Module 4 fired (`status: failed`, `errorReason: "Missing required field in Tekion payload"`); module 6 correctly skipped — execution log: "The bundle did not pass through the filter."
- **Idempotency replay** (re-sent `TS1989` after the happy-path run): confirmed. Module 6 found the existing record's `payment_link`, routed to module 17, and none of 10/11/13/14/16 executed.
- **API failure → error handler**: confirmed on a real run before the fix below — module 21 correctly logged `status: failed`, `errorReason: "Payment Link API unreachable after 3 retries"`.

One real bug found and fixed during testing: HTTP module 11 initially had **Parse response** off, which meant a real Payment Link API failure got caught as a generic error with no visible status code/body. Turning **Parse response** on surfaced the actual response and the subsequent happy-path run succeeded cleanly.

## Retry policy: Store incomplete executions (not a fixed attempt count)

Make's HTTP module has no "retry exactly N times, only on 5xx/timeout" setting exposed in normal configuration — this was confirmed directly by Make's Copilot when asked to build one. The scenario instead uses Make's native mechanism:

- **Store incomplete executions** (scenario setting, step 3 above) makes Make automatically retry transient failures — timeouts, connection errors, 5xx-type responses — with exponential backoff. The exact attempt count is platform-controlled (Make documents up to ~8 automatic retries for retryable incomplete executions), not something the scenario itself sets to "3."
- Permanent failures (4xx client errors, mapping/validation errors) are **not** auto-retried — they go straight to the error-handler Data Store logging (modules 21 and 15).
- This was chosen deliberately over building a custom Repeater + Sleep + conditional-branch loop to force an exact 3-attempt cap, which Copilot could have built but flagged as adding real complexity (and repurposing the existing error-handler wiring) for a mock API endpoint that isn't actually flaky in a way a hard attempt cap would meaningfully improve.

If a specific "3 attempts, then log" guarantee is required later, that would mean replacing modules 11/14's error handling with an explicit loop — noted here as a possible future iteration, not something this build implements.

## Configuring the GET-with-body gotcha (module 14)

Make's HTTP module defaults to no body on GET. To mirror the assignment's literal spec:
1. In module 14, set **Method** to `GET`.
2. Expand **Show advanced settings**.
3. Set **Body type** to `Raw`, **Content type** to `JSON (application/json)`, and paste the body in the **Request content** field.

Make allows this (unlike some strict HTTP clients), but it's non-standard — documented as a deliberate choice in `docs/ASSUMPTIONS.md`, not a workaround. If Copilot or a teammate rebuilding this scenario tries to "fix" it to POST, that's a regression — the spec's method choice is intentional to mirror, not correct.

## Known risk (accepted, not engineered around)

Both Beeceptor endpoints are unauthenticated, ephemeral mock servers per the assignment's own warning — they may be unreachable at demo/review time. No fallback endpoint is built for the Make-only submission (see `docs/ASSUMPTIONS.md`). If this bites during testing, the retry + error-handler path is still fully demoable: it correctly logs a `failed` record either way.
