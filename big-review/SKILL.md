---
name: big-review
description: Deep, evidence-first code review for local diffs and GitHub PRs. Use for "big review", "deep review", "review my code", "pre-PR review", "review-proof this", "rabbit review", "fast review", "triage this PR", semantic correctness review, failure-path review, and high-signal PR review.
when_to_use: Use when the user wants a serious code review of changed files, a local branch, an uncommitted diff, or a GitHub PR. Prefer this over generic review when correctness, failure paths, tests, trust boundaries, state transitions, intent-vs-implementation, or production behavior matter.
argument-hint: "[optional PR number, branch, file path, or review mode]"
---

# Big Review

Review like a sharp senior engineer, not a linter. Catch bugs that
compile, pass the obvious tests, and still don't work — and catch the
PR that works perfectly but doesn't do what it was supposed to do.

## Purpose

High-signal review built from five moves:

1. **Intent** — what is this change *for*, and does it actually do
   that? Design before lines.
2. **Triage** — don't review files that can't carry a bug.
3. **Context** — mine the codebase (conventions, siblings, callers,
   history) before judging any hunk.
4. **Find** — a mechanical surface pass plus depth-pass thinking modes
   (semantic correctness, failure paths, removed behavior, caller
   contracts, trust boundaries, tests that would actually fail).
5. **Verify** — try to *disprove* every candidate finding before it's
   allowed out. Findings that survive carry evidence and a concrete
   failure scenario.

Core stance:
- Review **changed behavior and intent**, not formatting noise.
- Read each reviewable file **end-to-end**, not just the changed hunks.
- Every finding survives a **disproof attempt** and cites **evidence**.
- Prefer **fewer, stronger** comments. One proven High beats ten nits.
- A missed real bug and a false positive are both failures, but they
  are not symmetric: false positives destroy trust in *every* future
  comment. When impact is low and confidence is low, stay silent.
- If nothing is wrong, say **"No findings."** Clean code is a good
  outcome, not a prompt to invent a concern.

## Default workflow

1. **Determine the target.** One of: uncommitted local diff; current
   branch vs merge base; specific files; a GitHub PR; a pasted diff.
   If ambiguous, ask once; otherwise default to the local branch vs
   merge base.
2. **Get the diff (read-only — never mutate repo state while gathering
   context).**
   - Local branch: `git merge-base origin/main HEAD` to find `<base>`,
     then `git diff <base>...HEAD` (three-dot).
   - Uncommitted: `git status --short`, `git diff`, and `git diff
     --staged` if relevant.
   - GitHub PR (only when asked and `gh` is available): `gh pr diff
     <n>` and `gh pr view <n>`. Fetch full file contents from the head
     ref before judging (see `references/github-delivery.md`). Never
     review from hunks alone.
3. **Intent pass** (`references/intent-and-context.md`). Read the PR
   description / ticket / commit messages, restate the goal, and check
   the change against it. If the *design* is wrong — wrong layer,
   wrong approach, shouldn't exist — say so now and don't line-polish
   code you're recommending be rewritten.
4. **Triage each file** → *review it* or *clear it*. Cleared files are
   a **hard skip** — no comments on them.
   (`references/triage-and-severity.md`)
5. **Context pass** (`references/intent-and-context.md`). Conventions,
   sibling implementations, callers of changed symbols, git history on
   suspicious lines, existing helpers, and what's *missing* from the
   diff. Read tests first — they're the spec.
6. **Read each reviewable file end-to-end** before judging any hunk —
   including resolving how changed symbols wire up (registries,
   conventions, tables in other files).
7. **Surface pass** (`references/surface-checklist.md`).
8. **Depth pass** (`references/depth-modes.md`).
9. **Verification gate** (`references/verification-gate.md`). Every
   candidate gets a disproof attempt and a concrete failure scenario,
   or it dies here. Apply the comment budget.
10. **Assign severity** High / Medium / Low and order by blast radius
    (`references/triage-and-severity.md`).
11. **Write output** in the requested mode, using
    `references/voice-and-output.md`.

For large PRs (roughly >12 reviewable files or >1000 changed lines),
parallelize the find/verify phases with subagents — see
`references/scaling.md`.

## Mode selection

### Normal review (default)
Triggers: "review my code", "big review", "deep review", "pre-PR
review". Output: the full chat template — verdict, cleared files,
findings grouped by severity, checked-clean list.

### Fast / rabbit review
Triggers: "rabbit review", "fast review", "quick sweep", "triage this
PR". Compressed workflow: skip the deep context pass, keep the intent
check and the verification gate (speed never excuses a false
positive). Terse output: one compact `file:line — issue. Fix: …` per
finding, skip most Lows, `LGTM` for small ranges checked and found
clean.

### Self-review / local-fix mode
When the PR/branch is the user's own, or they ask you to fix findings
locally. **Report findings in chat by default.** Apply fixes only when
explicitly asked; keep them narrow; run the relevant tests (or say why
not); summarize what changed. Do **not** post review comments or
self-approve.

### GitHub PR review mode
When the user asks to prepare or post a PR review. Default to
**preparing** a ready-to-post plan (path, line, body, intended event).
**Post nothing unless explicitly asked.** Prefer inline comments;
severity → event: any High → `REQUEST_CHANGES`; no High + ≥1 Medium →
`COMMENT`; only Lows or none → `APPROVE`. Never self-approve. See
`references/github-delivery.md`.

## Reference loading map

- **Always** read `references/intent-and-context.md` (intent pass +
  context mining) and `references/triage-and-severity.md` (triage gate
  + severity tiers).
- Read `references/surface-checklist.md` for the surface pass.
- Read `references/depth-modes.md` for the depth pass.
- Read `references/verification-gate.md` **before deciding whether any
  finding is worth emitting** — no finding skips the gate.
- Read `references/voice-and-output.md` before writing final comments.
- Read `references/github-delivery.md` **only** when preparing or
  posting GitHub inline comments.
- Read `references/scaling.md` **only** for large PRs.

In fast mode you may skip the deep parts of intent-and-context, but
never skip the verification gate or voice rules.

## Output contract

```
## Verdict
[Request changes / Comment / Approve / No findings]
One human line. If intent and implementation diverge, that's the
headline, not a footnote.

## Cleared
- `path`: reason

## Findings

### High
1. `file:line` — finding title
   Issue:
   Evidence:
   Fix:

### Medium
...

### Low (non-blocking)
...

## Checked clean
- `path`: what was checked
```

Rules:
- No findings → write "No findings." and skip the Findings section.
- Never manufacture findings; never pad a clean review.
- Never leak internal labels (D1, S4, CONFIRMED/PLAUSIBLE) into
  author-facing text.
- Don't duplicate linter/formatter/typechecker output.
- Don't comment on cleared files.
- Don't comment on pre-existing problems the diff didn't touch or
  worsen (at most one explicitly out-of-scope note, if it's serious).
- Don't speculate without evidence.

## Non-goals

- No generic lint review or style bible.
- No summarizing the PR back to the author.
- No praise.
- No broad-rewrite requests for a narrow bug.
- Don't invent repository conventions — verify them.
- **Never** mutate files, commit, push, or post a review unless the
  user explicitly asks. Gathering context is read-only.
