Worker -- Driver -- Cluster Manager 
1.Cluster Manager create the driver
Create container called Application master on driver
2.Driver request cluster manager to create worker machine. 2 containers created on the worker node.
3.Communication start between driver and worker node

每个 Spark Application（通常就是一个 Job 提交）都有自己的 Driver。Executor 可以有很多个，而 Driver 只有一个。

| 层次       | Master / Worker   | Driver / Executor                           |
| -------- | ----------------- | ------------------------------------------- |
| **属于谁**  | 集群（Cluster）       | Spark Application                           |
| **表示什么** | **机器（Node）或节点角色** | **运行进程（Process）**，在 YARN 上通常运行在 Container 中 |
1 core works on one partition at a time

**Driver Node** has the python wrapper and java wrapper, after wrapping, send the code to the spark core.
With pyspark
Pyspark Driver ----- > Application Driver(JVM)
With scala
Only Application Driver(JVM)

If using UDF or other pycodes，the **worker node** will each have a python workspace.
Else Worker Node do not need py env


Transformations（lazy）
Narrow: select(), filter(), withColumn(), drop
Can be parallel on data partitions

Wide: join(), agg()
broadcast join is not wide operation
perform after data grouped from multiple partitions

Actions
read() write() collect() take() show() foreach()

1 Each Action create a new job, driver create a plan

2 Driver will break the plan after each wide dependency, if no wide operation, only one stage.

3 The last operation of the stage will be the wide transformation, stage run one after another,

4 Data shared between stage with shuffle sort

5 multiple tasks in one job according to the partition

6 The tasks send the data back to the driver

**spark session/ spark context**s

spark context is the entry point to spark application It is in the driver node

Spark Context, SQL Context, Hive Context
With spark 2.0, we combined all of these context, using spark session

Code(narrow/ wide operation) -> optimization(in driver node)-> actions.      (Lazy evaluation)

DRIVER: Validate Analysis-> Logic plan -> optimized logic plan -> physical plans -> cost model chose a plan
-> WORKER 
```
DATAframe.explain()
```
output the physical model

Spark ecosystem

DataFrame is a dataset with datatype of row. dataset is type safety, it has defined type of data.

RDDs are immutable , fault tolerant, resilient
distributed, does not have schema, just data structure

RRD does not have scheme, so could not apply optimization
do not need to tell how to perform with dataframe.

JOB -> STAGE(similar transformation will be combined) -> TASK

```
spark.read.format("csv")\
		.option("header", True)\
		.load("")
		
# repartition
df = df.repartition(2)

# coalesce(narrow transform, no shuffle)


```

By default, spark create 200 partitions for wide transform

repartition(numofpartitions, cols). Hash based partition
repartitionByRange(numofpartitions, cols) Range of values based partitioning. Using data sampling for partition ranges.

Use case:
Dataframe reuse and repeated column filters
Dataframe partitions are not well distributed
Dataframe partition is skewed

Colease
Combine the partitions
Does not cause shuffle sort. Combine local partitions only. Will not increase the number of partitions. Can case skew.

AQE (adaptive query execution)

1 coalesce small tables
==spark.sql.adaptive.enabled==
==spark.sql.adaptive.coalescePartitions.initialPartitionNum==
==spark.sql.adaptive.coalescePartitions.minPartitionNum==
==spark.sql.adaptive.coalescePartitions.enable==
==spark.sql.adaptive.advisoryPartitionSizeInBytes==

2 exchange stage:
hash partition(separate by key )/ range partition(seperat by range)
Shuffle write(separate data in the local node) -> exchange(edge) -> Shuffle read(acquire required data)
one task -> one partition

when group by, first add partition id, conduct local count to reduce data transformation, then shuffle and do final count 

