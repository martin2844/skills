# Depth Modes

These are **thinking questions**, not labels. Apply the question to the
diff and report whatever doesn't survive. Never put the mode number in
an author-facing comment — the reader gets the finding, not the
scaffolding.

The illustrations are from a TypeScript/Node service; the *question* is
what transfers — translate it to the stack in front of you.

---

## 1. Comments / docstrings vs implementation

**Question:** Does this behavioral claim match what the code actually
does?
**Apply to:** docstrings, JSDoc, inline comments, README snippets the
PR touches, config docs, test names that encode behavior.
**Prove it by:** reading the implementation; reading parser/helper
behavior; checking the spec/tests; checking call sites. Open the
dependency's own spec when the claim is about a library.
**Good finding:** "Docstring says `AI_QA_MAX_AUTO_RETRIES=0` disables
auto-retry, but `parseIntEnv('0', 5)` returns `5` (`parseIntEnv.spec.ts:21`)
— `0` falls back to the default, so retries stay on. Fix the parser or
the docstring."
**Don't report:** merely redundant comments unless they create
confusion or rot risk.

## 2. Duplicate work / duplicate signals

**Question:** Is the same predicate, fetch, filter, transform, or
signal computed twice?
**Apply to:** duplicate DB queries, count-then-fetch patterns, repeated
filters/transformations, duplicate validation, repeated expensive
calls.
**Prove it by:** comparing predicates and query conditions; checking
whether one result can derive the other.
**Good finding:** "This counts `ai_qa` rows, then a second call fetches
all history and re-filters on the same predicate in memory — two
round-trips for one signal. Fetch once with `listSince(...)` and derive
the count from `.length`."
**Don't report:** intentional duplication for isolation, clarity, or
different consistency boundaries.

## 3. Abstraction scope and effectiveness

**Question:** Does this abstraction actually bound or protect what it
claims to?
**Apply to:** limiters, locks, caches, debouncers, throttles,
transactions, feature flags, queues, rate-limiters.
**Prove it by:** identifying where it's instantiated vs the scope it
must protect; comparing those scopes.
**Good finding:** "`pLimit(1)` is created inside the per-message SQS
handler, so each message gets a fresh limiter — it bounds nothing
across messages. Lift it to module scope or drop it."
**Don't report:** simple helpers not meant to bound shared behavior.

## 4. Labels / logs / errors with real values substituted

**Question:** If real runtime values are plugged into the string, is
the message true?
**Apply to:** logs, errors, audit rows, status labels, retry messages,
metric labels.
**Prove it by:** substituting first-call values; checking off-by-one
counts; checking what the numerator and denominator actually mean.
**Good finding:** "`attempt 1/2` is logged on the first run before any
retry — `1` is the verdict count (incl. the just-written row) and `2`
is the cap, so it reads as 'retry 1 of 2' when zero retries have
happened. Log verdict count and retries-used separately."
**Don't report:** wording preferences with no operational confusion.

## 5. State after every throw

**Question:** After each possible throw, what state are external
systems left in?
**Apply to:** DB writes, queue ack/retry, locks, transactions, audit
rows, cache mutation, partial files, background jobs, outbound side
effects.
**Prove it by:** tracing side effects before and after each awaited
call; checking catch/finally; checking retry behavior; checking whether
a marker/write is skipped on failure. Non-retryable errors are the
worst case — the queue won't redeliver, so broken state stays broken.
**Good finding:** "The `AutoFallback` marker is written *after*
`applyEnglishFallback`. If that throws a non-retryable
`ImageFallbackContentError`, the marker never lands, the next run sees
`count > cap`, re-enters the same branch, and the message poisons the
DLQ. Write the marker in both branches, tagging the error."
**Don't report:** pure in-memory failures with no lasting state, unless
they break a caller contract.

## 6. Tests that would fail if behavior broke

**Question:** Would these tests fail if the production behavior
regressed?
**Apply to:** new tests, changed branches, new error paths, mocks,
snapshots, behavior-claiming test names.
**Prove it by:** finding the production line the test should protect,
mentally mutating it, and checking whether the test would go red.
**Good finding:** "This test stubs the repo to return `[{id: 1}]` and
asserts `id: 1` comes back — it verifies the stub, not the handler's
mapping. Renaming the mapped field keeps it green. Assert on a value
the handler computes, or drive an in-memory repo."
**Don't report:** intentionally narrow tests when another test covers
the behavior.

## 7. Trust boundaries

**Question:** What does this trust that it should verify?
**Apply to:** request bodies, params, query strings, queue messages,
env vars, third-party responses, uploaded files, filesystem paths,
templates, raw queries, outbound URLs, user-controlled IDs.
**Check:** validation, escaping, authorization, path traversal,
SQL/query injection, SSRF-like outbound requests, template injection,
secrets/PII in logs.
**Good finding:** "`req.params.id` is interpolated straight into a raw
SQL string — fine while `id` is numeric, an injection the moment it
isn't. Parameterize, or validate the shape at the boundary."
**Don't report:** inputs already validated by a nearby trusted schema,
unless the trust boundary is genuinely unclear.

## 8. Evidence-first

**Question:** What's the evidence?
Every finding cites proof: file/line, a contradictory type, parser
behavior, a test/spec, a call site, an external state transition, a
runtime path.
**Good finding:** "`types.ts:226` declares `dataSetUrls: string[]`, so
this `?? []` fallback is dead unless the type is wrong."
**Don't report:** "I think this might be wrong" without proof. If you
can't cite it, inspect further or drop it.

