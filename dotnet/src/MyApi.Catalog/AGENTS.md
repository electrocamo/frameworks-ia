# AGENTS.md — Catalog Module

> Extends the root `AGENTS.md`. Read that file first, then this one.
> Rules here **override** the root document for this module only.

---

## Context

Product, category, and inventory management.
Highest read volume in the system — queries are optimized with Redis cache.

---

## ⚠️ Additional Skills for This Module

```
Are you adding a new read query that will be called frequently?
→ Review the cache rules in this file before implementing.

Are you modifying the Product or Category entity?
→ READ FIRST: ia/skills/ef-migrations/SKILL.md
```

---

## Cache Rules

- Read-only queries with more than 100 req/min **must** use `IDistributedCache` (Redis)
- Standard catalog TTL: **5 minutes**
- Mandatory cache key format: `catalog:product:{id}` — do not change this format
  without migrating existing keys
- Every Command that modifies a product must invalidate its cache
  using `CacheInvalidationBehavior`

## Search Rules

- Full-text search uses **PostgreSQL `tsvector`** — `LIKE '%term%'` is forbidden in production
- The `tsvector` index is updated by a DB trigger — never update it manually from code

## Query Rules

- Never nest `Include()` more than 2 levels deep — use `Select()` with projection instead
- Product lists are **always** paginated: minimum `pageSize=20`, maximum `pageSize=100`
- `AsNoTracking()` on absolutely every read query in this module

## Image Rules

- Image URLs are references to Azure Blob Storage — never store binaries in DB
- URL format: `https://{account}.blob.core.windows.net/products/{productId}/{filename}`
- Validate that every URL belongs to our domain before persisting

## Module-Specific Commands

```bash
# Run only catalog tests
dotnet test tests/MyApi.IntegrationTests --filter "Category=Catalog"

# Rebuild search indexes (only if confirmed corruption)
dotnet run --project tools/MyApi.Tools -- rebuild-search-index
```
