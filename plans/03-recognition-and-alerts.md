# Phase 3: Cloud Recognition and Staff Alerts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ingest edge visit crops, match them against a face recognition provider, manage customer profiles and visit cooldowns, and deliver useful Telegram alerts that staff can tag, confirm, or ignore.

**Architecture:** The visit endpoint authenticates the installation, stores the crop through an `ObjectStorage` adapter, and calls a `FaceMatcher` adapter. A pure `RecognitionService` applies threshold bands and cooldown rules, then a `MessageChannel` adapter delivers Telegram alerts. All provider calls sit behind Protocols so tests run offline with fakes. A Celery beat job purges untagged provisional profiles.

**Tech Stack:** FastAPI, SQLAlchemy 2.0 async, PostgreSQL 16, Celery + Redis, boto3 (R2), AWS Rekognition (boto3), Telegram Bot API via httpx, pytest.

**Spec:** `docs/tech.md` §3-§5, §10; `docs/business.md` §7; `plans/01-platform-foundation-and-admin.md`, `plans/02-edge-agent-capture.md`.

## Global Constraints

- Exact dependency pins only (`==`); resolve releases older than 72 hours.
- The visit upload accepts JPEG only, maximum 512 KB, dimensions at least 80x80.
- Match bands: `>= 0.92` matched, `0.80 - 0.9199` unconfirmed, `< 0.80` new.
- Visit cooldown: `120` minutes per customer per shop, configurable on `shops.visit_cooldown_minutes`.
- Provisional retention: `24` hours, configurable.
- Never log image bytes, API keys, bot tokens, or full provider responses.
- Every visit row stores the UTC arrival time and the shop's timezone is applied only for display.
- Alerts must be idempotent: an agent retry with the same `client_event_id` must not produce a second alert.
- All provider adapters are selected by configuration; fakes are used in tests.
- Raw video is never accepted by the API; only a single JPEG crop per request.

## File Structure

```
backend/app/
  models/customer.py            Customer, FaceEmbedding, Visit
  models/shop.py                add telegram_chat_id, visit_cooldown_minutes
  storage/base.py               ObjectStorage Protocol
  storage/memory.py             InMemoryObjectStorage
  storage/r2.py                 R2ObjectStorage
  recognition/base.py           FaceMatcher Protocol, Match, FakeFaceMatcher
  recognition/rekognition.py    AwsRekognitionMatcher
  recognition/service.py        recognize_visit, thresholds, cooldown
  messaging/base.py             MessageChannel Protocol, ArrivalAlert
  messaging/fake.py             FakeMessageChannel
  messaging/telegram.py         TelegramChannel
  api/visits.py                 POST /api/v1/visits
  api/telegram.py               webhook + staff actions
  api/admin_customers.py        customer/visit pages, shop chat config
  tasks/celery_app.py           Celery application
  tasks/purge.py                purge_provisional_customers
  templates/customers/list.html
  templates/customers/detail.html
  templates/shops/telegram.html
  alembic/versions/0003_customers_visits.py
backend/tests/
  test_visit_ingest.py
  test_recognition_service.py
  test_rekognition_adapter.py
  test_cooldown.py
  test_telegram_channel.py
  test_telegram_webhook.py
  test_admin_customers.py
  test_purge.py
  test_recognition_e2e.py
```

## Interfaces

