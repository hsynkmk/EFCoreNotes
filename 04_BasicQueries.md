# Basic Queries in EF Core

## Overview

EF Core provides a powerful LINQ-based query interface for retrieving data from the database. This guide covers the fundamental query operations and patterns.

## Key Features

- LINQ integration
- Query execution modes
- Filtering and sorting
- Projection
- Aggregation
- Navigation properties
- Async operations

## Basic Query Operations

### 1. Simple Queries

```csharp
// Get all blogs
var blogs = context.Blogs.ToList();

// Get first blog
var firstBlog = context.Blogs.First();

// Get blog by ID
var blog = context.Blogs.Find(1);

// Get single blog by condition
var blog = context.Blogs
    .Single(b => b.Url == "http://example.com");
```

### 2. Filtering

```csharp
// Where clause
var activeBlogs = context.Blogs
    .Where(b => !b.IsDeleted)
    .ToList();

// Multiple conditions
var featuredBlogs = context.Blogs
    .Where(b => b.Rating > 4 && b.IsPublic)
    .ToList();

// Contains
var blogIds = new[] { 1, 2, 3 };
var specificBlogs = context.Blogs
    .Where(b => blogIds.Contains(b.Id))
    .ToList();
```

### 3. Sorting

```csharp
// OrderBy
var orderedBlogs = context.Blogs
    .OrderBy(b => b.Name)
    .ToList();

// Multiple ordering
var complexOrder = context.Blogs
    .OrderByDescending(b => b.Rating)
    .ThenBy(b => b.Name)
    .ToList();
```

## Working with Related Data

### 1. Include Related Data

```csharp
// Single Include
var blogsWithPosts = context.Blogs
    .Include(b => b.Posts)
    .ToList();

// Multiple Includes
var blogsComplete = context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Owner)
    .ToList();

// Nested Include
var blogsWithPostComments = context.Blogs
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
    .ToList();
```

### 2. Filtered Include

```csharp
var blogsWithRecentPosts = context.Blogs
    .Include(b => b.Posts
        .Where(p => p.PublishedDate >= DateTime.Today.AddDays(-7)))
    .ToList();
```

## Projection

### 1. Select

```csharp
// Basic projection
var blogDtos = context.Blogs
    .Select(b => new BlogDto
    {
        Id = b.Id,
        Url = b.Url,
        PostCount = b.Posts.Count
    })
    .ToList();

// Anonymous type
var blogSummaries = context.Blogs
    .Select(b => new
    {
        b.Id,
        b.Url,
        PostCount = b.Posts.Count
    })
    .ToList();
```

### 2. Complex Projections

```csharp
var blogStats = context.Blogs
    .Select(b => new BlogStatistics
    {
        BlogId = b.Id,
        PostCount = b.Posts.Count,
        LatestPost = b.Posts
            .OrderByDescending(p => p.PublishedDate)
            .FirstOrDefault(),
        AverageRating = b.Posts.Average(p => p.Rating)
    })
    .ToList();
```

## Aggregation

### 1. Basic Aggregates

```csharp
// Count
var blogCount = context.Blogs.Count();
var activeBlogCount = context.Blogs.Count(b => !b.IsDeleted);

// Sum
var totalPosts = context.Posts.Count();
var averageRating = context.Blogs.Average(b => b.Rating);

// Min/Max
var oldestPost = context.Posts.Min(p => p.PublishedDate);
var highestRating = context.Blogs.Max(b => b.Rating);
```

### 2. Grouped Aggregates

```csharp
var blogStats = context.Posts
    .GroupBy(p => p.BlogId)
    .Select(g => new
    {
        BlogId = g.Key,
        PostCount = g.Count(),
        AverageRating = g.Average(p => p.Rating),
        LatestPost = g.Max(p => p.PublishedDate)
    })
    .ToList();
```

## Async Operations

### 1. Basic Async Queries

```csharp
// Async ToList
var blogs = await context.Blogs.ToListAsync();

// Async First
var blog = await context.Blogs.FirstAsync(b => b.Id == 1);

// Async Single
var blog = await context.Blogs
    .SingleOrDefaultAsync(b => b.Url == "http://example.com");
```

### 2. Async Aggregates

```csharp
var count = await context.Blogs.CountAsync();
var average = await context.Blogs.AverageAsync(b => b.Rating);
var exists = await context.Blogs.AnyAsync(b => b.Rating > 4);
```

## Best Practices

1. Query Optimization

```csharp
// DO: Use async methods
await context.Blogs.ToListAsync();

// DO: Use specific queries
var blog = await context.Blogs.FindAsync(id);

// DON'T: Load unnecessary data
// Bad: Loading all blogs just to count them
var count = context.Blogs.ToList().Count;
// Good: Count at database level
var count = await context.Blogs.CountAsync();
```

2. Performance Tips

```csharp
// DO: Use AsNoTracking for read-only scenarios
var blogs = await context.Blogs
    .AsNoTracking()
    .ToListAsync();

// DO: Select only needed columns
var urls = await context.Blogs
    .Select(b => b.Url)
    .ToListAsync();
```

## Performance Considerations

- Query execution strategy
- Tracking behavior
- Include patterns
- Projection optimization
- Async operation benefits
- Database round trips
- Memory usage

## Important Notes

- Queries are executed when enumerated
- Use async operations when possible
- Consider tracking overhead
- Be mindful of N+1 queries
- Use appropriate Include depth
- Consider pagination
- Monitor query performance
- Use appropriate indexes
