# Owned Entities in EF Core

## Overview

Owned Entities allow you to model entity types that can only exist as properties of other entity types. They are particularly useful for value objects in Domain-Driven Design (DDD) and for breaking down complex entities into smaller, more manageable parts.

## Key Features

- Can only exist as properties of owning entities
- Can be shared by multiple entity types
- Support value object patterns
- Can have their own properties and navigation properties
- Can be mapped to separate tables

## Configuration Methods

### 1. Using Fluent API

```csharp
public class Order
{
    public int Id { get; set; }
    public StreetAddress ShippingAddress { get; set; }
    public StreetAddress BillingAddress { get; set; }
}

public class StreetAddress
{
    public string Street { get; set; }
    public string City { get; set; }
    public string ZipCode { get; set; }
}

// Configuration
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>()
        .OwnsOne(o => o.ShippingAddress);

    modelBuilder.Entity<Order>()
        .OwnsOne(o => o.BillingAddress);
}
```

### 2. Using Attributes

```csharp
public class Order
{
    public int Id { get; set; }

    [Owned]
    public StreetAddress ShippingAddress { get; set; }

    [Owned]
    public StreetAddress BillingAddress { get; set; }
}
```

## Advanced Configurations

### 1. Table Splitting

```csharp
modelBuilder.Entity<Order>()
    .OwnsOne(o => o.ShippingAddress, sa =>
    {
        sa.Property(p => p.Street).HasColumnName("ShippingStreet");
        sa.Property(p => p.City).HasColumnName("ShippingCity");
        sa.Property(p => p.ZipCode).HasColumnName("ShippingZipCode");
    });
```

### 2. Separate Tables

```csharp
modelBuilder.Entity<Order>()
    .OwnsOne(o => o.ShippingAddress, sa =>
    {
        sa.ToTable("OrderShippingAddresses");
    });
```

### 3. Collection of Owned Types

```csharp
public class Customer
{
    public int Id { get; set; }
    public List<Address> Addresses { get; set; }
}

// Configuration
modelBuilder.Entity<Customer>()
    .OwnsMany(c => c.Addresses, a =>
    {
        a.WithOwner().HasForeignKey("CustomerId");
        a.Property<int>("Id");
        a.HasKey("Id");
    });
```

## Common Scenarios

### 1. Value Objects

```csharp
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public Money Price { get; set; }
}

// Configuration
modelBuilder.Entity<Product>()
    .OwnsOne(p => p.Price);
```

### 2. Complex Types

```csharp
public class Person
{
    public int Id { get; set; }
    public Name Name { get; set; }
    public Contact Contact { get; set; }
}

public class Name
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

public class Contact
{
    public string Email { get; set; }
    public string Phone { get; set; }
}

// Configuration
modelBuilder.Entity<Person>(b =>
{
    b.OwnsOne(p => p.Name);
    b.OwnsOne(p => p.Contact);
});
```

## Best Practices

1. Use owned entities for value objects
2. Consider table splitting for performance
3. Use meaningful prefix/suffix for column names
4. Handle null values appropriately
5. Consider using separate tables for large owned types

## Performance Considerations

- Table splitting can improve query performance
- Separate tables might be better for large objects
- Consider indexing strategy for frequently queried properties
- Be mindful of eager loading impact

## Important Notes

- Owned entities cannot have their own key values
- They follow the lifetime of their owner
- Cannot be referenced by other entities
- Support inheritance
- Can have navigation properties to regular entities
- Changes are tracked automatically with owner
