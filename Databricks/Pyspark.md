Pyspark transformation
Data read

```
Reading CSVs
df = spark.read.format("csv")\
		.option("header", True)\
		.option("inferShcema", True)\
		.load("/.../..../....")
		
# print the schema
df.shema

# read JSONs
// single line
nested field:
name_scheme = StructType([
	StructField('givenName', StringType())
	StructField('familyName', StringType())
])
all_scheme = StructType([
	StructField('name', name_scheme),
	...
])

df_json = spark.read.format("JSON")\
		.option("inferSchema", True)\
		.option("multiLine", True)\
		.load("/.../.../...")
//multi-line
.option("multiLine", True)
		
# read Parquet(store data by column, file itself contains the schema)
df_parquet = spark.read.format("parquet")\.
		load("/.../.../...")
		
# read JDBC
myconnection = {
	"user": "",
	"password": ""
}

# option1  
df = spark.read.jdbc(url = my_url, table = "orders", properities = myconnection)

# option2
df = spark.read.format("jdbc")\
		.option("url", my_url)\
		.option("user", "")\
		.load()

```
Correct Record Modes(used in json or csv, can have some data issues)

```
# Permissive mode, when meet error record, still run with the data
df_json = spark.read.format("JSON")\
		.option("inferSchema", True)\
		.option("multiLine", True)\
		.option("mode", "PERMISSIVE")\
		.load("/.../.../...")
		
# Drop Malformed
		.option("mode", "DROPMALFORMED")

# FAILPAST
		.option("mode", "FAILFAST")
```

DataSchema 
StructType
```
from pyspark.sql.type import structType
mySchema = StructType([(StructField('order_id'), StringType(), True) ....])

df_csv_custom = spark.read.format("csv")\
		.....
		.schema(mySchema)\
		.load()
```

Distinct
```
df.distinct()

df.dropDuplicates(subset = None)

df.dropDuplicates(['name', 'age'])

// dedup with watermarking
.withWatermark("order_timestamp", "30 seconds")
.df.dropDuplicates(['name', 'age'])
```
DDL Schema
```
my_ddl_schema = """
order_id INT,
customer_id STRING,
.....
"""

df_csv_custom = spark.read.format("csv")\
		.....
		.schema(my_ddl_schema)\
		.load()

```
Read From location instead of UC(Not for managed table)
```python
location = spark.sql('DESCRIBE DETAIL events').select('location').collect()[0][0]

df = spark.read.format('delta').load(location)

```
SELECT Alias
```
df_select = df.select('id', 'customer_id')
df_select = df.select(col('id'), col('customer_id'))

df_alias = df.select(col(id).alias('customer_id'), 'city')
```
```
# FILTER
df_filter = df.filter((col('city') == 'SGNS') | (col('city') == 'AGAE'))

# Isin
df_filter = df.filter(col('city').isin(['C1', 'C2']))
```

WithColumnRenamed (make change on the origin data)
```
df_remane = df.withColumnRenamed('id', 'customer_id')

# withColumn(transform the column or add a new one)
df_mod1 = df.withColumn('flag', lit(0))

df_mod2 = df.withColumn('shipping_address', regexp_replace('shipping_address', ',.*', ''))

df_mod3 = df.withColumn('total', col('price') *col('quantity'))\
		.withColumn('total', round(col('total), 2))
```

Type Casting

```
df_cast1 = df.withColumn('order_id',col('order').cast(StringType()))

df_cast2 = df.withColumn('order_id',col('order').cast('STRING'))
```

Sorting
```
df_sort1 = df.sort(col('order_id').desc())

df_sort2 = df.sort(['order_id', quantity'], ascending= [0,1])
```
Drop

DropDuplicates
```
df_dedup = df.dropDuplicates()

df_dedup2 = df.dropDuplicates(subset =['order_date', 'id'])
```
Union & UnionByName

```
# Union, union by sequence
df_union = df1.union(df2)

# Union by name, union with the column name.
df_union = df1.unionByName(df2)
```
Date Functions
```
df_curr = df.withColumn("current_time", current_timestamp())

df_add = df_curr.withColumn("current_time", date_add(current_time", 7))

df_sub = df_curr.withColumn("current_time", date_sub(current_time", 7))

df_diff = dff_curr.withColumn("duration", date_diff("current_time","order_date"))

df_format = df_curr.withColumn("current_time", date_format("current_time", "YYMMDD"))


```
String Functions
```
df_upper = df.withColumn("order_status", upper("order_status"))

length("order_status")
```
Handling Nulls
```
# All Nulls, items whose all columns are null will be removed
df_all_null = df.dropna('all')

# Any Null, any column is Null will be dropped
df_any_null = df.dropna('any')

# Drop null if record in either defined column is null
df_subset_null = df.dropna(subset = ['name', 'address'])

# Drop null if record in all defined column is null
df_subset_null = df.dropna(subset = ['name', 'address'], how = "all")

# Fill Nulls
df_fillna = df.fillna('dummy')

df_customer_fill = df.fillna({'name': 'dummy_name', 'address': 'dummy_address'}) 
```

