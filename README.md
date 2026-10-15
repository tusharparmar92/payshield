# payshield
A payment failure detection and reconciliation engine

PayShield is an automated payment failure detection and reconciliation engine built with **Python (FastAPI)** and **PostgreSQL**. It decouples authoritative payment provider ledgers from transient webhook delivery notifications, enforces cryptographic idempotency, detects payment failures and state drifts, and provides auditable operator remediation workflows.

---

## Key Architecture & Capabilities

1. **Dual-Feed Gateway Observability**:
   - `psp_authoritative_records`: Ground-truth ledger populated from provider settlement reports or direct query APIs.
   - `psp_webhook_events`: Transient HTTP delivery receipt log. A dropped webhook does not erase the underlying provider transaction.
2. **Deterministic Idempotency & Signature Verification**:
   - **Orders**: Client-provided `Idempotency-Key` verified against a canonical payload hash. Replaying identical requests returns cached responses (`Idempotent-Replayed: true`); reusing keys with altered payloads triggers `HTTP 409 Conflict`. Race conditions resolve cleanly.
   - **Webhooks**: Provider event IDs (`provider_event_id`) enforce database-level uniqueness `UNIQUE (psp_name, provider_event_id)`. Concurrent duplicate deliveries resolve safely via transaction rollback and retry lookup, returning HTTP 200 without duplicate processing.
   - **Signature Verification**: Incoming webhooks require HMAC-SHA256 signature verification over the raw request body via `X-PSP-Signature` and a configured shared secret.
3. **Monotonic Lifecycle State Machine**:
   - Strict status progression (`PENDING` -> `CAPTURED`, `PENDING` -> `FAILED`, `PENDING` -> `CANCELLED`). Delayed or out-of-order webhooks cannot regress a captured or terminal order.
4. **Read-Only by Default Reconciliation & Concurrency Control**:
   - The reconciliation engine is strictly diagnostic and passive: it calculates ledger variances, diagnoses root causes, and proposes recovery actions.
   - **No Automatic Money Movement**: Under no circumstance does the engine initiate unapproved refunds or move money.
   - **Idempotent Reconciliation Passes**: Re-running reconciliation over the same data window updates `last_seen_run_id` without creating duplicate active open discrepancies.
   - **Advisory Locking**: PostgreSQL advisory locking (`pg_try_advisory_lock`) serializes reconciliation runs and prevents concurrent execution conflicts, returning `HTTP 409 Conflict` under lock contention.
5. **8 Failure & Discrepancy Scenarios Detected**:
   - `DROPPED_WEBHOOK_UNRECONCILED_SUCCESS`: Payment captured on gateway, but webhook was dropped in transit. Proposed action: `SYNC_ORDER_TO_CAPTURED`.
   - `MISSING_PSP_RECORD`: Order marked captured internally, but absent from provider ledger.
   - `ORPHAN_PSP_CHARGE`: Customer charged by gateway without an internal order. Proposed action: `FLAG_FOR_OFFLINE_REFUND`.
   - `AMOUNT_MISMATCH`: Monetary variance between order amount and charged amount. Proposed action: `REQUEST_INVOICE_ADJUSTMENT`.
   - `CURRENCY_MISMATCH`: Currency mismatch between internal order and charged currency.
   - `STATUS_MISMATCH`: Gateway declined / failed while internal state indicates captured. Proposed action: `SYNC_STATUS_TO_FAILED`.
   - `DUPLICATE_PSP_CHARGE`: Multiple distinct gateway charges referencing the same order reference. Proposed action: `FLAG_FOR_OFFLINE_REFUND`.
   - `STALE_PENDING`: Orders pending past configured expiration threshold (default: 15 minutes) with zero gateway activity. Proposed action: `CANCEL_STALE_PENDING`.
6. **Authorized Operator Remediation Workflow**:
   - Operators review discrepancies and explicitly execute recovery actions via `POST /api/v1/reconciliation/discrepancies/{id}/actions/execute`.
   - **Remediation Safety**: Strictly enforces an explicit allowlist mapping permitted actions per discrepancy type (e.g. `STALE_PENDING` cannot be marked `CAPTURED`), rejecting incompatible actions before database mutations.
   - **Operator Authentication**: Authenticates operator identity using development tokens (`Authorization: Bearer <dev-token>` or `X-Operator-Token`), rejecting unauthenticated requests and ignoring spoofed client-supplied `operator_id` headers. *(Development-only mock authentication; not production-ready).*
   - Every action executes inside an isolated transaction and generates an immutable record in `audit_logs`.

---

## Entity Relational Model

```
payment_orders ──┬── psp_authoritative_records (Authoritative gateway ledger)
                 ├── psp_webhook_events        (Transient delivery logs)
                 └── reconciliation_discrepancies
                             ▲
                             │
                    reconciliation_runs
                             │
                        audit_logs
```

---

## Getting Started

### 1. Prerequisites
- Python 3.11+
- PostgreSQL (running locally or via container)

### 2. Environment Setup
```bash
# Clone the repository
git clone https://github.com/tusharparmar92/payshield.git
cd payshield

# Initialize virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Configure environment variables
cp .env.example .env
```

### 3. Running the Server
```bash
uvicorn app.main:app --reload --port 8000
```
Interactive API documentation will be available at:
- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`

---

## API Reference

### Orders
- `POST /api/v1/orders`: Create order with `Idempotency-Key` and payload hash verification.
- `GET /api/v1/orders/{order_ref}`: Retrieve order details and current status.
- `GET /api/v1/orders`: List orders with pagination and status filters.

### Mock PSP & Ingestion
- `POST /api/v1/mock-psp/charge`: Simulate payment charge with failure injections (`simulate_drop_webhook`, `simulate_amount_tamper`, `simulate_duplicate_charge`, `simulate_decline`).
- `POST /api/v1/mock-psp/webhook`: Ingest incoming webhooks with HMAC-SHA256 signature verification (`X-PSP-Signature`) and event-level idempotency.
- `POST /api/v1/ingestion/psp-settlement`: Ingest authoritative settlement batches into `psp_authoritative_records`.
- `POST /api/v1/ingestion/synthetic-scenario`: Seed synthetic dual-feed datasets with controlled anomaly ratios.

### Reconciliation & Operator Remediation
- `POST /api/v1/reconciliation/run`: Execute a read-only 2-way reconciliation pass (protected by PostgreSQL advisory lock).
- `GET /api/v1/reconciliation/runs`: List historical reconciliation runs.
- `GET /api/v1/reconciliation/discrepancies`: Query and filter anomalies by type, severity, and status.
- `POST /api/v1/reconciliation/discrepancies/{id}/actions/execute`: Execute authorized operator recovery action (`CANCEL_STALE_PENDING`, `SYNC_ORDER_TO_CAPTURED`, `SYNC_STATUS_TO_FAILED`, `FLAG_FOR_OFFLINE_REFUND`). Requires operator auth token header.
- `POST /api/v1/reconciliation/discrepancies/{id}/resolve`: Manually resolve or ignore discrepancy. Requires operator auth token header.

### Analytics & Health
- `GET /api/v1/analytics/summary`: Aggregate KPIs including match rate %, open variance, and defect breakdown.
- `GET /health`: Liveness and PostgreSQL database connectivity probe.

---

## Automated Test Suite

Run the complete test suite:
```bash
pytest -v
```


