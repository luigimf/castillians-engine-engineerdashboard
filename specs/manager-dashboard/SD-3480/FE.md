# FE — Subscriptions Page: Access

Angular handoff for **SD-3480**. The `/subscriptions` route, its menubar entry, who may reach it, and **which brands each Admin sees on it**. The cards inside are SD-3481, SD-3482 and SD-3483.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header is a navigation device only and **must not be built**.

---

## 1. Route and entry

- Route `/subscriptions` on the Manager dashboard. Menubar entry **Subscriptions**, per [Figma node 11613-8085](https://www.figma.com/design/A4ZWwuOmoeUr7p72TYIiLV/castillians.com?node-id=11613-8085) — same treatment as Organisation (SD-3477).
- Title `h1` **Subscriptions**, Bricolage 42px / 700.
- Single column of cards, 24px apart, opening directly with **Monthly Capacity Plans** (SD-3481). **No summary card.**

## 2. Who may reach it

| | Admin | Manager | Viewer |
|---|---|---|---|
| Menubar entry | Yes | **Absent** | **Absent** |
| Route | Yes | Refused → Virtual Benches with a plain explanation | Same |

Absent, never disabled. The guard is server-side; the absent entry is not the protection.

## 3. Which brands an Admin sees

> **Changed 23 Sep.** Previously this page was "visible to exactly one person" (one Admin per client). There is still one Admin per brand, but **several Admins can reach this page**, each with their own scope.

**An Admin sees their own brand and every brand below it — never a brand beside or above it.** The hierarchy is each brand's **Parent** field in Zoho (SD-3463).

```
Northmill Bank            (root)
└─ Northmill Insurance
   └─ Northmill Capital
      └─ Northmill Wealth
```

| Admin of | Sees |
|---|---|
| Northmill Bank (root) | Bank, Insurance, Capital, Wealth — the whole channel |
| Northmill Insurance | Insurance, Capital, Wealth |
| Northmill Capital | Capital, Wealth |
| Northmill Wealth | Wealth |

- The FE renders the brand groups it is given. It never filters, and never decides scope.
- Brands outside scope do not render: no group, no total, no filter option.
- Totals stay **per brand and per currency** (SD-3481). Never rolled into a parent.
- The same scope drives **Virtual Benches** (SD-3470) and **Organisation** (SD-3477), so the three pages never disagree.

## 4. States (§G2)

| State | Behaviour |
|---|---|
| Loading | Skeleton cards at natural height |
| Error | Inline with retry inside the page body |
| Empty | Monthly Capacity Plans card with its own empty line |

No pagination. Every transition `300ms cubic-bezier(0.35,0,0.25,1)`.

## Reference

```
BE.md                          scope resolution, endpoint, guard
../SD-3481/ ../SD-3482/ ../SD-3483/   the cards on this page
../SD-3477/                    Organisation — same entry pattern, same scope
../../ENGINEERING-BRIEF.md     §A4 BE-15/16 roles and subtree visibility, BE-27 currency
../../../prototype/index.html  → Manager → Subscriptions; Viewing as → Admin / Sub-brand Admin
```
