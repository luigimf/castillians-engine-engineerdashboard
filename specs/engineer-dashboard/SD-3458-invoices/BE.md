# BE — Invoices (`/invoices`)

Backend specification for **SD-3458**. Reading an engineer's earnings per billable period, and
the automatic invoice submitted on their behalf.

Depends on the period engine from **SD-3453** and the approved-hours store from **SD-3456**.

---

## 1. Integration points

| System | Field | Use |
|---|---|---|
| Internal Dashboard → Manage Subscription | `Configured Blended Hourly Rate` | Basis for the engineer rate |
| Internal Dashboard → Mark-Ups & Discounts | `Percentage Mark-up` | Divisor for the engineer rate |
| SFM | Supplier record → currency | The engineer's single currency |

---

## 2. Endpoint

### `GET /api/engineer/invoices?period={YYYY-MM}`

Omit `period` to get the **most recent closed period**, or the current one if none has closed.

```json
{
  "periodKey": "2026-08",
  "label": "August 2026",
  "isOngoing": true,
  "currency": "EUR",
  "rows": [
    { "benchName": "Core Platform",  "clientName": "Northmill Bank",      "hours": 25, "ratePerHour": 59.38, "earnings": 1484.38 },
    { "benchName": "Insurance Web",  "clientName": "Northmill Insurance", "hours": 37, "ratePerHour": 55.00, "earnings": 2035.00 },
    { "benchName": "Data Squad",     "clientName": "Vertex Health",       "hours": 37, "ratePerHour": 57.50, "earnings": 2127.50 }
  ],
  "total": 5646.88,
  "invoiceNumber": null,
  "earliestPeriod": "2026-05",
  "latestPeriod": "2026-08"
}
```

**Rules**

- One row per Virtual Bench the engineer logged **approved** hours against in that period.
  Pending and declined hours are excluded entirely.
- `ratePerHour` is the **engineer-facing** rate: `configuredBlendedRate ÷ (1 + markup)`. The
  configured rate must never appear in this response.
- `earnings = hours × ratePerHour`, and `total` is the sum of the rows.
- `isOngoing` is true while the period has not closed. `invoiceNumber` is `null` until it has.
- `earliestPeriod` / `latestPeriod` bound the client's navigator — it must not have to probe.
- **Single currency.** The engineer is a supplier with one currency on their SFM record; every
  row and the total use it, whatever the clients' own currencies are. Never mix, never total
  across currencies.
- Money is exact to the cent, never rounded to whole units. Currency **codes**, not symbols.

---

## 3. Automatic invoice submission

On the **last day of the month** the portal generates each engineer's invoice from their approved
work logs and submits it **on their behalf**. The engineer never files one manually.

**Acceptance criteria**

- One invoice per engineer covering all their benches, itemised per bench — the same breakdown
  this endpoint returns.
- Runs in the same job as the month-end finance export, so the two can never disagree.
- Stamps `invoiceNumber` and flips `isOngoing` to false for that period.
- Writes Invoice Number and Invoice Recvd onto the payroll checklist row.
- Emails the engineer a copy of what was submitted.
- Entries still awaiting approval at the cut-off are **excluded** and carry into the next
  period's invoice.

Hour verification needs no extra work here: the approval flow and the append-only edit history
already establish it.

---

## 4. Test cases worth writing first

1. Engineer with hours on three benches in one period → three rows, one currency, total is their sum.
2. Engineer with 8h awaiting approval → those hours absent from rows and total.
3. Declined hours → absent from rows and total.
4. Open period → `isOngoing` true, `invoiceNumber` null.
5. Closed period → `isOngoing` false, `invoiceNumber` populated.
6. Request with no `period` → returns the most recent **closed** period.
7. Request with no `period` and nothing closed yet → returns the current period, `isOngoing` true.
8. Period with no approved hours → empty `rows`, `total` 0, still a valid response.
9. Engineer paid in EUR working a USD client's bench → row and total both in EUR.
10. Grep the response for `configuredBlendedRate` → absent.
11. Configured 95.00 at a 60% mark-up → `ratePerHour` 59.38. Change the mark-up to 50% → 63.33,
    no deploy.
12. Month-end job runs → invoice submitted, number stamped, engineer emailed, checklist row updated.


---

## Invoice PDF download

### `GET /api/engineer/invoices/{period}/pdf`

Returns the **stored** invoice PDF for that period as `application/pdf`. Not re-generated on request — the engineer must receive the identical document that was submitted on their behalf, same invoice number and figures.

**Acceptance criteria**
- `404` when the period is still **open** — no invoice exists until 23:59 on its last day.
- `404` when the period had **no billable hours**.
- The client hides the download button in both cases rather than relying on the error.
- Where the engineer **uploaded their own** invoice, this endpoint returns **their** PDF, not a generated one.

---

## Invoice upload

### `POST /api/engineer/invoices/{period}/upload`

Multipart: the PDF plus an optional `message` string.

**Acceptance criteria**
- PDF only; reject other content types. Enforce the 10 MB ceiling server-side.
- Rejected when the period is already **closed and processed** — an invoice cannot be replaced after payment has run.
- On success the engineer is **flagged as self-invoicing for that period**, and:
  - they are **excluded from the month-end auto-submission run** — only their uploaded invoice is processed;
  - their PDF replaces the generated one in the month-end zip (BE-27), so the zip stays the complete set;
  - a notification fires to `sharedservices@castillians.com` with the PDF attached.
