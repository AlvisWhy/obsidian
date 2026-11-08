# 1. Compute Architecture：Serverless vs Classic Compute

Databricks Compute 根据计算资源的管理方式，可以区分为 Serverless Compute 和 Classic Compute。

## 1.1 Serverless Compute

Databricks-managed infrastructure

Serverless Compute 是 Databricks 管理底层计算基础设施的计算模式。

你的笔记：

```
Serverless compute: in Databricks subscription
```

这个理解基本正确，但需要补充：

Serverless 不代表完全没有服务器，而是用户不需要直接管理底层 VM、Cluster provisioning 等基础设施。

架构：

Customer

Submit SQL / Notebook / Job

Databricks Control Plane

Provisioning and orchestration

Databricks-managed Serverless Compute

Compute execution infrastructure

特点：

- Databricks 管理计算基础设施。
    
- 用户不需要手动配置 Worker VM。
    
- 通常能够更快启动计算任务。
    
- 可以减少基础设施管理工作。
    
- 支持的 Workload 和功能取决于具体 Serverless Compute 类型。
    
- 用户通常不能像 Classic Compute 一样自由配置所有底层 Cluster 参数。
    

适用场景：

- Serverless SQL Warehouses。
    
- Serverless Jobs。
    
- Serverless Notebook workloads（在支持的环境中）。
    

注意：Serverless 并不意味着没有 DBU 成本，也不意味着一定比 Classic Compute 便宜。

## 1.2 Classic Compute

Customer cloud infrastructure

Classic Compute 使用客户云账户中的计算基础设施。

你的笔记：

```
Classic compute: in customer subscription
```

基本正确。

例如：

- Azure Databricks：Customer Azure Subscription。
    
- AWS Databricks：Customer AWS Account。
    
- GCP Databricks：Customer Google Cloud Project。
    

架构：

Customer

Databricks Control Plane

Cluster management and orchestration

Customer Cloud Subscription

Driver VM

Worker VM

特点：

- 用户可以选择 VM 类型。
    
- 可以配置 Worker 数量。
    
- 可以配置 Driver 和 Executor 相关资源。
    
- 可以配置 Autoscaling。
    
- 用户对 Compute 配置有更高的控制能力。
    
- 需要考虑云基础设施的成本。
    

## 1.3 Serverless vs Classic Compute

|对比|Serverless|Classic|
|---|---|---|
|Infrastructure 管理|Databricks|Customer cloud account|
|VM 管理|Databricks 管理|用户配置或管理|
|Cluster 配置自由度|相对有限|更高|
|启动体验|通常更快|需要 Provisioning|
|运维复杂度|较低|较高|
|适用场景|快速执行、托管工作负载|自定义计算、特定配置需求|

面试问题

Q1. What is the difference between Serverless and Classic Compute in Databricks?

回答：

Serverless Compute is fully managed by Databricks, so customers do not need to manage the underlying compute infrastructure.

Classic Compute runs on the customer's cloud infrastructure, providing more control over VM types, cluster configurations and scaling.

Q2. Does serverless mean that there are no servers?

回答：

No. Serverless means that the underlying infrastructure is abstracted and managed by the cloud or platform provider. Compute resources still exist, but customers do not manage them directly.

Q3. When would you choose Classic Compute instead of Serverless?

回答：

I would choose Classic Compute when I need more control over VM configurations, Spark settings, libraries, networking or workloads that are not supported by Serverless.

# 2. Compute Types：All-Purpose vs Job Compute

这个模块主要讨论：谁创建 Compute、Compute 用来做什么，以及什么时候释放计算资源。

注意：这两个属于 Classic Compute 的类型，不是 Serverless vs Classic 的同一层级分类。

## 2.1 All-Purpose Compute

以前也经常称为 All-Purpose Cluster，现在 Databricks 文档主要使用 All-Purpose Compute。

你的笔记：

```
All purpose cluster:
- created manually
- persistent
- shared
- expensive
```

基本正确。

特点：

- 通常由开发人员手动创建。
    
- 主要用于交互式开发。
    
- 可以连接多个 Notebooks。
    
- 可以在多个用户之间共享（取决于 Access Mode 和权限）。
    
- 执行完一条 Notebook 命令后，Compute 不会自动终止。
    
- 可以设置 Auto Termination。
    

典型应用：

```
Developer
    |
    v
All-Purpose Compute
    |
    +---- Notebook A
    |
    +---- Notebook B
    |
    +---- Notebook C
```

例如：

开发人员正在开发一个 ETL Pipeline：

```
df = spark.read.table("bronze.orders")display(df)
```

然后修改 Transformation：

```
df = df.filter("amount > 100")
```

反复运行 Notebook，进行数据探索和调试。

这种情况下，All-Purpose Compute 非常方便。

缺点：

