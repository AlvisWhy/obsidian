# Module 1. Spark SQL 与 DataFrame 的关系

## 1.1 What is Spark SQL?

Spark SQL 是 Apache Spark 中用于结构化数据处理的模块。

可以通过两种方式处理数据：

- DataFrame API：使用 Python / Scala 编写 Transformation。
    
- Spark SQL：使用 SQL 查询数据。
    

两者底层都使用 Spark SQL Engine，可以共享 Catalyst Optimizer 和 Physical Execution Engine。

例如，同一个操作：

DataFrame API

```
df_filtered = (    df.filter(col("order_id").isin(1001, 1002, 10001))      .select("order_id", "customer_id"))
```

Spark SQL

```
df.createOrReplaceTempView("orders")df_sql = spark.sql("""    SELECT order_id, customer_id    FROM orders    WHERE order_id IN (1001, 1002, 10001)""")
```

两种方式都返回 Spark DataFrame。

```
df_sql.show()df_sql.printSchema()
```

注意：

`spark.sql()` 返回的是 DataFrame，而不是 Pandas DataFrame。

## 1.2 Temporary View

Temporary View 是将 DataFrame 注册成一个临时 SQL View。

```
df.createOrReplaceTempView("csv_view")
```

查询：

```
df_sql = spark.sql("""    SELECT *    FROM csv_view    WHERE order_id IN (1001, 1002, 10001)""")
```

特点：

- 只在当前 SparkSession 中有效。
    
- 不会将数据复制成一份新的物理表。
    
- View 主要保存查询逻辑。
    
- SparkSession 结束后，View 消失。
    
- 可以通过 `createOrReplaceTempView()` 覆盖已有 View。
    

结构：

```
SparkSession
    |
    |-- DataFrame
    |       |
    |       v
    |   Temporary View
    |       |
    |       v
    |   spark.sql()
    |
    |-- Another DataFrame
```

## 1.3 Global Temporary View

Global Temporary View 可以在同一个 Spark Application 的不同 SparkSession 之间共享。

创建：

```
df.createOrReplaceGlobalTempView("global_view")
```

查询：

```
df_sql = spark.sql("""    SELECT *    FROM global_temp.global_view""")
```

注意：

- 不同 SparkSession 可以访问。
    
- 必须使用 `global_temp` 这个默认 Database 前缀。
    
- 生命周期是 Spark Application 级别。
    
- Spark Application 结束后消失。
    
- 不是跨集群共享，也不是持久化表。
    

## 1.4 三种 View 的区别

|Feature|Temporary View|Global Temporary View|Permanent View|
|---|---|---|---|
|Scope|SparkSession|Spark Application|Catalog|
|Other Session Access|No|Yes|Yes，取决于权限|
|Persistence|No|No|Yes，保存 View 定义|
|Physical Data Copy|No|No|No|
|Lifetime|Session|Application|Until dropped|

创建 Permanent View：

```
CREATE OR REPLACE VIEW catalog.schema.orders_view AS
SELECT *
FROM catalog.schema.orders;
```

删除：

```
DROP VIEW IF EXISTS catalog.schema.orders_view;
```

这里需要特别注意：Permanent View 只是保存 SQL 查询定义，不会把底层查询结果自动物化成数据。

## 1.5 Interview Questions

Q1. What is the difference between a DataFrame and a Temporary View?

DataFrame 是 Spark 的分布式数据抽象，Temporary View 是注册在 Spark Catalog 中的 SQL 查询入口。两者可以引用同一份底层数据。

Q2. What is the difference between Temporary View and Global Temporary View?

Temporary View 属于当前 SparkSession，Global Temporary View 属于当前 Spark Application，可以被不同 SparkSession 访问。

Q3. Does creating a Temporary View trigger Spark execution?

不会。创建 View 本身通常不会触发数据计算，因为 Spark 使用 Lazy Evaluation。

Q4. Is a Spark View a materialized view?

不是。普通 View 保存查询逻辑，查询时执行。Materialized View 会持久化查询结果。

Q5. Does `spark.sql()` return a DataFrame?

Yes. It returns a Spark DataFrame that can be further transformed using DataFrame APIs.

# Module 2. Spark SQL DDL — Data Definition Language

DDL 用于创建、修改和删除数据库对象。

主要包括：

- CREATE
    
- ALTER
    
- DROP
    
- TRUNCATE
    

## 2.1 CREATE DATABASE / SCHEMA

```
CREATE DATABASE IF NOT EXISTS sparkcatalog.raw;
```

也可以：

```
CREATE SCHEMA IF NOT EXISTS sparkcatalog.raw;
```

在 Spark SQL 中，DATABASE 和 SCHEMA 通常是同义概念。

查看：

```
SHOW DATABASES;
```

## 2.2 CREATE TABLE

创建表：

```
CREATE TABLE IF NOT EXISTS sparkcatalog.raw.new_table (
    id INT,
    name STRING,
    age INT
)
USING DELTA;
```

这里：

- `sparkcatalog`：Catalog。
    
- `raw`：Schema / Database。
    
- `new_table`：Table。
    
- `USING DELTA`：指定数据格式。
    

也可以指定 Parquet：

```
CREATE TABLE sparkcatalog.raw.new_table (
    id INT,
    name STRING,
    age INT
)
USING PARQUET;
```

## 2.3 Managed Table vs External Table

这是 Spark SQL 和 Databricks 面试很容易问到的知识点。

Managed Table

```
CREATE TABLE sparkcatalog.raw.orders (
    order_id INT,
    amount DOUBLE
)
USING DELTA;
```

数据存储位置由 Catalog / Metastore 管理。

删除 Managed Table 时，通常会同时删除表数据。

External Table

```
CREATE TABLE sparkcatalog.raw.orders_external (
    order_id INT,
    amount DOUBLE
)
USING DELTA
LOCATION 's3://my-bucket/orders/';
```

数据存储位置由用户指定。

删除 External Table 时，通常只删除 Catalog 中的表定义，不会删除底层数据文件。

注意：具体行为还需要结合 Catalog 实现和存储管理方式理解，特别是 Databricks Unity Catalog 的 Managed / External Table。

## 2.4 INSERT INTO

插入数据：

```
INSERT INTO sparkcatalog.raw.new_table
VALUES
    (1, 'John', 25),
    (2, 'Alex', 24);
```

指定列：

```
INSERT INTO sparkcatalog.raw.new_table (
    id,
    name,
    age
)
VALUES
    (3, 'David', 28);
```

从另一个表插入：

```
INSERT INTO sparkcatalog.raw.new_table
SELECT
    id,
    name,
    age
FROM sparkcatalog.raw.source_table;
```

## 2.5 CREATE TABLE AS SELECT (CTAS)

CTAS 可以直接通过查询结果创建新表。

```
CREATE TABLE sparkcatalog.raw.high_value_orders
USING DELTA
AS
SELECT *
FROM sparkcatalog.raw.orders
WHERE amount > 1000;
```

它在数据工程中非常常用。

## 2.6 ALTER TABLE

增加字段：

```
ALTER TABLE sparkcatalog.raw.orders
ADD COLUMNS (
    country STRING
);
```

修改表名：

```
ALTER TABLE sparkcatalog.raw.orders
RENAME TO sparkcatalog.raw.orders_new;
```

删除表：

