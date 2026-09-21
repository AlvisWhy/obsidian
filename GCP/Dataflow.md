
Architecture

Beam/Dataflow             Spark
Worker          ≈        Executor
DoFn            ≈        Task中的执行逻辑
Pipeline        ≈        Spark Application
Runner          ≈        Spark执行引擎/cluster manager组合

1 Dataflow shuffle service:
**Worker 不需要自己负责保存和管理整个 shuffle 数据。**
Google Cloud 负责：
- 存储 shuffle data
- 分布式传输
- fault tolerance
- shuffle 数据管理
因此 Worker 可以更加独立地进行 scale 而不需要把大量 shuffle 状态绑定在 Worker 本地。
Dataflow Shuffle Service 把 shuffle 的中间数据从 Worker 本地磁盘中“外置”出来，由 Google 管理的 Shuffle Service 负责存储和传输。

2 Dataflow Streaming Engine

                ┌── Dataflow Worker
Pub/Sub ───────→│   主要负责计算
                │
                ↓
        Streaming Engine
        ├── Streaming state
        ├── Window-related processing
        ├── Shuffle
        └── 持久化/管理部分 streaming workload

Flex RS
“不急着跑的 Batch Job，用更便宜的计算资源来跑。”
FlexRS 会结合：

- 普通 Compute Engine VM
- **Preemptible VM / Spot VM**
- Dataflow 的资源调度

来降低 Batch Job 的计算成本。
代价是**Job 不一定立即获得全部资源，启动时间可能更长，而且 Spot/Preemptible worker 可能被回收。**

IAM:
User account: Dataproc Viewer/Developer/Admin
SA for dataflow service
SA for VM running dataflow jobs:  compute engine default service account At a minimum, your service account must have the Dataflow worker role and can be used. And BQ gcs role to interactive with other resources.

Quota:
public IP/ CPU/ DIsk(default 250G not using data shuffle service and 25G using it)
Using max scaling vm num while calculating the quota


```shell
gcloud dataflow jobs run job2 \
--gcs-location gs://dataflow-templates-us-east4/latest/Word_Count \
--region $REGION \
--worker-machine-type e2-standard-2 \
--staging-location gs://$PROJECT/tmp \
--parameters inputFile=gs://dataflow-samples/shakespeare/kinglear.txt,output=gs://$PROJECT/results/outputs --disable-public-ips

```
Apache Beam:

PCollection, immutable data 多个机器可以同时访问/修改同一份数据。
PTransform

- **ParDo** → _Transform each 
`ParDo` transform considers each element in the input `PCollection`

element和bundle关系
Pipeline
   │
   ↓
PCollection
   │
   ↓
Elements
   │
   │ Runner 把多个 Element 分成
   ↓
Bundles
   │
   ↓
DoFn
   │
   └── process(element)

**State API**
应该让runner 管理state， state都是per-key
```python
class MyDoFn(beam.DoFn):
	# When a worker starts
	def setup(self):
		pass
	# When a new data bundle received
	def start_bundle(self):
		pass
	# the transform take place / read state objects
	def process(self, element):
		pass
	# sink to external system / write state objects
	def finish_bundle(self):
		pass
	# used for close connections
	def teardown(self):
		pass

```

This is a bean class. 不能假设 `self.val` 是可靠、持久、全局的状态。
可以用state记录每个key的指定状态
State 非常适合：

- streaming
- incremental processing
- per-key counters
- session tracking
- 去重
- timers
- 保存窗口内的中间状态

```python
class CountFn(beam.DoFn):

    COUNT = beam.DoFn.StateParam(
        beam.transforms.userstate.ReadModifyWriteStateSpec(
            'count',
            beam.coders.VarIntCoder()
        )
    )
    
    

    def process(self, element, count=beam.DoFn.StateParam(COUNT)):
        current = count.read() or 0
        count.write(current + 1)
```
> **对于某个 key，跨多次 `process()` 调用保存一个 count。**

