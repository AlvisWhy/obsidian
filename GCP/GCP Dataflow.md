
# 1. Dataflow Architecture & Execution

## 1.1 Apache Beam、Dataflow 与 Spark 的关系

Apache Beam 是统一的分布式数据处理编程模型，Dataflow 是 Google Cloud 提供的托管执行服务。

Beam 负责定义数据处理逻辑，Dataflow Runner 负责将 Pipeline 提交到 Dataflow 执行环境中运行。

| Beam / Dataflow | Spark（概念类比）       | 作用               |
| --------------- | ----------------- | ---------------- |
| Pipeline        | Spark Application | 定义完整数据处理任务       |
| PCollection     | RDD / Dataset     | 分布式数据集合          |
| PTransform      | Transformation    | 数据转换操作           |
| DoFn            | Task 中的执行逻辑       | 定义元素处理逻辑         |
| Worker          | Executor 所在的执行节点  | 执行实际计算           |
| Runner          | Spark 执行引擎与资源管理体系 | 决定 Pipeline 如何执行 |

注意：以上是概念类比，并非严格的一一对应关系。

## 1.2 Dataflow Shuffle Service

### Shuffle 是什么？

在分布式计算中，某些操作需要按照 Key 重新分配数据，例如：

- GroupByKey
    
- Join
    
- CoGroupByKey
    
- 部分聚合操作
    

例如：

```text
Worker 1: A, B, A
Worker 2: B, A, C
Worker 3: A, C, B

          ↓ Shuffle

Worker 1: A, A, A, A
Worker 2: B, B, B, B
Worker 3: C, C
```

相同 Key 的数据需要被重新分配到相应的处理位置。

### Dataflow Shuffle Service 的作用

传统情况下，Worker 需要承担部分 Shuffle 中间数据的存储和管理工作。

Dataflow Shuffle Service 将 Shuffle 的中间数据存储和传输工作卸载到 Google 管理的 Shuffle Service。

主要优势：

1. 减少 Worker 对本地磁盘存储 Shuffle 数据的依赖。
    
2. 由托管服务负责 Shuffle 数据的存储、传输和管理。
    
3. 降低 Worker 与 Shuffle 数据生命周期之间的耦合。
    
4. 使 Worker 扩缩容更加灵活。
    
5. 适用于大量数据需要重新分布的 Batch Pipeline。
    

注意：Shuffle Service 不代表没有网络传输开销，也不意味着所有 Pipeline 都会自动变快。

## 1.3 Dataflow Streaming Engine

Streaming Engine 主要用于 Streaming Pipeline。

传统 Streaming Pipeline 中，Worker 需要承担更多 Streaming 数据处理相关工作。

启用 Streaming Engine 后，部分 Streaming 执行、状态管理和数据处理工作会被卸载到 Google 管理的后端服务。

```text
Pub/Sub
   |
   v
Dataflow Worker
   |
   | 主要执行用户定义的计算
   |
   v
Streaming Engine
   |
   ├── Streaming state management
   ├── Window-related processing
   ├── Shuffle
   └── Managed streaming execution
```

核心思想：

**将部分 Streaming 数据处理工作从 Worker 卸载到 Google 管理的基础设施，减少 Worker 的负担。**

### Shuffle Service vs Streaming Engine

|对比|Dataflow Shuffle|Streaming Engine|
|---|---|---|
|主要场景|Batch Pipeline|Streaming Pipeline|
|核心目的|管理 Shuffle 中间数据|卸载部分 Streaming 执行工作|
|主要收益|降低 Worker 对本地 Shuffle 的依赖|减少 Worker 承担的 Streaming 工作|
|典型场景|GroupByKey、Join|持续运行的实时数据处理|

## 1.4 FlexRS

FlexRS = Flexible Resource Scheduling。

核心思想：

对于不要求立即启动的 Batch Job，可以通过灵活的资源调度降低计算成本。

FlexRS 会结合：

- 普通 Compute Engine VM
    
- Preemptible VM / Spot VM
    
- Dataflow 资源调度机制
    

优势：

- 降低 Batch Pipeline 的计算成本。
    
- 适用于对启动时间不敏感的任务。
    

代价：

- Job 可能无法立即获得全部资源。
    
- 启动时间可能更长。
    
- Spot / Preemptible Worker 可能被回收。
    

适用场景：

- 非实时 Batch Job。
    
- 对执行时间有一定弹性的 ETL。
    
- 更关注成本而不是立即启动的任务。
    

不适合：

- 对启动时间和延迟要求严格的任务。
    
- 需要稳定、即时资源的实时 Pipeline。
    

## 1.5 IAM 与 Service Account

Dataflow 涉及不同身份，不能把用户账号、Dataflow Service Agent 和 Worker Service Account 混为一谈。

### 1. User Account

负责创建、提交、管理 Dataflow Job。

通常需要 Dataflow Developer 等相关权限，以及访问 Pipeline 所需资源的权限。

### 2. Dataflow Service Agent

Google 管理的服务身份，用于 Dataflow 服务本身执行管理操作。

通常由 Google Cloud 在启用 Dataflow API 时创建。

### 3. Worker Service Account

Dataflow Worker VM 使用的身份。

至少需要：

- Dataflow Worker Role。
    
- 访问输入数据源的权限。
    
- 写入输出数据源的权限。
    
- 访问所需 GCS Staging / Temp Bucket 的权限。
    

例如：

|资源|可能需要的权限|
|---|---|
|Dataflow|Dataflow Worker|
|GCS|Storage Object Viewer / Creator 等|
|BigQuery|BigQuery Job User、Data Editor 等|
|Pub/Sub|Pub/Sub Subscriber / Publisher|
|Bigtable|对应 Bigtable 读写权限|

原则：

**提交 Job 的用户需要提交权限，Worker Service Account 需要运行 Pipeline 所需的资源访问权限。**

## 1.6 Quota

Dataflow 运行时可能受到以下配额限制：

- CPU quota
    
- Public IP quota
    
- Disk quota
    
- Worker 数量相关限制
    
- 区域资源容量限制
    

计算资源需求时，需要考虑：

```text
Maximum Worker Count
        |
        v
Maximum CPU Consumption
        |
        v
Maximum Disk Consumption
        |
        v
Quota Requirements
```

Worker 数量不能只按照初始 Worker 数量估算，应考虑 `max_num_workers` 等最大扩缩容配置。

笔记中的 Disk 数值：

- 不使用 Dataflow Shuffle Service：默认磁盘记录为 250 GB。
    
- 使用 Shuffle Service：默认磁盘记录为 25 GB。
    

这类数值受具体配置、服务版本和配额政策影响，不应作为所有 Dataflow Job 通用的固定值。

## 1.7 常见命令

```bash
gcloud dataflow jobs run job2 \
  --gcs-location gs://dataflow-templates-us-east4/latest/Word_Count \
  --region $REGION \
  --worker-machine-type e2-standard-2 \
  --staging-location gs://$PROJECT/tmp \
  --parameters inputFile=gs://dataflow-samples/shakespeare/kinglear.txt,output=gs://$PROJECT/results/outputs \
  --disable-public-ips
```

参数解释：

|参数|作用|
|---|---|
|`--gcs-location`|指定 Dataflow Template|
|`--region`|指定 Job 运行区域|
|`--worker-machine-type`|Worker VM 类型|
|`--staging-location`|Staging 文件存储位置|
|`--parameters`|传递 Pipeline 参数|
|`--disable-public-ips`|Worker 不使用 Public IP|

## 面试问题

### Q1. What is the difference between Apache Beam and Google Cloud Dataflow?

中文解析：考察编程模型和执行服务的区别。

英文回答：

Apache Beam is a unified programming model for defining batch and streaming data processing pipelines.

Google Cloud Dataflow is a fully managed service that executes Apache Beam pipelines.

Beam defines the processing logic, while Dataflow provides the execution environment, resource management, autoscaling, and monitoring.

### Q2. What is the purpose of Dataflow Shuffle Service?

中文解析：考察 Shuffle 的作用，以及为什么将 Shuffle 与 Worker 解耦有价值。

英文回答：

Shuffle is required when data needs to be redistributed across workers, for example, during GroupByKey or join operations.

Dataflow Shuffle Service moves shuffle data management away from individual workers and into a managed service.

This reduces the dependency on worker-local storage and makes worker scaling more flexible.

### Q3. What is the difference between Dataflow Shuffle and Streaming Engine?

中文解析：一个主要解决 Batch Shuffle 数据管理问题，另一个针对 Streaming Pipeline 的执行和状态管理。

英文回答：

Dataflow Shuffle is primarily designed to manage intermediate data exchange during shuffle-heavy batch processing.

Streaming Engine is designed for streaming pipelines and offloads parts of streaming execution and state management to Google's managed infrastructure.

They address different execution patterns.

### Q4. What is FlexRS and when would you use it?

中文解析：考察成本与延迟之间的 Trade-off。

英文回答：

FlexRS is a flexible resource scheduling option for batch workloads.

It can reduce compute costs by using lower-cost resources and flexible scheduling.

However, job startup may be delayed, and preemptible resources may be reclaimed.

I would use it for batch workloads that are not time-sensitive.

---

# 2. Apache Beam Programming Model

# 2.1 Pipeline、PCollection、PTransform

## Pipeline

Pipeline 是完整的数据处理流程。

```python
with beam.Pipeline() as p:
    result = (
        p
        | beam.Create([1, 2, 3])
        | beam.Map(lambda x: x * 2)
    )
```

