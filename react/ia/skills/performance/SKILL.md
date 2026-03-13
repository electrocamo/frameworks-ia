---
name: performance
description: >
  Read when the task involves performance optimization: reducing re-renders,
  lazy loading, memoization, bundle size analysis, or Core Web Vitals
  improvements.
triggers:
  - performance
  - optimization
  - re-renders
  - memoization
  - useMemo
  - useCallback
  - React.memo
  - lazy loading
  - bundle size
  - Core Web Vitals
  - slow
  - laggy
---

# SKILL: Performance in React

## Fundamental rule — never optimize prematurely

```
1. First:  make it work correctly
2. Second: measure with React DevTools Profiler
3. Third:  optimize only what the Profiler identifies as a problem
```

Adding `useMemo` / `useCallback` / `React.memo` without measurement is an anti-pattern.
They have memory and complexity costs. Use them only when there is evidence of a problem.

---

## Lazy Loading — always for page components

```typescript
// src/app/router.tsx
// ALL page components must be lazy — they reduce the initial bundle

import { lazy, Suspense } from 'react'
import { createBrowserRouter } from 'react-router-dom'
import { PageSkeleton } from '@/components/ui/PageSkeleton'

// Named import inside lazy — compatible with named exports
const UsersPage = lazy(() =>
  import('@/features/users/components/UsersPage')
    .then(m => ({ default: m.UsersPage }))
)

const DashboardPage = lazy(() =>
  import('@/features/dashboard/components/DashboardPage')
    .then(m => ({ default: m.DashboardPage }))
)

export const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: (
      <Suspense fallback={<PageSkeleton />}>
        <DashboardPage />
      </Suspense>
    ),
  },
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

## React.memo — only for list item components

```typescript
// ✅ Useful: component in a list that receives stable props
export const UserCard = React.memo(({ user, onSelect }: UserCardProps) => {
  return (
    <div onClick={() => onSelect(user.id)}>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  )
})
UserCard.displayName = 'UserCard' // required for React DevTools

// ❌ Useless: root component, or one receiving object/array props that change every render
export const UserList = React.memo(({ filters }) => { ... })
// filters is a new object reference on every parent render — memo never bails out
```

---

## useCallback — only for handlers passed as props

```typescript
// ✅ Useful: function passed as prop to a memoized child
export const UserListPage = () => {
  const { mutate: deleteUser } = useDeleteUser()

  // useCallback prevents UserCard from re-rendering when UserListPage re-renders
  const handleDelete = useCallback((id: string) => {
    deleteUser(id)
  }, [deleteUser]) // deleteUser is stable (from useMutation)

  return (
    <ul>
      {users.map(user => (
        <UserCard key={user.id} user={user} onDelete={handleDelete} />
      ))}
    </ul>
  )
}

// ❌ Useless: handler not passed to any memoized component
const handleClick = useCallback(() => {
  setOpen(true)
}, []) // nobody benefits from this stability
```

---

## useMemo — only for expensive transformations

```typescript
// ✅ Useful: sorting/filtering a large array
export const ProductList = ({ searchTerm }: { searchTerm: string }) => {
  const { data: products } = useProducts()

  // Avoids recalculating on every render when products has not changed
  const filteredAndSorted = useMemo(() => {
    if (!products) return []
    return products
      .filter(p => p.name.toLowerCase().includes(searchTerm.toLowerCase()))
      .sort((a, b) => a.price - b.price)
  }, [products, searchTerm])

  return (
    <ul>
      {filteredAndSorted.map(p => (
        <ProductCard key={p.id} product={p} />
      ))}
    </ul>
  )
}

// ❌ Useless: trivial computation
const fullName = useMemo(
  () => `${user.firstName} ${user.lastName}`,
  [user.firstName, user.lastName]
)
// The cost of useMemo itself exceeds the cost of string concatenation
```

---

## Preventing re-renders — common anti-patterns

### Anti-pattern: new object/array literal on every render

```typescript
// ❌ New object reference on every render — breaks React.memo
export const UserList = () => {
  return <UserFilter config={{ sortBy: 'name', order: 'asc' }} />
  //                 ↑ object literal = new reference every render
}

// ✅ Constant defined outside the component
const DEFAULT_FILTER_CONFIG = { sortBy: 'name', order: 'asc' } as const

export const UserList = () => {
  return <UserFilter config={DEFAULT_FILTER_CONFIG} />
}
```

### Anti-pattern: Context that updates too frequently

```typescript
// ❌ The value object is new on every render — all consumers re-render
export const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('light')
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

// ✅ Separate the value from the setter
export const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('light')
  const value = useMemo(() => ({ theme }), [theme])
  const actions = useMemo(() => ({ setTheme }), []) // setTheme is stable

  return (
    <ThemeContext.Provider value={value}>
      <ThemeActionsContext.Provider value={actions}>
        {children}
      </ThemeActionsContext.Provider>
    </ThemeContext.Provider>
  )
}

// Even better: use Zustand instead of Context for dynamic global state
```

---

## Bundle Size — Rules

```typescript
// ✅ Granular lodash import — only the function you need
import debounce from 'lodash/debounce'
import groupBy from 'lodash/groupBy'

// ❌ Entire lodash bundle
import _ from 'lodash'

// ✅ date-fns tree-shaking — named imports only
import { format, parseISO, addDays } from 'date-fns'

// ❌ Full date-fns bundle
import * as dateFns from 'date-fns'
```

### Analyze the bundle

```bash
# Install the visualizer
pnpm add -D rollup-plugin-visualizer

# Add to vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer'
plugins: [react(), visualizer({ open: true, gzipSize: true, filename: 'stats.html' })]

# Build and inspect
pnpm build
# Opens stats.html automatically — shows treemap of bundle contents
```

---

## Images — Performance Defaults

```tsx
// Above-the-fold image (hero, avatar in header): load eagerly
<img
  src={heroImage}
  alt="Dashboard overview"
  loading="eager"
  fetchPriority="high"
  decoding="async"
/>

// Below-the-fold image (cards, thumbnails): defer loading
<img
  src={productImage}
  alt={product.name}
  loading="lazy"
  decoding="async"
/>

// Purely decorative image: empty alt so screen readers skip it
<img src={decorativeDivider} alt="" aria-hidden="true" />
```

---

## When to apply which optimization

| Situation | Optimization |
|---|---|
| Large page/feature that is not needed on first load | `React.lazy` + `Suspense` |
| Component in a list with many items and expensive renders | `React.memo` + `useCallback` on its handlers |
| Expensive computation on a large array | `useMemo` |
| Text input that triggers an API search | `useCallback` + `debounce` |
| Initial bundle > 200KB | Analyze with `rollup-plugin-visualizer` + more lazy loading |
| Re-renders identified in React DevTools Profiler | Identify root cause first, then memoize |
| Image above the fold causing LCP issues | `loading="eager"` + `fetchPriority="high"` |
| Images below the fold on a long page | `loading="lazy"` |
