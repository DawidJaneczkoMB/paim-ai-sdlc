---
name: backend-typescript
description: Use when building type-safe backends in TypeScript — typing API request/response contracts, domain models, branded entity IDs, generic repository/service patterns, Result types, and narrowing unknown data from databases or external APIs. Triggers on Node.js/Hono/Express/Fastify TypeScript projects, `any` in DTO types, missing type safety on DB results, unsound API contracts, ID mix-up bugs, or eliminating `any` from backend code.
---

# Backend TypeScript

## Overview

Type-safe patterns for backend TypeScript — domain models, API contracts, data validation boundaries, and generic infrastructure layers. Pair with the `node` skill for Node.js runtime patterns.

## Core Patterns

### Typed API Contracts

Use discriminated unions for all API responses. Never use `any` in handler return types.

```ts
type ApiResponse<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; code: string; message: string };

// Route handler
async function getUser(id: UserId): Promise<ApiResponse<User>> {
  const user = await userRepo.findById(id);
  if (!user) return { status: 'error', code: 'NOT_FOUND', message: 'User not found' };
  return { status: 'success', data: user };
}
```

### Branded Entity IDs

Prevent mixing `UserId`/`PostId`/`OrderId` — all are structurally identical `string`.

```ts
type Branded<TValue, TBrand> = TValue & { __brand: TBrand };

type UserId  = Branded<string, 'UserId'>;
type PostId  = Branded<string, 'PostId'>;
type OrderId = Branded<string, 'OrderId'>;

function getUser(id: UserId): Promise<User> { /* ... */ }

getUser(postId); // Error: 'PostId' is not assignable to 'UserId' ✓
```

### Narrowing External Data

DB results and third-party API responses arrive as `unknown`. Assert shape at the boundary — never cast.

```ts
// BAD - skips validation
const user = dbResult as User;

// GOOD - assert before trusting
function assertUser(value: unknown): asserts value is User {
  if (typeof value !== 'object' || value === null || !('id' in value)) {
    throw new Error('Invalid user shape from DB');
  }
}
assertUser(dbResult);
// dbResult is User here
```

### Generic Repository Type

```ts
type Repository<TEntity, TId> = {
  readonly findById:  (id: TId) => Promise<TEntity | null>;
  readonly findAll:   (filter?: Partial<TEntity>) => Promise<ReadonlyArray<TEntity>>;
  readonly create:    (data: Omit<TEntity, 'id' | 'createdAt'>) => Promise<TEntity>;
  readonly update:    (id: TId, patch: Partial<Omit<TEntity, 'id'>>) => Promise<TEntity>;
  readonly delete:    (id: TId) => Promise<void>;
};
```

### Result Type for Service Errors

Prefer `Result` over `throw` for expected failures (validation, not-found, conflict).

```ts
type Result<TValue, TError extends Error = Error> =
  | { ok: true; value: TValue }
  | { ok: false; error: TError };

async function createUser(data: CreateUserInput): Promise<Result<User, ValidationError | ConflictError>> {
  const existing = await userRepo.findByEmail(data.email);
  if (existing) return { ok: false, error: new ConflictError('Email in use') };
  const user = await userRepo.create(data);
  return { ok: true, value: user };
}
```

## Utility Types for Backend DTOs

| Utility | Use case |
|---------|----------|
| `Omit<User, 'id' \| 'createdAt'>` | Create input — strip server-generated fields |
| `Partial<Omit<User, 'id'>>` | Update/patch input |
| `Pick<User, 'id' \| 'email'>` | Public projection — strip sensitive fields |
| `Required<Config>` | Validated config after env parse |
| `Awaited<ReturnType<typeof fn>>` | Extract resolved type from async function |
| `Parameters<typeof fn>` | Wrap/proxy a function without repeating its signature |

## Rule Files

| Topic | File |
|-------|------|
| Request/response typing, discriminated API unions | [rules/api-contracts.md](rules/api-contracts.md) |
| Entity types, DTOs, branded IDs, opaque validated types | [rules/domain-types.md](rules/domain-types.md) |
| Type guards, narrowing, assertion functions | [rules/type-safety-patterns.md](rules/type-safety-patterns.md) |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `req.body as User` | Assert via type guard before trusting external data |
| `string` for all entity IDs | Branded types: `UserId`, `PostId`, `OrderId` |
| `Promise<any>` on service methods | Return `ApiResponse<T>` or `Result<T, E>` |
| Casting DB results: `row as User` | Validate shape at the DB/ORM boundary |
| `Partial<User>` for create input | Use `Omit<User, 'id' \| 'createdAt'>` |
| Arrow function with `asserts` return | Must use `function` declaration for assertion functions |
