# Type Safety Patterns

## Narrowing Unknown External Data

Data from databases, third-party APIs, and message queues arrives as `unknown`. Validate shape at the boundary.

### Assertion Functions

Throw on invalid shape. Narrow the type after the call.

```ts
// Must use `function` declaration — arrow functions don't support `asserts`
function assertUser(value: unknown): asserts value is User {
  if (
    typeof value !== 'object' ||
    value === null ||
    !('id' in value) ||
    !('email' in value) ||
    !('name' in value)
  ) {
    throw new Error(`Invalid user shape: ${JSON.stringify(value)}`);
  }
}

// DB result
const row = await db.query('SELECT * FROM users WHERE id = $1', [id]);
assertUser(row.rows[0]);
// row.rows[0] is User ✓
```

### Type Predicates

Return boolean — let the caller decide what to do on failure.

```ts
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'email' in value
  );
}

// Array filtering
const rows: unknown[] = await db.query('...');
const users = rows.filter(isUser); // User[] ✓
```

### Prefer Zod at IO Boundaries

For complex shapes use Zod — it generates both the validator and the type.

```ts
import * as z from 'zod';

const USER_SCHEMA = z.object({
  id:    z.string(),
  email: z.email(),
  name:  z.string(),
  role:  z.enum(['admin', 'moderator', 'member', 'guest']),
});

type User = z.infer<typeof USER_SCHEMA>;

// Parse DB result — throws with detailed message on mismatch
const user = USER_SCHEMA.parse(row);

// Or safe parse — returns Result-like object
const result = USER_SCHEMA.safeParse(row);
if (!result.success) {
  logger.error('DB row shape mismatch', result.error);
  throw new Error('Data integrity error');
}
const user = result.data; // User ✓
```

## Discriminated Union Narrowing

Model service results with a discriminant field. TypeScript narrows the shape in each branch.

```ts
type ServiceResult<T> =
  | { ok: true;  value: T }
  | { ok: false; error: AppError };

async function findUser(id: UserId): Promise<ServiceResult<User>> {
  const user = await userRepo.findById(id);
  if (!user) return { ok: false, error: new NotFoundError('User not found') };
  return { ok: true, value: user };
}

// Caller narrows
const result = await findUser(id);
if (!result.ok) {
  return c.json({ status: 'error', message: result.error.message }, 404);
}
// result.value is User ✓
return c.json({ status: 'success', data: result.value });
```

## Generic Type Guards

Reusable utility guards for common backend patterns.

```ts
// Filter nulls from arrays (DB LEFT JOIN results)
function isNotNull<T>(value: T | null | undefined): value is T {
  return value !== null && value !== undefined;
}

const rows = await db.query('...');
const users = rows.filter(isNotNull); // eliminates null rows

// Check property existence on unknown
function hasKey<K extends string>(
  obj: unknown,
  key: K,
): obj is Record<K, unknown> {
  return typeof obj === 'object' && obj !== null && key in obj;
}
```

## Exhaustiveness Checking

Use `never` to catch unhandled union members at compile time — critical for role/status switches.

```ts
type UserRole = 'admin' | 'moderator' | 'member' | 'guest';

function getPermissions(role: UserRole): ReadonlyArray<string> {
  switch (role) {
    case 'admin':     return ['read', 'write', 'delete', 'manage'];
    case 'moderator': return ['read', 'write', 'delete'];
    case 'member':    return ['read', 'write'];
    case 'guest':     return ['read'];
    default: {
      // Adding a new role without updating this switch → compile error ✓
      const _exhaustive: never = role;
      throw new Error(`Unhandled role: ${_exhaustive}`);
    }
  }
}
```

## Narrowing Pitfalls

### Assertion Arrow Functions Don't Work

```ts
// WRONG — TypeScript requires `function` declaration for `asserts`
const assertUser = (v: unknown): asserts v is User => { /* ... */ };

// CORRECT
function assertUser(v: unknown): asserts v is User { /* ... */ }
```

### Narrowing Doesn't Persist Across Callbacks

```ts
function process(value: string | null) {
  if (value !== null) {
    setTimeout(() => {
      // value is string | null again inside async callback
      console.log(value.toUpperCase()); // Error
    }, 0);

    // Capture in const before async boundary
    const safeValue = value; // string
    setTimeout(() => {
      console.log(safeValue.toUpperCase()); // OK ✓
    }, 0);
  }
}
```

### Type Guard Must Have Predicate Return Type

```ts
// WRONG — just returns boolean, doesn't narrow
function isUser(v: unknown) {
  return typeof v === 'object' && v !== null && 'id' in v;
}

// CORRECT — `v is User` is the narrowing annotation
function isUser(v: unknown): v is User {
  return typeof v === 'object' && v !== null && 'id' in v && 'email' in v;
}
```

## When to Use Each Technique

| Technique | Use Case |
|-----------|----------|
| `asserts value is T` | DB results, message queue payloads — throw fast on bad shape |
| `value is T` predicate | Filtering arrays, conditional processing |
| Zod `.parse()` | Complex nested shapes at HTTP/IO boundaries |
| Discriminated union | Service layer results, command/query responses |
| `never` exhaustive check | Role/status/event-type switches |
