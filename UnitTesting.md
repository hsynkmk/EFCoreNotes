# Unit Testing in EF Core

## Overview

Unit testing EF Core applications requires special consideration for database interactions. This guide covers strategies and best practices for effectively testing EF Core code, including mocking, in-memory databases, and test data setup.

## Key Areas

- Test setup
- In-memory database
- Mocking strategies
- Test data seeding
- Assertion patterns
- Test isolation
- Performance testing
- Integration testing

## Test Setup

### 1. Test Context

```csharp
public class TestBlogContext : BlogContext
{
    public TestBlogContext(DbContextOptions<BlogContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Add test-specific configurations
        modelBuilder.Entity<Blog>()
            .HasData(new Blog
            {
                Id = 1,
                Title = "Test Blog",
                Url = "http://test.com"
            });
    }
}

public class BlogTestBase
{
    protected readonly DbContextOptions<BlogContext> ContextOptions;

    protected BlogTestBase()
    {
        ContextOptions = new DbContextOptionsBuilder<BlogContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;

        using var context = new TestBlogContext(ContextOptions);
        context.Database.EnsureCreated();
    }
}
```

### 2. Test Data Factory

```csharp
public class TestDataFactory
{
    public static Blog CreateTestBlog(string title = "Test Blog")
    {
        return new Blog
        {
            Title = title,
            Url = $"http://{title.ToLower().Replace(" ", "-")}.com",
            CreatedAt = DateTime.UtcNow,
            Posts = new List<Post>
            {
                new Post
                {
                    Title = "Test Post 1",
                    Content = "Test Content 1"
                },
                new Post
                {
                    Title = "Test Post 2",
                    Content = "Test Content 2"
                }
            }
        };
    }

    public static async Task SeedTestDataAsync(BlogContext context)
    {
        if (!await context.Blogs.AnyAsync())
        {
            await context.Blogs.AddRangeAsync(
                CreateTestBlog("Blog 1"),
                CreateTestBlog("Blog 2"),
                CreateTestBlog("Blog 3")
            );

            await context.SaveChangesAsync();
        }
    }
}
```

## Unit Testing Patterns

### 1. Service Tests

```csharp
public class BlogServiceTests : BlogTestBase
{
    [Fact]
    public async Task CreateBlog_WithValidData_ShouldSucceed()
    {
        // Arrange
        using var context = new TestBlogContext(ContextOptions);
        var service = new BlogService(context);
        var blog = TestDataFactory.CreateTestBlog();

        // Act
        var result = await service.CreateBlogAsync(blog);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(blog.Title, result.Title);
        Assert.NotEqual(0, result.Id);
    }

    [Fact]
    public async Task GetBlog_WithInvalidId_ShouldReturnNull()
    {
        // Arrange
        using var context = new TestBlogContext(ContextOptions);
        var service = new BlogService(context);

        // Act
        var result = await service.GetBlogAsync(999);

        // Assert
        Assert.Null(result);
    }

    [Fact]
    public async Task UpdateBlog_WithValidData_ShouldUpdateSuccessfully()
    {
        // Arrange
        using var context = new TestBlogContext(ContextOptions);
        var service = new BlogService(context);
        var blog = await context.Blogs.FirstAsync();
        var newTitle = "Updated Title";

        // Act
        blog.Title = newTitle;
        var result = await service.UpdateBlogAsync(blog);

        // Assert
        Assert.Equal(newTitle, result.Title);
        var updatedBlog = await context.Blogs.FindAsync(blog.Id);
        Assert.Equal(newTitle, updatedBlog.Title);
    }
}
```

### 2. Repository Tests

```csharp
public class BlogRepositoryTests : IClassFixture<BlogTestFixture>
{
    private readonly BlogTestFixture _fixture;

    public BlogRepositoryTests(BlogTestFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task GetAllBlogs_ShouldReturnAllBlogs()
    {
        // Arrange
        using var context = new TestBlogContext(_fixture.ContextOptions);
        var repository = new BlogRepository(context);

        // Act
        var blogs = await repository.GetAllAsync();

        // Assert
        Assert.NotEmpty(blogs);
        Assert.Equal(3, blogs.Count());
    }

    [Theory]
    [InlineData("Blog 1")]
    [InlineData("Blog 2")]
    public async Task GetBlogByTitle_ShouldReturnCorrectBlog(string title)
    {
        // Arrange
        using var context = new TestBlogContext(_fixture.ContextOptions);
        var repository = new BlogRepository(context);

        // Act
        var blog = await repository.GetByTitleAsync(title);

        // Assert
        Assert.NotNull(blog);
        Assert.Equal(title, blog.Title);
    }
}
```

## Mocking Strategies

### 1. DbContext Mocking

