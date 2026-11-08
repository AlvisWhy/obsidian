

这份笔记建议按照 Spark 的执行链路来学习，而不是按照 API 名称零散记忆：

1. Spark 架构与资源管理：Driver、Executor、Cluster Manager、Deployment Mode
    
2. Spark 编程模型：RDD、DataFrame、SparkSession、Lazy Evaluation
    
3. Spark SQL 执行机制：Logical Plan、Catalyst、Physical Plan、Job、Stage、Task
    
4. Partition 与 Shuffle：Repartition、Coalesce、AQE、数据倾斜
    
5. Join 优化：Broadcast Hash Join、Sort Merge Join、Shuffle Hash Join
    
6. 内存管理与资源调优：JVM Heap、Memory Overhead、Spill、OOM、GC
    
7. 数据缓存与文件布局：Cache、Persist、Partition Pruning、DPP、Bucketing
    
8. Spark 运行优化：Broadcast Variables、Accumulators、Speculative Execution、Scheduling
    
9. Structured Streaming：Checkpoint、Output Mode、Trigger、Watermark、Window、foreachBatch
    
10. Spark Connect 与 Databricks Connect
    

下面会把你记录的内容保留下来，同时区分三类信息：

- 正确的核心概念：保留并梳理。
    
- 需要修正的地方：明确指出原笔记中容易导致面试答错的内容。
    
- 需要补充的知识：补足实际工程和面试中经常考察的部分。
    

# Module 1. Spark Architecture：Spark 集群架构

## 1.1 核心组件

Spark 的架构可以分成两层：

- Cluster 层：负责管理集群机器和资源。
    
- Application 层：负责执行一个 Spark Application 内的任务。
    

