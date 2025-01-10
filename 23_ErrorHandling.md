# Error Handling and Exception Management in EF Core

## Overview

Proper error handling and exception management are crucial for building robust applications with EF Core. This guide covers common exceptions, handling strategies, and best practices for managing database-related errors.

## Key Areas

- Common exceptions
- Exception handling strategies
- Retry policies
- Logging and monitoring
- Custom exceptions
- Transaction management
- Error prevention
- Recovery strategies

## Common EF Core Exceptions

### 1. Database Exceptions

```csharp
try
{
    await context.SaveChangesAsync();
}
catch (DbUpdateException ex)
{
    // Handle database update errors
    if (ex.InnerException is SqlException sqlEx)
    {
        switch (sqlEx.Number)
        {
            case 547: // Foreign key violation
                throw new BusinessException("Related record does not exist");
            case 2601: // Unique index violation
            case 2627: // Unique constraint violation
                throw new BusinessException("Record already exists");
            default:
                throw new BusinessException("A database error occurred");
        }
    }
}
catch (DbUpdateConcurrencyException ex)
{
    // Handle concurrency conflicts
    var entry = ex.Entries.Single();
    var currentValues = entry.CurrentValues;
    var databaseValues = entry.GetDatabaseValues();

    throw new ConcurrencyException("The record was modified by another user");
}
```

### 2. Connection Exceptions

```csharp
public async Task<Blog> GetBlogAsync(int id)
{
    try
    {
        return await _context.Blogs.FindAsync(id);
    }
    catch (SqlException ex) when (ex.Number == -2)
    {
        // Timeout
        throw new TimeoutException("Database operation timed out", ex);
    }
    catch (SqlException ex) when (ex.Number == 53)
    {
        // Connection error
        throw new ConnectionException("Unable to connect to database", ex);
    }
}
```

## Retry Policies

### 1. Basic Retry Logic

```csharp
public class RetryService
{
    private readonly BlogContext _context;
    private readonly ILogger<RetryService> _logger;

    public async Task<T> ExecuteWithRetryAsync<T>(
        Func<Task<T>> operation,
        int maxRetries = 3)
    {
        for (int i = 0; i <= maxRetries; i++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (IsTransient(ex))
            {
                if (i == maxRetries) throw;

                var delay = TimeSpan.FromSeconds(Math.Pow(2, i));
                _logger.LogWarning(ex,
                    "Retry {Attempt} of {MaxRetries} after {Delay}ms",
                    i + 1, maxRetries, delay.TotalMilliseconds);

                await Task.Delay(delay);
            }
        }

        throw new Exception("Unexpected code path");
    }

    private bool IsTransient(Exception ex)
    {
        return ex is SqlException sqlEx &&
            (sqlEx.Number == -2 || // Timeout
             sqlEx.Number == 53  || // Connection error
             sqlEx.Number == 40613); // Database unavailable
    }
}
```

### 2. Polly Integration

```csharp
public static class RetryPolicyExtensions
{
    public static IServiceCollection AddEfCoreRetryPolicy(
        this IServiceCollection services)
    {
        services.AddDbContext<BlogContext>((provider, options) =>
        {
            options.UseSqlServer(
                connectionString,
                sqlOptions => sqlOptions.EnableRetryOnFailure(
                    maxRetryCount: 3,
                    maxRetryDelay: TimeSpan.FromSeconds(10),
                    errorNumbersToAdd: new[] { 4060, 40197, 40501, 40613 }
                )
            );
        });

        return services;
    }
}
```

## Custom Exception Handling

### 1. Custom Exceptions

```csharp
public class DatabaseOperationException : Exception
{
    public string Operation { get; }
    public string EntityName { get; }

    public DatabaseOperationException(
        string operation,
        string entityName,
        string message,
        Exception innerException = null)
        : base(message, innerException)
    {
        Operation = operation;
        EntityName = entityName;
    }
}

public class EntityNotFoundException : Exception
{
    public Type EntityType { get; }
    public object EntityId { get; }

    public EntityNotFoundException(Type entityType, object entityId)
        : base($"Entity of type {entityType.Name} with id {entityId} was not found")
    {
        EntityType = entityType;
        EntityId = entityId;
    }
}
```

### 2. Exception Handlers

