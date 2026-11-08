# Google Cloud Data Governance、Dataplex 与 Knowledge Catalog 知识整理

## 1. Data Governance 的核心概念

Data Governance（数据治理）是对企业数据进行统一管理，确保数据具备以下属性：

- Discoverability（可发现性）：用户能够找到需要的数据。
    
- Understandability（可理解性）：用户知道数据的含义、来源和用途。
    
- Quality（质量）：数据准确、完整、一致。
    
- Security（安全性）：敏感数据得到保护，访问权限受到控制。
    
- Ownership（责任归属）：明确数据负责人和维护团队。
    
- Compliance（合规性）：满足隐私保护和监管要求。
    

数据治理不只是给数据添加标签，还包括数据目录、元数据管理、数据质量、访问控制、数据血缘和生命周期管理。

面试问题

1. What is data governance, and why is it important?
    
2. What is the difference between data governance and data management?
    
3. How do you ensure data discoverability, quality, and security in a data lake?
    
4. What is metadata management, and how does it support data governance?
    

## 2. Dataplex 与 Knowledge Catalog

### 2.1 Dataplex 的定位

Dataplex 是 Google Cloud 用于统一管理、组织和治理分布式数据资产的服务。它帮助企业管理分散在 BigQuery、Cloud Storage 等位置的数据，并提供数据发现、元数据管理、数据质量和治理能力。

可以从三个层面理解：

|层面|主要职责|
|---|---|
|Data organization|组织和管理数据资产|
|Metadata management|管理数据的描述信息、结构、标签和血缘|
|Data governance|支持数据质量、数据发现、权限和治理|

需要区分：Dataplex 的 lake、zone、asset 是数据组织模型；Knowledge Catalog 则侧重元数据目录、搜索和治理能力。两者相关，但不是同一层级的概念。

### 2.2 Knowledge Catalog 的定位

Knowledge Catalog 是 Google Cloud 的数据与 AI 资产元数据管理和发现能力体系。它帮助用户回答：

- What is this data? —— 数据是什么？
    
- Where is the data? —— 数据在哪里？
    
- Who owns the data? —— 谁负责维护？
    
- How can I discover it? —— 如何搜索和发现？
    
- What does this column mean? —— 字段的业务含义是什么？
    

可以将它理解为企业数据资产的统一目录。

### 2.3 Dataplex Lake、Zone 和 Asset

Dataplex 传统的 Lake 组织模型主要包含三个层级：

Lake

逻辑上的数据管理边界

Raw Zone

原始数据区域

Curated Zone

整理后的数据区域

Assets

实际的数据资源，例如 Cloud Storage bucket、BigQuery dataset 或 table

Lake： 用于组织一组相关的数据资产，提供逻辑管理边界。

Zone： Lake 内部的数据分类区域，常见类型包括：

- Raw zone：保存原始数据，通常尽量保留源数据格式和内容，允许数据结构尚未完全规范化。
    
- Curated zone：保存经过清洗、转换、验证的数据，通常面向分析和下游应用。
    

Asset： Zone 关联的实际数据资源，例如 Cloud Storage bucket 或 BigQuery dataset。

注意：Raw 和 Curated 是数据组织和治理上的逻辑分类，不代表所有 Raw 数据都没有 schema，也不代表 Curated 数据一定只能是 BigQuery table。

### 2.4 Lake 的删除顺序

删除 Lake 时，需要先解除其与底层数据资产的关联，再删除内部 Zone，最后删除 Lake。

正确顺序：

1. Detach assets from zones。
    
2. Delete zones。
    
3. Delete lake。
    

注意：删除 Dataplex 中的 Lake、Zone 或 Asset 管理关系，不应直接理解为删除底层 Cloud Storage 文件或 BigQuery 表。管理资源和底层数据资源的生命周期需要区分。

面试问题

1. What are Dataplex lakes, zones, and assets?
    
2. What is the difference between a raw zone and a curated zone?
    
3. How would you organize a data lake using Dataplex?
    
4. What happens when you delete a Dataplex lake?
    
5. Does deleting a Dataplex asset necessarily delete the underlying BigQuery table or Cloud Storage data?
    

## 3. Metadata、Aspect Type 与 Aspect

### 3.1 Metadata 的作用

Metadata（元数据）是描述数据的数据。

