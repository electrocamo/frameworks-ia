# AGENTS.md

> Read this file **in full** before writing any line of code.
> This is the single source of truth for the project. If anything in the
> codebase contradicts this file, follow this file and fix the code.

---

## What is this project?

SPA built with **React 18 + TypeScript strict + Vite**.
Consumes external REST APIs with JWT authentication.

- Code language: **English**
- Package manager: **yarn**
- Styles: **Tailwind CSS v4**
- Deploy target: **azure** 

---

## ⚠️ MANDATORY INSTRUCTION — Skill Routing

**Before writing any code**, identify your task in the list below.
If it matches, **STOP** and read the corresponding skill first. This is not optional.

```
Does your task involve...?

  Creating a component, hook, service, or feature end-to-end
  → READ FIRST: ia/skills/new-feature/SKILL.md

  Writing, fixing, or improving tests (unit, integration, e2e)
  → READ FIRST: ia/skills/testing/SKILL.md

  Global state (Zustand), server state (TanStack Query), or caching
  → READ FIRST: ia/skills/state-and-data/SKILL.md

  Forms, Zod validation, or Formik
  → READ FIRST: ia/skills/forms-and-validation/SKILL.md

  Reviewing a PR, auditing code, or performing a code review
  → READ FIRST: ia/skills/code-review/SKILL.md

  Performance optimization, lazy loading, memoization, or bundle size
  → READ FIRST: ia/skills/performance/SKILL.md
```

If your task is not in the list, continue with the rules in this file.
If your task touches **more than one category**, read all applicable skills before starting.

---

## Phase 0 — Before writing code (ALWAYS)

```bash
# 1. Read this entire AGENTS.md
# 2. Analyze the existing project structure
find . -type f \
  -not -path '*/node_modules/*' \
  -not -path '*/.git/*' \
  -not -path '*/dist/*' | head -60

# 3. Read package.json to understand current dependencies
cat package.json

# 4. Confirm the task scope — never assume, never guess
```

---

## Project Scaffolding — Initial Setup

When creating the project from scratch, execute in this exact order:

```bash
# 1. Initialize with Vite
yarn create vite@latest . -- --template react-ts
yarn install

# 2. Install all dependencies at once — never one by one
yarn add axios zod redux @tanstack/react-query react-router-dom \
         formik

yarn add -D vitest @vitest/coverage-v8 @testing-library/react \
         @testing-library/user-event @testing-library/jest-dom \
         msw @playwright/test \
         eslint @eslint/js typescript-eslint \
         eslint-plugin-react eslint-plugin-react-hooks \
         eslint-plugin-jsx-a11y prettier eslint-config-prettier \
         husky lint-staged @commitlint/cli @commitlint/config-conventional \
         tailwindcss @tailwindcss/vite

# 3. Create the full folder structure
mkdir -p src/{app,components/{ui,features},features,hooks,services,store,types,utils,config,lib}
mkdir -p src/tests/{mocks/fixtures,utils}
mkdir -p ia/skills
mkdir -p .github/workflows

# 4. Set up git hooks
yarn exec husky init
echo "yarn exec lint-staged" > .husky/pre-commit
echo "yarn exec commitlint --edit \$1" > .husky/commit-msg
```

---

## Essential Commands

```bash
# Development
yarn dev
yarn build
yarn preview

# Code quality — all four must pass before every commit
yarn type-check    # zero TypeScript errors
yarn lint          # zero ESLint warnings or errors
yarn test:run      # all tests green
yarn build         # successful production build

# Tests
yarn test          # watch mode (development)
yarn test:run      # single run (CI)
yarn test:coverage # with coverage report
yarn test:e2e      # Playwright end-to-end

# Formatting
yarn format        # run prettier on src/
yarn format:check  # verify formatting without modifying files
```

---

## Project Structure

```
src/
  app/
    router.tsx          ← route definitions (React Router)
    providers.tsx        ← global provider composition
  components/
    ui/                  ← atomic, reusable components (Button, Input, Modal)
    features/            ← feature-specific components
  features/
    [feature-name]/
      components/        ← components scoped to this feature
      hooks/             ← hooks scoped to this feature
      services/          ← API calls for this feature
      store/             ← Zustand store (only if global state is needed)
      types/             ← TypeScript types for this feature
      index.ts           ← barrel export (public API of the feature)
  hooks/                 ← shared, reusable hooks
  services/
    api-client.ts        ← configured axios instance
  store/                 ← global Zustand stores
  types/                 ← global and shared TypeScript types
  utils/                 ← pure utility functions
  config/
    env.ts               ← environment variables validated with Zod
  lib/
    query-client.ts      ← TanStack Query client configuration
  tests/
    setup.ts             ← Vitest global setup
    mocks/
      handlers.ts        ← MSW request handlers
      server.ts          ← MSW server instance
      fixtures/          ← reusable mock data factories
    utils/
      test-utils.tsx     ← custom render with all providers

ia/skills/             ← specialized skills — see routing instruction above
```

---

## Config Files — Exact Content

