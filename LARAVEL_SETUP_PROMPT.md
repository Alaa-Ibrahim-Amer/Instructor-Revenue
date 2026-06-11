# Laravel App Creation Prompt

Use this prompt when setting up the project with AI assistance or a developer.

---

## Prompt

Create a Laravel 11 application called **"instructor-revenue"** with the following exact specifications.

---

## Tech Stack

- **Laravel 11**
- **Livewire v3**
- **Alpine.js**
- **Filament v3** (admin panel)
- **Pest** (testing framework)
- **MySQL** (database)
- **Laravel Queues** (database driver for local, Redis-ready for production)
- **Docker** (optional but include a `docker-compose.yml`)

---

## Installation Steps

```bash
composer create-project laravel/laravel instructor-revenue
cd instructor-revenue

# Filament
composer require filament/filament:"^3.0" -W
php artisan filament:install --panels

# Livewire (comes with Filament but ensure it's explicit)
composer require livewire/livewire

# Pest
composer require pestphp/pest --dev --with-all-dependencies
composer require pestphp/pest-plugin-laravel --dev
php artisan pest:install
```

---

## Environment Configuration (.env)

```env
APP_NAME="Instructor Revenue "
APP_ENV=local
APP_DEBUG=true

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=instructor_revenue
DB_USERNAME=root
DB_PASSWORD=secret

QUEUE_CONNECTION=database

PLATFORM_REVENUE_CUT_PERCENTAGE=30
```

---

## Directory Structure to Create

```
app/
├── Models/
│   ├── Subscription.php
│   ├── CourseAccess.php
│   ├── DailyAllocation.php
│   ├── InstructorBalance.php
│   ├── Payout.php
│   ├── Refund.php
│   └── PlatformAccount.php
│
├── Services/
│   ├── RevenueAllocationService.php
│   └── PayoutService.php
│
├── Jobs/
│   ├── AllocateSubscriptionRevenueJob.php
│   └── ProcessInstructorPayoutJob.php
│
├── Console/Commands/
│   └── RunMonthlyPayoutsCommand.php
│
├── Payment/
│   ├── Contracts/
│   │   └── PaymentProviderInterface.php
│   └── MockPaymentProvider.php
│
└── Filament/Resources/
    ├── InstructorBalanceResource.php
    └── InstructorBalanceResource/
        └── Pages/
            ├── ListInstructorBalances.php
            └── ViewInstructorBalance.php

database/
├── migrations/
│   ├── create_subscriptions_table.php
│   ├── create_course_accesses_table.php
│   ├── create_daily_allocations_table.php
│   ├── create_instructor_balances_table.php
│   ├── create_payouts_table.php
│   ├── create_refunds_table.php
│   └── create_platform_accounts_table.php
├── factories/
│   ├── SubscriptionFactory.php
│   ├── CourseAccessFactory.php
│   └── PayoutFactory.php
└── seeders/
    └── DemoSeeder.php

tests/
├── Unit/
│   ├── RevenueAllocationTest.php
│   └── RoundingTest.php
└── Feature/
    ├── PayoutIdempotencyTest.php
    ├── JobRetryTest.php
    └── ProviderTimeoutTest.php

docs/
├── ARCHITECTURE.md
├── AI_USAGE.md
└── README.md
```

---

## Database Schema

### subscriptions
```sql
id                  BIGINT UNSIGNED PK
student_id          BIGINT UNSIGNED (FK to users)
plan                ENUM('monthly', 'quarterly', 'annual')
amount_paid         INT UNSIGNED (cents)
starts_at           DATE
ends_at             DATE
status              ENUM('active', 'cancelled', 'refunded') DEFAULT 'active'
cancelled_at        TIMESTAMP NULL
created_at / updated_at
```

### course_accesses
```sql
id                  BIGINT UNSIGNED PK
student_id          BIGINT UNSIGNED
course_id           BIGINT UNSIGNED
instructor_id       BIGINT UNSIGNED
accessed_at         DATE
UNIQUE(student_id, course_id, accessed_at)  ← one record per day per course
```