```
DROP TABLE IF EXISTS sparkcatalog.raw.orders;
```

清空表数据但保留表结构：

```
TRUNCATE TABLE sparkcatalog.raw.orders;
```

## 2.7 Interview Questions

Q1. What is the difference between DDL and DML?

DDL 用于定义和管理数据库对象，DML 用于操作数据。

Q2. What is the difference between Managed and External Tables?

重点解释数据位置的管理权，以及 DROP TABLE 后数据文件的处理差异。

Q3. What is CTAS?

Create Table As Select，利用 SQL 查询结果创建一张新表。

Q4. What is the difference between DROP and TRUNCATE?

DROP 删除表对象，TRUNCATE 删除表中的数据但保留表结构。

Q5. What is the difference between Delta and Parquet tables?

Parquet 是文件格式，Delta 在 Parquet 数据文件之上增加 Transaction Log 和事务管理能力。

# Module 3. Spark SQL DML — Data Manipulation Language

DML 用于插入、更新、删除和合并数据。

主要包括：

- INSERT
    
- UPDATE
    
- DELETE
    
- MERGE
    

## 3.1 INSERT

```
INSERT INTO orders
VALUES (1001, 2001, 500.0);
```

## 3.2 UPDATE

```
UPDATE orders
SET amount = 600
WHERE order_id = 1001;
```

注意：UPDATE 需要底层表格式和执行引擎支持，例如 Delta Lake。

## 3.3 DELETE

```
DELETE FROM orders
WHERE order_id = 1001;
```

同样需要支持 DELETE 的表格式，例如 Delta。

## 3.4 MERGE / UPSERT

你原来的笔记中 `MARGE` 和 `INSESRT` 是拼写错误。

正确写法：

```
MERGE INTO target_table AS t
USING source_table AS s
ON t.id = s.id

WHEN MATCHED THEN
    UPDATE SET *

WHEN NOT MATCHED THEN
    INSERT *;
```

逻辑：

```
Source Table
     |
     v
Compare with Target
     |
     |-------------------|
     |                   |
  Matched             Not Matched
     |                   |
   UPDATE               INSERT
```

也可以指定更新字段：

```
MERGE INTO customers AS t
USING updates AS s
ON t.customer_id = s.customer_id

WHEN MATCHED THEN
    UPDATE SET
        t.name = s.name,
        t.age = s.age

WHEN NOT MATCHED THEN
    INSERT (
        customer_id,
        name,
        age
    )
    VALUES (
        s.customer_id,
        s.name,
        s.age
    );
```

## 3.5 WHEN NOT MATCHED BY SOURCE

```
MERGE INTO customers AS t
USING source_customers AS s
ON t.customer_id = s.customer_id

WHEN MATCHED THEN
    UPDATE SET *

WHEN NOT MATCHED THEN
    INSERT *

WHEN NOT MATCHED BY SOURCE THEN
    DELETE;
```

适用于需要将 Target 同步为完整 Source Snapshot 的场景。

注意：如果 Source 只是增量数据，不能直接使用 `WHEN NOT MATCHED BY SOURCE THEN DELETE`，否则可能删除 Target 中大量仍然有效的数据。

## 3.6 Interview Questions

Q1. What is the difference between INSERT and MERGE?

INSERT 主要添加数据，MERGE 根据匹配条件决定 UPDATE、INSERT 或 DELETE。

Q2. What is UPSERT?

根据业务主键判断记录是否存在，存在则更新，不存在则插入。

Q3. Can you use UPDATE and DELETE on Parquet tables?

普通 Parquet 文件表通常不支持原地 UPDATE / DELETE，需要支持这些操作的表格式或其他数据处理方式。

Q4. What happens if the source contains duplicate keys during MERGE?

可能导致多个 Source Records 匹配同一个 Target Record，造成更新冲突或不确定结果，因此需要先去重。

# Module 4. PARTITION BY — Table Partitioning

你原来的笔记只记录了 `PARTITION BY()`，这里需要区分两个完全不同的概念：

1. Table Partitioning：物理数据存储分区。
    
2. Window Function PARTITION BY：窗口计算分组。
    

## 4.1 Table Partitioning

Partitioning 是将数据按照指定字段拆分成不同的存储目录。

例如订单表按照日期分区：

```
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    amount DOUBLE
)
USING DELTA
PARTITIONED BY (order_date);
```

数据目录可能如下：

```
orders/
    order_date=2026-01-01/
        part-001.parquet
        part-002.parquet

    order_date=2026-01-02/
        part-003.parquet

    order_date=2026-01-03/
        part-004.parquet

    _delta_log/
```

查询：

```
SELECT *
FROM orders
WHERE order_date = '2026-01-02';
```

Spark 可以利用 Partition Pruning，只读取相关日期分区的数据。

## 4.2 Partition Pruning

假设表有 365 天的数据。

不使用分区过滤：

```
SELECT *
FROM orders;
```

可能需要读取所有分区。

使用分区过滤：

```
SELECT *
FROM orders
WHERE order_date BETWEEN '2026-01-01'
                     AND '2026-01-07';
```

只需要读取符合条件的日期分区。

注意：

- Partitioning 不是越多越好。
    
- 不建议使用 Cardinality 非常高的字段作为分区列，例如 UUID、Transaction ID。
    
- 分区太多容易产生 Small Files。
    
- 需要结合数据规模、查询模式和文件大小决定分区策略。
    

## 4.3 Dynamic Partition Overwrite

Spark SQL 还支持动态分区写入，具体行为取决于写入模式和配置。

例如：

```
INSERT OVERWRITE TABLE orders
PARTITION (order_date)
SELECT
    order_id,
    customer_id,
    amount,
    order_date
FROM new_orders;
```

动态分区覆盖只覆盖本次写入涉及的分区，具体还要注意 `spark.sql.sources.partitionOverwriteMode` 的配置及表格式支持情况。

在 Delta Lake 中，通常还可以使用：

```
df.write \    .format("delta") \    .mode("overwrite") \    .option(        "replaceWhere",        "order_date = '2026-01-02'"    ) \    .save(path)
```

## 4.4 Window PARTITION BY

这是另一种用法：

```
SELECT
    customer_id,
    order_id,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
    ) AS customer_total
FROM orders;
```

它按照 `customer_id` 分组计算，但不会把原始记录合并成一行，也不会直接创建物理存储目录。

## 4.5 Interview Questions

Q1. What is partition pruning?

Spark 根据查询条件跳过不相关的物理数据分区，减少数据读取。

Q2. What factors should you consider when choosing partition columns?

查询频率、Cardinality、数据量、数据分布和文件大小。

Q3. What is the difference between table partitioning and window partitioning?

一个影响物理数据存储和读取，一个用于窗口函数计算。

Q4. What happens if you partition a table by a high-cardinality column?

可能产生大量小文件和目录，增加 Metadata 管理和文件读取开销。

# Module 5. CTE、Subquery 和 CASE WHEN

## 5.1 Common Table Expression (CTE)

CTE 使用 `WITH` 定义临时查询结果，方便拆分复杂 SQL。

正确写法：

```
WITH inner_query AS (
    SELECT
        *,
        CASE
            WHEN id % 2 = 0 THEN 'even'
            ELSE 'odd'
        END AS even_odd_flag
    FROM orders
)

SELECT *
FROM inner_query;
```

