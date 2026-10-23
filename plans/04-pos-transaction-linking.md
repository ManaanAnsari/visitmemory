# Phase 4: POS Transaction Linking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Capture purchases without POS APIs, link each transaction to the correct counter-camera visit and customer, and keep customer lifetime spend and last-purchase context accurate.

**Architecture:** Transactions arrive through three independent paths behind one ingestion contract: a JSON endpoint for the Windows receipt helper, a Telegram photo flow with an OCR adapter, and an admin manual entry. A pure correlation service matches a transaction to the nearest counter-camera visit within a time window. Linking updates customer aggregates in the same transaction.

**Tech Stack:** FastAPI, SQLAlchemy 2.0 async, PostgreSQL 16, boto3 (R2), Gemini REST OCR behind an adapter, Telegram Bot API, Python 3.12 Windows helper, pytest.

**Spec:** `docs/tech.md` §4.3, §7, §7.4; `docs/business.md` §7.1; `plans/03-recognition-and-alerts.md`.

## Global Constraints

- Exact dependency pins only (`==`); no unpinned models or SDKs.
- Money is integer cents plus ISO-4217 currency. Never floats.
- Correlation window is `90` seconds before and after the transaction, configurable per shop.
- Transaction idempotency key is unique per shop; a retry never duplicates a purchase or double-counts spend.
- OCR is behind an `OcrExtractor` Protocol; tests use a fake and never call Gemini.
- The raw receipt image is stored through `ObjectStorage`; it is never logged.
- The print helper is read-only: it never writes to the POS database or executable.
- Manual linking is audited and reversible until the commission period closes (Phase 5).

## File Structure

```
backend/app/
  models/transaction.py          Transaction
  models/customer.py            add last_purchase_item, last_purchase_at
  api/transactions.py           JSON ingestion + manual link endpoints
  api/receipts.py               OCR upload endpoint
  services/transactions.py      ingest_transaction, link_transaction, aggregates
  services/correlation.py       find_counter_visit
  ocr/base.py                   OcrExtractor Protocol, ReceiptExtraction, FakeOcrExtractor
  ocr/gemini.py                 GeminiOcrExtractor
  api/telegram.py               receipt photo handler
  api/admin_transactions.py     unlinked list + manual link + purchase history
  templates/transactions/list.html
  templates/transactions/link.html
  templates/customers/detail.html (modify: purchases section)
  alembic/versions/0005_transactions.py
  tests/test_transaction_ingest.py
  tests/test_correlation.py
  tests/test_ocr.py
  tests/test_receipt_photo.py
  tests/test_admin_transactions.py
  tests/test_transaction_e2e.py

pos_helper/
  requirements.txt
  helper/__init__.py
  helper/config.py
  helper/escpos.py               parser
  helper/watcher.py              FolderDropSource (primary) + WindowsPrintQueueSource (experimental)
  helper/api_client.py
  helper/main.py
  tests/fixtures/receipt_01.escpos
  tests/test_escpos.py
  tests/test_watcher.py
  packaging/build_windows.ps1
```

## Interfaces

- `Transaction(id, shop_id, customer_id, visit_id, client_event_id, receipt_number, total_cents, currency, items, occurred_at, extraction_method, raw_object_key, created_at)`
- `ingest_transaction(db, *, installation, payload: TransactionCreate, raw_receipt: bytes | None) -> TransactionIngestResult`
- `find_counter_visit(db, shop_id: UUID, occurred_at: datetime, window_seconds: int = 90) -> Visit | None`
- `link_transaction(db, transaction: Transaction, visit: Visit, actor: str) -> Transaction`
- `apply_purchase_aggregates(db, customer: Customer, transaction: Transaction) -> None`
- `OcrExtractor.extract(image_jpeg: bytes) -> ReceiptExtraction`
- `ReceiptExtraction(receipt_number, total_cents, currency, items, occurred_at, confidence)`
- `TransactionIngestResult(transaction: Transaction, status: Literal["linked", "unlinked", "duplicate"])`
- `TransactionCreate(client_event_id, occurred_at, total_cents, currency, receipt_number, items, extraction_method)`

## Review Focus

