# BE — Month-end report generation

Backend specification for **SD-3468**. Four reports, one snapshot, no SFM integration.

---

## Scope

| Report | Grain | Emailed | Downloadable |
|---|---|---|---|
| Supplier checklist | engineer × bench × Period Earned | month-end | any period (SD-3461) |
| SFM supplier upload | one row per supplier invoice | month-end | any period (SD-3461) |
| Client billing | one row per Virtual Bench | month-end | any period (SD-3461), per channel (SD-3462) |
| Engineer invoicing | engineer × bench | month-end + PDF zip (BE-27) | any period (SD-3464) |

Column-by-column definitions: **BE-22**. Zoho field mapping: **BE-23**. Invoice PDF: **BE-27**.

## The templates are the contract

Finance's two templates are **attached to SD-3468**: `Payroll 2026 checklist - Subscription.xlsx` and `SFM Supplier Invoices Upload.xlsx`. BE-22 transcribes them; **where a transcription and an attachment disagree, the attachment wins**.

**Where each populated column comes from** — transcribed from the annotation rows in John's sample (`SFM Supplier Invoices Upload.xlsx`, rows 3–4):

| Col | Header | Meaning | Source |
|---|---|---|---|
| D | `ENTITYCODE` | Engineer's SFM account number | **Zoho** |
| E | `NACCODE` | SFM nominal account number | **Zoho** |
| F | `TRANDATE` | Invoice date | **Invoice capture** |
| K | `INV NO` | Invoice number | **Invoice capture** |
| M | `DETAILS` | VBench code | **Invoice capture** |
| Q | `VATCODE` | VAT code | **Zoho** |
| R | `CURRENCY` | Billing currency | **Invoice capture** |
| S | `AMOUNT` | Invoice value in EUR | **Invoice capture** |
| U | `FAMOUNT` | Invoice value in foreign currency | **Invoice capture** |

"**Invoice capture**" now means **this platform**: the engineer invoice it generates, or the one the engineer uploaded in its place (BE-27, BE-28). Everything else on the row is either a **Zoho** reference code or a fixed constant.

> ⚠ **The sample's annotation rows are documentation, not data.** Rows 3 and 4 of the template hold the column meanings and their sources. The generated file carries **row 1 (headers) then data from row 2** — never the note rows.

The payroll checklist sample carries a **two-row header** and Finance's working notes in cell comments — reproduce the header exactly; the comments are context, not content.

**Acceptance criteria**
- A test asserts the generated header row is **byte-identical** to the sample's. A renamed or reordered header silently breaks the SFM import, and with no integration nothing else will catch it.
- The generated file contains **headers then data**. The sample's annotation rows are never emitted.
- Zoho supplies only the **static reference codes** (`ENTITYCODE`, `NACCODE`, `VATCODE`, payment type, IBAN, BIC, company name). Every invoice-derived value comes from the platform's own invoice record.

## SFM — file handoff, not an integration

```
platform → .xlsx in SFM's 22-column format → a person uploads it to SFM
```

**Acceptance criteria**
- **No read from SFM and no write to it.** No credentials, no endpoint, no batch posting, nothing to make idempotent against SFM.
- Every supplier and client reference code is read from **Zoho at generation time** (BE-23) and never cached in the portal — a correction in Zoho is picked up by the next generation, with no deploy.
- Row 1 is the header row **exactly as supplied**; the 22 columns are never reordered, renamed or omitted, even where the platform leaves a cell blank.
- Constants (`SYSTEMBOOK`, `BOOK`, `BATCH`, `DUEDATE`, `PAYTYPE`, `PAYEE`, `ANALYSIS`, …) are **named config values**, not literals in the generator. `BATCH` is expected to change per run.
- Exactly one of `AMOUNT` (EUR) or `FAMOUNT` (non-EUR) is set per row.
- The generated file is validated against a **real SFM import in a non-production company** before the first live run. The format is the whole contract.

## One snapshot

```
snapshot(period) = approved work logs at 23:59 on the period's last day, after BE-24 auto-submission
```

**Acceptance criteria**
- All four reports for a period derive from that one snapshot and **reconcile to the hour and the cent** with each other, with the Engineer Invoices page (SD-3458) and with the Channel page billing (SD-3462).
- **One generator per report**, called by both the scheduled job and the on-demand endpoint — never two code paths.
- Re-running a **closed** period returns a **byte-identical** file and sends nothing (§G5).
- Only **approved** hours are counted; pending-approval hours are excluded, as everywhere else.
- Supplier amounts use `configured ÷ (1 + mark-up)` (BE-08); client amounts use the configured blended rate, overage at 1.25× for the first 3 months (BE-06). The two rates are never mixed.
- No currency is ever converted; mixed-currency periods total **per currency**.

## Carry-over columns

The supplier checklist and the engineer invoicing export carry **Period Earned**, **Period Billed** and derived **Carried Over** (BE-22, SD-3467).

**Acceptance criteria**
- Rows are keyed **engineer × bench × Period Earned**; carried hours are never merged into the normal row.
- The SFM upload's 22 columns are untouched — an SFM row follows the invoice, issued in the Period Billed.
- A late approval never rewrites a closed period's files.

## Blocked rows

**Acceptance criteria**
- A record missing a **mandatory** Zoho finance field **blocks that row** and is **named** — in the month-end email body and in the on-demand response.
- **Every other row still generates.** One incomplete supplier record must not withhold the period.
- A blocked row is never emitted as a blank cell.
- The engineer's **address** is the only field where blank is valid (BE-27).

## Scheduling

**Acceptance criteria**
- Fires at **23:59 CET on the last calendar day** of the period, after engineer invoices are auto-submitted (BE-24).
- Four emails to `sharedservices@castillians.com`, one per report; engineer invoicing also carries the **zip of per-engineer PDFs**.
- The body lists entries **held back at the cut-off**, so Finance knows what is not in the file.
- A failed run is **retryable without double-sending**, regenerating from the same snapshot rather than live data.
- An engineer who **uploaded their own invoice** consumes no invoice number and appears in the zip with their own PDF (SD-3458).
- On-demand downloads send **no** email.

---

## Integration & sync

The reports author nothing; every figure is read from the surface that owns it.

| Value | Source | Also appears on |
|---|---|---|
| Hours | Approved work logs against each bench's own period (SD-3459) | Engineer Work Log, Internal Work Logs, Manager bench page |
| Capacity, overage | Bench subscription and overage state (SD-3465, BE-05) | Virtual Benches tab, Channel page |
| Supplier rate | `configured ÷ (1 + mark-up)` (BE-08) | Engineer Invoices page, invoice PDF |
| Client rate | Configured blended rate | Channel page billing |
| Period Earned / Billed | Derived at approval (SD-3467) | Entry label on Engineer and Internal |
| Invoice number | Database-owned sequence (§G5) | Invoice PDF, supplier checklist row |
| Finance reference codes | **Zoho**, maintained by our team (SD-3463) | Every report, the invoice PDF |

**Acceptance criteria**
- The on-demand file for a period is **identical** to the month-end attachment for that period.
- An entry approved after the cut-off lands in the **next** period's files as a carry-over.
- Five surfaces agree for a closed period: the four reports, the Engineer Invoices page and the Channel page billing.
- No finance field is cached in the portal.
