# Számlázás — Invoice Issuing Workflow

## Objective
Automates invoice creation in SLM Systems via its REST API, triggered either on a fixed monthly schedule (one recurring partner/amount) or on demand through a secured web form (variable partner/amount, entered by an authorized user).

## How it works

### Branch 1 — Scheduled monthly invoice
1. **Schedule Trigger** fires once a month.
2. Fixed partner and invoice data are read from the Data Table.
3. The API token and company VAT number are looked up from a separate token table.
4. A Code node assembles the nested JSON payload required by SLM's `invoiceToSLM` endpoint (dates calculated automatically: issue date = today, completion date = today − 1, due date = today + 5).
5. The payload is base64-encoded and sent as a `multipart/form-data` POST request.
6. The API response is checked: on success, a Telegram message and an email are sent confirming the invoice; on failure, the same channels report the API's own error message.

### Branch 2 — Web form invoice request
1. A hosted **n8n Form** collects: customer name, a customer-specific access code, net amount, and exchange rate.
2. Submitted amount/exchange rate are parsed to accept both comma and dot as decimal separator; invalid entries are rejected with a clear message shown directly on the form.
3. The customer name is looked up in the Data Table; the submitted access code is compared against the code stored for that partner. A mismatch ends the process with an on-form message — no invoice is created.
4. On a valid code, the same payload-building, base64-encoding, and API call logic as Branch 1 runs, using the form's amount/exchange rate instead of a fixed value.
5. The form itself displays the result to the submitter: success message on approval, or the API's returned error message on failure.

### Shared components
- **Token table**: stores the permanent SLM API token and the company's own VAT number, generated once via a separate manual-trigger sub-flow and reused by both branches.
- **Data Table**: holds all fixed invoice data (partner details, currency, VAT code, bank account, invoice book, etc.) common to both branches.

## Simplification decisions
- **Error handling is response-based, not code-based.** Rather than mapping SLM's numeric error codes (1–9) to custom messages, the workflow forwards the API's own `error_message` text directly to the user/notification channel. This keeps the logic simple while still giving specific, actionable feedback.
- **No separate status polling.** The workflow relies on the immediate response from `invoiceToSLM` rather than a follow-up call to `invoiceStatus`. This assumes the API's initial response reliably reflects the final outcome.
- **No explicit "partner not found" check.** If a submitted customer name doesn't match any Data Table row, the workflow does not short-circuit with a dedicated message; the passcode comparison step effectively catches most such cases, but the resulting message may be less specific than a dedicated check would give.
- **HTTP requests use automatic retry** (on transient failures) but do not use n8n's "Continue on error" branching — failures are expected to be caught by the workflow's own success/failure check on the API's returned payload rather than at the HTTP-node level.

## Possible improvements
- Add explicit `invoiceStatus` polling after invoice submission, to confirm final processing state rather than relying solely on the immediate response.
- Add a dedicated check for an empty Data Table lookup result, with a clear "partner not found" message distinct from "wrong code."
- Add a duplicate-submission guard on the web form branch (e.g., a short cooldown per partner) to prevent accidental double invoicing.
- Add a workflow-level Error Trigger as a catch-all safety net for unhandled node failures not currently covered by the response-based error handling.
- Extend to a third trigger branch for automated invoice creation from incoming email requests.
