# Intent and Context

Understanding is the bottleneck of review quality — most bad reviews
come from judging hunks without knowing what the change is for or how
the codebase around it works. Spend budget here before judging
anything.

## Intent pass

### 1. Learn what the change is for
Read, in order of availability: the PR description, the linked ticket
(`gh pr view` body often links it; fetch it if reachable), commit
messages, and the tests the PR adds or changes (tests are the spec —
read them *before* the implementation when possible).

Restate the goal in one sentence to yourself: *what problem, what
intended behavior change.* If you can't, that's already a finding for
larger changes ("the PR doesn't say what it's for") — or a question to
the author.

### 2. Check the implementation against the stated goal
- Enumerate the stated requirements / acceptance criteria. For each:
  implemented, partially implemented, missing, or can't-verify-from-code.
  A requirement the code silently doesn't meet is usually a **High** —
  the PR "works" and still fails.
- Flag **unrequested behavior changes**: the diff changes behavior the
  description never mentions (a default flipped, an error now
  swallowed, an endpoint's shape changed). Either it's a bug or the
  description is lying; both are findings.
- Edge cases in *product* terms: what happens to the existing user
  with NULL in this column? What does the caller see when this times
  out? If the answer matters and isn't covered by code or tests, say
  so concretely.

### 3. Design before lines
Ask once, early: **does this change even make sense?**
- Right layer/repo/abstraction? A special case bolted onto shared
  infrastructure is usually a sign the fix isn't deep enough.
- Does it duplicate a capability the codebase already has?
- Over-engineering: is it solving a speculative future problem instead
  of the current one?

If the answer is a fundamental design problem, **lead with that single
finding and stop polishing**. Don't deliver ten line-level nits on
code you're recommending be restructured — that's priority inversion
and it wastes everyone's time. Line-level review resumes after the
design question is settled (or if the design concern is a "worth
discussing" rather than a blocker, deliver it alongside, clearly
separated).

### 4. Uncertain behavior needs evidence, not reassurance
When a correctness question can't be settled by reading (does this
query plan hold at production size? does the provider retry?), the
right ask is **a test or empirical data** — not prose. Phrase it that
way: "a test pinning this would settle it" / "what does the prod data
show for X?". Never accept — or write — "this should be fine."

## Context pass

Do this *before* judging any hunk. Each check is cheap; skipping them
is where false positives and missed conventions come from.

### Conventions
- Read CLAUDE.md / contributing docs relevant to the changed paths.
- Look at how **sibling code** does the same thing (grep for a
  parallel implementation: another handler, another job, another
  migration). The siblings define the local convention.
- Never flag a "convention violation" you haven't verified is actually
  a convention — one neighboring example is evidence; your taste is not.

### Callers and contracts
- For every changed/renamed/removed public symbol, **grep for its call
  sites**. Check each one against the new contract: new precondition,
  changed return shape, new exception, changed timing/ordering.
  An un-updated call site is a finding even though it's outside the
  diff — the diff broke it.
- For symbols wired by convention (registries, `camelize` lookups,
  routing tables, DI containers), find the wiring before concluding
  "this is never registered."

### History
- When a touched line looks odd or a deletion looks risky, run
  `git log -L` / `git blame` on it. "Odd" code is often a deliberate
  fix for a past bug — deleting it reintroduces the bug, and the
  commit message tells you so.
- Files that historically change together but didn't in this PR are a
  hint something was missed (the config but not the doc, the schema
  but not the serializer).

### Reuse
- Before accepting new helper code, check whether the codebase already
  has it (shared/util modules, files adjacent to the change). The
  finding names the existing helper — "this re-implements `X` from
  `lib/y.ts`" — never a vague "consider reusing."

### What's NOT in the diff
The senior-reviewer move is noticing the absence. Check for:
- **Missing failure-path tests** — happy path tested, error path not.
- **Migration compatibility** — can the migration run against code
  that's still live (zero-downtime)? Can old and new code coexist
  during rollout?
- **Un-updated call sites** (above).
- **Feature-flag symmetry** — a flag checked in N places that's only
  handled in N−1; a flag added with no removal path.
- **Docs/config** — README, env examples, runbooks that the change
  invalidates.
- **Observability** — a new failure mode with no log/metric to detect
  it in prod (see depth mode 13).

Only report an absence when it has a concrete cost you can name; "no
docs" on an internal refactor is noise.
