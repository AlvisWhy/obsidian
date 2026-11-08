# 模块一：数据处理与计算引擎

## 1.1 Dataproc

核心定位：Managed Spark / Hadoop

Google Cloud Dataproc 是 Google Cloud 提供的托管 Hadoop 和 Spark 服务。

它解决的问题是：

当数据量较大，需要使用 Spark、Hadoop、Hive 等大数据计算框架时，不需要自己从零开始部署和维护集群。

### 主要组成

|组件|作用|
|---|---|
|Master Node|管理集群、协调任务|
|Worker Node|执行 Spark/Hadoop 计算|
|Secondary Worker|可选的额外计算节点|
|Dataproc Metastore|管理 Hive Metastore 元数据|
|Persistent History Server|持久化查看 Spark Job 历史记录|

### Dataproc Metastore

你原来的记录：

> centralized data for whole cluster

需要修正。

Dataproc Metastore 并不是集中存储整个集群的数据，而是集中管理 Hive Metastore 元数据。

例如：

- Database 名称
    
- Table 名称
    
- Column Schema
    
- Partition Metadata
    
- 表对应的 GCS 路径
    

假设你有两个 Spark 集群：

```
Dataproc Cluster A
       |
       |
       v
Dataproc Metastore
       ^
       |
       |
Dataproc Cluster B
```

两个集群可以访问同一个 Metastore，因此可以共享表的元数据定义。

注意：Metastore 存储的是元数据，不是实际的 Parquet 文件。

### Persistent History Server

它的作用是保存和查看已经结束的 Spark Job 历史记录。

Spark Event Log 会被保存到 GCS：

```
Spark Job
    |
    v
Spark Event Log
    |
    v
GCS
    |
    v
Persistent History Server
    |
    v
Spark History UI
```

这里需要区分：

|服务|解决的问题|
|---|---|
|Dataproc Metastore|不同集群共享 Hive 表元数据|
|Persistent History Server|查看 Spark 历史执行记录|
|GCS|持久化保存 Event Logs 和数据|

补充：Serverless Dataproc

Dataproc Serverless for Apache Spark 不需要用户自己管理固定的 Spark 集群。

但它并不意味着必须使用 Dataproc Metastore 或 Persistent History Server。

- Metastore：需要共享 Hive 元数据时使用。
    
- Spark Event Logging：需要配置事件日志存储位置。
    
- Persistent History Server：需要长期查看 Spark 执行历史时使用。
    

### Spark Event Log 配置示例

```
gcloud dataproc clusters create my-cluster \
  --region=us-central1 \
  --properties="spark:spark.eventLog.enabled=true,spark:spark.eventLog.dir=gs://my-bucket/spark-events"
```

注意这里是 `--properties`，不是 `--properities`。

### 面试可能问到的问题

Q1：Dataproc Metastore 和 Persistent History Server 有什么区别？

回答：

Dataproc Metastore 用于集中管理 Hive 表的元数据，使不同 Spark 集群能够共享数据库和表的定义。Persistent History Server 则用于持久化查看已经结束的 Spark Jobs 的执行历史。前者管理数据目录，后者管理执行历史。

Q2：Dataproc 和 Dataflow 有什么区别？

- Dataproc：适合 Spark、Hadoop 等已有大数据框架和代码。
    
- Dataflow：基于 Apache Beam，适合构建统一的 Batch 和 Streaming Pipeline。
    
- Dataproc 更强调 Spark/Hadoop 生态兼容性。
    
- Dataflow 更强调托管式数据处理和自动扩缩容。
    

Q3：为什么要把 Spark Event Log 存储到 GCS？

因为 Dataproc 集群可能被删除，集群本地的历史日志也可能随之消失。GCS 提供持久化存储，可以配合 History Server 查看历史任务。

## 1.2 Dataflow

核心定位：Serverless Batch / Streaming ETL

Google Cloud Dataflow 是 Google Cloud 基于 Apache Beam 的托管数据处理服务。

你原来写的：

> transformation tool, ETL flow, serverless dataproc

其中 Serverless Dataproc 这个说法不准确。

Dataflow 和 Dataproc 是两种不同的计算服务。

### Dataflow 的核心架构

```
       Data Sources
             |
      +------+------+
      |             |
    Pub/Sub        GCS
      |             |
      +------+------+
             |
             v
         Dataflow
       Apache Beam
             |
       Transformations
             |
      +------+------+
      |             |
   BigQuery       GCS
```

Dataflow 主要负责：

1. Read：读取数据。
    
2. Transform：数据清洗、转换、过滤、聚合。
    
3. Write：将处理后的数据写入目标系统。
    

### ETL 和 ELT

这是面试非常常见的问题。

ETL：Extract → Transform → Load

```
Source
  |
  v
Extract
  |
  v
Dataflow Transform
  |
  v
BigQuery
```

先处理数据，再加载到 BigQuery。

适合：

- 复杂数据清洗。
    
- 实时数据转换。
    
- 需要外部系统处理的逻辑。
    

ELT：Extract → Load → Transform

```
Source
  |
  v
BigQuery Raw Table
  |
  v
BigQuery SQL
  |
  v
Cleaned Table
```

先将原始数据加载到 BigQuery，然后使用 BigQuery SQL 进行转换。

适合：

- 数据量大但转换主要是 SQL。
    
- 希望充分利用 BigQuery 的计算能力。
    
- 希望保留原始数据供后续使用。
    

### Batch 和 Streaming

Dataflow 同时支持两种模式：

|Batch|Streaming|
|---|---|
|处理已经存在的数据|持续处理新进入的数据|
|有限数据集|无界数据流|
|可以批量执行|长时间运行|
|适合历史数据 ETL|适合实时事件处理|

### 面试可能问到的问题

Q1：什么情况下选择 Dataflow 而不是 Dataproc？

如果项目使用 Apache Beam，需要同时支持 Batch 和 Streaming，或者希望使用完全托管的自动扩缩容计算环境，通常选择 Dataflow。

如果已有 Spark 作业，或者依赖 Hadoop/Spark 生态，通常选择 Dataproc。

Q2：Dataflow 如何处理 Late-arriving Data？

回答思路：

- Event Time：事件实际发生时间。
    
- Processing Time：系统处理事件的时间。
    
- Window：定义事件聚合的时间范围。
    
- Watermark：估计系统已经处理到哪个事件时间。
    
- Allowed Lateness：允许迟到数据进入已经计算过的窗口。
    
- Trigger：决定什么时候输出计算结果。
    

Q3：Dataflow 如何避免重复数据？

可以通过：

- 使用业务唯一 ID。
    
- 去重逻辑。
    
- Beam 的状态和 Timer。
    
- 结合 Sink 的幂等写入能力。
    

需要注意，Dataflow 的 Exactly-once 处理能力不意味着所有外部 Sink 自动具备 Exactly-once 语义。

## 1.3 Datastream

核心定位：CDC / Database Replication

Google Cloud Datastream 用于从数据库中捕获数据变化。

它最重要的概念是：

CDC（Change Data Capture）

CDC 通常通过读取数据库的 Transaction Log 捕获数据变化，而不是不断扫描整个数据库。

例如 MySQL：

```
MySQL
   |
   | Binary Log
   |
   v
Datastream
   |
   v
Change Events
   |
   +------------------+
   |                  |
   v                  v
BigQuery             GCS
```

支持的数据库类型取决于 Datastream 当前的 Source Connector，常见场景包括 MySQL、PostgreSQL 和 Oracle 等。

### CDC 包含什么？

假设原始数据库：

|ID|Name|Salary|
|---|---|---|
|1|Alice|5000|

之后执行：

```
UPDATE employees
SET salary = 6000
WHERE id = 1;
```

CDC 会捕获类似这样的变化：

```
{
  "op": "UPDATE",
  "id": 1,
  "before": {
    "salary": 5000
  },
  "after": {
    "salary": 6000
  }
}
```

常见变化类型：

|操作|含义|
|---|---|
|INSERT|新增记录|
|UPDATE|更新记录|
|DELETE|删除记录|

### Datastream + Dataflow + BigQuery

你原来的架构是正确的，但需要明确：Dataflow 是可选的中间处理层，不是 Datastream 必须依赖的组件。

```
MySQL / PostgreSQL
         |
         v
      Datastream
         |
         v
     CDC Events
         |
         v
      Dataflow
         |
   +-----+------+
   |     |      |
   v     v      v
INSERT UPDATE DELETE
   |
   v
Deduplication
   |
   v
Ordering
   |
   v
Transformation
   |
   v
SCD Type 1 / 2
   |
   v
  BigQuery
```

需要注意：

Datastream 可以直接将数据复制到 BigQuery 或 GCS。是否需要 Dataflow，取决于你是否需要额外的业务转换。

### SCD Type 1 和 Type 2

这是数据工程面试高频知识。

SCD Type 1：覆盖历史数据

```
Before:
Alice, Salary=5000

After:
Alice, Salary=6000
```

不保留旧值。

SCD Type 2：保留历史数据

|ID|Salary|Start Date|End Date|Is Current|
|---|---|---|---|---|
|1|5000|Jan 1|Feb 1|False|
|1|6000|Feb 1|NULL|True|

每次数据变化都会保留历史版本。

### Soft Delete 和 Hard Delete

你写的：

> soft delete: bad for dataflow it will store all deleted data

这个逻辑需要修正。

Soft Delete 本身并不是 Dataflow 的问题。

- Hard Delete：真正删除数据库中的记录。
    
- Soft Delete：通过 `is_deleted=true` 或 `deleted_at` 标记删除。
    

例如：

```
id=1, name=Alice, is_deleted=true
```

在 CDC Pipeline 中：

- Hard Delete 需要正确处理 DELETE Event。
    
- Soft Delete 通常表现为 UPDATE Event。
    
- 如果不处理删除事件，目标端可能保留已经不应该存在的数据。
    
- 如果业务要求保留历史删除记录，则可以把删除事件写入历史表。
    

因此，关键不是 Soft Delete 好不好，而是目标端如何实现删除语义。

### 面试可能问到的问题

Q1：Datastream 和 Dataflow 有什么区别？

Datastream 负责捕获数据库变化，Dataflow 负责数据处理。Datastream 是 CDC 服务，Dataflow 是通用计算引擎。

Q2：为什么 CDC 比定期全表扫描更高效？

CDC 读取 Transaction Log，只处理发生变化的记录，避免每次扫描整个数据库。

Q3：如何保证 CDC 数据不会乱序或者重复？

需要考虑：

- Transaction Log Position / Offset。
    
- Primary Key。
    
- Event Timestamp。
    
- Change Sequence。
    
- Deduplication。
    
- 幂等写入。
    
- 目标端的事务和更新策略。
    

Q4：Datastream 可以直接替代 Dataflow 吗？

不能完全替代。Datastream 主要负责 CDC，复杂数据转换、聚合、业务逻辑仍然可以使用 Dataflow。

# 模块二：数据安全与隐私保护

## 2.1 Sensitive Data Protection（原 Cloud DLP）

核心定位：PII Detection & Protection

Sensitive Data Protection 是原来的 Cloud Data Loss Prevention（Cloud DLP）。

主要解决：

- 数据中是否包含敏感信息？
    
- 敏感信息存储在哪里？
    
- 如何在不影响业务使用的情况下保护敏感数据？
    
- 如何降低数据泄露和身份重新识别的风险？
    

例如：

```
Customer Data
     |
     v
Sensitive Data Protection
     |
     +----------------------+
     |                      |
     v                      v
  Inspection             Discovery
     |                      |
     v                      v
PII Detection          Data Inventory
     |
     v
De-identification
     |
     v
Protected Data
```

### 四个核心概念

你原来的逻辑基本正确，但 Inspection 和 Discovery 需要明确区分。

|功能|主要职责|举例|
|---|---|---|
|Inspection|检测具体数据中的敏感信息|找到邮箱、手机号、信用卡号|
|Discovery|持续发现和盘点敏感数据资产|发现哪些 BigQuery 表包含 PII|
|De-identification|转换或隐藏敏感信息|Masking、Tokenization|
|Risk Analysis|分析重新识别个人的风险|K-anonymity、L-diversity|

此外，还有 Re-identification：

在受控的场景下，将某些可逆脱敏数据恢复成原始值，例如使用对应的密钥还原加密数据。

### De-identification 方法

|方法|作用|示例|可逆性|
|---|---|---|---|
|Redaction|删除敏感内容|Alice → 空值|不可逆|
|Masking|使用字符遮盖|12345678 → ****5678|通常不可逆|
|Tokenization|使用 Token 替代原值|Alice → TOKEN_8f2a|取决于 Token 映射机制|
|Hashing|哈希转换|ID → Hash(ID)|通常不可逆|
|Encryption|加密|Alice → Ciphertext|有密钥时可逆|
|Bucketing|转换成范围|27 → 20–30|信息损失|
|Date shifting|日期偏移|2026-01-01 → 2026-01-15|取决于偏移量是否保留|

需要特别区分：

- Hashing 不等于 Encryption。
    
- Hashing 通常不能直接还原原始值。
    
- Encryption 可以使用密钥解密。
    
- Tokenization 需要区分随机 Token 和可逆映射 Token。
    

### De-identification Template

你记录的 De-identification Template 很重要。

可以预先保存一组转换规则，例如：

```
Template: customer_protection

Email:
    Masking

Phone:
    Redaction

Customer ID:
    HMAC

Date of Birth:
    Date Shifting
```

后续不同的 DLP 操作可以复用这些设置，而不用重复定义所有转换规则。

### Confidence Level：Likelihood

DLP 检测到一个 Finding 后，会提供 `likelihood`，用于描述该匹配结果有多大可能是真正的敏感信息。

|Likelihood|含义|判断|
|---|---|---|
|VERY_UNLIKELY|极不可能|匹配信号很弱|
|UNLIKELY|不太可能|匹配信号较弱|
|POSSIBLE|可能|默认最低阈值|
|LIKELY|很可能|匹配信号较强|
|VERY_LIKELY|极有可能|匹配信号很强|

例如：

```
1234567890
```

仅仅是一个数字，可能不一定是电话号码。

但如果同时满足：

- 电话号码格式。
    
- 附近出现 Phone Number 字段名称。
    
- 符合国家区号规则。
    

Likelihood 就可能更高。

注意：Likelihood 不是数据敏感等级，也不是精确的统计准确率百分比。

### 面试可能问到的问题

Q1：Inspection 和 Discovery 有什么区别？

Inspection 是检查具体数据内容，Discovery 是持续发现和盘点数据资产中的敏感信息。

Q2：Hashing 和 Encryption 有什么区别？

Hashing 通常是单向转换，Encryption 是可逆转换，需要密钥才能恢复原始值。

Q3：如果需要保留客户 ID 的关联分析能力，但又不希望暴露原始 ID，应该怎么做？

可以使用稳定的 HMAC Tokenization。

相同输入使用相同密钥和算法生成相同 Token，这样可以在不同数据集中进行关联，同时不直接暴露原始 ID。

Q4：如何降低重新识别用户的风险？

可以使用：

- Generalization / Bucketing。
    
- Suppression。
    
- K-anonymity。
    
- L-diversity。
    
- De-identification。
    

## 2.2 BigQuery IAM 权限

核心定位：Access Control

你记录的 `BigQuery User` 和 `BigQuery Job User` 是很重要的考试知识点。

先记住一个核心原则：

BigQuery 权限可以作用在不同资源层级，授予权限的资源层级决定了该权限的作用范围。

### BigQuery User

Role：

```
roles/bigquery.user
```

在 Project 层级：

- 创建 BigQuery Jobs。
    
- 执行 Query。
    
- 枚举自己的 Jobs。
    
- 取消自己的 Jobs。
    
- 枚举项目中的 Datasets。
    
- 创建新的 Datasets。
    

创建 Dataset 的用户会获得该新 Dataset 的 BigQuery Data Owner 权限。

在 Dataset 层级，该角色主要提供 Dataset 元数据读取、表枚举等能力。

### BigQuery Job User

Role：

```
roles/bigquery.jobUser
```

主要提供：

- 创建 BigQuery Jobs。
    
- 执行 Query。
    
- 运行其他 BigQuery Jobs。
    

通常授予 Project 层级。

### BigQuery Data Viewer

```
roles/bigquery.dataViewer
```

用于读取 Dataset 或 Table 中的数据，具体范围取决于授权层级。

### 常见角色比较

|Role|主要职责|
|---|---|
|BigQuery User|创建 Dataset + 执行 Jobs + 相关元数据访问|
|BigQuery Job User|创建和运行 Jobs|
|BigQuery Data Viewer|读取数据|
|BigQuery Data Editor|创建、修改和删除数据对象|
|BigQuery Data Owner|数据集级别的管理权限|
|BigQuery Admin|管理 BigQuery 资源|

### 典型考试场景

假设：

- Alice 需要运行 SQL。
    
- Alice 只能读取某个 Dataset。
    
- Alice 不允许创建或修改表。
    

可以考虑：

```
Project:
    BigQuery Job User

Dataset:
    BigQuery Data Viewer
```

这就是典型的 Least Privilege（最小权限原则）。

### 面试可能问到的问题

Q1：BigQuery Job User 和 BigQuery User 的区别？

Job User 主要用于创建和执行 Jobs；BigQuery User 还提供创建 Dataset 等能力。

Q2：用户可以运行 SQL，但无法读取表，可能是什么原因？

用户可能有 Job User 权限，但缺少目标 Dataset 或 Table 的数据读取权限。

Q3：什么是 Least Privilege？

只给用户完成工作所需的最低权限，避免授予不必要的管理权限。

## 2.3 BigQuery Row-level / Column-level Security

BigQuery 的数据安全可以分为三个层次：

