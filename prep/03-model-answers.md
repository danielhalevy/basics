# Three Model Answers

How to use these: read one answer, then close it and explain it out loud in ~10 minutes, following the 8 steps. Then check what you missed.

---

# MODEL ANSWER 1 — Real-time security telemetry → detections (the ExAR core)

**Prompt:** "Design a system that ingests network telemetry from agents on customers' servers and alerts their security teams when something suspicious happens."

## 1. Clarifying questions (ask + propose a number)
- Scale? "I'll assume ~5,000 customers, ~2M agents, ~1M events/sec at peak. OK?"
- What does the agent see? "Metadata only: connections, processes, logins. No payloads?"
- How fast? "High severity within ~1 minute, the rest within ~15 minutes?"
- Can we lose events? "I'll assume no — a missed event could be a missed attack."
- Data residency? "Any EU customers whose data must stay in the EU?"
- History? "Do analysts need to search past events? For how long — 90 days?"
- Action? "Alert only, no automatic blocking — correct?"

## 2. Functional requirements
- Ingest telemetry and detect suspicious activity: built-in rules, customer rules, and AI anomaly detection.
- Deliver alerts to customers (dashboard, webhook, email/Slack).
- Analysts can investigate an alert and search past events.
- Out of scope: automatic blocking, agent internals, auth/billing, UI details.

## 3. Non-functional requirements
- Scale: ~1M events/sec peak, 2M agents, 5k tenants of very different sizes.
- Freshness: high severity < 1 min end to end; others < 15 min.
- Durability: no event loss — at-least-once delivery with dedup.
- Availability: ingestion 99.9%+; if we are down we are delayed, not blind.
- Isolation & residency: strict tenant isolation; EU data stays in the EU.
- Retention: 90 days searchable; older in cheap archive.
- Cost: bounded per tenant.

## 4. Estimates
- 1M events/sec × ~1 KB ≈ 1 GB/sec ≈ ~80 TB/day raw.
- Conclusion: no per-event database writes; streaming pipeline; analytics in a partitioned warehouse.

## 5. Entities
- Tenant, User, Agent, Asset, Event, Rule, Detection, Alert. Every one carries tenant_id.

## 6. API
- Ingest: gRPC from agents, batched (1,000 events or 1 second), each agent authenticated with its own certificate. Batch has batch_id; each event has event_id, timestamp, type. Server acks only after the batch is safely in the queue.
- Query (REST): list/get/update alerts, search events (time range required), create/update/delete rules. Tenant comes from the token. Paginated.
- Outbound: signed webhooks with retries.

## 7. High-level design (narrate one event)
- The agent batches events and sends them over gRPC to a load balancer.
- A stateless ingest service checks the agent, validates, applies the tenant's rate limit, publishes to Pub/Sub, and only then acks the agent.
- Pub/Sub absorbs spikes and fans out to three consumers:
  - The stream processor (Dataflow) runs detection: short time windows in memory (port scans, brute force), fast lookups in Bigtable ("has this server ever talked to this IP?"), customer rules cached from Postgres.
  - An archiver writes raw Parquet files to GCS (cheap, for forensics and re-running rules).
  - A writer loads events into BigQuery (partitioned by date, clustered by tenant) for analyst search.
- Detections go to a second topic. A correlator groups them by tenant + asset + attack type within 15 minutes, so 50 detections become 1 alert, stored in Postgres.
- A notifier sends alerts out: high severity immediately, with retries and signed webhooks.
- The dashboard reads alerts from Postgres (fast) and runs event searches on BigQuery (slower, async).
- The whole stack is deployed as independent regional cells (EU, US). Tenants are pinned to their home region.
- AI anomaly detection: an hourly job builds compact behavior profiles per asset, a statistical score picks the top ~1%, and the LLM judges only those. Its findings become detections like any rule.