如果开发人员离开了，却没有终止 Compute，就会持续产生费用。

因此，不建议将 All-Purpose Compute 作为常规生产 Job 的计算资源。

## 2.2 Job Compute

Job Compute 是专门为自动化 Job 设计的计算资源。

你的笔记：

```
Job Cluster:
- created by jobs
- automated workload
- terminated once job ends
- isolated
```

基本正确。

更准确地说：

- 由 Lakeflow Jobs 在运行时创建或分配。
    
- 专门运行自动化 Tasks。
    
- 可以配置专属的计算资源。
    
- Job Run 完成后，新的 Job Compute 会被终止。
    
- 不同 Job Runs 可以使用独立的 Compute。
    
- 同一个 Job Run 中，多个 Tasks 可以共享同一个 Job Compute。
    

例如：

```
Daily ETL Job
      |
      v
Create Job Compute
      |
      +---- Task 1: Ingestion
      |
      +---- Task 2: Bronze
      |
      +---- Task 3: Silver
      |
      +---- Task 4: Gold
      |
      v
Job Completed
      |
      v
Terminate Job Compute
```

注意：

如果一个 Job 的多个 Tasks 使用同一个 Job Compute，那么 Compute 会在这些 Tasks 之间复用，而不是每个 Task 都创建一套 VM。

如果多个 Tasks 使用不同 Compute，则可以独立配置不同的计算资源。

## 2.3 All-Purpose vs Job Compute

|对比|All-Purpose|Job Compute|
|---|---|---|
|创建方式|通常手动创建|Job 运行时创建|
|主要用途|Interactive development|Automated workload|
|生命周期|持续存在，直到终止|通常随 Job Run 结束|
|Notebook|适合|也支持 Notebook Task|
|资源复用|多个用户/Notebook 可共享|同一个 Job Run 的 Tasks 可以共享|
|成本管理|需要关注空闲资源|运行结束后释放|
|生产 ETL|通常不推荐|推荐|

生产环境最佳实践：

优先考虑 Serverless Jobs 或 Classic Job Compute，而不是长期运行的 All-Purpose Compute。

Databricks 当前也推荐 Serverless 作为多数自动化工作负载的首选。

Databricks on AWS+2

## 2.4 面试问题

Q1. What is the difference between All-Purpose Compute and Job Compute?

回答：

All-Purpose Compute is mainly designed for interactive development and data exploration. It can remain running after notebook execution.

Job Compute is designed for automated workloads. It is provisioned for job execution and normally terminated after the job run completes.

Q2. Why shouldn't you use All-Purpose Compute for production ETL jobs?

回答：

All-Purpose Compute can remain running when idle, resulting in unnecessary costs. Dedicated Job Compute or Serverless Jobs provide better resource lifecycle management and cost attribution.

Q3. Can multiple tasks in the same Databricks Job share one Job Compute?

回答：

Yes. Multiple tasks can share the same Job Compute within a Job Run. This can reduce compute startup overhead and improve resource utilization.

# 3. Single-node vs Multi-node Compute

这个模块主要讨论 Spark Cluster 的物理计算架构。

## 3.1 Single-node Compute

Single-node Compute 只有一个计算节点。

这个节点同时承担：

- Driver responsibilities
    
- Spark execution
    

架构：

Single Node

One machine

Driver

Local Execution

特点：

- 只有一台 VM。
    
- 没有独立的 Worker Nodes。
    
- 适合小型数据处理。
    
- 适合开发、测试和数据探索。
    
- 不适合需要大量分布式计算的任务。
    

例如：

```
df = spark.read.csv("/Volumes/main/default/raw/small.csv")df.count()
```

对于小规模数据，Single-node Compute 就可能足够。

## 3.2 Multi-node Compute

Multi-node Compute 由 Driver Node 和多个 Worker Nodes 组成。

架构：

Driver Node

Spark Driver / DAG Scheduler / Task Scheduler

Distribute tasks

Worker 1

Executor

Tasks

Worker 2

Executor

Tasks

Worker 3

Executor

Tasks

特点：

- Driver 负责调度和协调。
    
- Workers 提供分布式计算资源。
    
- 每个 Worker 可以运行一个或多个 Executors。
    
- Executors 执行 Tasks。
    
- 适合大规模 ETL、Join、Aggregation 等工作负载。
    

例如：

```
df = spark.read.parquet("s3://bucket/large_dataset/")result = (    df.groupBy("country")      .sum("revenue"))
```

Spark 可以将数据分成多个 Partition，并分发给不同 Executors 并行处理。

## 3.3 Single-node vs Multi-node

|对比|Single-node|Multi-node|
|---|---|---|
|节点数量|1|多个|
|Driver|有|有|
|Worker Nodes|没有独立 Worker|有|
|分布式执行|不跨节点|支持|
|成本|通常较低|通常较高|
|适用场景|小型数据、开发测试|大规模 ETL|

