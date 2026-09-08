# Architecture

This document describes the technical architecture of the application: the stack, who owns what, where data lives, how (or whether) access is controlled, and the rules the codebase must never break. Update it as real decisions replace assumptions below.

## 1. Stack

| Layer | Technology | Role |
|---|---|---|
| Framework | Next.js (App Router) | Full-stack React framework — routing, Server Components, Route Handlers, and Server Actions all in one project. |
| UI | React | Component model for server- and client-rendered UI. |
| Styling | Tailwind CSS | Utility-first CSS, chosen for speed of iteration and because it's a skill you're actively practicing. |
| Components | shadcn/ui (optional) | Copy-in, unstyled-by-default primitives (not an installed dependency) — use where a component saves time, skip it elsewhere. |
| Language | TypeScript | Static types across `app/` → `services/` → `dao/` so the layer boundaries below are enforced at compile time, not just by convention. |
| Database | PostgreSQL | System of record for all structured, relational data. |
| DB hosting | Neon or Supabase | Managed serverless Postgres |
| ORM / query layer | Prisma | Type-safe queries + migrations, used exclusively inside `dao/`. |
| Cache | Build in Nextjs cache | on request it caches data |
| AI provider | OpenAI API (ChatGPT models) | Text generation, called through a single wrapped client. |
| Deployment | Vercel | Hosting |

## 2. System Boundaries

```
/
├─ app/                          # routing layer only — thin, no business logic
│  ├─ api/
│  │  └─ users/
│  │     └─ route.ts             # parses request → calls a service → returns response
│  ├─ users/
│  │  ├─ page.tsx                # Server Component, calls a service directly
│  │  └─ actions.ts              # Server Actions ('use server'), also thin
│  └─ layout.tsx
│
├─ services/                     # business logic layer
│  ├─ user-service.ts            # validation, orchestration, transactions
│  └─ ai-service.ts              # prompt building, cache lookup, calls OpenAI
│
├─ dao/                          # data access layer (repositories)
│  └─ user-dao.ts                # raw CRUD queries only, no business rules
│
├─ db/
│  ├─ client.ts                  # Prisma client singleton
│  └─ schema.prisma              # schema definition
│
├─ models/                       # (or types/)
│  └─ user.ts                    # shared types, zod schemas
│
└─ lib/
   ├─ openai-client.ts           # thin OpenAI SDK wrapper, no business logic
   └─ validation.ts, utils.ts …
```

| Folder | Owns | Must never contain |
|---|---|---|
| `app/` | Routing, request/response shaping, calling exactly one service per handler or action | Business logic, direct `dao/` calls, direct `openai-client` calls, raw queries |
| `services/` | Business rules, validation orchestration, transactions spanning multiple DAOs, calling `ai-service`/OpenAI | HTTP-specific objects (`req`/`res`), query-builder calls |
| `dao/` | Raw CRUD/query logic, mapping DB rows to model types | Business rules, calls into `services/` |
| `db/` | ORM client singleton, schema, migrations | Business logic, feature-specific queries |
| `types/` | Shared TS types, Zod schemas, pure data shapes | Logic, side effects, DB or network calls |
| `lib/` | Generic, feature-agnostic helpers and external API/cache clients | Domain-specific business rules |

## 3. Storage Model

| Data | Lives in | Why |
|---|---|---|
| Core domain entities (users, and whatever primary records the app introduces) | Postgres | Needs relational integrity, transactions, and joinable queries |
| AI request/response metadata (prompt, model, timestamp, token usage, response id) | Postgres | Structured and small enough to store, and useful to query/join against other entities |
| Full raw AI response text, keyed by a hash of the normalized prompt + params | Postgres | - |
| Fetch/RSC payload caching | Next.js built-in data cache | Framework-level caching for repeated reads |

## 4. Auth & Access Model

**Current state: no authentication.** Every route, page, and Server Action is effectively public, and no data is scoped to a user. This is a deliberate, current-phase decision — nothing below requires auth to exist yet.

## 5. AI & Background Task Model

- `services/ai-service.ts` owns the full AI request lifecycle: build and validate the prompt, call `lib/openai-client.ts` on a cache miss persist data to Postgres.
- `lib/openai-client.ts` is a thin wrapper around the OpenAI SDK. It holds the API key (server-only env var) and does nothing else — no caching, no business rules, no retries beyond basic transport-level handling.
- **Current execution model:** synchronous — a Route Handler or Server Action calls `ai-service`, waits for the result, and returns it. This is fine while generations are single-shot and fast.
- **Streaming:** not needed.

## 6. Invariants

Rules the codebase must never violate, regardless of how urgent a feature feels:

1. **Only `services/` may call `dao/` or `lib/openai-client.ts`.** `app/` never imports either directly. This is what keeps business rules in one place and makes `app/` safely replaceable (e.g. swapping a page for a different UI) without touching logic.
2. **`dao/` contains no business conditionals** — It only takes parameters and returns data. Every business rule lives in `services/`.
3. **Every mutation goes through a service function.** There is no direct use of `db/client.ts` (Prisma) outside `dao/` and `services/` — never from a component, page, or Route Handler.
4. **Cache is never the source of truth.** Anything readable from the Next.js data cache must be fully re-derivable from Postgres or file storage. No feature may depend on data that exists only in the cache.
5. **Server-only secrets stay server-only.** `DATABASE_URL`, `OPENAI_API_KEY`, are read only inside `db/`, `lib/`, and `services/` — never imported into a Client Component and never exposed via a `NEXT_PUBLIC_` variable.
6. **One schema per shape.** The Zod schema in `models/` (or `types/`) is the single source of truth for a given data shape — it validates input at the `app/` boundary and provides the inferred type used through `services/` and `dao/`. No parallel/duplicate type definitions.