States type:
1 value 2 bag 3 combining 4 map 5 set


Buffer state and count state for event trigger
                    user1
                      │
              ┌───────┴────────┐
              ↓                                                                ↓
         CountState)(CombiningValueStateSpec)        BufferState(BagStateSpec)
         count = 3         [A,B,C]
              │                │
              └───────┬────────┘
                      ↓
                 count == 100?
                      ↓
                    Yes
                      ↓
                批量处理 100 个
                      ↓
                  clear state

Should remember to clear the state or it will influence the following process\.

> **每个 user 在一个 window 内缓存 event，当 window 结束并经过 allowed lateness 后，Timer 触发，把这个 user 的所有 event 批量输出。**

```python
ALLOWED_LATENESS = 60  # 60 seconds


class BufferEvents(beam.DoFn):

    # State 1: 保存具体的 event
    BUFFER_STATE = BagStateSpec(
        'buffer',
        StrUtf8Coder()
    )

    # State 2: 记录 event 数量
    COUNT_STATE = CombiningValueStateSpec(
        'count',
        VarIntCoder(),
        sum
    )

    # Timer
    EXPIRY_TIMER = TimerSpec(
        'expiry_timer',
        TimeDomain.WATERMARK
    )

    def process(
        self,
        element,
        window=beam.DoFn.WindowParam,
        buffer_state=beam.DoFn.StateParam(BUFFER_STATE),
        count_state=beam.DoFn.StateParam(COUNT_STATE),
        expiry_timer=beam.DoFn.TimerParam(EXPIRY_TIMER)
    ):
        user_id, event = element

        # 1. 保存 event
        buffer_state.add(event)

        # 2. count + 1
        count_state.add(1)

        # 3. 设置 event-time timer
        expiry_timer.set(
            window.end + ALLOWED_LATENESS
        )

    @on_timer(EXPIRY_TIMER)
    def expire(
        self,
        window=beam.DoFn.WindowParam,
        buffer_state=beam.DoFn.StateParam(BUFFER_STATE),
        count_state=beam.DoFn.StateParam(COUNT_STATE)
    ):

        # 4. Timer 到时间后读取 State
        events = list(buffer_state.read())
        count = count_state.read()

        # 5. 输出
        yield {
            'window_start': window.start.to_utc_datetime(),
            'window_end': window.end.to_utc_datetime(),
            'count': count,
            'events': events
        }

        # 6. 清理 State
        buffer_state.clear()
        count_state.clear()
```

Pipeline：
Timer API


- **GroupByKey** → _Group values by key_
- **CoGroupByKey** → _Join multiple PCollections by key_
- **Combine** → _Aggregate values, often with efficient partial aggregation_
- **Flatten** -> If two P collections contain exactly the same type, they can be fused together in just one P collection using the Flatten transform.
- **Partition** -> Separate Pcollection to serveral PCollections 

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


```python

class CommonLog (typing.NamedTuple):
    ip: str
    user_id: str
    lat: float
    lng: float
    timestamp: str
    http_request: str
    http_response: int
    num_bytes: int
    user_agent: str

class PerUserAggregation(typing.NamedTuple):
    user_id: str
    page_views: int
    total_bytes: int
    max_bytes: int
    min_bytes: int

# register for serialize or deserialize
beam.coders.registry.register_coder(CommonLog, beam.coders.RowCoder)
beam.coders.registry.register_coder(PerUserAggregation, beam.coders.RowCoder)

with beam.Pipeline(options=pipeline_options) as p:
        (p | 'ReadFromGCS' >> beam.io.ReadFromText(known_args.input_path) 
         | 'ParseJson' >> beam.Map(parse_json).with_output_types(CommonLog)
         | 'PerUserAggregations' >> beam.GroupBy('user_id')
             .aggregate_field('user_id', CountCombineFn(), 'page_views')
             .aggregate_field('num_bytes', sum, 'total_bytes')
             .aggregate_field('num_bytes', max, 'max_bytes')
             .aggregate_field('num_bytes', min, 'min_bytes')
             .with_output_types(PerUserAggregation)
         | 'ToDict' >> beam.Map(to_dict)
         | 'WriteToBQ' >> beam.io.WriteToBigQuery(
             known_args.table_name, 
             schema=table_schema,
             create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED,
             write_disposition=beam.io.BigQueryDisposition.WRITE_TRUNCATE
           )
        )

```
Dataflow for streaming:

 Dataflow runner internally buffers elements into an in-memory collection called a "bundle." Your function is then applied to the entire bundle.

