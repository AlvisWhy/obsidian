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
iceberg behave better in renaming col and chaing type than databricks
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