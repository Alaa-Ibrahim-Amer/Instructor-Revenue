# AI Usage Documentation

## How AI Was Used During This Task

---

### Workflow Overview

AI was used as a **collaborative design partner** throughout the process — not as a code generator. The workflow was:

1. Read and analyzed the quest requirements
2. Used AI to break down the problem domain interactively — asking questions, challenging assumptions, and stress-testing decisions
3. Made architectural decisions jointly, with full understanding of the trade-offs
4. Used AI to help produce boilerplate code (migrations, factories, seeders) after the design was locked
5. Manually reviewed, modified, and tested all generated code

---

### Main Prompts / Workflows Used

**Phase 1 — Requirements Analysis**
- Asked AI to summarize the quest and flag intentionally unspecified rules
- Identified: revenue split strategy, when money is "earned", refund policy, rounding remainders — none of which were specified

**Phase 2 — Design Decisions (Interactive)**
Key decisions made through structured discussion:
- Revenue allocation strategy → proportional to courses accessed (not equal split, not watch time)
- Allocation frequency → daily (DB only, free)
- Payout frequency → monthly (to minimize provider fees)
- Money storage → integer cents (not DECIMAL, not float)
- Rounding → platform operations account (not discarded, not given to first instructor)
- Refund handling → two separate scenarios (cancellation vs full refund)
- Idempotency approach → three-layer: unique DB constraint + idempotency key + status machine

**Phase 3 — Code Generation**
- Migrations: generated from agreed schema, reviewed for correctness
- MockPaymentProvider: generated with random behavior, manually adjusted timeout simulation
- RevenueAllocationService: core logic hand-written after AI produced skeleton
- Tests: written manually based on known failure scenarios

---

### Fully Generated vs Manually Designed

| Component | Origin |
|---|---|
| Schema design | Jointly designed, fully understood |
| Migration files | AI-generated boilerplate, manually reviewed |
| RevenueAllocationService core math | Manually written |
| PayoutService idempotency logic | Manually written |
| MockPaymentProvider | AI-generated, manually adjusted |
| Artisan command structure | AI-generated skeleton, manually filled |
| Jobs | AI-generated skeleton, manually filled |
| Filament resource | AI-generated, manually adjusted |
| Tests | Manually written |
| Documentation | Jointly produced, personally reviewed |

---

### Engineering Decisions Made Personally

1. **Separating allocation from payout frequency** — AI initially suggested daily payouts. I pushed back because each provider call has fees. Daily allocation + monthly payout was my call.

2. **Platform operations account for remainders** — AI suggested giving remainders to the first instructor or the platform generically. I proposed a named platform_accounts table with typed entries so remainders are trackable and usable for future rewards.

3. **Two-scenario refund model** — AI initially treated all refunds the same. I separated cancellation (automated, proportional) from full refund (human decision, ownership-based) because they have fundamentally different causes and handling.

4. **Day-one weighting as future enhancement** — I noticed that day-one course accesses carry different intent signal than later accesses. Rather than over-engineering it now, I documented it as a future enhancement with clear reasoning.

5. **Provider transaction ID stored before the call** — This ordering detail is critical for timeout safety. I made sure this was explicit in the PayoutService, not left to chance.

---

### What Makes This Solution Different

- **Design-first approach** — every decision was made and understood before writing code, not reverse-engineered from generated output
- **Explicit assumption documentation** — every unspecified rule was identified, decided, and written down
- **Two-layer financial model** — allocation (daily, free) and payout (monthly, costly) are explicitly separate concerns
- **Platform operations account** — remainders are tracked and purposeful, not silently discarded
- **Refund ownership model** — platform vs instructor fault determines who absorbs loss, with clear escalation path

---

### AI Suggestions Rejected

- **Equal split allocation** — AI suggested this as the simplest option. Rejected because it unfairly treats instructors with more accessed content the same as those with less.
- **Immediate allocation on payment** — AI's first suggestion. Rejected because it creates refund complexity and doesn't reflect actual usage.
- **DECIMAL(10,2) for money** — AI mentioned this as a valid option. Chose integers instead for performance and to eliminate any floating point risk entirely.
- **Daily payouts** — AI's initial framing. Rejected due to provider fee implications.

---

### Trade-offs Intentionally Chosen

| Decision | Trade-off Accepted |
|---|---|
| Courses accessed (not watch time) | Less precise fairness, much simpler implementation and audit |
| Daily allocation (not on-access) | Slight delay in balance accuracy vs real-time, much simpler at scale |
| Monthly payout batching | Instructor waits up to 30 days, platform saves on provider fees |
| No automated full-refund clawback | Manual resolution burden, avoids complex automated clawback logic in v1 |
| Database queue driver | Lower throughput than Redis, acceptable for development and demo |
