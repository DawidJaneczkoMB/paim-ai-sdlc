---
name: tanstack-start
description: Use when building TanStack Start routes, server functions, middleware, SSR/streaming, route protection, session management, or API routes. Triggers on createServerFn, createFileRoute, beforeLoad, createMiddleware, useServerFn, or any TanStack Start task.
---

# TanStack Start

**Official Docs**: [TanStack Start](https://tanstack.com/start/latest)

## Server Functions

Use `createServerFn()` for all server-side logic. Never use raw `fetch` to hand-roll RPC.

```tsx
// BAD - manual fetch, no type safety
async function createPost(data: CreatePostInput) {
  const res = await fetch('/api/posts', {
    method: 'POST',
    body: JSON.stringify(data),
  });
  return res.json();
}

// GOOD - type-safe server function
export const createPost = createServerFn({ method: 'POST' })
  .validator(createPostSchema)
  .handler(async ({ data }) => {
    return await db.posts.create({ data });
  });
```

Default method is GET (idempotent, cacheable). Use POST for mutations.

Call server functions from loaders, components, hooks, or other server functions:

```tsx
// Route loader
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => getPost({ data: { id: params.postId } }),
});

// Component via useServerFn
function PostList() {
  const getPostsFn = useServerFn(getPosts);
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: () => getPostsFn(),
  });
}
```

Compose server functions by calling one from another:

```tsx
export const getPostWithComments = createServerFn()
  .validator(z.object({ postId: z.string() }))
  .handler(async ({ data }) => {
    const [post, comments] = await Promise.all([
      getPost({ data: { id: data.postId } }),
      getComments({ data: { postId: data.postId } }),
    ]);
    return { post, comments };
  });
```

## Input Validation

Always validate with `.validator()`. Never trust client data across the network boundary.

```tsx
const updateUserSchema = z.object({
  id: z.uuid(),
  name: z.string().min(1).max(100),
  email: z.email(),
});

export const updateUser = createServerFn({ method: 'POST' })
  .validator(updateUserSchema)
  .handler(async ({ data }) => {
    return await db.users.update({
      where: { id: data.id },
      data: { name: data.name, email: data.email },
    });
  });
```

Share schemas between client forms and server functions:

```tsx
// lib/schemas/post.ts
export const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
});
export type CreatePostInput = z.infer<typeof createPostSchema>;

// lib/posts.functions.ts
export const createPost = createServerFn({ method: 'POST' })
  .validator(createPostSchema)
  .handler(async ({ data }) => {
    /* ... */
  });

// components/CreatePostForm.tsx
const form = useForm<CreatePostInput>({
  resolver: zodResolver(createPostSchema),
});
```

## Middleware

Use `createMiddleware()` for cross-cutting concerns.

### Request Middleware

```tsx
export const authMiddleware = createMiddleware().server(
  async ({ next, request }) => {
    const session = await getSession(request);
    if (!session) throw new Error('Unauthorized');
    return next({ context: { session, user: session.user } });
  },
);
```

### Server Function Middleware

Use `type: 'function'` for middleware that needs client-side logic or input validation:

```tsx
const workspaceMiddleware = createMiddleware({ type: 'function' })
  .inputValidator(zodValidator(z.object({ workspaceId: z.string() })))
  .server(({ next, data }) => {
    console.log('Workspace:', data.workspaceId);
    return next();
  });
```

### Middleware Composition

```tsx
export const requireAuthMiddleware = createMiddleware()
  .middleware([authMiddleware])
  .server(async ({ next, context }) => {
    if (!context.user) throw redirect({ to: '/login' });
    return next({ context: { user: context.user } });
  });
```

### Middleware Factories

```tsx
function authorizationMiddleware(permissions: Record<string, Array<string>>) {
  return createMiddleware({ type: 'function' })
    .middleware([authMiddleware])
    .server(async ({ next, context }) => {
      const granted = await auth.hasPermission(context.session, permissions);
      if (!granted) throw new Error('Forbidden');
      return next();
    });
}

export const getClients = createServerFn()
  .middleware([authorizationMiddleware({ client: ['read'] })])
  .handler(async ({ context }) => {
    /* ... */
  });
```

## Route Protection

Use `beforeLoad` to check auth before routes load. Never check auth in component bodies.

```tsx
// Layout route protects all children
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ location }) => {
    const session = await getSessionData();
    if (!session)
      throw redirect({ to: '/login', search: { redirect: location.href } });
    return { user: session };
  },
  component: AuthenticatedLayout,
});

// Child routes inherit protection, context.user guaranteed
export const Route = createFileRoute('/_authenticated/dashboard')({
  loader: ({ context }) => fetchDashboardData(context.user.id),
});
```

Use pathless layout routes for grouped protection:

```
routes/
  _authenticated.tsx       # Requires login
  _authenticated/
    dashboard.tsx          # Any authenticated user
    _admin.tsx             # Admin layout
    _admin/
      users.tsx            # Admin only
```

## File Organization

| Suffix          | Purpose                          | Safe to Import on Client |
| --------------- | -------------------------------- | ------------------------ |
| `.ts`           | Shared types, utilities, schemas | Yes                      |
| `.server.ts`    | Server-only logic (db, secrets)  | No                       |
| `.functions.ts` | `createServerFn` wrappers        | Yes (RPC stub on client) |

```
lib/
  posts.ts             # Shared types + utilities
  posts.server.ts      # DB queries, internal logic
  posts.functions.ts   # Server function definitions
  schemas/
    post.ts            # Shared validation schemas
```

Never import `.server.ts` files in client code.

## SSR and Streaming

### Streaming with Suspense

Await only critical above-the-fold data. Prefetch non-critical data without awaiting.

```tsx
// BAD - blocks all HTML until slowest query resolves
loader: async ({ context: { queryClient } }) => {
  await Promise.all([queryClient.ensureQueryData(userQueries.profile()), queryClient.ensureQueryData(dashboardQueries.stats())]);
}

// GOOD - stream non-critical content
loader: async ({ context: { queryClient } }) => {
  await queryClient.ensureQueryData(userQueries.profile());
  queryClient.prefetchQuery(dashboardQueries.stats());
  queryClient.prefetchQuery(activityQueries.recent());
},
component: () => {
  const { data: user } = useSuspenseQuery(userQueries.profile());
  return (
    <div>
      <Header user={user} />
      <Suspense fallback={<StatsSkeleton />}><DashboardStats /></Suspense>
      <Suspense fallback={<ActivitySkeleton />}><RecentActivity /></Suspense>
    </div>
  );
}
```

Wrap each Suspense boundary with ErrorBoundary for independent error handling.

### Hydration Safety

Never use `Date.now()`, `Math.random()`, or `window` directly in render output. Pass dynamic values from the loader.

```tsx
// GOOD - value from loader is stable
export const Route = createFileRoute('/dashboard')({
  loader: () => ({ generatedAt: Date.now() }),
  component: () => {
    const { generatedAt } = Route.useLoaderData();
    return <span>{generatedAt}</span>;
  },
});
```

For client-only features (maps, window dimensions), use `lazy()` + Suspense or `useEffect`.

### Prerendering and Caching

```tsx
// vite.config.ts
export default defineConfig({
  server: {
    prerender: { routes: ['/', '/about', '/pricing'], crawlLinks: true },
  },
});

// ISR in loader
loader: async ({ params }) => {
  const post = await fetchPost(params.slug);
  setHeaders({
    'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
  });
  return { post };
};
```

Set `Cache-Control: private, no-store` for user-specific data.

## Error Handling

```tsx
// GOOD - catch, log, sanitize
export const createUser = createServerFn({ method: 'POST' })
  .validator(schema)
  .handler(async ({ data }) => {
    try {
      return await db.users.create({ data });
    } catch (error) {
      console.error('Failed to create user:', error);
      setResponseStatus(500);
      throw new AppError('Failed to create user', 'INTERNAL_ERROR', 500);
    }
  });
```

Use built-in helpers: `throw notFound()` for 404s, `throw redirect({ to: '/login' })` for auth redirects.

## Environment Variables

```tsx
// lib/env.server.ts
const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  SESSION_SECRET: z.string().min(32),
  NODE_ENV: z
    .enum(['development', 'staging', 'production'])
    .default('development'),
});
export const env = envSchema.parse(process.env);

// lib/env.ts (client-safe)
export const publicEnv = {
  appUrl: process.env.VITE_APP_URL ?? 'http://localhost:3000',
  stripePublicKey: process.env.VITE_STRIPE_PUBLIC_KEY!,
};
```

Never import `.server.ts` env files in client code. Never expose secrets in error messages.

## Server Routes (API Routes)

Use server routes for external consumers, webhooks, and custom response formats. Use server functions for internal RPC.

```tsx
// routes/api/webhooks/stripe.ts
export const Route = createFileRoute('/api/webhooks/stripe')({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const signature = request.headers.get('stripe-signature');
        if (!signature)
          return new Response('Missing signature', { status: 400 });
        const rawBody = await request.text();
        return new Response('OK', { status: 200 });
      },
    },
  },
});
```

## Session Security

Use HTTP-only cookies. Never store tokens in `localStorage`.

```tsx
export function getSession() {
  return useSession({
    password: process.env.SESSION_SECRET!,
    cookie: {
      name: '__session',
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 60 * 60 * 24 * 7,
    },
  });
}
```

Store minimal data in session. Fetch user details on demand.
