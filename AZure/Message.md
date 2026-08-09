Storage Queue
max size 64 kb

Event Grid(topics)
No queue/ no order

Service bus(256k advanced message)
FIFO, Dead letter queue, transcations, dup detection
price: tier, number of operatoions

Event Hub (bigdata with low latency, max 1MB)
Kafka cloud version
partition, consumer group,
Ingress: throughput unit (TU) - 1MB/ sec or 1000 events/sec
Egress: throughput unit (TU) - 2MB/ sec or 4096 events/sec