注意：

Single-node 不代表 Spark 完全没有并行执行。它仍然可以利用单台机器上的多个 CPU Cores，只是不能跨多台机器分布式执行。

## 3.4 Driver vs Executor

这个知识点应该和你之前学习的 Spark Architecture 联系起来。

|Driver|Executor|
|---|---|
|运行 Driver Program|运行 Task|
|创建 SparkSession|执行计算|
|构建 DAG|执行 Stage 中的 Tasks|
|调度任务|处理数据 Partition|
|收集执行结果|缓存数据|
|管理应用生命周期|向 Driver 汇报状态|

例如：

```
df = spark.read.table("bronze.orders")result = df.groupBy("country").count()result.show()
```

大致执行过程：

1. Driver 接收 Python API 操作。
    
2. Spark 构建 Logical Plan。
    
3. Catalyst Optimizer 优化执行计划。
    
4. Spark 生成 Physical Plan。
    
5. Driver 将任务调度给 Executors。
    
6. Executors 处理数据 Partition。
    
7. Driver 协调执行并返回结果。
    

## 3.5 面试问题

Q1. What is the difference between single-node and multi-node compute?

回答：

Single-node compute runs all Spark processing on a single machine, while multi-node compute distributes workloads across multiple worker machines using executors.

Q2. When would you use single-node compute?

回答：

I would use it for development, small datasets, lightweight transformations and workloads that do not require distributed processing.

Q3. What is the role of the Driver and Executors?

回答：

The Driver coordinates the Spark application, builds execution plans and schedules tasks. Executors run those tasks on data partitions and can cache intermediate data.

Q4. Can a single-node Spark application run tasks in parallel?

回答：

Yes. It can use multiple CPU cores on the same machine. However, it cannot distribute computation across multiple physical machines.

# 4. Access Modes — Dedicated vs Standard vs No Isolation Shared

这是你记录中需要重点更新的部分。

Access Mode 主要控制：

Who can access the compute resource and how user workloads are isolated.

它属于安全和权限控制机制，而不只是决定是否共享计算资源。

## 4.1 Dedicated Access Mode

Dedicated

你的笔记：

```
Dedicated (for single user)
```

基本正确，但不够完整。

Dedicated Compute 可以分配给：

- 一个指定用户。
    
- 一个指定 Group。
    

如果分配给 Group，Group 中符合权限要求的用户可以使用这个 Compute。

特点：

- Compute 专门分配给指定用户或 Group。
    
- 提供专用的执行环境。
    
- 适用于需要特殊权限或功能的 Workload。
    
- 可以支持一些 Standard 不支持的 Spark API 或特殊功能。
    

典型应用：

- 特殊 ML Workload。
    
- GPU Workload。
    
- 需要 R 语言的工作负载。
    
- 需要更高底层资源访问权限的场景。
    

## 4.2 Standard Access Mode

Recommended for most workloads

以前也被称为 Shared Access Mode。

你的笔记：

```
Standard (shared)
```

基本正确，但需要特别注意：

Standard 允许多个用户共享同一 Compute，但用户之间仍然有隔离机制。

例如：

```
           Standard Compute
                  |
          +-------+-------+
          |       |       |
       User A  User B  User C
          |       |       |
       Isolated Isolated Isolated
       Context  Context  Context
```

特点：

- 多个用户可以使用同一个 Compute。
    
- 用户之间有安全隔离。
    
- 支持 Unity Catalog。
    
- 适合一般 Data Engineering ETL。
    
- 可以降低重复创建 Compute 的成本。
    

Databricks 使用 Lakeguard 等隔离机制，防止共享 Compute 上的用户任意访问其他用户的执行环境和底层资源。

Databricks on AWS+1

## 4.3 No Isolation Shared

这是 Legacy Access Mode。

你原来的笔记：

```
No isolation shared
```

需要补充：

No Isolation Shared 允许多个用户共享 Compute，但不像 Standard 一样提供完整的用户工作负载隔离。

特点：

- 属于旧的 Access Mode。
    
- 安全隔离能力较弱。
    
- 不适合现代 Unity Catalog 治理环境。
    
- 新项目不应该优先选择。
    

建议使用 Standard 或 Dedicated。

## 4.4 三种 Access Modes 对比

|对比|Dedicated|Standard|No Isolation Shared|
|---|---|---|---|
|用户数量|指定用户或 Group|多用户|多用户|
|用户隔离|专用环境|有隔离|缺乏完整隔离|
|UC 支持|支持|支持|不适合现代 UC 工作流|
|适用场景|特殊功能、专用工作负载|普通 ETL、协作|Legacy|
|推荐程度|特定需求使用|首选|不推荐|

