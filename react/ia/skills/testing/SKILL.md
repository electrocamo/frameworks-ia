---
name: testing
description: >
  Read when the task is to write, fix, or review tests.
  Covers unit tests with Vitest + React Testing Library + MSW,
  integration tests, and end-to-end tests with Playwright.
triggers:
  - write tests
  - unit test
  - integration test
  - e2e test
  - coverage
  - MSW
  - Vitest
  - Playwright
  - testing-library
---

# SKILL: Testing Strategy

## Pyramid and minimum coverage

```
         ▲
        /E2E\        10%   Playwright — critical user flows end-to-end
       /──────\
      / Integr \    30%   RTL + MSW — components with real query lifecycle
     /──────────\
    / Unit Tests \  60%   RTL + MSW — components and hooks in isolation

Minimum coverage thresholds (CI fails below these):
  Lines:      85%
  Functions:  85%
  Branches:   80%
  Statements: 85%
```

---

## Global Setup

```typescript
// src/tests/setup.ts
import '@testing-library/jest-dom'
import { afterAll, afterEach, beforeAll } from 'vitest'
import { server } from './mocks/server'

// MSW intercepts all HTTP calls in tests
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())  // clear per-test overrides
afterAll(() => server.close())
```

```typescript
// src/tests/mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'
export const server = setupServer(...handlers)
```

```typescript
// src/tests/mocks/handlers.ts
import { http, HttpResponse } from 'msw'
import { mockUsers } from './fixtures/users'

// Default handlers — used in all tests unless overridden
export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json(mockUsers)
  }),

  http.get('/api/users/:id', ({ params }) => {
    const user = mockUsers.find(u => u.id === params.id)
    if (!user) return new HttpResponse(null, { status: 404 })
    return HttpResponse.json(user)
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json()
    return HttpResponse.json({ id: 'new-id-123', ...body }, { status: 201 })
  }),
]
```

```typescript
// src/tests/utils/test-utils.tsx
import { render, type RenderOptions } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { MemoryRouter } from 'react-router-dom'
import type { ReactElement } from 'react'

// Test QueryClient — no retry, no cache between tests
const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: 0 },
      mutations: { retry: false },
    },
  })

const AllProviders = ({ children }: { children: React.ReactNode }) => {
  const queryClient = createTestQueryClient()
  return (
    <MemoryRouter>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </MemoryRouter>
  )
}

// Use this custom render instead of @testing-library/react directly
const customRender = (ui: ReactElement, options?: RenderOptions) =>
  render(ui, { wrapper: AllProviders, ...options })

export * from '@testing-library/react'
export { customRender as render }

// Wrapper for renderHook with providers
export const QueryWrapper = ({ children }: { children: React.ReactNode }) => {
  const queryClient = createTestQueryClient()
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
}

// Helper for tests requiring an authenticated user
export const createAuthenticatedWrapper = () => {
  return ({ children }: { children: React.ReactNode }) => {
    // Set auth state before rendering
    useAuthStore.setState({ user: mockAuthUser, accessToken: 'mock-token' })
    return <AllProviders>{children}</AllProviders>
  }
}
```

---

## Component Tests — mandatory structure

```typescript
// src/features/users/components/UserList.test.tsx
import { render, screen, waitFor } from '@/tests/utils/test-utils'
import { server } from '@/tests/mocks/server'
import { http, HttpResponse } from 'msw'
import { UserList } from './UserList'

describe('UserList', () => {
  // ✅ Loading state
  it('renders skeleton while loading', () => {
    render(<UserList />)
    // The skeleton is visible before the query resolves
    expect(screen.getByTestId('user-list-skeleton')).toBeInTheDocument()
  })

  // ✅ Happy path
  it('renders user cards when data loads successfully', async () => {
    render(<UserList />)

    // Wait for async query to resolve
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument()
      expect(screen.getByText('Jane Smith')).toBeInTheDocument()
    })
  })

  // ✅ Error state
  it('renders error message when API returns 500', async () => {
    // Override the default handler for this specific test
    server.use(
      http.get('/api/users', () =>
        new HttpResponse(null, { status: 500 })
      )
    )

    render(<UserList />)

    await waitFor(() => {
      expect(screen.getByText('Failed to load users')).toBeInTheDocument()
    })
  })

  // ✅ Empty state
  it('renders empty state when no users exist', async () => {
    server.use(
      http.get('/api/users', () => HttpResponse.json([]))
    )

    render(<UserList />)

    await waitFor(() => {
      expect(screen.getByText('No users found')).toBeInTheDocument()
    })
  })
})
```