- `ObjectStorage.put_object(key: str, data: bytes, content_type: str) -> None`
- `ObjectStorage.presigned_get_url(key: str, expires_seconds: int = 600) -> str`
- `ObjectStorage.delete_object(key: str) -> None`
- `InMemoryObjectStorage.count(prefix: str) -> int` (test helper)`
- `FaceMatcher.index_face(shop_id: UUID, image_jpeg: bytes, external_id: str) -> str`
- `FaceMatcher.search_face(shop_id: UUID, image_jpeg: bytes) -> list[Match]`
- `FaceMatcher.delete_face(shop_id: UUID, external_face_id: str) -> None`
- `Match(external_face_id: str, similarity: float)`
- `MessageChannel.send_arrival(chat_id: str, alert: ArrivalAlert) -> MessageRef`
- `MessageChannel.send_silent(chat_id: str, text: str) -> MessageRef`
- `ArrivalAlert(visit_id, customer_id, customer_name, is_vip, is_unconfirmed, confidence, days_since_last_visit, last_item, snapshot_url)`
- `recognize_visit(db, matcher, shop_id, image_jpeg) -> RecognitionResult`
- `process_visit(db, *, installation, image_jpeg, camera_role, captured_at, client_event_id, matcher, storage, channel) -> VisitOutcome`
- `VisitOutcome(visit_id: UUID, status: Literal["matched", "unconfirmed", "new", "suppressed", "ignored", "duplicate"], customer_id: UUID | None)`

## Review Focus

1. **Retry creates duplicate alerts.** The agent retries the same `client_event_id` after a timeout and the customer is alerted twice. Expected: the second request returns the original outcome with `duplicate` and sends nothing.
2. **Wrong-person alert.** A 0.81 similarity is presented as certain and staff greet a stranger by name. Expected: the unconfirmed band is labelled as uncertain and staff can confirm or reject.
3. **Cooldown misses or over-suppresses.** The same shopping trip alerts twice, or a genuine return visit two hours later is silent. Expected: `120` minutes exactly, boundary tested.
4. **Staff ignore fails.** Employees receive alerts on every pass, or an ignored customer is still counted as a VIP arrival. Expected: ignored profiles are logged silently and never alerted.
5. **Purge deletes a real customer.** A named or purchasing customer has no recent visit and is purged with their face vector. Expected: only unnamed, non-VIP, purchase-free profiles older than 24 hours are purged.

---

### Task 1: Customer, face, and visit models with visit ingestion storage

**Files:**
- Create: `backend/app/models/customer.py`
- Modify: `backend/app/models/shop.py`
- Create: `backend/alembic/versions/0003_customers_visits.py`
- Create: `backend/app/storage/base.py`
- Create: `backend/app/storage/memory.py`
- Create: `backend/app/storage/r2.py`
- Create: `backend/app/api/visits.py`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_visit_ingest.py`

**Interfaces:**
- Produces tables:
  - `Customer(id, shop_id, display_name, is_vip, is_staff_ignored, staff_notes, lifetime_spend_cents, total_visits_count, primary_avatar_key, last_seen_at, created_at)`
  - `FaceEmbedding(id, customer_id, provider, external_face_id, quality_score, created_at)`
  - `Visit(id, shop_id, customer_id, installation_id, client_event_id, camera_role, snapshot_key, match_confidence, status, alert_dispatched, arrival_time, created_at)`, where `installation_id` is nullable so Phase 4 can create manual visits
  - `Shop.telegram_chat_id`, `Shop.visit_cooldown_minutes`
- Produces `POST /api/v1/visits` and `ObjectStorage` adapters.

- [ ] **Step 1: Write the failing test**

```python
async def test_visit_ingest_stores_crop_and_is_idempotent(client, installation, fake_storage):
    response = client.post(
        "/api/v1/visits",
        headers={"Authorization": f"Bearer {installation.raw_key}"},
        data={
            "client_event_id": "11111111-1111-4111-8111-111111111111",
            "captured_at": "2026-10-20T10:00:00+00:00",
            "camera_role": "ENTRANCE",
            "agent_version": "0.1.0",
        },
        files={"image": ("face.jpg", JPEG_BYTES, "image/jpeg")},
    )
    assert response.status_code == 201
    assert response.json()["status"] == "new"

    retry = client.post(... same payload ...)
    assert retry.status_code == 200
    assert retry.json()["status"] == "duplicate"
    assert fake_storage.count(prefix=f"shops/{installation.shop_id}/visits/") == 1
```

Additional tests: rejects non-JPEG, rejects oversized payloads, rejects missing auth, rejects a `POS_HELPER` installation (only `CAMERA` installations may upload visits), and updates `last_seen_at` on installation. In this task the route inserts a placeholder visit with `status="new"`; Task 3 replaces the placeholder with recognition.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && pytest tests/test_visit_ingest.py -v`
Expected: FAIL — visit model and route missing.

- [ ] **Step 3: Implement models, storage adapters, and upload route**

Store key `shops/{shop_id}/visits/{visit_id}.jpg`. Enforce `Content-Length <= 512KB` and JPEG magic bytes before reading the body. Validate `client_event_id` as UUID. Unique constraint `(installation_id, client_event_id)`. On conflict return `200` with the existing outcome and `status="duplicate"`. `R2ObjectStorage` uses boto3 with the endpoint URL from settings; `InMemoryObjectStorage` is test-only.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && pytest tests/test_visit_ingest.py -v`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add customer/visit models and idempotent visit ingestion"
```

---

### Task 2: Face matcher protocol and adapters

**Files:**
- Create: `backend/app/recognition/base.py`
- Create: `backend/app/recognition/rekognition.py`
- Create: `backend/tests/test_rekognition_adapter.py`

**Interfaces:**
- Produces `FaceMatcher`, `Match`, `FakeFaceMatcher`, `AwsRekognitionMatcher`.

- [ ] **Step 1: Write the failing tests**

```python
def test_fake_matcher_indexes_and_searches():
    matcher = FakeFaceMatcher()
    external_id = matcher.index_face(SHOP_ID, JPEG_A, "face-1")
    assert external_id == "face-1"
    matches = matcher.search_face(SHOP_ID, JPEG_A)
    assert matches[0].external_face_id == "face-1"
    assert matches[0].similarity == 1.0


