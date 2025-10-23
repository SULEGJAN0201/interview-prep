# SQL Query Optimization

## 📝 Key Concepts

Query optimization is the process of improving SQL query performance by reducing execution time and resource consumption.

### Key Performance Metrics
- **Execution Time**: How long the query takes to run
- **Logical Reads**: Number of pages read from cache
- **Physical Reads**: Number of pages read from disk
- **CPU Time**: Processor time used
- **Rows Affected**: Number of rows processed

## 🔑 Core Optimization Techniques

### 1. Use Proper Indexing
Indexes speed up data retrieval but slow down INSERT/UPDATE/DELETE operations.

### 2. Avoid SELECT *
Only select columns you need to reduce data transfer and memory usage.

### 3. Use WHERE Clauses Effectively
Filter data as early as possible to reduce the result set.

### 4. Minimize Subqueries
Use JOINs instead of subqueries when possible.

### 5. Use EXISTS Instead of IN
For large datasets, EXISTS is often faster than IN.

## 💻 Code Examples

### SELECT * vs Specific Columns
```sql
-- BAD: Returns all columns (slow, more data transfer)
SELECT * 
FROM Orders 
WHERE CustomerId = 123;

-- GOOD: Returns only needed columns
SELECT OrderId, OrderDate, TotalAmount 
FROM Orders 
WHERE CustomerId = 123;
```

### Using Indexes Effectively
```sql
-- Check existing indexes
EXEC sp_helpindex 'Orders';

-- Create index on frequently queried column
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId 
ON Orders(CustomerId);

-- Composite index for multiple columns in WHERE clause
CREATE NONCLUSTERED INDEX IX_Orders_CustomerDate 
ON Orders(CustomerId, OrderDate);

-- Include additional columns for covering index
CREATE NONCLUSTERED INDEX IX_Orders_Customer_Covering 
ON Orders(CustomerId) 
INCLUDE (OrderDate, TotalAmount);
```

### WHERE Clause Optimization
```sql
-- BAD: Function on indexed column prevents index usage
SELECT * FROM Orders 
WHERE YEAR(OrderDate) = 2024;

-- GOOD: Use range comparison to allow index usage
SELECT * FROM Orders 
WHERE OrderDate >= '2024-01-01' 
  AND OrderDate < '2025-01-01';

-- BAD: Implicit conversion prevents index usage
SELECT * FROM Orders 
WHERE OrderId = '123'; -- OrderId is INT, but comparing with VARCHAR

-- GOOD: Use correct data type
SELECT * FROM Orders 
WHERE OrderId = 123;

-- BAD: Leading wildcard prevents index usage
SELECT * FROM Customers 
WHERE LastName LIKE '%Smith';

-- GOOD: No leading wildcard allows index usage
SELECT * FROM Customers 
WHERE LastName LIKE 'Smith%';
```

### EXISTS vs IN
```sql
-- GOOD: EXISTS stops when first match is found
SELECT c.CustomerId, c.Name
FROM Customers c
WHERE EXISTS (
    SELECT 1 
    FROM Orders o 
    WHERE o.CustomerId = c.CustomerId
);

-- SLOWER for large datasets: IN creates full list
SELECT c.CustomerId, c.Name
FROM Customers c
WHERE c.CustomerId IN (
    SELECT CustomerId 
    FROM Orders
);

-- BEST: Use JOIN when you need data from both tables
SELECT DISTINCT c.CustomerId, c.Name
FROM Customers c
INNER JOIN Orders o ON c.CustomerId = o.CustomerId;
```

### JOIN vs Subquery
```sql
-- BAD: Correlated subquery (executes for each row)
SELECT 
    o.OrderId,
    o.OrderDate,
    (SELECT Name FROM Customers WHERE CustomerId = o.CustomerId) AS CustomerName
FROM Orders o;

-- GOOD: JOIN (executes once)
SELECT 
    o.OrderId,
    o.OrderDate,
    c.Name AS CustomerName
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.CustomerId;
```

### Pagination Optimization
```sql
-- BAD: OFFSET/FETCH can be slow for large offsets
SELECT OrderId, OrderDate, TotalAmount
FROM Orders
ORDER BY OrderDate DESC
OFFSET 100000 ROWS 
FETCH NEXT 20 ROWS ONLY; -- Still scans 100,000 rows

-- GOOD: Key-based pagination (much faster)
SELECT TOP 20 OrderId, OrderDate, TotalAmount
FROM Orders
WHERE OrderDate < @LastOrderDate 
   OR (OrderDate = @LastOrderDate AND OrderId < @LastOrderId)
ORDER BY OrderDate DESC, OrderId DESC;
```

