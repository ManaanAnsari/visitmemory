# Phase 6: Pilot Hardening and Launch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the system operable for a 5-store, 30-day pilot: observable, rate-limited, backup-restorable, deployable, and instrumented against the beta scorecard in `docs/business.md` §9.

**Architecture:** Production runs the existing FastAPI app and Celery worker behind Caddy in Docker Compose. Structured JSON logs carry a request id and redact secrets. Prometheus metrics and readiness checks expose health. Match feedback from staff confirm/reject actions feeds a pilot scorecard page. Retention jobs delete crops and raw receipts on a schedule.

**Tech Stack:** Docker Compose, Caddy, FastAPI, Celery, PostgreSQL 16, Redis 7, prometheus-client, structlog, pytest, shell runbooks.

**Spec:** `docs/business.md` §9; `docs/tech.md` §10; `guidelines/SECURITY_GUIDELINES.md`; `guidelines/MALWARE_CLEANUP_GUIDELINES.md`; all prior phase plans.

## Global Constraints

- Exact dependency pins only (`==`); container images use explicit version tags, never `latest`.
- No CI workflow changes in this phase. Security checks run as local scripts.
- No default secrets. Missing production secrets fail startup.
- Logs never contain passwords, tokens, API keys, raw image bytes, or full provider responses.
- Metrics never carry customer identifiers; labels are limited to shop slug and outcome.
- Backups are encrypted at rest and restoration is tested, not assumed.
- The pilot scorecard reads production data; it is admin-only.
- Retention defaults: visit crops 30 days, raw receipts 30 days, provisional profiles 24 hours.
- `pip-audit` reports zero known critical or high vulnerabilities before launch.

## File Structure

```
backend/app/
  observability/logging.py      JSON logging, request id, redaction
  observability/metrics.py      Prometheus counters/histograms
  observability/middleware.py   request id + metrics middleware
  api/health.py                 /health, /ready
  api/admin_pilot.py            scorecard page + csv
  models/feedback.py            MatchFeedback
  services/pilot.py             scorecard queries
  tasks/retention.py            crop/receipt retention jobs
  tasks/celery_app.py           beat schedule additions
  api/visits.py                 visit rate limit
  api/receipts.py               OCR rate limit
  api/telegram.py               webhook rate limit + feedback capture
  alembic/versions/0007_feedback.py
  tests/test_logging.py
  tests/test_ready.py
  tests/test_metrics.py
  tests/test_rate_limits_api.py
  tests/test_feedback.py
  tests/test_pilot_scorecard.py
  tests/test_retention.py

deploy/
  docker-compose.prod.yml
  Caddyfile
  env.production.example
  scripts/backup.sh
  scripts/restore.sh
  scripts/security_check.sh
  runbooks/deploy.md
  runbooks/restore.md
  runbooks/incident.md
  runbooks/pilot_onboarding.md
```

## Interfaces

- `configure_logging(level: str) -> None`
- `RequestContextMiddleware` sets `request.state.request_id` and logs one completion line
- `METRICS` registry with `visits_ingested_total{outcome}`, `visits_rejected_total{reason}`, `alerts_sent_total{channel,outcome}`, `ocr_requests_total{outcome}`, `http_request_duration_seconds{route,method,status}`
- `GET /ready -> 200 {"status":"ready"} | 503 {"status":"not_ready","failed":[...]}`
- `MatchFeedback(id, visit_id, shop_id, action, actor, created_at, updated_at)`, unique on `visit_id` so the latest staff decision replaces the previous one
- `build_pilot_scorecard(db, period_start, period_end) -> PilotScorecard`
- `PilotScorecard(period, stores_active, camera_viability_pct, match_precision_pct, alerts_per_store_week, tags_per_store_week, paid_shops, total_shops, paid_conversion_pct)`
- `purge_old_visit_crops(db, storage, now, retention_days=30) -> int`
- `purge_old_receipt_images(db, storage, now, retention_days=30) -> int`

