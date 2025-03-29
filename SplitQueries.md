# Split Queries in EF Core

## Overview

Split queries allow you to split a single LINQ query into multiple SQL queries. This is particularly useful when dealing with related collections that might cause performance issues due to cartesian explosion.

## When to Use

- When loading related collections that could result in large cartesian products
- When experiencing performance issues with Include operations on large datasets
- When you want more control over data loading patterns

## Basic Usage

```csharp
// Configure split queries at the context level (globally)
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(connectionString)
        .UseQuerySplitting(QuerySplittingBehavior.SplitQuery);
}

// OR use split queries for specific queries
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .AsSplitQuery()
    .ToListAsync();
```

## Advantages

1. Prevents cartesian explosion
2. Can improve performance for large datasets
3. Reduces memory usage
4. More efficient network bandwidth usage

## Disadvantages

1. Multiple round trips to the database
2. No guaranteed data consistency between splits
3. May not be suitable for all scenarios

## Best Practices

1. Use split queries when loading large related collections
2. Monitor performance impact
3. Consider data consistency requirements
4. Test with representative data volumes

## Example Scenarios

### Scenario 1: Blog with Many Posts

```csharp
// Without split query - might cause performance issues
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Comments)
    .ToListAsync();

// With split query - better performance
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Comments)
    .AsSplitQuery()
    .ToListAsync();
```

## Important Notes

- Split queries generate multiple SQL SELECT statements
- Each split operates independently
- Consider using split queries when experiencing memory pressure or timeout issues
- Not recommended for scenarios requiring strict data consistency