```csharp
public class BlogServiceMockTests
{
    private readonly Mock<BlogContext> _contextMock;
    private readonly Mock<DbSet<Blog>> _blogDbSetMock;

    public BlogServiceMockTests()
    {
        _blogDbSetMock = new Mock<DbSet<Blog>>();
        _contextMock = new Mock<BlogContext>();
    }

    [Fact]
    public async Task GetBlog_WithMockedContext_ShouldReturnBlog()
    {
        // Arrange
        var testBlog = TestDataFactory.CreateTestBlog();
        var blogs = new List<Blog> { testBlog }.AsQueryable();

        _blogDbSetMock.As<IQueryable<Blog>>()
            .Setup(m => m.Provider)
            .Returns(blogs.Provider);
        _blogDbSetMock.As<IQueryable<Blog>>()
            .Setup(m => m.Expression)
            .Returns(blogs.Expression);
        _blogDbSetMock.As<IQueryable<Blog>>()
            .Setup(m => m.ElementType)
            .Returns(blogs.ElementType);
        _blogDbSetMock.As<IQueryable<Blog>>()
            .Setup(m => m.GetEnumerator())
            .Returns(blogs.GetEnumerator());

        _contextMock.Setup(c => c.Blogs)
            .Returns(_blogDbSetMock.Object);

        var service = new BlogService(_contextMock.Object);

        // Act
        var result = await service.GetBlogAsync(testBlog.Id);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(testBlog.Title, result.Title);
    }
}
```

### 2. Repository Mocking

```csharp
public class BlogServiceWithMockedRepositoryTests
{
    private readonly Mock<IBlogRepository> _repositoryMock;
    private readonly BlogService _service;

    public BlogServiceWithMockedRepositoryTests()
    {
        _repositoryMock = new Mock<IBlogRepository>();
        _service = new BlogService(_repositoryMock.Object);
    }

    [Fact]
    public async Task GetBlogById_WhenBlogExists_ReturnsCorrectBlog()
    {
        // Arrange
        var testBlog = TestDataFactory.CreateTestBlog();
        _repositoryMock.Setup(r => r.GetByIdAsync(It.IsAny<int>()))
            .ReturnsAsync(testBlog);

        // Act
        var result = await _service.GetBlogAsync(1);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(testBlog.Title, result.Title);
        _repositoryMock.Verify(r => r.GetByIdAsync(1), Times.Once);
    }

    [Fact]
    public async Task CreateBlog_ShouldCallRepository()
    {
        // Arrange
        var testBlog = TestDataFactory.CreateTestBlog();
        _repositoryMock.Setup(r => r.AddAsync(It.IsAny<Blog>()))
            .ReturnsAsync(testBlog);

        // Act
        await _service.CreateBlogAsync(testBlog);

        // Assert
        _repositoryMock.Verify(r => r.AddAsync(It.IsAny<Blog>()), Times.Once);
    }
}
```

## Integration Testing

### 1. Test Fixtures

```csharp
public class BlogIntegrationTests : IClassFixture<TestDatabaseFixture>
{
    private readonly TestDatabaseFixture _fixture;

    public BlogIntegrationTests(TestDatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task CompleteWorkflow_ShouldSucceed()
    {
        // Arrange
        using var context = _fixture.CreateContext();
        var service = new BlogService(context);
        var blog = TestDataFactory.CreateTestBlog();

        // Act - Create
        var createdBlog = await service.CreateBlogAsync(blog);

        // Assert - Create
        Assert.NotNull(createdBlog);
        Assert.NotEqual(0, createdBlog.Id);

        // Act - Update
        createdBlog.Title = "Updated Title";
        var updatedBlog = await service.UpdateBlogAsync(createdBlog);

        // Assert - Update
        Assert.Equal("Updated Title", updatedBlog.Title);

        // Act - Delete
        await service.DeleteBlogAsync(updatedBlog.Id);

        // Assert - Delete
        var deletedBlog = await service.GetBlogAsync(updatedBlog.Id);
        Assert.Null(deletedBlog);
    }
}
```

### 2. Transaction Scope

```csharp
public class TransactionalTestBase : IDisposable
{
    protected readonly BlogContext Context;
    protected readonly IDbContextTransaction Transaction;

    public TransactionalTestBase()
    {
        Context = new BlogContext(new DbContextOptionsBuilder<BlogContext>()
            .UseSqlServer("your_test_connection_string")
            .Options);

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

## Best Practices

1. Test Organization

```csharp
// DO: Organize tests by feature
[Collection("Blog Tests")]
public class BlogCreationTests
{
    [Fact]
    public async Task CreateBlog_WithValidData_ShouldSucceed() { }

    [Fact]
    public async Task CreateBlog_WithInvalidData_ShouldThrowException() { }
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
- Resource cleanup

## Important Notes

- Test database security
- Environment-specific configuration
- Data cleanup strategies
- Transaction handling
- Async/await usage
- Exception handling
- Test reliability
- CI/CD considerations
