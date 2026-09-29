SDC change data 

3 types of ingestions

1 managed ingestion, not available for many use cases
2 community ingestion, not SLA
3 Autoloader, write your own logics

用另一个路径记录schema的变化

可以显式选择：
```
df = df.select(
    "*",
    "_metadata"
)
```
`_metadata` 是一个 **STRUCT**。
里面可以有：
```
_metadata
├── file_path
├── file_name
├── file_size
├── file_modification_time
└── ...
```
```
order_raw = spark.readStream.format("cloudFiles")
			.option("cloudFiles.format", "csv")
			.option( "cloudFiles.schemaLocation","/Volumes/main/raw/metadata/schema/" ) 
			.option( "checkpointLocation","/Volumes/main/raw/metadata/checkpoint/" )
			

order_raw = order_raw
			.withColumn("ingest_timestamp", col("_metadata.file_modification_time"))
```

Using autoloader reading nested json

```
df = (
    spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .load("s3://bucket/raw/")
)
```

schema ：

```
id: long

customer: struct
    ├── name: string
    └── address: struct
          ├── city: string
          └── zip: string

items: array
    └── element: struct
          ├── product: string
          └── price: long
```

直接访问：

```
df.select(
    "id",
    "customer.name",
    "customer.address.city"
)
```

处理 array：

```
from pyspark.sql.functions import explode

df.select(
    "id",
    explode("items").alias("item")
)
```

# **STREAM READING WRITING**

AutoLoader,

Auto Loader：

```
**.format("cloudFiles")**
```

增加了：

- 增量发现文件
- 自动 Schema 推断
- Schema 演化（Schema Evolution）
- 高效文件发现

# 文件发现方式

Auto Loader 有两种模式：

## 1. Directory Listing

不断扫描目录：

```
landing/

    a.csv
    b.csv
    c.csv
```

适合：

- 文件数量不多
- 简单场景

---

## 2. File Notification（推荐）

例如 Azure：

```
ADLS

↓

Event Grid

↓

Queue

↓

Auto Loader
```

新文件上传：
默认自动创建event 和queue，根据输入的storage位置路径自动识别根据哪个queue去刷新

Azure Event Grid：

立即Auto Loader：直接读取。

不用反复扫描整个目录。

---

# Schema Evolution（很重要）

今天：

CSV：

```
flight_id
airport
```

明天：

CSV：

```
flight_id
airport
airline
```

Auto Loader：

Auto Loader 常见的 `schemaEvolutionMode` 有：

| Mode               | 新 column 出现时 | 结果                    |
| ------------------ | ------------ | --------------------- |
| `addNewColumns`    | ✅ 添加新 column | Schema 自动扩展           |
| `rescue`           | 不修改原 schema  | 新字段放进 `_rescued_data` |
| `failOnNewColumns` | ❌ 直接失败       | Streaming query 停止    |
| `none`             | ❌ 不处理        | 新字段通常被忽略              |


```
.option(
    "cloudFiles.schemaEvolutionMode",
    "addNewColumns"
)
```

新增：

```
airline
```

不用改代码。

Checkpoint location will store
checkpoint/ 
offsets/ (Like offset in Kafka)
commits/ (success operation until which batch)
metadata/ sources/ 
state/
## 窗口聚合

```
stream \
    .withWatermark("event_time", "10 minutes") \
    .groupBy(window(col("event_time"), "5 minutes")) \
    .count()
```

Spark 需要一直记住：

```
09:00-09:05
    count = 128

09:05-09:10
    count = 156

09:10-09:15
    count = 97
```

这些窗口统计都会放在 State Store。

等 Watermark 判断：

> 这个窗口已经不会再有迟到数据了

Spark 才会把对应 State 删除。

#### Bad Data
类型不匹配会放入rescue column

**坏的 record / malformed input → bad records**  
**能解析，但字段无法匹配 schema → `_rescued_data`**

```
query = (spark.readStream
.format("cloudFiles")
.option("cloudFiles.format", "json")
// Use this to limit the maxmium loading for each batch avoid the OOM
.option("cloudFiles.maxBytesPerTrigger", "1g")
// Path for quarantine
.option("badRecordsPath", "...")
.schema(schema)
.load(f"{bookstore.dataset_path}/kafka-raw")
.withColumn("timestamp", (F.col("timestamp")/1000).cast("timestamp"))
.withColumn("year_month", F.date_format("timestamp", "yyyy-MM"))

.writeStream
.option("checkpointLocation", f"{bookstore.checkpoint_path}/bronze")
// Auto evolve for updated shceme
.option("mergeSchema", True)
.partitionBy("topic", "year_month")
// support schema evolving
.option("mergeSchema", "true")
// Set trigger interval
.trigger(availableNow=True)/.trigger(processingTime="30 seconds")
.table("bronze"))
// 让当前程序一直等待，直到 Streaming Query 停止。- `start()`：**启动**一个后台运行的 Streaming Query，并立即返回。
- `awaitTermination()`：**阻塞当前主线程**，让 Spark Application 不会因为主线程执行完而退出。
query.awaitTermination()

```

