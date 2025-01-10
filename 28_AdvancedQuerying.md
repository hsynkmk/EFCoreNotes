# Advanced Querying in EF Core

## Overview

Advanced querying in EF Core provides powerful capabilities for complex data retrieval and manipulation. This guide covers advanced query techniques, optimization strategies, and best practices for efficient database operations.

## Key Areas

- Complex queries
- Query optimization
- Custom projections
- Dynamic queries
- Aggregations
- Window functions
- Subqueries
- Advanced filtering

## Complex Queries

### 1. Advanced Joins

```csharp
public class BlogQueryService
{
    private readonly BlogContext _context;

    public async Task<List<BlogStatistics>> GetBlogStatisticsAsync()
    {
        return await _context.Blogs
            .GroupJoin(
                _context.Posts,
                blog => blog.Id,
                post => post.BlogId,
                (blog, posts) => new
                {
                    Blog = blog,
                    Posts = posts
                })
            .SelectMany(
                x => x.Posts.DefaultIfEmpty(),
                (blog, post) => new BlogStatistics
                {
                    BlogId = blog.Blog.Id,
                    BlogTitle = blog.Blog.Title,
                    PostCount = blog.Posts.Count(),
                    LastPostDate = blog.Posts.Max(p => p.PublishedDate),
                    TotalComments = blog.Posts.Sum(p => p.Comments.Count)
                })
            .ToListAsync();
    }

    public async Task<List<AuthorBlogSummary>> GetAuthorBlogSummariesAsync()
    {
        return await _context.Authors
            .Include(a => a.Blogs)
                .ThenInclude(b => b.Posts)
            .Select(a => new AuthorBlogSummary
            {
                AuthorId = a.Id,
                AuthorName = a.Name,
                BlogCount = a.Blogs.Count,
                TotalPosts = a.Blogs.Sum(b => b.Posts.Count),
                MostRecentPost = a.Blogs
                    .SelectMany(b => b.Posts)
                    .OrderByDescending(p => p.PublishedDate)
                    .FirstOrDefault()
            })
            .ToListAsync();
    }
}
```

### 2. Conditional Joins

```csharp
public class PostQueryService
{
    private readonly BlogContext _context;

    public async Task<List<PostDetails>> GetPostDetailsAsync(
        DateTime? fromDate = null,
        string category = null)
    {
        var query = _context.Posts
            .Include(p => p.Blog)
            .Include(p => p.Author)
            .AsQueryable();

        if (fromDate.HasValue)
        {
            query = query.Where(p => p.PublishedDate >= fromDate);
        }

        if (!string.IsNullOrEmpty(category))
        {
            query = query
                .Include(p => p.Categories)
                .Where(p => p.Categories.Any(c => c.Name == category));
        }

        return await query
            .Select(p => new PostDetails
            {
                Id = p.Id,
                Title = p.Title,
                BlogTitle = p.Blog.Title,
                AuthorName = p.Author.Name,
                PublishedDate = p.PublishedDate,
                Categories = p.Categories.Select(c => c.Name).ToList()
            })
            .ToListAsync();
    }
}
```

## Query Optimization

### 1. Efficient Loading

```csharp
public class BlogLoadingService
{
    private readonly BlogContext _context;

    public async Task<List<BlogSummary>> GetBlogSummariesEfficientlyAsync()
    {
        // Split query for better performance with large data sets
        return await _context.Blogs
            .AsSplitQuery()
            .Include(b => b.Posts)
            .Include(b => b.Author)
                .ThenInclude(a => a.Profile)
            .Select(b => new BlogSummary
            {
                Id = b.Id,
                Title = b.Title,
                AuthorName = b.Author.Name,
                PostCount = b.Posts.Count,
                RecentPosts = b.Posts
                    .OrderByDescending(p => p.PublishedDate)
                    .Take(5)
                    .Select(p => new PostSummary
                    {
                        Id = p.Id,
                        Title = p.Title,
                        PublishedDate = p.PublishedDate
                    })
                    .ToList()
            })
            .ToListAsync();
    }

    public async Task<List<Blog>> GetBlogsWithFilteredPostsAsync(
        DateTime fromDate)
    {
        return await _context.Blogs
            .Include(b => b.Posts.Where(p =>
                p.PublishedDate >= fromDate))
            .Where(b => b.Posts.Any())
            .ToListAsync();
    }
}
```