多个 CTE：

```
WITH order_summary AS (
    SELECT
        customer_id,
        SUM(amount) AS total_amount
    FROM orders
    GROUP BY customer_id
),

high_value_customers AS (
    SELECT *
    FROM order_summary
    WHERE total_amount > 1000
)

SELECT *
FROM high_value_customers;
```

优点：

- 提高 SQL 可读性。
    
- 将复杂逻辑拆分为多个步骤。
    
- 避免重复书写相同的子查询。
    
- 方便维护和调试。
    

注意：CTE 不一定会物化成物理临时表，Spark 优化器可以对其进行优化和展开。

## 5.2 Subquery

Subquery 是嵌套在其他 SQL 查询中的查询。

Scalar Subquery

返回一个值：

```
SELECT *
FROM orders
WHERE amount > (
    SELECT AVG(amount)
    FROM orders
);
```

IN Subquery

```
SELECT *
FROM orders
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE country = 'NL'
);
```

EXISTS Subquery

```
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

NOT EXISTS

```
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

EXISTS 通常用于检查是否存在满足条件的记录，不需要返回具体字段。

## 5.3 CASE WHEN

```
SELECT
    order_id,
    amount,
    CASE
        WHEN amount > 1000 THEN 'high'
        WHEN amount > 500 THEN 'medium'
        WHEN amount > 0 THEN 'low'
        ELSE 'invalid'
    END AS amount_category
FROM orders;
```

注意 CASE WHEN 是按照顺序判断的，满足第一个条件后就返回结果。

也可以使用：

```
CASE
    WHEN condition THEN result
    ELSE default_result
END
```

## 5.4 Interview Questions

Q1. What is a CTE and why do you use it?

使用 WITH 将复杂查询拆分为可读、可维护的逻辑块。

Q2. What is the difference between CTE and Temporary View?

CTE 的作用范围是单条 SQL 查询；Temporary View 在当前 SparkSession 中可以被多个 SQL 查询复用。

Q3. Does a CTE always materialize data?

No. CTE 通常是逻辑查询结构，不保证物理物化。

Q4. What is the difference between EXISTS and IN?

两者都可以用于判断匹配关系，但语义不同；EXISTS 检查是否存在匹配行，IN 检查值是否属于结果集合。还需要特别注意 NULL 的语义差异。

# Module 6. Spark SQL Scripting Statements

这一部分比普通 SQL 更进一步，允许在 SQL 中编写控制流程。

包括：

- DECLARE
    
- SET
    
- IF / ELSEIF / ELSE
    
- FOR
    
- WHILE
    
- LEAVE
    
- ITERATE
    

注意：SQL Scripting 并不是所有 Apache Spark 版本或 SQL 执行环境都支持。实际使用前需要确认 Spark / Databricks Runtime 版本及执行环境。

## 6.1 Variables

定义变量：

```
BEGIN
    DECLARE total INT DEFAULT 0;

    SET total = 100;

    SELECT total;
END;
```

变量用于在 SQL 脚本中保存临时结果。

## 6.2 IF / ELSEIF / ELSE

你的原始代码中 `DECLARE` 拼写错误，而且 `VALUES()` 不应该作为变量赋值的替代方式。

修正：

```
BEGIN
    DECLARE choice DOUBLE DEFAULT 2.9;
    DECLARE result STRING;

    IF choice < 2 THEN
        SET result = '111';

    ELSEIF choice < 3 THEN
        SET result = '222';

    ELSE
        SET result = 'NO';

    END IF;

    SELECT result;
END;
```

逻辑：

```
choice
   |
   v
choice < 2 ?
   |
   | Yes --> result = '111'
   |
   No
   |
   v
choice < 3 ?
   |
   | Yes --> result = '222'
   |
   No
   |
   v
result = 'NO'
```

## 6.3 FOR Loop

使用 FOR 遍历查询结果：

```
BEGIN
    DECLARE total INT DEFAULT 0;

    sumNumbers: FOR row AS
        SELECT id
        FROM range(1, 20) AS t(id)

    DO
        IF row.id > 10 THEN
            LEAVE sumNumbers;

        ELSEIF row.id % 2 = 0 THEN
            ITERATE sumNumbers;
        END IF;

        SET total = total + row.id;

    END FOR sumNumbers;

    SELECT total;
END;
```

这里：

- `FOR`：遍历查询返回的记录。
    
- `LEAVE`：退出循环。
    
- `ITERATE`：跳过当前循环，进入下一次循环。
    
- `SET`：修改变量。
    

这个例子只累加 1 到 9 之间的奇数。

结果：

```
25
```

## 6.4 WHILE Loop

```
BEGIN
    DECLARE counter INT DEFAULT 1;

    WHILE counter <= 5 DO

        SELECT counter;

        SET counter = counter + 1;

    END WHILE;
END;
```

## 6.5 LEAVE vs ITERATE

|Statement|Meaning|
|---|---|
|LEAVE|直接退出循环|
|ITERATE|跳过当前迭代，进入下一次循环|

类似 Python：

```
breakcontinue
```

## 6.6 Interview Questions

Q1. What is SQL Scripting?

允许在 SQL 中使用变量、条件判断和循环等控制流程。

Q2. What is the difference between LEAVE and ITERATE?

LEAVE 退出循环，ITERATE 跳过当前循环。

Q3. When would you use SQL Scripting instead of PySpark?

适合简单的数据库端控制逻辑、条件执行和 SQL 管理任务。复杂的数据处理、分布式 ETL 通常更适合 PySpark。

# Module 7. Auxiliary Statements — Metadata & Table Inspection

这一部分主要用于查看 Catalog、Schema、Table 和执行环境信息。

## 7.1 DESCRIBE DATABASE

```
DESCRIBE DATABASE sparkcatalog.raw;
```

查看详细信息：

```
DESCRIBE DATABASE EXTENDED sparkcatalog.raw;
```

可以查看：

- Database / Schema Name。
    
- Location。
    
- Description。
    
- Owner 等 Metadata。
    

## 7.2 DESCRIBE TABLE

```
DESCRIBE TABLE sparkcatalog.raw.orders;
```

查看完整信息：

```
DESCRIBE TABLE EXTENDED sparkcatalog.raw.orders;
```

查看列信息：

```
DESCRIBE TABLE EXTENDED
sparkcatalog.raw.orders;
```

对于 Delta Table，也可以：

```
DESCRIBE DETAIL sparkcatalog.raw.orders;
```

它可以提供：

- Format。
    
- Location。
    
- Number of Files。
    
- Size in Bytes。
    
- Partition Columns。
    
- Table Properties。
    

## 7.3 DESCRIBE QUERY vs EXPLAIN

你的笔记记录了：

```
DESCRIBE QUERY
SELECT * FROM orders;
```

不过，日常分析 Spark SQL 查询计划时，更推荐使用：

```
EXPLAIN
SELECT *
FROM orders
WHERE amount > 100;
```

查看详细执行计划：

```
EXPLAIN FORMATTED
SELECT *
FROM orders
WHERE amount > 100;
```

常见信息：

- Parsed Logical Plan。
    
- Analyzed Logical Plan。
    
- Optimized Logical Plan。
    
- Physical Plan。
    

`DESCRIBE QUERY` 的可用性和具体语法取决于 SQL 引擎版本，不要与 `EXPLAIN` 混为一谈。

