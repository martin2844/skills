# Fix Mode

`--fix` applies findings to the working tree after the report is
written. It runs only when the flag is explicit — never inferred.

## Rules

1. **Findings come first.** Always emit the full report before
   touching any file. The user sees what's about to change.
2. **No commits, no pushes.** Fix mode edits the working tree. It
   does not stage, commit, amend, or push. The user owns the
   commit.
3. **Per-batch verification.** After each batch of fixes, the bar
   is:
   - `tsc --noEmit` passes (or whatever the repo uses;
     `npm run build` / `npx tsc` / `pnpm tsc`).
   - `npm run lint` (or `eslint`) passes — at minimum no *new*
     errors vs the pre-fix baseline.
   - `npm run build` passes.
   If any step fails, **stop**, report what failed, and revert that
   batch's edits. Do not press on.
4. **Auth-semantic fixes get a re-read pass.** After applying an
   auth-related fix (adding `auth()`, adding ownership filter,
   replacing manual cookie parsing), re-read the entire function
   before moving on. The bar above doesn't catch logic errors.
5. **No browser testing.** Fix mode does not run the dev server or
   click through the UI. If a finding requires runtime verification
   (e.g. "stale data after mutation"), report it but don't auto-fix.

## What `--fix` *will* attempt

Verifiable, well-scoped fixes:

- **Boundary fixes**: add `import "server-only"` to a server file;
  move a server import out of a `"use client"` file by extracting
  to a server action.
- **Auth fixes**: add `await auth()` + user lookup at the top of an
  action that was missing it; add `where({ user_id: session.user.id })`
  to a query missing ownership filter; replace silent-disable cron
  check (`if (envSecret && ...)`) with fail-closed.
- **Caching fixes**: add `revalidatePath(...)` after a mutation
  that's missing it.
- **Convention fixes** (mechanical): replace `crypto.randomUUID()`
  with `nanoid()`; replace direct `clsx` imports with `cn()`;
  switch `'../../../foo'` to `'@/foo'`; replace direct `metadata`
  object with `buildPageMetadata(...)` call (only when the
  builder's API is unambiguous).
- **Effect fixes** (the mechanical subset): replace
  `useState('') + useEffect(setX)` with `useState(initial)`;
  convert derived-state effects to plain consts or `useMemo`.

## What `--fix` *will not* attempt

- **Structural rewrites.** Moving a 78-line `page.tsx` into a
  proper `components/pages/X/X.tsx` requires judgment about which
  pieces are sections, what props they take, etc. Report it; the
  user does the move.
- **Webhook idempotency.** The right design depends on the table
  schema and provider semantics. Report; user decides.
- **Race-condition fixes.** Atomic updates / locking strategy vary
  by database. Report; user decides.
- **Anything where the right fix has two reasonable answers** and
  no clear winner. Report both options as part of the finding;
  don't pick one silently.
- **Anything in test files.** Fix mode doesn't touch tests; if a
  fix breaks tests, the user updates the tests as part of accepting
  the fix.

## Batching strategy

1. Group fixes by category and by file. One file open per batch
   minimizes churn.
2. Apply all mechanical fixes (convention-level) first — they're
   safest and validate the toolchain works.
3. Then caching fixes (low semantic risk).
4. Then boundary fixes (medium risk — verify imports still
   resolve).
5. Then auth fixes (highest semantic risk — re-read after each).
6. After each batch: verification bar.

## Reporting after `--fix`

After fixes (or a failed fix attempt), append to the report:

```
## Fixes applied
- file:line — finding title (status: applied / skipped — reason)

## Verification
- tsc: pass
- lint: pass (0 new errors)
- build: pass

## Skipped (require human judgment)
- file:line — finding title — reason
```

If a verification step failed and a batch was reverted, say so
prominently:

```
## ⚠️  Batch reverted
- The auth batch (3 fixes in `app/actions/dms.ts`) failed tsc:
  <error excerpt>. All three edits reverted. Original findings
  still apply; see the report above.
```

## Read-only context, write only on apply

Gathering evidence (Read, Grep, git diff) is always read-only. The
moment fix mode applies an edit, it's making a change the user has
to accept or undo. Treat each Edit as a small, reviewable hunk —
don't refactor surrounding code "while you're there."

## When to bail entirely

- The repo has uncommitted changes the user might lose.
  → Report findings, but don't auto-apply. Say so: "uncommitted
  changes detected; re-run --fix after committing or stashing."
- The verification commands aren't defined.
  → Report findings; offer to apply with a `--no-verify` opt-in,
  but warn that the bar is unmet.
- The build was already failing before any fix.
  → Report findings; don't apply (your fixes would be impossible
  to attribute against a broken baseline).
