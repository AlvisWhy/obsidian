# Google Cloud Data Engineer：Streaming Data 知识整理

## 1. Streaming Architecture：流式数据处理基础

### 1.1 Streaming Data 的概念

Streaming Data（流式数据）指数据持续、实时或近实时地产生，并且需要在数据到达后持续处理。

常见数据来源：

- Application logs：应用日志。
    
- Clickstream：用户点击行为。
    
- IoT sensors：物联网传感器。
    
- Financial transactions：金融交易。
    
- Database CDC：数据库变更事件。
    

典型架构：

```
Data Sources
     │
     ↓
Message Broker
(Pub/Sub / Kafka)
     │
     ↓
Stream Processing
(Dataflow / Apache Flink)
     │
     ↓
Data Sink
(BigQuery / Bigtable / GCS)
     │
     ↓
Analytics / Dashboard / ML
```

各组件职责：

|组件|职责|
|---|---|
|Pub/Sub / Kafka|接收、缓冲、分发消息|
|Dataflow|实时处理、转换、聚合数据|
|BigQuery|分析和查询流式数据|
|Bigtable|低延迟读写和大规模数据访问|
|Cloud Storage|存储原始数据和处理结果|

需要区分：

- Message Broker：负责消息传输、缓冲和分发。
    
- Stream Processing Engine：负责对消息进行计算和转换。
    
- Data Sink：负责保存处理后的结果。
    

Pub/Sub 和 Kafka 本身不是完整的数据处理引擎，通常需要配合 Dataflow、Flink 或其他消费者应用。

面试问题

1. What is stream processing, and how is it different from batch processing?
    
2. What is the role of a message broker in a streaming architecture?
    
3. How would you design a real-time data pipeline on Google Cloud?
    
4. What is the difference between message ingestion and stream processing?
    

## 2. Pub/Sub 与 Managed Kafka

### 2.1 Google Cloud Pub/Sub

Pub/Sub（发布订阅消息服务） 是 Google Cloud 提供的全托管消息服务。

核心特点：

- Serverless：不需要用户管理 Broker、Partition 和服务器。
    
- Automatic scaling：根据消息流量自动扩展。
    
- Decoupling：Publisher 和 Subscriber 不需要直接通信。
    
- Message retention：在配置的保留期限内保存消息。
    
- Acknowledgment：Subscriber 处理完成后确认消息。
    
- Message ordering：可以通过 Ordering Key 保证特定消息顺序。
    
- GCP integration：与 Dataflow、BigQuery、Cloud Storage 等服务集成。
    

### 2.2 Managed Kafka

Google Cloud Managed Service for Apache Kafka 是托管式 Apache Kafka 服务。

它保留 Kafka 的核心架构和兼容性，同时由 Google Cloud 管理底层基础设施。

Kafka 的主要组件：

|组件|职责|
|---|---|
|Producer|发送消息|
|Topic|消息逻辑分类|
|Partition|Topic 的物理分片|
|Broker|存储和处理 Partition|
|Consumer|消费消息|
|Consumer Group|多个 Consumer 协同消费|
|Offset|记录 Consumer 的消费位置|

Kafka 的消息保存在 Topic 的 Partition 中，Consumer 通过 Offset 追踪读取进度。

### 2.3 Pub/Sub vs Managed Kafka

|比较|Pub/Sub|Managed Kafka|
|---|---|---|
|服务类型|Serverless 消息服务|托管式 Kafka 集群|
|基础设施管理|Google 自动管理|Google 管理集群基础设施，用户仍需配置集群和 Topic 等|
|消息组织|Topic + Subscription|Topic + Partition|
|消费进度|Subscription 的 Ack 状态|Consumer Group 的 Offset|
|消费模型|Pull / Push / 导出型 Subscription|Consumer 主动拉取|
|扩展|自动扩展|需要结合 Partition、Consumer 和集群容量规划|
|消息顺序|Ordering Key 范围内有序|单个 Partition 内有序|
|消息保留|按 Topic / Subscription 保留策略|按 Topic retention 配置|
|生态系统|GCP 原生集成|Kafka 生态系统和兼容客户端|
|典型用途|GCP 原生事件驱动架构|Kafka 生态、已有 Kafka 系统迁移、多云或跨平台集成|

选型思路：

