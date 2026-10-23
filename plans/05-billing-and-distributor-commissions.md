# Phase 5: Billing and Distributor Commissions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Record shop subscriptions and paid invoices, compute each distributor's commission and activation bounty exactly once per month, track payouts, and export auditable distributor statements.

**Architecture:** Plans, subscriptions, and invoices are platform records. A pure commission calculator converts paid invoice cents and a distributor rate snapshot into commission cents. An idempotent monthly run creates one commission entry per shop per period. Payouts group unpaid entries, and statements are a read model over entries and payouts. Stripe is an optional adapter behind `PaymentProvider`; manual invoice marking is the default for the pilot.

**Tech Stack:** FastAPI, SQLAlchemy 2.0 async, PostgreSQL 16, Jinja2 admin, stdlib HMAC for Stripe webhooks, pytest.

**Spec:** `docs/business.md` §4, §6.2; `docs/tech.md` §9; `plans/01-platform-foundation-and-admin.md`.

## Global Constraints

- Exact dependency pins only (`==`).
- All money is integer cents. No floats, no `Decimal` in the database; `Decimal` may only appear inside a test assertion helper.
- Commission rounding is half-up on integer cents: `(gross_cents * rate_bps + 5000) // 10000`.
- Default distributor values come from `docs/business.md` §6.2: `activation_bounty_cents=2000`, `commission_rate_bps=2500`, both snapshot into the commission entry.
- Periods are UTC calendar months: `period_start` is the first day at 00:00 UTC; `period_end` is the first day of the following month.
- One commission entry per `(shop_id, period_start)` and one invoice per `(shop_id, period_start)`.
- Bounty is paid exactly once per shop, on the first paid invoice in the shop's history.
- Every invoice, commission, payout, and void action writes an audit row.
- Manual billing is the default; no payment processor is required to run the pilot.
- Stripe webhooks are verified with HMAC-SHA256 before any state change.

## File Structure

```
backend/app/
  models/billing.py              Plan, Subscription, Invoice, CommissionEntry, Payout, PayoutItem
  billing/base.py                PaymentProvider Protocol, CheckoutSession, PaymentEvent
  billing/manual.py              ManualPaymentProvider
  billing/stripe.py              StripePaymentProvider (REST + HMAC)
  billing/plans.py               seed defaults, assign_plan
  billing/invoices.py            create_invoice, mark_invoice_paid
  billing/commissions.py         calculate_commission, run_commission_period
  billing/payouts.py             create_payout, mark_payout_paid, build_statement
  api/admin_billing.py           plans, subscriptions, invoices
  api/admin_commissions.py       commission run, payouts, statements
  api/stripe_webhook.py          POST /api/v1/stripe/webhook
  templates/billing/plans.html
  templates/billing/subscriptions.html
  templates/billing/invoices.html
  templates/billing/commission_run.html
  templates/billing/payouts.html
  templates/billing/statement.html
  alembic/versions/0006_billing.py
  tests/test_billing_models.py
  tests/test_invoices.py
  tests/test_commissions.py
  tests/test_commission_run.py
  tests/test_payouts.py
  tests/test_statement.py
  tests/test_stripe_webhook.py
  tests/test_billing_e2e.py
```

## Interfaces

- `seed_default_plans(db) -> list[Plan]` with codes `starter` (3900 USD), `growth` (5900 USD), `enterprise` (9900 USD).
- `assign_plan(db, shop_id: UUID, plan_code: str, actor: AdminUser) -> Subscription`
- `create_invoice(db, subscription_id: UUID, period_start: date, actor: AdminUser) -> Invoice`
- `mark_invoice_paid(db, invoice_id: UUID, paid_at: datetime, provider: str, provider_ref: str | None, actor: AdminUser) -> Invoice`
- `calculate_commission(gross_cents: int, rate_bps: int) -> int`
- `run_commission_period(db, period_start: date, actor: AdminUser) -> CommissionRunResult`
- `create_payout(db, distributor_id: UUID, period_start: date, actor: AdminUser) -> Payout`
- `mark_payout_paid(db, payout_id: UUID, reference: str, actor: AdminUser) -> Payout`
- `build_statement(db, distributor_id: UUID, period_start: date) -> DistributorStatement`
- `PaymentProvider.create_checkout(invoice: Invoice) -> CheckoutSession`
- `PaymentProvider.verify_webhook(payload: bytes, signature_header: str) -> PaymentEvent`

