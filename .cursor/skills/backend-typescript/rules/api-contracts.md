# API Contracts

## Discriminated Response Union

Define a single response wrapper used by every handler. The `status` discriminant lets callers narrow without casting.

```ts
type ApiResponse<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; code: string; message: string };

// Hono example
app.get('/users/:id', async (c) => {
  const result = await getUser(c.req.param('id') as UserId);
  if (result.status === 'error') {
    return c.json(result, result.code === 'NOT_FOUND' ? 404 : 500);
  }
  return c.json(result);
});
```

## Typed Route Handlers

Annotate handler return types explicitly — prevents drift between implementation and API surface.

```ts
type GetUserHandler = (id: UserId) => Promise<ApiResponse<User>>;

const getUser: GetUserHandler = async (id) => {
  const user = await userRepo.findById(id);
  if (!user) return { status: 'error', code: 'NOT_FOUND', message: 'User not found' };
  return { status: 'success', data: user };
};
```

## Request Body Typing

Never trust `req.body` as the declared type. Validate with Zod at the handler boundary, then narrow.

```ts
import * as z from 'zod';

const CREATE_USER_SCHEMA = z.object({
  name:  z.string().min(1).max(100),
  email: z.email(),
});

type CreateUserInput = z.infer<typeof CREATE_USER_SCHEMA>;

app.post('/users', async (c) => {
  const parsed = CREATE_USER_SCHEMA.safeParse(await c.req.json());
  if (!parsed.success) {
    return c.json({ status: 'error', code: 'VALIDATION', message: parsed.error.message }, 400);
  }
  const result = await createUser(parsed.data); // CreateUserInput ✓
  return c.json(result);
});
```

## Paginated Response Type

```ts
type PaginatedResponse<T> = {
  readonly data:       ReadonlyArray<T>;
  readonly total:      number;
  readonly page:       number;
  readonly pageSize:   number;
  readonly hasMore:    boolean;
};

type ListUsersResponse = ApiResponse<PaginatedResponse<PublicUser>>;
```

## Extracting Types from External Libraries

When a library does not export its types, derive them:

```ts
import { fetchUser } from 'external-lib';

// Extract what the function returns
type UserData = Awaited<ReturnType<typeof fetchUser>>;

// Wrap it without repeating the signature
const fetchUserWithAudit = async (
  ...args: Parameters<typeof fetchUser>
): Promise<UserData & { fetchedAt: Date }> => {
  const user = await fetchUser(...args);
  return { ...user, fetchedAt: new Date() };
};
```

## Service → Handler Type Flow

Keep types flowing top-down: define entity in domain → derive DTOs → derive API response.

```ts
// 1. Domain entity (source of truth)
type User = {
  readonly id:        UserId;
  readonly email:     ValidEmail;
  readonly name:      string;
  readonly createdAt: Date;
};

// 2. DTOs derived from entity
type PublicUser      = Omit<User, 'createdAt'>;       // response
type CreateUserInput = Omit<User, 'id' | 'createdAt'>; // create body
type UpdateUserInput = Partial<Omit<User, 'id'>>;      // patch body

// 3. API response derived from DTO
type GetUserResponse    = ApiResponse<PublicUser>;
type CreateUserResponse = ApiResponse<PublicUser>;
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `res.json(data as User)` | Validate shape; never cast response data |
| Handler returns `any` | Annotate return: `Promise<ApiResponse<T>>` |
| Error shape varies per route | Use single `ApiResponse<T>` union everywhere |
| `req.body.email` without validation | Parse with Zod `.safeParse()` first |
