# Databricks Data Engineer 面试知识体系整理（进阶篇）

这份笔记涉及的内容已经不只是 Databricks 基础架构，而是进入了 Data Engineer 日常工作中非常重要的几个领域：

- Streaming Data Ingestion
    
- Auto Loader & Schema Evolution
    
- Structured Streaming
    
- Delta Lake & Transaction Log
    
- Data Quality & Constraints
    
- CDC & SCD
    
- Table Optimization & Performance
    
- Declarative Pipelines（原 DLT，现 Lakeflow Spark Declarative Pipelines）
    
- Databricks Jobs、Monitoring & Lineage
    

我会按照数据从进入系统、处理、质量检查、增量更新、存储优化，到生产调度和监控的顺序重新组织，而不是简单按照原笔记顺序排列。

同时，对原笔记中容易产生误解的内容进行修正。尤其需要注意：

1. Auto Loader 的 Schema Evolution 和 Delta Table 的 Schema Evolution 是两个不同机制。
    
2. Checkpoint 不只是 Offset，还负责 Streaming Query 的状态恢复。
    
3. Delta Lake 的 Primary Key Constraint 默认不是强制唯一性约束。
    
4. VACUUM、Transaction Log Retention 和 Time Travel 的关系需要分开理解。
    
5. `ignoreDeletes`、`ignoreChanges` 和 Change Data Feed 不能混为一谈。
    
6. Liquid Clustering 并不是完全不重写数据。
    
7. Declarative Pipelines 中的 Streaming Table、Materialized View 和 CDC Flow 有不同用途。
    
8. SCD Type 2 和 Time Travel 解决的是不同层面的问题。
    

以下以你的原始笔记为基础，保留重要代码和知识点，并补充荷兰 Data Engineer / Data Platform Engineer 面试中常见的场景题及参考回答。

# Module 1. Data Ingestion & Auto Loader

Bronze Layer / Ingestion

## 1.1 数据摄取的三种方式

你原笔记记录了三种 Ingestion：

|类型|特点|适用场景|
|---|---|---|
|Managed Ingestion|Databricks 管理连接和摄取流程|官方支持的数据库和数据源|
|Community Connectors|社区提供的 Connector|官方 Managed Connector 未覆盖的场景|
|Auto Loader|自己编写数据处理逻辑|Cloud Object Storage 中的文件摄取|

需要补充：

- Managed Ingestion 不代表支持所有数据库、所有认证方式和所有 CDC 场景。
    
- Community Connector 不代表一定没有 SLA，而是不能默认认为 Databricks 官方为其提供完整的产品级支持和 SLA。
    
- Auto Loader 主要用于文件摄取，不是数据库 CDC Connector。
    

典型架构：

```
Source
  |
  ├── Database
  |      └── Managed Connector / CDC
  |
  ├── Kafka / Event Hubs
  |      └── Spark Structured Streaming
  |
  └── Cloud Object Storage
         └── Auto Loader
                |
                v
           Bronze Delta Table
```

## 1.2 Auto Loader 是什么？

Auto Loader 是 Databricks 用于增量摄取 Cloud Object Storage 文件的机制。

它使用：

```
.format("cloudFiles")
```

主要能力：

1. Incremental File Discovery：增量发现新文件。
    
2. Schema Inference：自动推断 Schema。
    
3. Schema Evolution：检测新增字段。
    
4. Scalable File Discovery：支持大规模文件发现。
    
5. Exactly-once processing：配合正确的 Checkpoint 和目标写入机制，实现可靠的文件处理语义。
    

例如：

```
from pyspark.sql.functions import colorder_raw = (    spark.readStream        .format("cloudFiles")        .option("cloudFiles.format", "csv")        .option(            "cloudFiles.schemaLocation",            "/Volumes/main/raw/metadata/schema/"        )        .load("s3://company-bucket/orders/"))order_raw = order_raw.withColumn(    "ingest_timestamp",    col("_metadata.file_modification_time"))
```

### Schema Location 和 Checkpoint Location 的区别

这是一个很重要的面试考点。

|Location|保存内容|作用|
|---|---|---|
|schemaLocation|Schema 信息和 Schema Evolution 历史|记录数据结构变化|
|checkpointLocation|Streaming 处理进度、状态和提交信息|恢复 Streaming Query|

注意：`schemaLocation` 和 `checkpointLocation` 不是同一个目录，也不能互相替代。

如果使用 `writeStream` 写入 Delta Table：

```
query = (    order_raw.writeStream        .format("delta")        .option(            "checkpointLocation",            "/Volumes/main/raw/metadata/checkpoint/"        )        .toTable("main.bronze.orders"))
```

在实际生产中，建议将 Schema 和 Checkpoint 分别放在持久化的、受控的存储路径中。

## 1.3 `_metadata`：文件元数据

Auto Loader 和 Spark 文件读取可以提供 `_metadata` 列。

它是一个 STRUCT，包含文件层面的元信息。

```
_metadata
    |
    ├── file_path
    ├── file_name
    ├── file_size
    ├── file_modification_time
    └── ...
```

使用：

```
df = (    spark.readStream        .format("cloudFiles")        .option("cloudFiles.format", "csv")        .load("s3://bucket/raw/"))df = df.select(    "*",    "_metadata")
```

也可以选择其中的具体字段：

```
df = df.withColumn(    "file_path",    col("_metadata.file_path"))
```

生产建议： 如果只需要文件名、路径、修改时间等信息，优先选择需要的字段，不要无意义地将整个 `_metadata` STRUCT 都写入目标表。

注意：`file_modification_time` 是文件修改时间，不一定等于数据真正产生的时间，也不等于 Databricks 摄取时间。

如果要记录真正的 Ingestion Timestamp：

```
from pyspark.sql.functions import current_timestampdf = df.withColumn(    "ingest_timestamp",    current_timestamp())
```

这两个字段最好分开保存：

|Column|Meaning|
|---|---|
|file_modification_time|Source file modification time|
|ingest_timestamp|Databricks ingestion processing time|
|event_timestamp|Business event occurrence time|

## 1.4 Auto Loader 的两种 File Discovery Mode

Auto Loader 有两种文件发现模式。

