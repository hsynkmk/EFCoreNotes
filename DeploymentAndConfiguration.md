# Deployment and Configuration in EF Core

## Overview

Proper deployment and configuration of EF Core applications is crucial for successful production environments. This guide covers deployment strategies, configuration management, and best practices for different hosting scenarios.

## Key Areas

- Environment configuration
- Connection string management
- Migration deployment
- Database initialization
- Hosting scenarios
- Configuration providers
- Deployment strategies
- Monitoring setup

## Environment Configuration

### 1. Configuration Setup

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            })
            .ConfigureAppConfiguration((hostingContext, config) =>
            {
                var env = hostingContext.HostingEnvironment;

                config.SetBasePath(env.ContentRootPath)
                    .AddJsonFile("appsettings.json", optional: false)
                    .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
                    .AddEnvironmentVariables()
                    .AddUserSecrets<Program>(optional: true)
                    .AddCommandLine(args);
            });
}
```

### 2. Environment-Specific Settings

```json
// appsettings.Development.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=BlogDb;Trusted_Connection=True;"
  },
  "DatabaseOptions": {
    "CommandTimeout": 30,
    "EnableDetailedErrors": true,
    "EnableSensitiveDataLogging": true
  }
}

// appsettings.Production.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=prod-server;Database=BlogDb;User Id=app_user;Password=***;"
  },
  "DatabaseOptions": {
    "CommandTimeout": 60,
    "EnableDetailedErrors": false,
    "EnableSensitiveDataLogging": false,
    "MaxRetryCount": 3,
    "MaxRetryDelay": "00:00:10"
  }
}
```

## Database Configuration

### 1. DbContext Configuration

```csharp
public class BlogContext : DbContext
{
    private readonly IConfiguration _configuration;
    private readonly IHostEnvironment _environment;

    public BlogContext(
        DbContextOptions<BlogContext> options,
        IConfiguration configuration,
        IHostEnvironment environment)
        : base(options)
    {
        _configuration = configuration;
        _environment = environment;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            var connectionString = _configuration.GetConnectionString("DefaultConnection");
            var dbOptions = _configuration.GetSection("DatabaseOptions").Get<DatabaseOptions>();

            optionsBuilder.UseSqlServer(connectionString, options =>
            {
                options.CommandTimeout(dbOptions.CommandTimeout);
                options.EnableRetryOnFailure(
                    maxRetryCount: dbOptions.MaxRetryCount,
                    maxRetryDelay: dbOptions.MaxRetryDelay,
                    errorNumbersToAdd: null);
            });

            if (_environment.IsDevelopment())
            {
                optionsBuilder
                    .EnableDetailedErrors()
                    .EnableSensitiveDataLogging();
            }
        }
    }
}
```

### 2. Options Pattern

```csharp
public class DatabaseOptions
{
    public int CommandTimeout { get; set; }
    public bool EnableDetailedErrors { get; set; }
    public bool EnableSensitiveDataLogging { get; set; }
    public int MaxRetryCount { get; set; }
    public TimeSpan MaxRetryDelay { get; set; }
}

public void ConfigureServices(IServiceCollection services)
{
    services.Configure<DatabaseOptions>(
        Configuration.GetSection("DatabaseOptions"));

    services.AddDbContext<BlogContext>((provider, options) =>
    {
        var dbOptions = provider.GetRequiredService<IOptions<DatabaseOptions>>().Value;
        options.UseSqlServer(
            Configuration.GetConnectionString("DefaultConnection"),
            sqlOptions => ConfigureSqlOptions(sqlOptions, dbOptions));
    });
}
```

## Migration Deployment

### 1. Migration Scripts

```csharp
public static class DatabaseMigrationExtensions
{
    public static IHost MigrateDatabase<TContext>(this IHost host)
        where TContext : DbContext
    {
        using (var scope = host.Services.CreateScope())
        {
            var services = scope.ServiceProvider;
            var logger = services.GetRequiredService<ILogger<TContext>>();
            var context = services.GetRequiredService<TContext>();

            try
            {
                logger.LogInformation("Migrating database associated with context {DbContextName}", typeof(TContext).Name);

                var strategy = context.Database.CreateExecutionStrategy();

                strategy.Execute(() =>
                {
                    if (context.Database.GetPendingMigrations().Any())
                    {
                        context.Database.Migrate();
                    }
                });

                logger.LogInformation("Migrated database associated with context {DbContextName}", typeof(TContext).Name);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "An error occurred while migrating the database used on context {DbContextName}", typeof(TContext).Name);
                throw;
            }
        }

