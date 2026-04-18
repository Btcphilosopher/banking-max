# banking max
## Autonomous Unified Runtime for Enterprise Monetary Systems

```
╔═══════════════════════════════════════════════════════════════╗
║  AUREUM CORE  ·  Core Banking Kernel  ·  v1.0.0              ║
║  Double-Entry Ledger  ·  Transaction Processing  ·  Risk     ║
╚═══════════════════════════════════════════════════════════════╝
```

A production-grade core banking engine implementing strict double-entry accounting,
atomic transaction processing, multi-currency operations, and a full risk & compliance
layer — built to the operational standards of a real financial institution.

---

## Architecture

```
aureum_core/
├── ledger/                   ← THE ACCOUNTING KERNEL
│   ├── accounts.py           │  Account domain models & types
│   ├── entries.py            │  Ledger entry models, EntrySet invariant
│   ├── transactions.py       │  Transaction envelope models
│   └── ledger_engine.py      │  LedgerEngine — sole authority for ledger mutation
│
├── transactions/             ← TRANSACTION PIPELINE
│   ├── processor.py          │  TransactionCommitEngine — full processing pipeline
│   ├── validator.py          │  Pre-commit validation gate
│   └── idempotency.py        │  Idempotency key management
│
├── payments/                 ← PAYMENTS INFRASTRUCTURE
│   ├── payment_gateway.py    │  Rail selection & dispatch
│   ├── settlement_engine.py  │  Payment lifecycle management
│   └── rails/
│       ├── faster_payments.py│  UK FPS — domestic GBP (near-instant)
│       ├── sepa.py           │  SEPA SCT / SCT Inst — EUR
│       └── swift.py          │  SWIFT GPI — international (T+2)
│
├── risk/                     ← RISK & COMPLIANCE
│   ├── aml.py                │  AML engine — 5 detection rules
│   ├── fraud_detection.py    │  Fraud engine — velocity & anomaly signals
│   └── limits.py             │  Per-account and daily velocity limits
│
├── currency/                 ← MULTI-CURRENCY
│   ├── fx_engine.py          │  FX conversion with spread modelling
│   └── currency_models.py    │  ISO 4217 currency definitions
│
├── audit/                    ← AUDIT & RECONCILIATION
│   ├── audit_log.py          │  Append-only event log
│   └── reconciliation.py     │  Ledger consistency verification
│
├── api/                      ← API LAYER
│   ├── routes.py             │  FastAPI endpoint definitions
│   ├── schemas.py            │  Pydantic request/response schemas
│   └── auth.py               │  JWT authentication & RBAC
│
├── storage/                  ← PERSISTENCE
│   ├── database.py           │  SQLAlchemy ORM models & session management
│   └── migrations.py         │  DDL supplements: triggers, views, functions
│
└── utils/
    ├── config.py             │  Centralised configuration
    └── logging.py            │  Structured JSON logging
```

---

## Financial Invariants

The system enforces three inviolable accounting invariants:

### 1. The Balance Invariant
Every transaction must satisfy:

```
Σ(debit entries) = Σ(credit entries)
```

This is enforced at three independent layers:
- **Model layer**: `EntrySet` raises `ValueError` on construction if unbalanced
- **Engine layer**: `LedgerEngine.commit_transaction()` re-verifies before persistence
- **Database layer**: PostgreSQL trigger `trg_ledger_entries_immutable` prevents mutation

### 2. The Append-Only Invariant
The ledger is an immutable record of history. Once committed:
- No ledger entry may be `UPDATE`d or `DELETE`d
- The `trg_ledger_entries_immutable` trigger enforces this at the database kernel level
- The same protection applies to the `audit_log` table

### 3. The Derived Balance Invariant
Account balances are **never stored** — they are always computed from the entry set:

```sql
balance = Σ(debit_entries.amount) - Σ(credit_entries.amount)
```

This eliminates the possibility of balance drift and ensures the ledger is always
the single source of truth.

---

## Transaction Pipeline

Every transaction flows through this deterministic pipeline:

```
  Client Request
       │
       ▼
  ┌─────────────────┐
  │ 1. Idempotency  │ ── Duplicate key? Return original result
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 2. Validation   │ ── Account state, currency match, solvency, limits
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 3. AML Screen   │ ── Threshold, structuring, aggregates
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 4. Fraud Screen │ ── Velocity, rapid succession, amount anomaly
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 5. Entry Build  │ ── Construct balanced EntrySet (DEBIT + CREDIT)
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 6. Ledger Commit│ ── Persist transaction + entries (atomic)
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 7. Audit Record │ ── Append immutable audit event
  └────────┬────────┘
           │
           ▼
     TransactionResult
```

---

## Setup Instructions

### Prerequisites
- Python 3.11+
- PostgreSQL 14+ (or Docker)

### 1. Clone and Install

```bash
git clone <repo>
cd aureum_core
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure Environment

```bash
cp .env.example .env
# Edit .env — set DATABASE_URL and JWT_SECRET_KEY
```

### 3. Start PostgreSQL (Docker)

```bash
docker-compose up -d postgres
```

### 4. Run the API

```bash
python -m aureum_core.main
# API available at http://localhost:8000
# Interactive docs: http://localhost:8000/docs
```

### 5. Run Tests

```bash
pytest tests/ -v
# Tests use in-memory SQLite — no PostgreSQL required for testing
```

---

## API Reference

### Authentication

```bash
# Obtain JWT token
POST /v1/auth/token
{
  "username": "operator",
  "password": "aureum_op_2024"
}

