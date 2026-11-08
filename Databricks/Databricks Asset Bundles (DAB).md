

Databricks Asset Bundles（现称 Declarative Automation Bundles，简称 DAB）的核心目标是：

将 Databricks 项目中的代码、Jobs、Pipelines、Clusters、权限和环境配置以 YAML + Python 等形式进行版本控制，并通过 CLI 部署到不同环境。

从 Data Engineer 的角度，可以把 DAB 分成以下 8 个模块：

1. Authentication & Profiles：身份认证与环境连接
    
2. Bundle Structure & Deployment Modes：项目结构与部署模式
    
3. Variables & Resource References：变量和资源引用
    
4. Job Parameters & Task Parameters：Job 和 Task 参数
    
5. Dynamic Value References：动态运行时参数
    
6. Task Values：Task 之间的数据传递
    
7. Python Wheel：Python 项目打包和依赖管理
    
8. Resource Binding：已有 Databricks 资源绑定
    

# 1. Authentication & Profiles

## 1.1 Databricks CLI Authentication

Databricks CLI 需要知道：

- Host：Databricks Workspace URL
    
- Authentication credentials：Token、OAuth 等认证信息
    
- Profile：保存的一组认证配置
    

这些配置通常保存在本地：

```
~/.databrickscfg
```

例如：

```
[DEV]
host = https://<dev-workspace>
token = <dev-token>

[TEST]
host = https://<test-workspace>
token = <test-token>

[PROD]
host = https://<prod-workspace>
token = <prod-token>
```

注意：

- Profile 是本地 CLI 的认证配置，不是 DAB 的环境配置本身。
    
- 不应该将 Token 提交到 GitHub。
    
- 实际生产环境更推荐 OAuth 或 Service Principal 等机器身份认证，而不是长期使用个人 PAT Token。
    
- Azure Databricks 也可以使用 Azure Service Principal 相关认证方式。
    

## 1.2 查看当前认证信息

```
databricks auth describe --profile DEV
```

查看当前 Profile 使用的认证方式和 Workspace 信息。

使用指定 Profile 执行命令：

```
databricks cluster create \
  --json @create-compute.json \
  --profile TEST
```

这里：

- `@create-compute.json`：从 JSON 文件读取 Cluster 配置。
    
- `--profile TEST`：使用 TEST 对应的认证信息。
    

## 1.3 面试问题

Q1. What is the purpose of a Databricks CLI authentication profile?

回答：

An authentication profile stores the workspace host and authentication credentials. It allows developers to connect to different Databricks workspaces without repeatedly specifying credentials.

Q2. How do you manage authentication across Dev, Test and Prod?

回答：

I use separate authentication profiles for different workspaces. For production, I prefer service principals or OAuth-based authentication, with credentials managed securely rather than hardcoded in configuration files.

Q3. Should you commit `.databrickscfg` to Git?

回答：

No. It contains sensitive authentication information and should never be committed to source control.

# 2. Bundle Structure & Deployment Modes

## 2.1 DAB 项目结构

一个典型的 Databricks Asset Bundle 项目：

```
dab_project/
│
├── databricks.yml
│
├── resources/
│   ├── jobs.yml
│   └── pipelines.yml
│
├── src/
│   ├── main.py
│   └── citibike_utils/
│       ├── __init__.py
│       └── transformations.py
│
├── tests/
│
├── setup.py
│
└── dist/
```

各个部分的职责：

|文件/目录|用途|
|---|---|
|`databricks.yml`|Bundle 主配置文件|
|`resources/`|定义 Jobs、Pipelines 等资源|
|`src/`|Python 项目代码|
|`tests/`|单元测试|
|`setup.py`|Python package 打包配置|
|`dist/`|生成的 wheel 文件|

DAB 的核心思想是 Infrastructure as Code（IaC）。

例如，Job 不需要每次都在 Databricks UI 中手动创建，而是可以在 YAML 中定义：

```
resources:
  jobs:
    daily_pipeline:
      name: daily_pipeline

      tasks:
        - task_key: ingest
          notebook_task:
            notebook_path: ../src/ingest.py
```

部署时由 Databricks CLI 根据配置创建或更新资源。

## 2.2 Development Mode vs Production Mode

DAB 有不同的 deployment mode：

- Development mode：开发环境。
    
- Production mode：生产环境。
    

例如：

```
bundle:
  name: citibike

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://<dev-workspace>

  prod:
    mode: production
    workspace:
      host: https://<prod-workspace>
```

### Development Mode

主要用于开发和测试。

常见行为：

- 资源名称可能添加开发者标识，避免不同开发者相互覆盖。
    
- 部署的资源可以带有开发环境标识。
    
