# Database Providers in EF Core

## Overview

EF Core supports multiple database engines through its provider model. Each provider enables EF Core to work with a specific database engine, translating EF Core operations into database-specific commands and queries.

## Supported Providers

### 1. Microsoft SQL Server

```csharp
// Installation
// Install-Package Microsoft.EntityFrameworkCore.SqlServer

// Configuration
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(
        "Server=.;Database=BlogDb;Trusted_Connection=True;",
        options => options
            .EnableRetryOnFailure()
            .MigrationsHistoryTable("__EFMigrationsHistory")
    );
}
```

### 2. SQLite

```csharp
// Installation
// Install-Package Microsoft.EntityFrameworkCore.Sqlite

// Configuration
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlite(
        "Data Source=blog.db",
        options => options
            .MigrationsHistoryTable("__EFMigrationsHistory")
    );
}
```

### 3. PostgreSQL (Npgsql)

```csharp
// Installation
// Install-Package Npgsql.EntityFrameworkCore.PostgreSQL

// Configuration
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseNpgsql(
        "Host=localhost;Database=blogdb;Username=myuser;Password=mypass",
        options => options
            .EnableRetryOnFailure()
            .MigrationsHistoryTable("__EFMigrationsHistory")
    );
}
```

### 4. MySQL/MariaDB

```csharp
// Installation
// Install-Package Pomelo.EntityFrameworkCore.MySql

// Configuration
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseMySql(
        "Server=localhost;Database=blogdb;User=myuser;Password=mypass;",
        ServerVersion.AutoDetect(connectionString),
        options => options
            .EnableRetryOnFailure()
            .MigrationsHistoryTable("__EFMigrationsHistory")
    );
}
```

### 5. Oracle

```csharp
// Installation
// Install-Package Oracle.EntityFrameworkCore

// Configuration
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseOracle(
        "Data Source=localhost:1521/XEPDB1;User Id=myuser;Password=mypass",
        options => options
            .MigrationsHistoryTable("__EFMigrationsHistory")
    );
}
```

## Provider-Specific Features

### 1. SQL Server Features

```csharp
// Memory-Optimized Tables
modelBuilder.Entity<Blog>()
    .ToTable("Blogs", b => b.IsMemoryOptimized());

// Temporal Tables
modelBuilder.Entity<Blog>()
    .ToTable("Blogs", b => b.IsTemporal());

// Sparse Columns
modelBuilder.Entity<Blog>()
    .Property(b => b.OptionalField)
    .IsSparse();
```

### 2. PostgreSQL Features

```csharp
// JSON Columns
modelBuilder.Entity<Blog>()
    .Property(b => b.Metadata)
    .HasColumnType("jsonb");

// Array Types
modelBuilder.Entity<Blog>()
    .Property(b => b.Tags)
    .HasColumnType("text[]");

// Enum Types
modelBuilder.HasPostgresEnum<BlogStatus>();
```

### 3. SQLite Features

```csharp
// SQLite Limitations
modelBuilder.Entity<Blog>()
    .Property(b => b.Rating)
    .HasConversion<double>();  // SQLite doesn't support decimal

// Connection Configuration
optionsBuilder.UseSqlite(
    connection,
    x => x.SuppressForeignKeyEnforcement()
);
```

## Provider Configuration

### 1. Connection Resiliency

```csharp
// SQL Server
optionsBuilder.UseSqlServer(
    connectionString,
    options => options
        .EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorNumbersToAdd: null)
);

// PostgreSQL
optionsBuilder.UseNpgsql(
    connectionString,
    options => options
        .EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorCodesToAdd: null)
);
```

### 2. Command Timeout

```csharp
// Global Timeout
optionsBuilder.UseSqlServer(
    connectionString,
    options => options.CommandTimeout(60)
);

// Per-Context Timeout
context.Database.SetCommandTimeout(TimeSpan.FromMinutes(2));

// Per-Query Timeout
context.Database.SetCommandTimeout(120);
```

## Migration Considerations

### 1. Provider-Specific SQL

```csharp
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        if (Database.IsSqlServer())
        {
            migrationBuilder.Sql("CREATE FULLTEXT INDEX...");
        }
        else if (Database.IsNpgsql())
        {
            migrationBuilder.Sql("CREATE INDEX USING GiST...");
        }
    }
}
```

### 2. Data Type Mappings

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Title)
    .HasColumnType(
        Database.IsSqlServer() ? "nvarchar(450)" :
        Database.IsNpgsql() ? "varchar(450)" :
        "varchar(450)");
```

## Best Practices

1. Connection Management

```csharp
// DO: Use connection pooling
services.AddDbContext<BlogContext>(options =>
    options.UseSqlServer(connectionString,
        sqlOptions => sqlOptions
            .MaxBatchSize(100)
            .EnableRetryOnFailure())
);

// DON'T: Disable connection pooling without good reason
// "Pooling=false" // Avoid unless necessary
```

2. Provider Selection

```csharp
// DO: Consider deployment environment
var provider = Configuration["Database:Provider"];
options => {
    switch (provider.ToLower())
    {
        case "sqlserver":
            options.UseSqlServer(connectionString);
            break;
        case "postgresql":
            options.UseNpgsql(connectionString);
            break;
        default:
            throw new ArgumentException($"Unsupported provider: {provider}");
    }
}
```

## Performance Considerations

- Connection pooling settings
- Command timeout configuration
- Batch size optimization
- Provider-specific indexes
- Query translation efficiency
- Transaction handling
- Concurrency strategies

## Important Notes

- Provider compatibility
- Version requirements
- Migration limitations
- Feature availability
- Performance characteristics
- Security considerations
- Deployment requirements
- Monitoring capabilities
