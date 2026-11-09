# Summary so far + Memory Palace

## Part 1 — Everything so far, in short points

### The interview flow (8 steps, ~45 min)
1. Functional reqs (top 3, what it DOES) + out of scope — 3-5 min
2. Non-functional reqs (HOW WELL, with numbers) — 2-3 min
3. Estimates (only numbers that change the design) — 2-3 min
4. Core entities (business nouns, 2-3 fields) — 1-2 min
5. API (ingest / query / outbound) — ~5 min
6. High-level design (one FR at a time, walk the data, "why" per box, park problems) — 10-15 min
7. Deep dives (fix the parked list against the NFRs, proactively) — 10-15 min
8. Wrap-up (trade-offs, monitoring, v2) — 2 min

### NFR checklist
Scale, Freshness (latency), Durability, Availability, Isolation + Residency, Retention, Cost. Always with a number.

### Numbers
- 1 day ~ 100k sec -> 1M/day ~ 10/sec; 1B/day ~ 10k/sec; peak 2-5x average
- Our system: 5k tenants, 2M agents, ~1M events/sec peak, ~1KB/event -> ~1 GB/s, ~40-80 TB/day raw
- Latency: RAM/Redis <1ms, same-region ~1ms, cross-continent ~100-150ms, Postgres ~1-10ms, BigQuery seconds, LLM 1-30s
- Capacity: Postgres ~5-10k writes/s, Redis ~100k ops/s, Kafka partition ~10 MB/s
- BigQuery ~ $6/TB scanned -> partition by date + cluster by tenant + select only needed columns

### Entities (one sentence)
"A Tenant's Users run Agents on Assets that emit Events; Rules turn Events into Detections, which are grouped into Alerts."
Every entity carries tenant_id.

### API
- Ingest: gRPC, batched (1,000 events or 1s), mTLS per agent -> tenant; batch_id + event_id + timestamp + type; ack only after durable write to Pub/Sub; 429 when over tenant limit
- Query: REST, tenant from token, full CRUD (GET/POST/PATCH/DELETE), cursor pagination, event search requires bounded time range (cost)
- Outbound: webhook/Slack/email, signed, retried, idempotency key
- Thinking questions: Who & how often? Lifecycle (CRUD)? Retried? Allowed? How big?

