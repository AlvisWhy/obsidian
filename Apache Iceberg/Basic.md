approach:

write the file -> write new metadata.json -> atomic sawp the pointer to new catalog

open lakehouse platform

open format -> open catalog(any engine can visit) -> open compute(spark, databricks read)

priorietary lakehouse: only certain vender can read
open lakehouse(parquet + iceberg): all system can read with rest catalog

atomic commits: events with hundreds of concurrent writer
evolution without rewrite: schema and partition changes without data rewrite
agnostic reads: any engine read simultaneously

iceberg diagram:
catalog pointer -> metadata.json -> manifest list -> manifest -> parquet
ice berg separate data from the catalog, delta keep the log besides the data

For schema evolution
iceberg behave better in renaming col and changing type than databricks
for databricks(have to enable column mapping mode before creating table)

Iceberg and repartition the new data within same table and keep previous partition without rewriting

iceberg is better in multi engine and schema evolving. Delta wins in merge heavy workload 


Architecture

catalog(one pointer a table) -> metadata.json(shcema + partition +history + current snapshot) -> manifest list(avro, one per snapshot) -> parquet files

manifest is where pruning
manifest: file-path, file_size, row_count, partition, lower_bound, upper_bound, null_count, status
Manifest List

第一层裁剪
利用 Manifest 的分区范围等统计信息，跳过不可能匹配的 Manifest
Manifest File

第二层裁剪
检查每个 Data File 的分区值，过滤不匹配的文件

for writing reverse the sequence. If one step fail, every previous become orphan and will be deleted.

Only the pointer changes to other metadata.json in same catalog when read old version
pruning at every layer
```SQL
CREATE TABLE catalog.db.orders (....) USING iceberg
TBLPROPERITEIS ('write.format.default'='parquet')

-- traditional balanced compression, fastest
CREATE TABLE catalog.db.orders (....) USING iceberg
TBLPROPERITEIS ('write.parquet.compression-codec'='snappy')

-- balanced small size, more cpu time encode/decode
CREATE TABLE catalog.db.orders (....) USING iceberg
TBLPROPERITEIS ('write.parquet.compression-codec'='zstd')

-- slowest for archive
CREATE TABLE catalog.db.orders (....) USING iceberg
TBLPROPERITEIS ('write.parquet.compression-codec'='gzip')
```
V-Order: write time optimization.(pre-sorted)
Z-Order: maintenacne-time optimization 

spark expose the manifest entries as a table
```SQL
SELECT
	file_path,
	partition,
	record_count,
	file_size_in_bytes,
	column_size,
	value_count,
	null_value_counts,
	lower_bounds,
	upper_bounds
FROM catalog.db.order.files

```

status: added(file created in this snapshot), existing, deleted(deleted from this snapshot but in dish)

more manifest, fast write for small commit. fewer manifest, fast read
default 8MB per manifest

Manifest List:
1 manifest list per snapshot
a snapshot is a full view of a table at certain time
```SQL
SELECT 
	commited_at,
	snapshot_id,
	parent_id,
	operation,            --append, overwrite, delete, replace
	manifest_list,
	summary               --per commit row counts, file count, added byte
FROM catalog.db.orders.snapshot
```


有时候会改metadata.json.有时候不会

|操作|通常是否创建新的 Metadata JSON|是否一定创建新 Snapshot|
|---|---|---|
|`INSERT INTO` 插入数据|是|是|
|`UPDATE` 更新数据|是|是|
|`DELETE` 删除数据|是|是|
|`MERGE INTO` 合并数据|是|通常是|
|`ALTER TABLE ADD COLUMN`|是|不一定|
|修改表属性 `SET TBLPROPERTIES`|是|通常否|
|修改分区规范|是|不一定|
|`CREATE TABLE`|创建初始 Metadata|创建时不一定有数据 Snapshot|
|`SELECT` 查询数据|否|否|
|查询历史数据 `VERSION AS OF`|否|否|
|`EXPIRE SNAPSHOTS`|通常会更新 Metadata|不一定创建新 Snapshot|
FROM the manifest list avro

```python

df.select(
	"manifest_path",
	"partition_spec_id",
	"added_data_files_count",
	"existing_data_files_count",
	"deleted_data_files_count"
).show()

```

