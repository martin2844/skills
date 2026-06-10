# Surface Checklist

Run mechanically after triage, only on files you're reviewing. This is
high-signal recurring-smell detection — **not** a style bible. If a
linter or formatter owns it, skip it.

1. **Consumer-naming comments.** Flag comments that name a consumer,
   ticket ID, route, or downstream queue ("used by X", "TS-1234 routes
   this") instead of explaining *why* the code exists. Those
   identifiers rot. Keep the invariant, drop the who — or delete the
   comment.

2. **Comments about non-events.** Delete comments that restate the
   code or explain what *didn't* happen. If removing it leaves no
   future reader confused, it's noise.

3. **Non-trivial inline types.** Hoist repeated or discriminated inline
   shapes to module scope and give them a name — especially a union
   that appears in more than one function.

4. **Boundary typing.** If callers need `as unknown as T[]` casts, the
   repo/client/library returned `any`. Push the type down to the
   boundary and drop the cast at every call site.

5. **Test constants.** When a test asserts a value that production
   exports as a constant (`MAX_X`), import the constant rather than
   hard-coding the number, so the test tracks it when it moves.

6. **Mock shape.** Prefer minimal mock factories. Don't build an
   elaborate object graph when the code reads two fields — a plain
   literal is enough.

7. **Hot paths.** For new CPU, I/O, image/file processing, large JSON,
   network calls in loops, or unbounded DB operations, require answers
   (in the PR description or a comment): where does it run? what bounds
   concurrency? what's the dominant cost? what happens during bursts?
   what mitigates production load?

8. **Dead defensive code contradicted by types.** If a fallback can't
   execute under the declared type contract (`x ?? []` where `x` is
   typed non-nullable `T[]`), either make the type honest
   (`x?: T[]`) or remove the fallback. (Overlaps depth mode 1 —
   attribute to depth when the contradiction is behavioral.)

9. **Duplicated validation.** If the same validation is copied across
   handlers or layers, ask whether it belongs at a shared boundary.

10. **Config / env defaults.** Check that docs, comments, and tests
    agree with how the config is actually parsed. (Overlaps depth
    mode 1.)

11. **Tests asserting mocks.** Flag tests that only assert values their
    own mocks return, rather than behavior the production code
    computes. (Overlaps depth mode 6.)

12. **Sensitive logs.** Flag logs or error strings that may expose
    secrets, tokens, credentials, or PII. (Overlaps depth mode 7.)

13. **Re-implemented helpers.** New code that rebuilds something the
    codebase already exports (a util, a client, a validation). Grep
    shared modules and files adjacent to the change; the flag names
    the existing helper to call instead — never a vague "consider
    reusing".

Keep it tight. A clean surface pass with two real flags beats twelve
reflexive ones.