## 8. Deep dives
- No loss, no duplicates: two acks (agent→ingest after publish; queue→processor after processing). Retries create duplicates, so every write is idempotent by event_id.
- Late data: windows use event time with ~5 minutes of allowed lateness; very late events go to a side path and are re-checked in batch.
- Noisy neighbors: per-tenant quotas; hot keys spread across workers; the biggest tenants get dedicated capacity.
- Queue failure: the queue is replicated. If it is down, ingest stops acking and agents buffer on disk and retry. Alert on the age of the oldest unread message.
- Cost: aggregate at the agent; BigQuery partitioning and clustering; required time ranges; LLM only on the shortlist.

## 9. Wrap-up
- Trade-off: at-least-once + dedup instead of exactly-once — simpler and robust.
- Monitor: end-to-end latency, queue backlog, error rate, cost per tenant, AI precision/recall.
- Next: automated response, cross-tenant threat intelligence.

---

# MODEL ANSWER 2 — LLM document extraction platform (your strength)

**Prompt:** "Enterprise customers upload documents (5–500 pages). Use LLMs to extract structured information and flag compliance issues. Customers review and correct the results."

## 1. Clarifying questions
- Volume? (Given: ~200k docs/day, average 40 pages, peaks 5×, bulk uploads of 50k.)
- Formats? "Are some scanned PDFs that need OCR?"
- Accuracy? "What accuracy is acceptable, and how do we measure it today?"
- Latency? (Given: single doc within 10 minutes; bulk can take hours with progress.)
- Cost? (Given: target ≤ $0.25 per document.)
- LLM limits? (Given: ~200k-token context; shared quota of 5M tokens/min.)
- Corrections? (Given: versioned with an audit trail.)

## 2. Functional requirements
- Customers upload documents, one at a time or in bulk.
- The system extracts structured fields and flags compliance issues, each with a page citation.
- Customers view, correct, and approve results; every correction is versioned and audited.
- Out of scope: AI rewriting documents, chat, comparing documents to each other.

## 3. Non-functional requirements
- Scale: 200k docs/day, ~8M pages/day, peaks 5×, bulk 50k at once.
- Latency: single doc ≤ 10 min (small docs faster); bulk in hours with visible progress.
- Cost: ≤ $0.25 per document on average.
- Accuracy: e.g. ≥ 95% field accuracy on a labeled test set, measured continuously.
- Durability: never lose a document or a correction.
- Security: encrypted, strict tenant isolation, 7-year retention, no training on customer data.

## 4. Estimates — this is where you find the crux
- 8M pages/day × ~500 tokens/page ≈ 4B tokens/day ≈ 2.8M tokens/min on average.
- At a 5× peak that is ~14M tokens/min, but the quota is 5M tokens/min.
- Conclusion: the LLM quota is the bottleneck. We need a queue, a token scheduler with priorities, and we must send fewer tokens per document.

## 5. Entities
- Customer, User, Document (status, hash), Page/Chunk, Job (stage, status, progress), Extraction (fields, confidence, page citations, model version), Finding (compliance issue, severity), Correction (versioned: who, when, old/new value).