### Aggregate Optimization
```sql
-- BAD: Multiple queries
SELECT COUNT(*) FROM Orders WHERE Status = 'Pending';
SELECT COUNT(*) FROM Orders WHERE Status = 'Completed';
SELECT COUNT(*) FROM Orders WHERE Status = 'Cancelled';

-- GOOD: Single query with conditional aggregation
SELECT 
    SUM(CASE WHEN Status = 'Pending' THEN 1 ELSE 0 END) AS PendingCount,
    SUM(CASE WHEN Status = 'Completed' THEN 1 ELSE 0 END) AS CompletedCount,
    SUM(CASE WHEN Status = 'Cancelled' THEN 1 ELSE 0 END) AS CancelledCount
FROM Orders;

-- Or use GROUP BY
SELECT 
    Status,
    COUNT(*) AS StatusCount
FROM Orders
GROUP BY Status;
```

### UNION vs UNION ALL
```sql
-- SLOWER: UNION removes duplicates (requires sorting)
SELECT CustomerId FROM Customers WHERE City = 'New York'
UNION
SELECT CustomerId FROM Orders WHERE OrderDate > '2024-01-01';

-- FASTER: UNION ALL keeps duplicates (no sorting)
SELECT CustomerId FROM Customers WHERE City = 'New York'
UNION ALL
SELECT CustomerId FROM Orders WHERE OrderDate > '2024-01-01';
```

### Using CTE vs Temp Tables
```sql
-- CTE: Good for simple, one-time use
WITH RecentOrders AS (
    SELECT CustomerId, OrderDate, TotalAmount
    FROM Orders
    WHERE OrderDate >= DATEADD(DAY, -30, GETDATE())
)
SELECT 
    c.Name,
    COUNT(*) AS OrderCount,
    SUM(ro.TotalAmount) AS TotalSpent
FROM Customers c
INNER JOIN RecentOrders ro ON c.CustomerId = ro.CustomerId
GROUP BY c.Name;

-- Temp Table: Better when reused multiple times or needs indexing
CREATE TABLE #RecentOrders (
    CustomerId INT,
    OrderDate DATE,
    TotalAmount DECIMAL(18,2)
);

CREATE INDEX IX_TempOrders_CustomerId ON #RecentOrders(CustomerId);

INSERT INTO #RecentOrders
SELECT CustomerId, OrderDate, TotalAmount
FROM Orders
WHERE OrderDate >= DATEADD(DAY, -30, GETDATE());

-- Now use #RecentOrders multiple times with fast lookups
```

### SET NOCOUNT ON
```sql
-- GOOD: Reduces network traffic
CREATE PROCEDURE GetCustomerOrders
    @CustomerId INT
AS
BEGIN
    SET NOCOUNT ON; -- Prevents "X rows affected" messages
    
    SELECT OrderId, OrderDate, TotalAmount
    FROM Orders
    WHERE CustomerId = @CustomerId;
END
```

## 🔍 Query Analysis Tools

### Using Execution Plans
```sql
-- Show actual execution plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT o.OrderId, c.Name
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE o.OrderDate >= '2024-01-01';

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```

### Find Missing Indexes
```sql
-- Query to find missing indexes
SELECT 
    migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) AS improvement_measure,
    'CREATE INDEX IX_' + OBJECT_NAME(mid.object_id) + '_' + 
        REPLACE(REPLACE(REPLACE(ISNULL(mid.equality_columns,''),', ','_'),'[',''),']','') AS index_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) > 10
ORDER BY improvement_measure DESC;
```

### Find Unused Indexes
```sql
-- Find indexes that are never used
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats s 
    ON i.object_id = s.object_id AND i.index_id = s.index_id
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
    AND i.index_id > 0  -- Exclude heaps
    AND s.index_id IS NULL  -- Never used
ORDER BY OBJECT_NAME(i.object_id), i.name;
```

## ⚠️ Common Pitfalls

### 1. Over-Indexing
```sql
-- Having too many indexes slows down INSERT/UPDATE/DELETE
-- Only create indexes that are actually used

-- Before creating index, check impact
-- Monitor index usage with sys.dm_db_index_usage_stats
```

