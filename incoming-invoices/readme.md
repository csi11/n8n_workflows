# Incoming Invoices — Workflow Documentation

## Goal

Automatically capture invoices arriving by email — whether as a direct PDF
attachment or as a link to a vendor's web portal — and save them to a
single Google Drive folder, with duplicate protection and Telegram
notifications for successes, skips, and failures.

## Trigger

- **Gmail Trigger**, polling daily at 10:30.
- No subject filter at the trigger level (`filters.q` empty) — all
  incoming mail is pulled and filtered downstream.
- "Download Attachments" is enabled, so PDF attachments arrive as binary
  data directly on the trigger item.

## 1. Subject filtering — `Code - filter emails`

- Normalizes the subject (lowercases, strips accents) and splits it into
  words.
- Keeps the email only if a word **starts with** one of:
  `szamla`, `bill`, `invoice`, `receipt`, `megrendelesed`.
- Drops the email if a word starts with `szamlakivonat` (bank statement),
  even if it also matches an include keyword — exclusion takes priority.
- Non-matching emails are dropped entirely; nothing downstream sees them.

## 2. Attachment vs. weblink split — `If - has attachment`

- Checks whether any binary attachment has `mimeType === 'application/pdf'`.
- **True** → attachment branch (section 3).
- **False** → weblink branch, routed into `Switch platforms` (section 4).

## 3. Attachment branch

- **`Code - atmnt name chk`**: loops over all attachments on the item.
  Excludes any attachment whose filename (normalized, word-boundary
  matched) starts with `receipt`, `nyugta`, `rendeles`, `aszf`, `gtc`, or
  `order`. Each surviving attachment becomes its own item, with its
  binary renamed to a consistent key (`data`).
  - If every attachment was excluded, the item is tagged
    `allExcluded: true` instead of being dropped, so it can be flagged
    downstream.
- **`If - nofile to save`**: checks `allExcluded`.
  - **True** → Telegram alert (🗑️ attachment doesn't look like an
    invoice) — email is not saved.
  - **False** → continues to the shared dedup/upload pipeline (section 6).

This same `Code - atmnt name chk` node is also the entry point that the
Billingo, Paddle, and Számlázz.hu branches feed into once they've turned
their weblink into a downloaded file — so the exclude-keyword check and
everything after it is shared across all four sources.

## 4. Platform detection — `Switch platforms`

- Reads the plain-text email body and checks for a known vendor domain:
  `paddle.com` → output 0, `billingo.hu` → output 1, `szamlazz.hu` →
  output 2, anything else → output 3 (unknown).
- **Unknown (output 3)** → Telegram alert (🚧 unrecognized platform) —
  email is skipped; the workflow does not attempt to guess. New
  platforms are added manually by extending this Switch node and adding
  a matching branch.

## 5. Weblink branches

All three follow the same shape: extract a link from the email body →
fetch/derive the real download URL → download the file → rename its
binary to the correct invoice filename → feed into the shared attachment
pipeline (`Code - atmnt name chk`). Each download HTTP Request node has
Retry On Fail enabled and routes its error output to the shared error
Telegram node (section 7).

### 5.1 Billingo
- `Code Billingo`: Billingo's email link is wrapped in an SES tracking
  redirect. The node decodes the wrapped URL, extracts the document
  token, and builds the direct download link:
  `https://app.billingo.hu/document-access/{token}/download`.
- `HTTP Request`: downloads the PDF directly (Billingo sets a proper
  `Content-Disposition` filename, so no manual rename step is needed).

### 5.2 Számlázz.hu
- `Code Szamlazz`: extracts the email's `fiok` link and its path token
  (used later as `partguid`).
- `HTTP Request1`: fetches that intermediate page as text.
- `Code Szamlazz2`: parses the page HTML for `szfej_id` (from the
  "Download PDF" button's href) and the invoice number (from the page's
  invoice-number label), then builds the final download URL:
  `.../szamla/?action=szamlapdf&szfej_id={id}&partguid={token}&content_disp_type=attachment`.
- `HTTP Request2`: downloads the PDF.
- `Code Szamlazz3`: renames the binary to `{invoiceNumber}.pdf`, since
  this endpoint doesn't return a usable filename on its own.

### 5.3 Paddle
- `Code Paddle`: extracts the `my.paddle.com/invoice/...` or
  `/receipt/...` link from the email text.
