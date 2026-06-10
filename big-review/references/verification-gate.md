# Verification Gate

No candidate finding reaches the author without passing this gate.
False positives are the failure mode that kills review tools: every
hallucinated bug teaches the author to skim the next ten real ones.
The gate exists to make *disproof cheap and emission expensive*.

## Step 1 — Reasoning before finding

For each candidate, write the causal chain (internally, never in the
output): **trigger → path → wrong outcome.** Concretely: what input,
state, timing, or environment occurs; which lines execute; what
incorrect result, crash, or stranded state follows.

If you cannot name the trigger, you don't have a finding — you have a
vibe. Go inspect until you can name it, or drop it.

## Step 2 — Active disproof attempt

Try to kill your own finding before the author has to:

- **Re-read the actual line.** Does the code really say what you think
  it says? Quote it.
- **Look for the guard elsewhere.** Types, schemas, callers,
  middleware, sibling code, tests — does something already enforce the
  invariant you're defending? If yes, there's no finding.
- **Check it's not pre-existing.** Diff the base version / `git blame`.
  If the problem exists on main and this diff didn't touch or worsen
  it, it's out of scope (at most one explicitly-labeled out-of-scope
  note if it's genuinely serious).
- **Check it's not intentional.** PR description, commit message,
  nearby comment, ticket. An intentional, stated behavior change is
  not a bug (though it can still be a bad idea — argue it as design,
  not as a defect).
- **Check the convention.** Before flagging "inconsistent with the
  codebase," confirm the codebase actually does it the other way.

## Step 3 — Verdict

- **Confirmed** — you can name the triggering input/state and the
  wrong outcome, and quote the evidence. Emit it.
- **Plausible** — the mechanism is real but the trigger depends on
  runtime conditions you can't see (race timing, env config, provider
  behavior, prod data shape). Realistic rare paths *count* — error
  handlers, cold caches, missing optional fields, retry storms,
  boundary values. Don't refute a finding merely for being
  "speculative" when the state is reachable.
- **Refuted** — the code doesn't say that, or a guard you can quote
  handles it. Drop it silently.

### Asymmetric confidence rule

Impact decides what to do with **Plausible**:
- **High impact + plausible** → emit, with the uncertainty explicit
  and a cheap way to settle it ("if the provider retries, this
  double-writes — a test pinning idempotency would settle it").
  A missed data-loss bug costs more than one hedged comment.
- **Low impact + plausible** → drop. Nobody needs a hedged nit.

## Step 4 — Known false-positive precedents

Accumulated patterns that look like findings and aren't. Don't emit:

- Anything a **linter, formatter, typechecker, or compiler** would
  catch — it's not your job and it's already caught.
- **Pre-existing issues** on untouched lines (see step 2).
- **Intentional behavior changes** stated in the PR description.
- "This **might break something elsewhere**" without a named call site
  you actually checked.
- **Generic robustness advice** ("consider adding error handling",
  "might want validation") with no concrete failure scenario.
- **Framework-guarded concerns**: e.g. React/Angular escape
  interpolated content (XSS needs `dangerouslySetInnerHTML` or
  equivalent); ORMs parameterize bound values.
- **Trusted inputs**: env vars, CLI flags, and server-side config are
  operator-controlled, not attacker-controlled.
- **Test/mock/fixture code held to production standards** — flag test
  bugs that make a test lie (see depth mode 6), not test style.
- **Defensive code you'd like to add** where types already rule the
  case out — that's a Low about honesty of types at most.
- A pattern you flagged that the **codebase does deliberately and
  consistently** (check siblings/history first).
- **Pedantic nitpicks a senior engineer wouldn't say out loud.** The
  test: would this comment survive being the only comment on the PR?

## Step 5 — Admission test

Before emitting any surviving finding:

1. Can I point to a **specific line**?
2. Can I describe a **concrete failure or maintenance cost** — not a
   vague worry?
3. Can I cite **evidence** (code, type, test, doc, call site, runtime
   path)?
4. Can I propose a **concrete fix** the author could apply directly?
5. Would I still leave this comment **if there were already five other
   comments** on the PR?

Mostly no → suppress. Borderline Lows failing (5) are exactly the
noise to cut.

## Step 6 — Budget

- Normal review: at most **~10 findings**, ranked most-severe first;
  if the gate passes more, cut from the bottom — correctness always
  outranks cleanup when the cap forces a cut.
- Fast review: at most ~6, Lows only if free.
- Merge findings that share a root cause into one comment; don't
  re-report the same bug at every call site.
- The win condition is the author **acting on every comment**, not the
  comment count. "No findings" is a successful review.

## Anti-sycophancy

You were asked to find issues; that is not evidence issues exist.
Models under "review this" pressure invent concerns on clean code.
If the change is good, the verdict is short and the review ends. Never
pad, never manufacture a Low to have something to say, never convert
"I didn't fully understand this" into a finding — go understand it or
ask a genuine question.