## Review Focus

1. **Observability leaks a secret.** A 500 log line includes the Authorization header, bot token, or OCR body. Expected: redaction at the formatter, tested.
2. **Backup that cannot restore.** The dump script runs but the restore path was never exercised, or R2 objects are missing. Expected: restore runbook tested against a scratch database.
3. **Pilot metrics mislead.** Match precision counts unconfirmed matches as true positives, or a store with no feedback is treated as 100 percent precise. Expected: explicit denominators and "insufficient data" states.
4. **Retention deletes evidence.** A crop needed for a dispute or a feedback row is deleted while its purchase remains. Expected: retention deletes only image objects and visit snapshot keys, never customers or purchases.
5. **Rate limit blocks the store.** A busy entrance exhausts the visit limit and the agent dead-letters real visits. Expected: limits sized above 2 FPS best-shot throughput and tested at the boundary.

---

### Task 1: Structured logging with request IDs and redaction

**Files:**
- Create: `backend/app/observability/logging.py`
- Create: `backend/app/observability/middleware.py`
- Modify: `backend/app/main.py`
- Modify: `backend/app/config.py`
- Create: `backend/tests/test_logging.py`

**Interfaces:**
- Produces `configure_logging(level: str) -> None` and `RequestContextMiddleware`.
- Every request logs `request_id`, `method`, `path`, `status_code`, `duration_ms`, and never query strings or headers.

- [ ] **Step 1: Write the failing tests**

