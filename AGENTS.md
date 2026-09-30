<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

## Application Building Context

Read the following files in order before implementing
or making any architectural decision:

1. `context/project-overview.md` — product definition,
   goals, features, and scope
2. `context/architecture.md` — system structure,
   boundaries, storage model, and invariants
3. `context/code-standards.md` — implementation rules
   and conventions
4. `context/ai-workflow-rules.md` — development workflow,
   scoping rules, and delivery approach
5. `context/progress-tracker.md` — current phase,
   completed work, open questions, and next steps

Update `context/progress-tracker.md` after each
meaningful implementation change.

If implementation changes the architecture, scope, or
standards documented in the context files, update the
relevant file before continuing.

<!-- END:nextjs-agent-rules -->

## Agent Delegation and Model Routing

Use project-scoped custom agents from `.codex/agents/` whenever work is
delegated. Choose the narrowest agent that can complete the task:

| Work | Agent | Model and effort |
|---|---|---|
| A bounded shell or Git operation, formatting command, or routine verification command with no diagnosis required | `ops_runner` | `gpt-5.6-luna` / `low` |
| Read-only file discovery or code-path mapping with a clear question | `scout` | `gpt-5.6-luna` / `low` |
| Architecture decisions, cross-layer features, difficult debugging, migrations, security-sensitive changes, or ambiguous requirements that require substantial reasoning | `deep_worker` | `gpt-5.6-terra` / `xhigh` |
| High-risk correctness, security, regression, or test-coverage review | `reviewer` | `gpt-5.6-terra` / `xhigh` |

The configured default for any otherwise-unspecified subagent is
`gpt-5.6-luna` with `low` reasoning effort. Do not use Terra for routine
commands, simple discovery, or mechanical checks. Do not use a low-effort
agent to make architectural decisions, diagnose a non-obvious failure, or
modify multiple application layers.

When an operation is standalone and falls in the first two rows, delegate it
to the named agent. For complex work, delegate only independent, bounded
subtasks; keep one agent responsible for a file at a time. The parent agent
must synthesize results and make any final product decision.

`ops_runner` may run only routine, explicitly bounded commands. It must return
control if output needs diagnosis or if a command would be destructive,
state-changing beyond the requested operation, or otherwise requires a design
decision.
