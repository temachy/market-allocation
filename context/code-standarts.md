# Code Standards

**Stack:** Next.js (App Router) · Postgres · TypeScript · Tailwind CSS · shadcn/ui

This document is the single source of truth for how code is written in this repository. It is not a style suggestion — PRs that violate these rules should be changed before merge. If a rule needs to change, update this file first, then the code.


## 1. Guiding Principles

- **Server-first.** Default to React Server Components. Only opt into Client Components when you need interactivity, state, or browser APIs.
- **Explicit over implicit.** No `any`, no magic strings, no silent fallbacks. Types and errors should be loud.
- **Colocate what changes together.** Feature code lives together (component + hook + types + test), not spread across parallel `components/`, `hooks/`, `types/` trees.
- **One way to do things.** If two patterns solve the same problem in this codebase, delete one. Consistency beats local optimization.
- **Tailwind only.** No CSS Modules, no styled-components, no inline `style={{}}` unless it's a computed/dynamic value that literally cannot be a class.

---

## 2. TypeScript Conventions

### 2.1 `type` vs `interface`

- Use **`interface`** for object shapes that represent entities, props, or anything that might be extended/implemented.
- Use **`type`** for unions, intersections, tuples, mapped types, and function signatures.

```ts
// ✅ Props → interface
interface UserCardProps {
  user: User;
  onSelect?: (id: string) => void;
}

// ✅ Union / derived → type
type Status = "idle" | "loading" | "success" | "error";
type UserId = User["id"];
```

### 2.2 Banned patterns

| Never | Instead |
|---|---|
| `any` | `unknown` + narrowing, or a real type |
| `as` casts to force-fit types | Fix the source type, or use a type guard |
| `// @ts-ignore` | `// @ts-expect-error` with a comment explaining why, or fix it |
| `enum` | `as const` object + derived union type |
| Non-null `!` assertions | Explicit checks, or `??`/optional chaining |
| Function overload soup | A single signature with a union or generic param |

```ts
// ❌ enum
enum Role { Admin, Editor, Viewer }

// ✅ as const union
export const ROLES = ["admin", "editor", "viewer"] as const;
export type Role = (typeof ROLES)[number];
```

### 2.3 Naming

- Types/interfaces/components: `PascalCase` — `UserProfile`, `OrderStatus`.
- Variables/functions: `camelCase` — `getUser`, `isLoading`.
- Booleans read as predicates: `isX`, `hasX`, `canX`, `shouldX`.
- Generics: single capital letter for simple cases (`T`, `K`, `V`); descriptive `PascalCase` for domain generics (`TData`, `TError`).
- No Hungarian notation, no `I` prefix on interfaces (`IUser` ❌ → `User` ✅).

### 2.4 Function & module style

- Prefer named exports everywhere **except** `page.tsx`, `layout.tsx`, and other Next.js special files, which require default exports.
- Arrow functions for callbacks and component bodies; `function` declarations for top-level, hoisted utilities.
- Every exported function has an explicit return type. Inference is fine for local, non-exported helpers.

```ts
// ✅
export function calculateTotal(items: LineItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.qty, 0);
}
```

- Validate all external input (API payloads, form data, env vars, search params) with **Zod**. Never trust `req.json()` or `searchParams` as typed without parsing.

```ts
const CreateOrderSchema = z.object({
  customerId: z.string().uuid(),
  items: z.array(z.object({ sku: z.string(), qty: z.number().int().positive() })),
});
type CreateOrderInput = z.infer<typeof CreateOrderSchema>;
```

---

## 3. Next.js App Router Conventions

### 3.1 Server vs Client Components

- **Default: Server Component.** Add `"use client"` only when the file needs `useState`, `useEffect`, event handlers, refs, browser-only APIs, or a third-party client-only library.
- Push `"use client"` as far down the tree as possible — wrap only the interactive leaf, not the whole page.
- Never fetch data in a Client Component with `useEffect`. Fetch in a Server Component/Server Action and pass data down as props, or use a data-fetching library (React Query/SWR) only when client-side revalidation is genuinely required.

