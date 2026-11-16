# Interview morning sheet (read 10 minutes, nothing new)

## The 8 rooms
Door: functional (3 guests, out-of-scope bin) -> Hallway: non-functional with numbers -> Living room: estimates (say "so what") -> Dining table: entities -> Kitchen: API -> Kids' room: high-level design -> Bathtub: deep dives -> Balcony: wrap-up

## Notebook template (write it at the start)
- DOES: "[actor] can [verb] [thing]" x3
- HOW WELL: scale / freshness / durability / availability / isolation+residency / retention / cost / quality (recall first)
- NOT MINE: out of scope
- After every number: "So what does this force the design to do?"

## Top 3 fixes from the mock
1. Structure: sentence starters. "Functionally, three things..." "Non-functionally, my checklist..."
2. Take the computation to the data: if it's in BigQuery, summarize with SQL there (Airflow hourly), move only small results.
3. Name mechanisms: baseline + peers -> score -> top candidates (cost dial) -> cheap-first cascade -> LLM verdict -> findings; feedback -> test set; precision vs recall (recall never drops); shadow mode; injected attacks.

## Numbers
- Day ~100k seconds: 50B/day ~ 500k/s
- Postgres ~10k writes/s, Redis ~100k ops/s, BigQuery ~$6/TB scanned, LLM seconds per call
- Always convert to the limit's unit (per minute vs per hour)

## Phrases
- "I'll assume X - does that sound right?"
- "Don't make the LLM the boss." / "Statistics narrow it down; the LLM judges the shortlist."
- "At-least-once plus idempotent writes; atomic claim against races."
- "We degrade into delay, not loss."
- "X over Y because Z; I'd revisit if W."

## Stories (use at least one)
- Armadillex: multi-stage LLM pipeline, cheap-first chosen by testing, never ask twice, on-demand vs batch paths, 75% cost cut, 15 -> 2 min; prefix caching lesson.
- Grid-Sight: Airflow + BigQuery pipelines.
- Hourglass King: AI agents with proof-based acceptance; cloud-save lost-ack bug; bot simulations to find the wall.
