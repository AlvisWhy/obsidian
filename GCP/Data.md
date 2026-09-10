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

pubsub
Pub/Sub Ordering 的代价就是：为了保证顺序，前面的消息处理失败时，同一个 ordering key 后面的消息可能被阻塞
exact once 影响速度

在topic通过data提取部分加入artifact可以后面subscriber filter， 不能直接filter message。