```tsx
// ✅ page.tsx — Server Component, fetches data
export default async function OrdersPage() {
  const orders = await getOrders();
  return <OrdersTable orders={orders} />;
}

// ✅ Only the interactive bit is a Client Component
"use client";
export function OrdersTable({ orders }: { orders: Order[] }) {
  const [sortKey, setSortKey] = useState<keyof Order>("createdAt");
  // ...
}
```

### 3.2 Required special files per route segment

Every route segment that renders UI must define, as applicable:

| File | Purpose | Required? |
|---|---|---|
| `page.tsx` | Route UI | Yes, for leaf routes |
| `layout.tsx` | Shared shell | Yes, at root; elsewhere when segments share chrome |
| `loading.tsx` | Suspense fallback | Yes, for any route with async data fetching |
| `error.tsx` | Error boundary (Client Component) | Yes, for any route that can throw |
| `not-found.tsx` | 404 state | When the segment has dynamic params |

```tsx
// error.tsx must be a Client Component
"use client";
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return <ErrorState message={error.message} onRetry={reset} />;
}
```

### 3.3 Data fetching & caching

- Use `fetch` with explicit caching intent every time — do not rely on the default.

```ts
// Static / rarely changes
fetch(url, { cache: "force-cache" });

// Revalidate on an interval
fetch(url, { next: { revalidate: 3600 } });

// Always fresh
fetch(url, { cache: "no-store" });
```

- Tag cached data for on-demand invalidation and call `revalidateTag`/`revalidatePath` from the mutating Server Action or route handler — never invalidate blindly with time-based revalidation when a mutation exists.

### 3.4 Routing conventions

- **Route groups** `(groupName)` organize routes without affecting the URL — use for layout segmentation (`(marketing)`, `(dashboard)`, `(auth)`).
- **Private folders** `_components`, `_lib` opt folders out of routing for colocated, non-route code.
- **Dynamic segments**: `[id]` for required, `[[...slug]]` for optional catch-all. Always type `params` and `searchParams` as `Promise` (async APIs) and `await` them.

```tsx
export default async function OrderPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const order = await getOrder(id);
  if (!order) notFound();
  return <OrderDetail order={order} />;
}
```

### 3.5 Metadata

- Static metadata: export a `metadata` object. Dynamic metadata: export `generateMetadata`. Never set `<title>`/`<meta>` manually in JSX.

```tsx
export async function generateMetadata({ params }: { params: Promise<{ id: string }> }): Promise<Metadata> {
  const { id } = await params;
  const order = await getOrder(id);
  return { title: `Order #${order.number} · Acme` };
}
```

---

## 4. API Route Standards

Applies to every `app/api/**/route.ts` (webhooks, external/mobile clients, non-form integrations).

### 4.1 File shape

```ts
// app/api/prices/[id]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";

const ParamsSchema = z.object({ id: z.string().uuid() });

