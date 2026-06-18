# System Design — In-Depth Guide

[← Sr. SWE, FinTech Study Guide](index.md) · [← All Guides](../index.md)

---

## Table of Contents

1. [Interview Approach](#interview-approach)
2. [Payments & Transaction Processing](#payments-transaction-processing)
3. [Reconciliation & Billing](#reconciliation-billing)
4. [AI-Enhanced Services](#ai-enhanced-services)
5. [Cross-Cutting Architecture](#cross-cutting-architecture)
6. [Database Design & CAP Theorem](#database-design-cap-theorem)
7. [Scalability & Reliability Patterns](#scalability-reliability-patterns)
8. [Security & Fintech Compliance](#security-fintech-compliance)
9. [Key Trade-offs](#key-trade-offs)
10. [Diagram Templates](#diagram-templates)
11. [References & Further Reading](#references-further-reading)

---

## Interview Approach

System design is **~50% of the interview**. Treat it as a collaborative design session.

### Recommended Flow (45–60 min)

```
1. Clarify requirements        (5–10 min)
2. High-level architecture     (10 min)
3. Deep dive on core component (15–20 min)
4. Handle probes               (10–15 min)
5. Summarize & trade-offs      (5 min)
```

### Clarifying Questions to Always Ask

| Category | Example Questions |
|----------|-------------------|
| **Scale** | How many TPS? Daily transaction volume? Number of users/regions? |
| **Latency** | What's the p99 SLA for payment confirmation? Is async acceptable for some flows? |
| **Consistency** | Strong consistency required, or eventual consistency with reconciliation? |
| **Failure tolerance** | What's acceptable downtime? RPO/RTO targets? |
| **Compliance** | PCI scope? Data residency requirements? Audit requirements? |
| **Integrations** | How many payment vendors? Existing systems to integrate with? |

**Terms:** **PCI scope** = which systems touch card data and must meet PCI-DSS. **RPO** = max acceptable data loss in a disaster. **RTO** = max acceptable downtime to restore service.

### Communication Tips

- **Think out loud** — explain your reasoning as you draw
- **Start simple** — monolith or few services, then decompose when scale demands it
- **Name your trade-offs** — "I'm choosing X over Y because..."
- **Adapt to feedback** — if interviewer says "what if the DB goes down?", add failover, don't defend the original design

---

## Payments & Transaction Processing

### Problem Statement

Design a real-time payment gateway handling **100k+ TPS** for global vehicle orders, charging services, and subscription billing.

### High-Level Architecture

```
                    ┌─────────────┐
   Clients ────────►│  API Gateway │ (rate limiting, auth, routing)
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────────┐
        │ Payment  │ │ Payment  │ │  Idempotency │
        │ Service  │ │ Validator│ │    Store     │
        └────┬─────┘ └──────────┘ └──────────────┘
             │
             ▼
        ┌──────────┐     ┌─────────────┐
        │  Outbox  │────►│  Event Bus  │ (Kafka / SQS)
        └──────────┘     └──────┬──────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
        ┌──────────┐     ┌──────────┐     ┌──────────────┐
        │ Ledger   │     │ Notify   │     │ Reconcile    │
        │ Service  │     │ Service  │     │ Worker       │
        └──────────┘     └──────────┘     └──────────────┘
```

### Core Concepts

#### Architecture Components

| Component | What it is |
|-----------|------------|
| **API Gateway** | Single entry point for clients — handles auth, rate limiting, routing, and TLS termination before requests hit internal services |
| **Idempotency store** | Fast lookup table (usually Redis) mapping idempotency keys to prior responses so retries are safe |
| **Outbox pattern** | Write business data *and* an outbound event record in the **same DB transaction**; a separate relay process publishes events to the message bus. Guarantees you never commit a payment without also recording the event to publish (avoids "money moved but nobody notified") |
| **Event bus** | Durable message broker (Kafka, SQS) that decouples services — producers publish events, consumers subscribe asynchronously |

#### Payment State Machine

Every payment moves through defined states. Design explicit transitions:

```
INITIATED → AUTHORIZED → CAPTURED → SETTLED
                ↓            ↓
            DECLINED     REFUNDED / PARTIALLY_REFUNDED
                ↓
            FAILED (retryable / non-retryable)
```

- **Idempotent transitions** — retrying `CAPTURE` with the same idempotency key must not double-charge
- **Terminal states** — once SETTLED or REFUNDED, no further transitions without a new transaction

#### Idempotency

**What it is:** An operation that produces the **same result** no matter how many times it's executed with the same input. Critical in payments because networks retry — a client may send the same charge request twice due to a timeout.

**Why it matters:** Without idempotency, a retry becomes a duplicate charge. With idempotency keys, the second request returns the original result without re-processing.

| Approach | Details |
|----------|---------|
| **Client-provided key** | Client sends `Idempotency-Key` header; server stores request hash + response for 24–72hrs |
| **Server-generated key** | For internal retries; keyed on `(merchant_id, order_id, amount, operation)` |
| **Storage** | Redis or dedicated idempotency table with TTL; check before processing, store result after |

```
POST /payments
Headers: Idempotency-Key: uuid-v4

1. Check idempotency store → if exists, return cached response
2. Process payment
3. Store result in idempotency store
4. Return response
```

#### ACID vs. BASE

| Model | What it means | When it applies |
|-------|---------------|-----------------|
| **ACID** | **A**tomicity (all-or-nothing), **C**onsistency (valid state), **I**solation (concurrent txs don't interfere), **D**urability (committed = permanent) | Single-database transactions — ledger writes within one DB |
| **BASE** | **B**asically **A**vailable, **S**oft state, **E**ventual consistency — prioritize availability; replicas may temporarily disagree | Distributed microservices at scale — reconcile drift asynchronously |
| **Exactly-once** | Each operation applied exactly one time — ideal but **very hard** in distributed systems. Practically achieved via **idempotency keys** + **at-least-once delivery** + deduplication | Payment APIs: safe retries without duplicate charges |

#### Distributed Transactions — Saga Pattern

**The problem:** A payment spans multiple services (charge card → write ledger → send receipt). You need all steps to succeed or be undone — but coordinating this across independent databases is hard.

**2PC (two-phase commit):** A central coordinator asks all services to *prepare* (lock resources), then *commit* only if everyone agrees. Provides strong atomicity but creates tight coupling, holds locks, and fails badly when any participant is slow or down. **Rarely used across microservices** in modern fintech — only realistic within a single database.

**Saga:** A sequence of **local transactions**, one per service. Each step commits on its own. If step 3 fails, run **compensating transactions** to undo steps 1 and 2 (e.g., refund the charge if the ledger write fails). Trade-off: the system is temporarily inconsistent between steps — reconciliation catches any drift.

##### Choreography vs. orchestration

These are the two ways to implement a saga:

| | **Choreography** | **Orchestration** |
|---|------------------|-------------------|
| **What it is** | No central brain — each service **reacts to events** and decides its own next action | A central **orchestrator** (workflow engine) **directs** each step and tracks overall state |
| **How it works** | Payment service publishes `PaymentCaptured` → ledger service hears it and writes an entry → notification service hears that and sends email | Orchestrator calls: (1) charge card → (2) write ledger → (3) notify; knows which step failed |
| **Analogy** | Dancers responding to music — everyone knows their cue | A conductor directing each section of the orchestra |
| **Pros** | Loose coupling; no orchestrator dependency | Clear visibility into flow; easy retries, timeouts, and compensation (e.g., Temporal) |
| **Cons** | Hard to see end-to-end flow; debugging "who listens to what?" gets messy as services grow | Orchestrator must be highly available; can become a bottleneck if poorly designed |
| **When to use** | Simple flows, few services, event-native teams | Complex multi-step financial workflows, payments, reconciliation |

**Orchestrated Saga (Temporal)** — Temporal is an **orchestration** tool: it runs workflow code that explicitly sequences steps and handles failure:
```
PaymentWorkflow:
  1. ReserveFunds(payment_provider)     → on failure: abort
  2. WriteLedgerEntry(ledger_service)   → on failure: ReleaseFunds (compensate)
  3. SendConfirmation(notify_service)   → on failure: log, retry (non-critical)
  4. EnqueueReconciliation(recon_queue)   → async, eventual
```

Each step is an **activity** with retry policy and timeout. Compensation runs in reverse on failure.

**Choreographed alternative:** Payment service commits and emits `PaymentCaptured` to Kafka; ledger and notification services each consume that event independently. No workflow engine — but you need clear event contracts and idempotent consumers.

#### Concurrency Control

**Why it matters:** Two requests might try to debit the same account simultaneously. Without coordination, you can overdraw or lose updates.

| Strategy | What it is | When to Use |
|----------|------------|-------------|
| **Pessimistic locking** | Lock the row **before** reading (`SELECT ... FOR UPDATE`) — other transactions wait | High contention on the same account (hot merchant wallet) |
| **Optimistic locking** | Read freely, but UPDATE includes a version check — if someone else modified the row, retry | Many accounts, low collision rate; better throughput |
| **Partitioning (sharding)** | Split data so concurrent payments on *different* accounts hit different DB shards — no lock contention between them | Scale beyond single-node DB limits |

#### Caching Strategy

| Data | Cache | TTL | Invalidation |
|------|-------|-----|--------------|
| Account balance | Redis | Short (seconds) or none for writes | Invalidate on ledger write |
| Payment vendor config | Redis | Minutes | Pub/sub on config change |
| Idempotency keys | Redis | 24–72 hrs | TTL expiry |
| FX rates | Redis | 1–5 min | Scheduled refresh |

**Critical:** Never cache write-path decisions (e.g., "is this payment duplicate?") without strong consistency guarantees.

#### Failure Recovery

| Failure | Recovery |
|---------|----------|
| **Pod crash mid-payment** | Durable workflow (Temporal) resumes from last completed step |
| **Payment provider timeout** | Retry with same idempotency key; query provider status before re-attempting |
| **Network partition** | **Circuit breaker** on vendor calls — after N failures, stop calling and fail fast instead of cascading timeouts; queue for async retry |
| **DB failover** | Multi-AZ RDS/Cloud SQL; connection pooling with retry; RPO < 1 min |

**Circuit breaker:** A wrapper around downstream calls that tracks failures. *Closed* = normal. After a threshold, *opens* = reject calls immediately. After a cooldown, *half-open* = try one request to see if the service recovered.

#### API Design

| Concern | REST | GraphQL |
|---------|------|---------|
| **Vendor integrations** | REST is standard for payment APIs (Stripe, Adyen) | Less common in payments |
| **Internal services** | Simple CRUD, well-understood | Flexible queries for dashboards |
| **Recommendation** | REST for payment operations; GraphQL optional for internal admin/reporting |

Key API properties:
- **Idempotency headers** on all mutating endpoints
- **Webhook callbacks** for async vendor events (with signature verification)
- **Versioning** (`/v1/payments`) for backward compatibility

**Further reading:** [Stripe — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) · [Microservices.io — Saga](https://microservices.io/patterns/data/saga.html) · [Microservices.io — Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) · [Adyen — Payment Lifecycle](https://docs.adyen.com/online-payments/payment-lifecycle/) · [References — Payments](references.md#payments-transactions)

---

## Reconciliation & Billing

### Problem Statement

Design a reconciliation engine matching invoices, payments, and ledger entries across vehicle orders, charging sessions, and subscription billing at massive scale.

### High-Level Architecture

```
  Source Systems                    Reconciliation Pipeline
  ┌─────────────┐
  │ Payment DB  │──┐
  ├─────────────┤  │    ┌──────────┐    ┌─────────────┐    ┌──────────────┐
  │ Invoice DB  │──┼───►│ Ingestion│───►│  Matching   │───►│ Discrepancy  │
  ├─────────────┤  │    │ (CDC/ETL) │    │  Engine     │    │  Resolution  │
  │ Vendor APIs │──┘    └──────────┘    └──────┬──────┘    └──────────────┘
  └─────────────┘                               │
                                                ▼
                                         ┌─────────────┐
                                         │ AI Anomaly  │
                                         │  Detector   │
                                         └─────────────┘
```

### Core Concepts

#### What is Reconciliation?

**Reconciliation** is the process of comparing records from multiple sources (internal ledger, payment vendor statements, invoices) to verify they match. Discrepancies — missing payments, duplicate charges, timing differences — must be detected and resolved. In fintech this often runs **async** (nightly batch or streaming) because perfect real-time match across all systems is impractical.

#### Double-Entry Bookkeeping

**What it is:** An accounting method where every transaction has equal **debit** and **credit** entries — money never appears or disappears, it moves between accounts. The books always balance.

Every financial event has balanced debit/credit entries:

```
Vehicle Purchase:
  Debit:  Accounts Receivable    $50,000
  Credit: Revenue                $50,000

Payment Received:
  Debit:  Cash                   $50,000
  Credit: Accounts Receivable    $50,000
```

Reconciliation verifies: **sum(debits) == sum(credits)** for every period, and external records match internal ledger.

**Ledger immutability:** Financial entries are append-only — corrections are new offsetting entries, not overwrites. Preserves audit trail.

#### Event-Driven Workflows (Temporal)

**What Temporal is:** A **workflow orchestration** engine for durable, long-running processes. Workflow state survives crashes; failed steps retry automatically. Replaces fragile cron jobs that lose progress if a pod dies mid-batch.

Replace brittle cron jobs with durable workflows:

```python
@workflow.defn
class DailyReconciliation:
    @workflow.run
    async def run(self, date: str):
        payments = await workflow.execute_activity(
            fetch_payments, date, start_to_close_timeout=timedelta(minutes=30)
        )
        invoices = await workflow.execute_activity(
            fetch_invoices, date, start_to_close_timeout=timedelta(minutes=30)
        )
        matches = await workflow.execute_activity(
            match_records, payments, invoices, start_to_close_timeout=timedelta(hours=2)
        )
        discrepancies = await workflow.execute_activity(
            detect_discrepancies, matches, start_to_close_timeout=timedelta(minutes=30)
        )
        if discrepancies:
            await workflow.execute_activity(
                route_for_review, discrepancies, start_to_close_timeout=timedelta(minutes=5)
            )
```

**Benefits:** Automatic retries, visibility into workflow state, survives worker crashes.

#### Hybrid Sync/Async

**Sync:** Caller waits for the full operation to complete before getting a response (e.g., user sees "payment approved" at checkout).

**Async:** Work is queued; caller gets acknowledgment immediately and result arrives later via webhook, polling, or email (e.g., nightly reconciliation report).

| Operation | Pattern | Rationale |
|-----------|---------|-----------|
| Payment confirmation to user | **Sync** — return success/failure immediately | User waiting at checkout |
| Ledger write | **Sync** within payment flow | Financial correctness |
| Reconciliation matching | **Async** — batch or stream | Can tolerate minutes of delay |
| Anomaly detection | **Async** — post-reconciliation | AI inference is slow; not user-facing |
| Audit report generation | **Async** — scheduled | End-of-day / end-of-month |

#### Sharding for Scale

**Sharding** splits a database into smaller pieces (shards), each holding a subset of data. Queries route to the correct shard by a key (e.g., `account_id`). Enables horizontal scale when a single DB can't handle write volume.

| Strategy | Details |
|----------|---------|
| **By tenant/region** | EU invoices in EU shard — data residency compliance |
| **By time** | Monthly partitions for ledger tables — efficient range queries |
| **By entity** | Hash on `account_id` for payment records — even distribution |

**CDC (Change Data Capture):** Streams database changes (inserts/updates) to the reconciliation pipeline in near-real-time, instead of polling or nightly dumps. Tools like Debezium read the DB transaction log and publish row changes to Kafka.

#### Anomaly Detection with AI

```
1. Batch reconciliation produces discrepancy set
2. For each discrepancy:
   a. Retrieve similar historical cases (vector DB)
   b. LLM classifies: timing difference / duplicate / fraud / data error
   c. Confidence > threshold → auto-resolve with known pattern
   d. Confidence < threshold → human review queue
3. Human resolution feeds back into training data / prompt tuning
```

### CI/CD for Safe Deployments

| Practice | What it is |
|----------|------------|
| **Canary deployments** | Route a small % of traffic (e.g., 5%) to the new version; monitor error rates before full rollout |
| **Feature flags** | Toggle new logic on/off at runtime without redeploying — enable for a subset of users or transactions |
| **Shadow mode** | Run new code in parallel, compare outputs to production, but **don't** act on results — validates correctness with zero user impact |
| **Automated rollback** | Revert to previous version automatically if error rate or latency breaches an SLO |
| **Expand-contract migrations** | Schema changes in phases: (1) add new column, (2) dual-write to old and new, (3) migrate reads, (4) drop old column — avoids downtime |

**Further reading:** [Temporal — Durable Execution](https://docs.temporal.io/temporal) · [Debezium — CDC](https://debezium.io/documentation/reference/stable/index.html) · [Martin Fowler — Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) · [References — Reconciliation](references.md#reconciliation-billing)

---

## AI-Enhanced Services

### Problem Statement

Design automated invoicing with AI Agents for **10M+ users/transactions** — invoice matching, fraud triage, and anomaly detection.

### Architecture

```
  ┌────────────┐     ┌──────────────┐     ┌─────────────┐
  │  Invoice   │────►│  Embedding   │────►│  Vector DB  │
  │  Ingestion │     │  Pipeline    │     │  (Pinecone/ │
  └────────────┘     └──────────────┘     │   pgvector) │
                                            └──────┬──────┘
                                                   │
  ┌────────────┐     ┌──────────────┐              │
  │  New       │────►│  LangChain   │◄─────────────┘
  │  Invoice   │     │  Agent       │
  └────────────┘     └──────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Rules    │  │ Human    │  │ Notify   │
        │ Engine   │  │ Review Q │  │ Service  │
        └──────────┘  └──────────┘  └──────────┘
```

### Core Concepts

#### Key AI Terms

| Term | What it is |
|------|------------|
| **LLM** | Large Language Model — predicts text; useful for classification and matching but non-deterministic (same input can yield different output) |
| **Embedding** | A numeric vector representing meaning of text — similar invoices have similar vectors |
| **Vector DB** | Database optimized for similarity search on embeddings (Pinecone, pgvector) — "find invoices like this one" |
| **RAG** | Retrieval-Augmented Generation — retrieve relevant documents first, then give them to the LLM as context so answers are grounded in real data, not hallucinated |
| **LangChain** | Framework for chaining LLM steps (retrieve → prompt → parse → validate) into pipelines or agents |

#### RAG Pipeline for Invoice Matching

```
1. Ingestion: Parse invoice (PDF/OCR) → extract structured fields
2. Embedding: Generate vector from (vendor, amount, line items, date)
3. Retrieval: Query vector DB for top-K similar historical invoices
4. Generation: LLM proposes match with reasoning, given retrieved context
5. Validation: Rules engine checks amount tolerance, date range, vendor ID
6. Decision: Auto-approve / escalate / reject
```

#### Addressing Non-Determinism

LLMs are **non-deterministic** — they can give different answers to the same question. In finance you cannot auto-post ledger entries from raw LLM output.

| Technique | What it does |
|-----------|--------------|
| **Temperature = 0** | Reduces randomness in model output — prefer for classification |
| **Structured output** | Force response as JSON matching a schema; reject malformed output |
| **Confidence thresholds** | Only auto-act above 0.90; 0.70–0.90 → human review; below 0.70 → reject |
| **Ensemble validation** | LLM + rules engine must both agree before auto-approval |
| **Human-in-the-loop** | Low-confidence or high-stakes decisions route to a human queue |
| **Feedback loops** | Human corrections stored; periodic prompt/retrieval tuning |
| **Model versioning** | Pin model versions; A/B test new versions in shadow mode |

#### Multi-Channel Notifications

After validation:
- **Email** — invoice confirmation, discrepancy alert
- **Push/SMS** — payment confirmation (time-sensitive)
- **In-app** — reconciliation status, action required
- **Webhook** — B2B partner integrations

Use an **event-driven notification service** — publish `InvoiceValidated`, `DiscrepancyDetected` events; notification service routes to appropriate channels based on user preferences.

**Further reading:** [RAG Paper](https://arxiv.org/abs/2005.11401) · [Pinecone — RAG Guide](https://www.pinecone.io/learn/retrieval-augmented-generation/) · [OpenAI — Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs) · [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) · [References — AI / LLM](references.md#ai-llm-in-production)

---

## Cross-Cutting Architecture

### Microservices (Go / C#)

| Service | Responsibility | Language Choice Rationale |
|---------|---------------|--------------------------|
| Payment Service | Process payments, manage state machine | Go — high throughput, low latency |
| Ledger Service | Double-entry bookkeeping, balance queries | Go or C# — strong typing for financial precision |
| Reconciliation Worker | Batch/stream matching | Go — concurrency primitives |
| Notification Service | Multi-channel delivery | C# — rich ecosystem for integrations |
| AI Agent Service | LLM orchestration, RAG | Python or Go — LangChain ecosystem |

### Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Containers | Docker | Consistent dev/prod environments |
| Orchestration | Kubernetes | Auto-scaling, self-healing, rolling deploys |
| Workflow engine | Temporal | Durable workflows for payments, reconciliation |
| Batch orchestration | Airflow | Scheduled ETL, report generation |
| Message bus | Kafka / SQS | Event-driven communication between services |

### Event Bus Patterns

| Pattern | What it is |
|---------|------------|
| **Event notification** | Lightweight signal that something happened — consumers fetch details if needed (`PaymentCaptured` triggers reconciliation) |
| **Event-carried state transfer** | Event payload includes full data so consumers don't need a callback (`PaymentCaptured` includes amount, account_id, timestamp) |
| **Event sourcing** | Store every state change as an immutable event log; current state is derived by replaying events. Full audit trail; can rebuild state or debug "what happened?" |
| **CQRS** | Command Query Responsibility Segregation — **separate models for writes** (commands that change state) and **reads** (optimized query views/dashboards). Write path enforces rules; read path can be denormalized for speed |

**Further reading:** [*Designing Data-Intensive Applications*](https://dataintensive.net/) · [Kafka Documentation](https://kafka.apache.org/documentation/) · [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) · [References — System Design](references.md#system-design-architecture)

---

## Database Design & CAP Theorem

Database choices drive consistency, scale, and operational complexity. In fintech interviews, be ready to justify **what you store where**, **how schemas enforce correctness**, and **what you sacrifice under failure**.

### CAP Theorem

**CAP** states that a distributed data store can guarantee at most **two** of three properties during a **network partition**:

| Property | What it means |
|----------|---------------|
| **Consistency (C)** | Every read returns the latest write (or an error) — all nodes agree on the same data |
| **Availability (A)** | Every request gets a response (not an error) — system stays up even if some nodes are stale |
| **Partition tolerance (P)** | System continues operating when network links between nodes fail |

**The reality:** Partitions **will** happen in distributed systems (AZ failure, network blip, split brain). So you are really choosing between **CP** and **AP** when things go wrong:

| Choice | Behavior during partition | Fintech example |
|--------|-------------------------|-----------------|
| **CP** | Reject reads/writes rather than return stale data | Ledger balance query — return error if primary is unreachable; don't guess |
| **AP** | Stay available; replicas may temporarily disagree | Payment status dashboard — show "processing" while replicas catch up; reconciliation fixes drift |

**Important nuance:** CAP is about **distributed** stores under partition. A single PostgreSQL instance gives you ACID consistency without this trade-off until you add replicas or shard. Most fintech cores use **strong consistency for money** (CP-ish within the ledger DB) and **eventual consistency at the edges** (AP with reconciliation for analytics, caches, read replicas).

```
                    ┌─────────────────────────────────┐
                    │         Network partition         │
                    └─────────────────────────────────┘
           CP choice                          AP choice
    "I'd rather error than lie"        "I'd rather serve stale than go down"
         Ledger write fails               Cache shows old balance
         Client retries safely            Reconciliation catches up later
```

### Technology Choices

| Technology | Type | Best for in fintech | Trade-offs |
|------------|------|---------------------|------------|
| **PostgreSQL** | Relational (SQL) | Ledger, payments, invoices — anything needing **ACID**, joins, constraints | Vertical scale limits; sharding is manual; excellent for correctness-first workloads |
| **MySQL / Aurora** | Relational (SQL) | Same as Postgres; common in AWS-native stacks | Similar trade-offs; Aurora adds managed replication/failover |
| **Redis** | In-memory key-value | Idempotency keys, rate limits, session cache, distributed locks | Not durable by default (use AOF/RDB); memory cost; not a system of record |
| **Kafka** | Distributed log | Event bus, CDC stream, audit trail of events | Not a queryable database — consumers build views |
| **DynamoDB / Cassandra** | Wide-column NoSQL | High-volume idempotency at scale, time-series events, global low-latency writes | Eventual consistency by default; no joins; schema design is access-pattern-first |
| **pgvector / Pinecone** | Vector store | Similarity search for RAG (invoice matching) | Complements OLTP DB; not for transactional money |
| **Snowflake / BigQuery** | Warehouse | Analytics, reconciliation reports, financial close | Not for OLTP; batch/near-real-time ingestion |

**Default fintech stack (interview-safe):** PostgreSQL as system of record + Redis for ephemeral/fast lookups + Kafka for events. Add DynamoDB or sharding only when Postgres write TPS or global latency forces it.

#### SQL vs. NoSQL — when to use which

| Use SQL when | Use NoSQL when |
|--------------|----------------|
| Data has relationships (accounts → payments → ledger entries) | Access pattern is simple key-value or wide-column (e.g., `idempotency_key → response`) |
| You need transactions across rows (double-entry) | You need horizontal write scale across regions with tunable consistency |
| Schema integrity matters (foreign keys, CHECK constraints) | Schema is flexible or evolves rapidly (event payloads) |
| You run complex queries (reconciliation joins) | You sacrifice ad-hoc queries for partition tolerance and throughput |

**Rule of thumb for money:** If losing or corrupting a row costs real dollars, prefer **SQL with ACID** unless you have a specific scale problem and a reconciliation story.

### Schema Design for Fintech

Design schemas to make **incorrect states unrepresentable** where possible.

#### Core tables (payments + ledger)

```sql
-- Payments: lifecycle state machine
CREATE TABLE payments (
    id              UUID PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    account_id      UUID NOT NULL REFERENCES accounts(id),
    amount_cents    BIGINT NOT NULL CHECK (amount_cents > 0),
    currency        CHAR(3) NOT NULL,
    status          VARCHAR(32) NOT NULL,  -- INITIATED, AUTHORIZED, CAPTURED, ...
    version         INT NOT NULL DEFAULT 1, -- optimistic locking
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Ledger: append-only double-entry
CREATE TABLE ledger_entries (
    id              UUID PRIMARY KEY,
    transaction_id  UUID NOT NULL,
    account_id      UUID NOT NULL,
    debit_cents     BIGINT NOT NULL DEFAULT 0 CHECK (debit_cents >= 0),
    credit_cents    BIGINT NOT NULL DEFAULT 0 CHECK (credit_cents >= 0),
    CHECK (debit_cents = 0 OR credit_cents = 0),  -- one side only
    CHECK (NOT (debit_cents = 0 AND credit_cents = 0)),
    currency        CHAR(3) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
    -- no updated_at: entries are immutable
);

-- Outbox: atomic publish with business write
CREATE TABLE outbox (
    id              UUID PRIMARY KEY,
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(128) NOT NULL,
    payload         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at    TIMESTAMPTZ  -- NULL until relay confirms
);
```

#### Schema design principles

| Principle | Why it matters |
|-----------|----------------|
| **Store money as integers (cents)** | Never `FLOAT`/`DOUBLE` — rounding errors compound at scale |
| **Immutable ledger rows** | Corrections are new entries, not UPDATEs — audit trail stays intact |
| **Idempotency key as UNIQUE constraint** | DB enforces dedup even if application logic fails |
| **Optimistic `version` column** | Safe concurrent updates without long-held locks |
| **Status as enum / CHECK constraint** | Illegal state transitions rejected at DB layer |
| **Timestamps in UTC (`TIMESTAMPTZ`)** | Global ops; convert at display layer |
| **Partition large tables by time** | `ledger_entries_2026_01` — efficient range scans for reconciliation |

#### Normalization vs. denormalization

| Approach | When to use |
|----------|-------------|
| **Normalized (3NF)** | Write path — payments, ledger, accounts. Avoid update anomalies |
| **Denormalized read models** | CQRS dashboards — `daily_revenue_by_region` materialized view updated from events |
| **Denormalize cautiously** | Only when read latency matters and you have a clear invalidation/rebuild path |

### Database Architecture Patterns

| Pattern | What it is | Fintech use |
|---------|------------|-------------|
| **Database per service** | Each microservice owns its schema; no cross-service JOINs | Payment service DB, ledger service DB — communicate via APIs/events |
| **Shared database (anti-pattern)** | Multiple services write same tables | Avoid — coupling, schema conflicts, unclear ownership |
| **Read replicas** | Async copy of primary for SELECT queries | Dashboards, reporting — accept replication lag (eventual consistency) |
| **Sharding** | Split rows across DB instances by key | Scale writes when single Postgres maxes out (~10k+ sustained write TPS depending on workload) |
| **Connection pooling (PgBouncer)** | Reuse DB connections | Essential at scale — pods don't open 1 connection each unbounded |

### CAP + Consistency in Practice

Map data types to consistency expectations:

| Data type | Consistency model | Technology | On partition / failure |
|-----------|-------------------|------------|------------------------|
| Ledger balance | Strong (linearizable within shard) | PostgreSQL primary | Fail writes; don't serve stale balance |
| Payment idempotency key | Strong dedup | PostgreSQL UNIQUE or Redis SETNX | Reject duplicate or return cached |
| Payment status (user UI) | Read-your-writes | Primary or sticky session to primary | Brief error → retry |
| Reconciliation dashboard | Eventual | Read replica or warehouse | Stale by minutes is OK |
| Cache (vendor config) | Eventual | Redis with TTL | Stale config OK for seconds |
| Event log | Ordered per partition | Kafka | Consumers track offset; replay on failure |

### Key Trade-offs (Database)

| Trade-off | Option A | Option B | Fintech guidance |
|-----------|----------|----------|------------------|
| **One DB vs. many** | Monolith DB | DB per service | Per-service for microservices; shared only during migration |
| **Postgres vs. DynamoDB** | Relational ACID | Managed NoSQL scale | Postgres until proven write/latency need; Dynamo for specific hot paths |
| **Sync replication vs. async** | Zero data loss (RPO=0) | Lower write latency | Sync for ledger primary; async OK for replicas |
| **FK constraints vs. app-level** | DB enforces referential integrity | Service validates | Prefer FKs on ledger/payments — catch bugs early |
| **Soft delete vs. hard delete** | `deleted_at` column | `DELETE` row | Financial records: **never hard delete** — soft delete or append-only |
| **JSONB columns vs. rigid schema** | Flexible payloads (events, metadata) | All columns typed | JSONB for outbox/events; core money fields always typed columns |

**Further reading:** [*DDIA* Ch. 5–9](https://dataintensive.net/) · [PostgreSQL Data Types — Numeric](https://www.postgresql.org/docs/current/datatype-numeric.html) · [CAP Twelve Years Later](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) · [References — Database Design](references.md#database-design)

---

## Scalability & Reliability Patterns

### Scaling Strategies

| Layer | Strategy |
|-------|----------|
| **API** | Horizontal pod autoscaling on CPU/request rate; CDN for static assets |
| **Database** | Read replicas, connection pooling (PgBouncer), sharding by account/region |
| **Cache** | Redis cluster; cache-aside pattern; avoid cache stampede with locking |
| **Queue** | Partitioned Kafka topics by entity; consumer groups for parallelism |
| **Workflows** | Temporal task queues per domain; scale workers independently |

### Reliability Targets

| Metric | What it means | Target | How to Achieve |
|--------|---------------|--------|----------------|
| **Availability** | % of time the service is up | 99.99% (~52 min downtime/year) | Multi-AZ, health checks, auto-failover, circuit breakers |
| **Durability (RPO)** | Recovery Point Objective — max data loss in a disaster | Zero for financial records | Sync replication, WAL, tested backups |
| **Recovery time (RTO)** | Recovery Time Objective — max time to restore service | Minutes | Automated failover, runbooks |
| **Latency (p99)** | 99th percentile response time | < 500ms payment confirmation | Caching, connection pooling, async non-critical paths |

### Observability Stack

```
Metrics (Prometheus)  →  Dashboards (Grafana)  →  Alerting (PagerDuty)
        +
Tracing (Jaeger/Tempo)  →  Distributed request visibility
        +
Logging (ELK/Loki)  →  Structured JSON logs, correlation IDs
        +
SLOs  →  Error budgets, burn-rate alerts
```

**Key dashboards for fintech:**
- Payment success rate (by vendor, region, payment method)
- Reconciliation lag (time since last successful run)
- Discrepancy count and resolution time
- Idempotency cache hit rate
- LLM inference latency and confidence distribution

**SLOs (Service Level Objectives):** Target reliability (e.g., 99.9% success rate). **Error budget** = allowed failures before you must stop shipping features and fix reliability.

### Error Handling

| Pattern | What it is |
|---------|------------|
| **Retry with backoff** | On transient failure, wait and retry — delay increases exponentially (+ jitter) to avoid hammering a recovering service |
| **Dead letter queue (DLQ)** | Queue for messages that failed processing after max retries — humans or tools investigate poison messages |
| **Circuit breaker** | Stop calling a failing downstream after N failures; probe periodically to detect recovery |
| **Replay** | Re-process events from a checkpoint after fixing a bug — rebuilds state from the event log |
| **Eventual consistency** | Replicas may temporarily disagree; given time (and reconciliation) they converge. Acceptable for dashboards; not for in-flight payment decisions |

**Further reading:** [Google SRE — Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/) · [OpenTelemetry](https://opentelemetry.io/docs/) · [AWS — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) · [References — Observability](references.md#observability-reliability)

---

## Security & Fintech Compliance

| Area | What it is / Why it matters |
|------|----------------------------|
| **Audit trails** | Immutable log of every financial operation — who, what, when, before/after state. Required for compliance and dispute resolution |
| **RBAC** | Role-Based Access Control — users get permissions by role (Finance = read-only, Ops = limited write). **Least privilege** = minimum access needed |
| **Encryption** | TLS in transit; AES-256 at rest. **KMS** manages encryption keys with rotation |
| **PCI-DSS** | Payment Card Industry Data Security Standard — rules for handling card data. **Reduce scope** by tokenizing cards (Stripe/Adyen holds the PAN; you store a token) |
| **Data residency** | Some regions require customer data stored in-country (e.g., EU). Affects where you shard and which vendors you use |
| **Vendor integrations** | Direct APIs to payment providers; verify **webhook signatures** so attackers can't forge payment events |
| **AI safeguards** | Don't send PII to external LLMs without a DPA; log every AI-assisted decision for audit |

**Further reading:** [PCI DSS Documentation](https://www.pcisecuritystandards.org/document_library/) · [OWASP Top 10](https://owasp.org/www-project-top-ten/) · [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) · [References — Security](references.md#security-compliance-fintech)

---

## Key Trade-offs

| Trade-off | Option A | Option B | When to Choose A | When to Choose B |
|-----------|----------|----------|-------------------|-------------------|
| **Sync vs. async** | Sync confirmation | Async with polling/webhook | User waiting (checkout) | Batch reconciliation, reports |
| **Caching vs. sharding** | Redis cache on single DB | Shard DB by account | < 10k TPS, simpler ops | > 50k TPS, hotspot accounts |
| **SQL vs. NoSQL** | PostgreSQL ACID ledger | DynamoDB hot-path idempotency | Correctness-critical money flows | Proven horizontal write scale need |
| **CP vs. AP** | Fail on partition (ledger) | Stay available (cache/dashboard) | Balances, captures, idempotency | Analytics, config cache, read replicas |
| **2PC vs. saga** | Distributed transaction | Saga with compensation | Rare; only within single DB | Cross-service financial flows |
| **Rules vs. AI** | Deterministic rules engine | LLM classification | Known patterns, regulatory audit | Novel patterns, unstructured data |
| **Monolith vs. microservices** | Single deployable | Service per domain | Early stage, small team | Independent scaling, team ownership |
| **Build vs. buy** | Custom payment gateway | Stripe/Adyen integration | Unique requirements, cost at scale | Speed to market, compliance handled |

### Legacy Migration Without Downtime

**Strangler fig pattern:** Gradually replace a legacy system by routing increasing traffic to the new service — like a fig tree growing around an old trunk until the old system can be removed.

```
Phase 1: Strangler — route new traffic to new service via API gateway
Phase 2: Dual-write — write to both old and new systems
Phase 3: Shadow-read — read from new, compare with old, alert on drift
Phase 4: Cutover — switch reads to new system
Phase 5: Decommission — stop dual-write, retire old system
```

**Dual-write:** Both systems receive the same data during migration so you can compare outputs before cutting over.

---

## Diagram Templates

### Component Diagram (start here)

```
[Client] → [API Gateway] → [Service A] → [DB]
                              ↓
                         [Event Bus] → [Service B] → [DB]
                              ↓
                         [Service C]
```

### Data Flow Diagram (for reconciliation)

```
[Source A] ──┐
[Source B] ──┼──► [Ingest] ──► [Transform] ──► [Match] ──► [Resolve]
[Source C] ──┘                                                      │
                                                                    ▼
                                                              [Report/Audit]
```

### Sequence Diagram (for payment with idempotency)

```
Client          API Gateway       Payment Svc       Idempotency Store    Vendor
  │                 │                 │                    │               │
  │── POST /pay ───►│                 │                    │               │
  │  Idempotency-Key│                 │                    │               │
  │                 │── forward ─────►│                    │               │
  │                 │                 │── check key ──────►│               │
  │                 │                 │◄── miss ────────────│               │
  │                 │                 │── charge ──────────────────────────►│
  │                 │                 │◄── success ─────────────────────────│
  │                 │                 │── store result ───►│               │
  │                 │◄── 200 OK ──────│                    │               │
  │◄── 200 OK ──────│                 │                    │               │
```

---

## References & Further Reading

| Topic | Resources |
|-------|-----------|
| **Foundational** | [*Designing Data-Intensive Applications*](https://dataintensive.net/), [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| **Payments & idempotency** | [Stripe — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests), [Stripe Engineering Blog](https://stripe.com/blog/engineering) |
| **Distributed transactions** | [Saga Pattern](https://microservices.io/patterns/data/saga.html), [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html), [Temporal Docs](https://docs.temporal.io/) |
| **Database design & CAP** | [*DDIA*](https://dataintensive.net/), [CAP — Twelve Years Later](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) |
| **Observability & SRE** | [Google SRE Book](https://sre.google/sre-book/table-of-contents/), [Prometheus Docs](https://prometheus.io/docs/introduction/overview/) |
| **AI in production** | [RAG Paper](https://arxiv.org/abs/2005.11401), [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) |
| **Full bibliography** | [References & Further Reading](references.md) |