def test_rekognition_normalizes_similarity_percent_to_unit_interval():
    adapter = AwsRekognitionMatcher(client=FakeRekognitionClient(...), ...)
    matches = adapter.search_face(SHOP_ID, JPEG_A)
    assert matches[0].similarity == 0.974


def test_rekognition_missing_face_returns_empty_list():
    ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_rekognition_adapter.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement adapters**

`AwsRekognitionMatcher` uses one collection per shop named `vm-{shop_id.hex}`, creates it lazily on first index, calls `IndexFaces` with `MaxFaces=1`, `QualityFilter="HIGH"`, and `SearchFacesByImage` with `MaxFaces=5`, `FaceMatchThreshold=80`. Convert `Similarity` percent to `[0, 1]`. Map `InvalidParameterException` for no face to an empty list. Delete uses `DeleteFaces`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_rekognition_adapter.py -v`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/recognition backend/tests/test_rekognition_adapter.py
git commit -m "feat: add face matcher protocol, fake, and rekognition adapter"
```

---

### Task 3: Recognition service, threshold bands, and provisional profiles

**Files:**
- Create: `backend/app/recognition/service.py`
- Create: `backend/tests/test_recognition_service.py`
- Modify: `backend/app/api/visits.py`

**Interfaces:**
- Consumes: `FaceMatcher`, `Customer`, `FaceEmbedding`.
- Produces:
  - `MATCH_THRESHOLD = 0.92`, `UNCONFIRMED_THRESHOLD = 0.80`
  - `recognize_visit(db, matcher, shop_id, image_jpeg) -> RecognitionResult`
  - `RecognitionResult(status, customer, confidence, face_embedding)`
  - `create_provisional_customer(db, shop_id, face_embedding) -> Customer`

- [ ] **Step 1: Write the failing tests**

```python
async def test_similarity_at_092_is_matched(db_session, fake_matcher): ...
async def test_similarity_just_below_092_is_unconfirmed(db_session, fake_matcher): ...
async def test_similarity_below_080_creates_provisional_customer(db_session, fake_matcher): ...
async def test_provisional_customer_has_null_name_and_not_vip(db_session, fake_matcher): ...
async def test_existing_face_is_not_reindexed(db_session, fake_matcher): ...
async def test_provider_error_does_not_create_a_customer(db_session, failing_matcher): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_recognition_service.py -v`
Expected: FAIL — service missing.

- [ ] **Step 3: Implement recognition**

Search first. If the best match is `>= 0.92` and maps to a customer in this shop, return `matched` and do not index. If `0.80 <= similarity < 0.92`, return `unconfirmed` with that customer. If no match above `0.80` or no mapped customer, index the face as a new external face and create a provisional `Customer` (`display_name=None`, `is_vip=False`) plus a `FaceEmbedding` with `provider="rekognition"` and `quality_score=None`. All lookups are scoped by `shop_id`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_recognition_service.py -v`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/recognition/service.py backend/app/api/visits.py backend/tests/test_recognition_service.py
git commit -m "feat: add recognition threshold bands and provisional customer creation"
```

---

### Task 4: Cooldown, staff-ignore, and alert decision

**Files:**
- Create: `backend/tests/test_cooldown.py`
- Modify: `backend/app/recognition/service.py`
- Modify: `backend/app/models/shop.py`

**Interfaces:**
- Produces `decide_alert(customer: Customer, now: datetime, cooldown_minutes: int) -> AlertDecision` where `AlertDecision(should_alert: bool, reason: Literal["ok", "cooldown", "staff_ignored"])`.

- [ ] **Step 1: Write the failing tests**

