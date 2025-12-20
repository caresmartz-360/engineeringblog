---
layout: default
title: "SQL Server Query Optimization: Avoid These Common Mistakes"
date: 2025-12-20 11:00:00 +0530
categories: engineering sql-server database performance
---

Your application is crawling. You're ready to throw money at new servers.

**Stop.** Before you spend thousands on hardware, check your queries. A single WHERE clause with a function can waste 95% of your I/O. A missing index on a join column can turn a 0.1-second query into a 30-second nightmare.

The good news? These problems are fixable. This guide covers the 12 mistakes in 90% of slow SQL databases—and how to destroy them.

## #1: Don't Use Functions on Columns in WHERE (The #1 Killer)

**This is the biggest performance mistake I see.** When you wrap a column in a function, SQL Server cannot use indexes. It must check every single row.

### The Problem

```
Index exists on HireDate
Query: WHERE YEAR(HireDate) = 2025
Result: Index is IGNORED. Must scan all rows calculating YEAR() for each one.
Time: 8 seconds for 2M rows
```

### Wrong Approaches
```sql
WHERE YEAR(OrderDate) = 2025
WHERE MONTH(OrderDate) = 12  
WHERE UPPER(LastName) = 'SMITH'
WHERE DATEDIFF(DAY, HireDate, GETDATE()) < 30
WHERE Salary * 1.1 > 50000
WHERE LEN(PhoneNumber) = 10
```

### Correct Approaches
```sql
WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01'
WHERE OrderDate >= '2025-12-01' AND OrderDate < '2026-01-01'
WHERE LastName = 'Smith'
WHERE HireDate >= DATEADD(DAY, -30, GETDATE())
WHERE Salary > 50000 / 1.1
WHERE PhoneNumber LIKE '[0-9][0-9][0-9]-[0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]'
```

**Impact:** Moving from YEAR(Date) to date ranges typically improves query time from 5-10 seconds to 0.1-0.5 seconds. **40x improvement.**

---

## #2: SELECT Only What You Need (Not SELECT *)

**SELECT \* wastes network bandwidth and memory.**

### Wrong
```sql
SELECT * FROM Employees;  -- 30 columns, 5KB per row
```

Returns 500MB for 100,000 rows.

### Right
```sql
SELECT EmployeeID, FirstName, LastName, Email FROM Employees;  -- 4 columns, 200 bytes
```

Returns 20MB for 100,000 rows. **25x less data.**

