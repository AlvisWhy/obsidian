Databricks Commands 可以分成两个部分：

1. Notebook Magic Commands：以 `%` 开头，用于在 Notebook 中执行不同类型的命令。
    
2. DBUtils（Databricks Utilities）：通过 Python API 操作文件系统、Widgets、Notebook、Secrets 等。
    

从 Data Engineer 的角度，最重要的是理解它们的用途、使用限制，以及如何在 ETL Pipeline 中应用。

# 1. Notebook Magic Commands

Magic Commands 是 Databricks Notebook 提供的特殊命令，用于切换执行环境或调用其他 Notebook 功能。

常见命令：

|Command|用途|
|---|---|
|`%fs`|操作文件系统|
|`%sh`|执行 Shell 命令|
|`%pip`|安装 Python Package|
|`%run`|执行另一个 Notebook|
|`%sql`|执行 SQL|
|`%md`|创建 Markdown 内容|
|`%scala`|执行 Scala|
|`%python`|执行 Python|

## 1.1 `%fs` — File System

用于操作 Databricks 支持的文件系统。

例如：

```
%fs ls /
```

查看根目录下的文件。

```
%fs ls /Volumes/main/default/raw/
```

查看 Unity Catalog Volume 中的文件。

```
%fs mkdirs /tmp/my_folder
```

创建目录。

```
%fs head /tmp/data.csv
```

查看文件内容。

```
%fs rm -r /tmp/my_folder
```

删除目录及其内容。

### `%fs` 与 Unity Catalog 的关系

这是你记录中非常重要的一点，需要进一步区分。

Unity Catalog 管理的表不仅包含数据文件，还包含：

- Table metadata
    
- Schema information
    
- Storage location
    
- Permissions
    
- Ownership
    
- Transaction metadata
    

例如：

```
Catalog: main
Schema: bronze
Table: orders
```

底层可能存储在 Cloud Storage 中：

```
s3://bucket/warehouse/orders/
```

如果直接使用文件系统命令移动、删除或修改 UC Managed Table 的底层文件，可能导致：

- Table metadata 与实际数据不一致。
    
- Delta transaction log 不完整。
    
- Table 查询失败。
    
- 数据权限和管理出现问题。
    

最佳实践：不要通过 `%fs` 或 `dbutils.fs` 直接修改 Unity Catalog 管理表的底层存储文件。

应该使用：

```
DROP TABLE main.bronze.orders;
```

或者：

```
df.write.format("delta").mode("overwrite").saveAsTable(    "main.bronze.orders")
```

根据具体需求使用 SQL 或 Spark DataFrame API 管理表。

补充：

并不是所有 UC 相关文件都不能通过文件系统 API 操作。UC Volume 是专门用于管理非表格文件的资源，可以通过 Volume 路径访问，例如：

```
/Volumes/main/default/raw/
```

但仍然需要遵守 Unity Catalog 的权限和路径管理规则。

## 1.2 `%sh` — Shell Command

用于在 Notebook 中执行 Linux Shell 命令。

例如：

```
%sh
pwd
```

查看当前工作目录。

```
%sh
ls -la
```

查看当前目录文件。

```
%sh
echo "Hello Databricks"
```

输出文本。

```
%sh
pip --version
```

查看 pip 版本。

适用场景：

- 检查文件。
    
- 执行 Bash 脚本。
    
- 调用系统命令。
    
- 调试环境。
    

注意：

`%sh` 执行的是计算环境上的 Shell 命令，不等于在所有 Worker 上同时执行。通常它运行在当前 Notebook 的 Driver 环境中。

## 1.3 `%pip` — Python Package Installation

用于在 Notebook 环境中安装 Python Packages。

例如：

```
%pip install pandas
```

安装 pandas。

```
%pip install requests==2.32.0
```

安装指定版本。

```
%pip install -r requirements.txt
```

根据 requirements 文件安装依赖。

安装后：