## PCollection

PCollection 是 Beam 的分布式数据集合。

主要特点：

1. Immutable：不可变。
    
2. 可以包含有限数据，也可以表示无限数据流。
    
3. 数据可能分布在多个 Worker。
    
4. 多个 Transform 可以读取同一个 PCollection。
    
5. 每个 Transform 会产生新的 PCollection，而不是直接修改原始数据。
    

例如：

```python
numbers = p | beam.Create([1, 2, 3])

doubled = numbers | beam.Map(lambda x: x * 2)

tripled = numbers | beam.Map(lambda x: x * 3)
```

`numbers` 没有被修改。

## PTransform

PTransform 是 Beam 的数据转换操作。

|Transform|作用|
|---|---|
|Create|创建 PCollection|
|Map|每个元素执行一次映射|
|FlatMap|每个元素可以输出零个或多个元素|
|Filter|过滤数据|
|ParDo|执行自定义 DoFn|
|GroupByKey|按 Key 分组|
|Combine|聚合|
|CoGroupByKey|多个 PCollection 按 Key 关联|
|Flatten|合并多个 PCollection|
|Partition|将一个 PCollection 拆分成多个 PCollection|

## 2.2 ParDo 和 DoFn

ParDo 是 Beam 中非常重要的 Transform。

它会针对输入 PCollection 中的每个 Element 调用 DoFn。

```python
class MyDoFn(beam.DoFn):

    def process(self, element):
        yield element * 2

result = (
    p
    | beam.Create([1, 2, 3])
    | beam.ParDo(MyDoFn())
)
```

结果：

```text
[2, 4, 6]
```

ParDo 的特点：

- 每个 Element 都会被处理。
    
- 一个 Element 可以输出零个、一个或多个 Element。
    
- 可以实现复杂的业务逻辑。
    
- 可以结合 State 和 Timer 实现有状态处理。
    

## 2.3 Element 与 Bundle

Beam Runner 会将 Element 分配到不同的 Bundle 中。

```text
Pipeline
   |
PCollection
   |
Elements
   |
Runner 分组
   |
Bundles
   |
DoFn
   |
process(element)
```

例如：

```text
PCollection:
[1, 2, 3, 4, 5, 6, 7, 8]

Bundle 1:
[1, 2, 3, 4]

Bundle 2:
[5, 6, 7, 8]
```

注意：

- Bundle 是 Runner 的执行单位。
    
- Bundle 大小由 Runner 决定。
    
- 不应该假设一个 Bundle 包含固定数量的 Element。
    
- Bundle 可能因失败而重新执行。
    
- `process()` 是针对单个 Element 调用的，而不是一次接收整个 Bundle。
    

## 2.4 DoFn Lifecycle

DoFn 生命周期：

```python
class MyDoFn(beam.DoFn):

    # DoFn 实例初始化时
    def setup(self):
        pass

    # 一个 Bundle 开始处理时
    def start_bundle(self):
        pass

    # 每个 Element 处理时
    def process(self, element):
        yield element

    # 一个 Bundle 处理结束时
    def finish_bundle(self):
        pass

    # DoFn 实例释放时
    def teardown(self):
        pass
```

|Lifecycle|作用|
|---|---|
|setup()|初始化资源，例如数据库连接、客户端|
|start_bundle()|Bundle 开始时执行|
|process()|处理单条 Element|
|finish_bundle()|Bundle 结束时执行|
|teardown()|释放资源|

注意：

1. `setup()` 不是每条数据都会执行。
    
2. `start_bundle()` 和 `finish_bundle()` 与 Bundle 生命周期相关。
    
3. `teardown()` 主要用于清理资源，不应依赖它完成关键数据写入，因为它不保证在所有失败场景中执行。
    
4. 不应该使用 DoFn 实例的普通 Python 变量保存需要跨 Bundle、跨 Worker 持久化的业务状态。
    

## 面试问题

### Q1. What is a PCollection in Apache Beam?

中文解析：考察 Beam 的数据抽象模型。

英文回答：

A PCollection is a distributed collection of elements in Apache Beam.

It can represent either bounded data for batch processing or unbounded data for streaming processing.

PCollections are immutable, meaning that transformations create new PCollections rather than modifying existing ones.

### Q2. What is the difference between ParDo and Map?

中文解析：考察简单转换和自定义处理逻辑的区别。

英文回答：

Map applies a function to each element and typically produces one output element per input element.

ParDo is more flexible because it supports custom DoFns, multiple outputs, side inputs, state, and timers.

I would use Map for simple transformations and ParDo for more complex processing logic.

### Q3. What is a Bundle in Dataflow?

中文解析：考察 Runner 如何将数据分配给 Worker 执行。

英文回答：

A bundle is a group of elements processed together by a runner.

The runner divides a PCollection into bundles and distributes them across workers.

Bundle size and execution are managed by the runner, so developers should not assume a fixed bundle size.

### Q4. Explain the lifecycle of a DoFn.

中文解析：考察 setup、start_bundle、process、finish_bundle、teardown 的区别。

英文回答：

The DoFn lifecycle includes setup, start_bundle, process, finish_bundle, and teardown.

Setup initializes resources, start_bundle runs when a bundle begins, and process handles individual elements.

Finish_bundle runs when a bundle completes, while teardown is used for resource cleanup.

---

# 3. State API & Timer API

这是 Dataflow Streaming 中非常重要的知识点。

核心问题：

**如果需要跨多条消息保存某个 Key 的状态，应该怎么做？**

不能直接依赖普通 Python 变量，而应该使用 Beam State API。

## 3.1 为什么需要 State？

例如：

```python
class CountFn(beam.DoFn):

    def __init__(self):
        self.count = 0

    def process(self, element):
        self.count += 1
        yield self.count
```

这种写法不能保证业务状态正确。

原因：

- Worker 可能扩缩容。
    
- DoFn 实例可能被重新创建。
    
- 不同 Bundle 可能在不同 Worker 执行。
    
- 同一个 Key 的数据可能在不同时间处理。
    

普通实例变量不能保证跨 Worker、跨 Bundle 的持久化和一致性。

State API 允许 Beam Runner 管理需要持久保存的状态。

## 3.2 State 的作用范围

Beam State 通常是：

**Per Key + Per Window**

也就是说，不是所有数据共享一个全局 State。

例如：

```text
Key = user1
Window = 10:00 - 10:01

State:
count = 10
events = [A, B, C]
```

另一个用户：

```text
Key = user2
Window = 10:00 - 10:01

State:
count = 5
events = [D, E]
```

即使是同一个用户，在不同 Window 中也有不同的 State。

使用 State 时，输入通常需要是 Key-Value 形式。

## 3.3 State 适用场景

- Streaming processing
    
- Incremental processing
    
- Per-key counters
    
- Session tracking
    
- Deduplication
    
- 保存 Window 内的中间状态
    
- 记录尚未满足触发条件的数据
    

## 3.4 State 类型

|State Type|作用|典型用途|
|---|---|---|
|ValueState|保存一个值|Counter、当前状态|
|BagState|保存多个元素|缓存 Event|
|CombiningValueState|保存聚合结果|Count、Sum|
|MapState|保存 Key-Value 映射|用户属性缓存|
|SetState|保存不重复的元素|去重|

具体可用的 State 类型与 Beam SDK 版本有关。

## 3.5 ValueState 示例

```python
class CountFn(beam.DoFn):

    COUNT = beam.DoFn.StateParam(
        beam.transforms.userstate.ReadModifyWriteStateSpec(
            'count',
            beam.coders.VarIntCoder()
        )
    )

    def process(
        self,
        element,
        count=beam.DoFn.StateParam(COUNT)
    ):
        current = count.read() or 0

        count.write(current + 1)
```

逻辑：

```text
user1 receives event A
        |
        v
count.read() = 0
        |
        v
count.write(1)

user1 receives event B
        |
        v
count.read() = 1
        |
        v
count.write(2)
```

对于同一个 Key 和 Window，可以跨多次 `process()` 调用保存 Count。

## 3.6 BagState + CombiningValueState + Timer

假设有一个实时业务：

每个用户产生 Event，需要缓存 Event。

当一个 Window 结束，并且允许迟到时间结束后，将该用户在这个 Window 内的 Event 一次性输出。

可以使用：

- BagState：缓存 Event。
    
- CombiningValueState：记录 Event 数量。
    
- Timer：控制什么时候输出。
    
- Window：限定数据所属的时间范围。
    

```text
user1
  |
  +----------------------+
  |                      |
  v                      v
BagState             CountState
[A, B, C]                3
  |                      |
  +----------+-----------+
             |
             v
       Timer 到期
             |
             v
       读取 State
             |
             v
       批量输出 Event
             |
             v
       Clear State
```

示例：

```python
ALLOWED_LATENESS = 60

class BufferEvents(beam.DoFn):

    BUFFER_STATE = BagStateSpec(
        'buffer',
        StrUtf8Coder()
    )

    COUNT_STATE = CombiningValueStateSpec(
        'count',
        VarIntCoder(),
        sum
    )

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

        # 保存 Event
        buffer_state.add(event)

        # Count + 1
        count_state.add(1)

        # 设置 Event-time Timer
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

        events = list(buffer_state.read())
        count = count_state.read()

        yield {
            'window_start': window.start.to_utc_datetime(),
            'window_end': window.end.to_utc_datetime(),
            'count': count,
            'events': events
        }

        # 清理 State
        buffer_state.clear()
        count_state.clear()
```

