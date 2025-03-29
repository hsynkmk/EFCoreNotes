# DbContext and Entities in EF Core

## Overview

DbContext and entities are the foundation of Entity Framework Core. DbContext represents a session with the database, while entities are the .NET classes that map to database tables.

## Key Features

- Database connection management
- Entity configuration
- Change tracking
- Query management
- CRUD operations
- Transaction handling
- Model building

## DbContext Basics

### 1. Creating a DbContext

```csharp
public class BloggingContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer(
            "Server=(localdb)\\mssqllocaldb;Database=Blogging;Trusted_Connection=True");
    }
}
```

### 2. Dependency Injection Setup

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<BloggingContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
}

// Usage in a service
public class BlogService
{
    private readonly BloggingContext _context;

    public BlogService(BloggingContext context)
    {
        _context = context;
    }
}
```

## Entity Configuration

### 1. Basic Entity

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; }
    public string Name { get; set; }
    public List<Post> Posts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public int BlogId { get; set; }
    public Blog Blog { get; set; }
}
```

### 2. Using Data Annotations

```csharp
public class Blog
{
    [Key]
    public int Id { get; set; }

    [Required]
    [MaxLength(500)]
    public string Url { get; set; }

    [StringLength(200)]
    public string Name { get; set; }

    [InverseProperty("Blog")]
    public List<Post> Posts { get; set; }
}
```

### 3. Using Fluent API

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>(entity =>
    {
        entity.HasKey(e => e.Id);
        entity.Property(e => e.Url).IsRequired().HasMaxLength(500);
        entity.Property(e => e.Name).HasMaxLength(200);
        entity.HasMany(e => e.Posts)
              .WithOne(e => e.Blog)
              .HasForeignKey(e => e.BlogId);
    });
}
```

## Advanced Configuration

### 1. Table Mapping

```csharp
modelBuilder.Entity<Blog>()
    .ToTable("Blogs", schema: "blogging")
    .HasKey(b => b.Id);
```

### 2. Property Configuration

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.CreatedDate)
    .HasDefaultValueSql("GETUTCDATE()");

modelBuilder.Entity<Blog>()
    .Property(b => b.Rating)
    .HasColumnType("decimal(5,2)");
```

### 3. Indexes

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.Url)
    .IsUnique();

modelBuilder.Entity<Post>()
    .HasIndex(p => new { p.Title, p.BlogId });
```

## Common Patterns

### 1. Base DbContext

```csharp
public abstract class ApplicationDbContext : DbContext
{
    protected ApplicationDbContext(DbContextOptions options)
        : base(options)
    {
    }

    public override int SaveChanges()
    {
        AddTimestamps();
        return base.SaveChanges();
    }

    private void AddTimestamps()
    {
        var entities = ChangeTracker.Entries()
            .Where(x => x.Entity is BaseEntity &&
                (x.State == EntityState.Added || x.State == EntityState.Modified));

        foreach (var entity in entities)
        {
            if (entity.State == EntityState.Added)
            {
                ((BaseEntity)entity.Entity).CreatedDate = DateTime.UtcNow;
            }

            ((BaseEntity)entity.Entity).ModifiedDate = DateTime.UtcNow;
        }
    }
}
```

### 2. Entity Base Class

```csharp
public abstract class BaseEntity
{
    public int Id { get; set; }
    public DateTime CreatedDate { get; set; }
    public DateTime ModifiedDate { get; set; }
    public string CreatedBy { get; set; }
    public string ModifiedBy { get; set; }
}

public class Blog : BaseEntity
{
    public string Url { get; set; }
    public List<Post> Posts { get; set; }
}
```

## Best Practices

1. Context Management

```csharp
// DO: Use using statement or dependency injection
using (var context = new BloggingContext())
{
    // Work with context
}

// DON'T: Keep context alive for long periods
public class BadService
{
    private readonly BloggingContext _context; // Long-lived context is bad
}
```

2. Entity Design

```csharp
// DO: Use meaningful property names and types
public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; }
    public BlogStatus Status { get; set; }
}

// DON'T: Use ambiguous names or types
public class Blog
{
    public int X { get; set; } // Unclear name
    public string Y { get; set; } // Unclear name
    public int Status { get; set; } // Should be enum
}
```

## Performance Considerations

- Context lifetime management
- Change tracking overhead
- Lazy loading implications
- Query execution strategies
- Connection management
- Model building performance
- Memory usage patterns

## Important Notes

- Dispose contexts properly
- Configure appropriate timeouts
- Handle connection resilience
- Consider multi-tenancy
- Plan for scalability
- Document configurations
- Monitor performance
- Use appropriate pooling
