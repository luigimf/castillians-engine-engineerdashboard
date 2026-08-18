# FE — Work Log: Entries

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.** The only change to it is the two new sub-tab entries — **Work Log** and **Invoices** — per the [engineer menubar Figma](https://www.figma.com/design/A4ZWwuOmoeUr7p72TYIiLV/castillians.com?node-id=2837-3986).
>
> Everything below the sub-tab row is in scope; the header above it is not.


Angular handoff for **SD-3456**. The Entries card on `/work-log` — list view, calendar view,
and the edit-history accordion.

> **On conventions.** `luigimf/castillians-engine` was empty when this was written, so the code
> follows standard Angular conventions rather than yours. Re-point it once source lands.

**Version-agnostic.** Registration is shown both ways; use whichever your app follows:

```ts
// NgModules (v11–14)                    // Standalone (v15+)
@NgModule({                              @Component({
  declarations: [EntriesComponent],        standalone: true,
  imports: [CommonModule],                 imports: [CommonModule],
})                                       })
```

Templates use `*ngIf` / `*ngFor` — swap for `@if` / `@for` on v17+ if you have adopted the new
control flow.

---

## 1. Design-system components

| Component | Used for | Status |
|---|---|---|
| `Card` | Card shell | ⬜ confirm exists |
| `StatusBadge` | Entry status | ⬜ confirm exists |
| `Button` | Edit action | ⬜ confirm exists |
| `Pagination` | List paging | ⬜ confirm exists |

**Likely missing:** the **month calendar grid**. Worth a shared component — the Manager
dashboard needs the same thing.

---

## 2. Models

```ts
export type EntryStatus = 'auto_approved' | 'approval_required' | 'approved' | 'declined';

export interface WorkLogEntry {
  entryId: string;
  date: string;
  hours: number;
  description: string;
  status: EntryStatus;
  canEdit: boolean;                 // false on prior periods and ended engagements
  declineMessage: string | null;
  history: HistoryRecord[];
}

export interface HistoryRecord {
  timestamp: string;
  actorName: string;                // 'Castillians Team' / 'Castillians System' come from the API
  actorType: 'engineer' | 'internal' | 'system';
  action: 'Created' | 'Edited' | 'Approved' | 'Declined' | 'Sent for approval';
  detail: string;
  diffs: HistoryDiff[];
}

export interface HistoryDiff { field: 'Hours' | 'Date' | 'Description'; from: string; to: string; }

export interface CalendarMonth {
  earliestMonth: string;            // 'YYYY-MM' — bounds the month arrows
  latestMonth: string;
  days: Record<string, { hours: number; entries: number }>;
}
```

**`auto_approved` and `approved` are distinct** — the engineer needs to tell an automatic pass
from a human sign-off. Do not collapse them.

---

## 3. Card header and view toggle

The toggle sits **beneath** the title, not inline with it.

```html
<header class="entries-header">
  <h2 class="entries-title">Entries</h2>

  <div class="view-toggle" role="tablist">
    <button type="button" role="tab" class="view-toggle__btn"
            [class.view-toggle__btn--active]="view === 'list'"
            [attr.aria-selected]="view === 'list'"
            (click)="view = 'list'">List</button>
    <button type="button" role="tab" class="view-toggle__btn"
            [class.view-toggle__btn--active]="view === 'calendar'"
            [attr.aria-selected]="view === 'calendar'"
            (click)="view = 'calendar'">Calendar</button>
  </div>
</header>
```

```scss
.entries-header { display: flex; flex-direction: column; gap: 14px; padding: 20px 30px 14px; }
.entries-title  { font-family: var(--font-display); font-size: 26px; font-weight: 700; margin: 0; }

.view-toggle { display: flex; align-items: center; gap: 8px; }
.view-toggle__btn {
  font-family: var(--font-body); font-size: 12px; font-weight: 500;
  padding: 0 12px; height: 32px; border-radius: var(--radius-md);
  background: #fff; color: var(--gray-700); border: 2px solid var(--border); cursor: pointer;

  &--active {
    font-weight: 600; background: var(--ink-900); color: #fff; border-color: var(--ink-900);
  }
}
```

---

## 4. Container behaviour

The card keeps a **fixed width and minimum height across both views**.

```scss
.entries-body { min-height: 420px; }   // covers the taller of list and calendar
```

Two defects seen during prototyping, both avoided by the above:

1. Switching to calendar shrank the card, because the grid was narrower than the list.
2. Navigating to a month with no entries collapsed it further.

---

## 5. Entry card

```html
<article class="entry" *ngFor="let e of pagedEntries">
  <div class="entry__head">
    <span class="entry__hours">{{ e.hours }}h</span>
    <span class="entry__date">{{ e.date | date: 'd MMM' }}</span>
    <app-status-badge [variant]="badgeVariant(e.status)">
      {{ badgeLabel(e.status) }}
    </app-status-badge>
  </div>

  <p class="entry__desc">{{ e.description }}</p>

  <p class="entry__decline" *ngIf="e.status === 'declined' && e.declineMessage">
    “{{ e.declineMessage }}”
  </p>

  <app-edit-history *ngIf="e.history.length > 1" [records]="e.history"></app-edit-history>
</article>
```

```scss
.entry {
  background: #fff; border: 1px solid var(--border); border-radius: 8px; padding: 18px 22px;

  &__head  { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
  &__hours { font-family: var(--font-display); font-size: 16px; font-weight: 700; }
  &__date  { font-family: var(--font-body); font-size: 13px; color: var(--gray-700); }
  &__desc  { margin: 6px 0 0; font-family: var(--font-body); font-size: 13px;
             line-height: 150%; color: #4d4d4d; }
  &__decline {
    margin-top: 8px; padding: 8px 12px; border-radius: 6px;
    background: var(--danger-50); color: var(--danger-500);
    font-size: 12px; line-height: 150%;
  }
}
```

Hours lead at 16px/700 with the date secondary — hours are what the engineer scans for.

```ts
badgeVariant(s: EntryStatus) {
  return { auto_approved: 'success', approved: 'success',
           approval_required: 'warning', declined: 'danger' }[s];
}
badgeLabel(s: EntryStatus) {
  return { auto_approved: 'Auto-approved', approved: 'Approved',
           approval_required: 'Approval required', declined: 'Declined' }[s];
}
```

Entries sort **most recent first**.

---

## 6. Edit history accordion

Bottom-left of the entry card. **Not a modal.** Hidden entirely when there is no history beyond
creation — hence the `*ngIf="e.history.length > 1"` above.

```html
<button type="button" class="history__toggle"
        [attr.aria-expanded]="open" (click)="open = !open">
  Edit history
  <svg class="history__chevron" [class.history__chevron--open]="open"
       width="11" height="8" viewBox="0 0 11 7" fill="none" aria-hidden="true">
    <path d="M1 1L5.5 5.5L10 1" stroke="currentColor" stroke-width="2"
          stroke-linecap="round" stroke-linejoin="round"/>
  </svg>
</button>

<div class="history__reveal" [class.history__reveal--open]="open">
  <div class="history__inner">
    <div class="history__row" *ngFor="let r of records; let i = index"
         [class.history__row--historic]="i > 0">
      <time class="history__time">{{ r.timestamp | date: 'd MMM y, HH:mm' }}</time>
      <p class="history__actor">{{ r.actorName }}</p>

      <div class="history__action">
        <span class="history__tag">{{ r.action }}</span>
        <span class="history__detail">{{ r.detail }}</span>
      </div>

      <div class="history__diff" *ngFor="let d of r.diffs">
        <span class="history__field">{{ d.field }}</span>
        <span class="history__from">{{ d.from }}</span>
        <span class="history__arrow" aria-hidden="true">→</span>
        <span class="history__to">{{ d.to }}</span>
      </div>
    </div>
  </div>
</div>
```

```scss
.history {
  &__toggle {
    display: inline-flex; align-items: center; gap: 8px; margin-top: 12px;
    background: none; border: none; padding: 0; cursor: pointer;
    font-family: var(--font-body); font-size: 12px; font-weight: 600; color: var(--gray-700);
  }
  &__chevron {
    transition: transform 300ms cubic-bezier(0.35, 0, 0.25, 1);
    &--open { transform: rotate(180deg); }
  }

  // grid-rows: smooth reveal with no hardcoded height, and the card grows rather than clipping
  &__reveal {
    display: grid; grid-template-rows: 0fr;
    transition: grid-template-rows 300ms cubic-bezier(0.35, 0, 0.25, 1);
    &--open { grid-template-rows: 1fr; }
  }
  &__inner { overflow: hidden; }

  &__row {
    background: #fff;                                  // current values
    border: 1px solid var(--border); border-radius: 8px;
    padding: 14px 18px; margin-top: 8px;
    &--historic { background: var(--gray-50); }        // historic values
  }

  &__time   { font-family: var(--font-body); font-size: 11px; color: var(--gray-700); }
  &__actor  { margin: 2px 0; font-family: var(--font-display); font-size: 13px; font-weight: 700; }
  &__action { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; margin-top: 6px; }

  &__tag {
    display: inline-flex; align-items: center; padding: 6px 9px;
    border: 1px solid var(--gray-150); border-radius: var(--radius-md); background: #fff;
    font-family: var(--font-body); font-size: 11px; font-weight: 500;
    line-height: 100%; color: var(--gray-700);
  }
  &__detail { font-size: 13px; line-height: 150%; color: var(--gray-700); }

  &__diff  { display: flex; align-items: baseline; gap: 8px; flex-wrap: wrap;
             margin-top: 8px; font-size: 13px; line-height: 150%; }
  &__field { font-family: var(--font-display); font-size: 11px; letter-spacing: 0.5px;
             text-transform: uppercase; font-weight: 600; color: var(--gray-700); flex: none; }
  &__from  { color: #9a9a9a; text-decoration: line-through; }
  &__arrow { color: #9a9a9a; flex: none; }
  &__to    { color: var(--ink-900); font-weight: 500; }
}
```

- History rows reuse the **entry's own card container** — white for current, grey for historic.
- A `Created` row shows the original description in quotes on its own line.
- Actor names come from the API. Do not construct `Castillians Team` client-side.
- Newest first.

---

## 7. List view

- Paginates. Controls render **only** when there is more than one page.
- The last entry carries bottom padding matching the card's 30px side gutters, or it reads as
  clipped.
- Empty state: plain sentence, `--gray-700`, 13px — "No hours logged for this bench in this
  period."

---

## 8. Calendar view

```html
<app-month-calendar [month]="month" [data]="calendar.days"
                    [earliest]="calendar.earliestMonth" [latest]="calendar.latestMonth"
                    (monthChange)="loadMonth($event)"></app-month-calendar>
```

- Each day shows its hours and entry count.
- **Prior months are browsable read-only** — the engineer can look back even though those
  entries cannot be edited.
- Month arrows disable at `earliest` / `latest`.
- Month label in Bricolage 24px/600 between two 35px chevron buttons.
- Days with entries are visually distinct; today is marked.
- An empty month still renders the full grid.

---

## 9. Editing

- The edit affordance renders only when `canEdit` is true — absent on prior-period entries.
- Opens the entry's values using the same fields and validation as Log Hours (SD-3455).
- On save the entry and every overview figure update immediately, no reload.

Status transitions are **server-owned** — see the BE doc. The one worth knowing while building:
an edit with the **same or fewer hours** changes nothing and notifies nobody.

---

## 10. Conventions

- Every transition: `300ms cubic-bezier(0.35, 0, 0.25, 1)`.
- Chevrons stroke at 2px, rotating 180° when their accordion opens.
- Tag padding even on all sides — status badges 6px, chips 10px.
- Toasts anchor top-right at `top: 130px`.

---

## Bundle contents

```
FE-ENTRIES.md   this file
BE-ENTRIES.md   list, calendar, edit and history endpoints + test cases
prototype/      open Castillians Platform.dc.html
```

In the prototype, open the **Engineer** dashboard → **Work Log**. Entries on **Core Platform**
carry multi-step histories with real diffs; one on **Payments Squad** is declined with a
reviewer message.


---

## Pagination

**5 entries per page.** Prev / next chevron buttons at 35px either side of a "Page N of M" label.

```ts
const ENTRIES_PER_PAGE = 5;   // named constant — this value will be tuned
```

- Controls render **only** when `total > ENTRIES_PER_PAGE`. A lone "Page 1 of 1" is noise.
- Chevrons disable at the first and last page.
- Logging a new entry, switching bench tab, or changing period all reset to **page 1**.
- The calendar view has no pagination — it shows the whole month, with its own month navigation.


---

## Interaction patterns

Platform-wide behaviour for pagination, empty/loading/error states, validation, concurrency, filters and money is specified once in **§G of `ENGINEERING-BRIEF.md`**. Read that section before implementing this story — it covers the edge cases this file does not repeat.

### Values for this surface

| Concern | Rule |
|---|---|
| Page size | **5 per page**, as `ENTRIES_PER_PAGE` — a named constant on both client and server |
| Controls | Render only when `total > 5`; prev/next **disabled** at the ends, not hidden |
| Reset to page 1 | On logging an entry, switching bench tab, or changing period |
| Beyond last page | The server returns the **last** page — a stale page number must not blank the card |
| Sort stability | Most recent first, tie-broken on entry id so rows never duplicate across page boundaries |
| Calendar view | **Not paginated** — returns the whole month, with its own month navigation |
| Container height | Fixed minimum across list, calendar, empty and loading states — no reflow when switching |
| Empty | "No hours logged for this bench in this period" — specific, not "No data" |
| History | Accordion hidden entirely when there is no history beyond creation |