## Review Focus

1. **Double payment.** Re-running the monthly job or retrying a webhook creates a second commission entry or invoice. Expected: unique constraints plus skip-if-exists behavior.
2. **Bounty repeated.** Every paid month pays the $20 activation bounty. Expected: bounty only on the shop's first paid invoice, tested across non-consecutive months.
3. **Rounding drift.** 25 percent of 39.99 produces inconsistent cents between runs or report lines. Expected: one integer formula used everywhere, boundary-tested.
4. **Payout without coverage.** A payout is marked paid while including entries from a future or already-paid period. Expected: only `ACCRUED` entries for the selected period are included, and totals reconcile.
5. **Forged Stripe event.** An unsigned or tampered payload marks an invoice paid. Expected: 400 with no state change.

---

### Task 1: Billing models, migration, and default plans

**Files:**
- Create: `backend/app/models/billing.py`
- Create: `backend/alembic/versions/0006_billing.py`
- Create: `backend/app/billing/plans.py`
- Create: `backend/tests/test_billing_models.py`

**Interfaces:**
- Produces tables:
  - `Plan(id, code, name, monthly_price_cents, currency, is_active, created_at)`
  - `Subscription(id, shop_id, plan_id, status, started_at, current_period_start, current_period_end, cancelled_at, created_at)`
  - `Invoice(id, shop_id, subscription_id, period_start, period_end, amount_cents, currency, status, paid_at, provider, provider_ref, created_at)`
  - `CommissionEntry(id, distributor_id, shop_id, invoice_id, period_start, gross_cents, rate_bps_snapshot, commission_cents, bounty_cents, status, created_at)`
  - `Payout(id, distributor_id, period_start, period_end, total_cents, status, paid_at, reference, created_at)`, unique on `(distributor_id, period_start)`
  - `PayoutItem(id, payout_id, commission_entry_id, created_at)`

- [ ] **Step 1: Write the failing tests**

```python
async def test_seed_default_plans_is_idempotent(db_session): ...
async def test_subscription_unique_per_shop(db_session): ...
async def test_invoice_unique_per_shop_and_period(db_session): ...
async def test_commission_unique_per_shop_and_period(db_session): ...
async def test_payout_item_cannot_reference_entry_twice(db_session): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_billing_models.py -v`
Expected: FAIL — models missing.

- [ ] **Step 3: Implement models and seed**

Migration 0006 creates the tables and constraints:
- `plans.code` unique
- `subscriptions.shop_id` unique
- `invoices (shop_id, period_start)` unique
- `commission_entries (shop_id, period_start)` unique
- `payouts (distributor_id, period_start)` unique
- `payout_items.commission_entry_id` unique

Statuses: subscription `ACTIVE|CANCELLED`; invoice `DRAFT|OPEN|PAID|VOID`; commission `ACCRUED|PAID|VOID`; payout `OPEN|PAID|VOID`. `seed_default_plans` inserts only missing codes. Plans are USD; shop currency may differ but platform billing remains USD for the MVP.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_billing_models.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add billing models and default subscription plans"
```

---

### Task 2: Subscription assignment, manual invoices, and paid state

**Files:**
- Create: `backend/app/billing/invoices.py`
- Create: `backend/app/api/admin_billing.py`
- Create: `backend/app/templates/billing/plans.html`
- Create: `backend/app/templates/billing/subscriptions.html`
- Create: `backend/app/templates/billing/invoices.html`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_invoices.py`