- Job 通常会被配置为暂停 schedule。
    
- 方便开发者独立测试和迭代。
    

重要：Development mode 应当为每个开发者配置独立的开发部署身份或隔离方式。

例如，Alice 和 Bob 部署同一个 Bundle 时，应该避免两人覆盖对方的开发资源。

### Production Mode

主要用于正式环境。

常见特点：

- 使用固定的资源名称。
    
- 部署到共享生产环境。
    
- 不应该依赖个人开发者身份。
    
- 通常通过 CI/CD pipeline 部署。
    
- 需要明确配置权限和部署身份。
    

## 2.3 部署命令

```
databricks bundle validate -t dev
```

检查 YAML 配置和 Bundle 定义是否有效。

```
databricks bundle deploy -t dev
```

部署到 Dev。

```
databricks bundle deploy -t prod
```

部署到 Prod。

```
databricks bundle run daily_pipeline -t dev
```

运行已部署的 Job。

注意：`validate` 主要进行配置验证，不代表所有运行时资源、权限和业务逻辑一定正确。

## 2.4 面试问题

Q1. What is the difference between development and production deployment modes?

回答：

Development mode is designed for isolated development and testing. It can apply developer-specific resource naming and development-oriented behavior. Production mode is intended for stable shared resources and controlled deployments through CI/CD.

Q2. How would you prevent two developers from overwriting each other's Databricks resources?

回答：

I would use development mode and developer-specific deployment isolation, so each developer deploys their own resources rather than sharing the same development Job names.

Q3. How would you deploy a Databricks Job across Dev, Test and Prod?

回答：

I would define the Job once in the Bundle resources YAML and configure different targets in `databricks.yml`. Each target can provide its own workspace, variables, permissions and deployment settings.

# 3. Variables & Resource References

这是 DAB 中非常重要的一部分。

Variables 的作用是让同一份 YAML 配置可以适用于不同环境，而不需要复制多份配置文件。

## 3.1 Simple Variables

在 `databricks.yml` 中定义：

```
variables:
  catalog:
    description: Target catalog
    default: dev_catalog

targets:
  dev:
    variables:
      catalog: dev_catalog

  prod:
    variables:
      catalog: prod_catalog
```

在其他配置中引用：

```
resources:
  jobs:
    daily_job:
      name: daily_job

      tasks:
        - task_key: transform
          notebook_task:
            notebook_path: ../src/main.py
            base_parameters:
              catalog: ${var.catalog}
```

引用语法：

```
${var.catalog}
```

部署时覆盖：

```
databricks bundle deploy -t prod \
  --var="catalog=citibike_prod"
```

这样可以在不修改 YAML 文件的情况下切换 Catalog。

## 3.2 Complex Variables

Complex variable 可以存储复杂的 YAML 结构，例如一个包含多个配置项的对象。

```
variables:
  cluster_config:
    type: complex
    default:
      spark_version: "..."
      node_type_id: "..."
      num_workers: 2
```

引用：

```
${var.cluster_config}
```

可以将复杂变量作为一个整体传递给支持对应结构的资源配置。

注意：Complex variable 并不是所有 YAML 属性都能直接接受的万能对象，必须确保引用位置的数据结构符合目标字段的 Schema。

## 3.3 Resource Lookup

有时候不需要新建 Cluster，而是想引用 Workspace 中已经存在的 Cluster。

可以使用 Lookup：

```
variables:
  all_purpose_cluster:
    lookup:
      cluster: "my-existing-cluster"
```

引用：

```
${var.all_purpose_cluster}
```

Lookup 的主要逻辑：

1. 根据资源名称查找已有资源。
    
2. 获取对应资源 ID。
    
3. 将 ID 用于其他资源配置。
    

注意：Lookup 和 Resource Binding 不一样。

- Lookup：根据名称查找已有资源 ID。
    
- Binding：将 Bundle 中定义的资源与已有 Databricks 资源关联。
    

## 3.4 面试问题

Q1. Why do we use variables in Databricks Asset Bundles?

回答：

Variables allow us to reuse the same resource definitions across multiple environments. For example, Dev and Prod can use different catalogs, cluster configurations and storage locations without duplicating YAML files.

Q2. What is the difference between a simple variable and a complex variable?

回答：

A simple variable stores a single value, such as a catalog name or environment identifier. A complex variable stores structured data, such as a cluster configuration object.

Q3. What is the difference between lookup and binding?

回答：

Lookup resolves an existing resource by its name and retrieves its ID. Binding associates a Bundle resource with an existing resource in the workspace, allowing the Bundle to manage or reference that resource.

# 4. Job Parameters & Task Parameters

这是 Databricks Jobs 中最重要的参数传递机制之一。

首先要区分三个概念：

