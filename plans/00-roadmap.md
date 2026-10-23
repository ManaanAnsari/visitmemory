# VisitMemory Phased Build Roadmap

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement each phase plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a pilot-ready VisitMemory MVP: an edge agent on a store PC, a cloud recognition and alerting backend, a POS receipt-linking path, and an operator admin panel that onboards distributors, attributes every shop to a distributor, and computes commissions payable.

**Architecture:** The edge agent samples an existing CCTV feed, runs face detection and quality gating locally, and uploads only qualified crops. The cloud backend matches faces, deduplicates visits, links receipts, and pushes alerts to Telegram. A server-rendered FastAPI admin panel manages distributors, shops, one-time activation codes, installations, subscriptions, and payout ledgers. Every edge installation is activated against a shop and distributor pair, so revenue attribution is deterministic from day one.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 async, Alembic, PostgreSQL 16, Redis 7, Celery, Jinja2 server-rendered admin, argon2-cffi, AWS Rekognition behind an adapter, Cloudflare R2 behind an adapter, Telegram Bot API, pytest, Docker Compose.

**Spec:** `docs/tech.md` (architecture), `docs/business.md` (pricing, distributor economics, beta gates), `docs/legal.md` (deferred, see Scope Boundary).

## Global Constraints

- Python 3.12 everywhere (`python_requires >= 3.12`), PostgreSQL 16, Redis 7.
- Every dependency pin is exact (`==`). No `^`, `~`, `>=`, or unpinned names in `requirements*.txt` or `package.json`. Resolve pins deliberately, one package at a time, using a version released more than 72 hours ago.
- No npm/Node build chain for the admin panel. Templates are Jinja2, CSS is one hand-written file, JS is limited to first-party inline snippets.
- No secrets in the repo. `.env` is git-ignored; only `.env.example` with placeholder names is committed. Any required secret with no value must fail startup, never fall back to a default.
- All timestamps stored in UTC (`TIMESTAMPTZ`); render in the shop's `timezone` only in UI.
- All money is integer minor units (cents) plus an ISO-4217 `currency` string. No floats for money.
- All primary keys are UUIDv4. All schema changes go through Alembic migrations.
- All state-changing admin and API operations write an `audit_logs` row.
- API paths are versioned under `/api/v1`. Admin pages live under `/admin`.
- Admin cookies: `HttpOnly`, `Secure` in production, `SameSite=Lax`; session tokens stored hashed, never raw.
- Activation codes and edge API keys are stored hashed; raw values are shown once at creation.
- Tests run offline by default. Every external service (Rekognition, R2, Telegram, Stripe, Gemini) sits behind a Protocol with a fake used in tests.
- Never store raw video. Store only the qualified crop required for the visit, subject to the retention job.

## Scope Boundary

**In scope for the MVP (Phases 1-4):** distributor/shop admin, activation and installation auth, edge capture and upload, cloud recognition and Telegram alerts, staff tagging, receipt photo/OCR, print-spooler receipt capture, purchase history.

**In scope for pilot commercialisation (Phase 5):** manual subscription invoicing, commission ledger, payout tracking, distributor statements, optional Stripe adapter.

**In scope for launch (Phase 6):** logging, metrics, rate limits, backups, deployment, onboarding polish, beta scorecard.

**Explicitly deferred (not in these plans):** WhatsApp Cloud API, Jev/LLM decision engine, multi-branch VIP roaming, POS screen overlay, multi-zone grid capture, enterprise SSO, automated card billing beyond the Stripe adapter task, mobile apps, an agent GUI onboarding wizard (MVP uses the CLI), POS screen-scrape and email-BCC receipt methods (MVP uses folder-drop spool files plus receipt photos), a `cameras` entity (MVP stores `camera_role` on each visit), and per-staff user accounts (MVP uses one Telegram group per shop).

**Legal/privacy:** `docs/legal.md` identifies consent obligations that are not resolved. These plans keep the consent gate, retention purge, and audit trail as configurable mechanisms, but do not claim compliance. Launch requires a separate legal decision.

## Phase Sequence

| Phase | Plan | Deliverable | Exit Criteria |
| :---: | :--- | :--- | :--- |
| 1 | `01-platform-foundation-and-admin.md` | Backend foundation, admin login, distributor/shop CRUD, activation codes, installation auth | An operator can create a distributor and shop, generate a code, and an agent can activate and receive an API key |
| 2 | `02-edge-agent-capture.md` | Edge agent capture, detection, quality gates, dedup, spool, upload, Windows packaging | Agent runs against a video source, produces qualified crops, survives network loss, ships as a Windows build |
| 3 | `03-recognition-and-alerts.md` | Visit ingestion, face matching, cooldown, Telegram alerts, staff tagging, purge job | A known face entering the feed produces a Telegram alert; an unknown face can be tagged |
| 4 | `04-pos-transaction-linking.md` | Receipt ingestion, OCR fallback, print spooler, correlation, purchase history | A counter-camera visit is linked to the correct receipt and shown on the customer profile |
| 5 | `05-billing-and-distributor-commissions.md` | Subscriptions, invoices, commission runs, payouts, statements | A paid month produces the correct distributor commission and an auditable payout |
| 6 | `06-pilot-hardening.md` | Observability, limits, backups, deploy, onboarding, scorecard | Five pilot stores can run for 30 days against the beta metrics in `docs/business.md` §9 |

