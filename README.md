# Entity Framework Core Notes

A structured guide to learning EF Core from foundational concepts to advanced techniques.

<details>
<summary>📚 Foundational Concepts</summary>

1. [DbContext and Entities](DbContextAndEntities.md)  
   - Core building blocks of EF Core
   - Context configuration and entity mapping

2. [Basic Queries](BasicQueries.md)  
   - CRUD operations fundamentals
   - LINQ syntax basics

3. [Migrations](Migrations.md)  
   - Database schema evolution
   - Migration commands and workflows
</details>

<details>
<summary>📐 Data Modeling</summary>

4. [Relationships](Relationships.md)  
   - Configuring 1:1, 1:many, many:many relationships

5. [Cascade Delete](CascadeDelete.md)  
   - Delete behaviors and referential integrity

6. [Global Query Filters](GlobalQueryFilters.md)  
   - Implementing soft delete patterns
   - Automatic filtering
</details>

<details>
<summary>🛠 Data Manipulation</summary>

7. [Value Conversions](ValueConversions.md)  
   - Custom data type mappings

8. [Owned Entities](OwnedEntities.md)  
   - Complex types and value objects

9. [Raw SQL Queries](RawSQLQueries.md)  
   - Executing SQL commands safely

10. [Transactions](Transactions.md)  
    - Managing atomic operations
</details>

<details>
<summary>⚡ Performance</summary>

11. [Indexes and Constraints](IndexesAndConstraints.md)  
    - Optimizing query performance

12. [Compiled Queries](CompiledQueries.md)  
    - Reusing query execution plans

13. [Bulk Operations](BulkOperations.md)  
    - Efficient bulk inserts/updates
</details>

<details>
<summary>🔐 Advanced Concepts</summary>

14. [Concurrency Handling](ConcurrencyHandling.md)  
    - Optimistic concurrency patterns

15. [Logging and Diagnostics](LoggingAndDiagnostics.md)  
    - Monitoring EF Core behavior

16. [Shadow Properties](FieldMappingAndShadowProperties.md)  
    - Metadata without entity properties

17. [Database Providers](DatabaseProviders.md)  
    - Working with different SQL engines
</details>

<details>
<summary>🏗 Application Integration</summary>

18. [Dependency Injection](DependencyInjection.md)  
    - Configuring EF Core in DI containers

19. [Testing](Testing.md)  
    - Unit testing strategies

20. [Error Handling](ErrorHandling.md)  
    - Exception management patterns
</details>

<details>
<summary>🚀 Production Ready</summary>

21. [Deployment and Configuration](DeploymentAndConfiguration.md)  
    - Environment-specific settings

22. [Versioning and Schema Evolution](VersioningAndSchemaEvolution.md)  
    - Managing breaking changes

23. [Security Best Practices](SecurityBestPractices.md)  
    - SQL injection prevention
</details>

<details>
<summary>🧠 Expert Techniques</summary>

24. [Temporal Tables](TemporalTables.md)  
    - Historical data tracking

25. [Change Tracking](ChangeTracking.md)  
    - State management deep dive

26. [Advanced Querying](AdvancedQuerying.md)  
    - Complex LINQ patterns
</details>

## Suggested Learning Path
1. Start with **Foundational Concepts**
2. Progress through **Data Modeling**
3. Master **Data Manipulation** techniques
4. Optimize with **Performance** section
5. Explore **Advanced Concepts**
6. Integrate with **Application Integration**
7. Prepare for production with **Production Ready**
8. Dive into **Expert Techniques**

```bash
# Clone repository
git clone https://github.com/hsynkmk/EFCoreNotes.git
```
## Contribution
- Contributions welcome! Please ensure:
- Follow existing documentation style
- Add practical examples
- Keep explanations concise
