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

Many traditional queues emphasize work distribution and may remove messages once acknowledged. Kafka stores an ordered log for a configured retention period, so multiple consumer groups can independently read and replay the same records. Kafka also scales through partitions and replication. Exact behavior depends on queue/broker technology and configuration, so avoid absolute comparisons.


### Q3. What is a Kafka topic?

A topic is a logical name for a stream of records, such as `orders-created` or `payments-updated`. A topic is divided into partitions; each partition is an ordered log with its own offsets and replicas. Topic configuration controls settings such as partition count, replication factor, retention, and cleanup policy (`delete` or `compact`). Consumer groups maintain independent offsets, so several services can consume the same topic for different purposes without competing with one another. Topic design should reflect event meaning, ownership, expected throughput, retention, access control, and ordering requirements.


### Q4. What is a partition, and why does Kafka use partitions?

A partition is an ordered log that enables parallel reads and writes across brokers. Each record in a partition has an offset. Partitioning is Kafka's primary scaling unit, but ordering is guaranteed only within a single partition, not across all partitions of a topic.


### Q5. What is a Kafka record?

A record can contain a key, value, timestamp, headers, and partition/offset metadata assigned by Kafka. The key can influence partition selection, while the value contains the event payload. Headers commonly carry tracing, schema, or routing metadata; avoid placing secrets or sensitive data there without protection.


### Q6. What is an offset?

An offset is a record's position within one partition. It is not a global topic sequence and should not be interpreted as a timestamp. Consumer-group offsets record the next position to read, and may be committed automatically or explicitly depending on consumer configuration.


### Q7. What is a Kafka broker?

A broker is a Kafka server that stores partition replicas, serves producer and consumer requests, and participates in cluster coordination. A topic's partitions are distributed among brokers. A broker can lead some partitions and host followers for others.


### Q8. What does a Kafka cluster provide?

A Kafka cluster is a set of brokers coordinated to host partition replicas, elect leaders, and serve clients. The cluster spreads partitions and replica copies to provide capacity and fault tolerance. Cluster health depends on both broker availability and the replica state of individual partitions.


### Q9. What is the controller in Kafka?

The controller coordinates cluster metadata and broker/partition state transitions. In modern Kafka deployments using KRaft, controller quorum members use a metadata log to agree on cluster metadata and leadership decisions. Brokers still serve normal producer and consumer traffic.


### Q10. How does KRaft differ from ZooKeeper-based Kafka?

KRaft uses Kafka's own quorum-based metadata management rather than external ZooKeeper for cluster metadata and controller coordination. It reduces the need to operate a separate coordination system. Migration procedures and support vary by Kafka version and deployment, so follow version-specific upgrade documentation.


### Section B — Producers and consumers (Q11–Q20)

### Q11. What happens when a producer sends a record with a key?

The producer's partitioner selects a partition. With typical keyed partitioning, records with the same key map to the same partition for a given partitioning configuration, preserving their partition order. Custom partitioners and partition-count changes can alter this mapping.


### Q12. What happens when a record has no key?

The partitioner chooses a partition according to the client version and configuration; modern clients may use sticky partitioning to batch efficiently rather than strictly round-robin each record. With no stable key, per-entity ordering is not naturally ensured.


### Q13. Does Kafka guarantee ordering?

Kafka preserves order within a partition, meaning a consumer reads that partition in offset order. It does not guarantee global order across multiple partitions. Producers, retries, and application-level asynchronous processing must also be configured or designed not to violate business ordering.


### Q14. What is a consumer group?

A consumer group is a set of consumers that cooperate to consume a topic's partitions. Each partition is assigned to at most one active consumer within the group at a time, while separate groups independently consume the topic. Group membership and partition assignment can change during rebalances.


### Q15. What happens when a consumer commits an offset?

The consumer group stores a position indicating where it should resume. If it commits before processing completes, a failure can skip unfinished work. If it commits after processing, a crash between processing and commit can cause redelivery. Commit strategy therefore affects loss/duplication behavior.


### Q16. How does Kafka retention work?

Retention policies delete old log segments based on time and/or size, subject to cleanup behavior. Reading a record does not normally delete it. Retention determines how far consumers can replay data and contributes directly to storage cost and recovery capability.


