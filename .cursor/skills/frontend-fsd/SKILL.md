---
name: fsd
description: Use when the user asks where to put a file, how to structure a feature/entity/widget, whether an import is valid, how to resolve cross-imports, or needs to refactor toward FSD. Also use when implementing patterns (auth, Redux, React Query) inside an FSD project.
---

# Feature-Sliced Design (FSD) v2.1

FSD organizes frontend code into a strict hierarchy of **Layers → Slices → Segments**. Its core promise: every file has exactly one right place, and imports only flow downward.

> **FSD v2.1 core principle: "Start simple, extract when needed."**  
> Place code in `pages/` first. Extract only when the team agrees it's necessary.

---

## Quick Orientation

Before writing any code, ask:

1. **Where is it used?** — Only one page → keep it there. Multiple pages → consider extracting.
2. **What layer?** — How broadly is it reused?
3. **What slice?** — Which business domain?
4. **What segment?** — What is the technical nature (`ui/`, `model/`, `api/`)?

**When in doubt, keep it in `pages/`. Extract upward only on confirmed reuse.**

---

## Folder Structure

```
src/
├── app/          # App-wide: routing, providers, global styles
├── pages/        # Route-level composition — owns substantial logic
├── widgets/      # Large UI blocks reused across 2+ pages
├── features/     # Reusable user interactions (2+ uses)
├── entities/     # Reusable business domain models (2+ uses)
└── shared/       # Infrastructure with no business logic
```

**Not all layers are required.** A valid minimal project:

```
src/
  app/       ← Providers, routing
  pages/     ← All page-level code
  shared/    ← UI kit, utils, API client
```

Add `widgets/`, `features/`, `entities/` only when they provide clear value.

**App and Shared** have no slices — segments only.  
**All other layers** use: `layer/slice/segment/`.

### Inside every slice (except App/Shared)

```
features/
└── user-filter/        ← slice
    ├── ui/             ← components, styles
    ├── model/          ← state, business logic
    ├── api/            ← backend requests
    ├── lib/            ← helpers internal to this slice
    ├── config/         ← feature flags, slice-level config
    └── index.ts        ← PUBLIC API — required
```

---

## The Import Rule

> A module may only import from layers **strictly below** its own layer.

| Layer    | Can import from                            |
| -------- | ------------------------------------------ |
| app      | pages, widgets, features, entities, shared |
| pages    | widgets, features, entities, shared        |
| widgets  | features, entities, shared                 |
| features | entities, shared                           |
| entities | shared (+ other entities via `@x` only)    |
| shared   | nothing project-specific                   |

**Never import upward. Never cross-import between slices on the same layer** (except entities via `@x`).

---

## Decision Framework

**Step 1 — Used in only one place?**  
→ Keep it in that `pages/` slice. (Duplication across pages is acceptable.)

**Step 2 — Is it reusable infrastructure with no business logic?**  
→ `shared/` (UI kit, utils, API client, auth tokens, CRUD helpers)

**Step 3 — Is it a complete user interaction reused in 2+ places?**  
→ `features/` (only with team agreement)

**Step 4 — Is it a business domain model reused in 2+ places?**  
→ `entities/` (only with team agreement; start without this layer)

**Step 5 — Is it app-wide configuration?**  
→ `app/` (providers, router, global styles)

---

## Layer Responsibilities

| Layer        | Put here                                                                          | Do NOT put here                               |
| ------------ | --------------------------------------------------------------------------------- | --------------------------------------------- |
| **shared**   | API client, UI kit, date/string utils, route constants, auth tokens, CRUD helpers | Business logic, feature-specific code         |
| **entities** | Business domain types + logic reused in 2+ places                                 | Single-use types, CRUD fetching, auth data    |
| **features** | Forms, actions, filters, searches — things users _do_ (reused 2+ places)          | Single-page-only interactions                 |
| **widgets**  | Nav bars, sidebars, dashboards reused across pages                                | Entire page content, single-use page sections |
| **pages**    | Page UI, forms, validation, data fetching, state, business logic                  | Code reused across pages                      |
| **app**      | Router config, global providers, global styles, analytics init                    | Feature-specific logic                        |

---

## Core Patterns

