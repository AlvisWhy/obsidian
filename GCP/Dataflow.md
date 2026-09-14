Apache Beam:
- **ParDo** → _Transform each element_
- **GroupByKey** → _Group values by key_
- **CoGroupByKey** → _Join multiple PCollections by key_
- **Combine** → _Aggregate values, often with efficient partial aggregation_

```
               Beam
                 │
        ┌────────┴────────┐
        ↓                 ↓
   CoGroupByKey         Combine
        │                 │
     JOIN 类            聚合类
        │                 │
 orders + users       SUM / COUNT
 orders + products    AVG / MAX
```

```pyhon
class MyDoFn(beam.DoFn):

    def process(self, element):
        yield element * 2

result = (
    p
    | beam.Create([1, 2, 3])
    | beam.ParDo(MyDoFn())
)

result = (
    p
    | beam.Create([
        ("A", 10),
        ("B", 20),
        ("A", 30)
    ])
    | beam.GroupByKey()
)

result = (
    p
    | beam.Create([
        ("A", 10),
        ("B", 20),
        ("A", 30)
    ])
    | beam.CombinePerKey()
)


result = (
    p
    | beam.Create([10, 20, 30, 40])
    | beam.CombineGlobally(sum)
)


def join_data(element):
    user_id, grouped = element

    orders = grouped["orders"]
    users = grouped["users"]

    for order in orders:
        for user in users:
            yield {
                "user_id": user_id,
                "order": order,
                "user": user
            }
| beam.Map(join_data)
```