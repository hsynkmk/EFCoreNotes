# Security Best Practices in EF Core

## Overview

Security is a critical aspect of any application using EF Core. This guide covers essential security practices, from protecting sensitive data to preventing common vulnerabilities.

## Key Areas

- Connection string security
- SQL injection prevention
- Sensitive data protection
- Authentication and authorization
- Data encryption
- Audit logging
- Access control
- Security configuration

## Connection String Security

### 1. Secure Storage

```csharp
// DON'T: Hard-code connection strings
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    // Bad practice
    optionsBuilder.UseSqlServer("Server=.;Database=BlogDb;User=sa;Password=123;");
}

// DO: Use configuration management
public class BlogContext : DbContext
{
    private readonly IConfiguration _configuration;

    public BlogContext(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer(
            _configuration.GetConnectionString("BlogDatabase")
        );
    }
}
```

### 2. Encrypted Configuration

```json
// secrets.json
{
  "ConnectionStrings": {
    "BlogDatabase": "Server=.;Database=BlogDb;Trusted_Connection=True;Encrypt=True;"
  }
}
```

## SQL Injection Prevention

### 1. Parameterized Queries

```csharp
// DON'T: Use string concatenation
// Bad practice
var title = "User Input";
var blogs = context.Blogs
    .FromSqlRaw($"SELECT * FROM Blogs WHERE Title = '{title}'");

// DO: Use parameters
public async Task<Blog> GetBlogByTitle(string title)
{
    return await context.Blogs
        .FromSqlRaw(
            "SELECT * FROM Blogs WHERE Title = @title",
            new SqlParameter("@title", title)
        )
        .FirstOrDefaultAsync();
}
```

### 2. LINQ Queries

```csharp
// DO: Use LINQ methods
public async Task<List<Blog>> SearchBlogs(string searchTerm)
{
    return await context.Blogs
        .Where(b => b.Title.Contains(searchTerm))
        .ToListAsync();
}
```

## Sensitive Data Protection

### 1. Data Encryption

```csharp
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }

    // DO: Encrypt sensitive data
    [Encrypted]
    public string SocialSecurityNumber { get; set; }
}

// Custom encryption value converter
public class EncryptedConverter : ValueConverter<string, string>
{
    public EncryptedConverter() : base(
        v => Encrypt(v),
        v => Decrypt(v))
    {}

    private static string Encrypt(string value)
    {
        // Implement secure encryption
        return Convert.ToBase64String(
            ProtectedData.Protect(
                Encoding.UTF8.GetBytes(value),
                null,
                DataProtectionScope.CurrentUser
            )
        );
    }

    private static string Decrypt(string value)
    {
        // Implement secure decryption
        return Encoding.UTF8.GetString(
            ProtectedData.Unprotect(
                Convert.FromBase64String(value),
                null,
                DataProtectionScope.CurrentUser
            )
        );
    }
}
```

### 2. Data Masking

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.Property(u => u.SocialSecurityNumber)
            .HasConversion<EncryptedConverter>();

        // Mask sensitive data in queries
        builder.Property(u => u.CreditCardNumber)
            .HasConversion(
                v => v,
                v => string.IsNullOrEmpty(v) ? v : $"****-****-****-{v.Substring(v.Length - 4)}");
    }
}
```

## Authentication and Authorization

### 1. User Context

```csharp
public class SecurityContext
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public int CurrentUserId =>
        int.Parse(_httpContextAccessor.HttpContext.User.FindFirst(ClaimTypes.NameIdentifier).Value);

    public bool IsAdmin =>
        _httpContextAccessor.HttpContext.User.IsInRole("Admin");
}

public class BlogService
{
    private readonly SecurityContext _securityContext;

    public async Task<Blog> GetBlog(int blogId)
    {
        var blog = await _context.Blogs
            .FirstOrDefaultAsync(b => b.Id == blogId);

        if (blog != null && !_securityContext.IsAdmin &&
            blog.AuthorId != _securityContext.CurrentUserId)
        {
            throw new UnauthorizedAccessException();
        }

        return blog;
    }
}
```

### 2. Row-Level Security

```csharp
public class SecureDbContext : DbContext
{
    private readonly SecurityContext _securityContext;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply row-level security
        modelBuilder.Entity<Blog>()
            .HasQueryFilter(b =>
                b.IsPublic ||
                b.AuthorId == _securityContext.CurrentUserId ||
                _securityContext.IsAdmin);
    }
}
```

## Audit Logging

### 1. Change Tracking

```csharp
public class AuditEntry
{
    public int Id { get; set; }
    public string EntityName { get; set; }
    public string Action { get; set; }
    public string Changes { get; set; }
    public int UserId { get; set; }
    public DateTime Timestamp { get; set; }
}

public class AuditableContext : DbContext
{
    private readonly SecurityContext _securityContext;

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var auditEntries = new List<AuditEntry>();

        foreach (var entry in ChangeTracker.Entries())
        {
            if (entry.State == EntityState.Modified ||
                entry.State == EntityState.Added ||
                entry.State == EntityState.Deleted)
            {
                auditEntries.Add(new AuditEntry
                {
                    EntityName = entry.Entity.GetType().Name,
                    Action = entry.State.ToString(),
                    Changes = JsonSerializer.Serialize(entry.CurrentValues),
                    UserId = _securityContext.CurrentUserId,
                    Timestamp = DateTime.UtcNow
                });
            }
        }

        var result = await base.SaveChangesAsync(cancellationToken);

        if (auditEntries.Any())
        {
            await AuditEntries.AddRangeAsync(auditEntries);
            await base.SaveChangesAsync(cancellationToken);
        }

        return result;
    }
}
```

### 2. Activity Logging

```csharp
public class SecurityLogger
{
    private readonly ILogger<SecurityLogger> _logger;

    public void LogSecurityEvent(string action, string details, int userId)
    {
        _logger.LogInformation(
            "Security Event: {Action} by User {UserId} - {Details}",
            action, userId, details);
    }
}
```

## Best Practices

1. Input Validation

```csharp
public class BlogValidator : AbstractValidator<Blog>
{
    public BlogValidator()
    {
        RuleFor(x => x.Title)
            .NotEmpty()
            .MaximumLength(200)
            .Matches(@"^[a-zA-Z0-9\s-]*$"); // Allow only alphanumeric

        RuleFor(x => x.Url)
            .NotEmpty()
            .Must(uri => Uri.TryCreate(uri, UriKind.Absolute, out _));
    }
}
```

2. Exception Handling

```csharp
public async Task<IActionResult> UpdateBlog(int id, BlogDto blogDto)
{
    try
    {
        var blog = await _context.Blogs.FindAsync(id);
        if (blog == null) return NotFound();

        if (blog.AuthorId != _securityContext.CurrentUserId)
            return Unauthorized();

        // Update blog
        await _context.SaveChangesAsync();
        return Ok();
    }
    catch (DbUpdateConcurrencyException)
    {
        // Log and handle concurrency conflicts
        return Conflict();
    }
    catch (Exception ex)
    {
        // Log but don't expose error details
        _logger.LogError(ex, "Error updating blog {Id}", id);
        return StatusCode(500, "An error occurred");
    }
}
```

## Performance Considerations

- Encryption overhead
- Audit log storage
- Query filter impact
- Authentication caching
- Connection pool security
- Transaction isolation
- Logging performance
- Security scanning

## Important Notes

- Regular security audits
- Dependency updates
- Security patches
- Access review
- Encryption key management
- Compliance requirements
- Security monitoring
- Incident response
