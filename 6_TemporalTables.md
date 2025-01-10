# Temporal Tables in EF Core

## Overview

Temporal Tables (also known as System-Versioned Temporal Tables) allow you to track and query the complete history of changes made to your data over time. This feature is supported in EF Core 6.0 and later versions.

## Key Features

- Automatic tracking of data changes
- Query data at any point in time
- Built-in support for audit trails
- Seamless integration with EF Core
- Support for cleaning up historical data

## Configuration

### 1. Basic Setup

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Url { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .ToTable("Blogs", b => b.IsTemporal());
}
```

### 2. Custom Period Columns

```csharp
modelBuilder.Entity<Blog>()
    .ToTable("Blogs", b => b
        .IsTemporal(t =>
        {
            t.HasPeriodStart("ValidFrom");
            t.HasPeriodEnd("ValidTo");
            t.UseHistoryTable("BlogsHistory");
        }));
```

## Querying Temporal Data

### 1. As of a Specific Point in Time

```csharp
var blogs = await context.Blogs
    .TemporalAsOf(DateTime.UtcNow.AddDays(-1))
    .ToListAsync();
```

### 2. Over a Time Range

```csharp
var changes = await context.Blogs
    .TemporalFromTo(startTime, endTime)
    .ToListAsync();
```

### 3. All Changes

```csharp
var allVersions = await context.Blogs
    .TemporalAll()
    .ToListAsync();
```

### 4. Between Time Periods

```csharp
var changes = await context.Blogs
    .TemporalBetween(startTime, endTime)
    .ToListAsync();
```

## Common Scenarios

### 1. Audit Trail

```csharp
public async Task<IEnumerable<Blog>> GetBlogHistory(int blogId)
{
    return await context.Blogs
        .TemporalAll()
        .Where(b => b.Id == blogId)
        .OrderBy(b => EF.Property<DateTime>(b, "PeriodStart"))
        .ToListAsync();
}
```

### 2. Point-in-Time Recovery

```csharp
public async Task<Blog> GetBlogAtPoint(int blogId, DateTime pointInTime)
{
    return await context.Blogs
        .TemporalAsOf(pointInTime)
        .FirstOrDefaultAsync(b => b.Id == blogId);
}
```

### 3. Change Analysis

```csharp
public async Task<IEnumerable<Blog>> GetChangesInTimeRange(
    DateTime start, DateTime end)
{
    return await context.Blogs
        .TemporalBetween(start, end)
        .OrderBy(b => EF.Property<DateTime>(b, "PeriodStart"))
        .ToListAsync();
}
```

## Advanced Usage

### 1. Cleanup Historical Data

```csharp
await context.Database.ExecuteSqlRawAsync(@"
    ALTER TABLE Blogs
    SET (SYSTEM_VERSIONING = OFF);

    DELETE FROM BlogsHistory
    WHERE PeriodEnd < @p0;

    ALTER TABLE Blogs
    SET (SYSTEM_VERSIONING = ON);",
    DateTime.UtcNow.AddYears(-1));
```

### 2. Custom History Table Naming

```csharp
modelBuilder.Entity<Blog>()
    .ToTable("Blogs", b => b
        .IsTemporal(t =>
        {
            t.UseHistoryTable("BlogsHistory");
        }));
```

## Best Practices

1. Use UTC timestamps for consistency
2. Plan for data retention and cleanup
3. Consider performance impact on writes
4. Index historical tables appropriately
5. Use appropriate security measures

## Performance Considerations

- Additional storage requirements
- Impact on write operations
- Index strategy for historical tables
- Query performance with large history tables
- Memory usage with temporal queries

## Important Notes

- Only supported in SQL Server 2016 and later
- Requires appropriate database permissions
- Cannot modify historical data
- All properties are included in history
- Transaction consistency is maintained
- Not supported with owned entities
- Memory requirements for large queries