- 如果主要使用 GCP，希望减少基础设施和 Kafka 集群管理工作，可以考虑 Pub/Sub。
    
- 如果需要 Kafka API、Kafka Connect、Kafka Streams 或兼容现有 Kafka 生态，可以考虑 Managed Kafka。
    
- 如果需要严格控制 Kafka Partition、Consumer Group、Offset 等消费模型，Kafka 的模型更直接。
    

注意：Pub/Sub 和 Kafka 都可以支持大规模流式数据处理，但两者的消费进度、消息分区和扩展方式不同，不能简单认为它们完全等价。

面试问题

1. What is the difference between Google Cloud Pub/Sub and Managed Kafka?
    
2. When would you choose Pub/Sub over Kafka?
    
3. What are the main components of Kafka?
    
4. What is the difference between a Kafka Topic and a Partition?
    
5. Why is Kafka widely used in event-driven architectures?
    

## 3. Pub/Sub 核心架构与消息流转

### 3.1 Topic、Publisher、Subscription、Subscriber

Pub/Sub 使用发布订阅模型，核心组件如下：

|组件|职责|
|---|---|
|Publisher|发布消息|
|Topic|接收并存储消息的逻辑入口|
|Subscription|定义消息如何交付给 Subscriber|
|Subscriber|接收并处理消息|
|Acknowledgment（Ack）|告诉 Pub/Sub 消息已经成功处理|

消息流转：

```
Publisher
    │
    │ Publish
    ↓
  Topic
    │
    ├──────────────┐
    ↓              ↓
Subscription A   Subscription B
    │              │
    ↓              ↓
Subscriber A    Subscriber B
```

重要：Topic 和 Subscription 是不同的概念。

- 一个 Topic 可以关联多个 Subscription。
    
- 每个 Subscription 有自己的消息消费状态。
    
- 不同 Subscription 可以独立消费同一个 Topic 的消息。
    
- 同一个 Pull Subscription 可以由多个 Subscriber 实例共同消费，形成负载分担。
    

例如：

```
              Orders Topic
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
    Analytics Sub        Fraud Sub
          │                   │
          ↓                   ↓
    Dataflow             Fraud Service
          │                   │
          ↓                   ↓
      BigQuery           Alert System
```

Analytics 和 Fraud 可以分别处理同一条订单消息，互不影响各自的消费进度。

### 3.2 Subscription 的四种类型

Pub/Sub 支持四种常见 Subscription 类型：

|类型|消息交付方式|是否需要自己实现 Subscriber|
|---|---|---|
|Pull|Subscriber 主动请求消息|是|
|Push|Pub/Sub 主动向 HTTP Endpoint 推送消息|是，需要提供 Endpoint|
|BigQuery|自动写入 BigQuery|不需要自己实现消息消费程序|
|Cloud Storage|自动导出到 Cloud Storage|不需要自己实现消息消费程序|

Pull Subscription

Subscriber 主动从 Pub/Sub 获取消息：

```
Pub/Sub
   │
   │ Pull
   ↓
Subscriber
   │
   │ Process
   ↓
   │ Ack
   ↓
Pub/Sub
```

适合：

- 需要控制消费速度。
    
- 需要批量拉取消息。
    
- 需要自定义消费逻辑。
    
- 需要 Exactly-once delivery。
    

Push Subscription

Pub/Sub 将消息发送到 Subscriber 提供的 HTTP Endpoint。

```
Pub/Sub
   │
   │ HTTP POST
   ↓
Subscriber Endpoint
   │
   │ Process
   ↓
HTTP 2xx Response
   │
   ↓
Pub/Sub considers message acknowledged
```

Subscriber 通过成功的 HTTP 响应确认消息。处理失败或未及时响应时，消息可能重新投递。

BigQuery Subscription

```
Publisher
   ↓
Pub/Sub Topic
   ↓
BigQuery Subscription
   ↓
BigQuery Table
```

Pub/Sub 自动将消息写入指定的 BigQuery Table，不需要额外部署一个自定义 Subscriber 来搬运数据。

Cloud Storage Subscription

```
Publisher
   ↓
Pub/Sub Topic
   ↓
Cloud Storage Subscription
   ↓
Cloud Storage Bucket
```

Pub/Sub 自动将消息批量导出为 Cloud Storage 对象。

