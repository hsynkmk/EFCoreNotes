# Versioning and Schema Evolution in EF Core

## Overview

Managing database schema changes and versioning in EF Core requires careful planning and execution. This guide covers strategies for evolving your database schema while maintaining data integrity and application compatibility.

## Key Areas

- Schema versioning
- Migration strategies
- Data evolution
- Breaking changes
- Backward compatibility
- Deployment coordination
- Version control
- Rollback procedures

## Schema Versioning

### 1. Migration Management

```csharp
public class BlogContext : DbContext
{
    public static readonly string SchemaVersion = "2.0";

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Version tracking table
        modelBuilder.Entity<SchemaVersion>()
            .ToTable("__SchemaVersions");

        modelBuilder.Entity<SchemaVersion>()
            .Property(v => v.Version)
            .IsRequired()
            .HasMaxLength(20);

        // Seed current version
        modelBuilder.Entity<SchemaVersion>().HasData(
            new SchemaVersion
            {
                Id = 1,
                Version = SchemaVersion,
                AppliedOn = DateTime.UtcNow
            });
    }
}

public class SchemaVersion
{
    public int Id { get; set; }
    public string Version { get; set; }
    public DateTime AppliedOn { get; set; }
    public string Description { get; set; }
}
```

### 2. Version Tracking

```csharp
public class DatabaseVersionManager
{
    private readonly BlogContext _context;
    private readonly ILogger<DatabaseVersionManager> _logger;

    public async Task<bool> IsDatabaseCurrentAsync()
    {
        var currentVersion = await _context.SchemaVersions
            .OrderByDescending(v => v.AppliedOn)
            .Select(v => v.Version)
            .FirstOrDefaultAsync();

        return currentVersion == BlogContext.SchemaVersion;
    }

    public async Task RecordVersionUpdateAsync(string version, string description)
    {
        await _context.SchemaVersions.AddAsync(new SchemaVersion
        {
            Version = version,
            Description = description,
            AppliedOn = DateTime.UtcNow
        });

        await _context.SaveChangesAsync();
    }
}
```

## Migration Strategies

### 1. Incremental Migrations

```csharp
public class AddBlogStatusMigration : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Add new column
        migrationBuilder.AddColumn<string>(
            name: "Status",
            table: "Blogs",
            type: "nvarchar(20)",
            nullable: true);

        // Migrate existing data
        migrationBuilder.Sql(@"
            UPDATE Blogs
            SET Status = CASE
                WHEN IsDeleted = 1 THEN 'Deleted'
                WHEN IsPublished = 1 THEN 'Published'
                ELSE 'Draft'
            END");

        // Remove old columns
        migrationBuilder.DropColumn(
            name: "IsDeleted",
            table: "Blogs");

        migrationBuilder.DropColumn(
            name: "IsPublished",
            table: "Blogs");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        // Revert changes in reverse order
        migrationBuilder.AddColumn<bool>(
            name: "IsDeleted",
            table: "Blogs",
            type: "bit",
            nullable: false,
            defaultValue: false);

        migrationBuilder.AddColumn<bool>(
            name: "IsPublished",
            table: "Blogs",
            type: "bit",
            nullable: false,
            defaultValue: false);

        // Migrate data back
        migrationBuilder.Sql(@"
            UPDATE Blogs
            SET IsDeleted = CASE WHEN Status = 'Deleted' THEN 1 ELSE 0 END,
                IsPublished = CASE WHEN Status = 'Published' THEN 1 ELSE 0 END");

        migrationBuilder.DropColumn(
            name: "Status",
            table: "Blogs");
    }
}
```

### 2. Data Migration

```csharp
public class DataMigrationService
{
    private readonly BlogContext _context;
    private readonly ILogger<DataMigrationService> _logger;

    public async Task MigrateUserDataAsync()
    {
        await using var transaction = await _context.Database
            .BeginTransactionAsync();
        try
        {
            // Batch process data
            var batchSize = 1000;
            var processed = 0;

            while (true)
            {
                var users = await _context.Users
                    .Where(u => u.Version < 2)
                    .Take(batchSize)
                    .ToListAsync();

                if (!users.Any()) break;

                foreach (var user in users)
                {
                    await MigrateUserAsync(user);
                    processed++;
                }

                await _context.SaveChangesAsync();
                _logger.LogInformation("Migrated {Count} users", processed);
            }

            await transaction.CommitAsync();
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync();
            _logger.LogError(ex, "Error during user data migration");
            throw;
        }
    }

    private async Task MigrateUserAsync(User user)
    {
        // Perform data migration logic
        user.FullName = $"{user.FirstName} {user.LastName}";
        user.Version = 2;

        // Create audit record
        await _context.UserMigrationLogs.AddAsync(new UserMigrationLog
        {
            UserId = user.Id,
            OldVersion = 1,
            NewVersion = 2,
            MigratedAt = DateTime.UtcNow
        });
    }
}
```

