# System Design Interview Prep — Research Notes

## Claude skills found online (GitHub, public)
| Skill | What's worth borrowing |
|---|---|
| booklib-ai/booklib `system-design-interview` | Maps Alex Xu's *System Design Interview* (Vol 1) chapters to topics; 4-step framework |
| abevz/devops-skills `interview-system-design` | 8-step answer checklist; feedback mode scores each step present/weak/missing + top 3 fixes |
| swaaap-nil/system-design-skill | "Take positions" rubric (no "we could use X or Y"); "name the crux"; every box must trace to a requirement |
| consistentJake/Claude-Code-As-Teacher `interviewer` | Interviewer persona + feedback protocol |
| techinterview.org "live-system-design-interviewer" prompt | One question per message; demand numbers; "why not the obvious alternative?"; never design for the candidate; time calls at 20/35/42 min; debrief |
| Hello Interview delivery framework | Requirements → Core entities → API → (Data flow) → High-level design → Deep dives, with timings and leveling (mid/senior/staff) |

## Interview framework I'll teach (Hello Interview + Alex Xu merged)
1. Functional requirements — top 3, prioritized (~5 min)
2. Non-functional requirements — 3-5, quantified (latency, availability, consistency, scale)
3. Back-of-envelope — only where numbers change the design
4. Core entities (~2 min)
5. API (~5 min)
6. High-level design — satisfy functional reqs, simple first (~10-15 min)
7. Deep dives — satisfy non-functional reqs, bottlenecks, failure modes (~10 min)
8. Wrap-up — trade-offs, what you'd monitor, evolution

## Leveling
- Mid: deliver a *working* end-to-end design.
- Senior: proactively find bottlenecks and lead deep dives; justify every choice against an alternative.
- Staff: name the crux, drive scope, talk org/operational trade-offs, cost.

## Books / resources most recommended
- *Designing Data-Intensive Applications* — Kleppmann (fundamentals: replication, partitioning, transactions, consistency)
- *System Design Interview* Vol 1 & 2 — Alex Xu (worked interview problems)
- Hello Interview (free written breakdowns), ByteByteGo

## Fundamentals syllabus (to cover before the mock)
1. Scaling basics: vertical vs horizontal, stateless services, load balancers
2. Storage: SQL vs NoSQL, indexes, replication (leader/follower, multi-leader, leaderless), sharding/partitioning, consistent hashing
3. Caching: cache-aside / write-through / write-back, eviction, invalidation, thundering herd, CDN
4. Async: queues vs logs (SQS vs Kafka), pub/sub, idempotency, at-least-once + dedup, outbox pattern
5. Consistency: CAP / PACELC, strong vs eventual, quorums, transactions, distributed locks
6. Communication: REST vs gRPC, WebSockets / SSE / long polling
7. Reliability: replication, retries with backoff + jitter, circuit breakers, rate limiting
8. Numbers: latency cheat sheet, QPS/storage estimation
9. Observability: metrics, logs, tracing, SLOs
10. Classic problems: URL shortener, rate limiter, news feed, chat, notification system, file storage, ticket booking, etc.

Sources: hellointerview.com, techinterview.org, github.com (repos above), educative.io, designgurus.io