3 Mange Data skew
==spark.sql.adaptive.enable = true==
==spark.sql.adaptive.skewJoin.enable = true==
==spark.sql.adaptive.skewJoin.skewedPartitonFactor=5==. Consider skew 5 times than median 
==spark.sql.adaptive.skewJoin.skewedPartitonThresholdInBytes = 256MB== Consider skew 5 times than 256 MB

JOINS
Shuffle Sort Merge Join
hash(key) to decide new partition locations. shuffling part is done, rearrange between executors. Same join key items will go to the same partition.
apply sorting before the final join 

Shuffle hash join
shuffle part the same
After shuffling, will create a hashtable for one of the joiners, it is in memory, the dataset should be small and it will be quick.

Broadcast Join
One joiner size is very small. The small join data will be replicated to each executor. Sharing will take place, no shuffling

With AQE enabled, for each stage the spark will recalculate the join table whether they should be broadcast joined.

```
df_broad = df1.join(boradcast(df2), df1['id'] == df2['id]', 'inner')
```
**Local Shuffle Reader** 是 AQE 的一个优化。

> **如果 Shuffle 后的数据已经在本地 Executor 上，Spark 就直接本地读取，而不是通过网络再拉一次数据。**
reduce the unused partitions

----------------------------------------------------

Memory Management & Deployment

### Driver Memory. 
--driver-memory 2G
1 JVM Heap Memory (spark.driver.memory), do everything, scheduling, metadata, query plans, .show()/ .collect(), broadcasting

2 Overhead Memory(spark.driver.memoryOverhead) 10% or 384MB, for Object Memory

Memory Overhead 是 Spark 为 Executor/Driver 预留的非 JVM Heap 内存。它主要用于 Python Worker、Shuffle/Netty 网络缓冲区、JVM Off-Heap 内存以及 Native Libraries（如 Pandas、NumPy、Arrow）。如果 Overhead 不足，即使 JVM Heap 没用满，Executor 也可能因为超出容器内存限制而被杀掉。

Driver OOM
df.collect, get all of the data as a list, all of the partition will come to the driver node. 

### Executor Memory
--exectuor-memory 10GB --num-executors 10
1 Heap Memory: Main Memory of Executor 10GB( Main memory for executor) ==spark.executor.memory== 
 all of the execution
2 Overhead Memory ==spark.memory.memoryOverhead==,10% or 384MB
 Non JVM tasks.  
3 OffHeap Memory (0 by default) Lives outside the JVM, Store serialized data, helps in GC overhead, helps in persist spark. ==spark.memory.offHeap.enable==. ==spark.memory.offHeap.size==
Mix offheap and heap memory can help reduce GC delays. It can also serve as memory of executor memory pool

4 Pyspark Memory (0 by default) Serialized and Deserialized between python and JVM
如果额外设置这个 pyspark在pyspark memory 如果不额外设置 就在overhead memory里面分

JVM Heap has 3 Components
1 Reserved Memory 300MB, For spark Engine itself
2 Spark Memory 60% of the remaining memory, ==spark.memory.fraction==
1)  Storage Memory (50% by default, dynamic) Used for caching ==spark.memory.storageFraction==
2)  Execution Memory (50% by default) Used for actual execution
Spark Pool Memory
storage memory can not evict the data from execution memory
Execution memory can evict the data from storagemomory
But if caching data is being used by any process, can not evict storage memory. Split the data to the disk.

Spilling to Disk
When data size is larger than the pool memory, the data will be splited to the disk.

3 User Memory 40% of the remaining memory (UDF, user defined datastructures, spark metadata). If not use UDF or user defined data, this can be set less

So nearly only 30% for executor RAM

Executor OOM
When executor can spill the data to the disk, why will OOM?
Data is highly skewed, driver can not execute the data.(major resaon)

Deployment Mode & Edge Node
Without edge node
dev submit job to the cluster manager

With edge node
machine as the edge node to access the cluster manager, dev submit the code to the edge node, and edge submit code to the cluste manager.