### High-level design
- FR1: Agents -> LB -> Ingest (auth, validate, rate-limit, ack #1 after publish) -> Pub/Sub raw-events -> fan-out:
  Stream processor (windows in memory, Bigtable lookups, rules cached from Postgres, ack #2 after outputs) -> Pub/Sub detections;
  Archiver -> GCS Parquet; BigQuery writer
- FR2: detections -> Correlator (group by tenant+asset+type+15min, upsert) -> Postgres alerts -> Pub/Sub alert-events -> Notifier (retries, signed, idempotency key). HIGH notifies immediately.
- FR3: Dashboard API -> Postgres (alerts, with event snapshot -> instant) ; BigQuery async search (partitioned/clustered, capped bytes, quotas) ; GCS for >90 days
- Regional cells: full independent stack per region (EU/US), tenant pinned to home region; only global rule catalog replicated
- AI (per interviewer): FINDS anomalies over compact aggregated behavior profiles (per asset/hour), batch/micro-batch, outputs detections. Never feed raw rows.

### Rules of thumb
- Where state lives: Minutes -> processor Memory; Billions of key lookups -> Bigtable; Big ad-hoc Questions -> BigQuery
- Two acks: never let go of data until the next hop holds it -> at-least-once -> therefore dedup/idempotency
- Funnel: 1M events/s -> ~100 detections/s -> ~1 alert/s; each layer uses storage that fits its volume
- Aggregate for patterns/counts; keep raw when the single event is evidence or timing matters (keep both: raw in GCS)
- BigQuery = analytics, never per-request backend; precompute -> serve from Postgres/Redis
- Start simple; add Redis etc. only when a number demands it
- Never drop an interviewer-given requirement; propose-and-confirm assumptions with numbers
- Use real stories: Armadillex multi-stage LLM pipeline (75% cost cut, 15m -> 2m), self-healing stuck-job fixer, Grid-Sight Airflow/BigQuery

## Part 2 — Memory Palace: "The House"
Walk through YOUR OWN home in this order. Swap the rooms for your real ones; keep the images.

1. FRONT DOOR — Functional requirements
   A bouncer shaped like a giant letter F asks every guest: "What do you want to DO here?"
   He lets in exactly 3 guests and kicks everyone else into a trash bin labeled OUT OF SCOPE.

2. HALLWAY — Non-functional requirements (your lettuce story)
   A SCALE on the floor. A FRESH lettuce holding a 1-minute stopwatch steps on it, lifts a heavy
   DURACELL battery that it never drops. Someone asks "Are you AVAILABLE?" - "No, I'm in ISOLATION,
   in my EU room." - "Fine, I'll keep this meeting on RETENTION for 90 days." The lettuce pays
   the bill with a credit card (COST). Every item in the hallway has a price tag with a NUMBER on it.

3. LIVING ROOM — Estimates
   The wall clock shows 100,000 instead of 12. Rule: only move furniture if the number changes the layout.

4. DINING TABLE — Entities (how to FIND them for any problem)
   Look around the table and ask 5 questions:
   - Who is SITTING? -> the actors (user, driver, customer, device...)
   - What is on their PLATE? -> the things they create/send (post, order, message, event...)
   - What comes OUT of the kitchen? -> what the system produces (feed, match, alert, report...)
   - Who PAYS the bill? -> the owner of everything (account / org / tenant) -> its id goes on everything
   - Is the food raw, cooked, eaten? -> status/lifecycle fields
   Each guest wears a NAME TAG with only 2-3 fields. Takes 1-2 minutes, then move on.

5. KITCHEN — API (three doors, for any system)
   - Delivery door = WRITE path (data comes IN): batched if high volume, numbered crates (idempotency id),
     receipt only after it's safely stored (ack).
   - Waiter door = READ path (data goes OUT on request): page numbers (pagination), identity read from
     the wristband = auth token, never from what the guest claims.
   - Take-out window = PUSH path (system calls others): sealed bags (signed), re-delivered if nobody answers (retries).
   Protocol: REST by default; gRPC for machine-to-machine high volume; WebSocket/SSE for live push to a browser.
   Poster on the fridge: "Wild Llamas Rarely Ask Sizes" = Who & how often? Lifecycle (CRUD)? Retried? Allowed? Size?

6. KIDS' ROOM — High-level design
   A toy train set. One track per functional requirement - build them one at a time.
   A toy train (one request / one piece of data) rides the track while you narrate its journey.
   Every station has a sign that says WHY. Next to the tracks: a PARKING LOT for hard problems you'll solve later.
   At every station ask: In/out? Where is state? What if it dies? Fast or slow path?
   Stop when every track exists; remove any station without a WHY sign.

7. BATHROOM — Deep dives (you DIVE into the tub)
   Toys in the tub, one per deep dive (filled in as we learn them).

8. BALCONY — Wrap-up
   Look at the view: trade-offs made, what you'd monitor (dashboards on the neighbors' windows), v2 on the horizon.

### Rules of thumb hooks (generic)
- Where state lives - "Memory for Minutes, Key-value store for Key lookups at huge scale, Warehouse for Wide questions"
  (GCP: process memory / Bigtable or Redis / BigQuery). Transactions & correctness -> relational DB (Postgres).
- Relay race baton: never let go of data until the next runner holds it (acks) -> two runners may hold it (duplicates) -> dedup.
- Coffee filter: big volume in, small result out - each layer uses storage that fits its volume.

### Rehearsal schedule
1. Now: walk the house from memory, out loud, no notes.
2. Tonight before sleep: walk it again; for each room say one example from our alerting system.
3. Tomorrow morning: walk it once more; then 5 minutes before the interview.
