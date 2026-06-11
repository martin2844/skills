# Scaling — Full-Repo Audits

Diff-mode audits are single-threaded by default — the scope is
small and the main conversation holds everything. `--full` mode is
different: a real Next.js app has dozens to hundreds of files
across the rule surfaces, and a sequential scan blows the context
window or takes forever.

For `--full` (and only for `--full`), spawn parallel category
subagents.

## When to parallelize

- Always on `--full`.
- Never on `--fix`-only invocations (those operate on findings
  already gathered).
- Never on diff mode, regardless of size — diff mode caps scope by
  definition. If a diff is genuinely huge, ask the user whether
  they want `--full` instead.

## Orchestration recipe

1. **Run discovery yourself** (`references/discovery.md`). Version,
   router, guide path, dominant patterns, ambient tooling. Everything
   downstream depends on this — never delegate it.
2. **Spawn one subagent per category in parallel** (one message,
   multiple Agent tool calls). The categories:
   - boundary (`rules-boundary.md`)
   - auth (`rules-auth.md`)
   - caching (`rules-caching.md`)
   - effects (`rules-effects.md`)
   - conventions (`rules-conventions.md`) — only if a guide was
     found
3. **Each subagent's prompt includes:**
   - Discovery output (version, router, guide path or "none",
     ambient tooling note).
   - The rule catalog file path it should load.
   - The scope: list of dirs/files to walk.
   - **Output contract**: return candidates as
     `file:line — title / trigger / path / wrong outcome / evidence`.
     Subagents emit *candidates*, not findings.
   - Constraints: read-only, no edits, no GitHub interaction.
4. **You run the verification gate yourself** on every candidate
   (`references/verification-gate.md`). Subagents may have skipped
   guard-lookups, missed a layout, or flagged a stale convention.
   Never relay a subagent's output unverified.
5. **Dedup before output**: merge findings within ~5 lines, sharing
   a root cause, or appearing across many files for the same reason
   (e.g. 12 actions all missing the documented auth pattern is ONE
   finding with 12 locations).
6. **Write the report yourself** (`references/delivery.md`). Voice
   and ordering never come from a subagent.

## Subagent type

Use `Explore` (read-only, fast) for finders. They get the rule
catalog and a scope list, and walk the codebase pattern-by-pattern.
If `Explore` isn't available, fall back to `general-purpose` but
explicitly forbid edits in the prompt.

## Anti-patterns to avoid

- **Don't split by file path.** Each subagent should hold the
  whole rule catalog for its category and walk all relevant files,
  so it can see cross-file patterns (e.g. "this action's auth
  guard lives in the layout"). Splitting by path loses that view.
- **Don't ask subagents to do verification.** They report
  candidates; the main thread runs the gate. Trying to push the
  gate into subagents either invites false positives (they don't
  see the layout three dirs away) or false negatives (they
  over-refute and drop real findings).
- **Don't let subagents write reports.** They emit structured
  candidates; the main thread writes prose. Voice is centralized.
- **Don't let subagents touch tests/builds.** Discovery happens
  once, in the main thread, before spawning. Subagents read; they
  don't run commands beyond what they need for evidence.

## When subagents return nothing

A category returning zero candidates is a successful run, not a
broken subagent. List it in the report's "Checked clean" section
with a one-line summary of what was checked.

## Context budget

For very large repos (1000+ files in `src/app/`), the boundary and
effects passes are the heaviest. If a subagent reports it's running
out of context, split *that* category by directory and re-spawn —
but only as a fallback, not the default. Most repos don't need it.