例如，一个 BigQuery table 除了存储业务数据，还可以记录：

- Table name：表名。
    
- Schema：字段名称和数据类型。
    
- Description：表和字段的业务含义。
    
- Owner：数据负责人。
    
- Classification：数据敏感等级。
    
- Lineage：数据来源和转换关系。
    
- Quality information：数据质量信息。
    

元数据能够帮助用户理解、发现、管理和保护数据。

### 3.2 Aspect Type 和 Aspect

Knowledge Catalog 使用 Aspect 机制为数据资产附加结构化元数据。

Aspect Type：定义元数据的模板或结构。

它规定一个 Aspect 可以包含哪些字段、字段类型以及相关约束。

Aspect：根据 Aspect Type 创建的具体元数据实例。

例如，定义一个用于描述敏感数据的 Aspect Type：

```
AspectType: DataClassification

Fields:
  - name: sensitivity
    type: string
  - name: protected
    type: boolean
  - name: owner
    type: string
```

然后将 Aspect 附加到具体的数据资产：

```
Aspect: DataClassification

sensitivity: HIGH
protected: true
owner: Security Team
```

可以理解为：

- Aspect Type = 数据标签的模板。
    
- Aspect = 某个具体数据资产上的标签实例。
    

### 3.3 Aspect 可以附加在哪些对象上？

Aspect 可以用于描述数据资产及其组成部分，例如：

- BigQuery table。
    
- BigQuery column。
    
- 其他受支持的数据资源及其元数据对象。
    

例如，BigQuery 中有一个 `customers` 表：

```
customers
│
├── id
│   └── INT64
│
├── email
│   └── STRING
│
└── phone
    └── STRING
```

可以给整个 table 添加业务元数据：

```
Table Aspect
├── Owner = Customer Data Team
└── Domain = Customer
```

也可以给 `email` column 添加敏感信息：

```
Column Aspect
├── Protected = true
├── Sensitivity = HIGH
└── Owner = Security Team
```

注意：Aspect 是元数据，不会自动改变 BigQuery column 的数据类型，也不会自动对数据进行加密或屏蔽。要实现访问限制、数据脱敏等安全效果，还需要结合 BigQuery IAM、Policy Tags、Data Policies 等机制。

### 3.4 Aspect 与 BigQuery Schema 的区别

|比较|BigQuery Schema|Aspect|
|---|---|---|
|主要用途|定义数据结构|描述数据资产|
|典型内容|字段名、类型、NULLABLE|Owner、Sensitivity、业务分类|
|是否影响数据存储结构|是|否|
|是否直接实现访问控制|否|否，通常需要配合安全策略|

面试问题

1. What is metadata, and why is it important in data governance?
    
2. What is the difference between an Aspect Type and an Aspect?
    
3. Can you attach metadata to a BigQuery column?
    
4. What is the difference between BigQuery schema and metadata?
    
5. Does adding a sensitivity Aspect automatically protect sensitive data?
    

## 4. Data Quality（数据质量）

### 4.1 Data Quality 的核心目标

Data Quality 用于判断数据是否满足预期规则，确保数据能够被可靠地用于分析、报表和机器学习。

常见的数据质量维度：

|维度|含义|示例|
|---|---|---|
|Completeness|完整性|`id` 不能为 NULL|
|Validity|有效性|email 必须符合格式|
|Uniqueness|唯一性|`customer_id` 不能重复|
|Consistency|一致性|不同表中的 customer 信息一致|
|Accuracy|准确性|金额与业务系统记录一致|
|Timeliness|时效性|数据必须在规定时间内到达|

### 4.2 Knowledge Catalog 的 Data Quality

Knowledge Catalog 支持通过 Data Quality Scan 对 BigQuery 表执行质量检查。

可以使用：

- Built-in rules：内置规则，例如非空、唯一性、正则表达式、集合匹配。
    
- Custom SQL rules：自定义 SQL 规则，处理复杂业务校验。
    
- Reusable rules：复用已有的数据质量规则或规则模板。
    
- Data quality results：保存和查看质量检查结果。
    

典型流程：

定义 Data Quality Rules

创建 Data Quality Scan / Task

执行规则，扫描 BigQuery 数据

生成质量检查结果和失败记录

查看结果、监控质量、定位问题

### 4.3 CloudDQ YAML Specification