```
import pandas as pdimport requests
```

### `%pip` 与 `pip` 的区别

|对比|`%pip`|`%sh pip install`|
|---|---|---|
|使用环境|Notebook Python 环境|Shell 环境|
|Notebook 支持|是|间接|
|Python 环境管理|与 Notebook Python 环境集成|不保证与 Notebook Kernel 一致|
|推荐用途|Notebook 中安装 Python 依赖|特殊系统或环境操作|

补充：

在 Databricks 中使用 `%pip` 安装或更新 Library 后，有些情况下需要重启 Python 环境才能让新依赖生效。

```
dbutils.library.restartPython()
```

注意：不要把 `%pip install` 当成生产环境唯一的依赖管理方法。生产 Jobs 更推荐在 Job 配置中声明 Library 或使用版本化的 Wheel。

## 1.4 `%run` — Execute Another Notebook

`%run` 用于执行另一个 Notebook，并将其中定义的变量和函数引入当前 Notebook 的执行环境。

例如：

Notebook A：`utils`

```
def clean_data(df):    return df.dropna()tax_rate = 0.21
```

Notebook B：

```
%run ./utils
```

之后可以直接使用：

```
cleaned_df = clean_data(df)print(tax_rate)
```

### `%run` 的重要特点

1. 会执行整个被引用的 Notebook。
    
2. 被引用 Notebook 中定义的变量和函数可以在当前 Notebook 中使用。
    
3. 通常用于共享 Notebook 代码。
    
4. 它不是简单的 Python module import。
    

### `%run` vs Python Import

|对比|`%run`|`import`|
|---|---|---|
|执行对象|Notebook|Python Module|
|执行代码|整个 Notebook|Module 首次导入时执行顶层代码|
|共享变量|直接进入当前 Notebook 命名空间|通过 module namespace 访问|
|适合场景|Notebook 之间共享逻辑|模块化 Python 项目|

例如：

```
%run ./file_name
```

如果 `file_name` 是另一个 Notebook，Databricks 会执行它。

生产环境中，如果公共逻辑比较复杂，建议整理为 Python Package，而不是过度依赖 `%run`。

# 2. DBUtils — Databricks Utilities

DBUtils 是 Databricks 提供的一组 API，用于在 Notebook 中操作 Databricks 环境。

常用模块：

|Module|用途|
|---|---|
|`dbutils.fs`|文件系统操作|
|`dbutils.widgets`|Notebook 参数化|
|`dbutils.notebook`|Notebook 调用与返回值|
|`dbutils.secrets`|安全读取 Secrets|
|`dbutils.jobs`|Job 相关功能，例如 Task Values|
|`dbutils.library`|Library 管理（部分功能已弃用）|

可以通过以下命令查看帮助：

```
dbutils.help()
```

查看具体模块：

```
dbutils.fs.help()dbutils.widgets.help()dbutils.notebook.help()
```

查看具体方法：

```
dbutils.fs.help("put")
```

## 2.1 `dbutils.fs` — File System Operations

用于在 Databricks 中操作文件和目录。

### 1. ls — List files

```
dbutils.fs.ls("/")
```

查看文件列表。

等价于：

```
%fs ls /
```

推荐在 Unity Catalog Volume 中使用：

```
dbutils.fs.ls(    "/Volumes/main/default/raw/")
```

### 2. mkdirs — Create directory

```
dbutils.fs.mkdirs(    "/Volumes/main/default/raw/config")
```

创建目录。

特点：

- 如果目录不存在则创建。
    
- 可以自动创建父目录。
    
- 如果目录已经存在，不会因为重复创建而报错。
    

### 3. put — Write a file

你原来的代码有一个需要修正的地方：`put()` 的第一个参数必须是文件路径，而不是目录路径。

正确示例：

```
config_string = """source = salesformat = deltaenvironment = dev"""dbutils.fs.put(    "/Volumes/main/default/raw/config/config.txt",    config_string,    True)
```

