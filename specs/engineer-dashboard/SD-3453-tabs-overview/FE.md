# FE — Work Log: Virtual Bench Tabs & Overview

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.** The only change to it is the two new sub-tab entries — **Work Log** and **Invoices** — per the [engineer menubar Figma](https://www.figma.com/design/A4ZWwuOmoeUr7p72TYIiLV/castillians.com?node-id=2837-3986).
>
> Everything below the sub-tab row is in scope; the header above it is not.


Angular handoff for **SD-3453**. The page frame for `/work-log` — bench tabs, overview card,
days-remaining chip, capacity bar.

> **On conventions.** `luigimf/castillians-engine` was empty when this was written, so the code
> below follows standard Angular conventions rather than yours. Once real source lands, the
> naming, module layout and styling approach here should be re-pointed at it — the structure
> and tokens are what matter, not the file scaffolding.

**Styling:** component-scoped SCSS against the design-system CSS custom properties. No utility
framework assumed; nothing here depends on a build-tool plugin.

---

## 1. Design-system components

You said some exist in code already. Where they do, **use them and delete the equivalent markup
below** — this doc restates their internals only so the visual spec is unambiguous.

| Component | Used for | Status |
|---|---|---|
| `Card` | Overview card shell | ⬜ confirm |
| `Avatar` | Manager thumbnails | ⬜ confirm |
| `Button` | Not used on this screen | — |

**Likely missing — flag if so:**

| Needed | Why it is not a one-off |
|---|---|
| **Capacity bar** | Colour-banded progress bar, used on all three dashboards. Should be one component taking `used`, `total` and rendering the band + tinted track. Currently repeated markup. |
| **Stat tile** | Label + value pair, ~9 instances on this screen alone, more on Internal and Manager. |
| **Expandable chip** | The days-remaining chip. Rarer — a one-off here is defensible. |

---

## 2. Route

Appended to the existing Engineer Dashboard routing. Shell, header and nav untouched.

```ts
// engineer-dashboard-routing.module.ts
const routes: Routes = [
  // …existing routes
  { path: 'work-log', component: WorkLogPageComponent, canActivate: [ActivatedEngineerGuard] },
  { path: 'invoices', component: InvoicesPageComponent, canActivate: [ActivatedEngineerGuard] },
];
```

Reached from the **engineer menubar dropdown** — the updated design is in Figma at
`node-id=2837-3986`. The dropdown gains **Work Log** and **Invoices** entries.

---

## 3. Models

```ts
// work-log.models.ts
export interface EngineerBench {
  benchId: string;
  benchName: string;
  clientName: string;
  benchType: 'Work From Anywhere' | 'EU & Data Safe';
  managers: BenchManager[];        // never empty — drives the Performance Log
}

export interface BenchManager { name: string; email: string; }

export interface BenchPeriod {
  periodStart: string;             // ISO date
  periodEnd: string;
  isFirstPeriod: boolean;
  daysRemaining: number;
  capacityHours: number;
  minLoggableDate: string;
  maxLoggableDate: string;         // earlier of today and periodEnd — server-computed
}

export interface EngineerAllocation {
  allocatedHours: number | null;   // null when the bench has unlimited overage
  loggedHours: number;             // excludes hours awaiting approval
  remainingHours: number;
  percentUsed: number;
  pendingApprovalHours: number;
  overageMode: 'off' | 'on' | 'unlimited';
}

export interface EngineerRate {
  currency: string;                // code, e.g. 'EUR'
  ratePerHour: number;             // already mark-up-adjusted; see BE-08
  earningsThisPeriod: number;
}
```

---

## 4. Service

```ts
@Injectable({ providedIn: 'root' })
export class WorkLogService {
  constructor(private http: HttpClient) {}

  getBenches(): Observable<EngineerBench[]> {
    return this.http.get<EngineerBench[]>('/api/engineer/benches');
  }

  getPeriod(benchId: string): Observable<BenchPeriod> {
    return this.http.get<BenchPeriod>(`/api/engineer/benches/${benchId}/period`);
  }

  getAllocation(benchId: string): Observable<EngineerAllocation> {
    return this.http.get<EngineerAllocation>(`/api/engineer/benches/${benchId}/allocation`);
  }

  getRate(benchId: string): Observable<EngineerRate> {
    return this.http.get<EngineerRate>(`/api/engineer/benches/${benchId}/rate`);
  }
}
```

Fetch period, allocation and rate together per bench:

```ts
readonly benchData$ = this.selectedBenchId$.pipe(
  switchMap(id => forkJoin({
    period: this.svc.getPeriod(id),
    allocation: this.svc.getAllocation(id),
    rate: this.svc.getRate(id),
  })),
  shareReplay(1),
);
```

Re-trigger `selectedBenchId$` after any entry is logged, edited or approved so every figure
recomputes without a reload.

---

## 5. Bench tabs

```html
<nav class="bench-tabs">
  <button *ngFor="let b of benches"
          type="button"
          class="bench-tab"
          [class.bench-tab--active]="b.benchId === selectedBenchId"
          (click)="selectBench(b.benchId)">
    {{ b.benchName }}
  </button>
</nav>
```

```scss
.bench-tabs { display: flex; gap: 8px; margin-bottom: 24px; flex-wrap: wrap; }

.bench-tab {
  font-family: var(--font-display);
  font-size: 24px;
  font-weight: 400;
  padding: 15px;
  border: none;
  border-radius: 6px;
  background: transparent;
  color: #828282;
  cursor: pointer;

  // weight must NOT change on hover — it reflows and shifts every tab beside it
  &:hover { color: #141313; }

  &--active {
    font-weight: 700;
    background: var(--ink-900);
    color: #fff;
    &:hover { color: #fff; }   // or the label vanishes against the ink fill
  }
}
```

Tabs **wrap**, they do not scroll horizontally. Only benches the engineer is *activated* on
appear — shortlisted benches are excluded by the API.

---

## 6. Overview card header

```html
<header class="overview-header">
  <h2 class="overview-title">Overview</h2>

  <span class="meta-tag">
    <span class="meta-tag__label">Client</span>
    <span class="meta-tag__value">{{ bench.clientName }}</span>
  </span>

  <span class="meta-tag">
    <span class="meta-tag__label">Current period</span>
    <span class="meta-tag__value">{{ period | periodRange }}</span>
  </span>

  <app-days-remaining-chip [days]="period.daysRemaining"></app-days-remaining-chip>
</header>
```

```scss
.overview-header {
  display: flex; align-items: baseline; gap: 14px; flex-wrap: wrap;
  padding: 28px 30px 7px;
}
.overview-title { font-family: var(--font-display); font-size: 32px; font-weight: 700; margin: 0; }

.meta-tag {
  display: inline-flex; align-items: center; gap: 8px;
  background: #fff;
  border: 1px solid var(--gray-150);
  border-radius: var(--radius-md);
  padding: 8px 12px;
  white-space: nowrap;

  &__label {
    font-family: var(--font-display); font-size: 11px; letter-spacing: 0.8px;
    text-transform: uppercase; font-weight: 600; color: var(--gray-700); line-height: 100%;
  }
  &__value {
    font-family: var(--font-display); font-size: 14px; font-weight: 700; line-height: 100%;
  }
}
```

---

## 7. Days-remaining chip

Colour-coded, and it **expands on click**. Not a tooltip — it must be reachable on touch.

```ts
@Component({
  selector: 'app-days-remaining-chip',
  templateUrl: './days-remaining-chip.component.html',
  styleUrls: ['./days-remaining-chip.component.scss'],
})
export class DaysRemainingChipComponent {
  @Input() days!: number;
  open = false;

  get tone(): 'green' | 'amber' | 'red' {
    if (this.days >= 16) return 'green';
    return this.days >= 6 ? 'amber' : 'red';
  }
}
```

```html
<div class="chip" [ngClass]="'chip--' + tone">
  <button type="button" class="chip__head" [attr.aria-expanded]="open" (click)="open = !open">
    {{ days }} days left
    <svg class="chip__chevron" [class.chip__chevron--open]="open"
         width="11" height="8" viewBox="0 0 11 7" fill="none" aria-hidden="true">
      <path d="M1 1L5.5 5.5L10 1" stroke="currentColor" stroke-width="2"
            stroke-linecap="round" stroke-linejoin="round"/>
    </svg>
  </button>

  <div class="chip__reveal" [class.chip__reveal--open]="open">
    <div class="chip__reveal-inner">
      <p class="chip__note">Your entries are submitted automatically on the last day of the period.</p>
    </div>
  </div>
</div>
```

```scss
.chip {
  display: inline-block;
  border: 1px solid;
  border-radius: var(--radius-md);
  overflow: hidden;

  &--green { background: #E7F7F1; border-color: #B9E6D4; color: #0A5C43; }
  &--amber { background: #FEF6E7; border-color: #F0D9A8; color: #7A5A12; }
  &--red   { background: #FDECEC; border-color: #F3C9C9; color: #8C1F1F; }

  &__head {
    display: inline-flex; align-items: center; justify-content: flex-end;
    width: 100%; border: none; background: transparent; padding: 8px 12px; cursor: pointer;
    font-family: var(--font-display); font-size: 11px; font-weight: 700;
    letter-spacing: 0.5px; text-transform: uppercase; line-height: 100%;
    color: inherit; white-space: nowrap;
  }

  &__chevron {
    margin-left: 8px; flex: none;
    transition: transform 300ms cubic-bezier(0.35, 0, 0.25, 1);
    &--open { transform: rotate(180deg); }
  }

  // grid-rows transition: animates cleanly with no hardcoded pixel height
  &__reveal {
    display: grid;
    grid-template-rows: 0fr;
    transition: grid-template-rows 300ms cubic-bezier(0.35, 0, 0.25, 1);
    &--open { grid-template-rows: 1fr; }
  }
  &__reveal-inner { overflow: hidden; }

  &__note {
    margin: 0; padding: 8px 14px 12px; max-width: 300px; text-align: right;
    font-family: var(--font-body); font-size: 12px; line-height: 150%; color: inherit;
  }
}
```

---

## 8. Overview stats

**Every figure is the signed-in engineer's own.** Never render bench-wide totals or allocation
percentages on this page.

| Tile | Value | Derivation |
|---|---|---|
| TOTAL HOURS | `allocation.allocatedHours` | The engineer's allocated ceiling — **not** bench capacity. `null` → render **No cap** |
| HOURS LOGGED | `allocation.loggedHours` | Their own hours; excludes pending approval |
| HOURS REMAINING | `allocation.remainingHours` | Suppressed entirely when unlimited |
| YOUR HOURLY RATE ({{currency}}) | `rate.ratePerHour` | Already mark-up-adjusted server-side |
| YOUR EARNINGS ({{currency}}) | `rate.earningsThisPeriod` | `loggedHours × ratePerHour` |
| MANAGERS | `bench.managers` | Horizontal list, avatar + name + email |

```html
<div class="stats-row">
  <div class="stat" *ngIf="!isUnlimited; else noCap">
    <span class="stat__label">Total hours</span>
    <span class="stat__value">{{ allocation.allocatedHours }}</span>
  </div>
  <ng-template #noCap>
    <div class="stat">
      <span class="stat__label">Total hours</span>
      <span class="stat__value">No cap</span>
    </div>
  </ng-template>

  <!-- HOURS LOGGED · HOURS REMAINING · rate · earnings -->
</div>

<p class="stats-note" *ngIf="allocation.pendingApprovalHours > 0">
  {{ allocation.pendingApprovalHours }}h awaiting approval, not counted above.
</p>
```

```scss
.stats-row {
  display: flex; flex-direction: row; column-gap: 60px; row-gap: 22px;
  align-items: flex-start; flex-wrap: wrap;
  padding: 14px 30px 26px;
}
.stat {
  display: flex; flex-direction: column;

  &__label {
    font-family: var(--font-display); font-size: 13px; letter-spacing: 1px;
    text-transform: uppercase; font-weight: 600; color: var(--gray-700);
  }
  &__value {
    font-family: var(--font-display); font-size: 26px; font-weight: 700; margin-top: 4px;
  }
}
```

- The three hour values render as **bare integers** — no `h` suffix.
- Currency code lives in the **label** — `YOUR HOURLY RATE (EUR)` — with a bare value.
- Managers lay out **horizontally**, not stacked.

---

## 9. Capacity bar

The engineer's logged hours against their **own** ceiling.

```ts
@Component({ selector: 'app-capacity-bar', /* … */ })
export class CapacityBarComponent implements AfterViewInit {
  @Input() percentUsed = 0;
  width = 0;                        // animates from 0 on mount

  get band(): 'green' | 'amber' | 'red' {
    if (this.percentUsed <= 59) return 'green';
    return this.percentUsed <= 89 ? 'amber' : 'red';
  }

  ngAfterViewInit(): void {
    requestAnimationFrame(() => (this.width = Math.min(this.percentUsed, 100)));
  }
}
```

```html
<div class="capacity" [ngClass]="'capacity--' + band">
  <div class="capacity__track">
    <div class="capacity__fill" [style.width.%]="width"></div>
  </div>
  <span class="capacity__pct">{{ percentUsed }}%</span>
</div>
```

```scss
.capacity {
  display: flex; align-items: center; gap: 12px;
  width: 206px;                 // fixed, so 100% sits identically across benches
  height: 29px; flex: none;

  &__track { flex: 1; height: 8px; border-radius: 4px; overflow: hidden; }
  &__fill  { height: 100%; border-radius: 4px;
             transition: width 300ms cubic-bezier(0.35, 0, 0.25, 1); }
  &__pct   { font-family: var(--font-display); font-size: 26px; font-weight: 700; flex: none; }

  &--green .capacity__track { background: #E7F7F1; } &--green .capacity__fill { background: #10b77f; }
  &--amber .capacity__track { background: #FEF6E7; } &--amber .capacity__fill { background: #f59f0a; }
  &--red   .capacity__track { background: #FDECEC; } &--red   .capacity__fill { background: #ef4343; }
}
```

The track is the **tinted variant** of the band colour, never grey.

---

## 10. Conventions

- Every transition: `300ms cubic-bezier(0.35, 0, 0.25, 1)`.
- Tag padding is even on all sides — chips 10px, status badges 6px.
- Card padding 30px; the stats row wraps rather than overflowing.
- Responsive across desktop, tablet and mobile; works in Chrome, Firefox and Safari.

---

## Bundle contents

```
FE-TABS-AND-OVERVIEW.md   this file
BE-TABS-AND-OVERVIEW.md   endpoints, payloads, worked examples, test cases
prototype/                open Castillians Platform.dc.html
```

In the prototype, switch to the **Engineer** dashboard via the header toggle and open the
**Work Log** tab. It is the source of truth for anything not spelled out here.


---

## Invoicing notice pill

Sits **top-right, aligned with the page title — above the bench tabs**, so it reads at page level rather than per bench. Markup and styling are **identical** to the pill on the Invoices page; a change to one must be applied to the other. See `specs/engineer-dashboard/SD-3458-invoices/FE.md` for the shared markup and the modal it opens.

Keep it to a single line, ~38px tall. An earlier full-height card threw the page layout off and was rejected.

## Managers — plural

`bench.managers` is an **array**. Render all of them horizontally:

```html
<div class="stat">
  <span class="stat__label">Managers</span>
  <div class="managers">
    <div class="managers__item" *ngFor="let m of bench.managers">
      <app-avatar [name]="m.name" [size]="32"></app-avatar>
      <span class="managers__name">{{ m.name }}</span>
    </div>
  </div>
</div>
```

```scss
.managers { display: flex; flex-direction: row; align-items: center; gap: 20px; flex-wrap: wrap; margin-top: 8px;
  &__item { display: flex; align-items: center; gap: 10px; }
  &__name { font-family: var(--font-display); font-size: 14px; font-weight: 600; line-height: 120%; }
}
```

A bench always has at least one manager — it drives the Performance Log — so treat an empty array as a data error rather than rendering nothing.


---

## Interaction patterns

Platform-wide behaviour for pagination, empty/loading/error states, validation, concurrency, filters and money is specified once in **§G of `ENGINEERING-BRIEF.md`**. Read that section before implementing this story — it covers the edge cases this file does not repeat.

### Values for this surface

| Concern | Rule |
|---|---|
| Pagination | **None** — the overview is a fixed set of tiles |
| Loading | Skeleton tiles at the card's natural height; no layout shift when figures arrive |
| Empty | A bench with no logged hours still renders every tile, showing `0` — never an empty card |
| Error | Inline message with retry **inside** the card; filters and tabs stay usable |
| Refetch | Every tile refetches after an entry is logged, edited or approved. Never patch these client-side — §G4 |
| Money | Currency code in the label, exact cents, no conversion — §G7 |
| Dates | `minLoggableDate` / `maxLoggableDate` come from the server; never derive a period boundary client-side |
