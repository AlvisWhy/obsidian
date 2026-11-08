## 1. Cloud Logging 与 Log Analytics

### 1.1 Cloud Logging

Cloud Logging 是 Google Cloud 提供的集中式日志管理服务，主要用于：

- 收集应用程序日志、系统日志和平台日志。
    
- 集中存储和管理日志。
    
- 查询和过滤日志。
    
- 分析日志。
    
- 基于日志创建 Metrics 和 Alerts。
    
- 将日志导出到其他服务。
    

常见日志来源：

|日志来源|说明|
|---|---|
|Application Logs|应用程序产生的日志|
|System Logs|VM 操作系统日志|
|Platform Logs|Google Cloud 平台服务产生的日志|
|Audit Logs|用户及系统对资源的操作记录|
|Network Logs|VPC Flow Logs、Firewall Logs 等|

### 1.2 Logs Explorer

Logs Explorer 用于查询和检查日志。

主要功能：

- 使用 Logging Query Language（LQL）过滤日志。
    
- 按时间、Resource、Severity 等条件查询。
    
- 查看具体 Log Entry。
    
- 排查应用程序和基础设施故障。
    

### 1.3 Log Analytics

Log Analytics 是 Cloud Logging 提供的 SQL 日志分析能力，底层使用 BigQuery 技术。

主要特点：

- 可以使用 SQL 分析日志。
    
- 支持对大量日志进行聚合、统计和分析。
    
- 可以通过 Linked BigQuery Dataset 在 BigQuery 中访问日志。
    
- 不需要将日志复制或重新导入 BigQuery。
    

典型架构：

```text
Application / Google Cloud Services
              |
              v
         Cloud Logging
              |
              v
          Log Bucket
              |
              v
         Log Analytics
              |
              v
       Linked BigQuery Dataset
              |
              v
          SQL Analysis
```

### 1.4 Log Analytics 的存储与计费

需要区分传统日志导出和 Log Analytics：

**Log Analytics + Linked BigQuery Dataset**

- 日志仍然存储在 Cloud Logging Log Bucket 中。
    
- BigQuery Linked Dataset 提供对日志的访问，不会复制一份日志到 BigQuery。
    
- 不产生额外的 BigQuery 数据摄取费用。
    
- 不产生额外的 BigQuery 存储费用。
    
- 使用 BigQuery SQL 查询时，仍可能产生 BigQuery 查询计算费用。
    
- Cloud Logging 自身的日志摄取、存储和超出默认保留期的费用仍然适用。
    

**传统 Log Sink 导出到 BigQuery**

- 将日志路由到 BigQuery Dataset。
    
- 日志会作为 BigQuery 数据存储。
    
- 需要考虑 BigQuery 存储费用。
    
- 查询会产生 BigQuery 查询费用。
    

### 1.5 Log Analytics vs Export to BigQuery

|对比|Log Analytics|Log Sink → BigQuery|
|---|---|---|
|数据存储位置|Cloud Logging Log Bucket|BigQuery Dataset|
|是否复制日志|不需要|需要路由写入|
|BigQuery 存储费用|不产生额外费用|可能产生|
|BigQuery 查询费用|产生|产生|
|日志生命周期|由 Cloud Logging 管理|由 BigQuery 管理|
|适用场景|日志分析、SQL 查询|长期数据分析、与业务数据关联|

**面试问题：**

1. Cloud Logging 和 Log Analytics 有什么区别？
    
2. Log Analytics 如何使用 BigQuery SQL 分析日志？
    
3. Log Analytics 和传统 Log Sink 导出到 BigQuery 有什么区别？
    
4. 使用 Linked BigQuery Dataset 是否需要支付 BigQuery Storage Cost？
    
5. 如果需要将应用日志与业务交易数据进行关联分析，应该选择什么方案？
    
6. Cloud Logging 的日志保留周期由什么决定？
    

---

## 2. Cloud Audit Logs

### 2.1 核心概念

Cloud Audit Logs 用于记录 Google Cloud 资源的管理操作和访问行为，帮助回答：

- Who：谁执行了操作？
    
- What：执行了什么操作？
    
- Where：操作发生在哪个资源？
    
- When：操作发生在什么时候？
    

主要用于：

- 安全审计。
    
- 合规性检查。
    
- 故障排查。
    
- 检测未授权访问。
    
- 追踪资源配置变更。
    