|类型|定义位置|作用范围|典型用途|
|---|---|---|---|
|Bundle Variables|`databricks.yml`|部署阶段|Dev/Prod Catalog|
|Job Parameters|Job 定义|整个 Job|processing_date|
|Task Parameters|Task 定义|单个 Task|文件路径、运行模式|

最重要的区别：

- Bundle Variables：部署时确定。
    
- Job Parameters：Job 每次运行时可以覆盖。
    
- Task Parameters：传递给特定 Task 的输入参数。
    

## 4.1 Job Parameters

Job Parameters 定义在 Job 层级，可以被多个 Tasks 使用。

例如：

```
resources:
  jobs:
    daily_pipeline:
      name: daily_pipeline

      parameters:
        - name: catalog
          default: dev_catalog

        - name: processing_date
          default: "2026-11-08"

      tasks:
        - task_key: bronze
          notebook_task:
            notebook_path: ../src/bronze.py

        - task_key: silver
          notebook_task:
            notebook_path: ../src/silver.py
```

Job Parameters 可以传递给多个 Tasks。

Notebook 中使用：

```
catalog = dbutils.widgets.get("catalog")processing_date = dbutils.widgets.get("processing_date")
```

### Job Parameters 的优势

假设每天运行一个 ETL Pipeline：

正常运行：

```
processing_date = 2026-11-08
```

如果某天数据处理失败，需要重新计算历史数据：

```
processing_date = 2026-11-01
```

不需要修改 Job 定义或者重新部署 Bundle，只需要在运行 Job 时覆盖参数。

CLI 示例：

```
databricks bundle run daily_pipeline -t dev -- \
  --processing_date=2026-11-01 \
  --catalog=prod_catalog
```

这里的 `--` 表示后面的参数是传递给 Job 的运行参数。

## 4.2 Backfill

Backfill = Reprocess historical data。

当数据 Pipeline 发生以下问题时，通常需要 Backfill：

- 某一天的数据没有成功处理。
    
- 数据源之前存在错误数据，现在需要重新计算。
    
- ETL Transformation 逻辑发生变化，需要重新生成历史数据。
    
- 之前的数据延迟到达，需要补充处理。
    

例如：

```
正常处理：

2026-11-01
2026-11-02
2026-11-03
2026-11-04
```

11 月 2 日的数据处理失败：

```
2026-11-01  Success
2026-11-02  Failed
2026-11-03  Success
2026-11-04  Success
```

Backfill 只需要重新处理 11 月 2 日，或者根据业务需求重新处理一个日期区间。

需要注意：

Backfill 不一定是 Databricks 自动提供的独立执行机制。通常是通过 Job Parameters、日期范围和 Pipeline 逻辑实现。

例如：

```
parameters:
  - name: start_date
    default: "2026-11-01"

  - name: end_date
    default: "2026-11-04"
```

Python：

```
start_date = dbutils.widgets.get("start_date")end_date = dbutils.widgets.get("end_date")df = spark.read.table("bronze.orders")filtered_df = df.filter(    (F.col("order_date") >= start_date) &    (F.col("order_date") <= end_date))
```

### 关于 `{{backfill.isodate}}`

你记录的：

```
{{backfill.isodate}}
```

不要把它当作所有 Databricks Job 都通用的 Backfill 参数。

Databricks 支持不同的动态日期引用，例如：

```
{{job.start_time.iso_date}}
```

但它代表 Job 开始运行的日期，不等于用户指定的历史 Backfill 日期。

实际工程中，建议通过 `start_date`、`end_date` 或 `processing_date` 这样的 Job Parameters 明确控制 Backfill 范围。

## 4.3 Task Parameters

Task Parameters 是单个 Task 的输入。

例如：

```
resources:
  jobs:
    daily_pipeline:
      tasks:

        - task_key: ingest

          notebook_task:
            notebook_path: ../src/ingest.py

            base_parameters:
              source_table: raw.orders
              output_table: bronze.orders
              processing_mode: incremental
```

Notebook 中读取：

```
source_table = dbutils.widgets.get("source_table")output_table = dbutils.widgets.get("output_table")processing_mode = dbutils.widgets.get("processing_mode")
```

`base_parameters` 是 Notebook Task 常见的参数传递方式。

其他 Task 类型有各自不同的参数配置方式：

|Task 类型|参数读取方式|
|---|---|
|Notebook|`dbutils.widgets.get()`|
|Python script|`sys.argv`|
|Python wheel|`argparse` / `sys.argv`|
|SQL|SQL task 参数配置|

例如 Python Script：

```
- task_key: transform

  spark_python_task:
    python_file: ../src/transform.py
    parameters:
      - "--processing_date"
      - "2026-11-08"
```

