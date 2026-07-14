# Assumptions & Design Decisions

## Assumptions

- **Trigger mechanism**: the assignment doesn't specify how Tekion delivers the payment request, so this workflow assumes a webhook POST with a JSON body matching the sample payload.
- **Mock endpoint reliability**: both Beeceptor URLs are unauthenticated and ephemeral, per the assignment's own warning. This build does not construct a fallback/mock endpoint for the Make-only submission — that risk is accepted rather than engineered around, to stay within scope. If it proves unreliable during review, the retry + error-handler behavior is still fully demoable (it correctly logs a `failed` record).
- **Payment record storage**: no real Tekion database exists in this context, so "the payment record" is represented by a Make Data Store (`PaymentRecords`), keyed by `referenceId`.
- **Retry policy**: Make has no module-level "retry N times" setting. The workflow instead relies on Make's scenario-level **Store incomplete executions** setting (enabled in Scenario settings, not represented in the blueprint JSON itself). With it on, Make automatically retries transient failures (timeouts, 5xx/connection errors) with exponential backoff — attempt count is platform-controlled, not a fixed number the scenario sets. Permanent failures (4xx, validation errors) are not retried and fall straight through to the error-handler Data Store logging. Validation failures (missing required fields) fail fast with no retry regardless — those aren't transient, retrying won't help. This was a deliberate choice over a hand-rolled Repeater+Sleep loop to force an exact attempt count, which would add real complexity for a mock API that isn't actually flaky in a way retries would fix.
- **Static tokens**: the `token` headers (`122334`, `12345`) are hardcoded in the HTTP modules since they're the assignment's own mock credentials, not real secrets. In a production build these would move to a Make connection/secret store.

## Documented design decisions (deliberate, not incidental)

### GET-with-body on the Update API
The assignment's "Update the Contact API" curl specifies `--request GET` while also sending a JSON body via `--data`. A body-bearing GET request is contrary to RESTful convention and is dropped or rejected by some HTTP clients and proxies.

**Decision**: the workflow is built to support this exactly as documented, to mirror the spec strictly rather than silently substituting POST/PUT/PATCH. Make's HTTP module requires explicitly enabling a request body on a GET call (not the default) — this is called out as deliberate configuration in `make-variant/make-variant-notes.md`, and flagged again here so it isn't mistaken for an oversight: standard REST practice would use POST, PUT, or PATCH for an update operation.

### Idempotency by referenceId
`referenceId` (e.g. `TS1989`) is treated as the idempotency key for the whole workflow. Before calling the Payment Link API, the scenario looks up any existing record for that `referenceId` in the Data Store. If one already has a `payment_link`, the workflow short-circuits and returns the existing link rather than generating a duplicate.

**Why**: payment link generation should not be re-triggered by a retried or duplicate Tekion request for the same reference — that would create multiple live payment links for the same invoice, which is a real-world correctness risk (customer could pay twice, or be confused by two valid links), not just an efficiency concern.

### Flat-to-nested field mapping
Tekion's input payload is flat (`customerName`, `customerPhone`, `customerEmail`), but the Payment Link API expects a nested `customer` object (`name`, `contact`, `email`). The mapping step (see `docs/README.md` field-mapping table) handles this transformation explicitly, and treats `customerEmail` as optional: if absent or blank, the `email` key is omitted from the nested object entirely (not sent as `null` or `""`), and `notify.email` is set to `false` accordingly rather than assuming every customer has an email on file.

### expire_by computed dynamically
The Payment Link API expects `expire_by` as a Unix timestamp. It isn't part of Tekion's input payload, and the sample value in the assignment doc (`1691097057`) is already in the past relative to today. The workflow computes it dynamically as now + 24 hours at request time, rather than hardcoding a value.