你记录的 CloudDQ YAML 属于 Knowledge Catalog 传统 Data Quality Task 的配置方式。

YAML 主要包含四个部分：

|配置项|作用|
|---|---|
|`rules`|定义需要执行的规则|
|`row_filters`|通过 SQL 条件筛选需要检查的数据|
|`rule_bindings`|将规则和过滤条件绑定到具体表|
|`rule_dimensions`|定义允许使用的数据质量维度|

例如，定义两个规则：

```
rules:
  - nonNullExpectation: {}
    column: id
    dimension: COMPLETENESS
    threshold: 1

  - regexExpectation:
      regex: '^[^@]+[@]{1}[^@]+$'
    column: email
    dimension: CONFORMANCE
    ignoreNull: true
    threshold: 0.85
```

含义：

Rule 1：检查 id 是否为空

```
nonNullExpectation: {}
column: id
dimension: COMPLETENESS
threshold: 1
```

- `nonNullExpectation`：要求字段非空。
    
- `column: id`：检查 id 字段。
    
- `dimension: COMPLETENESS`：属于完整性检查。
    
- `threshold: 1`：要求满足规则的比例达到 100%。
    

Rule 2：检查 email 格式

```
regexExpectation:
  regex: '^[^@]+[@]{1}[^@]+$'
column: email
dimension: CONFORMANCE
ignoreNull: true
threshold: 0.85
```

- `regexExpectation`：使用正则表达式检查字段。
    
- `ignoreNull: true`：空值不参与这条规则的判断。
    
- `threshold: 0.85`：要求至少 85% 的被检查记录符合规则。
    

注意：你记录的 YAML 片段只展示了 `rules` 和 `postScanActions`，并没有展示完整的四个配置部分。`postScanActions.bigqueryExport` 用于将扫描结果导出到 BigQuery，不是上述四个核心规则配置部分。

另外，CloudDQ YAML 属于传统 Data Quality Task 的配置方式。Google 当前文档将这类任务标记为 legacy offering，并建议新项目优先考虑内置的 Auto Data Quality 能力。两者不要混为一谈。

![](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com&sz=32)

Google Cloud Documentation

+1

### 4.4 Data Quality 与 Data Profiling 的区别

|比较|Data Profiling|Data Quality|
|---|---|---|
|目标|了解数据特征|判断数据是否符合规则|
|主要问题|数据是什么样的？|数据是否正确？|
|示例|NULL 比例、唯一值数量、数值分布|id 非空、email 格式正确|
|结果|统计信息和数据特征|规则通过率、失败记录|

面试问题

1. What are the key dimensions of data quality?
    
2. What is the difference between data profiling and data quality?
    
3. How would you validate the completeness and uniqueness of a BigQuery table?
    
4. How do you implement custom data quality rules?
    
5. What is the difference between built-in data quality rules and custom SQL rules?
    
6. How would you handle records that fail data quality checks?
    

## 5. Apache Iceberg：ACID、Snapshot 与 Pruning

### 5.1 Iceberg 的定位

Apache Iceberg 是一种开放的 Lakehouse Table Format，用于管理数据湖中的大型分析表。

它解决的问题包括：

- 如何管理表的元数据和数据文件？
    
- 如何保证并发读写的一致性？
    
- 如何追踪表的历史版本？
    
- 如何支持 Schema Evolution 和 Partition Evolution？
    
- 如何减少查询需要读取的数据量？
    

Iceberg 本身主要负责表格式和表元数据管理，实际数据通常存储在 Parquet、ORC 等文件中；Catalog 则负责定位和管理表的当前元数据入口。

### 5.2 ACID：解决数据写入的一致性

Iceberg 通过 Metadata、Snapshot 和原子提交机制实现表级事务管理。

核心概念：

|概念|作用|
|---|---|
|Data files|实际存储数据的文件|
|Manifest files|记录一组 Data files 的元数据|
|Manifest list|记录某个 Snapshot 使用的 Manifest files|
|Snapshot|表在某个时刻的状态|
|Metadata file|记录表结构、Snapshot、当前状态等元数据|

写入时，Iceberg 通常先生成新的数据文件和元数据文件，然后通过原子更新 Catalog 中的当前 Metadata 指针来提交新 Snapshot。

这样读者可以看到一个完整、已提交的表版本，而不会看到写入一半的状态。

