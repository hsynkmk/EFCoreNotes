# Transactions in EF Core

## Overview

Transactions in EF Core ensure data consistency by grouping multiple database operations into a single unit of work. If any operation fails, all changes within the transaction are rolled back.

## Key Features

- Automatic transaction management
- Explicit transaction control
- Distributed transaction support
- Savepoint management
- Cross-context transactions
- Multiple database support

## Basic Usage

### 1. Implicit Transactions

```csharp
using (var context = new BloggingContext())
{
    var blog = new Blog { Url = "http://example.com" };
    context.Blogs.Add(blog);

    var post = new Post { BlogId = blog.Id, Title = "Hello World" };
    context.Posts.Add(post);

    // Automatically wrapped in a transaction
    context.SaveChanges();
}
```

### 2. Explicit Transactions

```csharp
using (var context = new BloggingContext())
{
    using (var transaction = context.Database.BeginTransaction())
    {
        try
        {
            context.Blogs.Add(new Blog { Url = "http://example.com" });
            context.SaveChanges();

            context.Posts.Add(new Post { Title = "Post" });
            context.SaveChanges();

            transaction.Commit();
        }
        catch (Exception)
        {
            transaction.Rollback();
            throw;
        }
    }
}
```

## Advanced Scenarios

### 1. Cross-Context Transactions

```csharp
using (var connection = new SqlConnection(connectionString))
{
    connection.Open();

    using (var transaction = connection.BeginTransaction())
    {
        try
        {
            using (var context1 = new BloggingContext(connection))
            {
                context1.Database.UseTransaction(transaction);
                // Perform operations with context1
                context1.SaveChanges();
            }

            using (var context2 = new OrderContext(connection))
            {
                context2.Database.UseTransaction(transaction);
                // Perform operations with context2
                context2.SaveChanges();
            }

            transaction.Commit();
        }
        catch (Exception)
        {
            transaction.Rollback();
            throw;
        }
    }
}
```

### 2. Savepoints

```csharp
using (var transaction = context.Database.BeginTransaction())
{
    try
    {
        context.Blogs.Add(new Blog { Url = "http://example1.com" });
        context.SaveChanges();

        // Create savepoint
        transaction.CreateSavepoint("SavepointA");

        try
        {
            context.Blogs.Add(new Blog { Url = "http://example2.com" });
            context.SaveChanges();
        }
        catch
        {
            // Rollback to savepoint
            transaction.RollbackToSavepoint("SavepointA");
        }

        transaction.Commit();
    }
    catch (Exception)
    {
        transaction.Rollback();
        throw;
    }
}
```

### 3. Distributed Transactions

```csharp
using (var scope = new TransactionScope())
{
    using (var context1 = new BloggingContext())
    {
        context1.Blogs.Add(new Blog { Url = "http://example.com" });
        context1.SaveChanges();
    }

    using (var context2 = new OrderContext())
    {
        context2.Orders.Add(new Order { Amount = 100 });
        context2.SaveChanges();
    }

    scope.Complete();
}
```

## Common Patterns

### 1. Repository Pattern with Transactions

```csharp
public class BlogRepository
{
    private readonly BloggingContext _context;

    public async Task CreateBlogWithPosts(Blog blog, List<Post> posts)
    {
        using (var transaction = await _context.Database.BeginTransactionAsync())
        {
            try
            {
                _context.Blogs.Add(blog);
                await _context.SaveChangesAsync();

                foreach (var post in posts)
                {
                    post.BlogId = blog.Id;
                }
                _context.Posts.AddRange(posts);
                await _context.SaveChangesAsync();

                await transaction.CommitAsync();
            }
            catch
            {
                await transaction.RollbackAsync();
                throw;
            }
        }
    }
}
```

### 2. Unit of Work Pattern

```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly BloggingContext _context;
    private IDbContextTransaction _transaction;

    public async Task BeginTransactionAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task CommitAsync()
    {
        try
        {
            await _context.SaveChangesAsync();
            await _transaction.CommitAsync();
        }
        catch
        {
            await RollbackAsync();
            throw;
        }
    }

    public async Task RollbackAsync()
    {
        await _transaction?.RollbackAsync();
    }
}
```

## Best Practices

1. Transaction Scope

```csharp
// Keep transactions as short as possible
using (var transaction = context.Database.BeginTransaction())
{
    try
    {
        // Perform minimal required operations
        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}
```

2. Error Handling

```csharp
public async Task<bool> TryExecuteTransaction(Func<Task> operation)
{
    using var transaction = await _context.Database.BeginTransactionAsync();
    try
    {
        await operation();
        await transaction.CommitAsync();
        return true;
    }
    catch (Exception ex)
    {
        await transaction.RollbackAsync();
        _logger.LogError(ex, "Transaction failed");
        return false;
    }
}
```

## Performance Considerations

- Keep transactions short
- Avoid nested transactions
- Use appropriate isolation levels
- Consider connection pooling
- Monitor transaction timeouts
- Batch operations when possible

## Important Notes

- Always use try-catch blocks
- Properly dispose transactions
- Consider deadlock scenarios
- Monitor transaction logs
- Use async operations when possible
- Handle timeout scenarios
- Consider distributed transaction costs
- Test rollback scenarios
- Document transaction boundaries
