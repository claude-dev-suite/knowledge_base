# Entity Framework Core Migrations

> Official Documentation: https://learn.microsoft.com/ef/core/managing-schemas/migrations/

## Table of Contents

1. [Getting Started](#getting-started)
2. [Creating Migrations](#creating-migrations)
3. [Applying Migrations](#applying-migrations)
4. [Removing Migrations](#removing-migrations)
5. [Migration File Structure](#migration-file-structure)
6. [Customizing Migrations](#customizing-migrations)
7. [SQL in Migrations](#sql-in-migrations)
8. [Data Seeding](#data-seeding)
9. [Reverting Migrations](#reverting-migrations)
10. [Migration Bundles](#migration-bundles)
11. [Generating SQL Scripts](#generating-sql-scripts)
12. [Multiple Providers](#multiple-providers)
13. [CI/CD Migration Strategies](#cicd-migration-strategies)
14. [Migration Squashing](#migration-squashing)
15. [Handling Production Migrations](#handling-production-migrations)
16. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## Getting Started

### Install EF Core Tools

```bash
# Global tool (recommended)
dotnet tool install --global dotnet-ef

# Update to latest
dotnet tool update --global dotnet-ef

# Verify installation
dotnet ef --version
```

### Required NuGet Package

```bash
# Add the design-time package to the startup project
dotnet add package Microsoft.EntityFrameworkCore.Design
```

---

## Creating Migrations

### Basic Migration

```bash
# Create a new migration
dotnet ef migrations add InitialCreate

# With specific project and startup project
dotnet ef migrations add InitialCreate \
    --project src/MyApp.Data \
    --startup-project src/MyApp.Api

# With specific output directory
dotnet ef migrations add InitialCreate --output-dir Data/Migrations

# With specific context (when multiple DbContexts exist)
dotnet ef migrations add InitialCreate --context AppDbContext
```

### List Existing Migrations

```bash
# List all migrations
dotnet ef migrations list

# List with connection info
dotnet ef migrations list --connection "Server=localhost;Database=MyApp;..."
```

### PMC (Package Manager Console) Alternative

```powershell
Add-Migration InitialCreate
Add-Migration InitialCreate -Context AppDbContext -OutputDir Data/Migrations
```

---

## Applying Migrations

### Using CLI

```bash
# Apply all pending migrations
dotnet ef database update

# Apply to a specific migration
dotnet ef database update InitialCreate

# Apply with specific connection
dotnet ef database update --connection "Server=localhost;Database=MyApp;..."
```

### Programmatic Migration at Startup

```csharp
var app = builder.Build();

// Apply pending migrations on startup
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await context.Database.MigrateAsync();
}

app.Run();
```

### Check for Pending Migrations

```csharp
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();

    var pending = await context.Database.GetPendingMigrationsAsync();
    if (pending.Any())
    {
        Console.WriteLine($"Pending migrations: {string.Join(", ", pending)}");
        await context.Database.MigrateAsync();
    }

    var applied = await context.Database.GetAppliedMigrationsAsync();
    Console.WriteLine($"Applied migrations: {string.Join(", ", applied)}");
}
```

---

## Removing Migrations

```bash
# Remove the last migration (only if not applied)
dotnet ef migrations remove

# Force remove (deletes migration file even if applied - dangerous)
dotnet ef migrations remove --force
```

---

## Migration File Structure

Each migration generates three files:

```
Migrations/
├── 20240115120000_InitialCreate.cs          # Up() and Down() methods
├── 20240115120000_InitialCreate.Designer.cs  # Model snapshot at this point
└── AppDbContextModelSnapshot.cs              # Current model state
```

### Migration Class

```csharp
using Microsoft.EntityFrameworkCore.Migrations;

namespace MyApp.Data.Migrations;

/// <inheritdoc />
public partial class InitialCreate : Migration
{
    /// <inheritdoc />
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Categories",
            columns: table => new
            {
                Id = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(type: "nvarchar(200)", maxLength: 200, nullable: false),
                Description = table.Column<string>(type: "nvarchar(max)", nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Categories", x => x.Id);
            });

        migrationBuilder.CreateTable(
            name: "Products",
            columns: table => new
            {
                Id = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(type: "nvarchar(200)", maxLength: 200, nullable: false),
                Price = table.Column<decimal>(type: "decimal(18,2)", precision: 18, scale: 2, nullable: false),
                CategoryId = table.Column<int>(type: "int", nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Products", x => x.Id);
                table.ForeignKey(
                    name: "FK_Products_Categories_CategoryId",
                    column: x => x.CategoryId,
                    principalTable: "Categories",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Restrict);
            });

        migrationBuilder.CreateIndex(
            name: "IX_Products_CategoryId",
            table: "Products",
            column: "CategoryId");
    }

    /// <inheritdoc />
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Products");
        migrationBuilder.DropTable(name: "Categories");
    }
}
```

---

## Customizing Migrations

### Adding Custom Operations

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // Auto-generated code...
    migrationBuilder.AddColumn<string>(
        name: "Slug",
        table: "Products",
        type: "nvarchar(200)",
        maxLength: 200,
        nullable: true);

    // Custom: populate slugs from existing names
    migrationBuilder.Sql("""
        UPDATE Products
        SET Slug = LOWER(REPLACE(REPLACE(Name, ' ', '-'), '''', ''))
        WHERE Slug IS NULL
        """);

    // Now make the column required
    migrationBuilder.AlterColumn<string>(
        name: "Slug",
        table: "Products",
        type: "nvarchar(200)",
        maxLength: 200,
        nullable: false,
        defaultValue: "");

    migrationBuilder.CreateIndex(
        name: "IX_Products_Slug",
        table: "Products",
        column: "Slug",
        unique: true);
}
```

### Renaming Instead of Drop/Add

EF Core may interpret a renamed column as drop-then-add. Override this manually:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // EF generates: DropColumn + AddColumn (data loss!)
    // Replace with:
    migrationBuilder.RenameColumn(
        name: "ProductName",
        table: "Products",
        newName: "Name");
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.RenameColumn(
        name: "Name",
        table: "Products",
        newName: "ProductName");
}
```

---

## SQL in Migrations

### Raw SQL Statements

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // Create a view
    migrationBuilder.Sql("""
        CREATE VIEW vw_ActiveProducts AS
        SELECT p.Id, p.Name, p.Price, c.Name AS CategoryName
        FROM Products p
        INNER JOIN Categories c ON p.CategoryId = c.Id
        WHERE p.IsActive = 1
        """);

    // Create a stored procedure
    migrationBuilder.Sql("""
        CREATE PROCEDURE sp_GetTopProducts
            @count INT = 10
        AS
        BEGIN
            SELECT TOP (@count) *
            FROM Products
            ORDER BY Price DESC
        END
        """);

    // Create a function
    migrationBuilder.Sql("""
        CREATE FUNCTION fn_GetProductCount(@categoryId INT)
        RETURNS INT
        AS
        BEGIN
            RETURN (SELECT COUNT(*) FROM Products WHERE CategoryId = @categoryId)
        END
        """);

    // Insert reference data
    migrationBuilder.Sql("""
        INSERT INTO Categories (Name, Description)
        VALUES
            ('Electronics', 'Electronic devices and accessories'),
            ('Clothing', 'Apparel and fashion items'),
            ('Books', 'Physical and digital books')
        """);
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.Sql("DROP VIEW IF EXISTS vw_ActiveProducts");
    migrationBuilder.Sql("DROP PROCEDURE IF EXISTS sp_GetTopProducts");
    migrationBuilder.Sql("DROP FUNCTION IF EXISTS fn_GetProductCount");
}
```

### SQL from Embedded Resource

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    var assembly = Assembly.GetExecutingAssembly();
    using var stream = assembly.GetManifestResourceStream("MyApp.Data.Scripts.SeedData.sql");
    using var reader = new StreamReader(stream!);
    var sql = reader.ReadToEnd();
    migrationBuilder.Sql(sql);
}
```

---

## Data Seeding

### HasData (Model-Level Seeding)

```csharp
public class CategoryConfiguration : IEntityTypeConfiguration<Category>
{
    public void Configure(EntityTypeBuilder<Category> builder)
    {
        builder.HasData(
            new Category { Id = 1, Name = "Electronics", Description = "Electronic devices" },
            new Category { Id = 2, Name = "Clothing", Description = "Apparel" },
            new Category { Id = 3, Name = "Books", Description = "Physical and digital books" }
        );
    }
}

public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasData(
            new Product { Id = 1, Name = "Laptop", Price = 999.99m, CategoryId = 1 },
            new Product { Id = 2, Name = "T-Shirt", Price = 29.99m, CategoryId = 2 },
            new Product { Id = 3, Name = "C# in Depth", Price = 49.99m, CategoryId = 3 }
        );
    }
}
```

### Seeding with Owned Types

```csharp
builder.OwnsOne(p => p.Address).HasData(
    new { ProductId = 1, Street = "123 Main St", City = "Seattle", ZipCode = "98101" }
);
```

### Custom Seeding Logic (Application-Level)

```csharp
public static class DbSeeder
{
    public static async Task SeedAsync(AppDbContext context)
    {
        if (await context.Categories.AnyAsync())
            return;

        var categories = new List<Category>
        {
            new() { Name = "Electronics", Description = "Electronic devices" },
            new() { Name = "Clothing", Description = "Apparel" },
        };

        context.Categories.AddRange(categories);
        await context.SaveChangesAsync();
    }
}

// In Program.cs
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await context.Database.MigrateAsync();
    await DbSeeder.SeedAsync(context);
}
```

---

## Reverting Migrations

```bash
# Revert to a specific migration
dotnet ef database update AddProductSlug

# Revert all migrations (empty database)
dotnet ef database update 0
```

---

## Migration Bundles

Migration bundles are self-contained executables that apply migrations without needing the .NET SDK or source code.

```bash
# Create a bundle
dotnet ef migrations bundle

# With options
dotnet ef migrations bundle \
    --output ./efbundle \
    --self-contained \
    --target-runtime linux-x64

# Run the bundle
./efbundle --connection "Server=prod;Database=MyApp;..."
```

### Bundle in Docker

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet tool restore
RUN dotnet ef migrations bundle --output /app/efbundle --self-contained --target-runtime linux-x64

FROM mcr.microsoft.com/dotnet/runtime-deps:8.0
COPY --from=build /app/efbundle /app/efbundle
ENTRYPOINT ["/app/efbundle"]
```

---

## Generating SQL Scripts

### Basic Script Generation

```bash
# Generate script for all migrations
dotnet ef migrations script

# Generate script between two migrations
dotnet ef migrations script AddCategory AddProduct

# Generate script from beginning to specific migration
dotnet ef migrations script 0 AddProduct
```

### Idempotent Scripts

Idempotent scripts check which migrations have been applied and only run what is needed. Ideal for production deployments.

```bash
# Generate idempotent script
dotnet ef migrations script --idempotent --output migrate.sql
```

```sql
-- Generated idempotent script example
IF NOT EXISTS (
    SELECT * FROM [__EFMigrationsHistory]
    WHERE [MigrationId] = N'20240115120000_InitialCreate'
)
BEGIN
    CREATE TABLE [Categories] (
        [Id] int NOT NULL IDENTITY,
        [Name] nvarchar(200) NOT NULL,
        CONSTRAINT [PK_Categories] PRIMARY KEY ([Id])
    );
END;
GO

IF NOT EXISTS (
    SELECT * FROM [__EFMigrationsHistory]
    WHERE [MigrationId] = N'20240115120000_InitialCreate'
)
BEGIN
    INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
    VALUES (N'20240115120000_InitialCreate', N'8.0.0');
END;
GO
```

---

## Multiple Providers

### Provider-Specific Migrations

```csharp
public class AppDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        if (Database.IsSqlServer())
        {
            modelBuilder.Entity<Product>()
                .Property(p => p.CreatedAt)
                .HasDefaultValueSql("GETUTCDATE()");
        }
        else if (Database.IsNpgsql())
        {
            modelBuilder.Entity<Product>()
                .Property(p => p.CreatedAt)
                .HasDefaultValueSql("NOW()");
        }
    }
}
```

### Separate Migration Assemblies per Provider

```bash
# SQL Server migrations
dotnet ef migrations add InitialCreate \
    --project src/MyApp.SqlServerMigrations \
    --context AppDbContext

# PostgreSQL migrations
dotnet ef migrations add InitialCreate \
    --project src/MyApp.PostgresMigrations \
    --context AppDbContext
```

```csharp
// Program.cs - select provider at runtime
var provider = builder.Configuration["DatabaseProvider"];

builder.Services.AddDbContext<AppDbContext>(options =>
{
    _ = provider switch
    {
        "SqlServer" => options.UseSqlServer(
            builder.Configuration.GetConnectionString("SqlServer"),
            x => x.MigrationsAssembly("MyApp.SqlServerMigrations")),
        "PostgreSQL" => options.UseNpgsql(
            builder.Configuration.GetConnectionString("PostgreSQL"),
            x => x.MigrationsAssembly("MyApp.PostgresMigrations")),
        _ => throw new InvalidOperationException($"Unknown provider: {provider}")
    };
});
```

---

## CI/CD Migration Strategies

### Strategy 1: SQL Script in Pipeline

```yaml
# GitHub Actions
- name: Generate migration script
  run: |
    dotnet ef migrations script --idempotent --output migrate.sql \
      --project src/MyApp.Data \
      --startup-project src/MyApp.Api

- name: Apply migration
  run: |
    sqlcmd -S ${{ secrets.DB_SERVER }} \
           -d ${{ secrets.DB_NAME }} \
           -U ${{ secrets.DB_USER }} \
           -P ${{ secrets.DB_PASSWORD }} \
           -i migrate.sql
```

### Strategy 2: Migration Bundle in Pipeline

```yaml
- name: Build migration bundle
  run: |
    dotnet ef migrations bundle \
      --self-contained \
      --target-runtime linux-x64 \
      --output ./efbundle \
      --project src/MyApp.Data \
      --startup-project src/MyApp.Api

- name: Apply migrations
  run: |
    chmod +x ./efbundle
    ./efbundle --connection "${{ secrets.CONNECTION_STRING }}"
```

### Strategy 3: Programmatic at Startup (Non-Production)

```csharp
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await context.Database.MigrateAsync();
}
```

---

## Migration Squashing

When the migration history grows large, you can squash older migrations into a single baseline.

```bash
# 1. Ensure database is fully migrated
dotnet ef database update

# 2. Remove all migration files
rm -rf Migrations/

# 3. Create a fresh initial migration from current model
dotnet ef migrations add Baseline

# 4. Mark the baseline as already applied (for existing databases)
dotnet ef database update Baseline -- --skip
```

### Manual Squash Approach

```csharp
// For existing databases, create the baseline migration with empty Up/Down
public partial class Baseline : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Intentionally empty - represents current schema state
        // Only applied to new databases via the full schema in the snapshot
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        // Intentionally empty
    }
}
```

---

## Handling Production Migrations

### Safe Migration Checklist

| Step | Action |
|------|--------|
| 1 | Test migration on a copy of production data |
| 2 | Generate idempotent SQL script |
| 3 | Review the SQL script manually |
| 4 | Take a database backup |
| 5 | Apply during a maintenance window |
| 6 | Monitor for errors and performance issues |
| 7 | Have a rollback plan (Down migration or backup restore) |

### Zero-Downtime Migration Pattern

```csharp
// Step 1: Add new column (nullable) - deploy code that ignores it
public partial class AddEmailVerified : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<bool>(
            name: "EmailVerified",
            table: "Users",
            type: "bit",
            nullable: true); // Nullable first!
    }
}

// Step 2: Backfill data - deploy code that reads both
public partial class BackfillEmailVerified : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("UPDATE Users SET EmailVerified = 0 WHERE EmailVerified IS NULL");
    }
}

// Step 3: Make non-nullable - deploy code that requires it
public partial class MakeEmailVerifiedRequired : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AlterColumn<bool>(
            name: "EmailVerified",
            table: "Users",
            type: "bit",
            nullable: false,
            defaultValue: false);
    }
}
```

---

## Best Practices and Common Pitfalls

### Best Practices

| Practice | Description |
|----------|-------------|
| Name migrations descriptively | `AddProductSlug` not `Migration3` |
| Review generated migrations | EF may generate unexpected changes |
| Use idempotent scripts for production | Safe to re-run without errors |
| Test Down() methods | Ensure rollbacks work correctly |
| Store migrations in version control | Track all schema changes |
| Use separate migration assemblies for multiple providers | Keep provider-specific code isolated |
| Seed reference data with `HasData` | Ensures consistent baseline data |
| Use migration bundles for CI/CD | No SDK dependency on deployment servers |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Editing applied migrations | Snapshot mismatch, broken history | Create a new migration for changes |
| Renaming detected as drop/add | Data loss in production | Manually use `RenameColumn` / `RenameTable` |
| Missing `Down()` method | Cannot rollback migration | Always implement `Down()` methods |
| Auto-migrate at startup in prod | Concurrent instances may conflict | Use bundles or SQL scripts in CI/CD |
| Large data operations in migration | Long locks on production tables | Batch operations using raw SQL |
| Not testing with production data size | Migration may timeout | Test with realistic data volumes |
| Deleting snapshot file | EF cannot diff model correctly | Never delete `ModelSnapshot.cs` |
| Mixing code-first and database-first | Conflicting schema changes | Choose one approach per context |