Time window

```python
fixed_window = pcoll | beam.WindowInto(window.FixedWindows(60

sliding_window = pcoll | 'Window' >> beam.WindowInto(window.SlidingWindows(180, 15)) 

session_window = pcoll | beam.WindowInto(window.Session(120))
```

- **Data freshness（数据新鲜度）**：用户看到的数据有多“新，
- **System latency（系统延迟）**：一条数据从产生到被系统处理/输出，花了多久

                    ┌─ ingestion delay（数据本身延迟+ worker不够导致的ingestion延迟）
                    │
Freshness ≈ event ──┼─ processing latency （数据cpu处理不足）
                    │
                    └─ delivery/query delay
                         ↓
                      当前时刻

**Trigger** 决定什么时候输出结果
如果不指定 Trigger：
等 Watermark 到达 window end

3 types of triggers:
1 event-time trigger 根据 **Watermark / event time**：
2 Processing-time trigger 每多久输出一次结果
3 Data-driven / Count trigger 根据收到多少数据

| Trigger                              | 看什么时间                   |
| ------------------------------------ | ----------------------- |
| `AfterWatermark`                     | **Event Time 的进度**      |
| `AfterProcessingTime`                | **Worker 实际处理时间**       |
| `AfterPane.elementCountAtLeast(100)` | **收到多少条数据**             |
| AfterWatermark(late=AfterCount(1))   | watermark后每次收到一个累计消息就触发 |


一个 Window 可以 Trigger 多次
Tigger 计算内容：
1 Accumulating 每次计算trigger全部内容
2 Discarding 只输出上次trigger之后的数据
```python
| "Window" >> beam.WindowInto(
    beam.window.FixedWindows(60),
    trigger=beam.trigger.AfterWatermark(),
    accumulation_mode=beam.trigger.AccumulationMode.DISCARDING
)
```

```python
def add_timestamp(element: CommonLog):
    event_time = datetime.strptime(element.timestamp, '%Y-%m-%dT%H:%M:%S.%fZ')
    ts = event_time.timestamp()
    return beam.window.TimestampedValue(element, ts)

class GetTimestampFn(beam.DoFn):
    def process(self, element: int, window=beam.DoFn.WindowParam):
        window_start_str = window.start.to_rfc3339()
        output = {'page_views': element, 'timestamp': window_start_str}
        yield output
        
(p | 'ReadFromGCS' >> beam.io.ReadFromText(input_path)
       | 'ParseJson' >> beam.Map(parse_json).with_output_types(CommonLog)

       | 'AddEventTimestamp' >> beam.Map(add_timestamp)

       | "WindowByMinute" >> beam.WindowInto(beam.window.FiexedWindows(60))

       | "CountPerMinute" >> beam.CombineGlobally(CountComnineFn()).without_defaults()

       | "AddWindowTimestamp" >> beam.ParDo(GetTimestampFn())

       | 'WriteToBQ' >> beam.io.WriteToBigQuery(
           table_name,
           schema=table_schema,
           create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED,
           write_disposition=beam.io.BigQueryDisposition.WRITE_TRUNCATE
           )
    )

```


Databricks：

```
.withWatermark("event_time", "10 minutes")
```

