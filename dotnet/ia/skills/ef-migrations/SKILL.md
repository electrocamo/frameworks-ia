---
name: ef-migrations
description: >
  Read when the task involves creating, applying, reverting, or debugging
  EF Core migrations. Includes the step-by-step process, naming conventions,
  special cases (rename columns, seed data, production migrations),
  and common errors.
triggers:
  - create migration
  - apply migration
  - revert migration
  - dotnet ef migrations
  - entity change
  - DbContext change
  - migration error
---

# SKILL: Entity Framework Core Migrations

## Standard Process — always in this order

### 1. Modify the entity in Domain first

```csharp
// src/MyApi.Domain/Entities/Product.cs
public string? Sku { get; private set; }

public static Result<Product> Create(string name, string? sku, Money price, int stock)
{
    // include sku validation if applicable
}
```

### 2. Update the Fluent API configuration in Infrastructure

```csharp
// src/MyApi.Infrastructure/Persistence/Configurations/ProductConfiguration.cs
builder.Property(p => p.Sku)
    .HasMaxLength(50)
    .IsRequired(false);

builder.HasIndex(p => p.Sku)
    .IsUnique()
    .HasFilter("\"Sku\" IS NOT NULL"); // partial index — only non-null rows
```

### 3. Generate the migration

```bash
dotnet ef migrations add <MigrationName> \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api \
  --output-dir Persistence/Migrations
```

**Mandatory naming (descriptive PascalCase):**

| Operation | Correct name |
|---|---|
| Add table | `CreateOrdersTable` |
| Add column | `AddSkuToProducts` |
| Remove column | `RemoveDescriptionFromProducts` |
| Rename column | `RenameDescriptionInProducts` |
| Add index | `AddIndexOnProductsSku` |
| Seed data | `SeedDefaultCategories` |

```
✅ AddSkuToProducts
✅ CreatePaymentsTable
❌ Migration1
❌ Fix
❌ changes
```

### 4. Review the generated file BEFORE applying

Open the file in `src/MyApi.Infrastructure/Persistence/Migrations/` and verify:
- `Up()` does exactly what is expected
- `Down()` reverts correctly — critical for rollback
- No unexpected destructive operations (`DROP COLUMN`, `DROP TABLE`)

### 5. Apply in development

```bash
dotnet ef database update \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api
```

### 6. Verify

```bash
# Confirm the migration was applied
dotnet ef migrations list \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api

# Run integration tests to confirm nothing broke
dotnet test tests/MyApi.IntegrationTests
```

---

## Special Cases

### Renaming a column — do NOT use Drop + Add

```csharp
// ❌ Renaming only the C# property can generate Drop+Add (data loss)

// ✅ Use RenameColumn explicitly in the generated migration
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.RenameColumn(
        name: "Description",
        table: "Products",
        newName: "ShortDescription");
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.RenameColumn(
        name: "ShortDescription",
        table: "Products",
        newName: "Description");
}
```

### Migration with existing data transformation

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // 1. Add new columns as nullable first
    migrationBuilder.AddColumn<string>("FirstName", "Users", nullable: true);
    migrationBuilder.AddColumn<string>("LastName", "Users", nullable: true);

    // 2. Populate data with SQL
    migrationBuilder.Sql(@"
        UPDATE ""Users""
        SET ""FirstName"" = SPLIT_PART(""FullName"", ' ', 1),
            ""LastName""  = SPLIT_PART(""FullName"", ' ', 2)
        WHERE ""FullName"" IS NOT NULL
    ");

    // 3. Apply NOT NULL only after populating
    migrationBuilder.AlterColumn<string>("FirstName", "Users", nullable: false);
    migrationBuilder.AlterColumn<string>("LastName", "Users", nullable: false);
}
```

### Seed data

```csharp
// Only for immutable reference data: roles, permissions, fixed catalogs
// NEVER for variable business data
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.InsertData(
        table: "Categories",
        columns: ["Id", "Name", "CreatedAt"],
        values: new object[]
        {
            new object[] { 1, "Electronics", DateTime.UtcNow },
            new object[] { 2, "Clothing", DateTime.UtcNow }
        });
}
```

---

## Production Migrations — NEVER run `database update` directly

```bash
# Generate an idempotent SQL script for DBA review
dotnet ef migrations script \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api \
  --idempotent \
  --output migration.sql

# The DBA reviews and applies the script during the maintenance window
# In CI/CD pipelines: run the SQL script, not the update command
```

---

## Reverting a Migration

```bash
# Revert to a previous migration (TARGET = the migration you want to land on)
dotnet ef database update <PreviousMigrationName> \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api

# Remove the last migration file (only if NOT applied to any database)
dotnet ef migrations remove \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api
```

---

## Common Errors

| Error | Cause | Solution |
|---|---|---|
| `No migrations configuration type was found` | Missing `IDesignTimeDbContextFactory` | Create the factory in Infrastructure |
| `Unable to create an object of type 'AppDbContext'` | No connection string when generating | Verify `--startup-project` |
| `There is already an object named X` | Partially applied migration | Check `__EFMigrationsHistory` and fix state |
| `Cannot drop column because it's used by an index` | Wrong order in migration | Drop index before dropping column in `Up()` |
| Empty migration generated | No changes detected | Verify `DbSet<T>` and `OnModelCreating` are updated |
