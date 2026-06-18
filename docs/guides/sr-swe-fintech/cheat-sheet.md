# FinTech Engineering — Cheat Sheet

[← Study Guide](index.md) · [Full System Design →](02-system-design.md) · [Cram Plan →](cram-plan.md)

One-page reference for last-minute review. For depth, follow the links to full guides.

---

## 1. High-Throughput Ingestion & Payments

**Goal:** Extreme velocity (vehicle orders, charging micro-transactions) with **zero dropped records**.

| Topic | Remember |
|-------|----------|
| **Streaming** | Partitioned event streams (**Kafka**, **Kinesis**). Partition by `user_id` or `transaction_id` → strict ordering per key + horizontal scale |
| **Idempotency** | **Golden rule:** every mutating financial API accepts an idempotency key |
| **3-state idempotency** | `STARTED` → `COMPLETED` or `FAILED`. Duplicate while `STARTED` → **409 Conflict** or hold connection (prevent double-processing) |
| **Payment state machine** | Also know: `INITIATED` → `AUTHORIZED` → `CAPTURED` → `SETTLED` (vendor lifecycle) |
| **Optimistic locking** | Version column — high read / low write contention (billing address updates) |
| **Pessimistic locking** | `SELECT ... FOR UPDATE` — high write contention (prepaid balance drawdown, ledger) |
| **Isolation** | `SERIALIZABLE` = no anomalies, hurts throughput. **`REPEATABLE READ`** = pragmatic default if app handles phantom reads |
| **Money** | Store as **integer cents** / `NUMERIC` — never `FLOAT` |