### 重要补充

这段代码主要表达 State + Timer 的设计思路，实际运行还需要：

1. 输入元素具有 Key，例如 `(user_id, event)`。
    
2. Pipeline 必须正确配置 Window。
    
3. State 和 Timer 的声明要使用完整的 Beam SDK API。
    
4. `CombiningValueStateSpec` 的聚合行为需要与实际数据类型匹配。
    
5. Timer 的触发取决于对应 TimeDomain 和 Watermark。
    
6. 需要考虑迟到数据、State 清理和 Window 生命周期。
    

另外，Timer 不会自动等同于 Window Trigger。两者是不同机制：

- Window Trigger 决定 Window 何时输出 Pane。
    
- Timer 是用户在 DoFn 中设置的定时回调。
    

## 3.7 Timer API

Timer 可以在指定条件满足时执行回调。

常见类型：

|Timer 类型|依据|
|---|---|
|Event-time Timer|Watermark / Event Time|
|Processing-time Timer|实际处理时间|

典型用途：

- 延迟执行。
    
- Session 超时判断。
    
- 批量输出缓存数据。
    
- 处理长时间没有新消息的 Key。
    
- 定时清理 State。
    

## 面试问题

### Q1. Why do we need State API in Apache Beam?

中文解析：考察为什么不能依赖普通 Python 变量保存状态。

英文回答：

In distributed processing, workers can scale, restart, or process different bundles.

Therefore, normal instance variables are not reliable for maintaining business state across elements.

Beam State API allows us to maintain managed state for a specific key and window.

It is useful for counters, deduplication, session tracking, and incremental processing.

### Q2. What is the difference between ValueState and BagState?

中文解析：一个保存单个值，一个保存多个元素。

英文回答：

ValueState stores a single value, such as a counter or the current status of an entity.

BagState stores multiple elements and is useful when we need to buffer events for a particular key and window.

### Q3. What is the relationship between State and Timer?

中文解析：State 保存信息，Timer 决定何时执行后续逻辑。

英文回答：

State stores information across multiple element-processing calls, while Timer allows us to schedule a callback based on event time or processing time.

For example, I can use BagState to buffer events and an event-time timer to trigger batch processing after a specified time.

### Q4. Why should we clear State?

中文解析：防止状态持续累积、占用资源或影响后续处理。

英文回答：

State should be cleared when it is no longer needed.

Otherwise, it may continue to consume storage and cause unnecessary resource usage.

It can also lead to incorrect business results if old state is reused unexpectedly.

### Q5. What is the difference between Event-time Timer and Processing-time Timer?

中文解析：一个基于事件时间进度，一个基于 Worker 实际处理时间。

英文回答：

An event-time timer is triggered when the watermark reaches the specified event-time timestamp.

A processing-time timer is triggered according to the actual processing clock.

Event-time timers are useful for event-time-based business logic, while processing-time timers are useful for operational timeouts and periodic execution.

---

# 4. Data Transformation & Aggregation

## 4.1 GroupByKey

作用：

按照 Key 对 Value 进行分组。

输入：

```python
[
    ("A", 10),
    ("B", 20),
    ("A", 30)
]
```

代码：

```python
result = (
    p
    | beam.Create([
        ("A", 10),
        ("B", 20),
        ("A", 30)
    ])
    | beam.GroupByKey()
)
```

结果：

```python
[
    ("A", [10, 30]),
    ("B", [20])
]
```

核心理解：

**GroupByKey = 我要这一组数据。**

通常需要 Shuffle，因为相同 Key 的数据需要重新分布。

## 4.2 CombinePerKey

作用：

按照 Key 聚合数据。

```python
result = (
    p
    | beam.Create([
        ("A", 10),
        ("B", 20),
        ("A", 30)
    ])
    | beam.CombinePerKey(sum)
)
```

结果：

```python
[
    ("A", 40),
    ("B", 20)
]
```

核心理解：

**CombinePerKey = 我只要这一组数据计算出来的结果。**

## 4.3 CombineGlobally

对整个 PCollection 进行全局聚合。

```python
result = (
    p
    | beam.Create([10, 20, 30, 40])
    | beam.CombineGlobally(sum)
)
```

结果：

```python
[100]
```

## 4.4 Combine 的性能优势：Partial Combining

Combine 可以使用 CombineFn 进行局部聚合，再进行全局聚合。

```text
Worker 1:
A: 10, 20, 30
      |
      v
Local Combine
A: 60

Worker 2:
A: 40, 50
      |
      v
Local Combine
A: 90

       |
       v
     Shuffle
       |
       v
 Global Combine
       |
       v
     A: 150
```

假设有一百万条 A 数据：

- GroupByKey 可能需要传输大量明细数据。
    
- Combine 可以在本地先进行部分聚合。
    
- Shuffle 时只传输部分聚合结果。
    
- 最后再完成全局聚合。
    

因此：

**能使用 Combine 就不要先 GroupByKey 再自行聚合。**

但 CombineFn 必须能够正确合并局部聚合结果，适合可分解的聚合操作。

## 4.5 CoGroupByKey

用于多个 PCollection 按 Key 关联。

例如：

```python
orders = p | beam.Create([
    ("u1", "order1"),
    ("u2", "order2")
])

users = p | beam.Create([
    ("u1", "Alice"),
    ("u2", "Bob")
])
```

```python
grouped = ({
    "orders": orders,
    "users": users
} | beam.CoGroupByKey())
```

输出逻辑：

```text
u1:
  orders = [order1]
  users = [Alice]

u2:
  orders = [order2]
  users = [Bob]
```

可以进一步处理：

```python
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
```

```python
result = grouped | beam.FlatMap(join_data)
```

注意：这里的逻辑是按 Key 关联，但如果一个 Key 对应多个订单和多个用户，会产生组合结果，实际业务需要明确 Join 语义。

## 4.6 Side Input

Side Input 是将一个 PCollection 作为辅助数据，供主数据流查询。

例如：

- Orders 是主数据流。
    
- Users 是辅助数据。
    
- 每条 Order 需要查找对应 User。
    

```python
user_lookup = pipeline | beam.Create([
    ("u1", "Alice"),
    ("u2", "Bob")
])

orders = pipeline | beam.Create([
    ("u1", "order1"),
    ("u2", "order2")
])

user_dict = (
    user_lookup
    | beam.combiners.ToDict()
)

enriched = (
    orders
    | beam.Map(
        lambda order, users: (
            order[0],
            order[1],
            users.get(order[0])
        ),
        users=beam.pvalue.AsSingleton(user_dict)
    )
)
```

### CoGroupByKey vs Side Input

|对比|CoGroupByKey|Side Input|
|---|---|---|
|角色|两份或多份主数据流|主数据流 + 辅助数据|
|典型场景|大规模 Join|Lookup / Enrichment|
|数据处理|按 Key 重新分组|将辅助数据提供给主流|
|典型用途|Orders Join Users|Orders 查找 User 信息|

Side Input 适合相对较小、可以作为辅助数据使用的数据集。它不应该被简单理解为无限大的分布式 Hash Join。

## 4.7 Flatten 与 Partition

### Flatten

合并多个 PCollection。

```python
merged = (
    (collection1, collection2)
    | beam.Flatten()
)
```

要求输入 PCollection 的数据类型和 Schema 兼容。

### Partition

将一个 PCollection 根据条件拆分成多个 PCollection。

例如：

```text
Input
  |
  v
Partition
  |
  +----> Valid Data
  |
  +----> Invalid Data
```

## 面试问题

### Q1. What is the difference between GroupByKey and CombinePerKey?

中文解析：非常常见的性能与数据转换考点。

英文回答：

GroupByKey groups all values with the same key into an iterable.

CombinePerKey performs an aggregation for each key.

CombinePerKey can use partial aggregation before shuffle, which reduces the amount of intermediate data transferred across workers.

Therefore, when I only need an aggregation result, I prefer CombinePerKey over GroupByKey followed by manual aggregation.

### Q2. What is the difference between CoGroupByKey and Side Input?

中文解析：考察大规模 Join 与小型 Lookup 的使用场景。

英文回答：

CoGroupByKey is used to join multiple PCollections based on a common key.

Side Input is used when one PCollection acts as auxiliary data for another main input stream.

For example, I would use CoGroupByKey to join two large datasets, while Side Input is suitable for enriching orders with a relatively small user lookup table.

### Q3. Why is CombinePerKey often more efficient than GroupByKey?

中文解析：考察 Partial Combining 和 Shuffle。

英文回答：

CombinePerKey can perform local aggregation before the shuffle stage.

This reduces the amount of intermediate data that needs to be transferred between workers.

GroupByKey, on the other hand, needs to group values by key before downstream processing.

Therefore, CombinePerKey is generally more efficient when we only need an aggregation result.

### Q4. What is the difference between Flatten and Partition?

中文解析：一个合并多个 PCollection，一个拆分单个 PCollection。

英文回答：

Flatten merges multiple compatible PCollections into one PCollection.

Partition splits one PCollection into multiple PCollections based on a partitioning function.

---

# 5. Streaming Processing: Window、Watermark、Trigger

这是 Dataflow 面试中最重要的模块之一。

需要理解完整的数据处理过程：

```text
Incoming Events
      |
      v
Assign Event Timestamp
      |
      v
Window
      |
      v
Watermark
      |
      v
Trigger
      |
      v
Pane
      |
      v
Output
```

## 5.1 Event Time vs Processing Time

### Event Time

数据实际产生的时间。

