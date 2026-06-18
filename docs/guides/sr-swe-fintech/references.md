# References & Further Reading

[← Sr. SWE, FinTech Study Guide](index.md) · [← All Guides](../index.md)

Curated external resources for deeper study. Links are grouped by topic — start with **Essential** items, then drill into specifics.

## Essential Reading

| Resource | Why it matters |
|----------|----------------|
| [*Designing Data-Intensive Applications*](https://dataintensive.net/) (Martin Kleppmann) | The foundational text on transactions, replication, stream processing, and consistency — directly applicable to payments and ledgers |
| [Google SRE Book](https://sre.google/sre-book/table-of-contents/) | On-call culture, SLOs, error budgets, and running reliable production systems |
| [Stripe Engineering Blog](https://stripe.com/blog/engineering) | Real-world fintech engineering — idempotency, reliability, API design at scale |
| [System Design Primer](https://github.com/donnemartin/system-design-primer) | Broad overview of scalability patterns, trade-offs, and interview frameworks |

---

## Behavioral & Interview Skills

### STAR & Storytelling

| Resource | Topic |
|----------|-------|
| [Google Interview Tips — Using STAR](https://careers.google.com/apply/interview-tips/) | Official STAR framework guidance |
| [Amazon Leadership Principles](https://www.amazon.jobs/content/en/our-workplace/leadership-principles) | Useful lens for ownership, bias for action, and dive-deep stories |

### Ownership & Production Mindset

| Resource | Topic |
|----------|-------|
| [Charity Majors — Ops is Everyone's Job](https://charity.wtf/2023/11/07/ops-is-everyones-job-now/) | "You build it, you run it" in modern engineering orgs |
| [Will Larson — Staff Engineer](https://staffeng.com/book) | Operating at senior levels: influence, technical judgment, ownership |
| [Google Engineering Practices — Code Review](https://google.github.io/eng-practices/review/) | How to give and receive effective code review feedback |

### Incident Response & Postmortems

| Resource | Topic |
|----------|-------|
| [Google SRE — Managing Incidents](https://sre.google/sre-book/managing-incidents/) | Structured incident response for production outages |
| [Etsy Debriefing Facilitation Guide](https://github.com/Lucius-Business-Facilitation/Debriefing-Facilitation-Guide) | Blameless postmortem culture |

---

## Payments & Transactions

### Idempotency & API Design

| Resource | Topic |
|----------|-------|
| [Stripe — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) | Canonical implementation of idempotency keys in a payment API |
| [Stripe — Designing APIs for Humans](https://stripe.com/blog/api-versioning) | API versioning and backward compatibility |
| [PayPal Idempotency Best Practices](https://developer.paypal.com/api/rest/reference/idempotency/) | Another production idempotency reference implementation |

### Distributed Transactions & Sagas

| Resource | Topic |
|----------|-------|
| [Microservices.io — Saga Pattern](https://microservices.io/patterns/data/saga.html) | Choreography vs. orchestration for distributed transactions |
| [Microservices.io — Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) | Atomic DB write + event publish |
| [Temporal — Saga Documentation](https://docs.temporal.io/saga) | Durable saga implementation with compensation |
| [AWS — Saga Pattern for Distributed Transactions](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-data-persistence/saga-pattern.html) | AWS prescriptive guidance on saga trade-offs |

### Payment Systems at Scale

| Resource | Topic |
|----------|-------|
| [Adyen — Payment Lifecycle](https://docs.adyen.com/online-payments/payment-lifecycle/) | Payment state machine (authorisation, capture, settlement) |
| [Martin Fowler — Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) | Immutable event log as source of truth for financial records |
| [Martin Fowler — CQRS](https://martinfowler.com/bliki/CQRS.html) | Separating write and read models for ledger systems |

---

## Reconciliation & Billing

| Resource | Topic |
|----------|-------|
| [Temporal — Durable Execution](https://docs.temporal.io/temporal) | Why workflow engines replace brittle cron for reconciliation |
| [Debezium — Change Data Capture](https://debezium.io/documentation/reference/stable/index.html) | Streaming DB changes into reconciliation pipelines |
| [Kafka — Exactly-Once Semantics](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/) | Event delivery guarantees for financial event streams |
| [Expand-Contract Pattern](https://www.prisma.io/dataguide/types/relational/migration-strategies) | Safe schema migrations without downtime |

---

## Database Design

| Resource | Topic |
|----------|-------|
| [*Designing Data-Intensive Applications*](https://dataintensive.net/) (Martin Kleppmann) | Ch. 5–9: replication, partitioning, transactions, consistency |
| [CAP Twelve Years Later](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) (Eric Brewer) | Modern interpretation of CAP for practitioners |
| [PostgreSQL — Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html) | Store money as `NUMERIC` / integer cents, not float |
| [PostgreSQL — Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html) | Time-based partitions for ledger tables |
| [PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) | `FOR UPDATE`, advisory locks, deadlock behavior |
| [PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) | Isolation levels and anomaly prevention |
| [AWS — Data Persistence Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-data-persistence/welcome.html) | When to choose SQL vs NoSQL |
| [Use The Index, Luke](https://use-the-index-luke.com/) | SQL indexing and query performance |
| [Martin Kleppmann — Distributed Locking](https://martin.kleppmann.com/2016/08/08/how-to-do-distributed-locking.html) | Why distributed locks are harder than they look |
| [System Design guide → Database Design](02-system-design.md#database-design-cap-theorem) | CAP, schemas, technology trade-offs (this repo) |

---

## System Design & Architecture

### Scalability Patterns

| Resource | Topic |
|----------|-------|
| [System Design Interview (Vol. 1 & 2)](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF) | Structured approach to interview system design |
| [High Scalability Blog](http://highscalability.com/) | Real architecture postmortems from large-scale systems |
| [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/) | Reference architectures for distributed systems |

### Caching & Performance

| Resource | Topic |
|----------|-------|
| [Redis — Caching Strategies](https://redis.io/docs/latest/develop/use-cases/caching/) | Cache-aside, TTL, and invalidation patterns |
| [AWS — Caching Best Practices](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Strategies.html) | Cache stampede prevention and consistency |
| [Facebook — Scaling Memcache](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf) | Classic paper on distributed caching at scale |

### Messaging & Event-Driven Architecture

| Resource | Topic |
|----------|-------|
| [Kafka Documentation](https://kafka.apache.org/documentation/) | Partitioning, consumer groups, delivery semantics |
| [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) | Message routing, dead letter queues, pub/sub |
| [Martin Fowler — Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html) | When and how to use event-driven designs |

---

## Observability & Reliability

| Resource | Topic |
|----------|-------|
| [Google SRE — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) | Four golden signals, alerting philosophy |
| [Charity Majors — Observability Engineering](https://www.oreilly.com/library/view/observability-engineering/9781492076438/) | Metrics, traces, and logs for debugging production |
| [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/) | Metrics collection and PromQL |
| [OpenTelemetry Documentation](https://opentelemetry.io/docs/) | Vendor-neutral distributed tracing |
| [Grafana — SLO Guide](https://grafana.com/docs/grafana-cloud/alerting-and-irm/slo/) | Defining and monitoring SLOs / error budgets |

---

## Security & Compliance (FinTech)

| Resource | Topic |
|----------|-------|
| [PCI DSS Quick Reference](https://www.pcisecuritystandards.org/document_library/) | Card data handling requirements and scope reduction |
| [OWASP Top 10](https://owasp.org/www-project-top-ten/) | Web application security fundamentals |
| [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) | Prompt injection, data leakage risks for AI in finance |
| [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework) | Security controls and risk management |
| [SOC 2 Overview](https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2) | Audit and compliance expectations for SaaS/fintech |

---

## AI / LLM in Production

| Resource | Topic |
|----------|-------|
| [RAG Paper (Lewis et al.)](https://arxiv.org/abs/2005.11401) | Original Retrieval-Augmented Generation research |
| [LangChain Documentation](https://python.langchain.com/docs/introduction/) | Chaining LLM tasks, agents, and tool use |
| [OpenAI — Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs) | Enforcing JSON schema on LLM responses |
| [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) | Agent design patterns and guardrails |
| [Pinecone — RAG Guide](https://www.pinecone.io/learn/retrieval-augmented-generation/) | Practical RAG architecture with vector search |
| [pgvector Documentation](https://github.com/pgvector/pgvector) | Vector similarity search in PostgreSQL |

---

## Coding & Concurrency

### Go

| Resource | Topic |
|----------|-------|
| [Effective Go](https://go.dev/doc/effective_go) | Idiomatic Go patterns |
| [Go Blog — Pipelines and Cancellation](https://go.dev/blog/pipelines) | Concurrency patterns with channels and context |
| [Go Blog — Concurrency is not Parallelism](https://go.dev/blog/waza-talk) | Rob Pike's foundational talk (video + slides) |
| [Go sync package](https://pkg.go.dev/sync) | Mutex, WaitGroup, Once primitives |

### C#

| Resource | Topic |
|----------|-------|
| [Async/Await Best Practices](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming) | Avoiding deadlocks and thread-pool starvation |
| [.NET Task Parallel Library](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl) | Parallel and async patterns in C# |

### Resilience Patterns

| Resource | Topic |
|----------|-------|
| [Martin Fowler — Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html) | Failing fast when downstream is unhealthy |
| [AWS — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) | Retry strategies for transient failures |
| [Release It! (Michael Nygard)](https://pragprog.com/titles/mnee2/release-it-second-edition/) | Stability patterns for production systems |

---

## Infrastructure & DevOps

| Resource | Topic |
|----------|-------|
| [Kubernetes Documentation](https://kubernetes.io/docs/home/) | Deployments, probes, HPA, ConfigMaps |
| [Temporal Documentation](https://docs.temporal.io/) | Durable workflows, activities, retries |
| [Argo CD Documentation](https://argo-cd.readthedocs.io/) | GitOps continuous delivery |
| [Argo Rollouts](https://argo-rollouts.readthedocs.io/) | Canary and blue-green deployments |
| [The Twelve-Factor App](https://12factor.net/) | Cloud-native application design principles |
| [Docker Documentation](https://docs.docker.com/get-started/) | Containerization fundamentals |

---

## Testing

| Resource | Topic |
|----------|-------|
| [Google Testing Blog](https://testing.googleblog.com/) | Unit, integration, and production testing practices |
| [Martin Fowler — Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html) | Balancing test types |
| [Property-Based Testing with fastcheck (Go)](https://pkg.go.dev/github.com/leanovate/gopter) | Generative testing for idempotency invariants |
| [Testcontainers](https://testcontainers.com/) | Integration tests with real DB/Redis/Kafka in Docker |

---

## Related Guides

| Guide | Link |
|-------|------|
| Study Guide (overview) | [index.md](index.md) |
| Behavioral & Past Experience | [01-behavioral-past-experience.md](01-behavioral-past-experience.md) |
| System Design | [02-system-design.md](02-system-design.md) |
| Coding & Technical Depth | [03-coding-technical-depth.md](03-coding-technical-depth.md) |
| 48-Hour Cram Plan | [cram-plan.md](cram-plan.md) |
| Cheat Sheet | [cheat-sheet.md](cheat-sheet.md) |
| All guides | [Guides overview](../index.md) |