column id are stable across renames. renaming col not influence time traveling
Iceberg 的 Data Files 不要求必须按照 partition 值存放在对应的文件夹里，因此修改 partition 规则时，通常不需要立刻重写所有历史数据文件。
spec for partition
```json
{
  "default-spec-id": 1,
  "partition-specs": [
    {
      "spec-id": 0,
      "fields": [
        {
          "name": "country",
          "transform": "identity",
          "source-id": 2,
          "field-id": 1000
        }
      ]
    },
    {
      "spec-id": 1,
      "fields": [
        {
          "name": "event_time_day",
          "transform": "day",
          "source-id": 3,
          "field-id": 1001
        }
      ]
    }
  ]
}
```

metadata.json
snapshot, parent snapshot, timestamp, manifest list, summary
ref snapshot in metadata like git branch. and use tag for snapshot as release
tag is immutable, branch is mutable
ref for version
```json
{
  "current-snapshot-id": 105,
  "refs": {
    "main": {
      "snapshot-id": 105,
      "type": "branch",
      "min-snapshots-to-keep": 3,
      "max-snapshot-age-ms": 604800000
    },
    "release-v1": {
      "snapshot-id": 90,
      "type": "tag"
    }
  }
}
```

For concurrent write
read pointer -> write data to file -> write metadata -> CAS attempt(if CAS fail, back to write) -> visible to reader(commit complete)

CAS = Compare-And-Swap，可以理解为：
> 只有当当前值仍然等于我之前读到的值时，才允许我把它更新成新值；否则更新失败。

在 Iceberg 中，这通常属于乐观并发控制（Optimistic Concurrency Control）：
- 读取时不必先锁住整张表。
- 提交时检查基础版本是否仍然有效。
Lock is on CAS itself for miliseconds, so multiple writer processed in parallel. And conflid is rare within multiple partitions

第一层：Catalog CAS
检查当前表的元数据版本是否仍然符合提交时的预期。版本变了，旧提交不能直接覆盖新版本。

第二层：Iceberg 冲突验证
检查当前表是否仍然满足操作的前提，例如源文件是否还在、是否有并发写入影响本次删除或更新。
因此：CAS 失败可以通过刷新和重试解决；真实的数据冲突则可能必须让操作失败，或者重新执行整个操作。

V1 spec , V2 spec
let you change individual row without rewriting whole file
v1 append and overwrite only, best for append only, batch overwrite,
v2 add small elete files, default for new tables Like deletion vector

2 的 Delete Files 主要有两种：

- Position Delete：记录数据文件路径及行位置，删除特定行。
- Equality Delete：记录用于匹配删除行的列值，例如 `transaction_id = 'T100'`。
```SQL
ALTER TABLE sales.order SET
TBLPROPERITIES('format-version'='2')
```

Schema evolution and partition change

Data in parquet saved with col id, not col name
```SQL
-- Add new column
ALTER TABLE orders ADD COLUMN region STRING

-- rename col and col not change, so data not change
ALTER TABLE orders RENAME COLUMN amount TO revenue

-- Type promotion, 不能降低精度可以提升
```


```SQL
CREATE TABLE t (...) PARTITION BY (region)

-- Bucket: hash based, fixed cardinality
CREATE TABLE t (...) PARTITION BY (bucket(16, user_id))

--Truncate: prefix based
CREATE TABLE t (...) PARTITION BY (truncate(8,sku))

-- combine multiple transforms
CREATE TABLE t (...) PARTITION BY (day(ts), truncate(8,sku))
```

target 1GB ~ 100GB per partition
low cardinality: identity
medium cardinality: days(ts) or bucket
high cardinality: must bucket/ truncate

```SQL
ALTER TABLE orders ADD PARTITION FIELD days(order_date)

ALTER TABLE orders DROP PARTITION FIELD months(order_date)

SELECT spec_id, count(*) AS files
FROM sales_orders.files
GROUP BY spec_id

```

Sort tables
```SQL
-- ONLY future write will be sorted
ALTER TABLE orders WRITE ORDER BY(order_date, customer_id)

SELECT sort_order_id, count(*) AS files
FROM sales_orders.files
GROUP BY sort_order
```

SORT ORDER VS ZORDER