例如：

用户在 10:00:10 产生了一条 Event。

### Processing Time

Dataflow Worker 实际处理这条 Event 的时间。

例如：

这条 Event 因为网络延迟，在 10:00:30 才被处理。

```text
Event Time:      10:00:10
Processing Time: 10:00:30
```

两者不一定相同。

## 5.2 Data Freshness vs System Latency

### Data Freshness

用户看到的数据有多新。

例如：

当前是 10:05，但 Dashboard 显示的数据只更新到 10:00。

数据新鲜度较差。

### System Latency

一条数据从产生到处理、输出花费了多久。

可以将端到端延迟拆解为：

```text
Event Created
     |
     v
Ingestion Delay
     |
     v
Processing Delay
     |
     v
Delivery / Query Delay
     |
     v
Data Available to User
```

可能的延迟来源：

1. Source 到 Dataflow 的 Ingestion Delay。
    
2. Worker 处理能力不足导致的 Processing Delay。
    
3. Window / Trigger 等待导致的输出延迟。
    
4. Sink 写入和查询延迟。
    

## 5.3 Window

Window 将无限数据流切分成有限的数据集合，以便进行聚合。

三种常见 Window：

### Fixed Window

固定时间长度，窗口之间不重叠。

```python
fixed_window = (
    pcoll
    | beam.WindowInto(
        beam.window.FixedWindows(60)
    )
)
```

例如：

```text
10:00:00 - 10:01:00
10:01:00 - 10:02:00
10:02:00 - 10:03:00
```

适合：

- 每分钟订单数。
    
- 每小时销售额。
    
- 每天访问量。
    

### Sliding Window

窗口长度和滑动间隔可以不同，因此可能重叠。

```python
sliding_window = (
    pcoll
    | beam.WindowInto(
        beam.window.SlidingWindows(180, 15)
    )
)
```

例如：

- Window Size = 180 秒。
    
- Period = 15 秒。
    

每隔 15 秒生成一个覆盖过去 180 秒数据的窗口。

适合：

- 最近 3 分钟访问量。
    
- 每 15 秒更新一次最近 3 分钟指标。
    

### Session Window

根据数据活动情况动态生成窗口。

```python
session_window = (
    pcoll
    | beam.WindowInto(
        beam.window.Sessions(120)
    )
)
```

例如：

用户两次 Event 之间的间隔超过 120 秒，就可能形成新的 Session。

适合：

- 用户行为分析。
    
- 用户会话统计。
    
- 活动间隔分析。
    

## 5.4 Watermark

Watermark 用来表示系统对 Event Time 数据处理进度的估计。

例如：

```text
Watermark = 10:00:00
```

表示系统认为时间戳早于 10:00:00 的数据已经基本到齐。

注意：

**Watermark 不是绝对保证不会再有更早时间戳的数据到达。**

迟到数据仍然可能出现。

### Allowed Lateness

Allowed Lateness 允许 Window 在 Watermark 越过 Window End 后，继续接收一段时间的迟到数据。

例如：

```text
Window:
10:00 - 10:01

Watermark:
10:01

Allowed Lateness:
60 seconds
```

系统可以继续接收属于该 Window 的迟到 Event，直到相应的清理边界。

Allowed Lateness 并不意味着数据一定会被保留 60 秒后才输出。Trigger 可以先输出结果，之后再输出迟到数据更新。

## 5.5 Trigger

Trigger 决定 Window 什么时候输出结果。

常见 Trigger：

|Trigger|触发条件|
|---|---|
|AfterWatermark|Watermark 达到指定时间|
|AfterProcessingTime|经过指定处理时间|
|AfterCount|收到指定数量的 Element|
|Repeatedly|重复执行指定 Trigger|

### Event-time Trigger

```python
beam.trigger.AfterWatermark()
```

根据 Watermark 判断 Window 是否可以触发。

### Processing-time Trigger

```python
beam.trigger.AfterProcessingTime(10)
```

根据实际处理时间触发。

### Count Trigger

```python
beam.trigger.AfterCount(100)
```

收到 100 个 Element 后触发。

### Late Data Trigger

AfterWatermark 可以配合 Late Trigger。

例如：

```python
beam.trigger.AfterWatermark(
    late=beam.trigger.AfterCount(1)
)
```

Watermark 经过 Window End 后，如果有迟到数据，可以在收到迟到数据时触发新的输出。

## 5.6 Accumulating vs Discarding

一个 Window 可以输出多个 Pane。

两种主要 Accumulation Mode：

### Accumulating

每次触发输出当前 Window 的累计结果。

例如：

```text
第一次触发:
10

第二次触发:
15

第三次触发:
20
```

这里的 15 和 20 都是累计结果。

### Discarding

每次触发只输出上次触发之后的新数据所对应的结果。

例如：

```text
第一次触发:
10

第二次触发:
5

第三次触发:
5
```

### 示例

```python
result = (
    pcoll
    | "Window" >> beam.WindowInto(
        beam.window.FixedWindows(60),
        trigger=beam.trigger.AfterWatermark(),
        accumulation_mode=beam.trigger.AccumulationMode.DISCARDING
    )
)
```

## 5.7 TimestampedValue

如果输入数据没有正确携带 Event Time，需要根据业务时间字段设置 Timestamp。

```python
def add_timestamp(element):

    event_time = datetime.strptime(
        element.timestamp,
        '%Y-%m-%dT%H:%M:%S.%fZ'
    )

    ts = event_time.timestamp()

    return beam.window.TimestampedValue(
        element,
        ts
    )
```

然后：

```python
pcoll = (
    p
    | beam.io.ReadFromText(input_path)
    | beam.Map(parse_json)
    | beam.Map(add_timestamp)
    | beam.WindowInto(
        beam.window.FixedWindows(60)
    )
)
```

## 5.8 Windowed Aggregation 示例

每分钟统计访问量：

```python
result = (
    p
    | "Read" >> beam.io.ReadFromText(input_path)
    | "Parse" >> beam.Map(parse_json)
    | "AddTimestamp" >> beam.Map(add_timestamp)
    | "Window" >> beam.WindowInto(
        beam.window.FixedWindows(60)
    )
    | "Count" >> beam.CombineGlobally(
        beam.combiners.CountCombineFn()
    ).without_defaults()
)
```

注意：

如果使用 `CombineGlobally`，需要考虑 Window 和默认值行为；对于真实流式业务，还应明确 Trigger、Allowed Lateness 和 Accumulation Mode。

## 5.9 Dataflow Watermark vs Databricks Watermark

Databricks Structured Streaming：

```python
.withWatermark("event_time", "10 minutes")
```

这里的 10 minutes 是允许迟到数据处理的时间阈值之一，会影响状态清理以及迟到数据的处理行为。

不要把它简单理解成 Watermark 永远固定落后 10 分钟。

Beam 和 Spark 都有 Watermark 概念，但具体的计算方式、状态清理和触发语义需要结合各自执行引擎理解。

## 面试问题

### Q1. What is the difference between Event Time and Processing Time?

中文解析：考察事件实际发生时间与系统处理时间的区别。

英文回答：

Event Time is the time when an event actually occurred, while Processing Time is the time when the system processes the event.

They can be different because of network delays, late-arriving data, or processing backlogs.

For accurate real-time analytics, I generally prefer Event Time because it reflects the actual business event time.

### Q2. What is a Watermark in Dataflow?

中文解析：考察系统如何判断 Event Time 的处理进度。

英文回答：

A watermark is the system's estimate of how far event-time processing has progressed.

It helps the pipeline determine when a window can be triggered.

However, a watermark does not guarantee that no earlier event will arrive. Late events can still be handled using allowed lateness and late triggers.

### Q3. What is the difference between Fixed, Sliding, and Session Windows?

中文解析：考察不同 Window 的使用场景。

英文回答：

Fixed windows divide data into non-overlapping intervals.

Sliding windows have a fixed size but can overlap, allowing us to calculate metrics over a moving time range.

Session windows group events based on periods of activity separated by inactivity gaps.

I would use fixed windows for hourly reports, sliding windows for rolling metrics, and session windows for user activity analysis.

### Q4. What is the difference between Allowed Lateness and Trigger?

中文解析：Allowed Lateness 决定允许迟到数据进入的时间范围；Trigger 决定什么时候输出。

英文回答：

Allowed lateness defines how long a window can continue accepting late data after the watermark passes its end.

A trigger determines when the window emits results.

They work together: a window may emit an initial result and later emit updated results when late data arrives.

### Q5. What is the difference between Accumulating and Discarding mode?

中文解析：考察多次触发时输出累计值还是增量值。

英文回答：

Accumulating mode emits the full accumulated result for the window each time it fires.

Discarding mode emits only the data received since the previous firing.

The choice depends on whether downstream consumers need complete updated results or incremental updates.

---

# 6. Dataflow I/O & GCP Integration

## 6.1 TextIO

用于读取和写入文本文件。

### ReadFromText

```python
pcoll = (
    pipeline
    | "Read" >> beam.io.ReadFromText(file_name)
)
```

### ReadAllFromText

当文件名本身来自 PCollection 时，可以使用 ReadAllFromText。

```python
pcoll = (
    pipeline
    | "Create" >> beam.Create([file_name])
    | "ReadAll" >> beam.io.ReadAllFromText()
)
```

### WriteToText

```python
transformed_data | beam.io.WriteToText(
    known_args.output
)
```

区别：

