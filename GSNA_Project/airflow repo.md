
### Story 1: ELT Speed Optimization and elasticsearch load balancing

Maintain and optimize the etl pipeline for more than 100 countries(different countries have different data vol, so share same dataproc). Shorten the time by optimizing the GCP dataproc resource and airflow config after adding new business data.
Guarantee pipeline finish on time and manage resources

Constraing is that the elastic search engine is shared between nightly build and FVS engine, so we have to control no run out of resources among 2 systems

Trade off:
1 Dag total concurrency: use elt_concurrency and max active tasks to control the elt total throughput.
2 Allocate pool and pool slot for different kinds of task(because we shared the dag for ETL and loading data to elastic search)
3 forcemerge snapshot backup, node rotation to decrease the risks

High concurrency can shorten the time but without es constrain could make the downstream ES to break down

Actions:
1 for airflow max active run =1 max_active_task = etl_concurrency(grading in different envs)
2 etl use 3 pools, one for dataproc tasks, one for normal elasticsearch load, one for heavy elasticsearch load. Give different slots.
For speeding up total process, set high priority weight for crucial heavy task

For monitor:
1 moniter the run_elastic run time, queue time , job speed,
2 monitor GCP resources: executor number, memory pending, yarn memory pressure, spark stages. To check dataproc bottlenecks
3 monitor airflow ,and airflow cloudsql memory cpu usage to balance.
4 monitor elastic search cluster:
node jvm cpu gc time
bulk/write rejected
cluster health

txn tasks: high throughput, continue writing, hight pressure for build queue
case task: large document, mapping/segment pressure high

rules: if 5-10 high bulk reject > thredhold, or heap>75% or CPU>70%
slot decrease, every time change parameter, give 30mins to make it stable
To decrease bounding in dataproc clusters , decrease the scale down fraction....

store the log results to bigquery and make analysis according to previous results

result: ETL time decreaes 30%, ES loading peak decrease 25%, failure rate down to 5%.Find the suitable throughput point in system
### Story 2: responsible for upgrading from airflow2 to airflow3

1 debug refreshing airflow:
After airflow upgrading, some dags frequently re-load and the scheduler constantly refreshing the dag version, causing metadata DB writing pressure increase, memory allocation increase rapidly.

locate that the datetime.now() is placed in during the dag analysis, and join in the path spelling.
airflow 3 is more sensitive to the dag structure change, and recognized as dag definition change. triggering constant refresh. 
After comparing the dag logs, locate that the analysis dynamic change is the source.

Repair: Use the jinja template as substitute, like({{ ts.nodash ]}}) to join the path string

competence: from the resource altert to dag difference, and locate to the code, use smallest change for stable running

2 Migrate from ORM SDK based operation to API based utils in platform
airflow 3 not support for orm, design and construct api based platform utils, like overall token acquirment, 401/403 auto refresh tokens. pagination for getting task instances...

Use Rest APIs as substitution for the orms. Debug the loopback error and modify code to set session.trust_env = false/ session.proxies.update(_No-proxy). Because the code should call the machine itself. Use unified LB address to make it avaible for one node and multiple nodes mode, and inject this to env var

api request first use cache token send a request, if get 401/403, get the token again, making the business logic decoupled.

**For large dags**, using idompotent triggering , using customized  runid to control multiple retries. For large dags, using dag trigger operator and monitor task as a group to decouple, design the run_id to father run_id + child dag. Deal with the 409 concurrent conflict.

```python
def rest_trigger_dag_run(target_dag_id: str, conf: Optional[dict] = None, **context) -> None:
	parent_run_id = context.get("run_id") or uuid.uuid4().hex
	run_id = f"{parent_run_id}_{target_dag_id}"
	existing = api_request("GET", f"/api/v2/dag/{target_dag_id}/dagRuns/{run_id}")
	
	if existing.startus_code = 200:
		state = existing.json().get("state")
		if state in (None, "queued", "running"):
			return
	body = {
		"dag_run_id" = run_id
		"conf" = conf
	}
	
	...
```
 Using rest to trigger large dag can avoid blocking and timeout error. Control timeout ourselves instead of SDK. Can use airflow retries for error cases

3 Use monitor_operator to decouple trigger and monitor.
```python
def monitor_operator(dag_id, await_eventual_success: bool = True) ->PythonOperator:
	return PythonOperator(
		task_id = f"monitor_{dag_id}",
		python_callable = func_monitor_dag_status,
		op_kwargs = {"trigger_dag_id": dag_id},
		retry_exponential_backoff = True,
		retry_delay = datetime.timedelta(minute = 10),
		retries = 1000 if  await_eventual_success else 0
	)

def func_monitor_dag_status(trigger_id, poke_interval = 60) -> None:
	while True:
		sleep(poke_interval)
		dag_run = func_get_latest_dagrun(trigger_dag_id)
		if dag_run is None:
			continue
		state = dag_run.state
		if func_dag_state_embed = failed...
		
		...
```
post后返回409 代码并发竞争下一句被其他请求创建 打印日志后继续 不算失败（该dag已经被触发）
Use pythonoperator to encapsulate the rest_trigger_dag_run and set retry =3 retry delay = 30s

use conf as op_kwrgs as jinja run time rendering 
我们遇到的大dag触发超时的问题，吧sdk触发变成异步的rest路径，同时把runid设置成上下文绑定的idempotent key 先get 查重 再post创建 对409冲突处理 来封装dagrun trigger

### STORY 3 性能场景
3 Adjust the retry delay with deterministic jitter
Use stable jitter for es load task. We have many jobs to load data into es. If the es is break down. We use minsec + sha256(task name + dagname).hexdigest % delay range as the retry delay. 
In every rerun, it will get the same delay 

It can avoid 羊群效应  high conflict task will not retry at the same time after every failure. For test and debug. THe same retry sequence can be easy for comparing.

### STORY 4 工程化监控dag用时并比较

Create reuseable run time monitor for dags and task instances for monitor and statics
设计并落地airflow runtime metadata collection strategy. Using rest pagination to collect data. Using staging + merge write into bigquery idompotently. 

设置两条路 dag_run和task_instance 的两个。先建立表后装载。
```python
for dag_id in (dag['dag_id'] for dag in get_all("api/v2/dgas","dags"))
	result.append({
		"workspace_name": workspace_name,
		"instance_id": instance_id,
		"id": dag_run.get("id"),
		"run_type": dag_run.get("type"),
		"queued_at":..
		"execution_at":
		"start_date":
		"end_date":
		"state":
		}
	)

```

入仓先写stage 再做merge upsert 最后删staging. guarantee retry will not cause duplicate data
保持更好原子性，merge in bq 是DML事务语句 比先删后插入更安全 失败会直接回滚 可以随时回放

同时staging设置1天ttl删除后可见 便于debug可审计

```python
def generate_merge_query:
	return f"""
		MERGE INTO 
		USING
		ON
		WHEN MATCHED
		THEN UPDATE SET VALUES
		WHEN NOT MATCHED
		THEN INSERT 
		VALUES
	"""
def load_data_to_bigquery(
			projectid,dataset_id, table_id, schema, join_columns, data
		)-> None:
	staging_table_name = f"{table_id}_staging_{datetime.now().strftime("%Y%M%d%H%M%S")}"
	cilent = bigquery.Client(project = project_id)
	load_table(
		
	)
	merge_query = generate_merge_query(table_ref, stagin_table_ref, join_columns)

	join_config = bigquery.QueryJobConfig()
	client.query()
	client.delete_table(staging_table_ref, not_found_ok = True)
```