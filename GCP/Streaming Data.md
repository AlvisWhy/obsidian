
Pub/Sub and  manged kafka

1 Managed vs. serverless
2 Scalability auto/ adjust parameters
3 Message retention 7 days(can be long to 31) / even forever
4 Can at-least-once delivery or exact once delivery, can maintain order with key/ exactly once, ordered only within partition
5 pub sub is better integrated with GCP services

pubsub
For **Pull or Push subscriptions**, subscriber applications must actively consume the data
For **BigQuery or Cloud Storage subscriptions**, Pub/Sub automatically exports the messages to the specified destination. This process doesn't require an active subscriber application to handle the data transfer.

Pub/Sub Ordering 的代价就是：为了保证顺序，前面的消息处理失败时，同一个 ordering key 后面的消息可能被阻塞
exact once 影响速度

在topic通过data提取部分加入metadata(attribute)可以后面subscriber filter， 不能直接filter message。

```
publisher.create_topic(topic_name)

# Could send message with data and metadata(attribute)
publisher.publish(topic_name, b'My first message', author = 'dasg')

```

**straight-through**, where a single publisher sends messages to a single subscriber; **fan-in**, where multiple publishers send messages to a single topic; and **fan-out**, where one publisher sends messages to multiple subscribers.

