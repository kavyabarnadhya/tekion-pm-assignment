# Tekion Payment Link Generation Workflow

**Tekion PM-2 Integrations take-home submission.** An orchestration workflow that lets dealerships generate a shareable payment link for a customer directly from their management system — built and tested end-to-end in Make.com.

- **Live landing page:** https://tekion-payment-link-workflow.vercel.app
- **Live Make scenario (view-only):** https://us2.make.com/public/shared-scenario/1LNjHnuqvpF/tekion-payment-link-generation-workflow
- **Demo video:** _link to be added_

## What's in this repo

| Path | What it is |
|---|---|
| [`docs/README.md`](docs/README.md) | Full design doc — architecture diagram, scope-coverage table, field-mapping table, setup & execution instructions, tested results |
| [`docs/ASSUMPTIONS.md`](docs/ASSUMPTIONS.md) | Every assumption and deliberate design decision made where the spec was ambiguous, including two documented "gotchas" |
| [`docs/demo-script.md`](docs/demo-script.md) | Script for the recorded demo walkthrough |
| [`make-variant/blueprint.json`](make-variant/blueprint.json) | The Make.com scenario export — importable source of truth, tested working |
| [`make-variant/make-variant-notes.md`](make-variant/make-variant-notes.md) | Module-by-module build reference and verified test results |
| [`site/`](site/) | Static landing page (deployed to Vercel above), summarizing the problem, design, and deliverables in one place |

## Quick summary

- **Trigger:** custom webhook receiving Tekion's payment request payload.
- **Validation:** required fields checked before any external call; missing fields fail fast with no retry.
- **Idempotency:** `referenceId` is the idempotency key — a repeated request for the same reference returns the existing link instead of generating a duplicate.
- **Payment Link API call:** flat Tekion fields mapped to the API's nested `customer` object; `expire_by` computed dynamically as now + 24h.
- **Update API call:** deliberately implemented as a GET-with-body to mirror the spec exactly, even though that's non-standard REST — documented, not silently "fixed."
- **Retry:** Make's native "Store incomplete executions" — automatic backoff on transient failures, straight to error-handler logging on permanent ones.
- **Tested:** all 4 paths verified live — happy path, missing-field validation, idempotency replay, and API-failure error handling. See `docs/README.md` for the exact results.

See [`docs/README.md`](docs/README.md) for the full write-up.