注意：

- BigQuery 和 Cloud Storage Subscription 是导出型 Subscription。
    
- 它们不支持 Pub/Sub Exactly-once delivery。
    
- 如果需要自定义转换、复杂处理或实时聚合，通常使用 Pull Subscription 配合 Dataflow。
    

### 3.3 Fan-in、Fan-out 与 Straight-through

三种常见消息拓扑：

1. Straight-through：一对一

```
Publisher
    │
    ↓
  Topic
    │
    ↓
Subscriber
```

一个 Publisher 向一个 Subscriber 发布和传递消息。

2. Fan-in：多对一

```
Publisher A ──┐
              │
Publisher B ──┼──→ Topic ──→ Subscriber
              │
Publisher C ──┘
```

多个 Publisher 将消息发送到同一个 Topic，由 Subscriber 统一消费。

典型场景：多个微服务产生事件，统一进入一个消息系统。

3. Fan-out：一对多

```
                 Topic
                   ↑
               Publisher
                   │
          ┌────────┼────────┐
          ↓        ↓        ↓
         Sub A    Sub B    Sub C
          ↓        ↓        ↓
       Service A Service B Service C
```

一个 Publisher 发布消息，多个 Subscription 分别将消息传递给不同的业务系统。

典型场景：一个订单事件同时触发数据分析、库存更新和风控检查。

面试问题

1. What is the difference between a Topic and a Subscription in Pub/Sub?
    
2. Can multiple subscribers consume messages from the same subscription?
    
3. What is the difference between Pull and Push subscriptions?
    
4. What is the difference between a Pull subscription and a BigQuery subscription?
    
5. Explain fan-in and fan-out messaging patterns.
    
6. How would you distribute the same event to multiple downstream services?
    

## 4. Message Delivery Semantics：消息投递语义

### 4.1 At-least-once Delivery

At-least-once 表示消息至少会被成功投递一次，但可能被投递多次。

默认情况下，Pub/Sub 使用 At-least-once delivery。

消息流程：

```
Publisher
    │
    ↓
Pub/Sub
    │
    ↓
Subscriber receives message
    │
    ↓
Process message
    │
    ├── Ack succeeds
    │      ↓
    │   Message acknowledged
    │
    └── Ack fails / deadline expires
           ↓
        Redelivery
```

例如：

1. Subscriber 收到 `order_id=1001`。
    
2. Subscriber 完成数据写入。
    
3. 但 Ack 请求没有成功到达 Pub/Sub。
    
4. Pub/Sub 认为消息尚未成功确认。
    
5. 消息可能再次投递。
    

这样可能导致同一条消息被处理多次。

因此，Subscriber 应该尽可能实现 Idempotent Processing（幂等处理）。

例如：

```
MERGE INTO orders AS target
USING incoming_orders AS source
ON target.order_id = source.order_id

WHEN MATCHED THEN
  UPDATE SET target.amount = source.amount

WHEN NOT MATCHED THEN
  INSERT (order_id, amount)
  VALUES (source.order_id, source.amount);
```

通过业务主键和 MERGE 逻辑，避免重复消息产生重复记录。

### 4.2 Exactly-once Delivery

Pub/Sub 支持 Exactly-once delivery，但有重要限制：

- 仅支持 Pull Subscription，包括 StreamingPull。
    
- 不支持 Push、BigQuery 和 Cloud Storage Subscription。
    
- 保证范围是同一个 Cloud Region 内的 Pub/Sub 消息投递。
    
- 成功 Ack 后，不会因为同一条消息的重新投递而再次交付。
    
- 相比普通投递模式，会增加消息延迟。
    

Exactly-once 的核心机制：

1. Subscriber 收到消息。
    
2. Pub/Sub 为这次投递提供 Ack ID。
    
3. Subscriber 处理消息。
    
4. Subscriber 使用有效 Ack ID 发送 Ack。
    
5. Pub/Sub 确认 Ack 是否成功。
    
6. 如果消息被重新投递，之前的 Ack ID 将失效，只能使用最新投递对应的 Ack ID。
    

例如：

