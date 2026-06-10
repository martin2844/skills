# Scaling — Large PRs

Review quality collapses on big diffs: comment usefulness drops
sharply as changeset size grows, because the reviewer's understanding
is the bottleneck. For roughly **>12 reviewable files or >1000 changed
lines**, parallelize — don't skim.

## First: say so

If the PR is large enough that review quality suffers, the verdict
should note it once ("this would review better as two PRs — the schema
change and the handler rework are independent"). Then review it anyway.

## Orchestration recipe

1. **Do the intent pass and triage yourself** (cheap, and everything
   downstream depends on it). Note conventions from CLAUDE.md and the
   user's stated focus areas — these ride along in every subagent
   prompt.
2. **Spawn parallel finder subagents** (Explore or general-purpose, in
   one message so they run concurrently). Split by *angle*, not by
   file, so each keeps whole-change context:
   - line-level correctness over the diff + enclosing functions
   - removed behavior / caller contracts (depth modes 15–16)
   - state-after-throw, idempotency, races (modes 5, 10, 11)
   - tests-as-spec + tests-that-would-fail (mode 6) + what's missing
     from the diff
   - trust boundaries + data volume (modes 7, 14) when relevant
   Each finder returns **candidates**: `file:line`, the causal chain
   (trigger → path → wrong outcome), and quoted evidence. Tell finders
   to pass through every candidate with a nameable failure scenario —
   *finding* is their job, the gate is yours. Don't give finders write
   access.
3. **Verify every candidate yourself** through the verification gate
   (disproof attempt, verdict, FP precedents, admission test). Never
   relay a subagent's finding unverified — subagent output is a lead,
   not a finding. For very large hauls, spawn per-candidate verifier
   subagents whose brief is to *refute* with quoted evidence; you
   still own the final verdict.
4. **Dedup before output**: merge candidates within ~5 lines of each
   other or sharing a root cause; the budget (verification-gate step
   6) applies to the merged list.
5. **Write the output yourself** — voice and cover note never come
   from a subagent.

## When NOT to parallelize

Under the size threshold, subagents cost more than they add — the
serial workflow with full-file reads is more coherent because one
reader holds the whole change. Fast/rabbit mode never spawns
subagents.
