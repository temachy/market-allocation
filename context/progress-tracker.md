# Progress Tracker

Update this file after every meaningful implementation change.

## Current Phase

- Development workflow configuration

## Current Goal

- Route delegated work to the appropriate Codex agent, model, and reasoning effort.

## Completed

- Added project-scoped agent defaults and role definitions for routine operations, discovery, complex implementation, and review.
- Updated `AGENTS.md` with the required agent-selection policy.

## In Progress

- None yet.

## Next Up

- Use the defined roles for future delegated work.

## Open Questions

- None.

## Architecture Decisions

- Delegate bounded operational work and straightforward discovery to `gpt-5.6-luna` with low reasoning effort.
- Delegate high-complexity implementation and risk-focused review to `gpt-5.6-terra` with extra-high reasoning effort.

## Session Notes

- Agent definitions live in `.codex/agents/`; project defaults live in `.codex/config.toml`.