### `tsconfig.json`
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] },
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "skipLibCheck": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### `vite.config.ts`
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import path from 'path'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
})
```

### `vitest.config.ts`
```typescript
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: ['node_modules', 'src/tests', '**/*.d.ts', '**/*.config.*'],
      thresholds: {
        global: { branches: 80, functions: 85, lines: 85, statements: 85 },
      },
    },
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
})
```

### `src/config/env.ts`
```typescript
// Single entry point for all environment variables.
// Throws at startup if any required variable is missing or invalid.
import { z } from 'zod'

const EnvSchema = z.object({
  VITE_API_URL: z.string().url('VITE_API_URL must be a valid URL'),
  VITE_APP_ENV: z.enum(['development', 'staging', 'production']),
  VITE_APP_VERSION: z.string().default('0.0.1'),
})

const parsed = EnvSchema.safeParse(import.meta.env)

if (!parsed.success) {
  console.error('Invalid environment variables:', parsed.error.flatten().fieldErrors)
  throw new Error('Invalid environment variables. Check .env.example for required variables.')
}

export const env = parsed.data

// ✅ Correct usage anywhere in the app:
// import { env } from '@/config/env'
// const url = env.VITE_API_URL

// ❌ Never access import.meta.env directly outside this file
```

### `.env.example`
```bash
VITE_API_URL=https://api.example.com
VITE_APP_ENV=development
VITE_APP_VERSION=0.0.1
```

### `package.json` scripts
```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "type-check": "tsc --noEmit",
    "lint": "eslint . --ext .ts,.tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,css,md}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,css,md}\"",
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
}
```

---

## Code Conventions

### TypeScript — non-negotiable rules
- Zero `any` — use `unknown` with type guards, or generics
- Zero `// @ts-ignore` or `// @ts-expect-error` without a justifying comment
- Zero `// eslint-disable` without a justifying comment
- Interfaces for domain objects; `type` for unions and primitives
- Zod to validate **every** API response before using it in the app
- **Named exports only** — zero default exports anywhere

### Components
- Functional components only — zero class components
- Maximum 150 lines per component — decompose if exceeded
- Logic in hooks, presentation in components
- Always handle the three states: `loading`, `error`, `empty`
- Semantic HTML and `aria-*` attributes on interactive elements

### Data fetching
- Zero `fetch` or `axios` calls directly in components or `useEffect`
- All API calls go through `src/services/api-client.ts`
- All server queries use TanStack Query (`useQuery` / `useMutation`)

**Forbidden patterns:**
```typescript
// ❌ any type
const data: any = response.data

// ❌ fetch directly in component
useEffect(() => { fetch('/api/users').then(...) }, [])

// ❌ axios outside the service layer
const { data } = await axios.get('https://api.example.com/users')

// ❌ hardcoded URL
const API = 'https://api.example.com'

// ❌ default export
export default function UserCard() {}

// ❌ hardcoded secret
const TOKEN = 'sk-live-abc123'

// ❌ console.log in non-development code
console.log(userData)

// ❌ import.meta.env outside config/env.ts
const url = import.meta.env.VITE_API_URL
```

---

## Security — Non-Negotiable Rules

- Zero API keys, tokens, or secrets in any tracked file
- `.env.local` always in `.gitignore`
- Every environment variable documented in `.env.example` with fake values
- All external data validated with **Zod** before use in the application
- `src/config/env.ts` is the **only** access point to `import.meta.env`
- Access token lives **in memory only** (Zustand store) — never in `localStorage`
- Refresh token lives in `httpOnly` cookie — never accessible via JavaScript

---

## Git Conventions

**Commit format (Conventional Commits — mandatory):**
```
feat(auth): add login form with JWT validation
fix(api): handle timeout in response interceptor
test(users): add edge cases for useUser hook
refactor(components): extract Button into ui library
chore(deps): update vite to v6
```

**Branch strategy:**
```
main        ← production only, protected
develop     ← integration branch
feature/*   ← new features
fix/*       ← bug fixes
chore/*     ← config, deps, tooling
```

**Pre-commit sequence — all four must pass:**
```bash
yarn type-check && yarn lint && yarn test:run && yarn build
```

---

## Coverage Gates — CI fails if below these thresholds

| Metric | Minimum |
|---|---|
| Lines | 85% |
| Functions | 85% |
| Branches | 80% |
| Statements | 85% |

---

## CI/CD — `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: yarn
      - run: yarn install --frozen-lockfile
      - run: yarn type-check
      - run: yarn lint
      - run: yarn test:run -- --coverage
      - run: yarn build
      - uses: codecov/codecov-action@v4
```

---

## Delivery Checklist

Do not mark a task as complete until all applicable items are green:

```
CODE
[ ] yarn type-check → zero errors
[ ] yarn lint → zero warnings
[ ] No any, @ts-ignore, or eslint-disable without justification
[ ] No fetch/axios outside the service layer
[ ] No hardcoded URLs or tokens
[ ] Named exports only

COMPONENTS
[ ] Handles loading + error + empty states
[ ] Maximum 150 lines
[ ] Semantic HTML and basic accessibility
[ ] Logic extracted to custom hooks

TESTS
[ ] yarn test:run → fully green
[ ] Tests for each state: loading, success, error, empty
[ ] Coverage ≥ 85% on new code

SECURITY
[ ] No secrets in tracked files
[ ] External data validated with Zod

GIT
[ ] Commit in Conventional Commits format
[ ] yarn build → successful build
```
