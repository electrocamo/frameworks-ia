# frameworks-IA

Base frameworks to build AI-first solutions with React and .NET, using explicit engineering rules for humans and coding agents.

## What this repository is

This repository is a **foundation/blueprint** for two technology stacks:

- `react/`: frontend framework guidelines for React 18 + TypeScript + Vite.
- `dotnet/`: backend framework guidelines for ASP.NET Core .NET 10 — Modular Monolith with Clean Architecture per module.

It is designed so developers and AI agents can work with the same standards from day one.

## AI-first approach

The core of this repo is not only code structure; it is the **operating model**:

- Every stack has a root `AGENTS.md` with non-negotiable rules.
- Specific modules/features can override rules with local `AGENTS.md`.
- Reusable task playbooks live in `ia/skills/*/SKILL.md`.
- `.github/copilot-instructions.md` reinforces coding conventions for inline AI assistance.

In practice, developers and agents should read the relevant `AGENTS.md` before implementing anything.

## Repository structure

```text
frameworks-IA/
  react/
    AGENTS.md
    .github/copilot-instructions.md
    ia/skills/
      new-feature/
      testing/
      state-and-data/
      forms-and-validation/
      code-review/
      performance/
    src/features/
      auth/AGENTS.md
      dashboard/AGENTS.md

  dotnet/
    AGENTS.md
    .github/copilot-instructions.md
    ia/skills/
      new-feature/
      testing/
      code-review/
      ef-migrations/
      architecture-patterns/
      security-checklist/
    src/
      Api/MyApi/                          ← Host project (Program.cs, DI root)
      CommonUtils/                        ← Shared cross-cutting helpers
      Modules/
        ModuleAModule/                    ← Full module (owns data)
          ModuleA.Api/
          ModuleA.Application/
          ModuleA.Contracts/
          ModuleA.DbMigrator/
          ModuleA.Domain/
          ModuleA.Infrastructure/
          AGENTS.md
        ModuleBModule/                    ← Module with its own AGENTS.md
          ModuleB.Api/
          ModuleB.Application/
          ModuleB.Contracts/
          ModuleB.Domain/
          ModuleB.Infrastructure/
          AGENTS.md
      Tests/
        ModuleA.Test/
        ModuleB.Test/
```

## React framework (`react/`)

Defined stack and standards:

- React 18 + TypeScript strict + Vite
- Tailwind CSS v4
- TanStack Query + Zustand
- React Hook Form + Zod
- Axios service layer pattern
- Vitest + Testing Library + MSW + Playwright

Domain feature guidance includes:

- `auth`: JWT-sensitive rules (token handling, protected routes, validation).
- `dashboard`: high-volume widgets, query tuning, Suspense/ErrorBoundary patterns.

## .NET framework (`dotnet/`)

Defined stack and standards:

- ASP.NET Core .NET 10 (LTS)
- Modular Monolith with Clean Architecture per module
- MediatR + FluentValidation
- EF Core 8 + PostgreSQL 16
- Result Pattern + ProblemDetails
- Serilog + testing layers (unit, integration, architecture)

Each module follows this layer structure:

| Layer | Purpose |
|---|---|
| `{Module}.Domain` | Entities, Value Objects, domain errors |
| `{Module}.Application` | Commands, Queries, Handlers, Validators (MediatR) |
| `{Module}.Contracts` | DTOs and interfaces shared across modules |
| `{Module}.DbMigrator` | EF Core migrations for this module |
| `{Module}.Infrastructure` | EF DbContext, repositories, external services |
| `{Module}.Api` | Controllers / endpoints for this module |

Lightweight modules (no persistence) only need: `Api`, `Application`, `Contracts`.

Domain module guidance includes:

- `ModuleAModule`: cache rules, search strategy, query/pagination constraints.
- `ModuleBModule`: Stripe safety rules, idempotency, webhook validation, transaction safety.

## How to use this repository

1. Choose the stack (`react/` or `dotnet/`).
2. Read the root `AGENTS.md` for that stack.
3. If your task touches a specific module/feature, read its local `AGENTS.md`.
4. Route the task to the right skill in `ia/skills/.../SKILL.md`.
5. Implement and validate using the mandatory quality/security checklist in each stack guide.

## Quick start commands

Use these inside each stack folder.

### React

```bash
pnpm dev
pnpm type-check
pnpm lint
pnpm test:run
pnpm build
```

### .NET

```bash
docker-compose up -d
dotnet restore
dotnet build
dotnet run --project src/Api/MyApi
dotnet test
```

> Note: this repository currently acts as a framework baseline. Some runtime artifacts (for example full app scaffolding files) may be created during project bootstrap following each stack's `AGENTS.md`.

## Working agreement for teams

- Treat `AGENTS.md` as source of truth.
- Keep code in English and follow naming patterns defined per stack.
- Apply security rules by default (secret handling, validation, auth boundaries).
- Enforce quality gates before merge (type checks, lint, tests, build).
- Use Conventional Commits.

## Who should use this

- Teams standardizing React and .NET projects with consistent architecture.
- Teams using AI coding assistants and needing deterministic behavior.
- Developers onboarding into enterprise-grade patterns quickly.

## Next step

If you are starting a new implementation, begin by opening:

- `react/AGENTS.md` for frontend work.
- `dotnet/AGENTS.md` for backend work.

Then execute only the steps that match your task category and module scope.
