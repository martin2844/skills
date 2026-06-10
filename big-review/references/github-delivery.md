# GitHub Delivery

Read this **only** when the user asks to prepare or post a GitHub PR
review. Gathering context and reviewing are read-only; posting is an
explicit, separate action.

## Gathering context for a remote PR (read-only)

A PR you're reviewing usually isn't checked out, so you only have the
diff. Before judging any hunk, fetch the **full** contents of each
reviewable file (and any file a changed symbol depends on) from the
PR's head ref:

```
gh api repos/{owner}/{repo}/contents/{path}?ref={head} --jq '.content' | base64 -d
```

Get the head ref from `gh pr view <n> --json headRefName`. Use this to
verify the things hunks hide — how a new stage/handler/route is wired
up by convention or in a registry, what a base class's contract is,
whether a sibling already enforces an invariant. This is the
difference between "this symbol is never registered" (a false positive
from hunk-only reading) and confirming it resolves by a `camelize`
naming convention three files away. Fetching is read-only; it never
mutates the repo.

## Preparing review comments

- Prefer **inline comments** for line-specific findings.
- One short **top-level cover note**: a verdict and a pointer to the
  inline notes, in plain human voice — **not** a recitation of
  everything you verified, not the PR description quoted back, not a
  findings preview. One or two sentences. See the cover-note rule and
  worked rewrite in `voice-and-output.md`. (A detailed checked-clean
  list belongs in a *chat* review, not a posted cover note.)
- Each inline comment anchors to **its own** finding's changed line
  (not a shared line per file).
- If the real issue sits outside a diff hunk, anchor to the nearest
  changed line and name the true location in the body — GitHub rejects
  comments on lines outside the hunk.
- **Post nothing unless explicitly asked.**

## Review event mapping

- Any **High** → `REQUEST_CHANGES`.
- No High, **≥1 Medium** → `COMMENT`.
- **Only Low** findings → `APPROVE`, with a note that comments are
  non-blocking.
- **No findings** → `APPROVE`.
- If the PR author is the user, **do not self-approve** (see below).

## Posting checklist

Before posting, confirm:
1. The target PR number.
2. Every comment's line anchor is a real changed line.
3. No duplicate comments.
4. Severity → event mapping is correct.
5. No internal labels (D1, S5, Confirmed/Plausible) leaked into bodies.
6. Every comment is actionable (line + impact + fix).
7. Every finding passed the verification gate — nothing posts on a
   hunch. A `suggestion` block appears only when it fully fixes the
   issue as written.

**If only preparing:** output a ready-to-post plan — for each comment,
`path`, `line`, `body`, plus the intended top-level `event`. Stop
there.

**If posting:** use `gh` / the GitHub API only when it's available and
the user explicitly requested posting. The mechanics: one
`POST /repos/{owner}/{repo}/pulls/{n}/reviews` with a `comments: [...]`
array (`path`, `line`, `side: "RIGHT"`, `body`) and the top-level
`event` + cover-note `body`. To anchor lines without cloning:
`gh api repos/{owner}/{repo}/contents/{path}?ref={branch} --jq '.content' | base64 -d | nl -ba`.
Do not push commits or edit files as part of posting.

## Inline comment style

Good:
> `handlers/job.ts:42` — This catch marks the job complete even when
> `persist()` throws, so downstream sees a successful job with no write.
> Re-throw after logging, or move the completion marker after
> persistence.

Bad:
> `Medium / D5: this error handling seems problematic and may have
> wider impacts.`

(The bad one leaks the label, hedges, names no concrete failure, and
gives no fix.)

## Self-review rule

When the PR is the user's own:
- Don't post a review and don't approve (GitHub blocks self-approve
  with findings anyway, forcing a mismatched verdict).
- Report findings **in chat**.
- Apply fixes locally **only when the user explicitly asks**; keep them
  narrow; commit/push only on explicit request.