# Available users: admin / operator / readonly
```

### Create Account

```bash
POST /v1/accounts/create
Authorization: Bearer <token>

{
  "owner_id": "customer-001",
  "account_type": "asset",
  "currency": "GBP",
  "name": "Current Account"
}
```

### Execute Transfer

```bash
POST /v1/transactions/transfer
Authorization: Bearer <token>

{
  "source_account_id": "<uuid>",
  "destination_account_id": "<uuid>",
  "amount": "250.00",
  "currency": "GBP",
  "reference": "Invoice 1042",
  "idempotency_key": "client-ref-abc-123"
}
```

### Query Balance

```bash
GET /v1/balances/{account_id}
Authorization: Bearer <token>
```

### Fetch Transaction

```bash
GET /v1/transactions/{transaction_id}
Authorization: Bearer <token>
```

### Ledger History

```bash
GET /v1/ledger/{account_id}/entries?limit=50&offset=0
Authorization: Bearer <token>
```

### Reconciliation (admin only)

```bash
GET /v1/admin/reconciliation
Authorization: Bearer <admin_token>
```

---

## Example Usage

```python
import httpx

BASE = "http://localhost:8000/v1"

# Authenticate
r = httpx.post(f"{BASE}/auth/token", json={"username": "operator", "password": "aureum_op_2024"})
token = r.json()["access_token"]
headers = {"Authorization": f"Bearer {token}"}

# Create accounts
r = httpx.post(f"{BASE}/accounts/create", headers=headers, json={
    "owner_id": "A", "account_type": "asset", "currency": "GBP", "name": "Account A"
})
account_a = r.json()["id"]

r = httpx.post(f"{BASE}/accounts/create", headers=headers, json={
    "owner_id": "B", "account_type": "asset", "currency": "GBP", "name": "Account B"
})
account_b = r.json()["id"]

# Transfer
r = httpx.post(f"{BASE}/transactions/transfer", headers=headers, json={
    "source_account_id": account_a,
    "destination_account_id": account_b,
    "amount": "100.00",
    "currency": "GBP",
    "reference": "TEST-001"
})
print(r.json())
# → {"transaction_id": "TXN-...", "status": "COMMITTED", "entry_count": 2, ...}

# Check balances
bal_a = httpx.get(f"{BASE}/balances/{account_a}", headers=headers).json()
bal_b = httpx.get(f"{BASE}/balances/{account_b}", headers=headers).json()
print(f"A: {bal_a['balance']} GBP")  # A: -100.00 GBP (net debit)
print(f"B: {bal_b['balance']} GBP")  # B: -100.00 GBP (net credit surplus = negative balance by convention)
```

**Ledger Trace for 100.00 GBP transfer A → B:**

```
Transaction: TXN-XXXXXXXX-XXXXXXXX  [COMMITTED]
─────────────────────────────────────────────────
Entry 1:  DEBIT   Account A   100.0000000000 GBP
Entry 2:  CREDIT  Account B   100.0000000000 GBP
─────────────────────────────────────────────────
Σ Debits:  100.00 GBP
Σ Credits: 100.00 GBP
Balance:   ZERO  ✓  (invariant satisfied)
```

---

## Risk & Compliance

### AML Rules
| Rule | Trigger | Action |
|------|---------|--------|
| AML-001 | Single transaction ≥ £10,000 | Alert (no block) |
| AML-002 | Structuring: ≥2 sub-threshold transactions in 24h | Block |
| AML-004 | Suspiciously round amounts ≥ £5,000 | Alert |
| AML-005 | 24h aggregate ≥ £50,000 | Alert |

### Fraud Rules
| Rule | Trigger | Action |
|------|---------|--------|
| FRD-001 | ≥10 transactions in 5-minute window | Block |
| FRD-002 | ≥5 transactions in 30 seconds | Block |
| FRD-003 | Amount >10x 30-day average | Alert |

### Limits
| Limit | Value |
|-------|-------|
| Single transaction (GBP) | £1,000,000 |
| Daily velocity (GBP) | £5,000,000 |

---

## Payment Rails

| Rail | Currency | Max Amount | Settlement |
|------|----------|------------|------------|
| Faster Payments | GBP | £1,000,000 | ~2 seconds |
| SEPA Instant | EUR | €100,000 | ~10 seconds |
| SEPA CT | EUR | Unlimited | T+1 |
| SWIFT GPI | Any | Unlimited | T+2 |
| Internal | Any | Unlimited | Instant |

---

## Design Philosophy

AUREUM CORE is built on **Anglo-Futuristic Engineering Principles**:

- **Precision over convenience**: every financial amount uses `Decimal`, never `float`
- **Determinism**: same inputs always produce the same ledger state
- **Formal naming**: `TransactionCommitEngine`, `LedgerInvariantError`, `EntrySet`
- **No balance storage**: balances are mathematical derivations, not state
- **Append-only history**: the ledger accumulates; it never revises
- **Explicit failure**: violations raise typed exceptions, never fail silently
- **Defence in depth**: invariants enforced at model, engine, and database layers

---

## Role Reference

| Role | Permissions |
|------|-------------|
| `admin` | All operations including reconciliation |
| `operator` | Account creation, transfers, queries |
| `readonly` | Account and balance queries only |

---

*AUREUM CORE — sovereign-grade financial infrastructure.*