### 5.3 Snapshot 与 Time Travel

每次成功提交表变更，通常会创建一个新的 Snapshot。

例如：

```
Snapshot 100
    │
    ├── Insert data
    ↓
Snapshot 101
    │
    ├── Update data
    ↓
Snapshot 102
    │
    └── Delete data
```

通过 Snapshot，可以查询历史数据状态：

```
SELECT *
FROM catalog.db.orders
VERSION AS OF 101;
```

注意：历史 Snapshot 是否还能查询，取决于 Snapshot 保留策略以及对应数据文件是否已经被清理。Time Travel 不是永久保存所有历史数据。

### 5.4 Pruning：减少查询扫描量

Pruning 的目的是在查询时尽可能跳过不相关的数据文件。

Iceberg 的查询过程可以理解为：

```
Query
  │
  ↓
Snapshot
  │
  ↓
Manifest List
  │
  ↓
Manifest Files
  │
  ↓
Data Files
  │
  ↓
Read matching Parquet files
```

不同层级的元数据可以帮助排除不需要读取的文件：

1. Partition Pruning：根据分区信息排除不相关的分区。
    
2. Manifest Pruning：根据 Manifest 中的分区值、统计信息等排除不相关的 Manifest。
    
3. File-level Pruning：根据文件的列统计信息，例如 min/max、null count，跳过不匹配的 Data files。
    

例如：

```
SELECT *
FROM orders
WHERE order_date = '2026-11-01';
```

如果 Iceberg 能够利用分区信息和文件统计信息，就可以跳过不包含目标日期数据的文件。

### 5.5 ACID 与 Pruning 的区别

|功能|解决的问题|
|---|---|
|ACID / Atomic Commit|写入是否一致、并发提交是否安全|
|Snapshot|如何记录表的历史状态|
|Time Travel|如何读取历史版本|
|Partition Pruning|如何排除不相关的分区|
|Manifest Pruning|如何排除不相关的 Manifest|
|File-level Pruning|如何跳过不相关的数据文件|

核心记忆：

ACID 解决数据写入和读取的一致性问题；Pruning 解决查询需要读取多少数据的问题。

Iceberg 与 Knowledge Catalog 的关系：

- Iceberg：管理表格式、Snapshot、Schema Evolution 和文件元数据。
    
- Knowledge Catalog：帮助用户发现、理解、分类和治理数据资产。
    

两者职责不同，可以在 Lakehouse 架构中协同使用，但 Knowledge Catalog 并不取代 Iceberg 的表管理能力。

面试问题

1. What is Apache Iceberg, and why is it used in a data lakehouse?
    
2. How does Iceberg provide ACID transaction support?
    
3. What is the relationship between a Snapshot, Manifest List, and Manifest File?
    
4. How does Iceberg support time travel?
    
5. What is the difference between partition pruning and file-level pruning?
    
6. How does Iceberg handle concurrent writes?
    
7. What is the difference between Iceberg and a data catalog?
    

## 6. Schema on Write 与 Schema on Read

### 6.1 Schema on Write

Schema on Write 指的是在数据写入目标系统之前，先定义并执行数据结构和质量检查。

典型流程：

```
Source Data
    │
    ↓
Data Validation
    │
    ↓
Data Transformation
    │
    ↓
Schema Enforcement
    │
    ↓
BigQuery / Data Warehouse
```

例如：

源数据：

```
{
  "id": "1001",
  "amount": "120.5",
  "email": "alice@example.com"
}
```

写入前需要：

- 将 `id` 转换为正确的数据类型。
    
- 将 `amount` 转换为 NUMERIC。
    
- 检查 email 格式。
    
- 检查必要字段是否为空。
    
- 不符合规则的数据进入 Reject / Dead Letter Table。
    

优点：

- 数据结构更加规范。
    
- 下游查询更容易编写。
    
- 可以在数据进入分析层之前发现问题。
    
- 更容易保证数据质量和业务一致性。
    

缺点：

- 数据写入流程更加复杂。
    
- Schema 变化可能需要修改 ETL Pipeline。
    
- 处理未知字段或半结构化数据时灵活性较低。
    

### 6.2 Schema on Read

Schema on Read 指的是数据写入时不要求完全固定的数据结构，而是在读取或查询时再解释和应用 Schema。

