# Testing in EF Core

## Overview

Testing applications that use EF Core requires special consideration for database interactions. This guide covers various testing approaches, from in-memory testing to integration testing with real databases.

## Key Features

- In-memory database provider
- SQLite testing provider
- Mocking DbContext and DbSet
- Test fixtures
- Database seeding
- Transaction management
- Assertion helpers

## Testing Approaches

### 1. In-Memory Database Testing

```csharp
// Test Context
public class TestBlogContext : BlogContext
{
    public TestBlogContext(DbContextOptions<BlogContext> options)
        : base(options)
    {
    }
}

// Test Setup
public class BlogServiceTests
{
    private DbContextOptions<BlogContext> CreateNewContextOptions()
    {
        return new DbContextOptionsBuilder<BlogContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
    }

    [Fact]
    public async Task CreateBlog_ShouldAddNewBlog()
    {
        // Arrange
        var options = CreateNewContextOptions();
        using var context = new TestBlogContext(options);
        var service = new BlogService(context);

        // Act
        var blog = await service.CreateBlog("Test Blog");

        // Assert
        Assert.NotNull(blog);
        Assert.Equal("Test Blog", blog.Title);
        Assert.Single(await context.Blogs.ToListAsync());
    }
}
```

### 2. SQLite Testing

```csharp
public class SqliteBlogTests
{
    private DbContextOptions<BlogContext> CreateSqliteOptions()
    {
        var connection = new SqliteConnection("DataSource=:memory:");
        connection.Open();

        return new DbContextOptionsBuilder<BlogContext>()
            .UseSqlite(connection)
            .Options;
    }

    [Fact]
    public async Task TestBlogCreation()
    {
        // Arrange
        var options = CreateSqliteOptions();
        using var context = new BlogContext(options);
        await context.Database.EnsureCreatedAsync();

        // Act
        context.Blogs.Add(new Blog { Title = "Test Blog" });
        await context.SaveChangesAsync();

        // Assert
        var blog = await context.Blogs.FirstAsync();
        Assert.Equal("Test Blog", blog.Title);
    }
}
```

## Test Data Setup

### 1. Database Seeding

```csharp
public static class TestDataSeeder
{
    public static async Task SeedTestData(BlogContext context)
    {
        if (!await context.Blogs.AnyAsync())
        {
            await context.Blogs.AddRangeAsync(
                new Blog { Title = "Blog 1", Url = "http://blog1.com" },
                new Blog { Title = "Blog 2", Url = "http://blog2.com" }
            );
            await context.SaveChangesAsync();
        }
    }
}

[Fact]
public async Task TestWithSeededData()
{
    // Arrange
    using var context = new BlogContext(options);
    await TestDataSeeder.SeedTestData(context);

    // Act & Assert
    Assert.Equal(2, await context.Blogs.CountAsync());
}
```

### 2. Test Fixtures

```csharp
public class BlogTestFixture : IDisposable
{
    public BlogContext Context { get; }

    public BlogTestFixture()
    {
        var options = new DbContextOptionsBuilder<BlogContext>()
            .UseInMemoryDatabase("TestDb")
            .Options;

        Context = new BlogContext(options);
        SeedData().Wait();
    }

    private async Task SeedData()
    {
        await TestDataSeeder.SeedTestData(Context);
    }

    public void Dispose()
    {
        Context.Dispose();
    }
}

public class BlogTests : IClassFixture<BlogTestFixture>
{
    private readonly BlogTestFixture _fixture;

    public BlogTests(BlogTestFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task TestBlogRetrieval()
    {
        // Arrange
        var service = new BlogService(_fixture.Context);

        // Act
        var blogs = await service.GetAllBlogs();

        // Assert
        Assert.NotEmpty(blogs);
    }
}
```

## Integration Testing

### 1. Real Database Testing

```csharp
public class IntegrationTests : IDisposable
{
    private readonly BlogContext _context;
    private readonly IDbContextTransaction _transaction;

    public IntegrationTests()
    {
        var options = new DbContextOptionsBuilder<BlogContext>()
            .UseSqlServer("your_test_connection_string")
            .Options;

        _context = new BlogContext(options);
        _transaction = _context.Database.BeginTransaction();
    }

    [Fact]
    public async Task TestDatabaseOperation()
    {
        // Arrange
        var blog = new Blog { Title = "Integration Test Blog" };

        // Act
        _context.Blogs.Add(blog);
        await _context.SaveChangesAsync();

        // Assert
        var savedBlog = await _context.Blogs
            .FirstOrDefaultAsync(b => b.Title == blog.Title);
        Assert.NotNull(savedBlog);
    }

    public void Dispose()
    {
        _transaction.Rollback();
        _transaction.Dispose();
        _context.Dispose();
    }
}
```