```csharp
public class DatabaseExceptionHandler
{
    private readonly ILogger<DatabaseExceptionHandler> _logger;

    public async Task<TResult> HandleAsync<TResult>(
        Func<Task<TResult>> operation,
        string operationName)
    {
        try
        {
            return await operation();
        }
        catch (DbUpdateException ex)
        {
            _logger.LogError(ex, "Database update error during {Operation}",
                operationName);
            throw new DatabaseOperationException(
                operationName,
                GetEntityName(ex),
                "Failed to update database",
                ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error during {Operation}",
                operationName);
            throw;
        }
    }

    private string GetEntityName(DbUpdateException ex)
    {
        return ex.Entries.FirstOrDefault()?.Entity.GetType().Name ?? "Unknown";
    }
}
```

## Transaction Management

### 1. Transaction Handling

```csharp
public class BlogService
{
    private readonly BlogContext _context;

    public async Task<Blog> CreateBlogWithPostsAsync(
        Blog blog,
        IEnumerable<Post> posts)
    {
        using var transaction = await _context.Database
            .BeginTransactionAsync(IsolationLevel.ReadCommitted);
        try
        {
            _context.Blogs.Add(blog);
            await _context.SaveChangesAsync();

            foreach (var post in posts)
            {
                post.BlogId = blog.Id;
                _context.Posts.Add(post);
            }
            await _context.SaveChangesAsync();

            await transaction.CommitAsync();
            return blog;
        }
        catch (Exception)
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### 2. Savepoint Management

```csharp
public async Task ProcessBlogBatchAsync(IEnumerable<Blog> blogs)
{
    using var transaction = await _context.Database
        .BeginTransactionAsync();

    try
    {
        foreach (var batch in blogs.Chunk(100))
        {
            await transaction.CreateSavepointAsync($"Batch_{Guid.NewGuid()}");

            try
            {
                _context.Blogs.AddRange(batch);
                await _context.SaveChangesAsync();
            }
            catch (Exception)
            {
                await transaction.RollbackToSavepointAsync("Batch");
                _logger.LogWarning("Batch processing failed, continuing with next batch");
                continue;
            }
        }

        await transaction.CommitAsync();
    }
    catch (Exception)
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

## Best Practices

1. Exception Handling Middleware

```csharp
public class DatabaseExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<DatabaseExceptionMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (EntityNotFoundException ex)
        {
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Not Found",
                message = ex.Message
            });
        }
        catch (DbUpdateConcurrencyException ex)
        {
            context.Response.StatusCode = StatusCodes.Status409Conflict;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Conflict",
                message = "The record was modified by another user"
            });
        }
        catch (DatabaseOperationException ex)
        {
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Database Error",
                message = "An error occurred while processing your request"
            });

            _logger.LogError(ex, "Database operation failed");
        }
    }
}
```

2. Error Prevention

```csharp
public class BlogService
{
    private readonly BlogContext _context;

    public async Task<Blog> UpdateBlogAsync(int id, BlogDto dto)
    {
        // Validate input
        if (dto == null) throw new ArgumentNullException(nameof(dto));

        // Check existence
        var blog = await _context.Blogs.FindAsync(id) ??
            throw new EntityNotFoundException(typeof(Blog), id);

        // Optimistic concurrency
        _context.Entry(blog).Property("RowVersion")
            .OriginalValue = dto.RowVersion;

        try
        {
            // Update properties
            blog.Title = dto.Title;
            blog.Content = dto.Content;

            await _context.SaveChangesAsync();
            return blog;
        }
        catch (DbUpdateConcurrencyException)
        {
            // Handle concurrency conflict
            var currentValues = await _context.Blogs.FindAsync(id);
            if (currentValues == null)
                throw new EntityNotFoundException(typeof(Blog), id);

            throw new ConcurrencyException(
                "The blog was modified by another user",
                currentValues);
        }
    }
}
```

## Performance Considerations

- Exception handling overhead
- Retry policy impact
- Transaction isolation levels
- Logging performance
- Error tracking
- Recovery time
- Resource cleanup
- Connection management

## Important Notes

- Log all exceptions
- Use appropriate isolation levels
- Handle concurrency conflicts
- Implement retry policies
- Validate input data
- Manage transactions properly
- Monitor error rates
- Document error handling