Deployment modes
1 cluster mode, cluster manager choose a node from the cluster as the driver node
2 client mode, create the driver node on the edge node. driver dos not live in the cluster
3 local mode, everything is running locally. 

AQE adaptive query execution
added to spark 3.0 as optimization layer
1 dynamically coalesce the partitions.
combined the partitions with small number of data.
all tasks could be finished at the almost same time.

2 dynamically optimize the skewness.(the data size is at least 5 times the median, the skewed data must be at least than 256MB)
Too large partition will be splited

3 dynamically improves the join strategy.
According the size of dataframe, the AQE can change the join strategy.
physical plan --->(AQE)--->  final plan

-------------------------------------------------------------------------------
Cache & Persist
Lazy operation
Caching is a special type of persist, to save the data in memory.

for RDD storage is memory only. Using DataFrames, default is Memory_AND_DISK
Persist helps us to persist the data in memory and disk (as per our needs)
df size is larger than the storage memory, the data will be store in storage memory, the rest of data will be **recalculated**(Memory_Only) or **stored**(Memory_And_Disk) in disk.

Storage Levels
1 MEMORY_Only df.persist(StorageLevel.MEMORY_ONLY)
2 MEMORY_AND_DISK df.persist(StorageLevel.MEMORY_AND_DISK) = df.cache()
3 DISK_ONLY df.persist(StorageLevel.DISK_ONLY)
4 MEMORY_OLNY_2 , 2 times stored in memory, fault tolerance
5 MEMORY_ONLY_SER, only in scala stored as serialized data in memory.
6 MEMORY_AND_DISK_SER, only in scala stored as serialized data in memory and disk.
7 OFF_HEAP store outside JVM df.persist(spark.memory.offHeap.enable = True)
UNCached
	df.unpersist()

Partitions & Pruning
Partitions in pyspark

```
df = df.read.format("csv")\
		.option()
		.load("../../..")

df_part = df.withColumn("year", year(col("date")))\
			.withColumn("month", month(col("date")))

df_part.write.format("parquet")\
		.mode("append")
		.partitionBy("year", "month")
		.option("path", "../../../")
		.save()
```

Dynamic Partition Pruning
## 什么是 Dynamic Partition Pruning（DPP）？

> **在 Join 时，根据小表中的值，动态过滤大表不需要读取的 Partition。**

流程可以总结为：

1. **先过滤小表**
2. **得到 Join Key 的值集合**
3. **把这个值集合下推到大表扫描**
4. **如果大表按这个 Join Key 分区，就可以直接跳过大量 Partition**

#### Predicate Pushdown: 

First filter operation and dynamically optimize the sequence
→ 把 WHERE 条件推到数据源，存储层直接过滤
包括 patition pruning/ parquet orc filter/ row filter
简单列 常量比较可以下推（包括前缀匹配），计算跨表转化函数不能
不能pushdown
```SQL
WHERE a.price * b.rate > 100 
WHERE UPPER(name) = 'ALICE' 
WHERE CAST(id AS STRING) = '123'
LIKE '%keyword%'
```
So could only read the relevant files.
Get the filter from the unpartitioned dataframe, with broadcast exchange,
the join key of the broadcasted table should be the partitioned column of the table

Salting for skewness
add another column and give different val.

```
df = df.withColumn("salt_col", rand(3))
df_salt = df.withColumn("salt_group", concat(col("id"), lit("_"), col("salt_col")))
```

Salting on join
For the small table, add salt and duplicate each value on each salt val.
```
df_small = df.withColumn("salt_array", array([lit(i) for i in range(3)]))\
		.explode(col("salt_array"))
df_join = df_salt.join(df_small, df_salt["id"] == df_small['id'], 'left')
```
Hint

|   |   |
|---|---|
|`broadcast`|强烈建议使用 Broadcast Join（最常用）|

|         |                      |
| ------- | -------------------- |
| `merge` | 建议使用 Sort Merge Join |