**Interfaces:**
- Produces `assign_plan`, `create_invoice`, `mark_invoice_paid`, and admin routes:
  - `GET /admin/billing/plans`
  - `GET /admin/billing/subscriptions`
  - `POST /admin/shops/{id}/subscription`
  - `POST /admin/shops/{id}/invoices`
  - `POST /admin/invoices/{id}/mark-paid`

- [ ] **Step 1: Write the failing tests**

```python
async def test_assign_plan_creates_active_subscription(...): ...
async def test_assigning_new_plan_replaces_active_subscription_with_audit(...): ...
async def test_invoice_amount_comes_from_plan(...): ...
async def test_marking_invoice_paid_is_idempotent(...): ...
async def test_cannot_mark_void_invoice_paid(...): ...
async def test_invoice_pages_require_admin(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_invoices.py -v`
Expected: FAIL — modules and routes missing.

- [ ] **Step 3: Implement billing operations**

`create_invoice` computes `period_end` as the next month, copies `plan.monthly_price_cents` and currency, and creates the invoice with `status=OPEN`. `mark_invoice_paid` sets `status=PAID`, `paid_at` (default now UTC), `provider` (`manual` or `stripe`), and `provider_ref`, then returns the invoice unchanged if already paid. Because subscriptions are unique per shop, `assign_plan` updates the existing subscription's `plan_id` and start period rather than inserting a second row. All operations write audit rows with invoice id and amount.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_invoices.py -v`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add subscriptions, invoices, and manual paid marking"
```

---

### Task 3: Pure commission calculator

**Files:**
- Create: `backend/app/billing/commissions.py`
- Create: `backend/tests/test_commissions.py`

**Interfaces:**
- Produces `calculate_commission(gross_cents, rate_bps) -> int`.

- [ ] **Step 1: Write the failing tests**

```python
def test_25_percent_of_3900_is_975():
    assert calculate_commission(3900, 2500) == 975


def test_rounds_half_up_at_exact_half_cent():
    assert calculate_commission(1, 5000) == 1
    assert calculate_commission(3, 5000) == 2


def test_zero_rate_is_zero():
    assert calculate_commission(3900, 0) == 0


def test_full_rate_returns_gross():
    assert calculate_commission(3900, 10000) == 3900


def test_rejects_negative_inputs():
    with pytest.raises(ValueError):
        calculate_commission(-1, 2500)


def test_rejects_rate_above_10000():
    with pytest.raises(ValueError):
        calculate_commission(100, 10001)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_commissions.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement calculator**

```python
def calculate_commission(gross_cents: int, rate_bps: int) -> int:
    if gross_cents < 0:
        raise ValueError("gross_cents must be non-negative")
    if not 0 <= rate_bps <= 10000:
        raise ValueError("rate_bps must be between 0 and 10000")
    return (gross_cents * rate_bps + 5000) // 10000
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_commissions.py -v`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/billing/commissions.py backend/tests/test_commissions.py
git commit -m "feat: add integer half-up commission calculator"
```

---

### Task 4: Idempotent monthly commission run

**Files:**
- Modify: `backend/app/billing/commissions.py`
- Create: `backend/app/api/admin_commissions.py`
- Create: `backend/app/templates/billing/commission_run.html`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_commission_run.py`

**Interfaces:**
- Produces:
  - `CommissionRunResult(period_start, entries_created, entries_skipped, total_gross_cents, total_commission_cents, total_bounty_cents)`
  - `run_commission_period(db, period_start: date, actor: AdminUser) -> CommissionRunResult`
  - Admin route `POST /admin/billing/commission-runs` with `period_start`.

- [ ] **Step 1: Write the failing tests**