![Auto Loader 自动加载器与文件事件概述 - Azure Databricks | Microsoft Learn](https://images.openai.com/static-rsc-4/YqE8f6QrfU3poYtuhvJcbLEo8sJivZxm6FF-r42IDVawt2ZYP7frxnh2z290572xjoaWJxXW4pfPQnx5JoBwNKbM_XVUDgM8UVbK2HtKSTe-IfSy8Ov3ehotcX7xYJkWgVl1eAXJp_DkIPX92zucbgMOZNVh2V2_XDT4MrJVRnQ?purpose=inline)

[learn.microsoft.com](https://learn.microsoft.com/zh-cn/azure/databricks/ingestion/cloud-object-storage/auto-loader/file-events-explained)

A. Directory Listing

```
Landing Directory
      |
      v
Auto Loader
      |
      v
Scan Directory
      |
      v
Identify New Files
```

特点：

- 定期扫描目录。
    
- 对文件数量较少的场景比较简单。
    
- 需要考虑目录扫描成本。
    

B. File Notification / Managed File Events

```
Cloud Storage
      |
      v
Storage Events
      |
      v
Event Grid / Queue
      |
      v
Auto Loader
      |
      v
New Files
```

例如 Azure：

```
ADLS Gen2
   |
   v
Azure Event Grid
   |
   v
Queue
   |
   v
Auto Loader
```

特点：

- 通过事件通知发现文件。
    
- 不需要反复扫描整个目录。
    
- 更适合大规模文件摄取。
    

注意两个细节：

1. File Notification 模式与 Managed File Events 不是完全相同的概念，具体部署可以涉及 Databricks 管理的事件基础设施。
    
2. 不要理解成上传文件后，Event Grid 直接把文件内容推送到 Spark。事件主要用于通知有新文件，Auto Loader 仍然需要从 Cloud Storage 读取文件。
    

## 1.5 Schema Evolution

假设今天的 CSV：

```
flight_id,airport
1,AMS
2,RTM
```

明天增加了一个字段：

```
flight_id,airport,airline
1,AMS,KLM
2,RTM,Transavia
```

Auto Loader 可以检测新增字段。

常见模式：

|Mode|新字段出现时|处理方式|
|---|---|---|
|`addNewColumns`|检测到新字段|更新 Schema，并触发流重启|
|`addNewColumnsWithTypeWidening`|新字段或允许的类型拓宽|更新 Schema，并触发流重启|
|`rescue`|不更新原 Schema|将未匹配数据放入 `_rescued_data`|
|`failOnNewColumns`|检测到新字段|失败并停止 Streaming Query|
|`none`|不自动更新 Schema|不更新 Schema；未匹配数据可按配置被丢弃或救援|

示例：

```
.option(    "cloudFiles.schemaEvolutionMode",    "addNewColumns")
```

重要： `addNewColumns` 检测到新字段后，当前 Streaming Query 会失败，需要重新启动，之后使用更新后的 Schema 继续处理。

因此：

```
New Column
    |
    v
Schema Change Detected
    |
    v
Update Schema
    |
    v
Stream Restart
    |
    v
Continue Processing
```

### `_rescued_data`

当数据可以被读取，但字段无法匹配当前 Schema 时，可以将这些数据保存在 `_rescued_data` 中。

例如：

预期：

```
{
  "id": 1,
  "age": 25
}
```

实际：

```
{
  "id": 1,
  "age": "unknown",
  "country": "Netherlands"
}
```

其中：

- `country`：Schema 中不存在的字段。
    
- `age`：类型不符合预期。
    

根据具体格式和配置，这些无法正常匹配的字段可以进入 `_rescued_data`。

需要区分：

|问题|处理方式|
|---|---|
|新字段不在 Schema|`_rescued_data` 或 Schema Evolution|
|类型不匹配|`_rescued_data` 或解析错误|
|完全无法解析的 malformed record|Corrupt Record / Bad Records 处理|

`badRecordsPath` 和 `_rescued_data` 不是同一个东西。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on Google Cloud

+1

## 1.6 Auto Loader 读取 Nested JSON

例如 JSON：

```
{
  "id": 1,
  "customer": {
    "name": "Alice",
    "address": {
      "city": "Amsterdam",
      "zip": "1000"
    }
  },
  "items": [
    {
      "product": "Laptop",
      "price": 1200
    },
    {
      "product": "Mouse",
      "price": 30
    }
  ]
}
```

对应 Schema：

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

直接访问 STRUCT：

```
df.select(    "id",    "customer.name",    "customer.address.city")
```

处理 ARRAY：

```
from pyspark.sql.functions import explodedf.select(    "id",    explode("items").alias("item"))
```

结果：

|id|item.product|item.price|
|---|---|---|
|1|Laptop|1200|
|1|Mouse|30|

其他有用的函数：

```
from pyspark.sql.functions import (    explode,    explode_outer,    posexplode,    col)# 保留原始数组为空的记录df.select(    "id",    explode_outer("items").alias("item"))# 同时获取数组下标df.select(    "id",    posexplode("items").alias("position", "item"))
```

## Module 1 — 面试问题

Q1. What is the difference between Auto Loader and Spark Structured Streaming?

参考回答：

Spark Structured Streaming is the general streaming processing engine. Auto Loader is a specialized file ingestion mechanism built on top of Structured Streaming. It provides incremental file discovery, schema inference and schema evolution for cloud object storage.

Q2. What is the difference between schemaLocation and checkpointLocation?

Schema Location stores schema information and evolution history, while Checkpoint Location stores streaming progress, offsets and state information required for recovery.

Q3. What happens when a new column arrives with `addNewColumns`?

Auto Loader detects the new column, updates the schema and stops the current streaming query. The query must be restarted to continue processing with the updated schema.

Q4. What is the difference between `_rescued_data` and bad records?

Rescued data contains fields that cannot be matched to the expected schema, whereas malformed records are records that cannot be properly parsed according to the input format.

Q5. When would you choose File Notification over Directory Listing?

I would choose File Notification for large-scale ingestion where directory scanning becomes expensive. It allows Auto Loader to discover files through cloud storage events.

Q6. Why would you store file modification time and ingestion time separately?

File modification time describes the source file, while ingestion time records when the platform processed the data. They help investigate data latency, late arrivals and pipeline delays.

# Module 2. Structured Streaming & Stream Processing

Streaming Processing

这一模块主要回答：

- Streaming 是如何工作的？
    
- 如何保证数据不丢失、不重复？
    
- 如何处理迟到数据？
    
- 如何管理 Streaming State？
    
- 如何控制数据处理频率？
    

## 2.1 Structured Streaming 的基本工作模式

Structured Streaming 是 Spark 的流处理引擎。

它可以将不断到来的数据视为一个持续增长的表（Unbounded Table）。

主要有两种处理模式：

Micro-batch Processing

将数据分成多个小批次处理。

```
Incoming Data
      |
      v
  Micro-batch 1
      |
      v
  Micro-batch 2
      |
      v
  Micro-batch 3
      |
      v
  Delta Table
```

例如：

```
query = (    df.writeStream        .format("delta")        .option("checkpointLocation", checkpoint_path)        .trigger(processingTime="30 seconds")        .toTable("bronze.orders"))
```

每隔 30 秒尝试启动一个处理批次。

Continuous Processing

一种低延迟流处理模式。

不过需要注意：Spark Structured Streaming 的 Continuous Processing Mode 和 Databricks 中的 Continuous Pipeline Mode 不是一个概念。

在实际 Databricks 数据工程中，Micro-batch 是更常见的处理方式。

## 2.2 Trigger 的区别

|Trigger|作用|
|---|---|
|`processingTime="30 seconds"`|按时间间隔处理数据|
|`availableNow=True`|将当前可用数据处理完毕后停止|
|`once=True`|执行一个批次后停止，属于旧式触发模式|
|Continuous|持续运行，取决于具体 Pipeline / Streaming API|

示例：

```
.trigger(processingTime="30 seconds")
```

适合持续运行的 Streaming Job。

```
.trigger(availableNow=True)
```

适合 Scheduled Incremental Processing。

例如：

```
00:00 Job starts
00:00 - 00:05 Process available files
00:05 - 00:10 Process newly available files
00:10 No more available data
00:10 Job finishes
```

下次 Job 运行时，通过 Checkpoint 继续处理尚未处理的数据。

面试场景： 如果业务要求每天处理一次新到达的文件，但不需要 24 小时保持 Cluster 运行，可以考虑 Auto Loader + AvailableNow。

## 2.3 Checkpoint：Streaming 的恢复机制

Checkpoint 是 Structured Streaming 非常重要的机制。

你原笔记中记录：

```
checkpoint/
    |
    ├── offsets/
    ├── commits/
    ├── metadata/
    └── state/
```

各部分作用：

|Directory|作用|
|---|---|
|offsets|记录每个 Micro-batch 已处理的 Source Offset|
|commits|记录成功提交的 Batch|
|metadata|保存 Streaming Query 的相关元信息|
|state|保存 Stateful Processing 的状态|

例如：

```
Batch 0 → Commit Success
Batch 1 → Commit Success
Batch 2 → Failed
```

当 Job 重启：

```
Checkpoint
    |
    v
Find last committed batch
    |
    v
Resume processing
```

注意：

- 不要在正常重启时删除 Checkpoint。
    
- 不同 Streaming Query 不应该共用同一个 Checkpoint。
    
- 更换 Checkpoint Location 通常意味着一个新的 Query，从新的进度开始处理。
    
- Stateful Query 重启时，State Schema 需要与原来的兼容。
    
- Checkpoint 应保存在持久化存储中。
    

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

### Exactly-once Semantics

你需要区分三个概念：

|语义|含义|
|---|---|
|At-most-once|数据最多处理一次，可能丢失|
|At-least-once|数据至少处理一次，可能重复|
|Exactly-once|每条数据在目标结果中产生一次正确效果|

Spark Checkpoint 帮助跟踪数据处理进度，但不能单独保证所有外部 Sink 都具有 Exactly-once。

例如：

- Kafka Source + Delta Sink：在正确配置的情况下，可以获得端到端的 Exactly-once 处理语义。
    
- 外部 REST API：请求可能成功，但 Spark 没有收到成功响应，重试时可能重复调用。
    

所以对于外部系统，通常还需要：

- Idempotency Key
    
- Upsert
    
- Deduplication
    
- Transactional Writes（如果目标系统支持）
    

## 2.4 Watermark & Window Aggregation

Streaming 数据经常不是按照 Event Time 顺序到达的。

例如：

|Event|Event Time|Arrival Time|
|---|---|---|
|A|09:01|09:01|
|B|09:03|09:03|
|C|09:02|09:08|

C 是 Late-arriving Data。

如果系统在 09:03 就认为 09:00–09:05 的窗口已经结束，那么 C 就会被错误处理或丢弃。

因此需要 Watermark。

### Window Aggregation

```
from pyspark.sql import functions as Fresult = (    stream        .withWatermark("event_time", "10 minutes")        .groupBy(            F.window("event_time", "5 minutes")        )        .count())
```

假设：

```
Window Size = 5 minutes
Watermark Delay = 10 minutes
```

窗口：

```
09:00 - 09:05
09:05 - 09:10
09:10 - 09:15
```

Spark 需要维护这些窗口的统计状态。

例如：

```
09:00 - 09:05 → count = 128
09:05 - 09:10 → count = 156
09:10 - 09:15 → count = 97
```

这些统计结果保存在 State Store。

### Watermark 的作用

Watermark 用于：

1. 允许一定范围内的迟到数据。
    
2. 决定什么时候可以清理旧 State。
    
3. 限制 State Store 的增长。
    
4. 降低长期运行 Streaming Query 的内存和存储压力。
    

假设 Watermark 为 10 分钟：

```
Latest Event Time = 10:30

Watermark ≈ 10:20
```

对于 Event Time 早于 Watermark 的数据，系统不再保证继续接受并更新相关结果。

注意：Watermark 不是简单的定时器，也不是说每个窗口结束 10 分钟后就立即删除 State。它是基于已观测到的 Event Time 推进的处理边界。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on Google Cloud

## 2.5 Stream-Static Join

例如：

```
Streaming Orders
       |
       | JOIN
       |
Static Customer Table
```

代码：

```
orders = spark.readStream.table("bronze.orders")customers = spark.read.table("silver.customers")result = orders.join(    customers,    "customer_id")
```

特点：

- Streaming Data 是持续到达的。
    
- Static Data 是一个批次快照。
    
- 新到达的 Streaming Records 可以与当前 Static Data 进行 Join。
    

但需要注意：

如果 Customer Table 后续更新，之前已经处理过的 Order 不会因为 Static Table 变化而自动重新计算。

例如：

```
10:00 Order 1 joins Customer A
10:30 Customer A changes address
```

之前处理的 Order 1 不会自动使用新地址重新计算。

这就是你原笔记中：

> only stream table trigger the join processing

的准确含义。

如果需要随着维度变化更新历史聚合，Materialized View 或其他增量计算架构可能更适合。

## 2.6 ForeachBatch

`foreachBatch` 允许对每个 Micro-batch 使用普通 Batch DataFrame 处理逻辑。

常见场景：

- MERGE / UPSERT
    
- 写入多个 Delta Tables
    
- 调用 JDBC
    
- 调用 REST API
    
- 执行自定义 Batch Processing
    

代码：

```
from delta.tables import DeltaTabledef mergeToSilver(batchDF, batchId):    deltaTable = DeltaTable.forName(        spark,        "silver.flight"    )    (        deltaTable.alias("t")        .merge(            batchDF.alias("s"),            "t.flight_id = s.flight_id"        )        .whenMatchedUpdateAll()        .whenNotMatchedInsertAll()        .execute()    )
```

调用：

```
query = (    stream.writeStream        .foreachBatch(mergeToSilver)        .option("checkpointLocation", checkpoint_path)        .start())query.awaitTermination()
```

### batchDF 和 batchId

|Parameter|Meaning|
|---|---|
|batchDF|当前 Micro-batch 的 DataFrame|
|batchId|当前 Micro-batch 的 ID|

一个非常重要的面试考点：foreachBatch 默认是 At-least-once 语义。

如果 Batch 已经写入外部数据库，但提交状态还没有成功保存，重启时可能再次执行同一个 Batch。

因此：

- 使用 batchId 实现幂等处理。
    
- 对目标表使用 MERGE / UPSERT。
    
- 避免简单的重复 APPEND。
    

另外，`foreachBatch` 内部使用普通 Spark Batch API，因此可以使用 MERGE，而不是直接将 Streaming DataFrame 当作普通 Batch DataFrame 处理。

## Module 2 — 面试问题

Q1. What is the difference between processingTime and availableNow?

`processingTime` keeps a streaming query running and processes new data at regular intervals. `availableNow` processes all currently available data in one or more micro-batches and then terminates.

Q2. What is stored in a Structured Streaming checkpoint?

It stores source offsets, committed batches, query metadata and state information required to recover the streaming query.

Q3. Does Checkpoint guarantee exactly-once processing?

Not by itself. It tracks progress and supports recovery, but the actual guarantee also depends on the source and sink. External systems may require idempotent writes.

Q4. What is a Watermark and why is it important?

A watermark defines the lateness threshold for event-time processing. It helps process late events and clean up old state in stateful operations.

Q5. What happens if a streaming query uses an old checkpoint after changing its aggregation logic?

The query may fail to recover because the state schema or stateful operator structure is incompatible with the previous checkpoint.

Q6. Why would you use foreachBatch instead of a regular writeStream?

I would use foreachBatch when I need custom batch logic, such as MERGE, writing to multiple destinations or interacting with external systems.

Q7. What is the limitation of a Stream-Static Join?

Updates to the static DataFrame do not automatically reprocess previously processed streaming records.

# Module 3. Delta Lake & Schema Management

Data Storage / Data Integrity

这一模块是 Databricks 面试的核心。

Delta Lake 不只是 Parquet 文件格式，而是一个基于 Parquet Data Files 和 Transaction Log 的存储管理层。

## 3.1 Schema Enforcement

Schema Enforcement 是 Delta Lake 的数据写入保护机制。

假设目标 Table：

```
CREATE TABLE products (
    product_id INT,
    price DOUBLE
);
```

你尝试写入：

```
product_id
price
category
```

因为 `category` 不在目标 Schema 中，Delta 默认会拒绝不兼容的写入。

同样，如果类型不匹配，也可能报错。

目的：

- Prevent unexpected schema changes.
    
- Maintain data consistency.
    
- Avoid accidental data corruption.
    

## 3.2 Schema Evolution

Schema Evolution 则允许目标 Table 的 Schema 随着新数据发生变化。

例如：

```
Old Schema:
product_id
price
```

新数据：

```
product_id
price
category
```

可以使用：

```
df.write \    .format("delta") \    .option("mergeSchema", "true") \    .mode("append") \    .saveAsTable("products")
```

结果：

```
product_id
price
category
```

已有数据中，新增字段通常为 NULL。

### Schema Enforcement vs Schema Evolution

|Feature|Schema Enforcement|Schema Evolution|
|---|---|---|
|默认行为|验证 Schema|不自动接受所有变化|
|新字段|拒绝不兼容写入|可以添加|
|类型不匹配|拒绝|仅支持特定的类型演化|
|主要目的|数据保护|适应数据结构变化|

最佳实践： 不建议无条件开启全局 Schema Evolution。应该根据具体写入操作显式决定是否允许 Schema 变化。

### 三个不同的 Schema Evolution 机制

这点特别重要：

|Mechanism|管理对象|
|---|---|
|Auto Loader Schema Evolution|Source Files|
|Delta `mergeSchema`|Destination Delta Table|
|Declarative Pipeline Schema Evolution|Pipeline Datasets|

例如：

```
JSON file gets a new field
       |
       v
Auto Loader detects new field
       |
       v
DataFrame Schema evolves
       |
       v
Delta Table Schema needs to evolve
       |
       v
mergeSchema / Pipeline evolution
```

源数据能够识别新字段，不代表目标 Delta Table 一定自动接受新字段。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

## 3.3 ADD / RENAME / DROP COLUMN

### ADD COLUMN

```
ALTER TABLE products
ADD COLUMNS (
    category STRING
);
```

### RENAME COLUMN

```
ALTER TABLE products
RENAME COLUMN price_new TO price;
```

### DROP COLUMN

```
ALTER TABLE products
DROP COLUMN old_column;
```

这里需要特别修正你原笔记：

这些操作并不意味着所有情况下都只更新 Metadata 且永远不重写数据。

对于 Delta Column Mapping 已启用且满足兼容性要求的表，Rename 和 Drop 可以通过 Metadata 操作完成，不需要立即重写 Parquet 文件。

但 DROP COLUMN 不代表物理数据已经删除。

如果需要真正移除旧数据，还需要考虑：

- Rewrite data files
    
- Deletion Vectors（如适用）
    
- VACUUM
    
- Retention policies
    

而且 Schema 变更的兼容性取决于 Delta Protocol 和客户端版本。

## 3.4 Type Widening

Type Widening 是允许某些数据类型从较小范围拓宽到较大范围。

例如：

```
BYTE → SHORT → INT → LONG
FLOAT → DOUBLE
```

这样可以避免因为新数据的数值范围增大而直接导致类型不兼容。

例如：

原来：

```
sensor_value: INT
```

现在：

```
sensor_value: LONG
```

可以考虑启用 Type Widening：

```
ALTER TABLE sensor_reading
SET TBLPROPERTIES (
    'delta.enableTypeWidening' = 'true'
);
```

注意：

- 不是所有数据类型之间都可以拓宽。
    
- Type Widening 与普通 CAST 不同。
    
- 需要考虑 Writer Protocol 和 Reader Compatibility。
    

## 3.5 Generated Columns

Generated Column 是由表达式自动计算的字段。

例如：

```
CREATE TABLE product (
    product_id INT,
    price DOUBLE,

    price_taxed DOUBLE
    GENERATED ALWAYS AS (
        ROUND(price * 1.2, 2)
    )
);
```

插入数据：

```
INSERT INTO product (
    product_id,
    price
)
VALUES (1, 100);
```

系统自动计算：

```
product_id = 1
price = 100
price_taxed = 120
```

特点：

- 不需要手动提供 Generated Column。
    
- 由定义的表达式计算。
    
- 值保存在 Table 中，而不是每次查询动态计算。
    

适用场景：

- Derived Date Columns
    
- Computed Business Attributes
    
- Partitioning Columns
    

## 3.6 Identity Columns

Identity Column 用于自动生成唯一 ID。

有两种：

### GENERATED ALWAYS AS IDENTITY

```
CREATE TABLE customer (
    customer_sk BIGINT
        GENERATED ALWAYS AS IDENTITY,
    customer_name STRING
);
```

用户不能自行指定 Identity Column 的值。

### GENERATED BY DEFAULT AS IDENTITY

```
CREATE TABLE customer (
    customer_sk BIGINT
        GENERATED BY DEFAULT AS IDENTITY,
    customer_name STRING
);
```

允许用户提供自己的 ID；如果没有提供，则自动生成。

常见场景：

- Surrogate Key
    
- Data Warehouse Dimension Table
    
- Historical Data Migration
    

非常重要的补充：Delta Identity Columns 会禁用并发事务写入该表的能力。

因此，在高并发写入场景中不能简单地把 Identity Column 当作普通数据库 Sequence 使用。

## 3.7 VARIANT vs STRUCT

VARIANT 是用于存储半结构化数据的类型。

例如：

```
{
  "sensor_id": "5001",
  "temp": 24.2,
  "status": "active"
}
```

插入：

```
INSERT INTO iot_event
VALUES (
    1,
    PARSE_JSON(
        '{"sensor_id":"5001","temp":24.2}'
    )
);
```

读取：

```
SELECT
    payload:sensor_id::STRING AS sensor_id,
    payload:temp::DOUBLE AS temp
FROM iot_event;
```

### STRUCT vs VARIANT

|特性|STRUCT|VARIANT|
|---|---|---|
|Schema|明确、固定|灵活|
|字段类型|Schema 定义|可以包含不同类型的 JSON Value|
|字段访问|`customer.name`|`payload:field`|
|适用场景|稳定的数据结构|Schema 不稳定的半结构化数据|

VARIANT 在 Databricks 中并不是简单地等同于原始 JSON 字符串，它有自己的内部表示。

适合：

- IoT Events
    
- Third-party API Responses
    
- Dynamic JSON Payloads
    
- Semi-structured Data
    

## Module 3 — 面试问题

Q1. What is the difference between Schema Enforcement and Schema Evolution?

Schema Enforcement validates incoming data against the target table schema and rejects incompatible writes. Schema Evolution allows supported schema changes, such as adding new columns.

Q2. What happens to existing records when a new column is added?

Existing records normally return NULL for the newly added column unless a default or backfill operation is applied.

Q3. Can mergeSchema rename an existing column?

No. `mergeSchema` primarily handles adding columns. For renaming, use ALTER TABLE RENAME COLUMN with the appropriate Delta Column Mapping configuration.

Q4. What is Type Widening?

It allows supported changes from narrower data types to wider data types, such as INT to BIGINT or FLOAT to DOUBLE, without treating them as arbitrary incompatible schema changes.

Q5. What is the difference between Generated Columns and Identity Columns?

Generated Columns calculate values based on expressions. Identity Columns generate sequential surrogate identifiers.

Q6. What is the difference between STRUCT and VARIANT?

STRUCT uses a predefined schema, while VARIANT is designed for flexible semi-structured data with dynamic fields and types.

# Module 4. Data Quality, Constraints & SCD

Data Quality / Data Modeling

这一部分建议理解为两个不同层次：

1. 数据进入 Table 时如何保证数据合法。
    
2. 数据随着时间变化时如何保留历史。
    

## 4.1 Delta Constraints

Delta Lake 支持两类 Enforced Constraints：

- NOT NULL
    
- CHECK
    

例如：

```
CREATE TABLE orders (
    order_id BIGINT,
    amount DOUBLE,
    status STRING,

    CONSTRAINT valid_amount
    CHECK (amount > 0),

    CONSTRAINT valid_status
    CHECK (
        status IN (
            'pending',
            'completed',
            'cancelled'
        )
    )
);
```

如果插入：

```
INSERT INTO orders
VALUES (1, -100, 'pending');
```

因为 `amount > 0` 不满足，Transaction 会失败。

### NOT NULL

```
CREATE TABLE customer (
    customer_id BIGINT NOT NULL,
    name STRING
);
```

### CHECK

```
ALTER TABLE orders
ADD CONSTRAINT valid_amount
CHECK (amount > 0);
```

删除：

```
ALTER TABLE orders
DROP CONSTRAINT valid_amount;
```

注意：增加 Constraint 时，Databricks 会检查现有数据是否满足约束。如果不满足，添加操作会失败。

## 4.2 Primary Key / Foreign Key

这是原笔记中必须纠正的知识点。

在 Databricks Unity Catalog 中，Primary Key、Foreign Key 和 Unique Constraints 通常是 Informational Constraints。

它们主要用于声明数据关系，不会自动保证唯一性或非 NULL。

例如：

```
CREATE TABLE customers (
    customer_id BIGINT NOT NULL,
    customer_name STRING,

    CONSTRAINT pk_customer
    PRIMARY KEY (customer_id)
);
```

不能据此认为 Databricks 会自动拒绝重复的 `customer_id`。

如果需要真正保证唯一性，需要在 ETL 过程中主动检查：

```
SELECT
    customer_id,
    COUNT(*) AS cnt
FROM customers
GROUP BY customer_id
HAVING COUNT(*) > 1;
```

可以使用：

- Deduplication
    
- MERGE
    
- Data Quality Expectations
    
- Validation Queries
    

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

## 4.3 Delta Constraints vs Pipeline Expectations

两者容易混淆。

|特性|Delta CHECK|Pipeline Expectations|
|---|---|---|
|所属层|Delta Table|Lakeflow Pipeline|
|主要用途|强制数据完整性|数据质量监控和处理|
|违规行为|Transaction 失败|Warn / Drop / Fail|
|是否记录通过和失败数量|不作为主要功能|是|
|是否适合隔离坏数据|需要额外逻辑|可以结合 Quarantine Table|

### Pipeline Expectations

三种核心方式：

```
@dp.expect(    "valid_age",    "age >= 0")
```

记录违规数据，但不丢弃。

```
@dp.expect_or_drop(    "valid_age",    "age >= 0")
```

丢弃违规数据。

```
@dp.expect_or_fail(    "valid_age",    "age >= 0")
```

遇到违规数据时，Pipeline Update 失败。

### Expectation 与 Quarantine

如果不想直接丢弃坏数据，可以建立 Quarantine Table。

例如：

```
@dp.tabledef orders_valid():    return (        spark.readStream.table("bronze.orders")        .filter("amount > 0")    )
```

```
@dp.tabledef orders_quarantine():    return (        spark.readStream.table("bronze.orders")        .filter("amount <= 0")    )
```

实际生产还应考虑 NULL、Schema Error、缺失字段等情况，并避免重复读取产生不一致的结果。

## 4.4 SCD — Slowly Changing Dimensions

SCD 用于处理 Dimension Table 随着时间变化的情况。

例如：

```
Customer ID = 1001

2025:
City = Amsterdam

2026:
City = Rotterdam
```

不同 SCD 类型有不同策略。

|Type|处理方式|是否保留历史|
|---|---|---|
|Type 0|不允许改变|是，原值保持|
|Type 1|覆盖旧值|否|
|Type 2|新增版本记录|是|
|Type 3|使用额外字段保存有限历史|部分保留|

### SCD Type 1

直接更新：

```
MERGE INTO dim_customer t
USING staging_customer s
ON t.customer_id = s.customer_id

WHEN MATCHED THEN
UPDATE SET
    t.city = s.city

WHEN NOT MATCHED THEN
INSERT (
    customer_id,
    city
)
VALUES (
    s.customer_id,
    s.city
);
```

结果：

|customer_id|city|
|---|---|
|1001|Rotterdam|

Amsterdam 被覆盖。

### SCD Type 2

SCD Type 2 通过保留多个版本来保存历史。

常见字段：

```
customer_id
customer_name
city
effective_date
end_date
is_current
```

例如：

|customer_id|city|effective_date|end_date|is_current|
|---|---|---|---|---|
|1001|Amsterdam|2025-01-01|2026-06-01|false|
|1001|Rotterdam|2026-06-01|NULL|true|

处理流程：

```
Incoming Customer Update
          |
          v
Compare with Current Record
          |
          v
Detect Changed Attributes
          |
          v
Close Existing Record
          |
          v
Insert New Version
```

你原笔记中的 SQL 是这个逻辑：

```
-- Identify changed customers
CREATE TEMP TABLE changed_customers AS
SELECT src.*
FROM staging_customer src
LEFT JOIN dim_customer tgt
    ON tgt.customer_id = src.customer_id
   AND tgt.is_current = TRUE
WHERE tgt.customer_id IS NULL
   OR tgt.city <> src.city;
```

这里需要注意 NULL 比较问题。生产中更稳妥的方式是使用支持 NULL-safe comparison 的 `<=>`。

关闭旧记录：

```
UPDATE dim_customer tgt
SET
    end_date = CURRENT_DATE,
    is_current = FALSE
WHERE tgt.is_current = TRUE
AND EXISTS (
    SELECT 1
    FROM changed_customers c
    WHERE c.customer_id = tgt.customer_id
);
```

插入新版本：

```
INSERT INTO dim_customer (
    customer_id,
    city,
    effective_date,
    end_date,
    is_current
)
SELECT
    customer_id,
    city,
    CURRENT_DATE,
    NULL,
    TRUE
FROM changed_customers;
```

生产代码还需要处理：

- 同一天同一个 Key 多次变化。
    
- Late-arriving CDC Events。
    
- Duplicate Events。
    
- Source Deletes。
    
- Multiple Changes in One Batch。
    

### SCD Type 2 vs Delta Time Travel

这是非常值得面试的一个问题。

|SCD Type 2|Delta Time Travel|
|---|---|
|保存业务实体历史|保存 Table 版本历史|
|记录某个客户过去的地址|记录 Table 在某个时间点的状态|
|业务级历史|技术级数据版本历史|
|可以长期保留业务历史|受数据和 Log Retention 限制|

例如：

- SCD2：查询客户在 2025 年住在哪个城市。
    
- Time Travel：查询 Delta Table 在 2025 年某个版本是什么样子。
    

不能用 Time Travel 代替 SCD Type 2。

## Module 4 — 面试问题

Q1. What is the difference between CHECK and PRIMARY KEY constraints in Databricks?

CHECK is enforced and rejects writes that violate its condition. Primary Key constraints are informational and do not automatically enforce uniqueness or non-null values.

Q2. How would you implement data quality checks in a production pipeline?

I would use Delta constraints for hard integrity rules and pipeline expectations for row-level quality monitoring. Invalid records can be routed into quarantine tables for investigation.

Q3. What is the difference between SCD Type 1 and Type 2?

Type 1 overwrites existing values, whereas Type 2 inserts a new version and preserves historical records using effective dates and current-record indicators.

Q4. Why is SCD Type 2 useful in a Data Warehouse?

It allows analysts to understand historical business states, such as customer addresses or product categories at a specific point in time.

Q5. Can Delta Time Travel replace SCD Type 2?

No. Time Travel tracks technical table versions, while SCD Type 2 tracks business-level historical changes.

Q6. How would you detect duplicate primary keys in a Delta Table?

I would use GROUP BY and HAVING COUNT(*) > 1, or implement deduplication and key validation before writing the table.

# Module 5. CDC & Change Data Feed

Incremental Processing

这是你笔记中另一个核心模块。

先区分三个概念：

|Concept|作用|
|---|---|
|CDC|捕获源数据库发生的变化|
|CDF|读取 Delta Table 中发生的行级变化|
|MERGE|将源端变化应用到目标 Delta Table|

## 5.1 CDC 的基本流程

例如源数据库发生：

```
INSERT customer 1001
UPDATE customer 1002
DELETE customer 1003
```

CDC 将这些变化转换成事件：

|customer_id|operation|city|
|---|---|---|
|1001|INSERT|Amsterdam|
|1002|UPDATE|Rotterdam|
|1003|DELETE|NULL|

通常还会有：

- Event Timestamp
    
- Sequence Number
    
- Transaction ID
    
- Source Offset
    

然后目标端使用 MERGE 或 AUTO CDC 更新数据。

## 5.2 Delta MERGE

例如：

```
MERGE INTO silver.customers AS target
USING bronze.customers AS source
ON target.customer_id = source.customer_id

WHEN MATCHED AND source.operation = 'DELETE'
THEN DELETE

WHEN MATCHED AND source.operation = 'UPDATE'
THEN UPDATE SET *

WHEN NOT MATCHED AND source.operation <> 'DELETE'
THEN INSERT *;
```

### 一个非常重要的问题：Multiple Matches

假设 Source CDC：

|customer_id|city|timestamp|
|---|---|---|
|1001|Amsterdam|10:00|
|1001|Rotterdam|10:05|
|1001|Utrecht|10:10|

目标表：

```
customer_id = 1001
```

如果同一条 Target Record 匹配多条 Source Records，MERGE 可能报错，因为多个 Source Rows 试图更新同一 Target Row。

解决方法：

使用 Window Function 选取最新事件。

```
from pyspark.sql import functions as Ffrom pyspark.sql.window import Windoww = (    Window        .partitionBy("customer_id")        .orderBy(            F.col("event_timestamp").desc(),            F.col("sequence_number").desc()        ))latest_changes = (    source_df        .withColumn("rn", F.row_number().over(w))        .filter(F.col("rn") == 1)        .drop("rn"))
```

然后再执行 MERGE。

注意：上面的方式适用于只需要保留每个 Key 最新状态的场景。如果一个 Batch 内有多个事件，而你需要完整地保留所有中间变化，就不能简单地只取最新记录。

## 5.3 Change Data Feed（CDF）

CDF 是 Delta Lake 提供的行级变化读取能力。

可以记录：

- INSERT
    
- UPDATE_PREIMAGE
    
- UPDATE_POSTIMAGE
    
- DELETE
    

启用：

```
CREATE TABLE books (
    book_id BIGINT,
    title STRING
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

读取：

```
cdf_df = (    spark.readStream        .format("delta")        .option("readChangeFeed", "true")        .option("startingVersion", 2)        .table("customers_silver"))
```

CDF 通常还提供以下元数据字段：

|Column|Meaning|
|---|---|
|`_change_type`|insert / update_preimage / update_postimage / delete|
|`_commit_version`|Delta Commit Version|
|`_commit_timestamp`|Commit Time|

例如：

```
cdf_df.select(    "customer_id",    "city",    "_change_type",    "_commit_version")
```

### CDF 和普通 Delta Streaming Read 的区别

|特性|Normal Streaming Read|CDF|
|---|---|---|
|Append|支持|支持|
|Update|默认不适合直接读取|支持变化事件|
|Delete|默认不适合直接读取|支持删除事件|
|历史更新信息|不提供完整行级变化记录|提供 Row-level Changes|
|典型用途|Append-only 数据流|增量同步和下游 CDC|

CDF 是一种增量变化读取机制，不是自动帮你把数据同步到下游的工具。

下游还需要决定如何处理：

- Update
    
- Delete
    
- Deduplication
    
- Ordering
    
- Schema Evolution
    

## 5.4 CDF 和 VACUUM

你原笔记中：

> When running VACUUM, CDF data also deleted.

这个方向是对的，但需要更精确。

CDF 可能通过两种方式存储：

1. 利用 Delta Transaction Log 推导变化。
    
2. 某些场景下使用独立的 Change Data Files。
    

VACUUM 删除不再被需要的底层数据文件后，可能导致对应版本的 CDF 无法再读取。

因此：

- CDF 不是永久历史数据库。
    
- CDF 可读取的历史范围受 Retention 和 VACUUM 影响。
    
- 如果下游需要长期保留变更历史，需要将 CDF 消费到自己的历史表或归档存储。
    

## 5.5 AUTO CDC

如果使用 Lakeflow Pipelines，可以使用 AUTO CDC，而不需要自己编写完整的 MERGE 逻辑。

例如：

```
from pyspark import pipelines as dpdp.create_streaming_table(    name="silver_orders")dp.create_auto_cdc_flow(    target="silver_orders",    source="bronze_orders",    keys=["order_id"],    sequence_by="ts_ms",    apply_as_delete="op = 'd'",    except_column_list=["op", "ts_ms"],    stored_as_scd_type=1)
```

注意：你原笔记里的 API 参数 `key`、`store_as_scd_type` 等属于旧版本或不同 API 写法，当前 Python API 通常使用 `keys`、`stored_as_scd_type`。具体还要根据实际 Runtime / Lakeflow API 版本确认。

AUTO CDC 可以处理：

- Event Ordering
    
- Duplicate Keys
    
- SCD Type 1
    
- SCD Type 2
    
- Delete Events
    
- Out-of-order Events
    

特别适合数据库 CDC 到 Lakehouse 的场景。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

## Module 5 — 面试问题

Q1. What is the difference between CDC and CDF?

CDC captures changes from a source database. CDF exposes row-level changes made to a Delta Table, allowing downstream systems to consume inserts, updates and deletes.

Q2. Why can MERGE fail when the source contains multiple records for the same key?

Multiple source records may match the same target record, making the update ambiguous. I would deduplicate or order the source records based on event time and sequence number before MERGE.

Q3. What is the difference between UPDATE_PREIMAGE and UPDATE_POSTIMAGE?

UPDATE_PREIMAGE represents the row before an update, while UPDATE_POSTIMAGE represents the row after the update.

Q4. What happens to CDF data after VACUUM?

CDF data associated with removed files may no longer be available. Therefore, downstream consumers must process changes within the retention window or persist them independently.

Q5. When would you use AUTO CDC instead of manually writing MERGE?

I would use AUTO CDC when processing ordered database change events and implementing SCD Type 1 or Type 2. It reduces custom code for ordering, deduplication and history tracking.

# Module 6. Delta Lake Performance Optimization

Performance / Cost Optimization

这个模块可以按照四个问题来理解：

1. 如何减少小文件？
    
2. 如何减少查询扫描的数据量？
    
3. 如何提高 UPDATE / DELETE / MERGE 性能？
    
4. 如何自动维护 Delta Tables？
    

## 6.1 Partitioning

Partitioning 是根据某些字段将数据组织成不同的物理目录。

例如：

```
CREATE TABLE orders (
    order_id BIGINT,
    amount DOUBLE,
    order_date DATE
)
USING DELTA
PARTITIONED BY (order_date);
```

物理结构可能类似：

```
orders/
    |
    ├── order_date=2026-01-01/
    |       ├── part-001.parquet
    |       └── part-002.parquet
    |
    ├── order_date=2026-01-02/
    |       └── part-003.parquet
    |
    └── order_date=2026-01-03/
            └── part-004.parquet
```

如果查询：

```
SELECT *
FROM orders
WHERE order_date = '2026-01-02';
```

Spark 可以通过 Partition Pruning 跳过其他日期的目录。

### Partitioning 的优点

- 减少扫描的数据。
    
- 改善特定查询场景。
    
- 支持分区级数据管理。
    

### Partitioning 的缺点

过度分区会造成：

- Small Files。
    
- Metadata Overhead。
    
- 很多空目录。
    
- 写入任务过于分散。
    

例如：

```
Partition by country
```

通常可能比较合理。

但：

```
Partition by customer_id
```

如果 customer_id 有几百万个不同值，很可能产生严重的过度分区。

### 1 GB Rule

你原笔记中记录：

> Partition should be 1 GB above.

更准确地说，这是一个传统的经验性建议，不是 Databricks 的硬性规则。

可以将每个 Partition 最终产生的数据量在约 1 GB 以上作为初步参考，但实际设计应根据：

- Table Size
    
- Query Patterns
    
- Data Distribution
    
- File Size
    
- Write Frequency
    

来决定。

对于新的 Databricks Table，不应该默认先做传统 Partitioning，而应该考虑 Liquid Clustering。

## 6.2 Data Skipping

Data Skipping 是 Delta Lake 减少数据扫描的重要机制。

Delta Lake 在 Transaction Log 中维护文件级统计信息，例如：

- min value
    
- max value
    
- null count
    
- record count
    

例如：

```
File A:
age min = 18
age max = 30

File B:
age min = 31
age max = 50

File C:
age min = 51
age max = 80
```

查询：

```
SELECT *
FROM customers
WHERE age > 70;
```

Spark 可以跳过不包含相关数据的文件。

这叫 Data Skipping。

注意：

Data Skipping 不等于 Index。

它主要利用文件级统计信息减少不必要的文件扫描。

## 6.3 OPTIMIZE

Delta Lake 在频繁写入时可能产生很多小文件。

例如：

```
part-001.parquet  5 MB
part-002.parquet  8 MB
part-003.parquet  3 MB
part-004.parquet  6 MB
```

过多小文件会增加：

- File Listing Overhead。
    
- Metadata Processing。
    
- Query Planning Cost。
    
- Read Task Overhead。
    

执行：

```
OPTIMIZE orders;
```

可以将小文件合并成更大的文件。

例如：

```
Before:
1000 files × 5 MB

After:
5 files × 1 GB
```

这里只是示意，实际文件大小取决于配置和数据分布。

## 6.4 Optimize Write & Auto Compaction

这两个机制可以帮助减少 Small Files。

### Optimize Write

在写入过程中优化数据分布，尽可能生成更合理大小的文件。

### Auto Compaction

写入结束后检查是否有可以进一步合并的小文件。

例如：

```
ALTER TABLE orders
SET TBLPROPERTIES (
    'delta.autoOptimize.optimizeWrite' = 'true',
    'delta.autoOptimize.autoCompact' = 'true'
);
```

注意：

这两个设置不代表每次都生成完全相同大小的文件。Databricks 对不同 Table 类型和 Runtime 也可能有默认优化行为。

## 6.5 Z-Ordering

Z-Ordering 是一种多维数据布局优化方式。

例如：

```
OPTIMIZE orders
ZORDER BY (customer_id);
```

如果经常执行：

```
SELECT *
FROM orders
WHERE customer_id = 1001;
```

Z-Ordering 可以让相似的 Customer ID 更集中地分布在文件中。

![Delta 2.0 - The Foundation of your Data Lakehouse is Open | Delta Lake](https://images.openai.com/static-rsc-4/0qDE0Y06bzi6xc_WPS5rQdtW64Cycn5pT8YLCEy-9EijlMujFfUweG19qRKZLCg1MZ_vbhMnGdjOEPuG7ClD1N1I4932MdETLNGcT1631x8D6ZMNQ5V0q_fu5Ylts9ZW66_RdS5-Gp934WrToPdST-mkUewm-JXsqFvsjgmon4E?purpose=inline)

[delta.io](https://delta.io/blog/2022-08-02-delta-2-0-the-foundation-of-your-data-lake-is-open/)

适用场景：

- 查询经常过滤某些字段。
    
- 字段具有较高 Cardinality。
    
- 文件级 Data Skipping 能够有效减少扫描。
    

注意：

Z-Ordering 并不是传统数据库的 B-Tree Index。

它不会创建一个独立的索引结构，而是通过数据文件布局提升 Data Skipping 效果。

也不是 Cardinality 越高就一定越适合 Z-Ordering，还要看过滤频率和数据分布。

## 6.6 Liquid Clustering

Liquid Clustering 是 Databricks 更现代的数据布局优化方式。

例如：

```
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    amount DOUBLE
)
CLUSTER BY (customer_id, order_date);
```

也可以：

```
ALTER TABLE orders
CLUSTER BY (customer_id, order_date);
```

或者：

```
ALTER TABLE orders
CLUSTER BY AUTO;
```

### Liquid Clustering vs Partitioning vs Z-Ordering

![Liquid Clustering With Databricks Delta Lake](https://images.openai.com/static-rsc-4/ek1Lb57Wr4Gw8KStflKKTpstqRyivQaoRDqVpNJAnbLa70WXZoQj8NuV_2EBMS5OfPraRHdJR6nh5ln_Kr0DYYSrcLNzAe8FXINBLkdHqpjvxYQgvaHOU_yrXWPzpm2tWdd0XvZw94nt_UmXGT-Z068agsMlLPzTu8pcNlX8eXY?purpose=inline)

[dzone.com](https://dzone.com/articles/liquid-clustering-with-databricks-delta-lake)

|特性|Partitioning|Z-Ordering|Liquid Clustering|
|---|---|---|---|
|数据组织方式|固定目录分区|多维数据布局|自适应聚类布局|
|是否需要预先分区|是|否|否|
|是否支持修改 Key|有限制|可以重新执行 OPTIMIZE|支持调整 Clustering Keys|
|是否适合高 Cardinality|通常不适合直接分区|可以|可以|
|增量维护|传统分区|OPTIMIZE|Incremental Clustering|
|适用场景|明确且合理的分区键|现有 Delta 表优化|新建和持续演进的大型表|

关键点：

- Liquid Clustering 不需要传统的固定分区目录。
    
- 可以根据查询模式调整 Clustering Keys。
    
- 不需要一次性重新组织整张表。
    
- 通过后续 OPTIMIZE 逐步维护数据布局。
    

修正原笔记：Liquid Clustering 仍然会重写需要重新聚类的数据文件。它的优势是可以增量地执行聚类，而不是保证完全不重写已有数据。

另外，Liquid Clustering 与传统 Partitioning / Z-Ordering 不应同时使用在同一张表的同一数据布局策略中。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

## 6.7 Predictive Optimization

Predictive Optimization 是 Databricks 提供的自动化 Table Maintenance 功能。

它可以自动执行：

- OPTIMIZE
    
- VACUUM
    
- ANALYZE / Statistics Collection
    
- Liquid Clustering Maintenance（适用时）
    

例如：

```
Delta Table
    |
    v
Predictive Optimization
    |
    ├── Detect Small Files
    |
    ├── OPTIMIZE
    |
    ├── Collect Statistics
    |
    └── VACUUM
```

它使用 Databricks 管理的后台计算资源执行维护，而不是简单地占用你正在运行的普通 Job Cluster。

通常需要相应的 Unity Catalog Managed Table 和平台配置支持。

## 6.8 Deletion Vectors

Deletion Vector 是 Delta Lake 的一种逻辑删除机制。

传统 DELETE：

```
DELETE FROM orders
WHERE order_id = 1001;
```

可能需要重写相关数据文件。

Deletion Vector 则可以先记录哪些行已经被删除，而不立即重写整个 Parquet 文件。

例如：

```
Original Parquet:
Row 1
Row 2
Row 3
Row 4
Row 5

Deletion Vector:
Row 2 → Deleted
Row 4 → Deleted
```

读取时：

```
Visible Rows:
Row 1
Row 3
Row 5
```

优点：

- 提高 DELETE 性能。
    
- 提高 UPDATE 性能。
    
- 减少频繁小规模数据变更造成的文件重写。
    

但：

Deletion Vector 不是物理删除。

后续可能需要 OPTIMIZE 将逻辑删除状态物化到新的数据文件中。

## 6.9 VACUUM

VACUUM 用于删除已经不再被当前 Delta Table 版本引用的旧数据文件。

例如：

```
VACUUM orders RETAIN 720 HOURS;
```

720 Hours = 30 Days。

也可以设置：

```
ALTER TABLE orders
SET TBLPROPERTIES (
    'delta.deletedFileRetentionDuration' = 'interval 30 days'
);
```

注意：这里正确的 Table Property 是 `delta.deletedFileRetentionDuration`，不是原笔记中的 `delta.deleteFileRetentionDuration`。

### 为什么需要 VACUUM？

假设：

```
Version 1:
part-001.parquet

Version 2:
part-002.parquet

Version 3:
part-003.parquet
```

Version 2 更新时，可能不再引用 Version 1 的数据文件。

但是旧文件不会立即删除。

这样可以支持：

- Time Travel
    
- Concurrent Readers
    
- Historical Version Access
    
- Recovery
    

VACUUM 负责清理已经过期且不再需要的旧文件。

### VACUUM vs OPTIMIZE

|Feature|OPTIMIZE|VACUUM|
|---|---|---|
|主要目的|优化数据布局|删除旧文件|
|处理对象|当前有效数据文件|不再被引用的旧文件|
|是否改善 Small Files|是|不是|
|是否影响历史数据访问|可能产生旧版本文件|会删除过期历史数据文件|

## 6.10 Delta Transaction Log

每张 Delta Table 都有：

```
_delta_log/
    |
    ├── 00000000000000000000.json
    ├── 00000000000000000001.json
    ├── 00000000000000000002.json
    |
    ├── 00000000000000000010.checkpoint.parquet
    |
    └── _last_checkpoint
```

Transaction Log 记录：

- Add File
    
- Remove File
    
- Metadata Changes
    
- Protocol Changes
    
- Transaction Commit Information
    

每次成功的 Transaction 会产生新的 Table Version。

例如：

```
Version 0 → CREATE TABLE
Version 1 → INSERT
Version 2 → UPDATE
Version 3 → DELETE
```

查询历史：

```
DESCRIBE HISTORY orders;
```

Time Travel：

```
SELECT *
FROM orders VERSION AS OF 1;
```

或者：

```
SELECT *
FROM orders
TIMESTAMP AS OF '2026-11-01 10:00:00';
```

### Transaction Log Checkpoint

你原笔记中记录：

> Auto create parquet checkpoint every 10 commit.

这是 Delta Lake 的传统默认行为之一，但不应当作所有场景都固定每 10 个 Commit 创建一次的绝对规则。

Checkpoint 的作用：

- 减少读取大量 JSON Transaction Logs 的开销。
    
- 加快 Snapshot Reconstruction。
    
- 通过 Parquet 格式保存累计状态。
    

例如：

```
Read Snapshot:
    |
    ├── Latest Checkpoint
    |
    └── JSON Logs after Checkpoint
```

不需要从 Version 0 开始读取所有 JSON。

### Statistics

Delta 可以维护文件级统计信息。

你笔记中的前 32 列需要特别注意：

- `delta.dataSkippingNumIndexedCols` 是相关配置之一。
    
- 默认值通常是 32。
    
- 它影响哪些列会收集 Data Skipping Statistics。
    
- Nested Fields 也会影响列统计的范围。
    

对于很大的 Free Text Columns，如果它们排在前面，可能挤占统计列数，因此需要合理设计 Schema 和统计列配置。

### Log Retention vs Data File Retention

这是非常重要的区别。

|Property|作用|
|---|---|
|`delta.logRetentionDuration`|控制 Transaction Log 历史保留|
|`delta.deletedFileRetentionDuration`|控制删除文件保留期限|

默认情况下：

- Log Retention 通常为 30 天。
    
- Deleted File Retention 通常为 7 天。
    

因此：

Time Travel 能否成功，取决于需要的 Transaction Log 和实际数据文件是否仍然可用。

VACUUM 可能让旧版本数据即使还有部分 Log，也无法被完整读取。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

## Module 6 — 面试问题

Q1. What is the difference between Partitioning and Liquid Clustering?

Partitioning physically separates data into fixed directory partitions, while Liquid Clustering organizes data using clustering keys and allows incremental data layout optimization without requiring fixed partition directories.

Q2. What is Z-Ordering? Is it a traditional index?

No. Z-Ordering reorganizes data files to improve data skipping. It does not create a traditional B-Tree or separate index structure.

Q3. What is the difference between OPTIMIZE and VACUUM?

OPTIMIZE improves the layout of active data files, while VACUUM removes obsolete files that are no longer referenced by the current table version.

Q4. What is a Deletion Vector?

It records logically deleted rows without immediately rewriting the entire Parquet file. This improves DELETE and UPDATE performance.

Q5. What is the purpose of the Delta Transaction Log?

It records every transaction and enables ACID guarantees, snapshot reconstruction, concurrent access and time travel.

Q6. What is the difference between Log Retention and Deleted File Retention?

Log Retention controls how long transaction history is retained, while Deleted File Retention controls how long obsolete data files remain available before VACUUM can remove them.

Q7. How would you optimize a Delta Table with frequent MERGE operations?

I would analyze file sizes, clustering keys, query patterns and the frequency of updates. I would consider Deletion Vectors, Optimize Write, Auto Compaction and Liquid Clustering where appropriate.

# Module 7. Lakeflow Spark Declarative Pipelines（SDP）

ETL Architecture / Production Pipelines

你原笔记中这一部分叫 Declarative Pipeline / DLT。

这里需要了解产品名称的变化：

- Delta Live Tables（DLT）：旧名称。
    
- Lakeflow Spark Declarative Pipelines：Databricks 当前的产品体系。
    
- Apache Spark Declarative Pipelines（SDP）：开源的声明式 Pipeline 框架。
    
- Lakeflow Pipelines：Databricks 在 SDP 基础上提供的生产级能力。
    

这部分是目前 Databricks Data Engineer 非常值得掌握的知识。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

## 7.1 Declarative vs Imperative

首先理解两个概念。

### Imperative Programming

你告诉 Spark：

- 先执行什么。
    
- 后执行什么。
    
- 数据如何处理。
    
- 如何管理执行顺序。
    

例如：

```
df = spark.read.csv(path)df = df.filter("amount > 0")df.write.format("delta").saveAsTable(    "silver.orders")
```

你需要自己管理很多处理细节。

### Declarative Programming

你主要声明：

- 需要什么数据集。
    
- 数据集之间的依赖关系。
    
- 每个数据集的处理逻辑。
    

Pipeline Engine 自动管理：

- Dependency Resolution
    
- Execution Order
    
- Incremental Processing
    
- Orchestration
    
- Retry
    

例如：

```
from pyspark import pipelines as dp@dp.tabledef bronze_orders():    return (        spark.readStream            .format("cloudFiles")            .option("cloudFiles.format", "json")            .load("/Volumes/main/bronze/raw/")    )
```

Pipeline 会识别 Dataset Dependencies，并自动管理执行顺序。

## 7.2 三种 Dataset Types

这是最重要的知识点之一。

![What is Lakeflow Spark Declarative Pipelines - Azure Databricks | Microsoft Learn](https://images.openai.com/static-rsc-4/oX05n4h4BanXiCjJZ8lxE0uQT75f90LoVbUaV7EfFlcXu9NQ6-puclBnybPBvLK9N-GpRbYc9LdQ53xtZENJ2jE1Rnw75syfbS6KLEbkVWcLnQXQukF4Va0CjgV2R5UU8aNFRyGnRCYbhAf-yMkFqhFwKZgJNjMWnPx-cnVaWZk?purpose=inline)

[learn.microsoft.com](https://learn.microsoft.com/zh-cn/azure/databricks/ldp/concepts/)

|Type|用途|是否物理存储结果|
|---|---|---|
|Streaming Table|增量数据摄取和处理|是|
|Materialized View|预计算分析结果|是|
|Temporary View|中间查询逻辑|否|

### A. Streaming Table

适合：

- Auto Loader
    
- Kafka
    
- Event Hubs
    
- Append-only Sources
    
- Incremental Processing
    

例如：

```
from pyspark import pipelines as dp@dp.table(    name="bronze_orders")def bronze_orders():    return (        spark.readStream            .format("cloudFiles")            .option("cloudFiles.format", "json")            .load(PATH)    )
```

Streaming Table 的特点：

- 持续接收新数据。
    
- 使用 Checkpoint 跟踪处理进度。
    
- 适合 Incremental Processing。
    
- 可以作为 AUTO CDC 的目标表。
    

### B. Materialized View

适合：

- Aggregations
    
- Complex Joins
    
- BI Queries
    
- Gold Layer
    
- Precomputed Results
    

例如：

```
@dp.materialized_view(    name="gold_daily_revenue")def gold_daily_revenue():    return (        spark.read.table("silver.orders")            .groupBy("order_date")            .agg(                F.sum("amount").alias("revenue")            )    )
```

特点：

- 结果被物理存储。
    
- Pipeline 自动维护结果。
    
- 可以增量刷新。
    
- 如果查询或数据源不支持增量刷新，也可能进行完整重新计算。
    

### C. Temporary View

用于中间转换：

```
@dp.temporary_viewdef valid_orders():    return (        spark.read.table("bronze.orders")            .filter("amount > 0")    )
```

特点：

- 不单独物理存储结果。
    
- 仅在 Pipeline 中使用。
    
- 可以减少不必要的中间数据集。
    

### 三者如何选择？

|场景|选择|
|---|---|
|新文件进入 Bronze|Streaming Table|
|Kafka 事件持续进入|Streaming Table|
|CDC 更新 Silver|Streaming Table + AUTO CDC|
|多表 Join 生成 Gold|Materialized View|
|复杂 BI 聚合|Materialized View|
|中间过滤逻辑|Temporary View|

注意：Materialized View 不是普通 View，也不是 Streaming Table。它会保存预计算结果，并由 Pipeline 管理刷新。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

## 7.3 Flows

Flow 是数据从 Source 到 Target 的处理路径。

常见类型：

### Append Flow

用于向 Streaming Table 追加数据。

例如：

```
Auto Loader
    |
    v
Streaming Table
```

如果是单一 Append Source，可以直接通过 `@dp.table` 定义。

如果多个 Source 需要写入同一个 Streaming Table，可以使用：

```
dp.create_streaming_table(    name="bronze_events")dp.append_flow(    target="bronze_events",    name="events_from_source_a",    flow_func=read_source_a)dp.append_flow(    target="bronze_events",    name="events_from_source_b",    flow_func=read_source_b)
```

### AUTO CDC Flow

用于应用 CDC Events：

```
Bronze CDC
    |
    v
AUTO CDC Flow
    |
    v
Silver Streaming Table
```

示例：

```
dp.create_streaming_table(    name="silver_orders")dp.create_auto_cdc_flow(    target="silver_orders",    source="bronze_orders",    keys=["order_id"],    sequence_by="ts_ms",    apply_as_delete="op = 'd'",    except_column_list=["op", "ts_ms"],    stored_as_scd_type=2)
```

`sequence_by` 非常重要，它决定同一个 Key 的变化事件如何排序。

如果同一订单存在：

```
10:00 → created
10:05 → updated
10:10 → deleted
```

系统需要根据 Sequence 来确定最终状态和历史版本。

### Sink

Sink 用于将 Pipeline 数据输出到外部系统。

例如：

```
Streaming Table
      |
      v
Kafka / Event Hubs / External Destination
```

你原笔记中的：

```
dp.create_sink()dp.append_flow()
```

可以理解为定义 Sink 和将 Flow 输出到 Sink 的过程。具体 API 和参数需要根据当前 Lakeflow / SDP 版本使用对应的接口。

## 7.4 AUTO CDC SCD Type 1 / Type 2

AUTO CDC 的价值是减少手写 MERGE 代码。

SCD Type 1：

```
dp.create_auto_cdc_flow(    target="silver_orders",    source="bronze_orders",    keys=["order_id"],    sequence_by="ts_ms",    stored_as_scd_type=1)
```

只保存最新状态。

SCD Type 2：

```
dp.create_auto_cdc_flow(    target="silver_orders",    source="bronze_orders",    keys=["order_id"],    sequence_by="ts_ms",    stored_as_scd_type=2)
```

保留历史版本。

需要额外注意：

- Source CDC Events 必须有可靠的 Ordering。
    
- Key 必须能唯一识别业务实体。
    
- Delete Event 必须正确映射。
    
- 不要忽略同一 Key 的多个事件。
    

## 7.5 Development Mode vs Production Mode

你原笔记中：

> Development Mode: 复用 Cluster Production Mode: 每次使用新的 Job Cluster

这是一个过度简化的记忆方式。

更准确地说：

|Mode|主要目标|
|---|---|
|Development|快速迭代、调试、较低资源开销|
|Production|稳定性、隔离、可预测性|

Development Mode 通常：

- 使用更宽松的调度策略。
    
- 方便开发人员调试。
    
- 可以减少不必要的 Cluster 启停。
    

Production Mode 通常：

- 使用更严格的执行和重试策略。
    
- 更重视资源隔离。
    
- 可以配置 Production Compute。
    
- 更适合正式调度运行。
    

但不能认为 Production Mode 在所有配置下都一定每次创建新的 Job Cluster。实际 Compute 生命周期还取决于 Pipeline Compute Mode 和相关设置。

## 7.6 Expectations 与 Quarantine

Pipeline Expectations 是数据质量监控的重要功能。

例如：

```
@dp.table@dp.expect(    "valid_order_id",    "order_id IS NOT NULL")@dp.expect_or_drop(    "valid_amount",    "amount > 0")def silver_orders():    return spark.readStream.table(        "bronze.orders"    )
```

如果需要隔离坏数据：

```
@dp.tabledef orders_quarantine():    return (        spark.readStream.table("bronze.orders")            .filter(                "order_id IS NULL OR amount <= 0"            )    )
```

生产中建议让 Valid 和 Quarantine 数据来自一致的 Source Snapshot / Processing Flow，避免两条独立处理路径产生重复或不一致。

## 7.7 Event Log & Expectations Monitoring

Pipeline Event Log 是观察 Pipeline 执行情况的重要数据源。

它可以记录：

- Pipeline Updates
    
- Flow Progress
    
- Data Quality Expectations
    
- Errors
    
- Execution Information
    

例如：

```
SELECT *
FROM event_log('your_pipeline_id')
WHERE event_type = 'flow_progress';
```

你原笔记中的逻辑：

```
Pipeline Event Log
       |
       v
flow_progress
       |
       v
data_quality.expectations
       |
       v
passed_records / failed_records
       |
       v
DQ Monitoring Dashboard
```

可以通过 JSON 解析和 `EXPLODE` 将 Expectation 记录转换成普通表。

例如：

```
SELECT
    origin.flow_name,
    timestamp,
    exp.col.name AS rule_name,
    exp.col.passed_records,
    exp.col.failed_records
FROM ingestion_event_log_raw
LATERAL VIEW EXPLODE(
    FROM_JSON(
        details:flow_progress.data_quality.expectations,
        'array<struct<
            name:string,
            passed_records:int,
            failed_records:int
        >>'
    )
) exp;
```

这里：

- `EXPLODE`：将 Array 转换为多行。
    
- `LATERAL VIEW`：让每条原始记录展开自己的 Array。
    
- `CROSS JOIN LATERAL EXPLODE`：在支持的 SQL 语法中实现相同的逐行展开效果。
    

### Event Hooks

Event Hook 可以监听 Pipeline Events。

例如：

```
@dp.on_event_hook(    max_allowable_consecutive_failure=3)def event_handler(event: dict):    # Inspect event type    # Send notification    # Trigger custom handling    pass
```

适用场景：

- 失败告警。
    
- 自定义通知。
    
- Pipeline Monitoring。
    
- 外部系统集成。
    

## Module 7 — 面试问题

Q1. What is the difference between imperative and declarative pipelines?

Imperative pipelines require developers to explicitly define execution steps and orchestration. Declarative pipelines define the desired datasets and dependencies, while the pipeline engine manages execution order, incremental processing and retries.

Q2. When would you use a Streaming Table instead of a Materialized View?

I would use a Streaming Table for incremental ingestion and append-heavy workloads. I would use a Materialized View for analytical queries, aggregations and joins where precomputed results improve query performance.

Q3. Can a Materialized View be incrementally refreshed?

Yes. Databricks can incrementally refresh supported queries. However, if the query or source changes are not eligible for incremental refresh, the system may perform a full recomputation.

Q4. What is the difference between an Append Flow and an AUTO CDC Flow?

Append Flow adds new records to a target streaming table, while AUTO CDC applies inserts, updates and deletes based on business keys and sequence information.

Q5. Why would you use AUTO CDC instead of manual MERGE?

AUTO CDC provides declarative handling of event ordering, deduplication and SCD Type 1 / Type 2, reducing custom code and maintenance complexity.

Q6. How do you monitor data quality in a Lakeflow Pipeline?

I would use expectations to track passed and failed records, inspect the pipeline event log, and route invalid records into quarantine datasets when necessary.

Q7. What is the difference between a Pipeline and a Job?

A Pipeline defines the datasets, flows and transformation dependencies. A Job orchestrates execution of tasks, including pipelines, notebooks, Python scripts and other workloads.

# Module 8. Jobs, Scheduling, Monitoring & Lineage

Production Operations / DataOps

这部分是从 Junior Data Engineer 到 Mid-level Data Engineer 很重要的能力差异。

不仅要会写 ETL，还需要知道如何让 ETL 在生产环境中稳定运行。

## 8.1 Databricks Jobs

Databricks Jobs（Lakeflow Jobs）用于编排和调度数据工程任务。

一个 Job 可以包含多个 Tasks。

例如：

```
Daily ETL Job
      |
      ├── Task 1: Ingestion Pipeline
      |        |
      |        v
      |     Bronze Tables
      |
      ├── Task 2: Silver Transformation
      |        |
      |        v
      |     Silver Tables
      |
      └── Task 3: Gold Aggregation
               |
               v
           Gold Tables
```

常见 Task：

- Notebook Task
    
- Python Script Task
    
- SQL Task
    
- Pipeline Task
    
- dbt Task
    
- Spark Submit Task
    

### Job Trigger

你原笔记记录了四种：

|Trigger|说明|场景|
|---|---|---|
|Schedule|定时触发|每天凌晨运行 ETL|
|File Arrival|文件到达时触发|新文件进入 ADLS|
|Table Update|Delta Table 更新时触发|上游 Table 更新后运行|
|Manual / API|手动或 API 触发|外部系统调用|

例如：

```
New File Arrives
       |
       v
Ingestion Job
       |
       v
Bronze Table Updated
       |
       v
Transformation Job
       |
       v
Gold Table Updated
```

### Task Dependencies

例如：

```
Task A
  |
  v
Task B
  |
  v
Task C
```

Task B 只有在 Task A 成功后才执行。

也可以设置并行：

```
         Task A
            |
       ┌────┴────┐
       |         |
    Task B    Task C
       |         |
       └────┬────┘
            |
         Task D
```

### Retry & Failure Handling

生产环境需要配置：

- Retry Policy
    
- Timeout
    
- Maximum Concurrent Runs
    
- Notifications
    
- Task Dependencies
    
- Failure Handling
    

例如：

```
Task Failed
     |
     v
Retry 3 Times
     |
     ├── Success → Continue
     |
     └── Failed → Alert
```

需要注意：重试机制不代表业务逻辑自动幂等。特别是外部 API、JDBC 和非事务性 Sink，需要考虑重复写入。

## 8.2 Pipeline Monitoring

一个生产 Pipeline 至少应该监控四个维度。

|Category|Metrics|
|---|---|
|Reliability|Success Rate、Failure Rate|
|Performance|Execution Time、Processing Latency|
|Data Quality|Failed Records、Null Rate、Duplicate Rate|
|Data Freshness|Last Update Time、Data Delay|

例如：

```
Pipeline Monitoring
       |
       ├── Execution Status
       |
       ├── Processing Duration
       |
       ├── Input / Output Records
       |
       ├── Failed Records
       |
       └── Data Freshness
```

### Data Freshness

例如 Gold Table 每天早上 8 点必须更新。

可以检查：

```
SELECT
    MAX(ingest_timestamp) AS latest_ingestion
FROM prod.gold.daily_sales;
```

如果数据更新时间晚于 SLA，就应该触发告警。

### Data Quality Monitoring

例如：

```
SELECT
    COUNT(*) AS total_records,
    COUNT_IF(order_id IS NULL) AS null_ids,
    COUNT(DISTINCT order_id) AS distinct_orders
FROM prod.silver.orders;
```

可以监控：

- Null Rate
    
- Duplicate Rate
    
- Invalid Values
    
- Referential Integrity
    
- Record Count Changes
    

## 8.3 Referential Integrity

你原笔记中提到：

> not exists or anti joins

这是正确的思路。

假设：

```
orders
    |
    └── customer_id

customers
    |
    └── customer_id
```

需要保证每个 Order 都有对应的 Customer。

使用 LEFT ANTI JOIN：

```
SELECT o.*
FROM orders o
LEFT ANTI JOIN customers c
ON o.customer_id = c.customer_id;
```

返回的就是找不到 Customer 的 Orders。

也可以：

```
SELECT *
FROM orders o
WHERE NOT EXISTS (
    SELECT 1
    FROM customers c
    WHERE c.customer_id = o.customer_id
);
```

这些记录可以进入 Quarantine Table 或触发数据质量告警。

## 8.4 Data Lineage

Data Lineage 用于追踪数据从哪里来，经过哪些转换，最终被哪些 Table 或 Dashboard 使用。

例如：

```
Raw Orders
     |
     v
Bronze Orders
     |
     v
Silver Orders
     |
     v
Gold Daily Revenue
     |
     v
Power BI Dashboard
```

Databricks Unity Catalog 支持 Table Lineage。

查询：

```
SELECT
    source_table_full_name,
    target_table_full_name
FROM system.access.table_lineage
WHERE target_table_full_name =
    'prod.sales.daily_revenue';
```

可以查看数据从哪些源 Table 流向目标 Table。

需要注意：

- Table Lineage 记录表级别的访问和数据流关系。
    
- Column Lineage 可以提供更细粒度的字段级血缘信息。
    
- 并非所有操作和外部系统都能自动形成完整 Lineage。
    
- Lineage 不等于完整的业务文档，也不一定记录每个 Python 函数内部的全部逻辑。
    

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on Google Cloud

## 8.5 Production Best Practices

建议把你的所有知识点最终落实到以下生产标准：

|Area|Production Best Practice|
|---|---|
|Ingestion|Incremental ingestion，避免重复全量读取|
|Schema|Schema Enforcement + Controlled Evolution|
|Checkpoint|使用持久化路径，避免共享|
|Streaming|Watermark + State Management|
|CDC|Sequence + Deduplication + Idempotency|
|Data Quality|Expectations + Constraints + Quarantine|
|Delta|合理控制 Small Files|
|Performance|Data Skipping + Liquid Clustering|
|Retention|正确配置 Log / Data Retention|
|Orchestration|Jobs + Dependencies + Retries|
|Monitoring|Logs + Metrics + Alerts|
|Governance|Unity Catalog + Least Privilege|
|Deployment|Git + CI/CD + Separate Environments|

## Module 8 — 面试问题

Q1. How would you design a production Databricks ETL Job?

I would separate ingestion, transformation and aggregation into different tasks. I would define task dependencies, configure retries and timeouts, use a service principal for execution, and monitor pipeline status, data quality and data freshness.

Q2. What is the difference between a Job and a Pipeline?

A Pipeline defines how datasets are created and transformed. A Job orchestrates the execution of different workloads and can include one or more Pipeline tasks.

Q3. How would you monitor a failed ETL pipeline?

I would inspect the Job run status, task-level logs, Spark execution details and pipeline event logs. I would also configure failure notifications and retry policies.

Q4. How would you detect data quality issues in production?

I would combine schema enforcement, expectations, duplicate checks, referential integrity validation and data freshness monitoring.

Q5. What is Data Lineage and why is it important?

Data Lineage tracks how data flows from source systems to downstream tables and reports. It is useful for impact analysis, debugging, governance and auditing.

# Module 9. 综合面试：如何设计一个完整的 Databricks Data Engineering Pipeline？

这一部分是前面所有知识点的综合应用，也是荷兰 Data Engineer 面试中很值得准备的 System Design 场景。

假设你在荷兰一家电商公司工作。

业务需求：

- Source：PostgreSQL Orders Database。
    
- 每天产生数百万条订单。
    
- Orders 会发生 INSERT、UPDATE、DELETE。
    
- 需要保留订单历史。
    
- 需要每日生成 Revenue Dashboard。
    
- 要求数据质量可监控。
    
- 需要控制云计算成本。
    

## 9.1 整体架构

## Production Data Pipeline

Source Systems

PostgreSQL · APIs · Kafka

Ingestion

Lakeflow Connect CDC · Auto Loader

## Bronze Layer

Raw Data · Original CDC Events · Source Metadata

## Silver Layer

Deduplication · AUTO CDC · SCD Type 2 · Data Quality

## Gold Layer

Business Aggregation · Materialized Views · Revenue KPIs

Consumption

Power BI · Databricks SQL · Analytics

Cross-cutting capabilities

Unity Catalog · Data Lineage · Monitoring · Lakeflow Jobs · CI/CD · Cost Optimization

## 9.2 每个 Layer 应该做什么？

|Layer|主要职责|技术|
|---|---|---|
|Bronze|保存原始数据和 CDC Events|Auto Loader / CDC / Delta|
|Silver|数据清洗、去重、业务 Key、历史记录|AUTO CDC / SCD2 / Expectations|
|Gold|业务指标和分析结果|Materialized Views / Aggregations|

### Bronze

保留原始数据：

```
order_id
customer_id
amount
operation
event_timestamp
source_sequence
ingest_timestamp
```

原则：

- 尽量保留 Source 信息。
    
- 不轻易丢弃原始数据。
    
- 保留足够的 Metadata 支持排查问题。
    

### Silver

执行：

- Deduplication。
    
- Schema Validation。
    
- CDC Processing。
    
- SCD Type 2。
    
- Data Quality Checks。
    

### Gold

例如：

```
CREATE OR REPLACE TABLE gold.daily_revenue AS
SELECT
    DATE(order_timestamp) AS order_date,
    SUM(amount) AS total_revenue,
    COUNT(DISTINCT order_id) AS order_count
FROM silver.orders
WHERE is_deleted = FALSE
GROUP BY DATE(order_timestamp);
```

如果使用 Lakeflow Pipelines，可以根据业务刷新要求选择 Materialized View。

## 9.3 生产环境中会遇到什么问题？

|Problem|Solution|
|---|---|
|Source Schema Changed|Auto Loader / Controlled Schema Evolution|
|Duplicate CDC Events|Deduplication + Sequence|
|Out-of-order Events|Event Ordering / AUTO CDC|
|Late-arriving Events|Watermark / CDC Sequence|
|Invalid Records|Expectations + Quarantine|
|Streaming Job Restart|Checkpoint|
|Small Files|OPTIMIZE / Auto Compaction|
|Slow Queries|Data Skipping / Liquid Clustering|
|Deleted Historical Files|Retention Policy / VACUUM|
|Pipeline Failure|Job Retry + Alert|
|Unauthorized Access|Unity Catalog|
|Data Impact Analysis|Lineage|

# Module 10. 面试知识优先级与最终复习清单

如果你的目标是荷兰的 Junior / Mid-level Data Engineer、Data Platform Engineer，我建议按照下面的优先级复习。

复习进度

# 0/21

第一优先级：必须理解并能写代码

0/8

Auto Loader：cloudFiles、Schema Location、Checkpoint

Schema Evolution：addNewColumns、rescued_data、mergeSchema

Structured Streaming：Micro-batch、Trigger、Checkpoint

Watermark、Window Aggregation、State Store

Delta Lake：MERGE、Schema Enforcement、Time Travel

CDC、CDF、SCD Type 1 / Type 2

Lakeflow Pipelines：Streaming Table、Materialized View、AUTO CDC

Data Quality：CHECK、Expectations、Quarantine

第二优先级：需要理解原理和取舍

0/7

Partitioning、Data Skipping、Z-Ordering

Liquid Clustering、Predictive Optimization

OPTIMIZE、VACUUM、Deletion Vectors

Transaction Log、Checkpoint、Retention

foreachBatch、Exactly-once、Idempotency

Stream-Static Join、Stream-Stream Join

Jobs、Task Dependencies、Retry、Monitoring

第三优先级：进阶知识

0/6

VARIANT、Generated Columns、Identity Columns

Type Widening、Column Mapping

Pipeline Event Log、Event Hooks

Data Lineage、Referential Integrity

Lakeflow / SDP API 和运行模式

Production Deployment、CI/CD、Cost Optimization

Reset progress

## 最后：建议重点掌握的 5 个综合问题

这五道题比单纯背 API 更能体现 Data Engineer 的实际工程能力。

1. How would you design an incremental ingestion pipeline for millions of records per day?

考查 Auto Loader、CDC、Checkpoint、Incremental Processing、Cloud Storage。

2. How would you handle schema changes in a production data pipeline without losing data?

考查 Schema Evolution、Rescued Data、Schema Enforcement、Quarantine。

3. How would you implement CDC and maintain historical records in a Delta Lake?

考查 MERGE、CDF、AUTO CDC、SCD Type 2、Event Ordering。

4. How would you optimize a Delta Table that is growing rapidly and has slow query performance?

考查 Small Files、Partitioning、Data Skipping、Z-Ordering、Liquid Clustering、OPTIMIZE、VACUUM。

5. How would you make a Databricks pipeline production-ready?

考查 Data Quality、Idempotency、Monitoring、Retries、Unity Catalog、Lineage、CI/CD。

## 官方文档：建议作为最终复习资料

以下是根据当前 Databricks 文档整理的进一步阅读入口：

|主题|官方文档|
|---|---|
|Auto Loader|[Schema Evolution](https://docs.databricks.com/aws/en/data-engineering/schema-evolution?utm_source=chatgpt.com)|
|Structured Streaming|[Checkpointing](https://docs.databricks.com/aws/en/structured-streaming/checkpoints?utm_source=chatgpt.com)|
|Watermark|[Watermarking](https://docs.databricks.com/gcp/en/structured-streaming/watermarks?utm_source=chatgpt.com)|
|Delta Lake|[Delta Lake](https://docs.databricks.com/aws/en/delta?utm_source=chatgpt.com)|
|Constraints|[Table Constraints](https://docs.databricks.com/aws/en/tables/constraints?utm_source=chatgpt.com)|
|Declarative Pipelines|[Apache Spark Declarative Pipelines](https://docs.databricks.com/aws/en/ldp/concepts/spark-declarative-pipelines?utm_source=chatgpt.com)|
|Pipeline Best Practices|[Best Practices](https://docs.databricks.com/aws/en/ldp/best-practices?utm_source=chatgpt.com)|
|Pipeline Lifecycle|[Lakeflow Pipeline Lifecycle](https://docs.databricks.com/aws/en/ldp/concepts/how-to-use-pipelines?utm_source=chatgpt.com)|
|ETL Architecture|[Choose an ETL Approach](https://docs.databricks.com/gcp/en/data-engineering/choose-etl-approach?utm_source=chatgpt.com)|

我的建议： 你不需要把所有 Databricks API 都背下来。对于荷兰 Data Engineer 面试，更重要的是能解释每个技术解决什么问题、为什么选择它、有什么限制，以及出现故障时如何恢复。特别是 CDC、Stream