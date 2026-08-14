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

### Dataplex

> **Dataplex 是 Google Cloud 用来统一管理、治理和发现数据的一套数据管理平台。**


Dataplex
   │
   └── Knowledge Catalog
          │
          ├── Data assets
          ├── Metadata
          ├── Search
          ├── Aspects
          └── Governance

Dataplex 的设计目标不是单纯做一个“目录”。

它还负责：
```
Data management
Data organization
Data governance
Data ingestion / management
```
所以某些 Dataplex IAM role 会包含底层资源的相关权限。
例如一个 Writer 可能允许：
```
Dataplex resource
      ↓
写入 / 修改
      ↓
关联的数据资产
      ↓
GCS location
```

Google Cloud 正在把数据治理、catalog、metadata、lake management 等能力整合到 Knowledge Catalog 这个更大的产品体系中。原来dataplex的内容整合到knowledge catalog中

Data Quaility

Knowledge Catalog data quality check requirements are defined using the [CloudDQ](https://github.com/GoogleCloudPlatform/cloud-data-quality) YAML specification files. Once created, the YAML specification file is uploaded to a Cloud Storage bucket that is made accessible to the data quality job.

The [YAML file](https://github.com/GoogleCloudPlatform/cloud-data-quality/blob/main/REFERENCE.md) has four keys sections:

- A list of rules to run (either pre-defined or customized rules).
- Row filters to select a subset of data for validation.
- Rule bindings to apply the defined rules to the table(s).
- Optional rule dimensions to specify the types of the rules that the YAML file can contain.

```yaml
rules:
- nonNullExpectation: {}
  column: id
  dimension: COMPLETENESS
  threshold: 1
- regexExpectation:
    regex: '^[^@]+[@]{1}[^@]+$'
  column: email
  dimension: CONFORMANCE
  ignoreNull: true
  threshold: .85
postScanActions:
  bigqueryExport:
    resultsTable: projects/qwiklabs-gcp-02-273dfa11f1ba/datasets/customers_dq_dataset/tables/dq_results
```

This define a job and could run it to check data quality.