```python
async def test_run_creates_entry_for_paid_invoice_with_distributor(...): ...
async def test_run_skips_shop_without_distributor(...): ...
async def test_run_skips_unpaid_invoice(...): ...
async def test_run_is_idempotent_on_repeat(...): ...
async def test_bounty_paid_only_on_first_paid_invoice(...): ...
async def test_bounty_not_paid_on_second_nonconsecutive_month(...): ...
async def test_rate_snapshot_survives_distributor_rate_change(...): ...
async def test_run_rejects_future_period(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_commission_run.py -v`
Expected: FAIL — function missing.

- [ ] **Step 3: Implement the run**

Reject `period_start` later than the current UTC month. Query `PAID` invoices where `period_start == period_start` and the shop has `distributor_id`. For each, skip if a commission entry exists. Determine first invoice by checking whether any earlier `PAID` invoice exists for the shop; the current invoice counts as first only when none exists. Snapshot `rate_bps_snapshot = distributor.commission_rate_bps` and `bounty_cents = distributor.activation_bounty_cents` when first, else `0`. Insert `ACCRUED` entry. Commit per batch of 100. Audit one summary row with counts and totals.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_commission_run.py -v`
Expected: PASS (8 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add idempotent monthly distributor commission run"
```

---

### Task 5: Payout ledger, admin UI, and CSV export