|方法|作用|
|---|---|
|ReadFromText|读取指定文件|
|ReadAllFromText|根据 PCollection 中的文件路径读取多个文件|
|WriteToText|将数据写入文本文件|

## 6.2 FileIO：持续发现新文件

Java Beam 可以使用 FileIO 的匹配和持续监控能力。

```java
p.apply(
    FileIO.match()
        .filepattern("gs://bucket/input/*")
        .continuously(
            Duration.standardSeconds(30),
            Watch.Growth.afterTimeSinceNewOutput(
                Duration.standardHours(1)
            )
        )
);
```

逻辑：

1. 定期检查文件。
    
2. 发现新增文件。
    
3. 将新增文件加入后续处理流程。
    
4. 根据 Growth 条件判断是否继续监控。
    

适合文件持续写入 GCS 的场景。

## 6.3 BigQueryIO

用于读取和写入 BigQuery。

### Read From Query

```java
PCollection<Double> maxTemperatures =
    p.apply(
        BigQueryIO.read(
            (SchemaAndRecord elem) ->
                (Double) elem.getRecord().get("max_temperature")
        )
        .fromQuery(
            "SELECT max_temp FROM dataset.table"
        )
        .usingStandardSql()
        .withCoder(DoubleCoder.of())
    );
```

主要使用场景：

- 从 BigQuery Table 读取。
    
- 通过 SQL 查询读取。
    
- 将处理结果写入 BigQuery。
    

### Write to BigQuery

Python 示例：

```python
| "WriteToBQ" >> beam.io.WriteToBigQuery(
    table_name,
    schema=table_schema,
    create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED,
    write_disposition=beam.io.BigQueryDisposition.WRITE_TRUNCATE
)
```

常见参数：

|参数|作用|
|---|---|
|CREATE_IF_NEEDED|表不存在时创建|
|CREATE_NEVER|表必须已经存在|
|WRITE_TRUNCATE|写入前覆盖目标表|
|WRITE_APPEND|追加数据|
|WRITE_EMPTY|目标表必须为空|

实际项目中，还需要根据数据量和写入要求选择合适的 BigQuery 写入方式。

## 6.4 PubsubIO

Pub/Sub 是 GCP 的消息传递服务。

Dataflow 可以从 Pub/Sub Topic 读取消息：

```python
pipeline = (
    p
    | "Read PubSub" >> beam.io.ReadFromPubSub(
        topic=input_topic
    )
    | "Window" >> beam.WindowInto(
        beam.window.FixedWindows(60)
    )
)
```

典型架构：

```text
Application
     |
     v
Pub/Sub Topic
     |
     v
Dataflow
     |
     v
Transform / Window / Aggregate
     |
     v
BigQuery
```

适合：

- 实时日志处理。
    
- 实时交易处理。
    
- 实时指标统计。
    
- Event-driven Architecture。
    

## 6.5 KafkaIO、BigtableIO、AvroIO

|IO|用途|
|---|---|
|KafkaIO|从 Kafka 读取或写入数据|
|BigtableIO|读写 Cloud Bigtable|
|AvroIO|读写 Avro 文件|

Avro 的特点：

- Schema-based serialization。
    
- 支持结构化数据。
    
- 可以进行 Schema Evolution。
    
- 适合分布式数据交换和批量文件存储。
    

## 6.6 Batch vs Streaming Pipeline

|对比|Batch|Streaming|
|---|---|---|
|数据|有限数据集|持续到达的数据|
|执行|通常有开始和结束|长时间运行|
|典型输入|GCS、BigQuery|Pub/Sub、Kafka|
|处理方式|批量处理|持续处理|
|典型输出|BigQuery、GCS|BigQuery、Bigtable|
|延迟目标|通常更高|通常更低|

## 面试问题

### Q1. How would you design a real-time pipeline using Pub/Sub and Dataflow?

中文解析：考察端到端 Streaming 架构。

英文回答：

I would use Pub/Sub as the messaging layer to ingest events from upstream applications.

Dataflow would consume messages from Pub/Sub, validate and transform the events, assign event timestamps, and apply windowing and aggregation.

The processed data could then be written to BigQuery for analytics or Bigtable for low-latency serving.

I would also monitor processing latency, backlog, and failed messages.

### Q2. What is the difference between ReadFromText and ReadAllFromText?

中文解析：考察文件路径是固定输入还是动态输入。

英文回答：

ReadFromText reads from a specified file pattern or path.

ReadAllFromText takes file paths from a PCollection and reads the corresponding files.

ReadAllFromText is useful when file names are generated dynamically.

### Q3. How do you choose between Batch and Streaming?

中文解析：考察业务需求与数据处理架构的匹配。

英文回答：

I choose batch processing when the data is bounded and immediate results are not required.

I choose streaming when events need to be processed continuously with low latency.

For example, daily financial reports can use batch processing, while fraud detection and real-time monitoring are better suited to streaming pipelines.

---

# 7. Performance Optimization & Troubleshooting

这是生产 Dataflow 项目中非常重要的模块。

核心排查路径：

```text
Dataflow Insights
       |
       v
Hot Key Detection
       |
       v
Job Graph
       |
       v
Detailed Metrics
       |
       v
Logs
       |
       v
Identify Bottleneck
       |
       v
Optimize Pipeline
```

## 7.1 Dataflow Insights

Dataflow Insights 可以帮助识别 Pipeline 的性能问题。

重点关注：

- Hot Key Detection
    
- Data Skew
    
- Performance Recommendations
    
- Worker Resource Utilization
    

### Hot Key

Hot Key 指某些 Key 的数据量明显高于其他 Key。

例如：

```text
Key A: 1,000,000 records
Key B: 100 records
Key C: 200 records
```

当大量数据集中在 Key A 时，处理 Key A 的 Worker 可能成为瓶颈。

表现：

- 部分 Worker 忙碌。
    
- 其他 Worker 空闲。
    
- GroupByKey 处理时间变长。
    
- 整个 Pipeline 出现长尾延迟。
    

## 7.2 Job Graph

Job Graph 可以展示 Pipeline 的执行结构。

排查时：

1. 查看哪些 Step 耗时较长。
    
2. 查看 Shuffle 操作。
    
3. 查看是否存在数据倾斜。
    
4. 检查哪个 Transform 是瓶颈。
    
5. 查看上下游 Step 的处理速度。
    

## 7.3 Metrics

重点关注：

|Metric|可能反映的问题|
|---|---|
|CPU utilization|CPU 计算瓶颈|
|Memory utilization|内存压力|
|Disk utilization|磁盘 I/O 瓶颈|
|System latency|系统处理延迟|
|Backlog|待处理数据积压|
|Data freshness|数据新鲜度|
|Worker count|Worker 扩缩容情况|

## 7.4 Logs

检查：

- Worker 错误。
    
- API 请求失败。
    
- Serialization 错误。
    
- 数据格式错误。
    
- Sink 写入失败。
    
- 外部系统连接问题。
    

不要在高吞吐 Pipeline 中对每条数据大量输出 INFO 或 ERROR 日志。

原因：

- 增加 CPU 开销。
    
- 增加日志 I/O。
    
- 增加日志存储成本。
    
- 影响处理吞吐量。
    

更好的方法：

- 使用 Dead Letter Queue。
    
- 将异常数据单独保存。
    
- 统计失败数量。
    
- 记录必要的错误上下文。
    

## 7.5 Data Skew Optimization

常见方法：

### 1. withHotKeyFanout()

对于存在 Hot Key 的 Combine 操作，可以使用 Hot Key Fanout。

作用：

将一个热点 Key 的处理拆分成多个子任务进行局部聚合，再进行最终合并。

### 2. withFanout()

通过 Fanout 增加聚合并行度。

### 3. 增加随机前缀 / Hash Salting

例如：

```text
Original Key:
user1

Split Keys:
user1_0
user1_1
user1_2
user1_3
```

先根据拆分后的 Key 进行局部聚合，再去掉随机前缀进行最终聚合。

适合：

- 热点 Key。
    
- 数据倾斜。
    
- 单个 Key 数据量过大。
    

注意：

Dataflow Shuffle Service 可以改善 Shuffle 的数据管理，但不能直接解决业务 Key 分布不均衡的问题。

## 7.6 Hot Key Logging

可以开启相关 Hot Key 日志能力，帮助定位具体的热点 Key。

但日志本身不应成为高吞吐 Pipeline 的主要诊断方式。

应结合 Dataflow Insights、Job Graph 和 Metrics 分析。

## 7.7 Key 数量过少

如果 Key 数量太少，可能导致并行度不足。

例如：

```text
100 million records
Only 2 keys
```

即使有大量 Worker，也无法充分并行处理这两个 Key 对应的工作。

解决方案：

- 增加合理的 Key 分布。
    
- 对热点 Key 做 Hash Salting。
    
- 对不同 Window 的数据进行合理拆分。
    
- 采用局部聚合再全局聚合。
    

如果不同 Window 的处理彼此独立，可以考虑将 Window 信息纳入拆分 Key 的设计，但需要保证业务聚合语义正确。

## 7.8 Fusion Optimization

Fusion 是 Dataflow Runner 自动进行的执行优化。

多个 Transform 可能被合并到一个执行阶段中，减少中间数据传输和不必要的调度开销。

例如：

```text
Read
  |
  v
Map
  |
  v
Filter
  |
  v
ParDo
  |
  v
Write
```

Runner 可能将部分相邻 Transform Fusion 在一起。

优势：

- 减少中间数据交换。
    
- 减少调度开销。
    