## 9. Fix options

**Question:** Is there one obvious fix, or a real design choice?
Default to **one** concrete fix. Offer alternatives only when there's a
genuine tradeoff (narrow fix vs broader boundary fix; cleanup vs
compatibility; strict validation vs backward-compatible parser).
**Don't:** dump three options to look helpful, or dictate architecture
when a narrow fix is enough.

## 10. Idempotency and replay

**Question:** If this runs twice, retries, or replays, does it
duplicate or corrupt state?
**Apply to:** queues, webhooks, jobs, retries, migrations, cron tasks,
external API side effects.
**Good finding:** "This webhook inserts a charge row with no
idempotency key, so a provider retry double-charges. Key the insert on
the provider event ID."

## 11. Ordering and races

**Question:** Does this assume an order that isn't guaranteed?
**Apply to:** async work, concurrent requests, transactions, locks,
cache invalidation, time-based logic, background jobs.
**Good finding:** "This reads before acquiring the lock, so two
concurrent requests both observe missing state and both create it —
check-then-act race. Move the read inside the lock, or use an upsert."

## 12. Backward / forward compatibility

**Question:** Can old-producer/new-consumer and new-producer/old-consumer
coexist during rollout?
**Apply to:** API changes, event schemas, DB migrations, config
changes, queue payloads, feature flags.
**Good finding:** "The consumer now requires `foo`, but messages
already queued don't have it — they'll throw on drain. Make `foo`
optional during rollout, or add a compatibility path."

## 13. Observability for recovery

**Question:** If this fails in production, is there enough non-sensitive
context to debug and recover?
**Apply to:** new risky flows, background jobs, external calls,
migrations, fallback paths.
**Good finding:** "This catch logs only `'failed'`. Include the job ID
and the state marker (no secrets) so a failure can be diagnosed and
replayed."
**Don't report:** extra logging on obvious local validation errors.

## 14. Data volume and query shape

**Question:** Does this work at production data size?
**Apply to:** unbounded list operations, in-memory filtering, N+1
queries, missing pagination, missing indexes, joins, large JSON, file
scans.
**Good finding:** "This loads all history then filters in memory —
fine in tests, O(table) in prod. Push the predicate and pagination into
the query."

## 15. Removed behavior

**Question:** For every line the diff *deletes or replaces*, what
invariant or behavior did it enforce — and where is that re-established
in the new code?
**Apply to:** deleted guards, deleted branches, replaced functions,
"simplified" conditionals, moved/extracted code (extraction often drops
an anchor, a lock, or an early return on the way).
**Prove it by:** naming the old line's job, then searching the new code
for who does that job now; `git log -L` / `git blame` the deleted line —
if it was added to fix a bug, the commit says which bug just came back.
**Good finding:** "The old handler returned early when `items` was
empty (`handler.ts:31` pre-diff, added in a fix for #482); the rewrite
drops that check, so the batch call now fires with an empty payload —
the exact case #482 fixed. Restore the early return or guard in
`buildBatch`."
**Don't report:** deletions whose job you can show is now done
elsewhere — cite where.

## 16. Caller contract

**Question:** Does every call site still hold under this symbol's new
contract?
**Apply to:** changed signatures, return shapes, nullability, thrown
errors, sync→async changes, changed ordering/timing guarantees, renamed
or removed exports.
**Prove it by:** grepping for the symbol and reading each call site —
including dynamic wiring (registries, naming conventions, DI). The
diff won't show you the caller it broke; you have to go look.
**Good finding:** "`parseConfig` now returns `Config | null` instead of
throwing, but `boot.ts:18` still calls it bare and dereferences
`.port` — boot now crashes with a null deref instead of the old clear
error."
**Don't report:** call sites the diff already updated, or contracts
enforced by the compiler in a checked build.

## 17. Wrapper and proxy delegation

**Question:** Does this wrapper route everything to the wrapped thing —
and nothing back through itself?
**Apply to:** caches, decorators, adapters, proxies, middlewares,
client wrappers.
**Prove it by:** listing the methods callers actually use and checking
each is forwarded; checking internal calls go to the wrapped instance,
not back through a registry/global that re-enters the wrapper
(recursion, double-caching).
**Good finding:** "`CachedClient.fetchAll` calls `client.fetch` via the
registry, which resolves back to `CachedClient` — every batch item
re-enters the cache and takes the lock again. Call the wrapped
instance's method directly."
**Don't report:** intentionally-narrowed facades with a documented
surface.

---

## Adding new modes
When a real review surfaces something no mode covers, add a mode: the
*question* it asks, what to apply it to, how to prove it, a good
finding, and what not to report. Resist encoding the specific bug as a
pattern — what generalizes is the question. If it only matters for one
codebase, it's a war story, not a mode.

## Reference incidents (war stories — context, not modes)
- **PR #1061** — depth-pass haul: mode 1 (parseIntEnv vs docstring),
  mode 2 (count + history two-query waste), mode 3 (vestigial
  `pLimit(1)`), mode 4 (lying `attempt 1/2` label), mode 5 (sticky
  cap-state on a fallback throw).
- **PR #1049** — a "make it a step" architectural review; the design
  coupled a queue field to a single use case — mode 3 would have caught
  it earlier.
