# Migrations in EF Core

## Overview

Migrations provide a way to incrementally update the database schema to keep it in sync with the application's data model while preserving existing data. They are essential for database version control and deployment.

## Key Features

- Automatic migration generation
- Database schema versioning
- Data preservation during updates
- Rollback support
- Custom operations
- SQL script generation

## Basic Commands

### 1. Add Migration

```bash
# Using Package Manager Console
Add-Migration InitialCreate

# Using .NET CLI
dotnet ef migrations add InitialCreate
```

### 2. Update Database

```bash
# Using Package Manager Console
Update-Database

# Using .NET CLI
dotnet ef database update
```

### 3. Remove Migration

```bash
# Using Package Manager Console
Remove-Migration

# Using .NET CLI
dotnet ef migrations remove
```

## Migration Structure

```csharp
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Blogs",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Url = table.Column<string>(nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Blogs", x => x.Id);
            });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(
            name: "Blogs");
    }
}
```

## Advanced Operations

### 1. Custom SQL

```csharp
migrationBuilder.Sql(@"
    CREATE PROCEDURE GetMostPopularBlogs
    AS
    BEGIN
        SELECT * FROM Blogs
        WHERE Rating > 4
    END");
```

### 2. Data Seeding

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.InsertData(
        table: "Blogs",
        columns: new[] { "Url", "Rating" },
        values: new object[] { "http://sample.com", 5 });
}
```

### 3. Column Modifications

```csharp
migrationBuilder.AlterColumn<string>(
    name: "Url",
    table: "Blogs",
    type: "nvarchar(500)",
    nullable: false,
    oldClrType: typeof(string),
    oldType: "nvarchar(max)",
    oldNullable: true);
```

## Common Scenarios

### 1. Generate SQL Script

```bash
# Using Package Manager Console
Script-Migration

# Using .NET CLI
dotnet ef migrations script
```

### 2. Target Specific Migration

```bash
# Update to specific migration
Update-Database TargetMigration

# Generate script between migrations
Script-Migration FirstMigration LastMigration
```

### 3. Multiple DbContext Support

```bash
# Specify context
Add-Migration InitialCreate -Context BloggingContext
Update-Database -Context BloggingContext
```

## Best Practices

### 1. Migration Naming

```csharp
// Good names
Add-Migration AddBlogCreatedTimestamp
Add-Migration UpdatePostsTableAddCategories
Add-Migration AddIndexToPostsTitle
```

### 2. Testing Migrations

```csharp
public class MigrationTests
{
    [Fact]
    public async Task CanApplyMigrations()
    {
        var connection = new SqliteConnection("DataSource=:memory:");
        connection.Open();

        try
        {
            var options = new DbContextOptionsBuilder<BloggingContext>()
                .UseSqlite(connection)
                .Options;

            using (var context = new BloggingContext(options))
            {
                await context.Database.MigrateAsync();
            }
        }
        finally
        {
            connection.Close();
        }
    }
}
```

## Deployment Strategies

### 1. Automatic Migration

```csharp
public void Configure(IApplicationBuilder app)
{
    using (var scope = app.ApplicationServices.GetService<IServiceScopeFactory>().CreateScope())
    {
        var context = scope.ServiceProvider.GetRequiredService<BloggingContext>();
        context.Database.Migrate();
    }
}
```

### 2. Script-Based Deployment

```bash
# Generate idempotent script
Script-Migration -Idempotent > migrate.sql

# Generate from specific migration
Script-Migration BaseMigration TargetMigration > migrate.sql
```

## Performance Considerations

- Large migrations can impact deployment time
- Consider breaking large migrations into smaller ones
- Test migration performance with production-size data
- Use appropriate transaction scopes
- Consider maintenance window requirements

## Important Notes

- Always backup database before migrations
- Test migrations in non-production first
- Keep migrations small and focused
- Use meaningful migration names
- Consider rollback strategy
- Version control your migrations
- Handle data preservation carefully
- Test both Up and Down methods
- Consider concurrent deployment scenarios