注意：Access Mode 与 All-Purpose / Job Compute 是两个独立维度。

例如：

```
Classic Compute
    |
    +---- All-Purpose Compute
    |         |
    |         +---- Standard
    |         +---- Dedicated
    |
    +---- Job Compute
              |
              +---- Standard
              +---- Dedicated
```

并不是 All-Purpose 一定是 Standard，也不是 Job Compute 一定是 Dedicated。

## 4.5 面试问题

Q1. What is the difference between Dedicated and Standard access modes?

回答：

Dedicated compute is assigned to a specific user or group, whereas Standard compute allows multiple users to share the same compute resource with workload isolation.

Q2. Why would you choose Standard access mode for a Data Engineering team?

回答：

Standard access mode supports multiple users with workload isolation, making it suitable for collaborative ETL development while reducing the need for separate compute resources.

Q3. Is Standard Compute less secure because it is shared?

回答：

No. Shared does not mean that users can access each other's execution environments. Standard access mode provides workload isolation between users.

Q4. Is No Isolation Shared recommended for new Databricks projects?

回答：

No. It is a legacy access mode without the same level of user isolation. I would prefer Standard or Dedicated access modes.

# 5. Compute Optimization — Auto Termination & Autoscaling

这两个功能非常重要，因为它们直接影响 Databricks 的性能和成本。

注意：

- Auto Termination：控制什么时候关闭 Compute。
    
- Autoscaling：控制需要多少 Worker Nodes。
    

两者解决的是不同问题。

## 5.1 Auto Termination

Auto Termination 用于在 Compute 空闲一段时间后自动终止计算资源。

例如：

```
Auto Termination = 30 minutes
```

执行过程：

```
10:00 — User executes Notebook
10:15 — Notebook execution finishes
10:15 — Compute becomes idle
10:45 — Compute automatically terminates
```

特点：

- 减少空闲 Compute 的费用。
    
- 特别适合 All-Purpose Compute。
    
- 不会因为单个 Task 完成就立即终止。
    
- 需要达到配置的空闲时间才会终止。
    

例如：

假设开发人员每天只工作 2 小时，但 Compute 一直运行 24 小时。

开启 Auto Termination 后，可以显著降低空闲期间的费用。

注意：Auto Termination 判断的是 Compute 是否处于空闲状态，不只是 Notebook 页面有没有操作。正在执行 Spark Job 的时候，不应该简单地认为 Compute 是空闲的。

## 5.2 Autoscaling

Autoscaling 根据工作负载自动增加或减少 Worker Nodes。

例如：

```
Min Workers = 2
Max Workers = 8
```

当工作量增加：

```
2 Workers → 4 Workers → 8 Workers
```

当工作量减少：

```
8 Workers → 4 Workers → 2 Workers
```

架构：

Low workload

Worker 1

Worker 2

Idle

Idle

Idle

High workload

W1

W2

W3

W4

W5

W6

W7

W8

示意：实际 Worker 数量取决于配置和工作负载。

配置示例：

```
autoscale:
  min_workers: 2
  max_workers: 8
```

注意：

- Autoscaling 主要调整 Worker 数量。
    
- 不会自动解决所有 Spark 性能问题。
    
- 增加 Worker 不一定总能提高性能。
    
- Shuffle、数据倾斜、Driver 瓶颈等问题仍然需要优化。
    

另外，Databricks 的 Autoscaling 有自己的扩缩容策略，并不是简单地按照 CPU 使用率达到某个百分比就立即增加 Worker。

## 5.3 Auto Termination vs Autoscaling

|对比|Auto Termination|Autoscaling|
|---|---|---|
|作用|终止整个 Compute|调整 Worker 数量|
|触发条件|空闲时间|工作负载需求|
|目的|节省空闲成本|优化性能和资源利用率|
|典型配置|30 minutes|Min 2, Max 8|
|是否关闭整个 Compute|是|否|

## 5.4 面试问题

Q1. What is the difference between auto termination and autoscaling?

回答：

Auto termination shuts down compute after a specified period of inactivity. Autoscaling dynamically increases or decreases the number of workers based on workload requirements.

Q2. How would you reduce Databricks compute costs?

回答：

I would enable auto termination for interactive compute, use serverless or dedicated job compute for automated workloads, configure autoscaling and select appropriate VM sizes.

I would also monitor idle compute and avoid over-provisioning resources.

Q3. Does adding more workers always improve Spark performance?

回答：

No. Performance also depends on data partitioning, shuffle operations, data skew, memory usage and driver bottlenecks. Adding workers can even increase communication overhead if the workload is not suitable for distributed execution.

# 6. Databricks Compute Billing

你记录的公式：

```
Billing = DBU cost + VM infrastructure cost
```