Python：

```
import sysprocessing_date = sys.argv[2]
```

实际项目中更推荐：

```
import argparseparser = argparse.ArgumentParser()parser.add_argument("--processing_date", required=True)args = parser.parse_args()processing_date = args.processing_date
```

## 4.4 Job Parameters 与 Task Parameters 的重要限制

这里有一个容易在面试中被问到的细节：

Databricks 不允许在同一个 Job 中同时定义 Job-level `parameters` 和 Task-level `base_parameters`。

如果定义了 Job Parameters，需要把相关参数统一放在 Job 层级，再通过参数 pushdown 或 Dynamic Value References 传给 Task。

另外，对于支持 key-value 参数的 Tasks：

- Job Parameters 会自动向下传递。
    
- 同名参数发生冲突时，Job Parameter 优先。
    

因此，不要简单地认为 Task Parameters 永远可以覆盖 Job Parameters。

Databricks on AWS+1

## 4.5 面试问题

Q1. What is the difference between bundle variables, job parameters and task parameters?

回答：

Bundle variables are resolved during deployment and are useful for environment-specific configurations.

Job parameters are defined at the job level and can be overridden for each run.

Task parameters are configured for individual tasks and provide task-specific inputs.

Q2. What is backfilling in a data pipeline? How would you implement it in Databricks?

回答：

Backfilling means reprocessing historical data to recover missing records or apply updated transformation logic.

I would parameterize the pipeline with a processing date or a start and end date. Then I would allow users to override these parameters when triggering a job run.

I would also ensure that the write operation is idempotent to avoid duplicate records.

Q3. How do you pass parameters to a Databricks Notebook task and a Python Script task?

回答：

For Notebook tasks, I use `dbutils.widgets.get()` to retrieve parameters.

For Python Script tasks, I pass arguments through the task configuration and retrieve them using `argparse` or `sys.argv`.

Q4. Can you define job parameters and task base_parameters in the same Job?

回答：

No. Databricks Bundle validation does not allow job-level `parameters` and task-level `base_parameters` to coexist in the same Job. I would centralize parameters at the job level.

# 5. Dynamic Value References

## 5.1 什么是 Dynamic Value References？

Dynamic Value References 是 Databricks Job 在运行过程中提供的动态变量。

它们可以引用：

- Job metadata
    
- Job run metadata
    
- Task metadata
    
- Job Parameters
    
- Task Values
    
- Job execution information
    

与 Bundle Variables 不同，它们不是部署阶段解析，而是在 Job 运行时解析。

语法：

```
{{ }}
```

例如：

```
parameters:
  - name: job_id
    default: "{{job.id}}"

  - name: job_name
    default: "{{job.name}}"

  - name: run_id
    default: "{{job.run_id}}"
```

常见动态引用：

|Dynamic Reference|含义|
|---|---|
|`{{job.id}}`|Job ID|
|`{{job.name}}`|Job Name|
|`{{job.run_id}}`|当前 Job Run ID|
|`{{job.repair_count}}`|Job 修复次数|
|`{{job.start_time.iso_date}}`|Job 开始日期|
|`{{job.parameters.date}}`|Job Parameter 的值|
|`{{job.trigger.type}}`|Job 触发类型|

注意：`job.id` 和 `job.run_id` 是两个不同概念。

- Job ID：Job 本身的唯一 ID。
    
- Run ID：某一次执行的唯一 ID。
    

同一个 Job 可以执行很多次，因此 Job ID 不变，但 Run ID 不同。

## 5.2 如何传递给 Python Script？

Dynamic Value References 不能直接在 Python 代码里使用。

它们需要先在 Job 或 Task 配置中解析，再传递给 Python。

YAML：

```
- task_key: transform

  spark_python_task:
    python_file: ../src/transform.py

    parameters:
      - "{{job.id}}"
      - "{{job.name}}"
      - "{{job.run_id}}"
```

Python：

```
import sysjob_id = sys.argv[1]job_name = sys.argv[2]run_id = sys.argv[3]print(job_id)print(job_name)print(run_id)
```

执行时，Databricks 会先解析 `{{ }}`，然后将实际字符串传给 Python。

例如：

```
sys.argv[1] = "123456"
sys.argv[2] = "daily_pipeline"
sys.argv[3] = "987654"
```

## 5.3 Dynamic Value References vs Bundle Variables

这是非常值得掌握的面试对比。

|对比|Bundle Variables|Dynamic Value References|
|---|---|---|
|语法|`${var.name}`|`{{job.id}}`|
|解析时间|Deploy time|Runtime|
|主要作用|环境配置|运行时上下文|
|典型例子|Catalog、Cluster size|Job ID、Run ID|
|是否能随每次运行改变|需要重新部署变量值|自动根据当前运行变化|

