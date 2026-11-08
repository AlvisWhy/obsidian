
# 1. Storage & Metadata Architecture

## 1.1 Unity Catalog 的层级结构

![What is Unity Catalog in Azure Databricks?](https://images.openai.com/static-rsc-4/ofeS_ZvAOrSsifbx7ETOtjJ0uNbU5FyqhS3UWdwIPwsa8oMSYXUXz5ojpwoNSXqC0lQEAYDhrLm9qKRWpNruNfXMEIBxcdfkhbKuVhyLVREruEpI32fenULIxSkfZexl5OTBF0SaLUxlmPhab5I3hrJk8y2WocCFuDbxHxqR8BQ?purpose=inline)

[thetechplatform.com](https://www.thetechplatform.com/post/what-is-unity-catalog-in-azure-databricks)

Unity Catalog 的核心层级：

```
Account
  |
  └── Metastore
        |
        ├── Catalog (dev)
        |     |
        |     ├── Schema (bronze)
        |     |     ├── Table
        |     |     ├── View
        |     |     ├── Volume
        |     |     └── Function
        |     |
        |     └── Schema (silver)
        |
        └── Catalog (prod)
              |
              └── Schema (bronze)
                    └── Table
```

|层级|作用|说明|
|---|---|---|
|Account|Databricks 账户层|管理用户、Workspace、Metastore 等|
|Metastore|统一元数据和治理管理|通常按 Region 部署，一个 Metastore 可以关联多个 Workspace|
|Catalog|逻辑隔离容器|常用于 dev、test、prod 等环境|
|Schema|数据库逻辑容器|组织同一业务域或数据层的数据对象|
|Table|结构化数据对象|Managed Table 或 External Table|
|View|虚拟查询对象|保存 SQL 查询定义，不独立存储完整结果|
|Volume|非结构化文件管理对象|管理 CSV、JSON、PDF、图片等文件|
|Function|可复用函数|SQL UDF 等|

### 1.2 Metastore

Metastore 是 Unity Catalog 的最高级元数据容器。

主要特点：

- 在 Databricks Account 层创建，而不是在单个 Workspace 内创建。
    
- 通常一个 Region 可以创建一个 Metastore，但并不是每个 Region 只能有一个。
    
- 一个 Metastore 可以分配给多个 Workspace。
    
- Metastore 保存 Catalog、Schema、Table、Volume、权限等元数据。
    
- Metastore 可以配置默认 Managed Storage Location。
    
- 实际数据通常保存在云存储中，例如 Azure ADLS Gen2、AWS S3 或 GCP Cloud Storage。
    

注意：Metastore 不等于实际数据存储。

Metastore 管理元数据，实际数据文件仍然保存在 Cloud Object Storage 中。

另外，默认 Managed Storage Location 不一定意味着所有 Table 都使用同一个存储目录。Catalog、Schema 也可以配置自己的 Managed Storage Location。

查询当前 Metastore：

```
SELECT current_metastore();
```

### 1.3 Catalog

Catalog 是 Metastore 下的第一级逻辑容器。

典型设计：

```
Metastore
  |
  ├── dev
  |    ├── bronze
  |    ├── silver
  |    └── gold
  |
  ├── test
  |    ├── bronze
  |    ├── silver
  |    └── gold
  |
  └── prod
       ├── bronze
       ├── silver
       └── gold
```

Catalog 可以用于：

- Environment isolation：dev / test / prod。
    
- Business domain isolation：finance / sales / marketing。
    
- Access control：不同部门使用不同权限。
    
- Data organization：统一组织数据资产。
    

Catalog 并不一定必须按照 Environment 划分，也可以按照业务领域划分。

例如：

```
prod
  ├── sales
  ├── customer
  └── finance
```

### 1.4 Schema

Schema 相当于数据库中的逻辑命名空间。

例如：

```
SELECT *
FROM prod.bronze.orders;
```

这里：

- `prod`：Catalog
    
- `bronze`：Schema
    
- `orders`：Table
    

完整的三层命名：

```
catalog.schema.object
```

### 1.5 Managed vs External Objects

这是面试中很容易被问到的区别。

|特性|Managed Table|External Table|
|---|---|---|
|Metadata|Unity Catalog 管理|Unity Catalog 管理|
|Data files|Databricks 管理存储位置|用户指定的外部存储位置|
|Location|UC Managed Storage|External Location|
|DROP TABLE|通常删除元数据和数据|删除元数据，不删除外部数据|
|Lifecycle|Databricks 管理|用户负责外部数据生命周期|

重要补充：Managed Table 不等于 Delta Table。

在 Unity Catalog 中，Managed Table 可以使用受支持的表格式，例如 Delta 或 Iceberg，具体取决于 Databricks 的功能和配置。不能笼统地认为 Managed Table 必须是 Delta。

Volume 同样有两种：

- Managed Volume：Unity Catalog 管理存储位置和生命周期。
    
- External Volume：关联外部存储位置，数据由外部存储管理。
    

View 则不直接保存完整查询结果，而是保存查询定义。查询时根据定义访问底层数据。

## 1.6 Managed Storage Location

Managed Storage Location 决定 Managed Tables 和 Managed Volumes 的实际数据文件存放位置。

可以在三个层级配置：

```
Metastore
  └── Managed Storage Location
        |
        Catalog
          └── Managed Storage Location
                |
                Schema
                  └── Managed Storage Location
```

优先级：

```
Schema > Catalog > Metastore
```

即在创建 Managed Table 时，使用层级中最低、最具体的可用 Managed Storage Location。

例如：

```
Metastore: s3://company/default/

Catalog dev: s3://company/dev/

Schema bronze: s3://company/dev/bronze/
```

那么在 `dev.bronze` 中创建的 Managed Table 会使用 Schema 层级的存储位置。

如果 Schema 没有配置，则向上寻找 Catalog；如果 Catalog 也没有配置，则使用 Metastore 的配置。

注意：Metastore-level managed storage 现在不是必需配置。Databricks 推荐考虑以 Catalog 为单位进行存储隔离。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

### 面试问题

Q1. What is the difference between a Metastore and a Catalog?

参考回答：

- Metastore is the top-level metadata and governance container.
    
- Catalog is the first-level logical container within a Metastore.
    
- A Metastore can contain multiple catalogs.
    
- A workspace is assigned to one Metastore but can access multiple catalogs within it, subject to permissions and workspace bindings.
    

Q2. Can multiple workspaces share the same Metastore?

Yes. Multiple workspaces in the same region can be assigned to the same Metastore. This allows centralized governance and data sharing across workspaces.

Q3. What is the difference between Managed and External Tables?

Managed Tables have their storage lifecycle managed by Unity Catalog. External Tables register data located in externally managed storage. Dropping an external table removes its metadata registration but preserves the underlying data files.

Q4. What happens when you drop a Managed Table?

Unity Catalog manages the lifecycle of the underlying data files. Dropping a managed table removes its metadata and initiates deletion of its managed data, subject to the platform's retention and recovery behavior.

Q5. Where are Managed Tables physically stored?

In cloud object storage such as S3, ADLS Gen2 or GCS. Their storage location is determined by the applicable Managed Storage Location configuration.

Q6. What is the difference between a Schema and a Catalog?

A Catalog is a higher-level container for organizing data domains or environments, while a Schema organizes tables, views, functions and volumes within a Catalog.

# 2. Compute & Workspace Architecture

## 2.1 Workspace 和 Metastore 的关系

首先区分三个概念：

|Component|Responsibility|
|---|---|
|Account|管理 Databricks 账户、用户和 Workspace|
|Workspace|用户进行开发、Notebook、Job、SQL 等工作的环境|
|Metastore|集中管理数据对象元数据和权限|

架构：

```
Databricks Account
       |
       ├── Workspace A (dev)
       |       |
       |       └── Compute
       |
       ├── Workspace B (test)
       |       |
       |       └── Compute
       |
       └── Workspace C (prod)
               |
               └── Compute

       All workspaces
              |
              v
        Unity Catalog
          Metastore
              |
       ┌──────┼──────┐
       dev   test   prod
```

关键点：

1. 一个 Workspace 必须关联一个 Metastore。
    
2. 一个 Metastore 可以关联多个 Workspace。
    
3. Workspace 提供计算和开发环境。
    
4. Metastore 提供统一的数据治理。
    
5. 不同 Workspace 可以访问同一个 Catalog，但需要满足权限和 Workspace Binding 要求。
    

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

## 2.2 Workspace-Catalog Binding

这是你原来笔记里缺少的一个重要概念。

默认情况下：

```
Workspace A ──┐
              |
Workspace B ──┼── Metastore ── Catalog
              |
Workspace C ──┘
```

同一个 Metastore 下的 Catalog 默认可以被关联的 Workspace 访问。

但是可以通过 Workspace-Catalog Binding 限制：

```
Workspace A (dev)  ─── dev_catalog
Workspace B (test) ─── test_catalog
Workspace C (prod) ─── prod_catalog
```

例如：

- Dev Workspace 只能访问 dev_catalog。
    
- Prod Workspace 只能访问 prod_catalog。
    
- 即使某个用户拥有 prod_catalog 的 SELECT 权限，如果从没有绑定 prod_catalog 的 Dev Workspace 访问，仍然会被拒绝。
    

还可以将 Workspace 对 Catalog 的访问设置为 Read-only。

面试重点：Workspace Binding 和 Privilege 是两个不同的访问控制层。

- Privilege：决定用户可以对数据做什么。
    
- Workspace Binding：决定数据可以从哪些 Workspace 被访问。
    

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

## 2.3 Compute 如何访问 Unity Catalog？

Compute 是实际执行 SQL、Spark Job、Notebook 的资源。

例如：

```
User
 |
Workspace
 |
Compute (SQL Warehouse / Cluster)
 |
Unity Catalog
 |
Catalog.Schema.Table
 |
Cloud Storage
```

Compute 并不是直接依靠 Workspace Admin 身份访问所有数据。

它需要满足：

- Compute 使用支持 Unity Catalog 的访问模式和 Runtime。
    
- 用户或 Job 的运行身份拥有必要的数据权限。
    
- Workspace 与 Metastore 已经正确关联。
    
- Catalog 已经允许当前 Workspace 访问。
    

此外，Job 的实际权限通常取决于其 `Run as` 身份，而不是简单取决于创建 Job 的人。

### 面试问题

Q1. What is the difference between a Workspace and a Metastore?

A Workspace is a development and compute environment, whereas a Metastore is a centralized governance and metadata management layer.

Q2. Can two workspaces access the same catalog?

Yes. If they share the same Metastore and the Catalog is bound to both workspaces, users still need the appropriate privileges.

Q3. What is Workspace-Catalog Binding?

It restricts which workspaces can access a particular catalog. It provides an additional security boundary beyond user-level permissions.

Q4. If a user has SELECT permission on a production table, can they query it from any workspace?

Not necessarily. The workspace must be bound to the catalog containing that table, and the user must also have the required usage privileges.

Q5. What is the difference between a SQL Warehouse and a Metastore?

A SQL Warehouse provides compute resources for executing SQL queries. A Metastore manages metadata, data governance and permissions.

# 3. Access Control & Governance

这是 Unity Catalog 非常重要的面试模块。

## 3.1 Principals

首先，Unity Catalog 的权限是授予 Principal 的。

Principal 包括：

- User：个人用户。
    
- Group：用户组。
    
- Service Principal：用于自动化任务、应用程序和 CI/CD 的身份。
    

企业中通常推荐将权限授予 Group，而不是直接授予每个 User。

例如：

```
Group: data_engineers
  ├── User A
  ├── User B
  └── User C

Group: data_analysts
  ├── User D
  └── User E
```

然后：

```
data_engineers -> SELECT, MODIFY
data_analysts  -> SELECT
```

这样可以简化权限管理。

## 3.2 Unity Catalog 的权限类型

主要分成两种：

### A. Usage Privileges

负责允许用户进入某个层级、访问其下的数据对象。

|Privilege|作用|
|---|---|
|USE CATALOG|允许用户使用 Catalog 内的对象|
|USE SCHEMA|允许用户使用 Schema 内的对象|

这两个权限本身不会赋予用户读取数据的能力。

### B. Object Privileges

负责允许用户执行具体操作。

|Privilege|作用|
|---|---|
|SELECT|查询 Table / View|
|MODIFY|修改 Table 数据|
|CREATE TABLE|创建 Table|
|CREATE SCHEMA|创建 Schema|
|CREATE VOLUME|创建 Volume|
|EXECUTE|执行 Function|
|READ VOLUME|读取 Volume 文件|
|WRITE VOLUME|写入 Volume 文件|
|CREATE FUNCTION|创建 Function|

注意：`MODIFY` 是一个较宽泛的权限，包含表数据修改能力；对于需要更细粒度控制的场景，还可以使用 `INSERT`、`UPDATE`、`DELETE` 等权限。

## 3.3 用户读取 Table 需要什么权限？

假设：

```
prod.sales.orders
```

用户要执行：

```
SELECT *
FROM prod.sales.orders;
```

至少需要：

```
USE CATALOG ON prod
        +
USE SCHEMA ON prod.sales
        +
SELECT ON prod.sales.orders
```

例如：

```
GRANT USE CATALOG
ON CATALOG prod
TO `data_analysts`;

GRANT USE SCHEMA
ON SCHEMA prod.sales
TO `data_analysts`;

GRANT SELECT
ON TABLE prod.sales.orders
TO `data_analysts`;
```

注意：上面是按 Table 逐个授予 SELECT 的方式。

如果希望用户读取 Schema 下所有现有和未来创建的 Tables / Views：

```
GRANT SELECT
ON SCHEMA prod.sales
TO `data_analysts`;
```

这样就可以使用权限继承机制。

## 3.4 Privilege Inheritance（权限继承）

这是你原来笔记中最需要补充和修正的地方。

Unity Catalog 支持 Catalog 和 Schema 层级的权限继承。

例如：

```
Catalog: prod
    |
    └── Schema: sales
           |
           ├── Table: orders
           ├── Table: customers
           └── View: revenue
```

如果：

```
GRANT SELECT ON SCHEMA prod.sales
TO `data_analysts`;
```

那么用户可以获得 Schema 下当前及未来 Tables / Views 的 SELECT 权限。

同样，在 Catalog 层授予适用的对象权限，也可以向下继承。

但要区分两件事：

- Usage privileges：满足访问路径要求。
    
- Object privileges：决定能执行什么操作。
    

|授权位置|权限|是否向下继承|
|---|---|---|
|Catalog|SELECT|是，适用于下层 Tables / Views|
|Schema|SELECT|是，适用于下层 Tables / Views|
|Catalog|MODIFY|是，适用于相应的下层 Tables|
|Schema|MODIFY|是，适用于相应的下层 Tables|
|Catalog|USE SCHEMA|是，适用于下层 Schemas|
|Catalog|USE CATALOG|不向下作为其他权限继承|
|Schema|USE SCHEMA|用于该 Schema 的访问路径|
|Table|SELECT|只针对该 Table|

这里有两个容易混淆的点：

1. `USE CATALOG` 不会自动赋予 `USE SCHEMA`。
    
2. `USE SCHEMA` 不会自动赋予 `SELECT`。
    

但是，Catalog 层可以直接授予 `USE SCHEMA`，让它继承到 Catalog 下的 Schemas。

因此，不要简单记成所有父级权限都不会继承。

正确理解是：不同 Privilege 有不同的继承规则。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+2

## 3.5 Workspace Admin vs Catalog Owner vs Metastore Admin

这三个角色也需要区分。

|Role|主要职责|
|---|---|
|Account Admin|管理整个 Databricks Account|
|Metastore Admin|管理 Metastore 级别的治理和配置|
|Workspace Admin|管理单个 Workspace|
|Catalog Owner|管理指定 Catalog|
|Schema Owner|管理指定 Schema|
|Table Owner|管理指定 Table|

你原来的笔记：

> workspace admin can create catalog, but could not visit catalog created by others

需要更精确地理解。

- 某些自动启用 Unity Catalog 的 Workspace Admin 默认拥有 `CREATE CATALOG` 等权限。
    
- 但是，创建 Catalog 的权限不等于对所有其他 Catalog 的数据拥有 SELECT 权限。
    
- Catalog Owner 可以管理其 Catalog 的权限和对象。
    
- Workspace Admin 也不会因为这个身份自动获得所有 Catalog 的数据读取权限。
    

另外，Workspace Admin 的 Unity Catalog 默认权限取决于 Workspace 的启用方式和管理员策略，不能认为所有历史 Workspace 都有相同的默认权限。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

### 面试问题

Q1. What is the difference between USE CATALOG and SELECT?

USE CATALOG allows a user to interact with objects inside a catalog, while SELECT allows the user to read data from a table or view.

Q2. Does granting SELECT on a Schema automatically grant access to all tables?

Yes. SELECT privileges granted at the Schema level are inherited by existing and future tables and views in that Schema. However, the user still needs USE CATALOG and USE SCHEMA.

Q3. Does USE CATALOG automatically grant USE SCHEMA?

No, not directly. However, USE SCHEMA can be granted at the Catalog level and inherited by its child schemas.

Q4. What is the difference between a Workspace Admin and a Metastore Admin?

A Workspace Admin manages a specific workspace, while a Metastore Admin manages governance and administrative operations at the Metastore level.

Q5. How would you implement least-privilege access for a data engineering team?

I would create separate groups for engineers and analysts, grant permissions at the appropriate Catalog or Schema level, and use Service Principals for automated workloads. I would also apply Workspace-Catalog Binding to isolate production data.

Q6. A user has SELECT on a table but receives a permission error. What would you check?

I would check:

1. USE CATALOG.
    
2. USE SCHEMA.
    
3. SELECT on the table.
    
4. Workspace-Catalog Binding.
    
5. The identity used by the compute or Job.
    
6. Whether the compute supports Unity Catalog.
    

# 4. Data Ingestion & CDC

这一部分对应你笔记中的：

```
Connection
   |
Gateway
   |
Staging
   |
Ingestion Pipeline
   |
Destination Table
```

这里主要讲的是 Databricks Lakeflow Connect Managed Database Connectors 的 CDC 架构。

## 4.1 Connection

Connection 是 Unity Catalog 中管理外部数据库连接信息的对象。

例如：

```
Unity Catalog
    |
    └── Connection
          |
          ├── Database hostname
          ├── Port
          ├── Username
          ├── Authentication
          └── Database information
```

它的作用：

- 管理外部数据库连接配置。
    
- 提供数据库认证信息。
    
- 让 Databricks 可以连接外部数据源。
    
- 被相应的 Ingestion Pipeline 使用。
    

例如连接：

- PostgreSQL
    
- MySQL
    
- SQL Server
    
- Oracle
    

需要注意：Connection 不等于 Gateway。Connection 是连接配置对象，而 Gateway 是负责捕获数据库变化的运行组件。

## 4.2 CDC Architecture

![Watermark-Based Incremental Ingestion with Lakeflow Connect — SunnyData](https://images.openai.com/static-rsc-4/HiAANdmuFrelltn7KFVYM3O_IdJ-6gtsZpyt_Ysh2R_GwZe3ckfhyGAWmeWq1oUWD7HqqfZZw1Cbfev6DTJXZt7XDzp0Wag7qLyzRargaIIZlmAjc2YOtPtu8UXgxIOE-NRgH5dfiT8Vi3tsv541wzXJsixeZDJpZ5SFs3yp3w0?purpose=inline)

[sunnydata.ai](https://www.sunnydata.ai/blog/lakeflow-connect-query-based-capture-incremental-ingestion)

完整流程：

```
Source Database
       |
       | Database logs / CDC
       v
Ingestion Gateway
       |
       | Extract snapshots and changes
       v
Staging Storage
(UC Managed Volume)
       |
       v
Ingestion Pipeline
       |
       v
Destination Streaming Tables
       |
       v
Unity Catalog
```

每个组件的作用：

|Component|Responsibility|
|---|---|
|Connection|保存外部数据库连接配置|
|Ingestion Gateway|持续捕获数据库 Snapshot 和 Change Logs|
|Staging Storage|临时保存已经提取的数据变化|
|Ingestion Pipeline|将 Staging 中的数据应用到目标表|
|Destination Table|最终存储 CDC 数据的 Streaming Table|

### Ingestion Gateway

Gateway 主要负责从 Source Database 获取：

- Initial Snapshot：首次同步时获取数据库现有数据。
    
- Change Logs：获取后续发生的 INSERT、UPDATE、DELETE 等变化。
    
- Metadata：获取相关表结构信息。
    

Gateway 持续运行，确保数据库日志被及时读取，避免 Source Database 的日志在读取之前被截断。

Gateway 使用 Classic Compute，而不是简单地理解为一个网络网关。

### Staging Storage

Staging Storage 通常是 Unity Catalog Volume。

为什么需要 Staging？

假设：

```
Gateway: continuously captures CDC
Pipeline: runs every 30 minutes
```

如果没有 Staging，Gateway 和 Pipeline 就必须紧密同步。

有了 Staging：

1. Gateway 可以持续捕获数据库变化。
    
2. 数据先写入 Staging。
    
3. Pipeline 可以按照自己的调度周期运行。
    
4. Pipeline 从 Staging 获取数据并写入目标表。
    

优点：

- Decouple ingestion and processing。
    
- Improve fault recovery。
    
- Support independent scheduling。
    
- Reduce the risk of losing captured changes.
    

Databricks 的 Managed Database Connectors 会自动创建 Staging Volume，默认暂存数据会在 30 天后自动清理。

### Ingestion Pipeline

Pipeline 负责：

1. 从 Staging 获取数据。
    
2. 处理 Snapshot 和 CDC Events。
    
3. 根据配置应用数据变更。
    
4. 将结果写入 Destination Streaming Tables。
    

Destination Streaming Tables 是支持增量处理的 Delta Tables。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

## 4.3 Query-Based Connector vs CDC Connector

这是一个很值得面试准备的对比。

Databricks 有不同的数据库数据摄取方式。

|特性|CDC-based Connector|Query-based Connector|
|---|---|---|
|数据获取|数据库 Change Logs|定期执行 SQL Query|
|变化追踪|CDC / Database Logs|Cursor Column|
|Gateway|需要|不需要|
|Staging Volume|需要|不需要|
|Ingestion|持续捕获变化|Scheduled Query|
|Latency|通常较低|取决于运行频率|
|删除检测|支持相应 CDC Delete Events|不能天然捕获物理删除|
|适用场景|低延迟增量同步|定期增量数据同步|

### Query-Based Connector 工作原理

Query-Based Connector 不读取数据库的 CDC Logs，而是定期查询 Source Database。

例如，源表：

|id|name|updated_at|
|---|---|---|
|1|Alice|2026-11-01 10:00|
|2|Bob|2026-11-02 10:00|
|3|Carol|2026-11-03 10:00|

假设使用 `updated_at` 作为 Cursor Column。

第一次运行：

```
SELECT *
FROM source_table;
```

保存当前最大的 Cursor：

```
High Water Mark = 2026-11-03 10:00
```

第二次运行：

```
SELECT *
FROM source_table
WHERE updated_at > '2026-11-03 10:00';
```

只获取上次成功运行之后新增或更新的数据。

这里有几个重要概念：

Cursor Column

用来跟踪数据变化的字段，通常是：

- Timestamp
    
- Integer
    
- Date
    

High Water Mark

记录上次成功处理到的最大 Cursor Value。

Monotonically Increasing

Cursor Value 必须单调递增，不能随意下降。

例如：

```
100 → 101 → 102 → 103
```

如果某行更新后 Cursor 仍然是旧值，可能就无法被下一次查询捕获。

另外：

- NULL Cursor 的记录不会被摄取。
    
- 默认不是 Continuous Ingestion，而是 Scheduled Ingestion。
    
- 普通 Query-Based Connector 无法天然检测已经从源表物理删除的记录。
    
- SCD Type 1 / Type 2 模式还需要考虑 Primary Key、Late-arriving Records 等问题。
    

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on Google Cloud

+2

### 面试问题

Q1. What is the difference between CDC-based and query-based ingestion?

CDC-based ingestion captures database changes from transaction logs, while query-based ingestion periodically queries the source database using a cursor column to identify new or updated records.

Q2. Why does CDC ingestion require a staging layer?

The staging layer decouples continuous change capture from downstream processing. It also improves fault recovery and allows the ingestion pipeline to run independently.

Q3. What is the role of an Ingestion Gateway?

It continuously extracts initial snapshots and change logs from the source database before those logs are truncated.

Q4. What is a High Water Mark?

It represents the latest successfully processed cursor value. The next ingestion run uses it to identify records that need to be processed.

Q5. Can query-based ingestion detect deleted records?

Not inherently. If a source row is physically deleted, it will no longer appear in subsequent SELECT queries. CDC-based ingestion can capture delete events from database logs.

Q6. What happens if an ingestion pipeline fails?

Captured changes can remain in staging storage, allowing the pipeline to resume processing after recovery, subject to the connector's retention and recovery guarantees.

Q7. What is the difference between SCD Type 1 and SCD Type 2?

- SCD Type 1 overwrites existing records without retaining historical versions.
    
- SCD Type 2 preserves historical versions by maintaining multiple records with validity periods or other version-tracking fields.
    

# 5. Files & Storage Paths

这是 Databricks 中容易混淆的一个知识点。

你原来的笔记把 Workspace、DBFS、Unity Catalog 放在同一层级是合理的，但需要明确它们管理的是不同类型的资源。

## 5.1 Databricks 的三类文件管理方式

![Work with files on Azure Databricks - Azure Databricks | Microsoft Learn](https://images.openai.com/static-rsc-4/5LKhlnefcW1cBOyjourXEg4LgYnnbvRbzW_ySm1KGjMA5x8BqwFvmPo_zMWssFe1FmZrakpkGFoV-XhIgWaO5GgbKhF_fDcvyP4qBz8VjEizQCcscHAwokOC03chPKk75eObqErP15gQ0touxwz3fyqg3xKFr43MmeQa226jknM?purpose=inline)

[learn.microsoft.com](https://learn.microsoft.com/en-us/azure/databricks/files/)

```
Databricks
    |
    ├── Workspace Files
    |       |
    |       └── Notebook / Python / SQL / Code
    |
    ├── DBFS (Legacy)
    |       |
    |       ├── DBFS Root
    |       └── DBFS Mounts
    |
    └── Unity Catalog
            |
            ├── Tables
            |      └── Structured Data
            |
            ├── Volumes
            |      └── Non-tabular Files
            |
            └── External Locations
                   └── Cloud Storage Access
```

## 5.2 Workspace Files

Workspace Files 用于存储开发相关的文件。

例如：

```
/Workspace/
    |
    ├── Users/
    |     └── user@example.com/
    |            ├── notebook.py
    |            └── analysis.sql
    |
    ├── Shared/
    |
    └── Repos/
```

主要用途：

- Notebooks
    
- Python files
    
- SQL files
    
- Libraries
    
- Project code
    

注意：Workspace Files 不等于数据湖存储。

例如，你可以将 Python ETL 脚本存放在 Workspace，但不建议把大量生产数据文件放在 Workspace Files 中。

## 5.3 DBFS

DBFS = Databricks File System。

它是 Databricks 历史上用于访问云存储的文件系统接口。

以前经常看到：

```
dbfs:/FileStore/
dbfs:/mnt/
dbfs:/user/hive/warehouse/
```

其中：

- DBFS Root：Workspace 级别的传统文件存储。
    
- DBFS Mounts：将外部云存储挂载到 DBFS 路径。
    

例如：

```
dbutils.fs.ls("dbfs:/mnt/data/")
```

重要：DBFS Root 和 DBFS Mounts 已经 Deprecated，不建议用于新的生产架构。

Databricks 现在推荐使用：

- Unity Catalog Volumes
    
- External Locations
    
- Workspace Files
    

但是要注意，`dbfs:/` 这个 URI scheme 仍然存在。例如，UC Volume 仍然支持：

```
dbfs:/Volumes/catalog/schema/volume/
```

因此：

DBFS Root Deprecated ≠ dbfs:/ URI 完全不能使用。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+1

## 5.4 Unity Catalog Volumes

Volume 用于管理非结构化或非表格文件。

例如：

```
Catalog
  |
  └── Schema
        |
        └── Volume
              |
              ├── input/
              |     ├── orders.json
              |     └── customers.csv
              |
              ├── images/
              |
              └── output/
```

Volume 的完整访问路径：

```
/Volumes/<catalog>/<schema>/<volume>/<file_path>
```

例如：

```
/Volumes/prod/bronze/raw_files/orders.json
```

Python 读取：

```
df = spark.read.json(    "/Volumes/prod/bronze/raw_files/orders.json")
```

SQL 读取：

```
SELECT *
FROM json.`/Volumes/prod/bronze/raw_files/orders.json`;
```

Volume 有两种：

|Volume|特征|
|---|---|
|Managed Volume|UC 管理存储位置和生命周期|
|External Volume|关联外部存储路径，文件生命周期由外部管理|

### Volume vs Table

这是很重要的区别：

|特征|Table|Volume|
|---|---|---|
|用途|结构化数据|非表格文件|
|访问方式|`catalog.schema.table`|`/Volumes/...`|
|管理对象|Table Metadata|File Metadata / Access|
|典型格式|Delta / Iceberg|CSV / JSON / PDF / Images|
|权限|SELECT / MODIFY|READ VOLUME / WRITE VOLUME|

例如：

如果你有一个 CSV 文件：

```
/Volumes/dev/bronze/raw_files/orders.csv
```

它只是一个文件。

如果你使用 Spark 读取并写入 Unity Catalog Table：

```
df = spark.read.csv(    "/Volumes/dev/bronze/raw_files/orders.csv",    header=True)df.write.format("delta").saveAsTable(    "dev.bronze.orders")
```

那么：

```
/Volumes/dev/bronze/raw_files/orders.csv
```

是 Volume 中的原始文件。

```
dev.bronze.orders
```

是 Unity Catalog 中注册的 Table。

两者不是同一个对象。

![](https://www.google.com/s2/favicons?domain=https://docs.databricks.com&sz=32)

Databricks on AWS

+2

## 5.5 External Location 和 Storage Credential

这两个概念也建议补充，因为它们和 External Table、External Volume 密切相关。

### Storage Credential

Storage Credential 是 Unity Catalog 用来管理访问云存储所需身份凭证的对象。

例如：

```
Databricks
    |
    └── Storage Credential
            |
            └── IAM Role / Cloud Identity
```

它提供访问 S3、ADLS 或 GCS 的身份认证机制。

### External Location

External Location 将 Storage Credential 和具体的 Cloud Storage Path 关联起来。

例如：

```
Storage Credential
       |
       v
External Location
       |
       v
s3://company-data/bronze/
       |
       ├── External Table
       |
       └── External Volume
```

关系：

- Storage Credential：解决使用什么身份访问 Cloud Storage。
    
- External Location：解决可以访问哪个存储路径。
    

例如创建 External Location：

```
CREATE EXTERNAL LOCATION bronze_location
URL 's3://company-data/bronze/'
WITH (STORAGE CREDENTIAL bronze_credential);
```

用户需要相应的 `READ FILES`、`WRITE FILES` 或创建 External Table / Volume 所需的权限。

面试重点：External Location 不是 Table，也不是 Volume。它是 Unity Catalog 管理的云存储访问对象。

### 面试问题

Q1. What is the difference between Workspace Files and Unity Catalog Volumes?

Workspace Files are primarily used for development assets such as notebooks and scripts, while Unity Catalog Volumes provide governed storage for non-tabular data files.

Q2. What is the difference between DBFS and Unity Catalog Volumes?

DBFS Root and Mounts are legacy file access mechanisms. Unity Catalog Volumes provide centralized governance, access control and managed or external storage for files.

Q3. When should you use a Volume instead of a Table?

Use a Volume for non-tabular files such as JSON, CSV, images and PDFs. Use a Table for structured data that needs SQL querying, schema management and table-level governance.

Q4. What is the difference between a Managed Volume and an External Volume?

Managed Volumes have storage locations and lifecycle managed by Unity Catalog. External Volumes register an existing external storage location without taking over the underlying file lifecycle.

Q5. What is the difference between Storage Credential and External Location?

A Storage Credential defines the cloud identity used to access storage, while an External Location associates that credential with a specific cloud storage path.

Q6. Why should you avoid DBFS Mounts in new production projects?

They are deprecated and do not provide the same centralized governance model as Unity Catalog. External Locations and Volumes provide more granular access control and auditing.

# 6. 最后：整体知识架构与面试复习重点

把前面五个模块连接起来，你应该能够理解 Databricks 的完整数据架构。

## Databricks Lakehouse Architecture

Databricks Account

Users · Groups · Service Principals · Workspaces

Workspace + Compute

Notebooks · Jobs · SQL Warehouses · Clusters

## Unity Catalog Metastore

dev

Catalog

test

Catalog

prod

Catalog

Each Catalog contains Schemas, Tables, Views, Volumes and Functions

Governance

Privileges · Ownership · Lineage · Workspace Binding

Cloud Object Storage

AWS S3 · Azure ADLS Gen2 · GCP Cloud Storage

Data Ingestion

Source Database → Connection → CDC Gateway → Staging Volume → Ingestion Pipeline → Destination Tables

Query-based connectors use scheduled queries and cursor tracking instead of the CDC gateway and staging-volume architecture.

## 6.1 荷兰 Data Engineer 面试重点清单

以下是结合荷兰 Data Engineer、Data Platform Engineer 和 Cloud Data Engineer 岗位常见技术要求整理的练习题，不代表某个公司的实际题库。

Interview Practice Tracker

0/21 completed

1. Storage & Metadata

Explain the relationship between a Metastore, Catalog and Schema.

What is the difference between Managed and External Tables?

How does Managed Storage Location work?

Can multiple workspaces share one Metastore?

2. Workspace & Compute

What is the difference between a Workspace and a Metastore?

What is Workspace-Catalog Binding?

How does a Databricks Job access Unity Catalog data?

How would you isolate dev and prod environments?

3. Access Control

What permissions are required to query a Unity Catalog table?

Explain privilege inheritance in Unity Catalog.

What is the difference between Workspace Admin and Metastore Admin?

How would you implement least-privilege access?

4. CDC & Ingestion

Explain the architecture of Lakeflow Connect CDC ingestion.

Why do we need a staging volume?

What is the difference between CDC and query-based ingestion?

What is a High Water Mark?

How would you handle late-arriving records and deleted records?

5. Files & Storage

What is the difference between DBFS and Volumes?

When should you use a Volume instead of a Table?

Explain Storage Credentials and External Locations.

Why are DBFS mounts not recommended for new projects?

Reset progress

## 6.2 需要特别记住的 10 个知识点

|#|面试关键知识|一句话记忆|
|---|---|---|
|1|Metastore|Top-level metadata and governance container|
|2|Catalog|Logical isolation for domains or environments|
|3|Workspace Binding|Controls which workspaces can access catalogs|
|4|Managed vs External|Who manages the underlying data lifecycle|
|5|USE CATALOG / USE SCHEMA|Required usage privileges for accessing objects|
|6|Privilege Inheritance|Catalog and Schema grants can propagate to child objects|
|7|CDC Gateway|Continuously captures database changes|
|8|Staging Volume|Decouples change capture and downstream processing|
|9|Query-Based Connector|Uses cursor and high water mark|
|10|Volume|Governed storage for non-tabular files|
![[Pasted image 20260830141718.png]]