这个公式对于 Classic Compute 是一个很好的基础理解，但不能直接套用到所有 Compute 类型。

## 6.1 What is DBU?

DBU = Databricks Unit。

DBU 是 Databricks 用来衡量计算服务使用量的计量单位。

Databricks 会根据：

- Compute 类型。
    
- Workload 类型。
    
- Runtime。
    
- 使用时长。
    
- 定价类别。
    

计算对应的 DBU 消耗。

不同 Compute 类型可能有不同 DBU Rate。

例如：

```
Classic All-Purpose Compute
Classic Jobs Compute
Serverless Jobs
Serverless SQL Warehouse
```

这些类型的计费标准可能不同。

## 6.2 Classic Compute Billing

对于 Classic Compute，可以用下面的方式理解：

\[ \text{Total Cost} = \text{Databricks DBU Cost} + \text{Cloud Infrastructure Cost} \]

例如 Azure Databricks：

```
Databricks DBU charge
       +
Azure VM charge
       +
Other cloud resource charges
```

其他可能产生的云成本：

- Storage。
    
- Network transfer。
    
- Public IP 等相关资源。
    
- 其他依赖的云服务。
    

具体是否单独收费取决于云平台和资源配置。

## 6.3 Serverless Compute Billing

Serverless Compute 的收费方式不同。

它使用 Serverless 对应的计费价格，底层基础设施由 Databricks 管理。

因此，不应该简单地将 Serverless 计费理解为：

```
Customer pays DBU + separately manages and pays each VM
```

用户通常不会像 Classic Compute 一样单独管理底层 Worker VM。

## 6.4 Idle Compute 为什么会产生费用？

例如：

```
All-Purpose Compute:

09:00 - 10:00  Running ETL
10:00 - 18:00  Idle
18:00 - 19:00  Running ETL
```

如果没有 Auto Termination，空闲期间仍可能持续产生 DBU 和云基础设施费用。

这就是为什么：

- 开发环境应该启用 Auto Termination。
    
- 生产任务应该使用合适的 Job Compute。
    
- 应该定期检查长期运行的 Compute。
    

Databricks 也建议使用 Job Compute 或 Serverless Compute，以便更准确地追踪 Job 成本。

Databricks on AWS+2

## 6.5 面试问题

Q1. How is Databricks compute billed?

回答：

For classic compute, customers generally pay Databricks DBU charges plus the underlying cloud infrastructure costs. Serverless uses separate serverless pricing, with the infrastructure managed by Databricks.

Q2. Why does idle compute still cost money?

回答：

Because the compute infrastructure remains provisioned even when no workload is running. The resources can continue accumulating DBU and cloud infrastructure charges until termination.

Q3. How would you monitor and optimize compute costs in production?

回答：

I would monitor DBU consumption and cloud infrastructure costs, use job-level cost attribution, enable auto termination and autoscaling, and choose the appropriate compute type and VM size.

# 7. Spark Submit

这个模块与前面的 Compute Architecture 联系非常紧密。

首先要理解：

Spark Submit 是 Apache Spark 提供的应用程序提交工具，用于将 Spark Application 提交到指定的 Cluster Manager。

它主要用于独立部署 Spark Application，而不是像 Notebook 一样交互式地逐行执行代码。

## 7.1 Spark Submit Syntax

你记录的语法：

```
spark-submit \
  --class \
  --master \
  --deployment \
  [application-args]
```

需要修正：

正确参数是：

```
--deploy-mode
```

而不是：

```
--deployment
```

完整结构：

```
spark-submit \
  --class <main-class> \
  --master <cluster-manager> \
  --deploy-mode <client|cluster> \
  --conf <key=value> \
  --driver-memory <memory> \
  --driver-cores <number> \
  --executor-memory <memory> \
  --executor-cores <number> \
  --num-executors <number> \
  <application-jar-or-file> \
  [application-arguments]
```

每个参数的含义：

|参数|作用|
|---|---|
|`--class`|JVM Application 的 Main Class|
|`--master`|指定 Cluster Manager|
|`--deploy-mode`|指定 Driver 的部署位置|
|`--conf`|设置 Spark Configuration|
|`--driver-memory`|Driver Memory|
|`--driver-cores`|Driver CPU Cores|
|`--executor-memory`|每个 Executor 的 Memory|
|`--executor-cores`|每个 Executor 的 CPU Cores|
|`--num-executors`|Executor 数量|

注意：

`--class` 主要用于 Java/Scala 等 JVM Application。提交 Python Application 时，通常直接提供 `.py` 文件路径。

## 7.2 `--master` — Cluster Manager

决定 Spark Application 使用哪种执行环境。

常见值：

|Master|含义|
|---|---|
|`local`|本地单线程|
|`local[3]`|本地使用 3 个执行线程|
|`local[*]`|使用本地可用的处理器核心|
|`spark://host:7077`|Spark Standalone Cluster|
|`yarn`|Hadoop YARN|
|`k8s://...`|Kubernetes|