## 7.4 SHOW Statements

查看 Catalog：

```
SHOW CATALOGS;
```

查看 Database / Schema：

```
SHOW DATABASES;
SHOW SCHEMAS IN sparkcatalog;
```

查看 Tables：

```
SHOW TABLES IN sparkcatalog.raw;
```

查看 Table Properties：

```
SHOW TBLPROPERTIES sparkcatalog.raw.orders;
```

查看分区：

```
SHOW PARTITIONS sparkcatalog.raw.orders;
```

注意：`SHOW PARTITIONS` 主要适用于支持物理分区管理的表，不是所有表格式都支持。

## 7.5 REFRESH TABLE

```
REFRESH TABLE sparkcatalog.raw.orders;
```

作用：

刷新表的 Metadata 和相关缓存信息，使 Spark 重新获取最新的表和文件状态。

典型场景：

外部程序修改了文件系统中的数据文件，Spark 需要刷新之前缓存的表信息。

注意：

`REFRESH TABLE` 不等于重新加载或重建整个 Delta Transaction Log，也不等于执行 OPTIMIZE。

## 7.6 Interview Questions

Q1. How do you inspect a table's schema and location?

使用 `DESCRIBE TABLE`、`DESCRIBE DETAIL`。

Q2. What is the purpose of REFRESH TABLE?

刷新表的 Metadata 和缓存信息。

Q3. How do you inspect the Spark execution plan?

使用 `EXPLAIN FORMATTED`。

Q4. What is the difference between DESCRIBE TABLE and DESCRIBE DETAIL?

前者主要查看 Schema 和表信息，后者可以查看 Delta Table 的详细存储和文件统计信息。

# Module 8. Spark SQL Aggregate Functions

Aggregate Functions 用于对多条记录进行聚合计算。

常见函数：

|Function|Description|
|---|---|
|COUNT|统计记录数量|
|SUM|求和|
|AVG|平均值|
|MIN|最小值|
|MAX|最大值|
|COUNT(DISTINCT)|精确去重计数|
|APPROX_COUNT_DISTINCT|近似去重计数|
|COLLECT_LIST|收集成 Array，保留重复值|
|COLLECT_SET|收集成 Array，去除重复值|
|ARRAY_AGG|收集成 Array|
|CORR|计算相关系数|

## 8.1 ARRAY_AGG

将多行数据聚合成一个 Array。

```
SELECT
    order_id,
    ARRAY_AGG(customer_id) AS customers
FROM orders
GROUP BY order_id;
```

例如：

原始数据：

|order_id|customer_id|
|---|---|
|1|A|
|1|B|
|1|C|
|2|A|

结果：

|order_id|customers|
|---|---|
|1|[A,B,C]|
|2|[A]|

与 PySpark DataFrame API 对应：

```
df.groupBy("order_id").agg(    array_agg("customer_id").alias("customers"))
```

也可以使用：

```
SELECT
    order_id,
    COLLECT_LIST(customer_id)
FROM orders
GROUP BY order_id;
```

注意：

- `COLLECT_LIST` 保留重复值。
    
- `COLLECT_SET` 去除重复值。
    
- `ARRAY_AGG` 可以作为收集数组的聚合函数。
    
- 不指定排序时，不应假设 Array 中元素具有固定顺序。
    

## 8.2 CORR

计算两个数值字段之间的 Pearson Correlation Coefficient。

```
SELECT
    CORR(price, quantity) AS correlation
FROM orders;
```

结果范围：

\[ -1 \leq r \leq 1 \]

|Correlation|Meaning|
|---|---|
|+1|完全正相关|
|0|线性相关性为零|
|-1|完全负相关|

注意：你原来写的 `corr(price, customer)` 如果 customer 是字符串类型，就不适合作为相关系数的输入。应该选择两个数值字段。

## 8.3 COUNT DISTINCT vs APPROX_COUNT_DISTINCT

精确去重：

```
SELECT
    COUNT(DISTINCT customer_id)
FROM orders;
```

近似去重：

```
SELECT
    APPROX_COUNT_DISTINCT(customer_id)
FROM orders;
```

区别：

|Feature|COUNT DISTINCT|APPROX_COUNT_DISTINCT|
|---|---|---|
|Accuracy|Exact|Approximate|
|Performance|可能开销较高|通常更快|
|Memory|可能需要更多资源|通常更节省|
|Usage|财务、精确统计|大规模用户统计|

近似计数通常基于 HyperLogLog++ 算法。

还可以设置相对标准误差：

```
SELECT
    APPROX_COUNT_DISTINCT(
        customer_id,
        0.05
    )
FROM orders;
```

`0.05` 表示目标相对标准误差约为 5%，不是保证每次结果都落在该误差范围内。

## 8.4 Interview Questions

Q1. What is the difference between COLLECT_LIST and COLLECT_SET?

前者保留重复值，后者去除重复值。

Q2. What is the difference between COUNT DISTINCT and APPROX_COUNT_DISTINCT?

一个精确计数，一个使用概率算法进行近似计数。

Q3. When would you use APPROX_COUNT_DISTINCT?

大规模数据分析，例如每日活跃用户数、不同客户数统计。

Q4. What does CORR calculate?

两个数值字段之间的 Pearson 线性相关系数。

# Module 9. Complex Data Types — STRUCT, MAP

这是 Spark SQL 处理 Nested JSON 数据时非常重要的知识点。

Spark SQL 常见复杂数据类型：

- ARRAY：数组。
    
- STRUCT：结构体。
    
- MAP：键值映射。
    

## 9.1 STRUCT

STRUCT 可以将多个字段组合成一个结构化对象。

```
SELECT STRUCT(
    1,
    2,
    3,
    'abc'
) AS my_struct;
```

可以给字段命名：

```
SELECT NAMED_STRUCT(
    'id', 1,
    'name', 'John',
    'age', 25
) AS customer;
```

结果逻辑：

```
customer
    |
    |-- id: 1
    |-- name: John
    |-- age: 25
```

访问字段：

```
SELECT customer.name
FROM customers;
```

也可以：

```
SELECT customer['name']
FROM customers;
```

## 9.2 MAP

MAP 是 Key-Value 数据结构。

创建：

```
SELECT MAP(
    'a', 1,
    'b', 2,
    'c', 3
) AS my_map;
```

结果：

```
{
    "a": 1,
    "b": 2,
    "c": 3
}
```

访问 Value：

```
SELECT my_map['a']
FROM orders;
```

返回：

```
1
```

提取所有 Keys：

```
SELECT MAP_KEYS(
    MAP('a', 1, 'b', 2, 'c', 3)
);
```

结果：

```
['a', 'b', 'c']
```

提取所有 Values：

```
SELECT MAP_VALUES(
    MAP('a', 1, 'b', 2, 'c', 3)
);
```

结果：

```
[1, 2, 3]
```

## 9.3 STRUCT vs MAP

|Feature|STRUCT|MAP|
|---|---|---|
|Field Names|固定|动态 Key|
|Value Types|可以不同|所有 Value 使用相同 Value Type|
|Access|`struct.field`|`map['key']`|
|Use Case|固定 JSON Schema|动态 Key-Value|

例如：

STRUCT：

```
{
  "customer_id": 1001,
  "name": "John",
  "age": 25
}
```

