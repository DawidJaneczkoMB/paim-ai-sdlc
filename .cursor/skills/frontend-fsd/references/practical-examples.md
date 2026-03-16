# Practical Examples

Concrete code patterns for authentication, type definitions, API handling, and state management within FSD.

---

## Authentication Patterns

### Auth data → shared/auth (always)

Tokens, session state, and login utilities are infrastructure — keep in shared:

```typescript
// shared/auth/token.ts
const TOKEN_KEY = 'auth_token';
export const getToken = (): string | null => localStorage.getItem(TOKEN_KEY);
export const setToken = (token: string): void =>
  localStorage.setItem(TOKEN_KEY, token);
export const clearToken = (): void => localStorage.removeItem(TOKEN_KEY);

// shared/auth/session.ts
export interface Session {
  userId: string;
  email: string;
  role: 'admin' | 'user';
}

// shared/auth/index.ts
export { getToken, setToken, clearToken } from './token';
export { useSession, type Session } from './session';
```

### Auth UI — page (single use) or feature (multi-use)

```text
// Login form used only on login page:
pages/login/
  ui/LoginForm.tsx
  model/login.ts      ← Form state, validation
  api/login.ts        ← POST /auth/login
  index.ts

// Login form reused in multiple places:
features/auth/
  ui/LoginForm.tsx
  ui/RegisterForm.tsx
  model/auth.ts
  api/login.ts
  index.ts
```

### Do NOT create a `user` entity just for auth

```text
// BAD — Premature entity — just wraps login response
entities/user/model/user.ts

// GOOD — Auth data in shared; entity only when profiles are genuinely reused
shared/auth/session.ts       ← Session with userId, email, role
// Later, if user profile data is needed in 2+ places:
entities/user/model/user.ts  ← Profile data (displayName, avatar, bio)
```

---

## Type Definition Patterns

| Type scope                                | Location                                    |
| ----------------------------------------- | ------------------------------------------- |
| API response/request shapes used app-wide | `shared/api/types.ts` or domain-named files |
| Specific entity's domain model            | `entities/[name]/model/[name].ts`           |
| Types used only within one page           | `pages/[name]/model/[name].ts`              |
| Types used only within one feature        | `features/[name]/model/[name].ts`           |
| Generic utility types (`Nullable<T>`)     | `shared/lib/types.ts`                       |

```typescript
// shared/api/product.ts — raw API shapes
export interface ProductDTO {
  id: string;
  name: string;
  price: number;
}

// entities/product/model/product.ts — domain model with business logic
import type { ProductDTO } from '@/shared/api/product';

export interface Product {
  id: string;
  name: string;
  price: number;
  formattedPrice: string;
  isOnSale: boolean;
}

export const fromDTO = (dto: ProductDTO): Product => ({
  ...dto,
  formattedPrice: `$${dto.price.toFixed(2)}`,
  isOnSale: dto.price < 10,
});
```

**Key principle:** Raw API shapes → `shared/api/`. Domain models with business logic → `entities/`. If you only need the raw shape with no logic, `shared/api/` alone is sufficient.

---

## API Request Handling

### Shared API client

```typescript
// shared/api/client.ts
import axios from 'axios';
import { getToken } from '@/shared/auth/token';

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});

apiClient.interceptors.request.use((config) => {
  const token = getToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

### CRUD helpers in shared

```typescript
// shared/api/crud.ts
import { apiClient } from './client';

export const createCrudApi = <T>(resource: string) => ({
  getAll: () => apiClient.get<T[]>(`/${resource}`).then((r) => r.data),
  getById: (id: string) =>
    apiClient.get<T>(`/${resource}/${id}`).then((r) => r.data),
  create: (data: Partial<T>) =>
    apiClient.post<T>(`/${resource}`, data).then((r) => r.data),
  update: (id: string, data: Partial<T>) =>
    apiClient.put<T>(`/${resource}/${id}`, data).then((r) => r.data),
  remove: (id: string) => apiClient.delete(`/${resource}/${id}`),
});

// Usage in pages or features:
import { createCrudApi } from '@/shared/api/crud';
import type { ProductDTO } from '@/shared/api/product';
export const productsApi = createCrudApi<ProductDTO>('products');
```

### Page-specific API call (not extracted)

```typescript
// pages/product-detail/api/fetch-product.ts
import { apiClient } from '@/shared/api/client';
import type { ProductDTO } from '@/shared/api/product';

export const fetchProduct = async (id: string): Promise<ProductDTO> => {
  const response = await apiClient.get(`/products/${id}`);
  return response.data;
};
```

---

## State Management: Redux

Keep the entire slice (reducer + selectors + thunks) in a single domain-named file:

```typescript
// features/todo-list/model/todo.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { apiClient } from '@/shared/api/client';

interface Todo {
  id: string;
  title: string;
  completed: boolean;
}
interface TodoState {
  items: Todo[];
  loading: boolean;
}

export const fetchTodos = createAsyncThunk('todos/fetch', async () => {
  const response = await apiClient.get<Todo[]>('/todos');
  return response.data;
});

const todoSlice = createSlice({
  name: 'todos',
  initialState: { items: [], loading: false } as TodoState,
  reducers: {
    toggleTodo: (state, action) => {
      const todo = state.items.find((t) => t.id === action.payload);
      if (todo) todo.completed = !todo.completed;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.items = action.payload;
        state.loading = false;
      });
  },
});

export const { toggleTodo } = todoSlice.actions;
export const selectTodos = (state: RootState) => state.todos.items;
export default todoSlice.reducer;
```

Register slices in app:

```typescript
// app/providers/store.ts
import { configureStore } from '@reduxjs/toolkit';
import todoReducer from '@/features/todo-list/model/todo';
import userReducer from '@/entities/user/model/user';

export const store = configureStore({
  reducer: { todos: todoReducer, user: userReducer },
});
export type RootState = ReturnType<typeof store.getState>;
```

---

## State Management: React Query

### Query hooks in the owning slice

```typescript
// entities/user/api/user-queries.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { apiClient } from '@/shared/api/client';
import type { User } from '../model/user';

export const useUser = (userId: string) =>
  useQuery({
    queryKey: ['user', userId],
    queryFn: () => apiClient.get<User>(`/users/${userId}`).then((r) => r.data),
  });

export const useUpdateUser = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (data: { id: string; updates: Partial<User> }) =>
      apiClient.put(`/users/${data.id}`, data.updates),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({ queryKey: ['user', variables.id] });
    },
  });
};
```

### Page-specific queries (not extracted)

```typescript
// pages/dashboard/api/dashboard-queries.ts
import { useQuery } from '@tanstack/react-query';
import { apiClient } from '@/shared/api/client';

export const useDashboardStats = () =>
  useQuery({
    queryKey: ['dashboard', 'stats'],
    queryFn: () => apiClient.get('/dashboard/stats').then((r) => r.data),
    staleTime: 30_000,
  });
```

**Key principle:** Place React Query hooks in the slice that owns the domain. Page-specific → keep in page. Shared data → put hooks in the entity.