### 2. Custom Test Database Factory

```csharp
public class TestDatabaseFactory
{
    private static readonly object _lock = new object();
    private static bool _databaseInitialized;

    public static BlogContext CreateContext()
    {
        var options = new DbContextOptionsBuilder<BlogContext>()
            .UseSqlServer("your_test_connection_string")
            .Options;

        var context = new BlogContext(options);

        lock (_lock)
        {
            if (!_databaseInitialized)
            {
                context.Database.EnsureDeleted();
                context.Database.EnsureCreated();
                TestDataSeeder.SeedTestData(context).Wait();
                _databaseInitialized = true;
            }
        }

        return context;
    }
}
```

## Mocking

### 1. Mocking DbContext

```csharp
public class BlogServiceTests
{
    [Fact]
    public async Task TestWithMockedContext()
    {
        // Arrange
        var mockSet = new Mock<DbSet<Blog>>();
        var mockContext = new Mock<BlogContext>();

        var blogs = new List<Blog>
        {
            new Blog { Id = 1, Title = "Test Blog" }
        }.AsQueryable();

        mockSet.As<IQueryable<Blog>>()
            .Setup(m => m.Provider)
            .Returns(blogs.Provider);
        mockSet.As<IQueryable<Blog>>()
            .Setup(m => m.Expression)
            .Returns(blogs.Expression);
        mockSet.As<IQueryable<Blog>>()
            .Setup(m => m.ElementType)
            .Returns(blogs.ElementType);
        mockSet.As<IQueryable<Blog>>()
            .Setup(m => m.GetEnumerator())
            .Returns(blogs.GetEnumerator());

        mockContext.Setup(c => c.Blogs)
            .Returns(mockSet.Object);

        var service = new BlogService(mockContext.Object);

        // Act
        var result = await service.GetBlogById(1);

        // Assert
        Assert.NotNull(result);
        Assert.Equal("Test Blog", result.Title);
    }
}
```

### 2. Mocking DbSet

```csharp
public static class MockDbSetExtensions
{
    public static Mock<DbSet<T>> CreateMockDbSet<T>(IEnumerable<T> data)
        where T : class
    {
        var queryable = data.AsQueryable();
        var mockSet = new Mock<DbSet<T>>();

        mockSet.As<IQueryable<T>>()
            .Setup(m => m.Provider)
            .Returns(queryable.Provider);
        mockSet.As<IQueryable<T>>()
            .Setup(m => m.Expression)
            .Returns(queryable.Expression);
        mockSet.As<IQueryable<T>>()
            .Setup(m => m.ElementType)
            .Returns(queryable.ElementType);
        mockSet.As<IQueryable<T>>()
            .Setup(m => m.GetEnumerator())
            .Returns(() => queryable.GetEnumerator());

        return mockSet;
    }
}
```

## Best Practices

1. Test Organization

```csharp
// DO: Organize tests by feature
[Collection("Blog Tests")]
public class BlogCreationTests
{
    [Fact]
    public async Task CreateBlog_ValidData_ShouldSucceed() { }

    [Fact]
    public async Task CreateBlog_InvalidData_ShouldThrowException() { }
}

// DO: Use meaningful test names
[Fact]
public async Task GetBlogById_WhenBlogExists_ReturnsCorrectBlog() { }
```

2. Test Data Management

```csharp
// DO: Use test categories
[Fact]
[Trait("Category", "Integration")]
public async Task TestDatabaseOperation() { }

// DO: Clean up test data
public class TestBase : IDisposable
{
    protected readonly BlogContext Context;
    protected readonly IDbContextTransaction Transaction;

    public TestBase()
    {
        Context = new BlogContext(options);
        Transaction = Context.Database.BeginTransaction();
    }

    public void Dispose()
    {
        Transaction.Rollback();
        Transaction.Dispose();
        Context.Dispose();
    }
}
```

## Performance Considerations

- Test isolation
- Database cleanup
- Transaction management
- Parallel test execution
- Test data volume
- Mock vs real database
- Connection pooling

## Important Notes

- Test database security
- Environment-specific configuration
- Data cleanup strategies
- Transaction handling
- Async/await usage
- Exception handling
- Test reliability
- CI/CD considerations