```python
async def test_visit_at_exactly_120_minutes_is_suppressed(...): ...
async def test_visit_at_121_minutes_alerts(...): ...
async def test_staff_ignored_customer_is_suppressed(...): ...
async def test_suppressed_visit_is_still_recorded(...): ...
async def test_unconfirmed_match_alerts_with_uncertain_label(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_cooldown.py -v`
Expected: FAIL — `decide_alert` missing.

- [ ] **Step 3: Implement alert decision**

Cooldown compares `now - customer.last_seen_at` in UTC against `cooldown_minutes * 60` seconds. `<=` boundary suppresses; `>` alerts. Staff-ignored always suppresses regardless of cooldown. A suppressed visit still inserts a `Visit` row with `alert_dispatched=False` and status `suppressed` or `ignored`. Update `last_seen_at` and `total_visits_count` only when a new visit row is inserted, not on a duplicate.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_cooldown.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add visit cooldown and staff-ignore alert decisions"
```

---

### Task 5: Messaging protocol and Telegram channel adapter

**Files:**
- Create: `backend/app/messaging/base.py`
- Create: `backend/app/messaging/fake.py`
- Create: `backend/app/messaging/telegram.py`
- Create: `backend/tests/test_telegram_channel.py`
- Modify: `backend/app/config.py`

**Interfaces:**
- Produces `MessageChannel`, `ArrivalAlert`, `MessageRef`, `FakeMessageChannel`, `TelegramChannel`.

- [ ] **Step 1: Write the failing tests**

```python
def test_arrival_alert_renders_name_days_and_last_item():
    ...
    assert "Alex" in text
    assert "4 months" in text
    assert "Floral Summer Dress" in text


def test_unconfirmed_alert_is_labelled_uncertain():
    assert "Possible match" in text


def test_telegram_send_photo_includes_inline_buttons():
    channel = TelegramChannel(bot_token="test", client=httpx.Client(transport=...))
    channel.send_arrival("-100123", alert)
    request = captured_requests[0]
    assert "sendPhoto" in str(request.url)
    assert b"tag:" in request.read()


def test_telegram_redacts_token_on_error():
    ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_telegram_channel.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement channel and rendering**

`TelegramChannel.send_arrival` uses `sendPhoto` with the presigned snapshot URL, caption, `parse_mode="HTML"`, and inline keyboard rows:
- `[Tag name]` -> `tag:{visit_id}`
- `[VIP]` -> `vip:{customer_id}`
- `[Ignore / Staff]` -> `ignore:{customer_id}`

Caption format:
```
<b>{name}</b> is back{uncertain_suffix}
Last visit: {relative_time}
Last item: {last_item or "No recorded purchase"}
Match confidence: {confidence:.0%}
```
Use plain text alternatives if HTML escaping is needed. `send_silent` sends `disable_notification=True`. Telegram API errors raise `MessageDeliveryError` containing only status and error code, never the bot token.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_telegram_channel.py -v`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add messaging protocol and telegram arrival alerts"
```

---

### Task 6: Wired visit processing and Telegram webhook staff actions