- Re-uploading within an open period **replaces** the previous file rather than adding a second.

### Invoice PDF template

Specified in full as **BE-27** in `ENGINEERING-BRIEF.md`. In short: issued **from the engineer** (ten Zoho Finance fields) **to Castille Resources Ltd.**, one line item per Virtual Bench, totalled in the engineer's single currency.

---

## Test cases worth writing first

1. Download for a closed period with hours → returns the stored PDF, same invoice number as the submitted one.
2. Download for the **current open** period → `404`; button absent in the UI.
3. Download for a closed period with **zero** hours → `404`; button absent.
4. Upload a PDF, then run the month-end job → that engineer is **skipped**, and only the uploaded invoice is processed.
5. Upload a PDF, then download → returns **the uploaded file**, not a generated one.
6. Upload twice in one period → the second replaces the first; exactly one invoice on record.
7. Upload a non-PDF → rejected server-side.
8. Upload for a period already processed → rejected.
9. An engineer on three benches → one invoice, three line items, single currency total.
10. Month-end zip → contains exactly one PDF per engineer with billable hours, uploaded ones included, none for engineers with zero hours.


---

## Blank template endpoint

Calls the **BE-27 renderer with the data omitted** — same layout, header block, line-item table and To block, with empty rows. It is **not** a separate stored PDF: a hand-maintained file would drift from the generated invoices the first time either changed.

**Acceptance criteria**
- No personal or period data merged in; identical for every engineer.
- Addressed to **Castille Resources Ltd.**, matching the generated invoices.
- **The auto-generated download is this same template populated** — the engineer's Zoho header details, one line item per Virtual Bench for the period, and the totals, reconciling to the cent with the period rows on the Invoices page. An engineer who has seen the blank template must recognise their invoice as the same document filled in.
- A layout change is made **once, in the renderer**, and all three surfaces follow.

---

## Currency — follows the client engagement

**Corrected.** An engineer's rate and invoice are in **the currency we bill that client in** — bill a client in GBP and the engineer's rate is GBP, so they invoice us in GBP. Currency is **not** a fixed value on the supplier record.

**Acceptance criteria**
- Each invoice line resolves its currency from **that bench's client**.
- An engineer across clients billed in different currencies produces a total **per currency** — never combined, never converted.
- The Invoices page shows one total row per currency present in the period.


---

## Invoice PDF — one renderer, three surfaces

The same renderer serves the download endpoint, the per-engineer PDFs zipped into the month-end email, and the blank template. Never build a second. Full template spec is **BE-27** in `ENGINEERING-BRIEF.md`.

### Header

| Field | Rule |
|---|---|
| Invoice date | Period end date |
| Invoice number | `INV00001` upward — single global sequence |
| Name | Engineer's full name |
| Address | From Zoho; **blank when null** |

### Line items

One row per Virtual Bench:

| Column | Value |
|---|---|
| Description | `{Bench Name} - {Client Name}` |
| Hours | Hours logged on that bench in the period |
| Rate | That bench's engineer-facing rate |
| Amount | Hours × Rate |

### Multi-currency

- One **table per currency**, each with its own subtotal, **within the same PDF**.
- Tables ordered alphabetically by currency code; a single-bench currency still gets its own table.
- **One invoice number per PDF**, whatever the table count.
- Never combined, never converted.

### Numbering

```
INV00001, INV00002, … — zero-padded to 5 digits
```

**Acceptance criteria**
- A **single global sequence** across all engineers and periods — not per engineer, not per period.
- Allocated from a **database-owned sequence**. Never derive it by counting existing invoices: two concurrent month-end runs would collide on the same number.
- **Numbers are never reused.**
- An **uploaded invoice does not consume a number** — the sequence skips that engineer entirely, so there are no gaps.
- The number matches the payroll checklist row for that engineer.

### Test cases

1. Engineer on three EUR benches → one table, three rows, one subtotal, one invoice number.
2. Engineer on a EUR bench and a GBP bench → **two tables**, two subtotals, **one** invoice number.
3. Description renders exactly `Core Platform - Northmill Bank`.
4. PDF total reconciles with the Invoices page for that period, to the cent.
5. Two month-end runs fired concurrently → no duplicate invoice numbers.
6. Engineer uploads their own → no number allocated; the next generated invoice takes the next in sequence.
7. Zoho address null → the address block renders empty, not "N/A".
8. Zoho IBAN missing → PDF is **blocked** and reported in the email body.
9. Blank template → same layout, empty rows, no engineer or period data.


---

## Interaction patterns

Platform-wide rules for pagination contracts, empty/error states, validation order, optimistic updates, concurrency and money are specified once in **§G of `ENGINEERING-BRIEF.md`**. Read it before implementing — it covers the FE/BE interaction cases this file does not repeat.

### For this story

- No pagination — the period endpoint returns one period, plus the bounds of the available range.
- The download endpoint returns the **stored** PDF, not a re-render: same invoice number, same figures.
- It returns **404 when no invoice exists** for that period (open, or no billable hours) — the client hides the button rather than offering a download that fails.
- Invoice numbers come from a **database-owned sequence** (§G5); never derived by counting rows.
- Upload enforces the **PDF type and 10MB ceiling server-side**, whatever the client accepts.
