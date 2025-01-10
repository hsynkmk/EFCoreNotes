# Monitoring and Diagnostics in EF Core

## Overview

Effective monitoring and diagnostics are essential for maintaining and optimizing EF Core applications. This guide covers tools and techniques for monitoring performance, troubleshooting issues, and gathering insights about your application's database interactions.

## Key Areas

- Performance monitoring
- Query analysis
- Logging configuration
- Diagnostic tools
- Metrics collection
- Health checks
- Profiling
- Troubleshooting

## Performance Monitoring

### 1. Query Logging

```csharp
public class BlogContext : DbContext
{
    private static readonly ILoggerFactory _loggerFactory =
        LoggerFactory.Create(builder =>
        {
            builder
                .AddConsole()
                .AddDebug()
                .AddFilter(DbLoggerCategory.Database.Command.Name,
                    LogLevel.Information);
        });

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder
            .UseLoggerFactory(_loggerFactory)
            .EnableSensitiveDataLogging()
            .EnableDetailedErrors()
            .LogTo(Console.WriteLine, new[]
            {
                DbLoggerCategory.Database.Command.Name,
                DbLoggerCategory.Query.Name
            });
    }
}
```

### 2. Performance Metrics

```csharp
public class DatabaseMetrics
{
    private readonly ILogger<DatabaseMetrics> _logger;
    private readonly Metrics _metrics;

    public async Task<T> MeasureQueryExecutionAsync<T>(
        Func<Task<T>> query,
        string queryName)
    {
        using var timer = _metrics.CreateTimer("query_execution",
            new MetricTags("query", queryName));

        try
        {
            return await query();
        }
        catch (Exception ex)
        {
            _metrics.Increment("query_errors",
                new MetricTags("query", queryName));
            throw;
        }
    }
}

public class BlogService
{
    private readonly BlogContext _context;
    private readonly DatabaseMetrics _metrics;

    public async Task<List<Blog>> GetBlogsAsync()
    {
        return await _metrics.MeasureQueryExecutionAsync(
            async () => await _context.Blogs.ToListAsync(),
            "GetAllBlogs");
    }
}
```

## Query Analysis

### 1. Query Interceptor

```csharp
public class QueryInterceptor : DbCommandInterceptor
{
    private readonly ILogger<QueryInterceptor> _logger;

    public override ValueTask<InterceptionResult<DbDataReader>> ReaderExecutingAsync(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<DbDataReader> result,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation(
            "Executing query: {CommandText} with parameters: {@Parameters}",
            command.CommandText,
            command.Parameters.Cast<DbParameter>()
                .Select(p => new { p.ParameterName, p.Value }));

        return base.ReaderExecutingAsync(command, eventData, result, cancellationToken);
    }

    public override ValueTask<InterceptionResult<int>> NonQueryExecutingAsync(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        if (command.CommandText.StartsWith("INSERT") ||
            command.CommandText.StartsWith("UPDATE") ||
            command.CommandText.StartsWith("DELETE"))
        {
            _logger.LogWarning(
                "Executing modification query: {CommandText}",
                command.CommandText);
        }

        return base.NonQueryExecutingAsync(command, eventData, result, cancellationToken);
    }
}
```

### 2. Query Statistics

```csharp
public class QueryStatisticsCollector
{
    private readonly ConcurrentDictionary<string, QueryStats> _statistics =
        new ConcurrentDictionary<string, QueryStats>();

    public void RecordQuery(string query, TimeSpan duration)
    {
        var stats = _statistics.GetOrAdd(query, _ => new QueryStats());
        stats.RecordExecution(duration);
    }

    public IEnumerable<QueryAnalysis> GetSlowQueries(TimeSpan threshold)
    {
        return _statistics
            .Where(kvp => kvp.Value.AverageDuration > threshold)
            .Select(kvp => new QueryAnalysis
            {
                Query = kvp.Key,
                ExecutionCount = kvp.Value.ExecutionCount,
                AverageDuration = kvp.Value.AverageDuration,
                LastExecuted = kvp.Value.LastExecuted
            })
            .OrderByDescending(a => a.AverageDuration);
    }
}

public class QueryStats
{
    private readonly ConcurrentQueue<TimeSpan> _durations =
        new ConcurrentQueue<TimeSpan>();
    private const int MaxSamples = 100;

    public int ExecutionCount { get; private set; }
    public DateTime LastExecuted { get; private set; }

    public TimeSpan AverageDuration =>
        _durations.Any()
            ? TimeSpan.FromTicks((long)_durations.Average(d => d.Ticks))
            : TimeSpan.Zero;

    public void RecordExecution(TimeSpan duration)
    {
        _durations.Enqueue(duration);
        while (_durations.Count > MaxSamples)
        {
            _durations.TryDequeue(out _);
        }

        ExecutionCount++;
        LastExecuted = DateTime.UtcNow;
    }
}
```

## Health Monitoring

### 1. Database Health Checks

