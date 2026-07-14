# Demo Video Script (~4-5 min)

Record as a screen share of the Make.com scenario canvas plus the execution log. No need to read this word for word, just hit these beats.

## 1. Problem framing (20s)
"Dealerships today can't generate a shareable payment link for a customer directly inside their management system, and that gap is what this workflow closes. It's an orchestration layer: Tekion sends the payment request, the workflow calls a Payment Link API, and writes the result back to the payment record."

## 2. Architecture overview (30s)
Pull up the Mermaid diagram from `docs/README.md`, or the live Make canvas since they match.
Walk it left to right: "Webhook receives the request, we validate required fields, check for an existing link so we don't double-generate one, map the flat Tekion fields into the nested shape the Payment Link API expects, call it, extract the link, call the Update API to write it back, and persist the final record. Every external call has error handling, and transient failures get automatic retry with backoff."

## 3. Live demo, happy path (60s)
- Trigger the webhook with the sample payload (`curl` command from README, or Make's "Run once" plus pasted JSON).
- Show the execution log lighting up module by module.
- Open the `PaymentRecords` Data Store and show the new record: `referenceId: TS1989`, `payment_link`, `status: updated`.

## 4. A real bug hit and fixed (30s)

Worth keeping in. It's a better demo of actual debugging than an all-green happy path.

"When I first ran this end-to-end, the Payment Link API call failed and went straight to the error handler, but the logged error message wasn't useful, just a generic failure. Turned out the HTTP module didn't have Parse response enabled, so Make caught the non-2xx response without exposing the actual status code or body. I turned that on, re-ran it, and got the real response back. That's what made the happy path debuggable and confirmed working." Show module 11's config with Parse response toggled on, and the corrected execution log next to it if you saved a screenshot from the earlier failed run.

## 5. Live demo, failure path (60s)
Two quick sub-demos:
- **Validation failure**: re-send the payload with `paymentAmount` removed (or reuse the `TS9999` test payload). Show Route B firing immediately with no retry, a `status: failed` record with `errorReason: "Missing required field in Tekion payload"`, and module 6 (Get a record) skipped, the execution log literally says "The bundle did not pass through the filter."
- **API failure**: already covered in beat 4. Mention that the error-handler path (module 21) correctly logged a `failed` record on that real API failure before the fix.

## 6. Idempotency demo (20s)
Re-send the exact same `TS1989` payload. Show the Data Store lookup (module 6) finding the existing record and the scenario short-circuiting to module 17 ("Idempotent End"), so the Payment Link API isn't called a second time. Explain why: "this stops a duplicate Tekion request from generating two live payment links for the same invoice."

## 7. Close (20s)
Call out the two design decisions worth flagging:
- "The Update API is specified as a GET with a body, which isn't standard REST. I built it to match the spec exactly rather than quietly changing it to a POST, and noted that in the assumptions doc."
- "The mock endpoints are unauthenticated and can go offline. I accepted that risk for this scope instead of building a fallback server, and documented it."

Wrap: "This is all documented in the repo: README for architecture and setup, assumptions doc for every design decision, and the exported blueprint for anyone who wants to import and run it themselves."
