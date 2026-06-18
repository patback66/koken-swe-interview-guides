# Coding & Technical Depth — In-Depth Guide

[← Sr. SWE, FinTech Study Guide](index.md) · [← All Guides](../index.md)

---

## Table of Contents

1. [Interview Format](#interview-format)
2. [Backend & Systems Patterns](#backend-systems-patterns)
3. [Idempotent Transaction Handler](#idempotent-transaction-handler)
4. [Concurrency Deep Dive](#concurrency-deep-dive)
5. [Database Interactions](#database-interactions)
6. [Caching Strategies](#caching-strategies)
7. [AI / Agentic Integrations](#ai-agentic-integrations)
8. [Production Operations](#production-operations)
9. [Testing in Fintech](#testing-in-fintech)
10. [Practice Problems](#practice-problems)
11. [References & Further Reading](#references-further-reading)

---

## Interview Format

Coding is **~30% of the interview** — expect **light live coding** with heavy emphasis on **verbal walkthroughs** of patterns, edge cases, and production considerations.

| What to Expect | What to Do |
|----------------|------------|
| Whiteboard / shared editor exercise | Talk through your approach before writing code |
| "How would you implement X?" | Describe architecture, then sketch key functions |
| Edge case probes | Proactively mention failures, retries, race conditions |
| Production follow-ups | "How would you test this?" "How would you deploy this?" |

**Key principle:** In fintech, **correctness beats cleverness**. Show you think about idempotency, auditability, and failure modes before optimizing for performance.

---

## Backend & Systems Patterns

### Patterns You Should Know Cold

| Pattern | What it is | Fintech Example |
|---------|------------|-----------------|
| **Idempotency key** | Unique client-provided ID so retries return the same result without re-processing | Payment API receives same charge twice due to timeout — second request is a no-op |
| **Outbox pattern** | Write business record + event to outbox table in one DB transaction; relay publishes to message bus | Ledger write and `PaymentCaptured` event are never out of sync |
| **Saga** | Sequence of local transactions with compensating actions if a step fails | Reserve funds → write ledger → notify; refund if ledger fails |
| **Circuit breaker** | Stop calling a failing service after N errors; probe later for recovery | Payment vendor API down — fail fast instead of 30s timeouts |
| **Retry with backoff** | Retry transient failures with increasing delay (+ jitter) | Network blip to vendor — retry 3x before giving up |
| **Dead letter queue (DLQ)** | Holding queue for messages that failed max retries | Malformed event after 5 attempts — route for manual investigation |
| **Rate limiting** | Cap requests per client/time window (token bucket, sliding window) | 100 requests/min per merchant on payment API |

**Further reading:** [Microservices.io — Saga](https://microservices.io/patterns/data/saga.html) · [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) · [Martin Fowler — Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html) · [References — Resilience Patterns](references.md#resilience-patterns)

---

## Idempotent Transaction Handler

This is the most likely coding exercise. Below is a walkthrough structure — verbalize each layer during the interview.

### Requirements

```
Implement a handler that processes payment transactions idempotently.
- Accept: transaction_id (idempotency key), amount, account_id
- Return: success/failure status
- Duplicate requests with same transaction_id must return the original result
- Handle concurrent duplicate requests safely
```

### Go Implementation Sketch

```go
type TransactionResult struct {
    TransactionID string
    Status        string // "success", "failed"
    Amount        int64
    ProcessedAt   time.Time
}

type TransactionHandler struct {
    db    *sql.DB
    cache *redis.Client
}

func (h *TransactionHandler) Process(ctx context.Context, txID string, amount int64, accountID string) (*TransactionResult, error) {
    // 1. Check idempotency store (fast path)
    if cached, err := h.getCachedResult(ctx, txID); err == nil && cached != nil {
        return cached, nil
    }

    // 2. Acquire distributed lock to handle concurrent duplicates
    lock, err := h.acquireLock(ctx, txID)
    if err != nil {
        return nil, fmt.Errorf("acquire lock: %w", err)
    }
    defer lock.Release(ctx)

    // 3. Double-check after acquiring lock
    if cached, err := h.getCachedResult(ctx, txID); err == nil && cached != nil {
        return cached, nil
    }

    // 4. Process the transaction
    result, err := h.executeTransaction(ctx, txID, amount, accountID)
    if err != nil {
        return nil, err
    }

    // 5. Store result for future idempotent lookups
    if err := h.storeResult(ctx, txID, result); err != nil {
        return nil, fmt.Errorf("store result: %w", err)
    }

    return result, nil
}
```

### C# Implementation Sketch

```csharp
public async Task<TransactionResult> ProcessAsync(
    string transactionId, decimal amount, string accountId, CancellationToken ct)
{
    // 1. Fast path — check cache
    var cached = await _cache.GetAsync<TransactionResult>($"tx:{transactionId}", ct);
    if (cached != null) return cached;

    // 2. Distributed lock
    await using var lockHandle = await _lockProvider.AcquireAsync($"tx:{transactionId}", ct);

    // 3. Double-check
    cached = await _cache.GetAsync<TransactionResult>($"tx:{transactionId}", ct);
    if (cached != null) return cached;

    // 4. Execute within DB transaction
    await using var dbTx = await _db.BeginTransactionAsync(ct);
    var result = await ExecuteTransactionAsync(dbTx, transactionId, amount, accountId, ct);
    await dbTx.CommitAsync(ct);

    // 5. Cache result
    await _cache.SetAsync($"tx:{transactionId}", result, TimeSpan.FromHours(72), ct);
    return result;
}
```

### Edge Cases to Verbalize

| Edge Case | Handling |
|-----------|----------|
| **Concurrent duplicate requests** | Distributed lock (Redis Redlock) or DB unique constraint on `transaction_id` |
| **Crash after processing, before caching** | DB is source of truth — check DB if cache miss |
| **Partial failure** | DB transaction rolls back; return error (client retries with same key) |
| **Invalid amount (negative, zero)** | Validate before processing; return 400 (don't cache errors unless idempotent) |
| **Account not found** | Return error; consider whether to cache failures (usually no — account might be created) |
| **Timeout** | Client retries with same key; server returns cached result or re-processes safely |

**Further reading:** [Stripe — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) · [Martin Kleppmann — Distributed Locking](https://martin.kleppmann.com/2016/08/08/how-to-do-distributed-locking.html)

---

## Concurrency Deep Dive

**Goroutine:** Lightweight thread in Go — cheap to spawn thousands for I/O-bound work.

**Backpressure:** When downstream can't keep up, slow or block upstream producers (bounded channels, semaphores) instead of buffering unbounded and running out of memory.

### Go — Goroutines & Channels

```go
// Worker pool for processing reconciliation batches
func processBatches(batches [][]Record, numWorkers int) []Result {
    jobs := make(chan []Record, len(batches))
    results := make(chan Result, len(batches)*100)

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for batch := range jobs {
                for _, record := range batch {
                    results <- reconcile(record)
                }
            }
        }()
    }

    for _, batch := range batches {
        jobs <- batch
    }
    close(jobs)
    wg.Wait()
    close(results)

    // Collect results...
}
```

**Key concepts:**
- `sync.WaitGroup` — wait for all goroutines to finish
- Buffered channels — backpressure control
- `context.Context` — cancellation propagation
- `sync.Mutex` / `sync.RWMutex` — protect shared state
- **Data races** — use `go run -race` in tests

### C# — Async/Await

```csharp
// Parallel processing with concurrency limit
var semaphore = new SemaphoreSlim(maxConcurrency);
var tasks = records.Select(async record =>
{
    await semaphore.WaitAsync(ct);
    try
    {
        return await ReconcileAsync(record, ct);
    }
    finally
    {
        semaphore.Release();
    }
});
var results = await Task.WhenAll(tasks);
```

**Key concepts:**
- `async/await` — non-blocking I/O without thread-per-request
- `SemaphoreSlim` — limit concurrent operations
- `CancellationToken` — cooperative cancellation
- `Task.WhenAll` — parallel execution with bounded concurrency
- **Deadlocks** — never `.Result` or `.Wait()` on async code in ASP.NET

### When to Use What

| Scenario | Go | C# |
|----------|-----|-----|
| High-throughput I/O | Goroutines (lightweight) | async/await |
| CPU-bound parallelism | `runtime.GOMAXPROCS`, worker pool | `Parallel.ForEach` with limit |
| Shared state protection | `sync.Mutex`, channels | `lock`, `ConcurrentDictionary` |
| Cancellation | `context.Context` | `CancellationToken` |

**Further reading:** [Effective Go](https://go.dev/doc/effective_go) · [Go Blog — Pipelines](https://go.dev/blog/pipelines) · [Async/Await Best Practices (C#)](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming) · [References — Go](references.md#go)

---

## Database Interactions

For schema design, CAP theorem, and technology trade-offs (PostgreSQL vs Redis vs Kafka), see [System Design → Database Design](02-system-design.md#database-design-cap-theorem).

### Locking Strategies

**Why locking?** Two transactions updating the same account balance concurrently can cause lost updates or overdrafts without coordination.

#### Pessimistic Locking

**What it is:** Lock the row when you read it; other transactions **wait** until you commit.

```sql
-- Lock the row for the duration of the transaction
BEGIN;
SELECT balance FROM accounts WHERE id = $1 FOR UPDATE;
-- balance is locked; other transactions on this account wait
UPDATE accounts SET balance = balance - $2 WHERE id = $1;
COMMIT;
```

**Use when:** High contention on same account (e.g., frequent debits on one merchant wallet).

#### Optimistic Locking

**What it is:** Read without locking; on UPDATE, check a **version column** — if someone else changed the row since you read it, the update fails and you retry.

```sql
-- Read with version
SELECT balance, version FROM accounts WHERE id = $1;
-- Application checks business logic
UPDATE accounts
SET balance = $2, version = version + 1
WHERE id = $1 AND version = $3;
-- If rows affected = 0, someone else modified it → retry
```

**Use when:** Low contention, many accounts, reads >> writes.

### Eventual Consistency Models

Distributed systems often can't guarantee all readers see the same data instantly.

| Model | What it means | Fintech Use Case |
|-------|---------------|------------------|
| **Read-your-writes** | After you write, you always see your own write | Payment confirmation page shows the charge you just made |
| **Causal consistency** | Related events appear in order | Refund shows up after the original charge |
| **Eventual consistency** | All replicas converge given time | Reconciliation dashboard may lag minutes behind live payments |
| **CRDTs** | Conflict-free Replicated Data Types — data structures that merge without conflicts | Rare in core ledgers; more common in collaborative or offline-first apps |

### Transaction Isolation Levels

**Isolation** controls what concurrent transactions can see from each other.

| Level | What it prevents | Fintech Impact |
|-------|----------|----------------|
| **Read Uncommitted** | Nothing | Never use for financial data |
| **Read Committed** | Dirty reads | Default for most apps; acceptable for reads |
| **Repeatable Read** | Non-repeatable reads | **Pragmatic default** for many fintech workloads — good for balance checks; handle phantom reads in app logic if needed |
| **Serializable** | All anomalies | Highest safety, lowest throughput — use for hottest ledger paths if anomalies are unacceptable |

**Further reading:** [PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) · [PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) · [References — Database & Locking](references.md#database-locking)

---

## Caching Strategies

**Cache-aside:** App checks cache first; on miss, reads DB and populates cache. On write, update DB then invalidate cache.

### In-Memory vs. Distributed

| Aspect | In-Memory (app-level) | Distributed (Redis) |
|--------|----------------------|---------------------|
| **Latency** | Nanoseconds | Sub-millisecond (network) |
| **Consistency** | Per-instance; stale across pods | Shared across all instances |
| **Use case** | Config, computed values | Session data, idempotency keys, rate limits |
| **Invalidation** | TTL or explicit clear | TTL, pub/sub, or explicit delete |

### Cache-Aside Pattern

```
Read:
  1. Check cache → hit? return
  2. Query DB
  3. Write to cache with TTL
  4. Return

Write:
  1. Write to DB
  2. Invalidate cache (delete key)
  // Next read will populate cache
```

### Cache Stampede Prevention

**Cache stampede:** A popular key expires and hundreds of requests simultaneously hit the database.

| Technique | What it does |
|-----------|--------------|
| **Locking** | First miss acquires a lock; others wait and read from cache after refresh |
| **Probabilistic early expiration** | Refresh cache before TTL with probability based on key age |
| **Stale-while-revalidate** | Return stale value immediately; refresh asynchronously in background |

### Financial Data Caching Rules

- **Never cache balances for write decisions** without invalidation on every write
- **Cache idempotency results** — safe, immutable after creation
- **Cache vendor config** — safe with short TTL + pub/sub invalidation
- **Don't cache PII** in shared caches without encryption

**Further reading:** [Redis — Caching Strategies](https://redis.io/docs/latest/develop/use-cases/caching/) · [AWS — Caching Best Practices](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Strategies.html) · [References — Caching](references.md#caching-performance)

---

## AI / Agentic Integrations

### What is RAG?

**RAG (Retrieval-Augmented Generation):** Instead of asking the LLM from memory alone, first **retrieve** similar historical records from a vector database, then **generate** an answer using that context. Reduces hallucination because the model grounds its response in real data.

### LangChain Pipeline for Anomaly Detection

```python
from langchain.chains import RetrievalQA
from langchain.vectorstores import PGVector
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

# Setup
embeddings = OpenAIEmbeddings()
vectorstore = PGVector(connection_string=DB_URL, embedding_function=embeddings,
                       collection_name="historical_discrepancies")
llm = ChatOpenAI(model="gpt-4", temperature=0)

qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
    return_source_documents=True,
)

def classify_discrepancy(discrepancy: dict) -> dict:
    prompt = f"""
    Classify this financial discrepancy:
    Amount: {discrepancy['amount']}
    Expected: {discrepancy['expected']}
    Vendor: {discrepancy['vendor']}
    Date: {discrepancy['date']}

    Respond in JSON: {{"classification": "...", "confidence": 0.0-1.0, "reasoning": "..."}}
    """
    result = qa_chain.invoke({"query": prompt})

    # Validation layer — never trust raw LLM output
    parsed = parse_structured_output(result)
    if parsed["confidence"] < 0.85:
        return route_to_human_review(discrepancy, parsed)
    if not rules_engine.validate(parsed, discrepancy):
        return route_to_human_review(discrepancy, parsed)

    return auto_resolve(discrepancy, parsed)
```

### RAG for Invoice Matching

```
Pipeline:
1. Parse invoice → structured fields (vendor, amount, date, line items)
2. Generate embedding from concatenated fields
3. Vector search → top-5 similar historical invoices
4. LLM prompt: "Given this invoice and these candidates, which matches? Explain."
5. Structured output validation
6. Rules check: amount within tolerance? date within window? vendor ID match?
7. Decision: auto-approve / human review / reject
```

### Mitigating LLM Unreliability

**Hallucination:** LLM invents facts not in its training or context. In finance, never trust generated dollar amounts — only classify, match, or triage using retrieved evidence.

| Risk | Mitigation |
|------|------------|
| **Hallucination** | RAG with verified documents; never generate financial amounts — only classify/match |
| **Inconsistent output** | `temperature=0`; JSON schema enforcement; reject malformed responses |
| **Latency** | Cache embeddings; batch requests; smaller model for triage |
| **Cost** | Token limits on prompts; cache frequent queries; rules engine handles obvious cases |
| **Prompt injection** | Malicious text in invoice fields tries to override system instructions — sanitize input, separate system/user messages, never execute LLM-suggested SQL or API calls blindly |
| **Audit** | Log prompt, retrieved docs, raw output, parsed output, final decision |

### Validation Loop Pattern

```
┌─────────┐    ┌─────────┐    ┌──────────────┐    ┌────────┐
│  Input  │───►│   LLM   │───►│  Validator   │───►│ Output │
└─────────┘    └─────────┘    └──────┬───────┘    └────────┘
                                     │ fail
                                     ▼
                              ┌──────────────┐
                              │ Retry with   │ (max 2 retries)
                              │ refined prompt│
                              └──────┬───────┘
                                     │ still fail
                                     ▼
                              ┌──────────────┐
                              │ Human review │
                              └──────────────┘
```

**Further reading:** [LangChain Documentation](https://python.langchain.com/docs/introduction/) · [OpenAI — Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs) · [pgvector](https://github.com/pgvector/pgvector) · [References — AI / LLM](references.md#ai-llm-in-production)

---

## Production Operations

### Kubernetes Essentials

```yaml
# Deployment with health checks and resource limits
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: payment-service
        image: payment-service:v1.2.0
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
```

**Verbalize knowledge of:**
- **Liveness vs. readiness** — liveness restarts unhealthy pods; readiness removes from load balancer
- **HPA (Horizontal Pod Autoscaler)** — scale on CPU, memory, or custom metrics (request rate)
- **Resource requests/limits** — prevent noisy neighbor; enable scheduling
- **ConfigMaps / Secrets** — externalize config; never hardcode credentials

### CI/CD Pipeline

```
Code Push → Lint/Test → Build Docker Image → Push to Registry
    → Deploy to Staging → Integration Tests → Deploy to Prod (Canary)
    → Monitor SLOs → Auto-rollback if breach
```

| Tool | Role |
|------|------|
| **Jenkins** | Pipeline orchestration, legacy integrations |
| **Argo CD** | GitOps — desired state in Git, Argo syncs cluster |
| **Argo Rollouts** | Canary/blue-green with automated analysis |
| **Docker** | Consistent build artifacts |

### Temporal Workflows

**Temporal** is a **workflow orchestration** engine (see [System Design → Choreography vs. orchestration](02-system-design.md#choreography-vs-orchestration)). Workflow code is deterministic; all I/O happens in **activities**.

```go
func PaymentWorkflow(ctx workflow.Context, req PaymentRequest) error {
    ao := workflow.ActivityOptions{
        StartToCloseTimeout: 30 * time.Second,
        RetryPolicy: &temporal.RetryPolicy{
            InitialInterval:    time.Second,
            BackoffCoefficient: 2.0,
            MaximumAttempts:    3,
        },
    }
    ctx = workflow.WithActivityOptions(ctx, ao)

    // Step 1: Charge payment provider
    var chargeResult ChargeResult
    err := workflow.ExecuteActivity(ctx, ChargeActivity, req).Get(ctx, &chargeResult)
    if err != nil {
        return err // workflow fails; can be retried from beginning
    }

    // Step 2: Write ledger entry
    err = workflow.ExecuteActivity(ctx, WriteLedgerActivity, chargeResult).Get(ctx, nil)
    if err != nil {
        // Compensate: refund the charge
        _ = workflow.ExecuteActivity(ctx, RefundActivity, chargeResult).Get(ctx, nil)
        return err
    }

    // Step 3: Notify (non-critical — log failure, don't fail workflow)
    _ = workflow.ExecuteActivity(ctx, NotifyActivity, chargeResult).Get(ctx, nil)

    return nil
}
```

**Key concepts:**
- **Workflow** — orchestration logic (deterministic, no I/O directly)
- **Activity** — actual work (API calls, DB writes) with retry policies
- **Compensation** — undo previous steps on failure (saga pattern)
- **Durable execution** — survives worker crashes; resumes from last checkpoint

**Further reading:** [Temporal Documentation](https://docs.temporal.io/) · [Kubernetes Documentation](https://kubernetes.io/docs/home/) · [Argo Rollouts](https://argo-rollouts.readthedocs.io/) · [References — Infrastructure](references.md#infrastructure-devops)

---

## Testing in Fintech

### Test Pyramid for Financial Services

```
        ┌─────────┐
        │   E2E   │  Full payment flow with test vendor sandbox
        ├─────────┤
        │ Integr. │  Service + DB + message queue
        ├─────────┤
        │  Unit   │  Idempotency logic, state machine transitions, validation
        └─────────┘
```

### What to Test

| Layer | Examples |
|-------|----------|
| **Unit** | Idempotency key dedup; amount validation; state machine illegal transitions; LLM output parsing |
| **Integration** | Payment handler + DB + Redis; saga compensation on failure; event publish after DB write |
| **E2E** | Full checkout flow; reconciliation batch produces correct matches; refund flow |
| **Property-based** | `process(tx) + process(tx) == process(tx)` for any transaction |

### Test Patterns

```go
func TestIdempotentTransaction(t *testing.T) {
    handler := setupTestHandler(t)

    // First call succeeds
    result1, err := handler.Process(ctx, "tx-123", 1000, "acct-1")
    require.NoError(t, err)
    assert.Equal(t, "success", result1.Status)

    // Duplicate call returns same result
    result2, err := handler.Process(ctx, "tx-123", 1000, "acct-1")
    require.NoError(t, err)
    assert.Equal(t, result1, result2)

    // Verify only one ledger entry created
    count := countLedgerEntries(t, "tx-123")
    assert.Equal(t, 1, count)
}

func TestConcurrentDuplicateTransaction(t *testing.T) {
    handler := setupTestHandler(t)
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            _, err := handler.Process(ctx, "tx-456", 500, "acct-2")
            assert.NoError(t, err)
        }()
    }
    wg.Wait()

    count := countLedgerEntries(t, "tx-456")
    assert.Equal(t, 1, count)
}
```

### Docker for Local Dev

```yaml
# docker-compose.yml for local fintech stack
services:
  payment-service:
    build: .
    depends_on: [postgres, redis, kafka]
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/payments
      REDIS_URL: redis://redis:6379
      KAFKA_BROKERS: kafka:9092

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: payments
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass

  redis:
    image: redis:7-alpine

  kafka:
    image: confluentinc/cp-kafka:7.5.0
```

**Further reading:** [Martin Fowler — Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html) · [Google Testing Blog](https://testing.googleblog.com/) · [Testcontainers](https://testcontainers.com/) · [References — Testing](references.md#testing)

---

## Practice Problems

### Problem 1: Idempotent Transaction Handler
Implement a handler that processes payments idempotently. Handle concurrent duplicates, failures, and invalid input. Discuss testing strategy.

### Problem 2: Rate Limiter
Implement a token bucket or sliding window rate limiter for a payment API (e.g., 100 requests/minute per merchant). Discuss distributed vs. local implementation.

### Problem 3: Reconciliation Matcher
Given two sorted lists of transactions (internal ledger vs. vendor statement), find matches and discrepancies. Optimize for large datasets. Discuss streaming vs. batch.

### Problem 4: Payment State Machine
Implement a state machine for payment lifecycle (initiated → authorized → captured → settled). Reject invalid transitions. Make it testable.

### Problem 5: LLM Output Validator
Given an LLM response for invoice classification (JSON), write a validator that checks schema, confidence threshold, and business rules before auto-approval.

### Verbal Walkthrough Tips for Each

1. **Clarify inputs/outputs** before coding
2. **Start with the happy path**, then add error handling
3. **Mention concurrency** even if not asked — shows production thinking
4. **Propose tests** as you go — "I'd write a test for the duplicate case here"
5. **Discuss complexity** — time/space for reconciliation matcher
6. **Connect to production** — "In prod, I'd add metrics on cache hit rate and idempotency dedup rate"

---

## References & Further Reading

| Topic | Resources |
|-------|-----------|
| **Idempotency & payments** | [Stripe — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) |
| **Concurrency** | [Effective Go](https://go.dev/doc/effective_go), [Go Blog — Pipelines](https://go.dev/blog/pipelines) |
| **Database locking** | [PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) |
| **Resilience** | [Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html), [*Release It!*](https://pragprog.com/titles/mnee2/release-it-second-edition/) |
| **AI pipelines** | [LangChain Docs](https://python.langchain.com/docs/introduction/), [RAG Paper](https://arxiv.org/abs/2005.11401) |
| **Full bibliography** | [References & Further Reading](references.md) |
