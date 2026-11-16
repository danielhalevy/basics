# Learning plan & progress — System Design (Akamai ExAR)

Interview date: 2026-11-17 (in 2 days)
Pace: 4-5 sessions/day, 20-30 min each. Phase: 1 (LEARN)

## 2-day plan (priority order)
- Day 1 (Nov 15): Unit 3 request journey -> Unit 4 databases -> Unit 5 queues -> Unit 7 BigQuery -> small check
- Day 2 (Nov 16): warm-up recall -> Unit 11 LLMs/AI anomaly -> Unit 8 reliability + 9 multi-tenant -> Unit 12 model answer 3 -> one full mock
- Interview morning (Nov 17): 10-min recall walk, no new material

## Units (in order) — state / last date / next review
1. The interview game: what's scored, the 8 steps, phrasing — Learned (partly) / 2026-11-09 / due
2. Numbers & estimation — Learned (partly) / 2026-11-09 / due
3. The request journey: load balancer, stateless services, cache (Redis), CDN — Learned / 2026-11-15 / review 2026-11-16
4. Databases: relational vs document vs key-value vs warehouse; indexes — Learned / 2026-11-15 / review 2026-11-16 (replication+sharding moved to Unit 8)
5. Queues & logs: Pub/Sub vs Kafka vs Cloud Tasks, partitions, ordering, acks, at-least-once, idempotency — Learned / 2026-11-15 / review 2026-11-16
6. Processing: stream vs batch, windows, late data, Dataflow, Airflow — Learned (partly)
7. BigQuery in depth: partitioning, clustering, cost per bytes, slots, loading, serving vs analytics, views — Learned / 2026-11-15 / review 2026-11-16
8. Reliability: replication, failover, retries+backoff+jitter, circuit breaker, dead-letter queue, backpressure, queue failure — Learned (partly)
9. Multi-tenancy & multi-region: isolation, noisy neighbors, quotas, residency, regional cells — Learned (partly)
10. APIs: REST vs gRPC, pagination, idempotency keys, auth from token, webhooks — Learned (partly)
11. LLMs in production: cascade, cost, latency, quotas, evaluation, prompt injection, AI anomaly detection — Learned (partly)
12. Putting it together: the 3 model answers (prep/03-model-answers.md) — Not started

## Known weak spots
- Check-then-act without atomicity (race conditions)
- Reaches for BigQuery even when data is small
- Doesn't propose numbers; asks instead
- Drops requirements the interviewer gave
- Mixes Postgres (bank teller) and BigQuery (library)
- Security domain vocabulary

## Session log
- 2026-11-15 S1: Unit 3 done. Retell 8/10. Gaps: mixed LB role with statelessness; missed "next request hits a different instance"; cache fix = invalidate on write (he proposed version check). Strength: CDN reasoning, self-corrected that Redis can hold per-user data.
- 2026-11-15 S2: Unit 4 done. Retell 8.5/10. Gaps: thought document DBs suit 1M/s event streams (wrong); Bigtable = key/range lookups not big scans; missed race condition at first, then got it. Postgres vs BigQuery mix-up fixed.
- 2026-11-15 S3: Unit 5 done. Retell 8.5/10. Strong: stage-level checkpoints + versions, DLQ. Gaps: believed Pub/Sub/Cloud Tasks never duplicate (corrected with docs); "nothing can be done if order needed" -> key by agent; concurrent-run race -> atomic lease; "only one consumer group per partition" -> one consumer WITHIN a group.
- 2026-11-15 S4: Unit 7 done. Retell 8.5/10 (unit 5 retell regraded to 9). Math perfect ($312 -> <1 cent). Gaps: thought clustering slows inserts; Airflow is not a streaming writer; weekly partitions are for tiny/too-many partitions, not skewed days; vague on what clustering is (sorted blocks, skip).
- 2026-11-15 S5: small check 7.5/10. Q1 Bigtable vs Redis 9/10. Q2 concurrent runs 6/10: check must be ATOMIC (conditional write/lease), can't ask the LLM. Q3 dashboard 8/10: missed that alerts are low-volume and already in Postgres -> rollup in Postgres, BigQuery not needed.
- 2026-11-16 S6 warm-up 7.5/10: Q1 statelessness question NOT recalled (said readiness/LB; bonus credit for readiness checks). Q2 at-least-once + ID + atomic claim 9/10. Q3 hourly partition + cluster by customer + Postgres serve 8.5/10; missed incremental per-hour compute & selecting only needed columns.
  Units 4,5,7 -> Recalled after a day. Unit 3 statelessness: re-check later.
- 2026-11-16 S7: Unit 11 chunks 1-4 done. Strong (9-9.5): token math (missed per-minute unit once), Armadillex story polished (on-demand vs batch paths, prefix caching insight), context for LLM judgment. Eval answer 8.5: right instinct (fewer alarms may = missed attacks) but no method (golden set, recall, shadow mode). Awaiting retell.
- PLAN (Nov 16): warm-up recall units 3/4/5/7 -> Unit 11 LLM + AI anomaly -> Units 8+9 reliability & multi-tenant -> model answer 3 -> full mock
