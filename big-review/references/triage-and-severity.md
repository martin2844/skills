# Triage and Severity

## Triage

Classify every changed file before reviewing. This is what keeps the
review focused and lets it scale to large PRs without spraying nits.

### Review it
The diff touches behavior — anything that could introduce a production
bug:
- control flow, conditionals, branching
- function calls and arguments
- assignments and data structures
- error handling
- queries and migrations
- API contracts and serialized payloads
- queue / job / retry behavior
- config or env that changes runtime behavior

### Clear it
The diff cannot change behavior:
- formatting / whitespace only
- typo-only comment edits
- generated files
- lockfile churn with no relevant dependency-behavior change
- pure import reordering
- pure rename with no semantic change

### Rules
- **When in doubt, review it.** Reading a safe file costs seconds;
  waving through a behavior change costs a prod bug.
- **Cleared files are a hard skip** — no comments on them, ever.
- **List cleared files** in the cover note / Cleared section so the
  author sees the gate ran.
- If you catch yourself wanting to nit a cleared file, that's the
  style-noise instinct — suppress it.

## Severity

### High — must address before merge
- production correctness bug
- data loss
- security issue (injection, SSRF, auth bypass, secret/PII leak)
- broken invariant
- missing authorization
- sticky / stranded state after an error
- queue or job retry poison (non-retryable error wedges the queue)
- migration that can corrupt or strand data
- load-bearing docstring/config claim contradicted by implementation
- performance issue likely to hurt at production scale

### Medium — should address or explain
- recoverable bug
- confusing abstraction that doesn't fit
- test-isolation risk
- inefficient duplicate work
- missing validation with limited blast radius
- unclear behavior that could break the next PR
- real but non-blocking maintainability cost

### Low — optional
- naming
- small cleanup
- comment trim
- minor type ergonomics
- small symmetry improvement

### Rules
- The tier test is: **is this worth delaying the merge for?** High =
  yes, unconditionally. Medium = needs an answer or a fix, but a good
  answer unblocks it. Low = never delays anything, and says so.
- When unsure between two tiers, pick the **lower** one unless there
  is concrete production risk.
- Sort High → Medium → Low; within a tier, by blast radius.
- Severity controls the verdict. It is information, not a rhetorical
  weapon — don't inflate it to force action.