Split/ Index
```
df_split = df.withColumn('street_address', split('ship_address', ',')[0])
```
Explode(explode one row to multiple rows by the explode column array)
```
df_explode = df.withColumn("address_explode", explode_outer("address_array"))
```
Array Contains
```
df_arr_con = df.withColumn("city_1_flag", array_contains("address_array", "city1"))
```
Group by
```
df_agg = df.groupby("product_id", "customer_id")\
			.agg(sum('total_price').alias("total_price"))\
			.withColumn("total_price", round(col("total_price")/2))
```
Approximate Count
```
df = df.groupBy('product_id').agg(arppox_conunt_distinct('customer_id').alias('distinct'))

df = df.groupBy('product_id').agg(conuntDistinct('customer_id').alias('distinct'))
```

Collect List
```
df_collect_list = df.groupBy("customer_id").agg(collect_list("product_id").alias("product"))
```
Pivoting row -> column, the vaule of the row become the column name
```
df_pivot = df.groupBy("customer_id").pivot('order_status').agg(count('order_id'))
```

when-otherwise
```
df_ifelse = df_pivot.withColumn("return_flag", when(col('Return') == 1, 'low').when(col('Return')<=3, mid).when(col('Return')>3, 'high').otherwise('no return'))
```

Join
```
df_inner = df1.join(df2, df1.id = df2.id, 'inner')

df_inner = df1.join(df2, df1['id'] == df2['id'], 'inner')\
		.select(df1['id'])
		
# Anti join (exclude one side)
```
Window Function
```
df_row = df.withColumn("mark_seq", row_number().over(Window.orderBy('marks')))

# Deduplication
df_row = df.withColumn("mark_seq", row_number().over(Window.partitionBy("marks").orderBy('marks_seq')))
```

Row_Number & Rank & Dense_Rank
```
df_row = df.withColumn("mark_seq", dense_rank().over(Window.orderBy('marks')))
```

```
# Culmulate Sum
total_revenue = df.withColumn('total_revenue', sum('revenue').over(Window.orderBy('year')))

# the default is Unbounded preceding
total_revenue = df.withColumn('total_revenue', sum('revenue').over(Window.orderBy('year').rowBetween(Window.unboundedPreceding, Wondow.currentRow)))

# Change to all total to unboundedFollowing
sum('revenue').over(Window.orderBy('year').rowBetween(Window.unboundedPreceding, Wondow.unboundedFollowing)))

```
UDFs
```
def my_square(x):
	square = x * x
	final = rount(square, 2)
	return final
my_square_udf = udf(my_square)
df_cust = df.withColumn("price_square", my_square_udf('price'))

# When using call_udf in SQL, should first register the function
spark.udf.register("my_square_udf", my_square_udf)
df_cust = df.withColumn("price_square", call_udf("my_square_udf", col("price")))

@udf
def udf_function():
	.....

@udf(returnType = IntegerType())
def udf_function():
	.....
```
User Defined Table Functions(generate a dataFrame with vals)
```
# UDTF
@udtf(returnType = StringType())
class udtf_class:
	def eval(self, text):  #function name to be used as eval
		for word in text.split():
			yield (word,)

udtf_class(lit("sdag sag ewt2")) # This will create a dataframe
```

Contact & Contact_Ws

```
df_con = df.withColumn("costomer_id", concat(col("customer_d"), lit('|'), col("order_id"), lit('|')))

df_con = df.withColumn("costomer_id", concat_ws('|', col("customer_id", col("order_id")))
```

Write Files (.mode)
1 OverWrite: Overwrite the existing file
2 Append: Append a new file to the folder
3 Error: Throw an error when a file existed already
4 Ignore: If file exists, will do nothing.

```
df.write.format("csv")\
		.mode("overwrite")\
		.option("path", "../../..")\
		// only save as file
		.save()
		
		// Can be read and search as table 
		.saveAsTable(.path.name)

// Using replaceWhere to only overwrite specific data，Used mostly for partition column
df.write.mode("overwrite") .option( "replaceWhere", "date='2025-01-02'" )
```

