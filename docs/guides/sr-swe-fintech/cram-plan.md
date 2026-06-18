# 48-Hour Cram Plan

[← Sr. SWE, FinTech Study Guide](index.md)

A time-boxed study schedule for the onsite interview. Assumes **~16–20 hours of focused work** across two days, aligned to interview weighting: **system design (50%)**, **coding (30%)**, **behavioral (20%)**.

!!! tip "How to use this plan"
    - **Don't just read** — talk out loud, sketch diagrams, and verbalize code.
    - Mark items `[x]` as you complete them.
    - If you're ahead, spend extra time on mock interviews, not more reading.
    - If you're behind, cut the "stretch" items and protect sleep.

---

## Before You Start (30 min)

- [ ] Read the [Study Guide overview](index.md) — interview format, section weighting, key themes
- [ ] Block calendar time for the next 48 hours (see schedule below)
- [ ] Gather paper/whiteboard or Excalidraw for system design sketches
- [ ] Have 3 project experiences in mind (payments/fintech, AI integration, migration/scale)

| Block | Time | Focus |
|-------|------|-------|
| **Day 1** | ~8–10 hrs | Behavioral stories + system design deep dive |
| **Day 2** | ~8–10 hrs | Coding patterns + timed mocks + review |
| **Evening before interview** | 45 min | Light review only |
| **Morning of interview** | 20 min | Checklist + mindset |

---

## Day 1 — Behavioral + System Design

### Block 1: Behavioral foundations (2 hrs)

**Goal:** Have 3 polished STAR stories ready to deliver in under 3 minutes each.