例如：

```
spark-submit \
  --master local[3] \
  app.py
```

表示在本地运行 Spark Application，使用 3 个执行线程。

注意：

`local[3]` 不是一个真正的分布式 Spark Cluster，不会创建三台 Worker Machines。

## 7.3 `--deploy-mode` — Client vs Cluster

这是 Spark Submit 最重要的知识点。

它决定：

Where does the Spark Driver process run?

### A. Client Mode

你的笔记：

```
Driver runs on client machine.
No driver in the cluster.
```

基本正确，但需要注意 YARN 的 ApplicationMaster 仍然存在。

架构：

Client Machine

Driver runs here

spark-submit process

YARN Cluster

ApplicationMaster

Requests and negotiates cluster resources

Worker 1

Executor

Worker 2

Executor

特点：

- Driver 在提交任务的 Client Machine 上运行。
    
- Executors 在 Cluster 中运行。
    
- Client Machine 必须保持运行。
    
- Driver 日志通常可以直接从 Client 侧查看。
    
- 适合交互式 Shell、开发调试等场景。
    

例如：

```
spark-submit \
  --master yarn \
  --deploy-mode client \
  app.jar
```

### B. Cluster Mode

你的笔记：

```
Driver is running in the ApplicationMaster (AM)
container on a worker node in the cluster.
```

对于 YARN，这是正确的。

架构：

Client Machine

Submit application and disconnect

YARN Cluster

ApplicationMaster Container

Driver runs here

Worker 1

Executor

Worker 2

Executor

特点：

- Driver 在 Cluster 内运行。
    
- YARN ApplicationMaster Container 中运行 Driver。
    
- Client 不需要一直保持运行。
    
- 更适合生产环境的批处理 Application。
    
- Driver 日志需要从 Cluster 获取。
    

例如：

```
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  app.jar
```

重要补充：

ApplicationMaster 和 Driver 不是两个完全相同的概念。

在 YARN Cluster Mode 下，Driver 运行在 ApplicationMaster 进程所在的容器中。

而在 YARN Client Mode 下，ApplicationMaster 负责向 YARN 申请资源，Driver 仍然在 Client Machine。

这一点是面试时容易被追问的地方。

GitHub+2

## 7.4 Client Mode vs Cluster Mode

|对比|Client Mode|Cluster Mode|
|---|---|---|
|Driver 位置|Client Machine|Cluster 内|
|Executor 位置|Cluster|Cluster|
|Client 是否需要持续运行|是|提交后可以断开|
|Driver Logs|Client 侧|Cluster 侧|
|适用场景|交互式开发、调试|生产批处理|
|YARN AM|存在|存在|

注意：这张表主要描述 Spark on YARN 的典型部署方式。其他 Cluster Managers 的具体实现有所不同。

## 7.5 Resource Configuration

你的笔记中还有几个重要配置：

```
--driver-cores 2
--driver-memory 8G

--num-executors 4
--executor-cores 4
--executor-memory 8G
```

假设配置：

```
Driver:
    2 CPU Cores
    8 GB Memory

Executors:
    4 Executors
    Each Executor:
        4 CPU Cores
        8 GB Memory
```

理论上：

\[ \text{Total Executor Cores} =4\times4=16 \]

\[ \text{Total Executor Memory} =4\times8=32GB \]

这里没有包括：

- Driver Memory。
    
- Executor Memory Overhead。
    
- JVM Overhead。
    
- 其他系统保留资源。
    

### Memory Overhead

你记录的：

```
spark.ececutor.memoryOverhead = 0.2
```

有两个问题：

1. `ececutor` 拼写错误，应该是 `executor`。
    
2. `spark.executor.memoryOverhead` 和 `spark.executor.memoryOverheadFactor` 是不同配置。
    

正确示例：

```
--conf spark.executor.memoryOverhead=2g
```

表示指定 Executor 的额外非堆内存预算。

或者：

```
--conf spark.executor.memoryOverheadFactor=0.2
```

表示按照 Executor Heap Memory 的比例计算额外内存预算。

Memory Overhead 用于 JVM Heap 以外的内存开销，例如：

- Native Memory
    
- Python Worker Processes
    
- Off-heap Memory
    
- 其他执行环境开销
    

如果 Executor 出现 OOM，不一定只需要增加 `spark.executor.memory`，也需要检查 Memory Overhead。

## 7.6 完整 Spark Submit 示例

Scala / Java JAR：

```
spark-submit \
  --class com.example.SparkApplication \
  --master yarn \
  --deploy-mode cluster \
  --driver-memory 8g \
  --driver-cores 2 \
  --executor-memory 8g \
  --executor-cores 4 \
  --num-executors 4 \
  --conf spark.executor.memoryOverheadFactor=0.2 \
  application.jar \
  2026-11-08
```