## 6. API
- POST /documents → returns a signed upload URL; the client uploads straight to storage. Large files never pass through our API servers.
- POST /batches → for bulk uploads; GET /batches/{id} shows progress.
- GET /documents/{id} → status and progress.
- GET /documents/{id}/extraction → fields, findings, citations.
- PATCH /extractions/{id} → a correction; creates a new version (with a version check so two reviewers don't overwrite each other).
- Webhook when a document or batch finishes.

## 7. High-level design (narrate one document)
- The client uploads directly to GCS using the signed URL (encrypted per customer). The upload triggers a message on Pub/Sub.
- A preprocessing worker detects the format, runs OCR on scanned pages, extracts text per page, and saves it. It also builds a "document map": table of contents, section headings, and the definitions section.
- A job record goes into Postgres. A token scheduler hands out work within the 5M tokens/min budget:
  - Single uploads go to a high-priority lane; bulk uploads go to a low-priority lane.
  - Fair share per customer, so one 50k bulk upload doesn't starve everyone else.
  - The global token budget is a token bucket in Redis.
- Multi-stage extraction, cheapest first:
  - Stage 1: classify the document type and find the relevant sections (keywords or embeddings). Only relevant pages go to the LLM — often 20% of the document, not 100%.
  - Stage 2 (map): a cheap model extracts candidate fields from each chunk of ~20–30 pages. Each call includes the document map, so page 400 can be understood together with the definitions on page 5.
  - Stage 3 (reduce): merge the candidates and resolve conflicts. When a clause references another section ("as defined in 12.3"), fetch that section and re-check.
  - Stage 4: only low-confidence or conflicting fields are escalated to a strong model.
- Compliance checks: deterministic rules where possible ("renewal notice under 30 days"); the LLM only for nuanced clauses.
- Results go to Postgres; status is updated; the customer is notified; the dashboard reads Postgres.
- Corrections are stored as new versions (append-only) and are also added to the test set and used as examples for that customer's document types.

## 8. Deep dives
- Token quota and peaks (the crux): priority lanes, fair share, the token bucket; bulk jobs fill quiet hours; use the provider's cheaper batch mode for bulk; on 429 errors back off with jitter.
- Cost to ≤ $0.25: send only relevant pages; cheap model first; escalate only uncertain fields; cache results by document hash (vendors send the same templates); reuse cached shared instructions (prompt caching); track cost per document per customer. Story: "At Armadillex I built a multi-stage pipeline like this and cut cost 75% and latency from 15 to 2 minutes."
- Long documents: map-reduce with the document map in every call, overlapping chunks, and following cross-references.
- Accuracy and evaluation: a labeled test set per document type; field-level precision and recall; every value must cite a page, and we verify the quoted text actually exists there (catches hallucinations); low confidence goes to human review; new models and prompts run in shadow mode first.
- Reliability: every stage saves its output and is idempotent by (document, stage, version), so a crash resumes from the last finished stage instead of restarting a 500-page document. Jobs are leased with a timeout; a stuck job is detected and retried automatically. Story: "I built a self-healing stuck-job fixer across 60+ microservices."
- Security: encryption, tenant isolation, no training on customer data. Prompt injection: a document can contain "ignore previous instructions", so the document is always treated as data and output must be structured.

## 9. Wrap-up
- Trade-off: multi-stage cascade instead of one big call — more moving parts, far cheaper and fits the quota.
- Monitor: queue wait time per lane, tokens/min vs quota, cost per document, accuracy on the test set, correction rate.
- Next: comparing documents against each other, customer-specific fine-tuning.

---

# MODEL ANSWER 3 — AI anomaly detection over telemetry in BigQuery (what the interviewer said they do)

**Prompt:** "Customers' network connection logs already land in BigQuery — tens of billions of rows a day. Design a system that uses AI to find anomalous behavior per asset and shows it to analysts, while keeping cost under control."

## 1. Clarifying questions
- Scale? "I'll assume ~5k tenants, ~10M assets, ~50B rows/day. OK?"
- Freshness? "Is hourly enough, given that known-bad patterns are already caught by real-time rules?"
- What counts as anomalous? "Unusual for this asset compared with its own history and similar assets?"
- What matters more: catching everything or few false alarms? (Usually: track both, recall is critical.)
- Cost budget? Residency? Do analysts give feedback on findings?

## 2. Functional requirements
- Build a behavior baseline for each asset.
- Every hour, detect anomalous behavior and explain why it is anomalous.
- Analysts see the findings and mark them as real or false; that feedback improves the system.
- Out of scope: automated response, building the raw ingestion (already exists).

## 3. Non-functional requirements
- Freshness: results within ~1 hour of the window ending.
- Scale: 10M assets, ~50B rows/day.
- Cost: bounded BigQuery bytes scanned and LLM tokens per tenant.
- Quality: measured precision and recall on a labeled set; never silently suppress a rule-based alert.
- Isolation & residency; reproducibility (every finding records model and prompt version).

## 4. Estimates
- Raw data is huge, so we never send raw rows to the LLM.
- 10M assets: if 1% are candidates, that's 100k LLM calls per hour — too many. The scoring threshold is the cost dial: aim for ~0.1% (≈10k per hour) and group similar candidates.

## 5. Entities
- Asset, BehaviorProfile (asset, hour, features), Baseline (asset, rolling statistics), Candidate (score), Finding (verdict, confidence, category, reason, model version), Feedback (analyst label).

## 6. API
- GET /findings?asset=&from=&to= (paginated, tenant from token).
- POST /findings/{id}/feedback {label: real | false_positive, note}.
- PATCH /settings/sensitivity (per-tenant tuning).

## 7. High-level design (an hourly Airflow pipeline, per region)
- Airflow starts the run every hour. Airflow only coordinates; BigQuery and workers do the heavy work.
- Feature step: one BigQuery SQL job reads only the last hour's partition and builds a compact profile per asset: number of connections, distinct destinations, new destinations compared to the last 30 days, unusual ports, bytes in/out, off-hours activity. Results go to a profiles table (partitioned by hour, clustered by tenant).
- Baseline step: rolling statistics per asset are updated incrementally — not recomputed over 30 days every hour.
- Scoring step: cheap statistics compare each profile with the asset's own baseline and with similar assets (peer group). This picks the top ~0.1% as candidates.
- LLM step: for candidates only, build a small JSON (the unusual features + asset context such as "database server") and send it to the LLM through rate-limited workers. The LLM returns a structured verdict: anomalous yes/no, confidence, category, short reason.
- Findings are written to Postgres (for the dashboard) and BigQuery (for history and evaluation). High-confidence findings are published as detections into the alert pipeline.
- Analyst feedback goes into a labels table, which feeds evaluation and threshold tuning.

## 8. Deep dives
- Cost: partition pruning (scan 1 hour, not 30 days); incremental baselines; the scoring threshold as the cost dial; cheap model first; group identical patterns (a software update hitting 10k servers is judged once, not 10k times); per-tenant budgets.
- Evaluation: a golden set of labeled past incidents plus simulated attacks; measure precision and recall; new models/prompts run in shadow mode; track metrics per model version; analyst feedback becomes new labels.
- Baselines: new assets have no history, so use their peer group; account for time of day and weekday/weekend; keep past anomalies out of the baseline; an attacker who slowly ramps up activity can teach the baseline — compare against longer windows and peers.
- Reliability: each hourly run writes its output partition with overwrite, so a rerun is safe (idempotent) and backfills are easy; Airflow retries and alerts if a run is late.
- Freshness: hourly is fine for behavior; known-bad patterns stay in real-time streaming rules; can move to 5–15 minute micro-batches if needed.
- Safety: hostnames and process names can be written by attackers (prompt injection) — treat them as data, force structured output, the LLM cannot take actions.

## 9. Wrap-up
- Trade-off: hourly batch instead of streaming — far cheaper and simpler; real-time rules cover the urgent cases.
- Monitor: run duration, bytes scanned, tokens and cost per tenant, precision/recall, analyst feedback rate.
- Next: per-tenant learned models, smarter peer grouping.

---

# Phrases that work in any answer
- "I'll start simple and scale it."
- "I'll assume X — does that sound right?"
- "I'll park that and come back to it in the deep dive."
- "X over Y, because Z; I'd revisit if W."
- "We degrade into delay, not loss."
- "At-least-once delivery plus idempotent writes."
- "Statistics narrow it down; the LLM judges the shortlist."
- "Here's where I think this breaks first…"