### Q17. What is log compaction?

Compaction preserves the latest known value per key over time, while older values for the same key may be removed. A null-valued record can act as a tombstone that eventually removes a key's retained state. Compaction is useful for changelog/state topics, not for preserving every historical event forever.


### Q18. What is a replication factor?

The replication factor is the number of replicas configured for each partition, including one leader and follower replicas. Higher replication increases resilience but also storage and network cost. Replica placement matters: copies should not all share the same failure domain.


### Q19. What are partition leaders and followers?

The leader handles normal reads and writes for a partition under the standard model. Followers replicate the leader's log and can become leaders if eligible. A follower that falls too far behind may leave the ISR until it catches up again.


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

With appropriate processing and transaction configuration, Kafka can atomically commit output records and consumed offsets, preventing a restarted processor from exposing output for an input position that was not committed consistently. This guarantee has defined boundaries and does not automatically encompass arbitrary external systems.


### Q26. Why can consumers receive duplicates even when producers are idempotent?

Duplicates may come from consumer redelivery after a crash, independent producer submissions, upstream retry logic, outbox relay retries, application replay, or separate producer sessions. Producer idempotence only covers qualifying producer retries. Trace the event ID and origin through the whole pipeline.


### Q27. Auto commit or manual offset commit: how do you choose?

Auto commit is simpler but can commit positions independently of completion of business work, depending on poll and processing timing. Manual/container-managed commit gives more control over when work is considered done, but error and rebalance handling become important. Spring Kafka offers acknowledgment modes at the listener-container layer.


### Q28. How do you maintain consistency between a database and Kafka?

The classic dual-write problem occurs when a service commits a database transaction and then publishes to Kafka: a crash between those operations leaves database state and the event stream inconsistent. Reversing the order has the opposite failure mode—an event can be published for a database transaction that later rolls back. The transactional outbox solves this by writing both the business change and an outbox row in the same local database transaction. A polling relay or CDC connector such as Debezium later publishes outbox rows to Kafka. The relay can crash after publishing but before recording progress, so duplicates remain possible; consumers should use event IDs and idempotent processing. Monitor outbox age/backlog and define cleanup, retry, ordering, and schema policies.


### Q29. What is the idempotent consumer pattern?

The consumer uses a stable event ID or operation key to ensure repeated delivery does not repeat the business effect. A common approach inserts the ID into a table with a unique constraint and performs the business update in the same database transaction. If the ID already exists, the duplicate is treated as already handled.


### Q30. How should retries and dead-letter topics be designed?

Retry transient failures with bounded attempts and backoff. Classify permanent errors—such as irreparably invalid schema or business validation—as non-retryable and route them to a DLT/quarantine path. Preserve source metadata, alert on DLT growth, and establish safe replay procedures. Consider ordering: retry topics can let later events overtake an earlier failed event.


### Section D — Java and Spring Kafka (Q31–Q40)

### Q31. How do you configure a Java Kafka producer?

Create a KafkaProducer with bootstrap servers, key/value serializers, and reliability and performance properties. Reuse the producer rather than creating one per record, call send asynchronously when appropriate, and close it gracefully. Match acks, retries, idempotence, batching, compression, and timeouts to the workload.


### Q32. Is KafkaProducer thread-safe? Is KafkaConsumer thread-safe?

`KafkaProducer` is thread-safe and is normally created once per application component and reused; creating one per record wastes connections, buffers, and metadata work. `KafkaConsumer` is not thread-safe, so a single owner thread should perform consumer operations such as `poll`, `commit`, `seek`, `pause`, `resume`, and assignment handling. If records are dispatched to worker threads, the application must track which records have completed and commit only the highest safe contiguous offset per partition. Committing the maximum completed offset while an earlier record is still processing can skip that unfinished record after a crash. For per-key ordering, serialize work for each key or partition, and pause partitions when bounded worker capacity is exhausted.


### Q33. How do you handle the result of producer.send()?

send() is asynchronous and returns a Future; a callback can handle success or failure. Avoid blocking on every send unless synchronous confirmation is required, as this can reduce throughput. Surface failures through metrics/logging and configure retries and idempotence appropriately.


### Q34. What is the purpose of poll() in a Kafka consumer?