```
Message M
    │
    ↓
Delivery 1 → Ack ID 1
    │
    ├── Ack ID 1 过期 / 消息重新投递
    │
    ↓
Delivery 2 → Ack ID 2
    │
    ├── 使用 Ack ID 1 → 失败
    │
    └── 使用 Ack ID 2 → 成功
                         │
                         ↓
                   No redelivery
```

注意两个概念：

- Redelivery：消息没有成功确认，Pub/Sub 重新投递。
    
- Duplicate：已经成功确认的消息又被重复交付，或者 Publisher 重试导致相同业务事件以不同消息 ID 再次发布。
    

Exactly-once delivery 主要解决 Pub/Sub 层面的重复投递问题，不等于整个端到端业务事务绝对只执行一次。比如 Subscriber 写入 BigQuery 后，在 Ack 前崩溃，仍然需要考虑外部写入的幂等性。

### 4.3 At-least-once vs Exactly-once

|比较|At-least-once|Exactly-once|
|---|---|---|
|默认模式|是|否，需要启用|
|重复投递|可能发生|成功 Ack 后不会发生同一消息的重复投递|
|Ack 成功状态|不提供同等程度的确认语义|Subscriber 可以确认 Ack 是否成功|
|Subscription 类型|多种类型|仅 Pull|
|延迟|通常较低|通常更高|
|应用要求|建议幂等|仍需考虑业务幂等|
|适用场景|高吞吐、可容忍重复并能去重|对消息重复投递敏感的场景|

### 4.4 Acknowledgment Deadline

Acknowledgment Deadline 是 Pub/Sub 等待 Subscriber 确认消息的时间。

默认 Ack Deadline 通常是 10 秒，最大可配置为 600 秒。

如果 Subscriber 在 Deadline 到期前没有 Ack：

```
Subscriber receives message
          │
          ↓
     Processing
          │
          │ Ack Deadline expires
          ↓
     Message redelivery
```

对于处理时间较长的任务，可以通过 `ModifyAckDeadline` 延长消息的确认时间。

客户端库还可以自动延长 Ack Deadline，避免长时间处理任务因为初始 Deadline 过短而频繁重新投递。

面试问题

1. What is the difference between at-least-once and exactly-once delivery?
    
2. How does Pub/Sub prevent duplicate delivery in exactly-once mode?
    
3. What happens when a subscriber fails to acknowledge a message?
    
4. What is an acknowledgment deadline?
    
5. Why should a subscriber be idempotent even when exactly-once delivery is enabled?
    
6. Does exactly-once delivery guarantee exactly-once execution of an external database transaction?
    

## 5. Message Ordering：消息顺序

### 5.1 Pub/Sub Message Ordering

Pub/Sub 默认不保证消息按照 Publisher 的发送顺序交付。

如果需要保证消息顺序，可以启用 Message Ordering，并设置 Ordering Key。

例如：

```
Ordering Key = customer_001

Message 1
Message 2
Message 3
```

Pub/Sub 可以保证同一个 Ordering Key 的消息按照 Pub/Sub 接收到的顺序交付。

示例：

```
publisher.publish(    topic_path,    b"Order created",    ordering_key="customer_001")
```

不同 Ordering Key 之间不保证顺序：

```
customer_001:
    Message 1 → Message 2 → Message 3

customer_002:
    Message 1 → Message 2 → Message 3
```

两个 Key 的消息可以并行处理。

### 5.2 Ordering 的代价

Ordering 会带来一定的性能和可用性限制。

例如：

```
Ordering Key = customer_001

Message 1 → Processing failed
Message 2 → Waiting
Message 3 → Waiting
```

同一个 Ordering Key 的后续消息可能受到前面消息失败或重新投递的影响。

因此：

- 同一个 Key 内的消息处理可能被串行化。
    
- 某个 Key 的消息处理失败可能阻塞后续消息。
    
- 不同 Key 之间仍然可以并行处理。
    
- 热点 Ordering Key 可能成为吞吐量瓶颈。
    

Pub/Sub 每个 Ordering Key 的 Publisher 吞吐量也存在限制，需要结合实际消息大小和业务流量进行规划。

### 5.3 Pub/Sub Ordering vs Kafka Ordering

|比较|Pub/Sub|Kafka|
|---|---|---|
|顺序保证范围|同一个 Ordering Key|同一个 Partition|
|并行处理|不同 Ordering Key 可以并行|不同 Partition 可以并行|
|消息分组方式|Ordering Key|Partition Key|
|顺序控制|Subscription 启用 Message Ordering|Producer 将相同 Key 路由到同一 Partition|
|扩展瓶颈|热点 Ordering Key|热点 Partition|