```sql
""" applied all merge operation in this session """
SET spark.databricks.delta.schema.automerge.enable = true
```
or
```python
.option("merge_schema", true)

```
Rename Column
```SQL
ALTER TABLE products RENAME price_new To price
```
Generated column
```SQL
CREATE OR REPLACE TABLE product(
	product_id INT
	price DOUBLE
	price_taxed DOUBLE GENERATED ALWAYS AS (ROUND(price*1.2, 2))
)

```
**Schema enforcement** is your default safety net. It rejects writes with extra columns or type mismatches — before any damage reaches the table.

To add new columns during a write — use **`mergeSchema`**. Per-write, intentional, production-safe.  
Existing rows get **NULL** in the new column.  
`mergeSchema` cannot rename — only add. For true renames, use **`ALTER TABLE RENAME COLUMN`**.

**`ALTER TABLE`** operations are metadata-only. ADD, RENAME, and DROP columns update the schema in the transaction log without rewriting any Parquet files. **Instant on any table size.**  
DROP removes the column from queries immediately but does not erase the physical data — **OPTIMIZE and VACUUM** complete the erasure.

**Generated columns** are computed at write time and stored physically. Define them with `GENERATED ALWAYS AS (expression)`.  
Delta populates them automatically on every **INSERT or UPDATE**. Remember, they are physically stored.

Type widening
Only work in the type family
Byte -> Short -> Int -> Long
Float -> Double
can broaden the type of the data
```python
ALTER TABLE sensor_reading
SET TABLEPROPERTIES ('delta.enableTypeWidening' = true)
```

VARIANT Data Type (store json type data), is a binary type

||STRUCT|VARIANT|
|---|---|---|
|Schema|**固定**|**灵活**|
|数据类型|明确定义|可以是不同类型|
|类似|Object / Struct|JSON|
|字段是否提前定义|是|不一定|
|适合|已知、稳定的数据结构|半结构化 / schema 不稳定的数据|

```SQL
INSERT INTO iot_event VALUES
(1, PARSE_JSON('{"sensor_id":"5001","temp": 24.2}'))
```
query with :operator and ::cast
Have to cast while getting data or filtering
```SQL
SELECT 
payload:sensor_id::STRING AS sensor_id
payload:temp::DOUBLE AS temp
FROM iot_event

```
### foreachBatch

如果想：

- MERGE
- UPSERT
- 调用 JDBC
- 调用 REST API
- 写多个表

foreachBatch function should have batch and batchID param

```
from delta.tables import DeltaTable

def mergeToSilver(batchDF, batchId):

    deltaTable = DeltaTable.forName(
        spark,
        "silver.flight"
    )

    (
        deltaTable.alias("t")
        .merge(
            batchDF.alias("s"),
            "t.flight_id=s.flight_id"
        )
        .whenNotMatchedInsertAll()
        .execute()
    )
```

然后：

```
stream.writeStream \
      .foreachBatch(mergeToSilver) \
      .start()
```
```
# Quality Enforcement

Will verify all data in the table confirm the rules before adding rule.
```
#### CONSTRAINS

For streaming tables, we can not alter the constrains after creation。所以要在定义schema时加入

```SQL
ALTER TABLE order ADD CONSTRAINT timestamp_inrange CHECK(order_timestamp >= '2020-01-01')
```

Primary KEY Constraint
创建一个 constraint（约束），并把这个约束命名为 `pk_station
PRIMARY KEY constraint 会保证主键值唯一且非 NULL`
```SQL
CONSTRAINT pk_aqi PRIMARY KEY (station_id, measurement_time)
```

ALTER TABLE order DROP CONSTRAIN timestamp_inrange



SCD (Slowly Changing Dimensions)
Type 0 :No changed allow
Type 1: Overwrite
Type 2: New record with old ones

For SCD2:
Add current, effective date, end date in the table

This will increase cost and latency so not suitable for time travel.

# CDC 

Delta MERGE fail when CDC contains multiple updates for the same key
can be solved with rank over window to select the latest one to merge

