Freshness
```yml
freshness:
  warn_after:
    count: 365
    period: day
  
  error_after:
    count: 1460
    period: day
```

```shell
dbt run --select bronze_order
# run +1 dependency models
dbt run --select "bronze_order+1"
# run all down dependency models
dbt run --select "bronze_order+"
# run all up dependency models
dbt run --select "+bronze_order"

dbt run --exclude bronze_order
#exclude models with tag 
dbt run --exclude "tag:contains_pii"
```

dbt build

dbt build will run dbt run and dbt test
with --select, it will include all upstream tables

Test
Generic Tests:
pre-build, reusable test
unique, not null, accepted values, relationships
```yaml
# For relationsships ref another table
test:
	-relationships:
		field: product_id
		to: ref('bronze_order')

```
Singular Tests:
under test directory

seed folder
run with cmd dbt seed
CSV files that can be loaded directly in lakehouse as tables

5 kinds of materializations
view table incremental ephemeral(sub query pieces, temporary need) snapshot

- `view`：只保存查询 SQL，不保存查询结果。
    
- `table`：保存查询结果，每次重新构建整张表。
    
- `incremental`：保存查询结果，只处理新增或更新的数据。
    
- `ephemeral`：不创建数据库对象，作为 CTE 嵌入下游 SQL。
    
- `snapshot`：追踪数据历史变化，保存数据过去和现在的版本。