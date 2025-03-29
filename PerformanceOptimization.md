# Performance Optimization in EF Core

## Overview

Performance optimization in EF Core involves various techniques and best practices to improve application speed, reduce memory usage, and optimize database operations. This guide covers key strategies for achieving optimal performance.

## Key Areas

- Query optimization
- Change tracking optimization
- Connection management
- Caching strategies
- Batch operations
- Memory management
- Database design
- Monitoring and profiling

## Query Optimization

### 1. Efficient Loading Strategies

```csharp
// DO: Use projection for specific fields
var blogTitles = await context.Blogs
    .Select(b => new { b.Id, b.Title })
    .ToListAsync();

// DO: Use Include wisely
var blogsWithPosts = await context.Blogs
    .Include(b => b.Posts)
    .Where(b => b.Posts.Any())
    .ToListAsync();

// DON'T: Load unnecessary data
// Bad: Loading all fields when only title is needed
var blogs = await context.Blogs.ToListAsync();
var titles = blogs.Select(b => b.Title);
```

### 2. Tracking Behavior

```csharp
// Disable tracking for read-only queries
var blogs = await context.Blogs
    .AsNoTracking()
    .ToListAsync();

// Use tracking only when needed
var blog = await context.Blogs
    .AsTracking()
    .FirstOrDefaultAsync(b => b.Id == id);
```

### 3. Split Queries

```csharp
// Split complex queries
var blogsWithDetails = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Author)
        .ThenInclude(a => a.Address)
    .AsSplitQuery()
    .ToListAsync();
```

## Batch Operations

### 1. Bulk Insert

```csharp
// Efficient bulk insert
public async Task BulkInsertBlogs(IEnumerable<Blog> blogs)
{
    using var transaction = await context.Database.BeginTransactionAsync();
    try
    {
        context.ChangeTracker.AutoDetectChangesEnabled = false;

        foreach (var batch in blogs.Chunk(1000))
        {
            await context.Blogs.AddRangeAsync(batch);
            await context.SaveChangesAsync();
        }

        await transaction.CommitAsync();
    }
    finally
    {
        context.ChangeTracker.AutoDetectChangesEnabled = true;
    }
}
```

### 2. Bulk Update

```csharp
// Efficient bulk update
public async Task BulkUpdateBlogStatus(IEnumerable<int> blogIds, bool isActive)
{
    await context.Blogs
        .Where(b => blogIds.Contains(b.Id))
        .ExecuteUpdateAsync(s =>
            s.SetProperty(b => b.IsActive, isActive));
}
```

## Memory Optimization

### 1. Streaming Results

```csharp
// Stream large result sets
public async Task ProcessLargeDataSet()
{
    await foreach (var blog in context.Blogs
        .AsNoTracking()
        .AsAsyncEnumerable())
    {
        await ProcessBlog(blog);
    }
}
```

### 2. Pagination

```csharp
public async Task<List<Blog>> GetPagedBlogs(int pageNumber, int pageSize)
{
    return await context.Blogs
        .OrderBy(b => b.Id)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();
}
```

## Connection Management

### 1. Connection Resiliency

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(connectionString, options =>
        options.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorNumbersToAdd: null)
    );
}
```

### 2. Connection Pooling

```csharp
// Optimal connection string
"Server=.;Database=BlogDb;Trusted_Connection=True;Max Pool Size=100;Min Pool Size=10"
```

## Caching Strategies

### 1. Second-Level Cache

```csharp
public class BlogService
{
    private readonly IMemoryCache _cache;
    private readonly BlogContext _context;

    public async Task<Blog> GetBlogById(int id)
    {
        var cacheKey = $"blog_{id}";

        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.SetAbsoluteExpiration(TimeSpan.FromMinutes(10));
            return await _context.Blogs
                .AsNoTracking()
                .FirstOrDefaultAsync(b => b.Id == id);
        });
    }
}
```

### 2. Query Result Caching

```csharp
public class CachedBlogRepository
{
    private readonly IMemoryCache _cache;
    private readonly BlogContext _context;

    public async Task<IEnumerable<BlogSummary>> GetPopularBlogs()
    {
        return await _cache.GetOrCreateAsync("popular_blogs", async entry =>
        {
            entry.SetSlidingExpiration(TimeSpan.FromMinutes(5));
            return await _context.Blogs
                .Where(b => b.Rating >= 4)
                .Select(b => new BlogSummary
                {
                    Id = b.Id,
                    Title = b.Title,
                    Rating = b.Rating
                })
                .ToListAsync();
        });
    }
}
```

## Database Design Optimization

### 1. Indexing Strategy

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasIndex(b => b.Url)
        .IsUnique();

    modelBuilder.Entity<Post>()
        .HasIndex(p => new { p.BlogId, p.PublishedDate });
}
```

### 2. Efficient Relationships

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .HasForeignKey(p => p.BlogId)
    .OnDelete(DeleteBehavior.Cascade);
```

## Monitoring and Profiling

### 1. Query Logging

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging()
        .EnableDetailedErrors();
}
```

### 2. Performance Metrics

```csharp
public class PerformanceMonitor
{
    public static async Task<(T Result, TimeSpan Duration)> MeasureExecutionTime<T>(
        Func<Task<T>> operation)
    {
        var stopwatch = Stopwatch.StartNew();
        var result = await operation();
        stopwatch.Stop();

        return (result, stopwatch.Elapsed);
    }
}

// Usage
var (blogs, duration) = await PerformanceMonitor
    .MeasureExecutionTime(() => context.Blogs.ToListAsync());
```

## Best Practices

1. Query Optimization

```csharp
// DO: Use async methods
await context.Blogs.ToListAsync();

// DO: Filter data at database level
var recentBlogs = await context.Blogs
    .Where(b => b.CreatedDate >= DateTime.Today.AddDays(-7))
    .ToListAsync();

// DON'T: Filter in memory
var blogs = await context.Blogs.ToListAsync();
var recentBlogs = blogs
    .Where(b => b.CreatedDate >= DateTime.Today.AddDays(-7));
```

2. Change Tracking

```csharp
// DO: Disable tracking for read-only operations
var blogs = await context.Blogs
    .AsNoTracking()
    .ToListAsync();

// DO: Detach entities when no longer needed
context.Entry(blog).State = EntityState.Detached;
```

## Performance Considerations

- Query complexity
- Data volume
- Network latency
- Memory usage
- Connection pool size
- Transaction scope
- Batch size
- Cache invalidation

## Important Notes

- Monitor query performance
- Use appropriate indexes
- Implement caching strategy
- Optimize database design
- Handle large datasets
- Profile application
- Regular maintenance
- Performance testing