- Sort Order：按照指定列的顺序排序，例如先按 `country`，再按 `event_time`。更适合固定filter在很少的col上
    
- Z-Order：将多个列的值映射到多维空间中的 Z-order 曲线，让不同列的值在物理布局上尽可能具有局部性。适合多col的filter

measure the pruning after zroder, if improve over 30%, it is necessary for zordering

Rest api for open lakehouse

3 layers of API
1 namespace
Domain.layer/team.features pattern for naming convention

GET  /v1/namespaces
GET  /v1/namespaces/{namespace}
DELETE  /v1/namespaces/{namespace}
POST /v1/namespaces
(use . for nested namespaces)

2 tables
CREATE TABLE

ALTER TABLE

DROP TABLE

RENAME/COMMIT TABLE


3 commits

example body
if all requirements met, will update. If fail, reject (CAS)
```json
{
	"requirements": [
		{
			"type": "assert-current-snapshot-id",
			"snapshot-id": 12345
		},
	]
	"updates"[
		{
			"action": "set-snapshot",
			"snapshot": { ... new snapshot ... }
		}
	]
}

```

curl http://catalog/v1/config
curl http://catalog/v1/namespace

identity mode
1 federated, user token passed through, catalog enforeces per-user RBAC
2 service principle, engine has its own identity, catalog grants on the service
3 per-query token, 

configuration with spark

1 set spark conf
spark.sql.catalog.my_cat    = org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.my_cat.type    =rest
spark.sql.catalog.uri  = ....
spark.sql.catalog.token = ....
spark.sql.catalog.warehouse = s3://my-bucket

2 run query
```python
spark.sql("CREATE NAMESPACE local.demo")
spark.sql(
	"""
		CREATE TABLE local.demo.orders(
			id BIGINT,
			amount DECIMAL(10,2),
			region STRING
		)USING iceberg
		PARTITION BY(region)
	"""
)

```

Apache Polaris REST Catalog 是接口标准，Polaris 是实现这个标准的服务之一，管理多系统操作iceberg的catalog
REST Catalog · 权限控制 · 元数据提交协调

unity catalog oos(for databricks)

the basic dataframe
```python
# apend rows 
df.writeTo('catalog.schema.orders').append()

# Overwrite rows by predicate
df.writeTo("catalog.schema.orders") \
  .overwrite(col("region") == "EU")

# Overwrite only the partitins touched by df
df.writeTo('catalog.schema.orders').overwritePartitions()

# Automic replace - drops history references
df.writeTo('catalog.schema.orders').replace()


# pre-create the branch 
spark.sql("ALTER TABLE catalog.schema.orders" "CREATE BRANCH wap_branch")
# write the dataframe to the branch, main is untouched
df.writeTo("catalog.schema.orders.branch_wap_branch").append()

# forward main to wap_branch
spark.sql("""
	CALL catalog.system.fast_forward("catalog.schema.orders",'main,'wap_branch')
""")

# Trace the run_id for certain snapshot
df.writeTo('catalog.schema.orders')
	.option("snapshot.summary.run-id", "abc-123")
	.option("snapshot.summary.pipeline", "bronze_etl")
	.append()
```

Create temp branch before going to prod

```SQL
ALTER TABLE catalog.schema.orders
CREATE BRANCH wap_branch FROM main

SET spark.wap.branch = 'wap_branch'

INSERT INTO catalog.schema.orders
SELECT * FROM staging.orders_today

SELECT COUNT(*) AS row_count
FROM catalog.schema.orders.branch_wap_branch

SELECT (SELECT SUM(total) FROM orders.branch_wap_branch) AS new_total
	   (SELECT SUM(total) FROM orders) AS old_total
	   
-- set main to the branch stat
ALTER TABLE catalog.schema.orders
	SET CURRENT BRANCH wap_branch

-- fast forward main
-- fast_forward(table, target_branch, source_branch)

CALL catalog.system.fast_forward(
	'catalog.schema.orders',
	'main',
	'wap_branch'
)

-- Add tag on current snapshot
ALTER TABLE prod.db.orders
CREATE TAG `prod_v1`
-- Add tag for target version
AS OF VERSION 125
RETAIN 365 DAYS;

-- Read by tag
SELECT * FROM orders.tag_prod_v1

--drop tag
ALTER TABLE catalog.schema.orders
DROP TAG prod_v1

ALTER TABLE catalog.schema.orders
DROP BRANCH wap_branch

```
Add tag for audit checkpoint,
add tag before risky operations

