# FE — Month-end finance emails

Build spec for **SD-3498**, under epic **SD-3492**. **6 templates.** What the attachments *contain* is **SD-3468**; this is the emails that carry them.

| Template | To | Carries |
|---|---|---|
| `08-payroll-checklist.html` | Shared Services | Supplier checklist `.xlsx` |
| `08-sfm-supplier-upload.html` | Shared Services | SFM 22-column `.xlsx` |
| `09-client-billing.html` | Shared Services | Client billing `.xlsx` |
| `10-engineer-invoicing.html` | Shared Services | Invoicing `.xlsx` **+ zip of invoice PDFs** |
| `10-invoice-copy.html` | Engineer | Their own auto-submitted invoice |
| `13-engineer-invoice-upload.html` | Shared Services | An engineer's **own** uploaded invoice |

---

## Shared rules

Everything in **SD-3493** applies: the 600px Arial shell, the hidden preheader, label/value rows, the three panel meanings, BE-30 CTA and sign-in behaviour, money formatting, no unsubscribe, and the four-client test pass. This document covers only what is specific to these templates.

## Four emails, one snapshot

- All four Shared Services emails fire from the **same 23:59 job**, from **one snapshot**, so the attachments **reconcile to the cent and to the hour** (SD-3468).
- Each states the **period** in its subject and heading.
- The invoicing email's body lists **entries held back at the cut-off**, so Finance knows what is not in the file and what will appear next month as hours earned in an earlier period.
- **On-demand downloads send no email.**
- A **failed run is retryable without double-sending**.

## Totals carry their currency

Currency **codes**, exact cents. A mixed-currency period shows **one figure per currency** side by side — `EUR 41,286.75 · USD 9,412.40`. **Never combined, never converted** (BE-09, BE-27).

## The engineer's invoice copy

- Sent when the system files the invoice **on their behalf** (BE-24) — they never file one manually.
- Itemised **per bench**, matching the Invoices card on their dashboard.
- **Engineer-facing rate only.** The configured blended rate, mark-up and retained margin never appear (BE-04, BE-08).

## An uploaded invoice — the engineer stays in the run

**The criterion most likely to be built wrong, and getting it wrong means somebody is not paid.**

- Fires the moment an engineer uploads their own invoice, carrying their details, the period, their message, and **the PDF attached verbatim**.
- **Uploading suppresses generation only. It does not remove the engineer from the run.** They still appear on the **payroll checklist**, in the **SFM supplier upload** and in the **engineer invoicing spreadsheet**, with their row built from the same approved work logs. Only the invoice **document** differs.
- **The email states this on its face**, in an amber panel naming the three reports they remain in — so nobody hand-excludes them on reading it.
- Their checklist row stamps **Invoice Number** and **Invoice Recvd** from **their** invoice.
- **Zero approved hours is the only reason an engineer is absent from a run** (BE-24, BE-27).

## Attachments

- Filenames `castillians-{report}-{YYYY-MM}.xlsx`; the zip is `engineer-invoices-YYYY-MM.zip`.
- **A missing attachment is a failed send**, not a partial one. An email announcing a file it does not carry is worse than no email.
- A record blocked for missing Zoho finance data is **named in the body** (SD-3468); every other row still generates.

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| Every figure in the attachments | The month-end snapshot (SD-3468) | The four files, the dashboards, the Engineer Invoices page |
| Engineer-facing rate | `configured ÷ (1 + mark-up)` (BE-08) | Engineer Invoices page, invoice PDF |
| Subscription currency | Per bench (BE-27) | Every rate and total on the platform |
| Whether an invoice was uploaded | The engineer's own upload (BE-24) | The zip, the checklist row, this email |

**Acceptance criteria**

- An email's stated totals **equal its attachment's totals**.
- The four Shared Services emails for one period **reconcile with each other**.
- **Assert the uploaded-invoice case directly:** a run containing an uploaded invoice has the **same engineer count and payable total** as the same run with a generated one.
- **The retained margin never appears**, internal or engineer-facing (BE-04, BE-08).

---

## Reference

```
../SD-3493/                        the shell and shared rules
../../internal-dashboard/SD-3468/  what the attachments contain
../../ENGINEERING-BRIEF.md         BE-22 columns, BE-24 uploads stay in the run, BE-27 PDFs
```

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** the `BE.md` files in spec folders, which are backend specs for a single story.
