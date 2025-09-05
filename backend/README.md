# BondFlow Backend Service — Core API, Business Logic, and Orchestration

Secure, scalable, and auditable backend powering BondFlow: REST APIs, business workflows, data integrity, and orchestration across AI, Blockchain, KYC/AML, Payments, and Market Infra.

## Executive Snapshot
- Responsibilities: Secure APIs, core business logic, service orchestration (AI, Blockchain, KYC/AML, Payments), and data persistence
- Stack: Python 3.11 + FastAPI + PostgreSQL + Redis + Kafka + (MongoDB/Cassandra for high-volume logs) + Alembic + OpenTelemetry
- Patterns: Hexagonal architecture, CQRS-lite, Outbox/Saga, Idempotency, Event-driven integration, Zero-trust service-to-service
- Guarantees: Exactly-once effects at boundaries via idempotency + outbox; evidence-grade audit logs; regulator-ready trails

---

## 0) High-Level Architecture

```mermaid
flowchart LR
  C["Clients (Web/Mobile)"] --> AG["API Gateway / BFF"]
  AG --> AUTH["Auth / OIDC"]
  AG --> UM["User Service"]
  AG --> PM["Portfolio Service"]
  AG --> OM["Order Service"]
  AG --> REP["Reporting API"]

  OM --> AI["AI Gateway (Exec Optimizer)"]
  OM --> SET["Settlement Orchestrator"]
  SET --> PAY["Payments Adapter (UPI/RTGS)"]
  SET --> BC["Blockchain Bridge"]
  PM --> BC

  UM --> KYC["KYC/AML Adapter"]
  KYC -->|status| UM

  subgraph Data Plane
    DB["PostgreSQL (OLTP)"]
    RED["Redis (Cache/Rate Limit/Idempotency)"]
    MQ["Kafka (events)"]
    LOGS["MongoDB/Cassandra (audit/events)"]
  end

  UM --- DB
  PM --- DB
  OM --- DB
  SET --- DB
  AG --- RED
  OM --- MQ
  SET --- MQ
  REP --- LOGS
```

Key principles
- Public API edge is thin; domain services own business logic.
- All cross-service calls are authenticated (mTLS + OAuth2 service tokens) and retried with backoff.
- Every state change emits an event (outbox pattern) for analytics, audit, and downstreams.

---

## 1) Service Breakdown

| Service | Responsibilities | Tech |
|---|---|---|
| API Gateway / BFF | Routing, authN, rate limit, request validation, API versioning | FastAPI or Cloud/API Gateway + Lambda (optional) |
| User Service | Registration, login, RBAC/ABAC, profiles, consents | FastAPI, PostgreSQL, Redis |
| KYC/AML Adapter | Initiate KYC, poll status, store references (no raw PII) | FastAPI, Kafka, external SDKs |
| Portfolio Service | Holdings, PnL, coupon calendar, statements | FastAPI, PostgreSQL (read replicas), Redis |
| Order Service | Order lifecycle, pre-trade checks, AI suggestions, RFQ routing | FastAPI, PostgreSQL, Kafka |
| Settlement Orchestrator | DvP orchestration, saga handling, break resolution | FastAPI worker + Kafka |
| Blockchain Bridge | Token issue/transfer/burn, status/finality, corporate actions | FastAPI, gRPC/HTTP to chain nodes |
| Payments Adapter | UPI/RTGS/NEFT integrations, webhooks, reconciliation | FastAPI, webhooks, secure callbacks |
| Reporting API | Regulator/ops extracts, ISO 20022 feeds, audit export | FastAPI, Mongo/Cassandra/ClickHouse |

---

## 2) API Design Standards