这个 **10 minutes 是 watermark delay threshold**：

IO
Text FIle IO
```python

pcoll = (pipeline
	| 'Create' >> Create([file_name])
	| 'ReadAll' >> ReadAllFromText())
)
pcoll2 = pipeline | 'Read' >> ReadFromText(file_name)

transformed_data
	| 'write' >> WriteToText(
		known_args.output, coder=JsonCoder()
	)

```

```Java
p.apply(
	FileIO.match()
		.filepattern("...")
		.continuously(
			Duration.standardSecond(30),
			Watch.Growth.afterTimeSinceNewOutput(
				Duration.standHours(1)
			)
		)
)

```

BigQueryIO
```Java
PCollection<Double> maxTemperatures = 

p.apply(
	BigQueryIO.read(
		(SchemaAndRecord elem) -> (Double)
			elem.getRecord()
			.get("max_temperature")
	).fromQuery(
		"SELECT max_temp FROM
		....
		"
	).usingStandardSql()
	.withCoder(DoubleCOder.of())
);

```

PubsubIO
```Java
pipeline.apply(
	"read pubsub message", PubsubIO.readString().fromTopic(options.getInputTopic)
	.apply(
		Window.into(
			FixedWindows.of(
			Duration.standardMinutes(
				options.getWindowSize()
			))
		)
	)
)

```
```python
pipeline
 | "Read Pubsub Message" >> beam.io.ReadFromPubSub(topic = input_topic)
 | "Window into" >> GroupWindowIntoBatches(window_size)


```

KafkaIO
BigTableIO
AvroIO

Dataflow ML

Ingest event -> Transform(embedding) -> Enrichment Transform -> ML Transform for recommendation(Input into LLM for prediction)

Use gemini connection and gcs external url for the table
```SQL
CREATE OR REPLACE EXTERNAL TABLE
`gemini_demo.review_images`
WITH CONNECTION `us.gemini_conn`
OPTIONS (
object_metadata = 'SIMPLE',
uris = ['gs://qwiklabs-gcp-04-99513016e648-bucket/gsp1246/images/*']
);
```

```SQl
CREATE OR REPLACE MODEL `gemini_demo.gemini_flash`
REMOTE WITH CONNECTION `us.gemini_conn`
OPTIONS (endpoint = 'gemini-3.5-flash')
```
```SQL
CREATE OR REPLACE TABLE
`gemini_demo.customer_reviews_keywords` AS (
SELECT ml_generate_text_llm_result, social_media_source, review_text, customer_id, location_id, review_datetime
FROM
ML.GENERATE_TEXT(
MODEL `gemini_demo.gemini_flash`,
(
   SELECT social_media_source, customer_id, location_id, review_text, review_datetime, CONCAT(
      'For each review, provide keywords from the review. Answer in JSON format with one key: keywords. Keywords should be a list.',
      review_text) AS prompt
   FROM `gemini_demo.customer_reviews`
),
STRUCT(
   0.2 AS temperature, TRUE AS flatten_json_output)));
```

Error handling
failure types
1 failure building the pipeline
2 failure when starting the pipeline , like service account error or api enabled
3 failure during pipeline execution
4 failure 

In batch pipelines, job fail will retry 4 times
In streaming pipelines, job retry indefinitely

**Pipeline optimization:**

Encoding and decoding are a large source of overhead. 不要为了读取一个字段，把整个大对象都 decode
假设你有一个大的 sliding window, 先按小的window去combine 再聚合这些小的window
先把大量明细数据压缩成少量聚合结果，再进行后面的 Window/Combine

Fusion：Dataflow 会自动合并 Transform，可让一个worker去处理
不过Fusion + Fanout 可能产生问题 一个 worker 可能连续处理巨量 fanout 数据。
可以 强制把中间结果重新分发，让后面的工作重新获得并行化机会。
```
ParDo
 ↓
Fanout
 ↓
Reshuffle
 ↓
Next transform
```
Aggregation 会天然阻止 ParDo Fusion Dataflow 不会把两个 ParDo 跨过 aggregation 强行 Fusion。

