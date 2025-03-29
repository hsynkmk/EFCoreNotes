# Entity Framework Core Notes

A structured guide to learning EF Core from foundational concepts to advanced techniques.

<details>
<summary>📚 Core Fundamentals</summary>

1. [DbContext and Entities](DbContextAndEntities.md)  
   - Context lifecycle, entity configuration
2. [Basic Queries](BasicQueries.md)  
   - LINQ essentials, CRUD operations
3. [Migrations](Migrations.md)  
   - Database schema management
</details>

<details>
<summary>📐 Data Modeling</summary>
   
4. [Relationships](Relationships.md)  
   - Navigation properties, relationship types
5. [Cascade Delete](CascadeDelete.md)  
   - Delete behaviors and referential integrity
6. [Global Query Filters](GlobalQueryFilters.md)  
   - Implementing soft delete patterns
   - Automatic filtering
7. [Value Conversions](ValueConversions.md)  
   - Custom type mappings
8. [Owned Entities](OwnedEntities.md)  
   - Complex type configurations
</details>

<details>
<summary>🔄 Data Operations</summary>

9. [Raw SQL Queries](RawSQLQueries.md)  
   - SQL execution strategies
10. [Transactions](Transactions.md)  
    - ACID operations management
11. [Bulk Operations](BulkOperations.md)  
    - High-performance batch processing
12. [Split Queries](SplitQueries.md)  
    - Solving N+1 query issues
</details>

<details>
<summary>⚡ Performance</summary>

13. [Indexes and Constraints](IndexesAndConstraints.md)  
    - Query optimization techniques
14. [Compiled Queries](CompiledQueries.md)  
    - Reusable query plans
15. [Performance Optimization](PerformanceOptimization.md)  
    - Benchmarking and tuning
</details>

<details>
<summary>🔍 Advanced Features</summary>

16. [Concurrency Handling](ConcurrencyHandling.md)  
    - Optimistic concurrency patterns
17. [Temporal Tables](TemporalTables.md)  
    - Historical data tracking
18. [Change Tracking](ChangeTracking.md)  
    - State management internals
19. [Shadow Properties](FieldMappingAndShadowProperties.md)  
    - Metadata without entity fields
</details>

<details>
<summary>📊 Query Techniques</summary>
   
20. [Lazy Loading](LazyLoading.md)  
    - On-demand navigation loading
21. [Advanced Querying](AdvancedQuerying.md)  
    - Complex LINQ patterns
22. [Query Tags](QueryTagsAndLogging.md)  
    - Query annotation for logging
</details>

<details>
<summary>🛡️ Security & Reliability</summary>

23. [Security Best Practices](SecurityBestPractices.md)  
    - SQL injection prevention
24. [Error Handling](ErrorHandling.md)  
    - Exception management strategies
25. [Concurrency Handling](ConcurrencyHandling.md)  
    - Data conflict resolution
</details>

<details>
<summary>🏗️ Application Integration</summary>

26. [Dependency Injection](DependencyInjection.md)  
    - IoC container configuration
27. [Unit Testing](UnitTesting.md)  
    - Test patterns for EF Core
28. [Integration Testing](Testing.md)  
    - Database testing strategies
</details>

<details>
<summary>🚀 Production Readiness</summary>

29. [Deployment Strategies](DeploymentAndConfiguration.md)  
    - Environment configurations
30. [Schema Versioning](VersioningAndSchemaEvolution.md)  
    - Managing breaking changes
31. [Monitoring](MonitoringAndDiagnostics.md)  
    - Health checks and diagnostics
32. [Logging](LoggingAndDiagnostics.md)  
    - Query logging and analysis
</details>

<details>
<summary>🔌 Database Providers</summary>

33. [Database Providers](DatabaseProviders.md)  
    - SQL Server/PostgreSQL/SQLite
</details>

## Contribution
- Contributions welcome! Please ensure:
- Follow existing documentation style
- Add practical examples
- Keep explanations concise