- Versioning: URI-based v1, v2; deprecations with headers (Sunset) and changelog
- Resource naming: kebab-case paths; nouns for resources; verbs in actions subpaths if needed
- Response envelope:
```json
{ "data": { ... }, "error": null, "meta": { "request_id": "bf_01H..." } }
```
- Errors:
```json
{ "data": null, "error": { "code": "BF-ORD-INVALID", "message": "Invalid quantity", "details": {"field":"qty"} }, "meta": { "request_id": "..." } }
```
- Idempotency: Header Idempotency-Key on POST/PUT for mutating endpoints; stored in Redis with result hash
- Pagination: cursor-based; params limit, cursor; include next_cursor in meta
- Sorting/Filtering: whitelisted fields; strict validation; avoid SQL injection with ORM builders
- Auth: OAuth2/OIDC (PKCE for public clients); scopes per route; service-to-service via client credentials
- Rate limiting: token bucket per user/IP/scope; stricter on trading paths

---

## 3) Security & Compliance (Backend)

- mTLS inside cluster; SPIFFE/SPIRE or mutual certs for services
- OPA/Gatekeeper policies for deployments; secrets from Vault/KMS; no secrets in env files committed
- RBAC + ABAC (attributes: residency, investor category, KYC tier); SoD for admin roles
- Audit: append-only event store; WORM storage; Merkle-anchored digests on blockchain (periodic)
- PII: Field-level encryption (FPE/deterministic for joins); no PII in logs; DPDP consent and purpose tags

---

## 4) Domain Models (Schema Sketch)

PostgreSQL (OLTP) — simplified DDL

```sql
-- users
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  phone TEXT UNIQUE,
  password_hash TEXT,
  status TEXT CHECK (status IN ('pending','active','suspended')) DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- kyc_status
CREATE TABLE kyc_status (
  user_id UUID PRIMARY KEY REFERENCES users(id),
  tier TEXT, -- e.g., 'basic','full'
  status TEXT CHECK (status IN ('pending','verified','rejected')),
  kra_ref TEXT, ckyc_no TEXT, updated_at TIMESTAMPTZ
);

-- portfolios
CREATE TABLE portfolios (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  base_ccy TEXT DEFAULT 'INR',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- holdings
CREATE TABLE holdings (
  portfolio_id UUID REFERENCES portfolios(id),
  isin TEXT,
  qty NUMERIC(20,6) NOT NULL,
  avg_clean_price NUMERIC(12,4) NOT NULL,
  primary KEY (portfolio_id, isin)
);

-- orders
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  isin TEXT NOT NULL,
  side TEXT CHECK (side IN ('BUY','SELL')),
  qty NUMERIC(20,6) NOT NULL,
  limit_clean_price NUMERIC(12,4),
  status TEXT CHECK (status IN ('NEW','PENDING','PARTIAL','FILLED','CANCELLED','REJECTED')) DEFAULT 'NEW',
  idempotency_key TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_orders_idem ON orders(idempotency_key) WHERE idempotency_key IS NOT NULL;

-- trades
CREATE TABLE trades (
  id UUID PRIMARY KEY,
  order_id UUID REFERENCES orders(id),
  fill_qty NUMERIC(20,6),
  fill_clean_price NUMERIC(12,4),
  accrued_interest NUMERIC(12,4),
  fees NUMERIC(12,4),
  settled BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- payments
CREATE TABLE payments (
  id UUID PRIMARY KEY,
  user_id UUID,
  type TEXT CHECK (type IN ('ESCROW_DEBIT','ESCROW_CREDIT')),
  amount NUMERIC(14,2),
  status TEXT CHECK (status IN ('INITIATED','CONFIRMED','FAILED')),
  reference TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- outbox (for reliable events)
CREATE TABLE outbox (
  id BIGSERIAL PRIMARY KEY,
  aggregate_type TEXT,
  aggregate_id TEXT,
  event_type TEXT,
  payload JSONB,
  created_at TIMESTAMPTZ DEFAULT now(),
  published BOOLEAN DEFAULT FALSE
);
```

High-volume audit/events → MongoDB/Cassandra
- collections: audit_events, order_events, compliance_actions with TTL/indexing

---

## 5) Events & Topics (Kafka)

