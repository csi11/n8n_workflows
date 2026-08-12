# Stock Analytics — Trade Log Automation

## Purpose
Automatically logs every stock trade (from two brokers, KH and IB) into a single Google Sheet, using a common data model, with
duplicate protection and Telegram notifications.

## Data sources
- **KH** — daily confirmation email, subject contains
  `"tranzakciós összefoglaló"`. Trade data is in the HTML/plain-text email
  body, identified by **ISIN**.
- **IB** — daily "Trade Summary by Symbol" PDF
  attachment, subject contains `"Személyre szabott tevékenységkimutatás"`.
  Sent every day, even with no trades (near-empty PDF on those days).
  Identified by **ticker/symbol** (no ISIN available in this report type).

## Common output schema
Every transaction, regardless of source, is normalized to:

| Field    | Description                                   |
|----------|------------------------------------------------|
| date     | Trade date (KH) / statement generation date (IB) |
| ticker   | Stock ticker (ISIN converted to ticker for KH via lookup table) |
| price    | Average execution price                        |
| currency | Trade currency                                 |
| quantity | Signed quantity — negative = sell, positive = buy |
| dedupKey | `date+ticker+price+currency+quantity`, used to prevent duplicate rows |

## Workflow steps

1. **Gmail Trigger** — polls daily, filters by subject line for both
   KH and IB confirmation emails in one query. Attachments downloaded
   (Simplify off) so the IB PDF is available as binary data.

2. **Switch (IB or KH)** — routes based on subject line content into two
   branches.

3. **IB branch**
   - `Extract from File (PDF)` — pulls raw text from the attached PDF.
   - `Get Currencies (Data Table)` — fetches known currency codes
     (EUR, USD, HUF, etc.) from the `CurrencyPrefix` table, used to
     detect currency sections in the PDF text.
   - `Extract data IB` (Code node) — parses the "Trade Summary by Symbol"
     table: strips fused section labels ("Stocks", currency codes) from
     symbol names, ignores Totals rows, stops before the Forex section,
     and picks whichever side (Buy/Sell) is non-zero per symbol. Returns
     an empty array on no-trade days.

4. **KH branch**
   - `Extract data KH` (Code node) — regex-parses the plain-text email
     body into transaction pairs (trade line + settlement line),
     extracting date, ISIN, price, currency, and signed quantity
     (Eladás = sell, Vétel = buy).
   - `Get row(s) - ticker` (Data Table lookup) — looks up the ticker for
     each ISIN in the `ISIN_Ticker` table. Feeds the "found" path.
   - `If row does not exist` (Data Table check) — catches ISINs with no
     match in the table. Feeds the "not found" path, where the ISIN
     itself is used as a fallback "ticker" value (rather than leaving
     the row blank), and triggers a Telegram alert asking to add the
     missing ISIN to the lookup table.
   - Both paths are normalized into the common schema and reconnected
     via a **Merge (Append)** node.

5. **Edit Fields — message id** — both branches converge here; this step
   builds the `dedupKey` field used for duplicate detection.

6. **Append or update row in sheet** (Google Sheets) — upserts into the
   sheet using `dedupKey` as the matching column. If a transaction with
   the same key already exists (e.g. an email was accidentally forwarded
   twice), the row is silently overwritten with identical data rather
   than duplicated — no separate duplicate-detection branch or alert.

7. **Telegram notification** — after a successful write, sends a message
   confirming the ticker, quantity, price, and currency of the logged
   trade.

## Known limitations / design decisions
- IB's report type has no ISIN, only ticker — the reverse (ISIN→ticker)
  lookup table must be maintained manually as new instruments are traded.
- IB's "Trade Summary by Symbol" report aggregates same-day, same-symbol
  trades into a single line — per-execution granularity (timestamps,
  individual fills) is not preserved.
- Deduplication is content-based (`date+ticker+price+currency+quantity`),
  not based on Gmail message ID, since forwarded duplicate emails get new
  message IDs. This carries a small theoretical risk of two genuinely
  separate trades with identical values on the same day being treated as
  one — accepted as low-probability for personal trading volume.
- Account ID/name is not tracked — neither broker's source data exposes
  it in a way that could be reliably attributed per transaction.