例如：

```
# Deploy time
catalog: ${var.catalog}

# Runtime
run_id: "{{job.run_id}}"
```

## 5.4 面试问题

Q1. What are dynamic value references in Databricks Jobs?

回答：

Dynamic value references allow us to access runtime information such as job IDs, run IDs, job parameters and task values. They are resolved when the job runs and can be used to pass runtime context into tasks.

Q2. What is the difference between `job.id` and `job.run_id`?

回答：

`job.id` identifies the Job definition, whereas `job.run_id` identifies a specific execution of that Job. Multiple runs of the same Job share the same Job ID but have different Run IDs.

Q3. Can you directly use `{{job.run_id}}` inside a Python script?

回答：

No. Dynamic references are resolved in supported Job or Task configuration fields. I would pass the reference as a task parameter and retrieve the resolved value from the Python script arguments.

# 6. Task Values

## 6.1 什么是 Task Values？

Task Values 用于在同一个 Job 的不同 Tasks 之间传递运行时生成的数据。

例如，一个 ETL Pipeline：

Task 1 — Ingest

读取数据，计算实际处理日期和记录数量

Generate Task Values

taskValues.set()

Task 2 — Transform

读取上一个 Task 产生的参数

taskValues.get()

Task 3 — Validate

根据上游生成的数据执行验证

与 Job Parameters 的区别：

- Job Parameters：在 Job 运行前或运行时提供的输入。
    
- Task Values：在 Job 运行过程中，由某个 Task 动态生成的输出。
    

## 6.2 Task Values — Set

例如 Notebook Task 1：

```
# Calculate number of processed recordsrecord_count = df.count()# Expose value to downstream tasksdbutils.jobs.taskValues.set(    key="record_count",    value=record_count)
```

这里：

- `key`：Value 的名称。
    
- `value`：实际传递的数据。
    

注意：Task Values 主要用于传递轻量级运行信息，例如日期、数量、状态和路径，不应该用来传递大型 DataFrame。

## 6.3 Task Values — Get

假设 Task 2 依赖 Task 1：

```
resources:
  jobs:
    etl_job:
      tasks:

        - task_key: ingest
          notebook_task:
            notebook_path: ../src/ingest.py

        - task_key: transform
          depends_on:
            - task_key: ingest

          notebook_task:
            notebook_path: ../src/transform.py

            base_parameters:
              record_count: "{{tasks.ingest.values.record_count}}"
```

Task 2 的 Notebook：

```
record_count = int(    dbutils.widgets.get("record_count"))
```

注意这里有两个不同的机制：

1. `dbutils.jobs.taskValues.set()`：在上游 Task 中生成 Value。
    
2. `{{tasks.ingest.values.record_count}}`：在下游 Task 配置中引用上游 Value。
    

## 6.4 Python Script 中使用 Task Values

你原来记录的 Python Script 代码思路是对的，但 API 名称需要修正。

正确的 Python API 是：

```
dbutils.jobs.taskValues.get()
```

而不是：

```
dbutils.taskValue.get()
```

例如：

```
from pyspark.sql import SparkSessionfrom pyspark.dbutils import DBUtilsspark = SparkSession.builder.getOrCreate()dbutils = DBUtils(spark)record_count = dbutils.jobs.taskValues.get(    taskKey="ingest",    key="record_count",    default=0,    debugValue=0)print(record_count)
```

参数含义：

|参数|作用|
|---|---|
|`taskKey`|上游 Task 的名称|
|`key`|上游设置的 Value 名称|
|`default`|Value 不存在时返回的默认值|
|`debugValue`|在非 Job 环境下调试时使用的值|

`debugValue` 主要用于本地或非 Job 上下文的调试，不代表生产 Job 中可以跳过依赖关系。

## 6.5 Task Values vs Job Parameters

|对比|Job Parameters|Task Values|
|---|---|---|
|数据来源|用户、调度器、API|上游 Task|
|生成时间|Job 运行前/启动时|Job 运行过程中|
|主要用途|控制 Pipeline 输入|传递执行结果|
|传递方向|Job → Tasks|Upstream Task → Downstream Task|
|典型数据|日期、Catalog、运行模式|记录数量、文件路径、处理状态|

## 6.6 面试问题

Q1. How do you pass runtime information between tasks in Databricks Jobs?

回答：

I use Task Values. An upstream task can use `dbutils.jobs.taskValues.set()` to expose a value, and downstream tasks can retrieve it through dynamic value references or the Task Values API.

Q2. What is the difference between Job Parameters and Task Values?

回答：

Job parameters provide input values to a job, while task values are generated during execution and allow upstream tasks to share runtime information with downstream tasks.

