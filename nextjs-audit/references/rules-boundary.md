# Rules — Server/Client Boundary

The RSC boundary is the most expensive thing to get wrong in a
Next.js app: secrets leak into HTML, server modules end up in the
client bundle, props get serialized that shouldn't be. Each rule
below is a *question* — apply it to the diff, report what doesn't
survive the verification gate.

---

## B1. Server module imported into a client component

**Question:** Does a `"use client"` file import a server-only module?

**Apply to:** files with `"use client"` at the top, or files reached
transitively from one. Server-only modules: `@/lib/db`, `@/auth`,
`@/lib/admin`, anything that imports `server-only`, anything in
`@/services/`, Node built-ins (`fs`, `crypto`, `net`), drivers
(`better-sqlite3`, `pg`, `ioredis`).

**Prove it by:** grep for `"use client"` in the file (or any file in
its import chain). If the file itself doesn't have it but is
imported by a client file, the import chain still poisons it. Check
`package.json` `"browser"` field — if the offending module is
aliased to `false`, bundling succeeds but the runtime call throws.
Check `serverExternalPackages` in `next.config` — externalized
modules still can't be called client-side.

**Good finding:** "`components/ReportViewer.tsx:3` starts with
`"use client"` and imports `@/lib/db` directly to call
`db('purchases').first()`. `package.json` aliases `better-sqlite3` to
`false` in the browser bundle, so the call throws
`TypeError: db is not a function` the first time a user lands on
`/reporte/[id]`. Move the read into a server action and pass the
result as a prop, or fetch it in the parent server component."

**Don't report:** server module imports inside files that are *also*
imported from a server file but never reach a client boundary (RSC
files default to server). Confirm a `"use client"` is somewhere up
the chain before flagging.

---

## B2. Secret env var read in a client component

**Question:** Does a `"use client"` file read `process.env.X` where
`X` doesn't start with `NEXT_PUBLIC_`?

**Apply to:** any `process.env.<NAME>` access in a client file or
file reachable from one.

**Prove it by:** grep `process.env\.[A-Z_]+` inside `"use client"`
files. Cross-reference with the `NEXT_PUBLIC_` allowlist — anything
not prefixed is server-only by Next.js convention and resolves to
`undefined` (or in some builds, gets inlined as the value at build
time — both outcomes are bugs).

**Good finding:** "`components/checkout/PayButton.tsx:14` reads
`process.env.MERCADOPAGO_ACCESS_TOKEN` inside a `"use client"`
component to attach as an Authorization header. Either it resolves
to `undefined` and every request 401s, or (worse) Next inlines the
secret into the client bundle. Move the request to a server action
and have the client call the action."

**Don't report:** `NEXT_PUBLIC_*` reads. Server-file reads of secret
env vars (that's the whole point of secret env vars).

---

## B3. Sensitive field serialized across the RSC boundary

**Question:** Does a server component or server action return an
object containing a server-only field (password hash, secret token,
PII not meant for the client) into a client component's props?

**Apply to:** server actions whose return type or runtime shape
includes sensitive fields, and server components that pass DB rows
directly as props to client components without projection.

**Prove it by:** read the action's return value or the server
component's props. Look for `select *`-style passthroughs:
`return user;` where `user` came straight from
`db('users').first()`. Check the `users` schema (or migration) for
fields like `password_hash`, `secret`, `*_token`, `api_key`,
`stripe_customer_id`. RSC serializes whatever you hand the client
component — even fields the JSX doesn't render still ship in the
HTML/RSC payload.

**Good finding:** "`app/account/page.tsx:18` fetches
`db('users').where({ id }).first()` and passes the row to
`<AccountClient user={user} />`. The `users` table includes
`password_hash` (`migrations/20260101_users.js:12`), so the hash
ships in the RSC payload — visible in DevTools. Project to a
view-model: `{ id, name, email, image }`."

**Don't report:** projections that already exclude sensitive fields,
or fields that are *intended* to be public (display name, avatar).
Don't flag every DB-row passthrough — flag the ones that carry
secrets.

---

## B4. `"use server"` file exporting non-action values

**Question:** Does a `"use server"` file export anything that isn't
an async function?

**Apply to:** every file with `"use server"` at the top.

**Prove it by:** scan exports. Constants, types (without `type` /
`interface` keyword which would be erased), classes, sync functions
— all of these violate the server actions contract. Next.js
silently treats every export as an action endpoint; a non-async
export becomes a publicly callable endpoint with confusing behavior.
Types and interfaces are fine — they're erased.

**Good finding:** "`app/actions/report.ts:42` exports
`export const MAX_RETRIES = 3` from a `"use server"` file. Every
client now has a publicly-callable RPC for `MAX_RETRIES`. Move the
constant to a non-action module (`@/lib/constants.ts`) and import
it from the action."

**Don't report:** `type` / `interface` exports (erased at compile
time). `'use server'` imports of constants from other modules.

---

## B5. Server-only module without `import "server-only"`

**Question:** Is a module that *must not* ship to the client
unprotected by `import "server-only"`?

