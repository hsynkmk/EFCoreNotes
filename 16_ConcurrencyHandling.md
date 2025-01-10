# Concurrency Handling in EF Core

## Overview

Concurrency control in EF Core helps manage conflicts that can occur when multiple users try to modify the same data simultaneously. It provides mechanisms to detect and handle these conflicts to maintain data integrity.

## Key Features

- Optimistic concurrency control
- Concurrency tokens
- Conflict detection
- Resolution strategies
- Database concurrency stamps
- Custom conflict handling
- Transaction isolation levels

## Basic Configuration

### 1. Using Data Annotations

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; }

    [ConcurrencyCheck]
    public string Title { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

### 2. Using Fluent API

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .Property(b => b.RowVersion)
        .IsRowVersion();

    modelBuilder.Entity<Blog>()
        .Property(b => b.Title)
        .IsConcurrencyToken();
}
```

## Implementation Strategies

### 1. Optimistic Concurrency

```csharp
public async Task UpdateBlogAsync(Blog blog)
{
    try
    {
        context.Entry(blog).State = EntityState.Modified;
        await context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        var entry = ex.Entries.Single();
        var currentValues = entry.CurrentValues;
        var databaseValues = await entry.GetDatabaseValuesAsync();

        if (databaseValues == null)
        {
            // The entity was deleted by another user
            throw new BlogDeletedException(blog.Id);
        }

        // Update the original values with the database values
        entry.OriginalValues.SetValues(databaseValues);

        // Retry the save operation
        await context.SaveChangesAsync();
    }
}
```

### 2. Custom Resolution

```csharp
public async Task<Blog> ResolveConcurrencyConflict(Blog clientBlog)
{
    var saved = false;
    while (!saved)
    {
        try
        {
            context.Update(clientBlog);
            await context.SaveChangesAsync();
            saved = true;
        }
        catch (DbUpdateConcurrencyException ex)
        {
            foreach (var entry in ex.Entries)
            {
                var proposedValues = entry.CurrentValues;
                var databaseValues = await entry.GetDatabaseValuesAsync();

                foreach (var property in proposedValues.Properties)
                {
                    var proposedValue = proposedValues[property];
                    var databaseValue = databaseValues[property];

                    // TODO: Decide which value should be used
                    // proposedValues[property] = <chosen value>;
                }

                // Refresh original values to bypass next concurrency check
                entry.OriginalValues.SetValues(databaseValues);
            }
        }
    }
    return clientBlog;
}
```

## Advanced Scenarios

### 1. Multiple Concurrency Tokens

```csharp
public class Order
{
    public int Id { get; set; }

    [ConcurrencyCheck]
    public string Status { get; set; }

    [ConcurrencyCheck]
    public DateTime LastUpdated { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; }
}

// Configuration
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .IsConcurrencyToken();

modelBuilder.Entity<Order>()
    .Property(o => o.LastUpdated)
    .IsConcurrencyToken();
```

### 2. Custom Conflict Resolution

```csharp
public async Task<Blog> HandleConcurrencyConflict(Blog clientBlog,
    ConflictResolutionStrategy strategy)
{
    try
    {
        context.Update(clientBlog);
        await context.SaveChangesAsync();
        return clientBlog;
    }
    catch (DbUpdateConcurrencyException ex)
    {
        var entry = ex.Entries.Single();
        var databaseValues = await entry.GetDatabaseValuesAsync();

        switch (strategy)
        {
            case ConflictResolutionStrategy.ClientWins:
                entry.OriginalValues.SetValues(databaseValues);
                await context.SaveChangesAsync();
                return clientBlog;

            case ConflictResolutionStrategy.DatabaseWins:
                entry.Reload();
                return (Blog)entry.Entity;

            case ConflictResolutionStrategy.Merge:
                var merged = MergeValues(clientBlog,
                    (Blog)entry.Entity,
                    (Blog)databaseValues.ToObject());
                context.Update(merged);
                await context.SaveChangesAsync();
                return merged;

            default:
                throw new NotSupportedException();
        }
    }
}
```

## Common Patterns

### 1. Repository Implementation

```csharp
public class BlogRepository
{
    private readonly BloggingContext _context;

    public async Task<Blog> UpdateWithConcurrencyCheckAsync(Blog blog)
    {
        var strategy = _context.Database.CreateExecutionStrategy();

        return await strategy.ExecuteAsync(async () =>
        {
            using var transaction = await _context.Database
                .BeginTransactionAsync();
            try
            {
                _context.Update(blog);
                await _context.SaveChangesAsync();
                await transaction.CommitAsync();
                return blog;
            }
            catch (DbUpdateConcurrencyException ex)
            {
                await transaction.RollbackAsync();
                // Handle concurrency conflict
                throw new ConcurrencyException(ex);
            }
        });
    }
}
```

### 2. Service Layer Integration

```csharp
public class BlogService
{
    private readonly BlogRepository _repository;
    private readonly ILogger<BlogService> _logger;

    public async Task<BlogUpdateResult> UpdateBlogAsync(Blog blog)
    {
        try
        {
            var updatedBlog = await _repository
                .UpdateWithConcurrencyCheckAsync(blog);
            return BlogUpdateResult.Success(updatedBlog);
        }
        catch (ConcurrencyException ex)
        {
            _logger.LogWarning(ex, "Concurrency conflict detected");
            return BlogUpdateResult.Conflict(
                await _repository.GetCurrentVersionAsync(blog.Id));
        }
    }
}
```

## Best Practices

1. Choose appropriate concurrency tokens
2. Implement proper error handling
3. Consider user experience in conflict scenarios
4. Use appropriate transaction isolation levels
5. Test concurrent access scenarios
6. Document resolution strategies
7. Monitor concurrency conflicts

## Performance Considerations

- Impact of concurrency checks
- Transaction isolation level effects
- Lock duration and scope
- Database provider specifics
- Index usage on concurrency tokens
- Conflict resolution overhead
- Retry strategy impact

## Important Notes

- Test with realistic concurrent loads
- Consider business requirements
- Document resolution policies
- Monitor conflict frequency
- Handle disconnected scenarios
- Consider timestamp overflow
- Plan for scalability
- Test all resolution paths