### 2. Parameter Sniffing
```sql
-- Can cause performance issues with stored procedures
-- Problem: Execution plan optimized for first parameter value

-- Solution 1: Use OPTION (RECOMPILE)
SELECT * FROM Orders 
WHERE CustomerId = @CustomerId
OPTION (RECOMPILE);

-- Solution 2: Use local variables
DECLARE @LocalCustomerId INT = @CustomerId;
SELECT * FROM Orders 
WHERE CustomerId = @LocalCustomerId;

-- Solution 3: Use OPTIMIZE FOR
SELECT * FROM Orders 
WHERE CustomerId = @CustomerId
OPTION (OPTIMIZE FOR (@CustomerId = 1));
```

### 3. Implicit Conversions
```sql
-- BAD: VARCHAR column compared with INT (causes conversion)
-- Prevents index usage and slow performance
SELECT * FROM Orders 
WHERE OrderNumber = 12345; -- OrderNumber is VARCHAR

-- GOOD: Match data types
SELECT * FROM Orders 
WHERE OrderNumber = '12345';
```

## 🎯 Best Practices

1. **Always use WHERE clauses**: Filter data as early as possible
2. **Select only needed columns**: Avoid SELECT *
3. **Use appropriate indexes**: Create indexes on columns used in WHERE, JOIN, and ORDER BY
4. **Avoid functions on indexed columns**: Prevents index usage
5. **Use EXISTS over IN**: For better performance with large datasets
6. **Avoid cursors**: Use set-based operations instead
7. **Update statistics regularly**: Helps query optimizer make better decisions
8. **Monitor query performance**: Use execution plans and DMVs
9. **Use stored procedures**: Pre-compiled and can have better execution plans
10. **Batch large operations**: Break large INSERT/UPDATE/DELETE into smaller batches

## 🎤 Common Interview Questions

### Q1: What's the difference between a clustered and non-clustered index?
**Answer**: 
- **Clustered Index**: Physically sorts the data in the table. Only one per table. The table data IS the index.
- **Non-Clustered Index**: Separate structure that points to the data. Can have multiple per table. Contains index key and pointer to data row.

### Q2: How do you identify slow queries in SQL Server?
**Answer**: Multiple approaches:
- Use SQL Server Profiler or Extended Events to trace queries
- Query `sys.dm_exec_query_stats` DMV for execution statistics
- Enable Query Store (SQL Server 2016+) for historical query performance
- Check execution plans for expensive operations (table scans, sorts)
- Monitor wait statistics with `sys.dm_os_wait_stats`

### Q3: What is parameter sniffing and how do you deal with it?
**Answer**: Parameter sniffing is when SQL Server creates an execution plan based on the first parameter value used. This can cause performance issues if that first value isn't representative. Solutions:
- Use `OPTION (RECOMPILE)` to recompile each time
- Use local variables to avoid sniffing
- Use `OPTION (OPTIMIZE FOR)` for specific values
- Clear plan cache if needed

### Q4: When should you use a CTE vs a Temp Table?
**Answer**:
- **CTE**: Simple queries, used once, recursive queries. Defined in query scope only.
- **Temp Table**: Reused multiple times, need indexes, large result sets, need statistics. Exists in tempdb until dropped or session ends.

### Q5: What's a covering index?
**Answer**: A non-clustered index that includes all columns needed for a query, so SQL Server can satisfy the query entirely from the index without accessing the table (index seek + lookup eliminated). Created with INCLUDE clause.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Covering 
ON Orders(CustomerId) 
INCLUDE (OrderDate, TotalAmount);
```

## 📚 Quick Reference Checklist

Before deploying a query, check:
- ✅ Are you selecting only needed columns?
- ✅ Do appropriate indexes exist?
- ✅ Are you filtering with WHERE as early as possible?
- ✅ Are you avoiding functions on indexed columns?
- ✅ Have you checked the execution plan?
- ✅ Are data types matching in comparisons?
- ✅ Is SET NOCOUNT ON in stored procedures?
- ✅ Are you using JOIN instead of subqueries where appropriate?

## 🔗 Related Topics
- [Indexing Strategies](./indexing.md)
- [Execution Plans](./execution-plans.md)
- [Stored Procedures vs Ad-hoc Queries](./stored-procedures.md)

---
**Last Reviewed**: [Add date]
**Confidence Level**: ⭐⭐⭐⭐⭐ (1-5 stars)
