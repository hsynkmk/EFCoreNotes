# Global Query Filters in EF Core

## Overview

Global query filters allow you to define filters that are automatically applied to all queries for a specific entity type. Common use cases include multi-tenancy and soft-delete functionality.

## Key Features

- Automatically applied to all LINQ queries
- Can be disabled for specific queries using `IgnoreQueryFilters()`
- Can access DbContext instance properties
- Support navigation properties

## Configuration

```csharp
public class BloggingContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public int TenantId { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Soft delete filter
        modelBuilder.Entity<Blog>()
            .HasQueryFilter(b => !b.IsDeleted);

        // Multi-tenant filter
        modelBuilder.Entity<Blog>()
            .HasQueryFilter(b => b.TenantId == TenantId);
    }
}
```

## Common Use Cases

### 1. Soft Delete

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; }
    public bool IsDeleted { get; set; }
}

// Configuration
modelBuilder.Entity<Blog>()
    .HasQueryFilter(b => !b.IsDeleted);

// Usage
// Normal query - only returns non-deleted blogs
var blogs = context.Blogs.ToList();

// Override filter to see all blogs
var allBlogs = context.Blogs
    .IgnoreQueryFilters()
    .ToList();
```

### 2. Multi-tenancy

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; }
    public int TenantId { get; set; }
}

// Configuration
modelBuilder.Entity<Blog>()
    .HasQueryFilter(b => b.TenantId == context.TenantId);
```

### 3. Combined Filters

```csharp
modelBuilder.Entity<Blog>()
    .HasQueryFilter(b => !b.IsDeleted && b.TenantId == context.TenantId);
```

## Best Practices

1. Keep filters simple to avoid performance issues
2. Consider impact on Include operations
3. Test thoroughly with IgnoreQueryFilters()
4. Be cautious with navigation properties in filters
5. Document filter behavior for team awareness

## Performance Considerations

- Filters are applied at the database level
- Complex filters may impact query performance
- Consider indexing filtered columns
- Monitor generated SQL queries

## Important Notes

- Filters apply to all queries including Include statements
- Cannot be changed after model is built
- May affect performance if not properly designed
- Works with both sync and async queries
- Can be combined with other LINQ operations