export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const parsedParams = ParamsSchema.safeParse(await params);
  if (!parsedParams.success) {
    return NextResponse.json({ error: "Invalid price id" }, { status: 400 });
  }

  const price = await getPrices(parsedParams.data.id);
  if (!order) {
    return NextResponse.json({ error: "Price not found" }, { status: 404 });
  }

  return NextResponse.json({ data: price }, { status: 200 });
}
```

### 4.2 Rules

- **One exported function per HTTP verb** (`GET`, `POST`, `PATCH`, `DELETE`) — no manual `request.method` branching.
- **Every input is validated**: route params, query string, and body all go through Zod before use.
- **Response envelope is consistent** across the whole API:
  - Success: `{ data: T }`
  - Error: `{ error: string, details?: unknown }`
- **Status codes are used correctly** — don't return `200` with an error payload:
  | Code | Meaning |
  |---|---|
  | 200 | Success (GET/PATCH/DELETE) |
  | 201 | Resource created (POST) |
  | 400 | Validation failure |
  | 401 | Not authenticated |
  | 403 | Authenticated, not authorized |
  | 404 | Resource not found |
  | 409 | Conflict (duplicate, stale state) |
  | 422 | Semantically invalid (valid shape, invalid business rule) |
  | 500 | Unhandled server error |
- **No business logic in the handler.** Handlers parse input, call a service/`lib` function, map the result to a response. Keep the actual logic testable outside of Next's request/response types.
- **Every route wraps logic in try/catch** and never leaks raw error objects or stack traces to the client.

```ts
export async function POST(request: NextRequest) {
  try {
    const body = CreatePriceSchema.parse(await request.json());
    const price = await priceService.create(body);
    return NextResponse.json({ data: price }, { status: 201 });
  } catch (err) {
    if (err instanceof z.ZodError) {
      return NextResponse.json({ error: "Invalid input", details: err.issues }, { status: 400 });
    }
    console.error("[price:POST]", err);
    return NextResponse.json({ error: "Internal server error" }, { status: 500 });
  }
}
```

---

## 5. Project & File Organization

### Rules

- **Colocate feature-specific code** under its route segment in `_components/`, `actions.ts`, and local types. Only promote something to `components/shared`, `hooks/`, or `types/` when a **second** feature needs it.
- **`components/ui`** is shadcn/ui-generated output. Do not hand-modify these for one-off styling — extend via `className`/`cva` variants at the call site, or fork into `components/shared` if it diverges structurally.
- **No barrel files (`index.ts` re-exports)** inside feature folders — they hide circular dependencies and slow down tooling. Import directly from the source file. A single top-level `components/ui/index.ts` generated by shadcn tooling is the only exception.
- **One component per file.** File name matches the component name in kebab-case; the component itself is exported in PascalCase.
- **Tests** live next to the file they test: `prices-table.tsx` + `prices-table.test.tsx`.

---

## 6. Component Conventions

### 6.1 File structure, top to bottom

```tsx
"use client"; // 1. directive (only if needed)

// 2. external imports
import { useState } from "react";
import { z } from "zod";

// 3. internal absolute imports (@/)
import { Button } from "@/components/ui/button";
import { cn } from "@/lib/utils";

// 4. types
interface PricesProps {
  onFilterChange: (filters: Filters) => void;
  className?: string;
}

// 5. component
export function pricesFilters({ onFilterChange, className }: priceFiltersProps) {
  const [status, setStatus] = useState<Status>("all");
  // ...
  return <div className={cn("flex gap-2", className)}>{/* ... */}</div>;
}
```

### 6.2 Props

- Destructure props in the function signature, never access via `props.x` inside the body.
- Every component that renders a root DOM element accepts and forwards an optional `className` merged with `cn()`, so callers can extend styling without new variant props for every case.
- Boolean props default to `false`-equivalent behavior when omitted; never require a boolean prop to be explicitly passed.
- Prefer composition (`children`) over prop-drilled render configuration for anything more complex than 2–3 conditional fields.

### 6.3 Size limits

- A component file over ~150 lines is a signal to extract a subcomponent or a hook — not a hard rule, but a prompt to review.
- Business/formatting logic (currency, dates, computed labels) is extracted into a `lib/` or colocated `utils.ts` function, not inlined in JSX.

---

## 7. Styling Standards (Tailwind + shadcn/ui)

### 7.1 Tailwind only

- No CSS Modules, no styled-components/emotion, no plain `.css` files except `globals.css` (resets, font faces, CSS variables, Tailwind layers).
- `style={{}}` is allowed **only** for values computed at runtime that can't be a class (e.g. a dynamic `transform` from drag coordinates, a chart's computed width).

### 7.2 Class name construction

Always merge classes with `cn()` (a `clsx` + `tailwind-merge` wrapper), never string concatenation.

```ts
// lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
// ✅
<div className={cn("rounded-lg border p-4", isActive && "border-primary", className)} />

// ❌ string concatenation — no conflict resolution, easy to typo
<div className={`rounded-lg border p-4 ${isActive ? "border-primary" : ""} ${className}`} />
```

### 7.3 Variants: use `cva`, not conditional strings

Any component with more than one visual variant (size, intent, state) defines a `cva` config, not scattered ternaries.

```ts
import { cva, type VariantProps } from "class-variance-authority";

export const badgeVariants = cva(
  "inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground",
        destructive: "bg-destructive text-destructive-foreground",
        outline: "border border-input text-foreground",
      },
    },
    defaultVariants: { variant: "default" },
  },
);

