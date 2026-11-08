# Module 1. PySpark Architecture & Execution Fundamentals

虽然你的笔记主要是 Transformation API，但在面试中，面试官通常会先考察你是否理解 Spark 的执行机制。

## 1.1 Spark Architecture

Spark 主要包含三个核心角色：

|Component|Responsibility|
|---|---|
|Driver|运行 Spark Application，构建执行计划，调度任务|
|Cluster Manager|管理集群资源，分配 CPU、Memory|
|Executor|在 Worker Node 上执行 Tasks，处理数据并缓存数据|

执行流程：

```
Spark Application
       |
       v
     Driver
       |
       | Request resources
       v
 Cluster Manager
       |
       | Allocate resources
       v
 Worker Nodes
       |
       v
    Executors
       |
       v
      Tasks
```

注意：

- Driver 并不是 Cluster Manager 创建的。
    
- 在 YARN Cluster Mode 下，ApplicationMaster 会负责协调 Application 的资源申请和运行。
    
- 在 Spark Standalone 模式下，Driver 和 Worker 的启动方式不同。
    
- Executor 是运行在 Worker Node 上的进程。
    
- 每个 Executor 可以有多个 CPU cores，同时运行多个 Tasks。
    

## 1.2 Transformation vs Action

Spark 使用 Lazy Evaluation。

Transformation：

对 DataFrame 进行转换，但不会立即执行计算。

例如：

```
df_filter = df.filter(col("age") > 18)df_select = df_filter.select("name", "age")df_group = df_select.groupBy("age").count()
```

这些操作只是构建 Logical Plan。

Action：

触发实际执行。

```
df.show()df.count()df.collect()df.write.format("parquet").save("/path")
```

执行流程：

```
Transformations
      |
      v
Logical Plan
      |
      v
Catalyst Optimizer
      |
      v
Physical Plan
      |
      v
DAG Scheduler
      |
      v
Task Scheduler
      |
      v
Executors
```

## 1.3 Narrow vs Wide Transformation

这是 PySpark 面试非常重要的知识点。

Narrow Transformation

一个子 Partition 的数据只依赖于一个父 Partition，通常不需要 Shuffle。

例如：

- `select()`
    
- `filter()`
    
- `withColumn()`
    
- `map()`
    

Wide Transformation

一个子 Partition 可能依赖多个父 Partition，需要 Shuffle。

例如：

- `groupBy()`
    
- `distinct()`
    
- `dropDuplicates()`
    
- `join()`（不一定，Broadcast Join 可以避免 Shuffle）
    
- `orderBy()`
    