When running VACCUM, CDF data also deleted
CDF

Use CDF when small changes, and not only appending operations.
```
CREATE TABLE books (
    ...
)
TBLPROPERTIES (
    delta.enableChangeDataFeed = true
);
```

已有表：

```
ALTER TABLE books
SET TBLPROPERTIES (
    delta.enableChangeDataFeed = true
);
```
读
```
cdf_df = (spark.readStream
.format("delta")
.option("readChangeFeed", True)
.option("startingVersion", 2)
.table("customers_silver"))
```

Stream Static Join
only stream table trigger the join processing. 
With limitation that batch added data will not be joined for previous streaming data

The F.window function add a new time interval col for the table, and the agg functions can be operated based on the interval.
```
query = (spark.readStream
.table("books_sales")
.withWatermark("order_timestamp", "10 minutes")
.groupBy(F.window("order_timestamp", "5 minutes").alias("time"),"author")
.agg(
F.count("order_id").alias("orders_count"),
F.avg("quantity").alias ("avg_quantity"))
.writeStream
.option("checkpointLocation", f"{bookstore.checkpoint_path}/authors_stats")
.trigger(availableNow=True)
.table("authors_stats")
)
```
# Partitioning
Can not change after creating table, 
Partition should be 1 GB above
If is less than 1 gb, it is over-partitioning
```
CREATE TABLE table ... PARTITION BY(col_name)
```

## ignoreDeletes=true

```
.option("ignoreChanges","true")
```

例如：

```
spark.readStream \
    .option("ignoreDeletes", "true") \
    .table("users")
```

意思就是：

> **如果发现有人 DELETE/UPDATE 了数据，我就当没看见，继续处理后面的新增数据。**

> **`spark.readStream.table()` 默认假设源 Delta Table 是 append-only（只有新增，没有 UPDATE/DELETE）。**

如果这个假设被打破，Streaming 通常会报错。

Z-Order Indexing
for high cardinality cols
```
OPTIMIZE table
ZORDER BY colname
```

Liquid Clustering

所以 **Liquid Clustering 最大的优势不是“不重写”，而是“不需要一次性重写整张表”**。它把这个昂贵的过程拆分到后续维护过程中完成，这也是它比传统 `PARTITION + ZORDER` 更适合长期运行的大型数据平台的原因。
Can not be used with ZORDER and partitioning
increment optimize，without re-writing existing data
```
CREATE TABLE table ....
CLUSTER BY (col1, col2)

ALTER TABLE table
CLUSTER BY (<>)

CLUSTER BY AUTO
```

Predictive optimization

Predictive Optimization automatically maintains Delta tables by running optimization tasks such as OPTIMIZE （compact small files）, VACUUM, statistics collection, and Liquid Clustering maintenance without requiring manual scheduling.

Predictive Optimization 使用 Databricks 平台管理的后台计算资源（serverless maintenance compute），而不是你的 Job Cluster 或 All-Purpose Cluster。

Optimize:

The default optimize file size target is 1GB. For latter on operation

Optimize writes: attempt to write 128MB files, for on-write compaction
Auto compaction:
After writing, check if can further compacted. If yes, optimize the job to target size of 128MB
```
CREATE TABLE 
TBLPROPERITIES (delta.autoOptimize.optimizeWrite = true,
				delta.autoOptimize.autoCompact = true
)
```

Small files can better data skipping, help minimize rewrites during merge and delete.
For frequent merge operation, the files are smaller than 128MB

-**Deletion Vector**：逻辑删除，记录哪些行不可见。For fast delete or update. 
 **OPTIMIZE**：Will operate on the tables can do exact operation

Vaccum
```SQL
VACUUM my_table RETAIN 720 HOURS

"""or set for table"""

ALERT TABLE t SET TABLEPROPERITIES (
	delta.deleteFileRetentionDuration = 'interval N days'
)

CREATE TABLE orders (
    order_id BIGINT,
    amount DOUBLE,
    status STRING,

    CONSTRAINT valid_amount
    CHECK (amount > 0),

    CONSTRAINT valid_status
    CHECK (status IN ('pending', 'completed', 'cancelled'))
);

```

Identity Columns
让 Delta 自动帮你生成 ID。GENERATED ALWAYS规定不能自己制定surrogate key
```SQL
CREATE TABLE customer (
    customer_sk BIGINT
        GENERATED ALWAYS AS IDENTITY,

    customer_name STRING
);
```
GENERATED BY DEFAULT AS IDENTITY 不提供自动产生，可以提供，用于migration保留原有
```SQL
customer_sk BIGINT
    GENERATED BY DEFAULT AS IDENTITY
```
# Transaction log location files

