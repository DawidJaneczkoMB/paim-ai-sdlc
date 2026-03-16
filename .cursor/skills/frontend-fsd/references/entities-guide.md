# Entities Layer Guide

## What belongs in entities

An entity represents a real-world business concept that the application works with — the vocabulary your business uses. Examples: `User`, `Post`, `Order`, `Product`, `Comment`.

An entity slice can contain:

- **model/** — TypeScript types, interfaces, validation schemas, business logic that belongs to this concept
- **ui/** — the visual representation of this entity (reusable across pages/features; not full interactive blocks)
- **api/** — entity-specific query factories and fetch functions

## What does NOT belong in entities

| Thing                                                | Where it should go              |
| ---------------------------------------------------- | ------------------------------- |
| CRUD operations (generic fetch/create/update/delete) | `shared/api/endpoints/`         |
| Auth tokens, session data, current-user info         | `shared/auth/`                  |
| Cross-entity business logic                          | `features/` or `pages/`         |
| State that only one page/feature uses                | That page or feature's `model/` |

## Thin vs thick client check

Before creating entities, ask: does the frontend hold significant business logic, or does it just display backend data?

- **Thin client** (just displaying backend data with minimal logic) → you probably don't need an entities layer at all
- **Thick client** (client-side rules, complex data transformations, multi-step business processes) → entities are appropriate

It's completely valid to have no `entities/` layer.

## Defer creating entities

Do not create entities up-front. Start logic in `model/` of the page or feature you're building. Extract to entities only when:

- Multiple features or pages need the same business concept
- Requirements are stable enough that a refactor won't immediately undo your work

Entities are globally accessible — changes to them ripple upward into every layer. Extract late.

## Avoid excessive slicing in entities

Do not create a separate entity for every piece of related data:

```
# BAD — Over-sliced — order-item and order-customer-info as separate entities creates
#    coupling headaches and forces @x cross-imports
entities/
├── order/
├── order-item/
└── order-customer-info/

# GOOD — Consolidated — keep tightly coupled concepts together
entities/
└── order-info/
    └── model/
        └── order-info.ts   # includes items and customer info
```

## CRUD in shared/api, not entities

CRUD operations are boilerplate without business logic. Move them out:

```
# GOOD — Structure
shared/
└── api/
    └── endpoints/
        ├── order.ts    # getOrder, createOrder, updateOrder, deleteOrder
        ├── product.ts
        └── user.ts

entities/
└── order/
    └── model/
        └── apply-discount.ts   # Business logic, imports OrderDto from shared/api
```

## Auth data in shared, not entities

Even if you have a `user` entity, do not store auth tokens or session info there:

- Auth response shapes differ from general user shapes
- Tokens are needed across many layers — storing in entities creates cross-layer import complexity
- Use `shared/auth` for the token store and `shared/api` for auth endpoint calls

```
shared/
├── auth/
│   ├── use-auth.ts    # Current user info, token
│   └── index.ts
└── api/
    └── endpoints/
        └── auth.ts    # login(), logout(), refreshToken()
```

## Cross-imports between entities

When entity A's model genuinely contains entity B (e.g., an `Artist` has many `Songs`):

1. Create `entities/song/@x/artist.ts` — export only what artist needs
2. Import in `entities/artist/model/artist.ts` via `entities/song/@x/artist`

Keep cross-imports to the absolute minimum. If you're adding many `@x` files, reconsider whether some entities should be merged or whether the logic should move to a feature.
