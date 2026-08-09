Jinja runtime value injection
```python

BashOperator(
    task_id="show_xcom",
    bash_command="""
    echo "{{ ti.xcom_pull(task_ids='fetch_data') }}"
    """
)

```

Retry policy
```python
@task.bash(retries=3, retry_delay=timedelta(seconds=10))

```

logical_date: actual time the dag should run,

```python
# Using Jinja filter to get date
@task.bash
def print_logical_date_bash():
	return "echo 'The logical data is: {{logical_date | ds}}'"
```
JINJA data
日期
────────────────────
{{ logical_date }}
{{ logical_date | ds }}

{{ ds }}
{{ ts }}

{{ data_interval_start }}
{{ data_interval_end }}


DAG / Task
────────────────────
{{ dag.dag_id }}
{{ task.task_id }}
{{ run_id }}


Task Instance
────────────────────
{{ ti.task_id }}
{{ ti.dag_id }}

{{ ti.xcom_pull(task_ids='xxx') }}


参数
────────────────────
```python
@dag( params={ "country": "Taiwan" } )
{{ params.xxx }}
```


Airflow Variables
────────────────────
{{ var.value.xxx }}
{{ var.json.xxx }}


Connections
────────────────────
{{ conn.xxx.host }}
{{ conn.xxx.login }}