- user.created.v1
- kyc.verified.v1
- order.placed.v1
- order.accepted.v1
- order.routed.v1
- rfq.executed.v1
- trade.settled.v1
- payment.instruction.v1
- payment.status.v1
- corporate_action.coupon_paid.v1

Example event (order.placed.v1)
```json
{
  "event_id": "evt_01H...",
  "event_type": "order.placed.v1",
  "occurred_at": "2025-09-05T10:00:00Z",
  "data": {
    "order_id": "ord_01H...",
    "user_id": "usr_01H...",
    "isin": "INE123A07045",
    "side": "BUY",
    "qty": 25.5,
    "limit_clean_price": 99.8
  },
  "meta": { "request_id": "bf_01H...", "schema": "v1" }
}
```

---

## 6) Core Workflows

### 6.1 Onboarding & KYC

```mermaid
sequenceDiagram
  participant C as Client
  participant API as API Gateway
  participant U as User Service
  participant K as KYC Adapter

  C->>API: POST /v1/users (signup)
  API->>U: create user
  U-->>API: 201 user_id
  C->>API: POST /v1/kyc/start
  API->>K: start Aadhaar Offline/PAN checks
  K-->>API: pending (token)
  K-->>U: status=verified/rejected (webhook)
  C->>API: GET /v1/kyc/status
  API->>U: fetch status
  U-->>API: verified
  API-->>C: 200 { status: verified }
```

### 6.2 Order → Best-Ex → DvP Settlement

```mermaid
sequenceDiagram
  participant C as Client
  participant API as API Gateway
  participant O as Order Service
  participant AI as AI Gateway
  participant S as Settlement Orchestrator
  participant P as Payments Adapter
  participant B as Blockchain Bridge

  C->>API: POST /v1/orders (Idempotency-Key)
  API->>O: validate + create NEW
  O->>AI: suggest RFQ/limits + dealer ranking
  AI-->>O: policy + price bands
  O-->>API: 202 accepted (order_id)
  O->>S: order.accepted event

  S->>P: initiate escrow debit (UPI/RTGS)
  P-->>S: payment.status = CONFIRMED
  S->>B: submit token transfer (atomic leg)
  B-->>S: finality OK
  S->>O: update FILLED + create trade
  O-->>API: webhook/WS update to client
```

Notes
- Saga pattern: payment leg and token leg coordinated; compensations on failure (refund/force-cancel)
- Timeouts and retry budgets configured; persistent saga state in DB

---

## 7) Sample Endpoints

- POST /v1/orders
  - Headers: Authorization: Bearer, Idempotency-Key: uuid
  - Body:
```json
{ "isin":"INE123A07045", "side":"BUY", "qty": 10.5, "limit_clean_price": 99.75 }
```
  - Responses:
    - 202 Accepted with order_id
    - 409 Conflict on duplicate Idempotency-Key (returns original result)

- GET /v1/orders?status=PENDING&limit=20&cursor=abc

- GET /v1/portfolio/holdings
  - Returns holdings with clean/dirty, accrued, YTM, duration (server-side computed/memoized)

- POST /v1/kyc/start
  - Starts Aadhaar Offline + PAN verification; returns process token

- POST /v1/payments/webhook
  - Verified via HMAC; processes payment status updates

OpenAPI (fragment)
```yaml
paths:
  /v1/orders:
    post:
      operationId: createOrder
      parameters:
        - in: header
          name: Idempotency-Key
          required: false
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrder'
      responses:
        '202': { description: Accepted, content: { application/json: { schema: { $ref: '#/components/schemas/OrderAck' }}}}
        '400': { $ref: '#/components/responses/BadRequest' }
        '401': { $ref: '#/components/responses/Unauthorized' }
```

---

## 8) Business Rules (Pre-Trade)

