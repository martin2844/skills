# Rules — Caching, Revalidation, and Data Correctness

Caching is the silent-failure mode of Next.js apps: nothing crashes,
the user just sees yesterday's data. The audit looks for the gap
between "mutation happened" and "the page that shows the data knows
about it."

Version note: Next.js 14 cached `fetch()` by default; 15+ does not.
The audit adapts based on detected version (`references/discovery.md`).

---

## C1. Mutation without revalidation

**Question:** After a server action mutates data that's read by a
cached server component, does the action call `revalidatePath` or
`revalidateTag` for the affected route?

**Apply to:** every server action that performs an `insert` /
`update` / `delete`.

**Prove it by:** find the mutation. Find the route(s) that read the
mutated data. Check whether the action calls
`revalidatePath('/that-route')` or `revalidateTag('that-tag')`
after the mutation succeeds. Check the route's segment config:
`export const dynamic = 'force-dynamic'` or `export const revalidate
= 0` makes revalidation unnecessary. `export const revalidate = N`
makes the data go stale for up to N seconds without explicit
revalidation.

**Good finding:** "`app/actions/dms.ts:88` inserts a new vehicle
into `dms_vehicles` but never calls `revalidatePath`. The list page
at `app/dms/vehicles/page.tsx` has no `dynamic` / `revalidate`
config and uses default caching. After creation, the user navigates
back to the list and sees the old data until a hard reload. Add
`revalidatePath('/dms/vehicles')` after the insert."

**Don't report:** mutations to data only read by `force-dynamic`
pages, mutations where the next render happens via
`router.refresh()` triggered by the client (verify it does), or
mutations to data only ever read by the same action's caller (no
shared cache).

---

## C2. Revalidating the wrong path

**Question:** Does the `revalidatePath` argument actually name the
route that displays the mutated data?

**Apply to:** every `revalidatePath` call.

**Prove it by:** read the path. Dynamic routes need
`revalidatePath('/things/[id]', 'page')` — passing
`revalidatePath('/things/abc')` only revalidates that exact URL,
not other instances of the segment. Layouts vs pages: passing
`'layout'` revalidates the layout (and its descendants); `'page'`
revalidates only that specific page.

**Good finding:** "`app/actions/report.ts:54` calls
`revalidatePath('/reporte/' + purchaseId)` after marking the report
ready. The dynamic segment is `/reporte/[id]` — this only
revalidates that single URL string. Other open tabs to other
reports may have stale data via other code paths. If the only data
that changes is the one report, this is fine; but the typical fix
is `revalidatePath('/reporte/[id]', 'page')` to invalidate the
segment's cache."

**Don't report:** correctly-targeted revalidations. Don't quibble
over `'page'` vs `'layout'` granularity if the simpler form works.

---

## C3. Stale `fetch()` cache after mutation

**Question:** Does a server component `fetch()` a backend that the
action mutates, without tags or with the wrong cache config?

**Apply to:** `fetch(url, options)` calls in server components,
especially against `BACKEND_URL` or similar.

**Prove it by:** check the version. In Next.js 14, `fetch()` is
cached forever by default — mutations don't invalidate it without
`revalidateTag`. In 15+, fetch is uncached by default; cached
fetches must opt in with `cache: 'force-cache'` or
`next.revalidate`. Look for `next: { tags: [...] }` on cached
fetches and a matching `revalidateTag(...)` in mutations.

**Good finding:** "`app/dms/dashboard/page.tsx:14` does
`fetch(BACKEND_URL + '/leads', { next: { revalidate: 3600 } })`
without tags. `app/actions/dms.ts:120` creates a new lead but
can't invalidate this fetch — there's no tag to revalidate. Tag
the fetch (`tags: ['dms-leads']`) and call
`revalidateTag('dms-leads')` in the mutation."

**Don't report:** Next 15+ default (uncached) fetches when the
desired behavior is fresh-every-render. Tag-less fetches whose
data is never mutated by this app (truly read-only from a remote
source).

---

## C4. Multi-write mutation without a transaction

**Question:** Does this action write to multiple tables (or do
multiple writes to one table) without wrapping them in
`db.transaction(...)`?

**Apply to:** actions with two or more `db('...').insert/update/delete`
calls in sequence, or one DB call followed by an external side
effect (mail, charge, webhook out).

**Prove it by:** trace the sequence. If write A succeeds and write
B fails, what state is the system left in? For DB-only multi-writes,
a transaction is the answer. For DB + external side effect,
transactions don't help — the order matters and an outbox/idempotency
key may be needed.

**Good finding:** "`app/actions/report.ts:144` inserts into
`report_purchases`, then `transactions`, then `webhook_events` — no
transaction wrapper. A constraint failure on the second insert
leaves an orphan purchase with no transaction record. The next
poll-status call reads the purchase, finds no transaction, and
loops forever. Wrap in `db.transaction(async (trx) => { ... })`."

**Don't report:** single-write actions. Multi-writes whose
intermediate states are intentional (event sourcing, outbox
pattern) — confirm the design before flagging.

---

