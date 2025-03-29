# Query Tags and Logging in EF Core

## Overview

Query tags and logging features in EF Core help with debugging, performance monitoring, and troubleshooting by providing insights into how EF Core generates and executes SQL queries.

## Key Features

- Query tagging
- Built-in logging
- Custom logging providers
- Sensitive data logging
- Performance diagnostics
- Query plan analysis
- Error tracking

## Query Tags

### 1. Basic Query Tagging

```csharp
var blogs = await context.Blogs
    .TagWith("Get all active blogs")
    .Where(b => !b.IsDeleted)
    .ToListAsync();
```

### 2. Multiple Tags

```csharp
var blogs = await context.Blogs
    .TagWith("Get high-rated blogs")
    .TagWith("Marketing Department Query")
    .Where(b => b.Rating > 4)
    .ToListAsync();
```

### 3. Structured Tags

```csharp
var blogs = await context.Blogs
    .TagWith($"""
        Query Purpose: Monthly Report
        Requested By: {user.Name}
        Department: {user.Department}
        """)
    .ToListAsync();
```

## Logging Configuration

### 1. Basic Logging Setup

```csharp
public class BloggingContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder
            .UseSqlServer(connectionString)
            .LogTo(Console.WriteLine)
            .EnableSensitiveDataLogging()
            .EnableDetailedErrors();
    }
}
```

### 2. Custom Logger Integration

```csharp
public class BloggingContext : DbContext
{
    private readonly ILogger<BloggingContext> _logger;

    public BloggingContext(ILogger<BloggingContext> logger)
    {
        _logger = logger;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.LogTo(
            message => _logger.LogInformation(message),
            LogLevel.Information,
            DbContextLoggerOptions.DefaultWithLocalTime);
    }
}
```

## Advanced Logging

### 1. Filtered Logging

```csharp
optionsBuilder.LogTo(
    message => _logger.LogInformation(message),
    (eventId, level) => level == LogLevel.Information
        && eventId.Id == RelationalEventId.CommandExecuted.Id);
```

### 2. Custom Log Formatting

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.LogTo(
        (eventData, message) =>
        {
            var formattedMessage = $"[{DateTime.Now:HH:mm:ss}] {message}";
            if (eventData.EventId.Name == "CommandExecuted")
            {
                formattedMessage = $"SQL: {formattedMessage}";
            }
            Console.WriteLine(formattedMessage);
        });
}
```

### 3. Performance Logging

```csharp
public class PerformanceLogger
{
    private readonly Dictionary<string, Stopwatch> _queryTimers = new();

    public void LogQueryStart(string queryTag)
    {
        _queryTimers[queryTag] = Stopwatch.StartNew();
    }

    public void LogQueryEnd(string queryTag)
    {
        if (_queryTimers.TryGetValue(queryTag, out var timer))
        {
            timer.Stop();
            Console.WriteLine($"Query '{queryTag}' took {timer.ElapsedMilliseconds}ms");
        }
    }
}
```

## Common Scenarios

### 1. Debug Logging

```csharp
public async Task<List<Blog>> GetBlogsWithDebugInfo()
{
    return await _context.Blogs
        .TagWith("Debug: Getting blogs with related posts")
        .Include(b => b.Posts)
        .LogTo(message => Debug.WriteLine(message))
        .ToListAsync();
}
```

### 2. Audit Logging

```csharp
public class AuditLogger
{
    private readonly BloggingContext _context;
    private readonly ILogger<AuditLogger> _logger;

    public async Task LogDatabaseOperation(string operation, string details)
    {
        var audit = new AuditLog
        {
            Operation = operation,
            Details = details,
            Timestamp = DateTime.UtcNow,
            UserId = _currentUser.Id
        };

        _context.AuditLogs.Add(audit);
        await _context.SaveChangesAsync();
    }
}
```

### 3. Performance Monitoring

```csharp
public class QueryPerformanceMonitor
{
    private readonly ILogger<QueryPerformanceMonitor> _logger;
    private readonly Dictionary<string, List<long>> _queryStats = new();

    public async Task<T> TrackQueryPerformance<T>(
        IQueryable<T> query, string queryName)
    {
        var sw = Stopwatch.StartNew();
        var result = await query
            .TagWith($"Performance tracking: {queryName}")
            .ToListAsync();
        sw.Stop();

        if (!_queryStats.ContainsKey(queryName))
        {
            _queryStats[queryName] = new List<long>();
        }
        _queryStats[queryName].Add(sw.ElapsedMilliseconds);

        LogQueryStats(queryName);
        return result;
    }

    private void LogQueryStats(string queryName)
    {
        var stats = _queryStats[queryName];
        var avg = stats.Average();
        var max = stats.Max();
        _logger.LogInformation(
            $"Query '{queryName}' stats - Avg: {avg}ms, Max: {max}ms, Count: {stats.Count}");
    }
}
```

## Best Practices

1. Query Tag Guidelines

```csharp
// DO: Use meaningful, structured tags
query.TagWith("Purpose: Daily Report, Module: Accounting");

// DON'T: Use uninformative tags
query.TagWith("Get data"); // Too vague
```

2. Logging Best Practices

```csharp
// DO: Include context in logs
optionsBuilder.LogTo(message =>
    _logger.LogInformation($"[{Environment.MachineName}] {message}"));

// DO: Handle sensitive data appropriately
optionsBuilder.EnableSensitiveDataLogging(isDevelopment);
```

## Performance Considerations

- Log level impact on performance
- Storage requirements for logs
- Query tag overhead
- Logging provider efficiency
- Debug vs Release logging
- Log rotation strategies
- Monitoring overhead

## Important Notes

- Secure sensitive data in logs
- Consider log storage costs
- Monitor logging performance
- Use appropriate log levels
- Implement log rotation
- Plan log retention
- Consider compliance requirements
- Test logging in production scenarios
