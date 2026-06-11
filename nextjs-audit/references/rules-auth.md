# Rules — Auth, Authorization, and Trust Boundaries

Server actions and API routes are public HTTP endpoints. The fact
that a UI component is the only known caller does not protect them
— anyone can craft a POST. The audit treats every action and route
as if an attacker were calling it directly.

---

## A1. Server action missing authentication check

**Question:** Does this `"use server"` export touch user-specific
data without an `auth()` (or equivalent session) check?

**Apply to:** every `export async function` in any `"use server"`
file.

**Prove it by:** read the function top-to-bottom. Look for
`await auth()`, `requireUser()`, `requireAdmin()`, or any helper
that calls them. If absent, check whether the action only operates
on globally-public data (e.g. listing all published guides). If the
action reads or mutates user-scoped rows, the check must exist.
Walk up: is the action only ever called from a server component
inside a gated layout? Layout gating doesn't protect the endpoint
— the action is callable directly by anyone. **Layout gating is
not endpoint protection.**

**Good finding:** "`app/actions/dms.ts:42` exports
`updateAgency({ id, name })` which writes
`db('dms_agencies').where({ id }).update({ name })` with no auth
or ownership check. A logged-out attacker can rename any agency by
calling the action directly with its ID. Add
`const session = await auth();` and a `where({ owner_user_id:
session.user.id })` clause."

**Don't report:** actions that read public data, actions where the
first line is `await requireAdmin()`, actions whose only mutation
is on a session-scoped resource and the resource resolves via the
session (e.g. `await auth()` is the only argument source).

---

## A2. Server action missing authorization (IDOR)

**Question:** When the action *does* check authentication, does it
also verify the user owns the resource being read or modified?

**Apply to:** authenticated actions that read or write rows keyed
by a user-supplied ID.

**Prove it by:** find the query. Does it filter by the resource ID
*and* the user ID, or only by the resource ID? `WHERE id = ?` with
no user scoping is IDOR. Look for the join (`WHERE id = ? AND
user_id = ?`), an explicit ownership lookup before the mutation, or
a service helper that does the check.

**Good finding:** "`app/actions/report.ts:88` does
`db('report_purchases').where({ id: input.id }).update({ status:
'paid' })` after the auth check. The auth check is on the *user*,
not on whether this user owns this purchase. User A can mark user
B's purchase paid. Add
`.where({ id: input.id, user_id: session.user.id })`."

**Don't report:** mutations where the row was just inserted by this
action (the ID is server-generated, attacker can't supply it).
Mutations gated by `requireAdmin()` — admins are authorized for
everything by definition.

---

## A3. API route missing authentication or auth-by-querystring

**Question:** Is the API route gated by a real auth mechanism
(session, signed token, HMAC) or by something easily bypassed
(querystring secret, header presence)?

**Apply to:** every `route.ts` `GET`/`POST`/`PUT`/`DELETE` handler.

**Prove it by:** read the first 10 lines of the handler. Look for
`auth()`, `requireAdmin()`, HMAC verification
(`verifyHmacSignature` or similar), or webhook signature
verification. **Specifically check for the silent-disable pattern:
`if (envSecret && provided !== envSecret) return 401`** — if
`envSecret` is unset in some environment, the gate disappears
entirely. The check must fail closed: `if (!envSecret || provided
!== envSecret) return 401`.

**Good finding:** "`app/api/cron/check-report-jobs/route.ts:14`:
```
const cronSecret = process.env.CRON_SECRET;
if (cronSecret && secret !== cronSecret) return 401;
```
If `CRON_SECRET` is unset in any environment (staging, preview,
local), the route is open to the internet and triggers the cron
handler from any source. Fail closed:
`if (!cronSecret || secret !== cronSecret) return 401`. [Guide
conflict: `NEXTJS_GUIDE.md §6` documents this pattern as the cron
auth pattern — the convention should change.]"

**Don't report:** routes that genuinely need to be public (health,
NextAuth handler, public plate lookup). Routes whose auth check is
inside a wrapper helper called on line 1 — read the helper.

---

## A4. Webhook missing signature verification

**Question:** Does the webhook handler verify a provider signature
before trusting the body?

**Apply to:** every route that takes external POSTs from a third
party (payments, calendar, mail, etc.).

**Prove it by:** look for HMAC, JWT, or provider-specific signature
verification on the raw body *before* parsing/processing. Order
matters: `await req.json()` consumes the stream; some verifiers
need the raw body, so parsing first breaks them.

**Good finding:** "`app/api/webhook/mp/route.ts:8` calls
`await req.json()` before `verifyWebhookSignature(body, headers)`.
Mercado Pago's HMAC verifier needs the raw request body string; by
parsing first, the verifier silently fails open (or always
mismatches and the route 401s every legitimate webhook, depending
on the impl). Read `await req.text()` once, verify, then `JSON.parse`."

**Don't report:** webhooks behind a network-level filter
(documented in deployment) — but only if you can verify the filter
exists (e.g. an explicit IP allowlist in `middleware.ts`). Trust
nothing without evidence.

---

## A5. Webhook missing idempotency

**Question:** If the webhook fires twice for the same event, does
the handler write twice / double-charge / double-mail?

**Apply to:** every webhook route, especially payment and order
events.

