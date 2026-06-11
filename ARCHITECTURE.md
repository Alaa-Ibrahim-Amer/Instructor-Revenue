# Architecture Documentation

## Key Architectural Decisions

---

### 1. Money Storage — Integer Cents

All monetary values are stored as **unsigned integers representing cents** (e.g. $9.99 = 999).

**Why:** Floating point arithmetic is unreliable for financial calculations (`0.1 + 0.2 ≠ 0.3` in IEEE 754). Integer arithmetic is exact. This is the same approach used by Stripe and most financial systems.

**Trade-off:** All display logic must format cents to dollars. A small presentation cost for a large correctness gain.

---

### 2. Revenue Allocation Strategy — Proportional to Courses Accessed

When a subscription payment is split among instructors:

1. Platform takes its cut first (`PLATFORM_REVENUE_CUT_PERCENTAGE`, default 30%)
2. Remaining amount is split proportionally among instructors based on how many of their courses the student accessed that day
3. Formula: `instructor_share = floor((instructor_courses_accessed / total_courses_accessed) * instructor_pool)`
4. Rounding remainders go to the platform operations account

**Why proportional to courses accessed (not equal split):**
An instructor with 3 courses accessed contributes more value to the student's experience than one with 1 course accessed. Equal split would unfairly reward less-engaged instructors.

**Why not watch time:**
Watch time requires more complex tracking infrastructure not mentioned in the spec. Courses accessed is a reasonable proxy that's simpler to implement and audit.

**Future enhancement — Day-one weighting:**
Courses accessed on day one likely represent the primary reason the student subscribed. A multiplier on day-one access events could better reflect instructor contribution. Not implemented in v1 to keep allocation logic auditable and simple.

---

### 3. Allocation Frequency — Daily

The subscription fee is divided across the days of the term:

```
daily_rate = amount_paid / term_days
```

Each day, a batch job looks at that day's course access logs and distributes `daily_rate` among the instructors involved.

**Why daily:**
- The system must answer "how much is owed at any point in time" — daily allocation satisfies this
- Refunds become clean: only reverse unprocessed future allocations
- Decouples allocation (free, DB-only) from payout (costly, external provider)

---

### 4. Payout Frequency — Monthly

Payouts to instructors are batched monthly, regardless of allocation frequency.

**Why monthly:**
- Every payout call to the external provider has a transaction fee
- Daily or weekly payouts would multiply fees unnecessarily
- Monthly is standard in the industry (YouTube, Udemy, etc.)

---

### 5. Idempotency Approach

Three mechanisms work together to prevent double payments:

**A — Unique constraint on daily_allocations:**
```sql
UNIQUE(subscription_id, instructor_id, course_id, date)
```
Running the allocation job twice for the same day silently ignores duplicates (INSERT IGNORE / upsert).

**B — Idempotency key on payouts:**
```sql
idempotency_key VARCHAR(255) UNIQUE  -- e.g. "instructor_5_2024_01"
```
The monthly payout command generates a deterministic key per instructor per period. A second run finds the existing record and skips it.

**C — Payout status machine:**
```
pending → processing → paid
                    ↘ failed
```
A job only processes a payout in `pending` status. Once `processing` is set (with a DB lock), no other job can claim it. This prevents concurrent execution on multiple workers.

---

### 6. Provider Timeout Handling

The external provider can succeed but not return a response (network drop after money moved). Handling:

1. Generate and store `provider_transaction_id` **before** calling the provider
2. Set payout status to `processing` before the call
3. On timeout exception:
   - Do **not** retry immediately
   - Schedule a `checkStatus(transactionId)` call
   - If status returns `paid` → mark payout as paid, no retry
   - If status returns `failed` → reset to `pending` for retry
   - If status is still unknown → retry status check with backoff

**Why this matters:** Blind retry on timeout = double payment. Always check before retrying.

---

### 7. Refund Handling

Two distinct scenarios:

**Mid-term cancellation:**
- Student stops using the platform partway through their term
- Unused days are refunded proportionally: `refund = daily_rate * remaining_days`
- Daily allocations stop at cancellation date
- Already-allocated earnings are kept — instructors are not penalized for student cancellation

**Full refund (human decision):**
- Admin-initiated only, never automated
- Fault determines ownership:
  - Platform fault → platform absorbs loss via platform_accounts, instructor keeps earnings
  - Instructor fault → flagged, instructor balance adjusted, manual resolution
- Full automated clawback is a future plan (see below)

---

### 8. Rounding Remainder — Platform Operations Account

When cents don't divide evenly among instructors, the remainder goes to `platform_accounts` with type `rounding_remainder`.

**Why not discard:** The books must always balance. Every cent is accounted for.

**Future use:** Accumulated remainder can fund student rewards, promotions, or be absorbed as platform income.

---

## Scaling Considerations

The system is designed for 500,000 active subscriptions and tens of millions of records.

| Concern | Approach |
|---|---|
| Daily allocation at scale | Chunked batch processing, queue-based per subscription |
| Monthly payout at scale | One job per instructor, dispatched from Artisan command |
| Balance queries | Denormalized `instructor_balances` table — no expensive aggregation at query time |
| Index strategy | `daily_allocations(instructor_id, status, date)` for fast outstanding balance lookups |

---

## Known Limitations

1. **Watch time not tracked** — proportional allocation by courses accessed is a reasonable proxy but not perfectly fair
2. **Full refund clawback not automated** — requires manual admin intervention
3. **No multi-currency support** — system assumes a single currency
4. **Queue driver** — uses database driver locally; Redis recommended for production at scale
5. **Provider status check** — currently polling-based; webhook-based confirmation would be more robust in production

---

## Future Improvements

1. **Day-one weighting** — multiply day-one course access events by a configurable factor to better reflect subscription intent
2. **Automated refund clawback** — when instructor is at fault, automatically reverse pending allocations and freeze balance
3. **Plan upgrade mid-term** — prorate the remaining term at the new plan rate, create a new subscription record for the upgraded period, carry forward the old allocation records (see Senior Bonus section in review session)
4. **Webhook-based provider confirmation** — instead of polling `checkStatus`, receive provider callbacks for definitive payment confirmation
5. **Redis queues** — replace database queue driver for higher throughput at scale
6. **Audit log** — append-only event log for every financial state change (regulatory compliance)