Cloud Audit Logs 有四种类型：

1. Admin Activity Audit Logs
    
2. Data Access Audit Logs
    
3. System Event Audit Logs
    
4. Policy Denied Audit Logs
    

### 2.2 Admin Activity Audit Logs

记录用户主动执行的、会修改资源配置或元数据的操作。

例如：

- 创建 Compute Engine VM。
    
- 删除 Cloud Storage Bucket。
    
- 修改 IAM Policy。
    
- 修改网络配置。
    
- 修改数据库配置。
    

特点：

- 默认启用。
    
- 始终生成，不能禁用。
    
- 存储在 `_Required` Log Bucket。
    
- 保留期限为 400 天。
    
- 不收取 Cloud Logging 日志存储费用。
    

### 2.3 Data Access Audit Logs

记录用户对资源配置、元数据或用户数据的读取和修改操作。

分为三类：

|类型|说明|示例|
|---|---|---|
|ADMIN_READ|读取资源配置或元数据|查看 Bucket 配置、读取 Dataset 元数据|
|DATA_READ|读取用户提供的数据|查询 BigQuery 表、读取 GCS Object|
|DATA_WRITE|修改用户提供的数据|写入 BigQuery 表、创建或删除 GCS Object|

特点：

- 大多数 Google Cloud 服务默认关闭。
    
- BigQuery 是重要例外，其 Data Access Audit Logs 默认启用且不能禁用。
    
- 需要根据服务和资源配置启用。
    
- 可能产生较大的日志量。
    
- 可能产生额外 Cloud Logging 费用。
    
- 通常存储在 `_Default` Log Bucket。
    

**为什么默认关闭？**

因为 Data Access Logs 可能包含大量读取操作，特别是数据仓库、对象存储等服务，日志量可能非常大，因此需要根据审计需求进行配置。

### 2.4 System Event Audit Logs

记录 Google Cloud 系统或 Google 管理的服务代理自动执行的资源配置变更。

例如：

- MIG 自动扩容，创建新的 VM。
    
- MIG 自动缩容，删除 VM。
    
- Google Cloud 系统自动执行资源维护操作。
    

特点：

- 默认启用。
    
- 始终生成，不能禁用。
    
- 存储在 `_Required` Log Bucket。
    
- 保留期限为 400 天。
    
- 不收取 Cloud Logging 日志存储费用。
    

与 Admin Activity 的区别：

- Admin Activity：用户主动执行的管理操作。
    
- System Event：Google Cloud 系统自动执行的管理操作。
    

### 2.5 Policy Denied Audit Logs

记录由于安全策略或权限限制导致的访问拒绝事件。

例如：

- 用户没有 IAM Permission。
    
- 请求违反 VPC Service Controls。
    
- 请求违反其他安全策略。
    

特点：

- 默认生成。
    
- 不能直接禁用。
    
- 默认存储在 `_Default` Log Bucket。
    
- 可能产生 Cloud Logging 存储费用。
    
- 可以通过 Log Sink Exclusion Filter 排除，不将其存储在 Cloud Logging Log Bucket 中。
    

注意：排除日志存储不等于停止生成日志。

### 2.6 四种 Audit Logs 对比

|类型|记录内容|默认状态|能否禁用|默认存储位置|
|---|---|---|---|---|
|Admin Activity|用户主动修改资源配置|开启|不能|`_Required`|
|Data Access|读取配置、读取或修改数据|大多数服务关闭，BigQuery 例外|可以配置|`_Default`|
|System Event|Google Cloud 系统自动修改资源|开启|不能|`_Required`|
|Policy Denied|因策略限制导致访问失败|开启|不能直接禁用，可排除存储|`_Default`|

### 2.7 `_Required` vs `_Default` Log Bucket

Cloud Logging 默认创建两个 Log Bucket：

`**_Required**`

主要存储：

- Admin Activity Audit Logs。
    
- System Event Audit Logs。
    

特点：

- 固定保留 400 天。
    
- 不能删除。
    
- 不能修改其固定保留期限。
    
- 不能阻止这两类日志被写入。
    

`**_Default**`

主要存储：

- Data Access Audit Logs（如果已启用）。
    
- Policy Denied Audit Logs。
    
- 其他未被排除或路由到其他位置的日志。
    

特点：

- 默认保留期限通常为 30 天。
    
- 可以配置保留期限。
    