MAP：

```
{
  "attribute_1": "A",
  "attribute_2": "B",
  "attribute_3": "C"
}
```

## 9.4 Nested JSON Example

假设 JSON：

```
{
  "order_id": 1001,
  "customer": {
    "name": "John",
    "age": 25
  },
  "products": [
    "Laptop",
    "Mouse"
  ]
}
```

Spark SQL：

```
SELECT
    order_id,
    customer.name,
    customer.age,
    products[0] AS first_product
FROM orders;
```

注意：Spark SQL Array 索引从 0 开始，而 `element_at()` 通常使用从 1 开始的正数索引。

## 9.5 Interview Questions

Q1. What is the difference between STRUCT and MAP?

STRUCT 是固定字段结构，MAP 是动态 Key-Value 结构。

Q2. How do you query nested JSON fields in Spark SQL?

使用点号访问 STRUCT，使用方括号访问 MAP 和 ARRAY。

Q3. Why would you use STRUCT instead of storing nested data as JSON strings?

STRUCT 提供明确的数据类型和 Schema，支持直接查询和 Catalyst 优化，避免每次查询都解析 JSON 字符串。

Q4. How do you flatten nested JSON data?

使用字段访问、`EXPLODE`、`INLINE` 等函数。

# Module 10. Date & Time Functions

日期时间处理是 ETL 中非常常见的任务。

## 10.1 Current Date & Timestamp

```
SELECT CURRENT_DATE();

SELECT CURRENT_TIMESTAMP();
```

区别：

|Function|Return Type|Description|
|---|---|---|
|CURRENT_DATE|DATE|当前日期|
|CURRENT_TIMESTAMP|TIMESTAMP|当前时间戳|

## 10.2 Add / Subtract Date

增加月份：

```
SELECT ADD_MONTHS(
    CURRENT_DATE(),
    1
);
```

减少月份：

```
SELECT ADD_MONTHS(
    CURRENT_DATE(),
    -1
);
```

增加天数：

```
SELECT DATE_ADD(
    CURRENT_DATE(),
    7
);
```

减少天数：

```
SELECT DATE_SUB(
    CURRENT_DATE(),
    7
);
```

注意：`ADD_MONTHS` 与 `DATE_ADD` 的单位不同。

## 10.3 Date Difference

```
SELECT DATEDIFF(
    '2026-11-08',
    '2026-11-01'
);
```

结果：

```
7
```

`DATEDIFF(endDate, startDate)` 返回两个日期之间相差的天数。

## 10.4 Date Formatting

```
SELECT DATE_FORMAT(
    CURRENT_TIMESTAMP(),
    'yyyyMMdd'
);
```

结果类似：

```
20261108
```

注意：

- `yyyy`：四位年份。
    
- `MM`：月份。
    
- `dd`：日期。
    
- `HH`：24 小时制小时。
    
- `mm`：分钟。
    
- `ss`：秒。
    

不要混淆 `MM` 和 `mm`。

## 10.5 EXTRACT

提取年份：

```
SELECT EXTRACT(
    YEAR FROM CURRENT_TIMESTAMP()
);
```

提取月份：

```
SELECT EXTRACT(
    MONTH FROM CURRENT_TIMESTAMP()
);
```

提取一年中的第几天：

```
SELECT DAYOFYEAR(
    CURRENT_TIMESTAMP()
);
```

也可以：

```
SELECT EXTRACT(
    DOY FROM CURRENT_TIMESTAMP()
);
```

## 10.6 UNIX_TIMESTAMP & FROM_UNIXTIME

UNIX_TIMESTAMP：

```
SELECT UNIX_TIMESTAMP();
```

将时间转换为 Unix Timestamp，通常以秒为单位。

FROM_UNIXTIME：

```
SELECT FROM_UNIXTIME(0);
```

将 Unix Timestamp 转换为日期时间字符串。

Unix Epoch：

```
1970-01-01 00:00:00 UTC
```

注意：显示的时区取决于 Spark Session 的时区设置。

## 10.7 Timezone Conversion

你的原始笔记：

```
SELECT CONVERT_TIMEZONE(
    "America/Los_Angeles",
    CURRENT_TIMESTAMP()
);
```

这个函数的参数需要根据源时间类型和目标时区来理解。较新的 Spark SQL / Databricks 环境可以使用：

```
SELECT CONVERT_TIMEZONE(
    'UTC',
    'Europe/Amsterdam',
    TIMESTAMP_NTZ '2026-11-08 12:00:00'
);
```

表示将 UTC 的本地时间表示转换为 Amsterdam 的本地时间表示。

另外：

```
SET TIME ZONE 'Europe/Amsterdam';
```

可以设置当前 Session 的时区。

特别注意：

- `TIMESTAMP` 通常表示时间线上的一个时间点。
    
- `TIMESTAMP_NTZ` 表示不带时区语义的本地日期时间。
    
- 时区转换要先确认源时间的时区语义。
    
- 夏令时 DST 可能导致某些本地时间不存在或出现歧义。
    

## 10.8 Interview Questions

Q1. What is the difference between DATE and TIMESTAMP?

DATE 只表示日期，TIMESTAMP 还包含时间信息。

Q2. What is the difference between ADD_MONTHS and DATE_ADD?

前者以月份为单位，后者以天为单位。

Q3. What is Unix Timestamp?

从 1970-01-01 UTC 开始计算的时间秒数。

Q4. How do you handle different time zones in a data pipeline?

统一存储和处理时间点，通常采用 UTC，并在展示层转换为业务时区，同时正确处理 DST。

Q5. What is the difference between TIMESTAMP and TIMESTAMP_NTZ?

一个具有时间线 / 时区相关语义，另一个表示不带时区的本地日期时间。

# Module 11. Window Functions — LAG, LEAD, NTILE

你之前的 PySpark 笔记已经涉及 ROW_NUMBER、RANK、DENSE_RANK 和累计求和。这里补充 SQL 写法。

## 11.1 LAG

LAG 用来读取当前行之前的记录。

```
SELECT
    order_id,
    order_date,
    LAG(order_date, 1) OVER (
        ORDER BY order_date
    ) AS previous_order_date
FROM orders;
```

参数：

```
LAG(column, offset, default_value)
```

例如：

```
LAG(amount, 1, 0)
```

表示读取上一行的 amount，如果没有上一行，则返回 0。

注意：Default Value 必须与目标字段类型兼容。你原来的 `LAG(order_date, 1, 'NA')` 如果 order_date 是 DATE 类型，就不合适。

## 11.2 LEAD

LEAD 用来读取当前行之后的记录。

```
SELECT
    order_id,
    order_date,
    LEAD(order_date, 1) OVER (
        ORDER BY order_date
    ) AS next_order_date
FROM orders;
```

## 11.3 LAG Use Case — Calculate Previous Order Gap

```
WITH order_history AS (
    SELECT
        customer_id,
        order_date,
        LAG(order_date) OVER (
            PARTITION BY customer_id
            ORDER BY order_date
        ) AS previous_order_date
    FROM orders
)

SELECT
    customer_id,
    order_date,
    previous_order_date,
    DATEDIFF(
        order_date,
        previous_order_date
    ) AS days_between_orders
FROM order_history;
```

这个场景非常适合 Data Engineer 面试。

## 11.4 NTILE