**Prove it by:** look for an idempotency check before the mutation
— `webhooks` table query, `provider_event_id` unique index, an
`UPDATE … WHERE status = 'pending'` (state-machine guard), or
`INSERT ... ON CONFLICT DO NOTHING`. Providers retry; the audit
assumes they will.

**Good finding:** "`app/api/webhook/mp/route.ts:32` records the
payment row, then calls `sendReceipt(user.email)`, then writes the
`processed` marker. A provider retry (or a crash between
`sendReceipt` and the marker) sends the receipt twice. Move the
marker into the same transaction as the payment insert with a
unique key on `provider_event_id`."

**Don't report:** handlers that are read-only or have no
externally-visible side effect on a second call.

---

## A6. Cron route triggers expensive work without auth

**Question:** Is the cron handler protected, *and* does the
protection survive misconfiguration?

**Apply to:** every `app/api/cron/*` route.

**Prove it by:** see A3 (fail-closed check). Also check what the
handler does: any unauthenticated trigger that walks a queue,
sends emails, or hits a paid third-party API is a denial-of-wallet
vector.

**Good finding:** see A3 finding. The fail-open pattern is the
recurring issue here.

---

## A7. Middleware claimed but missing for a route

**Question:** Does `middleware.ts` actually cover the route that
the guide / common sense says it should?

**Apply to:** repos that have a `middleware.ts`. Read the `matcher`
config.

**Prove it by:** look at the matcher. Does it cover `/account/*`,
`/dms/*`, `/admin/*`? A missing matcher means the middleware never
runs for those paths. Cross-check with what each segment's layout
assumes.

**Good finding:** "`middleware.ts:42` matchers cover
`/dms/(dashboard|leads|settings)/:path*` but not
`/dms/agency/:path*`, and `app/dms/agency/[id]/page.tsx:8` assumes
the middleware redirected anonymous users to `/login` (no `auth()`
call). Anonymous users hit a server crash when `session` is
undefined. Add `/dms/agency/:path*` to the matcher, or move the
auth check into the page."

**Don't report:** intentional public segments under a gated parent
(verify by reading the layout).

---

## A8. Admin check by string equality on email

**Question:** Is admin identity decided by hardcoded email
comparison, and if so, is the email source trusted?

**Apply to:** `requireAdmin()` / `isAdmin()` helpers and inline
admin checks.

**Prove it by:** read the helper. `session?.user?.email === ADMIN_EMAIL`
is fine if `session` comes from NextAuth and the JWT/cookie can't
be forged. It is *not* fine if `email` is read from a header,
`searchParams`, body, or cookie that the user can set. (NextAuth
session cookies are signed and not user-settable.)

**Good finding:** "`lib/admin.ts:14` exports
`isAdmin(req) { return req.headers.get('x-user-email') === ADMIN_EMAIL }`,
and `app/api/admin/not-found-plates/route.ts:7` calls
`isAdmin(req)`. Any caller can set `x-user-email`. Use `auth()` to
get the session-bound email."

**Don't report:** session-derived email checks. Hardcoded admin
emails are a Nit (separate concern about deploy-time admin
management) — flag only if the guide says to use a different
mechanism.

---

## A9. Trust-boundary inputs unvalidated

**Question:** Do action inputs / route bodies / URL params get
schema-validated before use?

**Apply to:** every server action argument, every `await req.json()`
in a route, every read of `params` / `searchParams`.

**Prove it by:** check for Zod, Valibot, or a manual narrowing
guard. Look at how the value is used: as a DB key, in a query
predicate, in a redirect target, as a filesystem path, as an
outbound URL.

**Good finding:** "`app/actions/dms.ts:122` accepts
`{ slug: string }` and calls
`db('dms_agencies').where({ slug }).first()`. No validation: an
attacker passes `{ slug: { $gt: '' } }` (or an injected SQL
fragment via a string operator). Validate with
`z.object({ slug: z.string().regex(/^[a-z0-9-]+$/) })` before
querying."

**Don't report:** inputs already typed as primitives passed
through Knex/Prisma parameterized queries (knex parameterizes the
value but does NOT prevent NoSQL-style object injection if the
input could be an object — TypeScript's `string` annotation isn't
a runtime guarantee).

---

## A10. Error messages leaking sensitive context

**Question:** Does an error path return more information than the
caller should see?

**Apply to:** error responses, returned `error` strings in action
result types, logged-and-thrown patterns.

**Prove it by:** look at error responses. Stack traces shipped to
the client? Internal DB error messages exposed? Username
enumeration via different errors for "user not found" vs "wrong
password"?

**Good finding:** "`app/api/auth/register/route.ts:38` returns
`return NextResponse.json({ error: err.message })` for any thrown
error, including the underlying SQLite
`UNIQUE constraint failed: users.email` — this both leaks the DB
schema and enables user enumeration. Catch the unique violation
explicitly and return a generic message; log the details server-side."

**Don't report:** generic error returns that don't leak structure
or enable enumeration.

---

## Cross-cutting

- Auth findings are almost always Blockers or Should-fix; Nits are
  rare here.
- When a guide documents a risky auth pattern (A3 silent-disable,
  A8 header-trust), emit the finding at true severity AND name the
  guide conflict — the convention should change.
- Verify the guard isn't somewhere else (layout, middleware, data
  layer) before emitting. False positives on auth findings destroy
  the audit's credibility faster than anything else.
