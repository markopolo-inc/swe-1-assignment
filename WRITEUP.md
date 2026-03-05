# Ticket Service Take-Home Write-up

## Branch Mapping

- `vulnerable-baseline`: original unmodified codebase.
- `fixed-solution`: bug reproducer + fixes + this write-up.

## 1) Root Cause Analysis

The overselling and duplicate ticket-number bugs come from a race condition in `purchaseTickets`.

In the original implementation:
1. It reads `ticket_pools` (`SELECT *`) without locking.
2. It computes ticket numbers from `currentTotal = total - available`.
3. It inserts tickets one-by-one.
4. It decrements availability in a separate query.

Under concurrent requests, multiple transactions read the same `available` value at the same time, compute the same ticket range, and both proceed.

This causes:
- **Duplicate ticket numbers**: two users can both insert the same `ticket_number` range.
- **Overselling**: multiple requests each pass the `available < quantity` check, then all decrement availability afterward.

There is also no DB uniqueness constraint on `(event_id, ticket_number)`, so duplicates are accepted silently.

## 2) Reproduction Script

I added `src/reproduceOriginalBugs.ts`.

What it does:
- Resets a test event with a tiny pool (`total=32`, `available=32`).
- Fires many concurrent `POST /purchase` requests (`quantity=8` each).
- Reads DB state and checks:
  - issued count vs total and available
  - `COUNT(*)` vs `COUNT(DISTINCT ticket_number)`
  - duplicate ticket numbers list

Run:

```bash
npm run repro
```

Expected against vulnerable code: it reliably reports both overselling and duplicate ticket numbers.

## 3) Fix Implemented

### Application-level fix

In `src/ticketService.ts`, I changed purchase allocation to a single transaction with row-level locking:
- `BEGIN`
- `SELECT ... FROM ticket_pools WHERE event_id = $1 FOR UPDATE`
- Validate availability while row is locked
- Insert full ticket range in one SQL statement using `generate_series`
- Update `available`
- `COMMIT` (or `ROLLBACK` on error)

This serializes purchases per event and guarantees no two concurrent requests can allocate overlapping ticket ranges.

### Database-level safety fix

Added schema protections:
- Unique index on `(event_id, ticket_number)`
- Check constraints on `ticket_pools`:
  - `total >= 0`
  - `available >= 0`
  - `available <= total`

Added in:
- `init.sql` for fresh DB bootstrap
- `src/seed.ts` so existing local DBs get the same constraints/index idempotently

## 4) Tradeoffs

### Why this approach
- Minimal external behavior change (same API and response shape)
- Strong correctness guarantees with standard PostgreSQL transaction semantics
- Small, maintainable code change

### Cost
- Requests for the same event now serialize on the locked row, which can increase latency under extreme contention for one event.
- Requests for different events still run concurrently.

## 5) Bonus: Scalable Path for 10k+ RPS / Multi-Instance

For very high throughput across many service instances:
1. Keep DB uniqueness constraints as final correctness guardrail.
2. Replace per-request row-lock flow with **atomic range reservation**:
   - Add `next_ticket_number` to event pool.
   - Do one atomic `UPDATE ... SET next_ticket_number = next_ticket_number + quantity, available = available - quantity WHERE available >= quantity RETURNING old/new range`.
   - Then bulk insert reserved range.
3. Optionally partition `issued_tickets` by `event_id` or time for write scaling.
4. Use connection pooling tuned for DB cores and workload.

This reduces lock hold time and minimizes round trips while preserving correctness.

## 6) Post-fix Validation

Run:

```bash
npm run repro:fixed
```

Expected: no overselling and no duplicate ticket numbers observed.
