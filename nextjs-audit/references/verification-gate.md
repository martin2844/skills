# Verification Gate

No candidate finding reaches the report without passing this gate.
False positives are the failure mode that kills audit tools: every
hallucinated bug teaches the author to skim the next ten real ones.
The gate exists to make *disproof cheap and emission expensive*.

This is adapted from big-review's gate, tightened for the Next.js
domain. The asymmetries that bite this audit specifically:

- **Guards live in layouts, middleware, and HOFs.** A server action
  with no `auth()` call may still be safe because the route segment's
  layout enforces it, or the action is only called from a server
  component that already checked. *Look up.*
- **Server-only is enforced by bundler + types + runtime.** A
  `"use client"` file importing `db` is a real bug — even if it
  type-checks — when the `package.json` `"browser"` field aliases the
  module to `false`. *Check the config, not just the import.*
- **Caching is silent.** A missing `revalidatePath` doesn't crash; it
  shows stale data. The "trigger" is a real user navigating after a
  mutation. Name that path concretely.

## Step 1 — Reasoning before finding

For each candidate, write the causal chain (internally, never in the
output): **trigger → path → wrong outcome.** Concretely:

- **Trigger**: what user action, request shape, timing, or
  configuration kicks this off?
- **Path**: which lines execute, across which boundaries (client →
  action → service → db), and what gets skipped?
- **Wrong outcome**: what does the user see, what gets written, what
  data leaks, what stale state shows?

If you cannot name all three, you don't have a finding — you have a
vibe. Go inspect until you can name them, or drop it.

## Step 2 — Active disproof attempt

Try to kill your own finding before the author has to. The Next.js
disproof checklist:

- **Re-read the actual line.** Does the code really say what you
  think it says? Quote it.
- **Look up the segment tree.** Does a parent `layout.tsx` already
  enforce the auth/admin check? A `middleware.ts` matcher? Cleared.
- **Look for the guard at the data layer.** Does the service or
  repo function the action calls perform the check (`whereUserId`,
  `requireOwner`)? Cleared.
- **Check `package.json` `browser` field + `serverExternalPackages`.**
  Is the "client imports server module" finding actually blocked at
  bundle time? If so, downgrade to a Should-fix (still misleading,
  but won't ship a runtime crash).
- **Check `metadataBase` and `generateMetadata`.** A missing `path`
  in `buildPageMetadata` may already be supplied by a layout default.
- **Check `revalidate` on the route segment.** A mutation with no
  `revalidatePath` may still be correct if the route is
  `force-dynamic` or `revalidate = 0`.
- **Check if the env var is *server-only*.** Don't flag
  `process.env.FOO` in a server file as "client-leaked" — only flag
  it in `"use client"` files or in props/serialized data crossing the
  RSC boundary.
- **Check it's not pre-existing.** In diff mode, `git blame` the
  flagged line. If it predates the diff and the diff didn't touch or
  worsen it, drop it (at most one out-of-scope note if genuinely
  serious).
- **Check the convention.** Before flagging "doesn't match the
  guide", confirm the guide actually says it (quote the section), and
  confirm sibling files conform (so the convention is real, not
  aspirational doc).

## Step 3 — Verdict

- **Confirmed** — trigger / path / wrong outcome named, evidence
  quoted, guard ruled out. Emit it.
- **Plausible** — mechanism is real, trigger depends on runtime
  conditions you can't see (provider retry, race, env config, prod
  data shape). Realistic rare paths *count* — webhook retries, cold
  caches, missing optional fields, prerendered pages reading stale
  caches.
- **Refuted** — guard exists, code doesn't say that, framework
  handles it, convention isn't actually a convention. Drop it
  silently.

### Asymmetric confidence

- **High impact + Plausible** → emit, with the uncertainty explicit
  and a cheap way to settle ("if the provider retries this webhook,
  the second call writes again — an idempotency test would settle
  it"). A missed data-loss bug costs more than one hedged comment.
- **Low impact + Plausible** → drop. Nobody needs a hedged nit.

## Step 4 — Known false-positive precedents (Next.js-specific)

Don't emit findings that match these patterns:

- **TypeScript already catches it.** With `strict: true`, you don't
  need to flag null-deref in typed code, unused vars, or implicit
  `any`. The repo's own ESLint config covers `exhaustive-deps`,
  `no-explicit-any`, `no-unused-vars`.
- **Framework already escapes it.** React escapes interpolated
  children — XSS needs `dangerouslySetInnerHTML` or `next/script`
  with inline `dangerouslySetInnerHTML` to be a real finding.
- **ORM already parameterizes.** Knex/Prisma/Drizzle bound values
  aren't injection vectors; raw query strings are.
- **`"use server"` files are server-only by design.** Don't flag
  `process.env.SECRET` in an action as a leak — actions don't ship
  to the client. (The leak is when the action *returns* the value to
  the caller, or sets it on a prop.)
- **Server components are not client components.** A server component
  without `"use client"` cannot use `useState`/`useEffect` — but
  flagging this is the compiler's job, not yours.
- **`auth()` may not be where you expect.** If a layout calls
  `auth()` and redirects, every page underneath is gated. Walk up
  the segment tree before flagging "action missing auth".
- **Pre-existing issues** on untouched lines in diff mode.
- **Intentional behavior** stated in the PR description or commit
  message.
- **The guide says so.** A guide-sanctioned pattern that's only a
  convention concern (style, naming) is not a finding. Emit only
  when the pattern is a real risk (then per SKILL.md, emit + note
  the guide conflict).
- **Pedantic nits a senior wouldn't say out loud.** Would this
  comment survive being the only comment on a PR? No → drop.

## Step 5 — Admission test

Before emitting any surviving finding:

1. Can I point to a **specific file:line**?
2. Can I describe a **concrete failure or maintenance cost** — not a
   vague worry?
3. Can I cite **evidence** (quoted code, config, type, call site,
   guide section)?
4. Can I propose a **concrete fix** the author could apply directly?
5. Would I still emit this finding **if there were already five
   other findings in this category**?

Mostly no → suppress. Borderline Nits failing (5) are exactly the
noise to cut.

## Step 6 — Budget

- Diff mode: at most **~10 findings**, ranked most-severe first; if
  the gate passes more, cut from the bottom — Blockers always outrank
  Nits when the cap forces a cut.
- Full mode: per-category cap of ~8; global cap ~30. Merge
  same-root-cause findings into one (one missing-auth pattern across
  six actions is one finding with six locations, not six).
- The win condition is the author **acting on every finding**, not
  the finding count. "No findings" is a successful audit.

## Anti-sycophancy

You were asked to find issues; that is not evidence issues exist.
Under "audit this" pressure, models invent concerns on clean code.
If the app is well-built, the verdict is short and the audit ends.
Never pad, never manufacture a Nit to have something to say, never
convert "I didn't fully understand this" into a finding — go
understand it.