参数：

```
dbutils.fs.put(file, contents, overwrite)
```

|参数|含义|
|---|---|
|`file`|文件完整路径|
|`contents`|要写入的字符串|
|`overwrite`|是否覆盖已有文件|

`True` 表示允许覆盖已有文件。

注意：`put()` 适合写入小型文本文件，不适合写入大型数据集。大型数据应该使用 Spark DataFrame API。

### 4. cp — Copy files

```
dbutils.fs.cp(    "/Volumes/main/default/raw/source.csv",    "/Volumes/main/default/backup/source.csv")
```

复制文件。

递归复制目录：

```
dbutils.fs.cp(    "/Volumes/main/default/raw/",    "/Volumes/main/default/backup/",    True)
```

### 5. mv — Move files

```
dbutils.fs.mv(    "/Volumes/main/default/raw/source.csv",    "/Volumes/main/default/backup/source.csv")
```

移动文件。

### 6. rm — Remove files

```
dbutils.fs.rm(    "/Volumes/main/default/raw/source.csv")
```

删除文件。

递归删除目录：

```
dbutils.fs.rm(    "/Volumes/main/default/raw/",    True)
```

### 7. head — Read file content

```
dbutils.fs.head(    "/Volumes/main/default/raw/config.txt")
```

读取文件开头的内容。

### 常用命令总结

|Command|作用|
|---|---|
|`ls()`|查看文件|
|`mkdirs()`|创建目录|
|`put()`|写入字符串文件|
|`cp()`|复制文件|
|`mv()`|移动文件|
|`rm()`|删除文件|
|`head()`|读取文件开头|