|   |   |
|---|---|
|`shuffle_hash`|建议使用 Shuffle Hash Join|

|   |   |
|---|---|
|`repartition`|指定 Shuffle 后 Partition 数|

|   |   |
|---|---|
|`coalesce`|合并 Partition，减少 Task/输出文件|

|                        |         |
| ---------------------- | ------- |
| `repartition_by_range` | 按范围重新分区 |

### `REBALANCE`

```
df.hint("REBALANCE")
```

- **目标不是固定 Partition 数**
- 而是**让每个 Partition 的数据量尽量均衡**
- 主要为了优化最终输出文件
`REBALANCE` **依赖 AQE（Adaptive Query Execution）**。
repartition 是按行均匀分，rebalance是按文件大小均匀分

Examples:
 
df2.hint("broadcast") or broadcast(df2)

.hint("COLEASE", 5)

SQL Hints
```
spark.sql("""
	SELECT /*+ BORADCAST(df2) */
	*
	FROM df1
	JOIN df2
	ON df1.id = df2.id
""")
```

Broadcast Variable
This is a READ_ONLY variable that is broadcasted to all the executor nodes
Shared and immutable data, and serialized once per worker/ Cached on workers for future use.
Lay serialization.
```
my_map = {
	"1": "ediable",
	"2": "dsagsh",
	"3": "jrsestfd"
}
# broadcast to all nodes in the context
map_value = spark.sparkContext.broadcast(my_map)
# get the val
map_value.value
```

Accumulators
This is a WRITE_ONLY variable that is broadcasted to all the executor nodes for aggregation
Executors can add/update the values, Driver will read the value. Mainly used for tracking bad records.
```
badrecord_var = spark.sparkContext.accumulator(0)
spark.sparkContext.longAcculumator("badrecord_var")
```
Better use it in an action, because will guarantee the accuracy

Spark Speculative
spark.speculation = true
If a task runs too slowly, spark will start a new task and if it finished quickly, spark will kill the previous one. Only solve the problem of node error.

Spark Scheduling
Scheduling across Application
	static allocation, request resources required by the application for whole duration. First come , first serve. 
	 dynamic allocation spark.dynamicAllocation.enable = true 
					 spark.dynamicAllocation.executorIdleTimeout = 60s (will release executor when 60s no resources needed)
					 spark.dynamicAllocation.executorIdleTimeout = 1s (will request executor when 1s no resources)
					 
		will release when not need. And will apply once needed
	 
Scheduling within an application
By default, jobs run on after another, we could run them in parallel.

```
def do_job(f1, f2):
	df1 = spark.read.json(f1)
	df2 = spark.read.json(f2)
	outputs.append(df1.join(df2, "id", inner).count())

jobs = []

for i in range(0,2):
	thread = thread.Thread(target=do_job, args = (file1, file2))
	jobs.append(thread)

for(j in jobs):
	j.start()
```

1 by default, job get resource FIFO
2 set spark.scheduler.mode = FAIR, use round robin to give resource, all jobs get roughly equal resource.

Garbage Collection
A JVW process that reclaim the memory by removing unused objects
For temporary objects for shuffling, caching, serialization
When running GC, it stops the application, (Stop the World). No computation takes place during the step.

Pauses in GC
1 minor GC(Young Gen, Few milliseconds)
2 major GC(Old Gen, few seconds)

Reduce GC
1 Use Kryo Serilization, this is much more fast compact than Java serialization
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerlizer")

2 Avoid caching
3 Do not create too many partitions, control partition size
4 Tune JVM memory according to the needs
5 Avoid shuffling
6 Tune GC Algorithm
```
--conf "spark.executor.extraJavaOptions=-XX:+UseG1GC"
```

Executor roles in GC

Small Executor
GC frequency will be high, but pause will be lower

Big Executor
GC frequency will be less, but pause will be high

Bucketing