NTILE 将记录按照排序结果划分成 N 个尽可能均匀的组。

```
SELECT
    customer_id,
    amount,
    NTILE(3) OVER (
        ORDER BY amount DESC
    ) AS customer_group
FROM orders;
```

例如：

```
Group 1: Top 1/3
Group 2: Middle 1/3
Group 3: Bottom 1/3
```

如果每组无法平均分配，前面的组会优先获得额外记录。

你的原始代码：

```
NTILE(3) OVER (
    PARTITION BY order_status
)
```

还缺少 `ORDER BY`。如果目的是按金额划分每个状态下的客户层级，应写：

```
NTILE(3) OVER (
    PARTITION BY order_status
    ORDER BY amount DESC
)
```

## 11.5 Window Functions Summary

|Function|Description|
|---|---|
|ROW_NUMBER|每行分配唯一序号|
|RANK|相同值排名相同，后续排名跳号|
|DENSE_RANK|相同值排名相同，后续排名不跳号|
|LAG|读取之前的行|
|LEAD|读取之后的行|
|NTILE|将记录分成 N 个组|
|FIRST_VALUE|获取窗口中的第一个值|
|LAST_VALUE|获取窗口中的最后一个值|
|SUM OVER|窗口累计求和或窗口求和|
|AVG OVER|窗口平均值|

## 11.6 Interview Questions

Q1. What is the difference between LAG and LEAD?

LAG 读取前面的行，LEAD 读取后面的行。

Q2. What is the difference between ROW_NUMBER, RANK and DENSE_RANK?

- ROW_NUMBER：每行唯一编号。
    
- RANK：允许并列，后续排名跳号。
    
- DENSE_RANK：允许并列，后续排名不跳号。
    

Q3. What is NTILE used for?

将数据分成 N 个近似均匀的组，例如客户分层、收入分位数分析。

Q4. How would you find each customer's previous transaction date?

使用 `LAG()`，按 customer_id 分区，并按照 transaction_date 排序。

# Module 12. Array Functions

Array 是 Spark SQL 处理 JSON、Nested Data 和复杂数据结构的重要类型。

## 12.1 Create Array

```
SELECT ARRAY(1, 2, 3);
```

## 12.2 ARRAY_APPEND

在 Array 尾部增加元素：

```
SELECT ARRAY_APPEND(
    ARRAY(1, 2, 3),
    9
);
```

结果：

```
[1, 2, 3, 9]
```

## 12.3 ARRAY_DISTINCT

删除重复元素：

```
SELECT ARRAY_DISTINCT(
    ARRAY(1, 2, 3, 3)
);
```

结果：

```
[1, 2, 3]
```

## 12.4 ARRAY_COMPACT

删除 Array 中的 NULL：

```
SELECT ARRAY_COMPACT(
    ARRAY(1, 2, NULL, 3, NULL)
);
```

结果：

```
[1, 2, 3]
```

## 12.5 ARRAY_CONTAINS

判断 Array 是否包含指定值：

```
SELECT ARRAY_CONTAINS(
    ARRAY(1, 2, 3),
    2
);
```

结果：

```
TRUE
```

## 12.6 ARRAY_INSERT

在指定位置插入元素：

```
SELECT ARRAY_INSERT(
    ARRAY(1, 2, 3),
    1,
    8
);
```

结果：

```
[8, 1, 2, 3]
```

注意：`ARRAY_INSERT` 的位置采用从 1 开始的索引，与普通 Array 方括号的从 0 开始索引不同。

## 12.7 ARRAY_POSITION

查找元素的位置：

```
SELECT ARRAY_POSITION(
    ARRAY(1, 2, 3),
    2
);
```

结果：

```
2
```

返回的是从 1 开始的位置。如果没有找到，返回 0。

## 12.8 Other Useful Array Functions

建议额外记住：

```
-- Array length
SELECT SIZE(ARRAY(1, 2, 3));

-- Sort
SELECT ARRAY_SORT(ARRAY(3, 1, 2));

-- Reverse
SELECT ARRAY_REVERSE(ARRAY(1, 2, 3));

-- Flatten nested arrays
SELECT FLATTEN(
    ARRAY(
        ARRAY(1, 2),
        ARRAY(3, 4)
    )
);

-- Remove an element
SELECT ARRAY_REMOVE(
    ARRAY(1, 2, 3),
    2
);
```

## 12.9 Interview Questions

Q1. What is the difference between ARRAY and MAP?

ARRAY 是有序集合，MAP 是 Key-Value 结构。

Q2. What is the difference between ARRAY_DISTINCT and COLLECT_SET?

两者都可以用于去重，但前者是 Array Transformation，后者是 Group Aggregation。

Q3. How do you check whether an array contains a specific value?

使用 `ARRAY_CONTAINS()`。

Q4. How would you flatten nested arrays?

可以结合 `FLATTEN()` 或 `EXPLODE()`。

# Module 13. Spark SQL Configuration & Performance Tuning

你记录的这一部分很重要，但原始笔记缺少完整的配置名称和作用。

Spark SQL 配置通常通过：

```
spark.conf.set("spark.sql.shuffle.partitions", "200")
```

或者 SQL：

```
SET spark.sql.shuffle.partitions = 200;
```

查看：

```
SET spark.sql.shuffle.partitions;
```

## 13.1 Partition Configurations

|Configuration|Meaning|
|---|---|
|`spark.sql.files.minPartitionNum`|文件读取时建议的最小 Partition 数|
|`spark.sql.files.maxPartitionNum`|文件读取时建议的最大 Partition 数|
|`spark.sql.sources.parallelPartitionDiscovery.threshold`|触发并行文件路径发现的阈值|
|`spark.sql.shuffle.partitions`|Shuffle 操作默认 Partition 数|

### spark.sql.shuffle.partitions

默认通常是 200。

```
spark.conf.set(    "spark.sql.shuffle.partitions",    "100")
```

主要影响：

- GroupBy。
    
- Join。
    
- Distinct。
    
- Window 等涉及 Shuffle 的操作。
    

例如：

```
df.groupBy("customer_id").count()
```

如果 Shuffle Partition 太少：

- 每个 Partition 数据量过大。
    
- Task 可能执行时间较长。
    
- 容易出现 Memory Pressure。
    

如果 Partition 太多：

- Task 数量过多。
    
- Task Scheduling Overhead 增加。
    
- 可能产生大量小文件。
    

注意：AQE 开启时，Spark 可以根据运行时统计信息合并 Shuffle Partitions，因此不能只看初始配置判断最终 Partition 数量。

## 13.2 Broadcast Join Configurations

### autoBroadcastJoinThreshold

```
spark.conf.set(    "spark.sql.autoBroadcastJoinThreshold",    "10485760")
```

默认通常是 10MB。

当较小的一侧数据集低于阈值，并且满足相关条件时，Spark 可能自动选择 Broadcast Join。

例如：

```
SELECT *
FROM orders o
JOIN customers c
ON o.customer_id = c.customer_id;
```

如果 customers 很小，可以广播 customers。

也可以手动指定：

```
SELECT /*+ BROADCAST(c) */
    o.*,
    c.customer_name
FROM orders o
JOIN customers c
ON o.customer_id = c.customer_id;
```

### broadcastTimeout

```
spark.conf.set(    "spark.sql.broadcastTimeout",    "300")
```