→ Deep dive: [Payments](02-system-design.md#payments-transaction-processing) · [Database Design](02-system-design.md#database-design-cap-theorem)

---

## 2. Reconciliation, Orchestration & Sagas

**2PC fails** across microservices. Use **sagas** with compensating transactions.

| | Choreography | Orchestration |
|---|--------------|---------------|
| **What** | Services react to bus events | Central controller runs state machine |
| **Pros** | Loose coupling | Visible flow, retries, timeouts |
| **Cons** | Hard to debug at scale | Orchestrator must be HA |
| **Tools** | Kafka consumers | **Temporal**, **Step Functions**, Cadence |

**Compensation:** `Charge_Card` ✓ → `Update_Ledger` ✗ → orchestrator runs `Refund_Card`.

**Also know:** outbox pattern (atomic DB write + event), double-entry ledger, CDC/Debezium for reconciliation pipelines.

→ Deep dive: [Sagas & choreography](02-system-design.md#distributed-transactions--saga-pattern) · [Reconciliation](02-system-design.md#reconciliation-billing)

---

## 3. Observability, Security & Access Control

**"You build it, you run it"** — code isn't done until it's monitored in production.

| Topic | Remember |
|-------|----------|
| **RBAC** | Role-based access — good baseline |
| **Relation-based auth** | **Zanzibar** model (e.g., **SpiceDB**) — fine-grained permissions on hierarchical financial data where RBAC breaks down |
| **High-cardinality metrics** | Tag metrics (region, card brand, API, vendor) — alert on slices: *"Supercharger API error rate, EU, Visa"* |
| **Stack** | Prometheus/Grafana, **Datadog**, distributed tracing (Jaeger/Tempo) |
| **Trace propagation** | Pass `trace_id` (W3C `traceparent`) through API gateway → services → message brokers |
| **Event sourcing** | Append-only financial log — never UPDATE/DELETE finalized records; append reversing entries |
| **PCI** | Tokenize card data; minimize PCI scope |

**Resilience:** circuit breaker, DLQ, retry with exponential backoff + jitter, SLOs / error budgets.

→ Deep dive: [Observability](02-system-design.md#observability-stack) · [Security](02-system-design.md#security-fintech-compliance)

---

## 4. AI-Enhanced Fintech & Agents

**LLMs never autonomously execute financial transactions.**

| Safeguard | What |
|-----------|------|
| **RAG** | Vector DB stores embeddings (policies, tax rules, past invoice resolutions) → agent retrieves context before answering |
| **Deterministic fallback** | JSON schema or confidence threshold fails → **rules engine** takes over |
| **HITL** | Human-in-the-loop — high-value or low-confidence cases: AI drafts, **human clicks approve** |
| **Agentic chain** | Extract invoice → query vector DB for SLA → output structured JSON recommendation (LangChain, etc.) |
| **Never** | Auto-post to ledger from raw LLM output |

→ Deep dive: [AI-Enhanced Services](02-system-design.md#ai-enhanced-services) · [Coding — AI](03-coding-technical-depth.md#ai-agentic-integrations)

---

## 5. Behavioral — Builder Culture

| Trait | How to show it |
|-------|----------------|
| **First principles & pragmatism** | Defend simpler choices — monolith + Postgres beats K8s + Kafka if it solves the problem |
| **End-to-end ownership** | STAR stories: dev → CI/CD → prod → on-call → incident fix |
| **Data-driven outcomes** | Always attach a metric: *"Event-driven orchestration cut API latency 40% and DLQ alerts 50%"* |
| **Trade-off articulation** | Speed vs. correctness, build vs. buy, sync vs. async |

→ Deep dive: [Behavioral guide](01-behavioral-past-experience.md)

---

## Quick Reference Card

```
INGESTION     Kafka/Kinesis partitions by user_id | idempotency 3-state | cents not float
SAGAS         Orchestration > choreography at scale | compensate every step
DATA          CP for ledger | AP for cache/dashboards | outbox + event sourcing
OBSERVABILITY trace_id everywhere | tagged high-cardinality metrics | SLOs
AI            RAG + rules engine + HITL | never auto-move money
BEHAVIORAL    STAR + metrics | own it prod | pragmatism over hype
```

---

## Coverage: Cheat Sheet vs. Full Guides

### ✅ This cheat sheet covers well

- Streaming ingestion & partitioning
- Idempotency (including 3-state model)
- Locking & isolation trade-offs
- Saga choreography vs orchestration
- Observability, tracing, event sourcing for audit
- AI safety rails (RAG, HITL, deterministic fallback)
- Behavioral builder culture

### ⚠️ Critical topics — read the full guides for these

The quick reference above is dense but **incomplete** without the deep guides. Prioritize these if you haven't covered them:

| Topic | Why it matters | Where |
|-------|----------------|-------|
| **CAP theorem & DB choice** | Justify Postgres vs Redis vs DynamoDB under partition | [Database Design](02-system-design.md#database-design-cap-theorem) |
| **Schema design** | `payments`, `ledger_entries`, outbox SQL; constraints | [Schema Design](02-system-design.md#schema-design-for-fintech) |
| **Outbox pattern** | Atomic write + event publish | [Payments](02-system-design.md#architecture-components) |
| **Double-entry bookkeeping** | Debit/credit balance; reconciliation logic | [Reconciliation](02-system-design.md#double-entry-bookkeeping) |
| **CAP / CP vs AP** | What to fail on during partition | [CAP in Practice](02-system-design.md#cap-consistency-in-practice) |
| **Caching** | Cache-aside, stampede, what not to cache | [Coding — Caching](03-coding-technical-depth.md#caching-strategies) |
| **Idempotent handler (code)** | Lock, double-check, store — interview coding | [Coding](03-coding-technical-depth.md#idempotent-transaction-handler) |
| **CI/CD safety** | Canary, shadow mode, expand-contract migrations | [Reconciliation](02-system-design.md#cicd-for-safe-deployments) |
| **Payment vendor lifecycle** | Authorize → capture → settle (beyond 3-state) | [Payments](02-system-design.md#payment-state-machine) |
| **48-hour study schedule** | Structured prep timeline | [Cram Plan](cram-plan.md) |

### 📋 New concepts merged from this cheat sheet into our guides

These were light or missing in the deep guides and are now called out explicitly:

- **3-state idempotency** (`STARTED` / `COMPLETED` / `FAILED` + 409)
- **Kinesis** as streaming alternative
- **Step Functions** as orchestration option
- **Zanzibar / SpiceDB** for relation-based authorization
- **High-cardinality tagged metrics** (Datadog-style slicing)
- **`trace_id` propagation** across gateway, services, brokers
- **First principles / anti-over-engineering** in behavioral prep

---

## Related

| Resource | Link |
|----------|------|
| Study Guide (overview) | [index.md](index.md) |
| System Design (full) | [02-system-design.md](02-system-design.md) |
| Behavioral | [01-behavioral-past-experience.md](01-behavioral-past-experience.md) |
| Coding | [03-coding-technical-depth.md](03-coding-technical-depth.md) |
| References | [references.md](references.md) |
| 48-Hour Cram Plan | [cram-plan.md](cram-plan.md) |