不要使用大量log.error 或log.info，很影响性能
使用deadletter queue 然后统计数量

Data skew solution in dataflow
1 withFanout(int)
2 withHotKeyFanout(sfn)
3 Use dataflow shuffle service

To detect hot keys, set hotKeyLogginEnabled flag to true, print which key is hot
Too few key will cause not parallel inplementation, and thus idel worker, could use hashes separating keys out internally
如果不同 Window 之间彼此独立，那么可以把 Window 信息也加入 Key，让不同 Window 的数据分散到更多 Worker 上并行处理。

Switch to uncompressed files while using TextIO, or switch to compressed Avro format.

Dataflow 本身可以非常快，但它太快了，反而可能把下游外部系统“打爆”。
如果你的代码里面有：
```
call_external_api(element)
```
对于batch或straming 的backlog

解决方案：Batch external calls
```python
events | beam.GroupIntoBatches(100)
```
或者
```
@StartBundle
    ↓
创建 buffer

process()
    ↓
不断把数据放进 buffer

@FinishBundle
    ↓
批量发送
```
Updating

旧版 Streaming Pipeline
        │
        │ ① Snapshot
        ▼
保存 State + Buffer
        │
        │ ② Update
        ▼
新版 Streaming Pipeline
        │
        ├── 正常运行 → 继续处理
        │
        └── 发现有问题 → 从 Snapshot 回滚

When updating the pipeline, create dataflow snapshot
When taking snapshot, the dataflow wiil block
Snapshot 最重要的用途：Rollback

snapshot with source:
不仅保存 Dataflow 内部 State，还尽量把 Pub/Sub 中“还没处理的消息位置”与 State 对齐。

新旧pipeline要做transform name mapping
这样知道保存的state是哪个transform用的

哪些变化容易导致 Update 失败？
1 改了 Transform name，但是没有 mapping
2 增加/删除 Side Input
3 修改 Coder
4 删除 Stateful Operation

Old Streaming Job
        │
        │ Snapshot
        ▼
   Snapshot S1
        │
        │ create from snapshot
        ▼
New Job
        │
        ▼
Streaming Engine

Snapshot ↓ 解决的是： “我怎么保存/恢复当前 Streaming Pipeline 的状态？” Blue-Green ↓ 解决的是： “我怎么让新旧两个版本独立运行，验证后再切换？” Update ↓ 解决的是： “我怎么把正在运行的 Pipeline 换成新版本，同时保留兼容的状态？”

另一种方法
不使用cancel 使用 drain
如果修改导致 Streaming Update 不兼容，就 Drain 旧 Pipeline，然后用更新后的代码启动新的 Job。
`Drain` 可以理解成：
> **不要直接杀掉正在运行的 Streaming Pipeline，而是让它停止接收新的数据，同时把已经进入 Pipeline 的数据尽量处理完，然后让 Job 结束。**

no data is lost, but all windows are closed, resulting in incomplete aggregation, use beam paninfo object to identify and filter incomplete windows

Snapshot for pubsub
```
1 
gcloud pubsub snapshots create my-snapshot --subscription=my-sub
2 
gcloud dataflow jobs drain
3 把 Pub/Sub Subscription 的读取位置往回/往前移动
gcloud subsub subscriptions seek my-sub --snapshot=my-snapshot
4
gcloud dataflow jobs rub my-job-name --gcs_location=...
```
The data maybe duplicated

retention for Dataflow Snapshots is 7 days

Flex templates for dataflow
developer packages the pipeline artifacts into Docker image and stages the image on Google Container Registry In addition, the developer creates a metadata specification file on Cloud Storage.
the template launches service, reads the metadata spec file, downloads the Docker image, and invokes the pipeline using user-supplied values.