广播数据等待完成的超时时间，单位是秒。

如果广播数据太大或网络传输慢，可能出现 Broadcast Timeout。

## 13.3 Adaptive Query Execution

正确的配置名称：

```
spark.conf.set(    "spark.sql.adaptive.enabled",    "true")
```

相关配置：

```
spark.conf.set(    "spark.sql.adaptive.coalescePartitions.enabled",    "true")spark.conf.set(    "spark.sql.adaptive.skewJoin.enabled",    "true")
```

主要能力：

1. Dynamic Partition Coalescing。
    
2. Dynamic Join Strategy Optimization。
    
3. Skew Join Optimization。
    

## 13.4 Other Important Configurations

|Configuration|Purpose|
|---|---|
|`spark.sql.autoBroadcastJoinThreshold`|自动广播 Join 阈值|
|`spark.sql.broadcastTimeout`|Broadcast 超时时间|
|`spark.sql.adaptive.enabled`|开启 AQE|
|`spark.sql.adaptive.skewJoin.enabled`|开启 Skew Join 优化|
|`spark.sql.adaptive.coalescePartitions.enabled`|合并 Shuffle Partitions|
|`spark.sql.session.timeZone`|Session 时区|
|`spark.sql.caseSensitive`|SQL 字段名大小写敏感性|
|`spark.sql.files.maxPartitionBytes`|文件读取时每个 Partition 的目标最大字节数|

注意：不同 Spark 版本和 Databricks Runtime 可能有不同的默认值或配置支持情况。

## 13.5 Interview Questions

Q1. What is the default shuffle partition count in Spark SQL?

通常为 200，但可以通过 `spark.sql.shuffle.partitions` 修改。

Q2. What is autoBroadcastJoinThreshold?

控制 Spark 自动选择 Broadcast Join 时使用的数据大小阈值。

Q3. What happens if shuffle partitions are too high?

可能导致过多小 Task，增加调度开销和小文件问题。

Q4. What is the difference between shuffle partitions and file scan partitions?

Shuffle Partitions 控制 Shuffle 阶段的数据分布；File Scan Partitions 控制读取文件时的任务拆分。

Q5. What is the relationship between AQE and shuffle partitions?

初始 Shuffle Partition 可以根据配置确定，AQE 再根据运行时统计信息动态调整。

# Module 14. UDF — User Defined Functions

当 Spark SQL 内置函数不能满足业务逻辑时，可以创建自定义函数。

## 14.1 Python UDF Registration

定义 Python Function：

```
def my_upper_function(option: str) -> str:    return option.upper()
```

注册：

```
spark.udf.register(    "my_upper",    my_upper_function)
```

SQL 中使用：

```
SELECT
    my_upper(customer_name)
FROM customers;
```

也可以：

```
df = spark.sql("""    SELECT        my_upper(customer_name) AS customer_name    FROM customers""")
```

## 14.2 SQL UDF

如果逻辑可以使用 SQL 内置函数实现，不一定需要 Python UDF。

可以直接创建 SQL Function：

```
CREATE OR REPLACE TEMPORARY FUNCTION my_upper(x STRING)
RETURNS STRING
RETURN UPPER(x);
```

使用：

```
SELECT my_upper('john');
```

也可以创建持久化 Function（需要 Catalog 支持及相应权限）：

```
CREATE OR REPLACE FUNCTION
sparkcatalog.raw.my_upper(x STRING)
RETURNS STRING
RETURN UPPER(x);
```

区别：

|Type|Definition|Scope|
|---|---|---|
|Python UDF|Python Function|Spark Session 注册|
|SQL UDF|SQL Expression|Temporary 或 Persistent|
|Built-in Function|Spark 内置|直接使用|

## 14.3 UDF Performance

为什么 Python UDF 通常比 Spark Built-in Functions 慢？

执行过程大致为：

```
Spark JVM
    |
    v
Serialize Data
    |
    v
Python Worker
    |
    v
Execute Python Function
    |
    v
Serialize Result
    |
    v
Spark JVM
```

涉及：

- JVM ↔ Python Serialization。
    
- Python Worker Communication。
    
- Row-by-row Python Execution（普通 Scalar Python UDF）。
    

因此：

优先使用 Spark Built-in Functions，其次考虑 Pandas UDF，最后才考虑普通 Python UDF。

例如，不建议为了大写字符串使用 Python UDF：

```
df.withColumn(    "name",    upper(col("name")))
```

直接使用 Spark 内置函数即可。

## 14.4 Interview Questions

Q1. What is a UDF?

用户自定义函数，用于扩展 Spark SQL 的计算能力。

Q2. Why are Python UDFs usually slower than built-in functions?

因为需要 Python 和 JVM 之间的数据序列化与通信，而且普通 Python UDF 通常按行执行。

Q3. When should you use a UDF?

当内置函数无法实现特定业务逻辑时。

Q4. Why would you prefer SQL UDF over Python UDF?

如果逻辑可以通过 SQL 表达式实现，SQL UDF 更容易被 Spark SQL 优化器理解和优化。

# Module 15. SQL Query Files & Data Source Connectors

这一部分是 Spark SQL 中非常实用的文件读取能力。

你记录了三种方法，需要分别理解。

## 15.1 Create Temporary View from CSV

不需要先通过 Python DataFrame API 读取 CSV，也可以直接用 SQL 创建 View。

```
CREATE OR REPLACE TEMPORARY VIEW orders_csv
USING csv
OPTIONS (
    path '/data/orders.csv',
    header 'true',
    inferSchema 'true'
);
```

然后查询：

```
SELECT *
FROM orders_csv;
```

特点：

- View 是临时的。
    
- 数据读取由 CSV Data Source 完成。
    
- 不会自动把 CSV 转换成 Delta Table。
    
- 查询时读取底层文件。
    

## 15.2 Query Files Directly

Spark SQL 支持直接通过 Data Source 和路径读取文件。

例如 JSON：

```
SELECT *
FROM json.`/data/orders.json`;
```

Parquet：

```
SELECT *
FROM parquet.`/data/orders.parquet`;
```

CSV：

```
SELECT *
FROM csv.`/data/orders.csv`;
```

如果需要指定 Header、Schema 等选项，通常使用 Temporary View 或 DataFrame Reader API 更方便。

## 15.3 DataFrame Registration

如果已经通过 PySpark 读取了 DataFrame：

```
df = spark.read.parquet(    "/data/orders")
```

注册 View：

```
df.createOrReplaceTempView("orders")
```

然后使用 SQL：

```
df_sql = spark.sql("""    SELECT *    FROM orders    WHERE amount > 100""")
```

这是 PySpark 和 Spark SQL 混合开发最常用的方式。

你原来的：

```
spark.sql(    "SELECT * FROM {df} WHERE...",    df=df_csv)
```

不是正确的 DataFrame 注入方式。

正确做法是先注册 View，再通过 View 名称查询。

## 15.4 Data Source Formats

Spark SQL 常见数据源：

|Format|Usage|
|---|---|
|CSV|结构化文本文件|
|JSON|半结构化数据|
|Parquet|列式存储|
|ORC|列式存储|
|Delta|支持事务管理的 Lakehouse Table|
|JDBC|关系型数据库|

例如 JDBC：

