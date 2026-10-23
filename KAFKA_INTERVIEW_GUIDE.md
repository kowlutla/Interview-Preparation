# Apache Kafka Interview Preparation Guide

> **Audience:** Senior Software Engineer / Senior Java Backend Engineer  
> **Goal:** Prepare for Kafka interviews covering fundamentals, Java/Spring Kafka, delivery semantics, performance, production troubleshooting, architecture, and scenario-based system design.

## Important note about coverage

This guide includes the 30 scenario-based Q&As fully present in the conversation, plus a structured six-phase outline for the original 100-question set. The exact full text of the original 100 answers was not available in the current context, so this file does **not** claim to reproduce those 100 answers verbatim. Expand the outline sections with the original Q1–Q100 answer text when it is available.

## Table of contents

- [Study plan](#study-plan)
- [Original 100-question guide structure](#original-100-question-guide-structure)
- [Advanced scenario-based Q&A](#advanced-scenario-based-qa)
  - [Mock Interview 1 — Consumer reliability](#mock-interview-1--consumer-reliability)
  - [Mock Interview 2 — Data consistency](#mock-interview-2--data-consistency)
  - [Mock Interview 3 — Broker failures and reliability](#mock-interview-3--broker-failures-and-reliability)
  - [Mock Interview 4 — Consumer groups, offsets, and ordering](#mock-interview-4--consumer-groups-offsets-and-ordering)
  - [Mock Interview 5 — Retry architecture and delivery guarantees](#mock-interview-5--retry-architecture-and-delivery-guarantees)
  - [Mock Interview 6 — Java and Spring Kafka](#mock-interview-6--java-and-spring-kafka)
  - [Mock Interview 7 — System design and advanced architecture](#mock-interview-7--system-design-and-advanced-architecture)
- [References](#references)

## Study plan

| Phase | Focus | Original question range | Suggested focus |
|---|---|---:|---|
| 1 | Core concepts | Q1–Q20 | Architecture, partitions, producers, consumers |
| 2 | Delivery guarantees and Java | Q21–Q40 | Delivery semantics, Spring Kafka, transactions |
| 3 | Performance and production incidents | Q41–Q60 | Capacity, lag, operational diagnosis |
| 4 | System design | Q61–Q70 | Event-driven systems and architecture choices |
| 5 | Internals and advanced topics | Q71–Q90 | KRaft, Streams, Connect, CDC, schemas, security |
| 6 | Mock interviews | Q91–Q100 | Senior-level scenarios and trade-offs |
| Additional | Scenario-based interviews | Q101–Q130 | 30 real-world scenarios with follow-ups |

## Original 100-question guide structure

The following is the section structure established in the conversation. The full original answers should be added from the source conversation/export to preserve them exactly.

### Phase 1 — Core concepts (Days 1–3)

- **Section A — Kafka fundamentals and internals:** Q1–Q10
- **Section B — Producers and consumers:** Q11–Q20

Topics: brokers, topics, partitions, offsets, replication, leader election, consumer groups, retention, compaction, key-based partitioning, ordering, and producer/consumer internals.

### Phase 2 — Delivery guarantees and Java (Days 4–7)

- **Section C — Delivery guarantees and consistency:** Q21–Q30
- **Section D — Java and Spring Kafka:** Q31–Q40

Topics: at-most-once, at-least-once, exactly-once scope, idempotent producers, offset commits, transactions, retry/DLT strategies, Java client behavior, listener concurrency, tests, and Spring configuration.

### Phase 3 — Performance and production incidents (Days 8–11)

- **Section E — Performance and capacity planning:** Q41–Q50
- **Section F — Production troubleshooting and reliability:** Q51–Q60

Topics: throughput, batching, compression, consumer lag, hot partitions, backpressure, broker health, ISR, rebalances, metrics, and incident response.

### Phase 4 — System design (Days 12–14)

- **Section G — Distributed systems and architecture:** Q61–Q70

Topics: event-driven architecture, service boundaries, ordering, database/Kafka consistency, outbox, sagas, replay, and resilience.

### Phase 5 — Internals and advanced topics (Days 15–17)

- **Section H — Kafka internals worth understanding:** Q71–Q75
- **Section I — Streams, Connect, CDC and schemas:** Q76–Q85
- **Section J — Security and operational awareness:** Q86–Q90

Topics: KRaft, log segments, replication internals, Kafka Streams, Kafka Connect, Debezium, schema evolution, authentication, authorization, encryption, and operational safeguards.

### Phase 6 — Mock interviews (Days 18–20)

- **Section K — Difficult scenario questions:** Q91–Q100

Topics: incident diagnosis, event-driven design, payments, fraud detection, auditing, capacity decisions, trade-offs, and recovery.

---

# Advanced scenario-based Q&A

**Consistent answer format:** Each scenario includes the interviewer question, a detailed answer, and relevant follow-up questions with answers. Code and configuration are illustrative; verify settings against the Kafka and Spring Kafka versions used by your application.

## Mock Interview 1 — Consumer reliability

### Q101. Duplicate payment processing after a crash

**Question**

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

**Question**

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

**Question**

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

**Question**

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

**Question**

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

**Question**

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

**Question**

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

**Question**

Monitoring shows several partitions have fewer in-sync replicas than expected. What do you do?

**Detailed answer**

ISR shrinkage indicates that replicas are falling behind or are otherwise unable to stay in sync. Causes can include disk latency, network problems, overloaded brokers, long garbage-collection pauses, or replication throttling.

I would identify affected partitions and brokers; compare leader traffic, follower fetch latency, disk I/O, CPU, network, and broker logs; inspect restarts; verify replication settings; and restore replica health.

With `acks=all` and a minimum ISR requirement, producers may receive errors when too few replicas remain in sync. This can protect durability at the cost of availability.

**Follow-up: Would you lower `min.insync.replicas` immediately?**

Not as a reflex. It can restore writes but weakens durability protection. Diagnose the underlying issue and make the availability-versus-durability trade-off explicit.

**Key takeaway:** Treat ISR shrinkage as a data durability signal, not merely a monitoring warning.

### Q109. Cluster is healthy, but producers cannot publish

**Question**

Brokers are reachable, but producers report `NotEnoughReplicas` or related errors. Why?

**Detailed answer**

The cluster can be reachable while a partition lacks enough in-sync replicas to satisfy durability settings. Check `acks`, `min.insync.replicas`, current ISR size, replication health, and whether the producer targets the expected topic and cluster.

Do not disable durability checks without understanding the risk. Fix a slow follower or network problem and verify that replicas recover.

**Follow-up: Is increasing producer retries sufficient?**

No. Retries help with transient errors; they cannot fix a persistent configuration mismatch or a partition that permanently lacks enough eligible replicas.

**Key takeaway:** Diagnose ISR and topic durability configuration before changing retries.

### Q110. Consumer appears to miss data after a leader change

**Question**

After broker failure and leader election, a consumer appears to have missed a recently produced event. How do you investigate?

**Detailed answer**

Establish whether the event was acknowledged and what durability guarantee applied. Inspect producer acknowledgments and errors, leader-election logs, ISR membership, unclean leader election policy, event key/partition/offset, consumer offsets, and the target cluster.

If a new leader lacked records written to the old leader, data loss can occur under configurations that permit unclean election. If the producer did not receive acknowledgment, the event's outcome may be ambiguous.

**Follow-up: How would you prevent this class of incident?**

Use appropriate replication and ISR requirements, conservative leader-election policy, stable event IDs, and reconciliation for critical records. Replication reduces risk but does not replace application-level reconciliation.

**Key takeaway:** For critical data, combine cluster durability controls with business-level audit and reconciliation.

## Mock Interview 4 — Consumer groups, offsets, and ordering

### Q111. Consumer keeps rebalancing

**Question**

Your Spring Kafka consumer group repeatedly rebalances, and throughput drops. How do you troubleshoot it?

**Detailed answer**

A rebalance reassigns partition ownership. Frequent rebalances can result from unstable consumer membership, processing that exceeds `max.poll.interval.ms`, heartbeat failures, deployments, autoscaling, exceptions, crashes, or resource exhaustion.

I would inspect consumer logs, group membership, processing duration, heartbeat errors, deployment history, and JVM pauses. Then make processing time bounded, tune `max.poll.interval.ms` only when justified, adjust `max.poll.records`, avoid long blocking calls on the consumer thread, and address crashes or deployment churn. Consider cooperative rebalancing where supported.

**Follow-up: Should you increase `max.poll.interval.ms` to one hour?**

Not automatically. This may hide inefficient processing and delay detection of a stuck consumer. First identify why processing takes so long.

**Key takeaway:** Fix instability and blocking work before masking it with larger timeouts.

### Q112. Consumer commits offsets before processing finishes

**Question**

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

**Question**

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

**Question**

A consumer was offline for six hours. On restart it processes old events, and the application appears stale. What do you do?

**Detailed answer**

Distinguish delayed processing from genuinely stale business state. Kafka retains records according to topic retention; it does not guarantee that consumers only see current state.

Inspect consumer lag and oldest unprocessed event age, retention, throughput, timestamps and event-time semantics, and whether processing rebuilds state or triggers side effects. Define explicit stale-event policies, ensure replay does not overwhelm dependencies, and make operations idempotent.

**Follow-up: Should you reset offsets to the latest position?**

Only if the business explicitly accepts skipping pending records. Resetting offsets can cause data loss and is not a generic lag fix.

**Key takeaway:** Offset resets are business data decisions, not routine operational cleanup.

### Q115. More consumers than partitions

**Question**

You have 8 partitions and 20 consumers. Why does adding consumers not improve throughput?

**Detailed answer**

In one consumer group, a partition is assigned to at most one consumer at a time. With 8 partitions, at most 8 consumers can actively own partitions in that group.

Measure whether processing is CPU-bound, I/O-bound, or blocked downstream. More partitions may create more parallelism, but assess broker overhead, ordering, key distribution, and downstream capacity.

**Follow-up: Can you just double partition count?**

Not without a capacity and compatibility review. More partitions increase metadata, file, replication, and operational overhead; changing partition count can also change key-to-partition mapping.

**Key takeaway:** Scale partitioning based on measured throughput and ordering needs.

## Mock Interview 5 — Retry architecture and delivery guarantees

### Q116. Retrying breaks event ordering

**Question**

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

**Question**

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

**Question**

A transactional producer writes several records and then aborts. What should consumers observe?

**Detailed answer**

Kafka transactions allow multiple records to be committed or aborted as one transaction. Consumers configured with `isolation.level=read_committed` do not return aborted transactional records as committed application data.

Aborted records may still physically exist in the log; transaction markers and the last stable offset affect what such consumers can read. Verify transactional ID usage, transaction lifecycle, error handling, and consumer isolation level.

**Follow-up: Does `read_committed` eliminate all duplicates?**

No. It protects transactional visibility in Kafka. It does not automatically deduplicate repeated business events produced outside that transaction or repeated side effects in external systems.

**Key takeaway:** Kafka transactional visibility is not global exactly-once behavior across every system.

### Q119. Producer gets fenced by another instance

**Question**

A transactional producer suddenly receives a producer-fencing exception. What does it mean?

**Detailed answer**

A producer using a transactional ID can be fenced when another producer instance with the same transactional identity becomes active with a newer producer epoch. This prevents stale producers from continuing to write.

Check whether concurrent application instances accidentally share a transactional ID, whether a restarted instance took over, and whether deployment/autoscaling caused overlap. Correct the transactional identity lifecycle rather than retrying indefinitely.

**Follow-up: Should every pod use the exact same transactional ID?**

No. Concurrently active producers generally need distinct transactional IDs. Identity should be stable across restarts for the same logical producer when takeover semantics are needed, while avoiding collisions between concurrent producers.

**Key takeaway:** Transactional ID design is part of correctness and deployment safety.

### Q120. Duplicates appear even with idempotent producers

**Question**

The producer has `enable.idempotence=true`, but the business reports duplicate events. How do you investigate?

**Detailed answer**

Producer idempotence protects against qualifying retries within its supported producer-session scope. It does not globally deduplicate business events.

Investigate duplicate submissions by application code, multiple producers publishing the same business event, outbox republishing, upstream retries, consumer redelivery, and replay workflows. Use stable business event IDs and trace events across service boundaries.

**Follow-up: Would deduplicating on payload solve it?**

Not reliably. Legitimate events can have identical payloads. Use a stable event ID or business identifier with a clearly defined uniqueness scope.

**Key takeaway:** Producer idempotence and business-level deduplication solve different problems.

## Mock Interview 6 — Java and Spring Kafka

### Q121. Spring Kafka listener blocks too long

**Question**

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

**Question**

After increasing listener concurrency from 5 to 30, database timeouts rise sharply. Why?

**Detailed answer**

Kafka concurrency increases parallel processing and concurrent database operations. The pool or database itself may now be the bottleneck.

Inspect HikariCP active/idle/pending metrics, database CPU and lock waits, query latency, connection limits, listener concurrency, batch size, and transaction duration. Set concurrency based on downstream capacity, optimize queries, use bounded batching, and apply backpressure.

**Follow-up: Should you increase the connection pool to 100?**

Only if the database supports the additional concurrency. A larger pool can increase contention. Establish a sustainable concurrency budget across all application replicas.

**Key takeaway:** Scale consumers and database resources together, using measurements rather than guesswork.

### Q123. Batch listener partially fails

**Question**

A batch has 100 records. The first 70 succeed, record 71 fails, and the remaining records are not processed. How do you avoid unnecessary duplicate work?

**Detailed answer**

Batch recovery depends on listener acknowledgment mode, container configuration, and error handler. Determine whether the batch is processed in one database transaction and whether all operations can be rolled back together.

Options include atomic database batch transactions, record-level handling for precise failure isolation, Spring Kafka batch error handling when supported by the configuration, and idempotent successful operations so replay is safe.

**Follow-up: Is a Kafka batch the same as a database transaction?**

No. Polling a batch does not make database operations atomic. The database transaction boundary must be designed separately.

**Key takeaway:** Define failure granularity and transaction boundaries explicitly.

### Q124. Kafka consumer integration tests are flaky

**Question**

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

**Question**

A producer deploys a new event schema, and consumers start throwing deserialization errors. How do you prevent this?

**Detailed answer**

Treat schema evolution as a contract between independently deployed services. Use compatibility rules appropriate for the serialization format and registry, prefer backward-compatible changes, avoid removing or renaming required fields without a migration plan, and test old-producer/new-consumer as well as new-producer/old-consumer combinations.

When using Avro, Protobuf, or JSON Schema with a registry, enforce compatibility in CI and at schema registration. Route invalid records to controlled failure handling.

**Follow-up: Does adding a field always preserve compatibility?**

No. It depends on the format, default, requiredness, and compatibility mode. An optional field with a suitable default is often safer than a newly required field, but the actual schema rules must be checked.

**Key takeaway:** Schema compatibility must be tested as part of deployment, not after consumers fail.

## Mock Interview 7 — System design and advanced architecture

### Q126. Design an order-processing platform using Kafka

**Question**

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

**Question**

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

**Question**

The organization needs a tamper-resistant audit trail for important business actions. How would you design it?

**Detailed answer**

Kafka can transport and retain audit events, but it does not by itself guarantee a complete, legally compliant, immutable archive.

Design around a defined audit schema (actor, action, resource, timestamp, correlation ID, outcome), reliable event creation through an outbox or equivalent durable mechanism, restricted producer/consumer permissions, encryption, explicit retention and archival, monitoring for publication failures and missing sequences, and controlled audited export.

For stronger tamper evidence, archive to storage with immutability controls and integrity verification, such as signed manifests or cryptographic hashes.

**Follow-up: Is log compaction suitable for audit records?**

Usually not for a full audit trail. Compaction retains the latest value per key and may remove older values for the same key. Use retention and archival policies that preserve required history.

**Key takeaway:** Audit completeness and immutability need explicit controls beyond Kafka topic settings.

### Q129. Kafka storage grows faster than expected

**Question**

Kafka disk usage doubled in a week. Producers are healthy, and consumer lag is low. How do you investigate?

**Detailed answer**

Low lag does not imply low storage usage because Kafka retains records according to topic policies, not simply until consumers read them.

Inspect topic-level storage, `retention.ms`, `retention.bytes`, production rate, payload size, replication overhead, compression, compaction behavior, segment cleanup, and recovery/replay requirements.

Example rough calculation: if a topic receives 100 GB of uncompressed data daily and retains about seven days, retained primary log data is roughly 700 GB before replication, compression, indexes, and headroom. With replication factor 3, replicated storage can be roughly three times retained log data, subject to those factors.

**Follow-up: Can you reduce retention from seven days to one day immediately?**

Only after confirming recovery and replay requirements. Retention changes can make records unavailable sooner than expected. Ensure enough free disk for traffic spikes and maintenance.

**Key takeaway:** Plan for retention, replication, compression, recovery, and disk headroom—not just lag.

### Q130. Major production incident during a release

**Question**

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