这里：

```
application.jar
```

是 Spark Application。

```
2026-11-08
```

是传递给 Application 的参数。

Python Application：

```
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --executor-memory 8g \
  --executor-cores 4 \
  --num-executors 4 \
  main.py \
  2026-11-08
```

注意：这些是通用 Apache Spark/YARN 示例，不代表这些参数都能原样用于 Databricks Compute。

## 7.7 Spark Submit 在 Databricks 中的特殊注意事项

这里有一个非常重要的更新：

Apache Spark 的 `spark-submit` 和 Databricks 的 Spark Submit Task 不是完全相同的概念。

Databricks 曾经支持在 Lakeflow Jobs 中使用 Spark Submit Task，但这个 Task 类型已经进入弃用阶段。

截至 2026 年 11 月：

- Databricks 已经限制新建 Spark Submit Tasks。
    
- Databricks Runtime 17.3 及以上版本不支持该 Task。
    
- 对于 JVM Workloads，官方建议迁移到 JAR Task。
    
- Python Workloads 可以考虑 Python Script Task 或其他适合的 Task 类型。
    

因此，如果你在荷兰 Data Engineer 面试中被问到如何部署 Spark Application 到 Databricks，建议重点介绍：

- Notebook Task
    
- Python Script Task
    
- Python Wheel Task
    
- JAR Task
    
- Lakeflow Jobs
    

而不是把传统 Spark Submit 当作 Databricks 生产部署的首选。

Databricks on AWS+2

# 8. Spark APIs — SQL, DataFrame, Dataset

你最后记录的内容：

```
Spark give 2 interface to work
- Spark SQL and DataFrame API
- Dataset API (for Scala and Java)
```

方向正确，但需要更完整地理解 Spark API 的层次。

## 8.1 Spark SQL API

通过 SQL 语句操作结构化数据。

例如：

```
df = spark.sql("""    SELECT country, SUM(revenue) AS total_revenue    FROM sales    GROUP BY country""")
```

特点：

- 使用 SQL 语法。
    
- 适合熟悉 SQL 的 Data Engineers。
    
- 可以直接使用 SQL 进行 ETL。
    
- Spark 会对 SQL 进行优化。
    

## 8.2 DataFrame API

通过 Python、Scala、Java 或 R 编写 DataFrame 操作。

例如：

```
from pyspark.sql import functions as Fdf = spark.read.table("bronze.orders")result = (    df.filter(F.col("amount") > 100)      .groupBy("country")      .agg(          F.sum("amount").alias("total_amount")      ))
```

特点：

- 基于列和 Schema 操作。
    
- 支持 Transformation 和 Action。
    
- 可以利用 Catalyst Optimizer。
    
- 非常适合 Data Engineering ETL。
    

## 8.3 Dataset API

Dataset API 主要用于 Scala 和 Java。

它提供：

- Strong Typing
    
- Object-oriented transformations
    
- Compile-time type checking
    

例如 Scala：

```
case class Order(
    orderId: Int,
    amount: Double
)

val orders = Seq(
    Order(1, 100.0),
    Order(2, 200.0)
).toDS()
```

Dataset 可以使用强类型对象进行操作。

### DataFrame vs Dataset

|对比|DataFrame|Dataset|
|---|---|---|
|Schema|有|有|
|Strongly Typed|相对弱|是|
|语言支持|Python、Scala、Java、R|Scala、Java|
|Compile-time Type Safety|较弱|较强|
|Catalyst Optimization|支持|支持|

重要：

在 Scala 中：

```
DataFrame = Dataset[Row]
```

而 PySpark 没有单独暴露 Scala/Java 意义上的强类型 Dataset API。

在 Python 中：

```
df = spark.read.table("orders")
```

返回的就是 DataFrame。

## 8.4 Spark APIs 的整体关系

Apache Spark

Spark SQL

SQL queries

DataFrame / Dataset

Structured APIs

Catalyst Optimizer

Logical Plan → Optimized Logical Plan → Physical Plan

Spark Execution Engine

Tasks → Executors → Distributed Processing

补充：Spark 还有 RDD API，但现代结构化 ETL 开发通常优先使用 DataFrame / SQL API，因为它们可以利用 Spark SQL 的结构化优化能力。

Databricks on AWS+1

## 8.5 面试问题

Q1. What are the main APIs available in Apache Spark?

回答：

The main APIs include Spark SQL, DataFrame API and Dataset API. Spark also provides the lower-level RDD API. For most structured data engineering workloads, I prefer Spark SQL and DataFrame APIs.

Q2. What is the difference between DataFrame and Dataset?

回答：

