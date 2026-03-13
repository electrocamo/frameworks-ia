---
name: code-review
description: >
  Read when the task is to review a PR, audit existing code, or validate
  that an implementation meets project standards.
triggers:
  - code review
  - review PR
  - review code
  - pull request
  - audit
  - validate implementation
---

# SKILL: Code Review — React

## Review order — most critical to least critical

```
1. Security       — hardcoded secrets, sensitive data in logs, XSS
2. Correctness    — wrong logic, async bugs, unhandled edge cases
3. Architecture   — fetch in components, logic in JSX, wrong layers
4. Tests          — state coverage, assertion quality
5. Performance    — unnecessary re-renders, missing memoization
6. Accessibility  — roles, labels, aria attributes
7. Conventions    — naming, structure, TypeScript strict
```

---

## 🚨 Red Flags — Block the PR without exception

```typescript
// HARDCODED SECRET
const API_KEY = 'sk-live-abc123'
Authorization: 'Bearer hardcoded-token-here'

// FETCH DIRECTLY IN COMPONENT
useEffect(() => {
  fetch('/api/users').then(r => r.json()).then(setUsers)
}, [])

// AXIOS OUTSIDE API LAYER
const { data } = await axios.get('https://api.example.com/users')

// ANY TYPE IN TYPESCRIPT
const response: any = await api.get('/users')
const props: any = { ... }

// DEFAULT EXPORT
export default function UserCard() {}

// NOT HANDLING LOADING OR ERROR STATES
const { data } = useUsers()
return <ul>{data.map(...)}</ul>  // data may be undefined

// API DATA USED WITHOUT ZOD VALIDATION
const user = response.data  // no .parse() or .safeParse()

// CONSOLE.LOG IN NON-DEVELOPMENT CODE
console.log(user.email, user.password)

// COMPLEX LOGIC INLINE IN JSX
return (
  <div>
    {users
      .filter(u => u.active)
      .map(u => u.orders.reduce(...))
      .sort(...)}
  </div>
)

// HARDCODED URL
const url = 'https://api.production.com/users'

// import.meta.env ACCESSED DIRECTLY OUTSIDE API INFRA FILES
const apiUrl = import.meta.env.VITE_API_URL
```

---

## Full Review Checklist

### Security
```
[ ] No secrets or tokens in source files
[ ] No console.log with user or sensitive data
[ ] All external data validated with Zod before use
[ ] Environment variables accessed only in centralized infra files (e.g. src/api/httpInterceptor.ts)
```

### Correctness
```
[ ] Handles loading + error + empty in every component with async data
[ ] No race condition in useEffect (cleanup function present if needed)
[ ] useMutation has onError handled
[ ] enabled: false on queries that depend on optional/async params
[ ] Zod parse errors are caught and handled (not swallowed silently)
```

### Architecture
```
[ ] Zero fetch/axios calls outside src/api/ and approved service orchestrators
[ ] Zero business logic inside JSX
[ ] Named exports only — zero default exports
[ ] Logic in custom hooks, presentation in components
[ ] Components are max 150 lines
[ ] Imports come from feature barrel (index.ts), not internal paths
```

### Tests
```
[ ] Tests for loading, success, error, and empty states
[ ] Uses render from test-utils.tsx, not from @testing-library/react directly
[ ] MSW used to intercept HTTP — not vi.mock() of service modules
[ ] Assertions on observable behavior, not implementation details
[ ] No tests that only check "renders without crashing"
```

### Performance
```
[ ] useCallback on handlers passed as props to memoized child components
[ ] useMemo on expensive array transformations
[ ] React.memo on list item components
[ ] Lazy loading used for page-level components
[ ] No stale closure bugs in useEffect, useMemo, or useCallback deps
```

### Accessibility
```
[ ] Every input has an associated <label> via htmlFor
[ ] Form error messages have role="alert" and aria-describedby
[ ] Images have descriptive alt text (or alt="" if purely decorative)
[ ] Buttons have descriptive text or aria-label
[ ] Modals trap focus and restore it on close
```

---

## Comment Types for Reviews

```markdown
🚨 BLOCKING: There is a fetch call directly inside the component.
Move the call to src/api/apiUsers.ts (or the feature service orchestrator)
and access the data through a useUsers() hook backed by TanStack Query.

💡 SUGGESTION: Consider useCallback here — onSelectUser is recreated
on every render and is passed as a prop to UserCard which is wrapped in
React.memo. Not a blocker but prevents unnecessary re-renders.

❓ QUESTION: Why is disabled={!isValid && !isDirty} used here?
The other forms in the project only check !isValid.
If this is intentional, add a comment explaining the difference.

🔧 NIT: The component uses a default export.
Change to a named export for consistency with the rest of the project.
```

**Actionable feedback — always include the fix:**
```
❌ Vague: "This code is not correct."
✅ Actionable: "🚨 BLOCKING: useEffect without a cleanup function can cause
a state update on an unmounted component.
Add a cleanup:
  useEffect(() => {
    let cancelled = false
    fetchData().then(d => { if (!cancelled) setData(d) })
    return () => { cancelled = true }
  }, [])
Or better: move this to a useQuery hook."
```

---

## What Every New Feature PR Must Include

Request these before approving if any are missing:

1. **Tests for all 4 states**: loading, success, error, empty
2. **Named export** (no default export)
3. **Zod parse** on every API response the component depends on
4. **aria-** attributes on all interactive elements
5. **Logic extracted** to a custom hook — not inline in the component
6. A test verifying behavior in the **error state**