Kafka 中，如果需要保证同一个用户的事件顺序，可以将 `user_id` 作为 Message Key，使相同用户的消息进入同一个 Partition。

面试问题

1. Does Pub/Sub guarantee message ordering by default?
    
2. How do you enable message ordering in Pub/Sub?
    
3. What is an Ordering Key?
    
4. Why can message ordering reduce throughput?
    
5. What is the difference between Pub/Sub Ordering Key and Kafka Partition Key?
    
6. How would you handle a hot key in a streaming system?
    

## 6. Message Retention：消息保留机制

### 6.1 Subscription Message Retention

Pub/Sub 默认保留未确认的消息 7 天。

Subscription 的 Message Retention Duration：

- Default：7 days。
    
- Minimum：10 minutes。
    
- Maximum：31 days。
    

如果消息已经被 Ack，默认情况下 Pub/Sub 会从该 Subscription 中删除消息。

但是可以开启：

`Retain acknowledged messages`

这样已经 Ack 的消息也可以在配置的保留期限内继续保存，并支持通过 Seek 重新消费历史消息。

### 6.2 Topic Message Retention

Topic 也可以启用 Message Retention。

开启后，Topic 会在最多 31 天的配置期限内保留消息。

Topic Retention 和 Subscription Retention 的区别：

|比较|Topic Retention|Subscription Retention|
|---|---|---|
|保存位置|Topic 层|Subscription 层|
|主要作用|为多个订阅保留消息|保存某个订阅的消息积压或历史消息|
|默认|不启用|7 天|
|最大期限|31 天|31 天|
|Ack 后是否保留|由 Topic Retention 控制|需要开启 Retain acknowledged messages|

如果同时启用 Topic Retention 和 Subscription Retention，消息会按照两者中较长的保留期限来决定何时从 Subscription 中删除。

### 6.3 Message Retention vs Subscription Expiration

这两个概念容易混淆。

Message Retention： 消息保留多久。

Subscription Expiration： Subscription 多久没有活动后会被删除。

默认 Subscription Expiration 是 31 天无活动。

可以配置 Subscription 永不过期（Never expire）。

注意：

- Message Retention 最长 31 天。
    
- Subscription 可以配置为永不过期。
    
- Subscription 永不过期不代表消息永远保留。
    
- Pub/Sub 不是无限期消息存储系统。
    

### 6.4 Replay Messages

如果需要重新处理历史消息，可以使用 Pub/Sub Seek 功能。

例如：

```
Topic
  │
  ↓
Subscription
  │
  ├── Message 1 (Acked)
  ├── Message 2 (Acked)
  ├── Message 3 (Acked)
  │
  ↓
Seek to previous timestamp
  │
  ↓
Replay historical messages
```

典型场景：

- Subscriber 处理逻辑出现 Bug。
    
- 需要重新计算历史数据。
    
- 下游数据库出现故障。
    
- 需要恢复之前的消息处理状态。
    

面试问题

1. What is the default message retention period in Pub/Sub?
    
2. What is the maximum message retention period?
    
3. What is the difference between Topic Retention and Subscription Retention?
    
4. What happens to an acknowledged message by default?
    
5. How can you replay previously acknowledged messages?
    
6. What is the difference between Message Retention and Subscription Expiration?
    

## 7. Message Filtering 与 Attributes

### 7.1 Message Data 和 Attributes

Pub/Sub Message 主要包含两个部分：

- Data：实际的业务消息内容。
    
- Attributes：消息附带的 Key-Value Metadata。
    

例如：

```
publisher.publish(    topic_path,    b'{"order_id": 1001, "amount": 200}',    author="data_team",    event_type="order_created",    environment="production")
```

这里：

```
Data:
{
  "order_id": 1001,
  "amount": 200
}

Attributes:
{
  "author": "data_team",
  "event_type": "order_created",
  "environment": "production"
}
```

Attributes 主要用于描述消息、路由和过滤，不是业务数据本身。

### 7.2 Subscription Filtering

Pub/Sub 支持通过 Message Attributes 创建 Subscription Filter。