![Transformations and Actions in Spark Explained](https://images.openai.com/static-rsc-4/BuCqJCOOKpMD7oYSuAltJZ-NFsCVVgBtnq1cwi4uh5o-Yv54ryaonK8LprROsGF1Ls4VvT5BSqe8M-4PncGJyvjdDTD2_XFsdlO66rGLB06-s0hl5MpoFrl6V9sNsHtk2_XyjofzPYHGvyREn1YV4ea9XY0n1EXDIhBMbrVXNeM?purpose=inline)

[sparkplayground.com](https://www.sparkplayground.com/tutorials/spark-theory/transformations-actions)

## 1.4 Interview Questions

Q1. What is lazy evaluation in Spark?

回答要点：

- Transformations do not execute immediately.
    
- Spark builds a logical execution plan.
    
- Execution starts when an action is triggered.
    
- It allows Spark to optimize the entire execution plan.
    

Q2. What is the difference between narrow and wide transformations?

回答要点：

- Narrow transformations do not require data redistribution across partitions.
    
- Wide transformations typically require shuffle.
    
- Shuffle involves network transfer, disk I/O and serialization overhead.
    

Q3. What happens when you call `df.groupBy("customer_id").count()`?

回答要点：

1. Spark builds a logical aggregation plan.
    
2. Catalyst optimizes the plan.
    
3. Spark performs partial aggregation where possible.
    
4. Data is shuffled by `customer_id`.
    
5. Final aggregation is performed.
    
6. Results are returned or written when an action executes.
    

Q4. What is the difference between a Job, Stage and Task?

- Job: triggered by an Action.
    
- Stage: a group of tasks separated by shuffle boundaries.
    
- Task: the smallest unit of execution, processing one partition.
    

# Module 2. Reading Data Sources

PySpark 支持多种数据源，常见的有 CSV、JSON、Parquet、JDBC、Delta Lake。

## 2.1 Read CSV

```
df = (    spark.read    .format("csv")    .option("header", True)    .option("inferSchema", True)    .load("/data/orders.csv"))df.printSchema()df.show()
```

重要参数：

|Parameter|Meaning|
|---|---|
|header|第一行是否为列名|
|inferSchema|是否自动推断数据类型|
|delimiter|分隔符，默认逗号|
|encoding|文件编码|
|mode|错误记录处理模式|
|nullValue|指定哪个值代表 NULL|
|quote|引号字符|

注意：

- 正确写法是 `inferSchema`，不是 `inferShcema`。
    
- `df.schema` 返回 Schema 对象。
    
- `df.printSchema()` 用于打印 Schema。
    
- `inferSchema=True` 通常需要额外扫描数据来推断类型，因此生产环境更推荐显式定义 Schema。
    

## 2.2 Read JSON

JSON 可以是普通 JSON Lines，也可以是多行 JSON。

JSON Lines（每行一条 JSON）

```
{"id": 1, "name": "Alice"}
{"id": 2, "name": "Bob"}
```

```
df_json = (    spark.read    .format("json")    .option("multiLine", False)    .load("/data/orders.json"))
```

Multi-line JSON

```
[
  {
    "id": 1,
    "name": "Alice"
  },
  {
    "id": 2,
    "name": "Bob"
  }
]
```

```
df_json = (    spark.read    .format("json")    .option("multiLine", True)    .load("/data/orders.json"))
```

### Nested JSON Schema

对于嵌套 JSON，可以使用 `StructType` 定义 Schema。

```
from pyspark.sql.types import (    StructType,    StructField,    StringType,    IntegerType)name_schema = StructType([    StructField("givenName", StringType(), True),    StructField("familyName", StringType(), True)])my_schema = StructType([    StructField("id", IntegerType(), True),    StructField("name", name_schema, True)])df_json = (    spark.read    .schema(my_schema)    .json("/data/orders.json"))
```

读取嵌套字段：

```
df.select("name.givenName")
```

## 2.3 Read Parquet

```
df_parquet = (    spark.read    .format("parquet")    .load("/data/orders.parquet"))
```

Parquet 特点：

- Columnar Storage（列式存储）。
    
- Schema 通常存储在 Parquet 文件 Metadata 中。
    
- 支持 Compression。
    
- 支持 Column Pruning。
    
- 支持 Predicate Pushdown。
    

例如：

```
df = (    spark.read    .parquet("/data/orders.parquet")    .select("customer_id", "amount")    .filter(col("amount") > 100))
```

Spark 可以只读取需要的列，并尽可能在读取阶段过滤数据。

## 2.4 Read JDBC

```
my_connection = {    "user": "username",    "password": "password",    "driver": "com.mysql.cj.jdbc.Driver"}df = spark.read.jdbc(    url=my_url,    table="orders",    properties=my_connection)
```

也可以：

```
df = (    spark.read    .format("jdbc")    .option("url", my_url)    .option("dbtable", "orders")    .option("user", "username")    .option("password", "password")    .option("driver", "com.mysql.cj.jdbc.Driver")    .load())
```

### JDBC Parallel Read

如果数据库表很大，可以通过 Partition 参数并行读取。

```
df = (    spark.read    .format("jdbc")    .option("url", my_url)    .option("dbtable", "orders")    .option("user", "username")    .option("password", "password")    .option("partitionColumn", "order_id")    .option("lowerBound", 1)    .option("upperBound", 1000000)    .option("numPartitions", 8)    .load())
```

注意：

- `partitionColumn` 必须是数值、日期或时间戳类型。
    
- `lowerBound` 和 `upperBound` 用于计算分区步长，不是过滤条件。
    
- `numPartitions` 控制并行 JDBC 连接数量。
    
- 过多并发可能给数据库带来压力。
    

## 2.5 Read Error Handling

读取 CSV 和 JSON 时，可以指定错误记录处理模式。

```
df = (    spark.read    .format("json")    .option("mode", "PERMISSIVE")    .load("/data/orders.json"))
```

三种模式：

|Mode|Behavior|
|---|---|
|PERMISSIVE|尽可能读取数据，错误记录对应字段可能为 NULL|
|DROPMALFORMED|丢弃格式错误的记录|
|FAILFAST|遇到错误记录立即报错|

如果想保留错误记录，可以使用：

```
.option("columnNameOfCorruptRecord", "_corrupt_record")
```

注意：要保留原始错误内容，需要 Schema 中包含对应的字符串字段。

## 2.6 Custom Schema

生产环境中推荐显式定义 Schema，避免 Schema Inference 带来的额外开销和类型不稳定。

### StructType

```
from pyspark.sql.types import *my_schema = StructType([    StructField("order_id", IntegerType(), True),    StructField("customer_id", StringType(), True),    StructField("amount", DoubleType(), True)])df = (    spark.read    .format("csv")    .option("header", True)    .schema(my_schema)    .load("/data/orders.csv"))
```

### DDL Schema

也可以直接使用 SQL DDL 格式：

```
my_ddl_schema = """    order_id INT,    customer_id STRING,    amount DOUBLE"""df = (    spark.read    .format("csv")    .option("header", True)    .schema(my_ddl_schema)    .load("/data/orders.csv"))
```

## 2.7 Interview Questions

Q1. What is the difference between CSV and Parquet?

- CSV is row-oriented text format.
    
- Parquet is columnar binary format.
    
- Parquet supports compression and predicate pushdown.
    
- Parquet is generally more efficient for analytical workloads.
    

Q2. Why would you define an explicit schema instead of using inferSchema?

- Avoid schema inference overhead.
    
- Ensure consistent data types.
    
- Prevent unexpected schema changes.
    
- Improve production pipeline reliability.
    

Q3. How do you read a large relational database table efficiently using Spark?

- Use JDBC partitioning.
    
- Select only required columns.
    
- Push down filters where possible.
    
- Control the number of parallel JDBC connections.
    
- Avoid overloading the source database.
    

Q4. What is the difference between PERMISSIVE, DROPMALFORMED and FAILFAST?

解释三种模式的容错策略，以及什么时候需要保留坏数据用于后续 Data Quality 分析。

# Module 3. File Reading Options & Data Discovery

这部分是你笔记中比较容易遗漏的生产环境读取配置。

## 3.1 IgnoreCorruptFiles

当读取多个文件时，如果部分文件损坏，可以选择忽略。

```
df = (    spark.read    .option("ignoreCorruptFiles", "true")    .parquet("/data/orders/"))
```

作用：

- 忽略无法读取的损坏文件。
    
- 继续读取其他正常文件。
    
- 不代表自动修复损坏文件。
    

## 3.2 IgnoreMissingFiles

如果读取过程中部分文件不存在，可以忽略这些文件。

```
df = (    spark.read    .option("ignoreMissingFiles", "true")    .parquet("/data/orders/"))
```

例如：Spark 已经开始读取某个目录，但某个文件在读取过程中被其他任务删除。

## 3.3 Recursive File Lookup

递归读取目录下的所有文件，包括子目录。

```
df = (    spark.read    .format("parquet")    .option("recursiveFileLookup", "true")    .load("/data/orders/"))
```

注意：启用 `recursiveFileLookup` 后，Spark 默认不会根据目录结构推断 Partition Columns。

## 3.4 Path Global Filter

通过文件名或路径模式过滤需要读取的文件。

```
df = (    spark.read    .format("parquet")    .option("pathGlobFilter", "*.parquet")    .load("/data/orders/"))
```

`pathGlobFilter` 是正确的参数名称，不是 `pathGlobalFilter`。

## 3.5 Modification Time Filters

可以根据文件修改时间过滤文件：

```
df = (    spark.read    .format("parquet")    .option("modifiedBefore", "2026-01-01T00:00:00")    .option("modifiedAfter", "2025-01-01T00:00:00")    .load("/data/orders/"))
```

- `modifiedBefore`：只读取指定时间之前修改的文件。
    
- `modifiedAfter`：只读取指定时间之后修改的文件。
    

注意：这些是文件级过滤，不是数据记录中的时间字段过滤。

## 3.6 Read Delta Table from Storage Location

在 Databricks 中，可以通过 Unity Catalog 查询表的物理存储位置。

```
location = (    spark.sql("DESCRIBE DETAIL events")    .select("location")    .first()["location"])df = spark.read.format("delta").load(location)
```

这里需要区分：

- Unity Catalog Table：通过 Catalog、Schema、Table 名称访问。
    
- Delta Path：直接通过 Storage Location 访问 Delta 数据。
    

生产环境优先使用 Unity Catalog 管理的表名，因为直接访问路径可能绕过部分 Catalog 权限管理和治理机制。

注意：Managed Table 也有物理存储路径，并不是不能通过路径读取；只是直接读取路径不等同于通过 UC 表对象访问。

## 3.7 Interview Questions

Q1. What is the difference between ignoreCorruptFiles and ignoreMissingFiles?

- `ignoreCorruptFiles` ignores files that cannot be read because they are corrupted.
    
- `ignoreMissingFiles` ignores files that disappear during processing.
    

Q2. What happens when you enable recursiveFileLookup?

- Spark recursively reads files from nested directories.
    
- Partition discovery based on directory structure is disabled.
    

Q3. What is the difference between reading a Delta table by its table name and by its storage path?

- Table name access goes through the Catalog.
    
- Path access directly locates the Delta table.
    
- Catalog access provides centralized governance and permissions.
    

# Module 4. Basic DataFrame Transformations

这是日常 PySpark ETL 最常使用的部分。

## 4.1 Select & Alias

提取指定列：

```
from pyspark.sql.functions import *df_select = df.select("id", "customer_id")df_select = df.select(    col("id"),    col("customer_id"))
```

Alias：修改查询结果中的列名。

```
df_alias = df.select(    col("id").alias("customer_id"),    col("city"))
```

注意：`select().alias()` 不会修改原始 DataFrame。

## 4.2 Filter / Where

筛选数据：

```
df_filter = df.filter(col("city") == "SGNS")
```

多个条件：

```
df_filter = df.filter(    (col("city") == "SGNS") |    (col("city") == "AGAE"))
```

也可以使用 `where()`：

```
df_filter = df.where(col("age") > 18)
```

`filter()` 和 `where()` 功能相同。

### Isin

```
df_filter = df.filter(    col("city").isin(["C1", "C2"]))
```

排除指定值：

```
df_filter = df.filter(    ~col("city").isin(["C1", "C2"]))
```

## 4.3 Rename Columns

```
df_rename = df.withColumnRenamed(    "id",    "customer_id")
```

如果需要修改多个列，可以连续使用：

```
df_rename = (    df    .withColumnRenamed("id", "customer_id")    .withColumnRenamed("name", "customer_name"))
```

注意：PySpark DataFrame 是不可变的（Immutable）。

`withColumnRenamed()` 不会直接修改原始 DataFrame，而是返回一个新的 DataFrame。

## 4.4 WithColumn

`withColumn()` 用于：

1. 创建新列。
    
2. 修改已有列。
    
3. 对已有列进行转换。
    

创建新列：

```
df_mod = df.withColumn(    "flag",    lit(0))
```

字符串替换：

```
df_mod = df.withColumn(    "shipping_address",    regexp_replace(        col("shipping_address"),        ",.*",        ""    ))
```

计算金额：

```
df_mod = (    df    .withColumn(        "total",        col("price") * col("quantity")    )    .withColumn(        "total",        round(col("total"), 2)    ))
```

## 4.5 Type Casting

转换数据类型：

```
df_cast = df.withColumn(    "order_id",    col("order_id").cast(StringType()))
```

也可以：

```
df_cast = df.withColumn(    "order_id",    col("order_id").cast("STRING"))
```

常见类型：

|PySpark Type|SQL Type|
|---|---|
|StringType|STRING|
|IntegerType|INT|
|LongType|BIGINT|
|DoubleType|DOUBLE|
|FloatType|FLOAT|
|BooleanType|BOOLEAN|
|DateType|DATE|
|TimestampType|TIMESTAMP|

注意：无法转换的值通常会变成 NULL（在 ANSI 模式下，部分非法转换可能直接报错）。

## 4.6 Sorting

单列排序：

```
df_sort = df.sort(    col("order_id").desc())
```

多列排序：

```
df_sort = df.sort(    ["order_id", "quantity"],    ascending=[False, True])
```

也可以：

```
df_sort = df.orderBy(    col("order_id").desc(),    col("quantity").asc())
```

注意：全局排序通常需要 Shuffle，是开销较大的操作。

## 4.7 Drop Columns

删除列：

```
df_drop = df.drop("address")
```

删除多列：

```
df_drop = df.drop("address", "phone")
```

## 4.8 Interview Questions

Q1. What is the difference between select() and withColumn()?

- `select()` selects and projects columns.
    
- `withColumn()` adds a new column or replaces an existing column.
    
- Both return a new DataFrame.
    

Q2. Is a PySpark DataFrame mutable?

No. DataFrames are immutable. Transformations return new DataFrames.

Q3. What is the difference between filter() and where()?

They are equivalent APIs.

Q4. Why can repeated withColumn() operations cause performance issues?

A long chain of column additions can create a large logical plan. Catalyst can optimize many expressions, but excessively complex plans may increase planning overhead. For many columns, using a single `select()` with multiple expressions can be cleaner.

# Module 5. Data Deduplication, Union & Null Handling

## 5.1 Distinct vs DropDuplicates

两者都可以去重，但使用方式略有不同。

### Distinct

对所有列进行去重：

```
df_distinct = df.distinct()
```

### DropDuplicates

不指定列：

```
df_dedup = df.dropDuplicates()
```

效果与 `distinct()` 相同。

指定列：

```
df_dedup = df.dropDuplicates(    ["customer_id"])
```

这里有一个非常重要的面试知识点：

如果同一个 customer_id 对应多条记录，Spark 不保证保留哪一条记录。

如果你需要保留最新记录，应该使用 Window Function，而不是直接 `dropDuplicates()`。

例如：

```
window_spec = (    Window    .partitionBy("customer_id")    .orderBy(col("updated_at").desc()))df_dedup = (    df    .withColumn(        "rn",        row_number().over(window_spec)    )    .filter(col("rn") == 1)    .drop("rn"))
```

## 5.2 Streaming Deduplication with Watermark

在 Structured Streaming 中，可以使用 Watermark 限制状态存储。

```
df_dedup = (    streaming_df    .withWatermark("order_timestamp", "30 seconds")    .dropDuplicates(["customer_id", "order_id"]))
```

Watermark 的作用：

- 跟踪事件时间进度。
    
- 清理超过允许延迟范围的旧状态。
    
- 避免 Streaming Deduplication 的 State Store 无限增长。
    

注意：Watermark 不是简单的保留最近 30 秒的数据，也不意味着严格保证所有超过 30 秒的数据都会立即被丢弃。

## 5.3 Union vs UnionByName

### Union

按照列的位置合并：

```
df_union = df1.union(df2)
```

例如：

```
df1:
id | name

df2:
name | id
```

Union 会根据位置匹配，不会根据列名匹配。

### UnionByName

根据列名匹配：

```
df_union = df1.unionByName(df2)
```

如果列不同：

```
df_union = df1.unionByName(    df2,    allowMissingColumns=True)
```

缺少的列会补 NULL。

注意：

- `union()` 和 `unionByName()` 都不会自动去重。
    
- 如果想去重，可以在之后使用 `distinct()`。
    

## 5.4 Handling NULL Values

### Dropna

删除 NULL 记录。

删除所有列都为 NULL 的行：

```
df.dropna(how="all")
```

删除任意列为 NULL 的行：

```
df.dropna(how="any")
```

指定列：

```
df.dropna(    subset=["name", "address"])
```

只要指定列中有任意一列为 NULL，就会删除该行。

指定列全部为 NULL 才删除：

```
df.dropna(    subset=["name", "address"],    how="all")
```

还可以使用 `thresh`：

```
df.dropna(thresh=3)
```

表示至少需要有 3 个非 NULL 列，否则删除。

### Fillna

所有适用列填充相同的值：

```
df.fillna("unknown")
```

指定不同列：

```
df.fillna({    "name": "unknown",    "address": "unknown",    "age": 0})
```

也可以使用：

```
df.fillna(0, subset=["price", "quantity"])
```

注意：填充值必须与列的数据类型兼容。

### NULL vs NaN

这是面试中经常被混淆的地方：

- NULL：表示缺失值。
    
- NaN：浮点数中的 Not a Number。
    

它们不是同一个概念。

```
df.filter(col("name").isNull())df.filter(col("price").isNull())df.filter(isnan(col("price")))
```

`isNull()` 不等于 `isnan()`。

## 5.5 Interview Questions

Q1. What is the difference between distinct() and dropDuplicates()?

- `distinct()` removes duplicate complete rows.
    
- `dropDuplicates()` can remove duplicates based on selected columns.
    
- Neither guarantees which record is retained when subset columns have duplicates.
    

Q2. How would you keep the latest record for each customer?

使用：

- `Window.partitionBy(customer_id)`
    
- `orderBy(updated_at.desc())`
    
- `row_number()`
    
- Filter `row_number = 1`
    

Q3. What is the difference between union() and unionByName()?

- Union matches columns by position.
    
- UnionByName matches columns by name.
    
- `allowMissingColumns=True` allows missing columns and fills them with NULL.
    

Q4. Why do we use Watermark in Structured Streaming?

To control late-arriving data and clean old streaming state, preventing unbounded state growth.

Q5. What is the difference between NULL and NaN?

NULL represents missing or unknown data, while NaN is a special floating-point value.

# Module 6. Date & String Functions

## 6.1 Date Functions

导入：

```
from pyspark.sql.functions import *
```

获取当前时间：

```
df = df.withColumn(    "current_time",    current_timestamp())
```

增加日期：

```
df = df.withColumn(    "new_date",    date_add(col("order_date"), 7))
```

减少日期：

```
df = df.withColumn(    "new_date",    date_sub(col("order_date"), 7))
```

计算两个日期之间的天数：

```
df = df.withColumn(    "duration",    datediff(        col("current_date"),        col("order_date")    ))
```

注意：函数名是 `datediff()`，不是 `date_diff()`。

日期格式化：

```
df = df.withColumn(    "formatted_date",    date_format(        col("order_date"),        "yyyyMMdd"    ))
```

常见日期格式：

|Format|Meaning|
|---|---|
|yyyy|四位年份|
|MM|月份|
|dd|日期|
|HH|24 小时制|
|mm|分钟|
|ss|秒|

字符串转换成日期：

```
df = df.withColumn(    "order_date",    to_date(col("order_date"), "yyyy-MM-dd"))
```

字符串转换成 Timestamp：

```
df = df.withColumn(    "order_timestamp",    to_timestamp(        col("order_timestamp"),        "yyyy-MM-dd HH:mm:ss"    ))
```

计算时间差：

```
df = df.withColumn(    "duration_seconds",    unix_timestamp(col("end_time")) -    unix_timestamp(col("start_time")))
```

其他常见函数：

```
year("order_date")month("order_date")dayofmonth("order_date")dayofweek("order_date")quarter("order_date")weekofyear("order_date")
```

## 6.2 String Functions

转换大写：

```
df = df.withColumn(    "status",    upper(col("status")))
```

转换小写：

```
lower(col("status"))
```

去除首尾空格：

```
trim(col("name"))
```

只去除左边：

```
ltrim(col("name"))
```

只去除右边：

```
rtrim(col("name"))
```

字符串长度：

```
length(col("name"))
```

字符串替换：

```
regexp_replace(    col("address"),    ",.*",    "")
```

字符串截取：

```
substring(col("order_id"), 1, 5)
```

## 6.3 Split & Array Indexing

将字符串拆分为 Array：

```
df = df.withColumn(    "address_array",    split(col("shipping_address"), ","))
```

提取第一个元素：

```
df = df.withColumn(    "street_address",    col("address_array")[0])
```

或者：

```
df = df.withColumn(    "street_address",    split(col("shipping_address"), ",")[0])
```

注意：

PySpark DataFrame 的 Array 下标从 0 开始。

## 6.4 Concat & Concat_WS

拼接字符串：

```
df = df.withColumn(    "customer_order_id",    concat(        col("customer_id"),        lit("|"),        col("order_id")    ))
```

`concat_ws()`：使用指定分隔符拼接。

```
df = df.withColumn(    "customer_order_id",    concat_ws(        "|",        col("customer_id"),        col("order_id")    ))
```

区别：

- `concat()`：需要自己添加分隔符。
    
- `concat_ws()`：自动添加分隔符，并且通常会跳过 NULL 参数。
    

## 6.5 Interview Questions

Q1. What is the difference between date_add() and datediff()?

- `date_add()` adds days to a date.
    
- `datediff()` calculates the number of days between two dates.
    

Q2. What is the difference between concat() and concat_ws()?

- `concat()` concatenates columns directly.
    
- `concat_ws()` concatenates using a specified separator and skips NULL inputs.
    

Q3. How would you extract the city from a comma-separated address?

使用 `split()` 将字符串转换为 Array，再通过 Array Index 读取指定元素。

Q4. How do you handle different timestamp formats in an ETL pipeline?

使用 `to_timestamp()` 指定输入格式，统一时区，并处理无法解析的记录。

# Module 7. Array, Struct & Complex Data Types

对于处理 JSON、API 数据、Nested Data，这个模块非常重要。

## 7.1 Struct

Struct 是一个嵌套结构，类似于 JSON Object。

例如：

```
{
  "customer": {
    "id": 1,
    "name": "Alice"
  }
}
```

读取嵌套字段：

```
df.select(    col("customer.id"),    col("customer.name"))
```

也可以：

```
df.select("customer.*")
```

将 Struct 中的字段展开。

## 7.2 Array

Array 是一个元素集合。

```
df = df.withColumn(    "city_array",    split(col("cities"), ","))
```

判断 Array 是否包含某个值：

```
df = df.withColumn(    "city_flag",    array_contains(        col("city_array"),        "Amsterdam"    ))
```

## 7.3 Explode

`explode()` 将 Array 中的每个元素展开为单独的一行。

例如：

原始数据：

|customer_id|products|
|---|---|
|1|[A, B, C]|
|2|[D, E]|

执行：

```
df_explode = df.withColumn(    "product",    explode(col("products")))
```

结果：

|customer_id|product|
|---|---|
|1|A|
|1|B|
|1|C|
|2|D|
|2|E|

### Explode vs Explode_outer

```
df_explode = df.withColumn(    "product",    explode(col("products")))
```

```
df_explode_outer = df.withColumn(    "product",    explode_outer(col("products")))
```

区别：

- `explode()`：NULL 或空 Array 不产生对应输出行。
    
- `explode_outer()`：NULL 或空 Array 仍然保留一行，展开字段为 NULL。
    

## 7.4 Inline

如果 Array 中的元素是 Struct，可以使用 `inline()` 直接展开 Struct 字段。

```
df = df.select(    "customer_id",    inline(col("orders")))
```

这比先 explode 再提取 Struct 字段更加简洁。

## 7.5 Map / Dictionary

Map 是 Key-Value 结构。

例如：

```
{    "city": "Amsterdam",    "country": "Netherlands"}
```

读取：

```
df.select(    col("address_map")["city"].alias("city"))
```

也可以：

```
df.select(    col("address_map.city"))
```

前提是字段本身为 Map 或 Struct 时使用相应的访问方式。

## 7.6 Interview Questions

Q1. What is the difference between Array, Struct and Map?

- Array: collection of elements.
    
- Struct: collection of named fields with defined types.
    
- Map: key-value pairs.
    

Q2. What is the difference between explode() and explode_outer()?

Explain how they handle NULL and empty arrays.

Q3. How would you flatten a nested JSON dataset?

使用 Struct field access、`select("struct.*")`、`explode()` 和 `inline()`。

Q4. What happens to the row count after explode()?

如果一个 Array 有 N 个元素，该行通常会扩展成 N 行。

# Module 8. Aggregations & GroupBy

## 8.1 Basic Aggregations

```
df_agg = (    df    .groupBy("product_id", "customer_id")    .agg(        sum("total_price").alias("total_price"),        avg("total_price").alias("avg_price"),        count("*").alias("order_count")    ))
```

常见 Aggregation：

|Function|Meaning|
|---|---|
|count|统计行数或非 NULL 值|
|sum|求和|
|avg / mean|平均值|
|min|最小值|
|max|最大值|
|countDistinct|精确去重计数|
|approx_count_distinct|近似去重计数|
|first|第一个值|
|last|最后一个值|

注意：`count("*")` 会统计所有行；`count("column")` 不统计该列的 NULL 值。

## 8.2 Approximate Count Distinct

精确统计：

```
df_agg = (    df    .groupBy("product_id")    .agg(        countDistinct("customer_id").alias(            "distinct_customers"        )    ))
```

近似统计：

```
df_agg = (    df    .groupBy("product_id")    .agg(        approx_count_distinct(            "customer_id"        ).alias("distinct_customers")    ))
```

区别：

- `countDistinct()`：精确结果，但可能需要更多计算和 Shuffle 资源。
    
- `approx_count_distinct()`：使用 HyperLogLog++ 等近似算法，通常更节省资源，适合大规模数据的估算。
    

## 8.3 Collect_List vs Collect_Set

`collect_list()`：收集分组中的所有元素，保留重复值。

```
df_collect = (    df    .groupBy("customer_id")    .agg(        collect_list("product_id").alias("products")    ))
```

`collect_set()`：收集唯一元素，不保留重复值。

```
df_collect = (    df    .groupBy("customer_id")    .agg(        collect_set("product_id").alias("products")    ))
```

注意：两者都不保证元素顺序。

另外，如果某个 Group 非常大，聚合产生的 Array 可能占用大量内存。

## 8.4 Pivot

Pivot 将行转换为列。

原始数据：

|customer_id|order_status|order_id|
|---|---|---|
|1|Completed|101|
|1|Returned|102|
|2|Completed|103|

代码：

```
df_pivot = (    df    .groupBy("customer_id")    .pivot("order_status")    .agg(count("order_id")))
```

结果：

|customer_id|Completed|Returned|
|---|---|---|
|1|1|1|
|2|1|NULL|

如果已知状态集合，建议明确指定：

```
df_pivot = (    df    .groupBy("customer_id")    .pivot(        "order_status",        ["Completed", "Returned"]    )    .agg(count("order_id")))
```

这样可以避免 Spark 为推断 Pivot 列值而进行额外计算。

## 8.5 Conditional Logic: When / Otherwise

类似 SQL CASE WHEN。

```
df = df.withColumn(    "return_flag",    when(col("return_count") == 1, "low")    .when(col("return_count") <= 3, "mid")    .when(col("return_count") > 3, "high")    .otherwise("no return"))
```

也可以：

```
df = df.withColumn(    "discount",    when(col("amount") > 1000, 0.2)    .otherwise(0.1))
```

注意：条件从上往下判断，匹配第一个 TRUE 的条件后就返回结果。

## 8.6 Interview Questions

Q1. What is the difference between count(*) and count(column)?

- `count(*)` counts all rows.
    
- `count(column)` counts non-NULL values.
    

Q2. What is the difference between countDistinct() and approx_count_distinct()?

解释准确性、计算资源以及 HyperLogLog++ 的应用场景。

Q3. What is the difference between collect_list() and collect_set()?

- `collect_list()` retains duplicates.
    
- `collect_set()` removes duplicates.
    
- Neither guarantees ordering.
    

Q4. What is Pivot in PySpark?

Pivot transforms row values into columns, usually combined with aggregation.

Q5. What happens if one group contains billions of records?

可能导致数据倾斜、单个 Partition 过大、内存压力和 OOM，需要考虑预聚合、Skew Handling 和数据分区策略。

# Module 9. Joins

Join 是 Data Engineer 面试的重点之一。

## 9.1 Join Types

|Join Type|Behavior|
|---|---|
|Inner|只保留两边匹配的记录|
|Left|保留左表所有记录|
|Right|保留右表所有记录|
|Full Outer|保留两边所有记录|
|Left Semi|只返回左表中匹配的记录|
|Left Anti|返回左表中没有匹配的记录|
|Cross|笛卡尔积|

## 9.2 Basic Join

```
df_inner = df1.join(    df2,    df1.id == df2.id,    "inner")
```

如果 Join Key 相同：

```
df_inner = df1.join(    df2,    on="id",    how="inner")
```

也可以传入多个 Join Keys：

```
df_inner = df1.join(    df2,    on=["customer_id", "order_id"],    how="inner")
```

## 9.3 Select Columns After Join

如果两张表存在同名字段，建议使用 Alias。

```
left = df1.alias("l")right = df2.alias("r")df_join = (    left.join(        right,        col("l.id") == col("r.id"),        "inner"    )    .select(        col("l.id"),        col("l.name"),        col("r.amount")    ))
```

## 9.4 Left Semi Join

只保留左表中能够在右表找到匹配记录的行。

```
df_semi = df1.join(    df2,    df1.id == df2.id,    "left_semi")
```

可以理解为 SQL EXISTS。

## 9.5 Left Anti Join

保留左表中没有匹配记录的行。

```
df_anti = df1.join(    df2,    df1.id == df2.id,    "left_anti")
```

可以理解为 SQL NOT EXISTS。

实际使用场景：

- 找出尚未处理的订单。
    
- 找出源数据中不存在于目标表的记录。
    
- 数据质量检查。
    
- Incremental ETL。
    

## 9.6 Broadcast Join

当一张表比较小，可以将它广播到所有 Executors，避免大表之间的 Shuffle。

```
from pyspark.sql.functions import broadcastdf_join = df_large.join(    broadcast(df_small),    on="customer_id",    how="inner")
```

执行逻辑：

```
             Small Table
                  |
             Broadcast
                  |
       -----------------------
       |          |          |
    Executor   Executor   Executor
       |          |          |
       v          v          v
    Large      Large      Large
    Partition  Partition  Partition
```

优点：

- 避免大表之间的 Shuffle。
    
- 减少网络数据传输。
    
- 提高 Join 性能。
    

限制：

- 小表必须能够安全地放入各 Executor 的内存。
    
- 广播太大的表可能造成内存压力甚至 OOM。
    

## 9.7 Join Skew

Data Skew：部分 Join Key 出现频率特别高。

例如：

|customer_id|Count|
|---|---|
|A|100|
|B|200|
|NULL|10,000,000|

如果大量数据集中在 NULL 或某个热门 Key，某些 Tasks 会处理远多于其他 Tasks 的数据。

解决方式：

1. Broadcast Join（适用时）。
    
2. Salting。
    
3. AQE Skew Join Optimization。
    
4. 预聚合。
    
5. 合理处理 NULL Key。
    

Salting 思路：

```
Original Key: A

After Salting:
A_1
A_2
A_3
A_4
```

将热点 Key 分散到不同 Partition，降低单个 Task 的负担。

注意：Salting 需要对另一侧数据进行相应扩展或匹配，不能只修改一侧 Join Key，否则会漏掉匹配记录。

## 9.8 Interview Questions

Q1. What is the difference between Left Semi Join and Left Anti Join?

- Left Semi returns matching rows from the left DataFrame.
    
- Left Anti returns non-matching rows from the left DataFrame.
    
- Neither returns columns from the right DataFrame.
    

Q2. What is a Broadcast Join and when would you use it?

解释小表广播、避免 Shuffle，以及内存限制。

Q3. What is Data Skew and how do you solve it?

解释数据分布不均、Straggler Tasks、Salting、AQE。

Q4. Why can a Join produce more rows than the original DataFrame?

因为 Join Key 可能不是唯一的。如果两侧同一个 Key 都有多条记录，就会产生多对多匹配。

Q5. How would you optimize a large-to-large Join?

- Filter early.
    
- Select only required columns.
    
- Ensure compatible join keys and data types.
    
- Use suitable partitioning.
    
- Handle data skew.
    
- Inspect the physical execution plan.
    
- Leverage AQE where appropriate.
    

# Module 10. Window Functions

Window Function 是荷兰 Data Engineer 面试中非常值得重点练习的内容。

与 GroupBy 不同，Window Function 可以在保留原始行的基础上进行分组计算。

## 10.1 Window Structure

```
from pyspark.sql.window import Windowfrom pyspark.sql.functions import *window_spec = (    Window    .partitionBy("customer_id")    .orderBy("order_date"))df = df.withColumn(    "row_num",    row_number().over(window_spec))
```

三个核心概念：

- `partitionBy()`：定义分组。
    
- `orderBy()`：定义组内排序。
    
- `rowsBetween()` / `rangeBetween()`：定义 Window Frame。
    

## 10.2 Row_Number

给每个 Partition 中的记录分配唯一序号。

```
df = df.withColumn(    "row_num",    row_number().over(        Window        .partitionBy("customer_id")        .orderBy(col("order_date").desc())    ))
```

用途：

- 去重。
    
- 保留最新记录。
    
- Top N 查询。
    

## 10.3 Rank vs Dense_Rank vs Row_Number

假设分数为：

```
100, 90, 90, 80
```

|Score|Row_Number|Rank|Dense_Rank|
|---|---|---|---|
|100|1|1|1|
|90|2|2|2|
|90|3|2|2|
|80|4|4|3|

区别：

- `row_number()`：每条记录都有唯一序号。
    
- `rank()`：相同值排名相同，但后面会跳号。
    
- `dense_rank()`：相同值排名相同，但后面不跳号。
    

代码：

```
window_spec = Window.orderBy(    col("marks").desc())df = (    df    .withColumn(        "row_num",        row_number().over(window_spec)    )    .withColumn(        "rank",        rank().over(window_spec)    )    .withColumn(        "dense_rank",        dense_rank().over(window_spec)    ))
```

## 10.4 Deduplication with Window

例如每个客户只保留最新订单：

```
window_spec = (    Window    .partitionBy("customer_id")    .orderBy(        col("updated_at").desc(),        col("order_id").desc()    ))df_latest = (    df    .withColumn(        "rn",        row_number().over(window_spec)    )    .filter(col("rn") == 1)    .drop("rn"))
```

这里添加 `order_id` 是为了在 `updated_at` 相同的情况下提供确定性的排序。

## 10.5 Cumulative Sum

累计求和：

```
window_spec = (    Window    .partitionBy("customer_id")    .orderBy("order_date")    .rowsBetween(        Window.unboundedPreceding,        Window.currentRow    ))df = df.withColumn(    "cumulative_revenue",    sum("revenue").over(window_spec))
```

计算逻辑：

|Date|Revenue|Cumulative Revenue|
|---|---|---|
|Jan 1|100|100|
|Jan 2|200|300|
|Jan 3|150|450|

注意：你原笔记中的 `rowBetween()` 应改为 `rowsBetween()`。

## 10.6 Unbounded Following

如果想计算整个 Partition 的总和，并把相同总和写入每一行：

```
window_spec = (    Window    .partitionBy("customer_id")    .orderBy("order_date")    .rowsBetween(        Window.unboundedPreceding,        Window.unboundedFollowing    ))df = df.withColumn(    "total_revenue",    sum("revenue").over(window_spec))
```

注意：

- `unboundedPreceding`：从 Partition 开始。
    
- `currentRow`：当前行。
    
- `unboundedFollowing`：直到 Partition 结束。
    

如果不需要排序，也可以使用：

```
window_spec = Window.partitionBy("customer_id")
```

## 10.7 Lag & Lead

这是你笔记中值得补充的高频函数。

`lag()`：获取前一行的值。

```
df = df.withColumn(    "previous_revenue",    lag("revenue", 1).over(window_spec))
```

`lead()`：获取后一行的值。

```
df = df.withColumn(    "next_revenue",    lead("revenue", 1).over(window_spec))
```

常见用途：

- 比较本次订单与上次订单。
    
- 计算日环比。
    
- 检查状态变化。
    
- 检测连续事件。
    

例如计算收入变化：

```
df = df.withColumn(    "revenue_change",    col("revenue") -    lag("revenue", 1).over(window_spec))
```

## 10.8 Window Frame: Rows vs Range

这是一个容易被忽略的面试知识点。

- `rowsBetween()`：按照物理行的位置定义窗口。
    
- `rangeBetween()`：按照排序列的数值范围定义窗口。
    

例如：

```
Window.orderBy("salary").rowsBetween(-1, 0)
```

表示当前行及前一行。

而：

```
Window.orderBy("salary").rangeBetween(-100, 0)
```

表示排序值在当前 salary 减 100 到当前 salary 范围内的记录。

相同排序值可能进入同一个 Range Frame。

## 10.9 Interview Questions

Q1. What is the difference between GroupBy and Window Functions?

- GroupBy reduces the number of rows.
    
- Window Functions preserve the original rows and add calculated columns.
    

Q2. What is the difference between rank(), dense_rank() and row_number()?

通过排名是否跳号解释。

Q3. How do you get the latest record for each customer?

使用 `row_number()`，根据 `customer_id` 分组，按照 `updated_at DESC` 排序。

Q4. What is the difference between rowsBetween() and rangeBetween()?

解释物理行位置和排序值范围。

Q5. How do you calculate a running total?

使用 `sum().over()` 和 `rowsBetween(unboundedPreceding, currentRow)`。

Q6. What are lag() and lead() used for?

获取前一行和后一行的数据，常用于时间序列比较和变化检测。

# Module 11. UDFs & Pandas UDFs

这是 PySpark 中非常重要的性能优化知识。

## 11.1 Built-in Functions vs UDF

Spark 内置函数：

```
df = df.withColumn(    "square",    col("price") * col("price"))
```

Python UDF：

```
from pyspark.sql.functions import udffrom pyspark.sql.types import DoubleTypedef my_square(x):    return round(x * x, 2)my_square_udf = udf(    my_square,    DoubleType())df = df.withColumn(    "price_square",    my_square_udf("price"))
```

区别：

- Built-in Functions 可以被 Catalyst Optimizer 优化。
    
- Python UDF 通常需要在 JVM 和 Python Worker 之间进行数据转换。
    
- Python UDF 会引入额外的序列化和执行开销。
    

因此，优先使用 Spark Built-in Functions。

## 11.2 UDF Decorator

也可以直接使用装饰器：

```
@udf(returnType=DoubleType())def my_square(x):    return round(x * x, 2)
```

调用：

```
df = df.withColumn(    "price_square",    my_square(col("price")))
```

## 11.3 Register UDF for SQL

```
spark.udf.register(    "my_square_udf",    my_square)
```

然后可以在 SQL 中调用：

```
df.createOrReplaceTempView("orders")result = spark.sql("""    SELECT        price,        my_square_udf(price) AS price_square    FROM orders""")
```

## 11.4 Pandas UDF / Vectorized UDF

Pandas UDF 使用 Apache Arrow 进行 JVM 和 Python 之间的列式数据传输。

特点：

- 使用批次（Batch）处理数据。
    
- 减少逐行序列化和函数调用开销。
    
- 通常比普通 Python UDF 更快。
    
- 仍然需要考虑 Python 执行和内存开销。
    

### Series to Series

```
import pandas as pdfrom pyspark.sql.functions import pandas_udf@pandas_udf("double")def price_func(    price: pd.Series,    quantity: pd.Series) -> pd.Series:    return price * quantitydf = df.withColumn(    "total_price",    price_func(        col("price"),        col("quantity")    ))
```

这里输入和输出都是 Pandas Series。

### Iterator of Series to Iterator of Series

适合需要在多个 Batch 中复用初始化资源的场景，例如加载 ML Model。

```
from typing import Iteratorimport pandas as pd@pandas_udf("double")def iter_func(    batches: Iterator[pd.Series]) -> Iterator[pd.Series]:    model = load_model()    for batch in batches:        yield model.predict(batch)
```

注意：这个例子中的 `load_model()` 是示意函数，需要自己定义。

它的价值是：可以在一个 Iterator 调用过程中初始化模型一次，然后处理多个 Batch，而不是每个 Batch 都重新加载模型。

### applyInPandas

当需要对每个 Group 使用 Pandas 进行复杂计算时，可以使用 `applyInPandas()`。

```
def calc_avg(pdf: pd.DataFrame) -> pd.DataFrame:    return pd.DataFrame({        "customer": [pdf["customer"].iloc[0]],        "avg_amount": [pdf["amount"].mean()]    })result = (    df    .groupBy("customer")    .applyInPandas(        calc_avg,        schema="""            customer STRING,            avg_amount DOUBLE        """    ))
```

注意：

- 每个 Group 会被转换成 Pandas DataFrame。
    
- 需要先按 Group Key Shuffle。
    
- 单个 Group 太大时可能导致 Python Worker OOM。
    
- 不适合没有必要使用 Pandas 的简单聚合任务。
    

## 11.5 UDF vs Pandas UDF

|Feature|Python UDF|Pandas UDF|
|---|---|---|
|Processing|通常逐行调用|批次处理|
|Data Transfer|Pickle 等序列化机制|Apache Arrow|
|Performance|通常较慢|通常更快|
|Python Libraries|Supported|Supported|
|Memory Usage|逐行处理开销相对低|Batch 处理需要考虑内存|
|Optimization|Catalyst 对函数内部无法优化|同样无法优化函数内部逻辑|

## 11.6 Interview Questions

Q1. Why are Python UDFs slower than Spark built-in functions?

- JVM-Python serialization overhead.
    
- Python execution overhead.
    
- Limited Catalyst optimization of UDF internals.
    

Q2. What is Apache Arrow?

Arrow is a columnar in-memory data format that enables efficient data exchange between JVM and Python.

Q3. What is the difference between Python UDF and Pandas UDF?

普通 UDF 通常逐行执行，而 Pandas UDF 通过 Arrow 批次传输数据，进行向量化计算。

Q4. When would you use applyInPandas()?

当需要针对每个 Group 进行复杂 Pandas 操作，而内置 Spark SQL 函数无法方便实现时。

Q5. How would you optimize a slow Python UDF?

- Replace it with built-in Spark functions.
    
- Use Pandas UDF if appropriate.
    
- Avoid unnecessary Python-JVM data transfer.
    
- Reduce input columns and rows before UDF execution.
    

# Module 12. User Defined Table Functions (UDTF)

UDTF 与普通 UDF 的最大区别是：

- UDF：通常是一行输入对应一个值。
    
- UDTF：一行输入可以产生多行输出。
    

例如：

```
from pyspark.sql.functions import udtf@udtf(returnType="word STRING")class SplitWords:    def eval(self, text):        for word in text.split():            yield (word,)
```

调用：

```
df = spark.range(1).select(    SplitWords(lit("hello spark python")))
```

预期结果：

|word|
|---|
|hello|
|spark|
|python|

注意：

- UDTF 的输出 Schema 可以包含多个字段。
    
- `eval()` 使用 `yield` 返回多行结果。
    
- 需要注意具体 Spark 版本对 UDTF API 的支持。
    

## Interview Questions

Q1. What is the difference between UDF and UDTF?

UDF returns a scalar value, while UDTF can return multiple rows and columns.

Q2. When would you use UDTF instead of explode()?

当数据转换需要复杂的自定义 Python 逻辑，而不仅仅是展开已有 Array 时。

# Module 13. Writing Data & Output Modes

## 13.1 Write Data

写入 CSV：

```
df.write \    .format("csv") \    .mode("overwrite") \    .option("header", True) \    .save("/data/output/orders")
```

写入 Parquet：

```
df.write \    .format("parquet") \    .mode("overwrite") \    .save("/data/output/orders")
```

写入 Delta：

```
df.write \    .format("delta") \    .mode("overwrite") \    .save("/data/output/orders")
```

## 13.2 Save vs SaveAsTable

`save()`：

```
df.write.format("delta").save(    "/data/output/orders")
```

直接写入 Storage Path。

`saveAsTable()`：

```
df.write \    .format("delta") \    .mode("overwrite") \    .saveAsTable("catalog.schema.orders")
```

创建或写入 Catalog 中的表。

在 Databricks Unity Catalog 环境中，建议优先使用完整的三段式表名：

```
catalog.schema.table
```

## 13.3 Write Modes

|Mode|Behavior|
|---|---|
|overwrite|覆盖已有数据|
|append|追加数据|
|error / errorifexists|如果目标已存在则报错|
|ignore|如果目标已存在则不做操作|

例如：

```
df.write.mode("append").format("delta").save(path)
```

注意：Append 并不会自动检查业务主键，因此可能产生重复记录。

## 13.4 Partitioned Write

这是你原笔记中值得补充的重点。

```
df.write \    .format("parquet") \    .mode("overwrite") \    .partitionBy("year", "month") \    .save("/data/orders")
```

生成类似目录：

```
orders/
    year=2025/
        month=01/
        month=02/
    year=2026/
        month=01/
```

优势：

- Partition Pruning。
    
- 减少不必要的数据读取。
    
- 提高查询效率。
    

注意：

- 不要选择基数特别高的 Partition Column，例如 customer_id。
    
- Partition 过多可能造成大量小文件。
    
- Partitioning 应根据查询过滤条件和数据分布设计。
    

## 13.5 ReplaceWhere

Delta Lake 支持通过 `replaceWhere` 只覆盖满足指定条件的数据。

```
(    df.write    .format("delta")    .mode("overwrite")    .option(        "replaceWhere",        "order_date = '2025-01-02'"    )    .save(path))
```

这通常用于按日期重新处理某一天的数据。

例如：

```
Existing Data:
2025-01-01
2025-01-02
2025-01-03

Incoming Data:
2025-01-02

After ReplaceWhere:
2025-01-01
2025-01-02 (replaced)
2025-01-03
```

注意：

- `replaceWhere` 是 Delta Lake 的条件覆盖功能。
    
- 应启用和理解对应版本的约束检查，确保输入数据满足覆盖条件。
    
- 它不是普通 Parquet 写入的通用功能。
    

## 13.6 Small Files Problem

大量小文件会造成：

- File Listing Overhead。
    
- Metadata Overhead。
    
- 更多 Task 调度开销。
    
- 查询性能下降。
    

解决方式：

```
df.coalesce(10).write.format("parquet").save(path)
```

或者：

```
df.repartition(10).write.format("parquet").save(path)
```

区别：

- `coalesce()`：通常减少 Partition，不进行完整 Shuffle。
    
- `repartition()`：会 Shuffle，可以增加或减少 Partition，并且可以重新分布数据。
    

Databricks Delta Lake 还可以使用 OPTIMIZE 来合并小文件。

## 13.7 Interview Questions

Q1. What is the difference between append and overwrite?

Append adds new data, while overwrite replaces existing data according to the write operation and table configuration.

Q2. What is the difference between repartition() and coalesce()?

- Repartition performs a Shuffle.
    
- Coalesce usually reduces partitions without a full Shuffle.
    
- Repartition can increase or decrease partitions.
    

Q3. Why do we partition data when writing Parquet or Delta?

To enable partition pruning and reduce the amount of data scanned.

Q4. What is the small files problem?

Explain file listing, scheduling and metadata overhead.

Q5. How does replaceWhere work?

It replaces only data matching a specified predicate, useful for selective batch reprocessing.

# Module 14. Delta Lake Fundamentals

这是 Databricks / PySpark Data Engineer 面试的核心知识点。

## 14.1 What is Delta Lake?

Delta Lake 是建立在 Parquet 文件之上的 Storage Layer，增加了 Transaction Log 和事务管理能力。

基本结构：

```
Delta Table
    |
    |-----------------------|
    |                       |
Parquet Files           _delta_log/
    |                       |
Actual Data              Transaction Log
                            |
                            |-- Commit History
                            |-- Metadata
                            |-- Schema
                            |-- Add / Remove Files
                            |-- Protocol Information
```

Delta Lake 主要提供：

1. ACID Transactions。
    
2. Schema Enforcement。
    
3. Schema Evolution。
    
4. Time Travel。
    
5. MERGE / UPSERT。
    
6. DELETE / UPDATE。
    
7. Concurrent Writes。
    
8. Transactional Streaming and Batch Processing。
    

## 14.2 Delta Transaction Log

Delta Table 的核心是 `_delta_log`。

例如：

```
orders/
    part-00001.parquet
    part-00002.parquet

    _delta_log/
        00000000000000000000.json
        00000000000000000001.json
        00000000000000000002.json
        00000000000000000010.checkpoint.parquet
```

每次成功的 Transaction 都会产生新的 Version。

例如：

```
Version 0: Create Table
Version 1: Insert Data
Version 2: Update Data
Version 3: Delete Data
```

Delta Log 中主要记录：

- Add：新增数据文件。
    
- Remove：逻辑删除数据文件。
    
- Metadata：Schema、Partition 等表信息。
    
- Protocol：Reader 和 Writer Protocol。
    
- Commit Information：Transaction 相关信息。
    

重要理解：

Delta Lake 不会因为每次 Update 就直接修改原始 Parquet 文件。

例如：

```
Before:
part-001.parquet
part-002.parquet

After UPDATE:
part-001.parquet     (旧文件逻辑删除)
part-002.parquet     (旧文件逻辑删除)
part-003.parquet     (新文件)
part-004.parquet     (新文件)
```

旧文件不会马上被物理删除，因此 Delta Lake 可以实现 Time Travel。

## 14.3 ACID Transactions

|Property|Meaning|
|---|---|
|Atomicity|Transaction 要么全部成功，要么全部失败|
|Consistency|Transaction 后表必须满足一致性约束|
|Isolation|并发 Transaction 之间具有隔离保证|
|Durability|成功提交的数据可以持久保存|

Delta Lake 使用 Transaction Log 和 Optimistic Concurrency Control 等机制实现事务管理。

## 14.4 Schema Enforcement

Delta Lake 默认会检查写入数据的 Schema 是否与目标表兼容。

例如：

Target Table：

```
id INT
name STRING
```

Incoming Data：

```
id INT
name STRING
age INT
```

如果直接写入，可能出现 Schema Mismatch。

解决方法：

```
df.write \    .format("delta") \    .option("mergeSchema", "true") \    .mode("append") \    .save(path)
```

这可以启用 Schema Evolution，允许新增兼容字段。

注意：

- Schema Enforcement：控制写入的数据是否符合已有 Schema。
    
- Schema Evolution：允许 Schema 根据写入数据发生变化。
    
- `mergeSchema` 不是所有 Schema 变化都能自动处理，复杂类型变更仍然需要考虑兼容性。
    

## 14.5 Interview Questions

Q1. What is Delta Lake and why do we need it?

Parquet 本身不提供完整的事务管理。Delta Lake 通过 Transaction Log 增加 ACID、Time Travel、MERGE 和 Schema Enforcement 等能力。

Q2. What is the purpose of the Delta Transaction Log?

它负责记录数据文件变化、表 Metadata 和 Transaction Commit，使 Spark 能够重建当前表的 Snapshot。

Q3. Does Delta Lake update Parquet files in place?

通常不会。Update/Delete 通过新增文件和记录旧文件的逻辑删除来实现。

Q4. What is the difference between Schema Enforcement and Schema Evolution?

一个是限制写入 Schema，一个是允许 Schema 发生兼容性变化。

Q5. How does Delta Lake provide ACID transactions?

结合 Transaction Log、原子提交机制和 Optimistic Concurrency Control。

# Module 15. Delta Lake MERGE / UPSERT

这是数据仓库 Bronze → Silver → Gold 以及 Incremental ETL 非常重要的内容。

## 15.1 What is UPSERT?

UPSERT = UPDATE + INSERT。

逻辑：

```
Source Data
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

例如：

Target：

|id|name|
|---|---|
|1|Alice|
|2|Bob|

Source：

|id|name|
|---|---|
|2|Bob Updated|
|3|Charlie|

执行 UPSERT 后：

|id|name|
|---|---|
|1|Alice|
|2|Bob Updated|
|3|Charlie|

## 15.2 DeltaTable API

首先导入：

```
from delta.tables import DeltaTable
```

获取目标表：

```
target = DeltaTable.forPath(    spark,    "/data/delta/customers")
```

或者：

```
target = DeltaTable.forName(    spark,    "catalog.schema.customers")
```

执行 MERGE：

```
(    target.alias("t")    .merge(        source_df.alias("s"),        "t.customer_id = s.customer_id"    )    .whenMatchedUpdateAll()    .whenNotMatchedInsertAll()    .execute())
```

注意你原来的笔记有几个拼写错误：

- `alais` → `alias`
    
- `whenNotMathchedInsertAll` → `whenNotMatchedInsertAll`
    
- `execture` → `execute`
    

## 15.3 Conditional MERGE

如果只想更新部分字段：

```
(    target.alias("t")    .merge(        source_df.alias("s"),        "t.customer_id = s.customer_id"    )    .whenMatchedUpdate(        set={            "name": "s.name",            "email": "s.email"        }    )    .whenNotMatchedInsert(        values={            "customer_id": "s.customer_id",            "name": "s.name",            "email": "s.email"        }    )    .execute())
```

还可以加条件：

```
(    target.alias("t")    .merge(        source_df.alias("s"),        "t.customer_id = s.customer_id"    )    .whenMatchedUpdateAll(        condition="s.updated_at > t.updated_at"    )    .whenNotMatchedInsertAll()    .execute())
```

这样可以防止旧数据覆盖新数据。

## 15.4 MERGE SQL

```
MERGE INTO customers AS t
USING updates AS s
ON t.customer_id = s.customer_id

WHEN MATCHED THEN
    UPDATE SET
        t.name = s.name,
        t.email = s.email

WHEN NOT MATCHED THEN
    INSERT (
        customer_id,
        name,
        email
    )
    VALUES (
        s.customer_id,
        s.name,
        s.email
    );
```

## 15.5 WHEN NOT MATCHED BY SOURCE

这个条件经常用于 Source 和 Target 同步。

```
MERGE INTO customers AS t
USING updates AS s
ON t.customer_id = s.customer_id

WHEN MATCHED THEN
    UPDATE SET *

WHEN NOT MATCHED THEN
    INSERT *

WHEN NOT MATCHED BY SOURCE THEN
    DELETE;
```

含义：

- Matched：Source 和 Target 都存在。
    
- Not Matched：Source 有，Target 没有。
    
- Not Matched By Source：Target 有，Source 没有。
    

注意：`WHEN NOT MATCHED BY SOURCE` 会删除 Target 中所有不在 Source 中的匹配范围记录。如果 Source 只是增量数据，而不是完整快照，不能随意使用，否则会误删大量数据。

## 15.6 MERGE Important Considerations

Source Duplicate Keys

如果 Source 中同一个 Primary Key 出现多条记录，可能导致 MERGE 冲突或不确定的更新行为。

建议在 MERGE 前进行 Deduplication：

```
window_spec = (    Window    .partitionBy("customer_id")    .orderBy(col("updated_at").desc()))source_dedup = (    source_df    .withColumn(        "rn",        row_number().over(window_spec)    )    .filter(col("rn") == 1)    .drop("rn"))
```

NULL Join Keys

普通的：

```
t.id = s.id
```

对于两个 NULL 不会返回 TRUE。

如果需要 NULL-safe comparison，可以使用：

```
col("t.id").eqNullSafe(col("s.id"))
```

但通常业务主键应该尽量避免 NULL。

## 15.7 Interview Questions

Q1. What is the difference between INSERT, UPDATE and UPSERT?

解释三者的数据写入行为。

Q2. How does MERGE work in Delta Lake?

解释 Source 与 Target 根据 Join Condition 匹配，然后执行 Update、Insert 或 Delete。

Q3. What happens if the source contains duplicate keys?

可能导致 MERGE 产生冲突或不确定的更新，需要先 Deduplicate Source。

Q4. When would you use WHEN NOT MATCHED BY SOURCE?

当 Source 是完整数据快照，需要将 Target 同步到 Source 状态时。

Q5. How would you implement an incremental ETL pipeline using MERGE?

1. Read new source data。
    
2. Validate Schema。
    
3. Deduplicate source。
    
4. Apply transformations。
    
5. MERGE into target。
    
6. Validate output。
    
7. Track pipeline execution。
    

# Module 16. Delta Lake Time Travel & Restore

## 16.1 Describe History

查询 Delta Table 历史记录：

```
DESCRIBE HISTORY catalog.schema.orders;
```

可以看到：

- Version。
    
- Timestamp。
    
- Operation。
    
- Operation Parameters。
    
- User。
    
- Metrics。
    

例如：

```
version | operation
--------|----------
0       | CREATE TABLE
1       | WRITE
2       | MERGE
3       | DELETE
```

## 16.2 Time Travel by Version

读取指定 Version：

```
SELECT *
FROM catalog.schema.orders
VERSION AS OF 1;
```

## 16.3 Time Travel by Timestamp

```
SELECT *
FROM catalog.schema.orders
TIMESTAMP AS OF '2025-01-19T10:00:00';
```

也可以通过 PySpark：

```
df = (    spark.read    .format("delta")    .option("versionAsOf", 1)    .load(path))
```

## 16.4 Restore Table

恢复到指定 Version：

```
RESTORE TABLE catalog.schema.orders
TO VERSION AS OF 1;
```

重要区别：

Time Travel vs Restore

|Time Travel|Restore|
|---|---|
|读取历史 Snapshot|恢复表到历史状态|
|不改变当前表 Version|会产生新的 Commit|
|适合历史数据分析|适合误操作恢复|

例如：

```
Current Version: 5

RESTORE TO VERSION 2

New Version: 6

Version 6 contains the logical table state of Version 2.
```

Restore 不会把当前 Version 直接改成 2，而是创建一个新的 Version。

## 16.5 MERGE with Historical Source

可以把历史 Version 作为 Source：

```
MERGE INTO employees AS t

USING (
    SELECT *
    FROM employees VERSION AS OF 9
) AS s

ON t.employee_id = s.employee_id

WHEN MATCHED THEN
    UPDATE SET
        t.salary = s.salary;
```

## 16.6 VACUUM

你原来的笔记中还应该增加 VACUUM，因为它与 Time Travel 密切相关。

Delta Lake 默认通过保留旧文件支持历史 Snapshot。

但如果旧文件被 VACUUM 物理删除，对应历史 Version 就可能无法继续读取。

```
VACUUM catalog.schema.orders;
```

指定保留时间：

```
VACUUM catalog.schema.orders RETAIN 168 HOURS;
```

168 小时 = 7 天。

重要区别：

- Transaction Log 记录历史变化。
    
- Parquet 文件保存实际数据。
    
- VACUUM 清理不再被当前有效 Snapshot 引用的旧数据文件。
    
- 如果历史数据文件已删除，即使历史 Log 还存在，也不一定能够 Time Travel。
    

注意：不要随意关闭 Delta Lake 的 VACUUM 安全检查。过短的保留时间还可能影响长时间运行的 Reader 和并发操作。

## 16.7 Interview Questions

Q1. What is Delta Lake Time Travel?

可以通过 Version 或 Timestamp 查询历史 Snapshot。

Q2. What is the difference between Time Travel and RESTORE?

Time Travel 只读取历史数据，Restore 会创建新的 Commit 恢复历史状态。

Q3. What happens when VACUUM is executed?

删除不再被需要的旧数据文件，可能导致对应历史 Version 无法读取。

Q4. Why can't we always read every historical version?

因为旧文件可能已经被 VACUUM 清理，或者历史数据不再完整保留。

# Module 17. Idempotency & Reliable Data Pipelines

这是生产环境 Data Engineering 面试中非常重要的概念。

## 17.1 What is Idempotency?

幂等性：

Running the same operation multiple times produces the same final result as running it once.

例如：

第一次运行：

```
Order 1
Order 2
Order 3
```

第二次 Retry 后，目标表仍然应该是：

```
Order 1
Order 2
Order 3
```

而不是：

```
Order 1
Order 2
Order 3
Order 1
Order 2
Order 3
```

## 17.2 Why Idempotency Matters?

在生产环境中，Pipeline 可能遇到：

- Network Failure。
    
- Executor Failure。
    
- Job Timeout。
    
- Driver Failure。
    
- Retry。
    
- Partial Failure。
    

如果没有幂等性，重试可能产生重复数据。

## 17.3 Delta Transaction Identifiers

Delta Lake 可以通过：

- `txnAppId`
    
- `txnVersion`
    

帮助识别重复的写入事务。

例如：

```
(    df.write    .format("delta")    .mode("append")    .option("txnAppId", "daily_etl")    .option("txnVersion", 20260827)    .save(path))
```

解释：

|Parameter|Meaning|
|---|---|
|txnAppId|应用程序或写入任务的标识|
|txnVersion|该应用程序的事务版本号|

Delta Lake 可以识别相同 App ID 下已经成功提交过的事务版本，避免相同事务重复提交。

重要注意：

1. 相同逻辑事务重试时，需要使用相同的 `txnAppId` 和 `txnVersion`。
    
2. 新的逻辑批次需要新的 `txnVersion`。
    
3. `txnVersion` 应按应用程序维护正确的递增顺序。
    
4. 事务标识需要在重试过程中持久化，不能每次随机生成。
    
5. 这不是所有 Pipeline 级别的 exactly-once 保证，也不能代替业务主键去重。
    

## 17.4 Other Idempotency Strategies

### Strategy 1: MERGE

使用业务主键进行 Upsert：

```
(    target.alias("t")    .merge(        source.alias("s"),        "t.order_id = s.order_id"    )    .whenMatchedUpdateAll()    .whenNotMatchedInsertAll()    .execute())
```

重复运行相同的 Source，通常不会重复插入同一个 Order ID。

前提是 Source Key 唯一，并且更新逻辑是确定性的。

### Strategy 2: Overwrite Specific Partition

```
(    df.write    .format("delta")    .mode("overwrite")    .option(        "replaceWhere",        "order_date = '2026-08-27'"    )    .save(path))
```

重复覆盖同一天的数据，可以避免重复 Append。

前提是每次重跑都生成完整且一致的目标日期数据。

### Strategy 3: Deduplication

根据业务主键和更新时间进行去重。

```
window_spec = (    Window    .partitionBy("order_id")    .orderBy(col("updated_at").desc()))df = (    df    .withColumn(        "rn",        row_number().over(window_spec)    )    .filter(col("rn") == 1)    .drop("rn"))
```

## 17.5 Exactly-once vs At-least-once

|Guarantee|Meaning|
|---|---|
|At-most-once|数据最多处理一次，但可能丢失|
|At-least-once|确保至少处理一次，但可能重复|
|Exactly-once|每条逻辑数据只产生一次有效结果|

重要理解：

Exactly-once 通常是端到端语义，需要 Source、Processing、Sink 共同支持。

不能仅仅因为使用 Spark 或 Delta Lake，就认为整个 Pipeline 自动具备 Exactly-once。

## 17.6 Interview Questions

Q1. What is idempotency in data engineering?

重复执行相同任务不会造成额外的数据副作用，最终结果保持一致。

Q2. Why is idempotency important in ETL pipelines?

因为生产环境中存在失败和 Retry，需要防止重复写入。

Q3. How can you implement idempotency with Delta Lake?

- MERGE based on business keys.
    
- Overwrite specific partitions.
    
- Use transaction identifiers.
    
- Deduplicate source records.
    

Q4. What is the difference between at-least-once and exactly-once processing?

解释重试可能造成重复处理，以及端到端 Exactly-once 的要求。

# Module 18. PySpark Pandas API (pyspark.pandas)

这一部分需要和普通 PySpark DataFrame 区分。

## 18.1 What is Pandas API on Spark?

以前叫 Koalas，现在主要通过：

```
import pyspark.pandas as ps
```

提供类似 Pandas 的 API，但底层使用 Spark 执行分布式计算。

例如：

```
df = ps.read_csv("/data/orders.csv")
```

查看类型：

```
df.dtypes
```

选择列：

```
df[["customer_id", "price"]]
```

查看前几行：

```
df.head()
```

创建新列：

```
df["total_price"] = (    df["price"] * df["quantity"])
```

筛选：

```
df_filtered = df[df["price"] > 10]
```

排序：

```
df_sorted = df.sort_values(    "price",    ascending=False)
```

## 18.2 Pandas vs PySpark DataFrame vs Pandas API on Spark

|Feature|Pandas|PySpark DataFrame|Pandas API on Spark|
|---|---|---|---|
|Execution|Single machine|Distributed|Distributed|
|API|Pandas|Spark SQL|Pandas-like|
|Scalability|Limited by local resources|Cluster resources|Cluster resources|
|Optimization|Limited|Catalyst|Spark execution engine|
|Best Use|Small datasets|Large ETL|Pandas-style distributed workloads|

注意：

Pandas API on Spark 不是直接在本地运行的普通 Pandas DataFrame。

另外，`head()`、`to_pandas()` 等操作仍然需要注意数据量，尤其是将数据收集到 Driver 时。

## 18.3 Interview Questions

Q1. What is Pandas API on Spark?

它提供 Pandas 风格的 API，但使用 Spark 的分布式执行引擎。

Q2. What is the difference between Pandas and Pandas API on Spark?

Pandas 主要在单机内存中计算，而 Pandas API on Spark 使用分布式执行。

Q3. When would you use PySpark DataFrame instead of Pandas API on Spark?

当需要精细控制 Spark SQL 执行计划、Join、Partition、Shuffle 和性能优化时。

# Module 19. Spark Performance Optimization

这是荷兰 Data Engineer 面试中非常重要的模块。实际工作中，面试官通常不会只问 API，而是会给你一个运行很慢的 Spark Job，让你解释如何优化。

## 19.1 Partitioning

Spark 将数据拆分成多个 Partition，每个 Partition 可以分配给一个 Task 处理。

例如：

```
df.rdd.getNumPartitions()
```

查看 Partition 数量。

### Repartition

```
df_repartition = df.repartition(10)
```

按照某个 Key 重新分配：

```
df_repartition = df.repartition(    10,    col("customer_id"))
```

特点：

- 产生 Shuffle。
    
- 可以增加或减少 Partition。
    
- 可以按照指定 Key 重新分布数据。
    
- 适合需要重新均衡数据的场景。
    

### Coalesce

```
df_coalesce = df.coalesce(5)
```

特点：

- 通常不需要完整 Shuffle。
    
- 主要用于减少 Partition。
    
- 可能导致 Partition 数据分布不均。
    
- 不适合用来增加 Partition。
    

## 19.2 Cache vs Persist

如果一个 DataFrame 被多个 Action 重复使用，可以考虑 Cache。

```
df.cache()
```

等价于：

```
df.persist()
```

默认 Storage Level：

```
StorageLevel.MEMORY_AND_DISK_DESER
```

也可以指定：

```
df.persist(    StorageLevel.MEMORY_AND_DISK)
```

使用完之后释放：

```
df.unpersist()
```

例如：

```
df = spark.read.parquet("/data/orders")df.cache()df.count()df.filter(    col("amount") > 100).show()df.unpersist()
```

注意：

- `cache()` 本身是 Lazy 的。
    
- 数据在后续 Action 中才会被实际计算和缓存。
    
- Cache 不是永久存储。
    
- 如果 DataFrame 只使用一次，通常没有必要 Cache。
    

## 19.3 Catalyst Optimizer

Catalyst 是 Spark SQL 的查询优化器。

主要过程：

```
DataFrame API / SQL
         |
         v
Unresolved Logical Plan
         |
         v
Analyzed Logical Plan
         |
         v
Optimized Logical Plan
         |
         v
Physical Plan
         |
         v
Execution
```

常见优化：

- Predicate Pushdown。
    
- Projection Pruning。
    
- Constant Folding。
    
- Boolean Simplification。
    
- Join Reordering（取决于具体条件和统计信息）。
    

### Predicate Pushdown

尽可能在读取数据时过滤，而不是读取全部数据之后才过滤。

例如：

```
df = (    spark.read    .parquet("/data/orders")    .filter(col("amount") > 100))
```

Spark 可以尝试将过滤条件下推到数据源。

### Projection Pruning

只读取需要的列。

```
df = spark.read.parquet(    "/data/orders").select(    "customer_id",    "amount")
```

Parquet 支持列式读取，因此可以减少不必要的 I/O。

## 19.4 Adaptive Query Execution (AQE)

AQE 是 Spark SQL 的运行时优化机制。

开启：

```
spark.conf.set(    "spark.sql.adaptive.enabled",    "true")
```

AQE 可以根据实际运行时统计信息调整 Physical Plan。

常见功能：

1. Coalesce Shuffle Partitions

减少 Shuffle 后过多的小 Partition。

```
spark.conf.set(    "spark.sql.adaptive.coalescePartitions.enabled",    "true")
```

2. Dynamic Join Strategy

在满足条件时，将 Sort Merge Join 转换为 Broadcast Hash Join。

3. Skew Join Optimization

检测数据倾斜，并对倾斜 Partition 进行拆分处理。

注意：AQE 不能解决所有 Data Skew，也不能代替合理的数据建模。

## 19.5 Shuffle

Shuffle 是 Spark 性能问题的高频来源。

当数据需要跨 Partition 重新分布时，Spark 通常需要执行 Shuffle。

例如：

```
df.groupBy("customer_id").count()
```

执行逻辑：

```
Before Shuffle:

Partition 1: A, B, C
Partition 2: A, C, D
Partition 3: B, C, D

          |
          v

Shuffle by customer_id

          |
          v

Partition 1: A, A
Partition 2: B, B
Partition 3: C, C, C
Partition 4: D, D
```

Shuffle 可能涉及：

- Serialization。
    
- Network Transfer。
    
- Disk I/O。
    
- Sorting。
    
- Memory Pressure。
    

常见 Shuffle 操作：

- GroupBy。
    
- Distinct。
    
- OrderBy。
    
- Repartition。
    
- 部分 Join。
    
- Window Function。
    

## 19.6 Explain Execution Plan

通过 `explain()` 查看执行计划：

```
df.explain()
```

查看详细信息：

```
df.explain("formatted")
```

常见 Physical Operators：

|Operator|Meaning|
|---|---|
|FileScan|从文件读取数据|
|Filter|过滤数据|
|Project|选择或计算列|
|Exchange|数据重新分布，通常代表 Shuffle|
|HashAggregate|Hash 聚合|
|Sort|排序|
|SortMergeJoin|排序合并 Join|
|BroadcastHashJoin|广播 Hash Join|
|Window|Window Function|

面试时如果问你如何定位慢任务，可以回答：

1. Check Spark UI。
    
2. Identify the slowest Stage。
    
3. Inspect Shuffle Read / Write。
    
4. Check Task Duration Distribution。
    
5. Identify Data Skew。
    
6. Inspect Spill to Disk。
    
7. Review the Physical Execution Plan。
    
8. Optimize Partitioning, Join Strategy and Transformations。
    

## 19.7 Data Skew vs Small Files

两者都可能导致性能问题，但原因不同。

|Problem|Root Cause|Solution|
|---|---|---|
|Data Skew|数据集中在少数 Key|Salting、AQE、Broadcast|
|Small Files|文件数量太多、单文件太小|Compaction、OPTIMIZE|
|Too Many Partitions|Task 调度开销过高|Coalesce、AQE|
|Too Few Partitions|并行度不足|Repartition|
|Memory Spill|内存不足或数据处理过大|优化 Join、Partition、Aggregation|

## 19.8 Interview Questions

Q1. How would you optimize a slow PySpark job?

回答顺序：

- Inspect Spark UI.
    
- Identify slow stages and tasks.
    
- Check Shuffle and Data Skew.
    
- Use Predicate Pushdown and Column Pruning.
    
- Optimize Join Strategy.
    
- Tune Partition Count.
    
- Enable AQE.
    
- Cache reused DataFrames when appropriate.
    

Q2. What is the difference between repartition() and coalesce()?

重点解释 Shuffle 和 Partition 数量变化。

Q3. What is AQE?

Adaptive Query Execution uses runtime statistics to optimize the physical execution plan.

Q4. What is the difference between Cache and Persist?

`cache()` 是 `persist()` 的默认存储级别快捷方式。Persist 可以指定 Storage Level。

Q5. What is Exchange in the Spark Physical Plan?

Exchange 通常表示数据需要重新分布，常见于 Shuffle 操作。

Q6. How do you identify data skew in Spark UI?

观察同一 Stage 中 Tasks 的执行时间、Input Size、Shuffle Read 和 Spill 是否差异很大。

# Module 20. Spark Structured Streaming（建议补充）

你的原笔记已经涉及 Watermark，但没有完整覆盖 Streaming。对于荷兰 Data Engineer / Data Platform Engineer，这也是建议准备的知识点。

## 20.1 Batch vs Streaming

|Batch Processing|Streaming Processing|
|---|---|
|处理有限数据集|处理持续到达的数据|
|按批次执行|持续触发微批次或其他执行模式|
|适合历史数据处理|适合实时或近实时处理|
|例如每日 ETL|例如 Kafka Event Processing|

Spark Structured Streaming 默认采用 Micro-batch 执行模型。

## 20.2 Read Streaming Data

例如读取 Kafka：

```
df = (    spark.readStream    .format("kafka")    .option("kafka.bootstrap.servers", "host:9092")    .option("subscribe", "orders")    .load())
```

Kafka 常见字段：

- key
    
- value
    
- topic
    
- partition
    
- offset
    
- timestamp
    

## 20.3 Write Streaming Data

```
query = (    df.writeStream    .format("delta")    .outputMode("append")    .option("checkpointLocation", "/checkpoint/orders")    .start("/data/delta/orders"))
```

等待任务：

```
query.awaitTermination()
```

## 20.4 Output Modes

|Mode|Behavior|
|---|---|
|Append|只输出新增结果行|
|Update|输出本次发生变化的结果行|
|Complete|输出整个结果表|

不同操作和 Sink 支持的 Output Mode 有区别，并不是任意查询都支持三种模式。

## 20.5 Checkpoint

Checkpoint 用于保存 Streaming Query 的进度和状态信息。

通常包括：

- Source offsets。
    
- Committed batches。
    
- Stateful operator state。
    

作用：

- 支持故障恢复。
    
- 记录处理进度。
    
- 恢复 Stateful Processing 的状态。
    

注意：

Checkpoint 不等同于 Delta Transaction Log，两者解决的问题不同。

## 20.6 Event Time vs Processing Time

Event Time

事件在源系统实际发生的时间。

Processing Time

Spark 实际处理事件的时间。

例如：

```
Order Created: 10:00
Kafka Received: 10:05
Spark Processed: 10:10
```

- Event Time = 10:00
    
- Processing Time = 10:10
    

Event Time 对于处理 Late-arriving Data 非常重要。

## 20.7 Watermark

```
df = (    df    .withWatermark(        "event_time",        "10 minutes"    )    .groupBy(        window(            col("event_time"),            "5 minutes"        )    )    .count())
```

假设：

- Window Duration = 5 minutes。
    
- Watermark Delay = 10 minutes。
    

Watermark 允许一定程度的 Late-arriving Data，并帮助清理不再需要的 State。

## 20.8 Interview Questions

Q1. What is the difference between Event Time and Processing Time?

事件发生时间与系统实际处理时间。

Q2. What is Checkpointing in Structured Streaming?

用于保存处理进度和状态，以便失败后恢复。

Q3. What is Watermark?

用于管理 Late-arriving Data，并控制 State Retention。

Q4. What is the difference between Append, Update and Complete output modes?

解释输出新增记录、变化记录和完整结果表的区别。

Q5. How do you achieve fault tolerance in Structured Streaming?

使用 Checkpoint、可恢复的 Source、可靠 Sink，以及适当的幂等写入策略。

# Module 21. 生产环境 Data Engineering 场景题

前面是 API 和底层原理。真正的荷兰 Data Engineer 面试还经常会问下面这些综合场景题。

这些题目建议重点准备，因为它们可以同时考察 PySpark、SQL、Delta Lake 和 Pipeline Design。

## Scenario 1. How would you process a large dataset efficiently?

场景： 每天从 S3 / ADLS 读取 500GB Parquet 数据，清洗后写入 Delta Lake。

回答思路：

1. Read only required columns.
    
2. Apply filters early.
    
3. Define an explicit Schema if necessary.
    
4. Avoid unnecessary Python UDFs.
    
5. Check partition distribution.
    
6. Optimize joins using Broadcast Join when appropriate.
    
7. Handle Data Skew.
    
8. Write partitioned Delta tables where appropriate.
    
9. Monitor Spark UI and optimize small files.
    

## Scenario 2. How would you implement an incremental ETL pipeline?

场景： 每天只处理新增和更新的订单，而不是每天重新处理全部历史数据。

回答思路：

```
Source System
     |
     v
Extract Incremental Data
     |
     v
Bronze Delta Table
     |
     v
Data Validation
     |
     v
Deduplication
     |
     v
Business Transformations
     |
     v
MERGE INTO Silver Table
     |
     v
Data Quality Checks
```

关键知识：

- Watermark / Last Processed Timestamp。
    
- CDC。
    
- MERGE。
    
- Idempotency。
    
- Checkpoint。
    
- Retry。
    
- Schema Evolution。
    

## Scenario 3. How would you handle duplicate and late-arriving records?

回答思路：

1. Identify business keys.
    
2. Use `row_number()` to deduplicate.
    
3. Compare event timestamps or update timestamps.
    
4. Use Watermark for Streaming.
    
5. Use Delta MERGE for incremental updates.
    
6. Ensure retry operations are idempotent.
    

## Scenario 4. A Spark Job is suddenly running much slower. How would you investigate?

回答思路：

```
Spark UI
   |
   v
Identify Slow Stage
   |
   v
Check Task Distribution
   |
   |----------------------|
   |                      |
Data Skew             Shuffle Overhead
   |                      |
Salting / AQE         Partition Tuning
   |                      |
   |----------------------|
              |
              v
       Review Physical Plan
              |
              v
       Optimize and Retest
```

还要检查：

- Executor Memory。
    
- GC Time。
    
- Disk Spill。
    
- Input Size。
    
- Shuffle Read / Write。
    
- Failed Tasks。
    
- File Count。
    

## Scenario 5. How would you design a reliable data pipeline?

回答思路：

1. Use orchestration tools such as Airflow.
    
2. Make each pipeline task idempotent.
    
3. Implement retries with appropriate limits.
    
4. Use Delta Lake for transactional writes.
    
5. Implement schema validation.
    
6. Add data quality checks.
    
7. Monitor job execution and data freshness.
    
8. Implement logging and alerting.
    
9. Maintain audit metadata and lineage.
    

# Module 22. 面试复习优先级

如果目标是荷兰 Junior / Mid-level Data Engineer，我建议按下面的优先级准备，而不是平均分配时间。

## 第一优先级：必须掌握

核心面试知识

1. Transformation vs Action
    
2. Lazy Evaluation
    
3. Narrow vs Wide Transformation
    
4. Shuffle & Partition
    
5. DataFrame Transformations
    
6. GroupBy & Aggregation
    
7. Join Types & Broadcast Join
    
8. Window Functions
    
9. Delta Lake & MERGE
    
10. Idempotency
    
    ## 第二优先级：需要理解原理
    
    性能与生产实践
    
11. Catalyst Optimizer
    
12. AQE
    
13. Data Skew
    
14. Cache vs Persist
    
15. Schema Evolution
    
16. Time Travel & VACUUM
    
17. Checkpoint & Watermark
    
18. Pandas UDF & Apache Arrow
    
    ## 第三优先级：熟悉 API
    
    日常开发技能
    
19. CSV / JSON / Parquet / JDBC
    
20. Date & String Functions
    
21. Array / Struct / Map
    
22. Explode
    
23. Pivot
    
24. UDF / UDTF
    
25. Pandas API on Spark
    
26. File Read / Write Options
    

# Module 23. 你原始笔记中的重要错误汇总

下面这些代码问题建议单独记下来，避免面试时写错。

|原始写法|正确写法|
|---|---|
|`inferShcema`|`inferSchema`|
|`df.shema`|`df.schema` 或 `df.printSchema()`|
|`StructType` 导入路径|`pyspark.sql.types`|
|`properities`|`properties`|
|`date_diff()`|`datediff()`|
|`rowBetween()`|`rowsBetween()`|
|`Wondow`|`Window`|
|`arppox_conunt_distinct`|`approx_count_distinct`|
|`conuntDistinct`|`countDistinct`|
|`alais()`|`alias()`|
|`execture()`|`execute()`|
|`whenMatchedUpdateAll`|正确 API|
|`whenNotMathchedInsertAll`|`whenNotMatchedInsertAll`|
|`pathGlobalFilter`|`pathGlobFilter`|
|`udf(returnType=...)`|需要正确指定返回类型|
|`concat_ws()`|分隔符与各列分别传入|

另外三个特别重要的概念纠正：

1. `dropDuplicates()` 不等于保留最新记录。 必须使用 Window Function 明确排序。
    
2. Broadcast Join 不代表所有 Join 都没有 Shuffle。 它主要避免大表之间的 Shuffle。
    
3. Delta Transaction Log 不等于完整数据备份。 VACUUM 删除旧文件后，历史数据可能无法恢复。
    

# 最后：建议掌握的英文面试表达

面试时，不仅要知道 API，还要能够解释为什么使用它。

|Topic|建议掌握的关键词|
|---|---|
|Spark Execution|Lazy Evaluation, DAG, Job, Stage, Task|
|Transformations|Narrow, Wide, Shuffle|
|Optimization|Catalyst, AQE, Predicate Pushdown|
|Joins|Broadcast Hash Join, Sort Merge Join, Skew|
|Aggregation|Partial Aggregation, Shuffle|
|Window|Partition, Ordering, Frame|
|Delta Lake|ACID, Transaction Log, Snapshot|
|Incremental ETL|CDC, MERGE, UPSERT|
|Reliability|Idempotency, Retry, Checkpoint|
|Streaming|Event Time, Watermark, Exactly-once|

我建议你特别深入掌握三个实际工作场景：

- 如何处理每天新增、更新、删除的数据（CDC + MERGE）。
    
- 如何优化一个运行缓慢的 Spark Job（Spark UI + AQE + Shuffle）。
    
- 如何保证 ETL Pipeline 在失败重试后不会产生重复数据（Idempotency + Delta Lake）。
    

这三个场景能够把你记录的大部分 PySpark 知识点串联起来，也是从会写 PySpark 到能够独立设计 Data Engineering Pipeline 的重要区别。