`poll()` is not merely a fetch call: it returns available records and drives group coordination, heartbeats/consumer liveness behavior, and rebalance processing according to the client protocol and configuration. If application processing blocks the poll thread longer than `max.poll.interval.ms`, the consumer can be considered unhealthy and lose its assignment, causing a rebalance and possible redelivery. Keep processing time bounded, tune `max.poll.records` to a realistic workload, or move work to a carefully coordinated worker pool. If using workers, preserve per-partition commit ordering and stop or drain work safely during revocation. Increasing timeout values blindly can hide slow processing and delay recovery.


### Q35. When would you use manual acknowledgments in Spring Kafka?

Manual acknowledgment is useful when the application needs an explicit point to declare a record or batch processed. It must be coordinated with listener-container acknowledgment mode, exception propagation, rebalances, and transaction design. Acknowledging before business work is durable can lose work; acknowledging later can cause duplicate delivery.


### Q36. How does @KafkaListener concurrency work?

Spring Kafka listener concurrency creates multiple consumer threads/containers within the application, each participating in the consumer group. Effective parallelism is bounded by assigned partitions. Concurrency must also fit database pools, CPU, API quotas, and ordering constraints.


### Q37. How do you implement retry and DLT handling in Spring Kafka?

In Spring Kafka, container error handling is commonly configured with `DefaultErrorHandler` and, when appropriate, `DeadLetterPublishingRecoverer`. Configure a bounded backoff and retry count, classify exceptions so permanent failures (for example, invalid payloads) are not retried pointlessly, and decide what happens after retries are exhausted: publish to a DLT, quarantine, or stop processing. A DLT should preserve original topic/partition/offset and exception metadata, have access controls and alerting, and support controlled replay. Non-blocking retry topics can free the original partition to process later records, but this can break ordering and are not supported with every listener/transaction arrangement. Validate behavior against the exact Spring Kafka version, listener mode (record/batch), and transaction configuration.


### Q38. How do Kafka transactions work with Spring Kafka?

Spring Kafka supports transactions by configuring a `DefaultKafkaProducerFactory` with a `transactionIdPrefix` and using transactional templates and/or listener containers. In a consume-transform-produce flow, output records and the consumed offsets can be committed atomically to Kafka; if processing fails, the transaction can be aborted and the input records re-delivered. Downstream consumers that must hide aborted output should use `read_committed`. Spring's transaction synchronization can coordinate a Kafka transaction with a database transaction, but it is not a single atomic commit across both systems: one resource can commit while the other commit fails. For database-to-Kafka consistency, the transactional outbox is often the clearer design. Keep transaction IDs unique across concurrently running application instances and account for transaction timeouts and fencing.


### Q39. How do you test a Spring Kafka consumer?

Unit-test business logic without a broker where possible. Integration-test serialization, listener wiring, retry/DLT behavior, offsets, and transaction semantics using a real or compatible Kafka test environment. Synchronize on observable outcomes with bounded waits rather than fixed sleeps.


### Q40. How should a Kafka consumer shut down gracefully?

Stop accepting new work, allow in-flight processing to finish within a deadline, commit only positions corresponding to completed work, and close the consumer/container. Coordinate shutdown with worker pools and avoid committing offsets past unfinished asynchronous tasks. Test shutdown during processing and during rebalance.


## Phase 3 — Performance and production incidents (Q41–Q60)

### Section E — Performance and capacity planning (Q41–Q50)

### Q41. What determines Kafka producer throughput?

Producer throughput is determined by both client behavior and cluster capacity: records per batch, `batch.size`, `linger.ms`, compression, record size, `buffer.memory`, request limits, network bandwidth, broker disk/CPU, replication factor, acknowledgment level, and how evenly keys distribute traffic. Batching and compression often improve throughput by reducing request overhead and bytes transferred, but may increase latency or CPU use. More producer threads do not automatically help if a single partition, broker, or network link is the bottleneck. Benchmark with realistic payload sizes, key skew, replication, security, and consumer load; measure p95/p99 latency and error rate alongside throughput. Change one or two variables at a time and retain headroom for bursts and failures.


### Q42. What do batch.size and linger.ms do?