**Apply to:** files in `@/lib/` and `@/services/` that handle
secrets, DB access, or admin checks — but that don't already have
the protection (e.g. via `"use server"` directive, or via being
imported only by server files in practice).

**Prove it by:** check for `import "server-only"` at the top.
Without it, a future refactor (or autocomplete) can import the file
into a client component and bundle it. With it, the build fails
loudly. Confirm the project actually uses `server-only` (grep for
existing imports of it) — if not, this is a Nit, not a Should-fix.

**Good finding:** "`lib/api-client.ts:1` builds the HMAC backend
client (reading `CLASIFICAR_ADMIN_SECRET`). It has no
`import "server-only"` guard, so a `"use client"` file importing
`apiClient` from it would bundle the secret into the browser. Add
`import "server-only";` at the top."

**Don't report:** server-only files protected by another mechanism
(`"use server"` directive, or files that contain only types). Don't
require `server-only` on every server file — only on ones where the
risk of accidental client import is real (anything with secrets,
DB, or admin).

---

## B6. Server-only data shape leaks via `searchParams` / `params`

**Question:** Are URL params (`params`, `searchParams`) treated as
trusted input?

**Apply to:** any read of `params`/`searchParams` in a server
component or action.

**Prove it by:** trace the value. Used as a DB key? Validated
first? Used in a redirect target? Used in `revalidateTag`?
`searchParams` is attacker-controlled — anyone can craft a URL.

**Good finding:** "`app/reporte/[id]/page.tsx:14` does
`db('purchases').where({ id: params.id }).first()` with no
ownership check — any logged-in user appending another user's
purchase ID gets their report. Add a `where({ user_id: session.user.id })`
in the query, or look up via a join."

**Don't report:** params validated by a schema (Zod, etc.) before
use. Plate-string parsing that runs through a normalizer.

---

## B7. `getServerSession`-style helper called from a leaf component

**Question:** Is `auth()` (or equivalent session getter) called
inside a deeply nested component instead of at the route segment?

**Apply to:** server components and layouts that call `auth()`.

**Prove it by:** count how many times `auth()` is called for one
request. Each call costs a re-fetch unless cached. Pattern: the
page calls `auth()` once and passes the session down as a prop, or
the layout calls it and the page reads via a server-side cache.

**Good finding:** "Both `app/account/page.tsx:9` and
`components/account/UserMenu.tsx:6` (a server component rendered
inside the page) call `auth()`. The session is resolved twice per
request. Either pass the session as a prop, or memoize the call in
a `lib/session.ts` wrapper using React's `cache()`."

**Don't report:** repeated calls inside the same request when the
underlying `auth()` is already memoized. Verify before flagging.
This is a Nit unless the page is high-traffic.

---

## B8. `dangerouslySetInnerHTML` with non-constant input

**Question:** Does a `dangerouslySetInnerHTML` payload come from
something that isn't a built-time constant or a sanitized library
output?

**Apply to:** every `dangerouslySetInnerHTML={{ __html: … }}` and
every `<Script>` with inline content.

**Prove it by:** trace the source of `__html`. Constant string,
JSON-LD generated by a typed builder, sanitized MDX/markdown
output → fine. DB content, user input, plain string interpolation
→ XSS.

**Good finding:** "`components/Guide/MdxRenderer.tsx:24` sets
`dangerouslySetInnerHTML={{ __html: guide.body }}` where `body`
came from the `guides` table (`services/guide.ts:18`). An admin can
inject `<script>` into a guide — and the project's seed data shows
admin content isn't audited. Render through the MDX pipeline (which
escapes by default), or sanitize with DOMPurify on the server."

**Don't report:** JSON-LD `<script type="application/ld+json">`
blocks built from typed objects via `JSON.stringify` — Next.js
escapes those correctly when the input is a typed object literal.

---

## B9. Image / fetch target controlled by user input (SSRF)

**Question:** Does the server side fetch a URL or load an image
from a path the user controls?

**Apply to:** `fetch(userInput)`, `<Image src={userInput} />`
where `userInput` traces back to params, body, or DB rows seeded
from user input.

**Prove it by:** trace the URL source. Check `next.config.mjs`
`images.remotePatterns` — if a user-supplied URL passes through
`<Image>`, the Next image optimizer fetches it server-side, which
is an SSRF vector unless the host is allowlisted by `remotePatterns`.

**Good finding:** "`app/api/og/route.ts:18` fetches
`req.nextUrl.searchParams.get('image')` and pipes it into the
response. Any URL — including `http://169.254.169.254/...` (AWS
metadata) — is fetched from the server. Restrict to an allowlist
of hosts."

**Don't report:** `<Image>` calls where `src` is constrained by
`remotePatterns` and the value came from a trusted server source.

---

## Cross-cutting

- Boundary findings should name the **import chain** when relevant
  ("client A imports server B"). Without that, the author can't
  verify quickly.
- For `--full`, treat boundary as the highest-value pass. Findings
  here are usually Blockers.
- Merge findings: one wrongly-tagged module imported from N client
  files is one finding, not N.
