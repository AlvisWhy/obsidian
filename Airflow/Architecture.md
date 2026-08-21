
Task instance
Operator
Project Structure
1 dags 2 logs(show logs while running) 3 plugins 4 config

Core components of airflow
### Metadata DB (postgres database implicitly for airflow)

The Airflow Metadata Database stores workflow metadata such as DAG definitions, DAG runs, task instances, XComs, connections, variables, pools, and scheduler state. It does not store the business data being processed by the workflows.

### DAG File Processor

It will continuously scan the dags folder and will extract DAG structure

It will store the extracted SERIALIZED(json liked) dag data into the Metadata database

It will scan the folder in every 30 seconds( could change the time)

DAG file processor 先解析，存到metadata db 然后给scheduler运行

min_file_process_interval 
`min_file_process_interval` 限制的是"同一个 DAG 文件的最小重新解析间隔"，用于减少重复解析、降低 Scheduler 和 DAG File Processor 的 CPU 开销

### API Server

Web UI -- Fast API -- metadata database

### Scheduler
### Executor
How to run
### Worker
Run what script

### Queue
1 Execution queue 2 Actual Queue

Q1 Scheduler：
负责：
**决定能不能执行。**

Q2 Executor：
负责：
**真正把任务发送给 Worker。**

Scheduler
    │
    │ submit task
    ▼
CeleryExecutor
    │
    │ send task
    ▼
Celery Broker / Queue
    │
    ▼
Celery Worker
    │
    ▼
执行 task

### Triggerer
负责管理大量"等待事件发生"的异步任务（Deferrable Tasks），避免 Worker 一直被占用。

                Scheduler
                    │
            Queued / Running
                    │
                    ▼
                Worker
                    │
     ┌──────────────┴──────────────┐
     │                             │
普通任务                      Deferrable Task
     │                             │
     ▼                             ▼
 Success                     Deferred
                                    │
                                    ▼
                               Triggerer
                                    │
                            等待事件发生
                                    │
                                    ▼
                              Scheduler
                                    │
                                    ▼
                                 Worker
                                    │
                                    ▼
                                 Success

``` python
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

wait_file = S3KeySensor(
    task_id="wait_file",
    bucket_name="my-bucket",
    bucket_key="data.csv",
    deferrable=True,      # 开启 Triggerer
    poke_interval=60
)
```

```cfg
[triggerer]
default_capacity = 1000
job_heartbeat_sec = 5
```

How dags are synced

HOST machine <== bind Mount ==> opt/airflow/dags in docker

```yaml
volumes:

- ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags

- ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs

- ${AIRFLOW_PROJ_DIR:-.}/config:/opt/airflow/config

- ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins

```

XCOMs (cross communication) and Kwargs

When do_xcom_push = True, will auto 
```
@task.python

def fetch_data(do_xcom_push = True)->dict:
data = {"name": "Airflow", "version": 2.0}
return data
```


When using python operator, return value as the xcom
```
@dag(dag_id="xcom_dag")
def xcom_dag():

	@task.python
	def fetch_data(do_xcom_push = True):
		data = {"name": "Airflow", "version": 2.0}
		return data

  

	@task.python
		def process_data(pulled_data: dict):
			print(f"Processing data: {pulled_data}")

  

# No need to write dependencies between tasks, as the output of fetch_data will be automatically passed to process_data.
	
	pulled_data = fetch_data()
	
	process_data(pulled_data)
```
For bash operator, using pull to get x_com value
```
	ti.xcom_pull(key, task_id)
```

kwargs
kwargs{
	"ti", "logical_date","execution_date"
}

### Scheduling

start_date
end_date
catchup = True (补全从startdate开始没跑的所有)

1 Using presets
@hourly
@weekly
@daily

2 CRON

```
min    hour    day of month   month of year   day of week

0  1,2,13. * * *
0  10  *  *  Mon,Wed,Fri
0  10  1-10  * *  

```


```python
@dag(
	dag_id = "xcom_auto"
	start_date=pendulum.datetime(year=2026,month=1,day=1, tz="UTC")
	schedule="0 0 * * *",
	catch_up=False
	is_paused_upon_creation=False. #Specifies if the dag is paused when created for the first time.
)
```

3 Delta Triggers
Interval based trigger for every several days
```python
from airflow.timetables.trigger import DeltaTriggerTimetable
@dag(
	schedule =DeltaTriggerTimetable(delta=pendulum.duration(days=10)),
	start_date=pendulum.datetime(year=2026,month=1,day=1, tz="UTC")
)
```

For Interval Based

```python
from airflow.sdk import dag, task
from airflow.timetables.interval import CronIntervalTimetable

@dag(
	schedule=CronIntervalTimetable("0 0 * * *"),
	start_date="2026-01-01",
	catchup=False
)
def increment_loading():
	@task.python
	def extract_data(**kwargs):
	from_date = kwargs["data_interval_start"]
```
Will trigger the dag on the end date of the previous dag. 一次 DAG Run 对应一个时间区间，而不是仅仅对应一个“触发时间”。
Incremental Loading