- `HTTP Request3`: sends that URL to the ApiFlash screenshot API
  (`api.apiflash.com/v1/urltoimage`, full-page, PNG, 3s render delay) and
  gets back a rendered image of the invoice page. Chosen because
  Paddle's invoice page is JS-rendered with no direct PDF/API endpoint
  worth building for ~1 invoice/month.
- `Code Paddle2`: renames the binary to `{receiptNumber}.png`, extracted
  from the Paddle URL.
- Output is a PNG, not a PDF — this is intentional; the shared pipeline
  downstream doesn't check file type, only filename.

## 6. Shared dedup + upload pipeline

- **`Check existing filename`** (Google Sheets "Get Row(s)", filtered on
  the `invoice number` column matching the current binary's filename,
  `Always Output Data` on so a non-match still produces an empty item).
- **`If`**: checks whether `row_number` is present (i.e., a match was
  found).
  - **True** (duplicate) → Telegram alert (🗑️ duplicate skipped) — not
    uploaded.
  - **False** (new file) → `Code in JavaScript` re-attaches the original
    binary (lost when the Sheets node overwrote `$json`) by pulling it
    back from `Code - atmnt name chk`'s output, then continues to Upload.
- **`Upload file`** (Google Drive): uploads to a single hardcoded folder.
  File name is set explicitly via `{{ $binary.data.fileName }}` — leaving
  this field blank was found to unreliably fall back to the binary's own
  filename (produced "Untitled" uploads in testing). Retry On Fail is
  enabled; error output routes to the shared error Telegram node.
- **`Log uploaded files`** (Google Sheets "Append or Update Row"): logs
  `invoice number` (= uploaded file's Drive name) and `uploaded` (today's
  date, `yyyy-MM-dd`) into the same sheet used for the dedup check.
- **`Send a text message1`**: Telegram confirmation (📄 uploaded), naming
  the file and the original email subject.

## 7. Error handling

- **Unrecognized platform** (weblink email, no known vendor match) →
  🚧 Telegram alert, email skipped, no retry (this isn't a transient
  failure — it needs a manual workflow update to add the new platform).
- **Download/API failures** (any of the four HTTP Request nodes: Billingo,
  Számlázz.hu ×2, Paddle/ApiFlash) → Retry On Fail (n8n default settings),
  then on continued failure the error output routes to a single shared
  **`Send an error message`** Telegram node (❌ step failed, includes
  `$json.error.message` when available).
- **Drive upload failures** → same pattern: Retry On Fail, then routes to
  the same shared error Telegram node.
- **Duplicate files** → not treated as an error; separate 🗑️ alert as
  described in section 6.
- A separate, standalone **Error Workflow** is also attached at the
  workflow-settings level (`Settings → Error Workflow`), providing a
  second layer of catch-all alerting independent of the in-workflow error
  outputs described above.

## Storage

- **Google Drive**: single fixed folder (no year-based subfolder
  automation — this was attempted and dropped; see "Known limitations").
- **Google Sheets**: one sheet doubles as both the dedup index and the
  upload log, with two columns — `invoice number` (matched against
  filenames) and `uploaded` (date).

## Known limitations / possible improvements

- **No year-based folder structure.** Originally planned, but abandoned
  after the Google Drive node's Search operation returned a persistent
  `403 insufficientScopes` error (a known n8n Cloud bug), and a custom
  Google Cloud OAuth client + HTTP Request workaround also failed after
  full setup. All invoices currently land in one flat folder. Worth
  revisiting later — either once the underlying n8n bug is fixed, or via
  the Drive node's "List" operation instead of "Search".
- **Paddle output is PNG, not PDF.** ApiFlash is a screenshot API only;
  there's no PDF-rendering endpoint on the free tier used here. Given the
  volume (~1/month), converting PNG → PDF in-workflow wasn't pursued.
- **File-level dedup only, keyed on filename.** Two genuinely different
  invoices that happen to share a filename (unlikely but possible across
  vendors) would collide. No content-based dedup (e.g., hash, invoice
  number + amount) is implemented.
- **Számlázz.hu assumes one invoice per token/page.** The vendor's page
  structure supports pagination/multiple invoices per account link, but
  this was confirmed as not a real-world scenario for the current usage
  pattern, so it isn't handled.
- **Unknown-platform handling requires manual workflow edits.** New
  invoicing platforms need a person to add a new Switch branch, a link
  extraction Code node, and a download step — there's no generic/fallback
  scraping path.
- **Error message detail is limited.** The shared error Telegram alert
  currently cannot reliably report which specific node failed
  (`$json.error.node.name` was found empty in testing on this n8n
  version) — only the error message itself is available.
