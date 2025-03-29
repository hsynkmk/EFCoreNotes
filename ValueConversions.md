# Value Conversions in EF Core

## Overview

Value conversions allow you to transform property values when they are read from or written to the database. This feature enables storing data in a different format than how it's represented in your entity classes.

## Types of Conversions

### 1. Built-in Conversions

EF Core automatically handles many common conversions:

- Enum to string/number
- Boolean to number/string
- Guid to string
- Numbers to strings
- TimeSpan to ticks

### 2. Custom Value Converters

```csharp
// Example: Converting JSON to string
public class JsonConverter<T> : ValueConverter<T, string>
{
    public JsonConverter() : base(
        v => JsonSerializer.Serialize(v),
        v => JsonSerializer.Deserialize<T>(v))
    {}
}

// Usage in model configuration
modelBuilder.Entity<Blog>()
    .Property(b => b.Metadata)
    .HasConversion(new JsonConverter<Dictionary<string, string>>());
```

## Configuration Methods

### 1. Fluent API

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Using built-in conversion
    modelBuilder.Entity<Blog>()
        .Property(b => b.Status)
        .HasConversion<string>();

    // Using custom conversion
    modelBuilder.Entity<Blog>()
        .Property(b => b.Tags)
        .HasConversion(
            v => string.Join(',', v),
            v => v.Split(',', StringSplitOptions.None));
}
```

### 2. Value Converter Class

```csharp
public class ListToStringConverter : ValueConverter<List<string>, string>
{
    public ListToStringConverter() : base(
        v => string.Join(',', v),
        v => v.Split(',').ToList())
    {}
}

// Usage
modelBuilder.Entity<Blog>()
    .Property(b => b.Tags)
    .HasConversion(new ListToStringConverter());
```

## Common Scenarios

### 1. Enum Conversions

```csharp
public enum BlogStatus { Draft, Published, Archived }

public class Blog
{
    public int Id { get; set; }
    public BlogStatus Status { get; set; }
}

// Store as string
modelBuilder.Entity<Blog>()
    .Property(b => b.Status)
    .HasConversion<string>();
```

### 2. JSON Serialization

```csharp
public class Blog
{
    public int Id { get; set; }
    public Dictionary<string, string> Metadata { get; set; }
}

modelBuilder.Entity<Blog>()
    .Property(b => b.Metadata)
    .HasConversion(
        v => JsonSerializer.Serialize(v),
        v => JsonSerializer.Deserialize<Dictionary<string, string>>(v));
```

### 3. Encrypted Values

```csharp
public class EncryptedConverter : ValueConverter<string, string>
{
    public EncryptedConverter() : base(
        v => Encrypt(v),
        v => Decrypt(v))
    {}

    private static string Encrypt(string value) => // Encryption logic
    private static string Decrypt(string value) => // Decryption logic
}
```

## Best Practices

1. Keep conversions simple and efficient
2. Consider impact on querying
3. Handle null values appropriately
4. Test edge cases thoroughly
5. Document conversion behavior

## Performance Considerations

- Conversions run on every read/write operation
- Complex conversions may impact performance
- Consider caching converted values
- Be mindful of memory usage with large collections

## Important Notes

- Conversions are applied at the property level
- Cannot be changed after model is built
- May affect query performance
- Some conversions may limit LINQ capabilities
- Consider database collation for string conversions