按照 JOIN KEY 的 hash，把数据提前分成固定数量的桶，这样 JOIN 时可以让相同 key 的数据直接去对应的桶里匹配，减少 shuffle。

Number of buckets = Total Size of data/128MB
choose column with high-cardinality for bucketing, Bucketing can reduce or eliminate shuffle for GROUP BY/JOIN when the query's grouping/join key matches the bucketed column and Spark can exploit the existing distribution.

```
df.write.format("parquet")\
		.mode("append")\
		.bucketBy(3,"product_id")\
		.option("path", "")\
		.saveAsTable("table_name")
use ZORDER and cluster now

```
---------------------------------------------------------------
Bucketing = “提前把相同 JOIN/GROUP BY key 的数据分到固定桶里” → 减少 shuffle 

Z-ORDER = “让经常一起被查询的数据物理上更接近” → 提高 data skipping 

Liquid Clustering = “更现代、更灵活的数据布局管理” → 替代很多手动 bucketing/Z-ORDER 场景

Structured Streaming
Most narrow transformations are stateless, do not need the previous stream.
checkpoint--3 folders(
sources(track data process status), 
offsets(keep track of source), 
commits(which batch had successfully written))

Output modes
append(default) only for stateless transformation
```
spark.conf.set(spark.sql.streaming.schemaInference = True)
df = spark.readStream.format("json")\
		.option("inferShcema", True)\
		.load("../.././.")

df.writeStream.format("delta")\
		.outputMode("append")\
		.trigger(processingTime = "5 second")\
		.option("checkpointLocation", "/.../.../.../")\
		.option("path", "/../../../")
		.start()
		
```
Only output the rows which will not change. Only output when watermark is over.

Complete mode
Return all the rows every time we trigger.(including previous value)
Not supported for no streaming on streaming DataFrames/Datasets

Update mode
It will only output those rows which are updated

Trigger
1 Default, it will automatically trigger next when previous ends.
2 ProcessingTime, we define the interval spark need to trigger the job.
3 Once, Tigger and stop when finish. Load file in one micro batch.
```
.trigger(once = true)
```
4 Available, Tigger and stop when finish. Load file in multiple micro batches.
```
.trigger(available = true)
```
5 Continuous, continuously run the spark stream, instead of micro batch(not support for delta tables)
```
.trigger(continuous = "1 seeconds") 
# 1s for update the checkpoint
```
Archive Source
```
df = spark.readStream().format("json")\
		.......
		.option("cleanSource", "archive")\
		.option("sourceArchiveDir", "../../../")
```

ForEachBatch
with ForEachBatch using upsert
```
df = spark.readStream.table("")

def my_functoin(df, batch_id):

df.writeStream.foreachBatch(my_function)\
		.outputMode("complete")\
		.option("path", "../../../")\
		.start()
```

1 Merge/ Upsert stream can not merge without foreach batch
2 write multiple sinks

Window operations in stream
aggregation on the window as well as other columns

Tumbling window
It is a non overlapping fixed size window
trigger point to summarize.

Sliding window
It is an overlapping but fixed size window

Session window 
Non fixed window interval by default is 5 mins
window start point will reset once receive data, once the data is received in the window, the window will be extended

```
df = spark.readStream.table("")\
		.groupBy("color", window('event_time', '10 minutes')).agg(count('color')).alias('count')\
		.withWatermark("event_time", "10 minutes")
		
window(col("event"), winidowDuration, sildeDuration, offset)
# offset default is 0

# Sliding window
		.withWatermark("event_time", "10 minutes", "5 minutes")

# Session window
		df = spark.readStream.table("")\
		.groupBy("color", session_window('event_time', '5 minutes')).agg(count('color')).alias('count')\
```

Late arriving, watermark, avoid so many windows in memory, MAX(col("event_time")- watermark) = current _time

Spark Connect, introduce decoupled client server architecture
code -> logic plan -> grpc -> spark
databricks connect, IDE to run code in databricks cluster