# Relationships in EF Core

## Overview

EF Core supports various types of relationships between entities: One-to-One, One-to-Many, and Many-to-Many. Each type can be configured using either Data Annotations or Fluent API.

## Types of Relationships

### 1. One-to-One

```csharp
public class Blog
{
    public int Id { get; set; }
    public BlogDetails Details { get; set; }
}

public class BlogDetails
{
    public int Id { get; set; }
    public string Description { get; set; }
    public Blog Blog { get; set; }
    public int BlogId { get; set; }  // Foreign key
}

// Configuration
modelBuilder.Entity<Blog>()
    .HasOne(b => b.Details)
    .WithOne(d => d.Blog)
    .HasForeignKey<BlogDetails>(d => d.BlogId);
```

### 2. One-to-Many

```csharp
public class Blog
{
    public int Id { get; set; }
    public List<Post> Posts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public int BlogId { get; set; }  // Foreign key
    public Blog Blog { get; set; }
}

// Configuration
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .HasForeignKey(p => p.BlogId);
```

### 3. Many-to-Many

```csharp
public class Post
{
    public int Id { get; set; }
    public List<Tag> Tags { get; set; }
}

public class Tag
{
    public int Id { get; set; }
    public List<Post> Posts { get; set; }
}

// Configuration with join entity
public class PostTag
{
    public int PostId { get; set; }
    public Post Post { get; set; }
    public int TagId { get; set; }
    public Tag Tag { get; set; }
}

modelBuilder.Entity<PostTag>()
    .HasKey(pt => new { pt.PostId, pt.TagId });

modelBuilder.Entity<PostTag>()
    .HasOne(pt => pt.Post)
    .WithMany(p => p.PostTags)
    .HasForeignKey(pt => pt.PostId);

modelBuilder.Entity<PostTag>()
    .HasOne(pt => pt.Tag)
    .WithMany(t => t.PostTags)
    .HasForeignKey(pt => pt.TagId);
```

## Configuration Methods

### 1. Using Data Annotations

```csharp
public class Post
{
    public int Id { get; set; }

    [Required]
    [ForeignKey("Blog")]
    public int BlogId { get; set; }

    public Blog Blog { get; set; }
}
```

### 2. Using Fluent API

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Post>()
        .HasOne(p => p.Blog)
        .WithMany(b => b.Posts)
        .HasForeignKey(p => p.BlogId)
        .OnDelete(DeleteBehavior.Cascade);
}
```

## Advanced Configurations

### 1. Required vs Optional Relationships

```csharp
// Required relationship
modelBuilder.Entity<Post>()
    .HasOne(p => p.Blog)
    .WithMany(b => b.Posts)
    .IsRequired();

// Optional relationship
modelBuilder.Entity<Post>()
    .HasOne(p => p.Blog)
    .WithMany(b => b.Posts)
    .IsRequired(false);
```

### 2. Cascade Delete

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .OnDelete(DeleteBehavior.Cascade);
```

### 3. Shadow Properties

```csharp
modelBuilder.Entity<Post>()
    .Property<int>("BlogId");  // Shadow property

modelBuilder.Entity<Post>()
    .HasOne(p => p.Blog)
    .WithMany(b => b.Posts)
    .HasForeignKey("BlogId");
```

## Loading Related Data

### 1. Eager Loading

```csharp
var blogs = context.Blogs
    .Include(b => b.Posts)
    .ThenInclude(p => p.Comments)
    .ToList();
```

### 2. Explicit Loading

```csharp
var blog = context.Blogs.Find(1);
context.Entry(blog)
    .Collection(b => b.Posts)
    .Load();
```

### 3. Lazy Loading

```csharp
public class Blog
{
    public int Id { get; set; }
    public virtual ICollection<Post> Posts { get; set; }
}
```

## Common Scenarios

### 1. Self-Referencing Relationships

```csharp
public class Employee
{
    public int Id { get; set; }
    public int? ManagerId { get; set; }
    public Employee Manager { get; set; }
    public List<Employee> DirectReports { get; set; }
}

modelBuilder.Entity<Employee>()
    .HasOne(e => e.Manager)
    .WithMany(e => e.DirectReports)
    .HasForeignKey(e => e.ManagerId);
```

### 2. Composite Keys

```csharp
public class OrderDetail
{
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public Order Order { get; set; }
    public Product Product { get; set; }
}

modelBuilder.Entity<OrderDetail>()
    .HasKey(od => new { od.OrderId, od.ProductId });
```

## Best Practices

1. Use appropriate relationship types
2. Consider cascade delete implications
3. Choose proper loading strategies
4. Use foreign key properties
5. Consider indexing foreign keys
6. Handle circular references
7. Use appropriate delete behaviors

## Performance Considerations

- Choose appropriate loading strategy
- Index foreign key columns
- Consider impact of cascade operations
- Be mindful of eager loading depth
- Use selective loading when possible

## Important Notes

- Relationships affect change tracking
- Consider serialization implications
- Foreign keys improve performance
- Cascade delete needs careful consideration
- Loading strategy impacts performance
- Many-to-many requires join entity
- Consider database constraints