DeltaTriggerTimetable，CronIntervalTimetable
Could get 
```
{{ data_interval_start }} {{ data_interval_end }} {{ logical_date }}
```
For interval based scheduling with kwargs

||`CronIntervalTimetable`|`DeltaTriggerTimetable`|
|---|---|---|
|基础|Cron|`timedelta`|
|时间边界|**固定日历时间**|**相对时间间隔**|
|例子|每天 00:00|每隔 1 小时|
|`data_interval_start`|✅|✅|
|`data_interval_end`|✅|✅|
|关注点|“日历上的哪个时间点？”|“距离触发时间多久？”|

4 Special Events

```python
from airflow.timetables.events import eventTimeTable

evnet_list = EventListTable(
	events=[
		pendulum.datetime(year=2026,month=1,day=1, tz="UTC"),
	]
)

@dag(
	schedule = evnet_list 
)
```


### Parallel and Conditions

Conditional Tasks
```python
@dag
def parallel_dag(ti):

	@task.branch
	def load_data_branch():
			api_data = ti.xcom_pull(task_id = "fetch_api", key = "return_value")
			if(len(api_data)>10):
				return 's3_load'
			else:
				return 'glue_load'
	@task.python:
	def s3_load():
	...
	
	@task.python:
	def glue_load():
	...
	load_data_branch() >> [s3_load(), glue_load()]
```

Task Groups
```python
@task.group
def fetch_data_group():
```

Status Dependency

```python
@task.python(
	trigger_rule = "all_success"/"all_done"/
)
```

|Trigger Rule|含义|
|---|---|
|`all_success`|**所有 upstream 都成功**|
|`all_failed`|所有 upstream 都失败|
|`all_done`|所有 upstream 都执行完，不管成功失败|
|`one_success`|至少一个 upstream 成功|
|`one_failed`|至少一个 upstream 失败|
|`none_failed`|没有 upstream 失败，`success` / `skipped` 都可以|
|`none_failed_min_one_success`|没有失败，并且至少一个成功|
|`none_skipped`|没有 upstream 是 `skipped`|
|`always`|不管 upstream 什么状态，都运行|

Parent DAGs Orchestration

```python
from airflow import DAG
from airflow.operators.trigger_dagrun import TriggerDagRunOperator

with DAG(
    dag_id="parent_dag",
    schedule="@daily",
) as dag:

    trigger_child = TriggerDagRunOperator(
        task_id="trigger_child_dag",
        trigger_dag_id="child_dag",
        wait_for_completion=True,
        conf={
            "source": "parent_dag"
        },
    )
```

Connections in Airflow
Define the connection to the object want to use

Hooks

Asset
Data Aware Dag
Dag depend on the dataset instead of other dags
create materialized data and trigger other dags

DAG A：生产数据
```python
from airflow import DAG
from airflow.decorators import task
from airflow.datasets import Dataset
from datetime import datetime

orders_dataset = Dataset("s3://my-bucket/orders/")

with DAG(
    dag_id="producer_dag",
    start_date=datetime(2026, 1, 1),
    schedule="@daily",
    catchup=False,
) as dag:

    @task(outlets=[orders_dataset])
    def create_orders():
        # 这里生成订单数据
        print("Create orders data")

    create_orders()
```

DAG B：依赖 Dataset
```python
from airflow import DAG
from airflow.decorators import task
from airflow.datasets import Dataset
from datetime import datetime

orders_dataset = Dataset("s3://my-bucket/orders/")

with DAG(
    dag_id="consumer_dag",
    start_date=datetime(2026, 1, 1),
    schedule=[orders_dataset],
    catchup=False,
) as dag:

    @task
    def analyze_orders():
        print("Analyze latest orders")

    analyze_orders()
```
**Airflow 本身不会因为你直接在 BigQuery / GCS 里修改了数据，就自动知道 Dataset 更新了。** 通常需要由 Airflow task 在成功完成相关数据生产后用outlet声明/产生 Dataset event。**`outlets` 最好只放那些 task 确实负责更新的 Dataset**，否则就可能出现“假更新”，导致下游 DAG 被错误触发。

#### Airflow Flower

Flower 是 Celery 的 Web Monitoring Tool，用于监控 Celery Worker、Task 和 Queue 状态。在 Airflow 使用 CeleryExecutor 时，经常配合使用。

Celery 是一个分布式任务队列/执行框架；Airflow 可以通过 CeleryExecutor 使用 Celery Worker 来分布式执行 Task。

负责监控
Celery Queue
Celery Worker
Broker
Task Distribution
worker-1 正在执行什么
worker-2 是否在线
Redis 还有多少待执行任务

Set worker for a defined queue:

```Dockerfile
airflow-worker:
<<: *airflow-common -q compute_heavy_queue
command: celery worker
......
```

```python
@task(queue = "compute_heavy_queue")
```