# Apache Kafka Interview Preparation Guide

> **Audience:** Senior Software Engineer / Senior Java Backend Engineer  
> **Focus:** Kafka fundamentals, Java/Spring Kafka, delivery guarantees, performance, production troubleshooting, internals, and distributed-system design.  
> **Total:** 100 interview questions and answers.

## Table of contents

- [Phase 1 — Core concepts (Q1–Q20)](#phase-1--core-concepts-q1q20)
  - [Section A — Kafka fundamentals and internals (Q1–Q10)](#section-a--kafka-fundamentals-and-internals-q1q10)
  - [Section B — Producers and consumers (Q11–Q20)](#section-b--producers-and-consumers-q11q20)
- [Phase 2 — Delivery guarantees and Java (Q21–Q40)](#phase-2--delivery-guarantees-and-java-q21q40)
  - [Section C — Delivery guarantees and consistency (Q21–Q30)](#section-c--delivery-guarantees-and-consistency-q21q30)
  - [Section D — Java and Spring Kafka (Q31–Q40)](#section-d--java-and-spring-kafka-q31q40)
- [Phase 3 — Performance and production incidents (Q41–Q60)](#phase-3--performance-and-production-incidents-q41q60)
  - [Section E — Performance and capacity planning (Q41–Q50)](#section-e--performance-and-capacity-planning-q41q50)
  - [Section F — Production troubleshooting and reliability (Q51–Q60)](#section-f--production-troubleshooting-and-reliability-q51q60)
- [Phase 4 — System design (Q61–Q70)](#phase-4--system-design-q61q70)
  - [Section G — Distributed systems and architecture (Q61–Q70)](#section-g--distributed-systems-and-architecture-q61q70)
- [Phase 5 — Internals and advanced topics (Q71–Q90)](#phase-5--internals-and-advanced-topics-q71q90)
  - [Section H — Kafka internals (Q71–Q75)](#section-h--kafka-internals-q71q75)
  - [Section I — Streams, Connect, CDC and schemas (Q76–Q85)](#section-i--streams-connect-cdc-and-schemas-q76q85)
  - [Section J — Security and operational awareness (Q86–Q90)](#section-j--security-and-operational-awareness-q86q90)
- [Phase 6 — Mock interviews (Q91–Q100)](#phase-6--mock-interviews-q91q100)
- [Senior-level revision checklist](#senior-level-revision-checklist)
- [References](#references)

## How to use this guide

For each question, practice explaining the concept or failure mechanism, the proposed design, trade-offs, and how you would validate it in production. Examples are illustrative; tailor configuration and APIs to the Kafka and Spring Kafka versions in use.

## Phase 1 — Core concepts (Q1–Q20)

### Section A — Kafka fundamentals and internals (Q1–Q10)

### Q1. What is Apache Kafka, and what problems does it solve?

Apache Kafka is a distributed event-streaming platform built around durable, partitioned, replicated logs. Producers append records to topics, and consumer groups read those records independently at their own pace; records can be retained and replayed rather than being deleted simply because one consumer processed them. Partitioning enables horizontal throughput and parallelism, while replication provides fault tolerance. Kafka is commonly used for event-driven microservices, activity streams, log/data pipelines, CDC, and stream processing. It is not a database or a guarantee of exactly-once business effects by itself: the design must still address schemas, ordering, retries, consumer idempotency, retention, and downstream consistency.


### Q2. How is Kafka different from a traditional message queue?

Many traditional message queues emphasize competing consumers and work distribution; depending on the product, a message may be removed after acknowledgment. Kafka is a retained, partitioned log: each consumer group tracks its own offsets, and multiple groups can independently read the same event stream or replay it within retention. Kafka partitions provide horizontal parallelism and replicas provide fault tolerance. This makes Kafka a strong fit for event streams, replay, fan-out, and high-throughput pipelines, while a traditional queue may be simpler for per-message routing, request/reply, or task dispatch. The comparison is not absolute—modern brokers overlap in features—so choose based on ordering, retention, routing, latency, operational model, and delivery guarantees.


### Q3. What is a Kafka topic?

A topic is a logical name for a stream of records, such as `orders-created` or `payments-updated`. A topic is divided into partitions; each partition is an ordered log with its own offsets and replicas. Topic configuration controls settings such as partition count, replication factor, retention, and cleanup policy (`delete` or `compact`). Consumer groups maintain independent offsets, so several services can consume the same topic for different purposes without competing with one another. Topic design should reflect event meaning, ownership, expected throughput, retention, access control, and ordering requirements.


### Q4. What is a partition, and why does Kafka use partitions?

A partition is an append-only ordered log, identified within a topic, whose records have monotonically increasing offsets. Partitions can be placed on different brokers, allowing Kafka to spread storage and throughput across a cluster and let consumer groups process different partitions in parallel. A partition has one leader for normal reads and writes and may have follower replicas. Ordering is guaranteed within a partition, not across a topic's partitions. Partition count is therefore both a capacity decision and a semantic decision: it affects parallelism, key-to-partition mapping, and the ability to preserve per-entity order.


### Q5. What is a Kafka record?

A Kafka record consists primarily of a key, value, timestamp, and headers; the broker/client also associates it with a topic, partition, and offset. The key is often used for partition selection and grouping related events, while the value carries the event payload. Headers are useful for trace IDs, correlation IDs, schema metadata, and other cross-cutting context, but consumers should not treat them as a secure storage location. Decide whether timestamps represent event creation time or broker append time, and document that contract. A good event payload has a stable schema, explicit event identity, clear semantics, and only the data consumers actually need.


### Q6. What is an offset?

An offset identifies a record's position within a particular partition; it is not globally unique across a topic and is not a timestamp. The pair `(topic, partition, offset)` identifies a record location in the log. Consumer-group committed offsets represent the position from which the group should resume—conceptually the next record to consume, although APIs and monitoring tools may describe lag using adjacent offset positions. Offsets can have gaps from the consumer's perspective due to compaction or transactional visibility, so offset difference is not always the exact count of visible records. Committing an offset records progress; it does not acknowledge a business transaction in an external database.


### Q7. What is a Kafka broker?

A broker is one Kafka server process in the cluster. It stores partition replica logs and indexes, handles produce/fetch requests, participates in replication, and reports state to the cluster's controller/quorum. A broker may lead some partitions and host follower replicas for others, so load balancing is about partition placement and request traffic rather than assigning one broker per topic. Brokers advertise addresses that clients must be able to reach; a bootstrap server is only the initial discovery endpoint. Broker health checks should include request latency, disk and network saturation, replica status, and controller/metadata connectivity.


### Q8. What does a Kafka cluster provide?

A Kafka cluster is a set of brokers operating under shared metadata and coordination. Topics are divided into partitions, and replica placement distributes those partitions across brokers for capacity and fault tolerance. The controller/quorum manages metadata and leadership changes, while clients discover brokers and leaders through metadata. A cluster can appear partially healthy while particular partitions have no leader or insufficient in-sync replicas, so assess partition-level health rather than just broker process uptime. Production capacity planning must consider CPU, disk, network, replication overhead, partition count, retention, and failure headroom.


### Q9. What is the controller in Kafka?

The controller is responsible for cluster metadata and state transitions such as broker registration and partition leader changes; it does not process every producer or consumer request itself. In KRaft mode, a controller quorum replicates a metadata log and elects an active controller, removing ZooKeeper as Kafka's external metadata system. Broker nodes host partition replicas and serve client traffic; deployments may use combined broker/controller roles or dedicated controllers depending on cluster size and operational design. Controller quorum availability is important for metadata changes and leadership management, so monitor controller health and follow the version-specific deployment guidance.


### Q10. How does KRaft differ from ZooKeeper-based Kafka?

KRaft stores cluster metadata in Kafka's own replicated metadata log and uses a controller quorum for coordination; ZooKeeper-based clusters depended on a separate ZooKeeper ensemble. KRaft simplifies the architecture and lets Kafka manage metadata using its own consensus and log mechanisms. It does not mean every Kafka feature or migration procedure is identical across releases: supported upgrade paths, metadata versions, controller/broker roles, and operational commands depend on the Kafka version. In interviews, explain the architectural difference, then emphasize controlled upgrades and compatibility documentation rather than suggesting that migration is a simple configuration toggle.


### Section B — Producers and consumers (Q11–Q20)

### Q11. What happens when a producer sends a record with a key?

The producer's partitioner chooses the destination partition. With the default keyed strategy, a key is serialized and hashed to select a partition, so records with the same key normally go to the same partition while the partition count and partitioning configuration remain stable. This is useful for entities such as an order or customer because the partition provides a single ordered stream for that key. A custom partitioner, changed key serialization, or increased partition count can change the mapping, so do not assume old and new records for one key always remain in the same partition across configuration changes. If strict per-key order is a business requirement, plan partition changes as a migration and consider version/sequence checks in consumers.


### Q12. What happens when a record has no key?

Without a key, the partitioner chooses a partition using the behavior of that Kafka client version and configuration; modern clients commonly use a sticky strategy that sends a batch of records to one partition for a time to improve batching efficiency. This means records for the same business entity may be spread across partitions and have no shared ordering guarantee. If entity ordering matters, use a stable key such as `orderId` or `customerId`; if distribution matters more than ordering, a null key may be acceptable. Watch for key skew: a single popular key can create a hot partition even when other partitions are underused.


### Q13. Does Kafka guarantee ordering?

Kafka preserves the order of records within a partition, and the offset is the authoritative sequence for that partition. It does not provide a total order across partitions; a consumer reading several partitions observes separate ordered streams. To preserve per-entity order, use a stable key so related records share a partition, avoid concurrent producer logic that sends business events out of sequence, and do not process records for the same key concurrently without sequencing. Producer idempotence helps prevent retry-related reordering under supported settings, but it cannot correct an application that submits events in the wrong order. Increasing a topic's partition count can remap keys, so it needs special care when ordering is critical.


### Q14. What is a consumer group?

A consumer group is identified by `group.id`; its members share the work of consuming subscribed topics. For each partition, at most one consumer in a group owns it at a time, while different groups have independent offsets and can each read the full topic. The group coordinator manages membership and committed offsets, and a partition assignor determines how partitions are distributed. When a member joins, leaves, fails, or a subscription changes, Kafka may rebalance assignments; during that transition processing can pause and in-flight work may be redelivered. Adding consumers helps only while there are unassigned partitions and downstream capacity. A consumer group with more active consumers than available partitions cannot increase partition-level parallelism.


### Q15. What happens when a consumer commits an offset?

The consumer group stores a position indicating where it should resume. If it commits before processing completes, a failure can skip unfinished work. If it commits after processing, a crash between processing and commit can cause redelivery. Commit strategy therefore affects loss/duplication behavior.


### Q16. How does Kafka retention work?

Kafka retention is independent of whether a consumer has read a record. With `cleanup.policy=delete`, old log segments become eligible for deletion according to time- and/or size-based retention settings; deletion is segment-oriented, so it is not always exact to the millisecond or byte. With `cleanup.policy=compact`, Kafka retains the latest value for each key over time, subject to compaction and tombstone cleanup. A topic can also use `compact,delete` when both policies are needed. Choose retention based on replay/recovery requirements, legal obligations, storage budget, and maximum expected consumer outage. A consumer that falls behind beyond retained data may no longer be able to replay the missing history from that topic.


### Q17. What is log compaction?

Log compaction is a cleanup policy designed to retain the latest state for each key rather than every historical update. If a key has values A, B, and C over time, compaction may eventually remove older records and retain C, but cleanup is asynchronous and does not promise that only one record per key exists at every moment. A record with a key and null value is a tombstone; it marks deletion and is retained for a configured period before it can be cleaned up. Compaction is useful for state snapshots, changelogs, and rebuilding keyed state, but it is not an audit-history mechanism. If every event must remain available, use delete-based retention or archive the history separately.


### Q18. What is a replication factor?

Replication factor is the number of copies of each partition, including its leader. With replication factor 3, a partition normally has one leader and two followers, ideally placed across different brokers and failure domains. Replication improves tolerance of broker failure but consumes extra disk, network bandwidth, and replication I/O; it does not protect against every operator error, logical corruption, or region-wide outage. Durability depends on the replication factor together with ISR health, producer `acks`, `min.insync.replicas`, and leader-election policy. Monitor under-replicated and offline partitions so a cluster does not silently operate with reduced redundancy.


### Q19. What are partition leaders and followers?

Each partition has one leader replica responsible for handling normal producer writes and serving reads under the usual leader-based protocol. Follower replicas fetch data from the leader and maintain their own logs; eligible followers may become leader after a failure or controlled election. Kafka tracks replicas in the ISR based on replication progress. If a follower falls behind beyond configured thresholds, it may leave the ISR; when it catches up, it can rejoin. A leader election from an in-sync replica generally preserves acknowledged data better than electing a stale replica, which is why unclean leader election is a deliberate data-loss-versus-availability trade-off.


### Q20. What is the ISR, and why does it matter?

The in-sync replica set (ISR) contains the partition leader and follower replicas that are sufficiently caught up according to Kafka's replication rules. ISR membership matters because `acks=all` waits for the current ISR, not necessarily every configured replica. `min.insync.replicas` sets the minimum ISR size required for a write with `acks=all`; if the ISR falls below that threshold, the broker rejects the write instead of accepting it with weaker durability. For example, replication factor 3 with `min.insync.replicas=2` can tolerate one replica being unavailable while continuing to accept acknowledged writes, but it may stop writes if only one replica remains in sync. This is a durability/availability trade-off, and producer settings alone cannot compensate for poor replica placement or unsafe leader-election configuration.


## Phase 2 — Delivery guarantees and Java (Q21–Q40)

### Section C — Delivery guarantees and consistency (Q21–Q30)

### Q21. Explain at-most-once, at-least-once, and exactly-once processing.

**At-most-once:** commit the offset before processing (or otherwise avoid retrying completed delivery). A crash can lose the work, but the record is less likely to be processed again. **At-least-once:** process first and commit after success; a crash between the business operation and offset commit can redeliver the record, so duplicates are possible. **Exactly-once:** ensure a defined input-to-output operation is committed atomically within a supported boundary, for example Kafka input offsets and Kafka output records in a transaction. Do not claim that Kafka automatically makes a database update, HTTP call, email, or payment happen exactly once. For external effects, combine stable event IDs, idempotency keys or a deduplication table with durable state and reconciliation.


### Q22. What does producer acks control?

`acks` controls the acknowledgment level the producer requires before treating a send as successful. `acks=0` does not wait for a broker response, so the producer cannot reliably know whether the record reached Kafka. `acks=1` waits for the leader to append the record but not for followers to confirm replication; a leader failure before replication can lose an acknowledged record. `acks=all` (equivalent to `-1`) waits for all replicas currently in the ISR, subject to the configured ISR and broker behavior. For a common durability-oriented setup, combine `acks=all`, producer idempotence, replication factor 3, and `min.insync.replicas=2`, then test failure scenarios. This reduces risk but does not replace backups, monitoring, or careful leader-election settings.


### Q23. How does producer idempotence work?

Kafka producer idempotence assigns a producer identity and uses per-partition sequence numbers so the broker can recognize duplicate protocol-level retries and preserve order under the supported configuration. It is designed for retries caused by network errors or uncertain acknowledgments, not for detecting that two separate calls represent the same business event. If an application independently submits the same order twice, Kafka normally treats those as two records. Idempotence is enabled by default in modern Kafka clients unless incompatible settings override it; older client versions and customized settings should be checked explicitly. For broader guarantees, use a business event ID and idempotent consumer logic. Idempotent production also does not make external database or API effects exactly once.


### Q24. What is a Kafka transaction?

A Kafka transaction groups writes to multiple topic partitions into one atomic unit and can include the consumed offsets that correspond to the input records. A transactional producer is configured with a stable, unique `transactional.id`, initializes transactions, writes records, adds consumed offsets when appropriate, and commits or aborts. Downstream consumers that must not observe aborted transactional output should use `isolation.level=read_committed`; `read_uncommitted` can expose records from transactions that later abort. Transactions are useful for consume-transform-produce pipelines, but they add coordination and latency and are not a distributed transaction with arbitrary databases or HTTP services. The application must handle transaction aborts, fencing, timeouts, and retries correctly.


### Q25. What does exactly-once semantics mean in Kafka Streams or consume-transform-produce?

In a Kafka consume-transform-produce pipeline, exactly-once processing means output records and the offsets of the input records that produced them are committed atomically in a Kafka transaction. If the transaction aborts, the output is not visible to `read_committed` consumers and the input offsets do not advance as if the work succeeded, so the input can be processed again safely. Kafka Streams builds on this model and can include its state-store changelog records in the same processing guarantee. The guarantee requires the appropriate client/framework configuration and transactional visibility; it is not a claim that code executes only once. A database write, HTTP request, email, or payment outside Kafka is not included automatically and still needs idempotency or another consistency pattern.


### Q26. Why can consumers receive duplicates even when producers are idempotent?

Producer idempotence protects only a defined class of duplicate sends caused by protocol-level retries for that producer identity. A consumer can still process the same record again if it crashes after committing a database write but before committing its offset. Two service instances can also independently publish the same logical event, an outbox relay can publish twice after an uncertain acknowledgment, and an operator can replay a range. Carry a stable event ID and trace it through producer, topic, consumer, and external side effect. Deduplicate at the business boundary—ideally in the same database transaction as the mutation—and distinguish harmless redelivery from genuinely duplicated source events.


### Q27. Auto commit or manual offset commit: how do you choose?

With `enable.auto.commit=true`, the client periodically commits offsets for records returned by `poll()`, which can be unsafe if application processing has not completed when the commit occurs. With manual commits, the application commits only after the work it considers complete; this usually supports at-least-once processing but can redeliver records if the process crashes after the side effect and before commit. In Spring Kafka, container acknowledgment modes (`RECORD`, `BATCH`, `MANUAL`, `MANUAL_IMMEDIATE`, and others depending on version) determine when the container commits offsets. The correct choice depends on processing granularity, transaction usage, and failure handling. Always pair offset strategy with idempotent business effects and test crashes, listener exceptions, and partition revocation.


### Q28. How do you maintain consistency between a database and Kafka?

The classic dual-write problem occurs when a service commits a database transaction and then publishes to Kafka: a crash between those operations leaves database state and the event stream inconsistent. Reversing the order has the opposite failure mode—an event can be published for a database transaction that later rolls back. The transactional outbox solves this by writing both the business change and an outbox row in the same local database transaction. A polling relay or CDC connector such as Debezium later publishes outbox rows to Kafka. The relay can crash after publishing but before recording progress, so duplicates remain possible; consumers should use event IDs and idempotent processing. Monitor outbox age/backlog and define cleanup, retry, ordering, and schema policies.


### Q29. What is the idempotent consumer pattern?

The consumer uses a stable event ID or operation key to ensure repeated delivery does not repeat the business effect. A common approach inserts the ID into a table with a unique constraint and performs the business update in the same database transaction. If the ID already exists, the duplicate is treated as already handled.


### Q30. How should retries and dead-letter topics be designed?

Retry transient failures with bounded attempts and backoff. Classify permanent errors—such as irreparably invalid schema or business validation—as non-retryable and route them to a DLT/quarantine path. Preserve source metadata, alert on DLT growth, and establish safe replay procedures. Consider ordering: retry topics can let later events overtake an earlier failed event.


### Section D — Java and Spring Kafka (Q31–Q40)

### Q31. How do you configure a Java Kafka producer?

Configure a `Properties` object with `bootstrap.servers`, key/value serializers, and the reliability/performance settings required by the workload, then create and reuse a `KafkaProducer`. Common reliability choices include `acks=all`, idempotence enabled, appropriate delivery/request timeouts, and a replication/ISR policy on the brokers. Tune `batch.size`, `linger.ms`, compression, and buffer capacity only after measuring throughput and latency. Call `send()` asynchronously and handle callback failures; blocking on every record can reduce throughput. Close the producer during shutdown so buffered records can be delivered within a controlled timeout. Keep credentials outside source control and use the security protocol configured by the cluster.


### Q32. Is KafkaProducer thread-safe? Is KafkaConsumer thread-safe?

`KafkaProducer` is thread-safe and is normally created once per application component and reused; creating one per record wastes connections, buffers, and metadata work. `KafkaConsumer` is not thread-safe, so a single owner thread should perform consumer operations such as `poll`, `commit`, `seek`, `pause`, `resume`, and assignment handling. If records are dispatched to worker threads, the application must track which records have completed and commit only the highest safe contiguous offset per partition. Committing the maximum completed offset while an earlier record is still processing can skip that unfinished record after a crash. For per-key ordering, serialize work for each key or partition, and pause partitions when bounded worker capacity is exhausted.


### Q33. How do you handle the result of producer.send()?

`send(record)` normally queues the record for asynchronous transmission and returns a `Future<RecordMetadata>`; the callback reports success metadata (topic, partition, offset) or an exception. Handle callback failures explicitly because a successful method return only means the record was accepted into the producer's local path, not necessarily persisted by Kafka. Avoid calling `.get()` for every send on a hot path unless the application needs synchronous confirmation; it serializes work and harms throughput. If using callbacks, keep them lightweight and avoid throwing away errors. Apply bounded application-level retry only after understanding the failure, because blindly resending may create duplicate business events; use producer idempotence and stable event IDs as appropriate.


### Q34. What is the purpose of poll() in a Kafka consumer?

`poll()` is not merely a fetch call: it returns available records and drives group coordination, heartbeats/consumer liveness behavior, and rebalance processing according to the client protocol and configuration. If application processing blocks the poll thread longer than `max.poll.interval.ms`, the consumer can be considered unhealthy and lose its assignment, causing a rebalance and possible redelivery. Keep processing time bounded, tune `max.poll.records` to a realistic workload, or move work to a carefully coordinated worker pool. If using workers, preserve per-partition commit ordering and stop or drain work safely during revocation. Increasing timeout values blindly can hide slow processing and delay recovery.


### Q35. When would you use manual acknowledgments in Spring Kafka?

Manual acknowledgment is useful when the listener needs to decide explicitly when a record or batch is considered successfully processed—for example, after a database transaction completes. In Spring Kafka, the configured container `AckMode` controls how the acknowledgment is handled; `MANUAL` and `MANUAL_IMMEDIATE` have different commit timing semantics, so verify the version-specific behavior. Never acknowledge before the business effect is durable, or a crash may cause the record to be skipped. Acknowledge after success and design for redelivery if a crash happens before the offset is committed. Also account for listener exceptions, error-handler recovery, transactions, and partition revocation; manual acknowledgment is not itself an exactly-once mechanism.


### Q36. How does @KafkaListener concurrency work?

The `concurrency` setting on a Spring Kafka listener container creates multiple listener consumer instances/threads within the application. Those consumers join the configured group and receive partition assignments; for one topic with P partitions, useful active consumer parallelism for that topic is at most P, although assignments across multiple topics can vary. More concurrency can improve throughput only if there are enough partitions and downstream resources can handle the load. Excessive concurrency may increase database connection pressure, thread contention, rebalances, and request bursts. Keep in mind that ordering is per partition, so records from the same partition are not independently safe to process concurrently if their business effects depend on sequence.


### Q37. How do you implement retry and DLT handling in Spring Kafka?

In Spring Kafka, container error handling is commonly configured with `DefaultErrorHandler` and, when appropriate, `DeadLetterPublishingRecoverer`. Configure a bounded backoff and retry count, classify exceptions so permanent failures (for example, invalid payloads) are not retried pointlessly, and decide what happens after retries are exhausted: publish to a DLT, quarantine, or stop processing. A DLT should preserve original topic/partition/offset and exception metadata, have access controls and alerting, and support controlled replay. Non-blocking retry topics can free the original partition to process later records, but this can break ordering and are not supported with every listener/transaction arrangement. Validate behavior against the exact Spring Kafka version, listener mode (record/batch), and transaction configuration.


### Q38. How do Kafka transactions work with Spring Kafka?

Spring Kafka supports transactions by configuring a `DefaultKafkaProducerFactory` with a `transactionIdPrefix` and using transactional templates and/or listener containers. In a consume-transform-produce flow, output records and the consumed offsets can be committed atomically to Kafka; if processing fails, the transaction can be aborted and the input records re-delivered. Downstream consumers that must hide aborted output should use `read_committed`. Spring's transaction synchronization can coordinate a Kafka transaction with a database transaction, but it is not a single atomic commit across both systems: one resource can commit while the other commit fails. For database-to-Kafka consistency, the transactional outbox is often the clearer design. Keep transaction IDs unique across concurrently running application instances and account for transaction timeouts and fencing.


### Q39. How do you test a Spring Kafka consumer?

Keep most business-rule tests as ordinary unit tests with no broker. Add integration tests for serializers/deserializers, topic configuration assumptions, listener container wiring, partition/key behavior, retries and DLT publication, offset commits, and transaction visibility. Use `spring-kafka-test`, Testcontainers, or another environment appropriate to the project's version and CI constraints; an embedded test broker is not a substitute for every production-broker behavior. Assert observable outcomes with bounded polling or latches instead of fixed sleeps, and clean up topics and consumer groups between tests. Include failure-path tests, not just the happy path.


### Q40. How should a Kafka consumer shut down gracefully?

Graceful shutdown should stop new work, finish or safely cancel in-flight work within a bounded deadline, commit only offsets for completed records, and close the consumer or listener container. In Spring applications, stop the listener container through its lifecycle mechanism rather than abruptly killing consumer threads. If workers process records asynchronously, wait for completion and commit only the highest contiguous completed offset per partition; do not commit past unfinished tasks. If the shutdown deadline expires, leave uncompleted work uncommitted so Kafka can redeliver it. Test termination during normal processing, a downstream outage, and a rebalance, and make handlers idempotent so a retry is safe.


## Phase 3 — Performance and production incidents (Q41–Q60)

### Section E — Performance and capacity planning (Q41–Q50)

### Q41. What determines Kafka producer throughput?

Producer throughput is determined by both client behavior and cluster capacity: records per batch, `batch.size`, `linger.ms`, compression, record size, `buffer.memory`, request limits, network bandwidth, broker disk/CPU, replication factor, acknowledgment level, and how evenly keys distribute traffic. Batching and compression often improve throughput by reducing request overhead and bytes transferred, but may increase latency or CPU use. More producer threads do not automatically help if a single partition, broker, or network link is the bottleneck. Benchmark with realistic payload sizes, key skew, replication, security, and consumer load; measure p95/p99 latency and error rate alongside throughput. Change one or two variables at a time and retain headroom for bursts and failures.


### Q42. What do batch.size and linger.ms do?

`batch.size` is the target maximum size of a record batch per partition; it is not a global batch size for the whole producer. `linger.ms` lets the producer wait briefly for more records to arrive so it can send fuller batches rather than immediately issuing small requests. Larger batches and a modest linger can improve throughput and compression efficiency, but may increase send latency and memory pressure. If traffic is low, batches may remain small even with a large `batch.size`; if traffic is high, batches may fill quickly. Tune both with realistic record sizes and throughput/latency measurements, and keep `delivery.timeout.ms` and related timeouts consistent.


### Q43. When should you use compression?

Compression is valuable when network bandwidth, broker disk I/O, or storage cost is a bottleneck. Kafka compresses record batches, so batching usually improves compression efficiency; compression can reduce bytes replicated between brokers as well as bytes sent by producers. LZ4 and Snappy often favor speed, gzip may achieve a higher ratio at greater CPU cost, and Zstandard offers tunable trade-offs where supported. The best choice depends on payload shape—already-compressed images or encrypted blobs may gain little—and on CPU headroom, latency targets, and consumer decompression cost. Benchmark realistic payloads and report throughput, p95/p99 latency, CPU, and compression ratio rather than choosing by algorithm reputation alone.


### Q44. How do you calculate consumer lag?

Consumer lag is usually estimated per partition from the difference between the log end offset and the consumer group's committed offset; monitoring tools may use slightly different definitions and may also report records behind the high watermark. Sum lag across partitions only as a rough workload indicator, because offset distance is not the same as bytes or processing time when record sizes and rates vary. Track lag trend, oldest unprocessed event age, input rate, processing rate, and per-partition skew. Rising lag means incoming work exceeds sustainable processing for long enough; stable lag may be acceptable if the age stays within the service objective. Diagnose whether the bottleneck is partition count, hot keys, consumer CPU/GC, downstream latency, rebalances, or broker fetch performance before scaling.


### Q45. How do you detect and fix hot partitions?

Compare input rate, consumer lag, bytes, request latency, and processing time by partition rather than looking only at cluster-wide averages. A hot partition commonly comes from skewed keys—for example, one tenant or account producing far more events than others—or from uneven record sizes and processing cost. First optimize the work for that key and inspect whether the key choice is correct. You may isolate high-volume entities into a separate topic or use a composite/sharded key, but sharding one entity across partitions can break strict ordering and requires sequence/version handling or a merge step. Adding partitions alone may not help an existing hot key because all records with that key can still map to one partition.


### Q46. How do you choose partition count?

Choose partitions from measured throughput and the parallelism the workload needs, not from a universal rule. Estimate producer throughput per partition, consumer processing capacity per partition, key distribution, peak traffic, future growth, and the maximum recovery time you can tolerate. Within one consumer group, a partition is assigned to at most one consumer at a time, so partitions cap parallel consumer ownership; extra consumers beyond the partition count may sit idle. More partitions also increase metadata, replication traffic, open files, recovery/reassignment work, and can affect end-to-end latency. Increasing partition count can change keyed partition mapping for new records and may break per-key ordering across the transition; reducing partition count generally requires creating a new topic and migrating data.


### Q47. How do you estimate Kafka storage?

Approximate retained primary log storage as daily data volume multiplied by retention days, then adjust for compression and add replication copies, indexes, segment overhead, and free-space headroom. Also account for retention spikes, compaction, and data-growth trends. Validate against broker metrics and actual segment usage.


### Q48. Which consumer settings commonly affect throughput?

`max.poll.records` caps how many records the consumer returns in one `poll()` call; it does not change the underlying fetch size by itself. `max.partition.fetch.bytes` and `fetch.max.bytes` constrain fetched data, while `fetch.min.bytes` and `fetch.max.wait.ms` trade additional latency for fuller network responses. `max.poll.interval.ms` limits the time between calls to `poll()` and is critical when processing happens on the consumer thread. Tune these settings together based on record size, processing duration, available memory, and desired latency; oversized polls can cause long processing gaps, while tiny fetches can increase request overhead. Also monitor `session.timeout.ms`/heartbeat behavior and rebalance metrics rather than treating one timeout as a throughput knob.


### Q49. How do you apply backpressure in a Kafka consumer?

Backpressure means slowing intake when downstream processing cannot keep up. Use bounded worker queues and semaphores, cap concurrent database/API calls, and pause assigned partitions when the queue or dependency reaches a threshold; resume when capacity returns. Do not keep polling and buffering unbounded records, because memory exhaustion and increasingly stale work can follow. If processing is delegated to workers, track completion per partition and commit only a contiguous range of completed offsets. Define what happens during shutdown and partition revocation, and preserve per-key order if concurrent processing could reorder business updates. Monitor queue depth, processing time, lag, and dependency saturation so backpressure activates before an outage.


### Q50. Consumer processing is slower than database writes can sustain. What do you do?

Treat the database as the constrained dependency rather than assuming Kafka is slow. Check query latency, slow queries, lock contention, connection-pool utilization, transaction duration, CPU/IO, and the rate at which consumers issue requests. Optimize queries and indexes, batch writes where correctness permits, shorten transactions, and cap listener concurrency to the database's sustainable capacity. Apply backpressure or pause partitions during an outage instead of creating unbounded retries and in-memory queues. Scaling consumers helps only if the database has spare capacity; otherwise it increases contention and can make lag and recovery worse. Once the dependency recovers, ramp traffic gradually and monitor backlog drain rate.


### Section F — Production troubleshooting and reliability (Q51–Q60)

### Q51. How do you troubleshoot a broker unreachable error?

Start with the exact client error and determine whether failure occurs during bootstrap, metadata discovery, authentication, or a later produce/fetch request. Verify DNS, routing/firewall rules, broker `listeners` and `advertised.listeners`, TLS certificate trust/hostname matching, SASL mechanism and credentials, and load balancer behavior. A common failure is that the client can reach the bootstrap address but cannot reach the broker addresses returned in metadata. Test connectivity from the same network/container as the application, not only from an administrator's laptop. Correlate client logs with broker logs and check broker health, listener ports, and recent certificate or network changes before increasing retry timeouts.


### Q52. What do under-replicated partitions mean?

An under-replicated partition has at least one replica that is not currently in sync with the leader; the metric is an early warning that redundancy is reduced. Check whether the cause is a broker outage, overloaded disk, network saturation, long GC pauses, slow replica fetch, throttling, or a reassignment. Also inspect offline partitions and ISR shrink/expand events: an under-replicated partition with a healthy leader differs from a partition with no leader. Restore broker and replication health, verify replicas rejoin the ISR, and confirm the cluster has adequate failure tolerance before closing the incident. Avoid weakening durability settings merely to make the alert disappear.


### Q53. How do you debug repeated rebalances?

Repeated rebalances interrupt consumption and can increase lag or duplicate processing. Check consumer logs and group coordinator metrics for join/leave churn, failed heartbeats, session timeouts, `max.poll.interval.ms` violations, JVM pauses, network issues, rolling deployments, and autoscaling. Slow processing inside the poll loop is a common cause; reduce work per poll, bound downstream calls, or use a coordinated worker model. Cooperative assignment protocols can reduce how many partitions move during a rebalance, but they do not fix unstable members or long processing. Tune timeouts only after measuring real processing and failure-detection requirements, and verify the client/broker versions support the chosen protocol.


### Q54. How do you deal with a poison message?

A poison message is a record that repeatedly fails processing, often because of invalid data, an incompatible schema, or a deterministic business-rule error. First classify the exception: transient infrastructure failures may deserve bounded retries with backoff, while permanent deserialization/validation errors usually need quarantine or a DLT. Preserve the original payload, key, headers, source topic/partition/offset, exception details, and retry count so the failure can be diagnosed. Alert on DLT growth and define ownership, retention, and a controlled replay procedure. Decide explicitly whether later records may proceed: skipping a failed event can be acceptable for independent notifications but can corrupt state when later events depend on it.


### Q55. How do you safely replay DLT records?

Identify and fix the cause before replay. Preserve event IDs, record source metadata, and track replay attempts. Reprocess through a controlled pipeline with idempotent business effects and limits on replay rate. Do not blindly copy all records back to the original topic if ordering or duplicates can harm state.


### Q56. What is unclean leader election, and what is its trade-off?

Clean leader election chooses an eligible in-sync replica, helping preserve records that were replicated according to Kafka's durability rules. Unclean leader election allows an out-of-sync replica to become leader when no ISR replica is available; this can restore service sooner but the new leader may lack acknowledged records, which can be lost when the cluster converges. The decision is therefore availability versus potential data loss, not simply a performance setting. For systems where acknowledged events are business-critical, unclean election is generally treated cautiously and should be consistent with the configured durability policy. During an incident, check ISR state and election logs before changing the setting just to restore writes.


### Q57. What if a Kafka broker's disk is almost full?

Identify which topics and partitions consume space, check retention/compaction cleanup, replica placement, ingestion growth, and segment deletion progress. Expand capacity or reduce data only according to approved retention/recovery requirements. Maintain headroom for segment rolling, reassignment, and recovery.


### Q58. How do you differentiate producer timeout from failed publication?

A producer timeout is an uncertain outcome, not proof that Kafka rejected the record. The broker may have appended it and the acknowledgment may have been lost, or the request may never have succeeded. Check the exception type, delivery callback, `delivery.timeout.ms`, request/retry metrics, broker logs, and partition health. Do not blindly create a new business operation because the first send timed out; use a stable event ID and downstream deduplication where duplicates are harmful. Producer idempotence can make protocol retries safe within its scope, but it cannot identify an independent application-level resend as the same business event.


### Q59. What does NOT_LEADER_OR_FOLLOWER or LEADER_NOT_AVAILABLE suggest?

The client may have stale metadata during leader movement or the partition may temporarily lack an available leader. Metadata refresh and retry can resolve transient cases; persistent failures require inspecting partition state, broker availability, controller health, and election logs.


### Q60. Which Kafka metrics would you monitor in production?

Monitor Kafka infrastructure and application behavior together. Broker-side signals include offline and under-replicated partitions, ISR shrink/expand, request queue/latency/errors, network and disk throughput/latency, CPU/GC, controller health, and storage growth. Client-side signals include producer error/retry rates and send latency, consumer lag and oldest-record age, poll/processing duration, rebalance frequency, commit failures, and downstream dependency latency. Track DLT volume, retry rates, and event-processing success because broker metrics alone do not reveal business failure. Set alerts against SLOs and trends, distinguish warning signals from immediate data-integrity risks, and maintain dashboards that let responders isolate the affected topic, partition, group, and release.


## Phase 4 — System design (Q61–Q70)

### Section G — Distributed systems and architecture (Q61–Q70)

### Q61. Why choose event-driven architecture with Kafka?

Kafka is useful in event-driven architectures when services should publish facts without knowing every downstream consumer. The log buffers temporary rate differences, allows consumer groups to scale independently, and supports replay for recovery or rebuilding projections. This reduces synchronous coupling and can improve resilience, but introduces eventual consistency, more operational components, schema governance, duplicate delivery, ordering constraints, and harder end-to-end debugging. Kafka does not remove the need for service ownership, API/event contracts, idempotency, observability, and failure handling. Use it when asynchronous decoupling and event retention provide clear value, not as an automatic replacement for a direct synchronous call.


### Q62. How do you design event contracts between microservices?

Treat an event contract as a public API. Define the event name and meaning, producer ownership, stable event ID, entity/business key, timestamp semantics, schema format and compatibility rules, required versus optional fields, and how deletion or correction is represented. Prefer past-tense facts such as `OrderCreated` over ambiguous messages that mix a command with an event; consumers should not need access to the producer's database schema to interpret the event. Document whether order matters per entity and whether consumers may replay old versions. Add contract tests and a deprecation/migration policy so producers and consumers can be deployed independently.


### Q63. Design order creation and inventory reservation.

Create the order and an `OrderCreated` outbox row in one database transaction, then publish the outbox event to Kafka. The inventory service consumes it idempotently, reserves stock in its own transaction, and emits a result such as `InventoryReserved` or `InventoryRejected`. The order service transitions from `PENDING` to `CONFIRMED` or `REJECTED` based on that result; do not report success before the required business steps have completed. Use a stable order/event ID, unique constraints, explicit timeouts, retries, and reconciliation for ambiguous outcomes. If a later step fails, use a saga/compensating action (such as releasing a reservation or cancelling the order) rather than assuming a cross-service rollback. Define how duplicate, delayed, and out-of-order results are handled.


### Q64. How would you design payment processing with Kafka?

Use Kafka to distribute payment lifecycle events, but keep the authoritative payment state in a durable system designed for financial correctness. Give each payment attempt a stable idempotency key and use the payment provider's idempotency facility where available; persist request/response state and model transitions such as `PENDING`, `AUTHORIZED`, `CAPTURED`, `FAILED`, and `UNKNOWN` explicitly. If a request times out, the provider may still have processed it, so query/reconcile by the provider's reference instead of blindly issuing a new charge. Publish state changes reliably through an outbox or equivalent mechanism, make consumers idempotent, restrict sensitive data in events, and reconcile Kafka events against the payment ledger.


### Q65. Explain the transactional outbox pattern.

The transactional outbox addresses the dual-write problem by committing the business state change and an event row in the same database transaction. A separate publisher polls the outbox or a CDC tool reads the database log and forwards the event to Kafka. This ensures that a committed business change has a durable event to publish, without requiring a distributed transaction between the database and Kafka. The publisher can still crash after Kafka accepts a record but before it records progress, so duplicates are possible and consumers should deduplicate by stable event ID. Define event ordering (often by aggregate/entity key), retry and poison-row behavior, outbox cleanup, schema ownership, and monitoring for oldest unpublished row and backlog. CDC can reduce polling overhead but adds connector operations and recovery considerations.


### Q66. What is a saga?

A saga coordinates a multi-service business process as a sequence of local transactions, with events or commands advancing the workflow. In a choreography, services react to each other's events; in an orchestration, a coordinator explicitly manages state and next steps. If a later step fails, the workflow runs compensating business actions—for example, release a reservation or refund a payment—rather than rolling back a remote database transaction that already committed. Design explicit states, idempotent handlers, timeouts, retry policies, compensation failure handling, and reconciliation for ambiguous outcomes. Kafka can transport saga messages, but the application still owns the workflow state machine and operational recovery.


### Q67. How do you evolve schemas without breaking consumers?

Schema evolution should let producers and consumers deploy independently. Choose a format and compatibility policy that matches the rollout model: backward compatibility lets new consumers read old data, forward compatibility lets old consumers read new data, and full compatibility combines both under the registry's defined rules. Adding an optional field with a safe default is often compatible; renaming/removing a required field or changing its type can break consumers. A safe rollout commonly deploys consumers that tolerate the new field first, then updates producers, and only later removes deprecated fields after old consumers are retired. Compatibility checks do not catch every semantic change, so add contract tests and document field meaning, nullability, and units.


### Q68. How do you support replay without repeating harmful side effects?

Start by defining what replay is supposed to accomplish: rebuild a read model, recover missed processing, re-run a calculation, or intentionally repeat a business action. Rebuild projections into a fresh or versioned target when possible, rather than replaying historical events through code that sends emails, charges cards, or triggers other irreversible side effects. Use stable event IDs and idempotency keys, persist deduplication state with business changes, throttle replay to protect dependencies, and track replay batches and offsets. If business logic has changed, test the new version against representative historical data and plan how corrected outputs will be reconciled. A replay is a controlled operational procedure, not just resetting offsets.


### Q69. How would you plan multi-region Kafka disaster recovery?

Begin with explicit RPO (how much data loss is acceptable) and RTO (how long recovery may take), then choose a supported replication or backup/restore design. Document how topics, configs, ACLs, schemas, connector state, consumer offsets, and application state are recovered; replicating topic data alone may not reproduce a working service. Decide which region is authoritative, how producers switch, how duplicate or delayed events are handled, and how split-brain writes are prevented. Cross-cluster replication may not preserve all semantics such as offsets, transactions, or ordering in the same way as a single cluster. Run regular failover and failback exercises and measure actual recovery time rather than treating the runbook as proof.


### Q70. How do you build a reliable audit/event history?

Use explicit audit schemas, reliable source publication, restrictive ACLs, retention and archive policies, monitoring for gaps, and access auditing. Kafka alone does not prove legal immutability; archive to storage with suitable immutability/integrity controls when required. Avoid compaction if the full history must be retained.


## Phase 5 — Internals and advanced topics (Q71–Q90)

### Section H — Kafka internals (Q71–Q75)

### Q71. How does Kafka store records on disk?

Each partition is stored as an append-only log on disk, divided into segment files. Producers append record batches to the active segment; Kafka assigns offsets, and index files help locate offsets and timestamps without scanning the whole log. The log is written sequentially, which is efficient for disk and enables batch/network optimizations; replication copies the partition log to followers. When a segment rolls, older segments become candidates for retention deletion or compaction according to the topic cleanup policy. Actual disk use also includes indexes, metadata, and replicas. Kafka's performance depends heavily on sequential I/O, batching, page cache, and network throughput, so broker disk latency and page-cache pressure matter as much as raw disk capacity.


### Q72. What is a segment, and what are segment rolling settings?

A log segment is a file set containing a portion of a partition's log, with associated index files that help locate records. Kafka appends to the active segment and rolls to a new one when configured size or time conditions are met. Retention and cleanup operate largely at segment/record granularity, so segment settings influence how quickly data becomes eligible for cleanup and the amount of file/index overhead. Very small segments can increase file churn; very large segments can delay cleanup and make maintenance less granular. Tune only after understanding topic size, retention policy, broker I/O, and actual cleanup behavior.


### Q73. What is the high watermark?

The high watermark (HW) is the boundary of replicated log data considered committed under Kafka's replication protocol; ordinary consumers do not read beyond it. It helps prevent consumers from observing records that might be lost if a leader fails and an unreplicated tail is truncated. The HW is not the same as the log end offset (LEO), which marks the end of the replica's local log. For transactional consumers using `isolation.level=read_committed`, visibility is additionally constrained by the last stable offset (LSO), which stops before unresolved transactions. Interviewers often use these offsets to test whether you understand replication and transactional visibility separately.


### Q74. What is the difference between the log end offset and the last stable offset?

The log end offset marks the end of the current log. For transactional data, the last stable offset is the point before the first open transaction that prevents read_committed consumers from safely advancing further. A consumer can therefore be caught up to committed data while the log end offset is farther ahead.


### Q75. What happens during partition reassignment?

Partition reassignment changes which brokers host replicas, typically to rebalance storage/traffic, replace a broker, or adjust placement. Kafka must copy replica data to target brokers, which can consume network bandwidth and disk I/O and temporarily increase load. Plan the movement in controlled batches, apply appropriate throttles, check destination capacity and rack/failure-domain placement, and watch reassignment progress, under-replicated partitions, and client latency. After completion, confirm ISR health and balanced disk/network usage; a completed reassignment is not automatically proof that the cluster is optimally balanced.


### Section I — Streams, Connect, CDC and schemas (Q76–Q85)

### Q76. How does Kafka Streams differ from a plain consumer?

Kafka Streams is a client library for building stream-processing applications using Kafka topics as inputs and outputs. It provides operations such as stateless transformations, joins, aggregations, windowing, repartitioning, local state stores, changelog-based recovery, and configurable processing guarantees. A plain `KafkaConsumer` gives more direct control over polling and side effects, but the application must implement state management, partition/task coordination, recovery, and much of the processing topology itself. Choose Streams when the problem is naturally a Kafka-to-Kafka processing topology; choose a lower-level consumer or another processing engine when integration requirements or operational constraints justify it. Processing guarantees still have defined boundaries and depend on topology and configuration.


### Q77. What is the difference between KStream and KTable?

A `KStream` represents a sequence of records where each record is an event; multiple records with the same key are all meaningful. A `KTable` represents the latest value for each key, similar to a continuously updated table or key-value view, so a new record updates that key's current state. A `GlobalKTable` replicates the table's state to each application instance and can support lookup-style joins, with different scaling and update trade-offs. These distinctions affect join semantics: stream-stream joins are event/window oriented, while stream-table joins enrich events using the table's current value. The right abstraction depends on whether the data means events or current state.


### Q78. What are event-time windows and late-arriving events?

Windowing groups events by time ranges for aggregations such as counts or totals. Event time represents when the event occurred; processing time is when the application handles it, and ingestion/record timestamps may differ depending on configuration. Distributed streams can arrive out of order, so Kafka Streams window definitions include a grace period during which late records can still update a result. After the window closes, later records may be dropped from that window or handled according to the topology's semantics. Define timestamp extraction, allowed lateness, correction behavior, and downstream expectations explicitly—especially when late data can change financial or operational reports.


### Q79. How does Kafka Streams recover local state?

Stateful operations use local state stores and Kafka changelog topics to restore state after restart or task migration. Recovery time depends on state size, changelog volume, local disk, and restoration throughput. Monitor restore duration and ensure state/changelog retention matches the processing design.


### Q80. What is Kafka Connect?

Kafka Connect is a framework for integrating Kafka with external systems through source connectors (external system to Kafka) and sink connectors (Kafka to external system). Connect manages connector configuration, task execution, offsets, worker coordination, and operational status, so teams do not need to build a custom integration service for every data source. In distributed mode, worker configuration and internal config/offset/status topics must be durable and correctly replicated. Delivery guarantees depend on the connector, source offset semantics, sink transaction/idempotency support, and configuration; do not assume every connector provides exactly-once delivery. Monitor task failures, offset progress, connector lag, and target-system errors.


### Q81. What is CDC, and how does Debezium fit?

Change data capture (CDC) detects inserts, updates, and deletes from a database, often by reading its transaction log rather than repeatedly querying whole tables. Debezium provides source connectors for supported databases and emits change events to Kafka, including metadata useful for identifying the source and operation. Initial snapshots, log positions, schema changes, deletes/tombstones, transaction boundaries, and connector restarts all affect the event stream and must be understood. CDC is useful for data integration and for publishing an outbox reliably, but it does not remove the need for consumer idempotency or a schema-evolution plan. Validate connector-specific guarantees and monitor source lag and snapshot/recovery behavior.


### Q82. How does Kafka Connect manage source offsets?

Kafka Connect stores source connector progress in its internal offset topic so a source task can resume from a known position after restart; sink connectors also use consumer offsets for the Kafka records they have delivered. The meaning of an offset is connector-specific—for example, a database log position is different from a file byte position—and correctness depends on how the connector records progress relative to the external operation. Configure the internal topics with appropriate replication and retention, protect them from accidental deletion, and understand whether a connector can replay or duplicate data after failure. Test restart and recovery behavior rather than assuming offsets alone guarantee end-to-end exactly-once delivery.


### Q83. What is a schema registry used for?

A schema registry centralizes versioned schema definitions and compatibility checks for formats such as Avro, Protobuf, or JSON Schema. A serializer typically registers or looks up a schema and writes a schema identifier with the encoded payload according to that registry's wire format; the consumer uses the identifier to retrieve the matching schema. Compatibility modes (for example, backward, forward, or full, with transitive variants in some registries) define which schema changes are allowed relative to previous versions. Registry compatibility is a guardrail, not a substitute for clear event semantics or consumer contract tests. Schema IDs and wire formats are implementation-specific, so producers and consumers must use compatible serializers/deserializers.


### Q84. What is a tombstone record?

In a compacted topic, a tombstone is a record with a non-null key and a null value. It signals that the keyed state has been deleted, allowing consumers rebuilding a state store to remove that key. Kafka retains the tombstone for a configured tombstone retention period so downstream consumers have a chance to observe it; later compaction can remove both the tombstone and older values for that key. A null payload is different from a serialized empty object, string `null`, or an event whose value contains a deletion flag. Consumers and serializers must preserve that distinction, and tombstone cleanup should be aligned with the longest expected consumer outage/rebuild window.


### Q85. What does exactly-once mean in Kafka Streams?

Kafka Streams supports processing guarantees that coordinate consumed offsets, output-topic records, and state-store changelog updates. With exactly-once processing enabled, these Kafka-visible changes are committed transactionally so a restart does not expose output/state that is inconsistent with the committed input position. This does not mean the application code runs once; tasks may be retried after failures, and external side effects are outside the transaction. Choose the processing guarantee intentionally, understand the throughput/latency and transaction overhead, and use `read_committed` when downstream consumers must ignore aborted records. For external databases or APIs, use idempotency, an outbox, or a purpose-built sink connector with documented guarantees.


### Section J — Security and operational awareness (Q86–Q90)

### Q86. How do you secure Kafka clients with TLS and SASL?

TLS encrypts traffic in transit and lets clients validate broker certificates; mutual TLS can also authenticate clients through certificates. SASL authenticates a client using a supported mechanism such as SCRAM or OAUTHBEARER, while authorization is a separate decision typically enforced with ACLs or an authorizer. Configure the correct listener security protocol, truststore/CA chain, credentials, and mechanism on both client and broker. Use least-privilege identities, keep secrets out of code and logs, rotate credentials and certificates, and test expiration/renewal before production. Encryption and authentication do not replace topic/group/transaction authorization or payload-level privacy controls.


### Q87. What are Kafka ACLs?

Kafka ACLs define which authenticated principals may perform operations on resources such as topics, consumer groups, transactional IDs, and cluster-level resources. Authentication (TLS/SASL) establishes identity; ACLs or the configured authorizer determine what that identity can do. Apply least privilege: a producer usually needs write access to specific topics, a consumer needs read access to topics and its group, and a transactional producer may need permission for its transactional ID. Avoid broad wildcard access unless justified, separate service identities where practical, audit changes, and test authorization during deployment. Exact ACL behavior depends on the authorizer and Kafka configuration.


### Q88. How should Kafka credentials and certificates be handled?

Keep secrets out of source control and ordinary application logs. Use a managed secret store, restrict who can read credentials, rotate them safely, and monitor expiry. Use different credentials for environments and services with least privilege.


### Q89. What privacy risks arise from Kafka event payloads?

Events may replicate sensitive data into many topics, consumer groups, logs, DLTs, and archives. Minimize personal data, classify fields, restrict access, define retention/deletion obligations, encrypt where appropriate, and ensure dead-letter/replay tooling does not expose sensitive payloads broadly.


### Q90. What should a Kafka disaster-recovery plan include?

Define RPO/RTO, backups or replication strategy, cluster provisioning, topic configs, schemas, ACLs, offsets, connector state, and consumer recovery procedures. Test recovery, replay, and failback under realistic conditions. Ensure teams know which region or cluster is authoritative during failover.


## Phase 6 — Mock interviews (Q91–Q100)

### Q91. The record is likely to be delivered again because Kafka still has the previous committed offset, even though the database mutation already succeeded. This is the normal at-least-once failure window: the database and Kafka offset are separate transactional resources. Make the handler idempotent by storing a stable event ID (with a unique constraint) in the same database transaction as the business update; the second delivery then becomes a harmless no-op. Commit the Kafka offset only after the database transaction succeeds, and monitor repeated failures. A Kafka transaction alone cannot atomically commit an arbitrary external database write and the consumer offset.

On restart or reassignment, Kafka may deliver the record again because the committed position did not advance. Use a stable event ID and atomically persist a deduplication marker with the business mutation. Commit only after the DB transaction succeeds.


### Q92. Scenario: First establish whether lag is increasing, which consumer groups and partitions are affected, and whether user-facing SLAs or data correctness are at risk. Compare the deployment time with processing latency, error rates, rebalance frequency, CPU/GC, thread pools, database/API latency, and input rate. Check whether the release changed deserialization, batch size, retry behavior, concurrency, or offset commits. If the release is strongly implicated and rollback is safe, stabilize first; do not scale blindly if consumers are already overwhelming the database or if the partition count limits useful parallelism. After recovery, replay/reconcile any failed work and add a regression test or alert.

Compare release timestamps with lag, processing latency, errors, rebalance rate, CPU/GC, DB/API latency, and per-partition metrics. Roll back if the release is the likely cause and rollback is safe. Avoid scaling consumers until you know whether the bottleneck is partition parallelism or downstream saturation.


### Q93. Scenario: Two independent producers publish the same business event. Will idempotence deduplicate them?

No. Producer idempotence handles qualifying retries from a producer identity and sequence, not semantic duplicates from two independent producers. If both producers publish the same logical event, Kafka normally stores two records because it cannot infer that they represent one business action. Prefer one authoritative event owner where possible; otherwise include a stable business event ID and deduplicate at the consumer or persistence boundary. If two producers can race to create the event, define a source-of-truth rule or use a unique business constraint so the duplicate is rejected deterministically. Monitor duplicate rates and preserve enough provenance to identify which producer emitted each record.


### Q94. Scenario: Records for one order appear out of order. What do you check?

Check whether records use the same key and partition, whether partition count or partitioner configuration changed, whether producers publish concurrently, whether retries reorder sends, and whether consumers process records asynchronously without per-key sequencing. Use sequence numbers or state-version checks if required.


### Q95. Scenario: Events arrive late and have old timestamps. How do you process them?

First determine whether the timestamp represents business event time, producer creation time, or broker ingestion time; clocks can also be skewed. For analytics, use event-time processing with a defined grace period and a policy for updating or correcting results when late events arrive. For state-changing business workflows, inspect the entity ID and sequence/version, then decide whether the event is valid, stale, or requires reconciliation. Do not blindly reorder events by timestamp because timestamps may be inaccurate and Kafka only guarantees partition order. Measure lateness and record enough metadata to investigate the source of delayed delivery.


### Q96. Scenario: There is a 100 GB backlog. How do you estimate drain time?

Calculate the net drain rate as sustainable consumer processing throughput minus the current incoming data rate. If consumers can process 150 MB/s and producers continue adding 50 MB/s, the backlog shrinks at about 100 MB/s; a 100 GB backlog would take roughly 1,000 seconds (about 17 minutes) under idealized steady conditions. Use consistent units (decimal GB versus GiB), and remember that offset lag is not directly convertible to bytes without record-size information. Validate per-partition skew, downstream capacity, retries, replication and network overhead, and whether adding consumers is possible given the partition count. Scale gradually and watch downstream latency so recovery does not create a second incident.


### Q97. Scenario: A schema change breaks only one consumer. What is your approach?

Capture the exact failing record, deserialization exception, consumer version, schema ID, and registry compatibility mode. Determine whether the producer introduced a missing/renamed field, incompatible type change, changed enum value, or serializer/wire-format mismatch. Check whether the consumer is older than the schema it is reading and whether compatibility is configured in the direction the rollout requires. Restore a compatible producer/consumer combination or roll back the schema change, then add contract tests covering old and new schema versions. A registry compatibility check helps prevent common breakages but does not prove that application-level semantics remain compatible.


### Q98. Scenario: The database is unavailable but Kafka producers are healthy. Should consumers continue?

Do not keep retrying every record in a tight loop or buffer an unlimited number of records in memory. Use bounded exponential backoff with jitter, pause or reduce consumption when the database is unavailable, and make sure in-flight work and offset commits remain safe. Monitor lag, oldest-record age, retry rate, database recovery signals, and remaining Kafka retention; if the outage may outlast retention, escalate the data-loss risk early. When the database recovers, restore concurrency gradually to avoid a connection storm, and keep consumers idempotent because records may be redelivered after retries or restarts. Decide whether a DLT is appropriate: an outage affecting many valid records is often better handled by pausing than by sending everything to the DLT.


### Q99. Scenario: You need to migrate a consumer group or reset offsets. What safeguards are needed?

Before resetting offsets, capture the consumer group's current committed offsets per partition, the intended target offsets, the reason for the reset, and the expected replay/skip range. Stop or coordinate the consumers so they do not race with the administrative operation, and validate the plan in a non-production environment if possible. Confirm that retention still contains the requested offsets and that downstream side effects are idempotent; resetting backward can repeat payments/notifications or rebuild state, while resetting forward can skip required events. Use an approved maintenance window, record the change for audit, and monitor lag, errors, duplicate effects, and business reconciliation afterward. For migrations, document the source and target group IDs and avoid having both groups apply the same side effects unintentionally.


### Q100. Scenario: Interviewer asks how you lead a Kafka incident. What is your framework?

Start by stating customer/business impact, affected topics and groups, time window, and whether data may be lost, duplicated, or delayed. Assign an incident lead, a technical investigator, and a communication owner; preserve logs and metrics before making changes. Stabilize with reversible actions—such as pausing a problematic consumer or rolling back a bad release—while protecting data integrity. Form hypotheses and validate them against broker health, ISR/offline partitions, producer errors, consumer lag/rebalances, and downstream dependencies. After service recovers, reconcile business records and offsets, document root cause and contributing factors, and create measurable follow-ups: tests, alerts, capacity changes, and a runbook update.


## Senior-level revision checklist

Use this checklist to rehearse your answers aloud:

- [ ] Explain topics, partitions, offsets, leader/follower replicas, ISR, high watermark, and KRaft.
- [ ] Compare `acks`, `min.insync.replicas`, idempotent producers, transactions, and consumer isolation levels.
- [ ] Explain consumer groups, assignment/rebalances, commits, lag, backpressure, and per-partition ordering.
- [ ] Diagnose hot partitions, under-replicated partitions, broker/network/disk saturation, and producer timeouts.
- [ ] Design bounded retries, DLT/quarantine, safe replay, idempotent consumers, and graceful shutdown.
- [ ] Explain database/Kafka dual writes, transactional outbox, CDC, sagas, and reconciliation.
- [ ] Compare Kafka transactions with database transactions and state exactly-once boundaries.
- [ ] Explain producer batching/compression, partition sizing, retention, compaction, and storage estimation.
- [ ] Compare Kafka Streams, Kafka Connect, CDC, and schema registry/compatibility.
- [ ] Explain TLS/SASL, ACLs, secret rotation, privacy, and disaster recovery.
- [ ] Design order/inventory and payment flows with explicit states and compensations.
- [ ] Walk through an incident from detection and stabilization to reconciliation and prevention.

For each scenario, practice in this order: **clarify requirements → explain the failure mechanism → propose a design → discuss trade-offs → describe monitoring and recovery**.

## References

- [Apache Kafka documentation](https://kafka.apache.org/documentation/)
- [Spring for Apache Kafka reference](https://docs.spring.io/spring-kafka/reference/)
- [Kafka Streams documentation](https://kafka.apache.org/documentation/streams/)
- [Kafka Connect documentation](https://kafka.apache.org/documentation/#connect)
- [Debezium documentation](https://debezium.io/documentation/)
- [Kafka Improvement Proposals (KIPs)](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals)

> **Accuracy reminder:** Kafka and Spring Kafka behavior can vary by version and configuration. Validate production settings against the documentation for the exact version in use. Treat “exactly once” claims as scoped guarantees; they do not automatically encompass arbitrary external systems.
