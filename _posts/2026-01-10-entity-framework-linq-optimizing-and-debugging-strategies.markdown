---
layout: default
title: "Writing Optimized Entity Framework LINQ Queries"
date: 2026-01-10 11:00:00 +0530
categories: engineering entity-framework linq performance
---

Entity Framework can generate inefficient SQL when you're not careful with LINQ expressions. This guide focuses on writing optimized queries by following core principles that prevent common performance problems.

**Version**: This article assumes EF Core 6+. Some techniques require specific versions, which are noted inline.

**Target Audience**: Developers familiar with EF Core basics who need to write high-performance queries for production systems.

---

## Core Optimization Principles

All EF query optimization follows four core principles:

1. **Minimize Data Transfer**: Fetch only the data you need
2. **Minimize Round Trips**: Reduce database calls
3. **Preserve Query Translation**: Keep logic translatable to SQL
4. **Control Memory Usage**: Manage change tracking and object materialization

Every technique in this post maps to one or more of these principles.

---

## Principle 1: Minimize Data Transfer

Transfer only the data you actually need from the database.

### Use Projection Instead of Full Entities

```csharp
// Bad: Loads all columns for all customers
var customers = context.Customers
    .Where(c => c.Active)
    .ToList();

// Good: Loads only required columns
var customerData = context.Customers
    .Where(c => c.Active)
    .Select(c => new { c.Id, c.Name, c.Email })
    .ToList();
```

**Why this works**: Projection generates a SELECT with only the columns you specify. This reduces network transfer and memory allocation.

### Filtered Includes

```csharp
// EF Core 5+: Filter included data
var orders = context.Orders
    .Include(o => o.OrderItems.Where(i => i.Quantity > 0))
    .Where(o => o.CreatedDate > cutoffDate)
    .ToList();
```

**When to use**: When you need related data but only a subset of it. This prevents loading unnecessary child records.

### Pagination for Large Result Sets

```csharp
// Always paginate large queries
var page = context.Orders
    .OrderBy(o => o.OrderDate)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToList();
```

**Rule**: Never load unbounded result sets in production. Always use Skip/Take for pagination.

---

## Principle 2: Minimize Round Trips

Reduce the number of database calls.

### Eager Loading to Avoid N+1 Queries

```csharp
// Bad: N+1 queries (1 query for orders + N queries for customers)
var orders = context.Orders.ToList();
foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name); // Separate query per order
}

// Good: Single query with JOIN
var orders = context.Orders
    .Include(o => o.Customer)
    .ToList();
```

**Why this matters**: Each database round trip has latency cost. N+1 queries can turn a 10ms operation into seconds.

### Conditional Includes

```csharp
public IQueryable<Order> GetOrders(bool includeCustomer, bool includeItems)
{
    var query = context.Orders.AsQueryable();
    
    if (includeCustomer)
        query = query.Include(o => o.Customer);
        
    if (includeItems)
        query = query.Include(o => o.OrderItems);
        
    return query;
}
```

**When to use**: When different code paths need different levels of data. This avoids over-fetching while preventing N+1.

### Split Queries for Cartesian Explosion (EF Core 5+)

```csharp
// Problem: Multiple Includes create cartesian product
var orders = context.Orders
    .Include(o => o.Customer)
    .Include(o => o.OrderItems)      // Creates huge result set
    .Include(o => o.ShippingDetails)
    .ToList();

// Solution: Split into multiple queries
var orders = context.Orders
    .Include(o => o.Customer)
    .Include(o => o.OrderItems)
    .Include(o => o.ShippingDetails)
    .AsSplitQuery()
    .ToListAsync();
```

**Trade-off**: Split queries use more round trips but transfer less data. Use when Includes create large result sets (typically 1:N:N relationships).

**Real example**: In production, one Include on a 1:N:N graph generated 30MB per query and crashed under load. AsSplitQuery fixed it by using 3 smaller queries instead of 1 huge cartesian join.

### Batch Multiple Operations

```csharp
// Bad: Multiple SaveChanges calls
foreach (var customer in customers)
{
    context.Customers.Add(customer);
    context.SaveChanges(); // Database call per customer
}

// Good: Single SaveChanges
context.Customers.AddRange(customers);
context.SaveChanges(); // One transaction
```

---

## Principle 3: Preserve Query Translation

Keep LINQ expressions translatable to SQL so EF doesn't fall back to client evaluation.