1. **Duplicate purchase.** A spooler retry or Telegram redelivery double-counts lifetime spend and commissions. Expected: idempotency by `(shop_id, client_event_id)` and no aggregate change on duplicate.
2. **Wrong visit linked.** A transaction is linked to a customer who was not at the counter, poisoning alerts. Expected: only counter visits inside the window are candidates; ties resolve deterministically; no candidate leaves it unlinked.
3. **Currency mismatch.** A MYR receipt is added to a USD lifetime total. Expected: aggregates only when the transaction currency equals the shop currency; otherwise the transaction is stored and flagged.
4. **OCR misread.** A decimal or thousands separator is parsed as the wrong amount, or a total line is mistaken for an item. Expected: the parser rejects implausible totals and prefers the labelled total line.
5. **Partial link.** The transaction is linked but aggregates fail, or vice versa. Expected: linking and aggregates commit in one database transaction.

---

### Task 1: Transaction model and JSON ingestion

**Files:**
- Create: `backend/app/models/transaction.py`
- Modify: `backend/app/models/installation.py`
- Modify: `backend/app/models/customer.py`
- Create: `backend/alembic/versions/0005_transactions.py`
- Create: `backend/app/services/transactions.py`
- Create: `backend/app/api/transactions.py`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_transaction_ingest.py`

**Interfaces:**
- Produces models and `POST /api/v1/transactions` with `TransactionCreate` body:
```json
{
  "client_event_id": "uuid",
  "occurred_at": "2026-10-20T14:21:05+08:00",
  "receipt_number": "INV-1042",
  "total_cents": 58900,
  "currency": "MYR",
  "items": [{"name": "Floral Summer Dress", "price_cents": 58900, "sku": null}],
  "extraction_method": "PRINT_SPOOLER"
}
```
- Response `201`: `{"transaction_id": "...", "status": "linked|unlinked|duplicate", "visit_id": "..." | null}`.

- [ ] **Step 1: Write the failing tests**

```python
async def test_transaction_ingest_creates_unlinked_transaction(...): ...
async def test_duplicate_client_event_id_does_not_double_count(...): ...
async def test_transaction_without_counter_visit_is_unlinked(...): ...
async def test_transaction_rejects_non_positive_total(...): ...
async def test_transaction_rejects_unknown_currency(...): ...
async def test_transaction_requires_installation_auth(...): ...
async def test_pos_helper_installation_kind_is_recorded(...): ...
async def test_purge_keeps_customer_with_purchase(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_transaction_ingest.py -v`
Expected: FAIL — models and route missing.

- [ ] **Step 3: Implement model and ingestion**

Migration 0005 adds `customers.last_purchase_item`, `customers.last_purchase_at`, and the `transactions` table; it relies on `installations.kind` created in Phase 1. Transactions hold `items` as JSONB and may have null `customer_id`/`visit_id`. Unique index on `(shop_id, client_event_id)`. Validate currency against the shop currency for aggregation, but allow storing other currencies with `aggregate_applied=False`.

Ingestion: check duplicate, insert with `customer_id=NULL` and `visit_id=NULL`, commit, audit. Correlation and linking arrive in Tasks 2 and 3. Return `duplicate` with the existing transaction on conflict. Raw receipt bytes are stored only when provided by the multipart endpoint in Task 4.

Also update the Phase 3 purge query so a customer with any transaction is never purged, even when `display_name` is null, not VIP, and older than 24 hours.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_transaction_ingest.py -v`
Expected: PASS (8 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add transaction model and idempotent POS ingestion"
```

---

### Task 2: Counter-visit correlation service

**Files:**
- Create: `backend/app/services/correlation.py`
- Create: `backend/tests/test_correlation.py`

**Interfaces:**
- Produces `find_counter_visit(db, shop_id, occurred_at, window_seconds=90) -> Visit | None`.

- [ ] **Step 1: Write the failing tests**

```python
async def test_visit_at_exact_window_edge_is_eligible(...): ...
async def test_visit_outside_window_is_not_eligible(...): ...
async def test_entrance_visit_is_never_eligible(...): ...
async def test_unmatched_new_visitor_is_not_eligible(...): ...
async def test_nearest_visit_wins(...): ...
async def test_tie_resolves_to_latest_arrival(...): ...
async def test_shop_isolation_is_enforced(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_correlation.py -v`
Expected: FAIL — service missing.

- [ ] **Step 3: Implement correlation**

Query visits where `shop_id` matches, `camera_role='COUNTER'`, `customer_id IS NOT NULL`, `status IN ('matched', 'unconfirmed')`, and `arrival_time BETWEEN occurred_at - window AND occurred_at + window`. Order by `abs(arrival_time - occurred_at)` then `arrival_time DESC`, limit 1. Use UTC-aware timestamps only; reject naive datetimes with `ValueError`. Ingestion wiring arrives in Task 3.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_correlation.py -v`
Expected: PASS (7 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/correlation.py backend/tests/test_correlation.py
git commit -m "feat: add deterministic counter visit correlation"
```

---

### Task 3: Linking, aggregates, and last-purchase context

**Files:**
- Modify: `backend/app/services/transactions.py`
- Modify: `backend/app/recognition/service.py`
- Create: `backend/tests/test_transaction_aggregates.py`

**Interfaces:**
- Produces `link_transaction(db, transaction, visit, actor) -> Transaction` and `apply_purchase_aggregates(db, customer, transaction) -> None`.

- [ ] **Step 1: Write the failing tests**

```python
async def test_link_sets_customer_and_visit(...): ...
async def test_link_updates_lifetime_spend_and_last_purchase(...): ...
async def test_link_is_idempotent(...): ...
async def test_currency_mismatch_skips_aggregates(...): ...
async def test_manual_relink_reverses_previous_aggregates(...): ...
async def test_alert_after_purchase_uses_last_purchase_item(...): ...
async def test_ingest_links_counter_visit_after_correlation(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_transaction_aggregates.py -v`
Expected: FAIL — functions missing.

- [ ] **Step 3: Implement linking and aggregates**

On first link: set `customer_id`, `visit_id`, `aggregate_applied=True` only if currencies match, add `total_cents` to `lifetime_spend_cents`, set `last_purchase_item` to the first item name (prefer the highest price item) and `last_purchase_at` to `occurred_at`. On relink: subtract the old transaction total from the old customer, add to the new one, recompute `last_purchase_item` from the customer's newest remaining transaction. All in one `db.flush()`/commit with row locks (`SELECT ... FOR UPDATE`) on the affected customer rows. Wire `find_counter_visit` plus `link_transaction` into `ingest_transaction` so a transaction links on arrival.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_transaction_aggregates.py -v`
Expected: PASS (7 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: link purchases to customers and update lifetime aggregates"
```

---

### Task 4: OCR protocol, Gemini adapter, and receipt image storage

**Files:**
- Create: `backend/app/ocr/base.py`
- Create: `backend/app/ocr/gemini.py`
- Create: `backend/app/api/receipts.py`
- Create: `backend/tests/test_ocr.py`
- Modify: `backend/app/config.py`

**Interfaces:**
- Produces `OcrExtractor`, `ReceiptExtraction`, `FakeOcrExtractor`, `GeminiOcrExtractor`, `POST /api/v1/receipts/ocr`.

- [ ] **Step 1: Write the failing tests**

```python
def test_fake_extractor_returns_configured_receipt(): ...
def test_gemini_adapter_parses_json_response(): ...
def test_gemini_adapter_rejects_missing_total(): ...
def test_gemini_adapter_never_logs_api_key(caplog): ...
async def test_receipt_ocr_endpoint_stores_image_and_creates_transaction(...): ...
async def test_receipt_ocr_rejects_non_image(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_ocr.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement OCR**

`GeminiOcrExtractor` calls `POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent` with an inline JPEG part and a response schema requiring `receipt_number`, `total_cents`, `currency`, `occurred_at`, and `items[]`. Parse defensively: total must be a positive integer, currency must be three uppercase letters, and confidence below `0.5` returns a 422 to the caller. Model name and API key come from settings; missing key disables the adapter at startup rather than defaulting.

`POST /api/v1/receipts/ocr` accepts multipart `image`, stores it at `shops/{shop_id}/receipts/{uuid}.jpg`, runs OCR, then ingests with `extraction_method="BOT_PHOTO"`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_ocr.py -v`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add receipt OCR adapter and image ingestion endpoint"
```

---

### Task 5: Telegram receipt photo flow

**Files:**
- Modify: `backend/app/api/telegram.py`
- Modify: `backend/app/messaging/telegram.py`
- Create: `backend/tests/test_receipt_photo.py`

**Interfaces:**
- Consumes: Telegram webhook, `getFile`, OCR endpoint internals.
- Produces: photo message handler that creates a transaction and replies with a summary, or asks for manual entry.

- [ ] **Step 1: Write the failing tests**

```python
async def test_photo_from_shop_chat_creates_transaction(...): ...
async def test_photo_with_unreadable_receipt_returns_actionable_reply(...): ...
async def test_photo_from_unknown_chat_is_ignored(...): ...
async def test_same_file_unique_id_is_idempotent(...): ...
async def test_photo_over_10mb_is_rejected(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_receipt_photo.py -v`
Expected: FAIL — handler missing.

- [ ] **Step 3: Implement the photo handler**

On `message.photo`, take the largest size, `getFile`, download with a 15-second timeout and a 10 MB cap, and call the OCR ingestion path. Use `file_unique_id` as `client_event_id` to dedupe Telegram redelivery. Reply with the parsed total and item count, plus inline `[Edit amount] [Link customer]` buttons. On low confidence or parse failure, reply: `Could not read this receipt. Send a clearer photo or add the purchase from the admin panel.` Never echo raw OCR text.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_receipt_photo.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add telegram receipt photo purchase capture"
```

---

### Task 6: ESC/POS receipt parser

**Files:**
- Create: `pos_helper/requirements.txt`
- Create: `pos_helper/helper/__init__.py`
- Create: `pos_helper/helper/escpos.py`
- Create: `pos_helper/tests/fixtures/receipt_01.escpos`
- Create: `pos_helper/tests/test_escpos.py`

**Interfaces:**
- Produces:
  - `parse_escpos(raw: bytes) -> ParsedReceipt`
  - `ParsedReceipt(receipt_number: str | None, total_cents: int, currency: str, items: list[ParsedItem], occurred_at: datetime | None)`
  - `ParsedItem(name: str, price_cents: int, sku: str | None)`

- [ ] **Step 1: Write the failing tests**

```python
def test_parses_total_and_items_from_fixture(): ...
def test_parses_decimal_with_two_places(): ...
def test_parses_thousands_separator(): ...
def test_ignores_void_and_discount_lines(): ...
def test_returns_none_total_when_no_total_line(): ...
def test_recognizes_rm_usd_sgd_eur(): ...
def test_strips_escpos_control_sequences(): ...
```

Fixture `receipt_01.escpos` is a checked-in byte file with ESC/POS init, alignment, bold, item lines, `TOTAL RM 589.00`, and cut commands.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd pos_helper && pytest tests/test_escpos.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement parser**

Decode with `cp437` then fall back to `utf-8` with replacement. Remove ESC/POS commands via the standard escape sequences (`ESC @`, `GS V`, `ESC a`, `ESC E`). Normalize non-breaking spaces. Match total with:
```regex
(?:TOTAL|AMOUNT DUE|NET TOTAL|GRAND TOTAL)\s*[:.]?\s*(RM|USD|SGD|EUR|\$|€|£)?\s*([0-9][0-9,]*\.[0-9]{2})
```
Items are lines containing a trailing price that are not total, tax, change, discount, or void lines. Prefer the last total line. Convert currency symbols to ISO codes. Return `total_cents=None` when absent rather than guessing. `receipt_number` from `INV`, `RECEIPT`, `BILL`, or `#` patterns.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd pos_helper && pytest tests/test_escpos.py -v`
Expected: PASS (7 tests).

- [ ] **Step 5: Commit**

```bash
git add pos_helper
git commit -m "feat: add escpos receipt parser"
```

---

### Task 7: Windows receipt helper with folder-drop capture

**Files:**
- Create: `pos_helper/helper/config.py`
- Create: `pos_helper/helper/watcher.py`
- Create: `pos_helper/helper/api_client.py`
- Create: `pos_helper/helper/main.py`
- Create: `pos_helper/tests/test_watcher.py`
- Create: `pos_helper/packaging/build_windows.ps1`
- Modify: `pos_helper/requirements.txt`

**Interfaces:**
- Produces:
  - `ReceiptSource` Protocol with `pending() -> list[tuple[str, bytes]]` and `ack(name: str)`
  - `FolderDropSource(watch_dir: Path, processed_dir: Path)`
  - `WindowsPrintQueueSource(printer_name: str)` (experimental, Windows only)
  - `HelperConfig(api_base_url, api_key, watch_dir, processed_dir, currency)`
  - `run_once(helper_config, source, cloud) -> int`

- [ ] **Step 1: Write the failing tests**

```python
def test_folder_source_returns_new_files_and_ack_moves_them(tmp_path): ...
def test_folder_source_ignores_partial_files(tmp_path): ...
def test_run_once_posts_and_acks(tmp_path): ...
def test_run_once_leaves_file_on_network_failure(tmp_path): ...
def test_run_once_does_not_repost_processed_files(tmp_path): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd pos_helper && pytest tests/test_watcher.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement helper**

The recommended installation configures the POS printer to a Windows "print to file" port pointing at `C:\VisitMemory\spool\receipt.prn`, or uses a printer port that writes one file per job in a watched folder. `FolderDropSource` waits for a file to be stable for 2 seconds (size unchanged across two polls), reads it, and on successful post moves it to `processed/`. `run_once` derives `client_event_id` as `sha256(file_bytes)` so retries are idempotent. The helper activates like the camera agent with `installation_kind="POS_HELPER"` and stores its key with the same secret store. `WindowsPrintQueueSource` is behind a `windows` marker and documented as experimental; it must never write to the POS.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd pos_helper && pytest tests/test_watcher.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add pos_helper
git commit -m "feat: add windows receipt helper with durable folder-drop capture"
```

---

### Task 8: Admin purchase history, unlinked queue, and manual linking

**Files:**
- Create: `backend/app/api/admin_transactions.py`
- Create: `backend/app/templates/transactions/list.html`
- Create: `backend/app/templates/transactions/link.html`
- Modify: `backend/app/templates/customers/detail.html`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_admin_transactions.py`

**Interfaces:**
- Produces routes:
  - `GET /admin/transactions?shop_id=&linked=false`
  - `GET/POST /admin/transactions/{id}/link`
  - Customer detail includes purchases newest first with total, items, and source.

- [ ] **Step 1: Write the failing tests**

```python
async def test_unlinked_queue_lists_only_unlinked_transactions(...): ...
async def test_manual_link_to_customer_creates_or_selects_visit(...): ...
async def test_manual_link_updates_aggregates_once(...): ...
async def test_relink_reverses_old_customer_aggregates(...): ...
async def test_link_is_audited(...): ...
async def test_customer_detail_shows_purchases(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_admin_transactions.py -v`
Expected: FAIL — routes missing.

- [ ] **Step 3: Implement admin transaction pages**

Manual link requires selecting a customer in the same shop. If no counter visit exists, link the transaction to the customer's latest visit in the correlation window or create a `Visit` row with `camera_role="MANUAL"`, `status="matched"`, and `installation_id=NULL`. Reuse `link_transaction` so aggregates and audit behavior match automatic linking. The unlinked queue is ordered oldest first.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_admin_transactions.py -v`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add admin transaction queue and purchase history"
```

---

### Task 9: End-to-end transaction linking test

**Files:**
- Create: `backend/tests/test_transaction_e2e.py`

**Interfaces:**
- Consumes all Phase 3 and Phase 4 components.

- [ ] **Step 1: Write the failing test**

```python
async def test_counter_visit_then_receipt_links_and_updates_context(client, ...):
    # 1. Upload a COUNTER visit at 14:21:02 that matches VIP Alex.
    # 2. Post a transaction at 14:21:09 for RM589 Floral Summer Dress.
    # 3. Assert transaction is linked to Alex's visit.
    # 4. Assert Alex.lifetime_spend_cents == 58900.
    # 5. Assert Alex.last_purchase_item == "Floral Summer Dress".
    # 6. Advance to a later ENTRANCE visit; assert the alert caption includes the item.
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && pytest tests/test_transaction_e2e.py -v`
Expected: FAIL — integration path incomplete.

- [ ] **Step 3: Implement missing wiring**

No new abstractions. Fix whatever the test exposes in `process_visit`, `ingest_transaction`, or the alert builder. Keep the test using fakes for matcher, storage, and messaging.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && pytest tests/test_transaction_e2e.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "test: add end-to-end counter visit to purchase linking"
```

---

## Phase 4 Exit Checklist

- [ ] A spooler or OCR receipt within 90 seconds of a counter visit links to the right customer.
- [ ] A duplicate receipt event never double-counts lifetime spend.
- [ ] A receipt with no counter visit stays in the unlinked queue and can be linked manually.
- [ ] Linking and aggregates commit together; relinking reverses the old customer correctly.
- [ ] The next entrance alert includes the last purchase item and relative date.
- [ ] The Windows helper survives network loss and never modifies POS data.