        return host;
    }
}
```

### 2. Script Generation

```csharp
public class MigrationScriptGenerator
{
    public static async Task GenerateScriptAsync(string connectionString, string outputPath)
    {
        var optionsBuilder = new DbContextOptionsBuilder<BlogContext>();
        optionsBuilder.UseSqlServer(connectionString);

        using var context = new BlogContext(optionsBuilder.Options);
        var script = context.Database.GenerateCreateScript();

        await File.WriteAllTextAsync(outputPath, script);
    }
}

// Usage in deployment pipeline
public class DeploymentManager
{
    public async Task GenerateDeploymentScripts(string environment)
    {
        var connectionString = GetConnectionString(environment);
        var scriptPath = $"./migrations/{environment}_migration.sql";

        await MigrationScriptGenerator.GenerateScriptAsync(
            connectionString, scriptPath);
    }
}
```

## Deployment Strategies

### 1. Automated Deployment

```csharp
public static class WebApplicationExtensions
{
    public static async Task<IHost> InitializeDatabaseAsync(this IHost host)
    {
        using var scope = host.Services.CreateScope();
        var services = scope.ServiceProvider;
        var logger = services.GetRequiredService<ILogger<Program>>();
        var context = services.GetRequiredService<BlogContext>();

        try
        {
            logger.LogInformation("Initializing database...");

            if (context.Database.IsSqlServer())
            {
                await context.Database.MigrateAsync();
                await SeedData.InitializeAsync(services);
            }

            logger.LogInformation("Database initialized successfully");
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "An error occurred while initializing the database");
            throw;
        }

        return host;
    }
}
```

### 2. Health Checks

```csharp
public static class HealthCheckExtensions
{
    public static IServiceCollection AddDatabaseHealthChecks(
        this IServiceCollection services)
    {
        services.AddHealthChecks()
            .AddDbContextCheck<BlogContext>("database", tags: new[] { "ready" })
            .AddSqlServer(
                connectionString: Configuration.GetConnectionString("DefaultConnection"),
                healthQuery: "SELECT 1;",
                name: "sql",
                tags: new[] { "db", "sql", "sqlserver" });

        return services;
    }
}

public void Configure(IApplicationBuilder app)
{
    app.UseHealthChecks("/health", new HealthCheckOptions
    {
        ResponseWriter = async (context, report) =>
        {
            context.Response.ContentType = "application/json";

            var response = new
            {
                status = report.Status.ToString(),
                checks = report.Entries.Select(e => new
                {
                    name = e.Key,
                    status = e.Value.Status.ToString(),
                    duration = e.Value.Duration
                })
            };

            await context.Response.WriteAsJsonAsync(response);
        }
    });
}
```

## Best Practices

1. Configuration Management

```csharp
// DO: Use strongly-typed configuration
public class DatabaseSettings
{
    public string ConnectionString { get; set; }
    public int CommandTimeout { get; set; }
    public RetryOptions RetryOptions { get; set; }
}

public class RetryOptions
{
    public int MaxRetryCount { get; set; }
    public int MaxRetryDelaySeconds { get; set; }
}

// Usage
services.Configure<DatabaseSettings>(Configuration.GetSection("Database"));
```

2. Secure Configuration

```csharp
// DO: Use secure configuration providers
public void ConfigureServices(IServiceCollection services)
{
    var keyVaultUrl = Configuration["KeyVault:Url"];
    var credential = new DefaultAzureCredential();

    services.AddDbContext<BlogContext>((provider, options) =>
    {
        var secretClient = new SecretClient(new Uri(keyVaultUrl), credential);
        var connectionString = secretClient.GetSecret("DatabaseConnection").Value.Value;

        options.UseSqlServer(connectionString);
    });
}
```

## Performance Considerations

- Connection pooling
- Command timeout settings
- Migration performance
- Deployment downtime
- Resource allocation
- Monitoring overhead
- Backup strategies
- Scaling options

## Important Notes

- Environment-specific settings
- Security considerations
- Backup procedures
- Rollback strategies
- Monitoring setup
- Documentation requirements
- Compliance requirements
- Disaster recovery