### daily_allocations
```sql
id                  BIGINT UNSIGNED PK
subscription_id     BIGINT UNSIGNED FK
instructor_id       BIGINT UNSIGNED
course_id           BIGINT UNSIGNED
date                DATE
amount              INT UNSIGNED (cents)
status              ENUM('pending', 'paid', 'reversed') DEFAULT 'pending'
UNIQUE(subscription_id, instructor_id, course_id, date)  ← idempotency key
created_at / updated_at
```

### instructor_balances
```sql
id                  BIGINT UNSIGNED PK
instructor_id       BIGINT UNSIGNED UNIQUE
total_earned        INT UNSIGNED DEFAULT 0 (cents)
total_paid          INT UNSIGNED DEFAULT 0 (cents)
created_at / updated_at
```

### payouts
```sql
id                  BIGINT UNSIGNED PK
instructor_id       BIGINT UNSIGNED
amount              INT UNSIGNED (cents)
status              ENUM('pending', 'processing', 'paid', 'failed') DEFAULT 'pending'
idempotency_key     VARCHAR(255) UNIQUE  ← e.g. instructor_1_2024_01
provider_transaction_id VARCHAR(255) NULL
provider_status     VARCHAR(255) NULL
attempts            INT UNSIGNED DEFAULT 0
last_attempted_at   TIMESTAMP NULL
paid_at             TIMESTAMP NULL
created_at / updated_at
```

### refunds
```sql
id                  BIGINT UNSIGNED PK
subscription_id     BIGINT UNSIGNED FK
amount              INT UNSIGNED (cents)
reason              TEXT
fault               ENUM('platform', 'instructor')
handled_by          BIGINT UNSIGNED (FK to users/admin)
created_at / updated_at
```

### platform_accounts
```sql
id                  BIGINT UNSIGNED PK
type                ENUM('revenue_cut', 'rounding_remainder', 'refund_absorption')
amount              INT UNSIGNED (cents)
reference_id        BIGINT UNSIGNED NULL (polymorphic reference)
reference_type      VARCHAR(255) NULL
created_at
```

---

## Key Business Rules to Implement

1. **Platform cut**: configurable via `PLATFORM_REVENUE_CUT_PERCENTAGE` env (default 30%)
2. **Instructor share**: proportional to courses accessed that day out of total courses accessed
3. **Rounding remainders**: go to platform_accounts with type `rounding_remainder`
4. **Daily allocation**: runs as a scheduled job every day at midnight
5. **Monthly payout**: Artisan command `php artisan payouts:run` dispatches one job per instructor
6. **Idempotency**: `daily_allocations` unique key prevents duplicate allocation; `payouts.idempotency_key` prevents duplicate payouts
7. **Payout status machine**: pending → processing → paid (never goes backwards)
8. **Provider timeout handling**: store `provider_transaction_id` before calling provider, check status on retry

---

## MockPaymentProvider Behavior

```php
// Randomly does one of three things:
// 1. SUCCESS   — returns transaction ID immediately
// 2. FAILURE   — throws PaymentFailedException
// 3. TIMEOUT   — sleeps, throws TimeoutException (but money already moved)
//                checkStatus(transactionId) returns the real result
```

---

## Filament Screen (Read-Only)

- List all instructors with: total_earned, total_paid, outstanding balance
- Click into an instructor → see full payout history with status badges
- No create/edit/delete actions

---

## Tests Required (Pest)

1. `RevenueAllocationTest` — correct math, platform cut, rounding
2. `RoundingTest` — remainder goes to platform account, books balance
3. `PayoutIdempotencyTest` — running payout twice never double-pays
4. `JobRetryTest` — retried job never double-pays
5. `ProviderTimeoutTest` — timeout + retry resolves correctly via status check

---

## Docker Compose (Optional)

```yaml
services:
  app:
    build: .
    ports: ["8000:8000"]
  mysql:
    image: mysql:8
    environment:
      MYSQL_DATABASE: instructor_revenue
      MYSQL_ROOT_PASSWORD: secret
  queue:
    command: php artisan queue:work
```
