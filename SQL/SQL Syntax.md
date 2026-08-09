
`HAVING` 用来：

> **过滤聚合（GROUP BY）后的结果。**

```
SELECT studio, SUM(budget) FROM movie WHERE budget > 100 GROUP BY studio HAVING SUM(budget) > 500;
```

### Totally Count

```
SELECT o.CustomerID, COUNT(DISTINCT d.ProductID) AS UniqueProducts
FROM  Order_Details d
JOIN Orders o ON o.OrderID = d.OrderID
GROUP BY o.CustomerID
```
### Aggregation pattern

```
SELECT   CASE
          WHEN Stock < 50 THEN 'Low Stock'
          WHEN Stock BETWEEN 50 AND 150 THEN 'Medium Stock'
          ELSE 'High Stock'
         END AS StockLevel,
         COUNT(*) AS ProductCount
FROM     Products
GROUP BY StockLevel;

```
不是它真的：
```
SELECT
↓
GROUP BY
```

而是 MySQL 在解析 SQL 时，把 alias 展开了。
相当于自动改写：
```
GROUP BY BucketLabel
```
变成：
```
GROUP BY
CASE
    WHEN salary < 5000 THEN 'Low'
    ELSE 'High'
END
```

另一种
```SQL
SELECT   StockLevel,
         COUNT(*) AS ProductCount
FROM
(
    SELECT 
        CASE
            WHEN Stock < 50 THEN 'Low Stock'
            WHEN Stock BETWEEN 50 AND 150 THEN 'Medium Stock'
            ELSE 'High Stock'
        END AS StockLevel
    FROM Products
) t
GROUP BY StockLevel;


SELECT CASE

WHEN DATEDIFF(CURDATE(), LastLogin) <= 7 THEN 'Active Last Week'

WHEN DATEDIFF(CURDATE(), LastLogin) <= 30 THEN 'Active Last Month'

ELSE 'Inactive'

END AS ActivityLevel,

COUNT(*) AS CustomerCount

FROM Customers

WHERE LastLogin IS NOT NULL

GROUP BY ActivityLevel;
```

# Rolling Totals
```SQL
AVG(Column) OVER (
    PARTITION BY
    ORDER BY
    ROWS BETWEEN N PRECEDING AND CURRENT ROW
)
```

**Combining with** **`GROUP BY`** **(Nested Aggregation):**

When we need to calculate rolling metrics on already grouped data (such as finding a rolling average of daily total sales), we combine standard aggregate functions with window functions. This calculation happens in two phases:

- **The inner function:** A standard aggregate like `SUM(Column)` calculates the initial total for each group (e.g., total sales per day).
    
- **The outer function:** The window function, like `SUM(...) OVER(...)` or `AVG(...) OVER(...)`, takes those grouped results and calculates the rolling metric across the sequence.
    

**Typical structure for rolling metrics on grouped data:**

```SQL
SUM(SUM(Column)) OVER (ORDER BY ...)
```

Accumulat Calculation
```SQL
SELECT   OrderDate,
         SUM(TotalAmount) AS DailySales,
         SUM(SUM(TotalAmount)) OVER (ORDER BY OrderDate) AS RunningTotal
FROM     Orders
WHERE    OrderDate BETWEEN '2025-03-01' AND '2025-03-10'
GROUP BY OrderDate
ORDER BY OrderDate;
```

```SQL
SELECT   OrderDate,
         SUM(TotalAmount) AS DailySales,
         AVG(SUM(TotalAmount)) OVER (
             ORDER BY OrderDate
             ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
             ) AS MovingAvg7Days
FROM     Orders
GROUP BY OrderDate
ORDER BY OrderDate;
```

# Patch Gaps

1. **Patch** **`NULLs using`COALESCE`or`CASE`
```SQL
SELECT COALESCE(<column>, <default_value>) AS filled_value
FROM   <table>;
```
`COALESCE` 是 SQL 中非常常用的函数 从左到右，返回第一个不是 `NULL` 的值。

```SQL
COALESCE(value1, value2, value3, ...)
```

Generate a new array using `SELECT OVER`
```SQl
WITH DateRange AS (
  SELECT DATE('2025-03-01') + INTERVAL (n.n) DAY AS ReportDate
  FROM   (
          SELECT ROW_NUMBER() OVER () - 1 AS n
          FROM   Orders
          LIMIT  15
         ) AS n
)
SELECT    d.ReportDate,
          COUNT(o.OrderID) AS OrdersPlaced
FROM      DateRange d
LEFT JOIN Orders o ON o.OrderDate = d.ReportDate
GROUP BY  d.ReportDate
ORDER BY  d.ReportDate;
```
2. **Generate missing rows using a date/calendar table or CTE**

```SQL
WITH DateRange AS (...)

SELECT    ...
FROM      DateRange
LEFT JOIN ActualData ...
```

```SQL
WITH CTE1 AS (...), CTE2 AS (...), CTE3 AS (...) SELECT ...
```

# Existence Check
```SQL
SELECT ...
FROM   MainTable m
WHERE  EXISTS (
        SELECT 1
        FROM RelatedTable r
        WHERE r.foreign_key = m.primary_key
       );

-- NOT EXISTS form
SELECT ...
FROM   MainTable m
WHERE  NOT EXISTS (
        SELECT 1
        FROM RelatedTable r
        WHERE r.foreign_key = m.primary_key
       );
```

**Alternatives:**
- `IN` / `NOT IN`
- `LEFT JOIN ... IS NULL` (for `NOT EXISTS`)

# Deduplication

1. `DISTINCT` to remove complete row duplicates.
    
2. `GROUP BY` to group duplicate rows by key and apply aggregation.

3. `ROW_NUMBER()` or `RANK()` window functions to keep a single preferred row per group.
```SQL
WITH RankedOrders AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY OrderDate DESC) AS rn
  FROM   Orders
)
SELECT *
FROM   RankedOrders
WHERE  rn = 1;
```
    
1. `EXISTS` or `NOT EXISTS` to exclude rows that are not the latest/first per group.
```SQL
SELECT o.*
FROM   Orders o
WHERE  NOT EXISTS (
        SELECT 1
        FROM   Orders o2
        WHERE  o.CustomerID = o2.CustomerID AND o2.OrderDate > o.OrderDate
       );
```

# Pattern Search

There are two main ways we structure pattern searches:

1. **Basic wildcard matching using** **`LIKE`** **and** **`ILIKE`****:**
    
    NaN. `%` for any number of characters.
        
    NaN. `_` for a single character.
    
```SQL
SELECT column1, column2, ...
FROM   table_name
WHERE  column_name LIKE 'pattern';
```

**Advanced matching using** **`REGEXP`** **or** **`RLIKE`****:**

NaN. Allows full regular expressions for complex searches
```SQL
SELECT column1, column2, ...
FROM   table_name
WHERE  column_name REGEXP 'regex_pattern';
```
捕获重复字符
```SQL
SELECT CustomerName
FROM Customers
WHERE CustomerName REGEXP '(.)\\1';
```