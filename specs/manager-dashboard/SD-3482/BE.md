# BE — Subscriptions Page: Manage Subscription & New Bench Request

Backend specification for **SD-3482**.

> **`BE-nn` refers to numbered sections in `specs/ENGINEERING-BRIEF.md`**, not this file.

---

## 1. Guard

> **Changed 23 Sep.** Previously: Admin only (a single Admin per client).

- `POST /benches/{id}/capacity-requests` and `POST /bench-requests` → `403` unless the caller is an `admin` **and** the target brand is in `scope(caller)` (SD-3480 §2).
- Manager / Viewer → `403`. Admin outside the subtree → `403`.

## 2. Capacity requests

- The **same record** as the bench-page request (SD-3471). One queue.
- _Outdated on 6 Oct. Previously: "At most one open request per bench, across every Admin. A second POST while one is open → 409, returning the open request. The record stores raisedBy (member id and name). Returned on every read so the pending state can name them."_
- **Several open requests per bench are allowed.** A further `POST` while one is open → `201`, a new record. No `409`.
- The record stores `raisedBy` (member id and name) for the Internal queue. **No open-request field is returned to the Manager dashboard** — there is no pending state to render.
- Has **no effect** on `POST` for Request more capacity this month (SD-3472), and vice versa.
- Creates a request only. Nothing on the subscription changes until the team actions it (SD-3465).

## 3. New-bench requests

- The same record as SD-3470's request.
- `clientId` must be in `scope(caller)` → `422` otherwise.

## 4. Tests

- Root Admin raises a capacity request on Insurance Web → `201`.
- _Outdated on 6 Oct. Previously: "Insurance Admin then raises one on the same bench → 409, returns the root Admin's request with raisedBy."_
- Insurance Admin then raises one on the same bench → `201`, a second record; both carry their own `raisedBy`.
- With a capacity plan request open, Request more capacity this month on the same bench → `201`.
- Insurance Admin requests a bench for Northmill Bank → `422`.
- Any request leaves every figure on every surface unchanged.