- 提高执行效率。
    

潜在问题：

Fusion 有时可能限制后续计算的并行度。

例如，一个 Worker 连续执行大量计算，导致其他 Worker 无法充分参与。

## 7.9 Reshuffle

Reshuffle 可以重新分发数据，并打破部分 Fusion 边界。

```text
ParDo
  |
  v
Fanout
  |
  v
Reshuffle
  |
  v
Next Transform
```

使用场景：

- 需要重新分配数据。
    
- 需要增加后续阶段并行度。
    
- Fusion 导致并行度不足。
    
- 某个 Worker 承担过多连续处理工作。
    

### Fusion vs Reshuffle

|对比|Fusion|Reshuffle|
|---|---|---|
|目的|减少中间处理开销|重新分发数据|
|数据交换|尽量避免不必要交换|通常引入数据重新分布|
|并行度|可能限制并行度|可能恢复并行度|
|性能影响|减少开销|增加 Shuffle 开销|
|是否一定更快|不一定|不一定|

注意：

不要为了避免 Fusion 而到处使用 Reshuffle。Reshuffle 本身会产生数据交换开销。

## 7.10 Aggregation 与 Fusion

Aggregation 通常会引入数据重新分布和同步处理。

例如：

```text
ParDo
  |
  v
GroupByKey
  |
  v
ParDo
```

GroupByKey 需要 Shuffle，因此不能简单地把前后两个 ParDo 当作普通连续操作直接 Fusion。

## 7.11 Serialization Optimization

Encoding 和 Decoding 可能成为性能瓶颈。

优化方向：

- 避免反复序列化大型对象。
    
- 只传递后续步骤真正需要的字段。
    
- 避免为了读取一个字段而反复解析整个大型对象。
    
- 根据数据结构选择合适的 Coder。
    
- 对适合的场景使用结构化数据格式。
    

例如：

```text
Large Object
     |
     v
Extract Required Fields
     |
     v
Small Structured Record
     |
     v
Next Transform
```

## 7.12 Window Optimization

如果一个 Sliding Window 很大，直接处理大量明细数据可能带来较高开销。

可以考虑：

```text
Raw Events
    |
    v
Small Window Aggregation
    |
    v
Intermediate Aggregated Results
    |
    v
Larger Window Aggregation
    |
    v
Final Result
```

先对明细数据进行局部聚合，再对中间结果进行后续聚合。

但必须保证时间边界、迟到数据处理和聚合语义正确，不能随意把不同 Window 的结果合并。

## 7.13 Compression 与 File Format

TextIO 读取大量文本时，可能存在：

- 文件解析开销。
    
- 解压缩开销。
    
- Serialization 开销。
    
- 文件切分能力受限。
    

可以考虑：

- 使用适合并行读取的文件格式。
    
- 根据数据结构选择 Avro 或 Parquet 等格式。
    
- 根据压缩算法和文件大小权衡 CPU 与 I/O。
    

注意：不能简单认为压缩文件一定比未压缩文件快。压缩会降低 I/O，但也会增加解压缩 CPU 开销；部分压缩格式还会影响并行读取。

## 7.14 External API Bottleneck

Dataflow 本身处理能力可能很强，但下游外部系统的处理能力有限。

例如：

```python
def process(element):
    call_external_api(element)
```

如果每条数据都调用一次外部 API：

- API QPS 可能达到限制。
    
- 下游系统可能过载。
    
- 网络延迟增加。
    
- Pipeline 可能出现 Backlog。
    

解决方法：

### GroupIntoBatches

```python
events | beam.GroupIntoBatches(100)
```

将多条数据组织成 Batch，再批量调用外部服务。

### Bundle Buffering

```text
Start Bundle
     |
     v
Create Buffer
     |
     v
Process Elements
     |
     v
Add to Buffer
     |
     v
Finish Bundle
     |
     v
Batch External API Calls
```

注意：

StartBundle / FinishBundle 可以用于 Bundle 级别的资源管理和批量处理，但 Bundle 失败时可能重新执行。

外部 API 调用应考虑幂等性、重试和重复请求问题。

## 7.15 Dataflow Troubleshooting

如果 Dataflow Pipeline 突然变慢：

1. 检查 Dataflow Insights。
    
2. 检查 Hot Key 和 Data Skew。
    
3. 查看 Job Graph 中最慢的 Step。
    
4. 检查 CPU、Memory、Disk 和 Backlog。
    
5. 检查输入数据量是否突然增加。
    
6. 检查外部 API 或 Sink 是否变慢。
    
7. 检查日志和异常。
    
8. 根据瓶颈选择优化方法。
    

## 面试问题

### Q1. How would you troubleshoot a slow Dataflow pipeline?

中文解析：考察生产环境问题定位能力。

英文回答：

I would first check Dataflow Insights to identify potential hot keys and performance recommendations.

Then I would inspect the job graph to locate slow transformations.

I would correlate the slow steps with CPU, memory, disk, and backlog metrics.

Finally, I would review worker logs and downstream dependencies to determine whether the bottleneck comes from data skew, resource limitations, serialization, or external systems.

### Q2. What is a Hot Key and how do you solve it?

中文解析：考察数据倾斜及其解决方法。

英文回答：

A hot key occurs when a small number of keys receive a disproportionately large amount of data.

This can cause uneven workload distribution, where some workers become overloaded while others remain idle.

I would consider hot key fanout, key salting, or local aggregation to distribute the workload more evenly.

### Q3. What is Fusion in Dataflow?

中文解析：考察 Runner 如何减少中间数据交换。

英文回答：

Fusion is an optimization performed by the Dataflow runner.

It combines compatible transformations into fewer execution stages, reducing intermediate data exchange and scheduling overhead.

However, fusion can sometimes reduce parallelism, so I would inspect the job graph before deciding whether to introduce a reshuffle boundary.

### Q4. What is Reshuffle and when would you use it?

中文解析：考察如何打破 Fusion 并重新分配数据。

英文回答：

Reshuffle redistributes elements and can introduce a boundary that prevents certain fusion optimizations.

I would use it when fusion causes insufficient parallelism or when downstream processing needs to distribute data across workers again.

However, reshuffle introduces additional data exchange overhead, so it should only be used when necessary.

### Q5. How do you handle an external API bottleneck in Dataflow?

中文解析：考察 Pipeline 吞吐量与下游系统容量之间的关系。

英文回答：

If each element triggers an external API call, the downstream service may become a bottleneck.

I would consider batching multiple elements using GroupIntoBatches to reduce the number of API requests.

I would also implement retries, rate limiting, and idempotency to handle failures and duplicate requests.

### Q6. What metrics would you monitor in a production Dataflow pipeline?

中文解析：考察生产监控指标。

英文回答：

I would monitor system latency, data freshness, backlog, throughput, worker count, CPU utilization, memory usage, and error rates.

For streaming pipelines, I would pay particular attention to backlog growth and watermark progression because they can indicate processing delays.

---

# 8. Error Handling & Reliability

## 8.1 Pipeline Failure Types

Pipeline 可能在不同阶段失败：

### 1. Pipeline Construction Failure

Pipeline 在构建阶段失败。

例如：

- Python 语法错误。
    
- Beam Transform 参数错误。
    
- Schema 不匹配。
    
- Coder 配置错误。
    

### 2. Job Submission / Startup Failure

Job 提交或启动时失败。

例如：

- Dataflow API 未启用。
    
- Service Account 权限不足。
    
- GCS Staging Bucket 不可访问。
    
- Worker 资源无法分配。
    
- Region 配置错误。
    

### 3. Runtime Failure

Pipeline 执行过程中失败。

例如：

- Worker OOM。
    
- 数据解析异常。
    
- 外部 API 失败。
    
- BigQuery 写入失败。
    
- 网络连接失败。
    

### 4. Sink / External Dependency Failure

下游系统出现故障。

例如：

- BigQuery 配额限制。
    
- Pub/Sub 权限问题。
    
- 外部 API 限流。
    
- 数据库连接中断。
    

## 8.2 Batch vs Streaming Failure Recovery

Batch Pipeline：

- 失败后可以根据 Runner 的执行机制重新尝试。
    
- 可以重新执行失败的工作单元。
    
- 需要考虑重复执行和外部副作用。
    

Streaming Pipeline：

- 长时间持续运行。
    
- 某些可恢复错误可能被 Runner 自动重试。
    
- 发生不可恢复错误时，Job 仍可能失败，需要重新启动或恢复。
    
- 应设计持久化状态和恢复机制。
    

不要把 Batch 固定重试 4 次、Streaming 永远重试作为通用规则。实际重试次数与错误类型、Runner 和服务配置有关。

## 8.3 Dead Letter Queue

对于无法正常处理的数据，可以将其写入 Dead Letter Queue。

```text
Incoming Data
      |
      v
Validation
      |
      +------------------+
      |                  |
      v                  v
Valid Data          Invalid Data
      |                  |
      v                  v
Main Pipeline       Dead Letter Queue
      |                  |
      v                  v
BigQuery             Error Analysis
```

优势：

- 不让少量坏数据影响整个 Pipeline。
    
- 保留失败数据。
    
- 便于后续重新处理。
    
- 可以统计失败数量和失败原因。
    

## 8.4 Idempotency

在分布式处理系统中，数据可能因为失败恢复而被重复处理。

例如：

```text
Process Event A
      |
      v
Write to BigQuery
      |
      v
Worker Failure
      |
      v
Retry Event A
      |
      v
Write Event A Again
```

