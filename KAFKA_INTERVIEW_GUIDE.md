# Apache Kafka Interview Preparation Guide

> **Audience:** Senior Software Engineer / Senior Java Backend Engineer  
> **Focus:** Kafka fundamentals, Java/Spring Kafka, delivery guarantees, performance, production troubleshooting, internals, and distributed-system design.  
> **Total:** 130 interview questions and answers.

## Editorial note

The guide follows the six-phase structure discussed in the conversation. The exact wording of the original first 100 answers was not available in the current working context, so Q1–Q100 below are rebuilt answers aligned to the agreed topics, rather than a verbatim transcript. Q101–Q130 retain the detailed scenario-based answers already drafted.

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
- [Advanced scenario-based Q&A (Q101–Q130)](#advanced-scenario-based-qa)
  - [Mock Interview 1 — Consumer reliability](#mock-interview-1--consumer-reliability)
  - [Mock Interview 2 — Data consistency](#mock-interview-2--data-consistency)
  - [Mock Interview 3 — Broker failures and reliability](#mock-interview-3--broker-failures-and-reliability)
  - [Mock Interview 4 — Consumer groups, offsets, and ordering](#mock-interview-4--consumer-groups-offsets-and-ordering)
  - [Mock Interview 5 — Retry architecture and delivery guarantees](#mock-interview-5--retry-architecture-and-delivery-guarantees)
  - [Mock Interview 6 — Java and Spring Kafka](#mock-interview-6--java-and-spring-kafka)
  - [Mock Interview 7 — System design and advanced architecture](#mock-interview-7--system-design-and-advanced-architecture)
- [Senior-level revision checklist](#senior-level-revision-checklist)
- [References](#references)

## How to use this guide

For each question, practice explaining the concept or failure mechanism, the proposed design, trade-offs, and how you would validate it in production. Examples are illustrative; tailor configuration and APIs to the Kafka and Spring Kafka versions in use.

## Phase 1 — Core concepts (Q1–Q20)

### Section A — Kafka fundamentals and internals (Q1–Q10)

### Q1. What is Apache Kafka, and what problems does it solve?

What is Apache Kafka, and what problems does it solve?

**Answer**

Kafka is a distributed event-streaming platform that stores records in partitioned, replicated logs and lets independent applications publish, consume, process, and retain event streams. It decouples producers from consumers, supports replay, and scales through partitioning. It is not automatically a replacement for a database or every message broker; the right choice depends on replay, ordering, retention, latency, and delivery requirements.


### Q2. How is Kafka different from a traditional message queue?

How is Kafka different from a traditional message queue?

**Answer**

Many traditional queues emphasize work distribution and may remove messages once acknowledged. Kafka stores an ordered log for a configured retention period, so multiple consumer groups can independently read and replay the same records. Kafka also scales through partitions and replication. Exact behavior depends on queue/broker technology and configuration, so avoid absolute comparisons.


### Q3. What is a Kafka topic?

What is a Kafka topic?

**Answer**

A topic is a named stream of records. It is split into partitions, each an append-only ordered log. Topic-level settings control behavior such as replication factor, retention, and cleanup policy. Different consumer groups can read the same topic independently and maintain separate offsets.


### Q4. What is a partition, and why does Kafka use partitions?

What is a partition, and why does Kafka use partitions?

**Answer**

A partition is an ordered log that enables parallel reads and writes across brokers. Each record in a partition has an offset. Partitioning is Kafka's primary scaling unit, but ordering is guaranteed only within a single partition, not across all partitions of a topic.


### Q5. What is a Kafka record?

What is a Kafka record?

**Answer**

A record can contain a key, value, timestamp, headers, and partition/offset metadata assigned by Kafka. The key can influence partition selection, while the value contains the event payload. Headers commonly carry tracing, schema, or routing metadata; avoid placing secrets or sensitive data there without protection.


### Q6. What is an offset?

What is an offset?

**Answer**

An offset is a record's position within one partition. It is not a global topic sequence and should not be interpreted as a timestamp. Consumer-group offsets record the next position to read, and may be committed automatically or explicitly depending on consumer configuration.


### Q7. What is a Kafka broker?

What is a Kafka broker?

**Answer**

A broker is a Kafka server that stores partition replicas, serves producer and consumer requests, and participates in cluster coordination. A topic's partitions are distributed among brokers. A broker can lead some partitions and host followers for others.


### Q8. What does a Kafka cluster provide?

What does a Kafka cluster provide?

**Answer**

A Kafka cluster is a set of brokers coordinated to host partition replicas, elect leaders, and serve clients. The cluster spreads partitions and replica copies to provide capacity and fault tolerance. Cluster health depends on both broker availability and the replica state of individual partitions.


### Q9. What is the controller in Kafka?

What is the controller in Kafka?

**Answer**

The controller coordinates cluster metadata and broker/partition state transitions. In modern Kafka deployments using KRaft, controller quorum members use a metadata log to agree on cluster metadata and leadership decisions. Brokers still serve normal producer and consumer traffic.


### Q10. How does KRaft differ from ZooKeeper-based Kafka?

How does KRaft differ from ZooKeeper-based Kafka?

**Answer**

KRaft uses Kafka's own quorum-based metadata management rather than external ZooKeeper for cluster metadata and controller coordination. It reduces the need to operate a separate coordination system. Migration procedures and support vary by Kafka version and deployment, so follow version-specific upgrade documentation.


### Section B — Producers and consumers (Q11–Q20)

### Q11. What happens when a producer sends a record with a key?

What happens when a producer sends a record with a key?

**Answer**

The producer's partitioner selects a partition. With typical keyed partitioning, records with the same key map to the same partition for a given partitioning configuration, preserving their partition order. Custom partitioners and partition-count changes can alter this mapping.


### Q12. What happens when a record has no key?

What happens when a record has no key?

**Answer**

The partitioner chooses a partition according to the client version and configuration; modern clients may use sticky partitioning to batch efficiently rather than strictly round-robin each record. With no stable key, per-entity ordering is not naturally ensured.


### Q13. Does Kafka guarantee ordering?

Does Kafka guarantee ordering?

**Answer**

Kafka preserves order within a partition, meaning a consumer reads that partition in offset order. It does not guarantee global order across multiple partitions. Producers, retries, and application-level asynchronous processing must also be configured or designed not to violate business ordering.


### Q14. What is a consumer group?

What is a consumer group?

**Answer**

A consumer group is a set of consumers that cooperate to consume a topic's partitions. Each partition is assigned to at most one active consumer within the group at a time, while separate groups independently consume the topic. Group membership and partition assignment can change during rebalances.


### Q15. What happens when a consumer commits an offset?

What happens when a consumer commits an offset?

**Answer**

The consumer group stores a position indicating where it should resume. If it commits before processing completes, a failure can skip unfinished work. If it commits after processing, a crash between processing and commit can cause redelivery. Commit strategy therefore affects loss/duplication behavior.


### Q16. How does Kafka retention work?

How does Kafka retention work?

**Answer**

Retention policies delete old log segments based on time and/or size, subject to cleanup behavior. Reading a record does not normally delete it. Retention determines how far consumers can replay data and contributes directly to storage cost and recovery capability.


### Q17. What is log compaction?

What is log compaction?

**Answer**

Compaction preserves the latest known value per key over time, while older values for the same key may be removed. A null-valued record can act as a tombstone that eventually removes a key's retained state. Compaction is useful for changelog/state topics, not for preserving every historical event forever.


### Q18. What is a replication factor?

What is a replication factor?

**Answer**

The replication factor is the number of replicas configured for each partition, including one leader and follower replicas. Higher replication increases resilience but also storage and network cost. Replica placement matters: copies should not all share the same failure domain.


### Q19. What are partition leaders and followers?

What are partition leaders and followers?

**Answer**

The leader handles normal reads and writes for a partition under the standard model. Followers replicate the leader's log and can become leaders if eligible. A follower that falls too far behind may leave the ISR until it catches up again.


### Q20. What is the ISR, and why does it matter?

What is the ISR, and why does it matter?

**Answer**

The in-sync replica set contains replicas considered caught up enough to participate in normal leader-election and durability decisions. With producer acks=all and a suitable min.insync.replicas, Kafka can reject writes when too few replicas are in sync, reducing the risk of acknowledging data with weak replication.


## Phase 2 — Delivery guarantees and Java (Q21–Q40)

### Section C — Delivery guarantees and consistency (Q21–Q30)

### Q21. Explain at-most-once, at-least-once, and exactly-once processing.

Explain at-most-once, at-least-once, and exactly-once processing.

**Answer**

At-most-once may lose work but avoids retries that cause duplicate processing; at-least-once retries unfinished work but can duplicate effects; exactly-once has a specific scope and requires coordinated transactional or idempotent design. In distributed applications, exactly-once business effects across independent systems usually require idempotency, transactional state, and reconciliation.


### Q22. What does producer acks control?

What does producer acks control?

**Answer**

acks controls how much broker acknowledgment the producer waits for. acks=0 waits for no broker acknowledgment, acks=1 waits for the leader, and acks=all waits for the required in-sync replicas. Actual durability also depends on replication factor, min.insync.replicas, and leader-election settings.


### Q23. How does producer idempotence work?

How does producer idempotence work?

**Answer**

Idempotent production uses producer IDs and sequence numbers so Kafka can deduplicate qualifying retries from a producer session and preserve ordering under its supported settings. It does not deduplicate independent application submissions, nor does it guarantee a consumer's external side effect runs only once. Compatible configuration is required; current clients generally choose suitable defaults unless overridden.


### Q24. What is a Kafka transaction?

What is a Kafka transaction?

**Answer**

A Kafka transaction can atomically publish records across multiple Kafka partitions and commit consumed offsets as part of a consume-transform-produce workflow. Transactional producers use a transactional ID and transaction coordinator. Consumers needing committed transactional visibility should use read_committed isolation.


### Q25. What does exactly-once semantics mean in Kafka Streams or consume-transform-produce?

What does exactly-once semantics mean in Kafka Streams or consume-transform-produce?

**Answer**

With appropriate processing and transaction configuration, Kafka can atomically commit output records and consumed offsets, preventing a restarted processor from exposing output for an input position that was not committed consistently. This guarantee has defined boundaries and does not automatically encompass arbitrary external systems.


### Q26. Why can consumers receive duplicates even when producers are idempotent?

Why can consumers receive duplicates even when producers are idempotent?

**Answer**

Duplicates may come from consumer redelivery after a crash, independent producer submissions, upstream retry logic, outbox relay retries, application replay, or separate producer sessions. Producer idempotence only covers qualifying producer retries. Trace the event ID and origin through the whole pipeline.


### Q27. Auto commit or manual offset commit: how do you choose?

Auto commit or manual offset commit: how do you choose?

**Answer**

Auto commit is simpler but can commit positions independently of completion of business work, depending on poll and processing timing. Manual/container-managed commit gives more control over when work is considered done, but error and rebalance handling become important. Spring Kafka offers acknowledgment modes at the listener-container layer.


### Q28. How do you maintain consistency between a database and Kafka?

How do you maintain consistency between a database and Kafka?

**Answer**

A database commit followed by Kafka publication creates a dual-write gap. The transactional outbox pattern writes the business change and an outbox row in one database transaction; a separate relay or CDC connector publishes the event. Consumers should be idempotent because publication can repeat.


### Q29. What is the idempotent consumer pattern?

What is the idempotent consumer pattern?

**Answer**

The consumer uses a stable event ID or operation key to ensure repeated delivery does not repeat the business effect. A common approach inserts the ID into a table with a unique constraint and performs the business update in the same database transaction. If the ID already exists, the duplicate is treated as already handled.


### Q30. How should retries and dead-letter topics be designed?

How should retries and dead-letter topics be designed?

**Answer**

Retry transient failures with bounded attempts and backoff. Classify permanent errors—such as irreparably invalid schema or business validation—as non-retryable and route them to a DLT/quarantine path. Preserve source metadata, alert on DLT growth, and establish safe replay procedures. Consider ordering: retry topics can let later events overtake an earlier failed event.


### Section D — Java and Spring Kafka (Q31–Q40)

### Q31. How do you configure a Java Kafka producer?

How do you configure a Java Kafka producer?

**Answer**

Create a KafkaProducer with bootstrap servers, key/value serializers, and reliability and performance properties. Reuse the producer rather than creating one per record, call send asynchronously when appropriate, and close it gracefully. Match acks, retries, idempotence, batching, compression, and timeouts to the workload.


### Q32. Is KafkaProducer thread-safe? Is KafkaConsumer thread-safe?

Is KafkaProducer thread-safe? Is KafkaConsumer thread-safe?

**Answer**

KafkaProducer is thread-safe and can be shared. KafkaConsumer is not thread-safe; one thread should own its poll, assignment, seek, pause/resume, and commit interactions. Worker threads can process dispatched records only if the application coordinates completion and offset commits carefully.


### Q33. How do you handle the result of producer.send()?

How do you handle the result of producer.send()?

**Answer**

send() is asynchronous and returns a Future; a callback can handle success or failure. Avoid blocking on every send unless synchronous confirmation is required, as this can reduce throughput. Surface failures through metrics/logging and configure retries and idempotence appropriately.


### Q34. What is the purpose of poll() in a Kafka consumer?

What is the purpose of poll() in a Kafka consumer?

**Answer**

poll() fetches records and drives consumer-group coordination and rebalance callbacks. Consumers must call poll often enough to remain active within configured limits. Long-running work inside the poll thread can exceed max.poll.interval.ms.


### Q35. When would you use manual acknowledgments in Spring Kafka?

When would you use manual acknowledgments in Spring Kafka?

**Answer**

Manual acknowledgment is useful when the application needs an explicit point to declare a record or batch processed. It must be coordinated with listener-container acknowledgment mode, exception propagation, rebalances, and transaction design. Acknowledging before business work is durable can lose work; acknowledging later can cause duplicate delivery.


### Q36. How does @KafkaListener concurrency work?

How does @KafkaListener concurrency work?

**Answer**

Spring Kafka listener concurrency creates multiple consumer threads/containers within the application, each participating in the consumer group. Effective parallelism is bounded by assigned partitions. Concurrency must also fit database pools, CPU, API quotas, and ordering constraints.


### Q37. How do you implement retry and DLT handling in Spring Kafka?

How do you implement retry and DLT handling in Spring Kafka?

**Answer**

Use an appropriate container error handler and recoverer, such as DefaultErrorHandler with DeadLetterPublishingRecoverer, or configured non-blocking retry topics where suitable. Classify transient versus permanent exceptions, set bounded backoff, preserve headers and source metadata, and monitor recoveries. Exact APIs and capabilities vary by Spring Kafka version and listener type.


### Q38. How do Kafka transactions work with Spring Kafka?

How do Kafka transactions work with Spring Kafka?

**Answer**

Spring Kafka can use a transaction-capable producer factory and listener container so consumed offsets and produced Kafka records participate in Kafka transactions. Failure can roll back the transaction and make records eligible for redelivery. External database writes are not automatically in the same Kafka transaction.


### Q39. How do you test a Spring Kafka consumer?

How do you test a Spring Kafka consumer?

**Answer**

Unit-test business logic without a broker where possible. Integration-test serialization, listener wiring, retry/DLT behavior, offsets, and transaction semantics using a real or compatible Kafka test environment. Synchronize on observable outcomes with bounded waits rather than fixed sleeps.


### Q40. How should a Kafka consumer shut down gracefully?

How should a Kafka consumer shut down gracefully?

**Answer**

Stop accepting new work, allow in-flight processing to finish within a deadline, commit only positions corresponding to completed work, and close the consumer/container. Coordinate shutdown with worker pools and avoid committing offsets past unfinished asynchronous tasks. Test shutdown during processing and during rebalance.


## Phase 3 — Performance and production incidents (Q41–Q60)

### Section E — Performance and capacity planning (Q41–Q50)

### Q41. What determines Kafka producer throughput?

What determines Kafka producer throughput?

**Answer**

Throughput depends on batching, linger time, compression, record size, request size, broker/network capacity, partition distribution, acknowledgment requirements, and producer concurrency. Optimize from a representative load test and measure end-to-end latency as well as throughput.


### Q42. What do batch.size and linger.ms do?

What do batch.size and linger.ms do?

**Answer**

batch.size bounds the target batch size per partition, while linger.ms allows the producer to wait briefly so more records can join a batch. Larger batches can improve throughput but may add latency and memory usage. Behavior depends on traffic rate and record size.


### Q43. When should you use compression?

When should you use compression?

**Answer**

Compression reduces network and disk bytes and can improve throughput, at the cost of CPU and sometimes latency. LZ4 and Snappy often emphasize speed; gzip may compress more but use more CPU; Zstandard offers configurable trade-offs depending on version and environment. Benchmark real payloads.


### Q44. How do you calculate consumer lag?

How do you calculate consumer lag?

**Answer**

Lag is commonly measured as the difference between a partition's latest available offset position and the consumer group's committed/current position, with tooling-specific details affecting exact counts. Monitor both lag count and oldest-record age because a small lag of slow, time-sensitive events can matter more than a large backlog of low-priority events.


### Q45. How do you detect and fix hot partitions?

How do you detect and fix hot partitions?

**Answer**

Compare throughput, lag, record size, and processing time per partition. Skewed keys, such as one large tenant, can cause a hot partition even when cluster-wide utilization looks low. Improve the hot key's processing, separate traffic, or change partitioning only if ordering semantics permit.


### Q46. How do you choose partition count?

How do you choose partition count?

**Answer**

Estimate required throughput per partition, consumer parallelism, expected growth, key distribution, and failover/recovery requirements. More partitions enable more parallelism but increase metadata, open files, replication overhead, and operational work. Partition count is not easy to reduce in place.


### Q47. How do you estimate Kafka storage?

How do you estimate Kafka storage?

**Answer**

Approximate retained primary log storage as daily data volume multiplied by retention days, then adjust for compression and add replication copies, indexes, segment overhead, and free-space headroom. Also account for retention spikes, compaction, and data-growth trends. Validate against broker metrics and actual segment usage.


### Q48. Which consumer settings commonly affect throughput?

Which consumer settings commonly affect throughput?

**Answer**

max.poll.records controls records returned per poll; fetch sizes influence batching and network requests; max.poll.interval.ms limits time between polls; fetch wait/minimum bytes can trade latency for fuller fetches. Tune together based on payload sizes, processing duration, and consumer resources.


### Q49. How do you apply backpressure in a Kafka consumer?

How do you apply backpressure in a Kafka consumer?

**Answer**

Bound worker queues and concurrent requests, pause partitions when downstream capacity is exhausted, and resume them when capacity returns. Avoid unlimited in-memory buffering and ensure offset commits do not move past unfinished records. Consider per-key ordering when dispatching work asynchronously.


### Q50. Consumer processing is slower than database writes can sustain. What do you do?

Consumer processing is slower than database writes can sustain. What do you do?

**Answer**

Measure database latency, connection-pool saturation, locks, query plans, and concurrency. Batch operations where semantics permit, reduce transaction duration, optimize indexes, and cap concurrency. Increasing consumers can amplify database overload rather than solve it.


### Section F — Production troubleshooting and reliability (Q51–Q60)

### Q51. How do you troubleshoot a broker unreachable error?

How do you troubleshoot a broker unreachable error?

**Answer**

Check DNS and advertised.listeners, network paths, TLS/SASL configuration, load balancer behavior, firewall rules, and broker health. A bootstrap connection can succeed while a returned advertised broker address is unreachable from the client. Capture client logs and test connectivity to advertised endpoints.


### Q52. What do under-replicated partitions mean?

What do under-replicated partitions mean?

**Answer**

They indicate that some replicas are not caught up enough to be in sync. Investigate broker load, disk latency, network saturation, GC pauses, restarts, and replication throttles. Restore replica health and confirm ISR recovery before treating the incident as resolved.


### Q53. How do you debug repeated rebalances?

How do you debug repeated rebalances?

**Answer**

Inspect group membership churn, consumer crashes, processing time versus max.poll.interval.ms, heartbeat/session failures, network disruptions, deployments, and JVM pauses. Fix long blocking work and unstable instances; tune timeouts only to match legitimate processing needs. Cooperative assignment may reduce disruption where supported.


### Q54. How do you deal with a poison message?

How do you deal with a poison message?

**Answer**

Classify whether the failure is transient or permanent. Use bounded retries for transient errors; route permanent failures to a DLT/quarantine path with source metadata and alerting. Plan replay and respect ordering requirements—continuing past a failed record may not be valid for every business stream.


### Q55. How do you safely replay DLT records?

How do you safely replay DLT records?

**Answer**

Identify and fix the cause before replay. Preserve event IDs, record source metadata, and track replay attempts. Reprocess through a controlled pipeline with idempotent business effects and limits on replay rate. Do not blindly copy all records back to the original topic if ordering or duplicates can harm state.


### Q56. What is unclean leader election, and what is its trade-off?

What is unclean leader election, and what is its trade-off?

**Answer**

Unclean leader election allows a replica outside the ISR to become leader if no eligible in-sync replica is available. This can restore availability but may discard records that existed only on the former leader. Evaluate the setting against the business's tolerance for data loss.


### Q57. What if a Kafka broker's disk is almost full?

What if a Kafka broker's disk is almost full?

**Answer**

Identify which topics and partitions consume space, check retention/compaction cleanup, replica placement, ingestion growth, and segment deletion progress. Expand capacity or reduce data only according to approved retention/recovery requirements. Maintain headroom for segment rolling, reassignment, and recovery.


### Q58. How do you differentiate producer timeout from failed publication?

How do you differentiate producer timeout from failed publication?

**Answer**

A timeout means the client did not receive a result within the configured window; the broker may have written the record. Inspect delivery callback/error details, retries, producer metrics, broker logs, and event IDs. Idempotence protects qualifying retries, while business-level idempotency handles broader duplicates.


### Q59. What does NOT_LEADER_OR_FOLLOWER or LEADER_NOT_AVAILABLE suggest?

What does NOT_LEADER_OR_FOLLOWER or LEADER_NOT_AVAILABLE suggest?

**Answer**

The client may have stale metadata during leader movement or the partition may temporarily lack an available leader. Metadata refresh and retry can resolve transient cases; persistent failures require inspecting partition state, broker availability, controller health, and election logs.


### Q60. Which Kafka metrics would you monitor in production?

Which Kafka metrics would you monitor in production?

**Answer**

Monitor under-replicated/offline partitions, ISR changes, request latency and errors, network/disk/CPU, produce/fetch throughput, consumer lag and age, rebalance rate, failed/retried records, DLT volume, and storage growth. Define SLOs and alert around business impact rather than a noisy single metric.


## Phase 4 — System design (Q61–Q70)

### Section G — Distributed systems and architecture (Q61–Q70)

### Q61. Why choose event-driven architecture with Kafka?

Why choose event-driven architecture with Kafka?

**Answer**

It decouples producers and consumers, allows independent scaling, retains events for replay, and supports multiple consumers with different responsibilities. Trade-offs include eventual consistency, operational complexity, schema evolution, and duplicate/order handling.


### Q62. How do you design event contracts between microservices?

How do you design event contracts between microservices?

**Answer**

Define event names, ownership, schema, keys, timestamp semantics, event IDs, compatibility policy, and versioning expectations. Publish facts that have happened rather than commands disguised as events. Avoid leaking internal database structures into public contracts.


### Q63. Design order creation and inventory reservation.

Design order creation and inventory reservation.

**Answer**

Persist the order and outbox event transactionally; publish the event; let inventory reserve stock idempotently and emit a result. Model pending, confirmed, and rejected states explicitly. If inventory fails after order creation, use a saga/compensating action or mark the order for cancellation rather than assuming cross-service atomicity.


### Q64. How would you design payment processing with Kafka?

How would you design payment processing with Kafka?

**Answer**

Treat Kafka as event transport, not the payment ledger. Use stable payment/idempotency keys, durable state, gateway idempotency support, explicit state transitions, reconciliation, and carefully controlled retries. Avoid assuming a timeout proves the external charge failed.


### Q65. Explain the transactional outbox pattern.

Explain the transactional outbox pattern.

**Answer**

Write the business row and an outbox event in one database transaction. A relay or CDC connector publishes outbox rows to Kafka and tracks delivery. The relay can publish more than once, so consumers must deduplicate or be idempotent. Monitor outbox age and backlog.


### Q66. What is a saga?

What is a saga?

**Answer**

A saga coordinates a business workflow across services through local transactions and events/commands. On failure, it may execute compensating actions, such as refunding a payment if inventory cannot be reserved. Compensation is a new business action, not a rollback of an already committed remote transaction.


### Q67. How do you evolve schemas without breaking consumers?

How do you evolve schemas without breaking consumers?

**Answer**

Adopt compatibility rules appropriate for Avro, Protobuf, JSON Schema, or the chosen format. Add optional fields with suitable defaults where compatible; avoid removing or changing required fields without a migration plan. Deploy consumers that can handle the new shape before producers emit it when necessary.


### Q68. How do you support replay without repeating harmful side effects?

How do you support replay without repeating harmful side effects?

**Answer**

Use stable event IDs, idempotent consumers, replay-specific controls, and separate state-rebuild paths from external side effects where possible. Record replay source and maintain auditability. Define whether replay rebuilds a projection, reissues notifications, or repeats a business command.


### Q69. How would you plan multi-region Kafka disaster recovery?

How would you plan multi-region Kafka disaster recovery?

**Answer**

Define recovery-point objective and recovery-time objective, then choose replication, backup/restore, or active/passive patterns supported by the platform. Specify topic/config replication, schema/ACL restoration, offset and consumer-state strategy, failover ownership, and split-brain prevention. Test failover and failback regularly.


### Q70. How do you build a reliable audit/event history?

How do you build a reliable audit/event history?

**Answer**

Use explicit audit schemas, reliable source publication, restrictive ACLs, retention and archive policies, monitoring for gaps, and access auditing. Kafka alone does not prove legal immutability; archive to storage with suitable immutability/integrity controls when required. Avoid compaction if the full history must be retained.


## Phase 5 — Internals and advanced topics (Q71–Q90)

### Section H — Kafka internals (Q71–Q75)

### Q71. How does Kafka store records on disk?

How does Kafka store records on disk?

**Answer**

Partitions are append-only logs split into segment files. Writes are appended and offsets identify records within a partition; indexes help locate offsets and timestamps. Background retention or compaction removes eligible segments/records according to topic policy.


### Q72. What is a segment, and what are segment rolling settings?

What is a segment, and what are segment rolling settings?

**Answer**

A segment is a portion of a partition log. Brokers roll to a new segment based on size/time-related settings, and retention/cleanup generally removes or compacts eligible segments. Smaller segments can increase file/cleanup overhead; larger segments can affect cleanup granularity.


### Q73. What is the high watermark?

What is the high watermark?

**Answer**

The high watermark represents the point up to which records are considered replicated sufficiently for normal consumer visibility under Kafka's replication model. Consumers generally do not read beyond it in the standard path. Transactional read_committed consumers also have a last-stable-offset constraint.


### Q74. What is the difference between the log end offset and the last stable offset?

What is the difference between the log end offset and the last stable offset?

**Answer**

The log end offset marks the end of the current log. For transactional data, the last stable offset is the point before the first open transaction that prevents read_committed consumers from safely advancing further. A consumer can therefore be caught up to committed data while the log end offset is farther ahead.


### Q75. What happens during partition reassignment?

What happens during partition reassignment?

**Answer**

Partition reassignment moves replicas to different brokers to balance load or recover capacity. It can create extra network and disk I/O while replicas copy data. Plan concurrency/throttling, monitor progress and broker impact, and verify replication health afterward.


### Section I — Streams, Connect, CDC and schemas (Q76–Q85)

### Q76. How does Kafka Streams differ from a plain consumer?

How does Kafka Streams differ from a plain consumer?

**Answer**

Kafka Streams provides a library for transformations, joins, aggregations, windows, local state stores, and Kafka-backed changelogs. A plain consumer gives lower-level control but requires you to implement much of the state and recovery logic yourself.


### Q77. What is the difference between KStream and KTable?

What is the difference between KStream and KTable?

**Answer**

A KStream models a flow of records/events, where each record is an independent fact. A KTable models the latest value for each key, with updates replacing prior state for that key. The semantics influence joins, aggregations, and interpretation of updates.


### Q78. What are event-time windows and late-arriving events?

What are event-time windows and late-arriving events?

**Answer**

Windowed aggregation groups records by a time interval. Event time comes from the event timestamp; processing time is when the application handles it. Out-of-order or late records need a grace-period and correction policy, otherwise results may be incomplete or delayed.


### Q79. How does Kafka Streams recover local state?

How does Kafka Streams recover local state?

**Answer**

Stateful operations use local state stores and Kafka changelog topics to restore state after restart or task migration. Recovery time depends on state size, changelog volume, local disk, and restoration throughput. Monitor restore duration and ensure state/changelog retention matches the processing design.


### Q80. What is Kafka Connect?

What is Kafka Connect?

**Answer**

Kafka Connect is a framework for moving data between Kafka and external systems using source and sink connectors. It manages tasks, configurations, offsets, and connector lifecycle. Connector delivery guarantees depend on the connector and target-system semantics.


### Q81. What is CDC, and how does Debezium fit?

What is CDC, and how does Debezium fit?

**Answer**

Change data capture streams database changes from transaction logs or other supported mechanisms. Debezium provides connectors that capture changes and publish change events, often to Kafka. CDC can support outbox publication and data integration, but schema, snapshot, ordering, and recovery behaviors need configuration.


### Q82. How does Kafka Connect manage source offsets?

How does Kafka Connect manage source offsets?

**Answer**

Source connectors store progress according to connector behavior and worker configuration so they can resume after interruption. Correctness depends on source offset semantics, durability of connector configuration/offset topics, and connector capabilities. Review snapshot modes, restart behavior, and duplicate possibilities.


### Q83. What is a schema registry used for?

What is a schema registry used for?

**Answer**

A schema registry stores versioned schema definitions and can enforce compatibility policies, often using a schema ID in serialized payloads. It helps producers and consumers agree on serialization formats and schema evolution. The exact mechanism depends on registry and serializer implementation.


### Q84. What is a tombstone record?

What is a tombstone record?

**Answer**

For compacted topics, a record with a key and null value acts as a deletion marker. The tombstone is retained for a configured period so downstream state stores can observe the deletion; later compaction may remove it. Tombstones are not equivalent to an empty JSON object.


### Q85. What does exactly-once mean in Kafka Streams?

What does exactly-once mean in Kafka Streams?

**Answer**

Kafka Streams can coordinate input offsets, state-store changelog records, and output records transactionally under the configured processing guarantee. This prevents inconsistent Kafka-visible results after recovery within its documented scope. External writes to arbitrary systems still need their own consistency strategy.


### Section J — Security and operational awareness (Q86–Q90)

### Q86. How do you secure Kafka clients with TLS and SASL?

How do you secure Kafka clients with TLS and SASL?

**Answer**

TLS protects traffic in transit and can authenticate brokers; SASL mechanisms authenticate clients. Configuration must match broker listeners and trust/key material. Store credentials in a secrets manager, rotate them, and avoid logging secrets. Test certificate renewal and expiration ahead of time.


### Q87. What are Kafka ACLs?

What are Kafka ACLs?

**Answer**

Access-control lists authorize principals to perform operations on resources such as topics, consumer groups, and transactional IDs. Apply least privilege, keep producer and consumer identities distinct where practical, and review ACL changes as part of deployment.


### Q88. How should Kafka credentials and certificates be handled?

How should Kafka credentials and certificates be handled?

**Answer**

Keep secrets out of source control and ordinary application logs. Use a managed secret store, restrict who can read credentials, rotate them safely, and monitor expiry. Use different credentials for environments and services with least privilege.


### Q89. What privacy risks arise from Kafka event payloads?

What privacy risks arise from Kafka event payloads?

**Answer**

Events may replicate sensitive data into many topics, consumer groups, logs, DLTs, and archives. Minimize personal data, classify fields, restrict access, define retention/deletion obligations, encrypt where appropriate, and ensure dead-letter/replay tooling does not expose sensitive payloads broadly.


### Q90. What should a Kafka disaster-recovery plan include?

What should a Kafka disaster-recovery plan include?

**Answer**

Define RPO/RTO, backups or replication strategy, cluster provisioning, topic configs, schemas, ACLs, offsets, connector state, and consumer recovery procedures. Test recovery, replay, and failback under realistic conditions. Ensure teams know which region or cluster is authoritative during failover.


## Phase 6 — Mock interviews (Q91–Q100)

### Q91. Scenario: A consumer's DB write succeeds, but its offset commit fails. What happens?

Scenario: A consumer's DB write succeeds, but its offset commit fails. What happens?

**Answer**

On restart or reassignment, Kafka may deliver the record again because the committed position did not advance. Use a stable event ID and atomically persist a deduplication marker with the business mutation. Commit only after the DB transaction succeeds.


### Q92. Scenario: Lag starts right after a release. How do you triage?

Scenario: Lag starts right after a release. How do you triage?

**Answer**

Compare release timestamps with lag, processing latency, errors, rebalance rate, CPU/GC, DB/API latency, and per-partition metrics. Roll back if the release is the likely cause and rollback is safe. Avoid scaling consumers until you know whether the bottleneck is partition parallelism or downstream saturation.


### Q93. Scenario: Two independent producers publish the same business event. Will idempotence deduplicate them?

Scenario: Two independent producers publish the same business event. Will idempotence deduplicate them?

**Answer**

Generally no. Producer idempotence deduplicates qualifying retries within a producer identity/session, not semantically identical events emitted independently by two producers. Use a stable business event ID, source ownership, and downstream idempotency.


### Q94. Scenario: Records for one order appear out of order. What do you check?

Scenario: Records for one order appear out of order. What do you check?

**Answer**

Check whether records use the same key and partition, whether partition count or partitioner configuration changed, whether producers publish concurrently, whether retries reorder sends, and whether consumers process records asynchronously without per-key sequencing. Use sequence numbers or state-version checks if required.


### Q95. Scenario: Events arrive late and have old timestamps. How do you process them?

Scenario: Events arrive late and have old timestamps. How do you process them?

**Answer**

Determine whether the timestamp is event time or ingestion time. For analytics, use event-time windows with an explicit grace period and late-event correction strategy. For business workflows, validate state versions and define whether stale events are accepted, ignored, or sent for reconciliation.


### Q96. Scenario: There is a 100 GB backlog. How do you estimate drain time?

Scenario: There is a 100 GB backlog. How do you estimate drain time?

**Answer**

Estimate net drain rate as sustainable processing throughput minus ongoing producer throughput. If processing handles 150 MB/s and incoming traffic is 50 MB/s, net drain is about 100 MB/s, so 100 GB takes roughly 1,000 seconds under idealized steady conditions. Include partition skew, dependency capacity, and overhead in the real estimate.


### Q97. Scenario: A schema change breaks only one consumer. What is your approach?

Scenario: A schema change breaks only one consumer. What is your approach?

**Answer**

Identify that consumer's schema/serializer version, failing record, and compatibility mode. Check required fields, defaults, type changes, registry compatibility, and rollout order. Restore compatible schema behavior, then add cross-version contract tests.


### Q98. Scenario: The database is unavailable but Kafka producers are healthy. Should consumers continue?

Scenario: The database is unavailable but Kafka producers are healthy. Should consumers continue?

**Answer**

Avoid unbounded retries and memory queues. Apply bounded backoff, pause or limit consumption where appropriate, monitor lag and oldest-event age, and ensure retention can accommodate the outage. Once the database returns, restore processing gradually and watch for overload and duplicates.


### Q99. Scenario: You need to migrate a consumer group or reset offsets. What safeguards are needed?

Scenario: You need to migrate a consumer group or reset offsets. What safeguards are needed?

**Answer**

Document current group offsets, target offsets, replay range, and business impact. Test on staging, ensure idempotent effects, coordinate ownership so two consumers do not apply conflicting work, and capture metrics/audit evidence. Offset reset may skip or replay data and should be approved.


### Q100. Scenario: Interviewer asks how you lead a Kafka incident. What is your framework?

Scenario: Interviewer asks how you lead a Kafka incident. What is your framework?

**Answer**

Establish impact and data-integrity risk first. Stabilize the system with reversible changes, identify whether failures come from brokers, producers, consumers, partitioning, or downstream dependencies, and communicate status and decisions. Recover safely, reconcile critical business effects, then add tests, alerts, and runbooks based on the root cause.


## Advanced scenario-based Q&A

**Consistent answer format:** Each scenario includes the interviewer question, a detailed answer, and relevant follow-up questions with answers. Code and configuration are illustrative; verify settings against the Kafka and Spring Kafka versions used by your application.

## Mock Interview 1 — Consumer reliability

### Q101. Duplicate payment processing after a crash

Your Kafka consumer processes a payment event and updates a database. The application crashes immediately before committing the Kafka offset. On restart, Kafka delivers the same event again. How do you prevent a customer from being charged twice?

**Detailed answer**

Kafka can redeliver a record when processing succeeds but the offset is not committed. This is normal with at-least-once processing. I would solve this at the business-operation level rather than relying only on Kafka configuration.

1. Give every payment command a stable, unique `paymentId` or idempotency key.
2. Enforce uniqueness in the database with a unique constraint.
3. Process the event and persist the payment outcome in a database transaction.
4. If the same event arrives again, detect the existing payment and return the previously recorded result.
5. Commit the Kafka offset only after successful processing.

Example schema:

```sql
CREATE TABLE processed_payments (
    payment_id VARCHAR(100) PRIMARY KEY,
    status VARCHAR(30) NOT NULL
);
```

The application must atomically record the payment outcome and its idempotency key. A duplicate insert should not execute the payment operation again.

A database transaction alone cannot guarantee exactly-once behavior across Kafka, a database, and an external payment gateway. For external calls, use the gateway's idempotency mechanism where available, durable operation state, and reconciliation.

**Follow-up: Would Kafka's idempotent producer solve this?**

No. Producer idempotence prevents duplicates caused by qualifying producer retries within Kafka's supported guarantees. It does not stop a consumer from executing the same business operation twice.

**Key takeaway:** Combine Kafka delivery guarantees with idempotent business processing.

### Q102. Consumer lag increases continuously

Your consumer group processes 20,000 messages per second, but producers publish 35,000 messages per second. Lag keeps increasing. What do you do?

**Detailed answer**

First quantify backlog growth:

```text
Lag growth = 35,000 - 20,000 = 15,000 messages/second
```

At that rate, the backlog grows by about 900,000 messages per minute while both rates remain stable.

1. **Confirm the bottleneck:** Check per-partition lag, processing latency, CPU, garbage collection, database latency, downstream API latency, and consumer errors.
2. **Check partition parallelism:** A consumer group cannot actively assign more than one consumer per partition at a time.
3. **Improve processing efficiency:** Consider batching database writes, reducing unnecessary calls, and tuning fetch and serialization behavior.
4. **Scale where possible:** Add consumers up to partition parallelism, provided brokers and downstream dependencies can handle the load.
5. **Protect dependencies:** Apply bounded concurrency and backpressure.
6. **Plan recovery:** Ensure processing throughput exceeds the incoming rate.

If processing improves to 45,000 messages/sec while production remains 35,000, the backlog drains at 10,000 messages/sec.

**Follow-up: You have 12 partitions and 30 consumers. Will all 30 help?**

No. At most 12 consumers can actively own those partitions in the group. More consumers will be idle. Increasing partitions might help, but first assess key distribution, ordering requirements, broker capacity, and downstream limits.

**Key takeaway:** Scaling helps only when partition parallelism and the real bottleneck permit it.

### Q103. One partition is severely lagging

Eleven partitions are healthy, but one partition has massive lag. The consumer group has spare CPU. What is your diagnosis?

**Detailed answer**

I would suspect a hot partition or a partition-specific bottleneck. Possible causes include a highly skewed key, a slow dependency affecting that partition, unusually expensive event types, large records, uneven key distribution, or an unhealthy consumer instance.

I would inspect partition-level throughput, lag, processing duration, key frequency, message sizes, assignments, and downstream latency.

Possible remedies depend on ordering needs: optimize processing for the hot key, separate high-volume tenants or event types, redesign the partition key if ordering allows, or use key salting only when the resulting ordering changes are safe.

**Follow-up: Can you add partitions to solve it?**

Possibly, but not automatically. Existing records are not redistributed across old partitions, and the dominant key will still map to one partition under ordinary keyed partitioning. Changing partition count can also change future key-to-partition mapping.

**Key takeaway:** Diagnose key skew and ordering constraints before changing partition counts.

## Mock Interview 2 — Data consistency

### Q104. Database commit succeeds, but Kafka publish fails

An order service saves an order in its database and then publishes an `OrderCreated` event to Kafka. The database commit succeeds, but Kafka is unavailable. How do you prevent the event from being lost?

**Detailed answer**

Publishing directly after a database commit creates a dual-write problem. The database and Kafka are not part of one ordinary atomic transaction, so one operation can succeed while the other fails.

I would use the **transactional outbox pattern**:

1. Begin a database transaction.
2. Save the order.
3. Insert an outbox row describing the `OrderCreated` event in the same transaction.
4. Commit both rows atomically.
5. A publisher or CDC connector reads the outbox and publishes events to Kafka.
6. Track progress and retry transient failures.
7. Make consumers idempotent because publication may be repeated.

Example schema:

```sql
CREATE TABLE outbox_event (
    event_id VARCHAR(100) PRIMARY KEY,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload CLOB NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

If Kafka is unavailable, the event intent remains durable in the database and can be published later.

**Follow-up: Does the outbox guarantee exactly-once delivery?**

Not by itself. It guarantees that the event intent is stored atomically with the business change. Publication can still repeat, so consumers must tolerate duplicates. CDC can simplify publishing, but its recovery and delivery behavior must be validated.

**Key takeaway:** Use the outbox to solve the database/Kafka dual-write problem and idempotency to handle duplicate delivery.

### Q105. A poison message blocks a partition

One malformed event repeatedly fails deserialization. The consumer keeps retrying it, and later valid events on the same partition are not processed. What would you do?

**Detailed answer**

First distinguish transient errors from permanent failures.

- **Transient:** Temporary database outage, rate limiting, or network errors. Retry with bounded backoff.
- **Permanent:** Invalid schema, malformed payload, or an unrecoverable validation error. Route to a dead-letter topic (DLT) or quarantine mechanism.

For Spring Kafka, configure deserialization error handling, classify exceptions, set bounded retries, and publish unrecoverable records to a DLT. Retain context needed for diagnosis and safe replay: original topic, partition, offset, key, and exception details. Monitor DLT volume, protect sensitive data in logs, and define a controlled replay procedure.

**Follow-up: Should you retry a malformed payload 100 times?**

Usually not. If it is permanently invalid, retries waste resources and delay valid records. However, if strict ordering is essential, moving a failed event to a DLT and continuing may violate sequencing. The system needs an explicit policy to block, repair, or safely skip the event.

**Key takeaway:** Error handling is a business-ordering decision, not just retry configuration.

## Mock Interview 3 — Broker failures and reliability

### Q106. A Kafka broker suddenly goes down

One broker crashes during peak traffic. What happens to producers and consumers, and how do you recover?

**Detailed answer**

Kafka partitions are replicated across brokers. If the failed broker hosts a partition leader, an eligible in-sync replica (ISR) may become the new leader.

I would investigate which partitions were led by the failed broker, whether eligible replicas exist, whether producers are retrying, whether consumers see fetch failures or rebalances, and whether replication catches up after recovery.

If replication is healthy, leadership can move and clients refresh metadata. If a partition has no eligible replica, it may become unavailable. Unclean leader election can restore availability in some configurations but risks losing acknowledged data.

**Follow-up: How do you reduce impact?**

- Place replicas across separate brokers and failure domains.
- Choose an appropriate replication factor and `min.insync.replicas`.
- Configure producer acknowledgments and retries appropriately.
- Monitor under-replicated and offline partitions and ISR shrinkage.
- Test broker-failure scenarios.

**Key takeaway:** Replication improves fault tolerance, but guarantees depend on acknowledgments, ISR state, and leader-election policy.

### Q107. Producer times out. Was the message written?

A producer sends an order event, gets a timeout, and retries. How do you know whether the original message reached Kafka?

**Detailed answer**

A timeout does not necessarily mean the record was not written. The broker may have appended it, but the acknowledgment was lost or delayed.

I would enable producer idempotence where supported and configure acknowledgments and retries to meet durability requirements. Idempotence deduplicates qualifying retries within the supported producer-session guarantees.

Example Java configuration:

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer",
        "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer",
        "org.apache.kafka.common.serialization.StringSerializer");
props.put("enable.idempotence", "true");
props.put("acks", "all");
```

For deduplication across application restarts or independent producer sessions, use a stable business event ID and idempotent downstream processing.

**Follow-up: Does `acks=all` mean every replica has persisted the message?**

Not necessarily every configured replica. The leader waits for the required in-sync replicas according to cluster configuration. The guarantee depends on ISR state and related settings.

**Key takeaway:** A timeout can leave the write outcome ambiguous; protect business semantics with event IDs and idempotency.

### Q108. ISR shrinks during peak load

Monitoring shows several partitions have fewer in-sync replicas than expected. What do you do?

**Detailed answer**

ISR shrinkage indicates that replicas are falling behind or are otherwise unable to stay in sync. Causes can include disk latency, network problems, overloaded brokers, long garbage-collection pauses, or replication throttling.

I would identify affected partitions and brokers; compare leader traffic, follower fetch latency, disk I/O, CPU, network, and broker logs; inspect restarts; verify replication settings; and restore replica health.

With `acks=all` and a minimum ISR requirement, producers may receive errors when too few replicas remain in sync. This can protect durability at the cost of availability.

**Follow-up: Would you lower `min.insync.replicas` immediately?**

Not as a reflex. It can restore writes but weakens durability protection. Diagnose the underlying issue and make the availability-versus-durability trade-off explicit.

**Key takeaway:** Treat ISR shrinkage as a data durability signal, not merely a monitoring warning.

### Q109. Cluster is healthy, but producers cannot publish

Brokers are reachable, but producers report `NotEnoughReplicas` or related errors. Why?

**Detailed answer**

The cluster can be reachable while a partition lacks enough in-sync replicas to satisfy durability settings. Check `acks`, `min.insync.replicas`, current ISR size, replication health, and whether the producer targets the expected topic and cluster.

Do not disable durability checks without understanding the risk. Fix a slow follower or network problem and verify that replicas recover.

**Follow-up: Is increasing producer retries sufficient?**

No. Retries help with transient errors; they cannot fix a persistent configuration mismatch or a partition that permanently lacks enough eligible replicas.

**Key takeaway:** Diagnose ISR and topic durability configuration before changing retries.

### Q110. Consumer appears to miss data after a leader change

After broker failure and leader election, a consumer appears to have missed a recently produced event. How do you investigate?

**Detailed answer**

Establish whether the event was acknowledged and what durability guarantee applied. Inspect producer acknowledgments and errors, leader-election logs, ISR membership, unclean leader election policy, event key/partition/offset, consumer offsets, and the target cluster.

If a new leader lacked records written to the old leader, data loss can occur under configurations that permit unclean election. If the producer did not receive acknowledgment, the event's outcome may be ambiguous.

**Follow-up: How would you prevent this class of incident?**

Use appropriate replication and ISR requirements, conservative leader-election policy, stable event IDs, and reconciliation for critical records. Replication reduces risk but does not replace application-level reconciliation.

**Key takeaway:** For critical data, combine cluster durability controls with business-level audit and reconciliation.

## Mock Interview 4 — Consumer groups, offsets, and ordering

### Q111. Consumer keeps rebalancing

Your Spring Kafka consumer group repeatedly rebalances, and throughput drops. How do you troubleshoot it?

**Detailed answer**

A rebalance reassigns partition ownership. Frequent rebalances can result from unstable consumer membership, processing that exceeds `max.poll.interval.ms`, heartbeat failures, deployments, autoscaling, exceptions, crashes, or resource exhaustion.

I would inspect consumer logs, group membership, processing duration, heartbeat errors, deployment history, and JVM pauses. Then make processing time bounded, tune `max.poll.interval.ms` only when justified, adjust `max.poll.records`, avoid long blocking calls on the consumer thread, and address crashes or deployment churn. Consider cooperative rebalancing where supported.

**Follow-up: Should you increase `max.poll.interval.ms` to one hour?**

Not automatically. This may hide inefficient processing and delay detection of a stuck consumer. First identify why processing takes so long.

**Key takeaway:** Fix instability and blocking work before masking it with larger timeouts.

### Q112. Consumer commits offsets before processing finishes

A consumer commits offset 100, then crashes before processing record 100 successfully. What happens?

**Detailed answer**

If the committed position advances beyond an unprocessed record, a restarted consumer can skip that record. For at-least-once processing, commit only after successful processing.

Illustrative Java pattern:

```java
consumer.subscribe(List.of("orders"));

while (running) {
    ConsumerRecords<String, String> records =
            consumer.poll(Duration.ofMillis(500));

    for (ConsumerRecord<String, String> record : records) {
        process(record); // Must succeed before committing
    }

    consumer.commitSync();
}
```

This is illustrative, not a complete production consumer. Real code must handle exceptions, shutdown, rebalances, and any records that remain unprocessed.

**Follow-up: What if processing succeeds but the commit fails?**

The record may be processed again after restart. Idempotent business operations make that safe.

**Key takeaway:** Commit positions should reflect completed work, and processing should tolerate redelivery.

### Q113. Parallel processing while preserving per-customer ordering

A topic contains events for thousands of customers. You need higher throughput but must preserve order for each customer.

**Detailed answer**

Use customer ID as the record key so events for the same customer normally map to the same partition. Kafka guarantees ordering within a partition, not across the whole topic. Consumer code must also preserve order; dispatching records to arbitrary asynchronous workers may reorder their effects.

Example:

```java
ProducerRecord<String, String> record =
        new ProducerRecord<>(
                "customer-events",
                customerId,
                eventJson
        );
```

**Follow-up: What if one customer generates 40% of traffic?**

That customer becomes a hot key. Splitting that key across partitions improves parallelism only if strict per-customer ordering can be relaxed or enforced through a different sequencing architecture.

**Key takeaway:** Partitioning determines ordering boundaries; application concurrency must respect them.

### Q114. Consumer restarts and processes a large backlog

A consumer was offline for six hours. On restart it processes old events, and the application appears stale. What do you do?

**Detailed answer**

Distinguish delayed processing from genuinely stale business state. Kafka retains records according to topic retention; it does not guarantee that consumers only see current state.

Inspect consumer lag and oldest unprocessed event age, retention, throughput, timestamps and event-time semantics, and whether processing rebuilds state or triggers side effects. Define explicit stale-event policies, ensure replay does not overwhelm dependencies, and make operations idempotent.

**Follow-up: Should you reset offsets to the latest position?**

Only if the business explicitly accepts skipping pending records. Resetting offsets can cause data loss and is not a generic lag fix.

**Key takeaway:** Offset resets are business data decisions, not routine operational cleanup.

### Q115. More consumers than partitions

You have 8 partitions and 20 consumers. Why does adding consumers not improve throughput?

**Detailed answer**

In one consumer group, a partition is assigned to at most one consumer at a time. With 8 partitions, at most 8 consumers can actively own partitions in that group.

Measure whether processing is CPU-bound, I/O-bound, or blocked downstream. More partitions may create more parallelism, but assess broker overhead, ordering, key distribution, and downstream capacity.

**Follow-up: Can you just double partition count?**

Not without a capacity and compatibility review. More partitions increase metadata, file, replication, and operational overhead; changing partition count can also change key-to-partition mapping.

**Key takeaway:** Scale partitioning based on measured throughput and ordering needs.

## Mock Interview 5 — Retry architecture and delivery guarantees

### Q116. Retrying breaks event ordering

An `OrderCreated` event fails and is sent to a retry topic. Meanwhile, `OrderCancelled` is processed successfully. Events for the same order are now out of sequence. How do you solve it?

**Detailed answer**

Retry topics can allow later events to overtake an earlier failed event. Choose retry strategy based on ordering needs:

- **Strict ordering:** Pause/block processing for the affected partition or key until resolution.
- **Bounded in-place retry:** Retry briefly with backoff before moving to a terminal path.
- **Per-entity sequencing:** Track event sequence numbers or aggregate versions.
- **Relaxed ordering:** Use retry topics only when reordering is acceptable and consumers can reconcile state.

For order workflows, enforce valid state transitions so an out-of-order event cannot move an order into an invalid state.

**Follow-up: Is a DLT always the best answer?**

No. A DLT isolates failures but does not solve ordering automatically. The business must define what happens when an event cannot be processed.

**Key takeaway:** Retry architecture must respect business sequence and state transitions.

### Q117. Exactly-once processing across Kafka and a database

A requirement says: “Every Kafka event must update our relational database exactly once.” How would you design this?

**Detailed answer**

Clarify whether “exactly once” means that the final business effect must not be applied twice. An idempotent database transaction is often the practical approach.

1. Include a stable event ID.
2. Maintain a processed-event table with a unique constraint.
3. In one database transaction, claim the event ID and apply the business update.
4. Roll back both if the business update fails.
5. Commit the Kafka offset only after the database transaction succeeds.
6. Treat duplicate delivery as a no-op.

Conceptual flow:

```text
BEGIN DATABASE TRANSACTION
  Insert event_id into processed_events
  Apply business update
COMMIT DATABASE TRANSACTION
Commit Kafka offset
```

If the application crashes after the database commit but before offset commit, redelivery happens and the unique event ID prevents a second business effect.

**Follow-up: Does Kafka's transactional producer solve this automatically?**

No. Kafka transactions can atomically coordinate Kafka output records and consumed offsets within Kafka's transactional model. They do not automatically enlist a relational database in the same atomic transaction.

**Key takeaway:** Define the exactly-once boundary explicitly and use idempotency across system boundaries.

### Q118. Kafka transaction aborts after producing records

A transactional producer writes several records and then aborts. What should consumers observe?

**Detailed answer**

Kafka transactions allow multiple records to be committed or aborted as one transaction. Consumers configured with `isolation.level=read_committed` do not return aborted transactional records as committed application data.

Aborted records may still physically exist in the log; transaction markers and the last stable offset affect what such consumers can read. Verify transactional ID usage, transaction lifecycle, error handling, and consumer isolation level.

**Follow-up: Does `read_committed` eliminate all duplicates?**

No. It protects transactional visibility in Kafka. It does not automatically deduplicate repeated business events produced outside that transaction or repeated side effects in external systems.

**Key takeaway:** Kafka transactional visibility is not global exactly-once behavior across every system.

### Q119. Producer gets fenced by another instance

A transactional producer suddenly receives a producer-fencing exception. What does it mean?

**Detailed answer**

A producer using a transactional ID can be fenced when another producer instance with the same transactional identity becomes active with a newer producer epoch. This prevents stale producers from continuing to write.

Check whether concurrent application instances accidentally share a transactional ID, whether a restarted instance took over, and whether deployment/autoscaling caused overlap. Correct the transactional identity lifecycle rather than retrying indefinitely.

**Follow-up: Should every pod use the exact same transactional ID?**

No. Concurrently active producers generally need distinct transactional IDs. Identity should be stable across restarts for the same logical producer when takeover semantics are needed, while avoiding collisions between concurrent producers.

**Key takeaway:** Transactional ID design is part of correctness and deployment safety.

### Q120. Duplicates appear even with idempotent producers

The producer has `enable.idempotence=true`, but the business reports duplicate events. How do you investigate?

**Detailed answer**

Producer idempotence protects against qualifying retries within its supported producer-session scope. It does not globally deduplicate business events.

Investigate duplicate submissions by application code, multiple producers publishing the same business event, outbox republishing, upstream retries, consumer redelivery, and replay workflows. Use stable business event IDs and trace events across service boundaries.

**Follow-up: Would deduplicating on payload solve it?**

Not reliably. Legitimate events can have identical payloads. Use a stable event ID or business identifier with a clearly defined uniqueness scope.

**Key takeaway:** Producer idempotence and business-level deduplication solve different problems.

## Mock Interview 6 — Java and Spring Kafka

### Q121. Spring Kafka listener blocks too long

Your `@KafkaListener` calls a slow REST API; some calls take two minutes, and the group keeps rebalancing. How would you redesign it?

**Detailed answer**

Long processing can exceed `max.poll.interval.ms`, causing the consumer to be considered unresponsive and triggering a rebalance. Measure REST latency and decide whether synchronous processing is essential.

Improve connection/read timeouts, use bounded retries and backoff, limit records processed per poll, tune `max.poll.interval.ms` to match legitimate workload, and consider asynchronous processing only with explicit offset, concurrency, and ordering control. A durable work queue or separate processing stage may be appropriate for long and variable external calls.

```java
@KafkaListener(
    topics = "orders",
    groupId = "order-service"
)
public void consume(OrderEvent event) {
    orderService.process(event);
}
```

Do not swallow exceptions that should trigger error handling. Acknowledgment mode and the container error handler must match the delivery semantics.

**Follow-up: Can you call `CompletableFuture` and immediately return?**

Not safely by itself. If the offset advances before asynchronous work completes, a crash can lose the operation. Coordinate completion, offsets, concurrency, and ordering.

**Key takeaway:** Async execution needs explicit offset and failure semantics.

### Q122. Database connection pool is exhausted

After increasing listener concurrency from 5 to 30, database timeouts rise sharply. Why?

**Detailed answer**

Kafka concurrency increases parallel processing and concurrent database operations. The pool or database itself may now be the bottleneck.

Inspect HikariCP active/idle/pending metrics, database CPU and lock waits, query latency, connection limits, listener concurrency, batch size, and transaction duration. Set concurrency based on downstream capacity, optimize queries, use bounded batching, and apply backpressure.

**Follow-up: Should you increase the connection pool to 100?**

Only if the database supports the additional concurrency. A larger pool can increase contention. Establish a sustainable concurrency budget across all application replicas.

**Key takeaway:** Scale consumers and database resources together, using measurements rather than guesswork.

### Q123. Batch listener partially fails

A batch has 100 records. The first 70 succeed, record 71 fails, and the remaining records are not processed. How do you avoid unnecessary duplicate work?

**Detailed answer**

Batch recovery depends on listener acknowledgment mode, container configuration, and error handler. Determine whether the batch is processed in one database transaction and whether all operations can be rolled back together.

Options include atomic database batch transactions, record-level handling for precise failure isolation, Spring Kafka batch error handling when supported by the configuration, and idempotent successful operations so replay is safe.

**Follow-up: Is a Kafka batch the same as a database transaction?**

No. Polling a batch does not make database operations atomic. The database transaction boundary must be designed separately.

**Key takeaway:** Define failure granularity and transaction boundaries explicitly.

### Q124. Kafka consumer integration tests are flaky

Tests sometimes fail because the consumer has not processed the message before the assertion runs. How do you make tests reliable?

**Detailed answer**

Asynchronous processing should be tested with deterministic synchronization rather than fixed sleeps.

1. Start a real test broker or supported Kafka test environment.
2. Publish a uniquely identified event.
3. Wait for the expected result with a bounded condition or synchronization primitive.
4. Assert on database state or an emitted event.
5. Clean up test data and resources.

Also test duplicate delivery, retry exhaustion, invalid payloads, consumer restarts, and offset behavior.

**Follow-up: Should every test use a real Kafka broker?**

No. Unit tests can mock application boundaries; integration tests should use a real or compatible broker to validate Kafka-specific behavior.

**Key takeaway:** Use unit tests for logic and integration tests for broker behavior.

### Q125. A new event schema breaks consumers

A producer deploys a new event schema, and consumers start throwing deserialization errors. How do you prevent this?

**Detailed answer**

Treat schema evolution as a contract between independently deployed services. Use compatibility rules appropriate for the serialization format and registry, prefer backward-compatible changes, avoid removing or renaming required fields without a migration plan, and test old-producer/new-consumer as well as new-producer/old-consumer combinations.

When using Avro, Protobuf, or JSON Schema with a registry, enforce compatibility in CI and at schema registration. Route invalid records to controlled failure handling.

**Follow-up: Does adding a field always preserve compatibility?**

No. It depends on the format, default, requiredness, and compatibility mode. An optional field with a suitable default is often safer than a newly required field, but the actual schema rules must be checked.

**Key takeaway:** Schema compatibility must be tested as part of deployment, not after consumers fail.

## Mock Interview 7 — System design and advanced architecture

### Q126. Design an order-processing platform using Kafka

Design an order-processing system for high traffic that supports multiple downstream services and avoids losing order events.

**Detailed answer**

Separate independently scalable services and keep ownership of each domain's data clear.

```text
Client
  |
Order Service
  |
Database Transaction
  |-- Save Order
  |-- Save Outbox Event
          |
      Outbox Publisher / CDC
          |
      Kafka: order-events
       /       |        \\
 Inventory   Payment   Notification
 Service     Service     Service
```

Key decisions:

- Store the order and outbox event in one database transaction.
- Use `orderId` as the key if per-order ordering is required.
- Expect at-least-once delivery and make consumers idempotent.
- Use bounded retries and a DLT/quarantine strategy for terminal failures.
- Keep each service's database as the source of truth for its domain.
- Monitor lag, end-to-end latency, failure rates, DLT volume, and outbox backlog.
- Support controlled replay and reconciliation.

A saga may coordinate steps across services. If payment succeeds but inventory reservation fails, the workflow may need a refund or other compensating action.

**Follow-up: Would you put the entire workflow in one Kafka transaction?**

No. Kafka transactions do not automatically create an atomic transaction across multiple service databases and external payment systems. A saga, idempotent operations, durable state transitions, and compensation are more suitable.

**Key takeaway:** Kafka transports events; business workflows still need explicit consistency and compensation design.

### Q127. Design real-time fraud detection

A financial platform must detect suspicious transactions within seconds. How would you use Kafka?

**Detailed answer**

Publish transaction events to Kafka and build a stream-processing pipeline that evaluates rules and maintains state.

```text
Transaction Service
       |
  Kafka: transactions
       |
  Stream Processor
       |-- Velocity checks
       |-- Customer/device history
       |-- Risk scoring
       |
  Kafka: fraud-alerts
       |
  Alert / Review Service
```

Choose Kafka Streams or another stream-processing engine based on the platform and state-processing needs. Partition by the key required for state locality, such as customer ID. Define event-time processing, late-event behavior, latency objectives, acceptable false-positive rates, stable transaction IDs, and monitoring for lag, latency, and state-store health.

**Follow-up: What if events for one customer arrive out of order?**

Use event timestamps and explicit event-time semantics where appropriate. Define late-event handling, window grace periods, and correction policies. Event time is not automatically the same as processing time.

**Key takeaway:** Stateful stream processing requires explicit choices about keys, windows, time semantics, and late data.

### Q128. Design a Kafka-based audit logging system

The organization needs a tamper-resistant audit trail for important business actions. How would you design it?

**Detailed answer**

Kafka can transport and retain audit events, but it does not by itself guarantee a complete, legally compliant, immutable archive.

Design around a defined audit schema (actor, action, resource, timestamp, correlation ID, outcome), reliable event creation through an outbox or equivalent durable mechanism, restricted producer/consumer permissions, encryption, explicit retention and archival, monitoring for publication failures and missing sequences, and controlled audited export.

For stronger tamper evidence, archive to storage with immutability controls and integrity verification, such as signed manifests or cryptographic hashes.

**Follow-up: Is log compaction suitable for audit records?**

Usually not for a full audit trail. Compaction retains the latest value per key and may remove older values for the same key. Use retention and archival policies that preserve required history.

**Key takeaway:** Audit completeness and immutability need explicit controls beyond Kafka topic settings.

### Q129. Kafka storage grows faster than expected

Kafka disk usage doubled in a week. Producers are healthy, and consumer lag is low. How do you investigate?

**Detailed answer**

Low lag does not imply low storage usage because Kafka retains records according to topic policies, not simply until consumers read them.

Inspect topic-level storage, `retention.ms`, `retention.bytes`, production rate, payload size, replication overhead, compression, compaction behavior, segment cleanup, and recovery/replay requirements.

Example rough calculation: if a topic receives 100 GB of uncompressed data daily and retains about seven days, retained primary log data is roughly 700 GB before replication, compression, indexes, and headroom. With replication factor 3, replicated storage can be roughly three times retained log data, subject to those factors.

**Follow-up: Can you reduce retention from seven days to one day immediately?**

Only after confirming recovery and replay requirements. Retention changes can make records unavailable sooner than expected. Ensure enough free disk for traffic spikes and maintenance.

**Key takeaway:** Plan for retention, replication, compression, recovery, and disk headroom—not just lag.

### Q130. Major production incident during a release

After a deployment, message lag rises quickly, duplicate business operations appear, and consumers repeatedly rebalance. You lead the incident. What do you do?

**Detailed answer**

Prioritize stabilizing the system and protecting data before making multiple configuration changes.

**1. Establish impact**
- Identify affected topics, groups, services, and business operations.
- Measure lag growth, throughput, error and duplicate rates, and end-to-end latency.
- Determine whether messages are lost, delayed, duplicated, or just slow.

**2. Contain**
- Roll back the suspected deployment when evidence connects it to the incident.
- Avoid indiscriminately increasing concurrency if dependencies are overloaded.
- Protect critical operations with idempotency checks and rate limits where appropriate.
- Preserve logs, metrics, and relevant offset information.

**3. Diagnose**
Compare the release with the prior version and inspect listener duration, exceptions, poll intervals, rebalance logs, connection pool saturation, producer retry behavior, partition lag, hot keys, schema changes, and acknowledgment/retry configuration.

A common chain is that slower database calls increase listener duration, trigger rebalances, and cause previously processed but uncommitted records to be redelivered.

**4. Recover safely**
- Restore stable consumer membership and downstream capacity.
- Confirm consumers process the intended offsets.
- Let backlog drain at a sustainable rate.
- Reconcile duplicate business effects using event IDs and durable state.
- Verify no critical events were lost or skipped.

**5. Prevent recurrence**
- Add regression tests for slow processing, retries, rebalances, and duplicate delivery.
- Alert on lag growth rate, oldest-message age, rebalance frequency, and downstream saturation.
- Improve rollout safeguards and rollback criteria.
- Document root cause, contributing factors, and recovery procedures.

**Follow-up: Management asks you to reset all consumer offsets to latest to clear lag.**

Do not do this without explicit business approval. Moving offsets to latest can skip unprocessed records and cause silent data loss. Quantify the backlog, identify its cause, and agree on a recovery strategy that preserves required business events.

**Key takeaway:** Incident leadership means stabilizing service, protecting integrity, restoring operation, and proving recovery—not merely making the lag graph smaller.

---

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
