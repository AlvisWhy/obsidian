Apache Beam:
- **ParDo** → _Transform each 
`ParDo` transform considers each element in the input `PCollection`
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

CombinePerKey 和 GroupByKey
> **GroupByKey = 我要这一组数据。**  
> **CombinePerKey = 我只要这一组数据算出来的结果。**

Combine Functinon
 Local combine (pre-shuffle): On each worker, the _CombineFn_ reduces all records for a key down to a single "best" record for that worker. A million records for a hot key might become just one.2 Global combine (post-shuffle): The shuffle now moves only these few pre-combined records. The final worker simply merges these candidates to find the globally "best" record.

以及性能上的口诀：

> **能 Combine 就不要先 GroupByKey + 自己聚合。**  
> 因为 `CombinePerKey` 可以利用 **partial combining** 减少 Shuffle 数据量。


optimization

1 Using dataflow insight
Hot Key Detection
This is one of the most valuable insights. It alerts you when your data is not evenly distributed, a condition known as data skew.
It will identify the specific key that is causing a "hot spot" and overwhelming a single worker, which is a common cause of slowness in _GroupByKey_ operations.

Performance Recommendations
For issues other than hot keys, Insights may suggest changes to your pipeline options or code structure to improve throughput or reduce cost.

2 After checking Insights, your next step is to use the [Dataflow job graph(opens in a new tab)](https://cloud.google.com/dataflow/docs/guides/job-graph) to visually investigate the issue or to manually find bottlenecks. It transforms your complex code into an intuitive, visual workflow.
Check the slow steps and inspect step details

3 **Correlate with detailed metrics**
To get more context on how the bottleneck affected system resources, use the [job metrics(opens in a new tab)](https://cloud.google.com/dataflow/docs/guides/using-monitoring-intf) tab.
Check CPU memory IO

4 Review logs

For dataproc checking optimization

1. Use the Dataproc UI to check logs and high-level metrics.
    
2. Click to flip
    
    Use Spark UI to analyze jobs, stages, and executors.
    
3. Click to flip
    
    Use Metrics Explorer to analyze specific performance counters, and Gemini for AI-powered recommendations and actionable insights.

Dataflow for streaming:

 Dataflow runner internally buffers elements into an in-memory collection called a "bundle." Your function is then applied to the entire bundle.

Time window

```python
fixed_window = pcoll | beam.WindowInto(window.FixedWindows(60

sliding_window = pcoll | 'Window' >> beam.WindowInto(window.SlidingWindows(180, 15)) 

session_window = pcoll | beam.WindowInto(window.Session(120))
```

Dataflow ML

Ingest event -> Transform(embedding) -> Enrichment Transform -> ML Transform for recommendation(Input into LLM for prediction)