```python
def test_request_log_contains_request_id_and_no_authorization(caplog):
    response = client.get("/health", headers={"Authorization": "Bearer vm_live_secret"})
    line = caplog.records[-1].getMessage()
    assert "vm_live_secret" not in line
    assert "request_id" in line


def test_formatter_redacts_known_secret_keys():
    record = {"event": "error", "api_key": "vm_live_secret", "bot_token": "123:abc"}
    assert "vm_live_secret" not in format_record(record)
    assert "123:abc" not in format_record(record)


def test_traceback_is_redacted():
    ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_logging.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement logging**

Use `structlog` (pinned) with its stdlib integration and a JSON renderer. Generate `request_id` from an inbound `X-Request-ID` only when it matches `^[A-Za-z0-9-]{8,64}$`, otherwise generate a UUID4. Redact keys containing `password`, `token`, `secret`, `authorization`, `api_key`, and `cookie` at every nesting level. Log exceptions with type and message only; no request bodies. `X-Request-ID` is returned to the caller.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_logging.py -v`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add structured logging with request ids and secret redaction"
```

---

### Task 2: Readiness checks and Prometheus metrics

**Files:**
- Create: `backend/app/observability/metrics.py`
- Create: `backend/app/api/health.py` (move/extend existing health)
- Modify: `backend/app/main.py`
- Modify: `backend/app/api/visits.py`
- Modify: `backend/app/api/receipts.py`
- Modify: `backend/app/messaging/telegram.py`
- Create: `backend/tests/test_ready.py`
- Create: `backend/tests/test_metrics.py`

**Interfaces:**
- Produces `/ready`, `/metrics`, and the counters listed in Interfaces above.

- [ ] **Step 1: Write the failing tests**

```python
async def test_ready_returns_200_when_db_and_redis_are_up(client): ...
async def test_ready_returns_503_and_names_failed_dependency(client, broken_db): ...
async def test_metrics_exposes_visit_counter(client, ingested_visit): ...
async def test_metrics_labels_do_not_include_customer_id(client): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_ready.py tests/test_metrics.py -v`
Expected: FAIL — routes missing.

- [ ] **Step 3: Implement readiness and metrics**

`/ready` executes `SELECT 1` and `PING` with a 1-second timeout each; returns `503` with the failed dependency names, never exception text. `/metrics` uses `prometheus-client` and is not admin-protected in the MVP but binds only behind Caddy on an allowlisted path; document that scraping is private. Increment `visits_ingested_total` by outcome after commit. Timeout counters must not include shop slugs with spaces or user data.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_ready.py tests/test_metrics.py -v`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add readiness probes and prometheus metrics"
```

---

### Task 3: API rate limits at realistic store throughput

**Files:**
- Modify: `backend/app/api/visits.py`
- Modify: `backend/app/api/receipts.py`
- Modify: `backend/app/api/telegram.py`
- Create: `backend/tests/test_rate_limits_api.py`

**Interfaces:**
- Produces limits:
  - visits: `120` per minute per installation
  - OCR: `30` per minute per shop
  - Telegram webhook: `120` per minute per shop chat
- Every rejected request returns `429` with `Retry-After`.

- [ ] **Step 1: Write the failing tests**

```python
async def test_visit_limit_allows_120_and_rejects_121(...): ...
async def test_visit_limit_is_per_installation(...): ...
async def test_visit_429_includes_retry_after(...): ...
async def test_ocr_limit_is_per_shop(...): ...
async def test_webhook_limit_is_per_chat(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_rate_limits_api.py -v`
Expected: FAIL — limits not applied.

- [ ] **Step 3: Apply limits**

Reuse the `RateLimiter` from Phase 1. The visit limit is sized above the worst case of one qualified upload per 1.5-second track window; exceeding it indicates a broken agent or abuse. Rate-limited requests write a **warning** log with installation id, never the API key. The agent's retry path handles `429` (it already retries).

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_rate_limits_api.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add per-installation and per-shop api rate limits"
```

---

### Task 4: Match feedback and pilot scorecard

**Files:**
- Create: `backend/app/models/feedback.py`
- Create: `backend/alembic/versions/0007_feedback.py`
- Create: `backend/app/services/pilot.py`
- Create: `backend/app/api/admin_pilot.py`
- Create: `backend/app/templates/billing/pilot.html`
- Modify: `backend/app/api/telegram.py`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_feedback.py`
- Create: `backend/tests/test_pilot_scorecard.py`

**Interfaces:**
- Produces `MatchFeedback`, scorecard queries, `GET /admin/pilot?start=&end=`, `GET /admin/pilot.csv`.
- `confirm` callback writes `action="CONFIRMED"`; `reject` writes `action="REJECTED"`; `ignore` writes `action="STAFF_IGNORED"`.

- [ ] **Step 1: Write the failing tests**

```python
async def test_confirm_callback_writes_confirmed_feedback(...): ...
async def test_reject_callback_writes_rejected_feedback(...): ...
async def test_feedback_is_idempotent_per_visit_and_latest_action_wins(...): ...
async def test_precision_uses_confirmed_over_confirmed_plus_rejected(...): ...
async def test_precision_is_none_with_no_feedback(...): ...
async def test_camera_viability_counts_stores_with_weekly_accepted_visits(...): ...
async def test_tags_per_store_week_counts_staff_tag_actions(...): ...
async def test_paid_conversion_uses_shops_with_paid_invoice_in_period(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_feedback.py tests/test_pilot_scorecard.py -v`
Expected: FAIL — models and service missing.

- [ ] **Step 3: Implement feedback and scorecard**

Table `match_feedback(id, visit_id, shop_id, action, actor, created_at, updated_at)` with unique `visit_id`. A newer confirm/reject updates the row and timestamp; the latest decision is the label used by the scorecard. Scorecard:
- `stores_active`: shops with at least one visit in the period.
- `camera_viability_pct`: active shops with at least four distinct days containing an accepted visit, divided by active shops.
- `match_precision_pct`: `CONFIRMED / (CONFIRMED + REJECTED)`, or `None` when the denominator is zero.
- `alerts_per_store_week`: alerts dispatched divided by active shop-weeks.
- `tags_per_store_week`: staff tag actions divided by active shop-weeks.
- `paid_conversion_pct`: shops with a paid invoice in the period divided by active shops.

The page shows each metric with its denominator and a muted "insufficient data" state when the denominator is zero. CSV columns mirror the page.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_feedback.py tests/test_pilot_scorecard.py -v`
Expected: PASS (8 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add match feedback and pilot scorecard"
```

---

### Task 5: Retention jobs for crops and receipt images

**Files:**
- Create: `backend/app/tasks/retention.py`
- Modify: `backend/app/tasks/celery_app.py`
- Modify: `backend/app/config.py`
- Create: `backend/tests/test_retention.py`

**Interfaces:**
- Produces `purge_old_visit_crops`, `purge_old_receipt_images`, and beat entries at `03:00 UTC` daily.

- [ ] **Step 1: Write the failing tests**

```python
async def test_visit_crop_older_than_retention_is_deleted(...): ...
async def test_visit_crop_younger_than_retention_is_kept(...): ...
async def test_receipt_image_older_than_retention_is_deleted(...): ...
async def test_purchase_and_visit_rows_are_never_deleted(...): ...
async def test_retention_is_idempotent(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_retention.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement retention**

Delete only storage objects and null the corresponding `snapshot_key`/`raw_object_key`; never delete `visits`, `transactions`, `customers`, or `match_feedback`. Process in batches of 500 and log counts without identifiers. Defaults from settings: `VISIT_RETENTION_DAYS=30`, `RECEIPT_RETENTION_DAYS=30`. The provisional purge job from Phase 3 remains separate and runs at `02:00 UTC`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_retention.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add image retention jobs"
```

---

### Task 6: Production compose, TLS, backups, and restore runbook

**Files:**
- Create: `deploy/docker-compose.prod.yml`
- Create: `deploy/Caddyfile`
- Create: `deploy/env.production.example`
- Create: `deploy/scripts/backup.sh`
- Create: `deploy/scripts/restore.sh`
- Create: `deploy/runbooks/deploy.md`
- Create: `deploy/runbooks/restore.md`
- Create: `deploy/runbooks/incident.md`

**Interfaces:**
- Produces a deployable stack: `caddy`, `api`, `worker`, `beat`, `postgres`, `redis`.
- Backups: nightly `pg_dump -Fc` plus a storage sync; 14-day retention; restore script verifies row counts.

- [ ] **Step 1: Write the failing verification script test**

Create `backend/tests/test_deploy_config.py`:
```python
def test_prod_compose_has_no_latest_tags_and_no_privileged(): ...
def test_prod_compose_requires_secret_env_names(): ...
def test_caddyfile_uses_tls_and_hides_metrics(): ...
def test_backup_script_uses_strict_mode_and_fails_on_error(): ...
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && pytest tests/test_deploy_config.py -v`
Expected: FAIL — deploy files missing.

- [ ] **Step 3: Implement deployment**

`docker-compose.prod.yml`:
- images pinned (e.g., `postgres:16.4-alpine`, `redis:7.4-alpine`, `caddy:2.8-alpine`)
- API and worker run as non-root, `read_only: true` with `tmpfs` for `/tmp`
- healthchecks for Postgres, Redis, API
- secrets injected from `deploy/.env.production` with `0600`
- no published database or Redis ports
- Caddy terminates TLS, proxies `/api`, `/admin`, `/health`, `/ready`, and denies `/metrics` except from the private monitoring CIDR.

`backup.sh` uses `set -euo pipefail`, `pg_dump -Fc`, uploads to R2 with `aws s3 cp`, and writes a manifest with SHA-256. `restore.sh` restores into a scratch database, verifies table counts, and refuses to overwrite a non-empty production database without `--force`.

- [ ] **Step 4: Run tests and a dry run**

Run: `cd backend && pytest tests/test_deploy_config.py -v`
Run: `bash deploy/scripts/backup.sh --dry-run`
Expected: PASS; dry run prints planned commands and exits 0.

- [ ] **Step 5: Commit**

```bash
git add deploy backend/tests/test_deploy_config.py
git commit -m "feat: add production compose, tls, backup, and restore runbooks"
```

---

### Task 7: Telegram webhook registration and pilot onboarding runbook

**Files:**
- Modify: `backend/app/cli.py`
- Create: `deploy/runbooks/pilot_onboarding.md`
- Create: `backend/tests/test_telegram_cli.py`

**Interfaces:**
- Produces `python -m app.cli set-telegram-webhook --url https://...` and `python -m app.cli delete-telegram-webhook`.
- Runbook steps for distributor creation, shop creation, activation code, agent install, Telegram group id, test walk-through.

- [ ] **Step 1: Write the failing tests**

```python
def test_set_webhook_posts_secret_token_and_allowed_updates(...): ...
def test_webhook_cli_refuses_http_url(): ...
def test_webhook_cli_never_prints_bot_token(caplog): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_telegram_cli.py -v`
Expected: FAIL — CLI missing.

- [ ] **Step 3: Implement CLI and runbook**

`setWebhook` sends `url`, `secret_token` from settings, and `allowed_updates=["message", "callback_query"]`. Reject non-HTTPS URLs. The runbook includes a 15-minute checklist and a rollback section that revokes the installation and deletes the activation code.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_telegram_cli.py -v`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add backend deploy/runbooks/pilot_onboarding.md
git commit -m "feat: add telegram webhook cli and pilot onboarding runbook"
```

---

### Task 8: Security verification and pilot dry run

**Files:**
- Create: `deploy/scripts/security_check.sh`
- Create: `backend/tests/test_pilot_dry_run.py`

**Interfaces:**
- Produces one script that fails on any of:
  - unpinned Python requirements
  - `npm`/Node assets present
  - committed `.env` files
  - secrets matching `vm_live_`, `sk_live_`, or Telegram token patterns in tracked files
  - `pip-audit` critical/high findings
  - missing security headers in a live `TestClient` request

- [ ] **Step 1: Write the failing dry-run test**

```python
async def test_five_synthetic_stores_complete_pilot_flow(...):
    # Seed 5 shops across 2 distributors.
    # Simulate 30 days of visits, receipts, invoice payments.
    # Assert scorecard denominators are correct and every commission total reconciles.
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && pytest tests/test_pilot_dry_run.py -v`
Expected: FAIL — synthetic fixtures missing.

- [ ] **Step 3: Implement the dry run and security script**

Dry run uses fakes only. It creates deterministic data (fixed clock) and asserts:
- 5 active stores, 2 distributors
- expected commission entries for each paid month
- scorecard precision with known confirmations and rejections
- no unlinked transaction is counted in spend

`security_check.sh` uses `set -euo pipefail`, runs `pip-audit -r backend/requirements.txt`, greps tracked files for secret patterns with `git grep`, and runs the TestClient header test.

- [ ] **Step 4: Run the full suite and security check**

Run: `cd backend && pytest -v`
Run: `cd pos_helper && pytest -v`
Run: `cd edge && pytest -v`
Run: `bash deploy/scripts/security_check.sh`
Expected: all PASS, zero critical/high audit findings.

- [ ] **Step 5: Commit**

```bash
git add backend deploy/scripts/security_check.sh
git commit -m "test: add pilot dry run and security verification script"
```

---

## Phase 6 Exit Checklist

- [ ] `/ready` reports dependency health; `/metrics` exposes visit, alert, OCR, and HTTP metrics without customer identifiers.
- [ ] Logs are JSON, carry a request id, and redact secrets in tests.
- [ ] A 120 vis/min installation load does not 429 under normal 2 FPS best-shot operation; abuse does.
- [ ] The pilot scorecard shows denominators and "insufficient data" states rather than misleading percentages.
- [ ] Retention jobs delete images only, never purchases or feedback.
- [ ] A backup restores into a scratch database with matching row counts.
- [ ] The five-store dry run reconciles commissions, invoices, and scorecard metrics.
- [ ] `security_check.sh` passes with zero critical/high dependency findings.