### Public API (index.ts)

Every slice **must** have an `index.ts`. Only export what consumers need.

```ts
// features/user-filter/index.ts
export { UserFilter } from './ui/UserFilter';
export type { UserFilterProps } from './ui/UserFilter';
// Do NOT export internal hooks, api functions, or model internals
```

- Import from slices via the public API only: `import { X } from "@/features/user-filter"`
- Never: `import { X } from "@/features/user-filter/ui/UserFilter"`
- Inside a slice, use relative imports for same-slice files

### Domain-Based File Naming

Name files after the business domain, not their technical role:

```
// BAD — Technical-role naming
model/types.ts     ← Which types? Mixes domains
model/utils.ts

// GOOD — Domain-based naming
model/user.ts      ← User types + logic
model/order.ts     ← Order types + logic
api/fetch-profile.ts
```

### Model Segment = Business Brain

The `model/` segment holds state, derived data, and the contract between UI and API. Extract logic from components into `model/` hooks — keeps components declarative.

### Deferred Decomposition

Do not create entities/features prematurely. Start in the page, extract only when requirements stabilize and the team agrees. Code in `entities/` affects every layer above it — extract late.

### Slice Naming

Name slices after what they represent, not what they contain:

- GOOD: `user-filter`, `post-comments`, `checkout-form`
- BAD: `filter`, `modal`, `helpers`

---

## Cross-Import Resolution Order

When two slices on the same layer need to share code, try these **in order**:

1. **Merge slices** — if they always change together, they're one concept
2. **Extract shared logic to entities** — move domain logic down, keep UI up
3. **Compose in a higher layer (IoC)** — parent imports both slices, wires them via render props / slots / DI
4. **@x notation (last resort, entities only)** — explicit controlled cross-import

```
// @x example
entities/user/@x/order.ts     ← exposes only what order entity needs
entities/order/model/order.ts ← imports from "@/entities/user/@x/order"
```

> **@x rules:** Document why, review periodically, minimize exported surface, only between entities.

For full code examples of all 4 strategies → read `references/cross-import-patterns.md`

---

## Anti-Patterns

- **Creating entities too early** — single-use types belong in the page or feature
- **Putting CRUD in entities** — use `shared/api/` instead
- **Creating a `user` entity just for auth data** — auth tokens/session belong in `shared/auth/`
- **Wildcard re-exports** (`export * from`) in `index.ts` — always export explicitly
- **Deep imports into slices** — always use the public API (`index.ts`)
- **God slices** — split overly broad slices (e.g., `user-management/` → `auth/`, `profile-edit/`, `password-reset/`)
- **Excessive `@x`** — a symptom of too many premature entities; consider merging or moving back to pages
- **`shared/ui` with one giant `index.ts`** — split per-component to preserve tree-shaking
- **Technical-role file names** (`types.ts`, `utils.ts`, `helpers.ts`) — use domain names

---

## Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/app/*": ["src/app/*"],
      "@/pages/*": ["src/pages/*"],
      "@/widgets/*": ["src/widgets/*"],
      "@/features/*": ["src/features/*"],
      "@/entities/*": ["src/entities/*"],
      "@/shared/*": ["src/shared/*"]
    }
  }
}
```

---

## Reference Files

Read these **only when the specific situation applies**:

| Situation                                                             | File                                    |
| --------------------------------------------------------------------- | --------------------------------------- |
| Complete project tree with all layers and real file examples          | `./references/folder-structure.md`      |
| Folder structure with per-layer examples and naming conventions       | `./references/layer-structure.md`       |
| Cross-imports, @x pattern, IoC composition, excessive entity coupling | `./references/cross-import-patterns.md` |
| Auth patterns, type definitions, API client, Redux, React Query       | `./references/practical-examples.md`    |
| Public API, index.ts patterns, barrel files, tree-shaking             | `./references/public-api.md`            |
| Entities deep dive — what belongs, what doesn't                       | `./references/entities-guide.md`        |
| React Query / TanStack Query query keys and functions placement       | `./references/react-query.md`           |
| Authentication patterns — session, tokens, route guards               | `./references/auth.md`                  |

Do **not** preload all references — load only the one relevant to the current task.
