# AGENTS.md — ModuleB Module

> Extends the root `AGENTS.md`. Read that file first, then this one.
> Rules here **override** the root document for this module only.

---

## Context

ModuleB domain (for example: Stripe payment provider integration).
Every financial operation here is **irreversible in production** — apply double validation
before persisting anything.

---

## ⚠️ Additional Skills for This Module

```
Does your task involve a multi-step flow (charge + reduce stock + confirm order)?
→ READ FIRST: ia/skills/architecture-patterns/SKILL.md  (section: Saga Pattern)

Are you implementing or modifying the Stripe webhook?
→ READ FIRST: ia/skills/security-checklist/SKILL.md  (section: signature verification)
```

---

## Stripe-Specific Rules

- Never use the real API key in tests — use `stripe-mock` with Testcontainers
- Every Stripe call must include an `IdempotencyKey` — use the `OrderId` as the value
- Webhooks must respond `200 OK` within 5 seconds or Stripe will retry
- Always verify the webhook signature with `StripeClient.ConstructEvent` before processing
- Log only the first 20 characters of the `stripe-signature` header — never the full value

## Transaction Rules

- Every command that moves money uses `IUnitOfWork.BeginTransactionAsync()` explicitly
- When in doubt between failing and duplicating: **always fail conservatively**
- A failed payment is recoverable. A duplicate charge is not.

## Module-Specific Commands

```bash
# Start stripe-mock locally
docker run --rm -p 12111:12111 stripe/stripe-mock:latest

# Run only this module's tests
dotnet test tests/MyApi.IntegrationTests --filter "Category=ModuleB"
```

## Required Secrets

```bash
dotnet user-secrets set "Stripe:PublishableKey" "pk_test_..." --project src/MyApi.Api
dotnet user-secrets set "Stripe:SecretKey" "sk_test_..." --project src/MyApi.Api
dotnet user-secrets set "Stripe:WebhookSecret" "whsec_..." --project src/MyApi.Api
```
