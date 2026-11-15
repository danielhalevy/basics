# Hourglass King — how to use it in the interview

## The 60-second pitch ("tell me about a project you're proud of")
"On the side I'm building The Hourglass King, a 2D action game for Android in Godot. I'm one developer, and I built it by directing AI coding agents. It's in closed testing on Google Play: 12 releases so far, with the first production build next.
The interesting part is the engineering process. I run a coordinator agent that splits work into small parallel 'lanes', often 8-10 at once, over 100 sessions in the last few days alone. Nothing counts as done without proof: tests first, real-game screenshots or video, then dedicated QA lanes and a final release gate. The suite has about 6,000 automated tests. I own every design decision and play-test every build myself.
Two problems I'm proud of: cloud-save conflicts, where a lost upload reply left the sync state stale, and economy balancing, which I did with Monte-Carlo bot simulations instead of guesses."

## When to bring it up (don't force it; keep it to 60-90 seconds)
- "Tell me about yourself / a project": use the pitch.
- "How do you use AI coding tools?" (the job description asks for exactly this): agents are fast but sometimes confidently wrong, so I accept work based on proof, not trust.
- At the end, "Anything else you'd like to share?": a short version.

## Hooks inside the system design itself (one sentence, then back to the design)
- Acks / duplicates / lost replies: "I hit exactly this in my game's cloud saves: the upload succeeded but the reply was lost, so the client thought it hadn't synced and showed false conflict popups. The fix was tracking a per-device last-synced hash plus pending uploads, and only flagging a conflict when both sides had really changed."
- Batch pipelines and orchestration: "My animation pipeline is a batch job queue on Kaggle GPUs: queue file -> packed batch -> push, poll, fetch -> validate the output (size, format) -> a release gate. It's the same pattern as an Airflow DAG with validation steps."
- Evaluating AI before trusting it: "Before launch I ran bot simulations with different skill levels to find where free players hit a wall (floor 5) and fixed it with data. I'd do the same for an anomaly detector: evaluate offline on a labeled set before it goes live."
- Testing / reliability at scale: "6,000 tests with a hang watchdog and flake triage (does it fail alone, or only under load?)."
- Simplicity trade-off: "I deliberately had no custom backend; I used managed Play services and handled conflicts on the client to ship faster. That's the same 'start simple, add pieces only when a number demands it' principle."

## Likely follow-up questions, with short answers
- "How much did you write vs the AI?" -> "The agents write most of the code. I own the architecture, every design decision and every acceptance. Agents send me decision questions, nothing that changes content ships without my OK, and I approve visuals from screenshots and video."
- "How do you know the AI code is correct?" -> "Tests first, proof from the real game, dedicated QA lanes, a full suite plus lint plus smoke run on the final version, and my own play-test of every build. When I catch a class of mistake, I turn it into a written project rule so it doesn't come back."
- "What went wrong?" -> "Small UI mistakes slipped through, so I delayed a release and added visual and gameplay QA gates. Speed without gates wasn't worth it."
- "How does the save system survive a crash?" -> "Local save is the source of truth; writes are atomic; there are resume points per mode; kill-and-relaunch scenario tests; the cloud copy syncs on pause, close and every 2 minutes."
- "Why no backend?" -> "Cost and speed for a solo project: Play Games saved games, Billing and AdMob are managed. The trade-off is conflict handling on the client and less control; with real scale I'd add a small backend for saves and the economy."
- "How do you coordinate parallel agents without conflicts?" -> "Small, focused lanes with one task each, small commits to main, a coordinator that plans waves so lanes don't touch the same files, and a final gate on the combined result."

## Watch out
- Don't let it take over the system design. One sentence, then back to the design.
- Say "I directed and verified", never "the AI did it all."
- Know your numbers: ~6,000 tests, 12+ releases, 100+ lanes, 8-10 in parallel, ~1.5 months, solo.
- Be ready to explain one system deeply yourself (the save/cloud sync is the best one).
