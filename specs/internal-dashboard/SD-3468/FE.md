# FE — Month-end report generation

Handoff for **SD-3468**. Four generated spreadsheets and the scheduled run behind them.

**There is almost no new UI.** The download surfaces already exist:

| Report | Surface |
|---|---|
| Supplier checklist | Internal → Channel & Billing → **Download a Report** (SD-3461) |
| SFM supplier upload | Internal → Channel & Billing → **Download a Report** (SD-3461) |
| Client billing | **Download a Report**, and per channel on the Channel page (SD-3462) — carries the same Period Earned + hours-split columns as the supplier side |
| Engineer invoicing | Internal → Engagements → **Download Engineer Invoicing** (SD-3464) |

This story is what those buttons produce. Build the generators; do not build a second set of modals.

---

## No SFM integration

The platform **generates a file** in SFM's 22-column format and a team member uploads it to SFM by hand. Nothing is read from SFM, nothing is posted to it.

So **every reference code comes from Zoho** — SFM account number, nominal accounts, entity code, VAT code, payment type, IBAN, BIC, company name — maintained there by our team (SD-3463) and read at generation time.

---

## What the client must handle

- **Progress, not a frozen modal.** A generation that takes more than a moment shows progress in place; the modal stays interactive enough to cancel.
- **Failure surfaces inline with a retry**, inside the modal (§G2) — never a toast alone.
- **Blocked rows are reported, not hidden.** When a record is missing a mandatory Zoho finance field, the download still succeeds and the response names the blocked records; surface those names in the success state rather than silently shipping a short file.
- The **current, open** period carries a to-date note wherever it can be selected.
- **No email is sent for an on-demand download** — the file returns in the browser.
- Filenames `castillians-{report}-{YYYY-MM}.xlsx`. **.xlsx**, never CSV.

---

## Reference

```
BE.md                          generators, snapshot, scheduling, blocked rows
../SD-3461-channel-billing-cards/   the report picker modal
../SD-3464-engagements-benches/     the engineer invoicing download
../../ENGINEERING-BRIEF.md     BE-22 columns, BE-23 Zoho fields, BE-27 invoice PDF, §A6 SFM file handoff
../../../prototype/index.html  → Internal → Channel & Billing → Download a Report
```