Q3. Would you use Task Values to transfer a large DataFrame between tasks?

回答：

No. Task Values are intended for small metadata values, such as record counts, file paths or processing status. For large datasets, I would write the data to Delta tables or cloud storage and pass the table or file location to downstream tasks.

# 7. Python Wheel — Package Management

这是 Databricks 工程化开发中非常重要的知识点，尤其是多个 Job 需要复用相同 Python 代码时。

## 7.1 为什么需要 Python Wheel？

假设你有一个 Data Engineering 项目：

```
src/
├── ingestion.py
├── transformation.py
└── citibike_utils/
    ├── __init__.py
    ├── cleaning.py
    └── validation.py
```

多个 Jobs 都需要使用 `citibike_utils`。

如果每个 Job 都复制一份 Python 文件，会产生以下问题：

- 代码重复。
    
- 难以维护。
    
- 不同 Jobs 可能使用不同版本的代码。
    
- 修改一个公共函数，需要同步修改多个地方。
    

解决办法：

将 Python 项目打包成 `.whl` 文件，然后安装到 Databricks Cluster。

Wheel = Python 的一种二进制分发包格式（Binary Distribution Format）。

它可以将 Python Package 及其代码组织成一个可安装的文件。

## 7.2 使用 setuptools 创建 Wheel

你的原始代码有一个小错误：

```
package=find_packages(where="./src")
```

应该是：

```
packages=find_packages(where="src")
```

完整示例：

```
from setuptools import setup, find_packagessetup(    name="dab_project",    version="0.0.1",    description="Databricks project utilities",    packages=find_packages(where="src"),    package_dir={"": "src"},    install_requires=[        "setuptools"    ],    entry_points={        "console_scripts": [            "main=citibike_utils.main:main"        ]    })
```

项目目录：

```
dab_project/
│
├── setup.py
│
└── src/
    └── citibike_utils/
        ├── __init__.py
        ├── main.py
        └── transformations.py
```

这里几个重要参数：

|参数|作用|
|---|---|
|`name`|Package 名称|
|`version`|Package 版本|
|`packages`|需要打包的 Python Packages|
|`package_dir`|指定 Package 所在目录|
|`install_requires`|安装时需要的依赖|
|`entry_points`|暴露可执行的命令入口|

其中：

```
find_packages(where="src")
```

会自动查找 `src` 下的 Python Packages。

注意：通常需要在 Package 目录中包含 `__init__.py`。

## 7.3 Build Wheel

安装依赖：

```
pip install setuptools wheel
```

构建：

```
python setup.py bdist_wheel
```

生成：

```
dist/
└── dab_project-0.0.1-py3-none-any.whl
```

也可以使用现代 Python 项目打包方式，例如 `pyproject.toml` 配合 `python -m build`。

## 7.4 安装和 Import

安装：

```
pip install dist/dab_project-0.0.1-py3-none-any.whl
```

然后可以直接 Import：

```
from citibike_utils import transformations
```

注意：

- Wheel 中的 distribution name 是 `dab_project`。
    
- Python import name 是 `citibike_utils`。
    

两者不一定相同。

## 7.5 在 Databricks Asset Bundles 中部署 Wheel

首先，在 `databricks.yml` 中定义 Artifact：

```
bundle:
  name: citibike

artifacts:
  default:
    type: whl
    build: python setup.py bdist_wheel
    path: .
```

这表示：

1. Bundle 部署时执行 Build 命令。
    
2. 使用 setuptools 创建 Wheel。
    
3. 生成文件放入 `dist/`。
    
4. Bundle 将 Artifact 上传到 Workspace。
    

接下来定义 Job：

```
resources:
  jobs:
    citibike_job:

      name: citibike_job

      tasks:
        - task_key: wheel_task

          python_wheel_task:
            package_name: dab_project
            entry_point: main

          libraries:
            - whl: ../dist/*.whl

          job_cluster_key: wheel_cluster

      job_clusters:
        - job_cluster_key: wheel_cluster

          new_cluster:
            spark_version: "..."
            node_type_id: "..."
            num_workers: 2
```

这里有几个重要概念：

|配置|含义|
|---|---|
|`python_wheel_task`|执行 Python Wheel Task|
|`package_name`|对应 `setup.py` 中的 `name`|
|`entry_point`|对应 `entry_points` 中的命令名称|
|`libraries`|指定需要安装的 Wheel|
|`job_cluster_key`|指定 Task 使用的 Job Cluster|

## 7.6 Entry Point 的理解

例如：

```
entry_points={    "console_scripts": [        "main=citibike_utils.main:main"    ]}
```

代表：

```
main = citibike_utils.main:main
```

