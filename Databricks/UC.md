metastore (top level, only one per region, paired with default ADLS Storage)
|
catalog(logic container)
|
Schema(database)
|
(Tables(managed {must use delta format}, external), Volume(managed, external), View, Function)

```
SELECT current_metastore();
```