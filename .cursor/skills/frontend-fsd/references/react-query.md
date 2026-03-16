# React Query (TanStack Query) with FSD

The core question React Query raises in FSD: **where do query keys and query functions live?**

## Where to put query keys and functions

### Option A: In entities (preferred when entities are well-defined)

If your project has clear entities and each request maps to one entity, co-locate queries with their entity:

```
entities/
└── post/
    └── api/
        ├── post.queries.ts      # Query factory (keys + queryOptions)
        ├── get-post.ts          # Fetch function
        ├── get-posts.ts         # Fetch function
        └── index.ts
```

Export the query factory through the entity's public API:

```ts
// entities/post/index.ts
export { postQueries } from './api/post.queries';
```

### Option B: In shared/api (when entity split doesn't fit)

If requests don't map cleanly to entities, or entities are not used:

```
shared/
└── api/
    ├── queries/
    │   ├── post.ts
    │   └── document.ts
    └── index.ts
```

## Query Factory pattern

A query factory is an object whose values are functions returning `queryOptions`. This keeps all keys for an entity in one place and makes invalidation and refetching consistent.

```ts
// entities/post/api/post.queries.ts
import { keepPreviousData, queryOptions } from '@tanstack/react-query';
import { getPost } from './get-post';
import { getPosts } from './get-posts';

export const postQueries = {
  all: () => ['posts'],

  lists: () => [...postQueries.all(), 'list'],
  list: (page: number, limit: number) =>
    queryOptions({
      queryKey: [...postQueries.lists(), page, limit],
      queryFn: () => getPosts(page, limit),
      placeholderData: keepPreviousData,
    }),

  details: () => [...postQueries.all(), 'detail'],
  detail: (id?: number) =>
    queryOptions({
      queryKey: [...postQueries.details(), id],
      queryFn: () => getPost({ id }),
      staleTime: 5000,
    }),
};
```

Consuming the factory in a page:

```ts
// pages/post/ui/PostPage.tsx
import { useQuery } from '@tanstack/react-query';
import { postQueries } from '@/entities/post';

const { data, isLoading } = useQuery(postQueries.detail({ id }));
```

## Mutations

Keep mutations **separate from queries**. Two options:

### Near the place of use (recommended)

Put the mutation hook in the `api/` segment of the feature that triggers the action:

```ts
// features/update-post/api/use-update-title.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { postQueries } from '@/entities/post';

export const useUpdateTitle = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, newTitle }: { id: number; newTitle: string }) =>
      apiClient.patch(`/posts/${id}`, { title: newTitle }),

    onSuccess: (newPost) => {
      queryClient.setQueryData(
        postQueries.detail({ id: newPost.id }).queryKey,
        newPost,
      );
    },
  });
};
```

### Inline in the component

Define the `mutationFn` in shared or entities, then call `useMutation` directly in the component:

```ts
// In a page or feature component
const { mutate, isPending } = useMutation({
  mutationFn: postApi.createPost,
});
```

## QueryClient setup

Put the `QueryClient` instance in `shared/api`:

```ts
// shared/api/query-client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      gcTime: 5 * 60 * 1000,
    },
  },
});
```

The QueryClientProvider wrapper goes in `app/providers/`:

```ts
// app/providers/query-provider.tsx
import { QueryClientProvider } from "@tanstack/react-query";
import { queryClient } from "@/shared/api/query-client";

export const QueryProvider = ({ children }) => (
  <QueryClientProvider client={queryClient}>
    {children}
  </QueryClientProvider>
);
```

## API Client in shared

Centralize HTTP configuration in `shared/api/client.ts`. This is where you set base URL, headers, error handling, and token injection:

```ts
// shared/api/client.ts
export class ApiClient {
  constructor(private baseUrl: string) {}

  async get<T>(
    endpoint: string,
    params?: Record<string, string | number>,
  ): Promise<T> {
    // build URL, set headers, call fetch, handle errors
  }

  async post<T, D>(endpoint: string, body: D): Promise<T> {
    // POST with JSON body
  }
}

export const apiClient = new ApiClient(API_URL);
```

Entity and feature `api/` segments import and call this client — they never configure the HTTP layer themselves.

## Fetch functions

Write fetch functions as plain async functions (not hooks). They live in `api/` segments and are called by query factories or mutation functions:

```ts
// entities/post/api/get-posts.ts
import { apiClient } from '@/shared/api/client';

export const getPosts = async (
  page: number,
  limit: number,
): Promise<PostWithPagination> => {
  const result = await apiClient.get('/posts', { skip: page * limit, limit });
  return mapToPostWithPagination(result);
};
```

Separate DTOs (response shapes), mappers (DTO → domain model), and query types into their own files within `api/` for larger entities.
