# AI Workflow Rules

These rules govern how an AI coding agent operates in this repository.
They are binding, not advisory. If a rule conflicts with an instruction
given in a task, follow the rule and flag the conflict instead of
resolving it silently.

## 1. Core Approach

1. Work from a spec, not from assumption. Before writing code, identify
   the specification, ticket, or written requirement describing the
   change. If none exists, do not begin implementation — go to Section 4.
2. Build incrementally. Implement the smallest coherent unit of work
   that can be independently verified, then stop and report before
   continuing to the next unit.
3. Do not implement multiple unrelated units of work in a single pass,
   even if bundling them seems efficient.
4. Do not refactor, rename, or reorganize code outside the current
   unit's scope, even if you notice an improvement opportunity. Note it
   separately; do not act on it.
5. Prefer the smallest diff that satisfies the spec. Do not rewrite a
   file when a targeted edit will do.

## 2. Scoping Rules

1. Define the unit of work before touching any file. A unit is one
   behavior, one bug fix, one component, or one endpoint — not a
   feature area.
2. Touch only the files required to implement the current unit. If
   implementing it reveals that another file must also change, stop
   and report that dependency rather than expanding scope silently.
3. Never make speculative changes. "While I'm here" edits, unused
   abstractions, extra config options, or generalized code paths not
   required by the current spec are prohibited.
4. Do not add new dependencies, libraries, or tooling unless the spec
   explicitly requires them or their absence blocks the current unit.
   If one seems necessary, stop and ask before adding it.
5. If you find yourself editing a file not named in the current task's
   plan, stop and justify it explicitly, or revert the edit.

## 3. When to Split Work Into Smaller Steps

Split work into smaller steps whenever any of the following is true:

1. The change touches more than one layer of the stack (e.g. database,
   API, UI) — implement and verify each layer separately.
2. The change spans more than roughly 3–5 files or 150–200 lines of
   diff — break it into sequential units.
3. Parts of the change could be reviewed or tested independently —
   split so they can be.
4. You are not fully confident a single-pass implementation is
   correct — smaller steps reduce blast radius and isolate errors.
5. The task description contains multiple distinct outcomes ("add X
   and update Y and fix Z") — treat each as its own unit.

Do not split so finely that a step becomes untestable on its own (e.g.
a function with no caller yet). Every step must be independently
verifiable.

## 4. Handling Missing or Ambiguous Requirements

1. Never guess silently. If a requirement is missing, unclear, or
   contradictory, stop before writing implementation code.
2. State explicitly what is missing or ambiguous, propose the most
   likely interpretation, and ask for confirmation before proceeding —
   unless the ambiguity is trivial and inconsequential to correctness
   (e.g. a naming style already established elsewhere in the codebase).
3. Do not resolve ambiguity by picking whichever option is easiest to
   implement. Pick the option that best matches existing project
   conventions and state the reasoning.
4. If told to proceed without confirmation, record the assumption made
   (in a comment or commit message) so it can be revisited later.
5. Never invent requirements, business rules, or data shapes that were
   not specified anywhere in the spec, code, or conversation.

## 5. Protected Files — Never Modify Without Explicit Instruction

Do not edit, delete, or regenerate the following unless a task
explicitly names them as the target of the change:

1. Generated files of any kind — build output, lockfiles, generated
   types, generated API clients, and generated UI library components.
2. Third-party or vendored code.
3. Project-wide configuration files (CI pipelines, environment config,
   package manifests). These may be read for context but must not
   change as a side effect of an unrelated unit of work.
4. Database migration files that have already been merged — create a
   new migration instead of editing history.
5. Any file carrying an auto-generated marker (e.g. a header comment
   such as "DO NOT EDIT" or "@generated").

If a task appears to require touching a protected file, stop and ask
for explicit confirmation naming that file before proceeding.

## 6. Keeping Documentation in Sync

1. Any change that alters public behavior — an API signature, a CLI
   flag, a config option, user-facing UI behavior — must update the
   corresponding documentation in the same unit of work, not as a
   follow-up.
2. Do not leave a README, inline doc comment, or API reference
   describing behavior that no longer exists after your change.
3. If a change makes existing documentation inaccurate and you're not
   certain how to rewrite it, flag this rather than leaving it stale.
4. Do not document speculative or future behavior that isn't yet
   implemented.
5. Keep documentation edits proportional to the code change — update
   what was affected, don't rewrite unrelated sections.

## 7. Verification Checklist — Before Moving to the Next Unit

Confirm every item below before considering a unit complete:

- [ ] The change matches the spec exactly — no more, no less.
- [ ] Only files within the declared scope were touched (Section 2).
- [ ] No protected file (Section 5) was modified without explicit
      instruction.
- [ ] New or changed code is covered by a test, or a reason is given
      for why it can't be tested.
- [ ] Existing tests pass; none were weakened or deleted to make the
      change pass.
- [ ] Documentation affected by the change was updated (Section 6).
- [ ] No speculative or out-of-scope code was introduced.
- [ ] Any assumptions made about ambiguous requirements are recorded.
- [ ] The diff size was reviewed and, if unexpectedly large,
      reconsidered against Section 3.

Do not move to the next unit until every item above is checked. If any
item fails, fix it before continuing — do not carry it forward as
"cleanup later."