compaction on write
```SQL
ALTER TABLE catalog.schema.orders
SET TBLPROPERITIES (
	'write.target-file-size-byte' = '536870912'
)
ALTER TABLE catalog.schema.orders
SET TBLPROPERITIES (
	'write.distribution-mode' = 'hash' -- hash/range/none
	'write.spark.fanout.enabled' = 'true' 
)
```
Fanout Writer 可以避免某些写入场景中为了满足分区顺序而执行的额外 shuffle 或排序。
partitionBy()` 决定数据写到哪个分区目录；`repartition()` 决定数据如何重新分配到 Spark 执行分区。 对 Iceberg 而言，`write.distribution-mode` 又提供了表格式层面的写入分布要求。

For streaming jobs
use checkpoint to combine multiple workers and use x minute checkpoint to submit the data to iceberg
daily compaction to deal with small files

run time compaction/ daily schedule compaction/ auto compact

Delete methods

1 Position Delete
已经知道目标行在哪个文件、哪个位置时，很适合直接记录删除位置。
例如：读取数据文件后，执行某个更新操作，已经确定要替换的行。
read cheap， write cost much。
删除的时候写文件指向删除目标的index。

2 Equality Delete
更适合用列值描述要删除的记录，不需要把原文件行号作为删除条件。
例如：CDC 消息告诉引擎需要删除 `id=102`，引擎可根据 `id` 作为 equality field 生成删除记录
write cheap，read cost much。可定时清理

Debezium setup for cdc

postgreSQL 使用的是 WAL（Write-Ahead Log）+ Logical Decoding（逻辑解码）。
```SQL
ALTER SYSTEM SET wal_level = 'logical' --提供逻辑解码所需的信息，支持逻辑复制和 CDC。
ALTER SYSTEM SET max_replication_slots = 10 --设置最多可以同时存在多少个 replication slots
ALTER SYSTEM SET max_wal_sender = 10 --设置最多可以同时运行多少个 WAL sender 进程