batch.size bounds the target batch size per partition, while linger.ms allows the producer to wait briefly so more records can join a batch. Larger batches can improve throughput but may add latency and memory usage. Behavior depends on traffic rate and record size.


### Q43. When should you use compression?

Compression reduces network and disk bytes and can improve throughput, at the cost of CPU and sometimes latency. LZ4 and Snappy often emphasize speed; gzip may compress more but use more CPU; Zstandard offers configurable trade-offs depending on version and environment. Benchmark real payloads.


### Q44. How do you calculate consumer lag?

Consumer lag is usually estimated per partition from the difference between the log end offset and the consumer group's committed offset; monitoring tools may use slightly different definitions and may also report records behind the high watermark. Sum lag across partitions only as a rough workload indicator, because offset distance is not the same as bytes or processing time when record sizes and rates vary. Track lag trend, oldest unprocessed event age, input rate, processing rate, and per-partition skew. Rising lag means incoming work exceeds sustainable processing for long enough; stable lag may be acceptable if the age stays within the service objective. Diagnose whether the bottleneck is partition count, hot keys, consumer CPU/GC, downstream latency, rebalances, or broker fetch performance before scaling.


### Q45. How do you detect and fix hot partitions?

Compare throughput, lag, record size, and processing time per partition. Skewed keys, such as one large tenant, can cause a hot partition even when cluster-wide utilization looks low. Improve the hot key's processing, separate traffic, or change partitioning only if ordering semantics permit.


### Q46. How do you choose partition count?

Choose partitions from measured throughput and the parallelism the workload needs, not from a universal rule. Estimate producer throughput per partition, consumer processing capacity per partition, key distribution, peak traffic, future growth, and the maximum recovery time you can tolerate. Within one consumer group, a partition is assigned to at most one consumer at a time, so partitions cap parallel consumer ownership; extra consumers beyond the partition count may sit idle. More partitions also increase metadata, replication traffic, open files, recovery/reassignment work, and can affect end-to-end latency. Increasing partition count can change keyed partition mapping for new records and may break per-key ordering across the transition; reducing partition count generally requires creating a new topic and migrating data.


### Q47. How do you estimate Kafka storage?

Approximate retained primary log storage as daily data volume multiplied by retention days, then adjust for compression and add replication copies, indexes, segment overhead, and free-space headroom. Also account for retention spikes, compaction, and data-growth trends. Validate against broker metrics and actual segment usage.


### Q48. Which consumer settings commonly affect throughput?

max.poll.records controls records returned per poll; fetch sizes influence batching and network requests; max.poll.interval.ms limits time between polls; fetch wait/minimum bytes can trade latency for fuller fetches. Tune together based on payload sizes, processing duration, and consumer resources.


### Q49. How do you apply backpressure in a Kafka consumer?

Bound worker queues and concurrent requests, pause partitions when downstream capacity is exhausted, and resume them when capacity returns. Avoid unlimited in-memory buffering and ensure offset commits do not move past unfinished records. Consider per-key ordering when dispatching work asynchronously.


### Q50. Consumer processing is slower than database writes can sustain. What do you do?

Measure database latency, connection-pool saturation, locks, query plans, and concurrency. Batch operations where semantics permit, reduce transaction duration, optimize indexes, and cap concurrency. Increasing consumers can amplify database overload rather than solve it.


### Section F — Production troubleshooting and reliability (Q51–Q60)

### Q51. How do you troubleshoot a broker unreachable error?

Check DNS and advertised.listeners, network paths, TLS/SASL configuration, load balancer behavior, firewall rules, and broker health. A bootstrap connection can succeed while a returned advertised broker address is unreachable from the client. Capture client logs and test connectivity to advertised endpoints.


### Q52. What do under-replicated partitions mean?

An under-replicated partition has at least one replica that is not currently in sync with the leader; the metric is an early warning that redundancy is reduced. Check whether the cause is a broker outage, overloaded disk, network saturation, long GC pauses, slow replica fetch, throttling, or a reassignment. Also inspect offline partitions and ISR shrink/expand events: an under-replicated partition with a healthy leader differs from a partition with no leader. Restore broker and replication health, verify replicas rejoin the ISR, and confirm the cluster has adequate failure tolerance before closing the incident. Avoid weakening durability settings merely to make the alert disappear.