### 2. Compiled Queries

```csharp
public class CompiledQueries
{
    private static readonly Func<BlogContext, int, Task<Blog>> GetBlogById =
        EF.CompileAsyncQuery((BlogContext context, int id) =>
            context.Blogs
                .Include(b => b.Posts)
                .FirstOrDefault(b => b.Id == id));

    private static readonly Func<BlogContext, string, Task<List<Blog>>> GetBlogsByAuthor =
        EF.CompileAsyncQuery((BlogContext context, string authorName) =>
            context.Blogs
                .Include(b => b.Posts)
                .Where(b => b.Author.Name == authorName)
                .ToList());

    public async Task<Blog> GetBlogEfficientlyAsync(int id)
    {
        using var context = new BlogContext();
        return await GetBlogById(context, id);
    }

    public async Task<List<Blog>> GetAuthorBlogsEfficientlyAsync(string authorName)
    {
        using var context = new BlogContext();
        return await GetBlogsByAuthor(context, authorName);
    }
}
```

## Dynamic Queries

### 1. Expression Building

```csharp
public class DynamicQueryBuilder
{
    public static Expression<Func<Blog, bool>> BuildBlogFilter(
        string title = null,
        string authorName = null,
        DateTime? fromDate = null)
    {
        var parameter = Expression.Parameter(typeof(Blog), "blog");
        Expression expression = Expression.Constant(true);

        if (!string.IsNullOrEmpty(title))
        {
            var titleProperty = Expression.Property(parameter, "Title");
            var titleValue = Expression.Constant(title);
            var titleEquals = Expression.Call(
                titleProperty,
                "Contains",
                Type.EmptyTypes,
                titleValue);
            expression = Expression.AndAlso(expression, titleEquals);
        }

        if (!string.IsNullOrEmpty(authorName))
        {
            var authorProperty = Expression.Property(parameter, "Author");
            var nameProperty = Expression.Property(authorProperty, "Name");
            var authorValue = Expression.Constant(authorName);
            var authorEquals = Expression.Equal(nameProperty, authorValue);
            expression = Expression.AndAlso(expression, authorEquals);
        }

        if (fromDate.HasValue)
        {
            var dateProperty = Expression.Property(parameter, "CreatedAt");
            var dateValue = Expression.Constant(fromDate.Value);
            var dateGreaterThan = Expression.GreaterThanOrEqual(
                dateProperty,
                dateValue);
            expression = Expression.AndAlso(expression, dateGreaterThan);
        }

        return Expression.Lambda<Func<Blog, bool>>(expression, parameter);
    }
}
```

### 2. Dynamic Ordering

```csharp
public class DynamicOrderingService
{
    private readonly BlogContext _context;

    public async Task<List<Blog>> GetBlogsOrderedAsync(
        string orderBy,
        bool ascending = true)
    {
        var query = _context.Blogs.AsQueryable();

        switch (orderBy.ToLower())
        {
            case "title":
                query = ascending
                    ? query.OrderBy(b => b.Title)
                    : query.OrderByDescending(b => b.Title);
                break;

            case "date":
                query = ascending
                    ? query.OrderBy(b => b.CreatedAt)
                    : query.OrderByDescending(b => b.CreatedAt);
                break;

            case "posts":
                query = ascending
                    ? query.OrderBy(b => b.Posts.Count)
                    : query.OrderByDescending(b => b.Posts.Count);
                break;

            default:
                query = query.OrderBy(b => b.Id);
                break;
        }

        return await query.ToListAsync();
    }
}
```

## Advanced Filtering

### 1. Complex Predicates

