# AGENTS.md

> Read this file **in full** before writing any line of code.
> This is the single source of truth for the project. If anything in the codebase
> contradicts this file, follow this file and fix the code.

---

## What is this project?

Enterprise REST API built with **ASP.NET Core .NET 8 (LTS)** following **Clean Architecture**.
Multiple business domains organized as modules inside a modular monolith.

- Code language: **English**
- Database: **PostgreSQL 16**
- Authentication: **JWT Bearer**

---

## ⚠️ MANDATORY INSTRUCTION — Skill Routing

**Before writing any code**, identify your task in the list below.
If it matches, **STOP** and read the corresponding skill first. This is not optional.

```
Does your task involve...?

  Creating, modifying, reverting, or debugging an EF Core migration
  → READ FIRST: ia/skills/ef-migrations/SKILL.md

  Adding a new endpoint, use case, or full feature end-to-end
  → READ FIRST: ia/skills/new-feature/SKILL.md

  Writing, fixing, or improving tests (unit, integration, or architecture)
  → READ FIRST: ia/skills/testing/SKILL.md

  Reviewing a PR, auditing existing code, or performing a code review
  → READ FIRST: ia/skills/code-review/SKILL.md

  Implementing authentication, authorization, secrets, rate limiting,
  or reviewing security
  → READ FIRST: ia/skills/security-checklist/SKILL.md

  Implementing Result Pattern, CQRS, Repository, Value Objects,
  Options Pattern, or Saga Pattern
  → READ FIRST: ia/skills/architecture-patterns/SKILL.md
```

If your task is not in the list, continue with the rules in this file.
If your task touches **more than one category**, read all applicable skills before starting.

---

## Essential Commands

```bash
# Start local infrastructure (PostgreSQL + Seq + Redis)
docker-compose up -d

# Development cycle
dotnet restore
dotnet build
dotnet run --project src/MyApi.Api

# Tests
dotnet test                                           # all tests
dotnet test tests/MyApi.UnitTests                    # unit only (no Docker)
dotnet test tests/MyApi.IntegrationTests             # integration (requires Docker)
dotnet test tests/MyApi.ArchTests                    # architecture rules

# Migrations — see ia/skills/ef-migrations/SKILL.md for the full process
dotnet ef migrations add <Name> --project src/MyApi.Infrastructure --startup-project src/MyApi.Api
dotnet ef database update --project src/MyApi.Infrastructure --startup-project src/MyApi.Api

# Local secrets (first time only, per machine)
dotnet user-secrets set "Jwt:Secret" "dev-key-min-32-chars-local-only!!" --project src/MyApi.Api
dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;Database=myapi_dev;Username=postgres;Password=dev" --project src/MyApi.Api

# Check for vulnerable packages
dotnet list package --vulnerable --include-transitive
```

---

## Project Structure

```
src/
  MyApi.Domain/          ← Entities, Value Objects, domain errors. ZERO external dependencies.
  MyApi.Application/     ← Commands, Queries, Handlers, Validators, Behaviors (MediatR)
  MyApi.Infrastructure/  ← EF Core, repositories, external services, migrations
  MyApi.Api/             ← Controllers, Middleware, Program.cs

tests/
  MyApi.UnitTests/       ← Domain + Application. No real I/O.
  MyApi.IntegrationTests/← HTTP endpoints with Testcontainers + WebApplicationFactory
  MyApi.ArchTests/       ← Architecture rules with NetArchTest

ia/skills/             ← Specialized skills — read per the routing instruction above
```

**Dependency rule — NEVER violate:**
```
Domain ← Application ← Infrastructure
                     ← Api  (DI registration only)
```
- `Domain` references nothing external
- `Application` references `Domain` only
- `Infrastructure` references `Application` + `Domain`
- `Api` references `Application` + `Infrastructure` (DI registration only)

> If a module has its own `AGENTS.md`, read it after this file.
> Its rules override this document for that specific module.

---

## Code Conventions

| Element | Pattern | Example |
|---|---|---|
| Classes, public methods | PascalCase | `ProductRepository` |
| Interfaces | `I` + PascalCase | `IProductRepository` |
| Async methods | `Async` suffix | `GetByIdAsync` |
| Private fields | `_camelCase` | `_repository` |
| Input DTOs | `Request` suffix | `CreateProductRequest` |
| Output DTOs | `Response` suffix | `ProductResponse` |
| MediatR Commands | `Command` suffix | `CreateProductCommand` |
| MediatR Queries | `Query` suffix | `GetProductByIdQuery` |
| Handlers | `Handler` suffix | `CreateProductCommandHandler` |
| Validators | `Validator` suffix | `CreateProductCommandValidator` |
| Test names | `Method_Scenario_ExpectedResult` | `Handle_WhenNotFound_ReturnsError` |

**Always apply:**
- `CancellationToken` in every method with I/O
- `sealed` on handlers and validators (not designed for inheritance)
- `record` for Commands, Queries, and DTOs (immutable by default)
- `using` / `await using` on every `IDisposable`
- **Result Pattern** for business errors — never `throw` for normal flow
- **ProblemDetails** (RFC 7807) on all HTTP error responses

**Never do:**
```csharp
// ❌ Blocks the thread pool — causes deadlocks
_repo.GetAsync(id).Result
_repo.GetAsync(id).Wait()

// ❌ Business logic in Controller — belongs in Domain or Application
if (product.Price > 1000) { ... }

// ❌ DbContext directly in Handler — use IRepository
public class MyHandler(AppDbContext db)

// ❌ Hardcoded secrets
"Password=SuperSecret123"
"api_key=sk-live-..."

// ❌ Async void
public async void Process() { }

// ❌ Raw SQL with user string interpolation
FromSqlRaw($"SELECT * WHERE Name = '{userInput}'")
```

---

## Security — Non-Negotiable Rules

These rules apply to every task, always, without being explicitly requested:

- **Secrets**: User Secrets in development, environment variables or Key Vault in production. Never in `appsettings.json`.
- **Logs**: never log passwords, full tokens, credit card numbers, or PII.
- **Authorization**: every endpoint requires `[Authorize]`. Use `[AllowAnonymous]` only with a comment explaining why it is public.
- **Validation**: all client inputs pass through FluentValidation before reaching the Handler.
- **Queries**: EF Core parameterizes automatically. `FromSqlRaw` with string concatenation is forbidden.

---

## Delivery Checklist

Do not mark a task as complete until all applicable items are green:

```
CODE
[ ] dotnet build with no warnings  (TreatWarningsAsErrors = true)
[ ] No .Result / .Wait() in async code
[ ] No secrets in source code
[ ] CancellationToken propagated through all I/O

ARCHITECTURE
[ ] No business logic in Controllers
[ ] No DbContext directly in Handlers
[ ] Layer dependency rules not violated (verify with dotnet test tests/MyApi.ArchTests)

TESTS
[ ] dotnet test fully green
[ ] Unit tests for new logic
[ ] Integration tests for new endpoints
[ ] Coverage ≥ 80% in Domain and Application

SECURITY
[ ] Inputs validated with FluentValidation
[ ] Endpoints have [Authorize] or [AllowAnonymous] with comment
[ ] Errors return ProblemDetails

DATABASE
[ ] Migration generated if model changed
[ ] AsNoTracking() on read-only queries
```