典型流程：

```
Source Data
    │
    ↓
Raw Data Lake
    │
    ↓
Cloud Storage
    │
    ↓
Read / Query
    │
    ↓
Schema Interpretation
    │
    ↓
Analytics
```

例如，将原始 JSON 文件直接存储在 Cloud Storage：

```
{
  "id": "1001",
  "amount": "120.5",
  "email": "alice@example.com",
  "new_field": "value"
}
```

读取时再决定：

- 如何解析字段。
    
- 如何转换字段类型。
    
- 如何处理缺失字段。
    
- 如何处理新增字段。
    

优点：

- 数据采集阶段灵活。
    
- 可以保留原始数据。
    
- 更容易适应数据源变化。
    
- 适合 Raw Data Lake 和半结构化数据。
    

缺点：

- 下游读取逻辑更复杂。
    
- 不同消费者可能采用不同的 Schema。
    
- 如果没有统一的数据治理，可能导致数据质量不一致。
    
- 每次查询可能需要进行解析和转换，增加处理开销。
    

### 6.3 两者的对比

|比较|Schema on Write|Schema on Read|
|---|---|---|
|Schema 应用时机|写入时|读取时|
|数据写入前是否需要校验|通常需要|不一定|
|灵活性|相对较低|较高|
|数据质量控制|更靠近数据入口|更依赖下游处理|
|常见场景|Data Warehouse、Curated Layer|Raw Data Lake、原始 JSON|
|典型技术|BigQuery、ETL Pipeline|Cloud Storage、数据湖查询引擎|

### 6.4 实际项目中的组合方式

企业通常不会只选择一种方式，而是将两种方式结合起来：

```
                Source Systems
                       │
                       ↓
              Raw Data Lake
             Schema on Read
                       │
                       ↓
                Data Processing
             Validation / Cleaning
                       │
                       ↓
               Curated Layer
             Schema on Write
                       │
                       ↓
              BigQuery Analytics
```

例如：

1. 从 Pub/Sub 接收 JSON 事件。
    
2. 将原始事件保存到 Cloud Storage，保留原始数据。
    
3. 使用 Dataflow 解析、清洗和验证数据。
    
4. 将符合规则的数据写入 BigQuery。
    
5. 将不符合规则的数据写入 Dead Letter Table。
    
6. 使用 Knowledge Catalog 管理表的元数据、业务描述和质量信息。
    

面试问题

1. What is the difference between schema on write and schema on read?
    
2. What are the advantages and disadvantages of schema on read?
    
3. Why would you store raw JSON data in Cloud Storage before transforming it?
    
4. How would you handle schema evolution in a data lake?
    
5. Would you use schema on write or schema on read for a data warehouse?
    
6. How would you combine both approaches in a lakehouse architecture?
    

## 7. IAM 与 Data Governance 权限管理

### 7.1 Dataplex / Knowledge Catalog IAM

数据治理不仅要管理元数据，还需要控制用户对数据资产的访问权限。

需要区分三类权限：

|权限类型|控制内容|
|---|---|
|Catalog permissions|是否可以搜索、查看或修改 Catalog 元数据|
|Dataplex resource permissions|是否可以管理 Lake、Zone、Asset 等资源|
|Underlying data permissions|是否可以实际读取或修改 BigQuery、Cloud Storage 中的数据|

例如：

```
User
 │
 ├── Knowledge Catalog
 │     └── Search / View Metadata
 │
 ├── Dataplex
 │     └── Manage Lake / Zone / Asset
 │
 └── BigQuery / Cloud Storage
       └── Read / Write Data
```

关键区别：能够查看数据的 Metadata，不代表一定能够读取底层数据。

同样，能够管理 Dataplex 资源，也不代表用户自动获得所有底层数据的完整读写权限。

具体权限取决于所分配的 IAM Role、资源层级、权限继承关系以及对应服务的访问控制配置。

### 7.2 IAM Role 与底层资源权限

Dataplex 的部分 IAM Role 可能包含与关联数据资源交互所需的权限。

例如，在管理 Dataplex Asset 时，可能需要访问关联的 Cloud Storage 或 BigQuery 资源。

但不能简单认为：

- Dataplex Writer 一定可以修改所有 GCS 数据。
    
- 拥有 Catalog 权限就可以读取 BigQuery 数据。
    