**Files:**
- Modify: `backend/app/api/visits.py`
- Create: `backend/app/api/telegram.py`
- Create: `backend/app/models/staff_action.py`
- Create: `backend/alembic/versions/0004_staff_actions.py`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_telegram_webhook.py`
- Create: `backend/tests/test_recognition_e2e.py`

**Interfaces:**
- Consumes: recognition service, cooldown, `MessageChannel`.
- Produces:
  - `process_visit(db, *, installation, image_jpeg, camera_role, captured_at, client_event_id, matcher, storage, channel) -> VisitOutcome`
  - `PendingStaffAction(id, chat_id, user_id, action_type, target_id, expires_at, created_at)`
  - `POST /api/v1/telegram/webhook` verifying `X-Telegram-Bot-Api-Secret-Token`
  - Staff callbacks: `tag`, `vip`, `ignore`, `confirm`, `reject`

- [ ] **Step 1: Write the failing tests**

```python
async def test_matched_vip_triggers_one_telegram_alert(...): ...
async def test_same_client_event_id_sends_no_second_alert(...): ...
async def test_ignore_button_sets_staff_ignored_and_audits(...): ...
async def test_tag_flow_sets_display_name_and_marks_vip(...): ...
async def test_webhook_rejects_wrong_secret_token(...): ...
async def test_webhook_rejects_unknown_chat(...): ...
async def test_unconfirmed_confirm_action_upgrades_customer(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_telegram_webhook.py tests/test_recognition_e2e.py -v`
Expected: FAIL — webhook and process wiring missing.

- [ ] **Step 3: Implement processing and webhook**

`process_visit` order: idempotency check, store crop, recognize, find shop cooldown, decide alert, insert visit, update customer aggregates, dispatch only when allowed, mark `alert_dispatched`, write audit rows. A provider or messaging failure must not lose the visit: store the visit with status `matched`/`unconfirmed` and `alert_dispatched=False`, log a safe error, and return `201`.

Webhook: parse `callback_query`, answer it, then mutate and edit the original message caption to append the result. Tag flow stores a `PendingStaffAction` with 10-minute expiry and sends a `ForceReply` prompt; the next text message from that user resolves the action. All actions are scoped to the chat's shop.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_telegram_webhook.py tests/test_recognition_e2e.py -v`
Expected: PASS (7 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: wire visit processing and telegram staff actions"
```

---

### Task 7: Admin customer, visit, and Telegram chat configuration

**Files:**
- Create: `backend/app/api/admin_customers.py`
- Create: `backend/app/templates/customers/list.html`
- Create: `backend/app/templates/customers/detail.html`
- Create: `backend/app/templates/shops/telegram.html`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_admin_customers.py`

**Interfaces:**
- Produces routes:
  - `GET /admin/shops/{id}/customers`
  - `GET /admin/customers/{id}`
  - `POST /admin/customers/{id}` (name, VIP, ignore, notes)
  - `GET/POST /admin/shops/{id}/telegram`
  - `GET /admin/visits?shop_id=&status=`

- [ ] **Step 1: Write the failing tests**

```python
async def test_customer_detail_lists_visits_newest_first(...): ...
async def test_edit_customer_writes_audit_and_updates_vip(...): ...
async def test_telegram_chat_id_is_validated_as_numeric_or_channel_id(...): ...
async def test_customer_search_is_scoped_to_shop(...): ...
async def test_visits_never_render_snapshot_key_to_non_admin(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_admin_customers.py -v`
Expected: FAIL — routes missing.

- [ ] **Step 3: Implement admin pages**

Customer list shows display name, VIP flag, visits, last seen, and lifetime spend (zero until Phase 4). Detail shows visits with presigned crop URLs valid for 10 minutes. There is no hard delete in the admin UI; the purge job handles provisional data and a separate audited erasure action is out of scope.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_admin_customers.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add admin customer, visit, and telegram configuration pages"
```

---

### Task 8: Provisional profile purge job

**Files:**
- Create: `backend/app/tasks/celery_app.py`
- Create: `backend/app/tasks/purge.py`
- Create: `backend/tests/test_purge.py`
- Modify: `backend/app/config.py`

**Interfaces:**
- Produces `purge_provisional_customers(db, matcher, storage, now, retention_hours=24) -> PurgeResult`.
- Celery beat schedule daily at `02:00 UTC`.

- [ ] **Step 1: Write the failing tests**

```python
async def test_purges_unnamed_nonvip_customer_older_than_24h(...): ...
async def test_keeps_unnamed_customer_younger_than_24h(...): ...
async def test_keeps_named_customer(...): ...
async def test_keeps_vip_customer(...): ...
async def test_deletes_face_embedding_and_storage_objects(...): ...
async def test_purge_is_idempotent(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_purge.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement purge**

Select customers where `display_name IS NULL`, `is_vip = false`, `is_staff_ignored = false`, `lifetime_spend_cents = 0`, `created_at < now - retention`. For each: delete all face embeddings through `matcher.delete_face`, delete visit snapshot keys through storage, delete visits, then delete the customer. Commit per batch of 100 and write one audit row per purged customer with counts only. The function returns `PurgeResult(customers_deleted, visits_deleted, faces_deleted)`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_purge.py -v`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add provisional profile purge job"
```

---

## Phase 3 Exit Checklist

- [ ] A valid installation upload returns a visit outcome and stores exactly one crop per `client_event_id`.
- [ ] A face matching a VIP produces one Telegram alert with name, relative last visit, last item, and confidence.
- [ ] A retry does not alert twice.
- [ ] A staff-ignored profile never alerts, and the ignore action is audited.
- [ ] An unconfirmed match is labelled uncertain and can be confirmed or rejected.
- [ ] The purge job deletes only eligible provisional profiles and is idempotent.
- [ ] `pytest` passes with no network access; Telegram and Rekognition are fakes.
