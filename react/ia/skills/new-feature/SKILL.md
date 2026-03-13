---
name: new-feature
description: >
  Read when the task is to create a component, hook, service, or complete
  feature end-to-end. Defines the correct implementation order, which files
  to create, in what sequence, with ready-to-use examples.
triggers:
  - new component
  - new hook
  - new service
  - new feature
  - new page
  - new route
  - create feature
---

# SKILL: Creating a New Feature

## Implementation order — always in this sequence

```
1. types/[feature].types.ts       ← data contracts
2. services/[feature].service.ts  ← API calls with Zod validation
3. hooks/use[Feature].ts          ← logic with TanStack Query
4. components/[Feature].tsx        ← pure presentation
5. [Feature].test.tsx              ← co-located component tests
6. Update `src/router/AppRouter.tsx` ← only if a new route is needed
```

**Never skip steps. Never generate a component without its test file.**

---

## Step 1 — Types

```typescript
// src/features/users/types/user.types.ts
import { z } from "zod";

// Define the Zod schema first — derive TypeScript types from it
// Never write a type/interface and a Zod schema separately for the same shape
export const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(["admin", "user", "viewer"]),
  createdAt: z.string().datetime(),
});

export const UserListSchema = z.array(UserSchema);

export const CreateUserSchema = UserSchema.omit({ id: true, createdAt: true });

// Types derived from schemas — never write duplicate interfaces
export type User = z.infer<typeof UserSchema>;
export type CreateUserInput = z.infer<typeof CreateUserSchema>;
```

---

## Step 2 — Service

```typescript
// src/features/users/services/user.service.ts
import { apiClient } from "@/api/httpInterceptor";
import {
  UserSchema,
  UserListSchema,
  CreateUserSchema,
} from "../types/user.types";
import type { User, CreateUserInput } from "../types/user.types";

// Every API response is validated with Zod before returning
// This catches API contract changes at runtime, not in the UI
export const UserService = {
  getAll: async (): Promise<User[]> => {
    const { data } = await apiClient.get("/users");
    return UserListSchema.parse(data); // throws ZodError if shape changes
  },

  getById: async (id: string): Promise<User> => {
    const { data } = await apiClient.get(`/users/${id}`);
    return UserSchema.parse(data);
  },

  create: async (input: CreateUserInput): Promise<User> => {
    const validated = CreateUserSchema.parse(input); // validate input too
    const { data } = await apiClient.post("/users", validated);
    return UserSchema.parse(data);
  },

  update: async (
    id: string,
    input: Partial<CreateUserInput>,
  ): Promise<User> => {
    const { data } = await apiClient.patch(`/users/${id}`, input);
    return UserSchema.parse(data);
  },

  delete: async (id: string): Promise<void> => {
    await apiClient.delete(`/users/${id}`);
  },
};
```

---

## Step 3 — Hook (TanStack Query)

```typescript
// src/features/users/hooks/useUsers.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { UserService } from "../services/user.service";
import type { CreateUserInput } from "../types/user.types";

// Query keys as typed constants — never use inline strings
export const userKeys = {
  all: ["users"] as const,
  lists: () => [...userKeys.all, "list"] as const,
  detail: (id: string) => [...userKeys.all, "detail", id] as const,
};

export const useUsers = () => {
  return useQuery({
    queryKey: userKeys.lists(),
    queryFn: UserService.getAll,
    staleTime: 5 * 60 * 1000, // 5 min — do not refetch if data is fresh
  });
};

export const useUser = (id: string) => {
  return useQuery({
    queryKey: userKeys.detail(id),
    queryFn: () => UserService.getById(id),
    enabled: Boolean(id), // skip if id is undefined or empty
  });
};

export const useCreateUser = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: CreateUserInput) => UserService.create(input),
    onSuccess: () => {
      // Invalidate the list so it refetches fresh data
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
    },
  });
};

export const useDeleteUser = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => UserService.delete(id),
    onSuccess: (_, deletedId) => {
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
      queryClient.removeQueries({ queryKey: userKeys.detail(deletedId) });
    },
  });
};
```

---

## Step 4 — Component

```typescript
// src/features/users/components/UserList.tsx
// Rule: logic in the hook, presentation here
// Rule: always handle loading + error + empty
// Rule: named export — zero default exports

import { useUsers } from '../hooks/useUsers'
import { UserCard } from './UserCard'
import { UserListSkeleton } from '@/components/ui/Skeleton'
import { ErrorMessage } from '@/components/ui/ErrorMessage'
import { EmptyState } from '@/components/ui/EmptyState'

export const UserList = () => {
  const { data: users, isLoading, isError, error } = useUsers()

  if (isLoading) return <UserListSkeleton />

  if (isError) return (
    <ErrorMessage
      title="Failed to load users"
      message={error.message}
    />
  )

  if (!users?.length) return (
    <EmptyState
      title="No users found"
      description="There are no registered users yet."
    />
  )

  return (
    <ul role="list" aria-label="User list">
      {users.map((user) => (
        <li key={user.id}>
          <UserCard user={user} />
        </li>
      ))}
    </ul>
  )
}
```

---

## Step 5 — Co-located Test (mandatory)

See `ia/skills/testing/SKILL.md` for complete examples.

Minimum required per component:

- Loading state
- Success state with data (happy path)
- Error state
- Empty state (when applicable)

```typescript
// src/features/users/components/UserList.test.tsx
describe('UserList', () => {
  it('renders skeleton while loading', () => { ... })
  it('renders user cards when data loads successfully', async () => { ... })
  it('renders error message when query fails', async () => { ... })
  it('renders empty state when no users exist', async () => { ... })
})
```

---

## Step 6 — New Route (only when needed)

```typescript
// src/router/AppRouter.tsx
import { lazy, Suspense } from 'react'
import { createBrowserRouter } from 'react-router-dom'
import { PageSkeleton } from '@/components/ui/PageSkeleton'

// Always lazy-load pages — reduces initial bundle size
const UsersPage = lazy(() =>
  import('@/features/users/components/UsersPage')
    .then(m => ({ default: m.UsersPage })) // named export → lazy compatible
)

export const router = createBrowserRouter([
  {
    path: '/users',
    element: (
      <Suspense fallback={<PageSkeleton />}>
        <UsersPage />
      </Suspense>
    ),
  },
])
```

---

## Feature barrel export — index.ts

```typescript
// src/features/users/index.ts
// Export only what other features need to consume.
// Never export internal implementation details.
export { UserList } from "./components/UserList";
export { useUsers, useUser, useCreateUser } from "./hooks/useUsers";
export type { User, CreateUserInput } from "./types/user.types";
```

---

## httpInterceptor.ts — reference configuration

```typescript
// src/api/httpInterceptor.ts
import axios, { type AxiosError } from "axios";
import { store } from "@/store/store";
import { authActions } from "@/store/slices/auth.slice";

const API_BASE_URL = import.meta.env.VITE_API_URL as string;

export const apiClient = axios.create({
  baseURL: API_BASE_URL,
  headers: { "Content-Type": "application/json" },
  timeout: 10_000,
});

// Request interceptor: attach access token from memory store
apiClient.interceptors.request.use((config) => {
  const token = store.getState().auth.accessToken;
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor: handle global error cases
apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    if (error.response?.status === 401) {
      const refreshed = await tryRefreshToken();
      if (!refreshed) {
        store.dispatch(authActions.logout());
        window.location.href = "/login";
      }
    }
    return Promise.reject(error);
  },
);
```