例如：

```
attributes.event_type = "order_created"
```

只有符合条件的消息才会被交付给该 Subscription。

架构：

```
                  Topic
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
  Filter: order_created   Filter: payment
          │                   │
          ↓                   ↓
    Order Subscriber    Payment Subscriber
```

重要限制：

Pub/Sub Subscription Filter 只能根据 Message Attributes 过滤，不能直接解析 Message Data 中的 JSON 字段进行过滤。

例如：

```
{
  "order_id": 1001,
  "amount": 200
}
```

如果 `amount` 只存在于 Data 中，而没有作为 Attribute 发布，就不能直接使用 Subscription Filter 过滤 `amount > 100`。

解决方案：

1. Publisher 提前提取需要过滤的字段。
    
2. 将字段作为 Message Attributes 发布。
    
3. Subscriber 根据 Attributes 过滤消息。
    

例如：

```
publisher.publish(    topic_path,    b'{"order_id": 1001, "amount": 200}',    amount_category="high")
```

之后可以使用：

```
attributes.amount_category = "high"
```

注意：Filter 创建后不能修改或删除，因此设计 Topic 和 Subscription 时应该提前考虑过滤需求。

面试问题

1. What is the difference between message data and message attributes?
    
2. How does Pub/Sub subscription filtering work?
    
3. Can Pub/Sub filter messages based on JSON fields inside the message body?
    
4. How would you route different event types to different subscribers?
    
5. What would you do if you need to filter messages based on a field inside the payload?
    

## 8. Scalability、Backpressure 与 Dead Letter Topic

### 8.1 Pub/Sub Scalability

Pub/Sub 是 Serverless 服务，能够根据消息流量自动扩展。

用户不需要自行管理 Broker 数量或 Partition 数量。

但是 Subscriber 仍然需要具备足够的处理能力。

例如：

```
Publisher: 10,000 messages/sec
                  │
                  ↓
                Topic
                  │
                  ↓
       Subscriber capacity:
          5,000 messages/sec
                  │
                  ↓
              Backlog
```

如果 Publisher 的速度持续高于 Subscriber 的处理速度，消息积压（Backlog）就会增加。

解决方式：

- 增加 Subscriber 实例。
    
- 使用 StreamingPull 提高消费效率。
    
- 优化单条消息的处理逻辑。
    
- 对耗时操作进行异步化。
    
- 检查下游 BigQuery、数据库等是否成为瓶颈。
    

### 8.2 Backpressure

Backpressure 指下游处理能力不足时，需要控制上游消息流入或消费速度，避免系统资源耗尽。

例如：

```
Publisher
    │
    ↓
Pub/Sub
    │
    ↓
Subscriber
    │
    ↓
Slow Database
```

如果 Database 写入速度较慢，Subscriber 可能无法及时处理消息，导致 Backlog 增长。

需要监控：

- Subscription backlog。
    
- Oldest unacked message age。
    
- Subscriber throughput。
    
- Ack latency。
    
- Redelivery rate。
    
- Downstream latency。
    

### 8.3 Dead Letter Topic（DLQ）

如果消息多次处理失败，可以将其发送到 Dead Letter Topic。

```
Pub/Sub Topic
      │
      ↓
Subscriber
      │
      ├── Success → Ack
      │
      └── Failure
             │
             ↓
       Retry delivery
             │
             ↓
       Max attempts reached
             │
             ↓
       Dead Letter Topic
             │
             ↓
       Error Investigation
```

DLQ 主要用于：

- 隔离无法处理的消息。
    
- 避免 Poison Message 不断重试。
    
- 保留失败消息用于排查。
    
- 后续修复数据或处理逻辑后重新处理。
    

注意：Dead Letter Topic 需要配置正确的 IAM 权限和最大投递次数。最大投递次数并不等于绝对精确的重试次数，实际投递次数可能存在偏差。

面试问题

1. What causes a growing Pub/Sub backlog?
    
2. How would you scale Pub/Sub subscribers?
    
3. What is backpressure in stream processing?
    
4. What is a Dead Letter Topic?
    
5. How would you handle poison messages?
    
6. What metrics would you monitor in a Pub/Sub streaming pipeline?
    

## 9. Google Cloud Managed Kafka：扩展知识

