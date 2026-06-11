# Delivery — Voice, Output, and GitHub Comments

Write the report like a sharp teammate, not a linter. By the time
the report goes out, the internal scaffolding (rule IDs, gate
verdicts, candidate vs confirmed) is gone. The reader sees the
finding, the evidence, and the fix.

## Author-facing voice

- **Observation → evidence → impact → fix.** That's the shape of a
  finding. Two to four sentences. Anything longer needs a reason.
- **Talk about the code.** Never "you forgot", "you broke", "why
  did you…". Subject is the code: "this action skips…", "the route
  treats…".
- **Cite evidence inline.** `file:line` for code, `§X` for guide
  sections.
- **Reserve Blockers for things that bite production.** Don't
  inflate severity to force a fix. Should-fix is honest about most
  caching, IDOR-with-narrow-blast-radius, and convention-skipping
  bugs.
- **Lows ("Nits") are short and say they're optional.**

## Worked example

**Robotic (avoid):**

> **Should-fix — A2 (IDOR, Plausible):** The action exported as
> `updateAgency` in `src/app/actions/dms.ts` line 42, after passing
> our verification gate, is judged to be missing an ownership
> filter on the SQL UPDATE statement. The auth check is present
> but only verifies that *a* session exists; it does not verify
> that the session's user owns the `dms_agencies` row identified
> by `input.id`. This is an instance of the IDOR pattern documented
> in the rules-auth catalog.

**Human (use):**

> `app/actions/dms.ts:42` — `updateAgency` checks `auth()` but
> updates `dms_agencies` filtered only by `id`, so any logged-in
> user can rename any agency by calling the action with that
> agency's ID. Add `.where({ owner_user_id: session.user.id })` to
> the update.

Same content, ~1/3 the length, no scaffolding.

## What never goes in the report

- Internal labels (rule IDs like `A2`, gate verdicts like
  `Confirmed/Plausible`, severity reasoning).
- Praise of any kind. The audit found things or it didn't — both
  are fine; neither needs commentary.
- Restating what the diff does. The author wrote it.
- "Consider…" unless the finding is genuinely optional.
- Stacking long arguments on Nits.
- Pre-existing issues the diff didn't touch (diff mode), unless
  one is genuinely serious and gets one explicit out-of-scope note.

## Output structure

```
## Verdict
[one human line. Worst finding is the headline.]

## Target
Mode: diff | full
Next.js: 16, App Router
Guide: NEXTJS_GUIDE.md (or "none — best-practices mode")

## Findings

### Blockers
1. `file:line` — title
   Issue: trigger → path → wrong outcome (one sentence)
   Evidence: quoted line or config (one line)
   Fix: concrete change (one sentence)
   [Guide conflict: §X documents this — the convention should change.]

### Should fix
…

### Nits (non-blocking)
…

## Checked clean
- Boundary: no client files import server modules; secrets stay
  server-side.
- Auth: every action/route has a real session/HMAC check.
- Caching: mutations revalidate; no force-dynamic abuse found.
- Effects: useEffect usage is for legit external sync only.
- Conventions: page.tsx files, ID gen, and class merging all
  conform to the guide.
```

Rules:
- No findings → "No findings." Skip the Findings sections, keep
  Checked clean.
- Each finding is self-contained — the reader doesn't have to
  scroll to understand it.

## GitHub `--comment` delivery

When `--comment` is passed and the target is a GitHub PR:

1. **Default to *preparing*, not posting.** Output a plan: for each
   inline comment, `path`, `line`, `body`, and the intended event.
   Only post when the user explicitly confirms ("post them",
   "send", "yes").
2. **Inline comments anchor to the finding's specific changed
   line.** If the finding is about a line outside the diff hunk,
   anchor to the nearest changed line and name the true location
   in the body.
3. **Cover note**: one or two human sentences. Not a checklist
   recitation. Not a quote of the PR body. Just verdict + pointer
   to the inline comments.

   **Robotic (avoid):**
   > Audited 12 files. Boundary: clean. Auth: 2 findings —
   > one Blocker (`updateAgency` missing IDOR check on
   > `dms_agencies` row), one Should-fix (cron silent-disable
   > pattern in `check-report-jobs/route.ts`). Caching: 1
   > Should-fix on `vehicles` mutation. Effects: clean.
   > Conventions: 3 Nits (path aliases, ID gen). Verification
   > pass clean.

   **Human (use):**
   > Two real ones to look at — an IDOR in `updateAgency` and a
   > cron auth that disappears if `CRON_SECRET` is unset. Rest
   > are minor.

4. **Severity → event mapping:**
   - Any Blocker → `REQUEST_CHANGES`
   - No Blocker, ≥1 Should-fix → `COMMENT`
   - Only Nits → `APPROVE` with a non-blocking note
   - No findings → `APPROVE` ("audited clean")
   - **If the PR author is the user, do not self-approve.** Report
     in chat instead.
5. **Comment body** uses the same finding shape as the chat report,
   minus the severity heading (the inline anchor + the body's
   content are enough).
6. **Suggestion blocks** only when the fix is one-to-three lines
   and fully replaces the issue. For multi-line fixes, use a code
   block, not a suggestion.

## Posting mechanics (only when explicitly asked)

```
gh api repos/{owner}/{repo}/pulls/{n}/reviews \
  -F event=<EVENT> \
  -F body=<cover note> \
  -F 'comments[][path]=…' -F 'comments[][line]=…' -F 'comments[][body]=…' …
```

Confirm before posting:
1. The target PR number.
2. Every comment anchors to a real changed line.
3. No duplicates.
4. Severity → event mapping is correct.
5. No internal labels leaked.
6. Each comment is actionable (line + impact + fix).

Never post commits or push commits as part of `--comment`. The
audit's job is to communicate the findings, not to ship the fixes
— that's `--fix`.
