# Lazy Loading in EF Core

## Overview

Lazy loading is a design pattern where related data is automatically loaded from the database when you access a navigation property. This feature is useful when you don't know which related data you'll need ahead of time.

## Key Features

- Automatically loads related data on access
- Reduces initial query time
- Supports proxies and ILazyLoader
- Works with all types of relationships
- Can be disabled/enabled at context level

## Setup

### 1. Install Required Package

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.Proxies" Version="8.0.0" />
```

### 2. Enable Lazy Loading

```csharp
// In DbContext configuration
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseLazyLoadingProxies()
        .UseSqlServer(connectionString);
}

// Entity configuration
public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; }

    // Virtual keyword enables lazy loading
    public virtual ICollection<Post> Posts { get; set; }
}
```

## Implementation Methods

### 1. Using Virtual Properties

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; }
    public virtual ICollection<Post> Posts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public virtual Blog Blog { get; set; }
}
```

### 2. Using ILazyLoader

```csharp
public class Blog
{
    private ILazyLoader _lazyLoader;
    private ICollection<Post> _posts;

    public Blog(ILazyLoader lazyLoader)
    {
        _lazyLoader = lazyLoader;
    }

    public ICollection<Post> Posts
    {
        get => _lazyLoader.Load(this, ref _posts);
        set => _posts = value;
    }
}
```

### 3. Using Lazy Loading with Injection

```csharp
public class Blog
{
    private Action<object, string> _lazyLoader;
    private ICollection<Post> _posts;

    public Blog(Action<object, string> lazyLoader)
    {
        _lazyLoader = lazyLoader;
    }

    public ICollection<Post> Posts
    {
        get
        {
            _lazyLoader?.Invoke(this, nameof(Posts));
            return _posts;
        }
        set => _posts = value;
    }
}
```

## Common Scenarios

### 1. Basic Usage

```csharp
using (var context = new BloggingContext())
{
    var blog = context.Blogs.First();
    // Posts are loaded when accessed
    var postCount = blog.Posts.Count;
}
```

### 2. Selective Loading

```csharp
public class BloggingContext : DbContext
{
    public bool LazyLoadingEnabled { get; set; } = true;

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder
            .UseLazyLoadingProxies()
            .UseSqlServer(connectionString);
    }

    public override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseLazyLoadingProxies(LazyLoadingEnabled);
    }
}
```

## Best Practices

1. Use sparingly to avoid N+1 query problems
2. Consider eager loading for known requirements
3. Monitor database round trips
4. Disable in performance-critical scenarios
5. Use async operations when possible

## Performance Considerations

- Can lead to N+1 query problem
- Multiple database roundtrips
- Hidden performance costs
- Memory usage with large datasets
- Network overhead in distributed systems

## Common Pitfalls

1. N+1 Query Problem

```csharp
// Bad practice - generates N+1 queries
foreach (var blog in context.Blogs.ToList())
{
    Console.WriteLine($"Blog {blog.Url} has {blog.Posts.Count} posts");
}

// Better approach - eager loading
var blogs = context.Blogs
    .Include(b => b.Posts)
    .ToList();
```

2. Serialization Issues

```csharp
// May cause issues with serialization
public ActionResult GetBlog(int id)
{
    var blog = context.Blogs.Find(id);
    return Json(blog); // Could trigger lazy loading during serialization
}

// Better approach
public ActionResult GetBlog(int id)
{
    var blog = context.Blogs
        .Include(b => b.Posts)
        .FirstOrDefault(b => b.Id == id);
    return Json(blog);
}
```

## Important Notes

- Requires virtual navigation properties
- Only works within context lifetime
- Can cause performance issues if not used carefully
- Not recommended for web APIs
- May cause serialization problems
- Needs proxy generation support
- Consider memory implications
