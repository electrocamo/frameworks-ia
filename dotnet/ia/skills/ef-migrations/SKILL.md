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

## Key difference — modular monolith

Each module that owns data has its own `{Module}.DbMigrator` project and its own
`DbContext` in `{Module}.Infrastructure`. Migrations are scoped per module.

---

## Standard Process — always in this order

### 1. Modify the entity in Domain first

```csharp
// src/Modules/ModuleAModule/ModuleA.Domain/Entities/Product.cs
public string? Sku { get; private set; }

public static Result<Product> Create(string name, string? sku, Money price, int stock)
{
    // include sku validation if applicable
}
```

### 2. Update the Fluent API configuration in Infrastructure

```csharp
// src/Modules/ModuleAModule/ModuleA.Infrastructure/Persistence/Configurations/ProductConfiguration.cs
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
  --project src/Modules/ModuleAModule/ModuleA.DbMigrator \
  --startup-project src/Api/MyApi \
  --output-dir Migrations
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
✅ CreateModuleBTable
❌ Migration1
❌ Fix
❌ changes
```

### 4. Review the generated file BEFORE applying

Open the file in `src/Modules/ModuleAModule/ModuleA.DbMigrator/Migrations/` and verify:
- `Up()` does exactly what is expected
- `Down()` reverts correctly — critical for rollback
- No unexpected destructive operations (`DROP COLUMN`, `DROP TABLE`)

### 5. Apply in development

```bash
dotnet ef database update \
  --project src/Modules/ModuleAModule/ModuleA.DbMigrator \
  --startup-project src/Api/MyApi
```

### 6. Verify

```bash
# Confirm the migration was applied
dotnet ef migrations list \
  --project src/Modules/ModuleAModule/ModuleA.DbMigrator \
  --startup-project src/Api/MyApi

# Run the module's tests to confirm nothing broke
dotnet test src/Tests/ModuleA.Test
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
    migrationBuilder.AddColumn<string>("FirstName", "Users", nullable: true);
    migrationBuilder.AddColumn<string>("LastName", "Users", nullable: true);

    migrationBuilder.Sql(@"
        UPDATE ""Users""
        SET ""FirstName"" = SPLIT_PART(""FullName"", ' ', 1),
            ""LastName""  = SPLIT_PART(""FullName"", ' ', 2)
        WHERE ""FullName"" IS NOT NULL
    ");

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
  --project src/Modules/ModuleAModule/ModuleA.DbMigrator \
  --startup-project src/Api/MyApi \
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
  --project src/Modules/ModuleAModule/ModuleA.DbMigrator \
  --startup-project src/Api/MyApi

# Remove the last migration file (only if NOT applied to any database)
dotnet ef migrations remove \
  --project src/Modules/ModuleAModule/ModuleA.DbMigrator \
  --startup-project src/Api/MyApi
```

---

## Common Errors

| Error | Cause | Solution |
|---|---|---|
| `No migrations configuration type was found` | Missing `IDesignTimeDbContextFactory` | Create the factory in the DbMigrator project |
| `Unable to create an object of type 'DbContext'` | No connection string when generating | Verify `--startup-project` points to the host |
| `There is already an object named X` | Partially applied migration | Check `__EFMigrationsHistory` and fix state |
| `Cannot drop column because it's used by an index` | Wrong order in migration | Drop index before dropping column in `Up()` |
| Empty migration generated | No changes detected | Verify `DbSet<T>` and `OnModelCreating` are updated |