### Q53. How do you debug repeated rebalances?

Repeated rebalances interrupt consumption and can increase lag or duplicate processing. Check consumer logs and group coordinator metrics for join/leave churn, failed heartbeats, session timeouts, `max.poll.interval.ms` violations, JVM pauses, network issues, rolling deployments, and autoscaling. Slow processing inside the poll loop is a common cause; reduce work per poll, bound downstream calls, or use a coordinated worker model. Cooperative assignment protocols can reduce how many partitions move during a rebalance, but they do not fix unstable members or long processing. Tune timeouts only after measuring real processing and failure-detection requirements, and verify the client/broker versions support the chosen protocol.


### Q54. How do you deal with a poison message?

Classify whether the failure is transient or permanent. Use bounded retries for transient errors; route permanent failures to a DLT/quarantine path with source metadata and alerting. Plan replay and respect ordering requirements—continuing past a failed record may not be valid for every business stream.


### Q55. How do you safely replay DLT records?

Identify and fix the cause before replay. Preserve event IDs, record source metadata, and track replay attempts. Reprocess through a controlled pipeline with idempotent business effects and limits on replay rate. Do not blindly copy all records back to the original topic if ordering or duplicates can harm state.


### Q56. What is unclean leader election, and what is its trade-off?

Unclean leader election allows a replica outside the ISR to become leader if no eligible in-sync replica is available. This can restore availability but may discard records that existed only on the former leader. Evaluate the setting against the business's tolerance for data loss.


### Q57. What if a Kafka broker's disk is almost full?

Identify which topics and partitions consume space, check retention/compaction cleanup, replica placement, ingestion growth, and segment deletion progress. Expand capacity or reduce data only according to approved retention/recovery requirements. Maintain headroom for segment rolling, reassignment, and recovery.


### Q58. How do you differentiate producer timeout from failed publication?

A producer timeout is an uncertain outcome, not proof that Kafka rejected the record. The broker may have appended it and the acknowledgment may have been lost, or the request may never have succeeded. Check the exception type, delivery callback, `delivery.timeout.ms`, request/retry metrics, broker logs, and partition health. Do not blindly create a new business operation because the first send timed out; use a stable event ID and downstream deduplication where duplicates are harmful. Producer idempotence can make protocol retries safe within its scope, but it cannot identify an independent application-level resend as the same business event.


### Q59. What does NOT_LEADER_OR_FOLLOWER or LEADER_NOT_AVAILABLE suggest?

The client may have stale metadata during leader movement or the partition may temporarily lack an available leader. Metadata refresh and retry can resolve transient cases; persistent failures require inspecting partition state, broker availability, controller health, and election logs.


### Q60. Which Kafka metrics would you monitor in production?

Monitor under-replicated/offline partitions, ISR changes, request latency and errors, network/disk/CPU, produce/fetch throughput, consumer lag and age, rebalance rate, failed/retried records, DLT volume, and storage growth. Define SLOs and alert around business impact rather than a noisy single metric.


## Phase 4 — System design (Q61–Q70)

### Section G — Distributed systems and architecture (Q61–Q70)

### Q61. Why choose event-driven architecture with Kafka?

It decouples producers and consumers, allows independent scaling, retains events for replay, and supports multiple consumers with different responsibilities. Trade-offs include eventual consistency, operational complexity, schema evolution, and duplicate/order handling.


### Q62. How do you design event contracts between microservices?

Define event names, ownership, schema, keys, timestamp semantics, event IDs, compatibility policy, and versioning expectations. Publish facts that have happened rather than commands disguised as events. Avoid leaking internal database structures into public contracts.


### Q63. Design order creation and inventory reservation.

Persist the order and outbox event transactionally; publish the event; let inventory reserve stock idempotently and emit a result. Model pending, confirmed, and rejected states explicitly. If inventory fails after order creation, use a saga/compensating action or mark the order for cancellation rather than assuming cross-service atomicity.


### Q64. How would you design payment processing with Kafka?

Treat Kafka as event transport, not the payment ledger. Use stable payment/idempotency keys, durable state, gateway idempotency support, explicit state transitions, reconciliation, and carefully controlled retries. Avoid assuming a timeout proves the external charge failed.


### Q65. Explain the transactional outbox pattern.

