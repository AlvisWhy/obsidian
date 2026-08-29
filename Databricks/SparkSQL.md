
Temporary View
Only the session view
```
df.createOrReplaceTempView("csv_view")

spark.sql("SELECT * FROM csv_view")
spark.sql("""
		SELECT * FROM csv_view
		WHERE order_id IN (1001,1002,10001)""")
```

SparkSQL DFs
```
df_sql = spark.sql("""
			SELECT * FROM csv_view
			WHERE order_id IN (1001,1002,10001)""")
```

Global Temporary Views (cluster scoped, should be in same cluster to access)
```
df.creatOrReplaceGlobalTempView("global_view")
df.sql("SELECT * FROM global_temp.global_view")
```
DDL
```
spark.sql("CREATE TABLE IF NOT EXISTS sparkcatalog.raw.new_table")
spark.sql("""
		CREATE TABLE sparkcatalog.raw.new_table(
			ID INT,
			NAME STRING,
			AGE INT		
		)
		"""
	)

spark.sql("""
		INSERT INTO sparkcatalog.raw.new_table
		VALUES
		(1, "John", 25),
		(2, "Aweg", 24)
	""")
```

UPSERT
```
sparl.sql("""

		MARGE INTO schema_table t
		USING new_data s
		ON t.ID = s.ID
		WHEN MATCHED THEN
			UPDATE SET *
		WHEN NOT MATCHED THEN
			INSESRT *
	""")
```

PARTITION BY ()

SQL Functions
can use all of the SQL functions as it is in  sparkSql as well
```
WITH inner_query AS (
	SELECT *
		,
		CASE
		WHEN mod(id, 2) =0 THEN 'even'
		ELSE 'odd'
		END AS even_odd_flag
		FROM
			table
		)
	SELECT *
	FROM inner_query
)
```

SQL Scripting Statements

```
# For
BEGIN 
	DECLARE sum INT DEFAULT 0;
	sumNumber: FOR row AS SELECT num FOMR range(1,20) AS t(num) DO
		IF num >10 THEN
			LEAVE sumNumbers;
		ELSEIF num%2 == 0 THEN
			ITERATE sumNumbers;
		END IF;
		SET sum = sum + row.num;
	END FOR sumNumbers;
	VALUE (sum);
END;

# IF

BEGIN
	DECLEAR choice DOUBLE DEFAULT 2.9;
	DECLEAR result STRING
	IF choice < 2 THEN
		VALUES("111")
	ELSEIF choice <3 THEN
		VALUES("222")
	ELSE
		VALUES("NO")
	END IF
END

```

Auxiliary Statements
```
# Describe

DESCRIBE DATABASE sparkcatalog.raw

# Describe query
DESCRIBE QUERY SELECT * FROM table

# Refresh to update file table status
REFRESH TABLE sparkcatlalog.raw.part_table

# Show 
SHOW DATABASES
SHOW SCHEMAS

SHOW TABLE PROPERIETIES
SHOW PARTITIONS table_name
```

SPARKSQL Advanced Functions
```
# Aggregate Functions
# array_agg = collect_list()

SELECT array_agg(customer_id) FROM sql_tbl GROUP BY order_id
df.groupBy("customer_id").agg(array_agg("order_id"))

SELECT array_agg(col) as array_col FROM VALUES((1),(2),(3)) as tab(col)

SELECT corr(price, customer) FROM sql_tbl
```
Struct & Map
```
# create a dict. json, use struct
SELECT struct(1,2,3,"abc")

SELECT named_struct("a",1,"b",2,"c",3)

# Map function
SELECT map("a",1,"b",2,"c",3)
# Only use the vals
SELECT map_values(map("a",1,"b",2,"c",3))
# Only use the keys
SELECT map_keys(map("a",1,"b",2,"c",3))
```
Datetime Function
```
SELECT current_timestamp()

SELECT add_months(current_timestampe(),1)
SELECT add_months(current_timestampe(),-1)

SELECT convert_timezone("America/Los_Angeles", current_timestamp())
SELECT unix_timestamp()
SELECT dayname(current_timestamp()),
		dayofmonth(current_timestamp())
	
SELECT 
	extract(year from current_timestamp())
	extract(doy from current_timestamp()) #Since the beginner of the year

SELECT from_unixtime(0)

```

Window Functions lag(previous value) and lead(next value), ntile(segment data by n groups)
```
SELECT order_id, LAG(order_date, 1, 'NA') OVER (order by order_id) AS prev_order
FROM sql_tbl

SELECT order_id, customer_if, ntile(3) OVER (PARTITION BY order_status) AS order
FROM sql_tbl
```

Array Functions
```
SELECT array(1,2,3)

SELECT array_append(array(1,2,3), 9)

SELECT array_distinct(array(1,2,3,3))

# Remove Null
SELECT array_compact(array(1,2,Null,3,Null))

SELECT array_contains(array(1,2,3), 2)

SELECT array_insert(array(1,2,3), 1, 8)

SELECT array_position(array(1,2,3), 2)

```

SparkSQL Properties

minPartitionNum
maxPartitionNum
parallelPartitionDiscovery.threshold
shuffle.partitions
autoBroadcastJoinThreshold
broadcastTimeout
adaptive.enable

UDF in sparkSQL
```
def my_upper_function(option:str) -> str:
	return ...
spark.udf.register("my_upper",my_upper_function)

CREATE OR REPLACE FUNCTION
```
SPARKSQL Query FIles
```
CREATE OR REPLACE TEMPORARY VIEW orders_csv
USING csv
OPTIONS(
	path: "/.../.../...",
	header: "true",
	inferSchema: "true"
)

SparkingSQL Using DF
spark.sql("SELECT * FROM {df} WHERE..., df = df_csv")

```

Using fileformat connector
```
spark.sql("SELECT * FROM json.`/../../..`")
```