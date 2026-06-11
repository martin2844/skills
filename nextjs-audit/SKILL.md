---
name: nextjs-audit
description: Evidence-first Next.js (App Router, 14+) audit for any repo. Use for "nextjs audit", "audit this app", "framework audit", "RSC boundary check", "server action audit", "next audit", auditing server/client boundaries, auth on actions/routes, caching/revalidation correctness, useEffect anti-patterns, and convention conformance against the project's own guide.
when_to_use: Use when the user wants a Next.js-specific audit of a repo, branch, or PR — framework invariants, security boundaries, caching, effects, and convention drift. Prefer big-review for generic change-correctness review; prefer this when the question is "is this app built right as a Next.js app?"
argument-hint: "[--full | PR number | branch | path] [--fix] [--comment]"
---

# Next.js Audit

Audit like a Next.js domain expert, not a linter. The framework has
sharp edges that compile fine and pass tests: server actions are
public HTTP endpoints, RSC props serialize into the HTML, caching
defaults changed between majors, layouts don't re-run on soft
navigation. This skill hunts those — plus drift from the project's
own documented conventions.

## Inputs and modes

| Invocation | Behavior |
|---|---|
| `/nextjs-audit` | Diff mode: audit changed files (branch vs merge base, or uncommitted diff). Single-threaded. |
| `/nextjs-audit <PR#>` | Diff mode against a GitHub PR (fetch full files from head ref, never judge hunks alone). |
| `/nextjs-audit --full` | Whole-repo sweep. Parallel category subagents (`references/scaling.md`). |
| `--fix` | After the report, apply verified fixes (`references/fix-mode.md`). Never without the flag. |
| `--comment` | Post blocker/should-fix findings as inline GitHub PR comments (`references/delivery.md`). Never without the flag. |

## Workflow

1. **Discovery** (`references/discovery.md`). Detect Next.js version
   and router from `package.json` + tree. Pages Router → bail with a
   clear message. Find the project's convention guide
   (NEXTJS_GUIDE.md / AGENTS.md / CLAUDE.md / CONTRIBUTING.md /
   docs/) and extract auditable invariants. No guide → best-practices
   only + infer dominant codebase patterns.
2. **Scope.** Diff mode: list changed files (read-only git; three-dot
   diff vs merge base). Full mode: map `src/`/`app/` surface.
3. **Triage.** Skip files that can't carry a finding (formatting-only,
   generated, lockfiles). When in doubt, audit it.
4. **Category passes.** Each category has its own rule catalog —
   load only what the scope needs:
   - `references/rules-boundary.md` — server/client boundary, secret
     and data leakage
   - `references/rules-auth.md` — actions, routes, webhooks, crons,
     IDOR, middleware traps
   - `references/rules-caching.md` — revalidation, cache poisoning,
     transactions, races
   - `references/rules-effects.md` — the you-might-not-need-an-effect
     catalog
   - `references/rules-conventions.md` — conformance to the discovered
     guide
   In full mode, these run as parallel subagents (`references/scaling.md`).
5. **Verification gate** (`references/verification-gate.md`). Every
   candidate finding gets a disproof attempt — the guard may live in a
   layout, middleware, the data layer, or a sibling. Nothing emits
   without file:line evidence and a nameable failure scenario.
6. **Report** (`references/delivery.md`). Three tiers — Blocker /
   Should-fix / Nit. Risk is reported at true severity even when the
   project's guide blesses the pattern; the conflict is noted inline.
7. **`--fix`** if requested (`references/fix-mode.md`). Bar: tsc +
   lint + build green per batch; auth-semantic fixes get a re-read
   pass. Never commit or push.

## Core stance

- **Evidence first.** Every finding cites file:line and survives an
  active disproof attempt. A false positive in an audit poisons trust
  in the whole report.
- **The guide is input, not law.** Conventions from the project guide
  are enforced (mostly as nits/should-fix), but a genuinely risky
  pattern the guide endorses is still reported at true severity, with
  the conflict named — "the guide documents this; the convention
  itself should change."
- **Severity = real-world consequence.** Blocker: security, data
  loss, cross-user leakage, broken production behavior. Should-fix:
  correctness and caching bugs with bounded blast radius, auth
  hygiene. Nit: convention drift with no runtime consequence.
- **Fewer, stronger findings.** Merge same-root-cause findings. A
  clean category is reported as checked-clean, not padded.
- **Read-only by default.** Gathering context never mutates the repo.
  Fixes only via `--fix`; GitHub comments only via `--comment`.

## Reference loading map

- **Always**: `references/discovery.md`, `references/verification-gate.md`.
- Per category in scope: the matching `references/rules-*.md`.
- `references/scaling.md` — only for `--full`.
- `references/fix-mode.md` — only for `--fix`.
- `references/delivery.md` — before writing the report; the GitHub
  section only for `--comment` or PR targets.

## Output contract

```
## Verdict
One human line: overall shape of the audit. Worst finding is the
headline.

## Target
Mode (diff/full), Next.js version, router, guide found (path or
"none — best-practices mode").

## Findings

### Blockers
1. `file:line` — title
   Issue: trigger → path → wrong outcome
   Evidence: quoted code / config
   Fix: concrete change
   [Guide conflict: guide section X documents this pattern — the
   convention should change.]  ← only when applicable

### Should fix
...

### Nits (non-blocking)
...

## Checked clean
- category: what was checked and why it passed
```

Rules:
- No findings → "No findings." + Checked clean. Never pad.
- Never leak internal labels (Confirmed/Plausible, rule IDs) into the
  report.
- Don't duplicate what tsc/ESLint already enforces in this repo.
- Don't report pre-existing issues in diff mode unless the diff
  touched or worsened them (full mode reports everything in scope).
- Don't invent conventions — cite the guide section or the dominant
  codebase pattern you verified.

## Non-goals

- Not a generic code review (that's big-review) — this audits
  framework invariants and architecture, and goes deep on them.
- No style bible, no praise, no PR summary.
- Never mutate files, commit, push, or post comments without the
  explicit flag.