### Avoid Non-Translatable Methods

```csharp
// Bad: Forces client-side evaluation
var results = context.Customers
    .Where(c => c.Name.Contains("John"))
    .ToList()
    .Where(c => SomeComplexMethod(c.Email)); // Runs in C#, not SQL

// Good: Use database functions
var results = context.Customers
    .Where(c => c.Name.Contains("John") && 
                EF.Functions.Like(c.Email, "%@company.com"))
    .ToList();
```

**Rule**: Call ToList() only when you're done building the query. Everything after ToList() runs in memory, not the database.

### Use Database Functions for Complex Logic

```csharp
// Register custom database function
public static class CustomFunctions
{
    [DbFunction]
    public static decimal CalculateDiscount(decimal price, int customerId)
        => throw new NotImplementedException(); // Never called
}

// Configure in OnModelCreating
modelBuilder.HasDbFunction(typeof(CustomFunctions)
    .GetMethod(nameof(CustomFunctions.CalculateDiscount)))
    .HasName("fn_CalculateDiscount");

// Use in query
var discounts = context.Orders
    .Select(o => new {
        o.Id,
        Discount = CustomFunctions.CalculateDiscount(o.Total, o.CustomerId)
    })
    .ToList();
```

**When to use**: When business logic must run in SQL (for performance) and you can't express it in LINQ.

**When NOT to use**: For logic that needs external dependencies, is non-deterministic, or makes testing hard.

### Keep Expressions Simple

```csharp
// Bad: Complex nested conditions are hard to translate
var orders = context.Orders
    .Where(o => (o.Status == OrderStatus.Pending && 
                 o.CreatedDate > cutoff && 
                 (o.Priority > 5 || o.Customer.VipLevel > 2)) ||
                (o.Status == OrderStatus.Processing && 
                 o.AssignedTo != null))
    .ToList();

// Good: Build expressions step by step
Expression<Func<Order, bool>> urgentPending = o => 
    o.Status == OrderStatus.Pending && 
    o.CreatedDate > cutoff && 
    (o.Priority > 5 || o.Customer.VipLevel > 2);

Expression<Func<Order, bool>> assigned = o => 
    o.Status == OrderStatus.Processing && o.AssignedTo != null;

var filter = urgentPending.Or(assigned);
var orders = context.Orders.Where(filter).ToList();
```

**Why**: Simple expressions translate more reliably and generate better SQL.

---

## Principle 4: Control Memory Usage

Manage how EF tracks objects and uses memory.

### Use AsNoTracking for Read-Only Queries

```csharp
// Bad: Tracks all entities (uses memory, slower)
var customers = context.Customers
    .Where(c => c.Active)
    .ToList();

// Good: No tracking for read-only data
var customers = context.Customers
    .AsNoTracking()
    .Where(c => c.Active)
    .ToList();
```

**Rule**: Always use AsNoTracking for queries where you won't update the data. It's faster and uses less memory.

**Benchmark**: AsNoTracking can be 30-40% faster for large result sets.

### Disable Change Tracking for Bulk Operations

```csharp
public async Task BulkInsertAsync(IEnumerable<Customer> customers)
{
    context.ChangeTracker.AutoDetectChangesEnabled = false;
    
    try
    {
        foreach (var batch in customers.Chunk(1000))
        {
            context.Customers.AddRange(batch);
            await context.SaveChangesAsync();
            context.ChangeTracker.Clear(); // Release memory
        }
    }
    finally
    {
        context.ChangeTracker.AutoDetectChangesEnabled = true;
    }
}
```

**When to use**: For bulk inserts/updates (thousands of rows). Change tracking overhead becomes significant at scale.

### Clear Change Tracker in Long-Running Operations

```csharp
// Process large batch in chunks
for (int i = 0; i < totalCount; i += batchSize)
{
    var batch = context.Orders
        .AsNoTracking()
        .Skip(i)
        .Take(batchSize)
        .ToList();
    
    ProcessBatch(batch);
    
    context.ChangeTracker.Clear(); // Prevent memory growth
}
```

---

## Common Anti-Patterns

### Anti-Pattern 1: Using Include When Only Aggregates Are Needed