如果没有幂等性设计，可能产生重复数据。

常见解决方法：

- 使用唯一 Event ID。
    
- 使用 Upsert / Merge。
    
- 使用业务主键去重。
    
- 使用幂等写入逻辑。
    
- 使用具有明确提交语义的 Sink。
    

## 面试问题

### Q1. What are the common failure types in Dataflow?

中文解析：考察从构建、提交到运行阶段的故障定位。

英文回答：

Dataflow failures can occur during pipeline construction, job submission, startup, or runtime execution.

Common causes include invalid pipeline configuration, insufficient IAM permissions, worker resource issues, malformed input data, and downstream service failures.

I would first identify the failure stage and then inspect the corresponding logs and metrics.

### Q2. How do you handle bad records in a Dataflow pipeline?

中文解析：考察异常数据隔离与数据质量处理。

英文回答：

I would separate valid and invalid records during validation.

Valid records continue through the main pipeline, while invalid records are sent to a dead-letter queue with error details.

This prevents bad records from blocking the entire pipeline and allows us to analyze and reprocess them later.

### Q3. How do you prevent duplicate processing in a streaming pipeline?

中文解析：考察 Exactly-once、At-least-once 和幂等性。

英文回答：

I would assign a unique event identifier and implement idempotent writes in the sink.

For example, I could use a business key with a MERGE operation in BigQuery to avoid inserting duplicate records.

I would also consider the delivery guarantees of the source and sink because exactly-once processing depends on the entire pipeline, not just the processing engine.

### Q4. What happens when a Dataflow worker fails?

中文解析：考察分布式执行的容错能力。

英文回答：

Dataflow can detect worker failures and recover processing based on its execution and fault-tolerance mechanisms.

Depending on the pipeline and processing stage, work may be retried or reassigned to other workers.

However, external side effects should be designed carefully because retries may cause duplicate operations.

---

# 9. Dataflow ML & Generative AI Integration

## 9.1 Dataflow 与 ML Pipeline

Dataflow 可以用于构建数据处理和机器学习数据准备 Pipeline。

例如：

```text
Incoming Events
      |
      v
Data Ingestion
      |
      v
Data Transformation
      |
      v
Embedding Generation
      |
      v
Feature Enrichment
      |
      v
ML Inference
      |
      v
Prediction / Recommendation
      |
      v
BigQuery / Bigtable
```

典型步骤：

1. Ingest Event。
    
2. 清洗和转换数据。
    
3. 生成 Embedding。
    
4. Enrichment。
    
5. 调用 ML Model。
    
6. 输出 Prediction 或 Recommendation。
    

Dataflow 负责分布式数据处理与 Pipeline 编排，不应将其等同于模型训练服务。

## 9.2 BigQuery 与 Gemini 集成

你的笔记中还有一组 BigQuery ML Remote Model 的例子。

需要区分：

**这是 BigQuery 与 Gemini 的集成，不是 Dataflow 本身的 ML API。**

### 创建 GCS 图片 External Table

```sql
CREATE OR REPLACE EXTERNAL TABLE
`gemini_demo.review_images`
WITH CONNECTION `us.gemini_conn`
OPTIONS (
  object_metadata = 'SIMPLE',
  uris = [
    'gs://bucket/images/*'
  ]
);
```

用途：

将 GCS 中的图片对象作为外部数据资源访问。

### 创建 Remote Model

```sql
CREATE OR REPLACE MODEL
`gemini_demo.gemini_flash`
REMOTE WITH CONNECTION `us.gemini_conn`
OPTIONS (
  endpoint = 'gemini-3.5-flash'
);
```

用途：

通过 BigQuery Connection 调用外部 Gemini Model。

### 使用 ML.GENERATE_TEXT

```sql
CREATE OR REPLACE TABLE
`gemini_demo.customer_reviews_keywords` AS

SELECT
  ml_generate_text_llm_result,
  social_media_source,
  review_text,
  customer_id,
  location_id,
  review_datetime

FROM ML.GENERATE_TEXT(
  MODEL `gemini_demo.gemini_flash`,
  (
    SELECT
      social_media_source,
      customer_id,
      location_id,
      review_text,
      review_datetime,
      CONCAT(
        'For each review, provide keywords from the review. ',
        'Answer in JSON format with one key: keywords. ',
        'Keywords should be a list.',
        review_text
      ) AS prompt
    FROM `gemini_demo.customer_reviews`
  ),
  STRUCT(
    0.2 AS temperature,
    TRUE AS flatten_json_output
  )
);
```

执行逻辑：

1. 读取 Customer Reviews。
    
2. 构造 Prompt。
    
3. 调用 Remote Gemini Model。
    
4. 生成关键词。
    
5. 将结果保存到 BigQuery Table。
    

## 面试问题

### Q1. How can Dataflow be used in an ML pipeline?

中文解析：考察 Dataflow 在 ML 数据处理中的定位。

英文回答：

Dataflow can be used to build scalable data processing pipelines for machine learning workloads.

For example, it can ingest events, clean and transform data, generate features or embeddings, and invoke prediction services.

The results can then be stored in BigQuery or Bigtable for analytics or online serving.

### Q2. What is the difference between Dataflow ML processing and BigQuery ML?

中文解析：考察数据处理服务与数据库内 ML 能力的区别。

英文回答：

Dataflow is a distributed data processing service that can orchestrate feature engineering and inference workflows.

BigQuery ML allows users to create and run machine learning models directly within BigQuery using SQL.

For example, I can use Dataflow to process streaming events and BigQuery ML to run predictions on analytical datasets.

---

# 10. Streaming Pipeline Update、Snapshot、Drain

这是 Dataflow 生产运维中非常重要的高级知识。

主要解决：

**如何在 Streaming Pipeline 持续运行的情况下更新代码、保留状态、回滚版本，以及避免数据丢失？**

## 10.1 Snapshot

Snapshot 用于保存 Streaming Pipeline 的状态，以便后续恢复。

核心内容包括：

- Pipeline 的部分执行状态。
    
- Stateful Transform 的状态。
    
- 相关处理进度信息。
    

基本流程：

```text
Old Streaming Pipeline
        |
        v
Create Snapshot
        |
        v
Save Pipeline State
        |
        v
Create New Pipeline
        |
        v
Restore from Snapshot
```

### Snapshot 的用途

1. 保存当前 Pipeline 状态。
    
2. 为新 Pipeline 提供恢复起点。
    
3. 在更新失败时提供恢复手段。
    
4. 用于迁移和维护 Streaming Job。
    

注意：

Snapshot 不等于完整的系统备份，也不意味着可以对任何不兼容的 Pipeline 修改进行无条件恢复。

## 10.2 Snapshot with Source

对于 Pub/Sub 等 Source，还需要考虑 Source 的读取进度。

理想情况下：

```text
Pipeline State
      +
Source Processing Position
      |
      v
Consistent Recovery Point
```

不仅需要恢复 Dataflow 内部状态，还需要考虑 Source 中未处理消息的位置。

对于 Pub/Sub，可以结合 Pub/Sub Snapshot 和 Subscription Seek 机制管理消息重放。

对于 Kafka，可以利用 Offset 和消息保留机制进行重新消费。

注意：

Source 的恢复位置、Pipeline 状态和 Sink 的提交情况必须一起考虑，否则可能发生重复处理或遗漏。

## 10.3 Transform Name Mapping

更新 Pipeline 时，旧 Pipeline 和新 Pipeline 需要保持 Stateful Transform 的状态映射关系。

例如：

```text
Old Pipeline:
Transform A
     |
     v
State A

New Pipeline:
Transform B
     |
     v
State B
```

如果 Transform 名称发生变化，需要在支持的更新场景中提供对应的 Name Mapping。

否则，新 Pipeline 可能无法正确识别旧 Pipeline 中的状态。

## 10.4 哪些变化可能导致 Update 失败？

常见风险：

1. 修改 Stateful Transform 的名称，但没有正确映射。
    
2. 增加或删除 Side Input。
    
3. 修改 Coder。
    
4. 删除 Stateful Operation。
    
5. 改变不兼容的 Pipeline 结构。
    

核心原则：

**Streaming Pipeline Update 需要兼容旧 Pipeline 中的状态结构和状态编码方式。**

## 10.5 Update vs Snapshot vs Blue-Green

|方式|解决的问题|
|---|---|
|Snapshot|如何保存和恢复 Pipeline 状态|
|Update|如何更新 Pipeline 并保留兼容状态|
|Blue-Green|如何让新旧 Pipeline 独立运行并验证后切换|

### Snapshot

```text
Old Pipeline
     |
     v
Snapshot
     |
     v
Save State
```

### Update

```text
Old Pipeline
     |
     v
Update
     |
     v
New Pipeline
     |
     v
Continue Processing
```

### Blue-Green

```text
             +----------------+
             |                |
             v                v
       Old Pipeline      New Pipeline
             |                |
             v                v
        Old Target         New Target
             |                |
             +-------+--------+
                     |
                     v
              Validate Results
                     |
                     v
                Switch Traffic
```

Blue-Green 的优点：

- 新旧 Pipeline 可以独立验证。
    
- 可以逐步切换。
    
- 可以降低直接修改生产 Pipeline 的风险。
    

代价：

- 可能需要更多计算资源。
    
- 可能产生重复消费。
    
- 需要处理双写和结果一致性。
    

## 10.6 Drain

当 Pipeline Update 不兼容时，可以考虑 Drain。

Drain 的核心思想：