- 拥有 Dataplex 权限就自动拥有所有底层资源权限。
    

应该分别检查 Catalog、Dataplex 和底层资源的 IAM 权限。

### 7.3 Sensitive Data Governance

例如，企业有一个包含客户邮箱的 BigQuery 表：

```
customers
├── customer_id
├── email
└── phone
```

治理流程可以是：

1. 使用 Knowledge Catalog 记录表和字段的业务元数据。
    
2. 使用 Aspect 标记 `email` 和 `phone` 为敏感字段。
    
3. 使用 BigQuery Policy Tags 或 Data Policies 实施列级访问控制和数据脱敏。
    
4. 使用 IAM 控制用户能否访问数据。
    
5. 使用审计日志跟踪数据访问行为。
    

注意：Metadata Classification 和实际 Access Control 是两个不同的步骤。

面试问题

1. What is the difference between metadata access and data access?
    
2. How would you implement column-level security in BigQuery?
    
3. What is the relationship between IAM and Dataplex?
    
4. How would you protect PII data in a data lake?
    
5. Does classifying a column as sensitive automatically restrict access to it?
    

## 8. Google Cloud 数据治理整体架构

将前面的知识点组合起来，可以构建一个完整的数据治理流程。

Data Sources

Applications · Databases · Pub/Sub · External Systems

Raw Data Lake

Cloud Storage · Raw Zone · 原始数据保留

Schema on Read

Data Processing

Dataflow · ETL · Validation · Deduplication · Schema Enforcement

Curated Data Layer

BigQuery · Curated Zone · Cleaned Data

Schema on Write

Analytics / BI / Machine Learning

BigQuery SQL · Looker · ML Workloads

Governance Layer：Knowledge Catalog

Data Discovery

搜索数据资产

Metadata / Aspects

描述和分类数据

Data Governance

数据保护和访问治理

Data Quality

质量扫描和监控

Knowledge Catalog 是贯穿数据资产的治理与元数据层，不是只能放在 Curated Layer 之后的一个处理步骤。

### 8.1 组件职责总结

|产品 / 技术|核心职责|
|---|---|
|Cloud Storage|保存原始文件和数据湖文件|
|BigQuery|数据仓库、SQL 分析|
|Dataflow|批处理和流式数据处理|
|Apache Iceberg|Lakehouse 表格式、Snapshot、ACID|
|Dataplex Lake / Zone / Asset|组织和管理数据湖资源|
|Knowledge Catalog|元数据、搜索、数据发现和治理|
|Data Quality Scan|检查数据是否符合质量规则|
|IAM / Policy Tags|控制数据和元数据访问权限|

### 8.2 产品名称和考试注意事项

- Knowledge Catalog 是 Dataplex Universal Catalog 的新名称。原有 Dataplex Universal Catalog 的 API、配置和部署仍然可用，相关 `gcloud dataplex` 命令及 IAM 标识也保留了 `dataplex` 名称。
    
    ![](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com&sz=32)
    
    Google Cloud Documentation
    
    +1
    
- Dataplex 的 Lake、Zone、Asset 是传统数据湖组织模型，不要与 Knowledge Catalog 的 Entry、Entry Group、Aspect 混为一谈。
    
- Knowledge Catalog 的 Aspect 可以描述整个数据资产、列或资源之间的关系；它本身不是访问控制策略。
    
    ![](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com&sz=32)
    
    Google Cloud Documentation
    
    +1
    
- 传统 CloudDQ Data Quality Task 和当前 Auto Data Quality 是需要区分的两种实现方式。新项目优先了解 Auto Data Quality。
    
    ![](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com&sz=32)
    
    Google Cloud Documentation
    
    +1
    
- Iceberg 是开放表格式，不是 Dataplex 的一个 Zone，也不是 Knowledge Catalog 的替代品。
    

面试问题

1. How would you design an end-to-end data governance architecture on Google Cloud?
    
2. How do Dataplex and Knowledge Catalog support data governance?
    
3. How would you organize raw and curated data in a data lake?
    
4. How would you combine Dataflow, BigQuery, Iceberg, and Knowledge Catalog?
    
5. How would you implement data discovery, data quality, and PII protection in the same architecture?
    
6. What is the difference between a data catalog, a data lake management service, and a table format?