```csharp
// Bad: Include loads all child rows unnecessarily
var summaries = context.Orders
    .Include(o => o.OrderItems) // Loads ALL OrderItems into memory
    .Select(o => new OrderSummary
    {
        Id = o.Id,
        ItemCount = o.OrderItems.Count,
        TotalValue = o.OrderItems.Sum(i => i.Quantity * i.UnitPrice)
    })
    .ToList();

// Good: Let database compute aggregates
var summaries = context.Orders
    .Select(o => new OrderSummary
    {
        Id = o.Id,
        ItemCount = o.OrderItems.Count(), // Translated to SQL aggregate
        TotalValue = o.OrderItems.Sum(i => i.Quantity * i.UnitPrice)
    })
    .ToList();
```

**Why this works**: EF Core translates aggregates in projections to SQL correlated subqueries or JOINs with GROUP BY. No N+1 problem. Include is for loading entity graphs, not for calculations.

### Anti-Pattern 2: Premature ToList()

```csharp
// Bad: Loads all data then filters in memory
var active = context.Customers
    .ToList()
    .Where(c => c.Active); // Runs in C#, not SQL

// Good: Filter in database
var active = context.Customers
    .Where(c => c.Active)
    .ToList();
```

### Anti-Pattern 3: Ignoring Global Query Filters

```csharp
// Configure global filter
modelBuilder.Entity<Order>()
    .HasQueryFilter(o => o.IsActive);

// Problem: Applies to ALL queries, may hurt performance
var orders = context.Orders.ToList(); // Filter added automatically

// Solution: Document clearly and bypass when needed
var allOrders = context.Orders
    .IgnoreQueryFilters()
    .ToList();
```

**Trade-off**: Global filters are convenient but invisible. They can slow queries and make debugging harder. Use sparingly.

---

## Advanced Techniques

### Compiled Queries for Hot Paths

```csharp
// Compile frequently-used query
private static readonly Func<MyContext, DateTime, IEnumerable<Order>> 
    GetRecentOrders = EF.CompileQuery(
        (MyContext ctx, DateTime cutoff) => 
            ctx.Orders.Where(o => o.CreatedDate > cutoff));

// Use compiled query
public IEnumerable<Order> GetRecentOrders(DateTime cutoff)
{
    return GetRecentOrders(_context, cutoff).ToList();
}
```

**When to use compiled queries**:
- Query executed very frequently (thousands of times per second)
- Query shape is identical across calls
- You've already optimized the generated SQL
- Expression tree parsing shows up in profiling

**When NOT to use**: Compiled queries are rarely the bottleneck. Most performance problems come from bad SQL, not LINQ translation overhead. Optimize SQL first.

**Limitation**: Compiled queries are synchronous. Don't wrap in Task.Run - that hurts ASP.NET Core performance.

### Raw SQL for Complex Scenarios

```csharp
// Hierarchical data with recursive CTE
var hierarchy = context.Categories
    .FromSqlRaw(@"
        WITH CategoryHierarchy AS (
            SELECT Id, Name, ParentId, 0 as Level
            FROM Categories 
            WHERE ParentId IS NULL
            
            UNION ALL
            
            SELECT c.Id, c.Name, c.ParentId, ch.Level + 1
            FROM Categories c
            INNER JOIN CategoryHierarchy ch ON c.ParentId = ch.Id
        )
        SELECT * FROM CategoryHierarchy")
    .ToList();
```

**When to use**: When LINQ can't express the query (recursion, complex window functions). But you lose type safety and composability.

### Window Functions

```csharp
// Use FromSqlRaw for window functions
var rankings = context.Set<SalesRanking>()
    .FromSqlRaw(@"
        SELECT *, 
               ROW_NUMBER() OVER (ORDER BY TotalSales DESC) as Rank,
               SUM(TotalSales) OVER (PARTITION BY Region) as RegionalTotal
        FROM CustomerSales")
    .ToList();
```

---

## Index Awareness: The Missing Piece

EF optimization is meaningless without proper indexes. Even perfect LINQ generates slow SQL if indexes are missing.

### Index for WHERE Clauses

```sql
-- Query: context.Orders.Where(o => o.Status == "Pending")
CREATE INDEX IX_Orders_Status ON Orders (Status);
```

### Index for JOIN Columns

```sql
-- Query: context.Orders.Include(o => o.Customer)
-- Foreign key should have index
CREATE INDEX IX_Orders_CustomerId ON Orders (CustomerId);
```

### Composite Index for ORDER BY + WHERE

