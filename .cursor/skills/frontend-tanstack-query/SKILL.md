---
name: tanstack-query
description: Use when working with TanStack Query (React Query) — defining queries, mutations, caching, optimistic updates, pagination, infinite queries, prefetching, or debugging stale data. Triggers on useQuery, useMutation, QueryClient, queryOptions, queryKey, or any React Query task.
---

# TanStack Query (React Query)

**Official Docs**: [TanStack Query](https://tanstack.com/query/latest)

## Setup

```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,
      gcTime: 1000 * 60 * 5,
      retry: 1,
      refetchOnWindowFocus: true,
    },
  },
});
```

- `staleTime`: how long data is considered fresh (no refetch). Default `0` triggers refetch on every mount — always configure it.
- `gcTime`: how long unused cache entries live. Must be greater than `staleTime`.
- Never set `staleTime` higher than `gcTime`.

## queryOptions — Primary Abstraction

Use `queryOptions` for every query definition. Co-locates `queryKey` + `queryFn`, reusable across all query hooks and client methods.

```ts
import { queryOptions } from '@tanstack/react-query';

export function todoQueryOptions(id: string) {
  return queryOptions({
    queryKey: ['todos', id],
    queryFn: () => fetchTodo(id),
    staleTime: 5000,
    enabled: !!id,
  });
}

// Full type safety everywhere
useQuery(todoQueryOptions(id));
useSuspenseQuery(todoQueryOptions(id));
queryClient.prefetchQuery(todoQueryOptions(id));
queryClient.ensureQueryData(todoQueryOptions(id));
queryClient.getQueryData(todoQueryOptions(id).queryKey); // typed as Todo | undefined
queryClient.invalidateQueries({ queryKey: todoQueryOptions(id).queryKey });
```

## Query Key Factories

Combine key factories with `queryOptions` for hierarchical invalidation.

```ts
export const todoQueries = {
  all: () => ['todos'] as const,
  lists: () => [...todoQueries.all(), 'list'] as const,
  list: (filters: string) =>
    queryOptions({
      queryKey: [...todoQueries.lists(), filters],
      queryFn: () => fetchTodos(filters),
    }),
  details: () => [...todoQueries.all(), 'detail'] as const,
  detail: (id: string) =>
    queryOptions({
      queryKey: [...todoQueries.details(), id],
      queryFn: () => fetchTodo(id),
      staleTime: 5000,
    }),
};

// Invalidation at any granularity
queryClient.invalidateQueries({ queryKey: todoQueries.all() });
queryClient.invalidateQueries({ queryKey: todoQueries.lists() });
queryClient.invalidateQueries({ queryKey: todoQueries.detail(id).queryKey });
```

## Query Key as Dependency Array

Treat `queryKey` like `useEffect` deps. Every variable used in `queryFn` must appear in `queryKey`. When key changes, React Query refetches — do not use `refetch` to change parameters.

```ts
// BAD - key doesn't include filters, stale data
const { refetch } = useQuery({
  queryKey: ['todos'],
  queryFn: () => fetchTodos(filters),
});

// GOOD - key drives the query
const { data } = useQuery({
  queryKey: ['todos', filters],
  queryFn: () => fetchTodos(filters),
});
```

## Status Checks — Data First

Check `data` availability before `error` to keep stale data visible during background refetch failures.

```tsx
// BAD - replaces valid stale data with error screen
if (isPending) return <Spinner />;
if (isError) return <ErrorPage error={error} />;
return <TodoList todos={data} />;

// GOOD
if (data) return <TodoList todos={data} />;
if (isError) return <ErrorPage error={error} />;
return <Spinner />;
```

## Error Handling

### Error Boundaries

```tsx
useQuery({
  ...todoQueryOptions(id),
  throwOnError: (error) => error.status >= 500,
});
```

### Global Error Toasts

Place error callbacks on `QueryCache` / `MutationCache`, not on individual `useQuery` calls (those fire per observer, causing duplicate toasts).

```ts
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      if (query.state.data !== undefined)
        toast.error(`Background update failed: ${error.message}`);
    },
  }),
});
```

### Custom Retry Logic

```ts
retry: (failureCount, error) => {
  if (error.status === 404 || error.status === 403) return false;
  return failureCount < 3;
},
retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000),
```

## Mutations

### Basic Pattern

```tsx
const mutation = useMutation({
  mutationFn: createTodo,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: todoQueries.lists() });
  },
});
```

### Optimistic Updates

Follow the four-step pattern: cancel → snapshot → update → rollback context.

```tsx
useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({
      queryKey: todoQueries.detail(newTodo.id).queryKey,
    });
    const previous = queryClient.getQueryData(
      todoQueries.detail(newTodo.id).queryKey,
    );
    queryClient.setQueryData(todoQueries.detail(newTodo.id).queryKey, newTodo);
    return { previous };
  },
  onError: (_err, newTodo, context) => {
    queryClient.setQueryData(
      todoQueries.detail(newTodo.id).queryKey,
      context?.previous,
    );
  },
  onSettled: (_data, _err, newTodo) => {
    queryClient.invalidateQueries({
      queryKey: todoQueries.detail(newTodo.id).queryKey,
    });
  },
});
```

Always cancel outgoing queries before optimistic updates to prevent race conditions. Always invalidate in `onSettled`.

## Server State vs Client State

Never copy server data into `useState`.

```tsx
// BAD
const { data } = useQuery(todoQueryOptions(id));
const [todo, setTodo] = useState(data);

// GOOD
const { data: todo } = useQuery(todoQueryOptions(id));
```

Exception: form initialization — set `staleTime: Infinity` to prevent refetches overwriting user input.

## Data Transformation with select

```ts
// GOOD - transform via select, component only re-renders when selected value changes
const { data: completedTodos } = useQuery({
  ...todosQueryOptions,
  select: (data) => data.filter((t) => t.done),
});
```

## Dependent Queries

```tsx
const { data: user } = useQuery(userQueryOptions(userId));
const { data: posts } = useQuery({
  ...postsByUserQueryOptions(userId),
  enabled: !!user,
});
```

## Pagination

Use `placeholderData` (not `initialData`) to keep previous page data visible during transitions.

```tsx
const { data, isPlaceholderData } = useQuery({
  queryKey: ['todos', page],
  queryFn: () => fetchTodos(page),
  placeholderData: (prev) => prev,
});
```

- `placeholderData`: shown until real data arrives, does not count as fresh. Correct for pagination.
- `initialData`: treated as real, fresh data. Correct for seeding cache from parent query data.

## Infinite Queries

```tsx
useInfiniteQuery({
  queryKey: ['posts', 'infinite'],
  queryFn: ({ pageParam }) => fetchPosts(pageParam),
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  maxPages: 5,
});
```

Do not share the same `queryKey` between `useQuery` and `useInfiniteQuery`.

## Suspense

Prefer `useSuspenseQuery` — guarantees `data` is defined (no `undefined` checks).

```tsx
function TodoList() {
  const { data } = useSuspenseQuery(todosQueryOptions);
  return data.map((todo) => <Todo key={todo.id} todo={todo} />);
}

<ErrorBoundary fallback={<ErrorPage />}>
  <Suspense fallback={<Spinner />}>
    <TodoList />
  </Suspense>
</ErrorBoundary>;
```

Use `useSuspenseQueries` for parallel fetches inside a single Suspense boundary.

## Prefetching

```tsx
function TodoLink({ id }: { readonly id: string }) {
  const queryClient = useQueryClient();
  return (
    <Link
      to={`/todos/${id}`}
      onMouseEnter={() => queryClient.prefetchQuery(todoQueries.detail(id))}
    >
      Todo {id}
    </Link>
  );
}
```

## Query Cancellation

Always forward the `signal` from `queryFn` context to fetch calls.

```ts
queryFn: async ({ signal }) => {
  const res = await fetch(`/api/todos/${id}`, { signal });
  if (!res.ok) throw new Error('Fetch failed');
  return res.json();
},
```

The `fetch` API does not reject on 4xx/5xx. Always check `response.ok` and throw manually.

## TypeScript

Do not pass generics to `useQuery` manually. Let `queryOptions` infer types from `queryFn` return type.

```ts
// BAD
useQuery<Todo[], Error>({ queryKey: ['todos'], queryFn: fetchTodos });

// GOOD
useQuery(todosQueryOptions);
```

## Testing

```tsx
function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: { queries: { retry: false, gcTime: Infinity } },
  });
}

function createWrapper() {
  const queryClient = createTestQueryClient();
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}
```

Never share `QueryClient` across tests. Use `setQueryData` to seed cache in component tests. Set `retry: false`, `gcTime: Infinity`.

## Common Pitfalls

| Pitfall                                        | Fix                               |
| ---------------------------------------------- | --------------------------------- |
| `staleTime` > `gcTime`                         | Ensure `gcTime` > `staleTime`     |
| Same key for `useQuery` and `useInfiniteQuery` | Use separate keys                 |
| `queryClient.setQueryData` as local state      | Background refetches overwrite it |
| Error screen replacing stale data              | Check `data` before `error`       |
| Multiple assertions in `waitFor`               | Use `findBy` per element          |
| Calling `useQuery` conditionally               | Use `enabled` instead             |
| Ignoring `onSettled` in optimistic updates     | Always sync with server           |
