# Discovery

Before any rule runs, learn the repo. The wrong assumption about
version, router, or convention source is how audits become noise.

## 1. Detect Next.js version and router

Read `package.json`. Find `next` in `dependencies`. Parse the major
version.

- `next` < 14 → bail: "nextjs-audit supports Next.js 14+ App Router.
  Detected vX.Y.Z."
- No `next` dependency → bail: "Not a Next.js project."

Detect router by looking for either of:
- `app/` or `src/app/` directory with `layout.tsx|js` → **App Router**.
- Only `pages/` or `src/pages/` → **Pages Router**. Bail:
  "nextjs-audit currently supports App Router only. Pages Router
  detected at `<path>`."
- Both present → App Router with a Pages Router legacy surface. Audit
  the App Router; note Pages files as out-of-scope in the report.

Version-conditional checks the rule catalogs reference:

| Version | Behavior to remember |
|---|---|
| 14 | `fetch()` cached by default. `cache: 'no-store'` opts out. |
| 15 | `params` / `searchParams` become `Promise<…>` in async pages; `fetch` no longer cached by default. |
| 16 | Same as 15; Turbopack is dev default; React Compiler optional. |

## 2. Find the convention guide

The audit honors the project's documented invariants. Search, in this
order, at the repo root:

1. `NEXTJS_GUIDE.md`
2. `AGENTS.md`
3. `CLAUDE.md`
4. `CONTRIBUTING.md`
5. `docs/` containing any of: `architecture.md`, `conventions.md`,
   `nextjs.md`, `next.md`, `engineering.md`

Read the first one found. Don't stitch multiple guides together —
pick the most specific. If none exist, note "no guide found —
best-practices-only mode" in the report's Target section.

## 3. Extract auditable invariants from the guide

Skim for *enforceable* rules — phrases like "must", "never", "every",
"always", "the golden rule", section headings that name a layer.
Convert each into a sentence the audit can check, with a pointer to
the guide section. Examples of shapes that translate cleanly:

| Guide phrasing | Auditable invariant |
|---|---|
| "Every `page.tsx` must be server-only and under N lines" | page files: no `"use client"`, no JSX past N lines, exactly one `<Component />` rendered |
| "All UI lives in `src/components/pages/`" | no JSX literals in `src/app/**/page.tsx` other than the page wrapper |
| "Server actions follow this auth pattern" | every `"use server"` export starts with the auth check (or calls a helper that does) |
| "Use `nanoid` for IDs" | flag `crypto.randomUUID()` / `uuid` imports introduced in mutations |
| "Use `cn()` for class merging" | flag direct `clsx` / `twMerge` imports in components |
| "After any migration, update `dbschema.md`" | migration files changed → `dbschema.md` should also be changed |

Rules:
- **Don't invent conventions.** If the guide doesn't say it, it isn't
  a guide rule — at best it's a best-practice (handled by the rule
  catalogs).
- **Quote the section** when reporting a violation, so the author can
  push back on the convention if needed.
- A guide rule that contradicts a security best practice still
  *enforces the convention*, but the audit ALSO emits the underlying
  best-practice finding at true severity (see SKILL.md "guide is
  input, not law").

## 4. Infer dominant patterns (used when no guide, or to verify guide claims)

For best-practices-only mode (no guide), or to confirm a guide rule
matches reality, derive conventions from the codebase itself:

- Sample 5–10 sibling files in the same role (e.g. `src/app/**/page.tsx`)
  and look at what 80%+ do — that's the de facto convention.
- Treat that as the baseline; a single new file that diverges is the
  finding, not the eighty that conform.
- Never elevate a *minority* pattern to "the convention" because it
  looks cleaner. Audit against what the codebase actually does.

## 5. Detect ambient tooling (informs which findings to suppress)

Read once and remember:

- `tsconfig.json` — `strict`, `noUncheckedIndexedAccess`, `paths`. If
  strict mode is on, **don't emit findings the compiler already
  enforces** (null/undefined access in typed code, unused vars).
- `eslint.config.{mjs,js}` / `.eslintrc*` — note enabled rules.
  Don't duplicate `react-hooks/exhaustive-deps`,
  `@typescript-eslint/no-explicit-any`, etc.
- `next.config.{mjs,js,ts}` — `serverExternalPackages`,
  `experimental.serverActions.bodySizeLimit`, `output`, custom
  `headers()` (informs caching rule analysis), `images.remotePatterns`
  (informs SSRF / image rules).
- `package.json` `"browser"` field — server-only modules excluded
  from client bundles (informs boundary rules: a `"browser": { "knex":
  false }` entry means importing knex in a `"use client"` file
  silently breaks at runtime, not build — still a finding).
- `middleware.ts` — exists? matches what paths? (auth rule depends
  on this).
- `src/instrumentation.ts` — exists? (informs observability + startup
  rules).

## 6. Scope output

After discovery, the audit prints a one-line Target summary:

```
Target: diff (12 files) | Next.js 16 App Router | guide: NEXTJS_GUIDE.md
```

or

```
Target: full sweep | Next.js 15 App Router | guide: none — best-practices mode
```

Then the category passes run.
