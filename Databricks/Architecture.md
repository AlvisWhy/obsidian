For Storing
metastore (top account level, only one per region, paired with default ADLS Storage)
|
catalog(logic container could be for different envs)
|
Schema(database)
|
(Tables(managed {must use delta format}, external), Volume(managed, external), View, Function)

```
SELECT current_metastore();
```

For Computing

1 Work space can be assigned to the metastore
2 assign specific catalogs to the workspace

Access

![[Pasted image 20260830141718.png]]
To query table，user have to have use catalog and use schema and select/modify table access

workspace admin can create catalog, but could not visit catalog created by others
父级权限不会自动变成子级权限；但某些“对象权限”可以通过父级对象授予，从而覆盖下面的对象。

USE 权限：
父级 ≠ 子级
必须逐级满足访问路径

对象权限：
可以在较高层级授予，
从而作用到下面的对象

| 你给的权限               | 会不会自动给下面对象                    | 例子                      |
| ------------------- | ----------------------------- | ----------------------- |
| `USE CATALOG`       | ❌ 不会                          | 不会自动获得 Schema 的权限       |
| `USE SCHEMA`        | ❌ 不会                          | 不会自动获得 Table 的 `SELECT` |
| `SELECT ON SCHEMA`  | ✅ 会作用于 Schema 下的 Tables/Views | 可读取其中对象                 |
| `MODIFY ON SCHEMA`  | ✅ 会作用于 Schema 下可修改对象          | 可执行相应修改操作               |
| `SELECT ON TABLE`   | 只针对这个 Table                   | 不影响其他 Table             |
| `SELECT ON CATALOG` | ❌ 不能这样直接理解成所有表可读              | Catalog 层主要用于管理/访问能力    |

DB Connection

To update DB CDC or data to UC 

Connection  -> Gateway(keep loading data change) -> Staging(Store data change into a bucket) -> Ingestion pipeline -> Destination table

Query Based Connector: Keep query to get what changed since the last ingestion