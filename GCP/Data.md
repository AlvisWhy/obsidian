Dataflow
transformation tool, ETL flow, serverless dataproc
pub sub/ gcs ---> dataflow ---> BQ
ETL process data than load in the BQ
ELT use bq SLQ to process the data

Datastream
read transaction log from DB for CDC.
with no lantency， from all kinds of database to BQ/Cloud Storage
MySQL / PostgreSQL
        │
        │ CDC
        ▼
   Datastream
        │
        │ change events
        ▼
    Dataflow
        │
        ├── 处理 INSERT
        ├── 处理 UPDATE
        ├── 处理 DELETE
        ├── 去重 / 排序
        ├── 数据转换
        └── SCD Type 1 / 2
        │
        ▼
    BigQuery

Sensitive Data Protection(DLP before)
De-identify the PII before storing or scanning data in storage

DataStore/FireStore
Use index.yaml create index for complex search.

BigQuery User: (roles/bigquery.user) When applied to a dataset, this role provides the ability to read the dataset's metadata and list tables in the dataset. When applied to a project, this role also provides the ability to run jobs, including queries, within the project. A principal with this role can enumerate their own jobs, cancel their own jobs, and enumerate datasets within a project. **Additionally, allows the creation of new datasets within the project; the creator is granted the BigQuery Data Owner role(roles/bigquery.dataOwner) on these new datasets.** Lowest-level resources where you can grant this role: Dataset BigQuery Job User: (roles/bigquery.jobUser) Provides permissions to run jobs, including queries, within the project. Lowest-level resources where you can grant this role: Project

Datastore

key: json, can have multiple keys
properties: json

Cloud SQL expansion
Storage: set auto-storage-increase
cpu: set alert and manual change
could partition according to column, but no sharding support

For SQL, binary should be set True for Point in Time Recovery

For minimal down time in on-prem -> cloud sql migration:
1 open binlog for on-prem sql
2 migrate all data to cloudsql
3 increment replication from onprem to cloudsql(replay binlog). When data become consistent, completely switch to cloudsql

Bigquery can be partitioned by date for tables, set partition expiration could auto delete outdated data.

View in bigquery:
just SQL logics, not real data
materialized view will store the data

Cloud DLP(Data loss protection)
Can scan or mask PII data

                 Sensitive Data
                       │
                       ▼
              ┌─────────────────┐
              │   Discovery     │
              │ 找在哪里         │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Classification  │
              │ 是什么类型？     │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   Protection    │
              │ 怎么保护？       │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Masking      Tokenization  Redaction

Dataprep: graphic data washing, less used
Dataprex: better tool to operate on data.

BIGTABLE:
storage depends on row key to divide data to different nodes
uneven row key will cause latency with hotspot problem
Row key: search reference
Column Family: large category (should not be too many and confined after created)
Column qualifier: concret classification

选择routing
1 Single-cluster routing

例如：
```
Export Job
    ↓
export-profile
    ↓
cluster-us
```

这样你可以把 workload 隔离开。由于读写都在同一个 cluster，可以获得 read-your-writes consistency。

2 Multi-cluster routing
让 Bigtable 自动选择合适的 cluster。

Write
 ↓
cluster-us

马上 Read
 ↓
cluster-us

选择priority
**Bigtable App Profile** 理解成一个“**这类应用访问 Bigtable 时，应该怎么访问、访问哪个 cluster、用什么优先级**”的配置。
给线上应用profile high
给道出数据 low

一个实际项目可能这样设计

```
Bigtable Instance
│
├── cluster-us
└── cluster-eu
```
### Game Profile

```
game-profile

Routing:
    Multi-cluster

Priority:
    HIGH
```

### Export Profile

```
export-profile

Routing:
    Single-cluster

Cluster:
    cluster-us

Priority:
    LOW
```

### Backend Profile

```
backend-profile

Routing:
    Multi-cluster

Priority:
    MEDIUM
```

普通业务请求使用。

### Embedding database in BQ

文本
 ↓
生成 Embedding
```SQL
CREATE OR REPLACE MODEL `bqml_lab.embedding_model`
  REMOTE WITH CONNECTION DEFAULT
  OPTIONS (ENDPOINT = 'text-embedding-005');
```
 ↓
存进 BigQuery
```SQL
CREATE OR REPLACE TABLE `bqml_lab.embeddings` AS
SELECT * FROM ML.GENERATE_EMBEDDING( MODEL `bqml_lab.embedding_model`,
  (    SELECT
   title,
   url,
   abstract AS content
 FROM
   `bqml_lab.patent_data`
 LIMIT 200000))
WHERE LENGTH(ml_generate_embedding_status) = 0;
```
 ↓
CREATE VECTOR INDEX
```SQL
CREATE OR REPLACE VECTOR INDEX my_index
ON `bqml_lab.embeddings`(ml_generate_embedding_result)
OPTIONS(index_type = 'IVF',
  distance_type = 'COSINE',
  ivf_options = '{"num_lists":500}');
```
 ↓