### 9.1 Kafka Partition 与 Consumer Group

Kafka 使用 Partition 实现并行处理。

例如：

```
Topic: orders

Partition 0: M1 → M2 → M3
Partition 1: M4 → M5 → M6
Partition 2: M7 → M8 → M9
```

每个 Partition 是一个有序、不可变的消息记录序列。

Consumer Group 中的 Consumer 可以分配不同 Partition：

```
              orders Topic
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      Part 0      Part 1      Part 2
        │           │           │
        ↓           ↓           ↓
    Consumer A  Consumer B  Consumer C
```

同一个 Consumer Group 中：

- 一个 Partition 在同一时刻只分配给一个 Consumer。
    
- 多个 Partition 可以由多个 Consumer 并行处理。
    
- Consumer 数量超过 Partition 数量时，多出来的 Consumer 会处于空闲状态。
    

因此，Kafka 的 Partition 数量会影响 Consumer Group 的最大并行度。

### 9.2 Offset

Kafka 使用 Offset 记录 Consumer 的消费位置。

例如：

```
Partition 0

Offset: 0 → 1 → 2 → 3 → 4 → 5
                    ↑
             Committed Offset
```

Consumer 可以通过提交 Offset 记录已经处理到的位置。

如果 Consumer 崩溃并重新启动，可以从已提交的 Offset 继续消费。

与 Pub/Sub 不同：

- Pub/Sub 使用 Ack 确认消息。
    
- Kafka 使用 Offset 记录消费进度。
    

### 9.3 Managed Kafka 的扩展

Google Cloud Managed Service for Apache Kafka 需要用户配置集群资源，例如 vCPU 和内存。

服务会根据配置的资源自动部署 Broker。

如果需要扩容，可以增加集群资源。扩容可能会增加 Broker，并触发 Partition 在 Broker 之间重新平衡。

因此：

- Managed Kafka 并不等于 Serverless Pub/Sub。
    
- 用户仍然需要规划集群容量、Topic、Partition 和 Consumer。
    
- Kafka Topic 的 Message Retention 由用户配置。
    
- Managed Kafka 提供 Tiered Storage，可以将数据存储在本地高速存储和远端存储中。
    

面试问题

1. How does Kafka achieve parallel message processing?
    
2. What is the relationship between Partitions and Consumer Groups?
    
3. What happens if the number of consumers exceeds the number of partitions?
    
4. What is a Kafka Offset?
    
5. How does Kafka recover from consumer failures?
    
6. How does Managed Kafka scale when message throughput increases?
    

## 10. Pub/Sub 与 Kafka 高频对比总结

|面试考点|Pub/Sub|Kafka|
|---|---|---|
|Message routing|Topic + Subscription|Topic + Partition|
|Consumption progress|Ack|Offset|
|Default delivery|At-least-once|通常基于 Consumer Offset 和提交策略实现至少一次或其他语义|
|Ordering|Ordering Key|Partition|
|Parallelism|Subscriber 实例和消息 Key|Partition 数量和 Consumer 数量|
|Scaling|Serverless 自动扩展|集群资源和 Partition 规划|
|Retention|Subscription 默认 7 天，最长 31 天；Topic Retention 最长 31 天|Topic Retention 可配置|
|Replay|Seek|Offset 重置|
|Dead Letter|Dead Letter Topic|通常通过独立 DLQ Topic 实现|
|GCP Integration|原生集成|通过 Kafka 客户端和连接器集成|
|Infrastructure|无需管理 Broker|托管 Broker 集群|

### 关键概念速记

|关键词|必须理解的内容|
|---|---|
|At-least-once|可能重复投递，需要幂等处理|
|Exactly-once|Pub/Sub 仅 Pull 支持，Ack ID 需要有效|
|Ack Deadline|超时未确认可能重新投递|
|Ordering Key|相同 Key 内有序|
|Kafka Partition|单个 Partition 内有序|
|Consumer Group|多个 Consumer 分担 Partition|
|Offset|Kafka 消费进度|
|Subscription|Pub/Sub 消费状态和交付配置|
|Message Retention|消息保留期限|
|Seek|重新消费历史消息|
|Attributes|消息 Metadata，可用于过滤|
|Backlog|尚未处理的消息积压|
|DLQ|隔离无法正常处理的消息|