| Time | Task | Guide |
|------|------|-------|
| 30 min | Read STAR framework + story quality checklist | [Behavioral → STAR](01-behavioral-past-experience.md#the-star-framework) |
| 60 min | **Write** 3 stories (financial services, AI in finance, legacy migration) | [Core Story Themes](01-behavioral-past-experience.md#core-story-themes) |
| 30 min | **Speak** each story aloud; time yourself; add one metric per story | [Builder Culture Signals](01-behavioral-past-experience.md#builder-culture-signals) |

**Done when:** You can answer "Tell me about a time you owned a critical financial system" without notes.

- [ ] Story 1: Ownership / scaling payments or reconciliation
- [ ] Story 2: AI integration with production safeguards
- [ ] Story 3: Legacy migration or orchestration (e.g., Temporal)
- [ ] Each story has Situation, Task, Action, Result + one number

---

### Block 2: Behavioral probes (1 hr)

**Goal:** Pre-answer the hard follow-up questions.

| Time | Task | Guide |
|------|------|-------|
| 30 min | Read Common Probes section; draft bullet answers for each | [Common Probes](01-behavioral-past-experience.md#common-probes-deep-dive) |
| 30 min | Practice 2 follow-up drills per story ("What would you do differently?") | [Preparation Checklist](01-behavioral-past-experience.md#preparation-checklist) |

**Done when:** You have a 30-second answer for sensitive data, AI reliability, and stakeholder collaboration.

- [ ] Sensitive financial data answer ready
- [ ] AI reliability / human-in-the-loop answer ready
- [ ] Non-technical stakeholder answer ready
- [ ] 3 questions prepared to ask interviewers

---

### Block 3: System design — payments (2.5 hrs)

**Goal:** Whiteboard a payment gateway from scratch in 45 minutes.

| Time | Task | Guide |
|------|------|-------|
| 30 min | Read interview approach + clarifying questions | [Interview Approach](02-system-design.md#interview-approach) |
| 60 min | Read payments section: state machine, idempotency, sagas | [Payments](02-system-design.md#payments-transaction-processing) |
| 30 min | **Sketch** architecture diagram from memory (no notes) | [Diagram Templates](02-system-design.md#diagram-templates) |
| 30 min | **Timed mock (45 min):** "Design a payment gateway for 100k TPS" — talk out loud | [Study Guide → Payments](index.md#payments-transaction-processing) |

**Must-know concepts before moving on:**

- [ ] Idempotency key flow (check → process → store)
- [ ] Payment state machine (initiated → authorized → captured → settled)
- [ ] Saga with compensation vs. 2PC
- [ ] Outbox pattern for DB + event publish
- [ ] Sync vs. async paths (user confirmation vs. reconciliation)

**Stretch:** [Stripe — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) (15 min)

---

### Block 4: System design — reconciliation & billing (2 hrs)

**Goal:** Explain a reconciliation pipeline end-to-end.

| Time | Task | Guide |
|------|------|-------|
| 45 min | Read reconciliation section: double-entry, Temporal, sharding | [Reconciliation](02-system-design.md#reconciliation-billing) |
| 45 min | **Sketch** data flow: sources → ingest → match → discrepancy → resolve | [Reconciliation architecture](02-system-design.md#reconciliation-billing) |
| 30 min | List hybrid sync/async decisions for a billing system | [Hybrid Sync/Async](02-system-design.md#hybrid-syncasync) |

**Done when:** You can explain why reconciliation is async but payment confirmation is sync.

- [ ] Double-entry bookkeeping example ready
- [ ] Temporal vs. cron — one sentence why
- [ ] Sharding strategy (by tenant, time, or entity)
- [ ] CI/CD safety: canary, shadow mode, expand-contract migrations

---

### Block 5: System design — AI, reliability, security (2 hrs)

**Goal:** Design AI-assisted invoicing without trusting the LLM blindly.

| Time | Task | Guide |
|------|------|-------|
| 45 min | Read AI-enhanced services + RAG pipeline | [AI-Enhanced Services](02-system-design.md#ai-enhanced-services) |
| 30 min | Read scalability, observability, error handling | [Scalability & Reliability](02-system-design.md#scalability-reliability-patterns) |
| 30 min | Read security & compliance table | [Security](02-system-design.md#security-fintech-compliance) |
| 15 min | Review key trade-offs table | [Trade-offs](02-system-design.md#key-trade-offs) |

**Done when:** You can draw RAG pipeline + validation layer + human review queue.

- [ ] RAG steps: ingest → embed → retrieve → generate → validate → decide
- [ ] Non-determinism mitigations (temperature=0, structured output, confidence thresholds)
- [ ] Three observability dashboards you'd build for fintech
- [ ] PCI scope reduction strategy (tokenization)

**Stretch:** [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) skim (15 min)

---

### Block 5b: Database design & CAP (1 hr)

**Goal:** Explain schema choices and consistency trade-offs under probe.

| Time | Task | Guide |
|------|------|-------|
| 30 min | Read CAP theorem, technology choices, schema design | [Database Design](02-system-design.md#database-design-cap-theorem) |
| 15 min | Sketch `payments` + `ledger_entries` tables; explain why cents as BIGINT | [Schema Design](02-system-design.md#schema-design-for-fintech) |
| 15 min | For each data type (ledger, cache, dashboard), state CP or AP | [CAP in Practice](02-system-design.md#cap-consistency-in-practice) |

- [ ] Can explain C, A, P in one sentence each
- [ ] Can justify PostgreSQL for ledger vs Redis for idempotency vs MongoDB for document data
- [ ] Know why FLOAT is wrong for money
- [ ] Can describe read replica vs sharding trade-off

---

### Day 1 wrap-up (30 min)

- [ ] List your 3 weakest topics from today
- [ ] Sleep 7+ hours — cramming late hurts recall more than one more hour of reading

---

## Day 2 — Coding + Mocks + Review

### Block 6: Coding patterns (1.5 hrs)

**Goal:** Know the patterns cold; explain when to use each.

| Time | Task | Guide |
|------|------|-------|
| 30 min | Read backend patterns table | [Backend Patterns](03-coding-technical-depth.md#backend-systems-patterns) |
| 30 min | Read idempotent transaction handler + edge cases | [Idempotent Handler](03-coding-technical-depth.md#idempotent-transaction-handler) |
| 30 min | **Verbalize** the Go or C# sketch line-by-line without reading | [Implementation sketches](03-coding-technical-depth.md#go-implementation-sketch) |

**Done when:** You can whiteboard idempotent handler with concurrent duplicate handling.

- [ ] Idempotency key, distributed lock, double-check pattern
- [ ] Outbox, saga, circuit breaker, DLQ — one-line purpose each
- [ ] Edge cases: concurrent dupes, crash before cache, partial failure

---

### Block 7: Concurrency, DB, caching (2 hrs)

**Goal:** Answer database and concurrency probes confidently.

| Time | Task | Guide |
|------|------|-------|
| 45 min | Concurrency: Go goroutines + C# async; when to use what | [Concurrency](03-coding-technical-depth.md#concurrency-deep-dive) |
| 45 min | Locking: pessimistic vs. optimistic; isolation levels | [Database](03-coding-technical-depth.md#database-interactions) |
| 30 min | Caching: cache-aside, stampede prevention, fintech rules | [Caching](03-coding-technical-depth.md#caching-strategies) |

- [ ] Explain `SELECT FOR UPDATE` vs. optimistic versioning
- [ ] Why you never cache balances for write decisions
- [ ] Worker pool / semaphore for bounded concurrency

---

### Block 8: AI coding + production ops (1.5 hrs)

**Goal:** Describe an LLM pipeline with validation, not just API calls.

| Time | Task | Guide |
|------|------|-------|
| 45 min | Read AI/agentic section + validation loop | [AI Integrations](03-coding-technical-depth.md#ai-agentic-integrations) |
| 30 min | Read Temporal workflow sketch + K8s essentials | [Production Ops](03-coding-technical-depth.md#production-operations) |
| 15 min | Read testing pyramid + idempotency test example | [Testing](03-coding-technical-depth.md#testing-in-fintech) |

- [ ] Validation loop: LLM → validator → retry or human review
- [ ] Temporal: workflow vs. activity vs. compensation
- [ ] Liveness vs. readiness probes
- [ ] Property-based test idea for idempotency

---

### Block 9: Practice problems — verbal only (1.5 hrs)

**Goal:** Walk through solutions out loud; no IDE required.

| Time | Problem | Guide |
|------|---------|-------|
| 20 min | Idempotent transaction handler | [Problem 1](03-coding-technical-depth.md#problem-1-idempotent-transaction-handler) |
| 20 min | Rate limiter (token bucket) | [Problem 2](03-coding-technical-depth.md#problem-2-rate-limiter) |
| 20 min | Reconciliation matcher (two sorted lists) | [Problem 3](03-coding-technical-depth.md#problem-3-reconciliation-matcher) |
| 15 min | Payment state machine | [Problem 4](03-coding-technical-depth.md#problem-4-payment-state-machine) |
| 15 min | LLM output validator | [Problem 5](03-coding-technical-depth.md#problem-5-llm-output-validator) |

- [ ] All 5 problems verbalized with edge cases and tests mentioned

---

### Block 10: Timed mock interviews (2.5 hrs)

**Goal:** Simulate real interview pressure.

| Time | Mock | Format |
|------|------|--------|
| 50 min | **System design** — "Design reconciliation for vehicle + charging revenue at scale" | 5 min clarify, 35 min design, 10 min probes |
| 35 min | **Behavioral** — "Tell me about ownership of a financial system" + 3 follow-ups | STAR, 2–3 min per answer |
| 45 min | **Coding** — "Implement idempotent payment handler" verbal + edge cases | Talk through Go or C# approach |
| 30 min | Self-debrief: what went well, what to fix tomorrow morning | Notes only |

**Probes to practice answering mid-mock:**

- "What if the payment provider is down?"
- "How do you prevent duplicate charges?"
- "Where would you add observability?"
- "Why not use 2PC?"
- "How do you make the LLM output safe for ledger writes?"

- [ ] System design mock completed
- [ ] Behavioral mock completed
- [ ] Coding verbal mock completed
- [ ] Debrief notes written

---

### Block 11: Weak spots + essential references (1.5 hrs)

**Goal:** Fill gaps only — don't re-read everything.

| Time | Task |
|------|------|
| 45 min | Re-read sections tied to your Day 1 + Day 2 weak spots |
| 45 min | Hit **Essential Reading** only on [References](references.md#essential-reading) |

**Essential links (if you only read 5 things):**

1. [Stripe — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests)
2. [Microservices.io — Saga](https://microservices.io/patterns/data/saga.html)
3. [Microservices.io — Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
4. [Google SRE — Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
5. [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

- [ ] Weak spots addressed
- [ ] Essential references skimmed

---

### Day 2 wrap-up (30 min)

- [ ] Re-read your 3 STAR stories once (don't rewrite)
- [ ] Pack: water, notebook, charger
- [ ] Set alarm; target 7+ hours sleep

---

## Evening Before Interview (45 min max)

**Light review only — no new topics.**

| Time | Activity |
|------|----------|
| 15 min | Re-read [Study Guide](index.md) quick reference table |
| 15 min | Skim your STAR story bullet outlines |
| 15 min | Draw payment gateway diagram from memory once |

- [ ] No cramming past 9 PM if interview is next morning
- [ ] Questions for interviewers written down

---

## Morning of Interview (20 min)

- [ ] 3 STAR stories — can deliver opening line for each
- [ ] Idempotency flow — say it in 30 seconds
- [ ] Saga + outbox — one sentence each
- [ ] RAG + validation — draw on napkin
- [ ] "You build it, you run it" — one personal example
- [ ] 3 questions to ask ready
- [ ] Collaborative mindset: "working session, not exam"

---

## Quick Reference Card

Print or keep this visible during final review:

```
BEHAVIORAL (20%)
  STAR × 3 stories | ownership | incidents | AI safeguards | stakeholders

SYSTEM DESIGN (50%)
  Clarify → diagram → deep dive → trade-offs
  CAP: ledger=CP, cache/dashboard=AP + reconciliation
  DB: Postgres (ACID ledger), Redis (idempotency), Kafka (events)
  Payments: idempotency, state machine, saga, outbox
  Reconciliation: double-entry, async match, Temporal, sharding
  AI: RAG → validate → human review (never trust raw LLM for money)
  Ops: Prometheus/Grafana, tracing, SLOs, 99.99%

CODING (30%)
  Idempotent handler: check cache → lock → double-check → process → store
  Concurrency: worker pools, backpressure, context cancellation
  DB: pessimistic vs optimistic lock, serializable for ledger
  Test: unit (state machine) + integration (DB+Redis) + property (idempotency)
```

---

## If You Have More Time

| Extra time | Use it for |
|------------|------------|
| +1 day | Second system design mock (AI invoicing topic) |
| +2 hrs | [*Designing Data-Intensive Applications*](https://dataintensive.net/) Ch. 7–9 (transactions, consistency) |
| +1 hr | [Stripe Engineering Blog](https://stripe.com/blog/engineering) — pick 2 posts |
| +1 hr | Pair with a friend for a full 45-min mock |

## If You Have Less Time

Cut in this order (lowest ROI first):

1. Stretch reading links
2. Practice problems 4–5 (keep 1–3)
3. Block 8 production ops depth (keep Temporal + K8s probes only)
4. Block 5 security depth (keep PCI + audit trail bullets)

**Never cut:** STAR stories, payment gateway mock, idempotent handler verbal.

---

## Related

| Resource | Link |
|----------|------|
| Study Guide | [index.md](index.md) |
| Behavioral guide | [01-behavioral-past-experience.md](01-behavioral-past-experience.md) |
| System design guide | [02-system-design.md](02-system-design.md) |
| Coding guide | [03-coding-technical-depth.md](03-coding-technical-depth.md) |
| References | [references.md](references.md) |
| All guides | [Guides overview](../index.md) |
