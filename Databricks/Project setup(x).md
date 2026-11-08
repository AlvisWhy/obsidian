
Databricks | | (Managed Identity) -> Access Connector | | (RBAC permission) -> Azure Data Lake Storage Gen2

AZ
1 Create Access Connector
2 Create Azure data lake sotrage
3 Assign blob data contributor role to the connector
DB
4 Create Storage Credential with the Access connector
5 Create External Location
```
//Create external location
CREATE EXTERNAL LOCATION (IF NOT EXISTS databricks_course_ext_dl1_demo
URL '<bucket url>'
WITH (STORAGE REDENTIAL 'datbricks-sc')
COMMENT 'sagewgw4q'

%fs ls 'abfss://demo@databricks.dfs.core.windows.net/'
```
CREATE CATALOG for landing bronze silver gold

```
CREATE CATALOG [IF NOT EXISTS] <catalog_name>
	[MANAGED LOCATION 'location']
	[COMMENT '']
	

withColumn("time", F.timestamp())
withColumn("file", F.col('_metadata.file_path'))
```



