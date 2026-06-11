# Instructor Revenue 

A financial ledger system for an LMS platform that handles subscription revenue allocation to instructors, monthly payouts, and failure-safe money movement.

---

## Setup Instructions

### Requirements
- PHP 8.2+
- MySQL 8+
- Composer
- Node.js 18+

### Installation

```bash
# Clone the repository
git clone <repo-url>
cd instructor-revenue

# Install dependencies
composer install
npm install && npm run build

# Environment setup
cp .env.example .env
php artisan key:generate

# Configure your .env
# DB_DATABASE, DB_USERNAME, DB_PASSWORD
# PLATFORM_REVENUE_CUT_PERCENTAGE=30

# Run migrations
php artisan migrate

# Seed demo data
php artisan db:seed --class=DemoSeeder

# Start the queue worker
php artisan queue:work

# Start the server
php artisan serve
```

### Docker (Optional)

```bash
docker-compose up -d
docker-compose exec app php artisan migrate
docker-compose exec app php artisan db:seed --class=DemoSeeder
```

---

## How to Run Tests

```bash
# All tests
php artisan test

# Or with Pest directly
./vendor/bin/pest

# Specific test file
./vendor/bin/pest tests/Unit/RevenueAllocationTest.php

# With coverage
./vendor/bin/pest --coverage
```

---

## Key Artisan Commands

```bash
# Run daily revenue allocation (normally scheduled)
php artisan allocations:run --date=2024-01-15

# Run monthly payouts
php artisan payouts:run

# Check payout status (for timed-out transactions)
php artisan payouts:check-pending
```

---

## Assumptions Made

### Revenue Allocation
1. Only courses accessed by the student count toward instructor share calculation — instructors with unaccessed courses receive nothing from that subscription payment.
2. Allocation is **proportional to courses accessed** — if a student accessed 3 of Instructor A's courses and 1 of Instructor B's, A gets 75% of the instructor pool.
3. The platform cut is taken first; instructors split the remainder.
4. Rounding remainders (from integer cent division) go into a platform operations account for future use (e.g. rewards, promotions).
5. Allocation happens daily — the subscription fee is divided across the days of the term (daily_rate = amount_paid / term_days).
6. Courses accessed on day one are treated the same as any other day in the base implementation. A weighted "day-one bonus" multiplier is noted as a future enhancement.

### Subscriptions & Refunds
7. Students pay upfront for the full term on day one.
8. Mid-term **cancellation**: the unused portion of the subscription is refunded to the student proportionally. Allocations stop at cancellation date; already-allocated earnings are kept.
9. **Full refunds** are human-initiated only (admin decision), not automated.
10. Refund ownership: platform fault → platform absorbs loss, instructor keeps earnings. Instructor fault → flagged for manual resolution. Full automated recovery is a future plan.

### Payouts
11. Payout frequency is monthly, regardless of allocation frequency.
12. An instructor must have an outstanding balance > 0 to receive a payout.
13. Each payout covers all pending daily allocations up to the payout date.

### External Payment Provider
14. The mock provider can succeed, fail, or time out after already moving money.
15. On timeout, the system stores the transaction ID before calling the provider and uses a status check on retry — never blindly retrying a payment.

---

## Filament Admin Panel

Visit `/admin` after seeding to access:
- **Instructor Balances** — view total earned, total paid, outstanding per instructor
- **Payout History** — per-instructor payout log with status badges

Default admin credentials (seeded):
- Email: `admin@example.com`
- Password: `password`