解释：

- `citibike_utils.main`：Python Module。
    
- 第二个 `main`：Module 内部的函数。
    

Python：

```
def main():    print("Starting Citibike pipeline")
```

Databricks YAML：

```
python_wheel_task:
  package_name: dab_project
  entry_point: main
```

Databricks 就可以执行这个入口。

注意：标准 Python Wheel Task 需要配置一个有效的 entry point。 不能简单地删除 `entry_point` 就期待 Databricks 自动执行 Package 中的代码。

## 7.7 Wheel 是不是只需要安装一次？

这里需要区分：

- Python Package 是一个可复用的 Artifact。
    
- 每个 Task 使用的 Compute 都必须能够访问和安装这个 Package。
    

如果多个 Tasks 使用不同的 Compute，不能假设一个 Task 安装的依赖会自动出现在另一个 Task 的环境中。

通常应该在需要使用该 Wheel 的 Task 上声明 Library，或者使用合适的 Job 级依赖配置。

你记录的 `whl_setup` Task 也不是一个必须的安装初始化 Task。`python_wheel_task` 本身就是执行 Wheel 入口的 Task，配置 `libraries` 后，Databricks 会为该 Task 准备依赖。

## 7.8 Wheel vs Notebook

|对比|Notebook|Python Wheel|
|---|---|---|
|代码组织|Notebook 文件|Python Package|
|代码复用|相对有限|很适合|
|版本管理|Notebook / Git|Package version|
|适合场景|ETL 开发、交互式分析|模块化项目、共享业务逻辑|
|执行方式|Notebook Task|Python Wheel Task|

## 7.9 面试问题

Q1. Why do you package Python code as a wheel in Databricks?

回答：

A Python wheel allows us to package reusable Python code into a distributable artifact. Multiple Databricks Jobs can use the same package, improving code reuse, maintainability and version control.

Q2. What is the difference between `package_name` and `entry_point`?

回答：

`package_name` identifies the Python distribution defined in the setup configuration, while `entry_point` identifies the executable function registered in the package's entry points.

Q3. How would you share common transformation logic across multiple Databricks Jobs?

回答：

I would organize common logic into a Python package, build it into a wheel, and install that wheel as a dependency for the relevant Job tasks.

Q4. Why use a wheel instead of copying Python files between notebooks?

回答：

A wheel provides a reusable and versioned package. It avoids duplicated code and makes it easier to test, maintain and deploy shared business logic.

# 8. Resource Binding

## 8.1 什么是 Resource Binding？

实际工程中，有时候 Databricks Workspace 已经存在某个 Job。

例如：

```
Workspace:

Job ID: 123456
Job Name: daily_ingestion
```

后来你希望通过 DAB 管理这个 Job，而不是在 Workspace UI 中手动维护。

这时可以使用 Resource Binding。

它的作用是：

将 Bundle 中定义的 Resource 与 Workspace 中已经存在的 Resource 关联起来。

例如：

```
resources:
  jobs:
    daily_ingestion:
      name: daily_ingestion

      tasks:
        - task_key: ingestion
          notebook_task:
            notebook_path: ../src/ingestion.py
```

绑定：

```
databricks bundle deployment bind \
  daily_ingestion \
  123456 \
  --target dev
```

这里：

- `daily_ingestion`：Bundle 中定义的 Resource Key。
    
- `123456`：Workspace 中已经存在的 Job ID。
    
- `--target dev`：指定部署环境。
    

注意：第一个参数是 Bundle resource key，而不是随便填写一个 YAML 配置 ID。

绑定后，Bundle 就可以将该 Resource 与已有 Job 关联，在后续部署中管理它。

## 8.2 Binding vs Lookup

|对比|Resource Binding|Lookup|
|---|---|---|
|目的|关联已有 Bundle Resource|查找已有 Workspace Resource|
|主要输入|Resource Key + Resource ID|Resource Name|
|结果|建立部署资源关联|获取资源 ID|
|典型场景|将已有 Job 纳入 IaC 管理|引用已有 Cluster|

举个例子：

Lookup：

```
variables:
  existing_cluster:
    lookup:
      cluster: "my-existing-cluster"
```

根据 Cluster 名称找到 ID。

Binding：

```
databricks bundle deployment bind \
  daily_ingestion \
  123456 \
  --target dev
```

将现有 Job 纳入 Bundle 的管理。

## 8.3 面试问题

Q1. What is resource binding in Databricks Asset Bundles?

回答：

Resource binding allows us to associate an existing workspace resource, such as a Job, with a resource defined in a Bundle. This is useful when migrating manually created resources into Infrastructure as Code management.

Q2. What is the difference between resource binding and resource lookup?

回答：