- KYC status must be verified; residency/investor category checked
- Position & exposure limits: per user, issuer, rating bucket, duration bucket
- Lockups/eligibility: ERC-1400 partitions; check allowlists
- Price band: sanity check vs reference; prevent fat-finger
- Sufficient escrow balance confirmed prior to route
- Market state: instrument tradable today (holidays, suspensions)

---

## 9) Orchestration Patterns

- Outbox pattern: domain transaction writes event to outbox; publisher relays to Kafka
- Saga (Settlement):
  - Steps: reserve funds → debit success → token transfer → finalize trade
  - Compensations: refund on token failure; unlock on payment failure
- Idempotent consumers: use event_id dedupe table
- Distributed locks: Redis with fencing tokens for critical sections

---

## 10) Performance, Caching, and Scaling

- Horizontal pods for API and worker services; HPA on CPU/RPS/tail latency
- Redis caches:
  - Reference data (ISIN master, ratings) with TTL
  - Precomputed portfolio aggregates (invalidate on trade events)
  - Idempotency results (24h TTL)
- DB:
  - Read replicas for reporting; write-optimized primary
  - Partition high-volume tables by date (orders, trades)
  - Use composite indexes; avoid N+1 with JOINs/CTEs
- Async:
  - Heavy ops offloaded to workers via Kafka
  - WebSockets/SSE for real-time client updates

SLO targets
- p95 API latency < 120 ms (read), < 200 ms (write ack)
- Error rate < 0.5% (5xx)
- Order acceptance to fill median < 3s (subject to venue)

---

## 11) Observability & Ops

- Health: /healthz (liveness), /readyz (dependencies), /metrics (Prometheus)
- Tracing: OpenTelemetry (traceparent); propagate X-Request-ID through all services
- Logging: JSON, structured, PII-scrubbed; correlation IDs; log levels per module
- Dashboards: API latency, error budgets, Kafka lag, DB QPS/locks, payment and chain finality SLAs
- Alerting: multi-burn-rate alerts for SLO breaches; runbooks linked

---

## 12) Security Hardening

- CSP headers at edge; strict CORS; JWT scope validation per endpoint
- Input validation via Pydantic; centralized error mapper
- HMAC verification for webhooks; replay protection (timestamp + nonce)
- Rate-limits per scope and route; slow-down on suspicious patterns
- Secrets via Vault; DB creds dynamic; TLS everywhere; strong cipher suites

---

## 13) Code Snippets

Idempotent order endpoint (FastAPI)
```python
from fastapi import APIRouter, Header, Depends, HTTPException
from pydantic import BaseModel
from .deps import get_user, idem_store, db

router = APIRouter()

class CreateOrder(BaseModel):
  isin: str
  side: str
  qty: float
  limit_clean_price: float | None = None

@router.post("/v1/orders")
async def create_order(req: CreateOrder, idem_key: str | None = Header(default=None), user=Depends(get_user)):
  if idem_key:
    cached = await idem_store.get(idem_key)
    if cached:
      return cached
  # business checks ...
  order_id = await db.orders.create(user.id, req)
  resp = {"data": {"order_id": order_id}, "error": None, "meta": {}}
  if idem_key:
    await idem_store.set(idem_key, resp, ttl=86400)
  return resp
```

SlowAPI rate limit (example)
```python
from slowapi import Limiter
from slowapi.util import get_remote_address
limiter = Limiter(key_func=get_remote_address)
@app.post("/v1/orders")
@limiter.limit("20/minute; 200/hour")
def create_order(...): ...
```

Outbox publisher (simplified)
```python
def publish_outbox_batch(db, producer):
  rows = db.fetch("SELECT id, event_type, payload FROM outbox WHERE published=false LIMIT 500 FOR UPDATE SKIP LOCKED")
  for r in rows:
    producer.produce(topic=r.event_type, value=json.dumps(r.payload))
    db.execute("UPDATE outbox SET published=true WHERE id=%s", (r.id,))
```

---

## 14) Integrations