---

## Hook Tests

```typescript
// src/features/users/hooks/useUsers.test.ts
import { renderHook, waitFor } from '@testing-library/react'
import { server } from '@/tests/mocks/server'
import { http, HttpResponse } from 'msw'
import { QueryWrapper } from '@/tests/utils/test-utils'
import { useUsers } from './useUsers'

describe('useUsers', () => {
  it('returns user data on success', async () => {
    const { result } = renderHook(() => useUsers(), {
      wrapper: QueryWrapper,
    })

    // Initial state: loading
    expect(result.current.isLoading).toBe(true)

    // Wait for resolution
    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true)
    })

    expect(result.current.data).toHaveLength(2)
    expect(result.current.data?.[0].name).toBe('John Doe')
  })

  it('returns error state when API fails', async () => {
    server.use(
      http.get('/api/users', () => new HttpResponse(null, { status: 500 }))
    )

    const { result } = renderHook(() => useUsers(), {
      wrapper: QueryWrapper,
    })

    await waitFor(() => {
      expect(result.current.isError).toBe(true)
    })
  })
})
```

---

## Mutation Tests

```typescript
// src/features/users/hooks/useCreateUser.test.ts
import { renderHook, act, waitFor } from '@testing-library/react'
import { QueryWrapper } from '@/tests/utils/test-utils'
import { useCreateUser } from './useUsers'

describe('useCreateUser', () => {
  it('creates a user and returns its data on success', async () => {
    const { result } = renderHook(() => useCreateUser(), {
      wrapper: QueryWrapper,
    })

    act(() => {
      result.current.mutate({
        name: 'New User',
        email: 'new@example.com',
        role: 'user',
      })
    })

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true)
    })

    expect(result.current.data?.name).toBe('New User')
  })
})
```

---

## E2E Tests with Playwright

```typescript
// src/tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Authentication flow', () => {
  test('logs in and redirects to dashboard', async ({ page }) => {
    await page.goto('/login')

    await page.getByLabel('Email').fill('user@example.com')
    await page.getByLabel('Password').fill('password123')
    await page.getByRole('button', { name: 'Sign in' }).click()

    await expect(page).toHaveURL('/dashboard')
    await expect(page.getByText('Welcome back')).toBeVisible()
  })

  test('shows error on invalid credentials', async ({ page }) => {
    await page.goto('/login')

    await page.getByLabel('Email').fill('wrong@example.com')
    await page.getByLabel('Password').fill('wrongpassword')
    await page.getByRole('button', { name: 'Sign in' }).click()

    await expect(page.getByText('Invalid credentials')).toBeVisible()
    await expect(page).toHaveURL('/login') // did not redirect
  })

  test('redirects to login when accessing protected route unauthenticated', async ({ page }) => {
    await page.goto('/dashboard')
    await expect(page).toHaveURL('/login')
  })
})
```

---

## Testing Rules

| ✅ Always do | ❌ Never do |
|---|---|
| Test observable behavior (text, roles, events) | Test internal implementation (state, direct props) |
| Use `getByRole`, `getByLabelText`, `getByText` | Use `getByTestId` except as a last resort |
| Cover: loading + success + error + empty | Write a test that only checks "renders without crashing" |
| Use fixture factories for test data | Inline hardcoded objects inside each test |
| MSW to intercept HTTP calls | `vi.mock()` entire service modules |
| `waitFor` for async assertions | Manual `act` calls for TanStack Query |
| Import `render` from test-utils, not RTL directly | Import from `@testing-library/react` directly |

---

## Coverage Commands

```bash
pnpm test:coverage

# HTML report is generated at coverage/index.html
# Thresholds are configured in vitest.config.ts
# CI fails if any threshold is not met
```