```csharp
public class AdvancedFilterService
{
    private readonly BlogContext _context;

    public async Task<List<Post>> GetFilteredPostsAsync(
        PostFilterCriteria criteria)
    {
        return await _context.Posts
            .Include(p => p.Blog)
            .Include(p => p.Author)
            .Include(p => p.Categories)
            .Where(p =>
                (!criteria.FromDate.HasValue || p.PublishedDate >= criteria.FromDate) &&
                (!criteria.ToDate.HasValue || p.PublishedDate <= criteria.ToDate) &&
                (string.IsNullOrEmpty(criteria.AuthorName) ||
                    p.Author.Name.Contains(criteria.AuthorName)) &&
                (criteria.Categories == null ||
                    !criteria.Categories.Any() ||
                    p.Categories.Any(c => criteria.Categories.Contains(c.Name))) &&
                (criteria.MinComments == null ||
                    p.Comments.Count >= criteria.MinComments) &&
                (!criteria.IsPublished.HasValue ||
                    p.IsPublished == criteria.IsPublished))
            .ToListAsync();
    }
}

public class PostFilterCriteria
{
    public DateTime? FromDate { get; set; }
    public DateTime? ToDate { get; set; }
    public string AuthorName { get; set; }
    public List<string> Categories { get; set; }
    public int? MinComments { get; set; }
    public bool? IsPublished { get; set; }
}
```

### 2. Search Functionality

```csharp
public class SearchService
{
    private readonly BlogContext _context;

    public async Task<List<SearchResult>> SearchContentAsync(
        string searchTerm,
        SearchOptions options = null)
    {
        if (string.IsNullOrEmpty(searchTerm))
            return new List<SearchResult>();

        var query = _context.Posts
            .Include(p => p.Blog)
            .Include(p => p.Author)
            .AsQueryable();

        // Apply search term
        query = query.Where(p =>
            p.Title.Contains(searchTerm) ||
            p.Content.Contains(searchTerm) ||
            p.Blog.Title.Contains(searchTerm) ||
            p.Author.Name.Contains(searchTerm));

        // Apply options
        if (options != null)
        {
            if (options.FromDate.HasValue)
                query = query.Where(p => p.PublishedDate >= options.FromDate);

            if (options.AuthorId.HasValue)
                query = query.Where(p => p.AuthorId == options.AuthorId);

            if (options.Categories?.Any() == true)
                query = query.Where(p =>
                    p.Categories.Any(c =>
                        options.Categories.Contains(c.Name)));
        }

        // Project to search results
        return await query
            .Select(p => new SearchResult
            {
                Id = p.Id,
                Title = p.Title,
                Excerpt = p.Content.Substring(0,
                    Math.Min(200, p.Content.Length)),
                AuthorName = p.Author.Name,
                BlogTitle = p.Blog.Title,
                PublishedDate = p.PublishedDate,
                Relevance = (p.Title.Contains(searchTerm) ? 2 : 0) +
                           (p.Content.Contains(searchTerm) ? 1 : 0)
            })
            .OrderByDescending(r => r.Relevance)
            .ThenByDescending(r => r.PublishedDate)
            .Take(options?.MaxResults ?? 50)
            .ToListAsync();
    }
}

public class SearchOptions
{
    public DateTime? FromDate { get; set; }
    public int? AuthorId { get; set; }
    public List<string> Categories { get; set; }
    public int? MaxResults { get; set; }
}
```

## Best Practices

1. Query Optimization

```csharp
// DO: Use appropriate loading strategies
public async Task<List<BlogSummary>> GetBlogSummariesOptimizedAsync()
{
    return await _context.Blogs
        .AsNoTracking()  // For read-only scenarios
        .Select(b => new BlogSummary
        {
            Id = b.Id,
            Title = b.Title,
            PostCount = b.Posts.Count
        })
        .ToListAsync();
}
```

2. Performance Considerations

```csharp
// DO: Use paging for large result sets
public async Task<PagedResult<Blog>> GetPagedBlogsAsync(
    int pageNumber,
    int pageSize)
{
    var query = _context.Blogs
        .AsNoTracking()
        .OrderByDescending(b => b.CreatedAt);

    var totalCount = await query.CountAsync();
    var items = await query
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();

    return new PagedResult<Blog>
    {
        Items = items,
        TotalCount = totalCount,
        PageNumber = pageNumber,
        PageSize = pageSize
    };
}
```

## Performance Considerations

- Query complexity
- Loading strategies
- Index usage
- Memory usage
- Network traffic
- Batch size
- Caching strategy
- Connection management

## Important Notes

- Query optimization
- Index strategy
- Parameter usage
- Transaction scope
- Eager vs lazy loading
- Query tracking
- Memory management
- Performance monitoring