|机制|控制什么|主要机制|
|---|---|---|
|Dataset/Table IAM|能否访问数据集或表|IAM|
|Row-level Security|能看到哪些行|Row Access Policy|
|Column-level Security|能读取哪些列|Policy Tags|
|Dynamic Data Masking|如何展示敏感字段|Data Masking Policy|

### Row-level Security

例如 HR 表：

|Employee|Department|Salary|
|---|---|---|
|Alice|HR|5000|
|Bob|IT|6000|
|Charlie|Finance|7000|

HR 部门只能看到 HR 员工。

可以通过 Row Access Policy：

```
CREATE ROW ACCESS POLICY hr_access
ON `project.dataset.employees`
GRANT TO ('group:hr@example.com')
FILTER USING (department = 'HR');
```

如果某个用户属于 HR Group，那么查询时只能看到符合条件的记录。

你记录的 `SESSION_USER()` 也可以用于动态判断当前用户。

例如：

```
CREATE ROW ACCESS POLICY user_access
ON `project.dataset.employees`
GRANT TO ('allAuthenticatedUsers')
FILTER USING (
  email = SESSION_USER()
);
```

注意：实际使用时，需要确保授权对象、用户身份以及 Policy 的设计符合预期；不要直接将 `allAuthenticatedUsers` 用于敏感数据生产环境。

### Column-level Security

通过 Policy Tags 管理列级权限。

例如：

```
employees
   |
   +-- employee_id
   |
   +-- name
   |
   +-- salary
   |
   +-- social_security_number
```

可以将：

- salary
    
- social_security_number
    

标记为敏感字段。

只有具备对应 Policy Tag Fine-Grained Reader 权限的用户才能读取这些字段。

### Dynamic Data Masking

即使用户有权限查询某个字段，也可以对展示结果进行遮罩。

例如：

```
Original:
123456789

Masked:
XXXXXX789
```

需要注意：

- Column-level Security：控制是否有权读取字段。
    
- Data Masking：控制字段以什么形式展示。
    

### 面试可能问到的问题

Q1：Row-level Security 和 Column-level Security 有什么区别？

Row-level Security 控制用户可以看到哪些记录，Column-level Security 控制用户可以读取哪些字段。

Q2：Authorized View 是否可以绕过底层表的安全策略？

不能简单认为可以绕过。底层表的 Row-level Security 和 Column-level Security 仍然会影响查询结果。

# 模块三：数据存储与数据库选择

这是 Google Data Engineer 面试中非常重要的模块。

不要只背容量和产品名称，最重要的是理解：

不同数据库是为不同的数据访问模式设计的。

## 3.1 Firestore / Datastore

核心定位：NoSQL Document Database

Cloud Firestore 是 Google Cloud 的文档型 NoSQL 数据库。

你记录的：

> just like MongoDB

这个理解大体正确，但不能认为 Firestore 和 MongoDB 是同一个产品。

### 核心数据结构

```
Collection
    |
    +-- Document 1
    |       |
    |       +-- Field
    |       +-- Field
    |
    +-- Document 2
            |
            +-- Field
            +-- Field
```

例如：

```
{
  "customer_id": 1001,
  "name": "Alice",
  "address": {
    "city": "Amsterdam",
    "country": "Netherlands"
  },
  "orders": [101, 102, 103]
}
```

### Firestore 和 Datastore

Firestore 有两种主要模式：

- Firestore Native Mode：面向现代文档数据库应用。
    
- Firestore in Datastore Mode：兼容 Datastore 数据模型和 API。
    

Datastore Mode 中常见的概念：

|概念|作用|
|---|---|
|Kind|类似实体类型|
|Entity|数据实体|
|Key|唯一定位 Entity|
|Property|Entity 的字段|
|Index|支持查询|

Entity Key 不只是 JSON，它包含用于定位实体的路径信息，例如 Kind、ID 或 Name。

### Indexes

Firestore/Datastore 查询需要相应索引支持。

例如：

```
indexes:
  - kind: Movie
    properties:
      - name: actors
      - name: date

  - kind: Movie
    properties:
      - name: type
      - name: date
```

上面的索引分别支持不同的查询模式。

假设你有：

```
WHERE actors = 'Alice'
ORDER BY date
```

以及：

```
WHERE type = 'Action'
ORDER BY date
```

这两种查询需要的复合索引不一样。

为什么不能随意创建索引？

因为索引越多：

- 存储开销越大。
    
- 写入操作需要维护更多索引。
    
- 索引构建和维护成本增加。
    
- 存在索引数量和索引配置限制。
    

但这里不是因为多个索引会自动导致 SQL Cross Join。

Firestore 的索引设计和关系型数据库的 Join 优化是两个不同概念。

### 面试可能问到的问题

Q1：Firestore 和 Cloud SQL 有什么区别？

Firestore 是 NoSQL 文档数据库，适合灵活 Schema 和文档型访问；Cloud SQL 是关系型数据库，适合 SQL、Join 和事务型关系数据。

Q2：为什么需要 Composite Index？

当查询涉及多个字段的过滤和排序时，可能需要复合索引支持。

Q3：为什么不能创建过多索引？

增加存储、写入维护开销和索引管理复杂度。

## 3.2 Cloud SQL

核心定位：Managed Relational Database

Cloud SQL 是 Google Cloud 提供的托管关系型数据库服务。

主要支持：

- MySQL
    
- PostgreSQL
    
- SQL Server
    

适合传统 OLTP 应用，例如：

- 用户账户。
    
- 订单系统。
    
- 业务交易。
    
- 需要 SQL 和关系约束的系统。
    

### Storage 和 CPU 扩展

你的记录需要补充：

|资源|扩展方式|
|---|---|
|Storage|可以开启 Automatic Storage Increase|
|CPU|通过修改 Instance Machine Type 手动调整|
|Memory|通常随机器规格调整|
|Read Capacity|可以使用 Read Replicas 分担读取负载|

注意：Cloud SQL 的最大存储容量取决于数据库引擎、实例版本和配置，不应该把 64 TB 当作所有 Cloud SQL 实例通用的固定上限。

### Partition 和 Sharding

这是两个不同概念。

Partitioning

把同一张表的数据按某个字段分成多个逻辑分区。

例如：

```
orders
  |
  +-- partition_2026_01
  +-- partition_2026_02
  +-- partition_2026_03
```

Sharding

将不同数据分布到不同数据库实例。

例如：

```
Shard 1: Customer ID 1-1000
Shard 2: Customer ID 1001-2000
Shard 3: Customer ID 2001-3000
```

Cloud SQL 不提供原生自动水平 Sharding 服务。

### Point-in-Time Recovery（PITR）

PITR 可以将数据库恢复到过去某个时间点。

例如：

```
10:00 Database正常
10:30 删除了重要数据
11:00 发现问题
```

可以恢复到 10:29 左右的状态，而不是只能恢复到昨天的 Backup。

你原来的记录：

> For SQL, binary should be set True for Point in Time Recovery

需要补充：

- MySQL PITR 需要相应的 Binary Logging 配置。
    
- PostgreSQL PITR 依赖 WAL 日志和备份配置。
    
- SQL Server 使用其对应的备份和事务日志机制。
    

不能把 MySQL 的 Binary Logging 要求直接套用到所有 Cloud SQL 引擎。

### HA 和 Read Replica

这是高频考试考点。

