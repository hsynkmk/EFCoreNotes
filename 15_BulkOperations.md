# Bulk Operations in EF Core

## Overview

Bulk operations in EF Core allow you to perform operations on multiple entities efficiently. While EF Core has built-in methods for batch operations, there are also third-party libraries that provide enhanced bulk operation capabilities.

## Key Features

- Batch inserts
- Batch updates
- Batch deletes
- Transaction support
- Progress monitoring
- Memory optimization
- Performance tuning options

## Built-in Batch Operations

### 1. AddRange/UpdateRange/RemoveRange

```csharp
// Batch Insert
var blogs = new List<Blog>
{
    new Blog { Url = "http://blog1.com" },
    new Blog { Url = "http://blog2.com" }
};
context.Blogs.AddRange(blogs);
await context.SaveChangesAsync();

// Batch Update
var blogsToUpdate = await context.Blogs
    .Where(b => b.Rating < 4)
    .ToListAsync();
blogsToUpdate.ForEach(b => b.Rating = 4);
context.Blogs.UpdateRange(blogsToUpdate);
await context.SaveChangesAsync();

// Batch Delete
var blogsToDelete = await context.Blogs
    .Where(b => b.IsObsolete)
    .ToListAsync();
context.Blogs.RemoveRange(blogsToDelete);
await context.SaveChangesAsync();
```

### 2. ExecuteDelete/ExecuteUpdate (EF Core 7.0+)

```csharp
// Batch Delete
await context.Blogs
    .Where(b => b.CreatedDate < DateTime.Today.AddYears(-1))
    .ExecuteDeleteAsync();

// Batch Update
await context.Blogs
    .Where(b => b.Rating < 4)
    .ExecuteUpdateAsync(s =>
        s.SetProperty(b => b.Rating, b => 4));
```

## Third-Party Libraries

### 1. EFCore.BulkExtensions

```csharp
// Install package
// Install-Package EFCore.BulkExtensions

// Bulk Insert
await context.BulkInsertAsync(blogs);

// Bulk Update
await context.BulkUpdateAsync(blogs);

// Bulk Delete
await context.BulkDeleteAsync(blogs);

// Bulk Merge
await context.BulkMergeAsync(blogs);
```

### 2. Configuration Options

```csharp
var bulkConfig = new BulkConfig
{
    BatchSize = 1000,
    EnableStreaming = true,
    UseTempDB = true,
    PreserveInsertOrder = true
};

await context.BulkInsertAsync(blogs, bulkConfig);
```

## Advanced Scenarios

### 1. Batch Size Optimization

```csharp
public async Task ImportBlogsInBatches(IEnumerable<Blog> blogs, int batchSize = 1000)
{
    var batches = blogs.Chunk(batchSize);
    foreach (var batch in batches)
    {
        context.Blogs.AddRange(batch);
        await context.SaveChangesAsync();
        context.ChangeTracker.Clear();
    }
}
```

### 2. Progress Tracking

```csharp
public async Task ImportBlogsWithProgress(IEnumerable<Blog> blogs,
    IProgress<int> progress)
{
    var totalCount = blogs.Count();
    var processedCount = 0;

    foreach (var batch in blogs.Chunk(1000))
    {
        await context.BulkInsertAsync(batch);
        processedCount += batch.Length;
        progress.Report((int)((float)processedCount / totalCount * 100));
    }
}
```

### 3. Transaction Management

```csharp
using var transaction = await context.Database.BeginTransactionAsync();
try
{
    var bulkConfig = new BulkConfig { UseTempDB = true };
    await context.BulkInsertAsync(blogs, bulkConfig);
    await context.BulkUpdateAsync(existingBlogs, bulkConfig);
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

## Common Patterns

### 1. Repository Implementation

```csharp
public class BlogRepository
{
    private readonly BloggingContext _context;
    private const int BatchSize = 1000;

    public async Task BulkInsertAsync(IEnumerable<Blog> blogs)
    {
        await _context.BulkInsertAsync(blogs, new BulkConfig
        {
            BatchSize = BatchSize,
            EnableStreaming = true
        });
    }

    public async Task BulkUpdateAsync(IEnumerable<Blog> blogs)
    {
        await _context.BulkUpdateAsync(blogs, new BulkConfig
        {
            BatchSize = BatchSize,
            EnableStreaming = true
        });
    }
}
```

### 2. Service Layer Integration

```csharp
public class BlogImportService
{
    private readonly BlogRepository _repository;
    private readonly ILogger<BlogImportService> _logger;

    public async Task ImportBlogsAsync(Stream dataStream)
    {
        try
        {
            var blogs = await ParseBlogsFromStream(dataStream);
            await _repository.BulkInsertAsync(blogs);
            _logger.LogInformation($"Successfully imported {blogs.Count} blogs");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to import blogs");
            throw;
        }
    }
}
```

## Best Practices

1. Use appropriate batch sizes
2. Clear change tracker between batches
3. Handle large datasets in chunks
4. Implement proper error handling
5. Monitor memory usage
6. Use transactions for consistency
7. Consider database constraints

## Performance Considerations

- Choose optimal batch size
- Monitor memory consumption
- Consider database indexes
- Use appropriate transaction isolation
- Enable/disable triggers if necessary
- Consider table locks
- Monitor tempdb usage

## Important Notes

- Test with representative data volumes
- Consider impact on other operations
- Monitor database resources
- Handle constraint violations
- Consider identity insert settings
- Plan for rollback scenarios
- Document bulk operation patterns
- Consider maintenance windows