Binding associates an existing resource with a Bundle resource. Lookup searches for an existing resource by name and retrieves its ID, which can then be referenced by other resources.

Q3. Why would you use DAB instead of manually creating Jobs in the Databricks UI?

回答：

DAB provides Infrastructure as Code. It allows us to version-control resource definitions, review changes through pull requests, deploy consistently across environments and integrate with CI/CD pipelines.

# 9. Databricks CLI 配置文件：Here Document

你最后记录的部分是 Linux Shell 的 Here Document（Here Doc），它不是 Databricks 特有功能。

## 9.1 什么是 Here Document？

Here Document 可以把多行文本输入到命令中。

例如：

```
cat <<EOF > ~/.databrickscfg

[TEST]

host = https://<workspace-url>

azure_tenant_id = <tenant-id>

EOF
```

执行逻辑：

1. `cat`：读取标准输入。
    
2. `<<EOF`：告诉 Shell，直到遇到 `EOF` 才结束输入。
    
3. `>`：将内容重定向到指定文件。
    
4. `~/.databrickscfg`：Databricks CLI 配置文件。
    

最终生成：

```
[TEST]
host = https://<workspace-url>
azure_tenant_id = <tenant-id>
```

注意：

- `>` 会覆盖文件原来的内容。
    
- `>>` 会追加内容。
    
- `EOF` 是自定义结束标记，可以换成其他字符串。
    
- 这里的 `azure_tenant_id` 只是认证配置的一部分，不代表完整的 Azure Service Principal 认证配置。
    

## 9.2 面试问题

Q1. What is a Here Document in Linux?

回答：

A Here Document allows us to pass multiple lines of input to a command through standard input. It is commonly used in shell scripts to generate configuration files.

Q2. What is the difference between `>` and `>>`?

回答：

`>` overwrites the target file, whereas `>>` appends content to the existing file.

# 10. 整体知识体系总结：面试前需要掌握的核心逻辑

前面 9 个模块可以整合成一个完整的 Data Engineering CI/CD 工作流。

## 10.1 DAB 完整工作流程

1. Authentication

databrickscfg / CLI Profile / OAuth

2. Develop Bundle

databricks.yml / Resources YAML / Python code

3. Configure

Variables / Parameters / Resource References

4. Package

Build Python Wheel / Dependencies

5. Validate & Deploy

bundle validate → bundle deploy

6. Run Job

Job Parameters / Dynamic References

7. Execute Tasks

Task Values / Dependencies / Backfill

## 10.2 三种参数机制最重要的区别

这是整个笔记最值得记忆的部分：

|机制|什么时候确定|例子|
|---|---|---|
|Bundle Variables|Deploy time|Catalog、Schema、Cluster|
|Job Parameters|Job run time|Processing date、Backfill range|
|Task Values|Task execution time|Record count、Output path|

另外，Dynamic Value References 是一种引用运行时数据的语法，不是第四种独立的参数存储机制。

## 10.3 荷兰 Data Engineer 面试重点问题

以下是基于上述知识点整理的综合面试题，重点考查工程实践，而不是单纯记忆 CLI 命令。

Interview practice

0/8 completed

1How would you deploy the same Databricks Job to Dev and Prod?

2How would you handle a failed ETL pipeline that needs historical data reprocessing?

3How do you share Python transformation logic between multiple Jobs?

4How can you pass the output of one task to another?

5How would you prevent developers from overwriting each other's resources?

6What is the difference between deployment-time and runtime configuration?

7How would you migrate an existing Databricks Job into DAB?

8What would you include in a production Databricks CI/CD pipeline?

Reveal all answers

## 10.4 还需要补充学习的 DAB 知识点

如果目标是荷兰的 Junior / Mid-level Data Engineer，我建议接下来重点补充：

|优先级|知识点|为什么重要|
|---|---|---|
|高|CI/CD with GitHub Actions / Azure DevOps|生产部署和团队协作|
|高|Bundle permissions|Job、Cluster 的权限控制|
|高|Job dependencies & repair|生产 Pipeline 容错|
|高|Unity Catalog integration|Catalog、Schema、权限管理|
|高|Secrets management|安全管理凭证|
|中|Bundle artifacts|Wheel、JAR 等构建与发布|
|中|Deployment state|Bundle 如何追踪已部署资源|
|中|Terraform vs DAB|Databricks 资源与云基础设施管理|
|中|Serverless Jobs|Compute 管理与成本优化|

特别建议： 你需要把 DAB 和 GitHub Actions 结合起来学习。面试时仅仅知道 `databricks bundle deploy` 不够，还需要能够解释如何通过 Pull Request、自动测试、环境隔离和 Service Principal 将代码安全地部署到生产环境。