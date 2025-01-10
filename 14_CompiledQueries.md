# Compiled Queries in EF Core

## Overview

Compiled queries allow you to cache and reuse query execution plans, improving performance for frequently executed queries. They are particularly useful for repetitive queries with different parameters.

## Key Features

- Query plan caching
- Improved performance
- Parameter support
- Thread safety
- Support for async operations
- Compatible with most LINQ operations

## Basic Usage

### 1. Simple Compiled Query

```csharp
private static readonly Func<BloggingContext, int, Blog> GetBlogById =
    EF.CompileQuery((BloggingContext context, int id) =>
        context.Blogs
            .Include(b => b.Posts)
            .FirstOrDefault(b => b.Id == id));

// Usage
using (var context = new BloggingContext())
{
    var blog = GetBlogById(context, 1);
}
```

### 2. Async Compiled Query

```csharp
private static readonly Func<BloggingContext, int, Task<Blog>> GetBlogByIdAsync =
    EF.CompileAsyncQuery((BloggingContext context, int id) =>
        context.Blogs
            .Include(b => b.Posts)
            .FirstOrDefault(b => b.Id == id));

// Usage
using (var context = new BloggingContext())
{
    var blog = await GetBlogByIdAsync(context, 1);
}
```

## Advanced Scenarios

### 1. Multiple Parameters

```csharp
private static readonly Func<BloggingContext, string, int, List<Blog>> GetBlogsByUrlAndRating =
    EF.CompileQuery((BloggingContext context, string urlPattern, int minRating) =>
        context.Blogs
            .Where(b => b.Url.Contains(urlPattern) && b.Rating >= minRating)
            .ToList());
```

### 2. Complex Queries

```csharp
private static readonly Func<BloggingContext, int, IEnumerable<Post>> GetTopPostsByBlog =
    EF.CompileQuery((BloggingContext context, int blogId) =>
        context.Posts
            .Where(p => p.BlogId == blogId)
            .OrderByDescending(p => p.Views)
            .Take(10)
            .Include(p => p.Comments)
            .Include(p => p.Tags));
```

### 3. Aggregation Queries

```csharp
private static readonly Func<BloggingContext, int, double> GetAverageRating =
    EF.CompileQuery((BloggingContext context, int blogId) =>
        context.Posts
            .Where(p => p.BlogId == blogId)
            .Average(p => p.Rating));
```

## Common Patterns

### 1. Repository Implementation

```csharp
public class BlogRepository
{
    private static readonly Func<BloggingContext, string, Blog> GetBlogByUrl =
        EF.CompileQuery((BloggingContext context, string url) =>
            context.Blogs
                .FirstOrDefault(b => b.Url == url));

    private readonly BloggingContext _context;

    public Blog GetByUrl(string url) => GetBlogByUrl(_context, url);
}
```

### 2. Service Layer Integration

```csharp
public class BlogService
{
    private static readonly Func<BloggingContext, DateTime, List<Blog>> GetRecentBlogs =
        EF.CompileQuery((BloggingContext context, DateTime since) =>
            context.Blogs
                .Where(b => b.CreatedDate >= since)
                .OrderByDescending(b => b.CreatedDate)
                .ToList());

    private readonly BloggingContext _context;

    public List<Blog> GetRecentBlogs(DateTime since) =>
        GetRecentBlogs(_context, since);
}
```

## Performance Optimization

### 1. Query Result Caching

```csharp
public class CachedBlogRepository
{
    private static readonly Func<BloggingContext, int, Blog> GetBlogById =
        EF.CompileQuery((BloggingContext context, int id) =>
            context.Blogs.FirstOrDefault(b => b.Id == id));

    private readonly IMemoryCache _cache;
    private readonly BloggingContext _context;

    public Blog GetById(int id)
    {
        return _cache.GetOrCreate($"blog_{id}", entry =>
        {
            entry.SlidingExpiration = TimeSpan.FromMinutes(10);
            return GetBlogById(_context, id);
        });
    }
}
```

### 2. Batch Operations

```csharp
private static readonly Func<BloggingContext, int, List<Blog>> GetBlogsBatch =
    EF.CompileQuery((BloggingContext context, int batchSize) =>
        context.Blogs
            .OrderBy(b => b.Id)
            .Take(batchSize)
            .ToList());
```

## Best Practices

1. Query Definition

```csharp
// DO: Define as static readonly
private static readonly Func<BloggingContext, int, Blog> GetBlog =
    EF.CompileQuery((BloggingContext context, int id) =>
        context.Blogs.FirstOrDefault(b => b.Id == id));

// DON'T: Compile query on each use
public Blog GetBlog(int id) =>
    EF.CompileQuery((BloggingContext context, int blogId) =>
        context.Blogs.FirstOrDefault(b => b.Id == blogId))(_context, id);
```

2. Parameter Usage

```csharp
// DO: Use parameters
private static readonly Func<BloggingContext, string, Blog> GetBlogByUrl =
    EF.CompileQuery((BloggingContext context, string url) =>
        context.Blogs.FirstOrDefault(b => b.Url == url));

// DON'T: Use closure variables
private static readonly Func<BloggingContext, Blog> GetBlogByUrl =
    EF.CompileQuery((BloggingContext context) =>
        context.Blogs.FirstOrDefault(b => b.Url == someUrl)); // Wrong!
```

## Performance Considerations

- Compilation has upfront cost
- Best for frequently executed queries
- Memory usage for cached plans
- Parameter types affect caching
- Consider query complexity
- Monitor cache hit rates

## Important Notes

- Queries are cached per parameter combination
- Not all LINQ operations are supported
- Thread-safe execution
- Consider memory implications
- Test performance benefits
- Monitor query plan cache
- Document compiled queries
- Consider maintenance implications
