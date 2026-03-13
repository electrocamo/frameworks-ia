---
name: state-and-data
description: >
  Read when the task involves global state with Redux Toolkit, server state
  with TanStack Query, data caching, synchronization between local and server
  state, or advanced fetching patterns.
triggers:
  - global state
  - Redux
  - Redux Toolkit
  - react-redux
  - TanStack Query
  - useQuery
  - useMutation
  - cache
  - server state
  - client state
  - store
  - invalidate cache
  - staleTime
  - prefetch
---

# SKILL: State and Data

## Fundamental rule — two state types, two tools

```
Server State (data that lives on the server) → TanStack Query
  Examples: user list, products, API data, anything that can
            go stale, needs caching, or requires background sync

Client State (UI state that lives in the browser) → Redux Toolkit
  Examples: open modal, authenticated user, UI preferences,
            anything purely local to the client session
```

**Never put server state in Redux. Never put client state in TanStack Query.**

---

## TanStack Query — Patterns

### Basic query

```typescript
// src/features/products/hooks/useProducts.ts
import { useQuery, keepPreviousData } from '@tanstack/react-query'
import { ProductService } from '../services/product.service'
import type { ProductFilters } from '../types/product.types'

// Typed query key factory — never use raw inline strings
export const productKeys = {
  all: ['products'] as const,
  lists: () => [...productKeys.all, 'list'] as const,
  list: (filters: ProductFilters) => [...productKeys.lists(), filters] as const,
  details: () => [...productKeys.all, 'detail'] as const,
  detail: (id: string) => [...productKeys.details(), id] as const,
}

export const useProducts = (filters: ProductFilters = {}) => {
  return useQuery({
    queryKey: productKeys.list(filters),
    queryFn: () => ProductService.getAll(filters),
    staleTime: 5 * 60 * 1000,        // 5 min — skip refetch if data is fresh
    gcTime: 10 * 60 * 1000,          // 10 min — keep in inactive cache
    placeholderData: keepPreviousData, // show previous data while filters change
  })
}

export const useProduct = (id: string) => {
  return useQuery({
    queryKey: productKeys.detail(id),
    queryFn: () => ProductService.getById(id),
    enabled: Boolean(id),            // skip if id is falsy
    staleTime: 5 * 60 * 1000,
  })
}
```

### Mutation with cache invalidation

```typescript
export const useCreateProduct = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: ProductService.create,

    // Optimistic update — UI updates before the API confirms
    onMutate: async (newProduct) => {
      // Cancel in-flight queries to avoid overwriting optimistic update
      await queryClient.cancelQueries({ queryKey: productKeys.lists() })

      // Snapshot previous value for rollback
      const previous = queryClient.getQueryData(productKeys.lists())

      // Optimistically add the new item
      queryClient.setQueryData(productKeys.lists(), (old: Product[] = []) => [
        ...old,
        { ...newProduct, id: 'temp-id' },
      ])

      return { previous } // passed to onError as context
    },

    onError: (_err, _newProduct, context) => {
      // Roll back on failure
      if (context?.previous) {
        queryClient.setQueryData(productKeys.lists(), context.previous)
      }
    },

    onSettled: () => {
      // Always refetch after success or error to sync server truth
      queryClient.invalidateQueries({ queryKey: productKeys.lists() })
    },
  })
}

export const useDeleteProduct = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (id: string) => ProductService.delete(id),
    onSuccess: (_, deletedId) => {
      queryClient.invalidateQueries({ queryKey: productKeys.lists() })
      queryClient.removeQueries({ queryKey: productKeys.detail(deletedId) })
    },
  })
}
```

### Prefetching (improves UX on navigation)

```typescript
// Prefetch on hover — data is ready before the user clicks
export const usePrefetchProduct = () => {
  const queryClient = useQueryClient()

  return (id: string) => {
    queryClient.prefetchQuery({
      queryKey: productKeys.detail(id),
      queryFn: () => ProductService.getById(id),
      staleTime: 5 * 60 * 1000,
    })
  }
}

// Usage in component
const prefetchProduct = usePrefetchProduct()
<ProductCard
  onMouseEnter={() => prefetchProduct(product.id)}
  onClick={() => navigate(`/products/${product.id}`)}
/>
```

### QueryClient configuration

```typescript
// src/shared/services/query-client.ts
import { QueryClient } from '@tanstack/react-query'
import { type AxiosError } from 'axios'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000, // 1 min default
      retry: (failureCount, error) => {
        // Do not retry 4xx errors — they are client errors, not transient
        if (error instanceof AxiosError) {
          const status = error.response?.status
          if (status && status >= 400 && status < 500) return false
        }
        return failureCount < 2 // max 2 retries for 5xx errors
      },
    },
  },
})
```

---

## Redux Toolkit — Patterns

### Store configuration

```typescript
// src/store/store.ts
import { configureStore } from '@reduxjs/toolkit'
import { authReducer } from './slices/auth.slice'
import { uiReducer } from './slices/ui.slice'

export const store = configureStore({
  reducer: {
    auth: authReducer,
    ui: uiReducer,
  },
  devTools: import.meta.env.DEV,
})

export type RootState = ReturnType<typeof store.getState>
export type AppDispatch = typeof store.dispatch
```

### Auth slice (token in memory only)

```typescript
// src/store/slices/auth.slice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit'
import type { User } from '@/features/auth/types/auth.types'

interface AuthState {
  user: User | null
  accessToken: string | null // in memory ONLY — never persisted
}

const initialState: AuthState = {
  user: null,
  accessToken: null,
}

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    setUser(state, action: PayloadAction<User>) {
      state.user = action.payload
    },
    setAccessToken(state, action: PayloadAction<string>) {
      state.accessToken = action.payload
    },
    logout(state) {
      state.user = null
      state.accessToken = null
    },
  },
})

export const authActions = authSlice.actions
export const authReducer = authSlice.reducer
```

### Typed hooks + selectors

```typescript
// src/store/hooks.ts
import { useDispatch, useSelector } from 'react-redux'
import type { AppDispatch, RootState } from './store'

export const useAppDispatch = () => useDispatch<AppDispatch>()
export const useAppSelector = <T>(selector: (state: RootState) => T) =>
  useSelector(selector)
```

```typescript
// ✅ Granular selectors — only re-render on selected slice changes
const userName = useAppSelector((state) => state.auth.user?.name)
const accessToken = useAppSelector((state) => state.auth.accessToken)

// ❌ Avoid selecting large objects if you only need one field
const auth = useAppSelector((state) => state.auth)
```

### Selective persistence

```typescript
// Persist only non-sensitive UI preferences.
// Never persist auth.accessToken.
// Example strategy: persist `ui` reducer only.
```

---

## When to use what

| Situation | Tool |
|---|---|
| List or detail data from API | `useQuery` (TanStack Query) |
| Create / update / delete on server | `useMutation` (TanStack Query) |
| Authenticated user (shared across features) | Redux store (`auth` slice) |
| Modal open / closed (local to one component) | `useState` (local) |
| Modal open / closed (triggered from anywhere) | Redux store (`ui` slice) |
| UI preferences that persist across sessions | Redux + selective persistence |
| Filter state for a single page | `useState` (local — no need for a store) |
| Complex form with many fields | React Hook Form (see forms-and-validation skill) |