A DataFrame is a distributed collection of rows with named columns. A Dataset provides stronger compile-time type safety and is available in Scala and Java. In Scala, DataFrame is an alias for Dataset of Row.

Q3. Why would you prefer DataFrame API over RDD API?

回答：

DataFrame API provides schema information and allows Spark's Catalyst Optimizer to optimize execution plans. It generally offers better optimization opportunities for structured data processing.

# 9. 整体知识体系总结

## 9.1 Databricks Compute 分类结构

这是整份笔记最重要的知识框架。

Databricks Compute

Serverless

Databricks-managed

Serverless Notebooks

Serverless Jobs

Serverless SQL

Classic

Customer cloud infrastructure

All-Purpose

Job Compute

Pipeline Compute

Classic Compute 的其他配置维度

Architecture

Single / Multi-node

Access Mode

Standard / Dedicated / Legacy No Isolation Shared

Scaling

Autoscaling

Lifecycle

Auto Termination

注意：Single/Multi-node、Access Mode 和 Autoscaling 是不同的配置维度，不是 Serverless 和 Classic 的下一级产品分类。Serverless 的基础设施管理方式不同，不应直接套用 Classic Cluster 的配置模型。

## 9.2 核心概念对照表

|知识点|核心问题|关键答案|
|---|---|---|
|Serverless vs Classic|谁管理 VM？|Databricks vs Customer|
|All-Purpose vs Job|用于什么工作？|Interactive vs Automated|
|Single vs Multi-node|有多少节点？|One vs Multiple|
|Dedicated vs Standard|如何隔离用户？|Dedicated vs Shared with isolation|
|Auto Termination|什么时候关闭？|Idle duration|
|Autoscaling|需要多少 Workers？|Dynamic worker count|
|DBU|Databricks 如何计量？|Databricks usage unit|
|Spark Submit|如何提交 Application？|`spark-submit`|
|Client vs Cluster|Driver 在哪里？|Client vs Cluster|
|DataFrame vs Dataset|如何操作结构化数据？|Untyped vs Strongly typed|

## 9.3 荷兰 Data Engineer 综合面试题

下面是我建议你重点练习的 8 道英文问题。

Mock interview

0/8 answered

1You have a daily ETL pipeline that runs for 20 minutes. Which Databricks compute would you choose and why?

2A developer leaves an All-Purpose cluster running overnight. How would you prevent unnecessary costs?

3Your ETL workload processes 500 GB of data and requires large joins. Would you choose single-node or multi-node compute?

4Several engineers need to use the same compute resource. How would you configure access mode?

5What is the difference between autoscaling and auto termination?

6What is the difference between Spark client mode and cluster mode?

7Your Spark application runs out of memory. What configurations and performance factors would you investigate?

8How would you optimize Databricks compute costs without significantly affecting ETL performance?

Reveal all answers

## 9.4 官方文档（建议收藏）

|模块|官方文档|
|---|---|
|Compute Overview|[Databricks Compute](https://docs.databricks.com/aws/en/compute?utm_source=chatgpt.com)|
|Classic Compute|[Classic Compute Overview](https://docs.databricks.com/aws/en/compute/use-compute?utm_source=chatgpt.com)|
|All-Purpose / Job Compute|[Compute Selection Recommendations](https://docs.databricks.com/aws/en/compute/choose-compute?utm_source=chatgpt.com)|
|Standard Access Mode|[Standard Compute](https://docs.databricks.com/aws/en/compute/standard-overview?utm_source=chatgpt.com)|
|Dedicated Access Mode|[Dedicated Compute](https://docs.databricks.com/aws/en/compute/dedicated-overview?utm_source=chatgpt.com)|
|Auto Termination|[Manage Classic Compute](https://docs.databricks.com/aws/en/compute/clusters-manage?utm_source=chatgpt.com)|
|Spark Submit|[Apache Spark Submit](https://github.com/apache/spark/blob/master/docs/submitting-applications.md?utm_source=chatgpt.com)|
|Client vs Cluster|[Spark on YARN](https://github.com/apache/spark/blob/master/docs/running-on-yarn.md?utm_source=chatgpt.com)|
|Spark APIs|[Spark SQL, DataFrames and Datasets](https://spark.apache.org/docs/latest/sql-programming-guide?utm_source=chatgpt.com)|

最后的学习重点：

对于荷兰 Junior / Mid-level Data Engineer 岗位，我建议按照以下顺序掌握：

1. Driver、Executor、Worker 的关系。
    
2. All-Purpose Compute vs Job Compute。
    
3. Serverless vs Classic。
    
4. Standard vs Dedicated Access Mode。
    
5. Autoscaling、Auto Termination 和成本优化。
    
6. Spark Memory、Executor Cores 和 Spark UI。
    
7. Spark Submit 的 Client / Cluster Mode。