![Spark Architecture Explained: Understanding the Difference Between Driver, Executors, and Cluster Manager](https://images.openai.com/static-rsc-4/vMqyes0VVIFXY2DJSWKoCtBuTSwzEq_o5-0LgHG4GGh9tfBIQgbnSg5fpXtY_P3yC_R__PyQt3YFyP7k48wPC0Br1SDrwNc2ob6t3_rTlRQjlc7Pb8QUk5mW1FnxCW0XY9IexayNm7hR7iMnio6TYtunCuoLuVlh5ZX6nQPx7K0?purpose=inline)

[deeplearningnerds.com](https://www.deeplearningnerds.com/spark-architecture-explained-understanding-the-difference-between-driver-executors-and-cluster-manager/)

|Component|所属层级|主要职责|
|---|---|---|
|Cluster Manager|Cluster|管理集群资源，分配 CPU、Memory|
|Worker Node|Cluster|提供计算资源，运行 Executor|
|Driver|Application|运行主程序、生成执行计划、调度任务|
|Executor|Application|在 Worker Node 上执行 Task、缓存数据|

最重要的区别：

|对比|Master / Worker|Driver / Executor|
|---|---|---|
|属于谁|集群|Spark Application|
|表示什么|集群节点或节点角色|应用进程|
|数量|一个集群有多个 Worker|一个 Application 通常有一个 Driver，多个 Executor|
|生命周期|通常跨多个 Application|随 Application 创建和结束|

注意：一个 Worker Node 可以同时运行多个 Application 的 Executor。

## 1.2 Spark Application 的启动流程

你原来的流程需要修正：Cluster Manager 并不总是直接创建 Driver，也不一定存在 Application Master。 具体取决于部署模式和 Cluster Manager 类型。

以 YARN Cluster Mode 为例：

1. 用户向 YARN ResourceManager 提交 Spark Application。
    
2. YARN 分配 Container，启动 ApplicationMaster。
    
3. ApplicationMaster 在该 Container 中启动 Driver。
    
4. Driver 向 Cluster Manager 请求 Executor 所需的资源。
    
5. Cluster Manager 在 Worker Node 上分配 Container，启动 Executor。
    
6. Driver 与 Executor 建立通信。
    
7. Driver 将 Task 分配给 Executor。
    
8. Executor 执行 Task，并向 Driver 汇报执行状态和结果。
    

ApplicationMaster 不是 Driver 的通用同义词。

- YARN Cluster Mode：Driver 运行在 ApplicationMaster 所在的 Container 中。
    
- YARN Client Mode：ApplicationMaster 负责申请资源，Driver 在提交应用的客户端进程中运行。
    
- Kubernetes：Driver 和 Executor 通常分别运行在 Pod 中，不使用 YARN ApplicationMaster 这一机制。
    

## 1.3 Driver 与 Executor 的职责

Driver

- 运行 Spark Application 的主程序。
    
- 创建 SparkSession / SparkContext。
    
- 将用户代码转换成执行计划。
    
- 将 Job 拆分成 Stage 和 Task。
    
- 向 Cluster Manager 申请 Executor。
    
- 调度 Task 并追踪执行状态。
    
- 收集 Action 返回的结果。
    

Executor

- 执行 Driver 分配的 Task。
    
- 处理数据 Partition。
    
- 执行 Shuffle Read / Shuffle Write。
    
- 缓存或持久化数据。
    
- 将 Task 执行状态、指标和结果返回 Driver。
    

一个重要的面试表达：

> A Spark application typically has one driver process and multiple executor processes. The driver coordinates the application, while executors perform distributed computation on partitions of data.

## 1.4 一个 Core 与 Partition 的关系

你的笔记：

> 1 core works on one partition at a time.

更准确地说：

- 一个 Task 通常处理一个 Partition。
    
- 一个 Executor Core 同一时刻通常执行一个 Task。
    
- 因此，一个 Executor 有 4 个可用 Cores 时，通常可以并行执行 4 个 Task。
    
- 如果有 100 个 Partition，至少需要多个 Task 批次才能完成。
    

例如：

|配置|数量|
|---|---|
|Executor|5|
|每个 Executor Cores|4|
|总可用 Cores|20|
|Partition|100|
|每批并行 Task|20|
|理论 Task 批次|5|

这是假设没有资源竞争、Task 失败或其他调度因素的简化模型。

## 1.5 PySpark 与 JVM 的关系

Spark 的核心执行引擎是 JVM 实现的。

Scala Spark：

```
Scala Application
       |
       v
    Driver (JVM)
       |
       v
    Executors (JVM)
```

PySpark：

```
Python Driver Process
       |
       | Py4J
       v
JVM Driver Process
       |
       v
JVM Executors
       |
       +---- Python Workers (when needed)
```

- PySpark Driver 中的 Python 代码通过 Py4J 与 JVM Driver 通信。
    
- Spark SQL / DataFrame 的主要执行工作仍由 JVM Spark Engine 完成。
    
- 当使用 Python UDF、`mapInPandas`、`applyInPandas` 等 Python 计算时，Executor 侧可能启动 Python Worker。
    
- 仅使用 Spark SQL 内置函数时，通常不需要为每个 Executor 启动 Python Worker 来处理每条记录。
    

面试重点：

为什么优先使用 Spark 内置函数，而不是 Python UDF？

1. 内置函数由 Spark SQL 引擎执行。
    
2. 可以参与 Catalyst 优化。
    
3. 可以利用 JVM 内部执行机制。
    
4. 避免大量 JVM 与 Python 之间的数据序列化和传输。
    
5. Python UDF 通常会增加执行开销。
    

## 1.6 面试问题

Q1. What is the difference between a Driver and an Executor?

回答重点：

- Driver coordinates the Spark application.
    
- Executor performs distributed tasks.
    
- Driver creates execution plans and schedules tasks.
    
- Executors process partitions and can cache data.
    

Q2. What is the difference between a Worker Node and an Executor?

回答重点：

- Worker Node is a machine or cluster node.
    
- Executor is an application-level process.
    
- A Worker Node can host multiple Executors from different applications.
    

Q3. How does a Spark application start in YARN Cluster Mode?

回答重点：

- Submit application to ResourceManager.
    
- YARN launches ApplicationMaster.
    
- Driver starts inside the ApplicationMaster container.
    
- Driver requests Executor containers.
    
- Executors register with Driver and execute tasks.
    

Q4. Why is Python UDF usually slower than Spark built-in functions?

回答重点：

- JVM/Python serialization overhead.
    
- Data transfer between JVM and Python Worker.
    
- Reduced Catalyst optimization opportunities.
    
- Built-in functions execute directly within Spark's execution engine.
    

# Module 2. Spark Programming Model：RDD、DataFrame 与 SparkSession

## 2.1 Spark 的三种主要数据抽象

|类型|特点|适用场景|
|---|---|---|
|RDD|分布式对象集合，没有固定 Schema|底层控制、复杂对象处理|
|DataFrame|带 Schema 的分布式 Row 集合|SQL、ETL、数据分析|
|Dataset|带类型约束的分布式数据集合|Scala / Java 类型安全编程|

### RDD

RDD = Resilient Distributed Dataset。

主要特点：

- Distributed：数据分布在多个节点。
    
- Immutable：RDD 不可变。
    
- Fault-tolerant：通过 Lineage 在丢失 Partition 时重新计算。
    
- Lazy Evaluation：Transformations 不会立即执行。
    
- No fixed schema：RDD 本身不要求结构化 Schema。
    

例如：

```
rdd = spark.sparkContext.parallelize([1, 2, 3, 4])result = rdd.filter(lambda x: x > 2)
```

`filter()` 不会立即执行，直到 Action 发生。

RDD 的局限：

- 没有 DataFrame 那样明确的 Schema。
    
- Catalyst 无法像优化 DataFrame / SQL 一样理解和优化 RDD 内部的数据操作。
    
- 对结构化 ETL 来说，通常不如 DataFrame 高效。
    

### DataFrame

DataFrame 可以理解为：

> A distributed collection of rows organized into named columns with a schema.

例如：

```
df = spark.read.format("csv") \    .option("header", True) \    .option("inferSchema", True) \    .load("/data/orders.csv")
```

DataFrame 的优势：

- Schema 信息明确。
    
- Catalyst 可以优化执行计划。
    
- Tungsten / Whole-Stage Code Generation 等执行优化可以发挥作用。
    
- 支持 SQL 和 DataFrame API。
    

### Dataset

Dataset 是带有静态类型约束的数据抽象，主要用于 Scala / Java。

PySpark 中主要使用 DataFrame，不提供与 Scala Dataset 相同的强类型 Dataset API。

修正原笔记：

> DataFrame is a dataset with datatype of row.

可以理解为 DataFrame 在概念上接近 `Dataset[Row]`，但不应该把 Python DataFrame 和 Scala 的强类型 Dataset 混为一谈。

## 2.2 SparkSession 与 SparkContext

Spark 2.0 以后，SparkSession 成为结构化数据处理的主要入口。

```
from pyspark.sql import SparkSessionspark = SparkSession.builder \    .appName("DataEngineering") \    .getOrCreate()
```

|API|作用|
|---|---|
|SparkContext|Spark Core 的入口，负责底层 RDD 和集群通信|
|SQLContext|旧版 SQL / DataFrame 入口|
|HiveContext|旧版 Hive 集成入口|
|SparkSession|统一入口，支持 DataFrame、SQL、Catalog、Hive 集成等|

SparkSession 可以通过：

```
spark.sparkContext
```

访问 SparkContext。

面试重点：

SparkSession 是高层入口，SparkContext 仍然是底层 Spark Core 的重要对象。Spark 2.0 后不是 SparkContext 被删除，而是多个高层 Context 被 SparkSession 统一。

## 2.3 Transformation 与 Action

### Transformation：Lazy Evaluation

Transformation 生成新的 RDD / DataFrame，不立即执行。

Narrow Transformation

一个子 Partition 通常只依赖一个父 Partition，或者只依赖少量固定父 Partition，不需要全局数据重新分布。

例如：

```
df.select("id", "name")df.filter("age > 18")df.withColumn("total", col("price") * col("quantity"))df.drop("unused_column")
```

这些操作通常可以在当前 Partition 内完成。

Wide Transformation

一个子 Partition 需要从多个父 Partition 获取数据，通常涉及 Shuffle。

例如：

```
df.groupBy("country").count()df1.join(df2, "id")df.distinct()
```

需要注意：

- `join()` 不一定是 Wide Transformation。
    
- Broadcast Join 通常不需要对两侧数据进行 Shuffle。
    
- 某些 Join 可以通过已有的 Partitioning 避免 Shuffle。
    
- 是否发生 Shuffle 应看实际 Physical Plan，而不是只看 API 名称。
    

### Action

Action 会触发计算。

常见操作：

```
df.collect()df.take(10)df.show()df.count()df.write.format("parquet").save("/output")
```

`read()` 通常不是 Action。

它会创建 DataFrame / Logical Plan；真正的数据扫描通常在后续 Action 执行时发生。

`write()` 则会触发实际写入计算。

## 2.4 Spark 的执行层级：Job → Stage → Task

```
Spark Application
       |
       v
      Job
       |
       v
     Stage
       |
       v
     Task
       |
       v
   Partition
```

|概念|含义|
|---|---|
|Application|一次 Spark 应用|
|Job|通常由一个 Action 触发|
|Stage|可以一起执行的一组 Task|
|Task|一个 Partition 上的具体计算任务|

例如：

```
df = spark.read.parquet("/orders")result = df.groupBy("country").count()result.show()
```

大致执行过程：

1. `read.parquet()` 创建读取计划。
    
2. `groupBy()` 创建聚合逻辑计划。
    
3. `show()` 触发 Job。
    
4. Spark 根据 Shuffle Boundary 划分 Stage。
    
5. 每个 Stage 根据 Partition 数创建 Task。
    
6. Executor 执行 Task。
    

### Stage 与 Shuffle

你原来的笔记：

> Driver will break the plan after each wide dependency.

这是一个很好的理解方式。

更准确地说：

- Shuffle 通常形成 Stage Boundary。
    
- 一个 Stage 内可以执行多个 Narrow Transformation。
    
- Stage 之间通过 Shuffle 文件传递中间数据。
    
- 下游 Stage 需要等待上游 Stage 的 Shuffle 输出满足执行条件。
    

例如：

```
df.filter(col("amount") > 100) \  .groupBy("country") \  .sum("amount")
```

可以简化为：

```
Stage 1:
Read -> Filter -> Partial Aggregate -> Shuffle Write

Stage 2:
Shuffle Read -> Final Aggregate
```

注意：不是每个 Wide Transformation 都一定对应完全独立的一个 Stage；实际 Stage 划分还要看物理执行计划和具体依赖关系。

## 2.5 Lazy Evaluation 与优化

```
DataFrame Transformations
          |
          v
     Logical Plan
          |
          v
        Action
          |
          v
   Physical Execution
```

Spark 可以先收集整个 Transformation 链路，再进行优化，而不是每调用一次 API 就立即计算。

这也是 DataFrame / SQL API 相比传统逐行执行代码的重要优势。

## 2.6 面试问题

Q1. What is the difference between RDD, DataFrame and Dataset?

Q2. What is lazy evaluation in Spark? Why is it useful?

Q3. What is the difference between a narrow transformation and a wide transformation?

Q4. What is the difference between a Job, Stage and Task?

Q5. Does every join cause a shuffle?

回答 Q5 时需要强调：不一定。Broadcast Join、已有分区布局等情况可能避免 Shuffle。

# Module 3. Spark SQL Execution：Catalyst Optimizer 与 Physical Plan

这是 Spark 面试中非常重要的一部分。

## 3.1 Spark SQL 的执行流程

你的笔记中的顺序基本正确，但需要区分 Catalyst 的优化过程和 AQE 的运行时优化过程。

![Apache Spark’s Logical and Physical Plans Using Explain() Method](https://images.openai.com/static-rsc-4/2Y0DVqmWK0KLCqmni1qt7rAp8QdpAKgAVMZOM05kmMlC6gNCXrqnJ-qat-O9BZtCderRvHE6o9FoIV4EnnTRCUAZqeH69_bEksF6PnpIsfh6EY5_oq13JiqR2efdjnqlTmaJ4VJsJAdJ1kP696HYCxLQtmFaky2iVkCtjjv1vxo?purpose=inline)

[linkedin.com](https://www.linkedin.com/pulse/apache-sparks-logical-physical-plans-using-explain-method-akruvala)

完整流程：

```
Spark SQL / DataFrame API
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
Physical Plan Candidates
          |
          v
Selected Physical Plan
          |
          v
Code Generation / Execution
          |
          v
Executors
```

### 1. Unresolved Logical Plan

Spark 先将 SQL 或 DataFrame API 转换为逻辑计划。

例如：

```
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id
```

此时 Spark 已经知道用户希望执行：

- Scan
    
- Aggregate
    

但可能还没有确认：

- `orders` 表是否存在。
    
- `customer_id` 是否存在。
    
- `amount` 的数据类型。
    
- `SUM()` 是否支持该数据类型。
    

### 2. Analyzed Logical Plan

Analyzer 负责解析和验证：

- 表和 View 是否存在。
    
- Column 是否存在。
    
- Column 的 Data Type。
    
- Function 是否存在。
    
- Catalog 和 Schema 的引用是否正确。
    

### 3. Optimized Logical Plan

Catalyst 进行逻辑优化，例如：

|Optimization|作用|
|---|---|
|Predicate Pushdown|尽可能提前过滤数据|
|Projection Pruning|只读取需要的列|
|Constant Folding|提前计算常量表达式|
|Constant Propagation|利用已知常量简化表达式|
|Boolean Simplification|简化布尔条件|
|Join Reordering|在条件允许时调整 Join 顺序|
|Limit Pushdown|尽可能提前处理 Limit|

注意：Join Reordering 等优化受统计信息、Join 类型、条件和规则限制，不是所有查询都一定会重新排序。

### 4. Physical Plan

Spark 将逻辑计划转换成可以实际执行的物理算子，例如：

- `FileScan`
    
- `Filter`
    
- `Project`
    
- `HashAggregate`
    
- `Sort`
    
- `Exchange`
    
- `BroadcastHashJoin`
    
- `SortMergeJoin`
    

Physical Plan 会决定实际采用哪种 Join 算法、是否需要 Shuffle、是否需要 Sort 等。

### 5. AQE：运行时优化

AQE（Adaptive Query Execution）在执行过程中利用运行时统计信息调整 Physical Plan。

例如：

- 合并小的 Shuffle Partitions。
    
- 拆分倾斜的 Shuffle Partitions。
    
- 根据实际数据量调整 Join Strategy。
    

因此：

> Catalyst 主要负责计划阶段的优化；AQE 可以根据运行时统计信息进一步调整执行计划。

## 3.2 如何查看执行计划？

```
df.explain()
```

常用方式：

```
df.explain("simple")df.explain("extended")df.explain("formatted")df.explain("cost")
```

其中：

|参数|作用|
|---|---|
|simple|主要显示 Physical Plan|
|extended|显示 Logical Plan 和 Physical Plan 等信息|
|formatted|更易阅读的结构化计划|
|cost|显示统计信息和成本相关信息（如果可用）|

例如：

```
df1.join(df2, "id").explain("formatted")
```

你需要重点识别：

- `Exchange`：通常代表 Shuffle。
    
- `BroadcastExchange`：准备广播数据。
    
- `BroadcastHashJoin`：广播 Hash Join。
    
- `SortMergeJoin`：Sort Merge Join。
    
- `FileScan`：文件读取。
    
- `AdaptiveSparkPlan`：使用 AQE 的执行计划。
    

## 3.3 面试问题

Q1. Explain the Catalyst Optimizer in Spark.

Q2. What is the difference between a Logical Plan and a Physical Plan?

Q3. What is the difference between Catalyst Optimization and AQE?

Q4. How would you investigate a slow Spark query?

建议回答：

1. Inspect the execution plan using `explain("formatted")`.
    
2. Identify expensive operators such as Exchange, Sort and Join.
    
3. Check Spark UI for stage duration and Shuffle metrics.
    
4. Inspect partition sizes and data skew.
    
5. Review input file sizes and data skipping.
    
6. Apply appropriate optimizations and compare the execution metrics.
    

# Module 4. Partition 与 Shuffle：Spark 并行计算的核心

这是 Spark 性能调优中最重要的部分之一。

## 4.1 什么是 Partition？

Partition 是 Spark 分布式计算的基本数据处理单位。

例如，一个 DataFrame 有 100 个 Partition：

```
DataFrame
   |
   +-- Partition 0
   +-- Partition 1
   +-- Partition 2
   ...
   +-- Partition 99
```

通常：

- 一个 Task 处理一个 Partition。
    
- 一个 Stage 中的 Task 数量通常由该 Stage 的 Partition 数量决定。
    
- 多个 Executor 可以并行处理多个 Partition。
    
- Partition 太少会导致资源利用率不足。
    
- Partition 太多会增加 Task 调度、文件管理等开销。
    

注意：Partition 不等于文件。

一个 Partition 可能来自多个输入文件；一个输出 Partition 通常会写出一个或多个文件，具体取决于数据源和写入方式。

## 4.2 Shuffle 是什么？

Shuffle 是不同 Partition 之间重新分配数据的过程。

例如：

```
df.groupBy("country").count()
```

假设原始数据分布在 4 个 Partition：

```
Partition 1: NL, DE, NL
Partition 2: DE, FR, NL
Partition 3: FR, DE, NL
Partition 4: NL, FR, DE
```

为了按照 `country` 聚合，相同 country 的数据需要进入相同的目标 Partition：

```
Shuffle
   |
   +-- NL -> Partition A
   +-- DE -> Partition B
   +-- FR -> Partition C
```

Shuffle 大致包含：

1. Shuffle Write：上游 Task 将数据按照目标 Partition 规则写出。
    
2. Shuffle Read：下游 Task 读取所需的 Shuffle 数据。
    
3. Network Transfer：数据可能需要跨 Executor / Node 传输。
    
4. Sorting / Aggregation：根据算子要求进一步处理数据。
    

Shuffle 成本通常来自：

- 网络传输。
    
- 磁盘读写。
    
- 序列化和反序列化。
    
- 排序。
    
- 内存压力和 Spill。
    

## 4.3 Repartition

`repartition()` 会产生 Shuffle，可以增加或减少 Partition。

```
df = df.repartition(10)
```

也可以指定列：

```
df = df.repartition(10, "customer_id")
```

此时 Spark 通常使用 Hash Partitioning：

\[ partitionId = hash(customerId) \bmod 10 \]

相同 `customer_id` 的记录会进入相同的目标 Partition。

适用场景：

- Partition 数量太少，需要增加并行度。
    
- 数据分布不均，需要重新分配。
    
- 后续要按照某个 Join Key 或 Grouping Key 处理数据。
    
- 希望重新分布数据。
    

### repartitionByRange

```
df = df.repartitionByRange(10, "event_time")
```

按照列的范围分配 Partition。

Spark 会通过采样等机制估算范围边界，因此不代表每个 Partition 的数据量一定相同。

适用场景：

- 时间范围查询。
    
- Range-based 数据处理。
    
- 希望数据按照某个字段的值域组织。
    

## 4.4 Coalesce

```
df = df.coalesce(5)
```

`coalesce()` 通常通过合并已有 Partition 来减少 Partition 数量，不进行完整 Shuffle。

|对比|repartition|coalesce|
|---|---|---|
|是否 Shuffle|通常是|默认不是|
|增加 Partition|可以|不可以|
|减少 Partition|可以|可以|
|数据重新分布|可以|通常不充分|
|可能产生数据倾斜|是|是，尤其是减少过多时|

注意：

- `coalesce()` 默认适合减少 Partition。
    
- 它不保证每个 Partition 大小均匀。
    
- 如果原始数据已经严重倾斜，直接 `coalesce()` 可能让倾斜更严重。
    
- `coalesce()` 也有 `shuffle=True` 的选项，但这时就不再是简单的无 Shuffle 合并。
    

## 4.5 如何选择 Partition 数量？

你原来的笔记：

> ideal size for each partition is 128–256 MB.

这是一个常见的调优起点，不是 Spark 的强制标准。

需要考虑：

- 输入数据大小。
    
- Shuffle 数据大小。
    
- 每个 Task 的处理时间。
    
- Executor 的 CPU 和 Memory。
    
- 数据倾斜。
    
- 文件格式和压缩率。
    
- 查询算子的内存需求。
    

例如，假设 Shuffle 数据量为 200 GB，希望每个 Partition 大约 128 MB：

\[ N = \frac{200 \times 1024}{128} = 1600 \]

可以设置：

```
spark.conf.set("spark.sql.shuffle.partitions", 1600)
```

但要注意：

- `spark.sql.shuffle.partitions` 主要影响 SQL/DataFrame Shuffle。
    
- RDD 的默认并行度通常涉及 `spark.default.parallelism`。
    
- AQE 开启后，Spark 可以在运行时合并 Shuffle Partition。
    
- 不能只根据数据总量决定 Partition 数量，还要检查实际 Task 时间和数据分布。
    

## 4.6 AQE：Adaptive Query Execution

AQE 从 Spark 3.0 开始提供，是运行时执行计划优化机制。

```
spark.conf.set("spark.sql.adaptive.enabled", "true")
```

主要有三个优化方向。

### 1. Coalesce Shuffle Partitions

当 Shuffle 后产生大量很小的 Partition，AQE 可以将它们合并。

相关配置：

```
spark.sql.adaptive.coalescePartitions.enabledspark.sql.adaptive.advisoryPartitionSizeInBytesspark.sql.adaptive.coalescePartitions.initialPartitionNum
```

例如：

```
Before AQE:
1000 partitions
每个只有 1 MB

After AQE:
合并成更少的 Partition
每个接近目标大小
```

修正： `spark.sql.adaptive.coalescePartitions.minPartitionNum` 在不同 Spark 版本中的使用情况可能有变化；实际调优优先检查当前版本配置文档。不要把它当作必须配置的参数。

### 2. Optimize Skew Join

当某些 Partition 特别大，造成少数 Task 运行很久，AQE 可以识别并拆分倾斜 Partition。

```
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

常见默认判断配置：

```
spark.sql.adaptive.skewJoin.skewedPartitionFactorspark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes
```

你记录的：

- 5 倍 median。
    
- 256 MB。
    

可以作为理解默认阈值的例子，但实际默认值和判断逻辑受 Spark 版本及配置影响。

AQE 的倾斜 Join 优化通常需要同时考虑：

- Partition 大小超过绝对阈值。
    
- Partition 大小相对于中位数足够大。
    

### 3. Dynamically Switch Join Strategy

AQE 可以根据 Shuffle 后实际统计信息改变 Join Strategy。

例如：

```
Initial Physical Plan:
SortMergeJoin

Runtime Statistics:
One side is small enough

Final Physical Plan:
BroadcastHashJoin
```

### 4. Local Shuffle Reader

Local Shuffle Reader 是 AQE 的一种优化。

当 Spark 已经不需要原来的 Shuffle Partitioning 要求时，如果数据可以从本地 Executor 读取，就尽可能避免跨网络读取。

它的目标是减少不必要的网络传输，而不是简单地“删除未使用的 Partition”。

## 4.7 数据倾斜（Data Skew）

数据倾斜是指某些 Partition 的数据量明显大于其他 Partition。

例如：

|Partition|数据量|
|---|---|
|P1|100 MB|
|P2|120 MB|
|P3|110 MB|
|P4|5 GB|

结果：

- 大部分 Task 很快完成。
    
- P4 运行时间很长。
    
- Stage 必须等待慢 Task。
    
- Executor 可能出现 OOM 或 Spill。
    

### 解决方法

方法一：AQE

适用于 Spark 能识别并优化的 Shuffle Join 倾斜场景。

方法二：Salting

给倾斜 Key 增加随机或确定性的 Salt 值，将一个大 Key 分散到多个 Partition。

假设 `customer_id = 100` 是倾斜 Key：

```
100 -> 100_0
100 -> 100_1
100 -> 100_2
100 -> 100_3
```

PySpark 示例：

```
from pyspark.sql.functions import (    col, when, floor, rand, lit)df_salted = df.withColumn(    "salt",    when(col("customer_id") == 100,         floor(rand() * 4))    .otherwise(lit(0)))
```

Join 时需要让另一侧小表中对应的 Key 复制到所有 Salt 值：

```
from pyspark.sql.functions import explode, arraysalts = array(    lit(0), lit(1), lit(2), lit(3))small_salted = small_df.withColumn(    "salt",    explode(salts))result = df_salted.join(    small_salted,    ["customer_id", "salt"],    "inner")
```

注意：

- 只应该对需要处理的倾斜 Key 进行扩展。
    
- 小表复制会增加数据量。
    
- 如果对所有 Key 都复制，可能导致不必要的开销。
    
- 对非 Join 聚合，也可以采用局部聚合、拆分热点 Key、二次聚合等策略。
    

方法三：Broadcast Join

如果 Join 的一侧足够小，可以考虑广播小表，避免对两侧进行 Shuffle。

方法四：预聚合

在 Shuffle 前先做 Partial Aggregation，减少需要传输的数据量。

## 4.8 REBALANCE Hint

```
df.hint("REBALANCE")
```

REBALANCE 主要用于改善输出 Partition 的数据分布，特别是写文件前减少过大或过小的输出 Partition。

它通常依赖 AQE。

修正：

- `repartition`：按指定 Partition 数量或列重新分配数据。
    
- `REBALANCE`：目标是改善输出 Partition 的平衡性，不是简单地“按照文件大小均匀分”。
    
- REBALANCE 不保证每个输出文件大小完全相等。
    

## 4.9 面试问题

Q1. What is a shuffle in Spark, and why is it expensive?

Q2. What is the difference between repartition and coalesce?

Q3. How do you determine the optimal number of partitions?

Q4. What is data skew, and how would you solve it?

Q5. What are the three main optimizations provided by AQE?

Q6. What is the difference between repartition and rebalance?

# Module 5. Join Strategies：Spark Join 优化

## 5.1 Broadcast Hash Join

当一侧数据集足够小时，可以将小表广播到所有 Executor。

```
from pyspark.sql.functions import broadcastresult = df_large.join(    broadcast(df_small),    "id",    "inner")
```

执行逻辑：

1. Driver / Spark SQL 生成 Broadcast Exchange。
    
2. 小表被序列化并广播。
    
3. 各 Executor 获取广播数据。
    
4. 每个 Executor 使用本地小表 Hash Structure。
    
5. 大表 Partition 可以直接在本地与小表进行 Join。
    

![[Spark] BroadCast Hash Join(BHJ) / Shuffle Sort Merge Join(SMJ)](https://images.openai.com/static-rsc-4/Z4RR3rjAa0WLWk5LGtS0E2-6pO-ggB-7Oz1jpTXY390Stbn7wLnvKm7cKFfFtUpcNk3pCFbdZe03ARbktb5zzX8HPd10McYEQUnwvWDZlAwo8FYSumUcmCkVsFhg4dNj_nkNgM39Ww_EVrwc9PXMyec84W7dpNRPgbDL-QS4ykY?purpose=inline)

[yeo0.tistory.com](https://yeo0.tistory.com/entry/Spark-BroadCast-Hash-JoinBHJ-Shuffle-Sort-Merge-JoinSMJ)

优点：

- 避免大表和小表之间的 Shuffle。
    
- 通常速度快。
    
- 适合 Fact Table + Small Dimension Table。
    

风险：

- 小表过大，广播会增加网络和内存压力。
    
- 每个 Executor 都需要持有广播数据。
    
- 过大的 Broadcast 可能导致 Driver 或 Executor 内存问题。
    

## 5.2 Sort Merge Join

这是大表 Join 中很常见的策略。

执行逻辑：

1. 两侧数据按照 Join Key Shuffle。
    
2. 相同 Key 的记录进入对应的 Partition。
    
3. 对两侧数据排序。
    
4. 通过排序后的数据进行 Merge Join。
    

![Spark Join Strategies Explained: Sort Merge Join](https://images.openai.com/static-rsc-4/5cp98qv_9lRiUU1jS0Cl7LZM4gMmmau20kj8Heat3OqIndiGiOHiEvVnmlPTPWAaV8D2Ldl6lkR0JJQfwHr8e6M8hKZpni4_xmHPPiEFIr-p2-_jnXWPuTSANifIKwani7Olx5LKLyroGhvAxTBS5Ew30vqMfsFDujvpNrMziLY?purpose=inline)

[canadiandataguy.com](https://www.canadiandataguy.com/p/spark-join-strategies-explained-sort)

例如：

```
df1.join(df2, "customer_id")
```

适用：

- 两侧数据较大。
    
- Join Key 可排序。
    
- 不适合直接广播其中一侧。
    

缺点：

- Shuffle 成本高。
    
- Sort 也有 CPU 和内存开销。
    

## 5.3 Shuffle Hash Join

执行逻辑：

1. 两侧数据根据 Join Key Shuffle。
    
2. 在每个 Partition 内构建一侧数据的 Hash Table。
    
3. 使用另一侧数据进行匹配。
    

```
Shuffle
  |
  v
Partition A
  |
  +-- Build Hash Table
  |
  +-- Probe with Other Side
```

它通常适用于每个 Shuffle Partition 内 Build Side 足够小、能放入内存的情况。

修正： 不应该只因为一侧 DataFrame 总体较小就认为 Shuffle Hash Join 一定合适；需要看 Shuffle 后每个 Partition 的数据量和内存情况。

## 5.4 三种 Join 的对比

|Join|Shuffle|Sort|Hash Table|适用场景|
|---|---|---|---|---|
|Broadcast Hash Join|通常避免 Join Shuffle|通常不需要|是|一侧很小|
|Sort Merge Join|通常需要|是|不需要传统 Build Hash Table|两侧较大|
|Shuffle Hash Join|通常需要|不需要|是|每个 Partition 的 Build Side 较小|

## 5.5 AQE 如何优化 Join？

例如：

```
初始统计信息：
Table A = 10 GB
Table B = 2 GB

初始计划：
SortMergeJoin

实际运行时：
过滤后 Table B 只有 5 MB

AQE：
可能转换成 Broadcast Hash Join
```

这就是为什么 AQE 可以改善基于静态统计信息不准确的 Join Strategy。

## 5.6 Join Hints

```
df1.join(df2.hint("broadcast"), "id")
```

SQL：

```
SELECT /*+ BROADCAST(b) */
    *
FROM large_table a
JOIN small_table b
ON a.id = b.id
```

常见 Hints：

|Hint|含义|
|---|---|
|`BROADCAST`|优先 Broadcast Join|
|`MERGE`|优先 Sort Merge Join|
|`SHUFFLE_HASH`|优先 Shuffle Hash Join|
|`SHUFFLE_REPLICATE_NL`|Shuffle Replicate Nested Loop Join|

注意：Hint 是优化建议，不代表在所有情况下都能强制 Spark 使用指定策略；还要受 Join 类型、数据类型和物理执行条件限制。

## 5.7 面试问题

Q1. Explain Broadcast Hash Join and Sort Merge Join.

Q2. When would you use Broadcast Join?

Q3. Why can Broadcast Join cause OOM?

Q4. How does AQE change Join Strategy at runtime?

Q5. How would you optimize a Join between a 500 GB fact table and a 10 MB dimension table?

# Module 6. Memory Management：Spark 内存管理

这是原笔记中需要修正最多的模块之一。

## 6.1 Driver Memory

Driver 负责：

- 主程序执行。
    
- SparkSession / SparkContext。
    
- Logical Plan 和 Physical Plan。
    
- Job / Stage / Task 调度。
    
- 维护执行状态。
    
- 收集 Action 的返回结果。
    

常见配置：

```
--driver-memory 4G
```

Driver 可能因为以下原因 OOM：

```
df.collect()
```

`collect()` 会将所有结果收集到 Driver。

如果 DataFrame 有数百 GB，就可能超过 Driver 内存。

更安全的做法：

```
df.show(20)df.take(100)df.limit(1000).toPandas()
```

但 `toPandas()` 仍然会把数据收集到 Driver，因此只适合足够小的数据集。

## 6.2 Executor Memory

Executor 是真正执行 Task 的地方。

例如：

```
--executor-memory 10G
--executor-cores 4
--num-executors 10
```

Executor Container 的内存预算不等于 JVM Heap。

可以理解为：

```
Executor Container Memory
   |
   +-- JVM Heap
   |
   +-- Memory Overhead
   |
   +-- Off-Heap Memory (if configured)
   |
   +-- PySpark Memory (if separately configured)
```

### 1. JVM Heap

```
spark.executor.memory
```

主要用于：

- Spark Execution Memory。
    
- Spark Storage Memory。
    
- 用户创建的 JVM 对象。
    
- Spark 内部对象和元数据。
    

### 2. Memory Overhead

```
spark.executor.memoryOverhead
```

用于 JVM Heap 之外的额外内存开销，例如：

- JVM Native Memory。
    
- Netty / Network Buffer。
    
- Python Worker 进程内存（默认情况下通常由 Overhead 承担）。
    
- Native Libraries。
    
- 其他非 Heap 使用。
    

修正：

Memory Overhead 不是专门的 Python Memory，也不是所有 Python 数据都会自动单独放在一个独立的 PySpark Memory Pool 中。

### 3. Off-Heap Memory

```
spark.memory.offHeap.enabledspark.memory.offHeap.size
```

Off-Heap 是 JVM Heap 之外的内存使用机制。

开启后，可以让 Spark 使用配置的 Off-Heap Memory。

它并不意味着所有 Spark 缓存都会自动转移到 Off-Heap。

### 4. PySpark Memory

```
spark.executor.pyspark.memory
```

可以为 Python Worker 设置内存限制 / 预算。

如果没有单独配置，Python Worker 的内存通常需要纳入 Container 的 Memory Overhead 预算。

## 6.3 Spark Unified Memory Management

Spark 的 Execution Memory 和 Storage Memory 采用 Unified Memory Management。

主要参数：

```
spark.memory.fractionspark.memory.storageFraction
```

默认理解：

- `spark.memory.fraction` 通常为 0.6。
    
- `spark.memory.storageFraction` 通常为 0.5。
    

在 JVM Heap 中，Spark 会保留一部分 Reserved Memory，其余空间中有一部分属于 Spark Memory，另一部分属于 User Memory。

简化理解：

```
JVM Heap
   |
   +-- Reserved Memory
   |
   +-- Unified Spark Memory
   |      |
   |      +-- Execution Memory
   |      |
   |      +-- Storage Memory
   |
   +-- User Memory
```

Execution Memory

用于：

- Shuffle。
    
- Sort。
    
- Join。
    
- Aggregation。
    
- 其他执行期间的内存需求。
    

Storage Memory

用于：

- Cache。
    
- Persist。
    
- 持久化 RDD / DataFrame 数据。
    

### Execution 与 Storage 的关系

你的笔记中：

> storage memory can not evict the data from execution memory; execution memory can evict data from storage memory.

这是理解 Unified Memory 的重要规则：

- Execution Memory 可以借用 Storage Memory 空间。
    
- 当 Execution 需要更多内存时，可以驱逐部分可驱逐的 Storage 数据。
    
- Storage Memory 不能驱逐正在被 Execution 使用的内存。
    
- 已经被使用中的缓存 Block 不一定能立即驱逐。
    

## 6.4 Spill to Disk

当执行算子需要的内存不足时，Spark 可以将部分中间数据 Spill 到磁盘。

例如：

```
Sort / Aggregation
       |
       v
Memory not enough
       |
       v
Spill intermediate data to disk
       |
       v
Continue processing
```

Spill 可以避免部分内存不足的问题，但会显著增加磁盘 I/O。

为什么有 Spill 仍然会 OOM？

因为并不是所有内存使用都能通过 Spill 解决。

常见原因：

1. 单个 Task 处理的数据量过大。
    
2. 数据倾斜造成某个 Partition 特别大。
    
3. Python Worker 占用过多内存。
    
4. Driver 收集过多数据。
    
5. Broadcast Table 太大。
    
6. JVM / Native Memory 超过 Container 限制。
    
7. 用户代码创建了大量无法及时释放的对象。
    

## 6.5 Garbage Collection（GC）

GC 是 JVM 自动回收不再使用对象的机制。

常见区域：

- Young Generation：新创建对象。
    
- Old Generation：生命周期较长的对象。
    

常见 GC：

- Minor GC：主要清理 Young Generation。
    
- Major / Full GC：涉及 Old Generation 或整个 Heap 的回收。
    

GC 可能导致 Stop-the-World Pause，即部分应用线程暂停。

### 如何减少 GC 压力？

1. 使用合适的 Serialization。
    
2. 避免不必要的 Cache / Persist。
    
3. 避免在 Python / JVM 中创建大量临时对象。
    
4. 避免超大 Partition。
    
5. 减少不必要的 Shuffle。
    
6. 选择合理的 Executor Memory 和 Executor Cores。
    
7. 监控 GC Time 和 Executor 指标。
    

### Kryo Serialization

```
spark.conf.set(    "spark.serializer",    "org.apache.spark.serializer.KryoSerializer")
```

修正：

- Kryo 对 RDD 等对象序列化场景很有帮助。
    
- Spark SQL 内部的 DataFrame / Tungsten 执行不应简单理解为全部使用 Java Serialization。
    
- 不能保证开启 Kryo 就一定改善 DataFrame 性能。
    

## 6.6 Executor 大小如何选择？

你原来的结论：

> Small Executor: GC frequency high, but pause low. Big Executor: GC frequency low, but pause high.

可以作为一种常见趋势，但不是必然规律。

|小 Executor|大 Executor|
|---|---|
|单个 Executor 资源较少|单个 Executor 资源较多|
|故障影响范围可能较小|单个进程承载更多任务|
|GC 压力可能较低|Heap 较大时可能出现较长 GC Pause|
|Executor 数量通常更多|Executor 数量通常更少|

实际选择要结合：

- CPU / Memory 比例。
    
- Task 数量。
    
- Shuffle 数据规模。
    
- Python Worker 数量。
    
- GC Time。
    
- 节点网络和磁盘资源。
    

## 6.7 面试问题

Q1. What is the difference between JVM Heap and Memory Overhead?

Q2. What is the difference between Execution Memory and Storage Memory?

Q3. What is spilling in Spark? Does spilling always prevent OOM?

Q4. Why can a Spark Executor run out of memory even when the JVM Heap is not full?

Q5. What causes Driver OOM and how would you prevent it?

Q6. How would you diagnose high GC time in Spark?

# Module 7. Cache、Persist 与数据存储

## 7.1 Cache 与 Persist

Cache 和 Persist 都用于保存中间计算结果，避免后续重复计算。

```
df.cache()
```

等价于使用默认 Storage Level：

```
df.persist()
```

在 PySpark DataFrame API 中，默认 Storage Level 通常为 `MEMORY_AND_DISK_DESER`（具体以版本和 API 为准）。

修正原笔记：

- 不要认为 DataFrame 的默认 Cache 一定是 `MEMORY_AND_DISK`。
    
- RDD 和 DataFrame 的默认 Storage Level 不完全相同。
    
- Storage Level 可能随 Spark 版本改变。
    

## 7.2 Storage Levels

|Storage Level|含义|
|---|---|
|MEMORY_ONLY|只存内存，放不下的 Block 后续需要重新计算|
|MEMORY_AND_DISK|内存不足时将部分 Block 放磁盘|
|DISK_ONLY|只存磁盘|
|MEMORY_ONLY_2|内存保存两个副本|
|MEMORY_AND_DISK_2|内存和磁盘保存两个副本|
|OFF_HEAP|使用配置的 Off-Heap 存储|

注意：

- `_SER` 等旧式 Storage Level 的可用性和推荐程度取决于 Spark 版本。
    
- PySpark 的 StorageLevel API 与 Scala API 不完全相同。
    
- 不要把 `MEMORY_ONLY_2` 理解为“自动创建容灾备份的通用机制”，它是数据 Block 的副本存储策略。
    

## 7.3 Cache 的实际执行

```
df = spark.read.parquet("/orders")df_cached = df.cache()df_cached.count()
```

`cache()` 本身是 Lazy 的。

只有触发 Action 后，数据才会实际计算并缓存。

如果缓存空间不足：

- MEMORY_ONLY：无法缓存的 Block 可能在后续需要时重新计算。
    
- MEMORY_AND_DISK：无法放入内存的 Block 可以写入磁盘。
    

取消缓存：

```
df_cached.unpersist()
```

## 7.4 什么时候使用 Cache？

适用场景：

- 一个昂贵的 DataFrame 被多个下游任务重复使用。
    
- 多次执行不同聚合。
    
- 迭代式算法。
    
- 需要避免重复读取和计算。
    

不适用场景：

- DataFrame 只使用一次。
    
- 数据非常大，缓存成本高。
    
- 上游计算非常简单。
    
- Cache 会挤占 Execution Memory，导致其他任务变慢。
    

# Module 8. Partition Pruning、Predicate Pushdown 与 DPP

这三个概念非常容易在面试中混淆。

## 8.1 Partition Pruning

如果数据按照某些列进行物理分区，查询时 Spark 可以跳过不需要的目录 / 分区。

例如数据按照：

```
year=2025/month=01/
year=2025/month=02/
year=2026/month=01/
year=2026/month=02/
```

查询：

```
SELECT *
FROM orders
WHERE year = 2026
  AND month = 1
```

Spark 可以只读取：

```
year=2026/month=01/
```

这叫 Partition Pruning。

## 8.2 Predicate Pushdown

Predicate Pushdown 是尽可能将过滤条件下推到数据源或文件扫描层。

例如：

```
SELECT *
FROM orders
WHERE amount > 100
```

如果 Parquet 文件包含相关统计信息，Spark 可以利用文件级统计信息跳过不可能满足条件的数据文件或 Row Groups。

常见优化：

- Partition Pruning。
    
- Parquet / ORC Filter Pushdown。
    
- Data Skipping。
    

修正： 不能简单地把“某些表达式不能 Pushdown”理解为绝对规则。是否能下推，取决于数据源、表达式、函数支持和 Catalyst 优化结果。

例如：

```
WHERE UPPER(name) = 'ALICE'
```

可能无法被底层数据源直接执行为普通的列过滤，但 Spark 仍可能在上层执行 Filter。即使无法下推，也不代表整个查询无法优化。

## 8.3 Dynamic Partition Pruning（DPP）

DPP 主要用于 Join 场景。

例如：

- Fact Table 按 `customer_id` 分区。
    
- Dimension Table 过滤出少数 `customer_id`。
    
- 两张表按照 `customer_id` Join。
    

```
SELECT *
FROM fact_sales f
JOIN dim_customer d
  ON f.customer_id = d.customer_id
WHERE d.country = 'NL'
```

执行逻辑：

1. 先过滤 Dimension Table。
    
2. 得到符合条件的 Join Key。
    
3. 将 Join Key 用于动态过滤 Fact Table。
    
4. 跳过不相关的 Fact Table Partition。
    

![What is Dynamic Partition Pruning in Apache Spark 3.x ?](https://images.openai.com/static-rsc-4/9ZKJ536VpRLSAIFzw-m8PvbA5dWqJq1eVRrfk9byN4jCQqQ2P4RHqG2SLIwDKXRJ4-G1SCkRiyE2n7N7WyTcGzLF_1rWpRszY8Dwzh0X3fDfo5Vq56E_fLOCK9HoIkILFfJ8hoE7xriqGEWVku0erRly6hdlUcDySC885NIIENQ?purpose=inline)

[linkedin.com](https://www.linkedin.com/pulse/what-dynamic-partition-pruning-apache-spark-3x-vitthal-mirji)

关键前提：

DPP 的有效性依赖于 Join Key 与大表分区列等条件的匹配，以及查询计划是否能应用该优化。不是所有 Join 都能使用 DPP。

# Module 9. Bucketing、Z-ORDER 与 Liquid Clustering

这部分需要把传统 Spark 技术与现代 Lakehouse 技术区分开。

## 9.1 Bucketing

Bucketing 是将数据按照指定列的 Hash 值分配到固定数量的 Bucket。

```
df.write \    .bucketBy(8, "customer_id") \    .sortBy("customer_id") \    .saveAsTable("bucketed_orders")
```

简化逻辑：

\[ bucketId = hash(customerId) \bmod bucketCount \]

目的：

- 让相同 Key 的数据进入相同 Bucket。
    
- 在某些 Join / Aggregation 场景减少 Shuffle。
    
- 可以配合排序改善读取效率。
    

但它有局限：

- Bucket 数量固定。
    
- 需要表和查询的分布要求匹配。
    
- Spark 是否能利用 Bucketing 取决于物理计划和相关配置。
    
- 对云对象存储上的现代 Lakehouse 表，不一定是首选。
    

你记录的：

> Number of buckets = Total Size of data / 128 MB

不是通用公式。Bucket 数量需要根据数据量、查询模式、文件大小、并行度等综合决定。

## 9.2 Z-ORDER

Z-ORDER 主要用于 Delta Lake 的数据布局优化。

它让经常一起查询的列在物理数据布局上更接近，以改善 Data Skipping。

例如：

```
OPTIMIZE orders
ZORDER BY (customer_id, order_date)
```

适合：

- 经常按照某些字段过滤。
    
- 数据文件较多。
    
- 希望提升 Data Skipping 效果。
    

Z-ORDER 不是传统意义上的 B-Tree Index。

## 9.3 Liquid Clustering

Liquid Clustering 是现代 Delta Lake 的数据布局方式。

例如：

```
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    amount DECIMAL(10,2)
)
CLUSTER BY (customer_id, order_date);
```

它的优势：

- 不要求预先固定传统 Partition。
    
- 支持随着数据变化重新组织布局。
    
- 更适合高基数过滤列。
    
- 可以减少手动管理分区和 Z-ORDER 的负担。
    

## 9.4 三者比较

|技术|核心目的|是否固定数量|
|---|---|---|
|Bucketing|Hash 分桶，优化部分 Join / Group By|是|
|Z-ORDER|改善文件级数据布局与 Data Skipping|否|
|Liquid Clustering|更灵活的聚类数据布局|否|

面试重点：

不要把 Z-ORDER 和 Bucketing 说成完全相同的技术。

- Bucketing 关注 Hash 分桶和分布。
    
- Z-ORDER 关注多列数据的物理局部性。
    
- Liquid Clustering 提供更灵活的现代数据布局管理。
    

## 9.5 面试问题

Q1. What is the difference between partition pruning and predicate pushdown?

Q2. What is Dynamic Partition Pruning?

Q3. What is the difference between bucketing and partitioning?

Q4. What is Z-ORDER and how does it improve query performance?

Q5. What is Liquid Clustering, and why might you choose it over traditional partitioning?

# Module 10. Broadcast Variables、Accumulators 与 Speculative Execution

## 10.1 Broadcast Variables

Broadcast Variable 是只读共享变量，可以将较小的数据结构广播到 Executor。

```
lookup_map = {    "NL": "Netherlands",    "DE": "Germany",    "FR": "France"}broadcast_map = spark.sparkContext.broadcast(lookup_map)print(broadcast_map.value)
```

主要特点：

- Read-only。
    
- Driver 创建。
    
- Executor 可以读取。
    
- 适合共享小型 Lookup Data。
    
- 避免在每个 Task 中重复发送同一份数据。
    

注意：

- Broadcast Variable 和 Broadcast Join 不是同一个概念。
    
- Broadcast Variable 是 Spark Core 的共享变量机制。
    
- Broadcast Join 是 Spark SQL 的 Join Strategy。
    

## 10.2 Accumulators

Accumulator 是一种分布式聚合变量。

```
bad_records = spark.sparkContext.longAccumulator(    "bad_records")
```

在执行逻辑中增加：

```
def check_record(row):    if row["amount"] is None:        bad_records.add(1)
```

Driver 读取：

```
print(bad_records.value)
```

适用：

- 监控 Bad Records。
    
- 统计某些事件。
    
- Debug 和 Metrics。
    

修正：

Accumulator 不是严格意义上的通用分布式计数数据库。

尤其要注意：

- Transformation 中的 Accumulator 更新可能因为 Task Retry / 重复执行而出现非预期结果。
    
- 不应把它作为关键业务数据的唯一统计结果。
    
- 更可靠的业务计数应通过 DataFrame Aggregation 或持久化结果实现。
    

## 10.3 Speculative Execution

```
spark.conf.set("spark.speculation", "true")
```

当某些 Task 运行明显慢于同类 Task，Spark 可以在其他 Executor 上启动同一 Task 的副本。

如果副本先完成，Spark 可以采用成功结果，并忽略较慢的副本。

主要解决：

- Straggler Tasks。
    
- 个别节点性能异常。
    
- 临时资源竞争。
    

修正：

它不仅仅用于 Node Error，也不一定能解决数据倾斜。

如果某个 Task 慢是因为它的 Partition 本身大得多，那么重新执行同一个 Task 可能还是很慢。

另外，对外部系统写入等有副作用的 Task，要特别考虑重复执行带来的影响。

# Module 11. Spark Scheduling：资源调度

Spark 有两个不同层次的 Scheduling。

## 11.1 Across Applications

不同 Spark Application 之间如何分配资源。

例如：

```
Cluster Manager
    |
    +-- Application A
    |
    +-- Application B
    |
    +-- Application C
```

Cluster Manager 负责管理应用之间的资源分配。

Spark Application 可以使用 Static Allocation 或 Dynamic Allocation。

### Static Allocation

预先申请固定数量的 Executor。

```
--num-executors 10
```

优点：

- 资源配置简单。
    
- 资源使用较稳定。
    

缺点：

- 空闲时仍可能占用资源。
    
- 不能灵活适应工作负载变化。
    

### Dynamic Allocation

```
spark.conf.set("spark.dynamicAllocation.enabled", "true")spark.conf.set("spark.dynamicAllocation.minExecutors", "2")spark.conf.set("spark.dynamicAllocation.maxExecutors", "20")spark.conf.set("spark.dynamicAllocation.executorIdleTimeout", "60s")
```

主要逻辑：

- Task 需求增加时，申请更多 Executor。
    
- Executor 空闲一段时间后，可以释放。
    
- 在不同工作负载之间提高资源利用率。
    

修正：

你原来的：

> executorIdleTimeout = 1s will request executor when 1s no resources

是不正确的。

`executorIdleTimeout` 是 Executor 空闲多久后可以被移除的时间阈值，不是申请 Executor 的等待时间。

## 11.2 Within an Application

一个 Application 内可能同时存在多个 Job。

默认情况下，Job 的调度通常采用 FIFO。

如果希望多个 Job 更公平地共享资源，可以使用 FAIR Scheduler：

```
spark.conf.set("spark.scheduler.mode", "FAIR")
```

例如：

```
Job A: Long-running ETL
Job B: Small interactive query
```

FIFO 可能导致 Job B 等待 Job A。

FAIR Scheduler 可以让不同 Job 的 Task 更公平地获得资源。

注意：

- FAIR Scheduler 是 Application 内部 Job 调度机制。
    
- 它不是 Cluster Manager 层面的跨 Application 资源调度。
    
- 你笔记中的 Python Thread 示例是通过不同线程提交多个 Job 的一种方式；真正的并行度还取决于 Spark Scheduler 和资源情况。
    

## 11.3 面试问题

Q1. What is the difference between static allocation and dynamic allocation?

Q2. What is the difference between FIFO and FAIR scheduling?

Q3. What is speculative execution?

Q4. How do you handle resource contention between multiple Spark applications?

# Module 12. Structured Streaming：流式数据处理

Structured Streaming 是基于 Spark SQL Engine 构建的流处理引擎。

它允许使用与 Batch DataFrame 类似的 API 处理不断到来的数据。

例如：

```
stream_df = spark.readStream \    .format("json") \    .schema(schema) \    .load("/data/events")query = stream_df.writeStream \    .format("delta") \    .outputMode("append") \    .option("checkpointLocation", "/checkpoints/events") \    .start("/data/output")
```

## 12.1 Micro-batch Processing

Structured Streaming 默认主要采用 Micro-batch 模型。

```
Incoming Stream
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
Sink
```

每个 Micro-batch：

1. Source 提供一批新数据。
    
2. Spark 创建对应的执行任务。
    
3. Executor 处理数据。
    
4. Sink 写入数据。
    
5. Spark 提交进度信息。
    
6. 继续处理下一批数据。
    

注意：Structured Streaming 的逻辑是持续运行的查询，但底层默认并不是每条记录都单独创建一个 Spark Job。

## 12.2 Checkpoint

Checkpoint 用于保存 Streaming Query 的进度和恢复信息。

你记录的：

> sources / offsets / commits

是很好的初步理解，但不能把它们理解为所有版本、所有 Source 都严格只有三个目录。

Checkpoint 可能包含：

|信息|作用|
|---|---|
|offsets|记录 Source 读取进度|
|commits|记录成功提交的 Batch|
|sources|Source 相关元数据|
|state|Stateful Operator 的状态数据|
|metadata|Query 相关元数据|

例如：

```
checkpoint/
   |
   +-- metadata
   |
   +-- offsets/
   |
   +-- commits/
   |
   +-- sources/
   |
   +-- state/
```

Checkpoint 的核心作用：

- 记录数据读取进度。
    
- 帮助恢复失败的 Streaming Query。
    
- 恢复 Stateful Operation 的状态。
    
- 避免重新处理已经成功提交的 Batch（在支持相应恢复语义的情况下）。
    

Checkpoint 需要放在可靠、持久化的存储上。

例如云对象存储或分布式文件系统。

不要将其放在临时本地目录中。

## 12.3 Exactly-once Semantics

这是面试中非常重要的考点。

Structured Streaming 可以通过：

- Source Offset Tracking。
    
- Checkpoint。
    
- State Management。
    
- Sink Commit Protocol。
    

实现相应的端到端故障恢复语义。

但要特别注意：

> Checkpoint alone does not guarantee exactly-once delivery to every external system.

例如使用 `foreachBatch` 写入外部数据库时：

- Batch 可能重试。
    
- 同一个 `batch_id` 可能被再次执行。
    
- 如果写入逻辑不是幂等的，可能产生重复记录。
    

因此要使用：

- Idempotent Write。
    
- Upsert / MERGE。
    
- Batch ID 去重。
    
- 唯一键约束或事务机制。
    

## 12.4 Output Modes

Structured Streaming 有三种主要 Output Modes。

|Mode|含义|常见用途|
|---|---|---|
|Append|只输出新增且不会再改变的结果|无状态处理、已确定的窗口结果|
|Update|只输出本次发生变化的结果行|聚合结果更新|
|Complete|输出整个结果表|全量聚合结果|

### Append Mode

```
.writeStream \.outputMode("append")
```

例如：

```
stream_df.filter(col("amount") > 100)
```

只输出新到达且满足条件的记录。

对于有状态的窗口聚合，通常需要 watermark 来确定某些窗口不再接受迟到数据，然后才能输出最终结果。

修正：

Append Mode 不仅仅适用于 Stateless Transformation；它也可以用于带 Watermark 的 Stateful Aggregation。

### Update Mode

```
.writeStream \.outputMode("update")
```

只输出本次有变化的结果行。

例如：

```
Before:
NL -> 10

New events arrive:
NL -> 12

Update Mode outputs:
NL -> 12
```

### Complete Mode

```
.writeStream \.outputMode("complete")
```

每次触发都输出整个结果表。

适合：

- 数据量较小的全量聚合。
    
- 支持 Complete Mode 的 Sink。
    

但会带来重复输出整个结果集的成本。

注意： Output Mode 是否可用取决于具体算子组合和 Sink，并不是任意查询都支持所有模式。

## 12.5 Trigger

Trigger 决定 Structured Streaming 何时执行处理。

### 1. Default Trigger

```
.trigger(processingTime="0 seconds")
```

也可以不指定 Trigger。

当前 Micro-batch 处理完成后，Spark 会尽快开始下一批。

### 2. Processing Time Trigger

```
.trigger(processingTime="5 seconds")
```

尝试每隔 5 秒触发一次 Micro-batch。

如果上一批还没有完成，Spark 不会并行启动下一批相同 Query 的 Micro-batch；可能出现延迟或跳过预定触发时间。

### 3. Once Trigger

```
.trigger(once=True)
```

触发一个 Micro-batch，然后停止。

适合一次性处理一批可用数据的场景。

### 4. AvailableNow Trigger

```
.trigger(availableNow=True)
```

处理启动时可用的数据，可能分成多个 Micro-batch，全部处理完后停止。

它通常比 Once 更适合对可用数据进行分批处理。

修正：

Once 和 AvailableNow 的区别不是“一个处理一批，另一个处理多批”这么简单：

- Once：一个 Micro-batch 后停止。
    
- AvailableNow：处理启动时可用的数据，可以分成多个 Micro-batch，然后停止。
    

### 5. Continuous Processing

传统 Continuous Processing 模式：

```
.trigger(    continuous="1 second")
```

它不是默认 Micro-batch 模式。

需要注意：

- 支持的算子和 Sink 有限制。
    
- 处理保证和 Micro-batch 不完全相同。
    
- 不能把它理解为所有 Delta Sink 都支持 Continuous Mode。
    

另外，新版本 Spark 的实时执行模式持续演进，面试时应区分传统 Continuous Processing 和新版本的 Real-time Mode，不要把两者混为一谈。

## 12.6 Event Time 与 Processing Time

这是 Streaming 的基础概念。

|概念|含义|
|---|---|
|Event Time|事件在业务上实际发生的时间|
|Processing Time|Spark 实际处理该事件的时间|
|Ingestion Time|数据进入系统的时间|

例如：

```
Event happened: 10:00
Arrived at Spark: 10:10
Processed: 10:11
```

这里：

- Event Time = 10:00
    
- Ingestion Time = 10:10
    
- Processing Time = 10:11
    

在实时数据系统中，Event Time 通常更符合业务分析需求。

## 12.7 Watermark

Watermark 用于处理 Late-arriving Data，并帮助清理 Stateful Operator 中不再需要的状态。

例如：

```
df = spark.readStream.table("events")result = df \    .withWatermark("event_time", "10 minutes") \    .groupBy(        window("event_time", "5 minutes"),        "country"    ) \    .count()
```

Watermark 的含义可以理解为：

> Spark 根据已观察到的 Event Time 进度，判断允许迟到数据到达的时间范围，并在适当时机清理旧状态。

假设：

```
Watermark delay = 10 minutes
Maximum observed event time = 12:00
```

Watermark 大致推进到：

```
11:50
```

注意：

- Watermark 不是 Processing Time 定时器。
    
- 它不是“事件发生 10 分钟后自动处理”。
    
- Watermark 不是严格的全局最大 Event Time 减去延迟时间的简单实时公式；具体推进与 Micro-batch、输入分区、多个输入流等因素有关。
    
- 超过 Watermark 的迟到数据可能被丢弃，具体取决于算子和输出语义。
    

## 12.8 Window Operations

Window 用于根据 Event Time 将数据分组。

### 1. Tumbling Window

固定长度，不重叠。

例如：

```
window("event_time", "10 minutes")
```

```
10:00 - 10:10
10:10 - 10:20
10:20 - 10:30
```

每条记录只属于一个窗口。

### 2. Sliding Window

固定长度，但允许重叠。

```
window(    "event_time",    "10 minutes",    "5 minutes")
```

窗口：

```
10:00 - 10:10
10:05 - 10:15
10:10 - 10:20
```

每条记录可能进入多个窗口。

### 3. Session Window

Session Window 根据事件之间的间隔形成动态窗口。

例如：

```
from pyspark.sql.functions import session_windowresult = df.groupBy(    session_window("event_time", "5 minutes"),    "user_id").count()
```

假设同一个用户：

```
10:00 event
10:02 event
10:04 event
10:20 event
```

前三条事件可能形成一个 Session，10:20 的事件可能形成另一个 Session。

修正原笔记：

Session Window 的 5 分钟不是固定窗口长度，也不是“每收到一个事件就无条件延长”。它表示 Session Gap Duration：事件之间的间隔满足相应条件时，Session 可以继续延伸。

## 12.9 `foreachBatch`

`foreachBatch` 可以把每个 Micro-batch 当成一个普通 Batch DataFrame 处理。

```
def process_batch(batch_df, batch_id):    batch_df.createOrReplaceTempView("updates")    batch_df.sparkSession.sql("""        MERGE INTO target_table AS target        USING updates AS source        ON target.id = source.id        WHEN MATCHED THEN UPDATE SET *        WHEN NOT MATCHED THEN INSERT *    """)query = stream_df.writeStream \    .foreachBatch(process_batch) \    .option("checkpointLocation", "/checkpoints/orders") \    .start()
```

适用场景：

- MERGE / UPSERT。
    
- 一个 Stream 写入多个 Sink。
    
- 调用 Batch-only API。
    
- 自定义批处理逻辑。
    

修正：

你记录的：

> Stream can not merge without foreachBatch.

更准确地说：

- `foreachBatch` 是使用 Delta MERGE 处理 Micro-batch 的常见方法。
    
- 不是所有 Streaming Upsert 都必须使用 `foreachBatch`；具体还取决于引擎、Sink 和 API。
    
- `foreachBatch` 默认可能重复执行相同 Batch，因此要考虑幂等性。
    

## 12.10 Source Archive

```
stream_df = spark.readStream \    .format("json") \    .option("cleanSource", "archive") \    .option("sourceArchiveDir", "/archive") \    .load("/input")
```

这个功能是针对支持该选项的 File Stream Source 的文件清理机制。

注意：

- 不是所有 Streaming Source 都支持。
    
- 需要考虑文件系统、权限和 Archive 目录配置。
    
- 不应该把 Archive 当成 Checkpoint 的替代品。
    
- 对象存储上还需要特别考虑文件操作的成本和一致性语义。
    

## 12.11 Structured Streaming 面试问题

Q1. How does Structured Streaming work internally?

回答重点：

- Default micro-batch execution.
    
- Incremental processing.
    
- Source offsets.
    
- Checkpoint.
    
- State management.
    
- Sink commits.
    

Q2. What is the purpose of checkpointing?

Q3. What is the difference between Append, Update and Complete mode?

Q4. What is the difference between Event Time and Processing Time?

Q5. What is a watermark? How does it handle late-arriving data?

Q6. What is the difference between Tumbling, Sliding and Session Windows?

Q7. How do you implement an upsert in Structured Streaming?

Q8. Does Structured Streaming always guarantee exactly-once processing?

回答时一定要补充：Source、Checkpoint 和 Sink 的语义都很重要；外部 Sink 需要考虑幂等性。

# Module 13. Spark Connect 与 Databricks Connect

## 13.1 Spark Connect

Spark Connect 从 Spark 3.4 开始引入，是 Client-Server 架构。

传统 PySpark：

```
Python Client
     |
     v
Py4J
     |
     v
JVM Driver
     |
     v
Executors
```

Spark Connect：

```
Python Client
     |
     v
Spark Connect Protocol
     |
    gRPC
     |
     v
Spark Server
     |
     v
Executors
```

主要特点：

- Client 与 Spark Driver 解耦。
    
- 使用 gRPC 通信。
    
- Client 将 DataFrame API 表达为逻辑计划。
    
- Server 负责分析、优化和执行。
    
- 可以让远程 IDE / 应用连接 Spark 集群。
    

## 13.2 Databricks Connect

Databricks Connect 允许开发者在本地 IDE 中编写 PySpark 代码，并连接 Databricks 远程计算环境执行。

例如：

```
Local IDE
    |
    v
Databricks Connect
    |
    v
Databricks Cluster / Compute
    |
    v
Spark Executors
```

主要用途：

- 本地开发。
    
- 本地调试远程 Spark 程序。
    
- 使用远程 Databricks Compute。
    
- 避免本地必须拥有完整集群资源。
    

注意：

- Databricks Connect 与 Apache Spark Connect 有关联，但不是完全相同的产品名称。
    
- Databricks Connect 的具体功能和兼容性取决于 Databricks Runtime、客户端版本和计算类型。
    

## 13.3 面试问题

Q1. What is Spark Connect?

Q2. What is the difference between traditional PySpark and Spark Connect?

Q3. What is Databricks Connect used for?

Q4. Why would you use a remote Spark cluster instead of running Spark locally?

# Module 14. Spark 性能排查：实际 Data Engineer 工作流程

前面的模块是 Spark 原理，面试中还经常会给你一个实际场景：

> A Spark job that used to finish in 20 minutes now takes 2 hours. How would you investigate it?

建议按照下面的顺序回答。

## 14.1 第一步：Spark UI

Spark UI 常用入口：

- Jobs：查看 Job 执行情况。
    
- Stages：查看 Stage 耗时和 Task 分布。
    
- Executors：查看 Executor 资源和状态。
    
- SQL：查看 SQL 执行计划。
    
- Storage：查看 Cache / Persist。
    
- Environment：查看 Spark 配置。
    

重点指标：

|指标|说明|需要关注的问题|
|---|---|---|
|Duration|执行耗时|哪个 Stage 最慢|
|Input Size|输入数据量|是否读取过多数据|
|Shuffle Read|Shuffle 读取量|是否 Shuffle 过大|
|Shuffle Write|Shuffle 写入量|是否发生大量数据重分布|
|Spill|内存不足时写入磁盘的数据|是否存在内存压力|
|GC Time|JVM GC 耗时|是否有大量对象或 GC 压力|
|Task Duration|Task 执行时间|是否存在数据倾斜|
|Failed Tasks|失败 Task 数量|是否存在资源或数据问题|

## 14.2 第二步：识别常见问题

|现象|可能原因|优化方向|
|---|---|---|
|某一个 Task 特别慢|Data Skew|AQE、Salting、预聚合|
|Shuffle Read 很大|Shuffle 数据量大|减少数据、提前过滤、调整 Join|
|Spill 很大|内存压力|调整 Partition、优化 Join、减少缓存|
|GC Time 很高|JVM 对象和内存压力|调整 Executor、Serialization|
|Executor OOM|Partition 过大、Python 内存、Broadcast 过大|调整内存、拆分数据、优化计划|
|Driver OOM|collect、toPandas、过大广播元数据|避免收集大量数据到 Driver|
|Task 数量太少|Partition 太少|Repartition|
|输出文件过多|Partition 太多|Coalesce、AQE、REBALANCE|
|SQL 扫描数据过多|缺少过滤或数据布局不合理|Predicate Pushdown、Partition Pruning、Data Skipping|

## 14.3 第三步：优化执行计划

```
df.explain("formatted")
```

重点检查：

- 是否出现不必要的 `Exchange`。
    
- 是否使用了预期的 `BroadcastHashJoin`。
    
- 是否发生了不必要的 `Sort`。
    
- 是否有过大的 Scan。
    
- 是否有 Partition Pruning。
    
- 是否存在不合理的 Join 顺序。
    
- AQE 最终计划是否发生了预期变化。
    

## 14.4 第四步：优化代码

优先级通常可以这样安排：

1. 减少不必要的数据读取。
    
2. 尽可能提前过滤数据。
    
3. 只选择需要的列。
    
4. 使用 Spark Built-in Functions 替代 Python UDF。
    
5. 优化 Join Strategy。
    
6. 检查数据倾斜。
    
7. 优化 Partition 数量。
    
8. 减少不必要的 Cache。
    
9. 最后才考虑增加 Cluster Memory / CPU。
    

面试中不要一上来就说增加 Executor 数量。 先判断瓶颈在哪里，再决定是否需要扩容。

# Module 15. 原始笔记中需要重点纠正的内容

下面是最容易在面试中答错的地方，建议单独记忆。

|原笔记|正确理解|
|---|---|
|Cluster Manager 创建 Driver|取决于部署模式；YARN Cluster Mode 中 ApplicationMaster 所在 Container 启动 Driver|
|每个 Spark Job 有一个 Driver|通常一个 Spark Application 有一个 Driver；一个 Application 可以有多个 Job|
|一个 Core 处理一个 Partition|一个 Task 通常处理一个 Partition；一个 Core 同时通常运行一个 Task|
|所有 Join 都是 Wide Transformation|不一定，Broadcast Join 通常避免 Join Shuffle|
|`read()` 是 Action|`read()` 通常创建 DataFrame；`write()`、`collect()` 等触发执行|
|每个 Action 一定只有一个 Job|通常会触发 Job，但一个 Action 也可能涉及多个 Job 或内部执行阶段|
|每个 Stage 的最后一步都是 Wide Transformation|Shuffle 通常是 Stage Boundary；上游 Stage 执行 Shuffle Write，下游 Stage 执行 Shuffle Read|
|所有 Task 都把数据返回 Driver|通常 Task 直接写入 Sink 或 Shuffle；只有需要返回的结果才传回 Driver|
|`coalesce()` 可以增加 Partition|默认不能增加；主要用于减少 Partition|
|128–256 MB 是 Partition 的固定标准|只是常见调优起点，实际要看数据和算子|
|AQE 只合并小 Partition|还可以处理 Skew Join 和动态调整 Join Strategy|
|Broadcast Join 完全没有网络传输|小表仍然需要广播；它主要避免 Join 两侧的 Shuffle|
|Executor Spill 后不会 OOM|Spill 不能解决所有内存问题|
|Memory Overhead 只用于 Python|还包括 Native Memory、Network Buffer 等|
|Cache 默认一定是 MEMORY_AND_DISK|要区分 RDD 与 DataFrame，并检查当前版本默认 Storage Level|
|Watermark = 当前时间 - 延迟时间|它基于 Event Time 进度推进，不是 Processing Time 定时器|
|Append 只支持 Stateless|带 Watermark 的 Stateful Aggregation 也可以使用 Append|
|Once 一次处理所有可用数据|Once 只执行一个 Micro-batch；AvailableNow 可处理多批后停止|
|Session Window 固定 5 分钟|Gap Duration 决定 Session 的延伸条件，Session 本身长度可以变化|
|Checkpoint 保证任何 Sink Exactly-once|还需要 Source 和 Sink 的提交语义；外部 Sink 常需要幂等性|
|Accumulator 是准确的业务计数器|不应该作为关键业务数据的唯一统计来源|
|Speculative Execution 解决数据倾斜|它主要缓解慢 Task / Straggler，不会自动解决大 Partition|
|Dynamic Allocation 的 Idle Timeout 是申请 Executor 的等待时间|它控制 Executor 空闲多久后可以释放|
|Z-ORDER 和 Bucketing 一样|两者的布局和优化目标不同|
|Spark Connect 就是 Databricks Connect|有关联，但产品和适用环境不完全相同|

# Module 16. 荷兰 Data Engineer 面试：重点准备清单

结合荷兰 Data Engineer / Data Platform Engineer 常见的 Spark 工程能力要求，建议把准备时间集中在以下主题。

## 16.1 第一优先级：必须能够解释清楚

高优先级

|Topic|必须掌握的内容|
|---|---|
|Spark Architecture|Driver、Executor、Cluster Manager|
|Execution Model|Lazy Evaluation、Job、Stage、Task|
|Catalyst|Logical Plan、Physical Plan、AQE|
|Shuffle|Narrow / Wide、Shuffle Read / Write|
|Partition|Repartition、Coalesce、Partition 数量|
|Join|Broadcast、Sort Merge、Shuffle Hash|
|Data Skew|识别原因、AQE、Salting|
|Memory|Driver OOM、Executor OOM、Spill、GC|
|Structured Streaming|Checkpoint、Watermark、Output Mode|
|Performance Debugging|Spark UI、Explain Plan、Stage Metrics|

## 16.2 第二优先级：工程实践与场景题

中高优先级

|Topic|必须掌握的内容|
|---|---|
|Cache / Persist|何时缓存，何时不缓存|
|Dynamic Allocation|如何适应变化的工作负载|
|Scheduling|FIFO、FAIR|
|Broadcast Variables|与 Broadcast Join 的区别|
|Accumulators|使用场景和限制|
|Partition Pruning|如何减少扫描|
|DPP|Join 中的动态过滤|
|Bucketing|固定 Hash 分桶|
|Z-ORDER / Liquid Clustering|Lakehouse 数据布局|
|Spark Connect|Client-Server 架构|

## 16.3 第三优先级：需要知道，但不必过度深入

- JVM GC 算法细节。
    
- Off-Heap Memory 的底层实现。
    
- Spark 内部 Shuffle Manager。
    
- 各类 Storage Level 的历史版本差异。
    
- Continuous Processing 的底层实现。
    
- Cluster Manager 的内部调度算法。
    

# Module 17. 最值得练习的 15 道综合面试题

这些问题比单纯背定义更接近实际 Data Engineer 面试。

面试自测

0/15 已查看提示

1Explain how Spark executes a DataFrame query from transformations to execution.

2What happens internally when you perform a groupBy operation?

3Why is shuffle expensive, and how do you reduce it?

4How would you optimize a join between a 500 GB fact table and a 10 MB dimension table?

5A Spark job has one task running much longer than all other tasks. How would you investigate it?

6What is the difference between repartition and coalesce?

7What is AQE, and how does it improve query performance?

8What is the difference between Driver OOM and Executor OOM?

9What is the difference between Execution Memory and Storage Memory?

10How do you decide whether to cache a DataFrame?

11How do you troubleshoot a Spark application with high GC time?

12How does Structured Streaming recover from a failure?

13What is a watermark, and how does it handle late-arriving events?

14How would you implement an idempotent streaming upsert?

15How would you optimize a Spark pipeline processing several terabytes of data daily?

展开全部提示

## 最后：建议建立的 Spark 知识主线

把整个 Spark 串成一条完整的执行链路：

1. Application Submission

Cluster Manager · Deployment Mode

2. Driver & Executors

SparkSession · SparkContext · Resource Allocation

3. Logical & Physical Planning

Catalyst · Logical Plan · Physical Plan · AQE

4. Distributed Execution

Job · Stage · Task · Partition · Shuffle

5. Performance Optimization

Join · Skew · Memory · Cache · Partition Pruning

6. Data Output & Streaming

Batch Write · Structured Streaming · Checkpoint · Watermark

7. Production Monitoring

Spark UI · Logs · Metrics · Reliability · Cost

最重要的学习建议： 不要只背 Spark 参数。面试时更有价值的是能够解释一个数据集从 Driver 生成执行计划，到 Executor 执行 Task、发生 Shuffle、出现数据倾斜，再到你如何通过 Spark UI 定位并优化的完整过程。