- 可以配置 Log Sink Exclusion。
    
- 可以将日志路由到其他 Log Bucket 或外部目的地。
    

### 2.8 Audit Logs 权限

|IAM Role|权限|
|---|---|
|Logs Viewer|查看 Admin Activity、System Event、Policy Denied 等日志|
|Private Logs Viewer|包含 Logs Viewer 权限，并允许查看 Data Access Logs|

如果用户只有 `roles/logging.viewer`，通常无法查看 `_Default` Bucket 中的 Data Access Audit Logs。

**面试问题：**

1. Cloud Audit Logs 有哪四种类型？
    
2. Admin Activity 和 Data Access 有什么区别？
    
3. 为什么 Data Access Logs 默认关闭？
    
4. 哪种 Audit Logs 不能禁用？
    
5. Policy Denied Logs 可以禁用吗？如何控制其存储？
    
6. `_Required` 和 `_Default` Log Bucket 有什么区别？
    
7. 如果需要审计谁读取了 BigQuery 表，应该启用或查询哪种日志？
    
8. 如果 MIG 自动创建了一台 VM，应该查看哪种 Audit Log？
    
9. Logs Viewer 和 Private Logs Viewer 有什么区别？
    

---

## 3. Log Routing 与 Log Sink

### 3.1 Log Router

Log Router 用于将 Cloud Logging 接收到的日志发送到不同的目的地。

核心组件是 Log Sink。

Log Sink 包含：

- **Filter**：决定哪些日志需要路由。
    
- **Destination**：指定日志发送到哪里。
    
- **Writer Identity**：Sink 用于向目标资源写入日志的身份。
    

### 3.2 Log Sink 支持的 Destination

常见目的地：

|Destination|用途|
|---|---|
|Cloud Logging Log Bucket|集中存储和管理日志|
|BigQuery Dataset|使用 SQL 进行长期分析|
|Cloud Storage Bucket|长期归档和低成本保存|
|Pub/Sub Topic|将日志发送给实时处理系统|

例如：

```text
Cloud Logging
      |
      v
   Log Router
      |
      +--------> Log Bucket
      |
      +--------> BigQuery
      |
      +--------> Cloud Storage
      |
      +--------> Pub/Sub
```

### 3.3 Project-level Sink

Project Sink 只处理当前 Project 范围内生成的日志。

例如：

```text
Project A
    |
    v
Project Sink
    |
    v
BigQuery Dataset
```

适用场景：

- 单个 Project 的日志归档。
    
- 单个 Project 的日志分析。
    
- 单个 Project 的日志转发。
    

### 3.4 Folder-level Aggregated Sink

Aggregated Sink 可以在 Folder 层级创建，用于收集该 Folder 及其子级资源产生的日志。

例如：

```text
Organization
     |
     v
  Folder A
     |
     +------ Project A
     |
     +------ Project B
     |
     +------ Project C
              |
              v
      Aggregated Sink
              |
              v
       Central Logging
```

特点：

- 可以集中管理多个 Project 的日志路由。
    
- 可以通过 Sink Filter 选择需要收集的日志。
    
- 可以将日志发送到中央 Project 的 Log Bucket、BigQuery、Cloud Storage 或 Pub/Sub。
    

### 3.5 Organization-level Aggregated Sink

Organization Sink 可以收集整个 Organization 及其子级资源的日志。

例如：

```text
Organization
     |
     +------ Folder A
     |          |
     |          +------ Project A
     |          +------ Project B
     |
     +------ Folder B
                |
                +------ Project C
                +------ Project D
                     |
                     v
              Organization Sink
                     |
                     v
             Central Logging Project
                     |
                     v
             BigQuery / GCS / Pub/Sub
```

适用场景：

- 企业集中式日志管理。
    
- 全组织安全审计。
    
- 集中式数据分析。
    
- 合规性和长期日志归档。
    

### 3.6 Aggregated Sink 的注意事项

1. Aggregated Sink 可以在 Organization 或 Folder 层级创建。
    
2. 可以设置 `includeChildren`，决定是否收集子级资源的日志。
    
3. Aggregated Sink 不会自动将日志存入目标 Project 的默认 Log Bucket。
    
4. 需要确保 Sink 的 Writer Identity 在目标资源上具有写入权限。
    
5. 如果目标是 BigQuery，需要赋予相应 Dataset 写入权限。
    
6. 可以通过 Filter 只路由指定类型的日志。
    

