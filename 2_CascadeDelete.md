# Cascade Delete in EF Core

## Overview

Cascade delete is a feature that automatically deletes dependent entities when their principal/parent entity is deleted. EF Core supports various delete behaviors that can be configured for relationships between entities.

## Delete Behaviors

### 1. Cascade

- Dependent entities are automatically deleted
- Most common for required relationships

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .OnDelete(DeleteBehavior.Cascade);
```

### 2. Restrict

- Prevents deletion of the principal if there are related dependents
- Throws exception if deletion is attempted

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .OnDelete(DeleteBehavior.Restrict);
```

### 3. SetNull

- Sets foreign key properties to null
- Only valid for optional relationships

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .OnDelete(DeleteBehavior.SetNull);
```

### 4. NoAction

- No action is taken in the database
- Similar to Restrict but without explicit constraint

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .OnDelete(DeleteBehavior.NoAction);
```

## Configuration Methods

### 1. Using Fluent API

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasMany(b => b.Posts)
        .WithOne(p => p.Blog)
        .OnDelete(DeleteBehavior.Cascade);
}
```

### 2. Using Data Annotations

```csharp
public class Post
{
    public int Id { get; set; }

    [Required]
    [ForeignKey("BlogId")]
    public Blog Blog { get; set; }
}
```

## Best Practices

1. Consider data integrity requirements
2. Use Cascade for required relationships
3. Use Restrict when deletion should be prevented
4. Use SetNull for optional relationships
5. Test deletion behavior with sample data

## Common Scenarios

### Parent-Child Relationship

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<Post> Posts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int BlogId { get; set; }
    public Blog Blog { get; set; }
}

// Configuration
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .OnDelete(DeleteBehavior.Cascade);
```

## Important Notes

- Cascade delete can have performance implications
- Consider using batch delete for large datasets
- Be careful with circular cascade paths
- Different database providers might handle cascade delete differently
- Always test deletion behavior in a safe environment first
