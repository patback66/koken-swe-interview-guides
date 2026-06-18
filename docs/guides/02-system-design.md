# System Design — In-Depth Guide

[← Back to Study Guide](../index.md)

---

## Table of Contents

1. [Interview Approach](#interview-approach)
2. [Payments & Transaction Processing](#payments-transaction-processing)
3. [Reconciliation & Billing](#reconciliation-billing)
4. [AI-Enhanced Services](#ai-enhanced-services)
5. [Cross-Cutting Architecture](#cross-cutting-architecture)
6. [Scalability & Reliability Patterns](#scalability-reliability-patterns)
7. [Security & Fintech Compliance](#security-fintech-compliance)
8. [Key Trade-offs](#key-trade-offs)
9. [Diagram Templates](#diagram-templates)
10. [References & Further Reading](#references-further-reading)

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

#### Distributed Transactions — Saga Pattern

Avoid 2PC across microservices. Use **choreography** or **orchestration**:

**Orchestrated Saga (Temporal):**
```
PaymentWorkflow:
  1. ReserveFunds(payment_provider)     → on failure: abort
  2. WriteLedgerEntry(ledger_service)   → on failure: ReleaseFunds (compensate)
  3. SendConfirmation(notify_service)   → on failure: log, retry (non-critical)
  4. EnqueueReconciliation(recon_queue)   → async, eventual
```

Each step is an **activity** with retry policy and timeout. Compensation runs in reverse on failure.

#### Concurrency Control

| Strategy | When to Use |
|----------|-------------|
| **Pessimistic locking** | `SELECT ... FOR UPDATE` on account balance — high contention on same account |
| **Optimistic locking** | Version column; retry on conflict — lower contention, many accounts |
| **Partitioning** | Shard by `account_id` so concurrent payments on different accounts don't contend |

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
| **Network partition** | Circuit breaker on vendor calls; queue for async retry; alert on sustained failures |
| **DB failover** | Multi-AZ RDS/Cloud SQL; connection pooling with retry; RPO < 1 min |

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

**Further reading:** [Stripe — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) · [Microservices.io — Saga](https://microservices.io/patterns/data/saga.html) · [Microservices.io — Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) · [Adyen — Payment Lifecycle](https://docs.adyen.com/online-payments/payment-lifecycle/) · [References — Payments](../references.md#payments-transactions)

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

#### Double-Entry Bookkeeping

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

#### Event-Driven Workflows (Temporal)

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

| Operation | Pattern | Rationale |
|-----------|---------|-----------|
| Payment confirmation to user | **Sync** — return success/failure immediately | User waiting at checkout |
| Ledger write | **Sync** within payment flow | Financial correctness |
| Reconciliation matching | **Async** — batch or stream | Can tolerate minutes of delay |
| Anomaly detection | **Async** — post-reconciliation | AI inference is slow; not user-facing |
| Audit report generation | **Async** — scheduled | End-of-day / end-of-month |

#### Sharding for Scale

| Strategy | Details |
|----------|---------|
| **By tenant/region** | EU invoices in EU shard — data residency compliance |
| **By time** | Monthly partitions for ledger tables — efficient range queries |
| **By entity** | Hash on `account_id` for payment records — even distribution |

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

| Practice | Purpose |
|----------|---------|
| **Canary deployments** | Route 5% traffic to new version; compare error rates |
| **Feature flags** | Enable new matching logic for subset of transactions |
| **Shadow mode** | Run new reconciliation logic in parallel, compare outputs without affecting production |
| **Automated rollback** | Argo Rollouts or similar — auto-revert on SLO breach |
| **DB migration safety** | Expand-contract pattern; backward-compatible schema changes |

**Further reading:** [Temporal — Durable Execution](https://docs.temporal.io/temporal) · [Debezium — CDC](https://debezium.io/documentation/reference/stable/index.html) · [Martin Fowler — Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) · [References — Reconciliation](../references.md#reconciliation-billing)

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

| Technique | Details |
|-----------|---------|
| **Temperature = 0** | Reduce randomness for classification tasks |
| **Structured output** | Force JSON schema response; reject malformed output |
| **Confidence thresholds** | Only auto-act above 0.90; 0.70–0.90 → human review; below 0.70 → reject |
| **Ensemble validation** | LLM + rules engine must agree for auto-approval |
| **Feedback loops** | Human corrections stored; periodic prompt/retrieval tuning |
| **Model versioning** | Pin model versions; A/B test new versions in shadow mode |

#### Multi-Channel Notifications

After validation:
- **Email** — invoice confirmation, discrepancy alert
- **Push/SMS** — payment confirmation (time-sensitive)
- **In-app** — reconciliation status, action required
- **Webhook** — B2B partner integrations

Use an **event-driven notification service** — publish `InvoiceValidated`, `DiscrepancyDetected` events; notification service routes to appropriate channels based on user preferences.

**Further reading:** [RAG Paper](https://arxiv.org/abs/2005.11401) · [Pinecone — RAG Guide](https://www.pinecone.io/learn/retrieval-augmented-generation/) · [OpenAI — Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs) · [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) · [References — AI / LLM](../references.md#ai-llm-in-production)

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

| Pattern | Use Case |
|---------|----------|
| **Event notification** | `PaymentCaptured` → trigger notification, reconciliation |
| **Event-carried state transfer** | Include payment details in event to avoid sync calls |
| **Event sourcing** | Store all state changes as events — full audit trail, replay capability |
| **CQRS** | Separate write model (commands) from read model (queries/dashboards) |

**Further reading:** [*Designing Data-Intensive Applications*](https://dataintensive.net/) · [Kafka Documentation](https://kafka.apache.org/documentation/) · [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) · [References — System Design](../references.md#system-design-architecture)

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

| Metric | Target | How to Achieve |
|--------|--------|----------------|
| **Availability** | 99.99% (~52 min downtime/year) | Multi-AZ, health checks, auto-failover, circuit breakers |
| **Durability** | Zero data loss for financial records | Sync replication, WAL, regular backups with tested restore |
| **Latency (p99)** | < 500ms payment confirmation | Caching, connection pooling, async non-critical paths |

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

### Error Handling

| Pattern | Details |
|---------|---------|
| **Retry with backoff** | Transient failures (network, vendor timeout) — exponential backoff + jitter |
| **Dead letter queue** | Messages that fail max retries → DLQ for manual investigation |
| **Circuit breaker** | Stop calling failing vendor after N failures; periodic half-open probe |
| **Replay** | Re-process events from a checkpoint for recovery after bugs |
| **Eventual consistency** | Accept temporary inconsistency; reconciliation catches drift |

**Further reading:** [Google SRE — Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/) · [OpenTelemetry](https://opentelemetry.io/docs/) · [AWS — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) · [References — Observability](../references.md#observability-reliability)

---

## Security & Fintech Compliance

| Area | Requirements |
|------|-------------|
| **Audit trails** | Immutable log of every financial operation — who, what, when, before/after state |
| **RBAC** | Role-based access — Finance read-only, Ops limited write, Admin full |
| **Encryption** | TLS 1.2+ in transit; AES-256 at rest; KMS-managed keys |
| **PCI-DSS** | Minimize PCI scope — tokenize card data, use certified payment providers |
| **Data residency** | EU customer data in EU region — affects sharding and vendor choice |
| **Vendor integrations** | Direct APIs where possible to reduce fees; webhook signature verification |
| **AI safeguards** | No PII in prompts sent to external LLMs without DPA; log all AI decisions |

**Further reading:** [PCI DSS Documentation](https://www.pcisecuritystandards.org/document_library/) · [OWASP Top 10](https://owasp.org/www-project-top-ten/) · [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) · [References — Security](../references.md#security-compliance-fintech)

---

## Key Trade-offs

| Trade-off | Option A | Option B | When to Choose A | When to Choose B |
|-----------|----------|----------|-------------------|-------------------|
| **Sync vs. async** | Sync confirmation | Async with polling/webhook | User waiting (checkout) | Batch reconciliation, reports |
| **Caching vs. sharding** | Redis cache on single DB | Shard DB by account | < 10k TPS, simpler ops | > 50k TPS, hotspot accounts |
| **2PC vs. saga** | Distributed transaction | Saga with compensation | Rare; only within single DB | Cross-service financial flows |
| **Rules vs. AI** | Deterministic rules engine | LLM classification | Known patterns, regulatory audit | Novel patterns, unstructured data |
| **Monolith vs. microservices** | Single deployable | Service per domain | Early stage, small team | Independent scaling, team ownership |
| **Build vs. buy** | Custom payment gateway | Stripe/Adyen integration | Unique requirements, cost at scale | Speed to market, compliance handled |

### Legacy Migration Without Downtime

```
Phase 1: Strangler — route new traffic to new service via API gateway
Phase 2: Dual-write — write to both old and new systems
Phase 3: Shadow-read — read from new, compare with old, alert on drift
Phase 4: Cutover — switch reads to new system
Phase 5: Decommission — stop dual-write, retire old system
```

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
| **Observability & SRE** | [Google SRE Book](https://sre.google/sre-book/table-of-contents/), [Prometheus Docs](https://prometheus.io/docs/introduction/overview/) |
| **AI in production** | [RAG Paper](https://arxiv.org/abs/2005.11401), [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) |
| **Full bibliography** | [References & Further Reading](../references.md) |