interface BadgeProps extends VariantProps<typeof badgeVariants> {
  className?: string;
  children: React.ReactNode;
}
```

### 7.4 Design tokens, not raw values

- Color, spacing, radius, and font-size utilities must come from the Tailwind theme (`bg-primary`, `text-muted-foreground`, `rounded-lg`), which is itself driven by CSS variables in `globals.css` (shadcn's `--background`, `--primary`, etc.).
- **No arbitrary hex values in `className`** (`bg-[#3b82f6]` ❌). If a color is missing, add it to the theme in `tailwind.config.ts`, don't hardcode it inline.
- Arbitrary value syntax (`w-[137px]`) is allowed only for genuinely one-off layout constraints that don't map to the spacing scale (e.g. matching an external asset's exact pixel size) — not as a substitute for using scale utilities.

### 7.5 Ordering & formatting

- Class order is enforced automatically by `prettier-plugin-tailwindcss` — do not hand-order classes or argue about it in review. Run Prettier before commit.
- Long conditional class sets go in `cva` or a local `const classes = cn(...)` above the `return`, not inline as a 6-line ternary inside JSX.

### 7.6 Dark mode & theming

- Never branch styling with JS (`isDark ? "bg-black" : "bg-white"`). Use Tailwind's `dark:` variant, which reads off the `.dark` class driven by the CSS-variable theme.
- All new colors are added as a light/dark pair of CSS variables in `globals.css`, then referenced via the Tailwind theme — never as a one-off `dark:bg-[#111]`.

### 7.7 shadcn/ui usage

- Add components via the CLI (`npx shadcn@latest add button`) — never hand-copy component code from the docs, so version metadata stays consistent.
- Customize appearance through the theme (CSS variables) and `cva` variants, not by editing the generated primitive's internal class strings per-project.
- Compose, don't reinvent: build feature UI (e.g. an `PriceStatusBadge`) on top of primitives (`Badge`) in `components/shared`, rather than writing new raw HTML that duplicates a primitive shadcn already provides.

---

## 8. State & Data Fetching

- **Server state** (anything from the DB/API) is fetched in Server Components or Server Actions. Do not duplicate server state into `useState` "for convenience" — pass it down as props.
- **Client-only UI state** (open/closed, active tab, form field values) uses `useState`/`useReducer` locally in the Client Component that owns it.
- **Cross-component client state** (shopping cart, active theme) uses React Context or a small store (e.g. Zustand) — never prop-drill more than 2 levels, and never reach for global state before Context/props are proven insufficient.

---

## 9. Naming Conventions Reference

| Item | Convention | Example |
|---|---|---|
| Component file | kebab-PascalCase | `PricesStatusBadge.tsx` |
| Component name | PascalCase | `PricesStatusBadge` |
| Hook file & name | `useCamelCase` | `useCamelCase.ts` |
| Server Action file | `actions.ts` | `app/(dashboard)/prices/actions.ts` |
| API route file | `route.ts` | `app/api/price/route.ts` |
| Type/interface | PascalCase | `PriceStatus`, `UserCardProps` |
| Variable/function | camelCase | `getPricesTotal` |
| Constant (module-level, immutable) | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Zod schema | PascalCase + `Schema` suffix | `CreatePricesSchema` |
| Boolean | `is`/`has`/`can`/`should` prefix | `isLoading`, `hasError` |
| Env var | SCREAMING_SNAKE_CASE | `DATABASE_URL` |
| CSS variable (theme) | kebab-case, `--` prefix | `--primary`, `--radius` |

---

## 10. Linting, Formatting & Enforcement

- **ESLint** (`next/core-web-vitals` + `@typescript-eslint`) runs on every commit and in CI. No `eslint-disable` without an inline comment explaining why.
- **Prettier** + `prettier-plugin-tailwindcss` formats all files; no manual formatting debates in review.
- **Husky + lint-staged** run type-check, lint, and format on `pre-commit` for staged files.

```json
// package.json (relevant excerpt)
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css}": ["prettier --write"]
  }
}
```

- No config file (`tsconfig.json`, `.eslintrc`, `tailwind.config.ts`) is weakened to make a single PR pass — fix the code, not the rules.