```
df = (    spark.read    .format("jdbc")    .option("url", jdbc_url)    .option("dbtable", "orders")    .option("user", username)    .option("password", password)    .option("driver", "org.postgresql.Driver")    .load())df.createOrReplaceTempView("orders")
```

然后：

```
SELECT *
FROM orders
WHERE amount > 100;
```

## 15.5 Interview Questions

Q1. How can you query CSV files directly using Spark SQL?

可以通过 `CREATE TEMPORARY VIEW USING csv` 或 Data Source Path Syntax。

Q2. What is the difference between reading a file using DataFrame API and SQL?

两者都可以使用 Spark Data Source 读取数据，最终可以得到相同的 Spark SQL DataFrame 执行引擎。

Q3. Does creating a Temporary View copy the underlying data?

不会，通常只是注册查询逻辑或 DataFrame 引用。

Q4. How would you combine PySpark and Spark SQL in a data pipeline?

通过 DataFrame API 读取、清洗和注册 View，再使用 SQL 完成复杂 Join、Aggregation 和业务逻辑。

# Module 16. 综合面试场景题

下面这些题目建议重点练习。它们可以把前面的 SQL Functions、Window、JOIN、CTE、Delta Lake 和性能优化结合起来。

## Scenario 1. Find the latest record for each customer

问题： 一个 Customer 可能有多条更新记录，如何保留最新的一条？

```
WITH ranked_customers AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
)

SELECT *
FROM ranked_customers
WHERE rn = 1;
```

考察：

- CTE。
    
- ROW_NUMBER。
    
- Window PARTITION BY。
    
- Deduplication。
    

## Scenario 2. Calculate the previous transaction amount

问题： 如何计算每个客户上一笔交易金额？

```
SELECT
    customer_id,
    transaction_date,
    amount,
    LAG(amount) OVER (
        PARTITION BY customer_id
        ORDER BY transaction_date
    ) AS previous_amount
FROM transactions;
```

考察：

- LAG。
    
- Window Function。
    
- PARTITION BY。
    
- ORDER BY。
    

## Scenario 3. Find customers without orders

问题： 找出没有下过订单的客户。

方法一：LEFT ANTI JOIN。

```
SELECT c.*
FROM customers c
LEFT ANTI JOIN orders o
ON c.customer_id = o.customer_id;
```

方法二：NOT EXISTS。

```
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

考察：

- Join Types。
    
- Anti Join。
    
- EXISTS / NOT EXISTS。
    

## Scenario 4. Calculate running revenue

问题： 按照日期计算累计收入。

```
SELECT
    order_date,
    SUM(amount) AS daily_revenue,
    SUM(SUM(amount)) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS cumulative_revenue
FROM orders
GROUP BY order_date;
```

考察：

- GROUP BY。
    
- Aggregate Functions。
    
- Window Functions。
    
- Window Frame。
    

## Scenario 5. Convert nested JSON into relational data

问题： 一个订单包含多个产品，如何将产品数组展开？

原始结构：

```
order_id | products
---------|-----------------
1        | [Laptop, Mouse]
2        | [Phone, Charger]
```

SQL：

```
SELECT
    order_id,
    product
FROM orders
LATERAL VIEW EXPLODE(products) t AS product;
```

也可以使用 Spark SQL 的 `LATERAL VIEW OUTER EXPLODE` 保留空数组或 NULL 对应的原始记录。

考察：

- ARRAY。
    
- EXPLODE。
    
- Nested Data。
    
- LATERAL VIEW。
    

## Scenario 6. Build an incremental SQL pipeline

问题： 每天将新订单和更新订单同步到 Delta Silver Table。

```
MERGE INTO silver.orders AS t

USING (
    SELECT *
    FROM (
        SELECT
            *,
            ROW_NUMBER() OVER (
                PARTITION BY order_id
                ORDER BY updated_at DESC
            ) AS rn
        FROM bronze.orders
    )
    WHERE rn = 1
) AS s

ON t.order_id = s.order_id

WHEN MATCHED AND s.updated_at > t.updated_at
THEN UPDATE SET *

WHEN NOT MATCHED
THEN INSERT *;
```

考察：

- Incremental ETL。
    
- CTE / Subquery。
    
- Window Function。
    
- MERGE。
    
- Deduplication。
    
- Idempotency。
    

# Module 17. 面试复习优先级

针对荷兰 Junior / Mid-level Data Engineer 岗位，我建议你按照以下顺序复习 Spark SQL。

## 第一优先级：SQL 必须熟练

重点准备

1. JOIN（Inner / Left / Full / Anti）
    
2. GROUP BY & HAVING
    
3. CTE & Subquery
    
4. Window Functions
    
5. ROW_NUMBER / RANK / DENSE_RANK
    
6. LAG / LEAD
    
7. CASE WHEN
    
8. MERGE / UPSERT
    
9. NULL Handling
    
10. Deduplication
    
    ## 第二优先级：Spark SQL 原理
    
    理解底层机制
    
11. Temporary View vs Global Temporary View
    
12. DataFrame vs Spark SQL
    
13. Catalyst Optimizer
    
14. EXPLAIN / Physical Plan
    
15. Partition Pruning
    
16. Broadcast Join
    
17. Shuffle Partitions
    
18. AQE
    
    ## 第三优先级：Databricks 实际开发
    
    平台实践
    
19. Delta Table DDL / DML
    
20. Managed vs External Table
    
21. Time Travel
    
22. Schema Evolution
    
23. Array / Struct / Map
    
24. JSON Flattening
    
25. UDF
    
26. Date & Timezone
    
27. SQL Scripting
    
28. Catalog / Schema / Metadata
    

## 最后：Spark SQL 和 PySpark 如何串联？

你之前整理的 PySpark 和这次 Spark SQL 笔记，其实是同一套 Spark Engine 的两种使用方式。

建议从下面的角度理解：

|PySpark DataFrame API|Spark SQL|
|---|---|
|`select()`|SELECT|
|`filter()`|WHERE|
|`withColumn()`|SELECT + Expression|
|`groupBy().agg()`|GROUP BY|
|`join()`|JOIN|
|`dropDuplicates()`|DISTINCT / Window|
|`unionByName()`|UNION BY NAME|
|`row_number().over()`|ROW_NUMBER() OVER|
|`when()`|CASE WHEN|
|`explode()`|EXPLODE|
|`createOrReplaceTempView()`|CREATE TEMP VIEW|
|`DeltaTable.merge()`|MERGE INTO|

最重要的理解：

PySpark DataFrame API 和 Spark SQL 最终都可以转换成 Spark SQL Logical Plan，再由 Catalyst Optimizer 优化并生成 Physical Plan。

所以面试时不要把它们理解成两套完全独立的计算引擎。

对于实际工作：

- 简单的数据转换可以用 DataFrame API。
    
- 复杂的 JOIN、Aggregation、CTE 和 Window Logic 可以用 SQL。
    
- 需要结合 Python 业务逻辑时，可以使用 PySpark。
    
- 需要事务性增量更新时，可以使用 Delta MERGE。
    

一个需要特别记住的版本问题： Spark SQL 和 Databricks SQL 并非完全相同的 SQL 方言。尤其是 SQL Scripting、部分日期函数、DDL 和 Delta 专有语法，要确认实际 Runtime 是否支持，不能把某个 Databricks SQL Warehouse 的语法默认当成所有 Apache Spark 版本都支持。