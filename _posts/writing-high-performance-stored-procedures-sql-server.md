
---
layout: default
title: "Writing High-Performance Stored Procedures in SQL Server"
date: 2025-12-12
categories: engineering tutorial
---


Stored procedures are a powerful feature in SQL Server, enabling encapsulation of business logic, improved security, and performance optimization. However, performance issues usually show up in a few “hot paths”: schedule search, visit/EVV posting, payroll runs, invoice generation, authorization checks, dashboards, and large agency reporting. Most slowdowns come from predictable causes—non-SARGable filters, bad parameter sniffing, overusing scalar functions, poor indexing alignment, and unnecessary row-by-row logic.

This guide lists the pointers to keep in mind while writing stored procedures so that performance stays “up to the mark” as data grows.

## Best Practices for High-Performance Stored Procedures

### 1. Make predicates SARGable (index-friendly)
Most “mysterious slowness” is due to non-SARGable WHERE clauses.
Good (SARGable):
    WHERE VisitStart >= @FromDate
    AND VisitStart <  @ToDate

Bad (non-SARGable):
    WHERE CONVERT(date, VisitStart) = @Date

Other non-SARGable patterns to avoid:

    LIKE '%text%' (leading wildcard)
    ISNULL(Column, 0) = @x
    Column + 1 = @x
    functions on columns in WHERE/JOIN

Fix: move transformations to parameters (compute @StartOfDay/@EndOfDay), not on the column

### 2. Index alignment: join and filter on indexed columns (but don’t blindly add indexes)
Know the common filters for the module (AgencyId, ClientId, CaregiverId, VisitDate/StartTime, Status, EVV flags).

Ensure joins are typically on keys (PK/FK) or indexed columns.

For search/report endpoints, consider covering indexes with INCLUDE columns used by SELECT/ORDER BY.

### 3. JOIN vs EXISTS vs IN: when to use what
Use EXISTS for “presence checks”

If you only care whether related rows exist, EXISTS is usually better than joining and then grouping/distinct-ing.

Example: “Only visits that have an clock-in"
    WHERE EXISTS (
    SELECT 1
    FROM EVV v
    WHERE v.VisitId = Visits.VisitId
    )

Use JOIN when you need columns from the other table

If you need to return values from the related table, you must join (or use APPLY).

Use IN carefully

IN (subquery) can be fine, but EXISTS is often clearer and safer for correlation. Avoid huge literal IN (...) lists.

### 4. Temp tables vs table variables (and when to use each)
Table variables (@t)

Good for:

very small row counts,
simple logic,
when you don’t need indexes/stats (or you add a primary key).

Risk:

can mislead the optimizer about cardinality (depending on SQL Server version and compatibility level).

may choose bad plans when rowcount is larger than expected.

Temp tables (#t)

Good for:

medium/large intermediate sets,
multi-step transformations,
when indexes/stats will help the optimizer,
when you re-use the same intermediate result multiple times.

Rule of thumb:

If you expect more than a few hundred rows or plan quality matters, prefer #temp.
If you need indexing on the intermediate set, #temp is typically the right answer.

### 5. NOLOCK: understand it before using it
WITH (NOLOCK) / READ UNCOMMITTED can reduce blocking, but it can return:

dirty reads (uncommitted data),
missing rows or duplicate rows (due to allocation-order scans),
inconsistent results (especially for financial/payroll/invoice).

When it can be acceptable

Non-critical dashboards
Non-financial read-only data

### 6. Parameter sniffing: plan stability matters
SQL Server compiles a plan based on the first parameter values it “sniffs”. For multi-tenant systems, one tenant might have 200 rows and another has 20M—same procedure, radically different optimal plan.

Symptoms

Fast sometimes, slow sometimes
Works in QA, slow in prod
Big variance by tenant/date range

Mitigations (use selectively)

Ensure good indexes and SARGable predicates first.
For highly skewed parameters, consider:
    OPTION (RECOMPILE) for certain queries (costly but reliable for hot paths with skew)
    OPTIMIZE FOR UNKNOWN
    split logic: “small range” vs “large range” branches
    dynamic SQL (parameterized) for very conditional filters

Don’t blanket RECOMPILE everything—use it where skew is known and costly.

### 7. Use set-based logic, avoid RBAR (“row by agonizing row”)
Avoid:

cursors,
loops that call queries per row,
scalar UDFs inside large SELECTs (often performance killers).

Prefer:

set-based updates/inserts,
window functions (ROW_NUMBER, SUM() OVER),
MERGE only when you truly need it (and test carefully), otherwise UPDATE + INSERT patterns.

### 8. Tempdb and memory spills: prevent “silent killers”
Common causes:

huge sorts (ORDER BY without supporting index),
hash joins on large sets,
wide SELECTs causing large memory grants.

Developer habits to reduce this:

return only required columns,
filter early,
index to support joins/order,
materialize intermediate results into #temp with a targeted index when needed.


```sql
CREATE PROCEDURE usp_GetTopSellingProducts
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;

    SELECT TOP 10
        p.ProductID,
        p.ProductName,
        SUM(od.Quantity) AS TotalSold
    FROM
        OrderDetails od
        INNER JOIN Products p ON od.ProductID = p.ProductID
        INNER JOIN Orders o ON od.OrderID = o.OrderID
    WHERE
        o.OrderDate BETWEEN @StartDate AND @EndDate
    GROUP BY
        p.ProductID, p.ProductName
    ORDER BY
        TotalSold DESC;
END