**停止接收新的输入，让已经进入 Pipeline 的数据尽可能处理完成，然后结束 Job。**

```text
Streaming Pipeline
        |
        v
Drain
        |
        v
Stop New Input
        |
        v
Process In-flight Data
        |
        v
Finish Job
        |
        v
Start New Pipeline
```

### Drain vs Cancel

|对比|Drain|Cancel|
|---|---|---|
|目的|优雅停止|直接停止|
|输入|停止继续处理新输入|终止当前执行|
|已接收数据|尽可能处理完成|可能丢弃尚未处理的数据|
|适用场景|计划停机、Pipeline 更新|紧急终止|

### Drain 的注意事项

Drain 并不意味着所有业务 Window 都一定产生完整结果。

对于 Streaming Pipeline：

- 未完成的 Window 可能不会按照原来的业务时间边界自然完成。
    
- 部分聚合结果可能不完整。
    
- 应根据业务要求处理终止时产生的 Pane。
    
- 可以使用 PaneInfo 辅助判断输出属于哪个 Pane。
    
- 必须明确如何处理不完整的聚合结果。
    

不能简单认为 Drain 一定保证所有业务聚合完整，也不能简单认为所有数据一定不丢失。

## 10.7 Pub/Sub Snapshot 与 Seek

一种恢复流程：

```bash
# 1. 创建 Pub/Sub Snapshot
gcloud pubsub snapshots create my-snapshot \
  --subscription=my-sub
```

```text
2. Drain Old Dataflow Job
```

```bash
# 3. 将 Subscription Seek 到 Snapshot
gcloud pubsub subscriptions seek my-sub \
  --snapshot=my-snapshot
```

```text
4. Start New Dataflow Job
```

这样可以从指定的消息位置重新消费。

注意：

- Seek 可能导致消息重复消费。
    
- 必须考虑消息 Retention。
    
- 必须保证下游写入具有幂等性。
    
- Dataflow Snapshot 与 Pub/Sub Snapshot 是不同的概念，不能混为一谈。
    

## 10.8 Snapshot Retention

笔记记录：

Dataflow Snapshot 的保留时间为 7 天。

这个数值需要根据具体 Snapshot 类型及当前服务规则核实，不应将其作为所有 Snapshot 类型通用的固定规则。

## 10.9 Flex Templates

Flex Template 用于将 Dataflow Pipeline 打包并提供可复用的部署入口。

主要组成：

1. Pipeline Application。
    
2. Docker Image。
    
3. Template Metadata。
    
4. GCS 上的 Template Specification。
    

基本流程：

```text
Developer
    |
    v
Build Pipeline Application
    |
    v
Build Docker Image
    |
    v
Push Image to Artifact Registry
    |
    v
Create Template Specification
    |
    v
Store Specification in GCS
    |
    v
Submit Flex Template Job
    |
    v
Dataflow Launches Pipeline
```

Flex Template 可以让用户通过参数启动已经打包好的 Pipeline。

相较于直接提交 Pipeline 代码，Flex Template 更适合标准化、可重复的生产部署。

## 10.10 Pipeline Updating Strategies

三种常见方案：

### 方案 1：完全停止旧 Pipeline，再启动新 Pipeline

```text
Stop Old Pipeline
       |
       v
Handle Source Position
       |
       v
Start New Pipeline
```

适合不需要保留旧 Pipeline State 的场景。

需要考虑：

- Source Offset。
    
- Pub/Sub Snapshot。
    
- Kafka Replay。
    
- 数据重复。
    
- 数据缺失。
    
- 下游幂等性。
    

### 方案 2：使用 Update

```text
Old Pipeline
       |
       v
Compatible Update
       |
       v
New Pipeline
       |
       v
Reuse Compatible State
```

适合状态结构兼容的更新。

### 方案 3：Blue-Green Deployment

```text
Old Pipeline ------> Old Target
       |
       |
New Pipeline ------> New Target
       |
       v
Validate New Results
       |
       v
Switch Consumers
```

适合需要降低生产升级风险、进行新旧版本对比的场景。

## 面试问题

### Q1. What is a Dataflow Snapshot?

中文解析：考察 Streaming Pipeline 的状态保存与恢复。

英文回答：

A Dataflow snapshot captures the state of a streaming pipeline so that the pipeline can be restored later.

It is useful for pipeline migration, recovery, and version updates.

However, the new pipeline must be compatible with the saved state.

### Q2. What is the difference between Update and Drain?

中文解析：一个尝试保留状态进行更新，一个优雅地结束旧 Job 后再启动新 Job。

英文回答：

Update allows us to modify a running streaming pipeline while preserving compatible state.

Drain gracefully stops the pipeline after processing data that has already entered the pipeline.

I would prefer Update when the new pipeline is state-compatible, and Drain when the required changes are not compatible with the existing pipeline.

### Q3. What is the difference between Snapshot and Blue-Green Deployment?

中文解析：Snapshot 是状态恢复机制，Blue-Green 是部署策略。

英文回答：

A snapshot captures the internal state of a pipeline for recovery or migration.

Blue-green deployment is a deployment strategy where old and new pipelines run independently.

The new pipeline can be validated before switching downstream consumers to it.

### Q4. Why can a Dataflow pipeline update fail?

中文解析：考察 Stateful Transform 的兼容性。

英文回答：

A pipeline update may fail if the new pipeline is incompatible with the old pipeline's state structure.

For example, renaming stateful transforms without mapping, changing coders, or modifying stateful operations can cause compatibility issues.

Therefore, I would review the pipeline graph and state compatibility before performing an update.

### Q5. What is the difference between Drain and Cancel?

中文解析：考察优雅停止和直接终止。

英文回答：

Drain gracefully stops a streaming pipeline after processing in-flight data.

Cancel terminates the job without the same graceful processing behavior.

I would use Drain for planned pipeline maintenance and Cancel for emergency termination.

### Q6. How would you upgrade a production streaming pipeline with minimal risk?

中文解析：考察生产部署、状态迁移、数据一致性。

英文回答：

First, I would determine whether the new pipeline is compatible with the existing pipeline state.

If it is compatible, I would consider a Dataflow update.

If it is not compatible, I would consider draining the old pipeline and starting a new one from a controlled source position.

For higher-risk changes, I would use a blue-green deployment strategy, validate the new pipeline, and switch consumers only after confirming the results.

I would also ensure that downstream writes are idempotent to handle potential duplicate events.

---

# 11. 高频面试知识点总表

|模块|必须掌握的概念|重点面试方向|
|---|---|---|
|Architecture|Beam、Runner、Worker|Beam vs Dataflow|
|Shuffle|Shuffle Service|Shuffle 如何工作|
|Streaming Engine|Streaming Execution|Streaming Engine 的作用|
|FlexRS|Spot、资源调度|成本与延迟 Trade-off|
|Beam Model|Pipeline、PCollection、DoFn|Beam 编程模型|
|Bundle|Element、Bundle|Runner 如何执行任务|
|State|Value、Bag、Combining|如何跨 Element 保存状态|
|Timer|Event-time、Processing-time|如何延迟触发逻辑|
|Aggregation|GroupByKey、Combine|Partial Combining|
|Join|CoGroupByKey、Side Input|大规模 Join vs Lookup|
|Window|Fixed、Sliding、Session|Window 选择|
|Watermark|Event Time、Late Data|如何处理迟到数据|
|Trigger|AfterWatermark、AfterCount|什么时候输出|
|Accumulation|Accumulating、Discarding|多次触发如何输出|
|I/O|GCS、BigQuery、Pub/Sub|端到端 Pipeline 设计|
|Optimization|Fusion、Reshuffle|如何提高并行度|
|Data Skew|Hot Key、Fanout|如何解决数据倾斜|
|Monitoring|Insights、Job Graph、Metrics|如何定位性能瓶颈|
|Reliability|DLQ、Retry、Idempotency|如何处理失败和重复数据|
|Updating|Snapshot、Update、Drain|如何升级生产 Pipeline|
|Deployment|Flex Templates|如何标准化部署|

# 12. 最值得优先准备的 Dataflow 面试题

1. What is the difference between Apache Beam and Dataflow?
    
2. How does Dataflow Shuffle Service work?
    
3. What is the difference between Streaming Engine and Dataflow Shuffle?
    
4. What is the difference between PCollection and Bundle?
    
5. Why should we use State API instead of normal Python variables?
    
6. What is the difference between ValueState and BagState?
    
7. How do State and Timer work together?
    
8. What is the difference between GroupByKey and CombinePerKey?
    
9. Why is CombinePerKey more efficient in many aggregation scenarios?
    
10. What is the difference between CoGroupByKey and Side Input?
    
11. What is the difference between Event Time and Processing Time?
    
12. How does Watermark work?
    
13. How do you handle late-arriving data?
    
14. What is the difference between Fixed, Sliding, and Session Windows?
    
15. What is the difference between Trigger and Allowed Lateness?
    
16. What is the difference between Accumulating and Discarding?
    
17. How would you design a real-time pipeline using Pub/Sub and Dataflow?
    
18. How would you troubleshoot a slow Dataflow pipeline?
    
19. What is a Hot Key and how do you solve data skew?
    
20. What is the difference between Fusion and Reshuffle?
    
21. How would you handle an external API bottleneck?
    
22. How do you handle bad records in Dataflow?
    
23. How do you prevent duplicate processing?
    
24. What is the difference between Snapshot, Update, and Drain?
    
25. How would you upgrade a production streaming pipeline with minimal downtime?