CREATE PUBLICATION debezium_pub FOR ALL TABLES
SELECT pg_create_logical_replication_slot('debezium', 'pgoutput')
```
postgres -> Debezium -> Avro/json in schema registry-> Kafka topic -> iceberg table
check schema registry for schema consistency

For mergeing
1 filter by latest time ranking, merging is heavy computing task
2 schema may mismatch after the evolution

predicate pushdown and projection pushdown

|对比|Predicate Pushdown|Projection Pushdown|
|---|---|---|
|优化对象|行|列|
|对应 SQL|`WHERE`|`SELECT`|
|目的|减少需要处理的行|减少需要读取的列|
|典型例子|`WHERE country = 'NL'`|`SELECT order_id, amount`|
|Parquet 优化方式|统计信息、过滤器、跳过 Row Group 等|只读取需要的列数据|

what does not pushdown
complex UDF predicate
subquery predicates
cross row agg like lead/lag
predicate on derived col

1 partition pruning 
2 file stats pruning
3 row group pruning。Parquet 文件的元数据记录了每个 Row Group 的 `id` 最小值和最大值。
为了能够正常pruning，避免filter的时候使用函数包裹
use explain to check pruning（只能看partition和projection pruning的情况，无法看到file/row
group pruning）

`EXPLAIN`查看基本物理执行计划
`EXPLAIN EXTENDED`查看 Parsed、Analyzed、Optimized、Physical 等计划阶段
`EXPLAIN FORMATTED`以更易读的结构展示物理计划及节点详情
`EXPLAIN COST`查看优化计划及可用的统计信息、估算成本

```
== Physical Plan ==
*(1) Project [customer_id#12, amount#13]
+- *(1) Filter (isnotnull(amount#13) AND (amount#13 > 100))
   +- *(1) ColumnarToRow
      +- BatchScan orders [customer_id#12, amount#13, order_date#10, country#11]
         RuntimeFilters: []
         DataFilters: [isnotnull(country#11), (country#11 = NL),
                       isnotnull(amount#13), (amount#13 > 100)]
         PartitionFilters: [(order_date#10 >= 2026-10-01),
                            (order_date#10 < 2026-11-01)]
         PushedFilters: [IsNotNull(country), EqualTo(country,NL),
                         IsNotNull(amount), GreaterThan(amount,100)]
         ReadSchema: struct<customer_id:bigint,amount:double,
                            order_date:date,country:string>

```
`PartitionFilters`
有分区过滤
查看是否出现分区列的范围或等值条件。它是判断 Partition Pruning 的重要线索。
`PushedFilters`
过滤条件已下推
查看 `EqualTo`、`GreaterThan` 等条件是否被传递给数据源；它不直接证明实际跳过了多少数据。
`ReadSchema`
检查列裁剪
看扫描阶段读取了多少列，以及有没有不必要的宽列。
`Exchange`
检查 Shuffle
如果计划里出现 `Exchange hashpartitioning(...)`，通常意味着需要重新分布数据，例如为了 Join 或聚合。单纯的过滤查询一般不需要 Shuffle。

iceberg metadata tables
mytable.snapshots     get every snapshot
mytable.history     get full commit log
mytable.files         get every data file with col stats, can get file_size_in_byte for table

avg size below 100MB is a problem

mytable.partitions        get partition-level summary
mytable.manifests         get manifest level stats and counts

Time Travel
Keep at least one snapshot and expire snapshots older than 5 days
SET TBLPROPERTIES: history.expire.max-snapshot-age-ms

```
SELECT .... VERSION AS OF 1234
SELECT .... TIMESTAMP AS OF "2026-04-01 12:00:00"
SELECT .... FROM orders.branch_audit
```

To rewrite table to solve small files

```SQL

CALL system.rewrite_data_files(table => 'orders', strategy => 'binpack')

CALL system.rewrite_data_files(
    table => 'orders',
    strategy => 'sort',
    options => map(
        'target-file-size-bytes', '268435456'
    )
);
```

- `binpack`(daily)：主要将较小的数据文件合并、重新组织成接近目标大小的文件，减少小文件问题。
no reorder, cost lowest
- `sort`(weekly)：按照表的排序规则对数据排序后重写文件，除了改善文件大小，还可能改善查询时的文件裁剪效果。
- z-order for multiple column dimensional pruning while filtering. need more compute. 
```SQL
CALL system.rewrite_data_files(
	table   =>   'orders',
	strategy =>   'sort',
	sort_order => 'zorder(customer_id, product_id)'
	where => 'event_date >= current_date - interval 30 days'
)

```

used not more than 4 cols. used better for high cardinal cols.
not used for partition columns

```SQL
ALTER TABLE orders WRITE ORDERED BY (timestamp, region)

CALL system.rewrite_data_files (
	table => 'orders',
	strategy => 'sort',
	sort_order => 'timestamp, region',
	where => 'event_time < current_date -interval 7 days'
)

```

Clear data

Clear snapshot older than a week
```SQL
CALL system.expire_snapshots(table=>'order', older_than=>now()-interval 7 days)

-- or set history.expire.max-snapshot-age-ms=604800000
-- or set history.expire.min_snapshots-to-keep = 5
```

只要 Branch 仍然引用某个 Snapshot，Iceberg 的 `expire_snapshots` 就会考虑这个引用，不会简单地因为 Snapshot 过期就把它删除。Branch和tag用于保护snapshot不被删除
```SQL
-- Drop unused branches after retention, can check which branch is not cleared
SELECT * FROM mytable.ref WHERE branch_age > 30 days
```
Orphan files causes:
1 failed commits 2 crashed jobs 3 concurrent retries 4 buggy rewrite
```SQL
CALL system.remove_orphen_files(table=>'orders', older_than=>now() - interval 7 days)

```
Set the threshold for in flight writing files which are not committed 

mainfest count could slow down operations
```SQL
CALL system.rewrite_manifests(table=>'order')
-- Combine small files into targeted size manifests default 8MB per file

-- check the mainfest counts
SELECT count(*) FROM mytable.manifests
-- should not be above 1000 per table 
```

Rollback
```SQL
CALL system.roll_back_to_snapshot(table=>'orders', snapshot_id =>12345)

CALL system.roll_back_to_snapshot(table=>'orders', timestamp =>'2026-04-01')
```