```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly BlogContext _context;
    private readonly ILogger<DatabaseHealthCheck> _logger;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Check database connectivity
            await _context.Database.CanConnectAsync(cancellationToken);

            // Check for pending migrations
            var pendingMigrations = await _context.Database
                .GetPendingMigrationsAsync(cancellationToken);

            if (pendingMigrations.Any())
            {
                return HealthCheckResult.Degraded(
                    "Database needs migration",
                    new Dictionary<string, object>
                    {
                        { "PendingMigrations", pendingMigrations }
                    });
            }

            // Check connection pool
            var stats = await GetConnectionStats();

            return HealthCheckResult.Healthy("Database is healthy",
                new Dictionary<string, object>
                {
                    { "ConnectionPoolSize", stats.PoolSize },
                    { "ActiveConnections", stats.ActiveConnections }
                });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database health check failed");
            return HealthCheckResult.Unhealthy("Database is unhealthy", ex);
        }
    }
}
```

### 2. Connection Monitoring

```csharp
public class ConnectionMonitor
{
    private readonly ILogger<ConnectionMonitor> _logger;
    private readonly Counter _activeConnections;
    private readonly Counter _failedConnections;

    public void TrackConnection(DbConnection connection)
    {
        connection.StateChange += (sender, args) =>
        {
            if (args.CurrentState == ConnectionState.Open)
            {
                _activeConnections.Increment();
                _logger.LogDebug("Database connection opened. Active connections: {Count}",
                    _activeConnections.Value);
            }
            else if (args.CurrentState == ConnectionState.Closed)
            {
                _activeConnections.Decrement();
                _logger.LogDebug("Database connection closed. Active connections: {Count}",
                    _activeConnections.Value);
            }
        };
    }
}
```

## Diagnostic Tools

### 1. Query Plan Analyzer

```csharp
public class QueryPlanAnalyzer
{
    private readonly BlogContext _context;

    public async Task<QueryPlanAnalysis> AnalyzeQueryAsync(
        IQueryable query,
        string queryName)
    {
        var command = _context.Database.GetDbConnection().CreateCommand();
        command.CommandText = query.ToQueryString();

        await using var reader = await command.ExecuteReaderAsync();
        var plan = await reader.GetExecutionPlanAsync();

        return new QueryPlanAnalysis
        {
            QueryName = queryName,
            ExecutionPlan = plan,
            Warnings = AnalyzePlanWarnings(plan),
            Recommendations = GenerateOptimizationRecommendations(plan)
        };
    }

    private IEnumerable<string> AnalyzePlanWarnings(ExecutionPlan plan)
    {
        var warnings = new List<string>();

        // Check for table scans
        if (plan.ContainsTableScan)
        {
            warnings.Add("Query contains table scan operations");
        }

        // Check for missing indexes
        if (plan.HasMissingIndexSuggestions)
        {
            warnings.Add("Query could benefit from additional indexes");
        }

        return warnings;
    }
}
```

### 2. Performance Profiler

```csharp
public class DatabaseProfiler
{
    private readonly ILogger<DatabaseProfiler> _logger;
    private readonly Stopwatch _stopwatch;

    public async Task<ProfilerReport> ProfileQueryAsync(
        Func<Task> operation,
        string operationName)
    {
        var report = new ProfilerReport { OperationName = operationName };

        _stopwatch.Restart();
        try
        {
            await operation();
            report.Duration = _stopwatch.Elapsed;

            // Collect additional metrics
            report.CpuUsage = GetCurrentCpuUsage();
            report.MemoryUsage = GetCurrentMemoryUsage();
            report.IoOperations = GetIoOperations();

            return report;
        }
        finally
        {
            _stopwatch.Stop();
            _logger.LogInformation(
                "Operation {Operation} completed in {Duration}ms",
                operationName,
                _stopwatch.ElapsedMilliseconds);
        }
    }
}
```

## Best Practices

1. Logging Strategy

```csharp
// DO: Use structured logging
public class DatabaseLogger
{
    private readonly ILogger<DatabaseLogger> _logger;

    public void LogQueryExecution(
        string query,
        TimeSpan duration,
        int rowCount)
    {
        _logger.LogInformation(
            "Query executed: {Query} in {Duration}ms returning {RowCount} rows",
            query, duration.TotalMilliseconds, rowCount);
    }
}
```

2. Performance Monitoring

```csharp
// DO: Monitor key metrics
public class PerformanceMonitor
{
    private readonly ILogger<PerformanceMonitor> _logger;
    private readonly IMetrics _metrics;

    public async Task MonitorQueryPerformance(
        Func<Task> operation,
        string queryName)
    {
        using var timer = _metrics.Measure("query_duration");
        var memory = GC.GetTotalMemory(false);

        try
        {
            await operation();
        }
        finally
        {
            var memoryDelta = GC.GetTotalMemory(false) - memory;
            _metrics.Gauge("query_memory_usage", memoryDelta);
        }
    }
}
```

## Performance Considerations

- Logging overhead
- Monitoring impact
- Resource usage
- Sampling strategies
- Data retention
- Query complexity
- Connection pooling
- Memory management

## Important Notes

- Monitor production carefully
- Set up alerts
- Regular analysis
- Performance baselines
- Resource thresholds
- Security implications
- Data privacy
- Backup monitoring
