# Copilot Instructions — React Professional Framework

This file complements `AGENTS.md`. When an agent builds the project, these rules
apply to every line of code generated. When a developer writes code manually,
Copilot applies these same rules in every inline suggestion.

---

## Primary Rule

Follow the phases and patterns defined in `AGENTS.md`. If there is a conflict
between this file and `AGENTS.md`, **`AGENTS.md` takes precedence**.

---

## Mandatory Stack

| Concern | Library |
|---|---|
| Framework | React 18 + TypeScript strict |
| Build tool | Vite |
| Routing | React Router v6 |
| Server state | TanStack Query v5 |
| Client state | Redux Toolkit + React Redux |
| Schema validation | Zod |
| Forms | React Hook Form + @hookform/resolvers |
| HTTP client | Axios (wrapped in `src/api/httpInterceptor.ts`) |
| Styles | Tailwind CSS v4 |
| Unit/integration tests | Vitest + React Testing Library + MSW |
| E2E tests | Playwright |

---

## TypeScript — Strict Rules

```typescript
// ✅ Correct — interface for domain objects
interface User {
  id: string
  name: string
  email: string
  role: 'admin' | 'user' | 'viewer'
}

// ✅ Correct — type for unions and primitives
type Theme = 'light' | 'dark'
type ApiStatus = 'idle' | 'loading' | 'success' | 'error'

// ✅ Correct — Zod schema as source of truth, type derived from it
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'viewer']),
})
type User = z.infer<typeof UserSchema>  // no separate interface needed

// ✅ Correct — unknown + type guard instead of any
function processApiResponse(raw: unknown): User {
  return UserSchema.parse(raw) // throws ZodError if invalid
}

// ❌ Never — any type anywhere
const data: any = response.data
function process(input: any) { }

// ❌ Never — @ts-ignore or @ts-expect-error without justification
// @ts-ignore
const result = badFunction()

// ❌ Never — non-null assertion without a comment explaining why it's safe
const element = document.getElementById('app')!
// ✅ Correct version:
const element = document.getElementById('app')
if (!element) throw new Error('Root element #app not found in DOM')
```

---

## Components — Rules

```typescript
// ✅ Correct — named export, logic in hook, all states handled
export const UserList = () => {
  const { data: users, isLoading, isError, error } = useUsers()

  if (isLoading) return <UserListSkeleton />
  if (isError)   return <ErrorMessage title="Failed to load users" message={error.message} />
  if (!users?.length) return <EmptyState title="No users" description="No users found." />

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

// ❌ Never — default export
export default function UserList() { }

// ❌ Never — fetch inside useEffect
export const UserList = () => {
  const [users, setUsers] = useState([])
  useEffect(() => {
    fetch('/api/users').then(r => r.json()).then(setUsers)
  }, [])
  // no loading state, no error state, no empty state
}

// ❌ Never — complex logic inline in JSX
return (
  <ul>
    {users
      .filter(u => u.role !== 'viewer')
      .sort((a, b) => a.name.localeCompare(b.name))
      .map(u => <li key={u.id}>{u.orders.reduce((s, o) => s + o.total, 0)}</li>)
    }
  </ul>
)
// ✅ Move the transformation into useMemo or a derived variable in the hook
```

---

## Services — Rules

```typescript
// ✅ Correct — all API calls in the service layer, Zod validation on response
export const UserService = {
  getAll: async (): Promise<User[]> => {
    const { data } = await apiClient.get('/users')
    return UserListSchema.parse(data)  // validates and types the response
  },

  create: async (input: CreateUserInput): Promise<User> => {
    const { data } = await apiClient.post('/users', input)
    return UserSchema.parse(data)
  },
}

// ❌ Never — axios or fetch called directly outside src/api/ and approved service orchestrators
const { data } = await axios.get('https://api.example.com/users')  // in a component
fetch('/api/users').then(...)                                       // in a hook
```

---

## Hooks — Rules

```typescript
// ✅ Correct — one responsibility per hook, descriptive name
export const useUsers = () => {
  return useQuery({
    queryKey: userKeys.lists(),
    queryFn: UserService.getAll,
    staleTime: 5 * 60 * 1000,
  })
}

export const useCreateUser = () => {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: UserService.create,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: userKeys.lists() }),
  })
}

// ✅ Query key factories — typed constants, never inline strings
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  detail: (id: string) => [...userKeys.all, 'detail', id] as const,
}

// ❌ Never — mixing server logic and UI logic in the same hook
export const useUsersWithModal = () => {
  const query = useQuery({ queryKey: ['users'], queryFn: UserService.getAll })
  const [isModalOpen, setIsModalOpen] = useState(false)  // unrelated UI state
  return { ...query, isModalOpen, setIsModalOpen }
}
```