IVF (Inverted File Index)+ COSINE


相当于提前把：

```
100万个 vectors
```

整理成：

```
Cluster 1
Cluster 2
Cluster 3
...
Cluster 500
```

搜索的时候：

```
Query
 ↓
先判断最接近哪个 cluster
 ↓
只搜索相关 cluster
 ↓
找最相似 vectors
```
 ↓
VECTOR_SEARCH()
```SQL
SELECT table_name,
  index_name,
  index_status,
  coverage_percentage,
  last_refresh_time,
  disable_reason
FROM `qwiklabs-gcp-02-db04adbb32b7.bqml_lab.INFORMATION_SCHEMA.VECTOR_INDEXES`;
```
 ↓
找到语义最相似的内容


ARRAY AGG in BQ
```SQL
SELECT
  fullVisitorId,
  date,
  ARRAY_AGG(v2ProductName) AS products_viewed,
  ARRAY_AGG(pageTitle) AS pages_viewed
  FROM `data-to-insights.ecommerce.all_sessions`
WHERE visitId = 1501570398
GROUP BY fullVisitorId, date
ORDER BY date
```

- finding the number of elements with `ARRAY_LENGTH(<array>)`
- deduplicating elements with `ARRAY_AGG(DISTINCT <field>)`
- ordering elements with `ARRAY_AGG(<field> ORDER BY <field>)`
- limiting `ARRAY_AGG(<field> LIMIT 5)`

```SQL
SELECT DISTINCT
  visitId,
  h.page.pageTitle
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`,
UNNEST(hits) AS h
WHERE visitId = 1501570398
LIMIT 10
```

- You need to UNNEST() arrays to bring the array elements back into rows
- UNNEST() always follows the table name in your FROM clause (think of it conceptually like a pre-joined table)

A STRUCT can have:
- One or many fields in it
- The same or different data types for each field
- It's own alias

materialized view


BQ wirte:

||Storage Write API|Data Transfer Service|
|---|---|---|
|本质|**写入 API**|**数据搬运服务**|
|谁调用|Application / Dataflow 等|Google 托管服务|
|典型模式|Streaming|Batch / Scheduled|
|实时性|高|通常较低|
|是否需要自己写 pipeline|通常需要|很少|
|典型例子|Dataflow → BigQuery|Google Ads → BigQuery|
|重点|**怎么写进 BQ**|**怎么把数据搬到 BQ**|

- BigQuery Data transfer Service is a "set it and forget it" way to ingest data from supported third-party sources.

Default / At-least-once
→ retry 可能 duplicate

Committed Stream + offset
→ exactly-once
→ offset 已提交，重复提交会被识别

BQ CDC
`max_staleness` 给 BigQuery 一个“可以接受多旧”的窗口，让系统通过定期后台应用 CDC changes，减少查询时的 runtime merge。

BQ CDC 主要连 DB->datastream->BQ CDC
如果没有datastream需要其他cdc手动把db数据处理成cdc格式 然后给bq
DB ->Debezium / CDC logic -> Kafka / Pub/Sub ->  BigQuery

 Scheduled Query

```
10:00 → 执行 SQL → 结束
10:05 → 执行 SQL → 结束
10:10 → 执行 SQL → 结束
```

Continuous Query
10:00 │ ├── SQL 持续运行 ├── 新数据 → 处理 ├── 新数据 → 处理 ├── 新数据 → 处理 
SQL 持续处理不断进入 BigQuery 的 streaming data，而不是执行一次就结束。

**Reverse ETL**
Reverse ETL is the process of moving data from a storage system, like BigQuery, back into operational systems. This allows you to activate your data by making it available in the tools your business teams use every day like CRMs, marketing automation platforms, and advertising networks.
It is based on continuous query, can push query results to pussub bigtable another bq or spanner

Modifying the table's properties with **--require_partition_filter** is BigQuery's native and most direct way to enforce partition filtering. It's a table-level setting that causes any query against that specific table to fail if it doesn't include a predicate on the partitioning column, thereby preventing accidental full table scans and ensuring cost/performance control.

BQML:
Model preparation -> model training -> model evaluation

object table enable BQ visit multi modal data like text and image: 
```SQL
''' BUILD DATASET'''
CREATE EXTERNAL TABLE
	``
WITH CONNECTION ``
OPTIONS(
	object_metadata = ''
	uris = ['gs://.....']
	 
)
 '''CREATE MODEL'''
CREATE OR REPLACE MODEL ``
REMOTE WITH CONNECTION ``
OPTION (endpoint = 'gemini-pro')

'''USE MODEL'''
ML.GENERATE_TEXT(
MODEL project_id.dataset.model,
(
	SELECT social_media_resource, customer_id, location, review_text, CONCAT(
		'Classify the sentiment of the following text as pos or neg' AS prompt
		FROM `gemini_demo.customer_review`
	)
),
STRUCT(
	prmompt AS prompt
	[...]
)
)
```