stored in the 
_delta_log directory within table location

Auto create parquet checkpoint every 10 commit 
Also statistics on first 32 columns of the table
Nested fields also count

For free text fields, remove them from the first 32 columns

Will not be removed by vaccum
Will cleaned by Databricks automatically by default 30 days. So can time travel up to 30 days.
delta.logRetentionDuration = ...

# Declarative Pipeline

CREATE OR REFRESH OBJECT
OBJECT can be streaming/ Used for data ingestion from streaming sources(append only)
materialized view/ Used for precomputing complex BI queries or for data ingestion from no stream sources
temporary view
```
@dp.table(
	name = "bronze",
	partition_cols=["topic", "year_month"],
	table_properties={
	"delta.appendOnly": "true",
	"pipelines.reset.allowed": "false",
	}
)

def process_bronze():

schema = "key BINARY, value BINARY, topic STRING, partition LONG, offset LONG, timestamp LONG"

bronze_df = (spark.readStream
	.format("cloudFiles")
	.option("cloudFiles.format", "json")
	.schema(schema)
	.load(f"{dataset_path}/kafka-raw-etl")
	.withColumn("timestamp", (F.col("timestamp")/1000).cast("timestamp"))
	.withColumn("year_month", F.date_format("timestamp", "yyyy-MM"))
)
return bronze_df
```
**Development Mode：**

> **快速调试，复用 Cluster。**

**Production Mode：**

> **正式运行，每次使用新的 Job Cluster，保证稳定性和一致性。**

Data quality expectations
- **Constraint CHECK**：**强约束**，违反就让数据写入失败。
- **Constraint EXPECT**：**数据质量规则**，可以记录、丢弃或失败，由你决定。（Can only used in DLT）

```
CONSTRAIN name EXPECT [ON VIOLATION ACTION]

@dlt.expect("valid_age", "age >= 0")
@dlt.expect_or_drop("valid_age", "age >= 0")
@dlt.expect_or_fail("valid_age", "age >= 0")

constrains = {"name1", 'name2', 'name3'}
@dlt.expect_all("valid_age", "age >= 0")
@dlt.expect_all_or_drop("valid_age", "age >= 0")
@dlt.expect_all_or_fail("valid_age", "age >= 0")

Add invalid record in the quarantine table

@dp.table
@dp.expect_or_drop("valid_quantity", "quantity > 0")
def orders_silver():
	orders_df = process_orders()
	return orders_df

@dp.table
@dp.expect_or_drop("valid_quantity", "quantity <= 0")
def orders_quarantine():
	orders_df = process_orders()
	return orders_df
```

SDP (new version for DLT)

FLow types:
1 Append AutoLoader/Kafka/Streaming table -> Streaming Table/sink
2 Auto CDC Streaming table -> Streaming Table/sink
3 MV Flow Delta

Managed Table -> materialized view(usually for gold layer)
Kafka event-hub external delta -> sink dp.createsink() +dp.append_flow

Streaming table :
	new row arriving incrementally -> @dp.table (implicit append flow)
	 new rows arriving multiple sources -> @dp.create_streaming_table() + @dp.append_flow()
	 CDC events -> dp.create_streaming_table() + dp.create_auto_cdc_flow()
materialized_view -> @dp.materialized_view 

```python
@dp.table(name = "bronze")
@dp.expect("op_is_valid", "op IN ('c','u','d')")
@dp.expect_or_drop("ts_is_not_null", "ts_ms IS NOT NULL")
def bronze_layer_sdc():
	return (
		spark.readStream
			.format('cloudFiles')
			.option('cloudFiles.format', 'csv')
			.option('inferSchema', 'true')
			.option('header', 'true')
			.load(PATH)
			.select("*",
			col("_metadata.file_modification_time").alias("ingest_time")
	)

```

```python
dp.create_streaming_table(
	name = "silver_oders",
	expect_all_or_drop {
		"valid_order_id": "order_id IS NOT NULL",
		"valid_amounr": "amount > 0"
	}
)

dp.create_auto_cdc_flow(
	target = "silver_orders",
	source = "bronze"
	key = ["order_id"]
	sequence_by = "ts_ms"
	apply_as_delete = "op = 'd' or __deleted = 'true'"
	store_as_scd_type = "2"
	except_column_list = ["_ingest_time", "__deleted", "op", "ts_ms"]
)

 

```

Jobs

Tasks: pipeline/ python/ notebook

Trigger: schedule/ file arrival/ delta table change/manual API