Write the business row and an outbox event in one database transaction. A relay or CDC connector publishes outbox rows to Kafka and tracks delivery. The relay can publish more than once, so consumers must deduplicate or be idempotent. Monitor outbox age and backlog.


### Q66. What is a saga?

A saga coordinates a multi-service business process as a sequence of local transactions, with events or commands advancing the workflow. In a choreography, services react to each other's events; in an orchestration, a coordinator explicitly manages state and next steps. If a later step fails, the workflow runs compensating business actions—for example, release a reservation or refund a payment—rather than rolling back a remote database transaction that already committed. Design explicit states, idempotent handlers, timeouts, retry policies, compensation failure handling, and reconciliation for ambiguous outcomes. Kafka can transport saga messages, but the application still owns the workflow state machine and operational recovery.


### Q67. How do you evolve schemas without breaking consumers?

Adopt compatibility rules appropriate for Avro, Protobuf, JSON Schema, or the chosen format. Add optional fields with suitable defaults where compatible; avoid removing or changing required fields without a migration plan. Deploy consumers that can handle the new shape before producers emit it when necessary.


### Q68. How do you support replay without repeating harmful side effects?

Use stable event IDs, idempotent consumers, replay-specific controls, and separate state-rebuild paths from external side effects where possible. Record replay source and maintain auditability. Define whether replay rebuilds a projection, reissues notifications, or repeats a business command.


### Q69. How would you plan multi-region Kafka disaster recovery?

Define recovery-point objective and recovery-time objective, then choose replication, backup/restore, or active/passive patterns supported by the platform. Specify topic/config replication, schema/ACL restoration, offset and consumer-state strategy, failover ownership, and split-brain prevention. Test failover and failback regularly.


### Q70. How do you build a reliable audit/event history?

Use explicit audit schemas, reliable source publication, restrictive ACLs, retention and archive policies, monitoring for gaps, and access auditing. Kafka alone does not prove legal immutability; archive to storage with suitable immutability/integrity controls when required. Avoid compaction if the full history must be retained.


## Phase 5 — Internals and advanced topics (Q71–Q90)

### Section H — Kafka internals (Q71–Q75)

### Q71. How does Kafka store records on disk?

Partitions are append-only logs split into segment files. Writes are appended and offsets identify records within a partition; indexes help locate offsets and timestamps. Background retention or compaction removes eligible segments/records according to topic policy.


### Q72. What is a segment, and what are segment rolling settings?

A segment is a portion of a partition log. Brokers roll to a new segment based on size/time-related settings, and retention/cleanup generally removes or compacts eligible segments. Smaller segments can increase file/cleanup overhead; larger segments can affect cleanup granularity.


### Q73. What is the high watermark?

The high watermark represents the point up to which records are considered replicated sufficiently for normal consumer visibility under Kafka's replication model. Consumers generally do not read beyond it in the standard path. Transactional read_committed consumers also have a last-stable-offset constraint.


### Q74. What is the difference between the log end offset and the last stable offset?

The log end offset marks the end of the current log. For transactional data, the last stable offset is the point before the first open transaction that prevents read_committed consumers from safely advancing further. A consumer can therefore be caught up to committed data while the log end offset is farther ahead.


### Q75. What happens during partition reassignment?

Partition reassignment moves replicas to different brokers to balance load or recover capacity. It can create extra network and disk I/O while replicas copy data. Plan concurrency/throttling, monitor progress and broker impact, and verify replication health afterward.


### Section I — Streams, Connect, CDC and schemas (Q76–Q85)

### Q76. How does Kafka Streams differ from a plain consumer?

Kafka Streams provides a library for transformations, joins, aggregations, windows, local state stores, and Kafka-backed changelogs. A plain consumer gives lower-level control but requires you to implement much of the state and recovery logic yourself.


### Q77. What is the difference between KStream and KTable?

A KStream models a flow of records/events, where each record is an independent fact. A KTable models the latest value for each key, with updates replacing prior state for that key. The semantics influence joins, aggregations, and interpretation of updates.


### Q78. What are event-time windows and late-arriving events?

Windowed aggregation groups records by a time interval. Event time comes from the event timestamp; processing time is when the application handles it. Out-of-order or late records need a grace-period and correction policy, otherwise results may be incomplete or delayed.


