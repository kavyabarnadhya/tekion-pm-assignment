# Assumptions & Design Decisions

## Assumptions

- **Trigger mechanism**: the assignment doesn't specify how Tekion delivers the payment request, so this workflow assumes a webhook POST with a JSON body matching the sample payload.
- **Mock endpoint reliability**: both Beeceptor URLs are unauthenticated and ephemeral, per the assignment's own warning. No fallback endpoint was built for this reason; that risk is accepted to stay within scope. If Beeceptor is down during review, the error-handler behavior is still fully demoable since it logs a `failed` record correctly regardless of the underlying cause.
- **Payment record storage**: no real Tekion database exists here, so "the payment record" is represented by a Make Data Store (`PaymentRecords`), keyed by `referenceId`.
- **Retry policy**: Make has no module-level "retry N times" setting. The workflow instead relies on the scenario-level **Store incomplete executions** setting (Scenario settings; not represented in the blueprint JSON itself). With it on, Make automatically retries transient failures (timeouts, 5xx/connection errors) with exponential backoff. The attempt count is platform-controlled, not something the scenario fixes at a specific number. Permanent failures (4xx, validation errors) skip retry entirely and fall through to error-handler logging. Missing-field validation failures also skip retry, since they aren't transient and retrying changes nothing. A custom Repeater+Sleep loop could force an exact attempt count, but that's added complexity for a mock API that isn't flaky in a way retries would actually help with.
- **Static tokens**: the `token` headers (`122334`, `12345`) are hardcoded in the HTTP modules since they're the assignment's own mock credentials, not real secrets. A production build would move these to a Make connection or secret store.

## Design decisions worth calling out

### GET-with-body on the Update API
The assignment's "Update the Contact API" curl specifies `--request GET` while also sending a JSON body via `--data`. A body-bearing GET request goes against RESTful convention and some HTTP clients or proxies drop it outright.

The workflow supports this exactly as written, mirroring the spec rather than substituting POST/PUT/PATCH. Make's HTTP module needs a request body explicitly enabled on a GET call since that's not the default — covered in `make-variant/make-variant-notes.md`, and noted here too so it doesn't read as an oversight. Standard practice would use POST, PUT, or PATCH for an update; this build sticks to what the spec literally says instead.

### Idempotency by referenceId
`referenceId` (e.g. `TS1989`) works as the idempotency key for the whole workflow. Before calling the Payment Link API, the scenario checks the Data Store for an existing record under that reference. If one already has a `payment_link`, the workflow returns it and skips straight past the API calls.

Why this matters: a retried or duplicate Tekion request for the same reference shouldn't spin up a second live payment link. Beyond wasted API calls, that's a real correctness problem — a customer could end up paying twice, or get confused seeing two valid links for one invoice.

### Flat-to-nested field mapping
Tekion sends a flat payload (`customerName`, `customerPhone`, `customerEmail`); the Payment Link API wants a nested `customer` object (`name`, `contact`, `email`). The mapping step handles that translation and treats `customerEmail` as optional — if it's absent or blank, the `email` key is left out of the nested object entirely rather than sent as `null` or an empty string, and `notify.email` is set to `false` to match.

### expire_by computed dynamically
The Payment Link API wants `expire_by` as a Unix timestamp. It's not part of Tekion's input payload, and the sample value in the assignment doc (`1691097057`) is already in the past relative to today. The workflow computes it at request time as now plus 24 hours instead of hardcoding anything.