Each plan produces working, testable software on its own. Do not start a phase plan before the previous phase's exit criteria are met, with one exception: Phase 5 commission logic may be built in parallel with Phase 4 once Phase 3 is complete.

## Key Decisions

1. **Admin panel is server-rendered FastAPI + Jinja2, not a SPA.** It keeps the supply chain small, avoids a Node build step, and matches the security guidelines. If a richer UI is needed later, add an API and a separate frontend then.
2. **Admin panel is platform-operator-only.** Merchants and distributors do not log into it in the MVP. Distributor statements are exported by the operator.
3. **Attribution uses `distributor_code` + one-time `activation_code`.** A distributor code alone is not a secret and cannot authorize an installation. The activation code binds one installation to one shop, which is already assigned to one distributor. This prevents installers from misattributing shops and prevents commission theft.
4. **Edge auth is a per-installation bearer API key**, returned once at activation, stored as an HMAC-SHA256 hash with a server pepper. Revoking an installation invalidates the key immediately.
5. **Recognition is behind a `FaceMatcher` Protocol.** Phase 1 adapter is AWS Rekognition; tests use `FakeFaceMatcher`. A future ArcFace/Qdrant adapter must implement the same interface.
6. **Messaging is behind a `MessageChannel` Protocol.** MVP adapter is Telegram; tests use `FakeMessageChannel`.
7. **Storage is behind an `ObjectStorage` Protocol.** MVP adapter is Cloudflare R2 (S3-compatible); tests use `InMemoryObjectStorage`.
8. **Billing is behind a `PaymentProvider` Protocol.** MVP supports operator-recorded manual payments plus an optional Stripe Checkout adapter.
9. **Distributor defaults come from `docs/business.md` §6.2:** `activation_bounty_cents = 2000`, `commission_rate_bps = 2500` (25%), stored per distributor and snapshot into each commission entry.
10. **Commission periods are calendar months in UTC.** Full-month commission is paid when a shop has a PAID invoice covering that month; partial months are out of scope for the MVP.

## Dependency Graph

```
Phase 1 (admin, distributors, shops, activation, installation auth)
   |
   +--> Phase 2 (edge agent activation + upload)
   |        |
   |        +--> Phase 3 (recognition, customers, alerts)
   |                 |
   |                 +--> Phase 4 (receipts, correlation)
   |                 |
   |                 +--> Phase 5 (billing, commissions)
   |
   +--> Phase 6 (hardening, deploy, pilot)
```

Phase 5 depends on Phase 1 for shops/distributors and Phase 3 for active shops having visits. Phase 6 depends on all prior phases.

## Cross-Cutting Review Focus

These are the failure modes no single phase test fully covers. Each is pinned by a test in the owning phase plan.

1. **Activation replay or distributor mismatch.** Two installations activate the same code, or an installer enters a distributor code that does not match the shop's assigned distributor, and commissions are paid to the wrong party.
2. **Admin session and CSRF gaps.** An unauthenticated or cross-site request mutates distributors, shops, or payouts.
3. **Money and period rounding.** Commission runs use floats, local time, or non-idempotent logic and pay a distributor twice or the wrong amount.
4. **Lost or duplicated visits on flaky store networks.** The agent's spool drops events, or retries create duplicate visits and duplicate alerts.
5. **Wrong-person alert.** A loose match threshold greets a stranger by a VIP's name, which is worse than no alert.

## Testing Strategy

- Backend unit and API tests: `pytest`, `pytest-asyncio`, `httpx.AsyncClient` with `ASGITransport`.
- Database tests: real PostgreSQL 16 via Docker Compose (`TEST_DATABASE_URL`); never SQLite for schema tests because of JSONB and UUID behavior.
- External services: Protocol fakes only. Live AWS/Telegram/Stripe/Gemini tests are marked `integration` and skipped unless their env vars are present.
- Edge agent: synthetic frame sources and fake detectors; numeric tests for quality gates; no camera or model required in CI.
- Windows-only code: marked `windows` and skipped on other platforms, with pure-logic tests for parsers and state machines.
- Every task ends with `pytest` passing and a commit.

## Version Pinning Procedure

Before the first commit of any phase:

1. For each dependency, choose the latest stable release older than 72 hours.
2. Confirm a prebuilt wheel exists for CPython 3.12 on the target platform; do not install compilers to build a dependency.
3. Write the exact version with `==` in `requirements.txt` or `requirements-dev.txt`.
4. Copy the resolved set into the phase commit message or a `requirements.lock` generated by `pip freeze` from the verified environment.

## Open Decisions Deferred to Phase 5/6

- Payment provider for Malaysian retailers (Stripe availability vs local gateway).
- Hosting region and whether recognition moves to Malaysia-resident infrastructure (drives COGS).
- Whether distributors get a read-only portal login (MVP exports CSV instead).