### Q79. How does Kafka Streams recover local state?

Stateful operations use local state stores and Kafka changelog topics to restore state after restart or task migration. Recovery time depends on state size, changelog volume, local disk, and restoration throughput. Monitor restore duration and ensure state/changelog retention matches the processing design.


### Q80. What is Kafka Connect?

Kafka Connect is a framework for moving data between Kafka and external systems using source and sink connectors. It manages tasks, configurations, offsets, and connector lifecycle. Connector delivery guarantees depend on the connector and target-system semantics.


### Q81. What is CDC, and how does Debezium fit?

Change data capture streams database changes from transaction logs or other supported mechanisms. Debezium provides connectors that capture changes and publish change events, often to Kafka. CDC can support outbox publication and data integration, but schema, snapshot, ordering, and recovery behaviors need configuration.


### Q82. How does Kafka Connect manage source offsets?

Source connectors store progress according to connector behavior and worker configuration so they can resume after interruption. Correctness depends on source offset semantics, durability of connector configuration/offset topics, and connector capabilities. Review snapshot modes, restart behavior, and duplicate possibilities.


### Q83. What is a schema registry used for?

A schema registry stores versioned schema definitions and can enforce compatibility policies, often using a schema ID in serialized payloads. It helps producers and consumers agree on serialization formats and schema evolution. The exact mechanism depends on registry and serializer implementation.


### Q84. What is a tombstone record?

For compacted topics, a record with a key and null value acts as a deletion marker. The tombstone is retained for a configured period so downstream state stores can observe the deletion; later compaction may remove it. Tombstones are not equivalent to an empty JSON object.


### Q85. What does exactly-once mean in Kafka Streams?

Kafka Streams can coordinate input offsets, state-store changelog records, and output records transactionally under the configured processing guarantee. This prevents inconsistent Kafka-visible results after recovery within its documented scope. External writes to arbitrary systems still need their own consistency strategy.


### Section J — Security and operational awareness (Q86–Q90)

### Q86. How do you secure Kafka clients with TLS and SASL?

TLS encrypts traffic in transit and lets clients validate broker certificates; mutual TLS can also authenticate clients through certificates. SASL authenticates a client using a supported mechanism such as SCRAM or OAUTHBEARER, while authorization is a separate decision typically enforced with ACLs or an authorizer. Configure the correct listener security protocol, truststore/CA chain, credentials, and mechanism on both client and broker. Use least-privilege identities, keep secrets out of code and logs, rotate credentials and certificates, and test expiration/renewal before production. Encryption and authentication do not replace topic/group/transaction authorization or payload-level privacy controls.


### Q87. What are Kafka ACLs?

Access-control lists authorize principals to perform operations on resources such as topics, consumer groups, and transactional IDs. Apply least privilege, keep producer and consumer identities distinct where practical, and review ACL changes as part of deployment.


### Q88. How should Kafka credentials and certificates be handled?

Keep secrets out of source control and ordinary application logs. Use a managed secret store, restrict who can read credentials, rotate them safely, and monitor expiry. Use different credentials for environments and services with least privilege.


### Q89. What privacy risks arise from Kafka event payloads?

Events may replicate sensitive data into many topics, consumer groups, logs, DLTs, and archives. Minimize personal data, classify fields, restrict access, define retention/deletion obligations, encrypt where appropriate, and ensure dead-letter/replay tooling does not expose sensitive payloads broadly.


### Q90. What should a Kafka disaster-recovery plan include?

Define RPO/RTO, backups or replication strategy, cluster provisioning, topic configs, schemas, ACLs, offsets, connector state, and consumer recovery procedures. Test recovery, replay, and failback under realistic conditions. Ensure teams know which region or cluster is authoritative during failover.


## Phase 6 — Mock interviews (Q91–Q100)

### Q91. Scenario: A consumer's DB write succeeds, but its offset commit fails. What happens?

On restart or reassignment, Kafka may deliver the record again because the committed position did not advance. Use a stable event ID and atomically persist a deduplication marker with the business mutation. Commit only after the DB transaction succeeds.


### Q92. Scenario: Lag starts right after a release. How do you triage?

