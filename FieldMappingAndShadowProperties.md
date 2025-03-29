# Field Mapping and Shadow Properties in EF Core

## Overview

Field mapping allows you to map database columns to private fields in your entities, while shadow properties enable storing data in the change tracker without defining corresponding properties in your entity classes.

## Key Features

- Private field mapping
- Shadow property support
- Backing field configuration
- Custom column mapping
- Property value generation
- Change tracking integration
- Query capabilities

## Field Mapping

### 1. Basic Field Mapping

```csharp
public class Blog
{
    private string _url;
    public string Url
    {
        get => _url;
        set => _url = value;
    }
}

// Configuration
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .HasField("_url");
```

### 2. Private Field-Only

```csharp
public class Blog
{
    private string _url;
    private readonly List<Post> _posts = new();

    public IReadOnlyList<Post> Posts => _posts.AsReadOnly();
}

// Configuration
modelBuilder.Entity<Blog>()
    .Property("_url");

modelBuilder.Entity<Blog>()
    .HasMany("_posts")
    .WithOne();
```

### 3. Backing Fields

```csharp
public class Blog
{
    private string _title;
    public string Title
    {
        get => _title;
        set
        {
            _title = value;
            UpdateSlug();
        }
    }

    private string _slug;
    public string Slug => _slug;

    private void UpdateSlug()
    {
        _slug = _title?.ToLower().Replace(" ", "-");
    }
}
```

## Shadow Properties

### 1. Basic Shadow Properties

```csharp
// Configuration
modelBuilder.Entity<Blog>()
    .Property<DateTime>("LastUpdated");

// Usage
var blogs = context.Blogs
    .OrderBy(b => EF.Property<DateTime>(b, "LastUpdated"))
    .ToList();

// Updating
context.Entry(blog)
    .Property("LastUpdated")
    .CurrentValue = DateTime.UtcNow;
```

### 2. Common Shadow Properties

```csharp
// Configuration
modelBuilder.Entity<Blog>(b =>
{
    b.Property<DateTime>("CreatedAt");
    b.Property<DateTime>("UpdatedAt");
    b.Property<string>("CreatedBy");
    b.Property<string>("UpdatedBy");
});

// Automatic update
context.SaveChanges += (sender, args) =>
{
    var entries = context.ChangeTracker
        .Entries()
        .Where(e => e.State == EntityState.Added ||
                    e.State == EntityState.Modified);

    foreach (var entry in entries)
    {
        if (entry.State == EntityState.Added)
        {
            entry.Property("CreatedAt").CurrentValue = DateTime.UtcNow;
            entry.Property("CreatedBy").CurrentValue = currentUser.Id;
        }

        entry.Property("UpdatedAt").CurrentValue = DateTime.UtcNow;
        entry.Property("UpdatedBy").CurrentValue = currentUser.Id;
    }
};
```

### 3. Foreign Key Shadow Properties

```csharp
modelBuilder.Entity<Post>()
    .Property<int>("BlogId");

modelBuilder.Entity<Post>()
    .HasOne<Blog>()
    .WithMany()
    .HasForeignKey("BlogId");
```

## Advanced Scenarios

### 1. Composite Pattern

```csharp
public class AuditableEntity
{
    private DateTime _createdAt;
    private string _createdBy;
    private DateTime _updatedAt;
    private string _updatedBy;

    public DateTime CreatedAt => _createdAt;
    public string CreatedBy => _createdBy;
    public DateTime UpdatedAt => _updatedAt;
    public string UpdatedBy => _updatedBy;
}

public class Blog : AuditableEntity
{
    public int Id { get; set; }
    private string _url;
    public string Url
    {
        get => _url;
        set => _url = value;
    }
}
```

### 2. Value Objects with Private Fields

```csharp
public class Address
{
    private string _street;
    private string _city;
    private string _country;

    private Address() { }

    public Address(string street, string city, string country)
    {
        _street = street;
        _city = city;
        _country = country;
    }

    public string FullAddress => $"{_street}, {_city}, {_country}";
}

// Configuration
modelBuilder.Entity<Customer>()
    .OwnsOne(c => c.Address, address =>
    {
        address.Property("_street").HasColumnName("Street");
        address.Property("_city").HasColumnName("City");
        address.Property("_country").HasColumnName("Country");
    });
```

### 3. Query Filters with Shadow Properties

```csharp
modelBuilder.Entity<Blog>()
    .Property<bool>("IsDeleted");

modelBuilder.Entity<Blog>()
    .HasQueryFilter(b =>
        !EF.Property<bool>(b, "IsDeleted"));
```

## Common Patterns

### 1. Audit Trail Implementation

```csharp
public class AuditTrail
{
    public static void AddAuditProperties(ModelBuilder modelBuilder)
    {
        var entities = modelBuilder.Model
            .GetEntityTypes()
            .Where(e => typeof(IAuditable).IsAssignableFrom(e.ClrType));

        foreach (var entity in entities)
        {
            modelBuilder.Entity(entity.ClrType)
                .Property<DateTime>("CreatedAt")
                .HasDefaultValueSql("GETUTCDATE()");

            modelBuilder.Entity(entity.ClrType)
                .Property<string>("CreatedBy");
        }
    }
}
```

### 2. Soft Delete Pattern

```csharp
public static class SoftDeleteQueryExtension
{
    public static void AddSoftDeleteQueryFilter(
        this ModelBuilder modelBuilder)
    {
        var entities = modelBuilder.Model
            .GetEntityTypes()
            .Where(e => typeof(ISoftDelete)
                .IsAssignableFrom(e.ClrType));

        foreach (var entity in entities)
        {
            modelBuilder.Entity(entity.ClrType)
                .Property<bool>("IsDeleted");

            modelBuilder.Entity(entity.ClrType)
                .HasQueryFilter(e =>
                    !EF.Property<bool>(e, "IsDeleted"));
        }
    }
}
```

## Best Practices

1. Use private fields for encapsulation
2. Document shadow properties
3. Consider naming conventions
4. Handle null values appropriately
5. Use appropriate access modifiers
6. Consider serialization implications
7. Plan for maintenance

## Performance Considerations

- Impact on change tracking
- Query performance with shadow properties
- Memory usage
- Serialization overhead
- Indexing considerations
- Lazy loading implications
- Cache effectiveness

## Important Notes

- Private fields affect serialization
- Shadow properties aren't in entity classes
- Consider mapping complexity
- Test query performance
- Document shadow properties
- Consider maintenance impact
- Plan for versioning
- Handle migration scenarios
