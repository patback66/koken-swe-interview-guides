# Onsite Interview Study Guide

> **Role:** Sr. Software Engineer, FinTech

## Overview

This guide covers key topics for your onsite interview with the company's FinTech team, focusing on building scalable systems for **payments processing**, **financial reconciliation**, **billing/invoicing**, and **AI-driven workflows** that handle global transaction volumes and impact billions in revenue (e.g., from vehicle orders, charging services, and emerging mobility operations).

### Interview Format

| Aspect | Details |
|--------|---------|
| **Format** | One-on-one sessions, 45 minutes–1 hour each |
| **Style** | Behavioral discussions on past projects + live system design + coding |
| **Emphasis** | Ownership ("You build it, you run it"), pragmatic solutions, integrating AI Agents into mission-critical financial tasks like anomaly detection and invoice matching |
| **Structure** | Each interviewer covers a primary area — one focused on **coding**, one on **system design**, one on **behavioral/cultural fit**. Not every interviewer will combine all aspects. |

### Section Weighting

| Section | Weight | Deep-Dive Guide |
|---------|--------|-----------------|
| Behavioral & Past Experience | ~20% | [guides/01-behavioral-past-experience.md](guides/01-behavioral-past-experience.md) |
| System Design | ~50% | [guides/02-system-design.md](guides/02-system-design.md) |
| Coding & Technical Depth | ~30% | [guides/03-coding-technical-depth.md](guides/03-coding-technical-depth.md) |
| References & Further Reading | — | [references.md](references.md) |
| **48-Hour Cram Plan** | — | **[cram-plan.md](cram-plan.md)** |

> **Short on time?** Follow the [48-Hour Cram Plan](cram-plan.md) for a hour-by-hour schedule aligned to interview weighting.

---

## 1. Behavioral & Past Experience (~20%)

Interviewers will cover your background to assess fit for the company's **builder culture** in a fast-paced fintech environment, probing challenges in **distributed transactions**, **data consistency**, and transitions to modern, **AI-enhanced systems**.

### What to Prepare

#### Key Projects — Use STAR (Situation, Task, Action, Result)

Prepare **2–3 experiences** across these themes:

| Theme | Example Story Arc |
|-------|-------------------|
| **Ownership of core financial services** | Problem: scaling payments or reconciliation for high-volume transactions → Initiative: architecting microservices with idempotency → Solution: event-driven workflows for consistency → Outcomes: improved reliability and reduced operational load |
| **AI integrations in finance** | Using LLMs for automating tasks like ledger anomaly detection or support triage; safeguards for non-deterministic models → Outcomes: efficiency gains in workflows |
| **Legacy migrations** | Moving to orchestration engines like Temporal; handling concurrency and caching → Outcomes: reduced technical debt and higher availability |

#### Important Concepts to Weave In

- **End-to-end ownership** — You wrote it, deployed it, monitored it, and fixed it in production
- **Incident response** — How you handled a production outage or data discrepancy in a financial system
- **Cross-functional collaboration** — Working with Finance, Legal, or Compliance under deadline pressure
- **Technical leadership** — Mentoring, code reviews, driving architectural decisions without formal authority
- **Trade-off articulation** — Why you chose one approach over another (speed vs. correctness, build vs. buy)

#### Common Probes

| Probe Area | What They're Assessing |
|------------|------------------------|
| **Sensitive financial data** | Access controls, auditability, least-privilege, encryption at rest/in transit |
| **AI reliability in production** | Making LLMs viable for transactions via Vector DBs, LangChain, validation layers, human-in-the-loop |
| **Non-technical stakeholders** | Translating business needs (e.g., tax regulations, revenue recognition) into technical specs |
| **Mentoring / code reviews** | Raising the bar on a team, giving constructive feedback, unblocking others |

→ **Full guide:** [guides/01-behavioral-past-experience.md](guides/01-behavioral-past-experience.md)

---

## 2. System Design (~50%)

**Core focus:** Design end-to-end systems for fintech scale (e.g., billions in transactions, global ops), emphasizing **microservices**, **event-driven architectures**, **data consistency (idempotency)**, and **AI integration**.

Start with clarifying questions; iterate based on feedback. Cover ops like **observability** (Prometheus/Grafana) for **99.99% availability**.

### Targeted Topics & Examples

#### Payments & Transaction Processing

Design a real-time payment gateway for **100k+ TPS**:

- Handle distributed transactions with concurrency (DB locking, caching strategies)
- Ensure **idempotency** to prevent duplicates
- Integrate async reconciliation queues
- Discuss failure recovery (e.g., hardware faults via Kubernetes orchestration)
- API design (REST/GraphQL for global vendors)

**Key concepts:** ACID vs. BASE, saga patterns, outbox pattern, payment state machines, PCI-DSS scope, exactly-once semantics

#### Reconciliation & Billing

Design a massive-scale reconciliation engine for vehicle sales and charging revenue:

- Event-driven workflows (e.g., Temporal for orchestration) to match invoices and detect anomalies
- Hybrid sync/async (synchronous confirmations, async audits)
- AI Agents for pattern analysis
- Sharding for data consistency
- CI/CD (Jenkins/Argo) for safe deployments

**Key concepts:** Double-entry bookkeeping, ledger immutability, batch vs. streaming reconciliation, discrepancy resolution workflows, financial close cycles

#### AI-Enhanced Services

Design automated invoicing with AI Agents for **10M+ users/transactions**:

- LLMs (OpenAI API/LangChain) for invoice matching or fraud triage
- Embeddings in Vector DBs for reliable retrieval (RAG)
- Multi-channel user notifications post-validation
- Address non-determinism with thresholds/feedback loops

**Key concepts:** RAG architecture, confidence scoring, human-in-the-loop escalation, model versioning, prompt injection risks in financial contexts

### General Elements to Cover

| Category | Topics |
|----------|--------|
| **Architecture** | Microservices in Go/C# (or modern langs); Docker/Kubernetes for production; workflow tools (Temporal/Cadence/Airflow) for migrations |
| **Scalability & Reliability** | Event buses for async flows; caching (Redis for session data); error handling (replays, eventual consistency); monitoring (distributed tracing, metrics for incident resolution) |
| **Security / Fintech Nuances** | Compliance for financial data (audit trails, role-based access); vendor integrations (direct APIs to optimize costs); AI safeguards (making models deterministic for workflows) |
| **Trade-offs** | Sync vs. async for user feedback; simple caching vs. complex sharding; legacy migration paths without downtime |

### Tips

1. **Clarify requirements first** — volume, latency SLAs, failure rates, consistency requirements
2. **Draw diagrams** — components, data flows, failure domains
3. **Adapt to probes** — e.g., add observability when asked about restarts
4. **Explain decisions pragmatically** — e.g., why Go for performance in high-throughput payments

→ **Full guide:** [guides/02-system-design.md](guides/02-system-design.md)

---

## 3. Coding & Technical Depth (~30%)

Light live coding; focus on **verbal/whiteboard walkthroughs** of code patterns, especially for backend and AI reliability.

### What to Prepare

#### Backend & Systems

| Area | Concepts |
|------|----------|
| **Concurrency** | Go goroutines or C# async/await; worker pools; backpressure |
| **Database interactions** | Locking for transactions (pessimistic vs. optimistic); eventual consistency models (CRDTs, conflict resolution) |
| **Caching** | In-memory vs. distributed (Redis) for financial ops; cache invalidation, TTL strategies, stampede prevention |

#### AI / Agentic Integrations

| Area | Concepts |
|------|----------|
| **LLM pipelines** | LangChain for chaining tasks in anomaly detection; Vector DBs for RAG in invoice matching |
| **Reliability mitigations** | Validation loops, sampling to avoid hallucinations, structured output enforcement, fallback to rules-based logic |

#### Coding Style

- For exercises (e.g., simple idempotent transaction handler), **verbalize your process** — edge cases like failures, optimizations for scale
- Emphasize **testable code** — comprehensive unit/integration tests in fintech stacks (Go/C#, Docker for local dev)

#### Tech Depth

| Area | Concepts |
|------|----------|
| **Production ops** | Kubernetes configs, CI/CD pipelines, blue-green/canary deployments |
| **Orchestration** | Temporal workflows — activities, retries, compensation |
| **Limitations** | Auditing low-traffic AI features; cost controls on LLM API calls |

→ **Full guide:** [guides/03-coding-technical-depth.md](guides/03-coding-technical-depth.md)

---

## General Interview Tips

### Style & Duration

- One-on-one — 45 min–1 hour sessions
- Be **collaborative and pragmatic** — treat it as a working session, not an exam
- Think out loud; invite feedback on your approach

### Company FinTech Fit

Stress these themes throughout:

| Theme | How to Demonstrate |
|-------|-------------------|
| **Ownership** | Services from code to production health — on-call, dashboards, runbooks |
| **Cross-functional collaboration** | Working with PMs/Finance (e.g., for market expansions) |
| **AI passion** | Prototype to production — not just demos, but production-grade AI with guardrails |

---

## Quick Reference — Technologies Mentioned

| Category | Technologies |
|----------|-------------|
| Languages | Go, C# |
| Infrastructure | Docker, Kubernetes |
| Orchestration | Temporal, Cadence, Airflow |
| Observability | Prometheus, Grafana, distributed tracing |
| Caching | Redis |
| AI/ML | OpenAI API, LangChain, Vector DBs |
| CI/CD | Jenkins, Argo |
| APIs | REST, GraphQL |

---

## Further Reading

See the full **[References & Further Reading](references.md)** page for curated links by topic. Essentials to start with:

- [*Designing Data-Intensive Applications*](https://dataintensive.net/) — transactions, consistency, stream processing
- [Stripe — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) — production idempotency patterns
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/) — reliability, on-call, SLOs
- [Microservices.io — Saga Pattern](https://microservices.io/patterns/data/saga.html) — distributed transactions without 2PC
- [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — AI safety in production