![Best practices for setting up Cloud SQL for MySQL Instance | Google Cloud](https://images.openai.com/static-rsc-4/1gZZUEZKZHnY9kte214sGwICvx_vYUzscngqNXIJ-yhxNFRYFkFeACIgOWiMAzCdNJ7BG1dI9Z1yWeE8YraqgCl4VEs6RFKwPcUhtxOHL-eZQtz8kGnGsITnBrC8_JV6Os2QTP6tkSzFom8lFxKUq22z2tlcz6RFkqkL1FR8gJI?purpose=inline)

[cloud.google.com](https://cloud.google.com/mysql/cloudsql-setup)

High Availability（HA）

通常是在同一个 Region 内使用不同 Zone 的 Primary 和 Standby。

```
Region: us-central1

      Primary
         |
     Replication
         |
      Standby
```

发生符合条件的基础设施故障时，可以自动 Failover。

特点：

- 主要解决高可用。
    
- 自动 Failover。
    
- 不等于跨 Region 灾难恢复。
    

Cross-region Read Replica

```
Region A
  Primary
     |
     | Async Replication
     |
     v
Region B
  Read Replica
```

主要解决：

- Disaster Recovery。
    
- 跨区域读取。
    
- 降低主实例读取压力。
    

如果 Primary Region 发生灾难，通常需要手动 Promote Read Replica。

Cascading Read Replica

```
Primary
   |
   v
Read Replica A
   |
   v
Read Replica B
```

允许 Replica 继续为下游 Replica 提供复制数据。

是否支持以及具体限制需要看数据库引擎和 Cloud SQL 版本。

### On-premises 到 Cloud SQL：最小停机迁移

你记录的思路是正确的。

```
Step 1:
On-prem MySQL
Enable Binary Logging

Step 2:
Initial Full Migration
On-prem --> Cloud SQL

Step 3:
Incremental Replication
Binlog --> Cloud SQL

Step 4:
Data Consistency Check

Step 5:
Application Cutover
```

核心思想：

Initial Load + Continuous Replication + Cutover

先复制全部历史数据，再通过 Transaction Log 追赶增量变化。

这样不需要在整个迁移期间停止源数据库的写入。

Google Cloud Database Migration Service 也可以帮助执行这类迁移。

### 面试可能问到的问题

Q1：HA 和 Cross-region Read Replica 有什么区别？

HA 主要用于同一区域内的高可用，通常可以自动 Failover；跨区域 Replica 主要用于灾难恢复和读取扩展，发生故障时通常需要手动 Promote。

Q2：如何在尽量不停止业务的情况下迁移数据库？

先进行 Initial Load，然后持续复制 Binlog/WAL 等变化日志，待数据追平后进行最终切换。

Q3：Cloud SQL 和 Spanner 怎么选择？

Cloud SQL 适合传统关系型数据库和中小规模应用；Spanner 适合需要水平扩展、全球分布和强一致事务的超大规模关系型应用。

## 3.3 AlloyDB

核心定位：High-performance PostgreSQL-compatible Database

AlloyDB for PostgreSQL 是 Google Cloud 提供的 PostgreSQL 兼容数据库服务。

你记录的：

> advance postgre（max 128TB），both OLAP and OLTP

建议调整成：

AlloyDB 主要面向高性能 PostgreSQL OLTP 工作负载，同时通过 Columnar Engine 等能力支持分析型查询。

### 为什么需要 AlloyDB？

传统 PostgreSQL 在高并发和大规模场景中可能遇到：

- CPU 压力。
    
- 存储 I/O 瓶颈。
    
- 复杂分析查询影响事务性能。
    
- 读写扩展困难。
    

AlloyDB 通过优化存储、计算架构以及分析能力改善这些问题。

### Cloud SQL vs AlloyDB vs BigQuery

|产品|核心用途|工作负载|
|---|---|---|
|Cloud SQL|托管关系型数据库|OLTP|
|AlloyDB|高性能 PostgreSQL|OLTP + 分析增强|
|BigQuery|数据仓库|OLAP|

### 面试可能问到的问题

Q1：AlloyDB 和 Cloud SQL PostgreSQL 有什么区别？

两者都支持 PostgreSQL 生态，但 AlloyDB 针对高性能、可扩展的 PostgreSQL 工作负载进行了进一步优化，并提供 Columnar Engine 等分析能力。

Q2：AlloyDB 能否完全替代 BigQuery？

不能。AlloyDB 仍然主要是数据库服务，而 BigQuery 是面向大规模分析的 Serverless Data Warehouse。

## 3.4 Cloud Spanner

核心定位：Globally Distributed Relational Database

Cloud Spanner 是 Google Cloud 的分布式关系型数据库。

它最重要的特点：

- 水平扩展。
    
- 强一致性。
    
- 分布式事务。
    
- 高可用。
    
- 全球多区域部署。
    

### CAP Theorem

CAP 包含：

- Consistency：一致性。
    
- Availability：可用性。
    
- Partition Tolerance：分区容错性。
    

在发生网络分区时，分布式系统需要在一致性和可用性之间做取舍。

你原来的记录：

> We can use sync replica and async replica in other region (not transaction because not guarantee consistency)

这个表达不够准确。

需要区分：

1. 同步复制和异步复制。
    
2. CAP。
    
3. 数据库事务。
    
4. 跨区域一致性。
    

异步复制的 Replica 可能存在复制延迟，因此不能保证立即读到最新写入的数据。

但这并不代表异步复制数据库完全不能执行事务。

Spanner 的一个重要特性是通过分布式同步复制和 TrueTime 提供 External Consistency（外部一致性）。

### Table Interleaving

你原来的记录：

> table interleaving, for designated join (save joined data in same disk)

这个方向正确，但需要更准确。

Interleaved Table 的目的是将父表和子表按照主键关系组织，使相关数据在存储上具有更好的 locality。

例如：

```
Customers
   |
   +-- Customer ID = 1001
           |
           +-- Orders 1
           +-- Orders 2
           +-- Orders 3
```

可以减少某些父子数据访问时的分布式读取开销。

但它不是预先将 Join 结果保存成一张物理表。

另外，现代 Spanner 设计中需要理解主键前缀和数据 locality；不要把 Interleaving 简单理解成普通 SQL Join 优化。

### Key Visualizer

Spanner Key Visualizer 可以帮助分析：

- Key Range 热点。
    
- 读写负载。
    
- 数据访问分布。
    
- 潜在的热点区域。
    

### 面试可能问到的问题

Q1：Spanner 为什么可以实现全球强一致性？

通过分布式同步复制、分布式事务机制以及 TrueTime 等技术提供强一致性和 External Consistency。

Q2：什么是 Hotspot？

当大量读写集中在少数 Key Range 时，部分节点负载过高，导致整体性能下降。

Q3：为什么 Spanner 的 Primary Key 设计很重要？

因为数据分布和访问 locality 与 Key 设计密切相关。不合理的顺序 Key 可能导致写入集中。

## 3.5 Bigtable

核心定位：Wide-column NoSQL Database

Cloud Bigtable 是 Google Cloud 的高吞吐、低延迟 NoSQL 数据库。

典型应用：

- IoT。
    
- Time-series Data。
    
- 用户行为事件。
    
- 游戏实时数据。
    
- 大规模 Key-value 查询。
    

### Bigtable 数据结构

Bigtable 并不是传统意义上的无限列 SQL 表。

它采用 Wide-column 数据模型：

```
Row Key
   |
   +-- Column Family A
   |       |
   |       +-- qualifier_1
   |       +-- qualifier_2
   |
   +-- Column Family B
           |
           +-- qualifier_1
           +-- qualifier_2
```

三个核心概念：

|概念|作用|
|---|---|
|Row Key|唯一标识一行，决定排序和数据分布|
|Column Family|组织相关列，创建时定义|
|Column Qualifier|具体字段名称|

例如：

```
Row Key: user_1001

profile:name = Alice
profile:city = Amsterdam

activity:last_login = 2026-11-08
activity:score = 100
```

### Row Key 设计

这是 Bigtable 最重要的考点之一。

Bigtable 的数据按照 Row Key 字典序排序，并分布在不同 Tablet 中。

如果直接使用递增时间戳：

```
202611080001
202611080002
202611080003
202611080004
```

新数据可能持续集中写入某个 Key Range，产生 Hotspot。

更好的方式是根据查询模式设计 Row Key。

例如：

```
user_1001#20261108
user_1002#20261108
user_1003#20261108
```

如果查询模式是按照用户查找时间序列数据，这种设计可能比较合理。

对于高吞吐场景，可以加入 Hash Prefix 或 Shard ID：

```
03#20261108#user_1001
07#20261108#user_1002
01#20261108#user_1003
```

但需要注意：

加入随机前缀虽然能分散写入，却可能使按照用户查询时需要扫描多个分片。

因此：

Row Key 设计必须平衡写入分布和查询效率。

### 查询最新数据

你记录的：

```
MAX_TIMESTAMP - event_timestamp
```

是一种倒序时间设计思路。

例如：

```
Row Key = user_id#(MAX_TIMESTAMP - event_timestamp)
```

因为 Bigtable 按字典序排序，经过合理编码后可以让最新事件排在前面。

注意 Timestamp 需要固定长度、正确编码，否则字符串排序不一定等于时间排序。

### Hot Key 和 Hotspot

这两个概念相关，但可以区分：

- Hot Key：某个 Key 被非常频繁地访问。
    
- Hotspot：某个 Key Range 或 Tablet 负载过高。
    

常见原因：

- 单个用户请求量极高。
    
- 使用递增时间戳作为 Row Key 前缀。
    
- 写入集中到相邻 Key Range。
    

### Key Visualizer

Bigtable Key Visualizer 用于分析 Key Range 的读写负载，帮助发现热点。

![Usar o Key Visualizer  |  Bigtable  |  Google Cloud Documentation](https://images.openai.com/static-rsc-4/tltL4MEGDV7f7kLScNch0dpHRvGuTtPmvdIcv9yh3b-GgR1IVT1Vrh9CF2TgNgRy9K0YO0D286vp3_75wgtUtzXM7u30V1IXnBioaCEaKmajmb9pDmzMri9UIVvsybF6y_5SbZpYnNV9Vz-Io6n2CcJTscpeUl8U7qHSHo7Vyoo?purpose=inline)

[docs.cloud.google.com](https://docs.cloud.google.com/bigtable/docs/keyvis-getting-started?hl=pt-br)

### App Profile

你记录的 App Profile 是非常重要的知识点。

它决定应用如何访问 Bigtable。

主要涉及：

1. Routing Policy。
    
2. Request Priority。
    

假设：

```
Bigtable Instance
    |
    +-- cluster-us
    |
    +-- cluster-eu
```

Single-cluster Routing

指定一个固定 Cluster：

```
Export Profile
      |
      v
cluster-us
```

适合：

- 固定访问某个 Cluster。
    
- 隔离特定 workload。
    
- 需要在指定 Cluster 上保持 Read-your-writes Consistency 的场景。
    

Multi-cluster Routing

让 Bigtable 根据配置和集群状态选择可用 Cluster。

```
Application
     |
     v
Multi-cluster Routing
     |
   +---+---+
   |       |
   v       v
cluster-us cluster-eu
```

适合高可用和跨区域访问。

但要注意，Multi-cluster Routing 并不意味着可以无条件保证跨集群的立即 Read-your-writes Consistency。它需要结合 Bigtable 的一致性语义理解。

### Request Priority

Bigtable 支持不同请求优先级：

|Priority|用途|
|---|---|
|HIGH|线上关键业务|
|MEDIUM|普通业务|
|LOW|后台任务、数据导出|

例如：

```
Game Profile
    Routing: Multi-cluster
    Priority: HIGH

Export Profile
    Routing: Single-cluster
    Priority: LOW

Backend Profile
    Routing: Multi-cluster
    Priority: MEDIUM
```

这样可以对不同 Workload 进行访问策略隔离。

### 面试可能问到的问题

Q1：为什么 Bigtable 不建议使用递增 Timestamp 作为 Row Key？

因为新数据可能集中写入连续的 Key Range，导致 Hotspot。

Q2：如何解决 Bigtable Hotspot？

通过合理设计 Row Key，例如加入 Hash Prefix、Shard ID，或重新组织业务实体和时间字段。

Q3：Column Family 和 Column Qualifier 有什么区别？

Column Family 是较大的逻辑分类，在表 Schema 中预先定义；Qualifier 是具体的列标识，可以动态出现。

Q4：Single-cluster 和 Multi-cluster Routing 有什么区别？

Single-cluster 将请求路由到指定 Cluster；Multi-cluster 允许系统在配置的多个 Cluster 之间进行路由，适合高可用和多区域访问。

Q5：Bigtable 和 BigQuery 有什么区别？

Bigtable 面向低延迟 Key-value/Wide-column 访问；BigQuery 面向大规模 SQL 分析。

# 模块四：BigQuery 数据仓库核心知识

## 4.1 BigQuery 基本架构

核心定位：Serverless Data Warehouse

BigQuery 是 Google Cloud 的 Serverless Data Warehouse。

最重要的架构概念：

```
          BigQuery
              |
       +------+------+
       |             |
       v             v
   Storage        Compute
       |             |
  Columnar       Slots
  Storage        Execution
```

BigQuery 的 Storage 和 Compute 在架构上分离。

### 为什么需要分离？

因为：

- 数据存储和计算可以独立扩展。
    
- 不需要自己管理数据库服务器。
    
- 多个用户可以共享数据存储。
    
- 可以按需购买计算资源。
    

### View 和 Materialized View

Logical View

```
CREATE VIEW sales_view AS
SELECT *
FROM sales;
```

只保存 SQL 查询逻辑，不单独保存查询结果。

查询 View 时，BigQuery 会执行其底层 SQL。

Materialized View

```
CREATE MATERIALIZED VIEW `project.dataset.sales_mv`
OPTIONS (
  enable_refresh = TRUE,
  refresh_interval_minutes = 30
)
AS
SELECT
  customer_id,
  SUM(amount) AS total_amount
FROM `project.dataset.sales`
GROUP BY customer_id;
```

Materialized View 会存储预计算结果，并由 BigQuery 管理刷新。

|对比|Logical View|Materialized View|
|---|---|---|
|保存 SQL|是|是|
|保存预计算结果|否|是|
|查询时计算|通常需要|可利用预计算结果|
|刷新|查询时执行|后台自动刷新，也可按需刷新|
|典型用途|逻辑封装、权限控制|加速重复分析查询|

注意：Materialized View 有支持的 SQL 表达式和查询模式限制，并不是任意复杂 SQL 都能创建。

### 面试可能问到的问题

Q1：BigQuery 为什么可以实现 Storage 和 Compute Separation？

存储层和计算层独立管理，用户不需要自行配置计算服务器和数据存储服务器。

Q2：View 和 Materialized View 有什么区别？

View 保存查询逻辑，Materialized View 还保存预计算结果，可以减少重复计算。

## 4.2 Partitioning 和 Clustering

这是 BigQuery 最重要的性能优化知识之一。

### Partitioning

将表的数据划分为不同分区。

例如按照日期：

```
CREATE TABLE `project.dataset.orders`
(
  order_id INT64,
  customer_id INT64,
  order_date DATE,
  amount FLOAT64
)
PARTITION BY order_date;
```

数据会按照日期分区。

查询：

```
SELECT *
FROM `project.dataset.orders`
WHERE order_date >= '2026-11-01'
  AND order_date < '2026-12-01';
```

BigQuery 可以只扫描符合条件的分区。

这就是 Partition Pruning。

### Partition Expiration

可以设置分区自动过期。

例如：

```
Partition Expiration = 90 days
```

超过保留期限的分区会被自动删除。

适合：

- 临时数据。
    
- 日志数据。
    
- 有明确数据保留期限的业务。
    

### Require Partition Filter

你记录的逻辑正确。

通过：

```
--require_partition_filter
```

或者对应表配置，强制查询必须包含分区过滤条件。

这样可以避免用户意外扫描整个大表。

注意：过滤条件必须符合 BigQuery 对分区列过滤的要求。

### Clustering

按照一个或多个字段对数据进行组织。

例如：

```
CREATE TABLE `project.dataset.orders`
PARTITION BY DATE(order_timestamp)
CLUSTER BY customer_id, product_id
AS
SELECT *
FROM `project.dataset.raw_orders`;
```

区别：

|Partitioning|Clustering|
|---|---|
|按分区字段划分数据|按聚簇字段组织数据|
|主要减少扫描分区数量|主要减少需要读取的数据块|
|适合日期等分区字段|适合经常过滤、聚合的字段|
|有分区数量和粒度限制|适用于更细粒度的数据组织|

两者可以一起使用。

### 面试可能问到的问题

Q1：Partitioning 和 Clustering 有什么区别？

Partitioning 将数据划分为独立分区，Clustering 在分区或表内部按照指定字段组织数据。

Q2：什么时候应该使用 Partition Expiration？

当数据具有固定生命周期时，例如日志只保留 90 天，可以自动删除过期分区。

Q3：如何避免 BigQuery 用户意外扫描整个大表？

使用 Require Partition Filter，并设计合理的 Partitioning。

## 4.3 BigQuery Slots 和 Query Optimization

BigQuery 的计算资源叫 Slot。

可以把 Slot 理解为执行 SQL 时使用的计算资源单位。

查询越复杂、扫描和计算的数据越多，可能需要更多 Slot。

### 查看 Slot 使用情况

你记录的 SQL 基本正确，可以保留：

```
SELECT
  job_id,
  user_email,
  creation_time,
  total_slot_ms,
  ROUND(
    total_slot_ms /
      NULLIF(
        TIMESTAMP_DIFF(end_time, start_time, MILLISECOND),
        0
      ),
    2
  ) AS avg_slots,
  total_bytes_processed
FROM
  `region-us`.INFORMATION_SCHEMA.JOBS
WHERE
  creation_time >= TIMESTAMP_SUB(
    CURRENT_TIMESTAMP(), INTERVAL 1 DAY
  )
  AND job_type = 'QUERY'
  AND statement_type != 'SCRIPT'
  AND state = 'DONE'
ORDER BY
  total_slot_ms DESC
LIMIT 20;
```

这里最重要的指标：

|指标|含义|
|---|---|
|total_slot_ms|查询消耗的总 Slot 时间|
|total_bytes_processed|处理的数据量|
|avg_slots|平均使用的 Slot 数|
|creation_time|Job 创建时间|
|total_bytes_billed|计费的数据量，适用于相关计费模式|

计算公式：

\[ Average\ Slots = \frac{total\_slot\_ms}{Execution\ Time\ (ms)} \]

注意：

- `total_slot_ms` 越大，不一定意味着查询越慢。
    
- 平均 Slot 使用量高，可能意味着查询获得了更多并行计算资源。
    
- 查询性能还受数据扫描量、Shuffle、Join、Skew 等影响。
    

### Query Priority

BigQuery 主要有两种常见查询优先级：

|类型|使用场景|
|---|---|
|Interactive|需要尽快获得查询结果|
|Batch|非实时、可以排队执行的任务|

例如：

```
实时 Dashboard
      |
      v
Interactive Query

Nightly ETL
      |
      v
Batch Query
```

Batch Query 可以在计算资源可用时执行，适合不需要立即返回结果的任务。

### 面试可能问到的问题

Q1：如何找到 BigQuery 中消耗资源最多的 Query？

通过 `INFORMATION_SCHEMA.JOBS` 查询 `total_slot_ms`、`total_bytes_processed` 等指标。

Q2：Interactive 和 Batch Query 有什么区别？

Interactive 适合需要尽快执行的任务，Batch 适合可以等待调度的非实时任务。

Q3：一个 Query 扫描数据量很少但执行很慢，可能是什么原因？

可能涉及复杂 Join、Shuffle、数据倾斜、并发资源竞争、Slot 资源不足等问题。

## 4.4 BigQuery ARRAY、STRUCT 和 UNNEST

这是 BigQuery SQL 的一个特殊知识点。

BigQuery 支持 Nested 和 Repeated Fields，因此可以在一行中保存数组和嵌套结构。

### ARRAY

一个字段可以保存多个值。

例如：

```
Customer ID: 1001

Products:
    ["Laptop", "Phone", "Tablet"]
```

使用：

```
ARRAY_AGG(product_name)
```

可以将多行聚合成数组。

常见用法：

```
ARRAY_AGG(DISTINCT product_name)
```

去重。

```
ARRAY_AGG(product_name ORDER BY product_name)
```

排序。

```
ARRAY_AGG(product_name LIMIT 5)
```

限制元素数量。

```
ARRAY_LENGTH(products)
```

获取数组长度。

### UNNEST

UNNEST 的作用是将 ARRAY 中的元素展开为多行。

例如：

```
SELECT
  visitId,
  product
FROM `project.dataset.sessions`,
UNNEST(products) AS product;
```

原始数据：

|visitId|products|
|---|---|
|100|[A, B, C]|
|101|[D, E]|

UNNEST 后：

|visitId|product|
|---|---|
|100|A|
|100|B|
|100|C|
|101|D|
|101|E|

### STRUCT

STRUCT 是一个包含多个字段的结构体。

例如：

```
SELECT
  STRUCT(
    'Alice' AS name,
    25 AS age
  ) AS customer;
```

结果逻辑：

```
{
  "customer": {
    "name": "Alice",
    "age": 25
  }
}
```

ARRAY 和 STRUCT 可以嵌套使用：

```
ARRAY<STRUCT<
  product_name STRING,
  price FLOAT64
>>
```

### 面试可能问到的问题

Q1：UNNEST 的作用是什么？

将 ARRAY 中的元素展开为独立记录，方便进行过滤、聚合和 Join。

Q2：ARRAY_AGG 和 UNNEST 有什么区别？

ARRAY_AGG 把多行聚合成数组，UNNEST 把数组展开成多行。

Q3：为什么 BigQuery 支持 Nested 和 Repeated Fields？

可以减少某些复杂 Join，改善数据局部性，并更自然地表示一对多关系。

# 模块五：BigQuery 数据写入、CDC 与实时数据处理

## 5.1 BigQuery Storage Write API vs Data Transfer Service

核心定位：Data Ingestion

这两个服务经常被放在同一道题中比较，但它们不是同一类产品。

### Storage Write API

BigQuery Storage Write API 是向 BigQuery 写入数据的 API。

典型架构：

```
Application
     |
     v
Dataflow
     |
     v
Storage Write API
     |
     v
BigQuery
```

主要特点：

- 高吞吐写入。
    
- 支持 Streaming。
    
- 支持不同写入 Stream。
    
- 支持 Exactly-once 写入语义（Committed Stream + Offset）。
    

### Data Transfer Service

BigQuery Data Transfer Service 是托管式数据迁移服务。

典型架构：

```
Google Ads
    |
    v
BigQuery Data Transfer
    |
    v
BigQuery
```

支持的来源包括部分 Google SaaS、外部数据平台、GCS、Amazon S3 等，具体 Connector 和配置以实际支持列表为准。

适合：

- 定期导入数据。
    
- 定时同步数据。
    
- 不想自己开发完整数据 Pipeline 的场景。
    

### 对比

|对比|Storage Write API|Data Transfer Service|
|---|---|---|
|本质|写入 API|托管数据搬运服务|
|谁调用|Application / Dataflow|Google 托管服务|
|典型模式|Streaming|Scheduled / Batch|
|实时性|可以很高|通常较低|
|是否需要开发 Pipeline|通常需要|通常不需要复杂开发|
|核心关注点|如何写入 BigQuery|如何从数据源搬运数据|

### Storage Write API：Exactly-once

这是高频考点。

常见 Stream 类型：

|Stream|特征|
|---|---|
|Default Stream|默认写入，At-least-once|
|Committed Stream|通过 Offset 实现 Exactly-once|
|Buffered Stream|支持先写入缓冲区，再提交数据|
|Pending Stream|支持批量写入后提交|

核心逻辑：

```
Application
    |
    v
Committed Stream
    |
    v
Write Rows with Offset
    |
    v
Commit
```

假设：

```
Offset 0 -> Record A
Offset 1 -> Record B
Offset 2 -> Record C
```

如果客户端重试提交已经成功写入的 Offset，BigQuery 可以识别重复 Offset，避免重复写入。

注意：Exactly-once 的保证有特定的 Stream 和 Offset 范围，不等于整个业务 Pipeline 自动实现端到端 Exactly-once。

### 面试可能问到的问题

Q1：Storage Write API 和 Data Transfer Service 如何选择？

如果需要应用或 Dataflow 实时写入 BigQuery，选择 Storage Write API。如果需要定期从受支持的外部数据源同步数据，选择 Data Transfer Service。

Q2：Default Stream 和 Committed Stream 有什么区别？

Default Stream 提供 At-least-once 语义；Committed Stream 配合 Offset 可以提供 Exactly-once 写入语义。

Q3：为什么需要 Offset？

用来标记已经提交的写入位置，帮助系统识别重试和重复提交。

## 5.2 BigQuery CDC

BigQuery CDC 的主要目标是：

让 BigQuery 表能够接收源数据库的增量变化，并将变化应用到目标表。

典型架构：

```
MySQL / PostgreSQL
        |
        v
    Datastream
        |
        v
  BigQuery CDC
        |
        v
  BigQuery Table
```

如果没有 Datastream，也可以通过 Debezium 或其他 CDC 工具捕获变化，然后转换成 BigQuery Storage Write API 支持的 CDC 格式。

### BigQuery CDC 的核心字段

BigQuery CDC 使用 Storage Write API 的 CDC 写入能力。

常见关键字段：

- `_CHANGE_TYPE`
    
- `_CHANGE_SEQUENCE_NUMBER`
    

例如：

```
_CHANGE_TYPE = UPSERT
```

表示插入或更新。

```
_CHANGE_TYPE = DELETE
```

表示删除。

`_CHANGE_SEQUENCE_NUMBER` 用于帮助确定变化事件的顺序。

### 为什么需要 max_staleness？

这是一个非常容易考的概念。

假设：

```
Database Changes
      |
      v
BigQuery CDC
      |
      v
Background Apply
```

BigQuery 接收 CDC 事件后，并不一定立即将所有变化合并进基础表。

它可以先保存变化，再通过后台应用机制处理。

因此：

- `max_staleness` 定义查询可以接受的数据陈旧程度。
    
- BigQuery 可以利用后台应用 CDC 变化的机制。
    
- 当后台应用不够及时，查询可能需要在运行时合并基础表和未应用的变化。
    

这样可以在查询延迟和数据新鲜度之间进行权衡。

例如：

```
max_staleness = 10 minutes
```

表示可以接受一定程度的数据延迟。

### 面试可能问到的问题

Q1：BigQuery CDC 和 Datastream 有什么区别？

Datastream 是 CDC 数据捕获服务；BigQuery CDC 是 BigQuery 接收和应用 CDC 变化的能力。

Q2：BigQuery CDC 如何处理 DELETE？

通过 CDC Change Type 标识 DELETE，使目标表对应记录被删除。

Q3：max_staleness 的作用是什么？

允许 BigQuery 在一定时间范围内使用后台应用的 CDC 结果，从而减少查询时的实时合并开销，但会影响查询数据的新鲜度。

## 5.3 Scheduled Query vs Continuous Query

这是 BigQuery Batch 和 Streaming 分析的另一个重要区别。

### Scheduled Query

定时执行 SQL。

```
10:00 -> Execute SQL -> Finish
10:05 -> Execute SQL -> Finish
10:10 -> Execute SQL -> Finish
```

适合：

- 每日汇总。
    
- 每小时报表。
    
- 定时 ETL。
    
- 数据清洗。
    

### Continuous Query

Continuous Query 是持续运行的 SQL 查询。

```
10:00
  |
  +-- SQL starts
  |
  +-- New Data --> Process
  |
  +-- New Data --> Process
  |
  +-- New Data --> Process
  |
  +-- Continues running
```

适合：

- 实时数据转换。
    
- 持续处理进入 BigQuery 的 Streaming 数据。
    
- 将处理结果输出到支持的目标系统。
    

### Reverse ETL

你记录的 Reverse ETL 定义是正确的。

传统 ETL：

```
Operational DB
       |
       v
   BigQuery
```

Reverse ETL：

```
BigQuery
    |
    v
Business Data
    |
    v
CRM / Marketing / Operational System
```

例如：

- BigQuery 计算客户评分。
    
- 将评分推送到 CRM。
    
- CRM 根据客户评分触发营销活动。
    

但这里需要修正一个地方：

Reverse ETL 不等于 Continuous Query。

Reverse ETL 是一种数据流向和业务模式；Continuous Query 可以作为实现 Reverse ETL 的技术方式之一，也可以使用其他 ETL 或同步工具实现。

### 面试可能问到的问题

Q1：Scheduled Query 和 Continuous Query 有什么区别？

Scheduled Query 定期执行 SQL，Continuous Query 则持续处理不断进入的数据流。

Q2：什么是 Reverse ETL？

将数据仓库中处理好的数据回写到运营系统，使业务应用可以直接使用分析结果。

Q3：什么时候使用 Continuous Query？

当需要持续处理 Streaming 数据，而不是每隔几分钟执行一次批处理时，可以考虑 Continuous Query。

## 5.4 Pub/Sub

核心定位：Messaging / Event Streaming

Cloud Pub/Sub 是 Google Cloud 的消息传递服务。

常见架构：

```
Producer
   |
   v
 Pub/Sub Topic
   |
   v
Subscription
   |
   v
Consumer
```

例如：

```
Order Service
      |
      v
orders-topic
      |
      v
orders-subscription
      |
      v
Dataflow
      |
      v
BigQuery
```

### Topic 和 Subscription

|组件|作用|
|---|---|
|Topic|消息发布入口|
|Subscription|定义消费者如何接收消息|
|Publisher|发布消息|
|Subscriber|接收消息|

一个 Topic 可以有多个 Subscription。

例如：

```
orders-topic
      |
      +----------------+
      |                |
      v                v
analytics-sub      alert-sub
      |                |
      v                v
  Dataflow         Alert Service
```

不同订阅者可以分别消费同一份消息。

### Message Retention

你记录的两种方式非常重要。

默认情况下，消息 ACK 后不再保留供该 Subscription 重新消费。

如果启用：

```
gcloud pubsub subscriptions update SUB_ID \
  --message-retention-duration=2d \
  --retain-acked-messages
```

那么已经 ACK 的消息可以继续保留在 Subscription 的消息保留期限内。

之后可以通过 Seek 重新消费。

```
gcloud pubsub subscriptions seek SUB_ID \
  --time="2026-10-26T00:00:00Z"
```

注意：

- 必须在消息仍处于可保留的时间范围内。
    
- `seek` 会影响对应 Subscription 的消息消费位置。
    
- 不是简单地将消息复制到另一个 Subscription。
    
- 消息保留期限和 Topic Retention 是相关但不同的配置概念。
    

### Message Ordering

Pub/Sub 可以按照 Ordering Key 保证同一个 Key 的消息顺序。

例如：

```
order-123:
    Created
    Paid
    Shipped

order-456:
    Created
    Paid
```

同一个订单的事件需要按顺序处理。

Publisher：

```
from google.cloud import pubsub_v1publisher = pubsub_v1.PublisherClient()topic_path = publisher.topic_path(    "my-project",    "orders")future = publisher.publish(    topic_path,    b'{"status": "paid"}',    ordering_key="order-123")future.result()
```

创建 Subscription：

```
gcloud pubsub subscriptions create orders-sub \
  --topic=orders \
  --enable-message-ordering
```

注意：

- Publisher 需要提供 Ordering Key。
    
- Subscription 需要启用 Message Ordering。
    
- 保证的是同一个 Ordering Key 内的顺序。
    
- 不保证整个 Topic 的所有消息全局有序。
    
- 需要处理消息重试导致的重复消费。
    

### Pub/Sub 和 Kafka

|对比|Pub/Sub|Kafka|
|---|---|---|
|类型|Managed Messaging|Distributed Event Streaming|
|消息组织|Topic / Subscription|Topic / Partition|
|顺序|Ordering Key|Partition 内有序|
|扩展|Google 托管|依赖 Kafka 集群或托管服务|
|消费模式|Push / Pull|Consumer Pull|
|典型用途|GCP Event Pipeline|大规模 Event Streaming|

Kafka 中相同 Key 的消息通常会被路由到同一个 Partition，因此在该 Partition 内保持顺序。

### 面试可能问到的问题

Q1：Pub/Sub 如何实现消息顺序？

通过 Ordering Key，保证同一个 Key 内的消息顺序。

Q2：Pub/Sub 如何重新消费已经 ACK 的消息？

启用 `retain-acked-messages`，在保留期限内使用 Seek。

Q3：Pub/Sub 和 Kafka 有什么区别？

Pub/Sub 是 Google Cloud 全托管消息服务，Kafka 是基于 Partition 的分布式事件流平台。

Q4：Pub/Sub 如何避免重复消费？

消费者实现幂等操作，使用业务唯一 ID 去重，并正确管理 ACK。

## 5.5 Dataproc Workflow Templates

Workflow Template 解决的是：

使用模板化的方式，在 Dataproc 上组织和执行多个 Job。

例如：

```
Workflow Template
       |
       +-- Job 1: Spark ETL
       |
       +-- Job 2: Spark Aggregation
       |
       +-- Job 3: Export Result
```

可以通过参数化模板，重复运行相似的工作流。

你原来写的：

> 只用一个集群调度不同 job

需要修正。

Workflow Template 不仅仅是复用一个集群。它可以定义集群配置、Job 和执行顺序，也支持使用现有集群的工作流模式。

面试中需要区分：

- Workflow Template：定义和组织 Dataproc 工作流。
    
- Cloud Composer：管理更广泛的 DAG、跨服务调度和依赖关系。
    
- Cloud Scheduler：定时触发任务。
    

# 模块六：数据湖与外部数据访问

## 6.1 BigLake

核心定位：Lakehouse / External Data Access

BigLake 允许 BigQuery 访问外部对象存储中的数据。

例如：

```
GCS
S3
Azure Blob Storage
       |
       v
    BigLake
       |
       v
   BigQuery
       |
       v
      SQL
```

核心价值：

不需要将所有数据复制到 BigQuery Managed Storage，就可以通过 BigQuery SQL 查询外部数据。

### External Table vs BigLake Table

这是考试中非常重要的比较。

|对比|External Table|BigLake Table|
|---|---|---|
|数据位置|外部存储|外部存储|
|元数据|BigQuery|BigQuery|
|Access Delegation|传统 External Table 不提供 BigLake 的授权代理能力|支持|
|底层 GCS 权限|通常需要查询用户具备底层访问权限|Connection Service Account 需要|
|表级权限|支持|支持|
|Row-level Security|不具备 BigLake 的增强治理能力|支持相关能力|
|Column-level Security|不具备 BigLake 的增强治理能力|支持相关能力|
|DML 修改外部数据|通常不支持|不能将其当成普通 BigQuery Managed Table 执行任意 DML|

### Access Delegation

这是最重要的区别。

普通 External Table：

```
User
  |
  +-- BigQuery Permission
  |
  +-- GCS Permission
```

用户通常需要同时具有：

- BigQuery 表访问权限。
    
- 底层 GCS 数据访问权限。
    

BigLake Table：

```
User
  |
  v
BigQuery Table Permission
  |
  v
BigLake Table
  |
  v
Cloud Resource Connection
  |
  v
Service Account
  |
  v
GCS
```

通过 Cloud Resource Connection 的 Service Account 访问底层数据。

这样用户可以不直接拥有底层 GCS Bucket 的访问权限。

这就是 Access Delegation。

### 面试可能问到的问题

Q1：BigLake 和 External Table 有什么区别？

BigLake 在外部数据访问基础上提供更完善的访问委托和数据治理能力。

Q2：BigLake 为什么需要 Cloud Resource Connection？

用于提供 Service Account 身份，让 BigQuery 通过该身份访问底层对象存储。

Q3：BigLake 是否会将 GCS 数据自动复制到 BigQuery？

不会。BigLake 主要提供外部数据访问和治理能力。

## 6.2 RAG、Document AI 和 Embedding

核心定位：Retrieval-Augmented Generation

RAG 是一种结合外部知识检索与生成式 AI 的架构。

你记录的：

> Document AI transfer the pdf file into txts

是一个合理的起点，但完整流程需要增加 Parsing、Chunking、Embedding 和 Retrieval。

### RAG 完整流程

```
PDF / Documents
       |
       v
   Document AI
       |
       v
 Document Parsing
       |
       v
 Text / Tables
       |
       v
    Chunking
       |
       v
   Embedding
       |
       v
 Vector Database
       |
       v
  Vector Search
       |
       v
 Relevant Context
       |
       v
     Gemini
       |
       v
    Answer
```

### Document AI

Document AI 可以从 PDF、扫描文件等文档中提取结构化内容。

例如：

- OCR。
    
- 表格提取。
    
- 表单字段识别。
    
- 文档分类。
    
- 文档解析。
    

注意：普通 PDF 文本提取不一定必须使用 Document AI。对于扫描件、复杂表格、表单和版面分析，Document AI 更有价值。

### Chunking

为什么不能把整个 PDF 直接生成一个 Embedding？

因为：

- 文档可能超过模型输入长度限制。
    
- 整份文档的语义过于宽泛。
    
- 查询时难以定位具体段落。
    

所以需要切分成 Chunk：

```
PDF
 |
 +-- Chunk 1
 |
 +-- Chunk 2
 |
 +-- Chunk 3
 |
 +-- Chunk 4
```

每个 Chunk 单独生成 Embedding。

### Embedding

Embedding 是将文本转换成向量。

例如：

```
"Amsterdam is the capital of the Netherlands"
                 |
                 v
[0.12, 0.45, -0.31, ...]
```

语义相似的文本，其向量通常也比较接近。

### Retrieval

用户提问：

```
What is the capital of Netherlands?
```

先将问题生成 Query Embedding，然后查找最相似的文档 Chunk。

```
Query
  |
  v
Query Embedding
  |
  v
Vector Search
  |
  v
Top K Chunks
  |
  v
Gemini
  |
  v
Answer
```

### 面试可能问到的问题

Q1：为什么 RAG 需要 Chunking？

为了控制输入长度，提高检索粒度，让系统可以准确找到与问题相关的局部内容。

Q2：Embedding 和 Vector Search 有什么区别？

Embedding 负责把文本转换成向量，Vector Search 负责寻找与查询向量最相似的向量。

Q3：为什么 RAG 不直接把整个文档发给 Gemini？

可能存在上下文长度限制、成本较高、检索不精准等问题。RAG 通过检索相关 Chunk 降低输入量。

# 模块七：BigQuery AI、向量检索与多模态数据

## 7.1 BigQuery Embedding 和 Vector Search

核心定位：Semantic Search

BigQuery 不仅可以分析结构化数据，也支持向量检索。

### 第一步：创建 Embedding Model

你原来的 SQL 思路正确：

```
CREATE OR REPLACE MODEL
  `bqml_lab.embedding_model`
REMOTE WITH CONNECTION DEFAULT
OPTIONS (
  ENDPOINT = 'text-embedding-005'
);
```

这里通过 Remote Model 调用外部模型服务。

注意：具体模型 Endpoint 和 Connection 要根据当前 Vertex AI 模型支持情况选择，模型名称可能会更新。

### 第二步：生成 Embedding

```
CREATE OR REPLACE TABLE
  `bqml_lab.embeddings` AS

SELECT *
FROM ML.GENERATE_EMBEDDING(
  MODEL `bqml_lab.embedding_model`,
  (
    SELECT
      title,
      url,
      abstract AS content
    FROM `bqml_lab.patent_data`
    LIMIT 200000
  )
)
WHERE
  ml_generate_embedding_status = '';
```

生成的结果中会包含向量字段。

### 第三步：创建 Vector Index

```
CREATE OR REPLACE VECTOR INDEX my_index
ON `bqml_lab.embeddings`
(ml_generate_embedding_result)
OPTIONS (
  index_type = 'IVF',
  distance_type = 'COSINE',
  ivf_options = '{"num_lists":500}'
);
```

### IVF 是什么？

IVF：Inverted File Index。

核心思想是：

提前将大量向量划分为多个 Cluster。

例如：

```
1,000,000 Vectors
        |
        v
      IVF
        |
   +----+----+
   |    |    |
   v    v    v
 C1   C2   C3 ...
```

查询时：

```
Query Vector
      |
      v
Find Relevant Clusters
      |
      v
Search Candidate Vectors
      |
      v
Top K Results
```

这样可以减少需要比较的向量数量。

注意：

IVF 属于近似最近邻检索（ANN）索引思路，查询速度和召回率之间可能存在取舍。

### COSINE Distance

Cosine Similarity 衡量两个向量的方向相似度。

\[ \cos(\theta)= \frac{A\cdot B}{\|A\|\|B\|} \]

BigQuery Vector Search 可以配置 COSINE、EUCLIDEAN 等距离类型，具体支持的类型以当前产品文档为准。

### 第四步：VECTOR_SEARCH

例如：

```
SELECT *
FROM VECTOR_SEARCH(
  TABLE `bqml_lab.embeddings`,
  'ml_generate_embedding_result',
  TABLE `bqml_lab.query_embeddings`,
  query_column_to_search => 'query_embedding',
  top_k => 5,
  distance_type => 'COSINE'
);
```

### 查看 Vector Index 状态

```
SELECT
  table_name,
  index_name,
  index_status,
  coverage_percentage,
  last_refresh_time,
  disable_reason
FROM
  `project.bqml_lab.INFORMATION_SCHEMA.VECTOR_INDEXES`;
```

重点关注：

- `index_status`
    
- `coverage_percentage`
    
- `last_refresh_time`
    
- `disable_reason`
    

注意：创建 Index 不代表立即完成构建，查询时也需要考虑索引覆盖情况和查询回退机制。

### 面试可能问到的问题

Q1：Vector Index 为什么能加速向量搜索？

通过预先组织向量并缩小候选搜索范围，减少需要进行距离计算的向量数量。

Q2：IVF 和精确搜索有什么区别？

IVF 通过搜索部分相关 Cluster 来提高效率，可能牺牲部分召回率；精确搜索则需要更全面地比较候选向量。

Q3：Embedding 是什么？

Embedding 是把文本、图片等内容转换为数值向量，以便进行语义相似性计算。

## 7.2 BigQuery Object Table 和 Gemini 多模态处理

你记录的核心思路正确：

BigQuery 可以通过 Object Table 管理 GCS 中的非结构化文件，并结合 Gemini 等模型进行多模态分析。

### Object Table

Object Table 可以提供 GCS 对象的元数据访问，例如：

- URI。
    
- 文件大小。
    
- 文件类型。
    
- 对象更新时间。
    
- 对象访问信息。
    

它不是将图片或 PDF 内容直接转成普通 BigQuery 字段。

而是为对象数据提供可查询的元数据入口。

### 整体架构

```
GCS
 |
 +-- Images
 |
 +-- PDFs
 |
 +-- Videos
 |
 v
BigQuery Object Table
 |
 v
Remote Model
 |
 v
Gemini
 |
 v
Classification / Extraction / Summarization
 |
 v
BigQuery Results
```

### Remote Model

你记录的 SQL 需要区分三个阶段：

1. 创建 Object Table。
    
2. 创建 Remote Model。
    
3. 调用 ML.GENERATE_TEXT 或相应模型函数。
    

示意：

```
CREATE OR REPLACE MODEL
  `project.dataset.gemini_model`
REMOTE WITH CONNECTION `project.region.connection`
OPTIONS (
  ENDPOINT = 'gemini-2.5-flash'
);
```

然后可以把对象 URI 和文本 Prompt 一起传给支持多模态输入的模型接口。

例如：

```
Image URI:
gs://bucket/customer_review.png

Prompt:
Classify the sentiment of this customer review.
```

注意：实际 SQL 输入结构、ObjectRef 和模型支持能力需要遵循对应模型函数的接口，不是任意 `social_media_resource` 字段都可以直接被 Gemini 当作图片读取。

### 面试可能问到的问题

Q1：Object Table 和 External Table 有什么区别？

Object Table 主要用于访问对象存储中非结构化文件的元数据和对象引用；普通 External Table 通常面向结构化或半结构化数据文件的查询。

Q2：如何使用 BigQuery 分析图片？

可以使用 Object Table 组织对象信息，配合 Remote Model 调用多模态模型进行图片分析。

Q3：为什么需要 Remote Model？

因为模型推理可以由外部受支持的模型服务执行，而 BigQuery 提供 SQL 调用入口。

## 7.3 BigQuery ML（BQML）

核心定位：Machine Learning inside BigQuery

BigQuery ML 允许用户直接通过 SQL 创建、训练、评估和使用机器学习模型。

你记录的：

```
Model Preparation
       |
       v
Model Training
       |
       v
Model Evaluation
       |
       v
Prediction
```

建议补全为：

```
Data Preparation
       |
       v
Feature Engineering
       |
       v
Model Training
       |
       v
Model Evaluation
       |
       v
Model Deployment / Inference
       |
       v
Monitoring
```

### 常见模型类型

- Linear Regression。
    
- Logistic Regression。
    
- K-means。
    
- Time Series。
    
- Boosted Tree。
    
- Random Forest。
    
- DNN。
    
- Remote Model。
    

### 面试可能问到的问题

Q1：BigQuery ML 有什么优势？

可以使用 SQL 直接训练模型，不需要将数据大量导出到外部计算环境。

Q2：BQML 和 Vertex AI 有什么区别？

BQML 更适合在 BigQuery 中直接进行 SQL 驱动的机器学习；Vertex AI 提供更广泛的模型训练、部署、管理和 MLOps 能力。

Q3：什么时候使用 Remote Model？

当需要在 BigQuery 中通过 SQL 调用外部托管模型，例如 Embedding 或 Gemini 时，可以使用 Remote Model。

# 模块八：BigQuery 数据共享与加密保护

## 8.1 Data Sharing：Authorized View vs Analytics Hub

核心定位：Secure Data Sharing

### Authorized View

假设有一张包含敏感信息的表：

```
customers
   |
   +-- customer_id
   +-- name
   +-- email
   +-- phone
   +-- salary
```

你只希望向其他团队开放：

```
customer_public_view
   |
   +-- customer_id
   +-- name
```

可以创建 View：

```
CREATE VIEW `project.public.customer_view` AS
SELECT
  customer_id,
  name
FROM `project.private.customers`;
```

然后将 View 授权为 Authorized View，使其能够读取底层 Dataset 的数据。

使用者只需要访问被授权的 View，不需要直接获得底层 Dataset 的数据访问权限。

### Analytics Hub

BigQuery sharing (Analytics Hub) 是 BigQuery 的数据交换和共享能力。

它适合：

- 跨团队数据共享。
    
- 跨项目数据共享。
    
- 跨组织发布数据产品。
    
- 数据集订阅和发现。
    

主要概念：

|概念|作用|
|---|---|
|Exchange|数据交换空间|
|Listing|发布的数据产品|
|Subscription|订阅数据产品|
|Linked Dataset|订阅方关联的数据集|

### 对比

|对比|Authorized View|Analytics Hub|
|---|---|---|
|主要用途|通过 View 控制数据访问|发布、发现和订阅数据产品|
|典型场景|开放部分客户字段|跨团队、跨组织共享|
|是否复制数据|通常不需要|通常不需要|
|核心机制|View + IAM|Exchange + Listing + Subscription|

### 面试可能问到的问题

Q1：如何共享 BigQuery 数据而不暴露底层敏感字段？

使用 Authorized View，只开放经过筛选的字段。

Q2：Analytics Hub 和 Authorized View 有什么区别？

Authorized View 解决视图级数据访问控制；Analytics Hub 解决数据产品发布、发现和订阅。

Q3：Analytics Hub 是否必须复制数据？

通常不需要复制底层数据，订阅方通过 Linked Dataset 访问发布的数据。

## 8.2 BigQuery AEAD Encryption 和 Crypto-shredding

核心定位：Cryptographic Data Protection

AEAD：

Authenticated Encryption with Associated Data。

BigQuery 提供 AEAD 加密函数，可以对敏感字段进行加密和解密。

你记录的核心概念是 Crypto-shredding：

通过销毁解密所需的密钥，使加密后的数据无法再恢复。

### 第一步：生成 Keyset

```
SELECT
  customer_id,
  KEYS.NEW_KEYSET('AEAD_AES_GCM_256') AS keyset
FROM customer_ids;
```

这里为每个 Customer 生成 Keyset。

### 第二步：加密数据

```
SELECT
  customer_id,
  AEAD.ENCRYPT(
    keyset,
    email,
    CAST(customer_id AS STRING)
  ) AS encrypted_email
FROM customer_data_with_keysets;
```

其中：

- Keyset：保存加密密钥信息。
    
- Email：需要加密的原始数据。
    
- Associated Data：与密文绑定的附加数据。
    

这里的 `customer_id` 作为 Associated Data，可以用于验证数据完整性和上下文一致性。

解密时需要提供对应的 Keyset 和相同的 Associated Data。

### 第三步：解密

```
SELECT
  AEAD.DECRYPT_STRING(
    keyset,
    encrypted_email,
    CAST(customer_id AS STRING)
  ) AS email
FROM encrypted_customer_data;
```

### 第四步：Crypto-shredding

假设：

```
Customer A
   |
   v
Keyset A
   |
   v
Encrypted Email A
```

如果删除了唯一可用的 Keyset A：

```
Keyset A: Destroyed

Encrypted Email A: Still exists
```

原始数据将无法再通过该 Keyset 解密。

但是需要注意：

- Keyset 必须安全保存。
    
- 不能存在其他可用的密钥副本。
    
- 需要考虑备份、导出和历史版本。
    
- 单纯删除一行 Keyset 并不自动等于经过完整验证的合规数据销毁。
    

### 面试可能问到的问题

Q1：Hashing 和 AEAD Encryption 有什么区别？

Hashing 通常是单向的；AEAD Encryption 可以解密，并且提供数据完整性认证。

Q2：什么是 Crypto-shredding？

销毁解密密钥，让已经加密的数据无法再恢复为原始数据。

Q3：为什么使用 Associated Data？

将额外上下文绑定到密文，防止密文被错误地用于其他上下文。

# 模块九：Cloud SQL、Spanner 和 Bigtable 的场景选择

这一模块建议你单独记忆，因为考试很喜欢给出业务场景，让你选择数据库。

|业务需求|优先考虑|原因|
|---|---|---|
|传统关系型数据库|Cloud SQL|MySQL / PostgreSQL / SQL Server|
|高性能 PostgreSQL|AlloyDB|PostgreSQL 兼容和性能优化|
|全球分布式关系型数据库|Spanner|强一致性和水平扩展|
|超大规模低延迟 Key-value|Bigtable|Wide-column、高吞吐|
|文档型 NoSQL|Firestore|灵活文档结构|
|大规模分析型数据仓库|BigQuery|Serverless OLAP|
|外部对象存储查询|BigLake|统一数据湖访问|

### 高频场景判断

场景 1：

一个全球电商平台，需要多个 Region 的用户同时写入订单，并要求强一致性。

推荐：Spanner。

场景 2：

IoT 系统每秒写入大量传感器数据，主要按照设备 ID 和时间查询。

推荐：Bigtable。

场景 3：

应用使用 PostgreSQL，希望减少数据库管理工作，并提升性能。

推荐：AlloyDB 或 Cloud SQL PostgreSQL，根据性能要求和成本选择。

场景 4：

需要对 PB 级历史交易数据执行复杂 SQL 聚合。

推荐：BigQuery。

场景 5：

移动应用需要存储用户 Profile、动态 JSON 和嵌套文档。

推荐：Firestore。

### 面试可能问到的问题

Q1：Bigtable 和 Spanner 都能水平扩展，如何选择？

Bigtable 主要是 NoSQL Wide-column，适合 Key-based 高吞吐访问；Spanner 是关系型分布式数据库，支持 SQL、事务和强一致性。

Q2：为什么不用 Cloud SQL 存储 PB 级分析数据？

Cloud SQL 面向 OLTP，BigQuery 更适合大规模 OLAP。

Q3：什么时候选择 Firestore 而不是 Bigtable？

数据以文档形式组织、查询需要文档级访问时优先考虑 Firestore；大规模低延迟 Key Range 访问更适合 Bigtable。

# 模块十：其他需要补充的 Google Cloud 服务

## 10.1 Dataprep

Dataprep 是一个图形化数据清洗和转换工具，基于可视化方式构建数据处理步骤。

你写的：

> Dataprep: graphic data washing, less used

基本正确，但需要补充一个考试注意点：

Google Cloud Dataprep by Trifacta 已经停止服务（服务于 2025 年结束）。

因此，2026 年复习时不应该把它当作当前推荐的生产方案。

现在应该更多理解：

- Dataflow：复杂、可编程的数据转换。
    
- BigQuery SQL：仓库内数据转换。
    
- Dataplex：数据治理和管理。
    
- Dataplex 中的数据质量能力：用于数据质量检查。
    
- 其他受支持的数据准备工具：根据当前产品情况选择。
    

不要把 Dataprep 和 Dataplex 混淆。

## 10.2 Memorystore

核心定位：Managed In-memory Cache

Memorystore 是 Google Cloud 的托管内存数据存储服务。

常见技术：

- Redis。
    
- Valkey。
    
- Memcached（具体可用性取决于产品配置）。
    

典型架构：

```
Application
     |
     v
Memorystore
     |
     +-- Cache
     +-- Session
     +-- Frequently Accessed Data
```

例如：

用户请求：

```
Get Customer Profile
```

先检查 Cache：

```
Application
     |
     v
Memorystore
     |
     +-- Cache Hit --> Return
     |
     +-- Cache Miss
              |
              v
           Database
              |
              v
           Update Cache
```

注意：

Memorystore 是 Cache / In-memory Store，不是 Vault 或专门的密钥管理服务。

如果要保存和管理加密密钥，应该考虑 Cloud KMS。

### 面试问题

Q1：为什么使用 Memorystore 而不是直接查询 Cloud SQL？

减少数据库访问压力，降低重复读取的延迟。

Q2：Cache-Aside Pattern 是什么？

应用先查询 Cache，未命中时访问数据库，再将结果写回 Cache。

Q3：Cache 和数据库数据不一致怎么办？

需要设计 Cache Invalidation、TTL、Write-through 或 Write-around 等机制。

## 10.3 Cloud Composer

核心定位：Workflow Orchestration

Cloud Composer 是 Google Cloud 基于 Apache Airflow 的托管工作流服务。

典型架构：

```
Cloud Composer
      |
      v
   Airflow DAG
      |
      +-- Dataproc Job
      |
      +-- Dataflow Job
      |
      +-- BigQuery SQL
      |
      +-- GCS Operation
```

它负责：

- Task Scheduling。
    
- Dependency Management。
    
- Retry。
    
- Monitoring。
    
- Workflow Orchestration。
    

和 Dataproc Workflow Template 的区别：

|Composer|Dataproc Workflow Template|
|---|---|
|通用工作流调度|Dataproc 专用工作流|
|基于 Airflow DAG|基于 Dataproc Jobs|
|可调度多个 GCP 服务|主要组织 Dataproc 作业|
|支持复杂依赖|更适合 Spark/Hadoop 工作流|

### 面试问题

Q1：Cloud Composer 和 Dataflow 有什么区别？

Composer 负责调度和管理工作流，Dataflow 负责实际的数据处理。

Q2：为什么不用 Cloud Scheduler 代替 Composer？

Cloud Scheduler 适合定时触发任务，Composer 适合管理多个有依赖关系、需要重试和监控的复杂工作流。

## 10.4 Cloud Storage

Cloud Storage 是 Google Cloud 的对象存储服务，也是数据湖的重要基础设施。

建议补充以下知识：

### Storage Classes

|类型|适合场景|
|---|---|
|Standard|高频访问|
|Nearline|约每月访问一次|
|Coldline|约每季度访问一次|
|Archive|极低频访问|

核心区别是存储成本、读取成本和最短存储期限。

### Lifecycle Management

可以通过 Lifecycle Rules 自动管理对象生命周期。

例如：

```
0-30 days:
    Standard

30-90 days:
    Nearline

90-365 days:
    Coldline

365+ days:
    Archive
```

也可以配置过期删除。

### 面试问题

Q1：Nearline、Coldline 和 Archive 如何选择？

根据访问频率、读取费用、存储成本和最短存储期限综合选择。

Q2：GCS Lifecycle Management 有什么作用？

自动将数据迁移到更低成本的 Storage Class，或者删除满足条件的对象。

# 模块十一：BigQuery 和 Bigtable 的数据建模及高级知识

## 11.1 BigQuery Data Modeling

除了 ARRAY、STRUCT，还建议补充以下知识。

### Star Schema

```
             Fact Sales
                 |
       +---------+---------+
       |         |         |
       v         v         v
   Dim Date  Dim Customer Dim Product
```

- Fact Table：保存业务事件和度量值。
    
- Dimension Table：保存业务实体属性。
    

例如：

```
Fact Sales:
    order_id
    customer_id
    product_id
    date_id
    amount

Dim Customer:
    customer_id
    name
    city
```

### Denormalization

BigQuery 支持 Nested 和 Repeated Fields，因此可以通过嵌套结构减少某些 Join。

例如：

```
CREATE TABLE orders (
  order_id INT64,
  customer STRUCT<
    name STRING,
    city STRING
  >,
  products ARRAY<STRUCT<
    product_name STRING,
    price FLOAT64
  >>
);
```

这种设计对于父子关系查询比较有利。

### 面试问题

Q1：为什么 BigQuery 经常推荐 Denormalization？

可以减少 Join 和 Shuffle，提高某些分析查询的效率。

Q2：Star Schema 和 Nested Schema 如何选择？

需要根据数据关系、查询模式、数据更新方式和查询成本决定。

## 11.2 Bigtable 数据模型进阶

建议再补充两个概念。

### Garbage Collection

Bigtable 可以通过 Garbage Collection Policy 管理旧版本数据。

例如：

- 保留最近 N 个版本。
    
- 只保留最近 N 天的数据。
    

Bigtable 的单元格可以具有多个时间戳版本，因此需要合理设置 GC Policy，避免历史版本无限增长。

### Tablet

Bigtable 会将表的数据划分成多个 Tablet。

```
Bigtable Table
      |
      +-- Tablet 1
      |
      +-- Tablet 2
      |
      +-- Tablet 3
```

Tablet 会根据 Key Range 分布到不同节点。

当某个 Tablet 负载过高时，系统可以进行拆分和重新分配。

但如果 Row Key 设计本身存在严重热点，仅仅依靠自动拆分不一定能完全解决问题。

### 面试问题

Q1：Bigtable 的数据为什么会产生多个版本？

同一个 Cell 可以有不同 Timestamp 的值，支持版本化存储。

Q2：GC Policy 有什么作用？

控制旧版本数据的保留时间或数量，减少存储开销。

Q3：Tablet 和 Node 有什么区别？

Tablet 是数据分片，Node 是提供计算和服务能力的资源。一个 Node 可以服务多个 Tablet。

# 模块十二：你这份笔记中最值得修正的知识点

下面这些是我认为最容易在考试中答错的地方，建议单独背诵。

|原笔记|修正后的理解|
|---|---|
|Dataproc Metastore 保存整个集群的数据|只管理 Hive 元数据|
|Persistent History Server 保存数据|保存和查看 Spark Job 历史|
|Dataflow 是 Serverless Dataproc|Dataflow 是基于 Apache Beam 的独立计算服务|
|Datastream 没有延迟|Near-real-time CDC，不代表零延迟|
|Dataflow 不适合 Soft Delete|Dataflow 可以处理 Soft Delete，关键是目标端删除语义|
|Firestore 就是 MongoDB|两者都是文档型 NoSQL，但产品、数据模型和 API 不同|
|索引太多会导致 Cross Join|索引太多主要增加存储和写入维护开销|
|Cloud SQL 统一最大 64 TB|容量取决于引擎、版本和配置|
|AlloyDB 是 OLAP 和 OLTP 数据库|主要面向高性能 OLTP，具备分析增强能力|
|Spanner Interleaving 就是保存 Join 结果|主要改善父子表数据 locality|
|Bigtable 可以有无限列|Wide-column 模型，Qualifier 灵活，但仍有产品限制|
|Multi-cluster Routing 保证立即读到最新数据|需要结合一致性语义，不能直接这样假设|
|Reverse ETL 基于 Continuous Query|Reverse ETL 是数据回流模式，Continuous Query 只是实现方式之一|
|Storage Write API 所有模式都是 Exactly-once|Default Stream 为 At-least-once，Committed Stream + Offset 可以实现 Exactly-once|
|BigLake 会复制外部数据|BigLake 主要提供外部数据访问和治理|
|Memorystore 是 Vault|Memorystore 是内存数据存储，Cloud KMS 才是密钥管理服务|
|Object Table 直接存储图片二进制|主要管理对象引用和元数据|
|AEAD 删除一行 Keyset 就一定满足合规销毁|还需要确保不存在密钥副本和其他恢复途径|

# 模块十三：Google Data Engineer 高频场景题

这一部分比单独背产品定义更加重要。

考试往往给你一个业务场景，然后要求你选择成本最低、维护最简单、最符合技术要求的方案。

下面是值得重点练习的题目。

## 13.1 数据摄取与处理

|场景|推荐方案|核心理由|
|---|---|---|
|MySQL 实时同步到 BigQuery|Datastream|Transaction Log CDC|
|Pub/Sub 实时消息处理|Dataflow|Streaming Pipeline|
|GCS 文件批量 ETL|Dataflow / BigQuery SQL|根据转换复杂度选择|
|Spark 作业迁移到 GCP|Dataproc|兼容 Spark|
|定时从 Google Ads 导入 BQ|Data Transfer Service|托管数据迁移|
|高吞吐实时写入 BQ|Storage Write API|Streaming Ingestion|
|定期执行 SQL|Scheduled Query|定时执行|
|持续处理实时数据|Continuous Query|长时间运行 SQL|

## 13.2 数据库选择

|场景|推荐方案|
|---|---|
|全球分布式事务|Spanner|
|高吞吐时间序列|Bigtable|
|PostgreSQL 高性能|AlloyDB|
|传统关系型应用|Cloud SQL|
|文档型应用|Firestore|
|大规模分析|BigQuery|
|外部对象存储分析|BigLake|
|高频读取缓存|Memorystore|

## 13.3 数据安全

|场景|推荐方案|
|---|---|
|查找数据库中的 PII|Sensitive Data Protection Inspection|
|持续盘点敏感数据|Discovery|
|隐藏手机号|Masking|
|不可逆地转换用户 ID|Hashing / HMAC|
|可恢复地保护敏感信息|Encryption|
|只开放部分列|Authorized View / Column-level Security|
|只开放部分行|Row-level Security|
|共享数据产品|Analytics Hub|
|通过销毁密钥删除数据可恢复性|Crypto-shredding|

## 13.4 性能优化

|场景|推荐方案|
|---|---|
|减少 BigQuery 扫描数据量|Partitioning|
|减少分区内扫描量|Clustering|
|强制用户过滤分区|Require Partition Filter|
|重复聚合查询加速|Materialized View|
|分析资源消耗|INFORMATION_SCHEMA.JOBS|
|非实时任务节省计算资源|Batch Query|
|Bigtable 热点检测|Key Visualizer|
|Bigtable 分散写入|Row Key Design|
|Bigtable 业务访问隔离|App Profile|

# 模块十四：综合面试问题与回答

这里我挑选 15 道我认为最值得准备的题目。它们可以检验你是否真正理解整套 Google Cloud Data Engineering 架构。

## 第一组：数据处理架构

Q1：设计一个从 MySQL 到 BigQuery 的实时数据 Pipeline。

参考回答：

我会使用 Datastream 捕获 MySQL Binlog 中的 INSERT、UPDATE 和 DELETE 事件，并将 CDC 数据写入 BigQuery。

如果业务需要额外的数据清洗、字段转换或复杂业务逻辑，可以加入 Dataflow。

在目标端，需要根据 Primary Key、Change Sequence 和业务要求处理数据更新、去重和删除。

如果需要维护历史变化，可以采用 SCD Type 2。

Q2：Dataproc 和 Dataflow 如何选择？

参考回答：

Dataproc 更适合已经使用 Spark 或 Hadoop 的项目，可以复用现有代码和生态。

Dataflow 基于 Apache Beam，适合统一 Batch 和 Streaming 数据处理，提供托管计算、自动扩缩容和流处理能力。

Q3：ETL 和 ELT 如何选择？

参考回答：

如果需要复杂的预处理、实时转换或者外部系统处理，可以在 Dataflow 中进行 Transform 后写入 BigQuery。

如果数据可以先加载到 BigQuery，再通过 SQL 处理，那么 ELT 通常更简单，并且可以充分利用 BigQuery 的计算能力。

## 第二组：数据库与分布式系统

Q4：Bigtable 为什么会出现 Hotspot？如何解决？

参考回答：

Bigtable 按 Row Key 的字典序分布数据。如果使用递增时间戳作为 Row Key 前缀，新数据可能集中写入连续 Key Range。

可以通过 Hash Prefix、Shard ID 和重新设计 Row Key 来分散写入压力，同时需要平衡查询效率。

Q5：Spanner 和 Cloud SQL 的区别？

参考回答：

Cloud SQL 是托管关系型数据库，适合传统 OLTP 工作负载。

Spanner 是分布式关系型数据库，支持水平扩展、分布式事务和全球强一致性，适合需要全球部署和大规模事务处理的系统。

Q6：BigQuery 为什么适合 OLAP 而不是 OLTP？

参考回答：

BigQuery 的架构和列式存储针对大规模扫描、聚合和分析优化。

OLTP 需要大量小型随机读写、低延迟事务和细粒度更新，更适合 Cloud SQL、AlloyDB 或 Spanner。

## 第三组：数据治理与安全

Q7：如何在 BigQuery 中实现多租户数据隔离？

参考回答：

可以使用 Row-level Security，根据用户身份或租户 ID 过滤数据。

如果不同租户需要完全独立的权限边界，也可以考虑 Dataset 级别隔离。

还可以结合 Column-level Security 保护敏感字段。

Q8：如何在不暴露原始客户 ID 的情况下关联多个数据源？

参考回答：

可以使用稳定的 HMAC Tokenization。

不同数据源使用相同的密钥和算法处理相同客户 ID，就能生成一致的 Token，实现跨数据集关联，同时不直接暴露原始 ID。

Q9：BigLake 如何避免用户直接访问底层 GCS？

参考回答：

使用 BigLake Table 和 Cloud Resource Connection。

用户只需要相应 BigQuery 表访问权限，底层 GCS 由 Connection 对应的 Service Account 访问。

## 第四组：实时数据和可靠性

Q10：如何保证 Pub/Sub 消息不会因为重试导致业务重复处理？

参考回答：

使用业务唯一 ID 和幂等处理逻辑。

消费者在处理成功后再 ACK，并通过去重记录、数据库唯一约束或幂等写入避免重复产生业务影响。

Q11：如何实现 BigQuery Streaming 的 Exactly-once？

参考回答：

使用 BigQuery Storage Write API 的 Committed Stream，并通过 Offset 管理写入位置。重试时使用相同的 Offset，避免重复提交相同数据。

Q12：如果 CDC 事件乱序到达怎么办？

参考回答：

使用数据库主键、Transaction Log Position、Change Sequence Number 和事件版本等信息确定事件顺序。

在目标端拒绝过期事件，并通过幂等更新处理重复消息。

## 第五组：性能和成本

Q13：如何优化一个运行缓慢的 BigQuery Query？

参考回答：

首先检查 Query Execution Details，判断是否存在过多数据扫描、Shuffle 或数据倾斜。

然后考虑：

- Partitioning。
    
- Clustering。
    
- 减少不必要的字段读取。
    
- 优化 Join。
    
- 减少重复计算。
    
- Materialized View。
    
- Slot 资源和并发情况。
    

Q14：如何降低 BigQuery 的存储成本？

参考回答：

使用 Partition Expiration 清理过期数据，合理选择存储模型，减少不必要的重复数据，并利用长期存储价格机制。

Q15：如何设计跨区域灾难恢复？

参考回答：

需要根据 RPO 和 RTO 选择方案。

例如：

- Cloud SQL：跨区域 Read Replica。
    
- Bigtable：多区域集群和相应路由策略。
    
- Spanner：多区域实例配置。
    
- BigQuery：跨区域备份或数据复制方案。
    

需要区分自动 Failover、手动 Promote 和数据恢复。

# 模块十五：建议的最终复习路线

你现在记录的知识点很多，但有些是基础概念，有些是产品特性，还有些是高级架构。

建议按照以下顺序复习，而不是按照笔记原来的顺序反复看。

## Google Data Engineer 知识地图

1. 第一阶段：数据处理基础
    
    Dataproc、Dataflow、Datastream、Pub/Sub、Cloud Composer
    
    目标：能设计 Batch、Streaming、CDC Pipeline。
    
2. 第二阶段：数据存储与建模
    
    BigQuery、Bigtable、Firestore、Cloud SQL、AlloyDB、Spanner
    
    目标：根据业务需求选择数据库，理解 Partition、Clustering、Row Key。
    
3. 第三阶段：数据治理与安全
    
    Sensitive Data Protection、IAM、RLS、CLS、BigLake、Analytics Hub
    
    目标：理解数据权限、PII 保护和安全共享。
    
4. 第四阶段：高级数据工程
    
    Storage Write API、CDC、Continuous Query、Vector Search、BQML、RAG
    
    目标：理解实时数据处理和 AI 数据架构。
    
5. 第五阶段：性能与架构设计
    
    Slots、Materialized View、HA、PITR、Disaster Recovery、成本优化
    
    目标：能够针对 RPO、RTO、Latency、Cost 设计方案。
    

## 最后：考试和面试最值得优先掌握的 10 个主题

如果时间有限，我建议优先掌握以下内容：

|优先级|主题|必须理解的知识|
|---|---|---|
|1|BigQuery|Partition、Clustering、Slots、Cost|
|2|Dataflow|Batch、Streaming、Window、Watermark|
|3|Datastream|CDC、Transaction Log、SCD|
|4|Bigtable|Row Key、Hotspot、App Profile|
|5|Spanner|Consistency、TrueTime、Replication|
|6|Cloud SQL|HA、Read Replica、PITR、Migration|
|7|Pub/Sub|Ordering、Retention、ACK、Seek|
|8|Data Security|DLP、RLS、CLS、IAM|
|9|BigLake|External Table、Access Delegation|
|10|BigQuery AI|Embedding、Vector Search、RAG|

我认为你还需要特别加强一个能力：把单个产品的知识点连接成完整的数据架构。

例如，面试官问：

> Design a real-time data platform for an e-commerce company.

你应该能够自然地组织出：

```
                  E-commerce System
                         |
            +------------+------------+
            |                         |
            v                         v
        MySQL DB                 Application Events
            |                         |
            v                         v
        Datastream                 Pub/Sub
            |                         |
            |                         v
            |                      Dataflow
            |                         |
            +------------+------------+
                         |
                         v
                      BigQuery
                         |
              +----------+----------+
              |          |          |
              v          v          v
          Analytics   BigQuery ML  Vector Search
              |          |          |
              v          v          v
          Dashboard   Prediction    RAG
```

再考虑：

- Sensitive Data Protection：保护 PII。
    
- BigLake：访问外部数据。
    
- Cloud Composer：调度 ETL。
    
- IAM：控制访问权限。
    
- Partition / Clustering：优化查询成本。
    
- Cloud Monitoring：监控 Pipeline。
    

这才是 Professional Data Engineer 面试真正希望看到的系统性思维。


面试题目核心整理
# Google Cloud Data Engineer — 30 道模拟面试题库

15 个模块 · 30 道核心题 · 中英文对照 · 适合 Junior–Mid-level Data Engineer / Data Platform Engineer 面试

# Google Cloud Data Engineer – Mock Interview Question Bank

## Module 1. BigQuery Architecture and Fundamentals

### Q1. What is BigQuery, and how does its architecture separate storage and compute?

**中文解析**

这道题考察 BigQuery 的底层架构。

核心是理解 BigQuery 是一个 Serverless、fully managed 的数据仓库，并且采用 Storage 和 Compute 分离的架构。

回答时建议覆盖：

- Storage：负责持久化数据，底层采用分布式存储。
    
- Compute：负责执行 SQL 查询，由 Dremel 查询引擎处理。
    
- Colossus：Google 的分布式存储系统。
    
- Slots：代表 BigQuery 用于执行查询的计算资源。
    
- Storage 与 Compute 分离：可以独立扩展，多个计算工作负载可以访问同一份数据。
    

注意：不要把 BigQuery 描述成传统的单机数据库，也不要认为每个 Dataset 都需要独立的计算集群。

**Sample Answer (English)**

BigQuery is a fully managed, serverless data warehouse designed for large-scale analytical workloads.

One of its key architectural characteristics is the separation of storage and compute.

For storage, BigQuery uses Google's distributed storage infrastructure to store data independently of query execution resources.

For compute, BigQuery uses its distributed query engine, based on Dremel, to execute SQL queries in parallel. Slots represent the computational capacity available for query execution.

This separation provides several advantages. First, storage and compute can scale independently. Second, multiple teams can access the same datasets without managing separate database servers. Finally, users do not need to provision or maintain infrastructure.

This architecture makes BigQuery particularly suitable for large-scale OLAP workloads.

---

### Q2. What is the difference between a BigQuery Dataset, Table, View, and Materialized View?

**中文解析**

考察 BigQuery 的对象层次以及不同数据对象的存储和计算行为。

- Dataset：逻辑容器，管理表、视图和访问权限。
    
- Table：实际存储数据。
    
- View：保存 SQL 定义，查询时执行 SQL，不独立保存完整查询结果。
    
- Materialized View：预计算并存储部分查询结果，在满足条件时可以自动增量维护或由 BigQuery 优化查询。
    

面试中可以进一步比较：

|类型|是否保存数据|查询时是否执行原 SQL|
|---|---|---|
|Table|是|否|
|View|否|是|
|Materialized View|是，保存预计算结果|不一定，可能直接使用预计算结果|

不要将 Materialized View 和普通 View 混为一谈。

**Sample Answer (English)**

A Dataset is a logical container that organizes tables, views, and other BigQuery resources.

A Table stores actual data, while a View stores a SQL definition. When users query a standard view, BigQuery executes the underlying SQL against the referenced tables.

A Materialized View is different because it stores precomputed query results. BigQuery can maintain these results and use them to accelerate queries.

The main trade-off is between storage and query performance. Standard views avoid storing additional results, while materialized views can reduce query latency and processing costs for frequently executed queries.

I would use standard views for reusable query logic and materialized views when repeated analytical queries can benefit from precomputation.

---

## Module 2. BigQuery Performance and Cost Optimization

### Q3. How would you optimize a slow and expensive BigQuery query?

**中文解析**

这是 BigQuery 面试中的高频场景题。

建议按照以下顺序回答：

1. 检查 Query Execution Details，分析各 Stage 的耗时和数据处理量。
    
2. 减少扫描数据：避免 `SELECT *`，只读取需要的列。
    
3. 使用 Partitioning，减少需要扫描的分区。
    
4. 使用 Clustering，优化过滤和数据访问。
    
5. 减少大型 JOIN 的数据量，尽量在 JOIN 前过滤。
    
6. 避免不必要的重复计算，可以考虑 Materialized View 或中间结果表。
    
7. 检查 Shuffle、数据倾斜和高基数 JOIN。
    

需要知道：

- Partitioning：按照时间、整数范围或 ingestion time 等方式划分数据。
    
- Clustering：按照列值组织数据，提高过滤效率。
    
- `require_partition_filter`：强制查询提供分区过滤条件，防止无意中扫描大量数据。
    

**Sample Answer (English)**

I would start by analyzing the query execution details to identify the most expensive stages.

First, I would reduce the amount of data scanned by selecting only the required columns instead of using SELECT star.

Second, I would apply partitioning to reduce the number of partitions scanned. If the query frequently filters on specific columns, clustering could further improve data access efficiency.

Third, I would optimize joins by filtering large tables before joining them and checking whether there is data skew.

I would also look for repeated transformations that could be precomputed using materialized views or intermediate tables.

Finally, I would monitor bytes processed, slot consumption, execution time, and query costs to validate whether the optimization is effective.

---

### Q4. What are BigQuery Slots, and how do they affect query performance?

**中文解析**

这道题考察 BigQuery 的计算资源模型。

Slots 是 BigQuery 用于执行查询的虚拟计算单元。查询通常会被拆分为多个 Stage 和任务，分布式地使用计算资源。

需要理解：

- On-demand pricing：主要根据查询处理的数据量收费。
    
- Capacity-based pricing：通过购买或配置计算容量获得 Slots。
    
- Reservations：为不同项目或工作负载分配计算资源。
    
- Autoscaling：在支持的配置下动态调整计算容量。
    
- Slot contention：多个查询竞争计算资源，可能导致排队或执行时间增加。
    

注意：Slots 并不是 CPU 核心的简单等价物，也不是越多就一定能线性提升所有查询的速度。

**Sample Answer (English)**

Slots are virtual computational units used by BigQuery to execute queries.

When a query runs, BigQuery divides the workload into smaller execution tasks and distributes them across available slots.

The number of available slots can influence query concurrency and execution performance.

BigQuery provides different pricing models. Under on-demand pricing, users primarily pay based on the amount of data processed. Under capacity-based pricing, organizations allocate computational capacity through reservations.

Reservations are useful when different teams or workloads need resource isolation and predictable performance.

However, adding more slots does not guarantee linear performance improvements. Query structure, data distribution, shuffle operations, and bottlenecks also affect execution time.

---

## Module 3. Data Ingestion and Storage Write

### Q5. What is the difference between BigQuery Storage Write API and BigQuery Data Transfer Service?

**中文解析**

两者解决的问题不同。

**Storage Write API：**

- 用于程序化地向 BigQuery 写入数据。
    
- 适用于流式数据和高吞吐写入。
    
- 支持不同类型的 Write Streams。
    
- Committed Stream 配合 Offset 可以实现流级别的 exactly-once 写入语义。
    

**Data Transfer Service：**

- 用于定期从支持的数据源导入数据。
    
- 适合 SaaS 数据、Google Ads、Google Analytics 等来源的定时同步。
    
- 更偏向托管的数据导入任务。
    

面试重点：不要认为 Storage Write API 和 Data Transfer Service 是可以直接互换的服务。

**Sample Answer (English)**

The BigQuery Storage Write API is designed for programmatic and high-throughput data ingestion.

It supports streaming ingestion and different types of write streams. For example, committed streams combined with offsets can provide exactly-once write semantics within the stream.

BigQuery Data Transfer Service, on the other hand, is designed for managed and scheduled data ingestion from supported external sources.

For example, I would use Data Transfer Service to periodically import data from SaaS platforms, while I would use the Storage Write API for an application that continuously sends transaction events to BigQuery.

The choice depends on the ingestion pattern, latency requirements, and delivery guarantees.

---

### Q6. How would you migrate data from Cloud SQL to BigQuery?

**中文解析**

这是典型的数据库迁移和数据集成问题。

需要区分：

- Initial load：首次全量迁移。
    
- CDC：捕获增量变化。
    
- Datastream：Google Cloud 提供的 Serverless CDC 服务，可以捕获数据库变更。
    
- BigQuery：作为分析目标端。
    
- Dataflow：如果需要额外清洗、转换或复杂数据处理，可以加入 Pipeline。
    

典型架构：

`Cloud SQL → Datastream → BigQuery`

如果需要自定义转换：

`Cloud SQL → Datastream → Dataflow → BigQuery`

还要提到数据一致性、Schema Evolution、初始快照、删除事件以及监控。

**Sample Answer (English)**

I would use a combination of initial data loading and change data capture.

First, I would perform an initial snapshot of the Cloud SQL database and load the existing data into BigQuery.

Then, I would configure Datastream to capture ongoing database changes from the transaction logs.

Datastream can replicate inserts, updates, and deletes to BigQuery, allowing the analytical dataset to stay relatively up to date.

If additional transformations are required, I would introduce Dataflow between the CDC source and BigQuery.

I would also consider schema changes, duplicate events, transaction ordering, and monitoring to ensure data consistency.

For a production migration, I would validate row counts and business-level metrics before switching downstream analytical workloads.

---

## Module 4. Data Processing: Dataflow vs. Dataproc

### Q7. What is the difference between Dataflow and Dataproc?

**中文解析**

这是 GCP Data Engineer 必须掌握的服务选择题。

|对比|Dataflow|Dataproc|
|---|---|---|
|核心技术|Apache Beam|Spark、Hadoop 等|
|管理方式|Serverless 托管执行|托管集群|
|适用场景|Streaming、Batch Pipeline|Spark/Hadoop 工作负载|
|资源管理|Google 管理底层执行资源|需要配置集群和资源|
|代码迁移|需要使用 Beam 编程模型|适合已有 Spark/Hadoop 作业|

选择原则：

- 新建实时事件处理 Pipeline：优先考虑 Dataflow。
    
- 已有大量 Spark 作业：优先考虑 Dataproc。
    
- 需要灵活控制 Spark 集群：Dataproc 更合适。
    

注意：Dataflow 不是 Serverless Dataproc，它是 Apache Beam 的托管执行服务。

**Sample Answer (English)**

Dataflow and Dataproc are both data processing services, but they target different workloads.

Dataflow is a fully managed service for executing Apache Beam pipelines. It supports both batch and streaming processing and provides features such as autoscaling and dynamic work rebalancing.

Dataproc is a managed service for running open-source frameworks such as Apache Spark and Hadoop.

I would choose Dataflow when building a new streaming pipeline or when I need a serverless processing environment.

I would choose Dataproc when I already have Spark applications, need compatibility with the Hadoop ecosystem, or require more control over cluster configuration.

The decision depends on the existing technology stack, operational requirements, and workload characteristics.

---

### Q8. What are the differences between batch processing and stream processing?

**中文解析**

考察数据处理模式以及延迟、吞吐量、成本和一致性。

**Batch：**

- 数据先积累，再周期性处理。
    
- 更适合历史数据处理、离线 ETL 和大规模计算。
    
- 常见服务：BigQuery Scheduled Queries、Dataproc、Dataflow Batch。
    

**Streaming：**

- 数据持续进入 Pipeline。
    
- 适用于实时告警、实时风控和事件分析。
    
- 常见服务：Pub/Sub + Dataflow Streaming。
    

需要理解 Event Time、Processing Time、Watermark、Window、Late-arriving Data。

**Sample Answer (English)**

Batch processing handles data in predefined groups, usually on a scheduled basis.

It is suitable for historical data processing, periodic reporting, and large-scale transformations.

Stream processing handles events continuously as they arrive. It is useful for applications that require low-latency insights, such as fraud detection, monitoring, and real-time analytics.

In streaming systems, I also need to consider event time, processing time, windows, watermarks, and late-arriving data.

For example, I could use Dataproc or BigQuery for daily batch processing, while using Pub/Sub and Dataflow for real-time event processing.

The main trade-off is between processing latency, operational complexity, and cost.

---

## Module 5. Pub/Sub, CDC, and Real-Time Data

### Q9. How would you design a real-time data pipeline using Pub/Sub and Dataflow?

**中文解析**

典型架构：

`Application → Pub/Sub → Dataflow → BigQuery`

需要说明每个组件的职责：

- Application：产生事件。
    
- Pub/Sub：解耦生产者与消费者，提供消息缓冲。
    
- Dataflow：负责转换、聚合、过滤和窗口计算。
    
- BigQuery：提供分析查询能力。
    

面试官可能追问：

- 如何处理重复消息？
    
- 如何处理乱序消息？
    
- 如何处理迟到数据？
    
- 如何保证数据不丢失？
    
- 如何监控 Pipeline？
    

需要强调：Pub/Sub 的消息投递语义不等于整个 Pipeline 的端到端 exactly-once。

**Sample Answer (English)**

I would design the pipeline using Pub/Sub as the messaging layer, Dataflow as the processing engine, and BigQuery as the analytical destination.

Applications publish events to Pub/Sub topics. Pub/Sub decouples producers from consumers and provides buffering.

Dataflow consumes the messages, performs transformations, applies event-time windowing, and handles late-arriving events.

The processed data is then written to BigQuery.

To improve reliability, I would implement appropriate retry mechanisms, dead-letter handling, and idempotent processing where necessary.

I would also monitor message backlog, processing latency, failed records, and pipeline health.

This architecture provides scalability and allows producers and consumers to evolve independently.

---

### Q10. What is the difference between at-least-once and exactly-once processing?

**中文解析**

这是流处理系统的重要基础。

- At-most-once：最多处理一次，可能丢数据。
    
- At-least-once：至少处理一次，可能重复。
    
- Exactly-once：在特定系统边界和条件下，确保一次有效处理结果。
    

重点是区分不同层级：

1. 消息投递语义。
    
2. Pipeline 内部处理语义。
    
3. Sink 写入语义。
    
4. 业务层最终结果。
    

例如，Pub/Sub 可能发生重复投递；即使 Dataflow 提供处理保证，如果外部 Sink 不支持事务或幂等写入，也不能简单地说端到端 exactly-once。

Storage Write API 的 Committed Stream 配合 Offset 可以在相应写入边界提供 exactly-once 语义。

**Sample Answer (English)**

At-most-once means a message is processed zero or one time, so messages may be lost.

At-least-once means messages are guaranteed to be delivered at least once, but duplicates may occur.

Exactly-once aims to ensure that each event has one effective processing result.

However, exactly-once guarantees depend on the system boundary.

For example, a messaging service may redeliver messages, while a processing engine may provide exactly-once state management internally.

If the destination system does not support transactional or idempotent writes, duplicates may still appear in the final dataset.

Therefore, I would design the entire pipeline carefully, using checkpoints, offsets, idempotency keys, and appropriate sink guarantees.

---

## Module 6. BigLake and Data Lake Architecture

### Q11. What is BigLake, and how is it different from a traditional BigQuery external table?

**中文解析**

BigLake 的核心是统一分析访问与治理外部数据。

传统 External Table：

- 数据保存在 GCS 等外部存储。
    
- BigQuery 查询外部数据。
    
- 传统访问方式下，用户通常还需要具备底层数据访问权限。
    

BigLake：

- 支持对外部数据进行统一分析访问。
    
- 可以使用 Table Access Delegation，让 BigQuery 通过 Cloud Resource Connection 的服务账号访问底层数据。
    
- 有助于实现更集中化的数据访问控制。
    

注意：

BigLake 不会把外部 GCS 数据自动复制到 BigQuery 内部存储。访问能力和具体治理特性要结合使用的表类型与产品配置理解。

**Sample Answer (English)**

BigLake is designed to provide unified access and governance across data stored in different storage systems.

A traditional BigQuery external table allows users to query data stored outside BigQuery, such as files in Cloud Storage.

However, managing access to the underlying storage can become complicated.

BigLake supports capabilities such as table access delegation. With this approach, BigQuery can access the underlying data through a Cloud Resource Connection service account.

This helps centralize access management and reduces the need to grant every individual user direct access to the storage layer.

BigLake is especially useful when an organization wants to combine data lake storage with centralized analytical access and governance.

---

### Q12. How would you build a data lakehouse architecture on Google Cloud?

**中文解析**

这是综合架构题。

可以从三个层次回答：

1. Storage Layer：Cloud Storage 保存原始文件和数据湖数据。
    
2. Processing Layer：Dataflow / Dataproc 处理数据。
    
3. Analytical Layer：BigQuery 提供 SQL 分析能力。
    

进一步加入：

- BigLake：统一访问外部数据。
    
- Apache Iceberg：开放表格式，管理数据文件和表元数据。
    
- Dataplex：数据治理、发现、质量和管理。
    
- IAM：访问控制。
    
- Cloud Composer：工作流编排。
    

需要注意：Iceberg 表的 Catalog 管理、BigLake 表访问和 BigQuery 原生表是相关但不同的概念，不能简单视为同一个东西。

**Sample Answer (English)**

I would design the lakehouse architecture using Cloud Storage as the underlying data lake storage layer.

Raw data from different sources would first land in Cloud Storage.

Dataflow or Dataproc would process and transform the data. For open table formats, I could use Apache Iceberg to manage table metadata, snapshots, and schema evolution.

BigLake could provide unified analytical access to supported external data, while BigQuery would serve as the primary analytical engine.

For governance, I would use Dataplex and IAM to manage data discovery, quality, and access policies.

Finally, Cloud Composer could orchestrate the end-to-end workflows.

This architecture combines the flexibility of a data lake with the analytical capabilities of a data warehouse.

---

## Module 7. BigQuery AI, Vector Search, and RAG

### Q13. What is vector search, and how can you implement it in BigQuery?

**中文解析**

向量检索是语义搜索的重要技术。

流程：

1. 使用 Embedding Model 将文本转换成向量。
    
2. 将向量保存到 BigQuery。
    
3. 创建 Vector Index。
    
4. 使用 `VECTOR_SEARCH` 查找相似向量。
    
5. 根据相似度返回相关记录。
    

重点概念：

- Embedding：将文本、图片等转换为数值向量。
    
- COSINE：余弦相似度。
    
- IVF：一种近似最近邻检索方法。
    
- ANN：Approximate Nearest Neighbor。
    
- Vector Index：提高向量搜索效率。
    

注意：ANN 通常需要在查询速度、成本和召回率之间进行权衡。

**Sample Answer (English)**

Vector search is a technique used to find semantically similar items based on their numerical representations.

First, I would use an embedding model to convert text into vectors.

Then, I would store the embeddings in BigQuery and create a vector index to accelerate similarity searches.

When a user submits a query, I would generate an embedding for that query and use the VECTOR_SEARCH function to retrieve similar records.

BigQuery supports distance metrics such as cosine distance and indexing approaches such as IVF.

Approximate nearest neighbor search improves performance for large datasets, but it may introduce a trade-off between query speed and recall accuracy.

This approach is useful for semantic search, recommendation systems, and retrieval-augmented generation.

---

### Q14. How would you build a RAG application using Google Cloud services?

**中文解析**

RAG = Retrieval-Augmented Generation。

主要分为离线数据准备和在线查询两个阶段。

**Offline ingestion：**

`Documents → Document AI (if needed) → Chunking → Embedding → Vector Store`

**Online query：**

`User Query → Query Embedding → Vector Search → Retrieved Context → Gemini → Answer`

可以使用：

- Cloud Storage：文档存储。
    
- Document AI：OCR、文档解析和表格提取。
    
- Vertex AI Embeddings：生成向量。
    
- BigQuery Vector Search 或其他向量数据库：检索。
    
- Gemini：生成回答。
    

注意：Document AI 不是所有 PDF 都必须使用。如果文件本身已经是结构化文本，可以直接进行解析和 Chunking。

**Sample Answer (English)**

I would build the RAG application using separate ingestion and query workflows.

During ingestion, documents are stored in Cloud Storage. If the documents contain scanned images or complex layouts, I would use Document AI for OCR and document parsing.

Next, I would split the documents into smaller chunks and generate embeddings using a Vertex AI embedding model.

The embeddings and associated metadata would be stored in BigQuery or another vector database.

During query time, the user's question is converted into an embedding. The application retrieves relevant chunks using vector search and passes them to Gemini as additional context.

This approach helps the language model generate answers grounded in the organization's own documents.

---

## Module 8. Data Security, DLP, Encryption, and Sharing

### Q15. How would you protect sensitive data in BigQuery?

**中文解析**

这道题建议从多个安全层次回答。

- IAM：控制谁可以访问 Dataset、Table 和 Job。
    
- Row-level Security：使用 Row Access Policy 控制行级访问。
    
- Column-level Security：通过 Policy Tags 和 Data Catalog 相关能力控制敏感列。
    
- Dynamic Data Masking：根据用户权限动态隐藏部分字段。
    
- CMEK：使用 Customer-managed Encryption Keys。
    
- DLP：识别和保护敏感信息。
    
- Audit Logs：审计数据访问行为。
    

还要知道 BigQuery 的常见 IAM 角色：

- `roles/bigquery.user`
    
- `roles/bigquery.jobUser`
    
- `roles/bigquery.dataViewer`
    
- `roles/bigquery.dataEditor`
    

回答时强调 Least Privilege Principle。

**Sample Answer (English)**

I would apply a defense-in-depth approach to protect sensitive data in BigQuery.

First, I would use IAM to enforce the principle of least privilege.

Second, I would apply row-level access policies to restrict which records users can access.

For sensitive columns, I would use policy tags and column-level access controls. Dynamic data masking can also help hide sensitive values based on user permissions.

For encryption, I would rely on encryption at rest and consider customer-managed encryption keys when additional key control is required.

I would also use Cloud DLP to discover and classify sensitive information, and enable audit logging to monitor access patterns.

The goal is to protect data at multiple levels rather than relying on a single security mechanism.

---

### Q16. What is the difference between data masking, tokenization, and encryption?

**中文解析**

这道题考察数据安全基础概念。

- Encryption：使用密钥把明文转换为密文，需要密钥才能解密。
    
- Masking：隐藏数据的一部分或全部，例如 `123456` → `******`。
    
- Tokenization：使用 Token 替代敏感值，映射关系单独保存。
    
- De-identification：降低数据与具体个人关联的能力，是更广泛的数据处理目标。
    
- Re-identification：在特定条件下重新识别个人身份。
    

DLP 相关方法还包括：

- Inspection：识别敏感数据。
    
- Discovery：发现敏感数据所在位置。
    
- De-identification：脱敏。
    
- Risk Analysis：评估重新识别风险。
    

注意：Masking 不等于 Encryption，也不一定能满足所有匿名化要求。

**Sample Answer (English)**

Encryption, masking, and tokenization are different techniques for protecting sensitive information.

Encryption transforms plaintext into ciphertext using a cryptographic key. The original data can be recovered by authorized users with the appropriate key.

Masking hides part or all of a value. For example, a credit card number may be displayed with only its last four digits visible.

Tokenization replaces sensitive values with randomly generated tokens. The original values are stored separately in a secure mapping system.

I would choose the technique based on the business requirement.

For example, encryption is useful when authorized systems need to recover the original data, while masking is suitable for limiting exposure in analytical or reporting environments.

---

## Module 9. Database Selection: Cloud SQL, Spanner, Bigtable, and Firestore

### Q17. How would you choose between Cloud SQL, Spanner, Bigtable, and BigQuery?

**中文解析**

这是一道经典的数据库选型题。

|产品|核心场景|
|---|---|
|Cloud SQL|传统关系型 OLTP|
|AlloyDB|高性能 PostgreSQL 兼容数据库|
|Spanner|全球分布式、强一致关系型数据库|
|Bigtable|高吞吐、低延迟 Wide-column 数据库|
|Firestore|文档型 NoSQL 数据库|
|BigQuery|大规模 OLAP 分析|

关键是根据访问模式选型，而不是只比较数据规模。

例如：

- 电商订单事务：Cloud SQL 或 Spanner。
    
- 全球分布式订单系统：Spanner。
    
- IoT 时间序列数据：Bigtable。
    
- 用户文档和应用状态：Firestore。
    
- 数十亿条记录的离线分析：BigQuery。
    

**Sample Answer (English)**

I would choose the database based on the workload, consistency requirements, data model, and scalability requirements.

Cloud SQL is suitable for traditional relational workloads that require SQL and transactional consistency.

Spanner is a better choice for globally distributed applications that need strong consistency and horizontal scalability.

Bigtable is designed for high-throughput, low-latency access to wide-column data, such as IoT events and time-series workloads.

Firestore is useful for document-oriented application data.

BigQuery is designed for analytical workloads rather than high-frequency transactional updates.

The most important factor is understanding the application's access patterns before selecting the database.

---

### Q18. What is the difference between Cloud SQL High Availability and cross-region read replicas?

**中文解析**

考察数据库高可用和灾备能力。

**High Availability：**

- 通常通过主实例与备用实例实现。
    
- 主要应对区域内或实例级故障。
    
- 发生故障时进行自动故障切换。
    
- 重点是提升可用性。
    

**Cross-region Read Replica：**

- 在另一个 Region 创建只读副本。
    
- 适用于异地读取和灾难恢复设计。
    
- 通常存在复制延迟。
    
- 不等于自动完成主实例跨区域故障切换。
    

另外需要理解：

- PITR：Point-in-Time Recovery。
    
- Binary Logs：可以支持特定数据库引擎的时间点恢复或迁移场景。
    
- Cascading Replication：通过副本继续复制到其他副本，降低主实例复制压力，但要结合具体产品支持情况。
    

**Sample Answer (English)**

Cloud SQL High Availability and cross-region read replicas solve different problems.

High Availability is mainly designed to protect against instance or zonal failures within a region.

It typically uses a primary instance and a standby instance. If the primary instance becomes unavailable, Cloud SQL can perform an automatic failover.

A cross-region read replica is designed for read scaling and disaster recovery across regions.

However, replication is asynchronous in common configurations, so there may be replication lag.

For disaster recovery, I would define clear recovery point objectives and recovery time objectives, and test the promotion and failover procedures regularly.

I would also consider backups and point-in-time recovery as additional protection mechanisms.

---

## Module 10. GCP Services and Orchestration

### Q19. What is the difference between Cloud Composer, Cloud Scheduler, and Dataproc Workflow Templates?

**中文解析**

三者都可能涉及任务调度，但定位不同。

- Cloud Composer：托管 Apache Airflow，适合复杂 DAG、跨服务编排、依赖管理、重试和监控。
    
- Cloud Scheduler：简单的 Cron 定时触发器。
    
- Dataproc Workflow Templates：定义 Dataproc 工作流，包括作业及相关集群配置，适合 Dataproc 作业编排。
    

举例：

每天凌晨触发一套复杂 ETL，包含 Dataproc、Dataflow、BigQuery 和数据质量检查，优先考虑 Composer。

每天固定时间调用一个 HTTP Endpoint，可以考虑 Cloud Scheduler。

**Sample Answer (English)**

Cloud Composer, Cloud Scheduler, and Dataproc Workflow Templates serve different orchestration purposes.

Cloud Composer is a managed Apache Airflow service. It is suitable for complex workflows involving multiple services, dependencies, retries, and monitoring.

Cloud Scheduler is a lightweight scheduling service that triggers jobs based on cron expressions. It is useful for simple periodic tasks.

Dataproc Workflow Templates are designed specifically for defining and running Dataproc workflows, including Spark or Hadoop jobs and associated cluster configurations.

For a complex end-to-end data pipeline, I would generally use Cloud Composer.

For a simple scheduled HTTP request, Cloud Scheduler would be sufficient.

---

### Q20. What is the difference between GCS Storage Classes, and how would you optimize storage costs?

**中文解析**

GCS 常见存储类型：

- Standard：频繁访问。
    
- Nearline：低频访问。
    
- Coldline：更低频访问。
    
- Archive：长期归档。
    

优化思路：

1. 根据访问频率选择 Storage Class。
    
2. 使用 Object Lifecycle Management 自动转移或删除对象。
    
3. 使用合适的压缩格式，例如 Parquet。
    
4. 清理不再需要的临时数据。
    
5. 结合访问模式和 Retrieval Cost 评估成本。
    

注意：低频存储通常有最低存储期限和取回费用，不能简单认为 Archive 永远最便宜。

**Sample Answer (English)**

Google Cloud Storage provides several storage classes designed for different access patterns.

Standard is suitable for frequently accessed data.

Nearline, Coldline, and Archive are designed for progressively less frequent access.

To optimize costs, I would first analyze how frequently different datasets are accessed.

For example, recent operational data could remain in Standard storage, while historical data could move to Coldline or Archive.

I would use Object Lifecycle Management to automate storage transitions and deletion policies.

I would also consider retrieval costs, minimum storage durations, and data access requirements before selecting a storage class.

The goal is to optimize total cost rather than simply choosing the cheapest storage class.

---

## Module 11. Data Modeling: BigQuery and Bigtable

### Q21. What is the difference between Star Schema and Snowflake Schema?

**中文解析**

这是数据仓库建模基础。

**Star Schema：**

- 一个 Fact Table。
    
- 多个 Dimension Tables。
    
- 维度表通常反规范化。
    
- 查询时 JOIN 相对简单。
    

**Snowflake Schema：**

- 维度表进一步规范化。
    
- 减少冗余。
    
- 需要更多 JOIN。
    

BigQuery 通常适合利用 Nested and Repeated Fields 减少不必要的 JOIN。

需要理解：

- STRUCT：嵌套字段结构。
    
- ARRAY：数组。
    
- UNNEST：展开数组。
    
- ARRAY_AGG：聚合成数组。
    

**Sample Answer (English)**

A Star Schema consists of a central fact table connected to multiple dimension tables.

The dimension tables are usually denormalized, which makes analytical queries simpler and reduces the number of joins.

A Snowflake Schema further normalizes dimension tables into multiple related tables.

This reduces data redundancy but can increase query complexity.

In BigQuery, I would often consider a denormalized design using nested and repeated fields.

For example, I could store an order together with its line items in a nested structure.

This can reduce joins and improve analytical query performance, depending on the access pattern.

---

### Q22. How would you design a Bigtable row key to avoid hotspotting?

**中文解析**

Bigtable 的数据分布与 Row Key 设计高度相关。

如果所有新数据都使用递增时间戳作为 Row Key 前缀，写入可能集中到少数 Tablet，导致 Hotspotting。

解决方式：

- 使用 Hash Prefix。
    
- 将业务 ID 与时间戳组合。
    
- 设计能够分散写入的 Key。
    
- 同时考虑查询模式，避免为了分散写入而失去有效查询能力。
    

还要理解：

- Tablet：Bigtable 自动管理和分布数据的基本分片单元。
    
- Key Visualizer：分析访问模式和热点。
    
- Garbage Collection Policy：管理旧版本或过期数据。
    
- App Profiles：定义应用如何访问 Bigtable，例如路由策略。
    

**Sample Answer (English)**

Bigtable distributes data based on row key ranges.

If I use monotonically increasing timestamps as the beginning of row keys, new writes may concentrate on a small number of tablets.

This can create hotspots and reduce write performance.

To avoid this, I would design row keys that distribute writes more evenly.

For example, I could prepend a hash or bucket identifier to the business key and then append the timestamp.

However, row key design must also consider query patterns.

I would use Key Visualizer to identify hotspots and analyze access patterns before changing the key design.

The goal is to balance write distribution and efficient reads.

---

## Module 12. Data Governance, Metadata, and Operations

### Q23. What is the difference between Dataplex, Dataproc Metastore, and Dataproc Persistent History Server?

**中文解析**

这道题容易混淆，因为三个服务都与数据平台管理有关。

- Dataplex：数据治理、数据发现、数据质量和数据管理。
    
- Dataproc Metastore：托管 Hive Metastore 服务，管理 Hive/Spark 相关的表和 Schema 元数据。
    
- Dataproc Persistent History Server：保存和查看 Spark Event Logs，帮助分析历史 Spark 作业执行情况。
    

特别注意：

Persistent History Server 不是 Metastore 的替代品；Metastore 也不负责保存整个 Spark 集群的业务数据。

**Sample Answer (English)**

These services serve different purposes.

Dataplex focuses on data governance, discovery, metadata management, and data quality across an organization's data assets.

Dataproc Metastore is a managed Hive Metastore service. It stores metadata such as table definitions, schemas, and partition information for compatible processing engines.

Dataproc Persistent History Server is used to retain and inspect Spark application event logs.

It helps engineers investigate historical Spark jobs, including execution stages and performance issues.

In short, Dataplex is primarily a governance service, Dataproc Metastore manages table metadata, and Persistent History Server supports Spark job troubleshooting.

---

### Q24. How would you implement data quality checks in a production pipeline?

**中文解析**

面试官希望听到的不只是 SQL 检查，还包括质量体系。

可以从以下维度回答：

- Completeness：数据是否缺失。
    
- Uniqueness：主键是否重复。
    
- Validity：数据是否符合业务规则。
    
- Consistency：不同表之间是否一致。
    
- Timeliness：数据是否按时到达。
    
- Accuracy：数据是否与业务真实情况一致。
    

实施方式：

- BigQuery SQL assertions。
    
- dbt tests：`not_null`、`unique`、`accepted_values`、relationships。
    
- Dataflow pipeline validation。
    
- Dataplex data quality。
    
- Airflow / Composer 质量检查任务。
    

还应有失败处理策略：告警、隔离错误数据、重试或阻止下游任务。

**Sample Answer (English)**

I would implement data quality checks at multiple stages of the pipeline.

First, I would validate the incoming data for schema compatibility and required fields.

Then, I would apply checks for completeness, uniqueness, validity, and consistency.

For example, I could use dbt tests to validate primary key uniqueness and non-null constraints.

For more complex rules, I could implement SQL assertions or custom validation logic in Dataflow.

I would also monitor data freshness and record counts.

If a critical validation fails, I would prevent downstream jobs from running and send an alert.

This approach helps detect data issues early and improves the reliability of analytical datasets.

---

## Module 13. Performance, Reliability, and Cost Optimization

### Q25. How would you troubleshoot a BigQuery pipeline that suddenly becomes slower?

**中文解析**

建议按照系统化排查思路回答。

1. 确认是否发生数据量变化。
    
2. 检查 Query History 和 Execution Details。
    
3. 查看 Slot contention 和排队情况。
    
4. 检查分区、聚类以及查询计划变化。
    
5. 检查 JOIN 是否产生数据膨胀。
    
6. 检查上游数据延迟或 Schema 变化。
    
7. 对比近期代码、配置和数据分布变化。
    

不要一开始就简单地增加 Slots。需要先找到瓶颈。

**Sample Answer (English)**

I would start by comparing the current workload with previous successful executions.

First, I would check whether the input data volume has increased.

Then, I would inspect query execution details to identify expensive stages, shuffle operations, and potential data skew.

I would also check slot utilization and whether queries are experiencing resource contention.

Next, I would review recent SQL changes, partition filters, clustering configuration, and upstream schema changes.

If the issue is caused by data skew, I would optimize the join strategy or data distribution.

If it is caused by resource contention, I would review workload reservations and concurrency.

I would validate the fix using execution time, bytes processed, and slot consumption.

---

### Q26. How would you design a disaster recovery strategy for a data platform?

**中文解析**

重点是 RPO 和 RTO。

- RPO：最多可以接受丢失多少数据。
    
- RTO：最多可以接受多长恢复时间。
    

设计时考虑：

- BigQuery：数据集和跨区域备份策略，结合复制能力、导出和恢复需求。
    
- GCS：双区域或多区域存储、版本控制和生命周期管理。
    
- Cloud SQL：备份、PITR、跨区域只读副本及恢复演练。
    
- Spanner：多区域配置和复制。
    
- Pub/Sub：消息保留和重放策略。
    
- Dataflow：Pipeline 配置、部署和状态恢复能力。
    
- Terraform：基础设施重建。
    

不要只说“定期备份”，需要解释如何满足 RPO 和 RTO。

**Sample Answer (English)**

I would start by defining the business recovery requirements.

RPO defines the maximum acceptable amount of data loss, while RTO defines the maximum acceptable recovery time.

For storage, I would consider regional or multi-regional configurations and appropriate backup strategies.

For Cloud SQL, I would enable automated backups and point-in-time recovery, and consider cross-region replicas where necessary.

For BigQuery, I would evaluate the available replication, backup, and recovery options based on dataset location and business requirements.

I would also keep infrastructure definitions in Terraform so that critical services can be recreated consistently.

Finally, I would regularly test disaster recovery procedures rather than relying only on documented backup configurations.

---

## Module 14. End-to-End Data Architecture and System Design

### Q27. Design an end-to-end data platform for an e-commerce company.

**中文解析**

这是最值得重点准备的综合架构题。

建议按数据源、接入、处理、存储、分析、治理、监控回答。

一个可行方案：

1. MySQL / Cloud SQL：订单、用户、商品交易数据。
    
2. Datastream：CDC。
    
3. Pub/Sub：应用事件。
    
4. Dataflow：实时清洗、转换和聚合。
    
5. Cloud Storage：原始数据湖。
    
6. BigQuery：数据仓库。
    
7. dbt：数据转换、模型管理和测试。
    
8. Composer：工作流编排。
    
9. Dataplex / IAM / DLP：治理和安全。
    
10. Looker：BI 分析。
    

面试官会继续追问：如何处理重复、乱序、Schema Evolution、数据质量、成本、故障恢复。

**Sample Answer (English)**

I would design the platform using a combination of batch and streaming architectures.

For transactional data, I would use Cloud SQL as the source system and Datastream to capture database changes through CDC.

For application events, I would publish messages to Pub/Sub.

Dataflow would process streaming events, perform transformations, and write the results to BigQuery.

I would also land raw data in Cloud Storage to maintain a historical data lake.

In BigQuery, I would build analytical models using dbt, including staging, intermediate, and mart layers.

Cloud Composer would orchestrate scheduled workflows and dependencies.

For governance, I would use IAM, Dataplex, and DLP.

Finally, I would implement monitoring, data quality checks, and alerting to ensure the platform is reliable and scalable.

---

### Q28. How would you handle schema evolution in a data pipeline?

**中文解析**

Schema Evolution 指源数据结构发生变化时，Pipeline 如何保持兼容。

常见变化：

- 新增字段。
    
- 删除字段。
    
- 字段类型改变。
    
- 字段重命名。
    
- Nested Schema 变化。
    

建议回答：

- 使用 Schema Registry 或集中化 Schema 管理机制。
    
- 优先支持向后兼容的变化，例如增加可空字段。
    
- 对不兼容变更进行版本控制。
    
- 在 Pipeline 中做 Schema Validation。
    
- 保留原始数据，方便重新处理。
    
- 更新 BigQuery Schema 和下游 dbt Models。
    

注意：不要认为所有 Schema Change 都能自动、安全地被下游系统接受。

**Sample Answer (English)**

I would manage schema evolution through explicit schema validation and version control.

For compatible changes, such as adding nullable fields, I would update the schema and ensure downstream transformations can handle the new fields.

For breaking changes, such as changing a field's data type or removing a required field, I would introduce a new schema version and coordinate the migration with downstream consumers.

I would also preserve raw data so that it can be reprocessed if necessary.

In a production environment, I would monitor schema changes and run automated data quality tests before allowing the updated data to reach downstream analytical models.

---

## Module 15. Troubleshooting and Architectural Trade-offs

### Q29. Your Dataflow pipeline is experiencing high latency. How would you investigate it?

**中文解析**

这道题考察流处理性能排查。

排查方向：

1. Pub/Sub backlog 是否持续增长。
    
2. Dataflow worker 数量是否不足。
    
3. 是否存在数据倾斜。
    
4. 是否有昂贵的外部 API 调用。
    
5. 是否使用了不合适的 Window。
    
6. 是否存在频繁 Shuffle。
    
7. 是否发生频繁的 Worker Restart。
    
8. Sink 写入是否成为瓶颈。
    

还要区分：

- Event Time。
    
- Processing Time。
    
- Watermark。
    
- System Lag。
    

不要只看 Pipeline 总耗时，应该定位具体 Stage 的瓶颈。

**Sample Answer (English)**

I would first determine whether the latency is caused by message ingestion, data processing, or destination writes.

I would check the Pub/Sub backlog to see whether messages are accumulating before reaching Dataflow.

Next, I would inspect Dataflow metrics, including system lag, worker utilization, throughput, and processing time.

I would also investigate data skew, expensive transformations, external API calls, and shuffle operations.

If the workers are under-provisioned, I would evaluate autoscaling and worker configuration.

If the destination is the bottleneck, I would optimize the write strategy or increase the destination's ingestion capacity.

I would use pipeline metrics to identify the actual bottleneck before changing infrastructure.

---

### Q30. How would you choose between ETL and ELT?

**中文解析**

这是数据工程基础架构选择题。

**ETL：**

Extract → Transform → Load

数据先在外部计算引擎完成转换，再写入目标系统。

适合：

- 需要在进入目标系统前清洗数据。
    
- 需要复杂预处理。
    
- 对数据安全和敏感字段处理有前置要求。
    

**ELT：**

Extract → Load → Transform

先把原始数据加载到目标系统，再利用目标系统的计算能力完成转换。

适合：

- BigQuery 等云数据仓库。
    
- 大规模 SQL 转换。
    
- 希望保留原始数据并支持多种下游分析。
    

现代云数据平台经常采用 ELT，也可以将两者结合。

**Sample Answer (English)**

ETL stands for Extract, Transform, and Load. Data is transformed before it is loaded into the target system.

ELT stands for Extract, Load, and Transform. Raw data is first loaded into the target platform, and transformations are performed afterward.

For a cloud data warehouse such as BigQuery, I would often prefer ELT because BigQuery provides scalable SQL processing capabilities.

It also allows us to preserve raw data and build multiple analytical models from the same source.

However, ETL may be more suitable when data requires complex preprocessing or sensitive information must be removed before loading.

In practice, I would often combine both approaches depending on the data source and business requirements.

---

# Final Interview Preparation Checklist

## A. 必须熟练掌握的 10 个主题

1. BigQuery architecture: storage, compute, slots.
    
2. BigQuery optimization: partitioning, clustering, query execution.
    
3. Dataflow vs. Dataproc.
    
4. Pub/Sub architecture and delivery semantics.
    
5. Datastream and CDC.
    
6. Cloud SQL vs. Spanner vs. Bigtable.
    
7. BigLake and lakehouse architecture.
    
8. IAM, DLP, encryption, and data governance.
    
9. Cloud Composer and workflow orchestration.
    
10. End-to-end data platform design.
    

## B. 建议使用的英文回答结构

回答技术问题时，可以采用以下模板：

**1. Definition**

First, explain what the technology or concept is.

**2. How it works**

Then, explain its underlying architecture or working mechanism.

**3. When to use it**

Describe the scenarios where you would choose this solution.

**4. Trade-offs**

Finally, discuss its limitations, costs, or alternatives.

例如：

"I would choose Dataflow for this use case because it provides a managed execution environment for Apache Beam pipelines. It supports both batch and streaming workloads, and it offers autoscaling capabilities. However, if the existing application is heavily based on Spark, Dataproc might be a more practical choice."

## C. 面试时需要避免的回答问题

- 只解释产品定义，不说明使用场景。
    
- 只说某个服务性能好，不解释为什么。
    
- 把 at-least-once 和 exactly-once 混为一谈。
    
- 把 Dataflow 描述成 Serverless Dataproc。
    
- 把 BigLake 描述成会自动复制 GCS 数据的服务。
    
- 把 Spanner、Cloud SQL 和 Bigtable 当成可以互换的数据库。
    
- 只说使用 Partitioning，却无法解释 Partition Pruning。
    
- 只说使用 CDC，却不考虑重复、乱序和删除事件。
    
- 设计架构时忽略 IAM、监控、成本和故障恢复。
    

**Preparation Strategy:** 不要背诵 30 道答案。先练习用英文独立回答每道题，再用中文检查是否解释了原理、适用场景和 trade-offs。对于 Q27–Q29 这类综合场景题，尤其要练习画出架构图并解释每个组件的职责。

## 建议的练习顺序

30 道题中，我认为最值得优先练习的是：

|优先级|题目|原因|
|---|---|---|
|最高|Q3、Q4|BigQuery 性能和计算资源|
|最高|Q7、Q9、Q10|Dataflow、Pub/Sub、Exactly-once|
|最高|Q6、Q27|CDC 和端到端架构|
|较高|Q17、Q18|数据库选型和高可用|
|较高|Q15、Q16|数据安全和治理|
|较高|Q25、Q29|生产环境问题排查|