官方文档：[Databricks File System Utilities](https://docs.databricks.com/aws/en/dev-tools/databricks-utils?utm_source=chatgpt.com)。

Databricks on AWS+1

### 重要补充：DBFS vs Unity Catalog Volumes

现在的 Databricks 项目应该优先使用 Unity Catalog Volumes 管理非表格文件。

|存储方式|路径示例|建议|
|---|---|---|
|DBFS Root|`dbfs:/FileStore/`|Legacy，不推荐新项目使用|
|DBFS Mount|`/mnt/data/`|Legacy，不推荐新项目使用|
|UC Volume|`/Volumes/main/default/raw/`|推荐|
|Workspace Files|Workspace 文件路径|适合代码和项目文件|
|Cloud Storage|`s3://...`、`abfss://...`|适合云端对象存储|

需要特别理解：

- Unity Catalog Managed Table：应该通过 SQL 或 Spark Table API 管理。
    
- Unity Catalog External Table：应该通过受支持的表管理接口管理。
    
- Unity Catalog Volume：用于管理非表格文件，可以通过 `dbutils.fs` 操作。
    

不要直接移动或删除 UC Managed Table 的底层文件，因为这可能破坏 Delta Transaction Log 和表的数据一致性。

另外，DBFS Root 和 DBFS Mount 已经被 Databricks 标记为不推荐使用，新项目应该优先采用 Unity Catalog Volumes 和 External Locations。

Databricks on AWS+2

## 2.2 面试问题

Q1. What is the difference between `%fs` and `dbutils.fs`?

回答：

They provide essentially the same filesystem functionality. `%fs` is a notebook magic command that acts as shorthand for `dbutils.fs`. The latter is accessed through the Python, Scala or R API.

Q2. How would you manage non-tabular files in Unity Catalog?

回答：

I would use Unity Catalog Volumes. They provide governed access to non-tabular files such as CSV, JSON, configuration files and ML artifacts. I can access them through `/Volumes/catalog/schema/volume/` using Spark, Python or `dbutils.fs`.

Q3. Why should you avoid directly modifying the files of a Unity Catalog managed table?

回答：

Unity Catalog and Delta Lake manage table metadata and transaction logs. Directly moving or deleting underlying files can cause inconsistencies between the metadata and the physical data, potentially making the table unreadable.

Q4. What is the difference between `dbutils.fs.put()` and Spark DataFrame write?

回答：

`dbutils.fs.put()` writes a string to a file and is suitable for small configuration files. Spark DataFrame write is designed for structured datasets and distributed data processing, for example writing data in Delta format.

# 3. `dbutils.widgets` — Notebook Parameters

## 3.1 什么是 Widgets？

Widgets 是 Databricks Notebook 提供的交互式输入组件。

它允许用户在 Notebook 顶部设置参数，然后在代码中读取这些参数。

例如，一个 ETL Notebook 需要支持不同的日期：

```
processing_date = 2026-11-08
```

用户不需要修改 Python 代码，只需要改变 Widget 的值。

常见 Widget 类型：

|Widget|用途|
|---|---|
|`text()`|文本输入|
|`dropdown()`|下拉单选|
|`combobox()`|下拉选择或输入|
|`multiselect()`|多选|

## 3.2 text() — Text Input

你记录的代码有一个拼写错误，`default_val` 应该是 `defaultValue`，并且需要传入字符串。

```
dbutils.widgets.text(    "processing_date",    "2026-11-08")processing_date = dbutils.widgets.get(    "processing_date")print(processing_date)
```

执行后，Notebook 顶部会显示一个文本输入框。

Notebook Widget 示例

PROCESSING DATE

2026-11-08

用户可以修改这个值，再运行 Notebook。

注意：Widget 返回的值是 String。

如果需要日期类型：

```
from datetime import datetimeprocessing_date = dbutils.widgets.get(    "processing_date")processing_date = datetime.strptime(    processing_date,    "%Y-%m-%d").date()
```

## 3.3 dropdown() — Dropdown

适合用户从固定选项中选择。

```
dbutils.widgets.dropdown(    "environment",    "dev",    ["dev", "test", "prod"],    "Environment")environment = dbutils.widgets.get(    "environment")
```

这里：

- `environment`：Widget 名称。
    
- `dev`：默认值。
    
- `["dev", "test", "prod"]`：可选值。
    
- `Environment`：显示标签。
    

## 3.4 multiselect() — Multiple Selection

适合需要选择多个值的场景。

```
dbutils.widgets.multiselect(    "cities",    "Amsterdam",    ["Amsterdam", "Rotterdam", "Utrecht"],    "Select cities")cities = dbutils.widgets.get("cities")
```

## 3.5 remove() 和 removeAll()

删除单个 Widget：

```
dbutils.widgets.remove("processing_date")
```

删除所有 Widgets：

```
dbutils.widgets.removeAll()
```

注意：删除 Widget 后，如果要重新创建同名 Widget，通常需要在另一个 cell 中执行创建操作，避免同一 cell 内删除后立即重建导致冲突。

## 3.6 Widgets 在 ETL Pipeline 中的应用

假设你有一个 Bronze ETL Pipeline：

```
dbutils.widgets.text(    "processing_date",    "2026-11-08")dbutils.widgets.dropdown(    "load_type",    "incremental",    ["incremental", "full"])processing_date = dbutils.widgets.get(    "processing_date")load_type = dbutils.widgets.get(    "load_type")
```

根据参数执行不同逻辑：

```
if load_type == "full":    df = spark.read.table("raw.orders")else:    df = (        spark.read.table("raw.orders")        .filter(            f"order_date = '{processing_date}'"        )    )
```

在实际生产环境中，还应该对日期格式和参数值进行校验，避免 SQL 注入或非法参数。

### Widgets vs Job Parameters

这个知识点和你前面学习的 DAB 参数有直接联系。

|对比|Widgets|Job Parameters|
|---|---|---|
|所属|Notebook|Lakeflow Job|
|作用|接收 Notebook 输入|接收整个 Job 的输入|
|访问方式|`dbutils.widgets.get()`|可通过 `{{job.parameters.name}}` 等引用|
|使用场景|Notebook 交互、Notebook 参数|Job 调度、运行时参数传递|

两者不是竞争关系。

实际上，Databricks Job 可以把 Job Parameters 传递给 Notebook Widgets。

例如：

```
resources:
  jobs:
    daily_job:
      parameters:
        - name: processing_date
          default: "2026-11-08"

      tasks:
        - task_key: ingest

          notebook_task:
            notebook_path: ../src/ingest.py
```

Notebook：

```
processing_date = dbutils.widgets.get(    "processing_date")
```

Databricks 会将 Job 参数传给对应 Notebook。

## 3.7 面试问题

Q1. What are Databricks Widgets and why do you use them?

回答：

Widgets are interactive input controls that allow us to parameterize notebooks. They make notebooks reusable because users can change parameters such as processing dates, source tables and load types without modifying the underlying code.

Q2. How would you implement an incremental and full load using Widgets?

回答：

I would create a dropdown widget for the load type and a text widget for the processing date. The notebook would use these parameters to decide whether to load the entire dataset or only the records for a specific date.

Q3. What is the difference between Widgets and Job Parameters?

回答：

Widgets are notebook-level input controls, whereas Job Parameters are defined at the job level. Job parameters can be passed to notebook tasks and retrieved through widgets.

# 4. `dbutils.notebook` — Notebook Workflow

这是你记录中另一个非常重要的模块。

`dbutils.notebook` 主要提供两个方法：

```
dbutils.notebook.run()dbutils.notebook.exit()
```

它们用于实现 Notebook 之间的调用、参数传递和结果返回。

## 4.1 `dbutils.notebook.exit()` — Return a Value

用于结束当前 Notebook，并返回一个结果。

你的原始代码：

```
count = 20dbutils.notebook.exit(count)
```

需要修改：exit() 要求传入 String，而不是 Integer。

正确写法：

```
count = 20dbutils.notebook.exit(str(count))
```

它会：

1. 终止当前 Notebook 的执行。
    
2. 返回字符串 `"20"`。
    
3. 如果是被其他 Notebook 调用，调用方可以接收这个返回值。
    

## 4.2 `dbutils.notebook.run()` — Run Another Notebook

调用另一个 Notebook，并获取它的返回值。

语法：

```
dbutils.notebook.run(    path,    timeout_seconds,    arguments)
```

|参数|含义|
|---|---|
|`path`|被调用 Notebook 的路径|
|`timeout_seconds`|超时时间，单位秒|
|`arguments`|传递给被调用 Notebook 的参数字典|

### 实际案例

假设有两个 Notebook：

```
notebooks/
│
├── main.py
└── child.py
```

Child Notebook：child.py

```
# Define a widgetdbutils.widgets.text(    "processing_date",    "2026-11-08")processing_date = dbutils.widgets.get(    "processing_date")# Process datacount = 100# Return resultdbutils.notebook.exit(str(count))
```

Main Notebook：main.py

```
result = dbutils.notebook.run(    "./child",    60,    {        "processing_date": "2026-11-01"    })print(result)
```

输出：

```
100
```

执行逻辑：

Main Notebook

dbutils.notebook.run()

Pass arguments

processing_date = 2026-11-01

Child Notebook

读取参数、执行 ETL、计算结果

count = 100

dbutils.notebook.exit("100")

Main Notebook

result = "100"

注意：

- `run()` 返回 String。
    
- `exit()` 返回 String。
    
- `arguments` 中的值应该使用字符串。
    
- `timeout_seconds=0` 表示不设置普通执行超时。
    
- Notebook 调用最长不能超过 30 天。
    
- 返回值最大长度为 5 MB。
    

另外，`dbutils.notebook.run()` 在当前 Cluster 上运行被调用 Notebook，并启动一个临时的 Notebook Job 执行流程。它不等同于简单地在当前 Notebook 中执行一段 Python 函数。

Databricks on AWS+1

## 4.3 `%run` vs `dbutils.notebook.run()`

这是一个非常容易在 Data Engineer 面试中出现的对比题。

|对比|`%run`|`dbutils.notebook.run()`|
|---|---|---|
|主要用途|共享 Notebook 代码|调用其他 Notebook|
|是否执行被调用 Notebook|是|是|
|参数传递|可通过 Widgets|通过 arguments|
|返回值|没有独立返回值|有 String 返回值|
|变量共享|可以直接使用变量和函数|不直接共享 Python 命名空间|
|适用场景|代码复用、开发|动态 Notebook 调用、简单流程控制|

### 什么时候使用 `%run`？

例如：

```
%run ./common_utils
```

希望直接使用另一个 Notebook 定义的函数：

```
clean_data(df)
```

### 什么时候使用 `dbutils.notebook.run()`？

例如：

```
result = dbutils.notebook.run(    "./validate_data",    60,    {"table_name": "bronze.orders"})
```

希望：

- 调用其他 Notebook。
    
- 传递参数。
    
- 接收返回值。
    
- 根据返回值执行条件逻辑。
    

## 4.4 一个重要的生产环境补充：Lakeflow Jobs

实际工作中，不建议将复杂 ETL Pipeline 完全建立在 `dbutils.notebook.run()` 上。

例如：

```
Ingestion
   ↓
Bronze
   ↓
Silver
   ↓
Gold
   ↓
Data Quality
```

更推荐使用 Databricks Lakeflow Jobs（原 Workflows）管理：

- Task dependencies
    
- Scheduling
    
- Retries
    
- Failure handling
    
- Monitoring
    
- Job parameters
    

原因：

`dbutils.notebook.run()` 更适合动态调用 Notebook 的特殊场景，例如 Metadata-driven ETL。

Lakeflow Jobs 更适合生产环境的多任务工作流，具备更完善的调度和监控能力。

官方文档：[Orchestrate Databricks Notebooks and Modularize Code](https://docs.databricks.com/aws/en/notebooks/notebook-workflows?utm_source=chatgpt.com)。

Databricks on AWS

## 4.5 面试问题

Q1. What is the difference between `%run` and `dbutils.notebook.run()`?

回答：

`%run` executes another notebook inline and makes its variables and functions available in the current notebook namespace.

`dbutils.notebook.run()` invokes another notebook, allows us to pass parameters and retrieve a string return value.

Q2. How do you pass parameters and return results between notebooks?

回答：

I use `dbutils.notebook.run()` with an arguments dictionary to pass parameters. In the child notebook, I retrieve the arguments using widgets and return a result through `dbutils.notebook.exit()`.

Q3. Would you use `dbutils.notebook.run()` to orchestrate a large production ETL pipeline?

回答：

Not as my first choice. I would use Lakeflow Jobs because it supports task dependencies, scheduling, retries and monitoring. `dbutils.notebook.run()` is more suitable for dynamic notebook invocation or special metadata-driven workflows.

Q4. Can `dbutils.notebook.exit()` return a DataFrame?

回答：

No. It returns a string. For large datasets, I would write the DataFrame to a Delta table or cloud storage and return the table name or file path instead.

# 5. `dbutils.secrets` — Secrets Management

你原来的笔记没有记录这个模块，但它是生产环境非常值得掌握的 DBUtils 功能。

## 5.1 为什么需要 Secrets？

在 Data Engineering 项目中，我们经常需要访问：

- Database passwords
    
- API tokens
    
- Cloud credentials
    
- External service authentication
    

不能直接将密码写入 Notebook：

```
password = "my_password123"
```

因为代码可能被提交到 GitHub，或者被其他开发者看到。

Databricks 提供 Secret Scopes，用于集中管理敏感信息。

## 5.2 创建 Secret Scope

可以通过 Databricks CLI 或其他支持的管理接口创建 Secret Scope。

例如：

```
databricks secrets create-scope my_scope
```

然后添加 Secret：

```
databricks secrets put-secret my_scope db_password
```

实际输入 Secret 值时，应使用安全输入方式，避免在 Shell 历史记录中暴露。

## 5.3 获取 Secret

```
password = dbutils.secrets.get(    scope="my_scope",    key="db_password")
```

然后用于数据库连接：

```
jdbc_url = "jdbc:postgresql://host:5432/database"df = (    spark.read    .format("jdbc")    .option("url", jdbc_url)    .option("user", "db_user")    .option("password", password)    .option("dbtable", "orders")    .load())
```

注意：

- Secret 不应该打印到 Notebook Output。
    
- 应该通过权限控制限制谁可以访问 Secret Scope。
    
- Secret 的访问仍然受到 Workspace 和 Scope 权限控制。
    

## 5.4 面试问题

Q1. How do you manage database credentials in Databricks?

回答：

I use Databricks Secret Scopes to store sensitive credentials and retrieve them through `dbutils.secrets.get()`. This avoids hardcoding passwords in notebooks or source code.

Q2. Why shouldn't you hardcode API tokens in a Databricks notebook?

回答：

Hardcoded credentials can be exposed through source control, notebook outputs or shared code. I would use centralized secrets management with appropriate access controls.

# 6. 整体知识体系总结

## 6.1 Magic Commands vs DBUtils

这是首先要区分的概念。

|对比|Magic Commands|DBUtils|
|---|---|---|
|使用方式|`%` 开头|Python API|
|使用位置|Notebook Cell|Notebook 代码|
|主要用途|切换执行环境、调用 Notebook 功能|操作 Databricks 环境|
|示例|`%fs ls /`|`dbutils.fs.ls("/")`|
|典型应用|Shell、SQL、Python、文件操作|文件管理、参数化、Notebook 调用|

## 6.2 Notebook Workflow 核心区别

|功能|方法|核心作用|
|---|---|---|
|文件操作|`dbutils.fs`|管理文件和目录|
|参数输入|`dbutils.widgets`|接收 Notebook 参数|
|Notebook 代码复用|`%run`|导入另一个 Notebook 的变量和函数|
|Notebook 调用|`dbutils.notebook.run()`|调用其他 Notebook 并传递参数|
|返回执行结果|`dbutils.notebook.exit()`|返回 String|
|密码管理|`dbutils.secrets`|获取敏感信息|

## 6.3 你需要重点记住的几个易错点

|易错点|正确理解|
|---|---|
|`%fs` 可以操作 UC Table 底层文件|不应直接修改 Managed Table 底层文件|
|`%sh` 在所有 Workers 执行|默认只在 Driver 执行|
|`%run` 只导入函数，不执行代码|会执行整个被引用 Notebook|
|`%run` 和 `dbutils.notebook.run()` 一样|前者共享命名空间，后者独立调用并返回字符串|
|`dbutils.widgets.get()` 返回日期类型|返回 String|
|`dbutils.notebook.exit(20)`|应该使用 `dbutils.notebook.exit("20")`|
|`dbutils.fs.put()` 写入目录|必须指定文件路径|
|`dbutils.fs` 只能操作 DBFS Root|也可以操作支持的 UC Volume 路径|

## 6.4 荷兰 Data Engineer 综合面试题

下面这几个问题可以用来检验你是否真正理解了 Databricks Notebook 的工程实践。

Interview self-test

0/7 answers viewed

1. You have a configuration file in a Unity Catalog Volume. How would you read it in Databricks?

2. A notebook needs to process different dates without changing the source code. How would you implement this?

3. How would you reuse common functions across multiple notebooks?

4. How would you dynamically execute a notebook and get its processing result?

5. A notebook processes 10 million records. Would you return the DataFrame using dbutils.notebook.exit()?

6. How would you manage API keys and database passwords?

7. What is the difference between running a command on the driver and running it on executors?