## Breaking Changes

### 1. Compatibility Layer

```csharp
public class BlogCompatibilityService
{
    private readonly BlogContext _context;

    public async Task<BlogDto> GetBlogAsync(int id, string clientVersion)
    {
        var blog = await _context.Blogs
            .Include(b => b.Posts)
            .FirstOrDefaultAsync(b => b.Id == id);

        if (blog == null) return null;

        // Handle different client versions
        return clientVersion switch
        {
            "1.0" => MapToV1Format(blog),
            "2.0" => MapToV2Format(blog),
            _ => MapToCurrentFormat(blog)
        };
    }

    private BlogDto MapToV1Format(Blog blog)
    {
        return new BlogDto
        {
            Id = blog.Id,
            Title = blog.Title,
            // Map to old format
            IsActive = blog.Status != "Deleted",
            IsPublished = blog.Status == "Published"
        };
    }

    private BlogDtoV2 MapToV2Format(Blog blog)
    {
        return new BlogDtoV2
        {
            Id = blog.Id,
            Title = blog.Title,
            Status = blog.Status,
            // New fields in v2
            LastModified = blog.ModifiedAt
        };
    }
}
```

### 2. Schema Compatibility

```csharp
public class SchemaCompatibilityManager
{
    private readonly BlogContext _context;

    public async Task EnsureCompatibilityAsync()
    {
        // Check for required views
        if (!await ViewExistsAsync("vw_BlogStats"))
        {
            await CreateCompatibilityViewAsync();
        }

        // Check for backward compatibility columns
        if (!await ColumnExistsAsync("Blogs", "IsPublished"))
        {
            await CreateCompatibilityColumnsAsync();
        }
    }

    private async Task CreateCompatibilityViewAsync()
    {
        await _context.Database.ExecuteSqlRawAsync(@"
            CREATE VIEW vw_BlogStats AS
            SELECT b.Id, b.Title,
                   CASE WHEN b.Status = 'Published' THEN 1 ELSE 0 END as IsPublished,
                   CASE WHEN b.Status = 'Deleted' THEN 1 ELSE 0 END as IsDeleted
            FROM Blogs b");
    }
}
```

## Best Practices

1. Migration Planning

```csharp
// DO: Create reversible migrations
public partial class AddBlogCategories : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Create new table
        migrationBuilder.CreateTable(
            name: "BlogCategories",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(maxLength: 50, nullable: false)
            });

        // Add relationship
        migrationBuilder.AddColumn<int>(
            name: "CategoryId",
            table: "Blogs",
            nullable: true);

        migrationBuilder.CreateIndex(
            name: "IX_Blogs_CategoryId",
            table: "Blogs",
            column: "CategoryId");

        migrationBuilder.AddForeignKey(
            name: "FK_Blogs_Categories",
            table: "Blogs",
            column: "CategoryId",
            principalTable: "BlogCategories",
            principalColumn: "Id",
            onDelete: ReferentialAction.SetNull);
    }
}
```

2. Version Control

```csharp
// DO: Track schema changes
public class SchemaChangeLog
{
    public int Id { get; set; }
    public string Version { get; set; }
    public string ChangeType { get; set; }
    public string Description { get; set; }
    public string Script { get; set; }
    public DateTime AppliedOn { get; set; }
    public string AppliedBy { get; set; }
}

public class SchemaTracker
{
    private readonly BlogContext _context;

    public async Task LogSchemaChangeAsync(
        string version,
        string changeType,
        string description,
        string script)
    {
        await _context.SchemaChangeLogs.AddAsync(new SchemaChangeLog
        {
            Version = version,
            ChangeType = changeType,
            Description = description,
            Script = script,
            AppliedOn = DateTime.UtcNow,
            AppliedBy = Environment.UserName
        });

        await _context.SaveChangesAsync();
    }
}
```

## Performance Considerations

- Migration impact
- Data volume
- Downtime requirements
- Resource utilization
- Rollback performance
- Version compatibility
- Query optimization
- Index maintenance

## Important Notes

- Test migrations thoroughly
- Plan for rollbacks
- Document changes
- Version control
- Data backup
- Communication plan
- Monitoring strategy
- Recovery procedures
