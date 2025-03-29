# Indexes and Constraints in EF Core

## Overview

Indexes and constraints are database features that help maintain data integrity and improve query performance. EF Core provides various ways to configure these through both Data Annotations and Fluent API.

## Indexes

### 1. Single Column Index

```csharp
// Using Data Annotations
public class Blog
{
    public int Id { get; set; }

    [Index]
    public string Url { get; set; }
}

// Using Fluent API
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.Url);
```

### 2. Composite Index

```csharp
// Using Fluent API
modelBuilder.Entity<Blog>()
    .HasIndex(b => new { b.Url, b.Rating });
```

### 3. Unique Index

```csharp
// Using Data Annotations
public class Blog
{
    public int Id { get; set; }

    [Index(IsUnique = true)]
    public string Url { get; set; }
}

// Using Fluent API
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.Url)
    .IsUnique();
```

### 4. Named Index

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.Url)
    .HasDatabaseName("IX_Blogs_Url");
```

### 5. Filtered Index

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.Url)
    .HasFilter("[IsDeleted] = 0");
```

## Constraints

### 1. Primary Key

```csharp
// Using Data Annotations
public class Blog
{
    [Key]
    public int Id { get; set; }
}

// Using Fluent API
modelBuilder.Entity<Blog>()
    .HasKey(b => b.Id);

// Composite Primary Key
modelBuilder.Entity<BlogPost>()
    .HasKey(bp => new { bp.BlogId, bp.PostId });
```

### 2. Foreign Key

```csharp
// Using Data Annotations
public class Post
{
    public int Id { get; set; }

    [ForeignKey("Blog")]
    public int BlogId { get; set; }
    public Blog Blog { get; set; }
}

// Using Fluent API
modelBuilder.Entity<Post>()
    .HasOne(p => p.Blog)
    .WithMany(b => b.Posts)
    .HasForeignKey(p => p.BlogId);
```

### 3. Unique Constraint

```csharp
// Using Data Annotations
public class Blog
{
    public int Id { get; set; }

    [Index(IsUnique = true)]
    public string Url { get; set; }
}

// Using Fluent API
modelBuilder.Entity<Blog>()
    .HasAlternateKey(b => b.Url);
```

### 4. Required Property

```csharp
// Using Data Annotations
public class Blog
{
    public int Id { get; set; }

    [Required]
    public string Url { get; set; }
}

// Using Fluent API
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .IsRequired();
```

### 5. Maximum Length

```csharp
// Using Data Annotations
public class Blog
{
    [MaxLength(500)]
    public string Url { get; set; }
}

// Using Fluent API
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .HasMaxLength(500);
```

## Advanced Configurations

### 1. Include Properties in Index

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.Url)
    .IncludeProperties(b => new { b.Rating, b.Title });
```

### 2. Custom Collation

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .UseCollation("SQL_Latin1_General_CP1_CI_AS");
```

### 3. Check Constraints

```csharp
modelBuilder.Entity<Blog>()
    .ToTable(b => b.HasCheckConstraint("CK_Rating", "Rating >= 0 AND Rating <= 5"));
```

## Common Scenarios

### 1. Composite Keys with Auto-Generated Values

```csharp
modelBuilder.Entity<BlogPost>()
    .HasKey(bp => new { bp.BlogId, bp.PostId });

modelBuilder.Entity<BlogPost>()
    .Property(bp => bp.PostId)
    .ValueGeneratedOnAdd();
```

### 2. Cascading Referential Integrity

```csharp
modelBuilder.Entity<Post>()
    .HasOne(p => p.Blog)
    .WithMany(b => b.Posts)
    .HasForeignKey(p => p.BlogId)
    .OnDelete(DeleteBehavior.Cascade);
```

### 3. Soft Delete Index

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => new { b.IsDeleted, b.LastModified })
    .HasFilter("[IsDeleted] = 0");
```

## Best Practices

1. Use meaningful index names
2. Index foreign key columns
3. Consider composite indexes for frequent queries
4. Use filtered indexes for subset of data
5. Balance between indexes and write performance
6. Use appropriate constraint types
7. Consider database-specific features

## Performance Considerations

- Too many indexes affect write performance
- Indexes increase database size
- Consider maintenance overhead
- Monitor index usage
- Use covering indexes when appropriate
- Consider fill factor settings

## Important Notes

- Indexes impact write operations
- Consider database size implications
- Test with representative data volume
- Monitor index fragmentation
- Consider rebuild/reorganize strategy
- Balance constraints and performance
- Use appropriate clustering strategy
- Consider unique vs non-unique trade-offs
