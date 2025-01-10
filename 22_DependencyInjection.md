# Dependency Injection and Service Lifetime in EF Core

## Overview

Understanding dependency injection (DI) and service lifetime management is crucial for building maintainable applications with EF Core. This guide covers best practices for registering and using DbContext with dependency injection.

## Key Concepts

- DbContext registration
- Service lifetimes
- Scoped operations
- Factory patterns
- Unit of work
- Repository pattern
- Service configuration
- Lifetime management

## DbContext Registration

### 1. Basic Registration

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<BlogContext>(options =>
        options.UseSqlServer(
            Configuration.GetConnectionString("DefaultConnection")
        )
    );
}
```

### 2. Advanced Configuration

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<BlogContext>((serviceProvider, options) =>
    {
        var config = serviceProvider.GetRequiredService<IConfiguration>();
        var logger = serviceProvider.GetRequiredService<ILogger<BlogContext>>();

        options.UseSqlServer(config.GetConnectionString("DefaultConnection"))
            .EnableSensitiveDataLogging()
            .UseLoggerFactory(LoggerFactory.Create(builder =>
                builder.AddProvider(new CustomLoggerProvider(logger))));
    });
}
```

## Service Lifetimes

### 1. Scoped DbContext

```csharp
// Registration
services.AddDbContext<BlogContext>(options =>
    options.UseSqlServer(connectionString),
    ServiceLifetime.Scoped // Default lifetime
);

// Usage in Controller
public class BlogController : ControllerBase
{
    private readonly BlogContext _context;

    public BlogController(BlogContext context)
    {
        _context = context;
    }

    public async Task<IActionResult> Get()
    {
        return Ok(await _context.Blogs.ToListAsync());
    }
}
```

### 2. Pooled DbContext

```csharp
// Registration with pooling
services.AddDbContextPool<BlogContext>(options =>
    options.UseSqlServer(connectionString),
    poolSize: 32 // Maximum number of instances
);

// Factory pattern for custom scopes
services.AddSingleton<IBlogContextFactory>(sp =>
{
    var pooledFactory = sp.GetRequiredService<IDbContextFactory<BlogContext>>();
    return new BlogContextFactory(pooledFactory);
});
```

## Repository Pattern

### 1. Generic Repository

```csharp
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
}

public class Repository<T> : IRepository<T> where T : class
{
    private readonly BlogContext _context;
    private readonly DbSet<T> _dbSet;

    public Repository(BlogContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T> GetByIdAsync(int id)
    {
        return await _dbSet.FindAsync(id);
    }

    public async Task<IEnumerable<T>> GetAllAsync()
    {
        return await _dbSet.ToListAsync();
    }

    public async Task AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
        await _context.SaveChangesAsync();
    }

    public async Task UpdateAsync(T entity)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync();
    }

    public async Task DeleteAsync(T entity)
    {
        _dbSet.Remove(entity);
        await _context.SaveChangesAsync();
    }
}
```

### 2. Specific Repository

```csharp
public interface IBlogRepository : IRepository<Blog>
{
    Task<IEnumerable<Blog>> GetBlogsByAuthorAsync(int authorId);
    Task<Blog> GetBlogWithPostsAsync(int blogId);
}

public class BlogRepository : Repository<Blog>, IBlogRepository
{
    private readonly BlogContext _context;

    public BlogRepository(BlogContext context) : base(context)
    {
        _context = context;
    }

    public async Task<IEnumerable<Blog>> GetBlogsByAuthorAsync(int authorId)
    {
        return await _context.Blogs
            .Where(b => b.AuthorId == authorId)
            .ToListAsync();
    }

    public async Task<Blog> GetBlogWithPostsAsync(int blogId)
    {
        return await _context.Blogs
            .Include(b => b.Posts)
            .FirstOrDefaultAsync(b => b.Id == blogId);
    }
}
```

## Unit of Work Pattern

### 1. Unit of Work Implementation

```csharp
public interface IUnitOfWork : IDisposable
{
    IBlogRepository Blogs { get; }
    IPostRepository Posts { get; }
    Task<int> SaveChangesAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly BlogContext _context;
    private IBlogRepository _blogRepository;
    private IPostRepository _postRepository;

    public UnitOfWork(BlogContext context)
    {
        _context = context;
    }

    public IBlogRepository Blogs =>
        _blogRepository ??= new BlogRepository(_context);

    public IPostRepository Posts =>
        _postRepository ??= new PostRepository(_context);

    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}
```

### 2. Service Registration

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Register DbContext
    services.AddDbContext<BlogContext>();

    // Register repositories
    services.AddScoped<IBlogRepository, BlogRepository>();
    services.AddScoped<IPostRepository, PostRepository>();

    // Register unit of work
    services.AddScoped<IUnitOfWork, UnitOfWork>();
}
```

## Factory Pattern

### 1. DbContext Factory

```csharp
public interface IBlogContextFactory
{
    BlogContext CreateContext();
}

public class BlogContextFactory : IBlogContextFactory
{
    private readonly IDbContextFactory<BlogContext> _pooledFactory;

    public BlogContextFactory(IDbContextFactory<BlogContext> pooledFactory)
    {
        _pooledFactory = pooledFactory;
    }

    public BlogContext CreateContext()
    {
        return _pooledFactory.CreateDbContext();
    }
}
```

### 2. Usage in Services

```csharp
public class BlogService
{
    private readonly IBlogContextFactory _contextFactory;

    public BlogService(IBlogContextFactory contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public async Task ProcessBlogsAsync()
    {
        using var context = _contextFactory.CreateContext();
        // Use context within a specific scope
    }
}
```

## Best Practices

1. Service Registration

```csharp
// DO: Register dependencies with appropriate lifetimes
services.AddScoped<IBlogService, BlogService>();
services.AddTransient<IValidator<Blog>, BlogValidator>();
services.AddSingleton<IBlogContextFactory, BlogContextFactory>();

// DON'T: Use singleton for DbContext
// services.AddSingleton<BlogContext>(); // Wrong!
```

2. Dependency Resolution

```csharp
// DO: Constructor injection
public class BlogController : ControllerBase
{
    private readonly IBlogService _blogService;
    private readonly ILogger<BlogController> _logger;

    public BlogController(
        IBlogService blogService,
        ILogger<BlogController> logger)
    {
        _blogService = blogService;
        _logger = logger;
    }
}
```

## Performance Considerations

- Connection pooling
- Context pooling
- Service resolution
- Scope management
- Memory usage
- Disposal patterns
- Transaction handling
- Resource cleanup

## Important Notes

- Proper disposal
- Lifetime management
- Scope boundaries
- Thread safety
- Resource sharing
- Error handling
- Testing considerations
- Performance impact