Compare release timestamps with lag, processing latency, errors, rebalance rate, CPU/GC, DB/API latency, and per-partition metrics. Roll back if the release is the likely cause and rollback is safe. Avoid scaling consumers until you know whether the bottleneck is partition parallelism or downstream saturation.


### Q93. Scenario: Two independent producers publish the same business event. Will idempotence deduplicate them?

Generally no. Producer idempotence deduplicates qualifying retries within a producer identity/session, not semantically identical events emitted independently by two producers. Use a stable business event ID, source ownership, and downstream idempotency.


### Q94. Scenario: Records for one order appear out of order. What do you check?

Check whether records use the same key and partition, whether partition count or partitioner configuration changed, whether producers publish concurrently, whether retries reorder sends, and whether consumers process records asynchronously without per-key sequencing. Use sequence numbers or state-version checks if required.


### Q95. Scenario: Events arrive late and have old timestamps. How do you process them?

Determine whether the timestamp is event time or ingestion time. For analytics, use event-time windows with an explicit grace period and late-event correction strategy. For business workflows, validate state versions and define whether stale events are accepted, ignored, or sent for reconciliation.


### Q96. Scenario: There is a 100 GB backlog. How do you estimate drain time?

Estimate net drain rate as sustainable processing throughput minus ongoing producer throughput. If processing handles 150 MB/s and incoming traffic is 50 MB/s, net drain is about 100 MB/s, so 100 GB takes roughly 1,000 seconds under idealized steady conditions. Include partition skew, dependency capacity, and overhead in the real estimate.


### Q97. Scenario: A schema change breaks only one consumer. What is your approach?

Identify that consumer's schema/serializer version, failing record, and compatibility mode. Check required fields, defaults, type changes, registry compatibility, and rollout order. Restore compatible schema behavior, then add cross-version contract tests.


### Q98. Scenario: The database is unavailable but Kafka producers are healthy. Should consumers continue?

Avoid unbounded retries and memory queues. Apply bounded backoff, pause or limit consumption where appropriate, monitor lag and oldest-event age, and ensure retention can accommodate the outage. Once the database returns, restore processing gradually and watch for overload and duplicates.


### Q99. Scenario: You need to migrate a consumer group or reset offsets. What safeguards are needed?

Document current group offsets, target offsets, replay range, and business impact. Test on staging, ensure idempotent effects, coordinate ownership so two consumers do not apply conflicting work, and capture metrics/audit evidence. Offset reset may skip or replay data and should be approved.


### Q100. Scenario: Interviewer asks how you lead a Kafka incident. What is your framework?

Establish impact and data-integrity risk first. Stabilize the system with reversible changes, identify whether failures come from brokers, producers, consumers, partitioning, or downstream dependencies, and communicate status and decisions. Recover safely, reconcile critical business effects, then add tests, alerts, and runbooks based on the root cause.


## Senior-level revision checklist

Use this checklist to rehearse your answers aloud:

- [ ] Explain leader failure, replication, and ISR shrinkage.
- [ ] Diagnose consumer lag, rebalances, and hot partitions.
- [ ] Explain idempotency, commits, and duplicate delivery.
- [ ] Design outbox, retries, DLT, and replay workflows.
- [ ] Compare Kafka transactions with database transactions.
- [ ] Design order processing and fraud detection.
- [ ] Explain schema compatibility and Spring Kafka failure handling.
- [ ] Walk through an incident from detection through prevention.

For each scenario, practice in this order: **clarify requirements → explain the failure mechanism → propose a design → discuss trade-offs → describe monitoring and recovery**.

## References

- [Apache Kafka documentation](https://kafka.apache.org/documentation/)
- [Spring for Apache Kafka reference](https://docs.spring.io/spring-kafka/reference/)
- [Kafka Streams documentation](https://kafka.apache.org/documentation/streams/)
- [Kafka Connect documentation](https://kafka.apache.org/documentation/#connect)
- [Debezium documentation](https://debezium.io/documentation/)
- [Kafka Improvement Proposals (KIPs)](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals)

> **Accuracy reminder:** Kafka and Spring Kafka behavior can vary by version and configuration. Validate production settings against the documentation for the exact version in use. Treat “exactly once” claims as scoped guarantees; they do not automatically encompass arbitrary external systems.
