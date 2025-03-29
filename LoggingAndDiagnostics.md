# Logging and Diagnostics in EF Core

## Overview

EF Core provides comprehensive logging and diagnostics capabilities to help developers monitor, debug, and optimize their database operations. This documentation covers the various logging options, diagnostic tools, and best practices for monitoring EF Core applications.

## Key Features

- Built-in logging providers
- SQL query logging
- Performance diagnostics
- Error tracking
- Event counters
- Integration with logging frameworks
- Debug views

## Basic Logging Setup

### 1. Configure Logging

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(connectionString)
        .LogTo(Console.WriteLine)                    // Basic console logging
        .EnableSensitiveDataLogging()               // Include parameter values
        .EnableDetailedErrors();                    // Detailed error messages
}
```

### 2. Using ILoggerFactory

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<BlogContext>(options =>
        options.UseSqlServer(connectionString)
            .UseLoggerFactory(LoggerFactory.Create(builder =>
                builder
                    .AddConsole()
                    .AddDebug()
                    .SetMinimumLevel(LogLevel.Information)
            ))
    );
}
```

## Logging Categories

### 1. Database Commands

```csharp
optionsBuilder.LogTo(
    (eventId, logLevel) => eventId.Id == CoreEventId.ExecutedCommand.Id,
    message => Debug.WriteLine(message)
);
```

### 2. Query Compilation

```csharp
optionsBuilder.LogTo(
    (eventId, logLevel) =>
        eventId.Id == CoreEventId.QueryCompiled.Id,
    message => Debug.WriteLine(message)
);
```

## Advanced Logging Configuration

### 1. Custom Filtering

```csharp
optionsBuilder.LogTo(
    Console.WriteLine,
    (eventId, level) => level >= LogLevel.Warning
        || (level == LogLevel.Information
            && eventId.Id == CoreEventId.ExecutedCommand.Id)
);
```

### 2. Custom Formatting

```csharp
optionsBuilder.LogTo(
    Console.WriteLine,
    new[] { DbLoggerCategory.Database.Command.Name },
    LogLevel.Information,
    DbContextLoggerOptions.SingleLine | DbContextLoggerOptions.UtcTime
);
```

## Performance Diagnostics

### 1. Query Tags

```csharp
var blogs = await context.Blogs
    .TagWith("Get all active blogs")
    .Where(b => !b.IsDeleted)
    .ToListAsync();
```

### 2. Execution Statistics

```csharp
var stats = context.Database.GetDbConnection()
    .RetrieveStatistics();
// Access various statistics
Console.WriteLine($"Execution Time: {stats["ExecutionTime"]}");
```

## Event Counters

### 1. Enable Event Counters

```csharp
services.AddDbContext<BlogContext>(options =>
    options.UseSqlServer(connectionString)
        .EnableServiceProviderCaching(false)
        .EnableThreadSafetyChecks()
);
```

### 2. Monitor Event Counters

```powershell
dotnet counters monitor Microsoft.EntityFrameworkCore
```

## Debug Views

### 1. Change Tracker Debug View

```csharp
var debugView = context.ChangeTracker.DebugView;
Console.WriteLine(debugView.LongView);
```

### 2. Model Debug View

```csharp
var modelDebugView = context.Model.ToDebugString();
Console.WriteLine(modelDebugView);
```

## Integration with Popular Logging Frameworks

### 1. Serilog Integration

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .WriteTo.Console()
    .CreateLogger();

services.AddDbContext<BlogContext>(options =>
    options.UseSqlServer(connectionString)
        .UseLoggerFactory(LoggerFactory.Create(builder =>
            builder.AddSerilog(dispose: true)))
);
```

### 2. NLog Integration

```csharp
var logger = NLog.LogManager.GetCurrentClassLogger();

services.AddDbContext<BlogContext>(options =>
    options.UseSqlServer(connectionString)
        .UseLoggerFactory(LoggerFactory.Create(builder =>
            builder.AddNLog()))
);
```

## Performance Monitoring

### 1. Query Execution Time

```csharp
context.Database.SetCommandTimeout(TimeSpan.FromSeconds(30));

var stopwatch = Stopwatch.StartNew();
var result = await context.Blogs
    .TagWith($"Query executed at {DateTime.UtcNow}")
    .ToListAsync();
stopwatch.Stop();

Console.WriteLine($"Query took {stopwatch.ElapsedMilliseconds}ms");
```

### 2. Connection Monitoring

```csharp
context.Database.GetDbConnection().StateChange += (sender, e) =>
{
    Console.WriteLine($"Connection state changed from {e.OriginalState} to {e.CurrentState}");
};
```

## Best Practices

1. Production Logging

```csharp
// DO: Use appropriate log levels
optionsBuilder.LogTo(
    (eventId, logLevel) => logLevel >= LogLevel.Warning,
    message => _logger.LogWarning(message)
);

// DON'T: Enable sensitive data logging in production
// optionsBuilder.EnableSensitiveDataLogging(); // Avoid in production
```

2. Performance Optimization

```csharp
// DO: Use query tags for identification
var blogs = await context.Blogs
    .TagWith("GetActiveBlogsQuery")
    .Where(b => !b.IsDeleted)
    .ToListAsync();

// DO: Monitor query execution time
using var command = context.Database.GetDbConnection().CreateCommand();
command.CommandText = "SELECT COUNT(*) FROM Blogs";
var start = DateTime.UtcNow;
var result = await command.ExecuteScalarAsync();
var duration = DateTime.UtcNow - start;
```

## Performance Considerations

- Logging overhead
- Debug view impact
- Event counter resource usage
- Connection pooling monitoring
- Query plan caching
- Memory usage tracking
- I/O operations monitoring

## Important Notes

- Secure sensitive data in logs
- Configure appropriate log levels
- Monitor resource usage
- Regular performance auditing
- Log rotation and retention
- Error handling strategies
- Production vs development logging
- Compliance requirements
