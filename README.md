# Tekion Payment Link Generation Workflow

**Tekion PM-2 Integrations take-home submission.** An orchestration workflow that lets dealerships generate a shareable payment link for a customer directly from their management system, built and tested end-to-end in Make.com.

- **Live landing page:** https://tekion-payment-link-workflow.vercel.app
- **Live Make scenario (view-only):** https://us2.make.com/public/shared-scenario/1LNjHnuqvpF/tekion-payment-link-generation-workflow
- **Postman collection:** [`postman-collection.json`](postman-collection.json) — test the live webhook yourself
- **Demo video:** https://drive.google.com/file/d/1r95rzdrvE8bXSrJ2S_HaZoFMTyHmxXsY/view?usp=sharing

## Try it yourself

1. Import [`postman-collection.json`](postman-collection.json) into Postman. It has three ready-made requests: happy path, missing-field validation, and idempotency replay.
2. In Make, open the scenario and click **Run Once** before sending each request (the webhook only catches one request per click).
3. Hit **Send** on a request in Postman.
4. To generate your own happy-path run instead of reusing a tested one, change `referenceId` in the request body to any value that hasn't been sent before — that's the only field the idempotency check looks at. Everything else can stay the same.
5. To see the duplicate/idempotency behavior, send the exact same request a second time with the same `referenceId`. It'll short-circuit instead of generating a new link.
6. To see the missing-field validation, use the "Missing field" request as-is, or take any request and delete a required field (e.g. `paymentAmount`).

## What's in this repo

| Path | What it is |
|---|---|
| [`docs/README.md`](docs/README.md) | Full design doc: architecture diagram, scope-coverage table, field-mapping table, setup and execution instructions, tested results |
| [`docs/ASSUMPTIONS.md`](docs/ASSUMPTIONS.md) | Assumptions and design decisions made where the spec was ambiguous, including two documented gotchas |
| [`docs/demo-script.md`](docs/demo-script.md) | Script for the recorded demo walkthrough |
| [`make-variant/blueprint.json`](make-variant/blueprint.json) | The Make.com scenario export, importable and tested working |
| [`make-variant/make-variant-notes.md`](make-variant/make-variant-notes.md) | Module-by-module build reference and verified test results |
| [`site/`](site/) | Static landing page (deployed to Vercel above) covering the problem, design, and deliverables in one place |
| [`postman-collection.json`](postman-collection.json) | Import into Postman to test the live webhook yourself: happy path, missing-field validation, idempotency replay |
| `sample-payload-*.json` | The raw request bodies used above, for testing via curl or any other HTTP client |

## Quick summary

- **Trigger:** custom webhook receiving Tekion's payment request payload.
- **Validation:** required fields checked before any external call; missing fields fail fast with no retry.
- **Idempotency:** `referenceId` is the idempotency key. A repeated request for the same reference returns the existing link instead of generating a duplicate.
- **Payment Link API call:** flat Tekion fields mapped to the API's nested `customer` object, `expire_by` computed dynamically as now plus 24 hours.
- **Update API call:** implemented as a GET-with-body to mirror the spec exactly, even though that's non-standard REST. Documented rather than quietly fixed.
- **Retry:** Make's native "Store incomplete executions" setting handles automatic backoff on transient failures, and goes straight to error-handler logging on permanent ones.
- **Tested:** all 4 paths verified live, happy path, missing-field validation, idempotency replay, and API-failure error handling. See `docs/README.md` for the exact results.

See [`docs/README.md`](docs/README.md) for the full write-up.
