# Change Tracking in EF Core

## Overview

Change tracking is a feature that monitors changes to entities loaded into the DbContext. It helps EF Core determine which changes need to be persisted to the database when SaveChanges is called.

## Key Features

- Automatic tracking of entity states
- Support for disconnected scenarios
- Multiple tracking strategies
- Performance optimization options
- Relationship fixup

## Entity States

### 1. Added

```csharp
// Explicitly marking as added
context.Blogs.Add(new Blog { Url = "http://example.com" });

// Or using Entry
context.Entry(blog).State = EntityState.Added;
```

### 2. Modified

```csharp
// Automatic detection
var blog = context.Blogs.First();
blog.Url = "http://newurl.com";

// Explicit marking
context.Entry(blog).State = EntityState.Modified;
```

### 3. Deleted

```csharp
// Using Remove
context.Blogs.Remove(blog);

// Using Entry
context.Entry(blog).State = EntityState.Deleted;
```

### 4. Unchanged

```csharp
// Check if entity is unchanged
bool isUnchanged = context.Entry(blog).State == EntityState.Unchanged;
```

### 5. Detached

```csharp
// Entity not being tracked
var blog = new Blog();
bool isDetached = context.Entry(blog).State == EntityState.Detached;
```

## Tracking Configurations

### 1. No-Tracking Queries

```csharp
// Single query no-tracking
var blogs = context.Blogs
    .AsNoTracking()
    .ToList();

// Context-level no-tracking
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
```

### 2. Identity Resolution

```csharp
// Disable identity resolution
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTrackingWithIdentityResolution;
```

### 3. Explicit Loading

```csharp
var blog = context.Blogs.Find(1);
await context.Entry(blog)
    .Collection(b => b.Posts)
    .LoadAsync();
```

## Change Detection

### 1. Property-Level Tracking

```csharp
var entry = context.Entry(blog);
entry.Property(b => b.Url).IsModified = true;

// Get original values
var originalUrl = entry.Property(b => b.Url).OriginalValue;
```

### 2. Relationship Tracking

```csharp
var navigationEntry = context.Entry(blog)
    .Reference(b => b.Author);

bool isLoaded = navigationEntry.IsLoaded;
```

### 3. Collection Tracking

```csharp
var collectionEntry = context.Entry(blog)
    .Collection(b => b.Posts);

int count = collectionEntry.CurrentValue.Count;
```

## Performance Optimization

### 1. Bulk Operations

```csharp
// Disable auto-detection
context.ChangeTracker.AutoDetectChangesEnabled = false;

// Add multiple entities
context.Blogs.AddRange(blogs);

// Re-enable if needed
context.ChangeTracker.DetectChanges();
```

### 2. Selective Tracking

```csharp
public class BloggingContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder
            .UseSqlServer(connectionString)
            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
    }
}
```

## Common Scenarios

### 1. Disconnected Entities

```csharp
public void UpdateBlog(Blog blog)
{
    using var context = new BloggingContext();
    context.Attach(blog);
    context.Entry(blog).State = EntityState.Modified;
    context.SaveChanges();
}
```

### 2. Selective Property Updates

```csharp
public void UpdateBlogUrl(int blogId, string newUrl)
{
    using var context = new BloggingContext();
    var blog = context.Blogs.Find(blogId);
    context.Entry(blog)
        .Property(b => b.Url).IsModified = true;
    blog.Url = newUrl;
    context.SaveChanges();
}
```

## Best Practices

1. Use no-tracking queries for read-only scenarios
2. Disable change tracking when not needed
3. Be mindful of memory usage with large result sets
4. Use bulk operations for better performance
5. Clear context regularly in long-running processes

## Performance Considerations

- Tracking increases memory usage
- AutoDetectChanges can impact performance
- Large number of tracked entities affects memory
- Consider using no-tracking queries for reads
- Batch operations for better performance

## Important Notes

- Changes are tracked automatically
- SaveChanges processes all tracked changes
- Memory usage increases with tracked entities
- Performance impact with large datasets
- Consider disconnected scenarios
- Relationship fixup affects connected entities
- State changes trigger validation