Payments (UPI)
- Initiate: create collect request/UPI mandate; store payment_id
- Webhook: verify HMAC; update status; emit payment.status.v1
- Reconciliation: daily statements; idempotent upserts

Blockchain Bridge
- Submit transfer: signed tx via node; wait for finality; tx_hash recorded
- Retry policy: exponential backoff; circuit breaker on persistent failures
- Corporate actions: coupon distribution job uses snapshot + payout engine

KYC/AML
- Start flows; track tokens; status via webhook/polling
- Store only references/hashes; PII stays with provider; consent hash saved

Market Data
- Bond master sync; ratings updates; yield curve; calendars (holidays)

---

## 15) Local Dev — Getting Started

Prereqs: Python 3.11, Docker, Make

1) Setup
```bash
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

2) Infra (docker-compose)
```yaml
version: "3.9"
services:
  postgres:
    image: postgres:15
    env_file: .env
    ports: ["5432:5432"]
  redis:
    image: redis:7
    ports: ["6379:6379"]
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment: { ZOOKEEPER_CLIENT_PORT: 2181 }
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
    ports: ["29092:29092"]
```

3) Run
```bash
alembic upgrade head
uvicorn app.main:app --reload --port 8000
```

4) Tests
```bash
pytest -q
```

---

## 16) Configuration (.env example)

```
APP_ENV=dev
DATABASE_URL=postgresql+psycopg2://postgres:postgres@localhost:5432/bondflow
REDIS_URL=redis://localhost:6379/0
KAFKA_BROKERS=localhost:29092
OIDC_ISSUER=https://auth.dev.bondflow.in
OIDC_AUDIENCE=bondflow-api
WEBHOOK_HMAC_SECRET=changeme
PAYMENTS_API_KEY=changeme
BLOCKCHAIN_BRIDGE_URL=http://localhost:9000
```

---

## 17) Testing Strategy

- Unit: business logic, validators, calculators (accrued, dirty price)
- Integration: DB + Kafka + Redis; containerized test env
- Contract: Pact tests with frontend/partners; schema registry for events
- E2E (staging): synthetic user → KYC → order → settlement
- Performance: k6/Locust for hot endpoints; tail latency focus
- Chaos: fault injection on payments/chain; saga resilience tests

---

## 18) Error Codes (sample)

| Code | Meaning | HTTP |
|---|---|---|
| BF-AUTH-001 | Invalid/expired token | 401 |
| BF-KYC-002 | KYC not verified / not eligible | 403 |
| BF-ORD-003 | Invalid order params | 400 |
| BF-ORD-004 | Exposure/position limit breached | 403 |
| BF-ORD-005 | Instrument not tradable | 409 |
| BF-PAY-006 | Payment initiation failed | 502 |
| BF-SET-007 | Settlement timeout | 504 |

---

## 19) Reporting & Compliance

- Exports: CSV/Parquet for trades, positions, TDS, corporate actions
- ISO 20022 mapping for settlement/corporate action messages (internal schema)
- Regulator node feeds via Reporting API; immutable audit packs (hash-anchored)

---

## 20) Roadmap (Backend)

- Phase 1: Core APIs (User/Order/Portfolio), KYC/PAY/AI/BC adapters, outbox+saga, observability v1
- Phase 2: Read replicas, advanced caching, RFQ integrations, corporate actions automation, ISO 20022 feeds
- Phase 3: GraphQL read API, feature flags, canary deployments, cross-region DR
- Phase 4: Policy-as-code (OpenPolicyAgent) for compliance rules, attribute-based permissions UI, continuous controls monitoring

---

## 21) Contribution Guidelines (TL;DR)

- Code: Type hints, docstrings, Black/Ruff; small PRs; meaningful tests
- APIs: Follow OpenAPI; keep response envelope; deprecation plan in PR description
- Data: Migrations via Alembic; no raw PII in fixtures; use fakes
- Security: No secrets in code; threat model changes; run bandit/safety in CI
- Observability: Add traces/metrics; ensure request_id propagation
