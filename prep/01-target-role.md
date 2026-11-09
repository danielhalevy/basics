# Target: Senior Software Engineer, ExAR (Exposure Analysis & Response), Akamai — Req 4242

## Job description (key points)
- Data platform behind ExAR: security telemetry -> real-time detections & risk insights (threat hunting, asset segmentation)
- Stack: Python, advanced SQL, GCP, BigQuery, Airflow, LLMs for triage/investigation
- Own backend systems + pipelines at massive telemetry scale
- LLM workflows: build, deploy, *continuously evaluate*; balance accuracy vs latency vs cost
- SQL data model tuning, orchestration efficiency, cost control; multi-region, multi-tenant
- Code/design review, mentoring, observability, automated testing
- 5+ yrs backend, distributed systems, GCP

## Candidate (Daniel Halevi) — strengths that map directly
- GCP (Cloud Run, Cloud Tasks, Vertex AI), BigQuery, Airflow ETL (Grid-Sight)
- Multi-stage LLM batch pipeline, Gemini + Claude: 75% cost cut, 15m -> 2m latency (Armadillex)
- Self-healing stuck-job troubleshooter across 60+ microservices (reliability story)
- Kafka, K8s, Splunk/Grafana (Bank Hapoalim); mentoring; API standards across 10+ teams

## Gaps to drill (likely probe areas)
1. Streaming ingestion at scale: Pub/Sub, Dataflow/Beam, Kafka; backpressure; exactly-once vs at-least-once + dedup; late/out-of-order events; windowing
2. BigQuery internals: partitioning, clustering, slots/reservations, bytes-scanned cost, Storage Write API vs batch load, materialized views, BI Engine
3. Multi-tenancy: isolation models (shared tables + tenant_id + row-level security vs dataset-per-tenant), noisy neighbor, per-tenant quotas, cost attribution
4. Multi-region: data residency (EU), replication, failover
5. Detection engine: rules (streaming) vs analytics (batch), hot path vs cold path (Lambda/Kappa), alert dedup/suppression
6. LLM in production: eval harness, golden sets, precision/recall on triage verdicts, LLM-as-judge, model routing (cheap->expensive), caching, prompt injection from attacker-controlled telemetry, human-in-the-loop
7. Observability for data pipelines: freshness/completeness SLOs, data quality checks, lineage

## Likely interview prompts
- Design a pipeline ingesting network/host telemetry from thousands of customers' agents into near-real-time detections + a BigQuery analytics layer
- Design an LLM-powered alert triage / investigation assistant (accuracy vs latency vs cost, evaluation)
- Design an asset inventory + exposure/risk scoring service (multi-tenant)
- Design a threat-hunting query platform over petabytes of logs (cost control)

## Resume risk
Bullets are claim-heavy ("Dramatically..."). Be ready to explain the mechanism and numbers behind each:
the 75%/15m->2m pipeline, the self-healing troubleshooter, the 60% BigQuery speedup.

## Intel from Daniel's first-round conversation with the interviewer (overrides the JD text)
- AI is used to FIND ANOMALIES, not to summarize/triage alerts. "Summaries are nonsense, nobody needs them."
- Design implication: LLM/ML sits in the detection path as an anomaly finder over compact, aggregated
  behavior features (per asset/tenant), not raw rows; cost, latency, and evaluation (precision/recall) are key.
- Note: the pasted JD looks AI-generated; trust the interviewer's framing.
