# FE — Work Log: Log Hours

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.** The only change to it is the two new sub-tab entries — **Work Log** and **Invoices** — per the [engineer menubar Figma](https://www.figma.com/design/A4ZWwuOmoeUr7p72TYIiLV/castillians.com?node-id=2837-3986).
>
> Everything below the sub-tab row is in scope; the header above it is not.


Angular handoff for **SD-3455**. The Log hours card on `/work-log`.

> **On conventions.** `luigimf/castillians-engine` was empty when this was written, so the code
> follows standard Angular conventions rather than yours. Re-point it once source lands — the
> structure, values and behaviour are what matter, not the file scaffolding.

**Version-agnostic.** Registration is shown both ways; use whichever your app follows:

```ts
// NgModules (v11–14)                 // Standalone (v15+)
@NgModule({                           @Component({
  declarations: [LogHoursComponent],    standalone: true,
  imports: [CommonModule, FormsModule], imports: [CommonModule, FormsModule],
})                                    })
```

Everything else below is identical across versions. Templates use `*ngIf` / `*ngFor` — swap for
`@if` / `@for` if you are on v17+ and have adopted the new control flow.

---

## 1. Design-system components

| Component | Used for | Status |
|---|---|---|
| `Card` | Card shell | ⬜ confirm exists |
| `Input` | Hours field | ⬜ confirm exists |
| `Button` | Submit | ⬜ confirm exists |

**Likely missing:** the **date picker**. It must be custom (see §4) — if you already have one
that supports a min/max range with disabled days, use it and skip that section.

---

## 2. Component

```ts
@Component({
  selector: 'app-log-hours',
  templateUrl: './log-hours.component.html',
  styleUrls: ['./log-hours.component.scss'],
})
export class LogHoursComponent {
  @Input() benchId!: string;
  @Input() period!: BenchPeriod;
  @Output() logged = new EventEmitter<void>();

  date = new Date().toISOString().slice(0, 10);
  hours = '';
  description = '';
  hoursError: string | null = null;
  projection: Projection | null = null;   // from the API, never computed here
  submitting = false;

  constructor(private svc: WorkLogService) {}

  onHoursInput(raw: string): void {
    // type="text" + sanitise: a number input reports "" to change while holding "-19",
    // so a minus sign bypasses sanitising, and it steps on mouse-wheel.
    this.hours = raw.replace(/[^0-9.]/g, '');
    const n = parseFloat(this.hours);
    this.hoursError = n > 16 ? 'Enter hours between 0.5 and 16.' : null;
    this.refreshProjection();
  }

  submit(): void {
    if (this.submitting || !this.canSubmit) return;
    this.submitting = true;
    this.svc.createEntry({
      benchId: this.benchId,
      date: this.date,
      hours: parseFloat(this.hours),
      description: this.description.trim(),
    }).subscribe({
      next: () => { this.reset(); this.logged.emit(); },
      error: (e) => { this.serverError = e.error?.message ?? null; },
      complete: () => (this.submitting = false),
    });
  }
}
```

```ts
export interface Projection {
  state: 'blocked' | 'within_tolerance' | 'needs_approval' | null;
  message: string;      // server-supplied copy — do not construct it client-side
}
```

---

## 3. Field standards

Platform-wide, not specific to this card.

| Property | Value |
|---|---|
| Label font | Bricolage, 16px, **weight 400** |
| Typed input | weight 500 |
| Field height | 45px fixed |
| Border | 2px `--border`, focus `--ink-900` |
| Radius | 6px |
| Required marker | **black** asterisk, not red |

```scss
:host ::ng-deep .field label { font-weight: 400; }  // DS Input may bake a bolder weight
input, textarea, select { font-weight: 500; }
::placeholder { color: var(--gray-300); font-family: var(--font-body); }
```

---

## 4. Date field

Custom picker — the native control cannot express the period constraint legibly and its locale
mask varies by browser.

```html
<div class="field">
  <label class="field__label" for="log-date">Date</label>

  <button type="button" id="log-date" class="field__control date-trigger"
          (click)="pickerOpen = !pickerOpen">
    {{ date | date: 'd MMMM y' }}
    <span class="date-trigger__icon" aria-hidden="true"></span>
  </button>

  <app-date-picker *ngIf="pickerOpen"
                   [value]="date"
                   [min]="period.minLoggableDate"
                   [max]="period.maxLoggableDate"
                   (select)="onDateSelected($event)"
                   (dismiss)="pickerOpen = false"></app-date-picker>
</div>
```

**Picker behaviour**

- Defaults to today; anchored to the field; closes on outside click or Escape.
- Month-navigable, arrows disabled at the ends of the range.
- Days outside `min`…`max` render **disabled, not hidden**.
- Footer carries a **"Jump to today"** link that selects today and closes the panel — dimmed
  and disabled when today falls outside the period.
- The calendar icon sits **10px inside** the field, matching the text's left inset.

Where a native date input is used elsewhere, uppercase its placeholder — date fields only:

```scss
input[type='date'] { text-transform: uppercase; }
```

---

## 5. Hours field

```html
<div class="field">
  <label class="field__label" for="log-hours">Hours</label>
  <input id="log-hours" class="field__control" type="text" inputmode="decimal"
         placeholder="e.g. 6" [ngModel]="hours"
         (ngModelChange)="onHoursInput($event)">
  <p class="field__error" *ngIf="hoursError">{{ hoursError }}</p>
</div>
```

**`type="text"` with `inputmode="decimal"` is required, not stylistic.** `type="number"` fails
three ways: it reports `""` to change while internally holding `-19` so a minus bypasses
sanitising; it steps on mouse-wheel, silently altering a set value; and its spinners need
vendor-prefixed removal anyway.

Validation shows **live as the user types**, not on submit.

---

## 6. Description field

```html
<div class="field">
  <label class="field__label" for="log-desc">
    Work description <span aria-hidden="true">*</span>
  </label>
  <textarea id="log-desc" class="field__control field__control--area"
            placeholder="What did you work on?" [(ngModel)]="description"></textarea>
</div>
```

```scss
.field__control--area { min-height: 96px; resize: vertical; padding: 12px 14px; }
```

Mandatory — block submission client-side before the request is made.

---

## 7. Projected-state note

Sits above the submit button. Appears once hours are entered **and** the entry would cross a
threshold. State and copy come from the API.

```html
<div class="projection" *ngIf="projection?.state"
     [ngClass]="'projection--' + projection.state">
  {{ projection.message }}
</div>
```

```scss
.projection {
  border-radius: 6px; padding: 12px 14px;
  font-size: 12px; line-height: 150%; font-weight: 500; color: var(--ink-900);
  min-height: 62px;   // reserved, so the submit button does not jump

  &--blocked          { background: var(--danger-50);  border: 2px solid var(--danger-500); }
  &--within_tolerance { background: var(--gray-50);    border: 2px solid var(--border); }
  &--needs_approval   { background: var(--warning-50); border: 2px solid var(--warning-border); }
}
```

**The three strings, verbatim:**

| State | Copy |
|---|---|
| Blocked | "This entry exceeds your assigned capacity and overages are not open on this bench, so it can't be logged. Speak to your manager if more hours are needed." |
| Within tolerance | "This entry is within the hours agreed for this period and will be logged straight away." |
| Needs approval | "This entry exceeds your assigned capacity and will be sent to the Castillians team for approval." |

**No figures in the copy.** The note states the outcome, not the arithmetic — quoting hours
would leak bench-level detail this dashboard deliberately withholds.

Unlimited overage → **no note at all**.

---

## 8. Submit

```html
<button type="button" class="btn btn--primary btn--block"
        [disabled]="submitting" (click)="submit()">Log Hours</button>
```

The label stays **"Log Hours"** in every state — routing is automatic, and the note above
already explains what will happen. Do not switch it to "Log & Flag for Approval".

**On success:**

- The entry appears in the list immediately; every overview figure recomputes. No reload.
- **Pagination resets to page 1** on every list showing it — Engineer entries, Internal Work
  Logs, Manager bench logs — so a new entry can never land off the page being viewed.
- A toast fires; a distinct copy states when an entry has been sent for approval.

```scss
.toast-host {
  position: fixed; right: 24px; top: 130px;   // clears the sticky header and sub-tab row
  z-index: 95; width: 380px; max-width: calc(100vw - 48px);
}
```

---

## 9. Conventions

- Every transition: `300ms cubic-bezier(0.35, 0, 0.25, 1)`.
- All buttons and fields 45px fixed height.
- Chevrons stroke at 2px.

---

## Bundle contents

```
FE-LOG-HOURS.md   this file
BE-LOG-HOURS.md   create endpoint, auto-accept ceiling, validation order, test cases
prototype/        open Castillians Platform.dc.html
```

In the prototype, open the **Engineer** dashboard → **Work Log**. Log hours on **Insurance Web**
(overages off, near capacity) and **Core Platform** (overages on, past tolerance) to see both
note states.


---

## Interaction patterns

Platform-wide behaviour for pagination, empty/loading/error states, validation, concurrency, filters and money is specified once in **§G of `ENGINEERING-BRIEF.md`**. Read that section before implementing this story — it covers the edge cases this file does not repeat.

### Values for this surface

| Concern | Rule |
|---|---|
| Validation | Server-side is the gate; client checks are for immediacy — §G3 |
| Live validation | The 16-hour ceiling validates **as the user types**, not on submit |
| Invalid fields | `[data-invalid]` on the wrapper turns the field red; required markers are a **black** asterisk |
| Error order | The server returns the **first** failure, in the order given in `BE.md` — not an aggregate |
| Double-submit | Guarded **server-side**, not only by disabling the button |
| On success | Modal-free form: clears, fires a success toast, and **resets the Entries list to page 1** |
| Concurrency | Two engineers against the same remaining hours: exactly one succeeds — §G5 |
| Threshold copy | Comes from the API response; the client never recomputes a threshold |
