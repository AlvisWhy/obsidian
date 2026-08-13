Knowledge Catalog

There are two types of zones:

- Raw zones contain data in raw formats (such as files in Cloud Storage buckets) and are not subject to strict type-checking.
- Curated zones contain data that is cleaned, formatted, and ready for analytics such as BigQuery tables.



                 Managed Lake
                     │
        ┌────────────┴────────────┐
        │                         │
 Knowledge Catalog            Iceberg
        │                         │
   管理“元数据/目录”          管理“表”
        │                         │
   这是什么？                 数据怎么组织？
   在哪里？                   怎么更新？
   谁负责？                   怎么做 schema evolution？
   怎么发现？                 怎么做 time travel？

To delete a lake, you must first detach assets and then delete the zones.
detach the asset from the zone, then delete the zone, and last, delete the lake.

**Aspect Type = 定义“我要给数据加什么样的标签”的模板**
**Aspect = 根据这个模板创建出来的具体标签**
**Aspect 是可以加在 BigQuery table 的 column 上的**，也可以加在整个table上，所以你可以把它理解成“每个需要标记的字段或表可以有一个 Aspect”。
Using aspect to add metadata on the Curated zones assets lIke：

Column: Email

BigQuery schema
├── Type = STRING
└── Mode = NULLABLE

Aspect
├── Protected = true
├── Sensitivity = HIGH
└── Owner = Security Team


