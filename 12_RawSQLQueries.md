# Raw SQL Queries in EF Core

## Overview

EF Core allows executing raw SQL commands when LINQ queries aren't sufficient or when you need to leverage database-specific features. This can be done through various methods while still maintaining the benefits of the EF Core context.

## Key Features

- Execute raw SQL queries
- Map results to entities
- Execute non-query commands
- Support for stored procedures
- Parameter handling
- Multiple result sets

## Basic Queries

### 1. FromSqlRaw

```csharp
// Basic query
var blogs = context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs")
    .ToList();

// With parameters
var rating = 4;
var blogs = context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", rating)
    .ToList();
```

### 2. FromSqlInterpolated

```csharp
var rating = 4;
var blogs = context.Blogs
    .FromSqlInterpolated($"SELECT * FROM Blogs WHERE Rating > {rating}")
    .ToList();
```

## Advanced Queries

### 1. Combining with LINQ

```csharp
var blogs = context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs")
    .Where(b => b.Rating > 4)
    .OrderBy(b => b.Name)
    .ToList();
```

### 2. Multiple Result Sets

```csharp
using (var command = context.Database.GetDbConnection().CreateCommand())
{
    command.CommandText = @"
        SELECT * FROM Blogs;
        SELECT * FROM Posts;";

    context.Database.OpenConnection();

    using (var reader = command.ExecuteReader())
    {
        var blogs = context.Blogs.FromSqlRaw("SELECT * FROM Blogs").ToList();
        reader.NextResult();
        var posts = context.Posts.FromSqlRaw("SELECT * FROM Posts").ToList();
    }
}
```

## Stored Procedures

### 1. Calling Stored Procedures

```csharp
// Using FromSqlRaw
var blogs = context.Blogs
    .FromSqlRaw("EXEC GetBlogsByRating @p0", rating)
    .ToList();

// Using SqlParameter
var param = new SqlParameter("@rating", SqlDbType.Int) { Value = rating };
var blogs = context.Blogs
    .FromSqlRaw("EXEC GetBlogsByRating @rating", param)
    .ToList();
```

### 2. Output Parameters

```csharp
var paramRating = new SqlParameter
{
    ParameterName = "@rating",
    SqlDbType = SqlDbType.Int,
    Direction = ParameterDirection.Output
};

context.Database.ExecuteSqlRaw(
    "EXEC GetAverageRating @rating OUTPUT",
    paramRating);

var averageRating = (int)paramRating.Value;
```

## Non-Query Commands

### 1. ExecuteSqlRaw

```csharp
// Basic execution
var affected = context.Database.ExecuteSqlRaw(
    "UPDATE Blogs SET Rating = 5 WHERE Id = 1");

// With parameters
context.Database.ExecuteSqlRaw(
    "UPDATE Blogs SET Rating = {0} WHERE Id = {1}",
    newRating, blogId);
```

### 2. ExecuteSqlInterpolated

```csharp
var newRating = 5;
var blogId = 1;
context.Database.ExecuteSqlInterpolated(
    $"UPDATE Blogs SET Rating = {newRating} WHERE Id = {blogId}");
```

## Best Practices

### 1. Parameter Usage

```csharp
// DO: Use parameters
var blogs = context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > {0}", rating)
    .ToList();

// DON'T: Concatenate strings (SQL injection risk)
var blogs = context.Blogs
    .FromSqlRaw($"SELECT * FROM Blogs WHERE Rating > {rating}")  // Wrong!
    .ToList();
```

### 2. Column Mapping

```csharp
// Ensure all entity properties are returned
var blogs = context.Blogs
    .FromSqlRaw("SELECT Id, Url, Rating FROM Blogs")  // Must include all mapped properties
    .ToList();
```

## Common Scenarios

### 1. Complex Queries

```csharp
var result = context.Blogs
    .FromSqlRaw(@"
        WITH RankedBlogs AS (
            SELECT *, ROW_NUMBER() OVER (ORDER BY Rating DESC) AS Rank
            FROM Blogs
        )
        SELECT * FROM RankedBlogs WHERE Rank <= 10")
    .ToList();
```

### 2. Table-Valued Functions

```csharp
var blogs = context.Blogs
    .FromSqlRaw("SELECT * FROM dbo.GetTopBlogs(@p0)", count)
    .ToList();
```

### 3. Dynamic SQL

```csharp
public IList<Blog> GetBlogsByFilter(string column, string value)
{
    var sql = "SELECT * FROM Blogs WHERE @column = @value";
    var columnParam = new SqlParameter("@column", column);
    var valueParam = new SqlParameter("@value", value);

    return context.Blogs
        .FromSqlRaw(sql, columnParam, valueParam)
        .ToList();
}
```

## Performance Considerations

- Use parameters for better query plan caching
- Consider using stored procedures for complex queries
- Monitor query execution plans
- Use appropriate transaction isolation levels
- Be mindful of connection pooling

## Security Considerations

- Always use parameterized queries
- Avoid string concatenation
- Validate user input
- Use appropriate permissions
- Consider using stored procedures
- Implement proper error handling

## Important Notes

- Raw SQL must return all properties
- Cannot contain related data
- Must return entities of the same type
- Consider maintainability implications
- Test thoroughly with real data
- Monitor for SQL injection vulnerabilities
- Keep track of database compatibility