Delta format, SQL capibilities
(Parquet + delta_log/ transaction log)
ACID, Optimization/Indexing, Commit &Rollback, Structured Query Env

delta_log contains the commit history

Upsert/ Merge
```python
# UPSERT with DLETA LIBRARY
from delta.tables import DeltaTable

dlt_obj = DeltaTable.forPath(spark, "/../../..")
dlt_obj.alias("trg").merge(df_new.alais("src),"trg.order_id = src.order_id")\
		.whenMatchedUpdateAll()\
		.whenNotMathchedInsertAll()\
		.execture()

# mergeinto API
source.mergeInto("target", "id")
	.whenMatched().update({name: souce.name})
	.whenNotMatched.insertAll()
	.whenNotMatchedSouce().delete()
	.merge()

df_new.alias("src").mergeInto("table_name", col("src.id") = col("table_name.id"))\
		.whenMatched().updateAll()\
		.whenNotMatched().insertAll()\
		.merge()
```
```SQL
MERGE INTO customer AS target
USING updates AS source
ON t.cusomterID = u.customerID

WHEN MATCHED THEN
UPDATE SET
t.name = u.name
t.xxxx = u.xxxx

WHEN NOT MATCHED [BY TARGET] THEN
INSERT(name, xxxx)
VALUES("", "")

WHEN NOT MATCHED BY SOURCE THEN
DELETE
```
If value is a Dict, could read data by 

```
col("dict.key")
```

IgnoreCorruptFiles (Ignore the wrong type files)
```
df = spark.read.option("IgnoreCorruptFiles", "true")\
		.parquet("examples/..././1", "examples/..././2")
```

IgnoreMissingFiles 

pathGlobalFilter (only read files with name matching pattern)
```
df = spark.read.load(,pathGlobalFilter("../../...", "*.parquet"))
```

Recursive FIle Lookup
```
df = spark.raad.format("parquet")\
		.option("recursiveFileLookup", "true")
		...
```

Modification Time Path Filters
use  modifiedBefore = / modifiedAfter


pandas
```
import pyspark.pandas as ps
df = ps.read_csv('../../../')
df.dtypes
df[["", "", ""]].head()

df['new_col'] = df['price'] * 10
df[df['price]> 10]
df.sort_values("prices", ascending=False).head()
```

Pandas UDFs(Vectorized UDF)
With apache arrow, serialization and deserialization will be processed in **batch(**10k rows by default), converted to arrow format. Python calls are fewer. Faster than python UDF.
should convert python udf to pandas udf. 

```
# series to series, use function for each batch
@pandas_udf("double")
def price_func(col1: ps.series, col2: ps.series) -> series:
	return col1 *col2
# Panda series is a column of data in pandas
df = df.withColumn("total_price", price_func('price', 'quantity'))

# Iterator to Iterator, use python worker once, put batch into the worker to process. For instance, to load the ML model, with this, model can be loaded only once.
@pandas_udf("double"):
def iter_func(col1:Iterator[pd.Series]) -> Iterator[pd.Series]:
	for batch in col1:
		yield batch * 10

df = df.withColumn("total_price", iter_func('price'))

# applyInPandas
df.groupby("key").applyInPandas(custom_func, schema)

def calc_avg(pdf): return pd.DataFrame({ 
	"customer": [pdf["customer"].iloc[0]], 
	"avg_amount": [pdf["amount"].mean()] 
}) 

result = ( 
	df.groupBy("customer").
	applyInPandas( 
		calc_avg, 
		schema="customer STRING, avg_amount DOUBLE"
 ) )
```

```
// Time travel (History)
DESCRIBE HISTORY demo.delta_lake.xxx

SELECT * FROM demo.delta_lake.xxx
VERSION AS OF 1;

SELECT * FROM demo.delta_lake.xxx
TIMESTAMP AS OF '2025-01-19T..';

RESTORE TABLE demo.delta_lake.xxx TO VERSION AS OF 1;

MERGE INTO employee t
USING (
	SELECT employee_id, salary
	FROM employees s
	VERSION AS OF 9
) ON t.employee_id = s.employee_id
WHEN MATCH THEN UPDATE SET t.salary = s.salary
```

Idempotency
让 Delta Lake 能识别 retry 造成的重复写入，从而实现幂等性（idempotency）
ex: txnAppId = "daily_etl" txnVersion = 20260827
```python
df.write \ 
.format("delta") \ 
.option("txnAppId", "my_app") \ 
.option("txnVersion", "1") \ 
.mode("append") \ 
.save(path)
```