### The Real Cost
- **Network transfer time** is slow
- **Memory usage** increases
- **Covering indexes cannot help** (they'd need to include all 30 columns)

---

## #3: Add Indexes on Columns You Filter, Join, or Sort

**Indexes are lookup tables.** Without them, SQL Server must scan the entire table.

### Where to Add Indexes
```sql
-- Columns in WHERE clause
CREATE INDEX idx_Status ON Orders(Status);

-- Columns in JOIN conditions  
CREATE INDEX idx_CustomerID ON Orders(CustomerID);

-- Columns in ORDER BY
CREATE INDEX idx_OrderDate ON Orders(OrderDate);
```

### Covering Indexes: Get Everything from the Index
```sql
-- Regular index: finds rows, then goes to table for other columns (slow)
CREATE INDEX idx_Status ON Orders(Status);

-- Covering index: has everything needed, never touches table (fast)
CREATE INDEX idx_Status_Covering 
ON Orders(Status)
INCLUDE (OrderDate, Total, CustomerName);
```

**Example improvement:** 2.5 seconds → 0.1 seconds. **25x faster.**

---

## #4: Avoid Wildcards at the Start of LIKE

```sql
-- DON'T DO THIS (Cannot use index)
WHERE Email LIKE '%gmail.com'     

-- DO THIS (Can use index)
WHERE Email LIKE 'john%'          
WHERE Email LIKE 'john%@gmail.com'
```

---

## #5: Use EXISTS Instead of COUNT(*) > 0

```sql
-- Slow: checks ALL matching rows
SELECT * FROM Customers c
WHERE (SELECT COUNT(*) FROM Orders o WHERE o.CustomerID = c.CustomerID) > 0;

-- Fast: stops at first match
SELECT * FROM Customers c
WHERE EXISTS (
    SELECT 1 FROM Orders o WHERE o.CustomerID = c.CustomerID
);
```

**Impact:** EXISTS stops checking after finding one match. COUNT(*) must check all.

---

## #6: Use Inline TVF, Not Multi-Statement TVF

Table-valued functions have huge performance differences.

### Multi-Statement TVF (Slow)
```sql
CREATE FUNCTION dbo.GetActiveUsers(@MinSales INT)
RETURNS @Result TABLE (UserID INT, SalesCount INT)
AS
BEGIN
    INSERT INTO @Result
    SELECT UserID, COUNT(*) FROM Sales GROUP BY UserID
    HAVING COUNT(*) >= @MinSales;
    RETURN;
END;
```

SQL Server treats result as a black box. No index usage. **12 seconds.**

### Inline TVF (Fast)
```sql
CREATE FUNCTION dbo.GetActiveUsers(@MinSales INT)
RETURNS TABLE
AS
RETURN (
    SELECT UserID, COUNT(*) AS SalesCount FROM Sales 
    GROUP BY UserID
    HAVING COUNT(*) >= @MinSales
);
```

SQL Server merges this into your query. Uses indexes. **0.3 seconds. 40x faster.**

---

## #7: Don't Batch Large Operations

**Never update/delete millions of rows at once.** Lock the table, bloat the log, crash the server.

### Wrong
```sql
UPDATE Visits SET Processed = 1 WHERE VisitDate < '2020-01-01';
-- 10M rows, table locked for 20 minutes, other queries timeout
```

### Right
```sql
WHILE 1 = 1
BEGIN
    UPDATE TOP (5000) Visits 
    SET Processed = 1 
    WHERE VisitDate < '2020-01-01' AND Processed = 0;
    
    IF @@ROWCOUNT = 0 BREAK;
    WAITFOR DELAY '00:00:00.100';  -- Pause to let other queries run
END;
```

**Benefits:** Small transactions, no table locks, other queries work fine.

---

## #8: Check Execution Plans

Click "Include Actual Execution Plan" (Ctrl+M) when testing queries.

### Red Flags
- **Table Scan** - Scanning entire table (bad on large tables)
- **Key Lookup** - Going back to table for columns (use covering index)
- **Yellow warnings** - Missing indexes, implicit conversions

### Good Signs
- **Index Seek** - Using index correctly
- **Nested Loop** - Efficient for small datasets  
- **Hash Match** - Efficient for large datasets

**Example:** Found a Table Scan costing 92%. Added one index. Query: 45 seconds → 1.5 seconds.

---

## #9: Keep Statistics Updated

SQL Server uses statistics to **estimate** how many rows match your criteria. Old statistics = bad plans.

```sql
-- Update statistics monthly for large tables
UPDATE STATISTICS TableName;

-- Or entire database
EXEC sp_updatestats;
```

**Example:** Outdated stats said "10 rows will match" but actually 500,000 do. Wrong plan chosen. 18 seconds vs 2 seconds after updating.

---

## #10: Pagination Done Right

### Wrong (Scans Millions of Rows)
```sql
SELECT * FROM Orders ORDER BY OrderID 
OFFSET 50000 ROWS FETCH NEXT 100 ROWS ONLY;
-- Must sort and skip 50,000 rows every time
-- Page 1000: 12 seconds
```

### Right (Uses Index to Jump)
```sql
SELECT TOP 100 * FROM Orders 
WHERE OrderID > @LastOrderIDFromPreviousPage
ORDER BY OrderID;
-- Index jumps to position, returns 100 rows
-- Page 1000: 0.05 seconds
```

**Impact:** 200x faster on later pages.

---

## #11: Handle Parameter Sniffing

SQL Server creates a plan based on the **first parameter value**, then reuses it. Sometimes that's terrible for other values.

```sql
-- Solution: Recompile every time
SELECT * FROM Orders WHERE Status = @Status
OPTION (RECOMPILE);

-- Or use local variable trick
DECLARE @LocalStatus VARCHAR(20) = @Status;
SELECT * FROM Orders WHERE Status = @LocalStatus;
```

---

## #12: Avoid Cursors and Loops

SQL is set-based. Row-by-row defeats its purpose.

### Slow (Cursor)
```sql
-- 10,000 users: 8 minutes
DECLARE cur CURSOR FOR SELECT UserID FROM Users WHERE Active = 1;
OPEN cur;
FETCH NEXT FROM cur INTO @UserID;
WHILE @@FETCH_STATUS = 0
BEGIN
    UPDATE UserStats SET Total = (SELECT COUNT(*) FROM Orders WHERE UserID = @UserID)
    WHERE UserID = @UserID;
    FETCH NEXT FROM cur INTO @UserID;
END;
```

### Fast (Set-Based)
```sql
-- 10,000 users: 3 seconds
UPDATE u SET Total = (
    SELECT COUNT(*) FROM Orders WHERE UserID = u.UserID
)
FROM UserStats u
WHERE u.UserID IN (SELECT UserID FROM Users WHERE Active = 1);
-- 160x faster
```

---

## Quick Checklist

Before going to production:

- [ ] No functions on columns in WHERE clause
- [ ] No SELECT * (specific columns only)
- [ ] Indexes on WHERE, JOIN, and ORDER BY columns
- [ ] No leading wildcards in LIKE
- [ ] Using EXISTS, not COUNT(*) > 0
- [ ] Inline TVF instead of multi-statement
- [ ] Checked execution plan (no table scans on large tables)
- [ ] Statistics recently updated
- [ ] No cursors or loops

---

## When Query is Still Slow

- Query might need a **composite index** (multiple columns)
- Might need **query rewrites** (different logic)
- Might be a **locking or blocking** issue
- Might be **hardware-limited**


Most SQL Server performance issues are self-inflicted, not hardware-related.

Before scaling infrastructure, eliminate these first:

1. Never apply functions to indexed columns in WHERE — it kills index seeks
2. Avoid SELECT * — it wastes memory, bandwidth, and blocks covering indexes
3. Index what you filter, join, and sort on — prefer covering indexes
4. Use EXISTS over COUNT(*) > 0
5. Avoid leading wildcards in LIKE
6. Prefer inline TVFs — multi-statement TVFs are optimizer black holes
7. Batch large updates/deletes to avoid locks and log pressure
8. Kill cursors and loops — think set-based
9. Keep statistics fresh
10. Always inspect the execution plan — table scans and key lookups are red flags

Rule of thumb:
If you haven’t checked the execution plan, you haven’t finished the query.


Happy Coding!!!