```csharp
// Query with filter + sort
var orders = context.Orders
    .Where(o => o.Status == "Pending")
    .OrderBy(o => o.CreatedDate)
    .Skip(0)
    .Take(20)
    .ToList();
```

```sql
-- Composite index: filter columns first, then sort columns
CREATE INDEX IX_Orders_Status_CreatedDate 
ON Orders (Status, CreatedDate DESC);
```

**Critical**: Pagination without proper index on ORDER BY column is a disaster. The database must sort the entire table before applying Skip/Take.

### Covering Indexes

```csharp
// Projection query
var data = context.Orders
    .Where(o => o.Status == "Pending")
    .Select(o => new { o.Id, o.Total, o.CreatedDate })
    .ToList();
```

```sql
-- Covering index includes all columns in SELECT
CREATE INDEX IX_Orders_Status_Covering 
ON Orders (Status) 
INCLUDE (Id, Total, CreatedDate);
```

**Why**: Covering index allows index-only scan. Database doesn't need to read the table at all.

---

## Query Plan Visibility

EF optimization is meaningless if you don't inspect execution plans.

### View Generated SQL

```csharp
// Method 1: ToQueryString (EF Core 5+)
var query = context.Orders.Where(o => o.Status == "Pending");
var sql = query.ToQueryString();
Console.WriteLine(sql);

// Method 2: Logging
optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information);
```

### Analyze Execution Plans

**SQL Server**:
```sql
-- View actual execution plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
-- Run your query
-- Check "Actual Execution Plan" in SSMS
```

**PostgreSQL**:
```sql
EXPLAIN ANALYZE
SELECT * FROM Orders WHERE Status = 'Pending';
```

**What to look for**:
- Table scans instead of index seeks
- High row counts in intermediate steps
- Missing index warnings
- Expensive sort operations
- Implicit conversions

**Rule**: Never deploy a query to production without checking its execution plan under realistic data volumes.

---

## Decision Table: Include vs Projection vs Split Query

| Scenario | Use Include | Use Projection | Use Split Query |
|----------|-------------|----------------|------------------|
| Need full child entities | ✅ Yes | ❌ No | Maybe (if large) |
| Need only child aggregates | ❌ No | ✅ Yes | ❌ No |
| Multiple 1:N relationships | ❌ No | Maybe | ✅ Yes |
| 1:N:N relationships | ❌ No | Maybe | ✅ Yes |
| Read-only query | Use with AsNoTracking | ✅ Yes | Use with AsNoTracking |
| Will update entities | ✅ Yes | ❌ No | ✅ Yes |
| Small result sets (<100 rows) | ✅ Yes | Either | ❌ No |
| Large result sets (>1000 rows) | Maybe | ✅ Yes | ✅ Yes |

---

## Verification Checklist

Before submitting queries for code review, check:

- [ ] Used projection to fetch only needed columns
- [ ] Applied AsNoTracking for read-only queries
- [ ] Used Include to prevent N+1 queries (only when loading entity graphs)
- [ ] Applied pagination for large result sets
- [ ] Avoided ToList() in the middle of query building
- [ ] Used database functions instead of C# methods in Where/Select
- [ ] Disabled change tracking for bulk operations
- [ ] Verified generated SQL (use ToQueryString() or logging)
- [ ] Checked execution plan for table scans and missing indexes
- [ ] Verified indexes exist for WHERE, JOIN, and ORDER BY columns
- [ ] Confirmed pagination queries use indexed columns for ORDER BY

---

## Summary

Write optimized EF queries by following these principles:

1. **Minimize Data Transfer**: Use projection, pagination, filtered includes
2. **Minimize Round Trips**: Use eager loading, batch operations, consider split queries
3. **Preserve Query Translation**: Keep expressions simple, use database functions
4. **Control Memory**: Use AsNoTracking, clear change tracker, disable tracking for bulk ops

The biggest performance gains come from:
1. Understanding what SQL EF generates
2. Ensuring proper indexes exist
3. Checking execution plans under realistic data volumes

Optimized LINQ without indexes is still slow. Always verify generated SQL and execution plans for queries in hot paths.

---

## Additional Resources

- [EF Core Query Translation](https://docs.microsoft.com/en-us/ef/core/querying/)
- [Performance Best Practices](https://docs.microsoft.com/en-us/ef/core/performance/)
- [Query Tags and Logging](https://docs.microsoft.com/en-us/ef/core/querying/tags/)