---

## State Management — Decision Guide

```typescript
// Server data (from API) → TanStack Query
const { data: users } = useUsers()           // list from server
const { data: product } = useProduct(id)     // detail from server

// Authenticated user → Redux store (in memory, never localStorage)
const user = useAppSelector((state) => state.auth.user)
const dispatch = useAppDispatch()
const logout = () => dispatch(authActions.logout())

// Local UI state (modal, toggle) → useState
const [isOpen, setIsOpen] = useState(false)

// UI preferences persisted across sessions → Redux + persistence strategy
const theme = useAppSelector((state) => state.ui.theme)

// ❌ Never — server state in Redux
const users = useAppSelector((state) => state.users.list) // use TanStack Query instead

// ❌ Never — access token in localStorage
localStorage.setItem('token', accessToken)  // store in Redux memory only
```

---

## Forms — Rules

```typescript
// ✅ Correct — Zod schema → RHF resolver → derived type
const LoginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Minimum 8 characters'),
})
type LoginValues = z.infer<typeof LoginSchema>

export const LoginForm = () => {
  const { register, handleSubmit, formState: { errors } } = useForm<LoginValues>({
    resolver: zodResolver(LoginSchema),
  })

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        aria-describedby={errors.email ? 'email-error' : undefined}
        aria-invalid={Boolean(errors.email)}
        {...register('email')}
      />
      {errors.email && (
        <p id="email-error" role="alert">{errors.email.message}</p>
      )}
    </form>
  )
}

// ❌ Never — manual validation without schema
const [email, setEmail] = useState('')
const [emailError, setEmailError] = useState('')
const validate = () => {
  if (!email.includes('@')) setEmailError('Invalid email')  // use Zod instead
}
```

---

## Accessibility — Minimum Standards

```typescript
// ✅ Interactive elements always have a label or aria-label
<button type="button" aria-label="Delete user John Doe">
  <TrashIcon aria-hidden="true" />
</button>

// ✅ Form fields always have an associated label
<label htmlFor="search">Search products</label>
<input id="search" type="search" />

// ✅ Landmark roles on layout sections
<main>
  <nav aria-label="Main navigation">...</nav>
  <section aria-labelledby="products-heading">
    <h2 id="products-heading">Products</h2>
  </section>
</main>

// ✅ Error messages announced to screen readers
<p id="email-error" role="alert">{errors.email?.message}</p>

// ✅ Loading states communicated to assistive technology
<div aria-live="polite" aria-busy={isLoading}>
  {isLoading ? <Spinner /> : <UserList />}
</div>

// ❌ Never — icon buttons without accessible label
<button onClick={handleDelete}>
  <TrashIcon />   {/* invisible to screen readers */}
</button>

// ❌ Never — placeholder as sole label
<input type="email" placeholder="Enter your email" />
```

---

## Environment Variables — Rules

```typescript
// ✅ Access import.meta.env only in centralized infra files
const url = import.meta.env.VITE_API_URL

// ❌ Never — access import.meta.env inside feature components
const url = import.meta.env.VITE_API_URL   // in a component
const key = import.meta.env.VITE_API_KEY   // in a service
```

---

## Commits — Conventional Commits (Mandatory)

```bash
# Format: type(scope): description in lowercase
feat(auth): add password reset flow
fix(products): correct pagination offset calculation
test(users): add missing empty state test for UserList
refactor(api-client): extract refresh token logic into helper
chore(deps): update tanstack-query to v5.40
docs(readme): update local setup instructions
perf(dashboard): add useMemo to metrics transformation
a11y(forms): add aria-describedby to all form error messages
```

**Types:** `feat` | `fix` | `test` | `refactor` | `chore` | `docs` | `perf` | `a11y` | `build` | `ci`

---

## Absolute Prohibitions

Copilot and agents must never suggest:

- ❌ `any` type — use `unknown` + type guards, or generics
- ❌ `// @ts-ignore` or `// eslint-disable` — fix the root cause
- ❌ Default exports — named exports only
- ❌ `fetch` or `axios` outside `src/api/` and approved service orchestrators
- ❌ `import.meta.env` outside centralized infra files (e.g. `src/api/httpInterceptor.ts`)
- ❌ `localStorage` for access tokens — use Redux in-memory store
- ❌ API response data used without Zod validation
- ❌ Components without loading + error + empty state handling
- ❌ `useEffect` for data fetching — use TanStack Query
- ❌ Hardcoded URLs, tokens, or environment values in source files
- ❌ `console.log` in code committed to the repository
- ❌ Props drilling beyond 2 levels — use Redux or Context
- ❌ Class components — functional components with hooks only
