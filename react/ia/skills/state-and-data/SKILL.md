---
name: state-and-data
description: >
  Read when the task involves global state with Zustand, server state with
  TanStack Query, data caching, synchronization between local and server
  state, or advanced fetching patterns.
triggers:
  - global state
  - Zustand
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

Client State (UI state that lives in the browser) → Zustand
  Examples: open modal, authenticated user, UI preferences,
            anything purely local to the client session
```

**Never put server state in Zustand. Never put client state in TanStack Query.**

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
// src/lib/query-client.ts
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

## Zustand — Patterns

### Basic store

```typescript
// src/features/auth/store/auth.store.ts
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'
import type { User } from '../types/auth.types'

interface AuthState {
  // State
  user: User | null
  accessToken: string | null // in memory ONLY — never persisted

  // Actions — named as verbs
  setUser: (user: User) => void
  setAccessToken: (token: string) => void
  logout: () => void
}

export const useAuthStore = create<AuthState>()(
  devtools(
    (set) => ({
      user: null,
      accessToken: null,

      setUser: (user) =>
        set({ user }, false, 'auth/setUser'),

      setAccessToken: (token) =>
        set({ accessToken: token }, false, 'auth/setAccessToken'),

      logout: () =>
        set({ user: null, accessToken: null }, false, 'auth/logout'),
    }),
    { name: 'AuthStore' }
  )
)
```

### Store with selective persistence

```typescript
// src/store/ui.store.ts — UI preferences that should survive page reloads
import { create } from 'zustand'
import { persist, devtools } from 'zustand/middleware'

interface UIState {
  theme: 'light' | 'dark'
  sidebarCollapsed: boolean
  setTheme: (theme: 'light' | 'dark') => void
  toggleSidebar: () => void
}

export const useUIStore = create<UIState>()(
  devtools(
    persist(
      (set) => ({
        theme: 'light',
        sidebarCollapsed: false,
        setTheme: (theme) => set({ theme }),
        toggleSidebar: () =>
          set((state) => ({ sidebarCollapsed: !state.sidebarCollapsed })),
      }),
      {
        name: 'ui-preferences',
        // Only persist what matters — never persist sensitive data
        partialize: (state) => ({
          theme: state.theme,
          sidebarCollapsed: state.sidebarCollapsed,
        }),
      }
    ),
    { name: 'UIStore' }
  )
)
```

### Selectors — prevent unnecessary re-renders

```typescript
// ✅ Granular selector — only re-renders when user.name changes
const userName = useAuthStore((state) => state.user?.name)

// ❌ Full store subscription — re-renders on any state change
const { user, accessToken } = useAuthStore()

// ✅ Multiple values: use shallow equality
import { useShallow } from 'zustand/react/shallow'

const { theme, sidebarCollapsed } = useUIStore(
  useShallow((state) => ({
    theme: state.theme,
    sidebarCollapsed: state.sidebarCollapsed,
  }))
)
```

---

## When to use what

| Situation | Tool |
|---|---|
| List or detail data from API | `useQuery` (TanStack Query) |
| Create / update / delete on server | `useMutation` (TanStack Query) |
| Authenticated user (shared across features) | Zustand store |
| Modal open / closed (local to one component) | `useState` (local) |
| Modal open / closed (triggered from anywhere) | Zustand store |
| UI preferences that persist across sessions | Zustand + `persist` middleware |
| Filter state for a single page | `useState` (local — no need for a store) |
| Complex form with many fields | React Hook Form (see forms-and-validation skill) |