**重要：**

如果将 `_Required` 中的 Admin Activity 或 System Event 日志路由到另一个 Project，这些日志不会自动进入目标 Project 自己的 `_Required` 或 `_Default` Bucket。若希望在目标 Project 中通过 Cloud Logging 查询这些日志，需要额外配置目标 Project 的日志存储路由。

### 3.7 Project Sink vs Aggregated Sink

|对比|Project Sink|Folder Aggregated Sink|Organization Aggregated Sink|
|---|---|---|---|
|创建层级|Project|Folder|Organization|
|日志范围|当前 Project|Folder 及符合配置的子级|Organization 及符合配置的子级|
|集中管理|单个项目|多个项目|整个组织|
|典型用途|项目日志导出|部门级日志管理|企业级集中日志管理|

**面试问题：**

1. 什么是 Log Sink？它由哪些部分组成？
    
2. Log Sink 可以将日志发送到哪些目的地？
    
3. Project Sink 和 Aggregated Sink 有什么区别？
    
4. 如果企业有 100 个 Projects，需要将所有 Audit Logs 集中到一个 BigQuery Dataset，应该怎么设计？
    
5. Folder Aggregated Sink 和 Organization Aggregated Sink 有什么区别？
    
6. Aggregated Sink 的 `includeChildren` 有什么作用？
    
7. Log Sink 的 Writer Identity 为什么需要目标资源的 IAM 权限？
    
8. 如果将 Admin Activity Logs 路由到中央 Project，是否会自动存储到中央 Project 的 `_Default` Bucket？
    

---

## 4. 日志管理的典型数据工程架构

### 4.1 集中式日志分析

假设一个企业有多个 Google Cloud Projects：

```text
Organization
     |
     +------ Project A
     |          |
     |          +------ Application Logs
     |          +------ Audit Logs
     |
     +------ Project B
     |          |
     |          +------ Application Logs
     |          +------ Audit Logs
     |
     +------ Project C
                |
                +------ Application Logs
                +------ Audit Logs
                     |
                     v
             Aggregated Log Sink
                     |
                     v
             Central Logging Project
                     |
          +----------+----------+
          |          |          |
          v          v          v
       BigQuery     GCS       Pub/Sub
          |                     |
          v                     v
    SQL Analytics          Dataflow
          |                     |
          v                     v
    Reports / BI          Real-time Processing
```

### 4.2 不同需求如何选择

|需求|推荐方案|
|---|---|
|实时查看和排查日志|Logs Explorer|
|使用 SQL 分析 Cloud Logging 中的日志|Log Analytics|
|长期存储日志并进行复杂 SQL 分析|Log Sink → BigQuery|
|低成本长期归档|Log Sink → Cloud Storage|
|实时处理日志事件|Log Sink → Pub/Sub → Dataflow|
|集中管理多个 Project 的日志|Aggregated Sink|
|审计用户修改资源配置|Admin Activity Logs|
|审计用户读取 BigQuery 数据|Data Access Logs|
|追踪 Google Cloud 自动执行的资源变更|System Event Logs|
|追踪因安全策略导致的访问失败|Policy Denied Logs|

### 4.3 面试综合场景

**场景一：企业需要集中审计所有 Project 的资源变更。**

方案：

1. 在 Organization 层级创建 Aggregated Sink。
    
2. 配置 Filter，选择 Admin Activity Audit Logs。
    
3. 将日志发送到中央 Logging Project。
    
4. 根据需求存储在 Log Bucket 或 BigQuery。
    
5. 为 Sink Writer Identity 授予目标资源写入权限。
    

**场景二：安全团队需要知道谁读取了 BigQuery 敏感数据。**

方案：

1. 确认 BigQuery Data Access Logs 已启用。
    
2. 查询 `DATA_READ` 类型日志。
    
3. 使用 `Private Logs Viewer` 等适当权限查看日志。
    
4. 根据 Principal、Resource、Timestamp 等字段分析访问行为。
    
5. 可以使用 Aggregated Sink 集中收集多个 Project 的日志。
    

**场景三：数据团队需要实时分析应用程序错误日志。**

方案：

```text
Application
    |
    v
Cloud Logging
    |
    v
Log Sink
    |
    v
Pub/Sub
    |
    v
Dataflow
    |
    v
BigQuery
```

适用于实时日志分析、异常检测和告警处理。