# AGENTS.md — Feature: Auth

> Extends the root `AGENTS.md`. Read that file first, then this one.
> Rules here **override** the root document for this feature only.

---

## Context

Authentication module using JWT Bearer tokens.
Every operation here is security-sensitive — apply extra care before any change.

---

## ⚠️ Additional Skills for This Feature

```
Are you modifying the login or registration form?
→ READ FIRST: ia/skills/forms-and-validation/SKILL.md

Are you modifying the authenticated user state?
→ READ FIRST: ia/skills/state-and-data/SKILL.md
```

---

## Auth-Specific Rules

**Token storage:**
- Access token lives **in memory only** (`useAuthStore`) — never in `localStorage` or `sessionStorage`
- Refresh token lives in an `httpOnly` cookie — never accessible from JavaScript
- Never log tokens, even partially
- The axios interceptor handles token refresh automatically — do not duplicate that logic

**Protected routes:**
- Every authenticated route uses the `<ProtectedRoute>` component — never inline auth logic
- Post-login redirect targets the originally requested URL, not always `/dashboard`

**Validation:**
- Zod schemas defined in `src/features/auth/types/auth.schemas.ts`
- The same schemas are used in both the form (React Hook Form) and the API response validation

**Auth tests:**
- Use `createAuthenticatedWrapper()` from `src/tests/utils/test-utils.tsx`
  for tests that require an authenticated user
- Never hardcode tokens in tests — use the `mockAuthToken` helper

---

## Feature Structure

```
src/features/auth/
  components/
    LoginForm.tsx
    LoginForm.test.tsx
    ProtectedRoute.tsx
    ProtectedRoute.test.tsx
  hooks/
    useAuth.ts
    useAuth.test.ts
  services/
    auth.service.ts
    auth.service.test.ts
  store/
    auth.store.ts         ← access token in memory here
  types/
    auth.types.ts
    auth.schemas.ts       ← shared Zod schemas
  index.ts
```
