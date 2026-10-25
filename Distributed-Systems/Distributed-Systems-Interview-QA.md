# Distributed Systems & Event-Driven Architecture Interview Handbook

**Audience:** Java backend engineers with 4–6 years of experience preparing for Senior Software Engineer and Backend Engineer interviews.  
**Purpose:** A practical, interview-first guide to distributed correctness, failure handling, consistency, coordination, asynchronous messaging, and production operations.

The answers are written to support both understanding and spoken interview responses. Examples are illustrative pseudocode unless explicitly stated otherwise. Prefer the simplest design that meets the business invariant; distributed patterns add costs and are not goals in themselves.

## Contents

1. [Distributed systems fundamentals](#1-distributed-systems-fundamentals)
2. [Time, ordering, and coordination](#2-time-ordering-and-coordination)
3. [Consistency models](#3-consistency-models)
4. [CAP and PACELC](#4-cap-and-pacelc)
5. [Replication, partitioning, and storage](#5-replication-partitioning-and-storage)
6. [Consensus and coordination protocols](#6-consensus-and-coordination-protocols)
7. [Transactions across services](#7-transactions-across-services)
8. [Event-driven architecture fundamentals](#8-event-driven-architecture-fundamentals)
9. [Messaging semantics and delivery guarantees](#9-messaging-semantics-and-delivery-guarantees)
10. [Reliability and resilience patterns](#10-reliability-and-resilience-patterns)
11. [Event sourcing and CQRS](#11-event-sourcing-and-cqrs)
12. [Distributed caching and coordination](#12-distributed-caching-and-coordination)
13. [Observability and production reliability](#13-observability-and-production-reliability)
14. [Real-world interview scenarios](#14-real-world-interview-scenarios)
15. [References](#15-references)

## 1. Distributed systems fundamentals

### Q1. What is a distributed system, and why do we build one?

A distributed system is a collection of independent processes or machines that communicate over a network and cooperate to provide a capability. The parts may run on different hosts, fail independently, and observe different views of the system at the same time.

We distribute systems for horizontal scale, geographic reach, fault isolation, independent ownership, and access to data or compute near its users. For example, an order service, inventory service, and payment service may be owned and deployed separately because they have different business responsibilities and scaling needs.

Distribution is not free. A method call in one process usually has predictable local failure and latency characteristics; a network call can be delayed, duplicated by retry logic, lost, or have an unknown outcome. Distributed systems are therefore as much about defining behavior under failure as they are about moving computation to multiple machines.

### Q2. What changes when a local call becomes a network call?

A local call normally returns or throws within the same process, with shared memory and relatively low latency. A remote call crosses a network and can time out even when the remote service is still working. The request may never arrive, arrive and fail, succeed with the response lost, or finish after the caller gives up.

This creates an **ambiguous outcome**: a timeout tells the caller that it did not receive a timely response; it does not prove that the operation did not happen. If a payment request times out, blindly issuing a new payment can charge the customer twice.

Use explicit deadlines, bounded retries, idempotency keys for retryable mutations, and operation-status lookup where needed. Design APIs around realistic latency and failure rather than assuming a remote method behaves like a local function.

### Q3. Compare vertical scaling and horizontal scaling.

Vertical scaling increases the capacity of one machine, such as adding CPU or memory. It is operationally straightforward but has hardware limits and may leave a large single failure domain. Horizontal scaling adds instances and distributes work. It can improve capacity and availability, but requires load balancing, shared or partitioned state, coordination, and safe behavior when instances fail.

A stateless HTTP service is relatively easy to scale horizontally because any instance can handle a request. A stateful workload is harder: state must be replicated, partitioned, or externalized, and consistency requirements determine how much coordination is necessary. Horizontal scaling does not automatically solve a database bottleneck or a hot partition.

### Q4. Explain partial failure, and why it is central to distributed design.

A partial failure occurs when some components work while others are slow, unreachable, overloaded, or inconsistent. The whole system may not be clearly “up” or “down.” A service can accept requests while one dependency is timing out, or one region may be isolated from another.

The difficult part is uncertainty: a caller cannot instantly distinguish a crashed service from a delayed response or a broken network path. A robust design bounds waiting, isolates failures, preserves enough information to recover, and exposes degraded states rather than allowing one failing dependency to exhaust every thread and connection.

### Q5. Distinguish reliability, availability, durability, fault tolerance, and resilience.

- **Availability** is whether a service can successfully respond when requested, usually measured over a time window.
- **Reliability** is the broader ability to deliver correct service over time.
- **Durability** is the likelihood that committed data survives failures.
- **Fault tolerance** is the ability to continue meeting defined requirements despite specified faults.
- **Resilience** is the ability to absorb disruption, degrade safely, recover, and return to normal operation.

These properties are related but not interchangeable. A database can be available but return stale data; a system can return success while failing to durably persist the result. State which user-visible operation and correctness condition your metric measures.

### Q6. Why do timeouts not prove that a remote operation failed?

A timeout is a local observation about elapsed time. The request may have been processed successfully but its response delayed or lost. Alternatively, the request may still be queued or executing and may complete later.

For a read, retrying is often safe though it can add load. For a mutation, the caller needs an operation-level strategy: a stable idempotency key, deduplication record, queryable operation status, or a domain-specific reconciliation process. The server should persist the key and outcome atomically with the business effect when possible. An idempotency key is only useful if the server enforces its semantics and rejects conflicting reuse.

### Q7. How do redundancy and replication help, and what do they not solve?

Redundancy provides alternative components or paths; replication maintains copies of data or state. Multiple service instances behind a load balancer can tolerate an instance crash, while database replicas can support failover or reads.

Redundancy only helps when the failure domains are sufficiently independent. Replicas in one availability zone may all fail during a zone outage. Replication also introduces lag, conflict handling, and failover questions. A redundant architecture can still have a single point of failure in DNS, credentials, a control plane, a shared database, or an operator procedure.

### Q8. How should a system handle a dependency that is unavailable?

First identify whether the dependency is essential to the current operation. Apply a deadline, fail fast when the remaining budget is insufficient, and avoid unbounded queues. If safe, serve a stale or partial response, defer work to a durable queue, or offer a degraded feature. If the dependency is required for correctness—such as authoritative inventory validation—reject or defer the operation rather than fabricate success.

Recovery also matters: use bounded retries with backoff and jitter, monitor the dependency, and prevent a flood of waiting requests when it returns. A fallback must preserve business semantics; returning a guessed account balance is not graceful degradation.

## 2. Time, ordering, and coordination

### Q9. Why is there no single perfectly shared clock across machines?

Machines maintain clocks using local oscillators and synchronization protocols, but clocks drift, synchronization has uncertainty, and network delay varies. Wall-clock timestamps from two machines therefore cannot always establish which event truly happened first.

Use monotonic clocks to measure elapsed durations within a process; use wall-clock time for human-readable timestamps and cross-system correlation, with awareness of skew. Never assume a timestamp alone establishes a global total order. If ordering matters, define it explicitly using a sequence number, a single partition/leader, a transaction log, or causal metadata.

### Q10. What is the difference between wall-clock time and monotonic time?

Wall-clock time represents calendar time and can jump due to clock correction or administrative changes. Monotonic time is designed not to move backward and is appropriate for measuring durations such as request deadlines and timeout intervals.

For example, measuring a 500 ms timeout by subtracting two wall-clock timestamps can behave unexpectedly if the clock changes. Use the runtime's monotonic time source for elapsed time. Persisted event timestamps usually need wall-clock time, but should not be treated as a reliable ordering primitive.

### Q11. Explain event time versus processing time.

**Event time** is when the event occurred in the domain or was recorded by the producer. **Processing time** is when a consumer processes it. They can differ significantly because of queueing, retries, outages, replay, or clock skew.

A late-arriving event may have an older event timestamp than events already processed. Analytics systems often need a lateness policy or watermark to decide when a time window is considered sufficiently complete. Business state transitions may need sequence/version checks rather than relying on timestamps alone.

### Q12. What does the happened-before relation mean?

The happened-before relation captures causal ordering, not merely clock order. If event A occurs before event B in the same process, or A sends a message that B receives, then A happened before B. The relation is transitive. Events with no causal path between them are concurrent from the system's perspective.

This distinction matters when two regions independently update a record. Wall-clock timestamps may suggest an order, but they do not necessarily establish causality. Systems can preserve causal metadata or use conflict-resolution rules when concurrent updates are allowed.

### Q13. What is a Lamport clock, and what does it guarantee?

A Lamport clock assigns a logical counter to events. A process increments its counter for local events; when it sends a message, it includes the counter; on receipt, the receiver advances its counter to greater than both its local value and the received value.

If A happened before B, Lamport timestamps preserve that ordering. The reverse is not guaranteed: a smaller timestamp does not prove causality, and concurrent events can be ordered arbitrarily using a tie-breaker. Lamport clocks are useful for reasoning about logical order but do not provide a physical timestamp or detect all concurrent updates.

### Q14. What are vector clocks, conceptually?

A vector clock tracks a counter per participant or replica. Comparing two vectors can identify whether one causally precedes the other or whether they are concurrent. Concurrent vectors can indicate that two replicas independently changed a value and conflict resolution is required.

Vector clocks provide richer causal information than a single Lamport counter, but their metadata can grow with the number of participants and membership changes complicate the design. Backend engineers should understand the purpose and trade-off; implementing a production vector-clock protocol is specialized work.

### Q15. Why is leader election not the same as consensus?

Leader election attempts to select a node that coordinates work. Consensus is the stronger problem of having distributed participants agree on a value or ordered sequence despite failures and uncertain communication. A naïve election can produce two nodes that both believe they are leader during a partition or after a delayed message.

Production coordination usually combines elections with terms/epochs, quorum rules, and protocol guarantees that prevent conflicting leaders from committing conflicting decisions. A lease alone is not enough if an old leader can continue acting after its lease expires.

### Q16. What are leases and fencing tokens, and why are fencing tokens useful?

A lease grants authority for a limited period. If a client pauses for a long time, its lease may expire while it still believes it owns the resource. It can resume and write after a new client has acquired the lease.

A fencing token is a monotonically increasing number issued with each new ownership grant. The protected resource rejects operations with a token older than the newest accepted token. This protects against stale holders, provided the resource itself validates the token. A distributed lock that is not enforced by the resource cannot by itself guarantee correctness.

### Q17. Why are distributed locks difficult, and when should you use one?

Locks depend on communication, failure detection, and ownership expiry. A process can pause due to a long garbage-collection pause or network outage, lose its lease, and later resume. Two clients may then perform work unless the resource enforces fencing or another correctness mechanism.

Prefer database constraints, conditional writes, compare-and-set, unique keys, or partitioned single-writer processing when these directly enforce the invariant. Use distributed locks when coordination is truly required and the protected operation can validate ownership safely. Keep leases short enough for recovery but not so short that normal pauses constantly cause churn.

## 3. Consistency models

### Q18. What is linearizability?

Linearizability is a consistency guarantee under which each operation appears to take effect atomically at some point between its invocation and response, and the resulting order respects real-time ordering: if one operation completes before another begins, the first must appear earlier.

For a bank balance, a linearizable read after a completed deposit must reflect that deposit. Linearizability is intuitive for clients but can require coordination, especially across replicas or regions. It is a property of operations and a system's implementation, not a synonym for “the database is ACID.”

### Q19. What is sequential consistency, and how does it differ from linearizability?

Sequential consistency requires operations to appear in one total order that preserves each individual process's program order. Unlike linearizability, the total order does not have to respect real-time ordering between operations from different processes.

Linearizability is therefore stronger. The distinction matters in theory and in evaluating distributed primitives; in application discussions, be precise about the operation guarantees rather than describing every strong-looking behavior as “strong consistency.”

### Q20. Explain eventual consistency without oversimplifying it.

Eventual consistency means that if updates stop and communication/reconciliation continues, replicas will eventually converge. It does not mean that every read is random or that the system has no guarantees. A system may additionally provide read-your-writes, monotonic reads, or per-key ordering.

A product catalog can often tolerate a brief delay before a new description appears in every region. An inventory reservation or account debit usually needs stronger safeguards against overselling or double spending. Choose consistency from the business invariant, not from a blanket preference for performance.

### Q21. What are read-your-writes and monotonic reads?

**Read-your-writes** means a client that has completed an update can subsequently observe that update. **Monotonic reads** means that once a client has observed a value/version, it will not later observe an older one.

These session guarantees can improve user experience without requiring every read globally linearizable. Implementations may route a session to a leader, carry a version token, or wait until a replica catches up. The exact approach depends on the storage system and whether session state survives failover.

### Q22. What are quorum reads and writes?

In a replicated system with N replicas, a read quorum R and write quorum W are thresholds of replicas that must participate. A common intuition is that R + W > N makes read and write quorums overlap, and W > N/2 prevents two disjoint majority writes.

Overlap alone does not automatically guarantee linearizability. The protocol must also define versioning, conflict resolution, concurrent writes, replica repair, and what an acknowledgment means. Quorum behavior varies by database and consistency mode; avoid presenting the formula as a universal correctness proof.

### Q23. How does replication lag create stale reads?

With asynchronous replication, the primary can commit a write before a replica has applied it. A read routed to that replica may return an older value. This is expected behavior, not necessarily data loss.

Possible mitigations include reading from the leader for critical operations, waiting for a replica to reach a commit/version position, using session consistency tokens, or accepting bounded staleness where the database supports it. Read replicas improve read capacity but should not be assumed suitable for every read-after-write path.

### Q24. What is last-write-wins, and what are its limitations?

Last-write-wins selects one conflicting update using a timestamp or version comparison. It is simple and can converge, but “last” may be based on skewed clocks, and a valid concurrent update can be silently discarded. It is unsuitable when every update must be preserved, such as financial ledger entries.

Alternatives include optimistic concurrency with version checks, domain-specific merge rules, append-only records, or surfacing conflicts to a user. Conflict resolution should reflect the domain: merging shopping-cart additions is different from merging a legal address or a bank balance.

### Q25. How would you choose consistency for orders, inventory, payments, and a read model?

For payments and ledger-like effects, enforce uniqueness and atomic state transitions in an authoritative store; use idempotency keys and reconciliation to prevent duplicate effects. Inventory reservation often needs a conditional update or reservation record so two requests cannot consume the same stock. Order status can move through explicit states and use a saga when it spans services. A search or dashboard read model can usually be eventually consistent if its lag is observable and acceptable.

The key interview move is to state the invariant first. “We need strong consistency” is vague; “a stock unit can be reserved by at most one order” identifies the actual correctness requirement and the place where it must be enforced.

## 4. CAP and PACELC

### Q26. Explain the CAP theorem accurately.

CAP says that when a network partition occurs, a distributed data system cannot guarantee both linearizable consistency and availability for every request to the relevant data. Partition tolerance is not a convenient optional feature in a networked system; partitions are a failure the design must handle. The trade-off is about behavior during the partition.

An availability-oriented design may accept operations on both sides and reconcile conflicts later. A consistency-oriented design may reject or delay operations that cannot be safely coordinated. The classic phrase “choose any two of three” is misleading because it makes partition tolerance sound optional and hides that the trade-off is conditional on a partition. See Gilbert and Lynch's formal treatment in the references.

### Q27. What do CP and AP mean during a partition?

A CP-style choice preserves a consistency guarantee by refusing operations that cannot be safely coordinated. This may mean a minority partition cannot accept writes. An AP-style choice allows more requests to succeed on both sides, accepting that conflicting or stale results may need reconciliation.

These are simplified labels, not permanent identities for an entire product. Different operations, consistency settings, and failure conditions can behave differently. In an interview, name the operation, the partition, the guarantee preserved, and what the client experiences.

### Q28. What does PACELC add to CAP?

PACELC observes that if there is a **Partition (P)**, a system trades off **Availability (A)** and **Consistency (C)**; **Else (E)**, even without a partition, it may trade off **Latency (L)** and **Consistency (C)**. Synchronous cross-region coordination can increase normal-operation latency even when the network is healthy.

This framework helps explain why consistency decisions affect everyday performance, not just rare outages. It is a reasoning framework, not a complete classification of every database or workload.

### Q29. Does CAP mean we must always sacrifice availability or consistency?

No. CAP's constraint is about what guarantees can be simultaneously provided for affected operations during a partition. When communication is healthy, a system may offer strong consistency and high availability. During a partition, a system may preserve availability for some operations or partitions while rejecting others.

A design can also partition its data so that some keys remain reachable while others do not. Avoid global claims such as “this system is always CP” without specifying topology, operation, and configuration.

## 5. Replication, partitioning, and storage

### Q30. Compare synchronous and asynchronous replication.

With synchronous replication, the write acknowledgment waits for one or more replicas to confirm a specified level of durability or application. This can reduce the chance of losing acknowledged writes but increases latency and may reduce write availability when replicas are unreachable. Exact semantics depend on what is acknowledged: receipt, log persistence, or application.

With asynchronous replication, the primary can acknowledge sooner, but a failover may lose writes that had not reached the promoted replica. The right choice depends on the recovery point objective, latency budget, topology, and the database's precise guarantees.

### Q31. Compare primary-replica, multi-leader, and leaderless replication.

A primary-replica design routes writes through one primary and copies them to replicas. It simplifies write ordering but needs safe leader failover and may have replication lag. Multi-leader designs permit writes in multiple locations, improving locality and write availability but requiring conflict handling. Leaderless designs use replica quorums and a system-specific version/conflict protocol.

No topology removes trade-offs. Multi-leader writes are not automatically conflict-free, and leaderless quorums do not automatically imply linearizability. Understand the product's consistency protocol rather than relying on the architecture label alone.

### Q32. What is sharding, and how is it different from replication?

Sharding partitions a dataset into different subsets across nodes to distribute storage and workload. Replication maintains copies of data for availability, durability, or read capacity. A system can shard data and replicate each shard.

Choosing a shard key is crucial. A key with low cardinality or skew can create a hot shard, while a key that does not align with access patterns causes cross-shard queries and transactions. Rebalancing, key migration, and uneven growth must be planned rather than treated as one-time setup.

### Q33. How do you choose a partition key and avoid hot partitions?

Choose a key that distributes traffic and data reasonably evenly while supporting the dominant query patterns. If every request targets one popular customer or tenant, hashing the tenant ID alone may still create a hot key. Possible designs include sub-partitioning high-volume tenants, salting keys for aggregations, or isolating exceptional workloads.

Each mitigation has costs: salting makes reads and aggregates fan out; splitting a tenant complicates ordering and transactions. Monitor per-partition throughput, queue depth, storage, and latency—not just cluster-wide averages.

### Q34. What is split brain, and how can it be prevented?

Split brain occurs when two nodes or groups believe they are authorized to act as the active leader. Both may accept conflicting writes, creating divergent state. It can occur after partitions, flawed failover, or weak coordination.

Use a consensus-backed election or equivalent quorum protocol, epochs/terms, and fencing at the resource that accepts writes. A health check that merely says “the old leader is unreachable” is not enough to prove it cannot still operate. Failover should sacrifice some availability when necessary to prevent conflicting authoritative writes.

### Q35. What should a failover design consider?

Define which replica is eligible, how promotion is decided, how clients discover the new leader, how stale nodes are fenced, and whether acknowledged writes can be lost. Also account for recovery of the old leader: it must rejoin safely and reconcile or discard divergent state according to the protocol.

Test failover under network delay, not only a clean process crash. Monitor replication lag and recovery time. “We have replicas” is incomplete unless the promotion and client-routing behavior is understood and exercised.

## 6. Consensus and coordination protocols

### Q36. What problem does consensus solve?

Consensus allows distributed participants to agree on a value or sequence despite some failures and uncertain message delivery, subject to assumptions about the failure model and protocol. It is used for replicated metadata, leader terms, configuration changes, and logs that must have one consistent committed order.

Consensus is not magic fault elimination. A quorum-based system may stop making progress when it cannot reach a quorum, even if some individual nodes are healthy. This is often intentional: preserving a single committed history can be more important than accepting every write.

### Q37. Explain Raft at an interview level.

Raft organizes a replicated log around a leader. Nodes operate in terms/epochs; a candidate requests votes, and a majority elects a leader. The leader appends client commands to its log and replicates entries to followers. An entry becomes committed according to Raft's quorum and term rules, after which state machines apply it in order.

The core idea is a single agreed log for the replicated state machine. Safety rules prevent two conflicting entries from both being committed at the same log position. Raft also specifies log matching and membership-change behavior; a production implementation must follow the protocol rather than just “elect the node with the most votes.”

### Q38. What happens when a Raft leader fails?

Followers stop receiving heartbeats and eventually start an election, subject to randomized election timeouts. A candidate must obtain a majority of votes. If a majority cannot communicate, the cluster cannot elect a leader and new writes generally stop. When a leader is elected, log replication reconciles followers according to Raft's log consistency rules.

Timeouts can trigger an election even when a leader is merely slow or isolated, which is why terms and quorum rules matter. A healthy minority cannot safely invent a new committed history while a majority remains elsewhere.

### Q39. What is Paxos, and how much should a backend engineer know?

Paxos is a family of consensus protocols. At a conceptual level, proposers seek acceptance for a value through numbered proposals, and acceptors preserve promises that help ensure two different values cannot both be chosen. Multi-Paxos optimizes repeated decisions by establishing a stable leader-like proposer.

For most backend interviews, explain the consensus problem, quorum intersection, safety, and the difference between consensus and simple leader election. Do not pretend that a one-sentence description captures the full protocol or its liveness assumptions.

### Q40. Distinguish consensus, replication, leader election, and two-phase commit.

Replication copies state; it does not by itself determine a safe common history. Leader election chooses a coordinator but must be integrated with fencing and protocol rules. Consensus establishes agreement on a value or ordered log despite specified failures. Two-phase commit coordinates participants in a transaction so they decide to commit or abort, but it is not a consensus algorithm and can block while participants await a coordinator or decision.

These concepts can coexist in one system but solve different problems. A replicated log may use consensus, while a distributed business workflow may use a saga instead of a global atomic transaction.

### Q41. Explain safety versus liveness.

**Safety** means that something bad never happens—for example, two conflicting values are both committed as the same log position. **Liveness** means that something good eventually happens—for example, a request eventually completes.

A protocol can preserve safety by refusing to proceed without a quorum, sacrificing liveness during a partition. This is often the correct choice for authoritative state. In interviews, state the assumptions: no protocol can guarantee progress under every possible pattern of unbounded network delay and failures.

## 7. Transactions across services

### Q42. Why doesn't a local database transaction make a multi-service workflow atomic?

A local transaction covers resources controlled by one transactional system. If an Order service commits its database and then calls Inventory and Payment, those other services have independent transactions. A crash can occur after any step, and one service cannot roll back another service's already-committed side effect just by rolling back its own transaction.

Start by deciding whether the invariant truly needs one atomic boundary. If it does, consider whether the data belongs in one service/store. If separate ownership is justified, model a workflow with explicit states, durable progress, idempotent steps, and compensation or reconciliation.

### Q43. What is two-phase commit (2PC), and what are its trade-offs?

In the prepare phase, a coordinator asks each participant to prepare and promise it can commit. If all participants prepare successfully, the coordinator sends commit; otherwise it sends abort. Participants may hold locks or prepared state while awaiting the final decision.

2PC can provide atomic commit across participating transactional resources when supported correctly, but it adds coordinator and operational complexity and can block if a participant is prepared but cannot learn the decision. It is more appropriate when strong atomicity is required and the infrastructure supports distributed transactions than as a default across independently owned microservices.

### Q44. What is a saga?

A saga is a sequence of local transactions across services, with a compensating action for steps that need to be semantically undone if a later step fails. For example, an order workflow may reserve stock, authorize payment, and arrange shipment. If shipment creation fails, the workflow may release stock and void the authorization.

A saga is not a global ACID transaction. Other requests can observe intermediate states, compensation can fail, and some external actions cannot be reversed perfectly. Design durable state transitions, retry and timeout policy, idempotent commands, manual intervention paths, and reconciliation for stuck workflows.

### Q45. Compare saga orchestration and choreography.

In orchestration, a coordinator owns workflow progression and sends commands to participants. It makes the overall state machine and failure policy visible, which is helpful for long or branching workflows, but the orchestrator can become too central if it owns domain logic that belongs elsewhere.

In choreography, services react to events and emit subsequent events without one central coordinator. It can reduce direct coordination, but complex flows become harder to trace and can create hidden cyclic dependencies. Choose based on workflow complexity and operational clarity, not ideology; both still need idempotency, observability, and recovery.

### Q46. What is the transactional outbox pattern, and what problem does it solve?

The dual-write problem occurs when a service commits a database change and separately publishes an event. If the database commit succeeds but publication fails, downstream consumers never learn about the change; if publication happens first and the database transaction rolls back, consumers act on a fact that never committed.

With an outbox, the service writes its business data and an event record to an outbox table in the same local database transaction. A relay or change-data-capture process publishes committed outbox records. Publication may happen more than once, so consumers must be idempotent. The outbox closes the atomic gap between the business write and recording the intent to publish; it does not make downstream processing globally atomic.

```text
Local transaction:
  UPDATE orders ...
  INSERT INTO outbox(event_id, aggregate_id, type, payload, status) ...
Commit
        |
        v
Outbox relay / CDC -> broker -> idempotent consumer
```

### Q47. What are inbox and deduplication patterns?

A consumer can store a processed message identifier in an inbox/deduplication table and apply the business update in the same local transaction. If the same message is redelivered, the unique identifier prevents the business effect from being applied twice.

The deduplication record and business effect must share an atomic transaction, or another crash window remains. Choose an identifier with stable uniqueness and define its retention period based on replay and redelivery horizons. Deduplication protects repeated delivery of the same logical message; it does not automatically detect two different messages that represent the same business request.

### Q48. How should a payment/order/inventory workflow be designed?

Use explicit states and ownership: Order owns order lifecycle, Inventory owns reservations, and Payment owns payment attempts. An order request receives a stable idempotency key. The workflow records a pending order, requests an inventory reservation, then requests payment authorization, with events or commands recorded reliably via outbox.

If payment fails, release the reservation; if the final response times out, query payment status using the same operation identity instead of issuing an unrelated new charge. Model cancellation races and late responses explicitly. Use reconciliation to find reservations or authorizations that remain open beyond their expected lifetime.

### Q49. What is the difference between compensation and rollback?

A database rollback erases uncommitted changes within a transaction. A compensating action is a new business operation intended to counteract an earlier committed action. Refunding a settled payment is not identical to pretending the original payment never occurred; it may involve fees, delays, or regulatory records.

Compensation can fail or be only partial. Therefore, saga designs need retryable compensation, clear pending/failed states, audit history, and manual recovery. Do not promise “automatic rollback” for external side effects that cannot be atomically undone.

## 8. Event-driven architecture fundamentals

### Q50. What is event-driven architecture (EDA), and when is it useful?

EDA structures communication around events that describe facts that have occurred. Producers publish events; consumers subscribe and react independently. It is useful when multiple capabilities need to respond to a business change, when work can be asynchronous, or when producers and consumers should not need to be online at the same time.

The costs include eventual consistency, event-contract evolution, duplicate delivery, ordering constraints, replay behavior, and more difficult end-to-end debugging. A synchronous call may be clearer for an immediate validation or response. Use events where asynchronous decoupling provides a concrete benefit, not merely because a broker is available.

### Q51. Distinguish events, commands, and queries.

An **event** states a fact that has already happened, such as `OrderPlaced`. A **command** requests that something happen, such as `ReserveInventory`, and may be rejected. A **query** asks for information without intending to change state.

Naming and ownership should make this distinction clear. An event should not be phrased as an imperative command, because consumers should interpret it as an immutable historical fact. Commands need a responsible handler and a defined outcome; events may have many independent consumers.

### Q52. Compare event notification and event-carried state transfer.

An event notification says that something changed and points consumers toward additional information, often through an API. It keeps the event small but creates a runtime dependency on the source service and can make replay difficult if the source no longer retains the old state.

Event-carried state transfer includes enough relevant data for consumers to update their own read models without immediately calling the producer. It improves temporal decoupling but creates data duplication, privacy considerations, and schema-evolution obligations. Include only fields consumers are allowed to receive and genuinely need.

### Q53. What is temporal decoupling, and what does it not guarantee?

Temporal decoupling means the producer and consumer do not need to be available at the same instant: a durable broker or log can retain work until the consumer recovers. It does not guarantee infinite retention, exactly-once business effects, immediate processing, or that a consumer can handle the current event schema forever.

Define retention, retry policy, lag alarms, dead-letter recovery, and consumer compatibility. A queue postpones work; it does not remove the need to operate that work reliably.

### Q54. What are domain events and integration events?

A domain event represents something meaningful inside a domain model, such as `CreditLimitExceeded`. An integration event is a public contract intended to communicate a committed business fact to other systems. They may be related but need not have identical schemas or lifecycles.

Avoid publishing internal ORM entities or exposing every internal domain event as a public contract. Translate domain events into stable integration events at a deliberate boundary, and version them with consumers in mind.

### Q55. How should event contracts evolve?

Prefer additive, backward-compatible changes when old and new consumers may coexist: add optional fields with sensible defaults, preserve established meanings, and avoid reusing a field for a different purpose. Removing or changing semantics usually requires a migration plan, dual-read/write period, a new event version, or a coordinated cutover.

Compatibility depends on the serializer, schema registry rules, and consumer behavior. Test old-producer/new-consumer and new-producer/old-consumer combinations that can occur during rolling deployments. Keep schema ownership clear and document units, nullability, identifiers, and semantic meaning—not just field types.

### Q56. What makes a good event envelope?

A useful envelope often includes a unique event ID, event type, schema version, occurrence time, producer/source, aggregate or entity ID, correlation ID, causation ID, and payload. Include tenant or partitioning metadata only where required and safe.

These fields support deduplication, tracing, ordering checks, replay, and diagnosis. Timestamps are useful context but do not alone establish causal order. Do not place secrets or unnecessary personal data in long-lived events; retention and access policies apply to event logs too.

### Q57. What are common event-driven architecture anti-patterns?

Common problems include publishing events before the database transaction commits, using commands disguised as facts, creating oversized “god events,” tightly coupling consumers to internal schemas, relying on undocumented ordering, silently dropping poison messages, and adding a broker to a workflow that requires an immediate synchronous answer.

Another anti-pattern is choreography with so many event chains that no team can explain the overall business state. Improve explicit contracts, ownership, durable workflow state, tracing, replay procedures, and clear criteria for synchronous versus asynchronous communication.

## 9. Messaging semantics and delivery guarantees

### Q58. Explain at-most-once, at-least-once, and effectively-once processing.

**At-most-once** means a message is not processed more than once, but it may be lost. **At-least-once** means the system retries until delivery/processing is acknowledged under its assumptions, so duplicates are possible. **Effectively-once business processing** means duplicate attempts do not produce duplicate business effects, typically through idempotency, deduplication, transactions, or conditional state transitions.

Be precise about which layer the guarantee applies to: broker delivery, consumer invocation, database update, and external side effect are different boundaries. A broker's guarantee does not automatically extend to an unrelated payment provider.

### Q59. Why is end-to-end exactly-once business behavior difficult?

A consumer can update its database and crash before acknowledging the message. On redelivery, it may repeat the update. If it acknowledges first and crashes before updating the database, the update may be lost. A transaction spanning the broker, database, and an external provider is not generally available by default.

The practical solution is usually a stable event/operation ID, an atomic local transaction containing deduplication and state change, idempotent external APIs where available, and reconciliation for uncertain outcomes. Some platforms provide transactional processing within a defined boundary, but that is not a universal exactly-once guarantee across arbitrary external effects.

### Q60. What should a consumer do when processing fails?

Classify the error. Transient failures may be retried with bounded exponential backoff and jitter. Permanent validation or schema errors should not retry forever; quarantine them with enough metadata for investigation. Rate-limit or pause consumption if the downstream dependency is overloaded.

A dead-letter queue is not a graveyard: monitor its size and age, define an owner and replay procedure, and fix the cause before replaying. Replaying without idempotency can repeat side effects. Preserve original identifiers and correlation metadata so recovery is auditable.

### Q61. How should ordering be handled?

First define the required scope: global order is expensive, while per-entity or per-partition order is often sufficient. Use a stable partition key such as order ID when all events for one order must be processed sequentially. Even then, retries, multiple producers, asynchronous side effects, and concurrent handlers can complicate effective completion order.

Include entity versions or sequence numbers when consumers must detect gaps or stale updates. If an event with version 8 arrives before version 7, the consumer can buffer, retry, reject, or rebuild state according to a documented policy. Do not assume timestamps guarantee order.

### Q62. What is backpressure, and how do you control consumer lag?

Backpressure prevents producers from overwhelming downstream consumers by limiting in-flight work, queue size, concurrency, or request rate. Consumer lag is the gap between produced and processed work; its growth indicates that arrivals exceed processing capacity over time.

Measure lag and its age, processing throughput, retry rate, queue depth, and downstream saturation. Scale consumers only if the broker's partitioning and downstream dependencies permit it. If every consumer is blocked on a slow database, adding concurrency can worsen the database outage. Use bounded queues, bulkheads, admission control, and load shedding where appropriate.

### Q63. How should poison messages and delayed retries be managed?

A poison message repeatedly fails due to invalid data, incompatible schema, or a deterministic bug. Retrying it immediately blocks useful work and wastes capacity. Use a bounded retry policy, backoff, and a dead-letter/quarantine path with error classification and original message metadata.

Delayed retries can preserve throughput, but moving a failed message to a separate retry channel may change ordering. Before replaying, fix the underlying defect, validate the payload, decide whether intervening events have changed state, and ensure the handler is idempotent. Alert on age and count, not only whether the dead-letter queue exists.

### Q64. Why can retries amplify an outage?

When a dependency slows down, callers time out and retry, increasing its load precisely when it has less capacity. If several layers retry independently, one incoming request can produce many downstream attempts. This is retry amplification and can turn a partial outage into a cascading failure.

Use one clearly owned retry policy where possible, capped attempts, exponential backoff with jitter, an overall deadline, retry budgets, and circuit breaking or load shedding. Retry only errors that may be transient and operations that are safe to retry. A timeout on a mutation requires idempotency or status reconciliation.

## 10. Reliability and resilience patterns

### Q65. How should timeouts and deadline propagation work?

Set timeouts for every remote dependency, based on a measured latency distribution and the end-to-end user budget. A deadline is the latest time the caller is willing to wait; propagate the remaining budget downstream rather than giving every hop a fresh full timeout.

If an HTTP request has a 2-second total budget, three sequential calls with 2-second timeouts can take far longer than the user budget. Cancel work where possible when the deadline expires, while remembering that cancellation does not prove a remote mutation stopped. Log deadline exhaustion separately from application errors.

### Q66. Compare retries, circuit breakers, and bulkheads.

Retries attempt a failed operation again when the failure may be transient. Circuit breakers temporarily stop calls to a failing dependency after a threshold, allowing recovery and avoiding repeated waiting. Bulkheads isolate resources—such as thread pools or concurrency limits—so one dependency cannot consume all capacity.

They solve different problems and are often combined carefully. Retries without budgets can amplify load; a circuit breaker can reject calls that might otherwise succeed; a bulkhead can limit throughput. Tune them from observed failure modes and service objectives, not by copying a universal configuration.

### Q67. What is exponential backoff with jitter?

Exponential backoff increases the delay between repeated attempts, often by multiplying the previous delay up to a cap. Jitter randomizes the delay so many clients do not retry simultaneously after a shared outage. For example, a retry policy might choose a random delay within a capped exponential window.

The exact formula matters less than bounded attempts, an overall deadline, and load-aware limits. Do not retry every error; validation failures, authorization failures, and many conflict responses need different handling. For non-idempotent operations, reuse the same idempotency key across attempts.

### Q68. Explain rate limiting and load shedding.

Rate limiting constrains the rate or concurrency of work admitted from a client, tenant, or system. Load shedding deliberately rejects lower-priority or excess work when the system is saturated, preserving capacity for critical operations.

Return a clear retry signal when appropriate, apply fairness so one tenant cannot monopolize capacity, and distinguish quota exhaustion from system overload. Queueing all requests is not always graceful: unbounded queues turn overload into memory exhaustion and extreme tail latency.

### Q69. What is graceful degradation?

Graceful degradation provides a reduced but truthful capability when an optional dependency fails. A product page may omit recommendations while still showing the product. It should not fabricate a payment status, availability promise, or account balance.

Decide in advance which dependencies are critical, what fallback is valid, how stale the fallback may be, and how the user sees the degraded state. Measure fallback use and ensure cached or stale data is not mistaken for authoritative state.

### Q70. How should graceful shutdown handle in-flight work?

Stop accepting new work, signal readiness removal, allow a bounded drain period, finish or safely checkpoint in-flight operations, and close dependencies cleanly. For message consumers, stop polling or acquiring new messages and acknowledge only work whose effects are safely committed.

A process may still be terminated after the grace period, so recovery must tolerate interruption. Never acknowledge a message before durable processing simply to make shutdown faster. Ensure the orchestrator's termination grace period, application shutdown hooks, and consumer acknowledgment strategy agree.

## 11. Event sourcing and CQRS

### Q71. What is event sourcing, and when is it appropriate?

Event sourcing stores the sequence of domain events as the authoritative history from which current state can be derived. Instead of only storing the current account state, a ledger-like system may store accepted deposits, withdrawals, and adjustments, then derive a balance.

It is useful when history, auditability, temporal reconstruction, or domain transitions are central requirements. It adds complexity around event schema evolution, replay, snapshots, projections, correction events, and operations. It is not simply “logging every database change,” and it is usually unnecessary for straightforward CRUD applications.

### Q72. How does event replay work, and what can go wrong?

A projection or aggregate rebuild reads historical events and reapplies their state transitions. This requires deterministic handlers and compatible event schemas. If replay calls an external payment API or sends email again, it can repeat real-world side effects; replay should normally rebuild internal state without reissuing external actions.

Snapshots can reduce replay time but must be versioned and recoverable. When business logic changes, decide whether old events are interpreted using old semantics, upcast to a new schema, or replayed through a controlled migration. Test a rebuild against known state and make replay progress observable.

### Q73. What are CQRS and its trade-offs?

Command Query Responsibility Segregation separates the model used to change state from the model used to answer queries. The write model enforces business rules; one or more read models can be optimized for different access patterns.

CQRS can help when read and write needs differ substantially, but it creates projection lag, duplicated models, rebuilding and schema-evolution work, and more operational complexity. CQRS does not require event sourcing: the write side can use a conventional database and publish changes to build read models.

### Q74. How is event sourcing different from audit logging?

Audit logging records actions or changes for accountability and investigation, often alongside a conventional current-state database. Event sourcing treats the event log as the authoritative state history and derives current state from it. The audit log may be incomplete for reconstruction, while an event store must contain the events needed to rebuild domain state.

Both require careful retention, access control, privacy handling, and integrity. Do not adopt event sourcing merely because an audit trail is required; a well-designed append-only audit record may satisfy the requirement at much lower complexity.

### Q75. How do you correct an event-sourced record?

Avoid silently editing historical events if the log is intended to be immutable. Record a compensating or correction event that explains the business correction, then update projections by applying it. For example, an erroneous ledger entry is often offset by a reversal rather than erased.

Some systems support controlled event upcasting or administrative migration, but these need auditability and a reproducible plan. The correction mechanism should preserve the distinction between what was originally recorded and what the business later determined was correct.

## 12. Distributed caching and coordination

### Q76. Why is cache invalidation hard in distributed systems?

A write can update the source database while one or more caches still contain an older value. Invalidation messages can be delayed or lost, and concurrent readers may repopulate a stale value after invalidation. The cache's consistency guarantee therefore depends on write flow, invalidation strategy, and read behavior.

Common strategies include cache-aside with explicit invalidation, short TTLs, versioned keys, or write-through patterns. Each trades latency, staleness, and complexity differently. For correctness-critical decisions, validate against the authoritative store or use a conditional operation there.

### Q77. What is a cache stampede, and how do you mitigate it?

A cache stampede occurs when a popular key expires and many requests simultaneously query the backing store, creating a burst of load. Mitigations include request coalescing/single-flight, randomized TTLs, early refresh, stale-while-revalidate where safe, and rate-limited refresh workers.

A distributed lock may reduce duplicate refreshes but introduces lease-expiry and stale-holder concerns. Ensure the fallback does not allow unbounded database traffic during a cache outage, and avoid making a cache the only durable copy of business state.

### Q78. What happens if a distributed cache is unavailable?

The answer depends on whether it is a performance optimization or an essential coordination component. If it is a cache, fall back to the source of truth with concurrency limits and possibly degraded functionality. If it is used for sessions, rate limiting, or coordination, failure may affect correctness or security and should be handled explicitly.

Do not send unlimited traffic to the database when the cache fails. Use circuit breakers, admission control, bulkheads, and a recovery plan. The architecture should clearly identify which data is authoritative and what behavior is safe when cached data cannot be read.

## 13. Observability and production reliability

### Q79. How do logs, metrics, and traces complement each other?

Metrics show aggregate behavior and trends, such as request error rate, p99 latency, queue lag, and saturation. Logs provide event-level details for particular failures. Distributed traces connect spans across services to show where time was spent and where a request failed.

Propagate trace context through HTTP and message headers, and retain correlation IDs and business operation IDs where useful. Avoid high-cardinality metric labels such as raw user IDs. Logs and traces must not leak secrets or unnecessary personal data. Sampling should preserve enough failing and slow traces for diagnosis.

### Q80. What are SLI, SLO, and error budgets?

An SLI is a measured indicator of service behavior, such as successful requests within a latency threshold. An SLO is the target for that indicator over a defined window. An error budget is the tolerated amount of unreliability implied by the SLO.

For example, a 99.9% success SLO permits 0.1% of eligible requests to fail within its measurement window, subject to the exact SLI definition. Error budgets make reliability a measurable trade-off rather than a vague demand for “100% uptime.” They can guide release risk and prioritization of reliability work.

### Q81. Which signals help diagnose a slow event-driven pipeline?

Track incoming and completed event rates, consumer lag and oldest-message age, processing latency percentiles, retry and dead-letter counts, per-partition imbalance, downstream database latency, connection-pool saturation, and time spent waiting on dependencies. Correlate metrics with deploys and incident timelines.

A growing lag does not automatically mean consumers need more instances. The bottleneck could be a hot partition, serialized processing, a slow downstream database, retry storms, or insufficient partitions. Determine whether arrivals exceed sustainable processing capacity and identify the limiting stage before scaling.

### Q82. Explain RTO and RPO.

**Recovery Time Objective (RTO)** is the target time to restore a service after disruption. **Recovery Point Objective (RPO)** is the acceptable amount of data loss measured in time. A low RPO generally requires durable replication, backups, or recovery mechanisms that preserve recent writes; a low RTO requires tested failover and restore procedures.

Backups alone do not guarantee either objective. Test restoration, credentials, networking, data integrity, and application startup. Replication can copy accidental deletion or corruption, so point-in-time recovery and independent backups remain important.

### Q83. How should chaos engineering be used safely?

Chaos engineering tests a specific hypothesis about system behavior under a controlled fault, such as a dependency timeout or one node failure. Begin in a non-production environment or a tightly bounded production experiment with an observable steady state, abort conditions, and responsible operators.

Do not inject random failures without a question or recovery plan. Validate that alerts fire, blast radius stays bounded, and the system recovers. Follow each experiment with evidence and remediation; a chaos test that finds a weakness but is never fixed is not reliability improvement.

### Q84. What is a cascading failure, and how do you stop it?

A cascading failure happens when one component's degradation causes other components to overload or fail. For example, a slow database holds application threads; requests time out and retry; connection pools fill; health checks fail; and traffic shifts to fewer instances, further increasing load.

Contain it by bounding concurrency and queues, propagating deadlines, limiting retries, shedding noncritical work, isolating dependency pools, and protecting recovery capacity. During an incident, reduce incoming load and stop amplification before adding more traffic. Afterward, use traces and timelines to distinguish trigger from contributing conditions.

## 14. Real-world interview scenarios

### Q85. A service times out, but the remote operation may have succeeded. What do you do?

**Symptoms and investigation:** The caller sees a timeout, while the remote service or database may show a committed operation. Check request IDs, server logs, database state, and whether the request reached the handler. Do not infer failure from the caller's timeout alone.

**Remediation:** For a mutation, retry using the same idempotency key or query the original operation's status. Persist the key, request fingerprint, and outcome atomically with the business effect. If the operation is externally handled, use the provider's idempotency mechanism and reconcile uncertain states.

**Prevention and trade-off:** Define deadlines and operation semantics in the API. Idempotency records require retention and conflict rules, but are safer than guessing whether a timed-out mutation happened.

### Q86. A payment is processed twice after retries. How would you fix it?

Inspect whether the caller generated a new operation ID on each retry, whether the provider supports idempotency keys, and whether the local service commits deduplication separately from payment state. Determine if there are two provider charges or only duplicate local records.

Use a stable payment-attempt ID across retries, enforce uniqueness in the authoritative store, and send the same provider idempotency key on each attempt. Reconcile provider settlement records against local state and refund duplicates through an auditable business process. Never assume a retry library alone provides exactly-once charging.

### Q87. An order is created, but inventory reservation fails. What architecture would you use?

Represent the order as pending rather than confirmed. Run a saga: request a reservation, transition the order when reservation succeeds, then continue payment/fulfillment according to the business sequence. If reservation definitively fails, cancel or reject the order. If the result is uncertain, query reservation status using a stable workflow ID before issuing another command.

Publish state changes reliably with an outbox, make participant operations idempotent, and track saga age and stuck states. The user experience should explain that the order is pending when that is the truth, rather than falsely promising confirmation.

### Q88. A database commits, but event publication fails. What do you change?

This is the dual-write problem. Check whether the service performs a database commit and a separate broker publish, and inspect whether committed records lack corresponding events.

Write an outbox record in the same database transaction as the business update, then publish from the outbox using a relay or CDC. The relay must resume after crashes and tolerate duplicate publication; consumers deduplicate and process idempotently. Monitor oldest unpublished outbox age and backlog, and define cleanup only after safe publication is established.

### Q89. Consumers process events out of order. How do you diagnose and remediate it?

Determine the ordering contract: global, per key, per aggregate, or none. Inspect producer concurrency, partition/key assignment, retry topics, consumer parallelism, and whether processing completion can be reordered even when delivery is ordered.

If per-order order is required, route all events for an order through a stable partition and include aggregate version/sequence. Consumers can reject stale versions, detect gaps, or buffer and retry missing events. Do not use wall-clock timestamps as the sole ordering mechanism. If strict sequencing lowers throughput, explain why the business invariant justifies it.

### Q90. A consumer crashes after updating the database but before acknowledging the message. What happens?

The broker may redeliver the message because it did not observe acknowledgment. Without deduplication, the consumer can apply the database change twice. Acknowledge-after-processing avoids losing unprocessed work but intentionally permits duplicates after this crash window.

Store the message ID and apply the business update in one local database transaction. On redelivery, the inbox uniqueness constraint indicates the message was already committed; the consumer can acknowledge it without repeating the effect. Test crash points around commit and acknowledgment.

### Q91. A network partition separates two parts of a cluster. What should happen?

Start with the invariant. If conflicting writes would be harmful, allow only the partition with a valid quorum/leader to commit and reject or delay writes on the other side. If availability is more important and the domain supports merging, both sides may accept updates with explicit conflict resolution.

Do not let both sides silently believe they are the authoritative leader. Use quorum protocols, epochs, and fencing. When connectivity returns, reconcile only according to the defined consistency model. The trade-off should be a deliberate product decision, not an accidental consequence of timeout settings.

### Q92. A leader fails while writes are in progress. How do you reason about lost or uncertain writes?

Check the write acknowledgment semantics, replication protocol, commit index/position, and replica eligibility for promotion. A client that timed out may not know whether its write was committed. A new leader should follow the storage protocol's rules rather than selecting a replica simply because it is reachable.

Clients should retry mutations with stable idempotency keys or query the outcome. After failover, monitor replication recovery and reconcile business operations that may have an ambiguous result. The acceptable loss window must align with the system's RPO and durability contract.

### Q93. Replication lag makes a user see stale data. What are your options?

Confirm the lag and whether the request was routed to a read replica immediately after a write. If the user requires read-your-writes, route that session to the primary, wait for the replica to reach a version/position, or carry a consistency token if supported. For noncritical analytics, explain and tolerate bounded staleness.

Avoid routing every read to the primary without evaluating load. Define which endpoints require freshness and measure both lag and user-visible stale reads. A “read replica” is a capacity choice, not a promise that reads reflect the latest write.

### Q94. Two regions accept conflicting updates. How should the system resolve them?

First determine whether concurrent writes should have been accepted at all. For a bank balance or unique inventory reservation, use a single authoritative coordination path or a conflict-safe operation rather than blindly merging. For user preferences, concurrent updates may be resolved using explicit domain rules or surfaced to the user.

Timestamps and last-write-wins are simple but can discard valid updates under clock skew. Consider versions, causal metadata, append-only operations, or a domain-specific merge. State the latency and availability cost of stronger cross-region coordination.

### Q95. A hot partition becomes the throughput bottleneck. What do you inspect?

Break down traffic, storage, latency, and consumer lag per partition or key, and identify whether a few tenants or entities dominate. Check whether the partition key has low cardinality, skewed distribution, or a single popular entity.

Potential mitigations include splitting heavy tenants, adding a sub-key, salting write keys, isolating high-volume customers, or changing the access model. Rebalancing alone may not help if the hot key remains indivisible. Each mitigation can complicate ordering and aggregation, so validate with representative load tests.

### Q96. A distributed lock expires while its original holder is still working. What prevents stale writes?

The original holder may pause, lose its lease, and resume after another holder has acquired the lock. If the protected resource accepts the old holder's write, the lock did not preserve correctness.

Issue a monotonically increasing fencing token on each ownership grant and require the storage resource to reject stale tokens. Where possible, use conditional writes or version checks enforced by the authoritative database. Test long pauses and network partitions; a lock service's “lock acquired” response alone is not enough.

### Q97. A retry storm overwhelms a recovering dependency. How do you stop it?

Reduce incoming and retry traffic first: cap attempts, apply exponential backoff with jitter, enforce retry budgets and deadlines, open circuit breakers where appropriate, and shed noncritical work. Check whether multiple layers each retry the same request and multiply traffic.

After stabilizing, restore traffic gradually and watch queue depth, saturation, error rate, and latency. Add capacity only after understanding the bottleneck. Prevent recurrence with load tests, bounded concurrency, and a policy that distinguishes transient errors from permanent failures.

### Q98. An event-driven workflow is stuck halfway through. How do you investigate?

Find the workflow/saga ID and reconstruct its state transitions using correlation IDs, event IDs, command outcomes, retry history, and timestamps. Identify whether a message is unpublished, queued, repeatedly failing, acknowledged prematurely, or waiting on an external system.

Make workflow state durable and expose age-of-state alerts. Retry only idempotent steps; query external status when outcomes are uncertain. Provide an audited repair or compensation path for operators rather than editing database rows manually without recording the business decision.

### Q99. A downstream consumer cannot keep up with events. What do you do?

Compare arrival rate with sustainable processing throughput and find the limiting stage. Inspect partition skew, consumer CPU, database latency, connection pools, retries, and batch sizes. If work can be parallelized safely, add consumers or partition capacity; if the downstream database is saturated, more consumers may make it worse.

Use backpressure, bounded concurrency, batching, and priority policies where valid. Decide whether lag is acceptable based on the business freshness objective, and alert on oldest event age as well as count. Have a retention and replay plan in case lag exceeds the broker's retention window.

### Q100. A replay causes duplicate side effects. How do you prevent it?

Separate projection rebuilding from external actions. Replaying historical events to reconstruct a read model should not send new emails, charge cards, or create duplicate shipments by default. Use replay modes or separate projection handlers that do not execute live side effects.

Where a side effect must be reissued, require an explicit business operation with its own idempotency key and audit trail. Track replay identity and offsets/checkpoints, and verify the target state before resuming. Event history is valuable only if replay semantics are safe and understood.

### Q101. A multi-service operation must support cancellation and compensation. How would you model it?

Define durable workflow states such as `PENDING`, `RESERVING`, `AUTHORIZED`, `CONFIRMED`, `CANCELLING`, `CANCELLED`, and `MANUAL_REVIEW`. Cancellation is a new command, not deletion of history. The workflow must decide what happens if cancellation races with payment authorization or shipment dispatch.

Use a saga coordinator or clear event choreography, idempotent participant commands, outbox/inbox patterns, and explicit compensation. Some steps may be irreversible or require a refund rather than rollback. Define terminal states, timeout policy, operator recovery, and customer-visible status.

### Q102. The system must remain available during a regional outage. What trade-offs matter?

Clarify which capabilities must remain available, what data loss is acceptable, and which operations require global uniqueness or strong consistency. Multi-region deployment needs more than duplicated servers: routing, data replication, identity, secrets, dependencies, failover automation, and recovery procedures must also work.

Active-active writes can create conflicts and require coordination or domain-specific merge rules. A single-writer design simplifies correctness but may pause writes during failover or increase cross-region latency. Test regional failover, stale DNS/client routing, and return-to-service reconciliation against explicit RTO/RPO objectives.

### Q103. Should two services communicate using synchronous REST or asynchronous messaging?

Use synchronous REST when the caller needs an immediate answer, the operation is a bounded query or command, and coupling to dependency availability is acceptable. Use messaging when work can complete later, multiple consumers need the same fact, buffering is valuable, or producer and consumer availability should be decoupled.

A hybrid is common: an API accepts a command and returns an operation ID or pending status, then events drive the workflow. Messaging introduces eventual consistency, replay, deduplication, and operational lag; REST introduces runtime dependency and timeout/ retry concerns. Choose based on user experience and business invariants, not fashion.

### Q104. A read model is behind the source-of-truth database. How should the API behave?

First quantify lag and identify whether the projection is blocked, failing, or simply processing a burst. If the endpoint promises immediate read-your-writes, use a write response that returns the committed state, route the follow-up read to the authoritative store, or wait for a projection version to catch up.

For analytics or search, eventual consistency may be acceptable if the contract says so and lag is observable. Avoid claiming a command succeeded in the read model before it has applied. Provide repair/rebuild procedures and alert on the age of the oldest unprocessed event.

### Q105. How do you decide whether to introduce a distributed pattern or keep the design simple?

Start with the business invariant, scale and failure requirements, and current evidence. A single database transaction may be simpler and safer than a saga if the data naturally belongs together. A direct REST call may be clearer than a broker for an immediate validation. Event sourcing may be excessive if audit history alone is required.

Adopt the least complex design that meets explicit requirements, then validate it with failure tests and operational metrics. Explain the cost you accept: coordination latency, eventual consistency, duplicated data, recovery tooling, or reduced availability. Complexity should buy a specific guarantee or capability.

## 15. References

These sources provide foundational definitions and implementation guidance. Protocol details and product behavior should always be checked against the specific system/version in use.

1. Gilbert and Lynch, *Perspectives on the CAP Theorem* (IEEE Computer; MIT-hosted manuscript): https://doi.org/10.1109/MC.2011.389
2. Raft consensus algorithm, original paper and related material: https://raft.github.io/
3. AWS Prescriptive Guidance, Transactional Outbox pattern: https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html
4. AWS Prescriptive Guidance, Saga orchestration: https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html
5. AWS Prescriptive Guidance, Saga choreography: https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html
6. Google, *Site Reliability Engineering*, service best practices and error budgets: https://sre.google/sre-book/service-best-practices/
7. Martin Fowler, *What do you mean by “Event-Driven”?*: https://martinfowler.com/articles/201701-event-driven.html
8. Martin Fowler, *Event Sourcing*: https://martinfowler.com/eaaDev/EventSourcing.html
9. Martin Fowler, *CQRS*: https://martinfowler.com/bliki/CQRS.html
10. Kleppmann, *Designing Data-Intensive Applications*, O'Reilly — replication, partitioning, consistency, transactions, and stream processing: https://dataintensive.net/
11. NIST, *Contingency Planning Guide for Federal Information Systems* (recovery planning concepts): https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final

---

**Study advice:** For each scenario, practice stating (1) the invariant, (2) the failure window, (3) the recovery mechanism, and (4) the trade-off. That structure demonstrates engineering judgment more effectively than naming patterns without explaining why they fit.