**Files:**
- Create: `backend/app/billing/payouts.py`
- Create: `backend/app/templates/billing/payouts.html`
- Modify: `backend/app/api/admin_commissions.py`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_payouts.py`

**Interfaces:**
- Produces:
  - `create_payout(db, distributor_id, period_start, actor) -> Payout`
  - `mark_payout_paid(db, payout_id, reference, actor) -> Payout`
  - `void_payout(db, payout_id, reason, actor) -> Payout`
  - Admin routes `GET /admin/billing/payouts`, `POST /admin/billing/payouts`, `POST /admin/billing/payouts/{id}/mark-paid`, `POST /admin/billing/payouts/{id}/void`
  - `GET /admin/billing/payouts.csv?period_start=...`

- [ ] **Step 1: Write the failing tests**

```python
async def test_payout_includes_only_accrued_entries_for_period(...): ...
async def test_payout_total_matches_entries(...): ...
async def test_marking_payout_paid_marks_entries_paid(...): ...
async def test_duplicate_payout_for_same_distributor_and_period_is_rejected(...): ...
async def test_void_payout_returns_entries_to_accrued(...): ...
async def test_csv_contains_one_row_per_shop_with_correct_totals(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_payouts.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement payouts**

`create_payout` locks eligible `ACCRUED` entries for the period, rejects when none exist, computes `total_cents = sum(commission + bounty)`, creates `Payout` and `PayoutItem` rows. It does not mark entries paid until `mark_payout_paid`, which sets `status=PAID`, `paid_at`, reference, and updates entries to `PAID` atomically. `void_payout` sets payout `VOID` and entries back to `ACCRUED`, and is rejected if already paid. CSV streams with `text/csv` and a header row.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_payouts.py -v`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add distributor payout ledger and csv export"
```

---

### Task 6: Distributor statement

**Files:**
- Create: `backend/app/templates/billing/statement.html`
- Modify: `backend/app/billing/payouts.py`
- Modify: `backend/app/api/admin_commissions.py`
- Create: `backend/tests/test_statement.py`

**Interfaces:**
- Produces:
  - `DistributorStatement(distributor, period_start, entries, total_gross_cents, total_commission_cents, total_bounty_cents, payout)`
  - `build_statement(db, distributor_id, period_start) -> DistributorStatement`
  - Routes `GET /admin/billing/distributors/{id}/statement?period_start=` and `.../statement.csv`

- [ ] **Step 1: Write the failing tests**

```python
async def test_statement_lists_each_shop_and_payout_state(...): ...
async def test_statement_totals_reconcile_with_entries(...): ...
async def test_statement_csv_has_stable_columns(...): ...
async def test_statement_for_period_with_no_entries_is_empty_not_404(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_statement.py -v`
Expected: FAIL — statement missing.

- [ ] **Step 3: Implement statement**

Read-only. Columns: distributor code, shop slug, shop name, invoice amount, rate, commission, bounty, entry status, payout status. Totals are sums of entries. The CSV uses the same column order as the HTML table and ISO dates.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_statement.py -v`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add distributor statement view and csv"
```

---

### Task 7: Stripe Checkout adapter and webhook

**Files:**
- Create: `backend/app/billing/base.py`
- Create: `backend/app/billing/manual.py`
- Create: `backend/app/billing/stripe.py`
- Create: `backend/app/api/stripe_webhook.py`
- Modify: `backend/app/config.py`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_stripe_webhook.py`

**Interfaces:**
- Produces `ManualPaymentProvider`, `StripePaymentProvider`, `POST /api/v1/stripe/webhook`.
- Settings: `stripe_secret_key`, `stripe_webhook_secret`, both optional; when absent, Stripe routes return `404` and manual billing remains available.

- [ ] **Step 1: Write the failing tests**

```python
def test_valid_signature_is_accepted(): ...
def test_tampered_payload_is_rejected(): ...
def test_expired_timestamp_is_rejected(): ...
async def test_checkout_completed_marks_invoice_paid_once(...): ...
async def test_duplicate_webhook_is_idempotent(...): ...
async def test_amount_mismatch_does_not_mark_paid_and_flags_invoice(...): ...
async def test_stripe_disabled_returns_404(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_stripe_webhook.py -v`
Expected: FAIL — adapter missing.

- [ ] **Step 3: Implement Stripe adapter**

Verify `Stripe-Signature` as `t=<unix>,v1=<hex>` with `hmac.new(secret, f"{t}.{payload}".encode(), sha256)` using `constant_time_compare`; reject timestamps older than 300 seconds. Handle `checkout.session.completed`: resolve invoice by `client_reference_id`, compare `amount_total` and `currency`, then call `mark_invoice_paid(provider="stripe", provider_ref=session_id)`. A mismatch sets `invoice.status="OPEN"` and writes an audit row `stripe_amount_mismatch`. Duplicate events are idempotent. `create_checkout` calls the Stripe REST API directly with form encoding, not an SDK.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_stripe_webhook.py -v`
Expected: PASS (7 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add verified stripe checkout adapter and webhook"
```

---

### Task 8: End-to-end billing and commission test

**Files:**
- Create: `backend/tests/test_billing_e2e.py`

**Interfaces:**
- Consumes all Phase 1 and Phase 5 components.

- [ ] **Step 1: Write the failing test**

```python
async def test_paid_month_produces_correct_commission_and_statement(client, ...):
    # 1. Admin creates distributor at 25% and a shop assigned to it.
    # 2. Admin assigns the starter plan and creates an invoice for 2026-10-01.
    # 3. Admin marks the invoice paid.
    # 4. Commission run for 2026-10-01 creates 975 commission + 2000 bounty.
    # 5. Create payout, mark it paid.
    # 6. Statement totals equal 2975 and show the payout as PAID.
    # 7. Re-running the commission run and webhook adds nothing.
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && pytest tests/test_billing_e2e.py -v`
Expected: FAIL — integration path incomplete.

- [ ] **Step 3: Fix wiring**

No new abstractions. Keep fakes for any provider; the manual provider is sufficient.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && pytest tests/test_billing_e2e.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "test: add end-to-end invoice, commission, and payout flow"
```

---

## Phase 5 Exit Checklist

- [ ] A shop can be assigned a plan and invoiced for a UTC month.
- [ ] Marking an invoice paid twice creates no duplicate state.
- [ ] The monthly run creates exactly one entry per shop per period, with the correct 25 percent commission and a single first-invoice bounty.
- [ ] A payout includes only accrued entries for its period, and totals reconcile with the statement.
- [ ] Stripe and manual providers cannot both mark the same invoice paid with conflicting references.
- [ ] `pytest` passes offline; Stripe is exercised through signed fixture payloads.