## C5. Check-then-act race

**Question:** Does this action read a row, branch on it, then write
based on the read — without a lock or atomic update?

**Apply to:** actions that look like:
`if ((await db('x').where(...).first()).status === 'available') { await db('x').update({ status: 'taken' }) }`

**Prove it by:** trace the sequence. Two concurrent invocations
both read 'available', both write 'taken'. Look for
`UPDATE … WHERE status = 'available'` (atomic), `SELECT … FOR
UPDATE` (SQLite doesn't have this, but Postgres does), or
unique-constraint-driven safety.

**Good finding:** "`app/actions/plate.ts:88` does
`const job = await db('jobs').where({ plate }).first()` then
`if (!job) await db('jobs').insert({ plate, status: 'pending' })`.
Two requests for the same plate both see no row and both insert.
The `jobs` table has no unique index on `plate`
(`migrations/20260301_jobs.js:8`). Add a unique index and use
`INSERT ... ON CONFLICT DO NOTHING`, or do an
`UPDATE … RETURNING` claim."

**Don't report:** check-then-acts protected by a unique constraint
on the relevant column (the second insert fails loudly — that's
the lock).

---

## C6. Cache key derived from unstable input

**Question:** Does a cached fetch or `cache()`'d function use
something order-sensitive, time-sensitive, or non-canonical in its
key?

**Apply to:** `unstable_cache` calls, `cache()` wrappers, fetch
tags built from input.

**Prove it by:** look at the tag/key. Built from a JSON-stringified
object (key order matters)? Includes `Date.now()`? Includes the
session (every user gets their own cache slot, which can be
intentional or accidental)?

**Good finding:** "`lib/cached.ts:14` does
`unstable_cache(fn, [JSON.stringify(input)], { tags: ['x'] })`.
JSON key order isn't guaranteed across clients —
`{a:1,b:2}` and `{b:2,a:1}` produce different cache entries. Use a
canonical serializer or list the input fields explicitly."

**Don't report:** primitive-keyed caches, tag-only caches without
key expansion.

---

## C7. `force-dynamic` everywhere (cargo-cult opt-out)

**Question:** Is the route opted out of caching for no clear
reason?

**Apply to:** `export const dynamic = 'force-dynamic'` declarations.

**Prove it by:** check what the page renders. Pure-static content?
Public read-only data? Then `force-dynamic` makes every request
re-render unnecessarily. Look for the reason in a comment or in
the data fetches — if the page uses `auth()`, `cookies()`,
`headers()`, or `searchParams`, Next will mark it dynamic anyway,
making the directive redundant.

**Good finding:** "`app/multas/[jurisdiccion]/page.tsx:1` exports
`dynamic = 'force-dynamic'` but renders static fines info from
local JSON. The opt-out drops the static cache and re-renders on
every request, eating CPU and slowing TTFB. Remove the directive;
the page becomes a static segment and renders from the build."

**Don't report:** pages that genuinely need to be dynamic (use
session, cookies, real-time data) — the directive is fine even
when redundant. This is a Nit.

---

## C8. `cookies()` / `headers()` accidentally turning a page dynamic

**Question:** Does a server component read `cookies()` or
`headers()` in a way that opts the entire page out of static
rendering, when only a small part needed it?

**Apply to:** `cookies()` / `headers()` / `draftMode()` reads in
server components or shared helpers.

**Prove it by:** find the call. Is it in a leaf component that
only renders for some users? Could it be moved into a client
component (where it doesn't affect server caching) or scoped to a
route that doesn't need to be static?

**Good finding:** "`components/layout/Header.tsx:14` (a server
component used in the root layout) calls `cookies()` to read a
theme preference. This forces the *entire app* to be dynamic.
Move theme reading into a client component, or read the cookie in
the root layout and pass it down as a prop only to the parts that
need it."

**Don't report:** intentional cookie/header reads in pages already
dynamic for other reasons.

---

## C9. Background work in a request path

**Question:** Does this action/route fire-and-forget a Promise
without awaiting it, expecting it to complete?

**Apply to:** `void doWork()`, `doWork().catch(...)` patterns
without `await`.

**Prove it by:** trace the work. Serverless runtimes (Vercel,
Lambda) freeze the function after the response is sent — un-awaited
work is killed mid-flight. On a long-running Node server it may
work; on serverless it doesn't.

**Good finding:** "`app/actions/email.ts:42` does
`void sendFollowUp(user.email)` after returning the success
response. On Vercel deployments, the function is frozen as soon
as the action returns; the email never sends. Use `after()` (Next
15+) or move to a queue."

**Don't report:** fire-and-forget on a known-non-serverless
deployment (Docker `output: 'standalone'` on a long-lived host),
when verified.

---

## Cross-cutting

- Caching findings are typically Should-fix unless data correctness
  is at stake (then Blocker) or pure performance (then Nit).
- Pair caching findings with the segment config: a missing
  `revalidatePath` is only a finding if the route actually caches.
- Knowing the Next.js version is mandatory before judging cache
  findings — see `references/discovery.md`.
