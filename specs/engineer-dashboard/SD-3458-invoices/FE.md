# FE — Invoices (`/invoices`)

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.** The only change to it is the two new sub-tab entries — **Work Log** and **Invoices** — per the [engineer menubar Figma](https://www.figma.com/design/A4ZWwuOmoeUr7p72TYIiLV/castillians.com?node-id=2837-3986).
>
> Everything below the sub-tab row is in scope; the header above it is not.


Angular handoff for **SD-3458**. The Invoices page on the Engineer Dashboard.

> **On conventions.** This repo was empty when the doc was written, so the code follows standard
> Angular conventions rather than ours. Re-point it once source lands.

**Version-agnostic** — NgModule and standalone registration both shown:

```ts
// NgModules (v11–14)                  // Standalone (v15+)
@NgModule({                            @Component({
  declarations: [InvoicesPageComponent], standalone: true,
  imports: [CommonModule],               imports: [CommonModule],
})                                     })
```

---

## 1. Design-system components

| Component | Used for | Status |
|---|---|---|
| `Card` | Page and row cards | ⬜ confirm exists |

**Likely missing:** the **period navigator** (chevron / label / chevron). SD-3456 needs the same
control for its calendar — worth one shared component.

---

## 2. Route

```ts
{ path: 'invoices', component: InvoicesPageComponent, canActivate: [ActivatedEngineerGuard] }
```

Reached from the engineer menubar dropdown, alongside **Work Log**.

---

## 3. Models

```ts
export interface InvoicePeriod {
  periodKey: string;            // 'YYYY-MM'
  label: string;                // 'August 2026'
  isOngoing: boolean;           // true when the period has not closed
  currency: string;             // the engineer's own currency code
  rows: InvoiceRow[];
  total: number;
  invoiceNumber: string | null; // null while the period is open
  earliestPeriod: string;       // bounds the navigator
  latestPeriod: string;
}

export interface InvoiceRow {
  benchName: string;
  clientName: string;
  hours: number;
  ratePerHour: number;          // engineer-facing rate, mark-up already removed
  earnings: number;
}
```

**One currency only.** An engineer is a supplier with a single currency on their record, so every
row and the total share it regardless of which clients' benches they worked on. There is never
more than one total to reconcile.

---

## 4. Service

```ts
getInvoicePeriod(periodKey?: string): Observable<InvoicePeriod> {
  const q = periodKey ? `?period=${periodKey}` : '';
  return this.http.get<InvoicePeriod>(`/api/engineer/invoices${q}`);
}
```

Called with no argument, it returns the **most recent closed period**, or the current one if none
has closed yet.

---

## 5. Page header

```html
<h1 class="page-title">Invoices</h1>
<p class="page-subtitle">
  What you have been paid for each billable period, itemised by Virtual Bench.
</p>
```

```scss
.page-title {
  font-family: var(--font-display); font-size: 42px; font-weight: 700; margin: 6px 0 8px;
}
.page-subtitle {
  font-family: var(--font-body); font-size: 16px; color: var(--gray-700);
  line-height: 150%; margin: 0 0 28px; max-width: 640px;
}
```

Matches the Work Log page's title treatment.

---

## 6. Period navigator

```html
<div class="period-nav">
  <button type="button" class="period-nav__btn"
          [disabled]="period.periodKey === period.earliestPeriod"
          (click)="previous()" aria-label="Previous period">
    <svg width="12" height="9" viewBox="0 0 11 7" fill="none" style="transform: rotate(90deg)">
      <path d="M1 1L5.5 5.5L10 1" stroke="#4d4d4d" stroke-width="2"
            stroke-linecap="round" stroke-linejoin="round"/>
    </svg>
  </button>

  <span class="period-nav__label">{{ period.label }}</span>

  <button type="button" class="period-nav__btn"
          [disabled]="period.periodKey === period.latestPeriod"
          (click)="next()" aria-label="Next period">
    <svg width="12" height="9" viewBox="0 0 11 7" fill="none" style="transform: rotate(-90deg)">
      <path d="M1 1L5.5 5.5L10 1" stroke="#4d4d4d" stroke-width="2"
            stroke-linecap="round" stroke-linejoin="round"/>
    </svg>
  </button>
</div>
```

```scss
.period-nav {
  display: flex; align-items: center; gap: 12px;

  &__btn {
    display: inline-flex; align-items: center; justify-content: center;
    width: 35px; height: 35px; border-radius: var(--radius-md);
    background: #fff; border: 2px solid var(--border); cursor: pointer;
    &:disabled { opacity: 0.4; cursor: not-allowed; }
  }
  &__label {
    font-family: var(--font-display); font-size: 15px; font-weight: 600;
    min-width: 130px; text-align: center;
  }
}
```

Arrows disable at `earliestPeriod` / `latestPeriod` — no navigating into empty months.

---

## 7. Breakdown rows

One row per Virtual Bench worked in the period.

```html
<div class="inv-row" *ngFor="let r of period.rows">
  <div class="inv-row__bench">
    <span class="inv-row__name">{{ r.benchName }}</span>
    <span class="inv-row__client">{{ r.clientName }}</span>
  </div>

  <div class="inv-row__cell">
    <span class="inv-row__label">Hours</span>
    <span class="inv-row__value">{{ r.hours }}</span>
  </div>

  <div class="inv-row__cell">
    <span class="inv-row__label">Rate ({{ period.currency }})</span>
    <span class="inv-row__value">{{ r.ratePerHour | number: '1.2-2' }}</span>
  </div>

  <div class="inv-row__cell">
    <span class="inv-row__label">Earnings ({{ period.currency }})</span>
    <span class="inv-row__value">{{ r.earnings | number: '1.2-2' }}</span>
  </div>
</div>
```

```scss
.inv-row {
  display: grid;
  grid-template-columns: minmax(200px, 1fr) 110px 130px 150px;
  column-gap: 20px; align-items: center;
  background: #fff; border: 1px solid var(--border); border-radius: 8px;
  padding: 18px 22px;

  &__name   { font-family: var(--font-display); font-size: 16px; font-weight: 600; display: block; }
  &__client { font-family: var(--font-body); font-size: 12px; color: var(--gray-700); }

  &__cell  { text-align: right; }
  &__label { font-family: var(--font-display); font-size: 11px; letter-spacing: 0.6px;
             text-transform: uppercase; font-weight: 500; color: var(--gray-700); display: block; }
  &__value { font-family: var(--font-display); font-size: 18px; font-weight: 700; margin-top: 2px; }
}
```

- **Currency code in the label** — `EARNINGS (EUR)` — with a bare value beneath.
- **Exact cents**, never rounded to whole units.
- Hours render as bare integers.

---

## 8. Total

```html
<div class="inv-total">
  <span class="inv-total__label">
    {{ period.isOngoing ? 'Total to date (period ongoing)' : 'Paid this period' }}
  </span>
  <span class="inv-total__value">
    {{ period.currency }} {{ period.total | number: '1.2-2' }}
  </span>
</div>

<p class="inv-note" *ngIf="period.isOngoing">
  This period is still ongoing. Your invoice is submitted automatically on the last day of the
  month, covering everything approved by then.
</p>

<p class="inv-note" *ngIf="!period.isOngoing && period.invoiceNumber">
  Invoice {{ period.invoiceNumber }} — submitted automatically on your behalf.
</p>
```

```scss
.inv-total {
  display: flex; align-items: baseline; justify-content: space-between; gap: 12px;
  margin-top: 4px; padding-top: 16px; border-top: 1px solid var(--gray-75);

  &__label { font-family: var(--font-display); font-size: 18px; font-weight: 600; }
  &__value { font-family: var(--font-display); font-size: 22px; font-weight: 700; }
}
.inv-note {
  font-family: var(--font-body); font-size: 13px; color: var(--gray-700);
  line-height: 160%; margin: 12px 0 0;
}
```

The label switches on `isOngoing` — an open period reads **"Total to date (period ongoing)"**, a
closed one **"Paid this period"**.

---

## 9. Empty state

```html
<p class="inv-empty" *ngIf="!period.rows.length">
  No approved hours in this period.
</p>
```

Plain sentence, `--gray-700`, 13px. The card keeps its height — navigating to an empty period
must not collapse it.

---

## 10. Conventions

- Every transition: `300ms cubic-bezier(0.35, 0, 0.25, 1)`.
- All buttons 45px fixed height, except the 35px navigator chevrons.
- Chevrons stroke at 2px.
- Card padding 30px; rows wrap rather than overflowing.

---

## Reference

Open `/prototype/index.html` → **Engineer** dashboard → **Invoices** tab.


---

## Invoicing notice pill

Identical markup on **both** the Work Log and Invoices pages. A change to one must be applied to the other.

```html
<div class="invoice-notice">
  <span class="invoice-notice__icon"></span>
  <span class="invoice-notice__label">Invoicing is handled for you</span>
  <button class="btn btn--tertiary btn--sm" (click)="openUploadModal()">Or upload your own</button>
</div>
```

```scss
.invoice-notice {
  display: inline-flex; align-items: center; gap: 12px;
  box-sizing: border-box;
  border: 1px solid #F0D9A8;
  border-radius: var(--radius-md);
  background: #FEF6E7;
  padding: 9px 14px;

  &__icon {
    width: 16px; height: 16px; flex: none;
    background: #7A5A12;
    mask: url(assets/icons/check.svg) center / contain no-repeat;
  }
  &__label {
    font-family: var(--font-display); font-size: 14px; font-weight: 500;
    color: #7A5A12; white-space: nowrap;
  }
}
```

Sits **top-right, aligned with the page title** — on Work Log it goes *above* the bench tabs so it reads at page level, not per bench. Keep it to one line, ~38px: an earlier full-height card threw the page layout off.

---

## Download invoice

A Small Tertiary Button on the right of the period navigation row.

```html
<button *ngIf="canDownloadInvoice"
        class="btn btn--tertiary btn--sm"
        (click)="downloadInvoice()">Download invoice</button>
```

```ts
get canDownloadInvoice(): boolean {
  // No invoice exists until the period closes — hours can still be logged — and there is
  // nothing to download when no hours were billed.
  return this.period.isClosed && this.rows.length > 0;
}

downloadInvoice(): void {
  this.svc.getInvoicePdf(this.period.key).subscribe(blob => {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = \`castillians-invoice-\${this.period.key}.pdf\`;
    a.click();
    URL.revokeObjectURL(url);
  });
}
```

The PDF is the **same document** submitted on the engineer's behalf, fetched from the server — not re-rendered client-side. Same invoice number, same figures.

---

## Upload modal

Drop zone per the Figma:

```scss
.dropzone {
  display: flex; align-items: center; justify-content: center; gap: 20px;
  cursor: pointer; box-sizing: border-box;
  border: 2px solid var(--border);
  border-radius: 8px;
  background: var(--drop-bg, #F8F8F8);
  padding: 26px 24px;
  transition: background-color 300ms cubic-bezier(0.35, 0, 0.25, 1);

  &:hover { --drop-bg: #F2F2F2; }

  &__badge { width: 45px; height: 45px; flex: none; }   // assets/icons/upload-badge.svg
  &__lead  { font-family: var(--font-display); font-size: 16px; line-height: 130%; color: #4d4d4d;
             strong { font-weight: 600; color: var(--ink-900); } }
  &__hint  { font-family: var(--font-body); font-size: 12px; color: #4D4D4D; margin-top: 2px; }
}
```

- Label is two-tone: **Click to upload** bold, *or drag and drop* grey. Swaps to the filename once chosen.
- Hint reads "Upload in PDF, under 10MB".
- `accept="application/pdf"` only.
- The badge is a supplied asset (`upload-badge.svg`) rendered as `<img>`, **not** a CSS mask — masking would flatten its two-tone fill.


---

## Blank invoice template

Beneath the drop zone, for engineers without an invoice of their own.

```html
<div class="template-offer">
  <span class="template-offer__label">No invoice of your own?</span>
  <a class="btn btn--tertiary btn--sm"
     href="assets/engineer-invoice-template.pdf"
     download="Castillians-invoice-template.pdf">Download template</a>
</div>
```

Static asset — the same file for every engineer, no data merged in. Addressed to **Castille Resources Ltd.**

---

## Upload UI — platform default

This drop zone is the **platform standard for every file-upload surface**. Reuse it; do not design another.

```scss
.dropzone {
  display: flex; align-items: center; justify-content: center; gap: 20px;
  cursor: pointer; box-sizing: border-box;
  border: 2px solid var(--border);
  border-radius: 8px;
  background: var(--drop-bg, #F8F8F8);
  padding: 26px 24px;
  transition: background-color 300ms cubic-bezier(0.35, 0, 0.25, 1);

  &:hover { --drop-bg: #F2F2F2; }

  &__badge { width: 45px; height: 45px; flex: none; }
  &__lead  { font-family: var(--font-display); font-size: 16px; line-height: 130%; color: #4d4d4d;
             strong { font-weight: 600; color: var(--ink-900); } }
  &__hint  { font-family: var(--font-body); font-size: 12px; color: #4D4D4D; margin-top: 2px; }
}
```

- The whole panel is a `<label>` wrapping a hidden `<input type="file">` — keyboard focus and screen-reader access come for free, with no extra ARIA.
- Badge is `assets/icons/upload-badge.svg` as an `<img>`, **not** a CSS mask — masking flattens its two-tone fill.
- `accept` must always match the formats named in the hint.
- The size limit in the hint is enforced **server-side** too; a hint alone stops nothing.
- Worth extracting as a shared component the second time it is needed.

Full spec: **Upload UI — platform default** in `specs/ENGINEERING-BRIEF.md`.


---

## Interaction patterns

Platform-wide behaviour for pagination, empty/loading/error states, validation, concurrency, filters and money is specified once in **§G of `ENGINEERING-BRIEF.md`**. Read that section before implementing this story — it covers the edge cases this file does not repeat.

### Values for this surface

| Concern | Rule |
|---|---|
| Pagination | **None** — one period at a time, with bounded prev/next navigation |
| Navigation bounds | Arrows disable at the earliest period with data and at the current one |
| Landing period | Opens on the **most recently closed** period, not the open one |
| Empty period | "No hours were logged in this period" and a zero total; the card must not collapse |
| Download button | Hidden when the period is still open **or** has no billable hours — not disabled |
| Multi-currency | One total row **per currency**; never combined, never converted — §G7 |
| Upload | PDF only, 10MB ceiling enforced **server-side** too; file is mandatory with inline validation |
| On success | Modal closes, success toast states that the upload replaces the generated invoice |
