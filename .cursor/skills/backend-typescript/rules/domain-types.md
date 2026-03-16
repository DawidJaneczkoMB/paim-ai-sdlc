# Domain Types

## Branded Entity IDs

TypeScript uses structural typing — `UserId`, `PostId`, and `OrderId` are all `string`, so they're interchangeable without brands.

```ts
// Pattern
type Branded<TValue, TBrand> = TValue & { __brand: TBrand };

// Entity IDs
type UserId  = Branded<string, 'UserId'>;
type PostId  = Branded<string, 'PostId'>;
type OrderId = Branded<string, 'OrderId'>;

// Now mixing IDs is a compile error
function getUser(id: UserId): Promise<User> { /* ... */ }

const postId = 'post_123' as PostId;
getUser(postId); // Error: 'PostId' is not assignable to 'UserId' ✓
```

### Creating Branded Values

```ts
// Factory function — validates format, returns branded type
function parseUserId(raw: string): UserId {
  if (!raw.startsWith('user_')) throw new Error('Invalid UserId format');
  return raw as UserId;
}

// From DB / crypto — trust the source, cast once at the boundary
const id = crypto.randomUUID() as UserId;
```

## Opaque Validated Types

Brand primitives that have been validated to prevent using unvalidated data in sensitive operations.

```ts
type ValidEmail    = Branded<string, 'ValidEmail'>;
type HashedPassword = Branded<string, 'HashedPassword'>;
type ApiKey        = Branded<string, 'ApiKey'>;

// Must go through validation to get the branded type
function assertValidEmail(email: string): asserts email is ValidEmail {
  if (!email.includes('@') || email.length < 5) {
    throw new Error('Invalid email format');
  }
}

// DB layer requires validated types — unvalidated strings won't compile
async function createUser(data: {
  email:    ValidEmail;
  password: HashedPassword;
}): Promise<User> { /* ... */ }

// Handler enforces the validation boundary
async function handleRegistration(raw: { email: string; password: string }) {
  assertValidEmail(raw.email);
  const hashed = await bcrypt.hash(raw.password, 10) as HashedPassword;
  return createUser({ email: raw.email, password: hashed });
}
```

## Entity and DTO Definitions

Define the entity once. Derive all DTOs from it — never define the same fields twice.

```ts
// Domain entity (full, internal)
type User = {
  readonly id:             UserId;
  readonly email:          ValidEmail;
  readonly hashedPassword: HashedPassword;
  readonly name:           string;
  readonly role:           UserRole;
  readonly createdAt:      Date;
  readonly updatedAt:      Date;
};

// DTOs derived from entity
type PublicUser      = Omit<User, 'hashedPassword'>;               // API response
type CreateUserInput = Omit<User, 'id' | 'createdAt' | 'updatedAt'>; // create body
type UpdateUserInput = Partial<Omit<User, 'id' | 'createdAt'>>;    // patch body
type UserCredentials = Pick<User, 'email' | 'hashedPassword'>;      // auth query
```

## Role / Status Enums

Use `as const` arrays for iterable values and literal union for the type.

```ts
const USER_ROLES = ['admin', 'moderator', 'member', 'guest'] as const;
type UserRole = (typeof USER_ROLES)[number]; // 'admin' | 'moderator' | 'member' | 'guest'

// Exhaustiveness check at compile time
function getPermissions(role: UserRole): ReadonlyArray<string> {
  switch (role) {
    case 'admin':     return ['read', 'write', 'delete', 'manage'];
    case 'moderator': return ['read', 'write', 'delete'];
    case 'member':    return ['read', 'write'];
    case 'guest':     return ['read'];
  }
}
```

## Value Objects

Wrap primitives that have domain meaning to prevent primitive obsession.

```ts
type Money = {
  readonly amount:   number; // in cents
  readonly currency: 'USD' | 'EUR' | 'GBP';
};

type Address = {
  readonly street:  string;
  readonly city:    string;
  readonly country: string;
  readonly zip:     string;
};

type Order = {
  readonly id:         OrderId;
  readonly userId:     UserId;
  readonly total:      Money;    // not just `number`
  readonly address:    Address;  // not 4 flat string fields
  readonly status:     OrderStatus;
  readonly createdAt:  Date;
};
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `type UserId = string` | Use `Branded<string, 'UserId'>` |
| Separate `CreateUserDto` type with same fields | Derive: `Omit<User, 'id' \| 'createdAt'>` |
| `string` for validated email | `ValidEmail = Branded<string, 'ValidEmail'>` |
| `'admin' \| 'user'` duplicated across files | Single `as const` array + `(typeof ROLES)[number]` |
| Flat primitive fields (`street`, `city`, `zip`) | Group into value object `Address` |
