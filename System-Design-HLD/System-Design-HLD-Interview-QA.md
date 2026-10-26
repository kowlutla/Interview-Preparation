# System Design HLD Interview Handbook

**Audience:** Java backend engineers with 5–10 years of experience preparing for senior and staff-leaning backend/product-company interviews.  
**Purpose:** A progressive guide to architecture reasoning, capacity estimation, data ownership, consistency, failure handling, operations, and cost.  
**Status:** Rebuilding for the 5–10 year experience band. Includes eight worked designs and 15 core-concept deep dives; the full 300-question bank and 80 scenarios have received substantial depth rewrites, with the remaining audit focused on topic coverage, technical verification, readability, and rendered diagrams. Not a final sign-off yet.

> **How to use this guide:** Learn fundamentals in order, then practise scenarios aloud. State assumptions, identify the critical business invariant, estimate scale, draw the simplest viable architecture, and deepen the part the interviewer challenges. Naming technologies is not the goal; explaining why the design satisfies requirements is.

### What interviewers expect from a 5–10 year candidate

At this level, an answer should normally go beyond naming a component. Be ready to explain the workload and invariant that justify it, the request/data flow, the guarantee the component provides, its bottleneck and failure modes, and why you chose it over the main alternatives. The interviewer may ask you to estimate capacity, trace a timeout across three services, explain what happens during a partition, or describe a zero-downtime migration.

Use this answer pattern when useful:

1. **State the requirement and invariant.** What must be fast, available, fresh, unique, ordered, or durable? Which errors are unacceptable?
2. **Explain the mechanism.** Show how requests are routed, data is partitioned, transactions are protected, or work is retried—not just which technology is chosen.
3. **Quantify the design.** Estimate peak throughput, storage, fan-out, connection demand, queue drain, or replication cost with assumptions.
4. **Describe failure and recovery.** Include partial success, timeouts with uncertain outcomes, overload, stale data, leader failover, replay, and restore—not just a process crash.
5. **Compare alternatives and operational cost.** Explain what you give up, how you observe it, how you roll it out, and what evidence would make you change the design.

A strong answer is not necessarily longer in every interview. It is deep enough to answer “why?”, “what if it fails?”, and “how do you know it works?” without hand-waving.

## Table of contents

1. [HLD interview method](#1-hld-interview-method)
2. [Requirements and quality attributes](#2-requirements-and-quality-attributes)
3. [Capacity estimation](#3-capacity-estimation)
4. [Networking and request delivery](#4-networking-and-request-delivery)
5. [APIs and service boundaries](#5-apis-and-service-boundaries)
6. [Scaling and traffic management](#6-scaling-and-traffic-management)
7. [Databases and data modelling](#7-databases-and-data-modelling)
8. [Caching and content delivery](#8-caching-and-content-delivery)
9. [Distributed systems and consistency](#9-distributed-systems-and-consistency)
10. [Messaging and event-driven architecture](#10-messaging-and-event-driven-architecture)
11. [Transactions and business correctness](#11-transactions-and-business-correctness)
12. [Resilience and failure handling](#12-resilience-and-failure-handling)
13. [Security and abuse prevention](#13-security-and-abuse-prevention)
14. [Observability and operations](#14-observability-and-operations)
15. [Multi-region and disaster recovery](#15-multi-region-and-disaster-recovery)
16. [Cost and architectural evolution](#16-cost-and-architectural-evolution)
16a. [Core concept deep dives: databases, distributed coordination, and traffic infrastructure](#core-concept-deep-dives-partitioning-sharding-and-traffic-infrastructure)
17. [Advanced HLD deep-dive Q&A](#17-advanced-hld-deep-dive-qa)
18. [Senior-level scenario bank](#18-senior-level-scenario-bank)
19. [End-to-end case-study index](#19-end-to-end-case-study-index)
20. [Cross-references and authoritative reading](#20-cross-references-and-authoritative-reading)

---

## 1. HLD interview method

### Q1. What does a High-Level Design interview evaluate?

An HLD interview evaluates whether you can turn an ambiguous product goal into an architecture that satisfies business requirements at a plausible scale—and explain what happens when parts fail. Interviewers assess requirement discovery, capacity estimation, component boundaries, data ownership, API/event flow, consistency, fault tolerance, security, operations, cost, and the clarity of your reasoning. They are not simply testing whether you know the names of databases, queues, caches, or load balancers.

At 5–10 years of experience, the key difference is how you defend the design. If you choose a cache, explain the reuse/freshness assumptions, invalidation, stampede behavior, and cache-outage capacity. If you shard a database, explain the shard key, query locality, global uniqueness, rebalancing, and cross-shard cost. If you move work to a queue, explain durable acceptance, duplicate processing, ordering, retries, expiry, and status visibility.

Start from the user journey and the business invariant. For ticket booking, preventing two customers from buying the same seat matters more than choosing a framework. Then estimate demand, draw the simplest viable architecture, trace one normal request, and explore the most dangerous failure paths. A strong answer is explicit about assumptions, guarantees, trade-offs, and how you would validate the design with tests or production signals.

### Q2. How is HLD different from LLD?

High-Level Design (HLD) explains the system's major building blocks, responsibilities, data ownership, request/event flows, storage choices, scaling approach, and failure/recovery behavior. It answers questions such as: which service owns an order, how a booking is made atomic, where asynchronous work runs, and how the system behaves when a region or dependency fails.

Low-Level Design (LLD) focuses on internal implementation: classes, interfaces, method contracts, object collaboration, state transitions, and design patterns. For example, LLD may define the classes and methods used by a payment workflow; HLD decides how the payment service interacts with the order store, provider, event stream, idempotency record, and reconciliation process.

The boundary is not absolute. A database constraint, idempotency key, or fencing token may look like an implementation detail, but it belongs in HLD if it determines correctness or changes the architecture. Likewise, a retry policy is not just a line of code if it can multiply traffic across a fleet and create a cascading failure.

A useful rule for interviews: include any detail that could change correctness, capacity, availability, security, operational ownership, or the choice between architectures. Defer class structure and method-level details unless the interviewer asks or the implementation decision affects one of those properties.

### Q3. How should you structure a 45–60-minute design interview?

A useful default is to spend roughly the first five minutes clarifying user journeys, scope, and business invariants; the next five minutes defining quality targets and estimating load; the next ten minutes drawing the main components and request/data flow; and the remaining time deepening data modelling, consistency, bottlenecks, failure handling, and trade-offs. Keep the schedule flexible—the interviewer may want to explore one critical problem in depth.

Do not spend fifteen minutes listing every requirement or calculating overly precise capacity from assumptions you cannot know. Identify the two or three facts that materially change the architecture, state reasonable assumptions for the rest, and move forward. Show one main happy-path flow before adding optional boxes. Then select likely pressure points from the problem: duplicate bookings, hot partitions, fan-out latency, retry storms, or a regional failure.

Reserve time to explain how the design is operated: SLOs, important metrics, rollout/rollback, backup/restore, and what happens when a dependency is unavailable. End with the strongest trade-off and the next bottleneck if demand grows. The interviewer should hear your reasoning, not just see a diagram. A senior candidate can change direction when new constraints appear and explain why the design changes.

### Q4. What should you clarify before drawing the architecture?

Start with the users and their core journeys: what must they be able to do, which result counts as success, and what must never go wrong? Identify business invariants early—for example, no duplicate charge, no double booking, tenant isolation, or durable acceptance of a job. Those invariants often determine the data model and consistency boundary more strongly than the choice of framework.

Next ask about scale and workload shape: active users, average and peak requests/second, burst duration, read/write mix, payload size, fan-out, data growth, retention, and geographic distribution. Clarify measurable quality targets such as p95/p99 latency, availability, freshness, RTO/RPO, and cost constraints. For external dependencies, ask about quotas, latency, error behavior, and whether operations are idempotent.

Ask which operations are synchronous and which may complete asynchronously. A notification platform with a two-second delivery goal is different from bulk email that may take an hour. A catalogue can tolerate stale descriptive data, while a payment or inventory write may need an authoritative transaction.

Prioritize questions that could change the architecture. If exact numbers are unavailable, state reasonable assumptions, show the arithmetic, and continue. An interview is not a requirements workshop with unlimited time; the objective is to uncover the few constraints that materially change the design and make the remaining assumptions explicit.

### Q5. How do functional and non-functional requirements differ?

Functional requirements describe **what the system does**: create an order, reserve a seat, send a notification, search products, or upload a file. Non-functional requirements describe the constraints and quality of that behavior: latency, throughput, availability, durability, freshness, privacy, security, retention, recoverability, and cost.

The distinction matters because the same feature can lead to different architectures under different targets. A daily report can run as a batch job; a dashboard that must update within a second may need streaming ingestion and a materialized read model. A payment endpoint that can take minutes may return a durable operation ID and expose pending status, whereas a low-latency lookup should usually complete within a short request deadline.

Make non-functional requirements measurable wherever possible. “Fast” should become a target such as p99 below a specified threshold at expected peak load. “Highly available” should identify the eligible operations, measurement window, and expected behavior during a dependency or regional failure. “Reliable” should include correctness and recovery, not just successful HTTP responses.

Also distinguish global requirements from per-operation requirements. A product can tolerate eventual consistency for search and analytics while requiring transactional correctness for inventory. In interviews, tie each quality target to a design choice and an observable metric rather than treating non-functional requirements as a generic checklist.

### Q6. How do you handle an ambiguous prompt such as “design a notification system”?

Turn the broad prompt into concrete journeys and promises. Ask which channels are needed (email, SMS, push, in-app), expected volume and peak bursts, delivery deadlines, user preferences/quiet hours, provider quotas, retries, priority classes, retention, and whether notifications may expire. Clarify what “delivered” means: accepted by your API, durably queued, accepted by the provider, confirmed by a carrier, or acknowledged by the recipient's device. These are different guarantees.

Identify which work must be synchronous. The API may validate the request and durably record the notification, then return an operation ID while provider delivery runs asynchronously. But “accepted” must mean the service has recorded responsibility; an in-memory task that disappears on restart is not durable acceptance.

Then design around the failure cases: duplicate source events, provider timeout after possible send, provider throttling/outage, invalid recipients, out-of-order callbacks, and a backlog that misses its deadline. Use stable operation identities, an outbox or durable enqueue path, channel/priority queues, provider-aware rate limits, bounded retries with jitter, a monitored dead-letter path, and reconciliation for uncertain outcomes.

Keep analytics and noncritical personalization off the critical acceptance path when possible. In an interview, state which notifications may be delayed or dropped, how a caller can query status, and what metric measures end-to-end completion rather than just API success.

### Q7. How do you avoid overengineering?

Begin with the simplest design that meets the stated requirements, and identify which limit is likely to appear first. A modular monolith with a relational database may be ideal for a small team and moderate traffic. A broker is justified when work must be durably deferred, independently retried, or consumed by multiple subscribers; a cache helps when repeated reads are expensive and freshness is manageable; a search engine is justified when query semantics exceed the practical strengths of the primary datastore.

Every component creates operational costs: deployment, configuration, monitoring, security, capacity planning, backup/restore, upgrades, incident response, and new failure modes. A service mesh, separate read model, or multi-region active-active setup is not automatically an improvement just because it appears in mature architectures.

Ask what requirement each component satisfies, what new guarantee or capability it provides, and what failure scenario it introduces. If the justification is only “we might need it someday,” consider documenting an evolution path and the measurable trigger for adding it instead.

Senior-level pragmatism is neither “keep everything in one service forever” nor “split everything into microservices.” Choose the least complex architecture that meets the current contract, protect the important invariants, and explain how you would evolve it when load, team structure, or product requirements change.

### Q8. What makes an architecture diagram useful?

A useful HLD diagram makes the important architectural choices visible. It should show the main actors and trust boundaries, major components, synchronous calls, asynchronous events, authoritative data stores, caches/read models, and any important third-party dependencies. Draw the critical user journey first, then add components only when they represent a responsibility or constraint worth discussing. A diagram with six named boxes can be stronger than one with twenty generic boxes because the interviewer can follow the flow and understand ownership.

Use visual conventions consistently. For example, solid arrows can represent request/response calls and dashed arrows can represent asynchronous events; label arrows with the operation or event, and annotate important guarantees such as “durable acceptance,” “read replica; eventual,” or “authoritative reservation.” Do not imply a synchronous guarantee merely by drawing a line. Show where authentication/authorization occurs and where untrusted input crosses a boundary.

A generic backend flow might look like this; it is a starting point, not a design to paste into every problem.

```mermaid
flowchart LR
  C[Client] --> E[Edge / API Gateway]
  E --> A[Stateless Application]
  A --> D[(Authoritative Database)]
  A --> K[(Optional Cache)]
  A --> O[Outbox / Event Publisher]
  O --> B[Message Broker]
  B --> W[Async Workers]
  W --> X[External Provider]
```

Annotate the request's critical path and the failure behavior. If the cache is unavailable, where does traffic go? If the broker is down, can the API still accept durable work? If the provider times out after doing the side effect, where is the operation identity stored? These questions make the diagram useful as an architecture argument instead of decoration.

Keep data ownership explicit. A service should have one authoritative owner for each important business fact, even if several read models contain copies. Include regional boundaries, replication directions, queues, or sharding only when they change correctness, capacity, or recovery. In an interview, draw the simplest viable baseline and then deepen the specific bottleneck or failure mode the interviewer probes.

### Q9. How should you respond when the interviewer changes a requirement?

Restate the new requirement and identify which assumption it invalidates. Then trace its impact through user-visible behavior, business invariants, data model, request path, consistency, capacity, security, recovery, and cost. Do not start the whole design again unless the new requirement changes a fundamental boundary; adapt the affected parts and explain why.

For example, if notifications that could arrive within several minutes must now arrive within two seconds, you may need priority queues, reserved provider capacity, tighter deadlines, pre-rendered templates, and a different retry policy. But a strict latency target cannot make an external provider more reliable. You must also explain whether the system rejects, queues, or reports pending status during provider outage.

If the requirement changes from eventual to strong consistency, identify where coordination is needed and what availability/latency trade-off follows. If the requirement becomes global, account for network distance, data residency, replicated state, and safe writer ownership rather than adding a global load balancer alone.

In an interview, state the consequences and ask whether the business accepts them. A senior candidate shows flexibility without hiding the costs of a changed requirement.

### Q10. Why do candidates underperform in HLD interviews?

Common failures are designing before clarifying the business invariant, skipping rough capacity estimates, choosing technology by popularity, and drawing boxes without defining data ownership or flow. Candidates may say “use caching,” “add Kafka,” or “shard the database” without explaining which bottleneck the component addresses, what guarantees it provides, or what new failure modes it introduces.

A second failure is assuming distributed operations are atomic or exactly once. A timeout does not prove a write failed; a broker acknowledgement does not mean a consumer completed the business action; a replica may be stale; and a lease may expire while a worker continues running. Senior interviews often probe these crash windows because the architecture's correctness depends on them.

Another common gap is operational realism. A design may scale during normal traffic but fail during cold-cache startup, a zone outage, provider throttling, schema migration, or backlog replay. Adding application instances can increase database connections and downstream pressure rather than improve throughput.

Improve by making the reasoning explicit: requirement and invariant → workload/estimates → proposed mechanism → alternative and cost → failure behavior → observable proof. Admit uncertainty when a product-specific guarantee needs validation and explain how you would test it. Interviewers are testing judgment and the ability to reason through unknowns, not whether the candidate memorized a standard diagram.

### Q11. How do you know the design is complete enough?

You should be able to walk through the main user journey from request to response and identify the authoritative owner of every important fact. State the business invariant the design protects, the API/event contract, the main synchronous and asynchronous flows, and the consistency guarantee the caller can rely on.

Show a plausible capacity estimate: peak request/write rate, data growth, downstream fan-out, connection demand, queue drain, and headroom during a node/zone failure. Identify the likely bottleneck and explain which metric would reveal it. If the design uses caches or replicas, state the staleness contract and what happens when lag exceeds it.

Then walk through failure scenarios: a request times out after committing, a consumer crashes after a side effect before acknowledgement, a shared dependency becomes slow, a cache goes cold, a region is lost, or a restore is needed after corruption. Include security/tenant boundaries, observability, rollout/rollback, and cost.

You do not need to specify every class or configuration value. Instead, name unresolved assumptions and explain how you would validate them with load tests, failure injection, query plans, restore exercises, or business-level reconciliation. The design is complete enough when major decisions, guarantees, limitations, and validation plans are explicit.

### Q12. How should you communicate a trade-off?

Make the reasoning conditional on a requirement and say what you gain, what it costs, and what could make you choose differently. For example: “Because this flow must prevent duplicate seat reservations, I would enforce uniqueness or a conditional state transition in the authoritative database. That gives a clear correctness boundary, but contention on a very hot event may limit throughput. If the business supports temporary allocation or partitioned inventory buckets, I would evaluate those approaches next.”

Avoid statements such as “Redis is faster,” “NoSQL scales better,” or “microservices are more available” without describing the workload and guarantees. A cache can lower database load but add a network hop and stale-data risk. A read replica can scale eligible reads but return an older value. Sharding can increase independent capacity but complicate cross-shard queries, uniqueness, transactions, backups, and rebalancing.

A strong trade-off answer includes failure behavior and observability: what happens when the component is unavailable, which metric confirms the choice is helping, and what condition triggers an architectural change. Also state the assumptions. If the load estimate is illustrative, say so.

Senior-level judgment is not about finding a universal best technology. It is about showing why a decision is appropriate for the present requirements, where it is weak, and how you would evolve or roll it back safely.

## 2. Requirements and quality attributes

### Q13. Explain scalability, availability, reliability, durability, and fault tolerance.

These terms describe different properties, so avoid using them as synonyms:

- **Scalability** is the ability to handle more work by adding resources or changing the design without unacceptable degradation. Adding application instances helps only if the database or another shared dependency is not already the bottleneck.
- **Availability** is the likelihood that a request receives a usable response within the promised conditions and time. A process returning HTTP 200 is not enough if the response violates the business contract.
- **Reliability** is the broader ability to keep performing the intended function correctly over time. It includes more than whether an endpoint is reachable.
- **Durability** means acknowledged data survives the failures the design claims to tolerate. The guarantee depends on the database's commit and replication settings, not merely on having multiple copies.
- **Fault tolerance** is the ability to continue providing an acceptable level of service despite specified failures, often through redundancy, isolation, or graceful degradation.

For example, a catalogue can remain available while showing a slightly stale product description. A payment service should not preserve availability by returning an unverified balance or accepting conflicting debits. A durable database may still be temporarily unavailable during failover.

### Q14. Latency versus throughput: what is the difference?

**Latency** is the time an operation takes from the relevant start point to completion. **Throughput** is the amount of work completed per unit of time, such as successful requests/second or events/second. They are related, but they are not interchangeable. Batching can raise throughput while making an individual request wait longer; conversely, a service can have low latency at light load but collapse as arrival rate approaches its sustainable capacity.

Queueing is the key connection. When incoming work approaches the rate a component can complete, small increases in load can cause a disproportionate increase in waiting time. If arrivals exceed completions for long enough, queues grow without bound. Therefore, report successful throughput and latency together—high throughput made mostly of retries or work completed after its deadline is not useful capacity.

Measure latency by stage: queue wait, connection acquisition, application processing, database time, downstream calls, and response transfer. Distinguish average latency from percentiles, and separate request latency from time-to-complete for asynchronous work. A service that returns 202 in 20 ms but takes 15 minutes to finish the operation has low acceptance latency, not necessarily good end-to-end latency.

In an interview, state which metric represents the user's objective and which shows resource capacity. For a real-time API, p95/p99 and successful requests within deadline may be decisive. For a batch pipeline, sustained processing rate and time-to-drain may matter more, provided data correctness and completion deadlines are preserved.

### Q15. Why do P95 and P99 latency matter?

P95 is the value at or below which 95% of measured requests complete; P99 is the corresponding threshold for 99%. They reveal the **tail** of the latency distribution, which averages can hide. If average latency is 80 ms but p99 is 2.5 seconds, a meaningful subset of users is experiencing a very different service than the average suggests.

Tail latency often comes from queueing near saturation, hot keys, lock contention, slow database plans, connection-pool waits, garbage-collection pauses, network loss, retries, or an unusually slow downstream dependency. A fan-out endpoint can be dominated by its slowest required dependency even when the others are fast. Retries may improve eventual success while increasing p99 and consuming extra capacity.

Percentiles must be interpreted carefully. State the endpoint/request class, measurement window, traffic population, success/error treatment, and whether the percentile is measured at the client or server. Low-volume services can have unstable p99 estimates, and comparing percentiles from different workloads or windows can mislead. Per-route and per-tenant views can reveal a hotspot that a fleet-wide percentile hides.

Use traces to decompose the slow path and metrics to track trends. A useful investigation compares successful throughput, timeout rate, queueing, saturation, and p95/p99 before and after a change. In interviews, explain which user journey has the tail problem, where its time is spent, and which intervention reduces the tail without merely moving the queue downstream.

### Q16. What are SLI, SLO, and SLA?

An **SLI (service-level indicator)** is the actual measurement of a service outcome—for example, the percentage of eligible requests that return a correct response within 300 ms, or the percentage of accepted export jobs that finish within ten minutes. An **SLO (service-level objective)** is the target for that indicator over a defined window, such as 99.9% of eligible requests succeeding over 30 days. An **SLA (service-level agreement)** is an external commitment to customers and may specify remedies, service credits, and how availability is measured if the commitment is missed.

The distinction is useful because teams can meet an infrastructure metric while failing the user's journey. If HTTP 200 responses are returned while orders remain stuck in PENDING, an availability SLI based only on status codes can look healthy while business completion is broken. For asynchronous APIs, measure both durable acceptance and end-to-end completion if both matter. For a search system, response latency and freshness may be separate indicators.

Define the denominator carefully: which requests are eligible, what counts as success, whether client errors are excluded, whether retries count as additional attempts, and what observation window applies. Time-based uptime and request-based success are not interchangeable; an outage during a high-traffic campaign can consume many more failed requests than an outage at a quiet time.

SLOs should guide engineering decisions. Track error-budget burn rate, set actionable alerts, and agree how risky releases or reliability work change when the budget burns too quickly. Avoid setting targets without understanding the cost of meeting them: a more stringent SLO may require redundancy, better capacity headroom, stronger consistency, or less feature work. A senior answer explains how the SLI reflects business value, how it is measured, and what action the team takes when it misses the objective.

### Q17. What is an error budget, and how should teams use it?

An error budget is the amount of unreliability permitted by an SLO during its measurement window. If the SLO is 99.9% successful requests over 30 days, the allowed failure fraction is 0.1% of eligible requests. For a time-based objective that treats the whole 30-day period uniformly, 0.1% of 30 days is about 43.2 minutes of unavailability. A request-based SLO and a time-based SLO measure different things, so do not convert one to the other without stating assumptions.

The budget is useful because it makes the reliability-versus-change trade-off explicit. When error-budget consumption is healthy, teams can continue shipping within the agreed risk policy. If the budget burns too quickly, a team may pause risky releases and prioritize reliability work, capacity fixes, or dependency remediation. This should be a pre-agreed operating policy rather than an improvised punishment after an incident.

Track burn rate as well as remaining budget. A service can consume a month's budget in a short window even when its 30-day availability chart still looks acceptable. Use alerts that identify fast burn and connect it to user impact. Define eligible traffic, what constitutes success, exclusions, and the measurement window; otherwise the SLO can be gamed unintentionally.

The error budget is not a target to spend and not permission to neglect low-volume but high-severity correctness failures. A payment system may meet its availability SLO while still incorrectly charging customers. Pair service-level indicators with business invariants, incident review, and controlled release practices.

### Q18. How do you estimate end-to-end availability across dependencies?

If a request succeeds only when every required dependency succeeds, and their failures are independent, multiply their availability estimates. For example, three serial dependencies each available 99.9% of the time give roughly (0.999^3 = 99.70%) end-to-end availability. The user journey is less available than any one dependency because any required component can cause the request to fail.

This is a first-order estimate, not a precise production prediction. Failures are often correlated: several services may share a database, network, identity provider, control plane, deployment, or external quota. Conversely, caching, bounded retries within the caller's deadline, replicas, and safe fallbacks may preserve user-visible success even when a component fails. The model needs to reflect which components are truly required for the specific operation.

Use the dependency graph to classify each edge as required or optional. If recommendations fail, a product page might still succeed without that section. If payment authorization fails or its outcome is unknown, returning a fabricated success is not an acceptable availability improvement. Likewise, a fallback to a stale cache is only valid when its freshness and security contract permits it.

Measure availability at the user-journey boundary and separately track dependency health to aid diagnosis. Define the eligible request set and what counts as success; HTTP 200 alone can misrepresent business completion. In an interview, show the simple calculation, state its independence assumption, then explain graceful degradation, correlated failure, and which correctness-critical operations must fail safely.

### Q19. What is graceful degradation?

Graceful degradation means preserving the most valuable and correct parts of a service when a dependency is unavailable or the system is overloaded. A shopping page might display product details but omit recommendations; a video service might lower resolution to sustain playback. The degraded behavior should be designed and tested rather than improvised during an incident.

Start by separating critical dependencies from optional capabilities for each user journey. For a read endpoint, a cached description may be safe if its freshness is within the contract. For checkout, a stale inventory count may be fine for display but must not authorize the final reservation. For payments, a timeout must be represented as pending/unknown until status is reconciled—not converted into success or a definitive failure without evidence.

A degraded mode has to preserve security, business invariants, and API meaning. Do not return an empty list when the correct semantic is “search unavailable” unless the contract distinguishes those outcomes. Expose partial-result metadata where clients need to know completeness. Decide how long stale data may be served, what happens after that limit, and how the system recovers when the dependency returns.

Graceful degradation also requires capacity planning. Serving cached data or switching to a secondary provider can overload the fallback if every request moves at once. Use bounded concurrency, load shedding, priorities, and controlled recovery. In an interview, state what capability is sacrificed, which invariant remains protected, what the caller sees, and how metrics show the degraded mode is active.

### Q20. How do you prioritize conflicting quality attributes?

Start from business impact and the cost of an incorrect result instead of trying to maximize every attribute simultaneously. Identify which errors are reversible, which are costly, and which would violate a financial, security, safety, regulatory, or contractual invariant. The right trade-off can vary by operation inside the same system.

For example, a catalogue may remain available with slightly stale descriptions during a regional outage, while checkout must revalidate price and inventory. A payment service may reject a write when it cannot establish an authoritative owner rather than risk a duplicate charge. That design sacrifices write availability in some failure states to preserve correctness. Conversely, analytics can often accept delayed events and catch up later.

Make quality attributes measurable: p99 latency, availability SLO, maximum staleness, RTO/RPO, durable acceptance, cost per successful operation, or the permitted error rate. If the requirements conflict, show which one wins and why. Explain the user-visible degradation and whether the system fails open, fails closed, queues work, or returns a pending state.

Do not label a design “highly available” without saying what happens during partition, dependency loss, or overload. A senior answer explains the trade-off for a specific invariant and states what evidence could change the decision—for example, a new latency target, legal requirement, or verified alternative consistency protocol.

### Q21. What is blast radius?

Blast radius is the scope of impact when something fails: the number of users, tenants, regions, operations, or business functions affected. A failure isolated to a recommendation worker has a smaller blast radius than a shared identity-provider failure that prevents every service from authenticating requests. A bad release deployed everywhere can turn a minor code issue into a platform-wide outage.

Reduce blast radius by isolating workloads and failure domains: bulkheads, bounded queues, per-tenant quotas, separate capacity for critical traffic, staged rollouts, canaries, and independent regional resources where justified. Ensure one noisy tenant cannot exhaust shared connection pools or one poison message cannot block unrelated work. For stateful systems, isolate write ownership and prevent a stale leader from committing.

Redundancy alone is insufficient if replicas share the same database, credentials, network, control plane, policy store, or faulty release. Similarly, adding service instances may expand the blast radius if they all connect to a saturated shared dependency. Model what happens after losing one instance, zone, region, or critical service—not only the normal healthy topology.

Track impact by user journey and tenant, time to detect, time to mitigate, and the number of components affected. In interviews, identify the most dangerous shared dependency and explain what boundary limits its impact. Then describe how you would test that boundary through a staged rollout or controlled failure exercise.

## 3. Capacity estimation

### Q22. Estimate requests per second from daily active users.

Start with user behavior and convert it to traffic; do not jump straight from user count to server count. Suppose 2 million daily active users each make 40 API requests per day. That is 80 million requests/day, or about 926 requests/second on average because (80{,}000{,}000 / 86{,}400 approx 926). If peak traffic is four times the average, the initial peak estimate is roughly 3,700 requests/second.

Then refine the model. Not every user is active at the same time; requests are not evenly distributed across the day; and a single user action may fan out into multiple backend calls. If 30% of API requests trigger three downstream calls, the average internal-call multiplier is higher than one. Also account for retries, polling, scheduled jobs, mobile reconnects, bulk operations, and separate read/write mixes. Do not multiply every request by every downstream service unless the request path actually calls all of them.

Estimate each major resource separately: edge requests, application CPU, database reads and writes, cache lookups, broker events, external provider calls, and network bytes. The bottleneck may be database writes or a third-party quota rather than API QPS. Use measured request distributions and load testing to validate the assumptions. In an interview, label every number as an assumption, show the arithmetic, and identify which assumption has the greatest impact on capacity.

### Q23. Why is average traffic insufficient?

Average traffic hides the distribution of demand over time. A service averaging 1,000 requests/second may receive 5,000/s during a campaign or flash sale. If capacity is provisioned only for the average, queues grow, tail latency rises, clients time out, and retries can add even more load. The length of the burst matters as much as its peak.

Estimate peak-to-average ratio, burst duration, read/write mix, request cost, and downstream fan-out. A request that triggers three database reads and a provider call is more expensive than a cache hit. Also include background jobs, message consumers, retries, cache misses after deployment, mobile reconnects, and batch replay. An average external-provider quota can hide brief bursts that exceed the provider's fixed limit.

Whether excess demand may wait depends on the business contract. A report export can be durably queued and return an operation ID; a seat reservation may need a fast rejection or waiting room rather than allowing requests to time out while competing for the final seat. Admission control and per-tenant fairness are often more effective than simply adding instances.

Load-test steady-state, burst, cold-cache, and dependency-degraded conditions. Track successful throughput, p95/p99, queue age, saturation, and rejected work. In interviews, state how peak assumptions were chosen, which resource saturates first, and whether the system should queue, shed, or reject excess work.

### Q24. How do you estimate storage?

Estimate logical data first, then account for physical overhead and lifecycle. A basic model is:

```text
logical data = new records per day × average stored bytes per record × retention days
```

Suppose a system stores 5 million events/day at an average 1.5 KB each and keeps 180 days online. Raw payload is roughly (5M × 1.5	ext{ KB} × 180), or about 1.35 TB in decimal units. This is not the required provisioned disk. Add indexes, row/page overhead, metadata, write-ahead logs, temporary space, compaction or vacuum headroom, and safety margin. A relational table with multiple indexes can take substantially more physical space than the application payload alone.

Next distinguish logical from physical copies. Three replicas do not mean the logical dataset tripled in business meaning, but the storage fleet must hold multiple physical copies. Backups and point-in-time recovery logs may require additional storage and retention; snapshots may be incremental or full depending on the product. Search indexes, caches, and materialized read models can create additional derived copies.

Estimate growth, not just launch-day size. Include update churn, deletion/retention requirements, peak ingestion, compaction, and restoration bandwidth. If data is time-partitioned, old partitions may be detached or archived more efficiently than row-by-row deletion, depending on the database's capabilities. Finally, distinguish hot storage from archive/object storage and explain retrieval time, cost, legal retention, encryption, and deletion requirements. Validate the estimate against real serialized records and observed index/storage ratios.

### Q25. How do you estimate bandwidth?

Estimate ingress and egress separately using traffic rate and bytes per operation. If the service receives 4,000 requests/second with an average request size of 2 KB, ingress payload is about 8 MB/s before protocol overhead. If responses average 12 KB, egress is about 48 MB/s. These are decimal payload estimates; TLS records, TCP/IP headers, retransmissions, connection setup, compression, and uneven peaks increase or change actual wire traffic.

For internal traffic, follow the request path rather than counting only client bytes. An API request can trigger several downstream RPCs, copy data into a broker, replicate writes to multiple nodes, and update a search index. Cross-zone and cross-region transfers may have both latency and monetary costs. Replication traffic is related to write volume and payload/index/log behavior, not simply the number of API requests.

Use peak rates and bursts for network interface sizing, and measure both throughput and packet/connection behavior. A workload with small messages can be limited by packets per second or connection churn even when its Mbps looks modest; a bulk export may saturate bandwidth while using little CPU. Compression helps only if CPU cost and content characteristics make it worthwhile. Identify where CDN caching, direct object-storage uploads, batching, streaming, or data locality reduce expensive transfers. Treat estimates as a starting point and validate with load tests, packet/bytes metrics, and the actual cloud billing dimensions.

### Q26. How do you estimate queue processing capacity?

Compare the arrival rate with the sustainable completion rate under realistic processing cost. Let (lambda) be arrivals per second and (mu) be completed messages per second. If (lambda > mu) for a sustained period, the backlog grows at approximately (lambda-mu). Adding a queue absorbs a temporary mismatch; it does not solve a permanent throughput deficit.

For a burst, estimate backlog and drain time. If 1.2 million events arrive during an incident, workers can complete 8,000/s, and new arrivals continue at 3,000/s, the net drain is 5,000/s. Ideal drain time is (1{,}200{,}000/5{,}000=240) seconds, or four minutes, if both rates stay constant. If arrival rate is at or above service rate, the backlog will not shrink.

Measure throughput at the full processing boundary, including database calls, provider quotas, retries, and per-key ordering constraints. A worker that dequeues 10,000 messages/s but commits only 2,000/s is not a 10,000/s worker. Estimate concurrency from operation duration, but avoid increasing concurrency beyond downstream connection or rate limits. Consider partition count, consumer-group parallelism, batch size, lock contention, message size, and failure/replay costs.

Track queue age as well as depth: ten million old events and ten million newly arrived events have different user impact. Set capacity and recovery targets explicitly, bound admission, define expiry/dead-letter rules, and load-test what happens when a dependency slows down rather than assuming the happy-path processing rate will remain constant.

### Q27. How do retries affect capacity?

Capacity must be sized for **attempts**, not only original logical requests. If 1,000 requests/s each trigger two retries, the dependency can receive up to 3,000 attempts/s for that traffic. The real amplification may be larger when retries occur at multiple layers—client, gateway, service library, mesh, and broker—or when timed-out operations continue running while retries begin.

Retries can help transient failures but also create positive feedback: a dependency slows, requests time out, callers retry, concurrency rises, and the dependency becomes slower. During an outage, retries may consume the capacity needed for fresh requests and delay recovery. Attempts can also duplicate side effects if the operation is not idempotent.

Use bounded attempts, exponential backoff with jitter, an end-to-end deadline, and a retry budget. Classify failures: retrying validation errors or permanent business rejection does not help. Assign a clear retry owner where possible and avoid retries that cannot complete within the remaining deadline. For mutations, use a stable idempotency/operation identity and reconcile uncertain outcomes; a timeout does not prove the first attempt failed.

Measure original requests separately from physical attempts, retry success rate, downstream saturation, timeout rate, and queue age. In interviews, quantify the worst-case amplification and explain how the retry policy changes during partial failure and recovery. Retrying more is not equivalent to higher reliability if it overwhelms the dependency.

### Q28. How do you estimate cache memory?

Estimate the number of keys that must be resident at the same time, then use measured key/value sizes and the cache engine's memory overhead. If five million values average 2 KB, the payload alone is about 10 GB in decimal units. The provisioned memory needs to be higher because keys, object metadata, allocator overhead, fragmentation, data structures, expiry metadata, and replication all consume resources. Actual overhead depends on the cache engine, key distribution, and data types, so sample real entries rather than relying only on application payload size.

Not every key must be cached simultaneously. Identify the active working set and reuse distribution, then choose an eviction policy consistent with access patterns. A cache that is too small may repeatedly evict popular keys; a cache that is too large may waste money, while a low-reuse workload may not benefit from a cache at all. Large values can dominate capacity even when they represent a small share of requests.

Treat TTL as part of the sizing model. A 15-minute TTL does not imply a fixed 15-minute working set if entries are refreshed, invalidated, or expire in bursts. Add TTL jitter for many similar keys and include the memory spike during warm-up or bulk refresh. If the cache is replicated or deployed across zones, estimate memory per node and total fleet memory separately.

Load-test with realistic keys, values, concurrency, and expiration behavior. Monitor used memory, fragmentation, evictions, hit ratio, origin load, and p99 latency. A high hit ratio does not guarantee value if cached values are expensive to maintain or if misses produce a stampede. In an interview, show the payload arithmetic, list the omitted overheads, and identify what evidence would change the capacity estimate.

### Q29. What changes when traffic grows 10×?

Do not simply multiply the server count by ten. A 10× increase can change bottleneck location, key popularity, cache behavior, query plans, contention, network fan-out, provider quota usage, and failure probability. Some costs scale linearly; others grow non-linearly near saturation or due to coordination.

Trace a representative user operation and apply the multiplier through its dependency graph. If 10,000 requests/s each cause two database reads and 0.2 writes on average, the database sees about 20,000 reads/s and 2,000 writes/s before retries, background work, and other callers. A cache may protect reads when hit ratio is high, but a cold-cache rollout could suddenly expose the database to the full read volume. A fixed external provider quota does not grow just because the application autoscaled.

Then ask which resource saturates first: CPU, memory/GC, connection pool, database locks, storage IOPS, broker partitions, cache bandwidth, load-balancer connections, or an external quota. Check whether the next tier has enough headroom during one zone/node loss; if capacity only works with every instance healthy, the system has no failure margin.

The correct response may involve query optimization, batching, admission control, per-tenant quotas, cache protection, asynchronous work, more partitions, or sharding—but only the observed bottleneck justifies the mechanism. Plan migration and operational complexity too. A senior answer describes how to scale in stages, what metrics trigger each stage, how the design behaves while scaling, and which failure tests prove it can sustain the new load.

### Q30. What is capacity headroom?

Capacity headroom is the usable capacity available above expected demand. It absorbs bursts, workload uncertainty, rolling-deployment surge, a node/zone loss, maintenance, and the extra traffic that can appear during failover. The right margin depends on burst speed, provisioning delay, request variability, and the failure model the service must tolerate.

Do not treat autoscaling as instant capacity. New instances take time to provision, start, fetch configuration, establish connections, and warm caches. If traffic rises in seconds, the system may need minimum warm capacity, admission control, a waiting room, or load shedding while scaling catches up. A service sized to 100% utilization under normal conditions has no room for recovery or variance.

Calculate capacity after a failure, not just with every node healthy. Two regions each running near 80% utilization cannot necessarily absorb all traffic after one region fails, even if the combined nominal capacity looks adequate. Also account for bottlenecks that do not scale with the application tier, such as a primary database, global lock, fixed external quota, or hot partition.

Measure successful throughput and latency under normal peak and at least one failure scenario. Headroom should be tied to an explicit SLO and tested with load/stress exercises rather than a generic percentage. In interviews, state which failure must be tolerated, which component limits throughput, and how fast spare capacity becomes usable.

## 4. Networking and request delivery

### Q31. What happens when a client makes an HTTPS request?

At a high level, the client resolves the hostname through DNS, opens or reuses a transport connection, negotiates TLS when needed, sends the HTTP request, and receives a response. In a typical system, the request may pass through a CDN, WAF, public load balancer, API gateway, and one or more application services before reaching a database or external dependency. The exact path depends on DNS, network topology, connection reuse, and where TLS is terminated.

**DNS and connection:** DNS maps the hostname to an address or routing endpoint, but answers can be cached, so changes are not an instantaneous global failover mechanism. The client may need a TCP handshake (for HTTP/1.1 or HTTP/2 over TCP) or use QUIC for HTTP/3. Persistent connections reduce setup overhead but can remain tied to a backend or region after DNS changes.

**TLS:** During the TLS handshake, the endpoints negotiate cryptographic parameters and authenticate the server through its certificate chain; optional client certificates can authenticate the client. Session resumption can reduce handshake cost. If TLS terminates at a proxy, the proxy becomes a trust boundary; the proxy-to-backend hop still needs protection where the network is not inherently trusted. TLS protects data in transit but does not replace application authorization.

**HTTP routing and processing:** The proxy/load balancer selects an eligible target, potentially based on L4 connection state or L7 HTTP attributes. The application authenticates/authorizes the request, validates input, and performs synchronous work or durably accepts asynchronous work. It may wait on thread pools, connection pools, locks, storage, or downstream calls. Those waits often dominate application execution time.

**Response and diagnosis:** The response returns through the same logical path, and each hop can add queueing, retries, buffering, or timeout behavior. Persistent connections, TLS resumption, compression, CDN caching, and HTTP/2 multiplexing can reduce costs but do not fix a saturated database or unsafe retry policy. When diagnosing latency, break down DNS, connect, TLS, server wait, upstream calls, and response transfer rather than blaming application code by default.

### Q32. Layer 4 versus Layer 7 load balancing?

A **load balancer** distributes incoming connections or requests across a pool of backends. Its job is not simply to “spread traffic evenly”: it must choose eligible destinations, avoid unhealthy instances, respect connection and timeout behavior, and keep the system within capacity. Load balancing may happen at several layers of one request path—for example, DNS or a global traffic manager chooses a region, a Layer 4 balancer chooses a node, and a Layer 7 proxy chooses an application service.

A **Layer 4 (L4) balancer** makes decisions using transport-level information such as source/destination IP addresses, ports, and TCP/UDP connection state. It usually does not need to parse HTTP paths or headers. This can make it efficient and useful for generic TCP services, database proxies, or high-throughput connection distribution. The trade-off is that it cannot make rich request-level decisions such as routing `/payments` differently from `/catalogue` based on HTTP semantics.

A **Layer 7 (L7) balancer** understands an application protocol such as HTTP. It can route by hostname, path, headers, cookies, method, or other request attributes; terminate TLS; enforce request limits; and support canary routing. That flexibility costs processing, configuration complexity, and a larger security-sensitive surface. TLS termination also changes the trust boundary: the proxy-to-backend hop must be protected where the network is not inherently trusted.

**Example:** `api.example.com/orders` may reach an L7 ingress or gateway, which routes to the order service. That service may itself be reached through an internal L4 or L7 balancer. The L7 gateway should not be assumed to replace internal discovery, database connection management, or application-level authorization.

Both layers need readiness-aware health checks, bounded connection/request queues, sensible timeouts, connection draining during deployment, and observable per-backend traffic. A health check can tell the balancer that a target should receive no new requests; it cannot prove that a database write is safe or that an old leader has lost write authority. Nor does a load balancer remove a bottleneck in a shared database or external provider.

### Q33. Which load-balancing algorithms are common, and when would you use each?

The algorithm is only one part of the decision. First determine what is being balanced—TCP connections, HTTP requests, long-lived streams, or work units—and whether backend capacities and request costs are similar.

- **Round robin** rotates through eligible backends. It is simple and works reasonably well when instances have similar capacity and requests have similar cost. It can be unfair when one request runs for milliseconds and another for minutes.
- **Weighted round robin** assigns a larger share to stronger or intentionally preferred instances. Weights need to reflect real capacity and be adjusted carefully during partial degradation.
- **Least connections** prefers the backend with fewer active connections. It can help when connection duration correlates with work, but it is a poor proxy for load if a few connections can trigger expensive work or if HTTP/2 multiplexes many requests over one connection.
- **Least outstanding requests / least request** can be more meaningful for request-oriented traffic, but it still needs accurate in-flight accounting and does not know the full cost of each request.
- **Random choice (often power of two choices)** samples a small number of backends and chooses the less loaded one. It can avoid centralized load-measurement overhead while achieving good distribution in large pools.
- **Consistent hashing** maps a key (such as tenant ID) to a stable backend or cache shard. It helps with locality and reduces remapping during topology changes, but hot keys can overload one target and membership changes require a well-defined ring or routing map.
- **Source-IP or cookie affinity** keeps a client on the same backend. This can help legacy stateful applications, but it reduces flexibility and can create imbalance; it should not be a substitute for shared session storage in a horizontally scaled design.

In an interview, state the assumption behind the choice. For ordinary stateless HTTP APIs, round robin or a request-aware policy plus health checks may be sufficient. For long-lived connections, account for connection duration and draining. For tenant affinity, use hashing only if locality is worth the skew risk. Always measure successful throughput, tail latency, queue depth, and backend imbalance rather than judging the policy by even request counts alone.

### Q34. Are health checks enough for safe failover?

No. A health check is a signal, not a complete failover protocol. **Liveness** asks whether a process should be restarted; **readiness** asks whether it should receive new traffic. A process can be alive but not ready, and a shared database outage can make every instance fail a dependency check even though restarting the instances will not fix the database.

Health checks have limits. A TCP probe may show that a port accepts connections while every business request fails. A deep readiness probe can be too expensive or depend on a shared service; if every instance becomes unready at once, the load balancer can remove all capacity and make the outage worse. Use cheap bounded probes, thresholds/hysteresis, and separate diagnostic signals from routing/lifecycle decisions.

Planned removal needs a sequence: mark the instance unready, stop assigning new work, drain in-flight requests, and terminate within a deadline. Long-lived WebSockets and message consumers need protocol-specific drain/reconnect behavior. A health probe also cannot prove that a stateful writer has lost authority. Leader promotion requires an explicit ownership protocol and fencing at the protected resource.

Failover should evaluate data safety, not just reachability: is the candidate sufficiently caught up, what RPO is promised, and can the old writer still commit? In an interview, distinguish traffic health from business-path health and explain the action triggered by each signal. Test shared dependency failure, flapping health, and stale leaders—not only a process crash.

### Q35. What are the limitations of sticky sessions?

Sticky sessions use a cookie, source IP, or other affinity key to route a client repeatedly to the same backend. Affinity can help a legacy application that stores session state in process memory or benefits from local caches, but it makes traffic distribution less flexible. A few very active users can overload one instance while other instances remain idle, and the instance becomes a hidden dependency for those sessions.

If the selected backend fails or is removed during deployment, a new backend may not have the session state. The user may be logged out, lose workflow progress, or encounter inconsistent behavior. IP-based affinity is especially brittle for mobile clients and users behind NAT, where one IP can represent many users and a user’s IP can change. Cookie affinity is more precise but depends on correct cookie/security behavior and load-balancer support.

Prefer stateless application instances where practical, with session state in a shared durable store or a properly protected client token. This improves failover and horizontal scaling, but shared session storage introduces its own latency and availability considerations. Client-held tokens should not contain mutable secrets or become a substitute for server-side revocation when immediate invalidation is required.

Sticky routing can still be justified for long-lived connections or cache locality, but isolate the stateful behavior and plan draining/failover. For WebSockets, affinity may preserve the live connection, but after reconnect the client may land on another instance and needs a mechanism to recover missed events. Monitor per-instance load, session migration/failure, reconnect rate, and imbalance.

In an interview, explain the specific benefit of affinity and the failure mode it introduces. “Sticky sessions solve state” is incomplete unless the state and recovery contract are explicit.

### Q36. How should timeout budgets be designed?

Start from the user-visible latency objective and set one **end-to-end deadline** for the operation. Allocate part of that budget to each required step—queueing, connection acquisition, downstream calls, database work, commit, and response serialization—while keeping enough time for the caller to receive a result. If an API deadline is one second, a downstream call should not independently wait two seconds. Nested operations should use the remaining budget rather than restart the clock.

Understand what each timeout actually covers. A client-library “read timeout” may exclude DNS, connection-pool acquisition, TLS, or request upload time. A connection timeout may only cover opening a socket, not the full server operation. An idle timeout for a long-lived connection has different semantics again. Misunderstanding these differences can leave requests waiting much longer than expected.

Set timeouts from observed latency distributions and the cost of waiting. A timeout that is too short creates false failures and retries for work that would have completed; a timeout that is too long ties up threads, connections, and queue slots while users wait. Use per-dependency timeouts and circuit breakers as part of the same policy, and propagate deadlines across HTTP/RPC and async workflows where supported.

Retries consume the same budget. Before retrying, ask whether enough time remains for another attempt and whether repeating the operation is safe. A timeout means the outcome may be unknown for mutations; use an idempotency key or status query rather than assuming the remote operation failed. For asynchronous jobs, use a separate business deadline and retry/expiry policy instead of keeping the HTTP request alive.

Monitor timeout rate, time spent waiting at each hop, attempts per logical request, cancellation propagation, and late completion after caller timeout. In an interview, show the full budget across the call graph and explain what happens when the deadline expires at each layer.

### Q37. When should you use WebSockets, Server-Sent Events, or long polling?

**WebSockets** provide a persistent, bidirectional connection, making them useful for chat, collaboration, games, and interactive applications where client and server both send frequent messages. They require connection lifecycle management, heartbeats, authentication refresh, reconnect behavior, fan-out routing, and capacity planning for concurrent sockets and outbound buffers.

**Server-Sent Events (SSE)** provide a server-to-client event stream over HTTP. They are often simpler when the server pushes updates but the client sends commands through ordinary HTTP requests. Support for reconnect/event IDs makes resumption convenient, but you still need retention and replay semantics if events must not be lost. **Long polling** holds an HTTP request until data is available or a timeout occurs, after which the client reopens the request. It is easier in some environments but creates more request/connection churn and can produce synchronized polling spikes.

Choose based on directionality, expected concurrent connections, update frequency, client/proxy support, network behavior, and the required delivery guarantee. Mobile push notifications serve a different role: they can alert or wake an offline device, but should not be treated as a durable record of application state.

None of these transports alone guarantees durable delivery. Persist important events/state, use stable event IDs or cursors, handle duplicate replay, and provide a snapshot when a reconnect cursor has expired. Apply backpressure and bound per-client buffers so a slow client cannot consume unbounded memory.

In interviews, distinguish transport from delivery semantics. Explain what happens during disconnect, reconnect storm, missed events, authorization expiry, and gateway failure—and what data remains authoritative when the connection itself disappears.

### Q38. What does a CDN solve and not solve?

A Content Delivery Network (CDN) caches eligible assets or HTTP responses at edge locations closer to users. It can reduce geographic latency, origin request volume, and bandwidth costs, especially for versioned static assets and public content whose cache policy is clear. A CDN can also provide edge TLS termination, DDoS protection, and request filtering depending on the service.

It does not automatically make personalized or authorized responses safe to share. Cache keys, cache-control headers, cookies, query parameters, and variation dimensions must reflect everything that changes the response. Omitting tenant/user/authorization context can leak private data. Sensitive responses should be private/non-cacheable unless the architecture deliberately provides safe isolation.

A CDN also does not solve write consistency, an overloaded origin on cache misses, or stale business-critical decisions. TTL and purge behavior determine how quickly content changes become visible, but invalidation may be delayed and different edge locations can converge at different times. A cold cache or large purge can create an origin stampede unless request coalescing, rate limits, or origin shields are used.

Choose CDN caching for content with reuse and acceptable freshness, and keep authoritative writes at the owning service/database. Versioned URLs are often safer than purging immutable assets. Measure hit ratio alongside origin offload, cache-miss latency, stale-content incidents, purge duration, and origin saturation. In interviews, describe the cache key and invalidation contract, not just the geographic benefit.

## 5. APIs and service boundaries

### Q39. REST versus RPC/gRPC: how do you choose?

REST over HTTP is often a good fit for public, resource-oriented APIs because browsers, gateways, debugging tools, and client libraries support it widely. It uses familiar HTTP semantics and works well when clients need independent evolution, cacheable reads, and straightforward interoperability.

gRPC uses generated, strongly typed service contracts and Protocol Buffers, with streaming support and efficient binary serialization. It can fit internal service-to-service calls or high-volume communication where contract generation and streaming are valuable. But browser support may require a gateway/bridge, and operational debugging or proxy compatibility can differ from ordinary JSON/HTTP.

Do not choose based only on claims that one protocol is faster. End-to-end latency depends on network round trips, payload size, connection reuse, serialization/deserialization, concurrency, proxy behavior, and downstream work. A chatty gRPC call graph can be slower than a coarse-grained REST API; compact serialization will not fix a slow database or a saturated connection pool.

Also consider deadlines, cancellation, idempotency, API versioning, authentication, observability, and how clients handle partial failure. gRPC deadlines are useful only if they are propagated and enforced; the server still needs safe handling of a side effect after the caller times out. Benchmark representative traffic if performance is the deciding factor.

In interviews, state who the consumers are, which capabilities matter (streaming, browser compatibility, cacheability, generated contracts), and what the team can operate. Prefer the protocol that fits the contract and topology, not the one that sounds more advanced.

### Q40. How do you change an API without breaking clients?

Treat API compatibility as both a schema and behavior contract. Prefer additive changes that preserve existing field meanings—for example, adding an optional response field that old clients can ignore. A change can still be breaking when the JSON shape remains valid: changing `amount` from gross to net, changing ordering, altering pagination semantics, or making an optional field suddenly required can break consumers.

For incompatible changes, use a versioning or migration plan, publish deprecation timelines, measure which clients still use the old contract, and remove it only when the agreed policy permits. Versioning may use paths, headers, media types, or a new endpoint, but version labels alone do not protect against incompatible shared database writes or event schemas.

Rolling deployments require old and new versions to coexist. Deploy backward-compatible readers before new writers, use additive schema changes, backfill gradually, and retire old fields only after no supported code path needs them. For event APIs, account for delayed messages and replay: a new consumer may read an event published months ago, and old consumers may receive a new payload during rollout.

Use consumer/provider contract tests, mixed-version integration tests, deprecation telemetry, and rollback tests. Keep semantic guarantees documented: idempotency scope, error codes, rate limits, ordering, freshness, and whether HTTP 202 means durable acceptance or final completion. In an interview, explain the coexistence window and how you know when it is safe to remove the old behavior.

### Q41. Offset versus cursor pagination?

**Offset pagination** asks for a page after skipping a number of rows, such as `LIMIT 50 OFFSET 50000`. It is simple for clients, can support direct page-number navigation, and may be adequate for small or mostly static result sets. But the database may have to scan or discard many rows at large offsets; latency can grow with page number. Concurrent inserts/deletes before the offset can also make later pages contain duplicates or skip records.

**Cursor/keyset pagination** carries a position from the previous page, typically based on a stable sort key plus a unique tie-breaker such as `(created_at, id)`. The next query seeks from that position instead of skipping a large prefix, which can be more efficient when a matching index exists. The order must be deterministic, and the cursor should be opaque or integrity-protected so clients cannot alter filters, tenant context, or position. Revalidate authorization on every page; a cursor is not an access token.

Cursor pagination is not automatically a consistent snapshot. If sort keys change or records are inserted between requests, the result set can still evolve. That may be acceptable for a feed, but exports, billing reports, and audit jobs may need a fixed snapshot/version boundary or an asynchronous export from a consistent point in time. Long-running database snapshots can themselves create storage/retention pressure, so define the operational cost.

Choose from access pattern and user experience: offset for small/static datasets and random page access; cursor for large, changing datasets with sequential navigation. Define tie-breaking, filters, cursor expiry, deleted-item behavior, and page-size limits. In an interview, show the query/index that supports the cursor and state whether the API promises stable snapshot semantics or only efficient continuation.

### Q42. What is an idempotency key?

An idempotency key is a client-supplied operation identity that lets a server recognize repeated attempts at the same logical operation. It is especially important when a caller times out or loses a response after the server may already have committed a side effect. The key does not magically make an operation idempotent; the server must persist and enforce the relationship between the key and the operation.

For example, a payment client sends `POST /payments` with key `K`. The service should atomically claim `K` within a defined scope, such as tenant + endpoint + key, and record the request fingerprint. Concurrent requests using the same key must not both begin independent charges. A repeated key with the same logical payload should return the existing operation/result or its current status; reusing the key with a different payload should be rejected rather than silently returning a result for another request.

The idempotency record and business transition must be coordinated. If the service writes the key record, crashes before creating the payment, and later replays a “completed” response, the contract is broken. Use a database transaction or durable state machine that records the key, request identity, and operation status consistently. For external providers, pass an idempotency identity downstream where supported and reconcile uncertain provider outcomes; your database cannot atomically commit with an unrelated provider.

Define retention explicitly. The key must live long enough to cover legitimate client retries and downstream uncertainty. After expiry, the same key might be treated as new, so document that boundary and consider durable business-level uniqueness for high-impact operations. Idempotency prevents duplicate execution within the designed scope and retention model; it is not the same as deduplication and does not make all distributed side effects exactly-once.

### Q43. What are a reverse proxy, forward proxy, load balancer, API gateway, and service mesh? How do they differ?

These terms overlap because they describe different responsibilities, not always mutually exclusive products.

- A **forward proxy** represents clients when they access external destinations. It can enforce outbound access policy, hide client network details, cache selected responses, or inspect traffic where legally and operationally appropriate. Corporate egress proxies are a common example.
- A **reverse proxy** sits in front of servers and receives requests on their behalf. It can terminate TLS, route traffic, compress responses, enforce request limits, and hide backend topology. NGINX, Envoy, and many ingress products can act as reverse proxies.
- A **load balancer** selects a healthy backend among several candidates. It may operate at L4 or L7 and is often implemented as part of a reverse proxy, cloud network service, or both.
- An **API gateway** is an edge-facing policy and API-management layer. In addition to routing, it commonly applies authentication integration, quotas, request/response transformations, API version routing, and external-client policies. It should not become the only place where business authorization lives; services must still enforce access to their own resources.
- A **service mesh** manages service-to-service communication within a distributed application. Depending on the product and deployment model, it can provide workload identity, mutual TLS, retries/timeouts, traffic shifting, and telemetry through sidecar or node-level proxies and a control plane. It does not automatically solve business-level idempotency, data consistency, or authorization decisions.

A typical request path may be:

```text
Client
  -> DNS / global traffic manager (select region)
  -> CDN or edge protection (optional)
  -> L4/L7 load balancer or ingress
  -> API gateway / reverse proxy (external API policies)
  -> application service
  -> internal service proxy / mesh (optional)
  -> downstream service
```

Not every deployment needs every box. A gateway may also be the L7 load balancer; an ingress controller may combine reverse-proxy and gateway functions; a service mesh may be unnecessary for a small system. Multiple layers can accidentally multiply retries, timeouts, TLS hops, and operational complexity. Define the owner of each policy, the timeout budget across the full path, and which component is authoritative for identity and authorization.

### Q44. When should a monolith be split into microservices?

Split when there is a clear boundary that benefits from independent deployment, scaling, ownership, security, or fault isolation—not simply because the codebase is large or the team uses microservices elsewhere. A modular monolith can keep modules independently structured while retaining in-process calls and simpler transactional behavior. If module boundaries and ownership are unclear, moving the same tangled dependencies over HTTP usually creates a distributed monolith.

Look for evidence: one capability has a substantially different scaling profile; teams regularly block one another's releases; a domain needs a different security or reliability boundary; or a stable business capability can own its data and contract. Verify that the proposed service has cohesive responsibility, an explicit API/event contract, and few reasons to change in lockstep with its neighbors.

Extraction introduces real costs: network failure, timeouts, retries, version compatibility, distributed tracing, deployment pipelines, secrets, service discovery, data duplication, and cross-service workflows. A database transaction that once covered several tables may become a saga or a sequence of eventual updates. Before extracting, define data ownership and decide how to move traffic and data safely.

A Strangler-style migration can route one capability at a time to the new service, use compatibility adapters, replicate or migrate data with validation, and compare outputs before cutover. Preserve a rollback path where feasible and monitor both business correctness and latency. Explain why the service boundary is stable, what coupling it removes, and which new distributed-systems problem it introduces. “More scalable” is not sufficient without identifying what scales independently.

### Q45. Why is database-per-service not an automatic win?

Data ownership is a useful service boundary: the owning service controls its schema, validates invariants, and exposes supported ways to read or change the data. This reduces the risk that unrelated teams couple their releases to another service's tables. But **database-per-service does not necessarily mean one database server or cluster per service**; smaller services can share managed infrastructure while keeping schema and access ownership separate.

The cost appears when a workflow needs data owned by several services. A SQL join across shared tables becomes API calls, events, or a denormalized read model. Those calls can fail independently, increase latency, and return data from different points in time. A local transaction that once updated two tables may become a saga with partial progress and compensation. Analytics and reporting also need a deliberate way to combine data.

Choose the boundary around a cohesive business capability and its invariants, not around every entity or table. Keep a service's data private by convention and access controls; don't let another service make direct writes just because the database is reachable. Where cross-module transactions are frequent and the domain is still tightly coupled, a modular monolith with a shared relational database may be safer and simpler.

If the data is split, define the authoritative owner for each fact, APIs/events for access, consistency lag, deletion propagation, and reconciliation. Plan schema evolution and data migration with backward compatibility. The success measure is independent evolution with manageable operational cost—not the number of database instances.

## 6. Scaling and traffic management

### Q46. Vertical versus horizontal scaling?

**Vertical scaling** increases the capacity of one machine or database node—more CPU, memory, I/O, or a larger service tier. It is often simpler because the application may not need to distribute state or coordinate across nodes. The limits are the maximum available instance size, cost curve, maintenance/upgrade constraints, and the larger failure domain of one node. A bigger primary database can be a pragmatic step if a measured bottleneck can be solved within its limits.

**Horizontal scaling** adds more instances/nodes. Stateless application servers are often straightforward to scale out behind a load balancer. But adding nodes helps only if work can be distributed: in-memory session state may need externalization, a shared database may remain the bottleneck, and stateful components require replication, partitioning, coordination, or ownership boundaries. More nodes can also multiply connection pools, caches, retries, and control-plane load.

The choice should follow the bottleneck. CPU-bound stateless work may scale horizontally. A database limited by write locks may need query/model changes before either strategy helps. A fixed provider quota will not improve just because the worker fleet doubles. Consider how the system behaves when a node or zone is lost, not only its peak throughput with every node healthy.

In an interview, state the resource that is limiting the system, the simplest way to raise its safe capacity, and the new failure modes the chosen approach introduces. Show what evidence would trigger a move from vertical scaling to distribution, and how you would migrate safely without sacrificing correctness.

### Q47. Why might adding application instances not improve throughput?

Horizontal scaling helps only when additional instances contribute capacity to the limiting part of the system. If the database is saturated, a global lock serializes work, a cache is missing too often, or an external provider enforces a fixed quota, more application instances can simply send more concurrent work into the bottleneck. The result may be more timeouts, connection contention, and retry amplification rather than more successful completions.

Look for the queue and saturation signal in the full request path: application CPU/GC, thread pools, connection-pool acquisition, database locks/query plans, broker lag, cache latency, network, and provider throttling. Low application CPU can coexist with poor p99 if threads are waiting on connections or downstream responses.

Before scaling, bound concurrency and pool sizes, fix slow queries, reduce unnecessary fan-out, batch work where semantics allow it, and apply admission control/load shedding under overload. If the bottleneck is parallelizable CPU work, horizontal scaling may help; if it is one hot partition or key, the data placement or operation model may need to change.

Measure successful throughput, p95/p99, error rate, queue age, and downstream saturation before/after scale-out. Instance count by itself is not a capacity metric. In an interview, identify the bottleneck, explain why more instances would or would not affect it, and show how the design behaves during a rollout or when one failure domain is removed.

### Q48. What is backpressure?

Backpressure prevents producers from overwhelming a slower consumer by slowing intake or making the overload visible upstream. It can be implemented through bounded queues, consumer demand/credit, concurrency limits, rate limits, blocking, or explicit rejection. Without a bound, a queue can turn a brief overload into growing memory or storage use and waiting times longer than the operation's deadline.

A queue absorbs temporary mismatch only. Let arrival rate be λ and successful completion rate be μ. If arrivals stay above completions, backlog grows at approximately λ-μ. Adding a queue does not fix a permanent capacity deficit. Decide whether work should wait durably, be delayed/throttled, or be rejected quickly; the answer depends on whether the operation remains useful after its deadline.

Backpressure should be propagated as far as the architecture permits. If a worker is blocked on a provider quota, increasing consumer count may worsen throttling. If a database is saturated, request handlers may need to reduce concurrency before their pools fill. For an asynchronous API, accepted work must be durably recorded; for an interactive request, returning a fast 429/503 may be better than queueing work that cannot finish on time.

Monitor queue depth/bytes, oldest age, arrival/completion rates, blocked producers, and end-to-end completion SLO. In interviews, distinguish backpressure from load shedding: backpressure asks upstream to slow down, while shedding intentionally rejects/drops work to protect the system. Explain the behavior at the exact point where the queue or concurrency budget reaches its limit.

### Q49. What is load shedding?

Load shedding deliberately rejects or drops lower-priority work when demand exceeds safe capacity, preserving resources for critical operations. For example, a platform may defer recommendation refreshes and bulk exports before allowing checkout requests to wait until they time out. It is a stability mechanism for overload, not a permanent substitute for capacity planning.

Define priorities and admission limits before the incident. Shed work before thread pools, memory, or database connections are exhausted; once the system is deeply saturated, even deciding what to reject may be slow. Rejection should be explicit and semantically safe. A 429/503 or pending response must make clear whether the operation was accepted and whether retrying is appropriate. For mutations, never reject after the side effect may have committed without giving the caller a way to determine the outcome.

Use tenant-aware quotas, bounded queues, concurrency limits, and weighted scheduling so a noisy tenant cannot starve others. Be careful with automatic client retries: a fleet that retries every rejection immediately can create another traffic spike. For queued work, set expiry and replay policy so the system does not spend recovery capacity on notifications or jobs whose business deadline has passed.

Measure shed requests by route/priority/tenant, successful throughput for protected work, queue age, and error-budget burn. In interviews, explain which work is sacrificed, why the choice preserves the core invariant, and how overload is communicated to clients. A safe fast rejection can be better than an apparent success that later fails or produces incorrect state.

### Q50. Why can autoscaling make an incident worse?

Autoscaling can worsen an incident when the scaling signal does not represent the actual bottleneck. Adding application instances may multiply database connections, cache misses, queue consumers, or calls to a rate-limited provider. More replicas can also amplify retries, create synchronized cache warm-ups, or add startup load exactly when a dependency is unhealthy.

First identify what the system is waiting on. CPU may be low while threads wait for the database pool; CPU may be high because a request pattern causes wasteful serialization; consumer lag may come from a provider quota rather than insufficient worker count. Scale on signals correlated with sustainable successful throughput—such as queue age or a verified CPU-bound workload—and set explicit downstream concurrency budgets.

Autoscaling has a reaction delay. Instances must provision, start, load configuration, establish connections, and warm caches. For sudden spikes, keep warm capacity, use admission control/virtual waiting rooms, and reserve headroom for the critical path. Set maximum scale limits so one traffic spike cannot create unlimited database connections or provider calls. Scale-in also needs graceful draining so jobs and long-lived connections do not fail mid-operation.

Test scale-out, scale-in, cold cache, zone loss, and recovery from downstream throttling. Monitor successful throughput, p99, instance count, connection pool wait, cache miss rate, provider 429s, and queue age together. In an interview, explain what triggers scale-out, which dependency caps useful capacity, and how you prevent scaling from shifting the incident downstream.

### Q51. How do you isolate a noisy tenant?

A noisy tenant consumes a disproportionate share of shared resources and degrades other tenants' latency or availability. Limiting requests per second alone is not enough: one request may be a cheap cache hit while another scans millions of rows, sends a large export, or fans out to many downstream calls. Define budgets for the resources that matter—concurrency, CPU, query cost, queue occupancy, storage, egress, and expensive operations.

Start with per-tenant attribution in metrics and traces. Then apply layered isolation: per-tenant rate/concurrency limits at admission, bounded queues, weighted fair scheduling, query timeouts, separate worker pools for bulk jobs, and per-tenant storage limits. If workloads are very different, put bulk or premium tenants on dedicated pools or shards. Shared infrastructure is cost-efficient, but the resource scheduler must prevent one tenant from monopolizing it.

The isolation boundary must match the bottleneck. A tenant-specific rate limiter will not fix one query holding a global database lock; a separate application pool will not protect a shared primary from expensive SQL. For hot tenants, dedicated shards may help but complicate cross-tenant operations and placement. For bursty work, queue it with an explicit service objective and apply backpressure.

Decide what happens when a tenant reaches its budget: reject, defer, shed optional work, or offer a higher-capacity tier. Return a clear error and retry guidance rather than allowing unbounded queueing. Test fairness during spikes and tenant migrations, and watch for cardinality costs when creating metric labels for every tenant; high-cardinality telemetry may need aggregation or top-N views.

### Q52. How do you estimate queue drain time?

Estimate the backlog and the **net** processing rate, accounting for arrivals that continue while the queue drains. If there are 3.6 million messages, workers successfully complete 12,000 messages/s, and new messages arrive at 4,000/s, the net drain rate is 8,000/s. The ideal drain time is 3,600,000 / 8,000 = 450 seconds, or 7.5 minutes.

This approximation assumes stable rates, similar message cost, sufficient broker/network capacity, and no hidden downstream limit. Use successful completions, not dequeue or attempt rate; retries and poison messages may consume capacity without reducing business backlog. If arrival rate equals or exceeds successful completion, the queue will not shrink no matter how long you wait.

Validate that the system can actually sustain the assumed completion rate. More consumers may help if partitions and processing are the bottleneck, but can overload a database or hit provider quotas. Partition skew, per-key ordering, batch size, I/O, and message size affect per-worker throughput. During recovery, reserve capacity for live high-priority traffic and replay backlog at a rate downstream dependencies can accept.

Track queue age alongside depth and estimate whether work will finish before business deadlines. If backlog includes expired notifications or outdated jobs, apply an explicit expiry policy rather than blindly processing everything. In an interview, show the arithmetic, state assumptions, and explain what you would do if the projected drain time exceeds the SLA.

## 7. Databases and data modelling

### Q53. How do you choose SQL versus NoSQL?

Start from the business invariants, data relationships, query shapes, transaction boundaries, and operational constraints. A relational database is a strong default when records have meaningful relationships, joins and flexible queries are useful, and transactions/constraints must protect correctness. Many relational engines can also scale to substantial workloads with indexing, partitioning, replicas, and careful connection/query management.

NoSQL is not one category with one consistency or scalability model. A key-value store fits predictable key-based access; a document database suits aggregate-shaped records with flexible schemas; a wide-column store targets high-volume workloads with carefully chosen partition keys and known queries; a graph store fits relationship traversal; search engines optimize full-text relevance and facets. Each system's partitioning, consistency, transactions, and failure behavior differ, so read the specific product guarantees.

Use concrete access patterns to compare choices. What are the top reads/writes? Is there a global unique constraint? Must updates across records be atomic? Will queries require arbitrary joins or only lookups by a known key? What happens when one tenant/key is hot? How are indexes rebuilt, data backed up/restored, and shards rebalanced? What is the team capable of operating at 3 a.m.?

Polyglot persistence can be appropriate—for example, a relational order store, search index, and object store for files—but it creates synchronization and reconciliation work. Keep the source of truth explicit, define propagation lag and delete handling, and never treat a derived search/analytics view as authoritative for a transaction-critical decision unless the architecture explicitly guarantees it.

A senior answer should say why one store matches the workload, which limits were measured, what trade-offs are accepted, and why simpler alternatives are insufficient. “NoSQL scales better” or “SQL is always consistent” are not precise architectural arguments.

### Q54. OLTP versus OLAP?

**OLTP (online transaction processing)** handles operational reads and writes that support user-facing workflows: creating orders, updating account settings, reserving inventory, or recording a payment. It tends to require low-latency point/range queries, concurrent transactions, constraints, and predictable write behavior. **OLAP (online analytical processing)** supports aggregations and exploration over large datasets, often scanning and joining much more data to answer business or product questions.

Running heavy analytical queries on the transactional primary can harm OLTP latency by consuming CPU, memory, I/O, buffer cache, and connection slots. A common architecture sends committed changes through CDC or an event pipeline into a warehouse, lakehouse, or analytical store optimized for scans and aggregates. This isolates resource profiles, but introduces replication delay, transformations, data-quality controls, and reconciliation requirements.

Choose the data path based on freshness and query needs. A dashboard that tolerates five-minute delay may use scheduled micro-batches; near-real-time metrics may need streaming ingestion. Do not replicate sensitive operational data to analytics without considering access control, masking, retention, deletion, and residency. Nor should a derived analytical table be treated as authoritative for a payment or inventory decision simply because its query is fast.

In an interview, name the transaction latency and correctness requirements, analytical scan patterns, freshness target, and expected data volume. Explain how data gets from source to analytical store, how schema changes and deletes propagate, and how you detect lag or inconsistent aggregates.

### Q55. What makes a good partition or shard key, and how do you choose one?

A partition key determines how records are divided among storage or compute partitions. In a sharded database, it also helps determine which shard owns a record. A good key balances **distribution**, **query locality**, **write throughput**, **growth**, and **operational manageability**. There is rarely a key that optimizes every query.

Evaluate these dimensions explicitly:

1. **Cardinality and skew:** There should be enough distinct key values to spread the workload. High cardinality alone is not sufficient: a few tenants may generate most traffic, so a nominally high-cardinality key can still be skewed.
2. **Access patterns:** If most queries fetch a user's orders, partitioning by user ID can localize those reads. If common queries need a global time range, user-based partitioning may require scatter-gather across many partitions.
3. **Write distribution:** A monotonically increasing timestamp or ID can send new writes to one range partition. Hashing or bucketing can spread writes, but may make ordered range scans more expensive.
4. **Data locality and invariants:** If a transaction must atomically update several records, colocating them on one shard can simplify correctness. Cross-shard transactions may require coordination or a redesigned workflow.
5. **Growth and movement:** Consider how to add capacity, split a large tenant, rebalance data, and migrate the key scheme. A key that works at today's scale can become expensive to change later.

**Example:** For a multi-tenant SaaS product, `tenant_id` may be a natural routing key because most reads are tenant-scoped. But a very large tenant can become a hot shard. A practical design might route ordinary tenants by tenant ID and split selected large tenants into buckets, while preserving a mapping service that tells the application where each tenant's data lives. That increases routing and migration complexity, so it should be justified by observed skew.

Before choosing a key, list the top queries and write paths, estimate per-key traffic, identify operations that must be atomic, and test both average and worst-case tenants. A key should be selected from the workload and invariants—not because a database product recommends a fashionable strategy.

### Q56. What is a hot partition?

A hot partition receives a disproportionate share of traffic or data, so it becomes the bottleneck while other partitions remain underused. Common causes include one popular user/tenant/item, a monotonic key that directs new writes to the newest range, or a partition key that does not match the workload's common access patterns. A dataset can be balanced by row count and still be heavily skewed by request frequency or query cost.

Diagnose using per-partition throughput, CPU/I/O, queueing, lock waits, p95/p99, storage growth, cache hit rates, and top keys/tenants. Fleet averages conceal hot spots. Separate read-hot from write-hot behavior: caching or request coalescing may reduce hot reads, but it will not fix a serialized write to a global counter or a single mutable row.

Mitigations depend on the invariant. A hot read key can use local/shared caching or replicated read paths if freshness allows. A large tenant may move to a dedicated shard. A hot write key may be split into buckets or sharded counters, but that changes query/aggregation complexity and may break ordering. A time-range hotspot may need hash/bucket distribution or partition splitting. If the key must remain globally serialized for correctness, the throughput limit may be inherent and should be explicit.

Do not add partitions blindly: a hot logical key may still map to one physical partition. Validate the new design under realistic skew, including reads/aggregations, rebalancing, and failure. In an interview, explain how you found the bottleneck, which mitigation is compatible with the invariant, and how you prevent the hotspot from reappearing after migration.

### Q57. What is consistent hashing?

Consistent hashing maps keys and nodes onto a logical ring or equivalent hash space so that adding or removing a node typically remaps only a portion of keys rather than reshuffling nearly every key. Virtual nodes (vnodes) let each physical node own multiple ranges, improving distribution when nodes have different capacity or when the key space is uneven. This can reduce cache churn or data movement during membership changes.

It is a **placement technique**, not a complete storage architecture. Consistent hashing does not guarantee perfectly balanced request load, prevent a hot key, provide replication, or define data consistency. If one product ID receives 40% of traffic, its key remains hot regardless of how evenly other keys are distributed. Systems still need membership changes, replica selection, failure detection, routing metadata, rebalancing, and read/write semantics.

Compare it with other strategies. Simple modulo hashing can remap many keys when node count changes. Range partitioning supports range queries but may create hotspots; hash-based placement spreads keys but can make range scans scatter-gather. Rendezvous hashing is another option that ranks candidate nodes per key and can simplify weighted placement in some environments.

In an interview, explain what you want to minimize—remapping, hotspot risk, range-query cost, or operational complexity—and why consistent hashing fits. Describe what happens when a node is added/removed, how replicas are selected, how data moves, and how clients find the current owner. Do not claim that consistent hashing alone guarantees elasticity or even load distribution.

### Q58. What are read replicas good for, and what can go wrong?

A read replica maintains a copy of data from a primary or leader so read traffic can be distributed and recovery options improved. With **asynchronous replication**, the primary can acknowledge a commit before every replica has applied it. This means a replica may return an older value, omit a recently inserted row, or temporarily disagree with another replica. Replication is not the same as a consistent snapshot across all nodes.

The routing policy must reflect the read's correctness requirement:

- A catalogue description may tolerate bounded staleness and can be served from a replica or cache.
- A user's immediate read after changing a setting may require **read-your-writes**. The application can route that user's next read to the primary, wait until a replica reaches a known log position if the database supports it, or use a consistency token.
- A balance, reservation, or authorization decision may need an authoritative read and a transactional constraint; simply choosing the replica with the lowest observed lag is not enough to guarantee correctness.

Replicas also have costs: network and storage, replication apply CPU, operational complexity, and potential query contention. Long-running analytical reads can slow replication or consume resources needed for transactional queries. Monitor lag in time and in log/offset position where available, query latency, replica saturation, and failover readiness.

During failover, establish which node is authoritative before allowing writes. Promoting a lagging replica can lose acknowledged data unless the system's replication and commit guarantees rule that out. Prevent the old primary from continuing to accept writes—often called **split brain**—and understand how clients refresh connections and retry in-flight operations. A replica is useful only when its consistency and failover guarantees match the business contract.

### Q59. Why is read/write splitting not simply “send reads to replicas”?

Read/write splitting is a correctness policy as much as a capacity technique. The naive rule “all SELECT statements go to replicas” fails because a read can be part of a write workflow or be expected to observe a previous write. For example, an order service may insert an order and then immediately query it to build a response; a lagging replica can incorrectly make the order appear missing.

Classify operations by their consistency needs rather than SQL syntax alone. Strongly consistent reads, read-after-write flows, authorization checks, inventory checks, and decisions that guard a subsequent mutation may need the primary or a database-supported consistency mechanism. Low-risk browse and reporting reads may use replicas. Some systems use **sticky primary reads** for a short period after a write; others use log-position or session consistency tokens if the database supports them.

Failover complicates routing: the old primary may be unavailable, the new primary may have lagged, and connection pools may still hold stale endpoints. Routing logic needs topology discovery, bounded retry behavior, and clear handling of uncertain write outcomes. Monitor replica lag and reject or reroute reads when the freshness contract cannot be met.

Do not assume splitting will improve throughput automatically. Replicas can become overloaded, expensive queries can interfere with replication, and the primary still handles writes and may also receive critical reads. Measure the actual bottleneck and define the consistency promise for each important user journey before adding replicas.

### Q60. How do you perform a zero-downtime schema migration?

Use an **expand–migrate–contract** approach so old and new application versions can coexist while production writes continue. First add the compatible schema element (new column/table/index or new field). Deploy code that can operate with the old and new representation. Backfill existing rows in bounded batches, capture concurrent changes, validate results, switch reads/writes gradually, and remove the old schema only in a later contract step.

The dangerous race is a backfill reading an old row, a live request committing a newer value, and the backfill overwriting it. Use version-aware conditional writes, a consistent snapshot plus CDC from a known log position, or another explicit strategy to reconcile the snapshot with concurrent changes. Include deletes and retries. A job completing without error is not proof that the migrated data is correct.

Plan for database-specific operational risks: locks, long transactions, index creation, write amplification, replication lag, storage growth, and query-plan changes. Throttle the backfill so it does not starve production requests. Use checkpoints and make each batch idempotent and restartable.

Rollback must be designed before deployment. A previous application version may not understand data written in the new representation, so define the compatibility window and point after which a forward fix is safer than reverting. Validate counts, checksums, invariants, and shadow reads before cutover. In an interview, explain mixed-version behavior, concurrent-write capture, rollback boundaries, and which metrics stop the migration if production health deteriorates.

```mermaid
flowchart LR
  A[Expand: add compatible schema] --> B[Deploy code supporting old and new]
  B --> C[Backfill in bounded batches]
  C --> D[Capture concurrent changes]
  D --> E[Validate counts and versions]
  E --> F[Switch reads and writes]
  F --> G[Contract: remove old schema later]
```

### Q61. Normalization versus denormalization?

Normalization organizes authoritative facts so each fact has a well-defined owner and updates do not require changing inconsistent copies. For example, storing a customer's address once avoids updating many duplicated order rows when the address changes. Normalization can reduce storage and prevent update anomalies, but heavily normalized read paths may require joins and can be expensive at high read volume.

Denormalization duplicates or precomputes information to optimize a specific read pattern—for example, a product-search document or a customer order-history projection. It can reduce joins and improve latency, but creates new obligations: how changes propagate, how stale the copy may be, how deletes/tombstones are handled, and how the view is rebuilt when events are missed or the schema changes.

The key distinction is **authoritative write model versus derived read model**. It is often reasonable to normalize the transactional source and denormalize for search, feeds, dashboards, or other read-heavy access patterns. But avoid several independently writable “sources of truth” for the same fact unless a conflict-resolution protocol is explicit. A stale read model should not authorize a payment/refund or finalize inventory if the operation needs current authoritative state.

Choose from measured query cost, read/write ratio, update frequency, freshness target, and operational capability. In an interview, explain the source of truth, the propagation method (transactional update, outbox, CDC, or batch), the staleness contract, and the reconciliation/rebuild plan. “Denormalization makes reads faster” is incomplete unless you account for the write and correctness costs.

### Q62. How do you recover from database failure?

First classify the failure because “database down” covers very different situations: one node or availability zone failed; the primary is healthy but overloaded; replication lag is growing; storage is corrupt; an operator deleted data; credentials expired; or an entire region is unavailable. Failover, restore, and capacity remediation are not interchangeable responses.

For node failure, a managed or self-operated cluster may promote a replica, but the promotion protocol must establish a single write authority and account for the replica's replication position. Clients must refresh connections and handle in-flight writes whose outcomes are uncertain. For overload, adding replicas or promoting another node without reducing load may reproduce the failure. Apply admission control, shed optional work, reduce expensive queries, and protect the recovery path.

For corruption, accidental deletion, or malicious modification, replication can reproduce the bad state. You need backups, point-in-time recovery where supported, retention of logs/WAL/binlogs, and a tested restore process. Define **RPO** (maximum tolerable data loss) and **RTO** (maximum tolerable recovery time) with the business. A backup is not proven until you restore it into an isolated environment, validate consistency and application behavior, and measure the elapsed time.

During recovery, protect against a thundering herd of reconnects and retries. Restore service gradually, validate critical invariants, compare source and derived stores, and keep the system in read-only or degraded mode when correctness cannot yet be guaranteed. Record a runbook for promotion, fencing, connection refresh, validation, and rollback. A dashboard that says a replica is healthy is not equivalent to a proven recoverable system.

## 8. Caching and content delivery

### Q63. When does caching help, and when can it make a system worse?

Caching helps when data is read repeatedly, expensive to compute or fetch, and has a freshness tolerance that the cache design can honor. Typical wins include public content, product metadata, expensive configuration reads, authorization metadata with an appropriate revocation model, and derived views. The benefit depends on reuse and locality—not simply on having a lot of requests.

The primary performance question is whether the cache reduces end-to-end work. A remote cache adds a network hop, serialization, key lookup, and its own failure modes. If values are rarely reused or are tiny and cheap to retrieve from a local database buffer, the cache may increase latency. Caching can also shift a bottleneck rather than remove it: high hit ratios may conceal an overloaded origin until a cache outage or cold deployment suddenly sends most traffic to the database.

Define the cache contract per data class: what is authoritative, how stale may a value be, how is it invalidated, and what happens on a miss or cache outage? A product description may tolerate minutes of staleness; inventory at checkout may require an authoritative conditional write. Never let an eventually consistent cache silently decide a correctness-critical financial or inventory transition.

Measure hit ratio together with origin load, p95/p99 latency, memory utilization, evictions, stale-read incidents, and the effect of cache failure. Estimate memory using actual serialized key/value sizes plus metadata and allocator overhead. Add protection against stampedes, hot keys, cache penetration, and mass expiry. The best cache is not the one with the highest hit ratio; it is the one whose total cost and freshness behavior improve the user journey.

### Q64. Explain cache-aside, including its race conditions.

In cache-aside, the application checks the cache first. On a hit, it returns the cached value. On a miss, it loads the authoritative store, populates the cache, and returns the value. For writes, a common sequence is to commit the database change and then invalidate the corresponding cache key. This keeps the database as source of truth and makes the cache an optional acceleration layer.

The simple pattern has races. Suppose reader A misses the cache and reads old value (v1) from the database. Writer B commits (v2) and deletes the cache key. If reader A then writes (v1) into the now-empty cache, stale data is restored after invalidation. TTL limits how long the stale value may live but does not prevent the race. Concurrent misses can also cause a stampede.

Possible mitigations depend on how strict freshness must be: version each value and reject older fills; use compare-and-set or conditional population; serialize refill for hot keys; publish invalidations through reliable change capture; or bypass the cache for correctness-critical reads. An outbox/CDC event can improve the reliability of invalidation delivery, but asynchronous propagation still creates a staleness window. Do not claim that “update DB, then delete cache” provides a globally atomic transaction across both systems.

Design for failure too. If the database commit succeeds and cache invalidation fails, return behavior must reflect the committed mutation while a retry/reconciliation path clears stale cache state. If the cache fails completely, limit origin concurrency and shed optional traffic rather than allowing all misses to overload the database. Cache entries should include appropriate tenant/auth dimensions so a cache-key omission cannot leak one user's data to another.

### Q65. What is a cache stampede, and how do you mitigate it?

A cache stampede occurs when many requests simultaneously miss or refresh the same expensive key and all hit the origin. It can happen after a popular key expires, after a cache restart, or when many keys share similar TTLs. The origin experiences a sudden surge exactly when the cache stops protecting it.

Use the mitigation that matches the key's access pattern. **Single-flight/request coalescing** makes concurrent requests for the same key share one in-flight refill. **TTL jitter** spreads expirations across time. **Refresh-ahead** refreshes hot entries before expiry. **Stale-while-revalidate** serves an explicitly permitted stale value while one worker refreshes it. **Per-key concurrency limits** cap refill work. A distributed lock can help across application instances, but introduces lock expiry, owner failure, and latency; fencing/versioning may be needed if stale refill is correctness-sensitive.

Protect the source even if the cache layer fails: cap concurrent origin requests, keep database pools bounded, and shed low-priority workloads under overload. Avoid a design where every application instance retries an expired key at once. Measure origin QPS during expiry, refresh duration, coalescing effectiveness, stale age, and cache warm-up after deployment. Stale-while-revalidate is suitable for noncritical content, not automatically for balances or stock quantities. For correctness-sensitive state, use an authoritative transactional operation rather than trying to make the cache behave like a lock or database.

### Q66. Cache penetration, hot keys, and stampedes—what differs, and how do you diagnose them?

These symptoms can all increase origin traffic but have different causes:

- **Cache penetration:** requests repeatedly ask for keys that do not exist, so every request misses. Validate inputs, use short-lived negative caching where safe, or use a Bloom filter to avoid many definitely absent lookups. Bloom filters may have false positives, so the database must still decide whether a record exists.
- **Hot key:** one valid key receives disproportionate traffic. Coalesce concurrent reads, replicate or locally cache the value, and use request shaping. If updates are frequent, fan-out invalidation and freshness requirements become central.
- **Stampede:** many requests attempt to regenerate the same key at once, commonly after expiry. Use single-flight, TTL jitter, refresh-ahead, or stale-while-revalidate when the data contract permits it.

Analyze cache miss reason, key distribution, origin QPS, refill concurrency, TTL-expiry patterns, and request source. Negative caching can be unsafe when a record may be created immediately after a miss; use a short TTL or versioned invalidation. Local caches can reduce network traffic but replicate stale copies across application instances. A hot-key mitigation can also move load to a different bottleneck, such as a single lock or a fan-out invalidation service. Choose based on the data's freshness contract and confirm through metrics and load tests, not by applying every cache pattern at once.

### Q67. How should TTLs be chosen?

Choose a time-to-live (TTL) based on the cost of stale data, how frequently the source changes, reuse/access patterns, memory budget, and how much load the origin can handle when entries expire. A public description may tolerate minutes of staleness; an inventory value shown before checkout may tolerate staleness for display but must not make the final reservation decision. Authorization data needs a clear revocation policy.

TTL is not a complete consistency guarantee. It does not prevent a stale reader from repopulating a cache after an update, and it does not ensure a replica or upstream projection is current when the cache is filled. If a source update event is delayed beyond the TTL or a read starts before a write and finishes afterward, stale state can persist longer than expected.

Use TTL with other mechanisms according to the requirement: randomize expiry with jitter to avoid synchronized cache misses; use refresh-ahead for very hot keys; coalesce concurrent misses; use version-aware fills or reliable invalidation when stale overwrite is dangerous; and serve stale-while-revalidate only for data whose contract permits it. Sensitive/personalized responses also need correct tenant/user dimensions and cache-control rules.

Estimate cache memory using actual serialized entry sizes plus metadata/allocator overhead, and measure hit ratio alongside p99, stale-read rate, eviction, origin load, and refresh concurrency. In interviews, explain the allowed stale window, how TTL interacts with invalidation, and what the system does after a cache outage or mass expiration.

### Q68. What if Redis becomes unavailable?

First establish what Redis is being used for. If it is a disposable cache, the application may fall back to the database—but an immediate fallback from every instance can create a cache-collapse event and take down the origin. If it stores sessions, rate-limit state, ephemeral coordination, or workflow state, bypassing it may create security or correctness failures. The same technology can therefore need different outage behavior for different data sets.

For a cache-aside use case, use bounded database concurrency, request coalescing, load shedding, and possibly carefully permitted stale/local-cache responses. Avoid unlimited retries or creating a new Redis connection per request. For sessions, decide whether requests should fail, reauthenticate, or use an alternative secure session store. For rate limiting, define whether a bounded local limiter is acceptable or whether failure must be closed. For locks, do not blindly proceed as if ownership were acquired.

A Redis cluster/failover setup still needs a clear persistence and acknowledgement model. Depending on configuration and failover timing, recently acknowledged writes may be lost or some operations may observe stale/changed ownership. Do not treat replication as a guarantee of zero data loss. If Redis is part of a correctness-critical workflow, persist the business state in an authoritative datastore and define how to recover or reconcile Redis state.

Test full cache failure and partial failures: connection timeouts, elevated latency, failover, connection pool exhaustion, and hot-key concentration. Monitor errors, command latency, evictions, replica lag, connection counts, and the load shifted to the database. A fallback is safe only when it preserves the business invariant and keeps the remaining dependencies within their own capacity limits.

### Q69. How do you keep cache and database consistent?

An ordinary database transaction and an independent cache do not share one atomic commit. This means a cache update can fail after a database commit, or concurrent readers can refill a stale value. The first design decision is not “which invalidation technique?” but “what consistency does this particular read require?”

For many read-heavy domains, the authoritative database plus cache-aside invalidation and a bounded TTL is adequate. The cache is allowed to be stale for a known window, and correctness-sensitive operations go to the authoritative store. For stricter requirements, version cache entries and compare versions during refill, use a reliable change stream/outbox to deliver invalidations, or use a datastore with a consistency model that directly supports the required reads. Each method still has its own failure and recovery semantics; an asynchronous invalidation event cannot eliminate its propagation window.

A classic race is: reader reads old value, writer commits new value and deletes cache, reader then repopulates the old value. A version check, conditional fill, or coordination mechanism can prevent some variants. A transactional outbox can ensure the invalidation intent is durably recorded with the database mutation, but the consumer may be delayed or redeliver the event, so invalidation processing must be idempotent and lag observable.

For payments, balances, inventory allocation, or permissions that require immediate revocation, do not rely on cache TTL as the only correctness mechanism. Use the source of truth for the decision, or define a token/version/lease contract that bounds stale authorization explicitly. Document stale-read tolerance, cache outage behavior, invalidation latency objective, and reconciliation plan.

## 9. Distributed systems and consistency

### Q70. What is a partial failure, and why is it more dangerous than a simple error?

A partial failure occurs when some components or communication paths work while others are failed, slow, unreachable, or uncertain. A remote call may time out even though the server completed the operation; a broker may accept a message while its acknowledgement is lost; one replica may lag while another is current. The caller does not automatically know whether the side effect occurred.

This uncertainty is different from a definitive rejection. If an inventory reservation times out, sending a new reservation with a new identity can reserve twice. If a payment provider charges a card but the response is lost, recording “failed” and initiating a new charge can duplicate the debit. Preserve an operation ID, make the mutation idempotent at a defined boundary, and query status or reconcile when the outcome is unknown.

Use deadlines and bounded retries to control resources, but do not interpret timeout as rollback. Design the request state machine to distinguish accepted, pending, succeeded, failed, and needs-reconciliation states where appropriate. For async workflows, acknowledge only after the work is durably accepted; for database-plus-event flows, use a transactional outbox and idempotent consumers.

Partial failure must also appear in observability: trace the request across services, record durable state transitions, and alert on operations that remain unknown too long. In an interview, walk through the failure between each pair of steps—before the remote effect, after the effect but before response, and after response but before local commit—and explain what the retry does in each case.

### Q71. Explain strong and eventual consistency with an example.

**Strong consistency** is an umbrella term; be precise about the actual guarantee. In many interviews, “strongly consistent read” means the read reflects writes completed before that read began, often described using linearizability. A linearizable operation appears to take effect atomically at a point between invocation and response and respects real-time ordering. The precise guarantee depends on the system's protocol and API.

**Eventual consistency** means that if updates stop and the system continues making progress, replicas will eventually converge. It does not promise that a read immediately after a write returns the latest value, nor does “eventual” specify a maximum staleness time. Concurrent writes may need conflict resolution or a deterministic merge rule. Some systems offer intermediate guarantees such as read-your-writes, monotonic reads, or bounded staleness.

Example: updating a user's profile display name can often tolerate a short propagation delay to caches and search indexes. Deducting money from an account or allocating the final seat needs an authoritative concurrency-control boundary; showing a stale “available” value can be acceptable for browsing but not for completing the reservation.

Choose consistency per operation based on the invariant, not per technology label. A single application may use a strong transactional write for the order, eventual propagation to search and analytics, and session consistency for a user's immediate status page. Explain the user's observable behavior, the latency/availability trade-off, and how the system repairs derived views when events are delayed or lost.

### Q72. What is read-your-writes consistency, and how can you provide it?

Read-your-writes is a session consistency guarantee: after a client successfully writes a value, its subsequent reads in that session should observe that write or a later one. It is weaker than full global linearizability because unrelated clients may still see older data, and it does not imply that every replica has converged.

A common failure occurs when a write goes to a primary and the next read is sent to an asynchronous replica. The replica has not yet applied the write, so the user's update appears to disappear. The write succeeded; the routing policy violated the session expectation.

Approaches include routing the session's reads to the primary for a period, using a database-supported session/consistency token, or waiting until the replica has applied a known log position before serving the read. A sticky-primary policy is simple but can reduce read scalability. A token-based approach can preserve replica reads if the datastore exposes the necessary replication position and the application correctly enforces it. A fixed sleep is not a correctness guarantee because lag varies.

Define behavior during failover. The write may be acknowledged on the old primary but absent from a promoted replica, depending on replication guarantees. Read-your-writes cannot restore data that has been lost by the chosen durability policy. Critical mutations therefore need a commit guarantee and idempotent retry/status behavior, not just a clever read router. Test with real replica lag and failover rather than only unit-testing the routing code.

### Q73. Explain CAP carefully, without reducing it to “pick any two.”

CAP describes a trade-off that becomes unavoidable when a distributed data system experiences a network partition. **Consistency** in the CAP formulation means a single-copy/linearizable view of the data. **Availability** means every request sent to a non-failed node eventually receives a response, though that response may not reflect the latest write. **Partition tolerance** means the system continues operating despite communication failures between parts of the system.

When the network is healthy, CAP does not say that a system must choose only two properties. During a partition, a system that preserves linearizable consistency may have to reject or block some operations because it cannot safely determine the latest value or current write authority. A system that keeps accepting conflicting operations on both sides may remain responsive but must reconcile divergent state later, so it is not providing linearizable consistency for those operations.

This is not the same as saying “SQL is CP and NoSQL is AP.” Product behavior depends on replication protocol, operation, consistency level, and configuration. Even one database can offer stronger guarantees for some operations and weaker ones for others. CAP also does not mean that latency is irrelevant or that a partition is the only failure mode that matters.

Example: during a regional partition, a catalogue may accept independent view counts or queue noncritical analytics events. A payment ledger or seat inventory may reject writes in the region that cannot prove ownership, because accepting both sides could create conflicting debits or duplicate reservations. Explain which operations remain available, which are rejected, and what recovery/reconciliation is required. PACELC adds that even without a partition, systems trade off latency and consistency, which helps reason about normal operation as well as failure.

### Q74. What does PACELC add?

PACELC extends the CAP discussion by naming the trade-off present during normal operation as well as during a network partition. In the shorthand, during a **Partition (P)** a system may trade **Availability (A)** against **Consistency (C)**; **Else (E)**, when no partition occurs, it may still trade **Latency (L)** against **Consistency (C)**. For example, a local replica can answer quickly but may be stale, while a read coordinated with a remote leader may provide a stronger guarantee at the cost of a cross-region round trip.

The value of PACELC is that it stops the discussion from focusing only on rare partitions. Geographic placement, quorum settings, leader routing, and consistency levels can all influence normal request latency. A system may also use different policies for different operations rather than making one global choice.

Use the model to frame the design question, not as a complete classification of a database. The actual behavior depends on the product's replication protocol, read/write API, configuration, and failure semantics. CAP consistency generally means linearizable/single-copy consistency in that formulation; product documentation may use “consistency” more broadly, so be precise about the guarantee being discussed.

In an interview, connect the trade-off to a real operation. A catalogue may prefer local low-latency reads and tolerate a small lag; a financial write may require one authoritative owner even if the region is temporarily unable to reach it. State what the user observes during normal operation and during partition, and how the design recovers afterward.

### Q75. What are quorum reads and writes, and what does the quorum formula not guarantee?

A replicated datastore may require a write to be acknowledged by at least (W) replicas and a read to consult at least (R) replicas out of (N). In a simple model, (R + W > N) ensures that every read set and write set overlap in at least one replica. For example, with (N=3), a write quorum (W=2) and read quorum (R=2) intersect. That intersection is useful only if the protocol knows how to identify and return the newest valid version.

The arithmetic by itself does **not** universally guarantee linearizability, read-your-writes, or conflict-free values. Concurrent writes may be accepted, replicas can have different versions, and a product may use sloppy quorums, hinted handoff, or other policies under failure. Leader-based databases may provide their guarantees through a different replication/consensus protocol, even if the documentation uses the word “quorum.”

Understand the concrete system's semantics: what counts as an acknowledgement (memory, durable log, or applied state), whether a read performs version comparison or repair, what happens when membership changes, and how failover treats acknowledged writes. A write that reaches two replicas but is later overwritten by a conflicting concurrent update is not the same as a linearizable commit.

Quorum size trades latency and availability against consistency and durability, but the exact trade-off depends on topology and protocol. In an interview, define the required guarantee first, then explain how the chosen datastore's read/write policy achieves it and what happens during partition, concurrent writes, or replica recovery.

### Q76. Why are distributed locks difficult?

A distributed lock tries to ensure that only one worker owns a resource at a time, but distributed processes can pause, lose connectivity, or resume after the lock's lease has expired. The lock service may believe ownership transferred while the old worker still runs. An explicit release does not fix process pauses, and a lease expiry does not forcibly stop the old process from issuing writes.

Consider worker A obtains a 30-second lease and begins generating a report. A pauses for 40 seconds due to a long runtime pause or network isolation. Worker B acquires the expired lease and starts a new report. If A resumes and writes its result, both workers may perform the side effect even though the lock service only recognizes B. A lock alone therefore does not protect the target resource from stale owners.

For mutable shared state, use a monotonically increasing fencing token or epoch assigned on ownership change, and make the resource reject writes with older tokens. If the resource cannot validate fencing, rely on an idempotent/conditional operation or redesign ownership around a database transaction or unique constraint. A lock should also have clear acquisition timeout, lease renewal, crash recovery, monitoring, and semantics for owner loss.

Do not use a distributed lock to compensate for missing database invariants when a unique constraint or conditional update can solve the problem more directly. If a scheduled job may run twice, make its state transition idempotent and record durable job identity. The interview answer should specify what exact resource is protected, who validates ownership, and what happens if two workers overlap after a pause.

### Q77. Does exactly-once delivery exist?

Avoid using “exactly once” without naming the boundary. Networks can lose acknowledgements, consumers can crash after performing a side effect but before acknowledging a message, and publishers can retry when they do not know whether a write was committed. At the transport level, most systems expose some combination of at-most-once and at-least-once behavior; duplicates and uncertain outcomes must be considered.

Some brokers and stream processors provide exactly-once processing semantics within a defined transactional boundary—for example, coordinating consumed offsets and produced records. That guarantee does not automatically cover an external payment provider, email service, or arbitrary database unless that external side effect participates in the same protocol or the application adds deduplication/idempotency.

A robust business pattern is to assume redelivery, give each logical operation a stable identity, and make state changes idempotent. A consumer can record the message ID and perform its database mutation in one local transaction, then acknowledge/commit the offset. If the crash occurs before the transaction, the message is retried; if it occurs after commit but before acknowledgement, the deduplication record prevents repeating the effect. This only works if the effect and dedupe record share an atomic boundary. For external calls, use provider idempotency keys when available and reconciliation for uncertain results.

State which layer provides which guarantee: broker delivery, stream-processing transaction, database mutation, or end-user-visible business outcome. An interview-quality answer explains the crash windows and the exact mechanism that closes each one instead of claiming the entire distributed system is exactly once.

### Q78. How do you handle out-of-order events?

Start by defining the actual ordering requirement. Many business workflows require order only per entity, such as per account, order, or inventory item; globally ordering every event creates a coordination bottleneck and is rarely necessary. If order matters per key, route that key consistently to one ordered partition and process its events sequentially, while remembering that producer retries and application concurrency can still introduce semantic reordering.

Include a monotonic entity version or sequence number where possible. A consumer can reject a stale update when its version is less than or equal to the version already applied. If it receives version 12 while the current state is version 10 and version 11 is required, it must decide whether to buffer, retry, query authoritative state, or mark the entity for reconciliation. The right choice depends on whether every intermediate event matters or whether the event describes the latest state.

Timestamps alone are often unsafe ordering signals because clocks can skew and events may be emitted long after the underlying action. Use aggregate versions, log offsets, or domain sequence numbers with explicit scope. For events that can arrive after deletion, retain a tombstone/version long enough to prevent an older update from recreating the entity.

Observe duplicate/out-of-order counts, sequence gaps, and oldest buffered events. Ensure replays and DLQ recovery preserve the required ordering. In an interview, explain what order is guaranteed by the broker, what order is guaranteed by the consumer, and how the system recovers from a missing or permanently invalid event.

### Q79. What is reconciliation, and why do distributed systems need it?

Reconciliation detects and repairs discrepancies between the state a system expects and the state actually observed. It is important because retries, timeouts, delayed events, manual intervention, consumer bugs, partial failures, and external systems can leave two components temporarily or permanently inconsistent. A robust architecture should not assume every event will arrive exactly once and in order forever.

A reconciler compares authoritative state with derived or external state using stable business identifiers and well-defined rules. For example, a payment workflow can periodically compare internal payment attempts with the provider's transaction status and settlement reports. If the provider charged a payment but the callback was lost, the reconciler can advance the internal state; if the internal service believes a payment succeeded but the provider reports failure, it should move to investigation or compensating action rather than blindly overwrite the record.

Reconciliation needs idempotent repair operations, bounded batch sizes, rate controls, auditability, and observability. It should distinguish safe automatic repairs from conflicts requiring human review. Avoid “last write wins” when the discrepancy is about a financial or inventory invariant. Use source versions, event IDs, timestamps with explicit semantics, and status-transition rules to prevent a stale repair from overwriting newer state.

Measure mismatch count, oldest unresolved discrepancy, repair success, and repeated oscillation. Reconciliation does not excuse poor transactional design; it is the safety net for failures that cannot be prevented atomically across boundaries. Design it alongside the initial workflow, including access controls for repair tools and a way to explain every automated state change.

## 10. Messaging and event-driven architecture

### Q80. Queue versus event stream?

A **queue** usually models work to be performed. A consumer receives a task, processes it, and acknowledges completion; the broker's delivery semantics determine when work can be redelivered. A **retained event stream** models a sequence of facts that happened. Consumers can often read independently from their own offsets and replay retained history to rebuild a projection or recover after a bug. Broker products can support both patterns, so the distinction is about semantics and operational contract rather than product branding.

For a task queue, the main questions are visibility/lease timeout, acknowledgement, retries, dead-letter handling, fairness, and whether one task should be processed by only one worker at a time. For an event stream, key questions include partition ordering, retention, consumer offsets, replay, schema evolution, and how multiple consumer groups read the same events independently. Some queue systems also support competing consumers and delayed retries; some log systems can implement work distribution with consumer groups.

Example: “send this email” is a work item; “order 123 was confirmed” is a business fact that can feed email, analytics, search, and customer history independently. Do not use a queue acknowledgement as evidence that an external side effect definitely completed if the worker can crash between the side effect and ack. The consumer still needs idempotency or deduplication.

Choose the abstraction based on whether a message represents an instruction or an immutable fact, whether history must be replayable, and who owns retry and retention policy. In either model, define delivery guarantees, ordering scope, poison-message behavior, idempotency, capacity, and what happens when the backlog exceeds its business deadline.

### Q81. What is a Kafka partition key, and how does it affect ordering and throughput?

A producer's partitioner uses the key to select a partition. With a stable key-to-partition mapping, records sharing a key normally go to the same partition, and Kafka preserves their log order within that partition. This is why an order-processing topic often uses `orderId` when all events for a single order must be processed in sequence.

The key determines both **where the data lands** and **which work can run in parallel**. A good key has enough distinct values to distribute load and aligns with the entity whose ordering matters. A key such as `country` may create a few hot partitions, while a random key spreads traffic but destroys per-order ordering. One extremely popular account can dominate a partition regardless of how many other keys exist.

Partition count and key mapping changes also matter. Increasing the number of partitions may remap keys for common partitioner implementations; messages for one key written before and after the change can land in different partitions, weakening assumptions about total order across the transition. If ordering is a correctness requirement, plan topology changes explicitly or use a stable routing/bucketing strategy.

Check key skew, per-partition throughput and lag, record size, and consumer processing time. More partitions increase parallelism but also broker metadata, replication, and rebalance cost; more consumers in one group cannot make a single partition concurrently processed in-order by several consumers. Choose the key from business invariants and access patterns—not just a desire for a uniform distribution.

### Q82. What is consumer lag, and how should you respond to it?

Consumer lag measures how far a consumer group is behind the latest records available in a stream, commonly expressed as offset difference. It is a symptom, not a full diagnosis. Lag may rise because arrival rate increased, processing slowed, one partition is hot, the consumer is retrying poison messages, database writes are blocked, or a downstream provider is throttling.

Measure more than the aggregate offset count: **oldest unprocessed event age**, arrival and successful completion rates, per-partition lag, processing duration, retries, rebalance frequency, downstream latency, and the business deadline for the event. A backlog of 10 million low-cost events can recover faster than 100,000 events requiring slow external API calls.

Estimate net drain as successful completion rate minus new arrival rate. If workers complete 15,000/s while new records arrive at 9,000/s, net drain is 6,000/s; a 1.2-million backlog takes about 200 seconds under stable conditions. If net drain is zero or negative, adding consumers will help only if consumer parallelism is the bottleneck and the downstream can accept more work.

Mitigate based on cause: add consumers/partitions if safe parallelism is available, increase batch efficiency, optimize downstream writes, isolate slow keys, fix retry loops, or enforce provider quotas. Scale-out can overload a database or create more throttling, so cap concurrency and preserve per-key ordering. Define when events expire, are dead-lettered, or become business-invalid, and verify the end-to-end completion SLO after the backlog drains.

### Q83. How should retries and dead-letter queues work?

Retries are useful for transient faults, but every retry consumes capacity and may repeat a side effect. Classify failures first. A timeout, connection reset, or throttling response may be transient; schema validation errors, authorization failures, and business-rule rejections are usually permanent. Respect provider retry hints where supported, and do not retry every failure indiscriminately.

Use bounded attempts, exponential backoff, and jitter so many clients do not retry at the same moment. Apply an overall deadline or retry budget in addition to a per-attempt timeout. The retry policy should have one clear owner where possible; retries in the client, gateway, service, SDK, and broker can multiply. Most importantly, retry only when the operation is safe to repeat or uses a stable idempotency identity. A timeout means the result may be unknown, not that the operation definitely failed.

A dead-letter queue (DLQ) isolates messages that have exhausted the normal policy or are known to be unprocessable. It is not a substitute for error classification or a place to discard important work silently. Store the original payload or a safe reference, failure reason, attempt history, schema/version metadata, and correlation IDs. Alert on oldest age and count, define ownership and retention, and provide controlled replay after the cause is fixed.

Replay is another production workload: rate-limit it, preserve ordering where the domain requires it, and make consumers idempotent. A poison message may repeatedly break a partition or block later events; moving it aside changes processing semantics and must be an explicit business decision.

### Q84. What is the transactional outbox pattern?

The transactional outbox addresses the **dual-write problem**: an application changes its database and publishes an event to a broker, but the database commit and broker publish are two separate operations. If the database commits and publishing fails, downstream consumers never hear about the change. If the event publishes but the database rolls back, consumers observe a fact that did not commit.

With an outbox, the application writes both the business change and an outbox row in the same local database transaction. A separate publisher reads committed outbox rows and sends them to the broker. The publisher then marks the row as sent or stores publication progress. Because the business mutation and event intent share one ACID transaction, the system does not lose the intent merely because the broker is temporarily unavailable.

The publisher can crash after publishing but before marking the row sent, so duplicate publication is expected. Consumers must therefore be idempotent, usually using a stable event ID and an inbox/processed-event record coordinated with the consumer's local state change. The outbox provides reliable eventual publication, not end-to-end exactly-once side effects.

Operational design matters: poll with bounded batches or use CDC on the outbox; preserve the required per-aggregate ordering; avoid one locked publisher query becoming a bottleneck; monitor oldest unpublished event, backlog size, publisher errors, and duplicate rates; and define retention/cleanup. If an event contains sensitive data, minimize the payload and control access. The interview-quality explanation should show the failure window it closes, the duplicate window it still has, and how consumers recover safely.

### Q85. Saga choreography versus orchestration?

A saga coordinates a long-running business workflow whose steps commit in separate local transactions. Since there is no single ACID transaction spanning all services, the design records progress and defines what happens when a step fails or the workflow times out. A saga is not a database rollback: a compensation is a new business action that attempts to counteract an earlier action and may itself fail or be impossible.

In **choreography**, each service reacts to events and publishes the next event. It avoids a central coordinator and can keep services loosely coupled at runtime, but the end-to-end flow is distributed across handlers. It becomes hard to understand which step comes next, detect a stuck workflow, or change event contracts when there are many branches.

In **orchestration**, a workflow coordinator issues commands to participants, records state, applies deadlines, and decides which step or compensation follows. The control flow is easier to inspect and operate, but the orchestrator becomes a critical component and needs durable state, high availability, versioned workflow definitions, and idempotent command handling.

Example: an order saga may reserve inventory, authorize payment, and arrange shipment. If payment authorization fails, inventory reservation may be released. If shipment was already dispatched, compensation may require a return or refund rather than reversing the external action. Each step needs a stable operation ID, explicit success/failure/unknown states, timeout policy, and reconciliation. Choose choreography for simple event reactions; prefer orchestration when the workflow has branching, long waits, human intervention, compensation, or strict operational visibility.

### Q86. How do you handle poison messages?

A poison message repeatedly fails because of malformed data, an incompatible schema, a deterministic consumer bug, or another non-transient condition. Retrying without classifying the failure wastes capacity and may block other work. Capture the original event or a secure reference, event ID, source offset/partition, schema version, attempt history, error class, and correlation context, then move it to a monitored dead-letter/quarantine path when the policy permits.

The critical question is whether later events depend on the failed one. If ordering matters per account/order, skipping the poison event can corrupt state; pause or isolate the affected partition/key, repair the payload/consumer, then replay it before dependent events. If the domain explicitly allows skipping, record that decision and run reconciliation to establish the resulting state. Do not treat “put it in a DLQ” as an automatic correctness-preserving action.

Classify deterministic failures separately from transient ones. Use bounded attempts and backoff for transient dependency errors; permanent schema/validation errors require repair rather than a faster retry. Replay must be rate-limited and idempotent, preserve event identity for deduplication, and avoid re-triggering irreversible side effects. For privacy-sensitive data, ensure the DLQ has appropriate access and retention controls.

Monitor DLQ count and age, blocked-partition duration, repeated failure reason, and affected business entities. Assign ownership and a controlled remediation process; an unattended DLQ is delayed data loss. In interviews, state whether ordering is global, per partition, or per entity, then explain how remediation preserves or intentionally relaxes that guarantee.

## 11. Transactions and business correctness

### Q87. How do you prevent duplicate payment charges?

Treat payment processing as a durable state machine, not a single request that either succeeds or fails. The critical case is a timeout after the provider may have charged the customer but before your service receives the response. Retrying a new charge with a new identity can create a second debit.

Create a stable payment/attempt ID before calling the provider and enforce uniqueness for the logical business operation (for example, one payment attempt per order and payment revision). Accept a client idempotency key and atomically bind it to the request fingerprint and payment record. Concurrent requests using the same key must resolve to the same operation. Pass a stable provider idempotency key if the provider supports it, but verify that provider's key scope and retention period rather than assuming it is permanent or globally unique.

Use explicit states such as CREATED, SUBMITTING, AUTHORIZED, CAPTURED, FAILED, and UNKNOWN/RECONCILIATION_REQUIRED, with legal transitions. Do not mark a payment failed merely because the HTTP call timed out. Query provider status, consume signed callbacks/webhooks idempotently, and reconcile internal records against provider transaction reports. Keep an auditable ledger of financial state transitions; avoid mutating a single status field in ways that erase evidence of prior actions.

The database transaction and external provider operation are not one atomic transaction. The design must account for crashes before the call, after provider success, after response receipt, and before the local status commit. If a duplicate side effect is discovered, define safe compensation and customer communication. Explain which invariants are enforced locally, which guarantees depend on the provider, and how an unknown outcome eventually becomes resolved.

### Q88. How do you prevent double booking a seat?

The invariant is that at most one active booking may own a seat for a given event. An application-side check such as “query whether the seat is available, then insert a booking” is not sufficient: two concurrent requests can both read “available” before either writes. The invariant must be enforced atomically by the authoritative store.

One approach is a unique constraint on `(event_id, seat_id)` for active reservations, possibly using a supported partial/filtered index where the database allows it. Another is a conditional update such as changing a seat from AVAILABLE to HELD only if its current state is still AVAILABLE and checking that exactly one row was updated. The transaction then records the hold/booking and its expiry. The loser sees a conflict and must return a clear sold-out/already-held result.

For high-demand events, design the hold lifecycle carefully: a hold has an owner, expiry, and state transition; expiry is not simply a background delete based on an arbitrary worker clock. Use conditional transitions and a durable scheduler/reconciliation process. Payment can complete after the hold has expired, so the workflow must define whether it revalidates inventory, compensates/refunds, or uses a stronger reservation protocol.

A cache can show an approximate availability view, but it should not authorize the final booking. Add request idempotency so client retries do not create duplicate booking attempts. Load-test contention on the same hot seat/event, inspect lock waits, and define how the system behaves if the database response times out after the commit.

### Q89. Optimistic versus pessimistic concurrency control?

**Optimistic concurrency control** assumes conflicting updates are relatively uncommon. A read includes a version or expected value, and the write succeeds only if that version is unchanged. For example, `UPDATE inventory SET quantity = quantity - 1, version = version + 1 WHERE sku = ? AND version = ? AND quantity > 0`; if zero rows are updated, the caller must reread and decide whether to retry or report a conflict. This avoids holding a lock throughout application thinking time but can cause repeated conflicts under heavy contention.

**Pessimistic concurrency control** acquires a lock before modifying a contested record, so conflicting transactions wait or fail. It can be appropriate for very hot resources or short critical sections, but long transactions, inconsistent lock ordering, and remote calls while holding locks can create blocking and deadlocks.

Do not choose based only on a slogan such as “optimistic is faster.” Estimate conflict rate, transaction duration, number of hot keys, and the cost of failed attempts. Use atomic conditional updates or constraints when they naturally express the invariant. If stronger isolation is required to protect a multi-row rule, a single row version may not be sufficient.

Both strategies need explicit retry limits and error handling. A serialization failure or deadlock should usually be retried only after rolling back the full transaction and only if repeating the transaction is safe. External side effects should not occur inside a retryable database transaction unless they are separately idempotent. Explain how the database enforces the invariant, what the loser observes, and how retries affect tail latency under contention.

### Q90. What happens when a request times out after a successful write?

A timeout after a mutation creates an **uncertain outcome**. The server or database may have committed the operation, but the response may have been lost on the network. The client cannot infer “failed” solely from not receiving a response. Retrying with a new identity can duplicate the side effect.

For mutation APIs, create a stable operation identity (often an idempotency key or resource ID) and persist the status through a defined state machine. The retry should return the same logical operation or its current state, not start a second operation. For long-running actions, return an operation ID and expose a status endpoint or callback. Distinguish ACCEPTED/PENDING, SUCCEEDED, FAILED, and UNKNOWN/RECONCILIATION_REQUIRED when those states are meaningful to the contract.

The server must coordinate the operation record with the durable business mutation. A separate idempotency table updated independently can still leave a crash window; place both changes in one transaction or design a recoverable workflow that reconciles them. If the write called an external provider, query provider status or reconcile by a stable merchant/reference ID before deciding whether to retry.

Clients should use bounded retries with backoff, but the API contract must be safe even when retries are delayed or repeated by intermediaries. Include correlation IDs and audit records so support can determine what happened. In interviews, draw the crash points before and after commit and explain the client-visible response at each point.

### Q91. Why is a saga not a distributed ACID transaction?

A local ACID transaction gives atomicity at its transaction boundary: either its changes commit or they do not. A saga is a sequence of separate local transactions across services. Intermediate results can be visible while the workflow is still running, and compensation does not erase history or guarantee that the original effect can be perfectly undone.

For example, an order service may create an order, inventory may reserve a unit, and a payment provider may authorize a card. If shipment later fails, the system may release inventory and void or refund the payment. A refund is a new financial event, not a time machine that makes the authorization never have happened. If the external action is irreversible, the business may need manual resolution or a different workflow.

A saga therefore needs a durable state machine: unique workflow/step IDs, explicit transitions, idempotent participants, timeouts, retry policy, compensation rules, and a reconciliation path. It must handle a crash after a participant commits but before the coordinator records the response; the participant's command should be safe to repeat or queryable by operation ID. Compensation itself can fail and may need retry or escalation.

Use a single database transaction when the invariant can be protected within one database and the transaction can remain short. Use a saga when the business process spans independently owned systems or long-running work. Do not choose a saga simply to avoid transaction design; it deliberately trades atomic visibility for independent progress and operational complexity.

### Q92. How should webhook consumers handle duplicate callbacks?

Assume webhooks can be delivered more than once, delayed, retried out of order, or sent again after the sender fails to observe the acknowledgement. The receiver should return a prompt acknowledgement only after it has durably accepted responsibility for the event. Performing a long business workflow synchronously in the callback handler increases timeout and redelivery risk.

Use the provider's event ID as a deduplication identity where its uniqueness scope is documented. In one local transaction, record that event ID in an inbox/processed-event table and apply the corresponding state transition. If the same event arrives again, recognize it and return success without repeating the side effect. If the event ID may only be unique per account or event type, include that scope in the key. Validate the request signature, timestamp/replay protections, and relevant account/resource identity before trusting the payload.

Deduplication alone does not solve out-of-order events. An old “payment pending” event might arrive after “payment succeeded.” Prefer querying the provider's current status or using monotonic state-transition/version rules rather than applying callbacks blindly. If the event is accepted to a queue, make the enqueue durable before acknowledging; otherwise the provider may stop retrying after the application loses the event.

Provide retry-safe downstream actions, dead-letter handling for invalid payloads, observability by event ID, and a reconciliation job for missing callbacks. Never assume that receiving a success webhook is the only proof of business completion or that HTTP delivery exactly once is guaranteed.

## 12. Resilience and failure handling

### Q93. How should retries be designed?

A retry policy must specify **which failures**, **how many attempts**, **how long to wait**, **the overall deadline**, and **what makes repeating the operation safe**. Retrying every error is dangerous: validation and authorization errors are usually permanent, while timeouts, connection resets, and throttling may be transient. The exact decision depends on the dependency contract.

Use exponential backoff with jitter to spread retries over time. Bound per-request attempts and total retry time, and use a retry budget so retry traffic cannot grow without limit during an incident. Respect provider retry-after headers when applicable. A retry must fit inside the caller's overall deadline; a downstream call should not continue for longer than the request that depends on it. For asynchronous work, use durable delayed retries rather than holding a request thread for minutes.

The important ambiguity is whether the first attempt took effect. A timeout from a GET and a timeout from a payment mutation have different consequences. Use idempotency keys or operation IDs for mutations, and reconcile external outcomes before issuing a new logical operation. Assign one clear layer to own retries where possible: client, SDK, gateway, service, mesh, and broker retries can otherwise multiply.

During overload, retries may need to be suppressed or shed before new requests are rejected. Monitor attempts per logical request, retry success rate, retry-generated QPS, timeout rate, and downstream saturation. A policy that eventually succeeds after many attempts can still be harmful if it destroys p99 latency and recovery capacity. State the failure model and the side-effect guarantee when explaining a retry strategy.

### Q94. What does a circuit breaker do, and when can it make things worse?

A circuit breaker stops or limits calls to a dependency when observations indicate that continuing to call it is unlikely to succeed or may worsen overload. In a common state machine, **closed** allows calls while tracking outcomes; **open** rejects/fails fast for a cooling period; **half-open** permits a limited number of probes to test recovery. The exact thresholds, rolling window, and treatment of slow calls depend on the implementation.

It protects resources by preventing every caller from waiting on a dependency that is already failing. But a breaker is not a queue, a timeout, or a bulkhead. A timeout bounds waiting, a bulkhead caps concurrent resource usage, and a breaker suppresses calls based on failure observations. They often complement one another.

Bad configuration can create problems. A shared breaker that is too sensitive may open during a brief noisy period and cut off healthy traffic; one breaker per process may open at different times and create a synchronized flood of half-open probes. Treating business 4xx responses as dependency failures can open the breaker incorrectly. Falling back to a cache or alternate provider can also be unsafe if the fallback violates data freshness or correctness.

Define which outcomes count as failures, minimum sample volume, open duration, probe limits, and the behavior when open. Monitor open/half-open transitions, rejected calls, slow-call rates, and recovery. Use jitter or controlled probes at scale, and do not let each layer independently retry through the breaker. In an interview, explain what protects the caller from waiting, what protects the dependency from concurrency, and how the system recovers after the dependency is healthy again.

### Q95. What is a bulkhead, and how is it different from a circuit breaker?

A bulkhead isolates resource consumption so a failing or slow workload cannot consume the capacity needed by unrelated work. It can mean separate thread/connection pools, per-dependency concurrency limits, separate queues, independent worker pools, or dedicated capacity for critical tenants. The analogy is a ship's compartments: flooding one compartment should not sink the whole vessel.

For example, if a service uses one shared pool for recommendations and checkout, slow recommendation calls may occupy every worker and prevent checkout from completing. A dedicated pool or concurrency limit for recommendations caps the damage. However, bulkheads consume resources and can leave capacity idle in one pool while another is saturated; sizing and prioritization must reflect the workload.

A **circuit breaker** observes failures or slow calls and temporarily stops sending calls to an unhealthy dependency, typically moving through closed, open, and half-open states. A bulkhead limits how much resource is exposed; a circuit breaker limits calls while failure is observed. They are complementary, not substitutes. A breaker alone may still allow too many concurrent calls before it opens, and a bulkhead alone may keep issuing calls that will all fail.

Use bulkheads where dependency latency, tenant load, or task class can vary independently. Set queue and pool bounds, define rejection/fallback behavior, and measure wait time, concurrency, and rejected work. Avoid proliferating tiny pools without evidence, since they add tuning complexity and can reduce aggregate utilization.

### Q96. Why can retries cause cascading failures?

Retries turn one logical request into multiple physical attempts. When a dependency becomes slow, callers may time out and retry while the original work is still running. The dependency now has more concurrent requests, becomes slower, triggers more retries, and may fail completely. This positive feedback loop is a common route from a small incident to a system-wide outage.

For example, 1,000 requests/s with an average of two extra attempts can create up to 3,000 attempts/s at the dependency while the original request rate is unchanged. If each attempt fans out to more services, the multiplication can be larger. Multiple retrying layers multiply policies: three client attempts combined with three gateway attempts and three service attempts can theoretically produce up to 27 downstream attempts for one original request, depending on the exact path and early failures.

Mitigate this with bounded attempts, exponential backoff with jitter, an end-to-end deadline, retry budgets, concurrency limits, circuit breakers, bulkheads, and load shedding. Retry only transient failures and only when repeated execution is safe. Prefer a single well-defined retry owner for a given operation. During overload, drop or reject retries as well as new requests; otherwise retries can consume the capacity needed for fresh work and recovery.

Instrument original request rate separately from attempt rate, plus retry count, timeout rate, queue age, active concurrency, and successful throughput. A lower error rate after the system is saturated may require reducing traffic sharply before capacity returns, because only a fraction of instances may remain healthy. Validate the policy under dependency latency and partial-failure tests.

### Q97. Liveness versus readiness?

A **liveness** check answers whether a process is functioning well enough that restarting it may help. A **readiness** check answers whether the instance should receive new traffic right now. They are not synonyms and should not be tied blindly to the same dependencies.

If liveness depends on a remote database, one database outage can cause every application instance to restart at once. The restart storm does not repair the database and may increase connection churn. Readiness can reasonably consider whether the instance can serve critical work, but if every instance becomes unready because of one shared dependency, the load balancer may remove all capacity. Separate optional capabilities from those required for each route.

Keep probes cheap, bounded, and stable. Use thresholds/hysteresis to avoid flapping, ensure readiness is updated before shutdown, and drain ongoing requests. A process-level probe can be healthy while a business journey is broken because of a bad dependency response or incorrect data; synthetic checks and service-level indicators complement health endpoints.

For stateful writers, neither liveness nor readiness proves exclusive write ownership. Promotion needs an explicit protocol, epoch, or fencing rule. In an interview, describe the consequence of each probe failure: restart, stop new traffic, alert only, or mark a dependency degraded. The action should fit the failure, not create a synchronized system-wide reaction to a shared outage.

### Q98. How should a service recover after a long downstream outage?

Recovery is often more dangerous than the initial outage because a backlog, retry queue, and many waiting clients can all resume at once. Simply marking the dependency healthy and releasing all traffic may recreate the overload. The service needs a controlled recovery plan, not just automatic retries.

Before resuming, determine whether queued work is still valid, whether ordering matters, whether any items have expired, and whether the downstream quota or capacity has changed. Use a bounded replay rate, concurrency limits, exponential backoff with jitter, and priority classes that reserve capacity for live critical traffic. Drain oldest or most urgent work according to the business objective, not necessarily FIFO for every queue.

Make consumers idempotent and reconcile uncertain operations before replaying them. A message may have succeeded downstream before the response was lost; replaying with a new operation identity can duplicate the side effect. For external providers, check retry-after signals and account-level quotas. If the backlog cannot drain before its business deadline, expire or reject stale work with a visible reason rather than spending capacity on useless attempts.

Track queue depth and age, successful drain rate, provider throttling, duplicate/conflict count, and live-traffic latency. Estimate time-to-drain using net throughput (mu-lambda), and communicate the estimate as conditional on stable rates. Increase capacity gradually and monitor the impact; a dependency returning to health does not mean it is ready for peak traffic immediately.

## 13. Security and abuse prevention

### Q99. Authentication versus authorization?

**Authentication** establishes who or what is making a request. **Authorization** decides whether that authenticated identity may perform a specific action on a particular resource under the current policy. Authentication might validate a user session, signed token, client certificate, or workload identity; authorization evaluates permissions, ownership, tenant boundaries, resource state, and requested action.

A valid token is not proof that every operation is allowed. A user may be authenticated but lack permission to refund an order, read another tenant's records, or access an administrative endpoint. Conversely, authorization based only on a user ID supplied in a request is unsafe because the caller may change it. The service owning the resource must enforce the relevant policy, even when a gateway performs authentication.

Design tokens and identity claims carefully. Validate signature, issuer, audience, expiry, key rotation, and permitted algorithms; avoid trusting arbitrary headers that a caller could spoof. Keep access tokens short-lived where appropriate, and define how revocation and permission changes propagate. Caching authorization decisions can improve latency but creates a stale-permission window that must be explicit—especially for account disablement and high-impact administrative actions.

In a multi-tenant system, tenant context must be derived from trusted identity or a validated membership relation and propagated through every data access path. A service should not rely on the UI or gateway alone to filter rows by tenant. In an interview, trace identity from entry to the resource-owning service and state which component makes the final authorization decision.

### Q100. How do you secure service-to-service communication?

Use workload identity, authenticated transport, least-privilege authorization, and safe secret/key lifecycle management. Private network placement is useful defense in depth but should not be treated as proof of identity: internal networks can be misconfigured or compromised, and requests may traverse shared infrastructure.

Common mechanisms include mutual TLS between workloads, short-lived workload credentials, signed tokens with narrowly scoped audience/claims, or platform identity systems. The receiving service should validate the caller's identity and authorize the specific operation. A service mesh can automate certificate issuance, rotation, and mTLS, but it does not determine whether a caller may cancel a particular order or read a tenant's records.

Separate user identity from workload identity. A service receiving a user request may need both: its own workload credential to call another service and the user's delegated identity or explicit authorization context. Avoid forwarding a broad bearer token to every downstream service or trusting caller-supplied identity headers. Define how delegation, audience restriction, and service-to-service permissions work.

Operationally, rotate credentials without simultaneous outage, monitor certificate expiry and authentication failures, protect secrets in logs and dumps, and design behavior when the identity control plane is temporarily unavailable. Local verification of short-lived signed credentials can reduce request-path dependence on an identity provider, but it has a revocation/freshness trade-off. Threat-model lateral movement, replay, confused-deputy behavior, and compromised service accounts. In a senior interview, explain identity issuance, verification, authorization, rotation, and incident revocation—not just “use HTTPS.”

### Q101. How do you prevent API abuse?

Treat abuse prevention as layered admission control rather than one global rate limit. Apply inexpensive controls early—request size limits, connection limits, malformed-input rejection, IP/network heuristics, and coarse edge throttling—before requests consume database or provider capacity. Then apply authenticated limits by user, API key, tenant, endpoint, and operation cost. An upload, search query, or bulk export may be much more expensive than a simple read.

IP-only limiting is insufficient: many legitimate users can share a NAT address, while distributed attackers can rotate IPs. Identity-aware controls improve fairness but can be bypassed when authentication is free or easy to automate. Combine limits with quotas, concurrency caps, behavioral detection, bot signals, proof-of-work or challenge mechanisms where appropriate, and explicit protection for expensive endpoints. Do not put sensitive security decisions solely in a client-controlled header.

Bound resource use at every stage: request body size, decompression ratio, query complexity, page size, execution time, concurrent jobs, and output bytes. For long-running work, queue it under an authenticated quota and return an operation ID instead of holding a request thread. Respect provider quotas across all service instances; per-instance limits can multiply with autoscaling.

Define failure behavior. If a rate-limit store fails, a strict paid quota may need fail-closed behavior, while a coarse protective limit may allow a bounded local fallback. Return clear 429/4xx responses and retry guidance, but avoid revealing sensitive detection rules. Monitor rejection rate, costly endpoint usage, tenant fairness, false positives, and attack-driven traffic. Load-test controls so the limiter itself does not become the bottleneck.

### Q102. What security issues arise in multi-tenant systems?

The core risk is **cross-tenant access**: a caller authenticated for tenant A can read, change, export, or infer tenant B's data. This can happen through a missing tenant filter, a cache key that omits tenant identity, an object-storage path derived from user input, a search query that forgets its tenant constraint, a background job running with excessive permissions, or an authorization bug in an internal service.

Model tenant identity as a trusted part of the request context, not an arbitrary parameter. Resolve membership and permissions from trusted claims or a validated relationship; then enforce isolation at the resource-owning service and, where supported, at the datastore layer with row-level security, separate schemas/databases, or separate accounts for stronger isolation. Every layer must agree: database queries, caches, search indexes, object storage, event payloads, logs, exports, and administrative tools.

Test isolation with adversarial cases, not only happy paths. Try changing object IDs, tenant headers, cursor tokens, cache keys, event routing keys, and background-job parameters. For async processing, persist the tenant identity with the job and revalidate authorization when appropriate. Ensure global administrators and support tooling have audited, time-bounded access.

Tenant isolation also includes resource fairness and operational blast radius. Per-tenant quotas prevent one tenant from starving others, but quota controls are not a substitute for data access controls. A shared table may be cost-efficient; dedicated database or encryption-key boundaries may be required for particularly sensitive workloads or contractual requirements. Document what isolation level is promised and verify it in tests, logs, backup/restore, and incident response.

## 14. Observability and operations

### Q103. What should you monitor in a distributed service?

Monitor whether users can complete important journeys, not just whether machines are running. Start with service-level indicators such as successful request ratio, latency percentiles, freshness, and completion time for asynchronous operations. Break them down by endpoint, tenant tier, region, and request class where that helps explain impact without creating unmanageable metric cardinality.

For each critical dependency, monitor the four basic resource/behavior dimensions: **latency, traffic, errors, and saturation**. Saturation means a resource is approaching its useful limit—CPU, memory/GC, thread pools, database connections, lock waits, queue depth/age, broker lag, cache memory, provider quotas, or network bandwidth. A service can have low CPU and still fail because requests are waiting for a connection or a lock.

For asynchronous systems, include oldest message age, arrival/completion rates, retry attempts, dead-letter count, workflow age, and end-to-end completion success. For databases, track query latency and plan changes, replication lag, lock waits, storage/IOPS, pool acquisition wait, and restore/failover status. For caches, track hit ratio alongside origin load, stale-data incidents, evictions, and refill concurrency.

Build alerts around user impact and fast error-budget consumption, not every noisy metric crossing a threshold. Dashboards help diagnose; alerts should trigger an action. Add release markers, dependency maps, and trace correlation. Monitor the monitoring system's own failure modes and protect sensitive data. A useful senior answer explains what signal detects the incident, which hypothesis that signal tests, and what response it should trigger.

### Q104. Logs, metrics, and traces: what is each for?

**Metrics** are numeric time-series aggregates such as request rate, error ratio, queue age, or p99 latency. They are efficient for dashboards, alerting, and spotting patterns over time. **Logs** are timestamped records of discrete events, useful for explaining what a specific process or operation did. **Traces** connect spans across service boundaries to show the path and latency of one distributed request or workflow.

They solve different questions. Metrics can tell you that p99 doubled after a release; a trace can show that 90% of the time is spent waiting on the inventory service; logs can reveal the exact timeout or error category for one correlated operation. Structured logs should include stable correlation identifiers and useful fields, not just unsearchable free text. Trace context should propagate through HTTP/RPC and messaging boundaries, including async correlation where a workflow crosses time.

Sampling is necessary at high volume, but tail-based sampling or error/latency-aware retention can help preserve diagnostically valuable traces. Never assume every request has a trace. Cardinality and cost matter: raw user IDs as metric labels can explode the time-series count, while high-volume debug logs can become a security and cost problem.

Avoid logging secrets, access tokens, payment credentials, or unnecessary personal data. Keep event IDs for investigations without exposing sensitive payloads. In a senior interview, describe how you correlate one user-visible failure across the entry point, asynchronous event, database transaction, and external provider, and how alerts remain useful when telemetry itself is delayed or unavailable.

### Q105. How do you investigate high p99 latency when CPU is low?

Low CPU does not imply unused end-to-end capacity. Requests may be waiting for database connections, row locks, a remote service, network buffers, a rate limiter, a thread/worker slot, or a queue. A small fraction of slow operations can dominate p99 while averages and aggregate CPU look healthy.

Start by confirming the measurement: endpoint, region, time window, traffic mix, and whether the percentile includes client/network time. Compare successful throughput, error rate, timeout rate, and percentiles before/after the change. Break latency into queueing, application processing, connection acquisition, database execution, downstream calls, and serialization using tracing or targeted instrumentation.

Check saturation at the correct layer: thread-pool active/queued tasks; connection-pool wait and hold time; database lock waits, slow plans, and replication lag; downstream tail latency; open connections; GC pauses; network retransmissions; and per-tenant or per-key skew. An overloaded shared dependency can cause lots of application threads to sleep, resulting in low application CPU and high latency.

Do not respond by immediately adding application instances or increasing pool sizes. That may send more concurrent work into the saturated database or provider. Test a hypothesis with a controlled change: reduce concurrency, shed optional fan-out, roll back a release, isolate slow queries, or route traffic differently. Then verify that p99 and successful throughput improve, not just that one metric moves. State how you would distinguish queueing delay from service time and locate the first growing queue in the request path.

### Q106. Canary versus blue-green deployment?

A **canary** gradually sends a small fraction of traffic or a limited cohort to the new version, then expands if health and business indicators remain within agreed thresholds. It reduces the blast radius and allows observation under real traffic, but the old and new versions run concurrently, so APIs, events, and database schemas must be compatible. A low-volume service may not produce enough signal quickly, and tenant or request skew can make a small percentage unrepresentative.

A **blue-green deployment** runs two complete environments: blue (current) and green (candidate). After validation, traffic is switched to green. This can make cutover and application rollback simpler, but duplicates capacity and does not automatically make database changes reversible. Long-lived connections, queued jobs, external side effects, and state written by the new version can complicate a traffic switch.

Choose based on rollback needs, environment cost, traffic volume, statefulness, and how safely versions can coexist. Define automated gates using error rate, latency, saturation, and business correctness (for example, order conversion or payment discrepancies). Use staged traffic and a bake time; instant rollback after one noisy sample can also cause oscillation.

The hardest part is often the data migration, not the binary deployment. Use expand–migrate–contract: add compatible fields, deploy readers/writers that support both forms, backfill, validate, then remove old fields only after no supported version uses them. A rollback plan must state whether the previous app can still read data written by the new version. Test this explicitly before relying on the deployment strategy.

### Q107. Why do schema compatibility and feature flags matter during deployments?

In a rolling deployment, old and new application versions may run simultaneously while sharing the same database and message topics. If the new version immediately renames a column, changes an event field's meaning, or assumes every consumer understands a new enum value, requests and messages crossing version boundaries can fail.

Use backward- and forward-compatible changes where feasible. A common database pattern is **expand–migrate–contract**: add a compatible field/table, deploy code that can handle old and new representations, backfill in bounded batches, verify, switch reads/writes, then remove the old representation after the compatibility window. For events, prefer additive optional fields, explicit schema versions, stable meanings, and consumer tolerance for unknown fields where the format allows it. Removing or reinterpreting fields is a breaking change even if the serialization still parses.

Feature flags can decouple deployment from activation, letting teams deploy code dark, enable it for a cohort, and disable the behavior if metrics deteriorate. But flags create multiple code paths and require ownership, expiry, access control, and testing. A flag cannot make an incompatible database write safe if the new schema is already being used by other services.

Rollback is safe only if the previous application can operate on the data and events already produced. Plan for mixed versions, background jobs, cached schemas, and consumers that lag. For each migration, state the compatibility window, validation method, cutover gate, and rollback point. Track flag usage and remove obsolete flags rather than letting temporary controls become permanent hidden architecture.

### Q108. What belongs in an incident runbook?

A runbook should help an on-call engineer restore a safe service under pressure without relying on tribal knowledge. Include the symptom and user impact, severity/escalation criteria, dashboards and logs, common hypotheses, safe diagnostic commands, and step-by-step mitigations with expected outcomes. Separate read-only diagnostics from operations that change state or risk data loss.

For example, a database saturation runbook might guide the engineer to inspect connection-pool wait, active sessions, lock waits, slow queries, and replica lag; then apply approved load shedding or pause a backfill before any risky failover. It should warn against common harmful reactions such as increasing connection pools without checking database limits, restarting every application instance, or replaying a queue at full speed after a recovery.

Document ownership and access paths, dependencies on external providers, emergency credentials with audited use, communication channels, and rollback steps. Include conditions under which the engineer must stop and escalate rather than continue—for example, uncertain payment outcomes or possible split-brain write ownership. State how to validate recovery through user-level SLOs and business invariants, not just a green health endpoint.

Runbooks should be tested in game days and updated after incidents. Record the last validation date and the telemetry needed to make the procedure work. Avoid hard-coded assumptions that become stale; commands and thresholds should be verified against current operational tooling. A senior candidate should explain how the runbook limits secondary damage while the team diagnoses the root cause.

## 15. Multi-region and disaster recovery

### Q109. Active-passive versus active-active regions?

In **active-passive**, one region serves the primary workload while another is warm or ready to take over. This simplifies write ownership and can reduce conflict handling, but failover takes time and the standby may have capacity, configuration, or data lag that is not visible during normal operation. The failover plan needs a known promotion procedure, routing change, and validation.

In **active-active**, multiple regions serve live traffic concurrently. This can reduce geographic latency and improve use of capacity, but “active-active” must be qualified: compute may be active in every region while writes for a given entity still route to one authoritative region. Truly concurrent multi-region writes require conflict prevention or resolution, including uniqueness, ordering, deletes, and business invariants. Conflict-free merge is not possible for every domain—two regions cannot independently guarantee a globally unique username during a partition without coordination or reduced write availability.

Compare data replication mode, regional failure behavior, latency targets, residency rules, recovery time, cost, and operational maturity. If writes are synchronously replicated across regions, commit latency depends on network distance and writes may be unavailable when the required quorum cannot be reached. Asynchronous replication lowers latency but can lose recent acknowledged data on failover depending on the commit/promotion policy.

For most designs, start from the business invariant and consider active-passive or per-entity single-writer before proposing global multi-writer. Test a real region loss, including routing, old-writer fencing, in-flight operations, identity/control-plane dependencies, queue ownership, and failback. DNS changes alone do not provide data correctness.

### Q110. Define RTO and RPO, and how do they affect architecture?

**Recovery Time Objective (RTO)** is the target maximum time to restore an acceptable service after a disruption. **Recovery Point Objective (RPO)** is the target maximum amount of data loss measured in time. They express different business tolerances: a service might need to return in 15 minutes but tolerate five minutes of lost updates, or tolerate no acknowledged-write loss while taking longer to restore read-only access.

RPO influences durability and replication choices. Asynchronous replication can have nonzero data loss if a primary fails before changes reach the promoted replica; synchronous or quorum-based commit policies can reduce that risk but may increase write latency and reduce availability under partition. Backups and point-in-time recovery address corruption or accidental deletion, not just node failure. The details depend on the selected database's commit and replication semantics.

RTO includes more than restarting a process. It may include detection, decision-making, leader fencing/promotion, DNS or routing convergence, client reconnection, data verification, cache/search rebuild, and workload ramp-up. A backup restore may meet RPO but miss RTO if restoring several terabytes takes hours.

Derive objectives from business impact and test them end to end. Run restore and failover exercises with measured volumes, verify application invariants, and account for dependencies such as KMS, identity systems, container images, DNS, and operator access. State whether the target is per component or for the complete user journey; a database RTO is not automatically the product's RTO.

### Q111. How do you handle a regional outage without conflicting financial writes?

First distinguish a region being unavailable from a network partition in which both regions can still accept requests but cannot coordinate. The latter is more dangerous: both regions may believe they are allowed to write. The design's first obligation is to preserve the financial invariant—no duplicate debit, inconsistent ledger transition, or falsely confirmed payment.

**Single-writer or globally consistent store:** Route writes for an account/ledger to one authoritative owner, or use a datastore whose documented transaction/consensus semantics provide the needed global guarantee. During failover, establish that the old writer cannot commit and verify the candidate's replication position against the RPO. Use an epoch/fencing token or the database's supported leadership protocol; a health check or DNS change alone does not transfer write authority safely.

**When ownership cannot be established:** Reject the financial mutation, or durably record a pending command only if the API explicitly promises deferred processing and the operation is still valid when executed. A queued command is not a completed debit. Preserve the original idempotency/operation identity and revalidate authoritative account state once service recovers. For payment-provider calls, reconcile with provider status before retrying any uncertain operation.

**Trade-off:** Strong coordination can reduce write availability and increase cross-region latency. If the business requires no conflicting financial writes, accepting independent writes in both regions is not an acceptable default. Active-active compute and local reads can still provide availability without allowing active-active writes for every entity.

**Recovery:** Fence the old region, promote one authority, verify ledger invariants, reconcile in-flight payment attempts and pending commands, refresh routing gradually, and rebuild derived views. Ensure failback does not make the recovered region a second writer.

**Signals:** Track owner epochs, rejected/pending writes, unknown payment outcomes, replication lag, ledger mismatches, routing convergence, and recovery time. In an interview, explain precisely what the client sees during the outage and what proof permits writes to resume.

### Q112. What is split brain, and how do you prevent it?

Split brain occurs when two nodes or regions both believe they are the authoritative writer for the same logical data. It can happen after a network partition, failed leadership transition, stale routing, or an unsafe failover procedure. Both sides may accept conflicting writes, and reconciliation can be impossible if the business operations cannot be merged.

A health check only reports a signal; it does not establish exclusive ownership. Prevent split brain with a protocol that defines who can promote, how quorum or a trusted coordination service selects authority, and how the previous owner is fenced. A monotonically increasing epoch/fencing token should be checked by the protected write resource so a paused old leader cannot resume and commit with stale authority. The storage layer must enforce the rule; simply writing the token in application logs is not sufficient.

During regional failover, isolate or fence the old writer before enabling the new one, verify the candidate's data position and promotion eligibility, then update routing. If the old region is unreachable and cannot be proven fenced, the system may need to reject writes until ownership is safe. This reduces availability but protects correctness.

Test the partition scenario where both regions can serve clients but cannot communicate with each other. Define which operations, if any, remain safe on each side. Global unique identifiers do not solve conflicting updates to the same business entity. Explain the recovery plan for divergent data and how the system detects stale leaders, not only the ideal leader-election path.

### Q113. Why are backups and replication different?

Replication maintains one or more current copies to improve availability, read capacity, or durability under certain failures. Backups preserve recoverable historical states so data can be restored after logical corruption, accidental deletion, a faulty migration, or malicious modification. They solve overlapping but different problems.

If an application issues a destructive update or an attacker deletes records, synchronous or asynchronous replicas may faithfully reproduce the damage. A backup or point-in-time recovery log may let the operator return to a state before the mistake. Conversely, restoring an old backup to recover one failed node may take far longer than promoting a healthy replica and can lose changes made after the backup point.

Define backup frequency, retention, encryption, immutability/access separation, point-in-time recovery coverage, and the target restore granularity. Consider whether backups include schema/configuration and how to rebuild derived data such as search indexes. Legal deletion and data residency requirements must also account for backup retention and restore procedures.

A backup is only useful if it can be restored. Regularly restore into an isolated environment, validate record counts and business invariants, measure restore time, and rehearse how to prevent restored data from overwriting newer valid data. Record RPO/RTO from the measured exercise. In an interview, say what failures replication handles, what failures backups handle, and how you would prove recovery works.

## 16. Cost and architectural evolution

### Q114. How do you reduce cost without violating latency or reliability?

Optimize cost per useful business outcome, not the cloud bill in isolation. Start by measuring the major cost drivers: compute utilization, idle capacity, database instance/storage, requests and egress, cross-region replication, cache memory, observability retention, managed-service premiums, and third-party API charges. Tie them to workload and service objectives so cheaper infrastructure does not create slower or less reliable user journeys.

Low-risk improvements often include rightsizing, autoscaling with conservative minimum capacity, scheduling nonproduction environments, compressing or batching transfers, reducing unnecessary log volume, moving cold data to cheaper storage, and eliminating repeated work with caching or precomputation. But each has trade-offs: scale-to-zero can worsen cold starts; compression consumes CPU; shorter log retention can impair investigations; aggressive autoscaling may overload a shared database.

Do not remove redundancy without understanding its impact on failure headroom and RTO/RPO. A service provisioned only for average traffic may be cheap until a spike or zone loss creates a severe outage. Use load tests and capacity forecasting to find safe efficiency, and model cost at peak as well as average. For caches and replicas, include their operational cost and the origin work they save; they may increase the bill but reduce incidents or improve latency.

Treat savings as an experiment: define the metric, the acceptable guardrails, rollback trigger, and measurement period. A senior answer describes both the financial benefit and the reliability debt being accepted—or avoided.

### Q115. When is a modular monolith preferable?

A modular monolith deploys as one application but enforces clear internal module boundaries, ownership, and interfaces. It is often preferable when a product is evolving quickly, the team is small or medium-sized, domain boundaries are still changing, and most operations benefit from simple local transactions and low-latency in-process calls.

The key word is **modular**. Modules should have explicit APIs, own their data structures, and avoid reaching into each other's internal tables or classes arbitrarily. If every feature can directly change every other module's internals, the architecture is a monolith without meaningful boundaries. Enforcing those boundaries makes later extraction more deliberate.

Compared with microservices, a modular monolith has fewer network failure modes, simpler local debugging and deployment, less duplicated infrastructure, and easier cross-module transactions. Its costs include coupled release cadence, a shared process/resource failure domain, and less independent scaling. Strong module separation and runtime metrics can reveal which capability would benefit most from extraction later.

Split a module when the benefit is concrete: independent scaling, a different security/reliability boundary, independent ownership/release needs, or a stable domain contract. Before extracting, define data ownership, API/event semantics, failure behavior, and migration strategy. In an interview, proposing a modular monolith can be the more senior answer when it satisfies requirements; the sophistication is in explaining the boundary and evolutionary path, not in maximizing service count.

### Q116. Explain the Strangler pattern.

The Strangler pattern replaces parts of a legacy system incrementally rather than rewriting everything in one release. A routing layer or façade directs selected capabilities to a new implementation while remaining functionality stays in the old system. Over time, more responsibilities move behind the new boundary until the legacy path can be retired.

A safe migration begins by choosing a narrow business capability with measurable behavior and a clear owner. Define the contract, identify the authoritative data store, and map hidden dependencies such as batch jobs, reports, integrations, and administrative tools. Build compatibility adapters when clients cannot migrate at once. Route a small cohort or operation type to the new implementation, compare results, and expand only after correctness and latency are validated.

Data is usually the hardest part. Avoid uncontrolled dual writes to old and new stores. Use a defined source of truth, CDC/event propagation or controlled migration, stable IDs, versioned routing, and reconciliation. Shadow reads can compare outputs without exposing the new result; shadow writes require much more caution because they may create real side effects. Ensure the new version can handle in-flight workflows and old event schemas.

Rollback has a boundary: once the new system has accepted writes that the old one cannot understand, routing back may lose or corrupt state. Plan reverse replication or forward repair, define the point of no return, and retain audit data. Monitor traffic split, mismatch counts, migration lag, business outcomes, and remaining legacy dependencies. Strangling is an operational program with phases and exit criteria, not just a proxy in front of two applications.

### Q117. How do you evolve from thousands to millions of users?

Do not tie architecture to a user-count milestone alone. A million mostly inactive users can be easier than ten thousand users generating expensive, highly concurrent writes. Evolve based on measured workload: peak QPS, read/write mix, payload size, storage growth, key skew, latency objective, and dependency quotas.

A sensible evolution path is to begin with a well-instrumented, modular application and an appropriate primary database. First improve correctness and query plans, add indexes deliberately, bound connection pools and queues, and load-test. Add caching for repeated reads, replicas for read workloads that tolerate their consistency, and asynchronous workers for durable work that need not block the user response. Apply admission control and per-tenant fairness before overload.

When the bottleneck persists, scale the constrained component: more application instances for stateless CPU-bound work; database tuning or vertical scale for database limits; partitions for retention/query pruning; sharding for independently distributed data; additional broker partitions when parallelism and ordering allow it. Every step changes operational failure modes, so add observability, runbooks, rollback, and capacity headroom as the architecture grows.

Avoid “scale out everything” as a default. Stateless app nodes may be cheap to add, but each may create more database connections and downstream concurrency. A cache can lower normal load but worsen a cold-start incident. More services may allow independent scaling but add network and consistency cost. Explain what is limiting the system today, what evidence supports the next step, and how you would safely migrate rather than drawing a future-state diagram disconnected from current constraints.

### Q118. How do you reason about cost per request?

Cost per request is a useful efficiency measure, but define the denominator carefully. Dividing total monthly infrastructure cost by raw HTTP requests can make a system look cheaper by serving requests that fail or do not complete the intended user journey. Prefer cost per successful business operation where possible, and separate fixed baseline cost from variable cost.

Estimate major cost drivers: compute time, database queries/IO, cache and broker operations, object storage, data transfer/egress, logging/tracing, cross-region replication, and third-party API charges. A single request may trigger several downstream calls or enqueue work that completes later; include that fan-out and asynchronous cost rather than counting only the edge request. Retry storms, cache misses, and failed operations can raise cost without increasing useful output.

Compare architectures under the same workload and SLO. A cache or replica may add direct cost but lower database capacity, reduce p99 latency, or improve resilience. Compression can reduce egress while increasing CPU. Autoscaling can reduce idle cost but may increase cold starts and fail to react quickly to spikes. Evaluate both steady state and failure conditions.

Use measurements from load tests and cloud billing, then watch cost per successful operation over time by endpoint or workload class. In interviews, show which costs scale linearly, which are shared/fixed, and how you avoid lowering cost by silently weakening reliability or correctness.

## Core concept deep dives: partitioning, sharding, and traffic infrastructure

These modules explain concepts that often appear in senior HLD interviews as follow-up questions. They complement the numbered Q&A with mechanisms, assumptions, and failure behavior; they are not a reason to add technology without a requirement.

### Deep dive A. What is the difference between partitioning, sharding, replication, and clustering?

These terms describe different dimensions of a data system:

- **Partitioning** divides one logical dataset into smaller pieces according to a rule. Range partitioning groups values such as dates into ranges; hash partitioning maps a key through a hash; list partitioning assigns explicit value groups. Partitions may remain within one database instance.
- **Sharding** distributes pieces of a logical dataset across independently managed database nodes. A routing layer maps a record/query to the shard that owns it. A shard can itself have local partitions and replicas.
- **Replication** keeps copies of the same logical data on multiple nodes. It can improve availability and eligible read capacity, but asynchronous copies may lag. Replication alone does not necessarily raise the write capacity of one leader.
- **Clustering** is a broad term for multiple nodes operating as one system. A cluster may provide replication, partitioning, failover, or combinations; the word alone does not specify consistency or durability guarantees.

Example: an orders table might be partitioned by month for pruning and retention, sharded by tenant ID across database nodes for capacity, and replicated within each shard for recovery/read scaling. The design then has separate concerns: local partition pruning, shard routing/rebalancing, and replica lag/failover. Partitioning can improve maintenance without overcoming a single-node write limit; sharding can add capacity while making cross-shard queries, uniqueness, transactions, and operations harder.

### Deep dive B. How does range partitioning differ from hash partitioning?

**Range partitioning** places rows into ordered ranges—for example, orders by month. It works well for time-window queries, partition pruning, and lifecycle operations such as archiving old periods. The risk is skew: if all new writes land in the newest range, that partition can become hot. In a distributed store, boundary splits and range movement also need operational planning.

**Hash partitioning** applies a hash function to a key to spread values across partitions. It often distributes point writes more evenly and avoids a single newest-range hotspot, but weakens natural ordering: a range query may need to contact many partitions. Changing the partition count or hash mapping can require data movement unless the database hides it behind a virtual-partition/routing layer.

A composite strategy is common: partition by time for retention and then distribute each time partition by a hash key. This supports both lifecycle and write distribution, at the cost of additional routing and query complexity. Choose based on dominant access patterns, data growth, skew, and operational constraints. Explain which queries become efficient, which become scatter-gather, and how the design handles a hot partition.

### Deep dive C. How does database sharding work end to end?

A sharded database divides rows by a routing key. For example, `hash(tenant_id) -> shard_id` maps a tenant to a shard. The application, database proxy, or routing service uses that mapping before executing a query. A normal request should include the shard key and route to one shard; a query without the key may require scatter-gather—send work to several shards, merge results, and apply global ordering/pagination. That raises fan-out latency and failure exposure.

Sharding changes familiar database assumptions:

- **Uniqueness:** a local unique index may enforce uniqueness only within one shard. Global uniqueness may need a coordinator, allocated ID ranges, a dedicated uniqueness service, or routing that ensures conflicting values share an owner.
- **Joins/transactions:** operations spanning shards can be slow or unsupported. Colocate related data when query patterns and invariants justify it; otherwise use explicit workflow coordination or a denormalized read model.
- **Rebalancing:** adding a shard does not move old data automatically. Plan snapshot copy, capture of concurrent changes, validation, versioned routing, cutover, rollback/forward repair, and cleanup.
- **Hot keys/tenants:** balanced row counts do not guarantee balanced traffic. One tenant or key may dominate a shard and require dedicated placement or sub-sharding.
- **Operations:** backups, restore, schema changes, failover, monitoring, and capacity planning become per-shard and fleet-wide problems.

Do not shard merely because a table is large. First test query/index improvements, native partitioning, read replicas, caching, archival, vertical scaling, and workload isolation. Shard when measured capacity or isolation needs justify the added operational and consistency cost.

### Deep dive D. What is the difference between a shard key and a partition key?

Products sometimes use these terms interchangeably, so define them in the design. Here, a **partition key** selects a partition of a logical dataset; a **shard key** selects which independently managed shard owns the record. A shard can contain multiple local partitions, and distributed SQL products may hide physical routing from the application.

The key needs to balance data distribution, common query locality, write throughput, and business invariants. A `tenant_id` key makes tenant-scoped reads efficient but may create a hot shard for a large tenant. An `order_id` key may spread traffic well but make “all orders for one tenant” expensive. A monotonic timestamp can help range scans but direct new writes to one hot range.

Before deciding, list the high-volume queries, write paths, uniqueness constraints, and operations that must be atomic or ordered. Estimate cardinality and skew rather than treating them as the same thing. A key with many distinct values can still be skewed if most requests target a small subset. In an interview, explain the query patterns the key localizes, the queries it makes more expensive, and how you migrate if the distribution becomes unacceptable.

### Deep dive E. How do you detect and mitigate hot shards or hot partitions?

A hot shard receives a disproportionate amount of work while other shards remain underused. Row count alone may hide it: one shard can contain a modest share of the data but most requests. Compare per-shard QPS, bytes, CPU, I/O, lock waits, queue depth, p95/p99 latency, replication lag, and tenant/key contribution. Use per-partition rather than fleet averages.

Mitigations depend on the cause:

- For hot reads, use request coalescing, caching, or replicated/local read paths if freshness allows.
- For a hot tenant, move it to a dedicated shard or split it into buckets. This increases routing and can complicate tenant-wide queries/transactions.
- For monotonic range hotspots, use hash/bucket distribution or redesign the key, recognizing that ordered scans may become more expensive.
- For hot counters, consider sharded counters, batching, or approximate aggregation if the business invariant permits it.
- For uneven query cost or row size, rebalance by resource consumption rather than row count alone.

Do not blindly add partitions: more partitions will not fix a serialized hot key or global lock. Validate the change against locality, ordering, uniqueness, and migration requirements. In an interview, explain how you found the skew, why the chosen mitigation addresses the bottleneck, and how you know the hotspot actually moved or disappeared.

### Deep dive F. How do forward proxies, reverse proxies, load balancers, gateways, and service meshes fit together?

These roles overlap because they describe responsibilities rather than always separate products.

- A **forward proxy** represents clients accessing external destinations and can enforce outbound policy.
- A **reverse proxy** receives requests on behalf of servers and can terminate TLS, route, compress, or apply request limits.
- A **load balancer** selects an eligible backend among several candidates, at L4 or L7.
- An **API gateway** commonly manages external API policies such as routing, authentication integration, quotas, request transformations, and versioning.
- A **service mesh** focuses on service-to-service communication, often providing workload identity, mTLS, traffic policy, and telemetry using proxies/control-plane configuration.

A request might traverse DNS/global routing, CDN or edge protection, an L4/L7 load balancer, an API gateway, an application service, and an internal service proxy. Not every system needs every layer; an ingress controller can combine gateway and reverse-proxy functions.

At each hop, ask who owns TLS termination, caller identity, authorization, discovery, timeouts, retries, buffering, and logs. If multiple layers independently retry, they can multiply requests; three attempts at one layer combined with three at another can result in up to nine downstream attempts. Establish an end-to-end deadline and a clear retry owner where possible. In an interview, identify the purpose and failure mode of each layer instead of drawing every box by default.

### Deep dive G. How do DNS, a global traffic manager, a load balancer, and service discovery differ?

**DNS** resolves names to records and is often part of reaching a service, but resolver/client caching means it is not an instantaneous per-request failover mechanism. **Global traffic management** chooses a region or endpoint based on policy such as health, geography, or latency. **Regional load balancing** distributes traffic among targets within a network/region. **Service discovery** helps a client or proxy find current internal service instances.

The layers have different scopes and health signals. A region can accept TCP connections while business writes fail. A global manager may steer new connections away from a region but cannot necessarily move established WebSockets or reverse a write already accepted there. A service registry can return stale endpoints even when its control plane is healthy.

Design failover around user-visible behavior: what qualifies a region as healthy, whether data is current enough, how write ownership changes, and how in-flight/unknown operations are reconciled. Routing change alone does not establish data authority. For stateful systems, use a promotion/epoch/fencing protocol. In an interview, distinguish traffic steering from replication and safe write-ownership transfer.

### Deep dive H. What does a health check mean at each layer?

A **liveness check** asks whether a process should be restarted; **readiness** asks whether it should receive new traffic; **dependency checks** report the state of databases, brokers, or identity services; **synthetic checks** exercise an end-to-end user journey. These signals answer different questions and should not be collapsed into one Boolean.

If readiness depends on every optional dependency, a recommendation-service outage can remove all application instances from rotation. If liveness depends on a shared database, the outage can trigger a restart storm. Conversely, a process probe can be green while all business requests fail due to an exhausted pool or invalid data.

Keep probes cheap and bounded, use thresholds/hysteresis to avoid flapping, and separate process-local health from shared-dependency degradation. During planned removal, mark the instance unready and drain ongoing work. For stateful writers, a health check alone cannot decide who owns writes; use a leader/lease/epoch protocol with fencing at the protected resource.

In an interview, state what action follows each probe: restart, stop new traffic, alert, or degrade one feature. The action must match the failure and not amplify it.

### Deep dive I. What is the difference between queueing, backpressure, and load shedding?

**Queueing** absorbs a temporary difference between arrival and service rates but does not create capacity. **Backpressure** asks upstream producers to slow down or stop when downstream capacity is limited. **Load shedding** deliberately rejects or drops lower-priority work to keep the system stable and preserve critical operations.

Let arrival rate be λ and successful service rate be μ. If λ is persistently greater than μ, backlog grows at about λ-μ. If the backlog is 3.6 million messages, arrivals are 4,000/s, and workers complete 12,000/s, the net drain is 8,000/s and ideal drain time is 450 seconds (7.5 minutes), assuming stable rates and no hidden bottleneck. If arrivals equal/exceed completions, the queue will not drain.

Use bounded queues, age/depth alerts, producer throttling, concurrency limits, deadlines, priorities, and expiry policy. Rejecting an interactive request quickly can be safer than accepting work that will miss its deadline. For a mutation, never shed after it may have committed without returning an operation identity or providing status lookup.

### Deep dive J. How should a senior engineer explain a design decision?

Use a repeatable chain: **requirement → invariant → workload → design choice → alternatives → failure behavior → operational proof**.

For example: “Most reads are tenant-scoped, and the current primary is approaching its write limit. I would first measure whether query tuning, indexes, partition pruning, or replicas address the bottleneck. If writes still require distribution, I would consider sharding by tenant because it localizes common access patterns. I would measure tenant skew and explain global search, uniqueness, migrations, backups, and cross-shard reporting. I would not claim that sharding is free or guarantees balanced load.”

This structure shows judgment rather than vocabulary recall. It gives interviewers useful places to probe: key selection, consistency, failover, capacity, migration, and cost. State assumptions, name the weakest part of the design, and identify the measurement or failure test that would validate it. You do not need a production-ready blueprint in every detail, but you must defend the decisions that affect correctness and system limits.

### Deep dive K. How do database indexes work, and why can adding an index make a system slower?

An index is an auxiliary data structure that helps the database find rows without scanning the full table. B-tree/B+ tree indexes are common for equality lookups, ordered access, and range scans; hash, bitmap, inverted, spatial, or specialized indexes suit different workloads and engines. An index helps only when the query planner can use it and the index is selective enough to reduce work.

Composite-column order matters. An index on `(customer_id, created_at)` may help fetch one customer's recent orders, but it may not efficiently serve a query filtering only by `created_at`. Expressions, implicit type conversions, collation, null handling, and query shape can also prevent use of an expected index. Inspect actual query plans and rows scanned rather than assuming an index is used.

Every insert/update/delete may have to maintain indexes. They consume disk and memory, enlarge backups/replication traffic, and can increase write amplification and page contention. Too many overlapping indexes can slow writes without materially improving reads. A query returning a large fraction of a table may still be faster with a scan.

In an interview, explain the workload and evidence: selectivity, actual plan, rows estimated versus returned, index size, query latency, and write overhead. “Add an index” is not a universal answer; it will not fix N+1 queries, a hot row lock, a saturated connection pool, or a bad data model.

### Deep dive L. How do B-tree indexes differ from LSM-tree storage, and when does the distinction matter?

A B-tree-based engine maintains ordered pages and updates them through its storage mechanisms; it is commonly effective for mixed transactional workloads, point lookups, and range scans. Actual behavior also depends on write-ahead logging, buffer pools, page splits, and vacuum/compaction behavior, so it is misleading to say every write is simply “in place.”

An LSM-tree typically buffers writes and writes immutable sorted files, then merges them through compaction. That can make sustained write ingestion efficient, but read paths may consult multiple structures and compaction can create write amplification, temporary disk-space demand, or resource spikes. Different implementations use filters, caches, and compaction strategies to manage those trade-offs.

Neither structure is universally faster. Compare the actual workload: read/write ratio, point versus range queries, update frequency, latency tail, data size, compaction behavior, durability configuration, and operating requirements. A write-heavy stream might suit LSM-based storage; a transactional system with selective range queries may benefit from B-tree behavior. In an interview, explain the trade-off and validate it with representative benchmarks instead of choosing a database solely by the data-structure name.

### Deep dive M. What should you understand about transaction isolation and concurrency control in HLD?

Isolation describes what concurrent transactions may observe. The exact behavior depends on database engine and isolation level; common anomalies include dirty reads, non-repeatable reads, phantoms, lost updates, and write skew. Snapshot-based isolation can prevent some anomalies while still allowing write skew in certain systems. Do not assume isolation-level labels have identical semantics across vendors.

Choose the guarantee from the invariant. For a seat reservation, a unique constraint or conditional update in the authoritative database is often clearer than “check availability, then insert.” For a balance read-modify-write flow, use an atomic update, version check, lock, or ledger model. A transaction alone is not enough unless its isolation and constraint rules actually protect the condition.

Stronger isolation may increase lock waits, aborts, deadlocks, and retries. Weaker isolation can improve concurrency but requires explicit handling of anomalies. Keep transactions short, avoid remote calls while holding database locks, and use bounded retries only after rolling back the full transaction. A retried database transaction must not duplicate an external side effect.

In an interview, name the invariant, show how the database enforces it under concurrent requests, state what the losing transaction observes, and explain the throughput/latency trade-off.

### Deep dive N. What are consensus, leader election, quorum, and fencing—and why are they not interchangeable?

**Leader election** chooses which node is currently authorized to coordinate work. **Consensus** is a distributed agreement protocol for choosing values or a replicated sequence of decisions despite specified failures and network delays. Protocols such as Raft use terms, logs, and quorum rules under their assumptions; choosing the node with the lowest ID is not a safe consensus protocol.

A **quorum** is the number/subset of participants whose responses satisfy a protocol. In a simple replicated-store model, (R+W>N) makes read and write sets intersect, but that arithmetic alone does not guarantee linearizability, freshness, or conflict resolution. Those depend on replication protocol, versions, leadership, read repair, membership, and what an acknowledgement means.

**Fencing** prevents an old or paused owner from writing after ownership has transferred. A lease timeout cannot stop a paused process from resuming. The protected resource must validate a monotonically increasing epoch/token and reject stale owners.

These mechanisms solve different parts of the problem: consensus may select a leader, a lease may limit expected ownership duration, fencing protects the resource from stale writers, and a quorum defines acknowledgements for a given operation. In an interview, specify the failure model and who actually validates write authority.

### Deep dive O. How do you choose between a relational database, key-value store, document store, wide-column store, search engine, and object storage?

Choose from the access pattern and correctness requirements, not from a fashionable category label. Relational databases are strong for structured relationships, constraints, joins, and transactions. Key-value stores fit predictable key-based access; document stores can fit aggregate-oriented flexible records; wide-column stores target distributed workloads with deliberately designed keys/queries; search engines provide text analysis and relevance; object storage holds durable blobs/files rather than arbitrary low-latency row updates.

The choice also depends on consistency, latency, write rate, retention, query flexibility, cost, backup/restore, security, team expertise, and the consequence of service failure. A “NoSQL” label does not guarantee scalability, and a relational database is not automatically limited to small systems.

A common architecture intentionally uses several stores: a relational database owns orders, CDC/outbox updates a search index, and object storage holds uploaded files. That creates propagation lag, schema mapping, deletion handling, rebuild, and reconciliation requirements. State the authoritative owner, allowed staleness, and how derived stores are recreated after corruption.

In an interview, compare two plausible choices against the actual workload and explain the operational cost of each. Use the simplest storage model that meets the requirements and can be safely operated.

## 17. Advanced HLD deep-dive Q&A

This section connects cross-cutting principles to real architecture decisions. The questions test whether you can explain the mechanism, its guarantees, alternatives, and operational consequences—not simply name a pattern or product.

### Q119. How do you turn requirements into an architecture instead of jumping directly to components?

Translate the product request into **user journeys, business invariants, measurable quality targets, and workload assumptions** before selecting components. For a ticketing system, journeys include search events, view availability, hold a seat, pay, and receive confirmation. The invariant might be “one seat cannot be sold to two orders”; latency targets differ between search and checkout.

Next quantify the expected traffic and data: peak requests, reads/writes per journey, event rates, record growth, retention, fan-out, and external quotas. Draw the simplest architecture that meets these constraints, showing trust boundaries, synchronous calls, asynchronous events, persistent stores, and the owner of each fact. Do not add Kafka, Redis, a service mesh, or a search engine until a requirement justifies the capability.

Then walk through the main flow and its failure windows: what if a dependency times out, a write commits but the response is lost, an event is duplicated, or a region fails? Choose consistency and recovery behavior per operation. Identify bottlenecks and describe how the design evolves if the workload changes.

Finally, state trade-offs and operational proof: what you would load-test, which SLOs you would measure, how you would roll out schema changes, and which unresolved assumptions could change the architecture. This flow is reusable across interview prompts. It produces a design that is tied to requirements and gives the interviewer clear places to probe, rather than a catalogue of technologies.

### Q120. How do you distinguish a system boundary from a deployment boundary?

A **system boundary** defines responsibility, data ownership, trust, and the contract one capability exposes to others. A **deployment boundary** defines what is built, released, scaled, and operated as one unit. They may align, but they are not the same. A modular monolith can have strong domain boundaries inside one process; multiple services can still behave as one tightly coupled system if they share data and must always deploy together.

Use a system boundary when responsibilities or invariants belong to different owners—for example, a payment service owns payment state and exposes commands/status rather than allowing an order service to update its tables directly. Use separate deployments when independent scaling, release cadence, failure isolation, security, or organizational ownership provides concrete value.

Every deployment split adds a distributed boundary: network failure, timeouts, retries, version compatibility, tracing, secrets, discovery, and possibly a saga or eventual projection. Every logical boundary does not need its own database server; a team may share managed infrastructure while preserving table/schema ownership and access controls.

In an interview, first draw ownership boundaries and data flow, then decide which components need separate deployment. Explain why the boundary is cohesive, what contract it exposes, and which operational cost follows from deploying independently. Avoid equating “well-designed system” with “many microservices.”

### Q121. How do you decide which work belongs on the synchronous request path?

The synchronous path should contain the work needed to return a truthful and useful answer to the caller. Include the minimum validation, authorization, and authoritative state transition needed to uphold the business invariant. Move work that can safely happen later—emails, analytics, search indexing, bulk reports, noncritical enrichment—to a durable asynchronous path when it would otherwise increase latency or couple availability to optional dependencies.

The choice is not “synchronous is bad, asynchronous is good.” If the user asks to reserve inventory, the system may need to commit a hold before claiming it is reserved. If the user requests an export, the service can durably record the job and return an operation ID rather than keep an HTTP request open for 20 minutes. If a payment is merely accepted for processing, the API must communicate PENDING/ACCEPTED instead of promising that payment is completed.

Async introduces its own contract: durable acceptance, queue retention, idempotent consumers, status visibility, retries, ordering, expiry, and dead-letter remediation. An in-memory fire-and-forget task is not durable acceptance. Decide what the system returns if the broker is down, whether the operation can expire before execution, and how clients discover final success or failure.

Estimate the latency and failure contribution of each synchronous dependency. Every required fan-out call adds latency and can reduce availability. Keep optional features off the critical path when the business allows degradation, but do not move a correctness-critical check behind eventual processing. In an interview, explain both why work is asynchronous and what guarantee is established before returning the HTTP response.

### Q122. How do you identify the authoritative source of truth in a system with several databases?

For each important business fact, name the system that owns the right to accept and validate changes. An order database may own order status, a payment ledger may own financial movements, and a search index may be a derived projection of catalogue data. There can be many copies, but the architecture needs an explicit rule for which one is authoritative when they disagree.

Ownership is about more than where data physically lives. It includes which service may mutate the fact, which invariants are enforced transactionally, how changes are published, and how a derived copy is rebuilt. If every service can update the same shared rows, service boundaries are weak even if the system has many databases.

For example, a search index may show “available” based on a recent event, but checkout should ask the inventory owner to perform an authoritative conditional reservation. Search is optimized for discovery and may lag. Likewise, an analytics warehouse can report a payment count but should not authorize a refund based on a possibly delayed aggregate.

Use contracts and identifiers to coordinate across owners: domain events/outbox, APIs for commands, CDC for data-change propagation where appropriate, and reconciliation for missed or conflicting updates. Document propagation lag and deletion behavior. Avoid treating a read replica, cache, materialized view, and transactional source as interchangeable. In an interview, describe what happens when two stores disagree, which store wins, and how the other copies converge without silently overwriting newer data.

### Q123. What is a materialized view, and when is it useful?

A materialized view stores the result of a query or transformation so that readers do not have to recompute it every time. It may live inside a database, a search engine, a key-value store, or another analytical/read-optimized system. Examples include a user's order summary, a product-search document, a news-feed candidate list, or daily revenue aggregates.

Materializing a view trades cheaper reads for extra write and operational work. The system must decide how the view is updated: synchronously in the source transaction, asynchronously from an outbox/event stream, through CDC, or through periodic batch recomputation. Synchronous maintenance can add latency and couple the write to the projection; asynchronous maintenance creates a freshness window and requires retry, deduplication, ordering, deletion handling, and reconciliation.

The view should have an explicit source of truth and a rebuild plan. If a projection is corrupted, can it be regenerated from authoritative data or retained events? If the pipeline has lag, what does the UI show? If two updates arrive out of order, can a stale event overwrite a newer view? Version numbers, idempotent upserts, tombstones for deletes, and replay checkpoints are common parts of a robust design.

Use a materialized view when many reads reuse an expensive computation, read latency matters, or the query shape differs substantially from the write model. Avoid it when data changes constantly, the view is rarely reused, or the cost of maintaining freshness outweighs the read benefit. In an interview, explain refresh semantics, staleness, rebuild, and reconciliation—not merely that a view makes reads faster.

### Q124. How do you choose between request-time fan-out and precomputation?

Request-time fan-out composes data by calling several services when the client asks. It keeps results relatively fresh and avoids storing every combination in advance, but latency is affected by downstream calls, and the endpoint inherits their failure modes. If all ten parallel dependencies must succeed, the endpoint's availability is lower than any one dependency under an independence assumption, and the tail is often dominated by the slowest required call.

Precomputation builds a materialized view, feed, search result, or aggregate ahead of the request. Reads become faster and can be served from a purpose-built store, but writes/events must update the projection, propagation lag becomes visible, and rebuild/reconciliation is required. A hybrid often works best: precompute stable or expensive parts, then fetch a small amount of critical/fresh data at request time.

Example: a news feed can precompute candidate post IDs, then fetch current privacy/deletion state before returning the response. A dashboard can serve recent aggregates from a materialized view but label its freshness. A checkout flow should not rely on a delayed precomputed inventory view to finalize a reservation.

Estimate read rate, update rate, fan-out depth, data freshness requirement, and cost of recomputation. Precomputation becomes attractive when many reads reuse the same expensive result, but high write churn and personalized combinations can make it wasteful. Define how events are deduplicated and ordered, how deletes propagate, and how the view is rebuilt from a known source of truth. Decide what happens when the projection is stale or unavailable rather than silently serving old data.

### Q125. What is the difference between a control plane and a data plane?

The **control plane** decides or distributes how a system should behave: routes, service membership, configuration, deployment state, policies, quotas, or ownership metadata. The **data plane** handles the actual user or workload traffic using that configuration. Keeping them distinct helps prevent routine administrative operations from becoming a synchronous dependency of every request, but the boundary is logical rather than necessarily a separate product or cluster.

For example, a control plane may publish a new set of healthy service endpoints and route rules. Proxies and applications cache validated configuration and use it to route millions of requests through the data plane. If the control plane is temporarily unavailable, the data plane may continue using its last known-good configuration—but only if that configuration is safe to retain. Expired credentials, revoked permissions, or a compromised route may require stricter behavior than an ordinary routing update.

The design needs an explicit policy for stale configuration: what can continue, for how long, and what must fail closed? Avoid blocking each request on a central configuration lookup, since this can turn a control-plane incident into a global data-plane outage. Conversely, indefinite stale configuration can prevent security revocations or ownership changes from taking effect. Use versioned/validated updates, staged rollout, last-known-good behavior where appropriate, observable configuration age, and a rollback mechanism.

A useful interview distinction is that **data-plane redundancy does not automatically imply control-plane resilience**. Replicated gateways may still all be affected by a failed global rate-limit store, identity provider, certificate-rotation system, or configuration service. Identify which services are required in the hot path, and which can be cached or handled asynchronously.

### Q126. How do you choose between pushing and pulling work?

With **push**, a producer actively sends work to consumers or invokes them. Push can reduce latency and work well when consumers are ready and capacity is known, but a fast producer can overwhelm a slow consumer unless the receiving system has flow control or bounded admission. With **pull**, consumers request work when they have capacity. Pull naturally limits intake and lets workers regulate concurrency, but may add polling latency and must handle fairness and empty polls efficiently.

The best choice depends on workload shape, consumer heterogeneity, latency objectives, and broker capabilities. A queue with a visibility timeout or consumer credit can combine durable buffering with pull-based capacity control. A streaming system often has consumers fetch batches by offset, which gives the client control over processing rate and progress. A push-based HTTP callback system needs delivery retries, endpoint health handling, backoff, and a durable queue to absorb recipient downtime.

Do not confuse push delivery with real-time completion. A producer can publish an event quickly, while the consumer processes it later; acknowledgements and end-to-end status need clear semantics. For high-volume systems, batch pulls can reduce per-message overhead but increase per-message latency and complicate failure/replay. For interactive control events, push may be preferable if the consumer is capable of handling bursts.

Explain how the design behaves when consumers are slow or unavailable: producer blocking, queue growth, rejection, delayed delivery, or message expiry. Whichever pattern is selected, define who controls concurrency, how backlog is bounded, and how producers learn that the system cannot accept more work safely.

### Q127. Why are bounded queues important?

A queue absorbs a temporary difference between incoming and outgoing work; it does not create processing capacity. An unbounded in-memory queue can consume all memory, while an unbounded durable queue can create enormous storage bills and hold work past its useful deadline. As queues grow, wait time rises and requests can expire before they are processed.

Bound queues by the resource they protect: item count, bytes, per-tenant capacity, age, or a combination. A queue of 10,000 tiny events is different from 10,000 payloads of several megabytes. Choose the response when a limit is reached: block producers (backpressure), reject quickly, shed low-priority work, spill to durable storage, or expire stale items. Each choice has a user-visible contract.

A bounded queue should be paired with admission control and monitoring. Track depth, bytes, oldest age, arrival rate, successful completion rate, retry rate, and time-to-drain. If the queue fills, inspect whether the root cause is insufficient workers or a downstream dependency limit; adding consumers can worsen database saturation or provider throttling.

For a request queue, estimate the maximum acceptable wait from the end-to-end deadline and service rate, not just available memory. For a durable job queue, define retention, cancellation, replay, and dead-letter handling. In an interview, explain why the bound exists, what happens when it is reached, and how the system preserves critical work without pretending every request can be accepted during overload.

### Q128. How do you estimate whether a queue can recover from a burst?

Estimate how much backlog a burst creates, then calculate how quickly the system can drain it while new work continues. If arrivals are (lambda) events/second and successful processing capacity is (mu), net drain is (mu-lambda) only when (mu>lambda). For backlog (B), a first approximation is (B/(mu-lambda)). If processing capacity is no greater than arrivals, the queue cannot recover without reducing input or adding sustainable capacity.

For example, a 900,000-message backlog with 7,000 completions/s and 2,500 new arrivals/s drains at 4,500/s, so ideal drain time is 200 seconds. This assumes similar event cost, stable provider quotas, no partition skew, and no additional failures. If retries or poison messages consume worker capacity, use successful completion rate rather than attempted throughput.

Then validate the operational constraints. Can the database tolerate the replay write rate? Do external providers impose account-wide quotas? Does consumer parallelism depend on partition count or per-key ordering? Will catch-up traffic starve live interactive requests? A design may reserve a portion of capacity for real-time work, replay backlog at a safe rate, or expire events whose business deadline passed.

Monitor backlog age and drain rate as well as depth; a queue that is shrinking too slowly can still violate the product's completion SLO. The recovery plan should include ramp-up, retry jitter, idempotency, and reconciliation for ambiguous prior attempts. In an interview, show the arithmetic, state assumptions, and explain what you would change if the expected drain time exceeds the business deadline.

### Q129. How do you select a partition key when both ordering and throughput matter?

The partition key is not merely a distribution choice; it defines the unit for which the system can preserve locality or ordering cheaply. If all events for an order must be applied in sequence, use an order-level key so those events are routed together. If the key is too broad—such as one global key—the workload can collapse onto a single partition. If it is too narrow or random, related events may be scattered and the desired ordering lost.

Estimate both **key cardinality** and **traffic skew**. A million distinct keys does not help if one tenant or account emits most events. Use per-key and per-partition throughput measurements, not only total throughput. Also ask whether the business needs total order or just monotonic state updates; version checks can sometimes tolerate reordering without serializing every event.

If a single key is inherently hot, mitigations include caching hot reads, batching updates, aggregating partial counters, or splitting the key into buckets. Bucketing spreads load but breaks straightforward per-key ordering and may require a merge/coordination stage. A large tenant can sometimes be sub-sharded while ordinary tenants remain colocated.

Finally, plan for topology changes. Increasing partition count or changing a hash function may remap keys; older and newer events for the same entity can then be processed on different partitions during migration. Define a transition protocol or versioned routing strategy when ordering is critical. Explain which invariant forces the key choice, which queries benefit, how skew is detected, and how a hot key will be handled.

### Q130. When is a global lock a design smell?

A global lock serializes otherwise independent work, so throughput is bounded by the critical section and every participant becomes dependent on the lock's availability. It can be correct for a genuinely global invariant, but it is a warning sign when the lock exists only because the data model or workflow does not express ownership clearly.

Ask what invariant the lock protects. If two requests only conflict when they update the same account, use per-account concurrency control instead of one system-wide lock. If the database can enforce the rule through a unique constraint, conditional update, optimistic version, or a short transaction, that is often easier to reason about than a distributed lock. If the work is an aggregate, partitioning by aggregate key may preserve local ordering without global coordination.

Global coordination has a throughput and reliability cost: one slow owner can block all callers, lock-service failure can stop writes, and lease expiry can permit overlap with a stale worker unless fencing is enforced. Long-running work should not hold a lock across network calls if the invariant can be split into durable state transitions.

Do not remove a lock without proving the invariant still holds. Sometimes a single sequencer or leader is required for unique global ordering, and the cost should be explicit. In an interview, describe the critical section, expected conflict rate, the smallest ownership boundary that can protect it, and how you prevent stale owners or stuck locks.

### Q131. What is the difference between deduplication and idempotency?

**Deduplication** attempts to recognize that a message or request has already been observed, usually by storing a stable identifier and rejecting or skipping repeats. **Idempotency** is a behavioral property: repeating the same logical operation produces the same intended state as executing it once. They are related but not the same.

For example, if a consumer sees event ID 123 twice, an inbox table can deduplicate delivery. But if processing event 123 sends an email through an external provider and the consumer crashes before recording completion, replay may send the email again unless the provider supports idempotency or the application can query/reconcile the send. Likewise, a “set status to PAID” operation may be state-idempotent while a “increment balance by 100” operation is not unless guarded by operation identity.

A robust consumer records the event ID and applies its local database change in one transaction. If the external side effect cannot join that transaction, use a stable downstream operation ID and reconciliation. Define the uniqueness scope of IDs and retention period; deleting dedupe records too early can allow old events to repeat side effects.

Deduplication state can grow large, so use retention aligned with broker replay windows, business retry windows, and regulatory needs. If events can be replayed from months-old history, a 24-hour dedupe table may not protect a high-impact operation. In interviews, identify the side effect, its atomic boundary, its stable identity, and the duplicate path after every crash point.

### Q132. How should you handle a request that depends on several downstream services?

Start by separating **required dependencies** from optional enrichment. A page may require product and pricing data but tolerate missing recommendations. If every dependency is treated as mandatory, one low-value service can break the whole user journey. Define fallback semantics before designing fan-out.

For required calls, propagate an end-to-end deadline and allocate a budget to each dependency. Run independent calls concurrently when that reduces latency, but control concurrency so one incoming request does not trigger unlimited fan-out. Use bounded connection pools, request limits, and bulkheads. Retries must fit inside the remaining deadline and should not multiply at several layers. For writes, preserve a stable operation identity and treat timeout outcomes as uncertain.

Each downstream has its own latency and error distribution. The response time of parallel fan-out is approximately the slowest required call plus orchestration overhead; serial calls accumulate latency. If the endpoint requires all (n) dependencies and their failures were independent, end-to-end availability would be the product of their availabilities. Real failures are correlated, so this is an initial model, not an exact forecast.

For expensive or frequently requested combinations, consider precomputed read models instead of request-time fan-out. That reduces latency and coupling but introduces staleness and projection repair. Instrument each dependency's contribution to end-to-end latency, show which fallback was used, and test partial failures. A senior answer explains how the endpoint remains useful without hiding incorrect results.

### Q133. How do you reason about availability for a fan-out endpoint?

Identify which downstream results are required for the response. If all (n) independent dependencies must succeed, a first-order estimate is the product of their availabilities. If five required services are each 99.9% available, then (0.999^5 approx 99.50%). Real failures are correlated—shared networks, databases, credentials, or deployments can affect several services together—so the multiplication is only an initial model.

Now decide whether every dependency truly must be on the critical path. An optional recommendation service can degrade to an empty list, but a payment authorization failure must not be replaced with a fabricated success. A cached response or precomputed projection can remove a dependency from the synchronous path, but only if its freshness and correctness contract is acceptable.

Latency also composes: serial calls add their latency, while parallel calls are bounded by the slowest required call. Parallelism reduces latency but increases instantaneous concurrency and may amplify load during bursts. Set per-call timeouts from an end-to-end deadline, cap fan-out, and avoid retries that continue after the client has already given up.

Measure the end-to-end journey, per-dependency success and p99 latency, fallback frequency, and business outcome. In an interview, show the dependency graph, mark required versus optional edges, estimate the availability/latency impact, and explain graceful degradation without weakening the core invariant.

### Q134. How do you choose a cache consistency model?

Choose consistency from the business operation, not from the cache product. Ask how stale the value may be, whether a read after a write must see the new value, whether revocation must be immediate, and whether incorrect decisions are reversible. “TTL of 30 seconds” is a staleness budget only under assumptions about invalidation, refill races, and cache availability; it is not a full consistency guarantee.

For public content, eventual consistency with TTL and asynchronous invalidation may be sufficient. For session settings, read-your-writes may be needed so the user's own update appears immediately. For permissions, a stale authorization cache can allow access after revocation, so use a clear revocation strategy, short-lived credentials, version checks, or an authoritative policy check for high-risk operations. For inventory or balances, a cached value can support display but should not authorize the final mutation.

Define the source of truth, invalidation/refresh mechanism, maximum tolerated stale age, and outage behavior. Versioned cache entries can prevent an older refill from overwriting a newer value; outbox/CDC invalidations can make change notification durable; single-flight can limit duplicate refills. These techniques reduce specific failure modes but introduce their own complexity.

If the consistency requirement cannot be guaranteed with the cache, bypass it for that decision. Measure stale-read incidents and invalidation lag as product correctness signals, not only hit ratio. In an interview, give a concrete example and explain what the user observes during a write, concurrent read, delayed invalidation, and cache failure.

### Q135. What should a cache key include?

A cache key must include every input dimension that can change the cached result, and exclude dimensions that do not. For a public product detail page, the key may be product ID plus locale and representation version. For a personalized response, it might also need tenant, user/permission scope, currency, feature-flag cohort, or query filters. Omitting one of these can cause incorrect responses or a cross-tenant data leak.

Treat key design as part of correctness and security. Do not trust arbitrary client-supplied strings as a complete identity; derive canonical values from validated input and trusted authentication context. Normalize case, encoding, default parameters, and query ordering where semantic equivalence requires it. Hash long or sensitive key material when appropriate, while keeping debugging and collision behavior understood.

Include versioning where schema or serialization changes can make old cached values incompatible. When invalidating a resource, know which key variants are affected; broad prefix invalidation may be expensive or risky. Tenant- or user-specific keys can also create high cardinality and memory pressure, so account for eviction and TTL policy.

A safe response cache must honor HTTP cache-control semantics, vary dimensions, authorization, cookies, and private/public boundaries. Never let a shared CDN or distributed cache store a personalized payload under a key that all users share. Test by changing one dimension at a time—tenant, user, locale, permission, feature flag, or filter—and verifying that data does not cross boundaries.

### Q136. How should a design respond to a cold cache after deployment or failover?

A cold cache removes the origin offload that normal traffic may rely on. If a cache restart or rollout causes millions of misses at once, the origin database can saturate even though request volume has not changed. Capacity planning should include this cold-cache condition, not just steady-state hit ratio.

Use controlled warm-up for predictable hot keys, staggered deployment, jittered TTLs, request coalescing, and origin-side concurrency limits. Prewarming the entire keyspace may be wasteful or impossible; prioritize keys by recent traffic and business value. Consider a local cache layer only if its invalidation and memory behavior are understood. Serve stale data while refreshing only for data whose contract permits staleness.

Keep application scaling from amplifying the miss storm: every new instance should not independently preload the same large dataset. Bound database connection pools and queries per key. If the cache is unavailable, explicitly degrade, throttle, or reject low-priority requests rather than opening the origin to unlimited concurrency.

Watch cache hit ratio, origin QPS, database saturation, fill concurrency, eviction rate, and p99 during the ramp. Recovery should be gradual, with stop/rollback thresholds if the origin approaches its safe operating limit. In an interview, explain what changes under a cache outage and why the origin remains protected until the cache is warm again.

### Q137. What is the difference between a replica, a backup, and a read model?

A **replica** is a live copy of a datastore maintained for availability, read scaling, or recovery. Depending on replication mode, it may lag behind the primary and may or may not contain every acknowledged write when promoted. A **backup** is a retained recovery point intended to restore data after loss, corruption, accidental deletion, or malicious change. A **read model** is a projection optimized for specific query patterns, often derived from events or CDC and potentially delayed.

These are not interchangeable. A replica may reproduce a bad delete or corruption quickly, so it is not an independent backup. A backup is usually not suitable for serving current reads because it is older and restoring it may take time. A read model may omit details or denormalize state by design, so it should not be treated as the source of truth for operations it does not own.

Each has a separate recovery contract. Replicas need lag, failover, and split-brain management. Backups need retention, access separation, immutability where appropriate, and restore testing against RPO/RTO. Read models need versioned schemas, replay/rebuild procedures, delete/tombstone handling, and monitoring for projection lag and mismatches.

For example, an order database can be the authority, its replica can serve eligible reads, a backup can recover from accidental data corruption, and a search index can support text queries. If the search index is lost, rebuild it from the authority or retained events. In an interview, state what each copy is for and which guarantee it does—and does not—provide.

### Q138. How do you choose a consistency boundary for a business workflow?

A consistency boundary is the set of state changes that must agree atomically to preserve a business invariant. Start by writing that invariant in business language. “An order cannot be confirmed unless inventory is reserved” may require order and inventory state to coordinate, while “the analytics dashboard reflects the order within one minute” can tolerate asynchronous propagation.

If the invariant fits within one database, a local transaction, constraint, or conditional update is usually the simplest and strongest boundary. Splitting that transaction across services makes partial progress visible and requires explicit coordination. A saga can manage the lifecycle, but it does not provide atomic visibility: one service may have reserved inventory while payment is still pending.

Some operations naturally have two outcomes: a durable command is accepted now, while completion occurs later. Expose a pending state and operation ID instead of pretending that asynchronous completion is synchronous. Define timeout and cancellation behavior, including whether external effects can still complete after the caller cancels.

Put authority where the invariant can be enforced. A cache or search index may present approximate availability, but the inventory owner must atomically reserve the unit. For financial workflows, use ledger/state transitions, idempotency, reconciliation, and explicit unknown states. In an interview, identify the invariant, the smallest transaction boundary that protects it, and the cross-boundary protocol for progress, failure, compensation, and repair.

### Q139. What is the dual-write problem, and what patterns solve it?

The dual-write problem occurs when one logical business operation must update two systems independently—for example, commit an order in a database and publish an event to a broker. The writes can diverge because there is no shared atomic commit. Database-first can lose the event; broker-first can publish an event for a transaction that rolls back.

The most common solution for a database plus broker is the transactional outbox: persist the business change and an outbox record in one database transaction, then publish the outbox asynchronously. Change-data-capture can stream committed changes from the database log or the outbox, depending on the design. Consumers must be idempotent because publication may repeat after a crash.

For two independent databases or an external provider, an outbox alone does not make the entire operation atomic. You may need a saga with explicit state transitions and compensations, an inbox at the consumer, an idempotency key accepted by the provider, or reconciliation of uncertain outcomes. The right pattern depends on which system is authoritative, whether the side effect is reversible, and what consistency the product promises.

Avoid naive application-level “write A, then write B” with a catch block: a process can crash between the writes, and a retry can repeat one side. Also avoid pretending that a distributed transaction is available unless every participant and operational constraint truly supports it. Describe the exact failure window the pattern closes, what duplicates remain possible, and how the system detects and repairs inconsistent state.

### Q140. When is event sourcing justified?

Event sourcing stores the sequence of domain events that represent state changes, then derives current state by replaying or projecting those events. It can provide a detailed audit history, support temporal questions, and allow new read projections to be built from past events. It is not simply publishing events after writing current state; the event log is the authoritative record of changes.

The approach is valuable when the domain genuinely needs immutable history—such as financial ledgers or complex state transitions—and the team can define stable event semantics, versioning, replay, and projection operations. It has significant costs: event schemas become long-lived contracts, replay can be slow, projections can lag, historical bugs may reproduce during replay, and changing the meaning of an event is much harder than changing a table column.

Snapshots can reduce replay time but must be consistent with the event position. Projections should be rebuildable and idempotent. Corrections are usually new compensating events rather than destructive edits to history, but sensitive-data deletion and retention requirements need deliberate design; immutable history does not override privacy or legal obligations. Protect event logs and backups as carefully as source-of-truth data.

Do not adopt event sourcing merely because the system uses Kafka. Many applications work better with a conventional transactional database plus outbox/domain events for downstream integration. Choose it when immutable business history and replay materially justify the complexity, and explain how the system handles schema evolution, duplicate delivery, failed projection rebuilds, and reconciliation.

### Q141. How should API rate limits be distributed across multiple application instances?

First define the scope of the limit. A quota might apply per IP address, user, API key, tenant, endpoint, provider account, or the whole platform. If the limit is 1,000 requests per minute per tenant, a local counter on each of ten application instances can unintentionally allow about 10,000 requests per minute when traffic is spread across all ten. The same limit must be enforced at the scope promised by the contract.

There are three common designs:

- **Local per-instance limiter:** low latency and works even when a shared store is unavailable, but the aggregate limit is approximate and depends on routing. It is suitable for a coarse safety guard or when small quota overshoot is acceptable.
- **Shared/distributed limiter:** instances coordinate through a shared store or dedicated rate-limit service. It can enforce a more consistent global budget but adds a network hop, contention, and a dependency that can itself become saturated. Atomicity, clock behavior, failover, and key cardinality matter.
- **Hierarchical token allocation:** a global controller allocates bounded token budgets or leases to local instances. Requests are decided locally until the allocation is exhausted, reducing hot-path coordination. The trade-off is temporary unused capacity and possible bounded overshoot or uneven fairness while budgets are redistributed.

Choose the semantics before choosing the algorithm: strict contractual quota, abuse protection, and load shedding are different use cases. A fixed window is simple but can permit a burst at the boundary; a sliding window smooths the accounting but costs more state; a token bucket explicitly permits a configured burst while bounding the long-term rate. Also distinguish rate from concurrency: 100 requests per second can be harmless for 5 ms calls but dangerous if each call occupies a worker for 30 seconds.

Define fail-open versus fail-closed behavior. If a billing quota must not be exceeded, failing open may violate a contract; if the limiter is an approximate abuse guard and rejecting every legitimate request would cause a major outage, a bounded local fallback may be better. Monitor allowed/rejected counts, latency, store errors, key cardinality, and quota overshoot, and test behavior as the instance count changes.

### Q142. How do token bucket and leaky bucket differ conceptually?

A **token bucket** accumulates tokens at a configured refill rate up to a capacity. Each admitted request consumes tokens, so the bucket permits a defined burst while constraining long-term average traffic. If the bucket holds 100 tokens and refills at 20 tokens/second, an empty bucket recovers at that rate; a full bucket can admit a burst of up to 100 requests immediately if each costs one token.

A **leaky-bucket shaper** smooths accepted work by releasing it at a controlled rate, often by queueing bursts and draining them steadily. Implementations using a leaky-bucket idea may instead reject overflow; state whether the algorithm shapes or polices traffic because the behavior and latency differ.

Use token bucket when bursts are allowed within a clear budget, such as an API quota. Use shaping when downstream capacity benefits from a smoother stream and the operation can wait. Neither alone protects against high concurrency from slow requests, so combine rate limits with concurrency limits and bounded queues. A request that occupies a worker for 30 seconds consumes very different resources from a 5 ms request at the same rate.

For distributed limits, define whether the budget is local, tenant-wide, account-wide, or global. Local token buckets can overshoot an aggregate limit as instances scale; shared state improves coordination but adds a bottleneck. In interviews, specify burst capacity, refill rate, quota scope, overflow behavior, and fail-open/closed policy rather than naming the algorithm alone.

### Q143. What is admission control, and how does it differ from rate limiting?

Rate limiting controls how much traffic is permitted over a defined interval or token budget. **Admission control** decides whether the system can safely accept additional work right now based on available capacity, concurrency, queue length, deadline, or resource saturation. A request may be under its rate quota but still be rejected because all database connections are busy or the remaining deadline is too short.

Rate limits protect fairness and external contracts—for example, 500 requests/minute per tenant. Admission control protects system stability—for example, allow only 200 concurrent expensive report jobs or reject an interactive request if its queue wait would exceed the latency SLO. These controls complement one another.

Use queue bounds, per-route concurrency budgets, workload priorities, deadline-aware admission, and explicit overload responses. Do not simply queue every request: an unbounded backlog consumes memory/storage and lets requests expire before useful work begins. For accepted asynchronous work, persist the command durably and return a pending operation ID; for rejected work, clearly state that it was not accepted.

The admission decision should use signals that respond to actual saturation and should not itself become a bottleneck. Define fail-open/closed behavior if a shared limiter or capacity signal is unavailable. In an interview, contrast allowed rate with available capacity, explain which resource is protected, and say how callers should respond to rejection.

### Q144. How should you design for a downstream provider's quota?

First determine exactly how the provider counts quota: per account, credential, endpoint, region, operation type, concurrency, or time window. A per-instance limiter is not sufficient when the provider enforces one account-wide budget; autoscaling from five to twenty instances can multiply outbound attempts without increasing the allowed quota.

Coordinate work at the provider's quota scope using a shared limiter, durable scheduler, or hierarchical token allocations. Reserve capacity for critical operations, coalesce duplicate calls when safe, and queue deferrable work. Count retries against the same budget, respect Retry-After signals, and use bounded exponential backoff with jitter. For different operation costs, allocate budget by cost rather than treating every request as equal.

Plan for provider outage and recovery. A backlog replayed at full speed may exceed quota again and starve live traffic. Use priority classes, rate-controlled catch-up, idempotency, and reconciliation for uncertain outcomes. A secondary provider may have different semantics or limits, so failover must be evaluated rather than assumed transparent.

Monitor attempt rate, throttling responses, provider latency, queue age, success ratio, and quota utilization across regions. Define what happens when the budget is exhausted—defer, reject, return pending, or use an approved fallback. In an interview, explain quota ownership, fleet-wide coordination, and why retries and failover must share the same budget.

### Q145. What makes a multi-tenant architecture fair?

Fairness means one tenant should not be able to consume so much shared capacity that other tenants violate their SLOs. Apply budgets to the resources that matter: request rate, concurrent operations, database/query cost, queue occupancy, storage, egress, and expensive background jobs. A single requests-per-second limit is insufficient because requests can have very different cost.

Use per-tenant quotas and concurrency limits, bounded tenant-aware queues, weighted fair scheduling, and separate pools for high-cost work. For highly skewed workloads, dedicated shards or infrastructure may be justified. But isolation has a cost: dedicated capacity can sit idle, and hard partitioning may make cross-tenant aggregate queries more difficult. A hybrid model can share capacity for ordinary tenants and isolate consistently large or regulated tenants.

Fairness is not necessarily equal distribution. Contracts may define service tiers, burst allowances, priority for critical workloads, or different retention/throughput budgets. Protect the system from clients spoofing tenant or priority context; derive identity from trusted authentication and enforce isolation at the resource-owning service.

Measure per-tenant latency, rejection, queue age, and resource use. Avoid unbounded high-cardinality metric labels by aggregating or retaining top-N views. In an interview, state the fairness policy, the resources it protects, how priority is authenticated, and how you avoid starvation during overload.

### Q146. How should service-to-service authentication influence the architecture?

Service-to-service authentication determines how one workload proves its identity to another; it should be part of the system's trust boundaries, not an assumption that private network traffic is safe. Common approaches include mutual TLS with workload certificates, short-lived signed credentials, or platform-issued workload identity. The receiving service should authenticate the calling workload and then authorize the requested operation.

Keep workload identity separate from end-user identity. An order service calling a payment service may present its own workload credential and also carry a narrowly scoped user or order authorization context. Blindly forwarding a broad user token or trusting a caller-supplied identity header can create a confused-deputy problem. Downstream services must validate token issuer, audience, expiry, signature/key rotation, and relevant claims.

The architecture must define identity issuance, rotation, revocation, and what happens when the identity control plane is unavailable. Local validation of a short-lived signed token can keep the hot path independent of a remote identity check, but revocation is less immediate; introspection can provide fresher policy but adds latency and a dependency. mTLS encrypts and authenticates the channel, but does not itself grant access to every resource.

Use least privilege for service accounts, limit credentials to an audience/purpose, audit privileged calls, and avoid logging secrets. In an interview, trace how a caller's identity is established, propagated, and checked at the resource-owning service, including what happens after a credential leak or during key rotation.

### Q147. How do you design observability for asynchronous workflows?

An asynchronous request may return before the business operation completes, so HTTP latency and status code are not enough to measure success. Assign a stable operation/workflow ID at acceptance, propagate it through outbox records, messages, retries, consumer logs, and external calls, and record durable state transitions with timestamps. Trace context can cross messaging boundaries, but tracing alone is not a substitute for a durable business history.

Measure acceptance rate, end-to-end completion rate, completion-time percentiles, oldest pending age, queue depth, arrival/completion rate, retry count, dead-letter count, and the number of workflows stuck in each state. Break down by tenant, priority, provider, and workflow version where safe. A queue can have low depth but contain one very old critical operation; age and business deadline are often more informative than depth alone.

Separate technical delivery from business completion. “Broker acknowledged message” means the broker accepted it; “consumer acknowledged” means the consumer declared the delivery handled under its protocol; neither necessarily means an email arrived or a payment settled. Use domain states and provider receipts/reconciliation to measure the actual outcome. Alert on stalled state transitions and SLO burn, not just broker health.

Telemetry itself can be duplicated or delayed. Keep correlation identifiers stable through retries, avoid high-cardinality metrics for every workflow ID, and protect sensitive payloads. In an interview, explain how on-call staff locate one operation across queue wait, consumer processing, database commit, and external provider, and how dashboards reveal silent loss or stuck workflows.

### Q148. Why can infrastructure health checks miss business failures?

Infrastructure health checks usually measure whether a process is alive, whether it can accept traffic, or whether a dependency responds. They do not prove that the business operation is correct. An API may return HTTP 200 while reading stale data, skipping a required authorization check, failing to publish an event, or marking an order complete before payment is confirmed.

Use health probes for lifecycle and routing decisions, and business-level SLIs/synthetic checks for user outcomes. A checkout synthetic can verify that a test order reaches the expected state; an async workflow monitor can verify that accepted jobs complete within their target and that no operations remain UNKNOWN too long. Run checks safely against test identities and avoid generating real external charges or notifications.

Avoid making every liveness probe call every dependency. A shared dependency outage can mark every instance unhealthy and trigger a restart storm without improving the underlying service. Readiness should indicate whether the instance can safely serve its intended traffic; diagnostics can report partial dependency degradation separately.

Combine metrics, traces, and durable state-transition records. Monitor success by journey, error-budget burn, stale-read rate, duplicate/conflicting transitions, queue age, and reconciliation mismatches. In an interview, explain which failures process health can detect, which require business-level validation, and how probes avoid amplifying an outage.

### Q149. How do you set an RTO and RPO from business requirements?

Derive recovery objectives from the business impact of an outage and data loss, rather than choosing numbers because a platform supports them. **RTO** asks how long a critical user journey can remain unavailable before the harm becomes unacceptable. **RPO** asks how much recently accepted data the business can afford to lose, expressed as a time window. Different workflows may need different targets: catalogue browsing may tolerate more interruption than payment authorization or inventory allocation.

Work with the product/business owner to quantify impact over time, regulatory obligations, customer commitments, and dependencies. Then build a recovery timeline that includes detection, decision/escalation, fencing the old writer, promotion/restore, routing convergence, cache and read-model rebuilding, client reconnection, and validation. The database's RTO is only one piece of end-to-end recovery.

RPO drives replication and durability choices. If zero acknowledged-write loss is required, asynchronous replication alone may not meet the requirement under primary loss; the actual synchronous/quorum commit guarantee and failure model must be verified. Backups and point-in-time recovery protect against corruption but have their own restore time. RTO drives automation, standby capacity, recovery procedures, and how often the team tests them.

Write the target in business terms and measure it in exercises. Restores should verify data integrity and application behavior, not just that database startup succeeded. Include dependencies such as DNS, identity/KMS, deployment artifacts, network access, and operator credentials. If the measured restore takes six hours but the stated RTO is one hour, the architecture does not meet its requirement regardless of dashboard status.

### Q150. Why can active-active compute still use a single write leader?

Compute can be active in several regions while writes for a given entity are routed to one authoritative region, shard, or leader. This keeps application instances close to users for reads and stateless processing while retaining a single serialization point for updates that must not conflict. “Active-active deployment” therefore does not necessarily mean “both regions can independently commit every write.”

For example, users in Europe and Asia may read a replicated catalogue locally, but a payment account or inventory item has a home region that owns its writes. If another region receives a write, it forwards the command to the owner or rejects it during loss of ownership. This can add cross-region latency and make writes unavailable during a partition, but avoids concurrent conflicting debits or overselling.

The design needs a routing/ownership map, safe leader transfer, fencing tokens or an equivalent protocol, replication monitoring, and behavior during region loss. If the owner is unreachable, the system must decide whether the invariant permits queuing the command, redirecting it after safe promotion, or rejecting it. DNS or nearest-region routing cannot make that correctness decision.

True multi-writer operation is possible for some data models, especially when independent updates can merge, but conflict resolution is not universally safe. A globally unique name, final inventory unit, or bank balance requires coordination or a deliberate availability trade-off. In an interview, distinguish stateless compute redundancy, replicated reads, single-writer per entity, and true active-active writes.

### Q151. What is a safe failover sequence for a stateful service?

A safe failover must restore service without allowing two writers or silently losing data that the system promised to preserve. The sequence depends on the database/consensus protocol, but the decision points should be explicit:

1. **Detect and validate the failure.** Distinguish an instance-local failure from a regional partition or shared-dependency outage. Avoid promoting because one probe failed.
2. **Establish exclusive ownership.** Fence the old leader or obtain proof from the system's election/quorum protocol that it can no longer commit. An unreachable old process may resume later, so timeout alone is not proof.
3. **Choose a promotable candidate.** Check the candidate's log/replication position and the configured durability guarantee. A lagging replica may violate the RPO if promoted.
4. **Promote once and update routing.** Use an epoch/term/lease that stale clients and writers cannot reuse. Refresh connection pools and discovery data without creating a thundering herd.
5. **Validate the system, then restore traffic gradually.** Check data invariants, error rate, latency, replication, queue ownership, and dependent workflows. Keep a rollback/failback plan that does not re-enable the old writer unsafely.

An acknowledged write may be absent on the candidate if the original commit policy permitted asynchronous loss. The application needs idempotency/status handling for in-flight requests whose response was lost. After the failover, compare affected operations and reconcile derived stores. In an interview, describe the exact point at which the old leader loses authority, what evidence justifies promotion, and how the data-loss risk matches the RPO.

### Q152. How do you prevent a failover loop?

A failover loop occurs when the system repeatedly switches between regions or leaders because health signals flap, dependencies are shared, or the new target is not truly capable of serving traffic. Each switch can interrupt connections, invalidate caches, amplify retries, and create additional data or ownership risks.

Use layered health signals with different scopes: process readiness, application success, dependency health, and synthetic user-journey checks. Require sustained failure or a threshold across a window before triggering automatic failover, and add hysteresis/cooldown so transient recovery does not immediately trigger failback. Avoid treating every downstream error as proof a region is unhealthy; an identity provider or database shared by both regions may be the common root cause.

Separate failover from failback. Failover selects a safe new authority; failback should happen only after the original region's data is current, write ownership is transferred safely, and the team has validated that the underlying problem is fixed. Gradually shift traffic and keep a rollback path. For stateful systems, health checks alone cannot fence a leader.

Record why the controller made each decision, including signal values, thresholds, current leader epoch, and action. Test flapping networks, partial regional connectivity, and a region that can serve reads but not writes. In an interview, explain which signals trigger the transition, what state prevents repeated oscillation, and how you ensure that routing changes do not create split brain.

### Q153. What is the role of a canary in reducing release risk?

A canary exposes a small, controlled cohort of traffic to a new version before expanding rollout. It limits blast radius while comparing latency, error rate, saturation, and business outcomes against the stable version. The cohort must be representative: routing only 5% of traffic but excluding the largest tenants or most expensive request types can hide the actual regression.

Define promotion and abort criteria before deployment. Use a bake period long enough for the relevant traffic patterns and asynchronous work to appear. For low-volume services, use synthetic transactions and targeted invariant checks because too few real requests may provide weak statistical evidence. Watch business metrics as well as process health; a deployment may return 200s while creating duplicate jobs or stale projections.

Canaries require compatibility between old and new versions. Database and event schemas should support mixed versions, and queues may deliver old messages to new consumers. A rollback cannot automatically reverse external side effects or incompatible data writes. Design the expand–migrate–contract path and explicitly state the point after which forward repair is safer than rollback.

Canary traffic controls may be weighted at a gateway, service mesh, or deployment platform, but the routing mechanism itself needs failure handling. In an interview, explain how the cohort is selected, what signals control expansion, how you handle low volume, and which data changes constrain rollback.

### Q154. How should a system support backward-compatible event evolution?

Events are contracts between producers and consumers that may be deployed at different times and may process retained messages long after they were published. A schema change is therefore not just a code change: old consumers may receive new events, and new consumers may replay old event versions.

Prefer additive changes with clear semantics. Adding an optional field with a safe default is often compatible, while removing a field, changing its type, renaming it, or reusing it with a different meaning can break consumers even if a serializer still accepts the payload. Consumers should define how they handle unknown fields and enum values based on the format and business requirements. A schema registry and compatibility checks can catch structural incompatibilities, but cannot prove that the new field's meaning is correct.

Use explicit event versions only when needed and define the conversion/compatibility strategy. During rollout, deploy consumers that can handle both old and new forms before producers emit the new form. Keep historical replay and DLQ payloads in mind; a consumer may encounter events from months ago. Avoid embedding transient internal database models directly into public domain events.

Test compatibility with representative old/new producer and consumer versions, including replay. Monitor deserialization errors, unknown values, and version distribution. If a breaking change is unavoidable, publish a migration and retirement plan with a clear support window. In an interview, explain mixed-version operation and rollback, not just “add a version field.”

### Q155. How do you reduce cloud cost without creating reliability debt?

Start with actual cost drivers and workload shape: idle compute, database/storage, I/O, network egress, cross-region replication, logging/tracing, cache infrastructure, and provider API charges. Optimize cost per successful business operation under the same SLO, not cost in isolation. A lower bill caused by failing requests or reducing retention below the recovery requirement is not a real improvement.

Low-risk opportunities include rightsizing, removing unused resources, scheduling development environments, compressing/batching traffic, archiving cold data, optimizing expensive queries, and sampling high-volume telemetry intelligently. Each has trade-offs: scale-to-zero may create cold starts; compression consumes CPU; shorter log retention can impair investigations; removing replicas reduces failure headroom.

Do not remove redundancy or backup retention without recalculating failure capacity and RTO/RPO. A service sized only for average load can become expensive during an outage because retries and emergency scaling multiply costs. Measure steady-state and peak/failure scenarios. Consider the engineering/on-call cost of a service too, not only its instance charge.

Treat savings as a controlled experiment: establish a baseline, define latency/error/correctness guardrails, roll out gradually, and keep a rollback trigger. In interviews, quantify the expected savings, name the reliability risk being accepted, and explain how you verify the savings persist without worsening the user journey.

### Q156. How do you evaluate a proposed new technology?

Start with the problem statement and measurable acceptance criteria. A technology should solve a named limitation—such as a demonstrated query pattern, throughput ceiling, latency requirement, durability guarantee, or operational bottleneck. “It is popular,” “it is modern,” or “it scales” is not evidence that it fits this system.

Compare alternatives against the workload: data model, consistency, latency percentiles, sustained throughput, failure behavior, retention, security, portability, and cost. Include operational maturity: on-call skills, dashboards, backup/restore, upgrades, incident behavior, capacity testing, vendor support, and migration/exit costs. A faster benchmark is relevant only if the benchmark matches the real payload, concurrency, key distribution, failure mode, and durability settings.

Prototype the riskiest assumption rather than building a full proof of concept for everything. For example, benchmark a hot-key access pattern, test restore from a realistic data volume, or verify how the product behaves when a replica is promoted. Include failure injection and operational tasks, not only happy-path performance.

Record the decision, constraints, rejected alternatives, known risks, and evidence that would trigger revisiting it. Design for reversibility where possible: adapters, portable data formats, gradual traffic rollout, and an export plan. In an interview, show that you can choose a reasonable technology and also explain when you would *not* choose it.

### Q157. What belongs in an architecture decision record?

An architecture decision record (ADR) captures the context and reasoning behind a significant design choice so future engineers can understand why it was made and when it should be reconsidered. Include the problem, constraints, requirements, options considered, decision, rationale, consequences, risks, and evidence such as benchmarks or failure tests. Keep it concise enough to maintain but specific enough to be useful.

For example, an ADR might choose a relational primary plus an asynchronous search index because checkout needs transactional correctness while catalog search needs relevance ranking. The record should explain the expected indexing lag, rebuild approach, data owner, failure behavior, and why querying only the primary was rejected—not merely say “use Elasticsearch.”

Record assumptions explicitly: traffic range, data volume, consistency target, team skills, cost limits, and operational constraints. State what would trigger revisiting the decision, such as sustained write throughput exceeding a tested limit, query latency missing its SLO, or regulatory requirements changing. An ADR is not a permanent contract that prevents new evidence from changing the architecture.

Link the ADR from the design and operations documentation, assign an owner, and update the status if a decision is superseded. In an interview, explain that good architecture makes decisions and trade-offs visible; it does not pretend the chosen option is universally optimal.

### Q158. How do you explain the limits of your own design in an interview?

State the assumptions that make the design reasonable, then name the bottleneck or failure mode most likely to invalidate them. For example: “This design routes writes for each account to one shard, which gives us a clear consistency boundary. A single hot account can still become a bottleneck, and splitting that account would complicate ordered updates and reconciliation.” This demonstrates that you understand both the benefit and the cost.

Separate hard guarantees from best-effort behavior. If a read model is asynchronous, state its freshness target and what happens when lag exceeds it. If a region fails, explain the RPO and whether writes pause while a new authority is established. If a provider response times out after a possible side effect, do not promise a definitive failure until the outcome is resolved.

Offer an evolution path tied to evidence: query/index tuning before sharding, replicas for eligible reads, capacity isolation for noisy tenants, and migration when a measured limit is reached. Mention the operational cost—monitoring, on-call, backups, rollout and recovery—not just the future architecture diagram.

Do not recite a generic “trade-offs” list. Connect each limitation to the requirement it compromises, how it will be measured, and what condition would trigger a change. In interviews, acknowledging a real limit and designing a safe response is stronger than claiming the architecture scales infinitely or has no single point of failure.

### Q159. How do you decide whether an index belongs in the database or a search engine?

Use the primary database's indexes for transactional access patterns such as equality/range filters, joins, uniqueness, and ordered retrieval that fit its query engine. A database index participates in the database's transaction and constraint model, but every index adds storage and write-maintenance cost. Inspect query plans, selectivity, rows scanned, sort cost, and read/write ratio before adding one.

A search engine is useful when the workload requires full-text analysis, relevance ranking, faceting, typo tolerance, autocomplete, or specialized document retrieval at scale. It is usually a derived read model fed from events or CDC, so changes may lag and updates/deletes need robust propagation. A search result can help users discover an item but should not automatically authorize a transaction-critical decision such as final inventory reservation or payment.

The architecture must define which system owns the canonical data, how changes reach the search index, how schema/analyzer changes are deployed, and how the index can be rebuilt. A rebuild should not lose live updates: take a consistent source boundary, bulk-load, capture concurrent changes, catch up, validate, and switch aliases/routing safely. Deletion tombstones may be needed so old replayed updates do not resurrect removed documents.

Choose based on query semantics and operational cost, not raw benchmark speed. If a relational index meets the latency and throughput target, a separate search cluster adds failure modes and eventual consistency without enough benefit. If relevance search is core to the product, forcing it into SQL can create slow or brittle queries. State the freshness contract users will observe.

### Q160. What is CQRS, and when is it worth the complexity?

Command Query Responsibility Segregation (CQRS) separates the model used to change state from the model used to answer reads. A command expresses an intent such as “reserve this seat” and is validated against authoritative invariants; a query reads a model optimized for a particular view. CQRS does not inherently require separate services, databases, or event sourcing, though those patterns are sometimes combined with it.

It is valuable when reads and writes have materially different needs: for example, an order write model needs transactional state transitions, while a customer-facing order history needs denormalized status, product names, and shipment information. A dedicated read model can reduce joins and allow independent scaling. It also creates asynchronous propagation, stale reads, projection rebuilds, delete handling, schema evolution, and more complex debugging.

Define the consistency contract for the read model. If a user confirms an order and immediately opens its status, read-your-writes may be required; route to the authoritative source or carry a version/consistency token if the storage system supports it. Do not allow a stale read model to make a correctness-critical mutation.

Use CQRS only when a clear workload/domain problem justifies the extra model and synchronization. For simple CRUD with similar read/write shapes, one well-designed relational model is often better. In an interview, describe the command transaction, event/projection path, freshness target, rebuild procedure, and fallback when the projection is behind or unavailable.

### Q161. What is change data capture (CDC), and how is it different from application events?

Change data capture (CDC) observes committed changes from a database's transaction log or change mechanism and makes them available to downstream systems. It is useful for synchronizing a search index or analytical store, migrating data, and integrating a legacy application that cannot reliably publish business events itself. CDC can capture inserts, updates, and deletes with source ordering/positions depending on the connector and database.

Application domain events communicate business meaning, such as `OrderConfirmed` or `PaymentCaptured`. CDC may instead report a row changed from status A to status B, including storage-oriented fields that are not a stable business contract. A row update is not automatically a meaningful domain event, and downstream consumers should not infer business intent from arbitrary schema mutations unless that contract is explicit.

CDC pipelines need checkpoint/offset management, replication-log retention, schema evolution, delete/tombstone behavior, idempotent consumers, backpressure, and monitoring. If the connector falls behind longer than the source retains its log, it may need a snapshot/rebootstrap. A CDC stream also does not magically solve the dual-write problem if the application wrote the wrong state or if an external side effect was not committed in the source database.

Choose CDC for faithful data movement and change propagation; choose application events when consumers need a domain-level statement of what happened. Some designs use both: a transactional outbox emits domain events, while CDC mirrors operational tables to analytics. Document event semantics, ordering scope, recovery and replay, privacy filtering, and how derived systems are reconciled.

### Q162. How do you handle deletes in a replicated read model?

Deletes are tricky because removing a record from a source can erase the information a downstream consumer needs to know that the record once existed. If an older update event is replayed after the delete, it can recreate the record unless the consumer has a tombstone or version that proves the delete is newer.

Use an explicit deletion event or tombstone with a stable entity ID and version/sequence. The projection should apply the delete only if it is not older than the current record, and should reject older updates after the tombstone. If the source uses CDC, understand how deletes and tombstones are represented and how long the connector/broker retains them. A soft-delete flag may simplify propagation, but it does not automatically meet privacy or physical-erasure requirements.

Design retention deliberately. If tombstones are removed before any delayed consumer or replay window has passed, an old event can resurrect the record. If legal deletion requires removing personal information, use an architecture that can erase or cryptographically render inaccessible the required data while preserving permissible audit records; “event sourcing” or “keep tombstones forever” is not a universal solution.

Test out-of-order update/delete events, full index rebuild, consumer reset, backup restore, and tenant migration. Track delete lag and resurrection incidents. In an interview, identify which system owns deletion, how it propagates to caches/search/analytics, what prevents stale resurrection, and how the design satisfies retention and privacy rules.

### Q163. How do you choose between soft deletes and hard deletes?

A **soft delete** marks a record as deleted or inactive while retaining its data. It can support recovery, audit history, referential integrity, and downstream propagation, but every query must apply the correct filter and the retained data still has storage/security implications. A forgotten filter can expose data that users believe has been deleted.

A **hard delete** physically removes the row or object from the primary store. It can simplify some queries and help meet deletion requirements, but cascades, foreign-key behavior, caches, search indexes, analytics copies, backups, and event history still need deliberate handling. Removing the source row does not instantly remove every replica or backup copy.

The choice depends on legal retention, privacy, auditability, business recovery, and data relationships. A common lifecycle is soft delete or tombstone for immediate application semantics, followed by an asynchronous purge after a defined retention period. For privacy requirements, use a documented deletion workflow across downstream projections and backups; in some architectures, cryptographic erasure may be relevant, but it must satisfy the actual legal and product requirements.

Use versioned tombstones so delayed old events do not resurrect deleted records. Define which records remain for audit and which sensitive fields must be erased or rendered inaccessible. In an interview, explain the user-visible meaning of deletion, propagation/retention, and how you prevent deleted data from reappearing through replay or restore.

### Q164. What is a data retention policy, and how does it affect architecture?

A retention policy defines how long each data class must remain available and what happens when it expires. Different data may need different rules: transaction records, operational logs, temporary uploads, event streams, analytics aggregates, backups, and customer profiles should not automatically share one retention period. Requirements may come from product behavior, cost, audit obligations, privacy, and jurisdiction-specific rules.

Retention influences partitioning and storage. Time-partitioned tables can make it easier to detach/archive old ranges than to delete millions of rows individually. Event brokers need enough retention for the longest credible consumer outage, repair, and replay window; otherwise a lagging consumer may need a full rebuild from another source. Object storage may use lifecycle transitions to move cold data to cheaper tiers, but retrieval latency and cost change.

Deletion must propagate to caches, search indexes, read models, analytics, exports, and backup recovery workflows. Tombstones may need to remain long enough to prevent stale events from resurrecting data. Conversely, retaining tombstones or immutable event payloads forever can conflict with privacy requirements. Define the treatment of legal holds, restored backups, and derived datasets.

Calculate the storage/cost impact, assign owners, and automate expiry with monitoring and audit. Test that expiration jobs are idempotent and cannot delete newer data accidentally. In an interview, tie retention to data model, partition strategy, replay/restore capability, and the actual business or compliance requirement.

### Q165. How do you handle online backfills without overwriting newer data?

An online backfill transforms existing records while production writes continue. The main race is that the backfill reads an old value, a live request writes a newer value, and the backfill later overwrites that new value with its stale result. The migration can finish with no errors and still corrupt data.

Define a source snapshot or consistent boundary and use version-aware writes. For each record, retain a version/update timestamp/sequence and update the destination only if the backfill's version is newer than the destination version—or ensure the backfill cannot overwrite a row changed since it was read. Timestamps require care because clocks can skew; database-generated versions or log positions are safer when available. Another pattern is to copy a snapshot, then replay CDC changes after the snapshot's log position before cutover.

Run bounded batches with throttling so the migration does not overload the primary or increase production p99 latency. Make the worker restartable and idempotent; record checkpoints and per-partition progress. Validate counts, sampled checksums, constraints, null/missing values, and business invariants. Capture deletes as well as inserts/updates.

A safe cutover includes a lag threshold, reconciliation of mismatches, read shadowing where feasible, and a rollback/forward-repair plan. Do not treat “backfill job completed” as proof of consistency. Explain how concurrent writes are captured, how stale work is rejected, and how you will stop or resume the migration during an incident.

### Q166. How do you estimate shard count without pretending the estimate is exact?

Estimate shard count from several independent limits, then use the most restrictive credible one. First measure or load-test **per-shard safe capacity** using the expected query mix, index size, writes, transactions, and latency objective. A shard's marketing maximum or peak benchmark is not a safe production target; include headroom for skew, failover, maintenance, and bursts.

A starting model is:

- **Storage constraint:** required logical data divided by usable logical capacity per shard.
- **Throughput constraint:** expected peak reads/writes divided by the safe per-shard throughput for that same workload.
- **Hot-key/tenant constraint:** the largest tenant or key must fit within one shard's limit—or the data model must explicitly allow that tenant/key to be split.
- **Operational constraint:** query fan-out, connection budgets, rebalance duration, backup windows, replica placement, and the number of nodes the team can safely operate.

Take the maximum of the constraints, then add an explicit growth and failure margin. These calculations are a first sizing pass, not proof: sharding changes query plans, network traffic, lock contention, and cache behavior, so benchmark with realistic skew and failure conditions.

**Illustrative storage example:** assume 12 TB of logical live data and a target of no more than 1 TB of usable logical data per shard. The storage constraint alone suggests at least 12 shards. If each logical shard has one primary and two replicas, there are 36 stored copies across the fleet before counting indexes, logs, temporary migration copies, and backups. That replication factor does not mean there are 36 independent logical shards. If write throughput or the largest tenant requires 18 shards, throughput/skew—not storage—becomes the governing constraint.

Do not start by selecting an arbitrary number such as 16 or 32 because it is a power of two. Explain the measurements that would validate your estimate and keep enough flexibility to move or split data later.

### Q167. What makes re-sharding difficult, and how would you do it safely?

Re-sharding is a live data migration plus a routing migration. The hard part is not copying existing rows; it is keeping the old and new placements consistent while writes continue, clients run different software versions, and retries or failures occur at every stage. A partial cutover can cause missing reads, duplicated writes, conflicting owners, or data that is difficult to roll back.

A disciplined migration usually has these phases:

1. **Define a stable routing abstraction.** Use a versioned mapping from logical key or virtual partition to physical shard, so adding nodes does not require embedding a fixed modulo calculation in every client. Confirm that all readers, writers, background jobs, and repair tools use the routing layer.
2. **Prepare the destination.** Create schemas and indexes, apply compatible migrations, and verify capacity, access controls, backup and replication settings.
3. **Copy a consistent baseline.** Snapshot or copy the source data in bounded chunks. Track progress by partition and use retry-safe writes. A snapshot timestamp or log position defines the boundary between copied data and later changes.
4. **Capture ongoing writes.** Replay the source change log/CDC stream after the baseline position, preserving the required ordering per key. Uncoordinated dual writes to old and new shards are dangerous: one side can succeed while the other fails. If dual writes are used, the repair protocol and authority model must be explicit.
5. **Validate and shadow-read.** Compare counts, checksums, sampled records, deletes, and key invariants. Send shadow reads to the destination and compare results without changing user-visible responses.
6. **Cut over using an ownership epoch or version.** Prevent stale application instances from continuing to write to the old location. Switch a small cohort first, monitor errors and mismatch rates, then expand. Retain the old copy until the rollback window and reconciliation checks are satisfied.
7. **Retire deliberately.** Stop old writes, prove no supported client still depends on the old mapping, and only then delete old data.

Rollback is not automatically safe after new writes have been accepted by the destination. You need a defined reverse-replication or reconciliation path, or a clear point after which rollback means forward repair rather than switching back. Track copy lag, CDC lag, per-shard load, conflicts, mismatch counts, and cutover progress. In a senior interview, include how you handle deletes, stale route caches, a migration worker crash, and a rollback after writes reach the new shard.

### Q168. How do you generate unique IDs across regions?

Choose an ID strategy from the requirements: uniqueness scope, global ordering, approximate time ordering, generation latency, sortability, index behavior, privacy, and whether IDs must be generated during a network partition. Different schemes solve different needs; no strategy gives all properties for free.

A central database sequence is simple and can provide a monotonic sequence, but it introduces coordination and can become a bottleneck or single-region dependency. Range allocation lets a coordinator lease blocks of IDs to nodes, reducing per-ID coordination but requiring safe range ownership and handling unused IDs. Random UUID-style identifiers allow decentralized generation with a very low collision probability when the space and randomness are adequate, but they are not naturally time-ordered and may affect index locality. Time/worker/sequence schemes (such as Snowflake-like designs) can be compact and roughly time-sortable, but require worker identity allocation, bit-layout capacity planning, and clock rollback handling.

Uniqueness does not imply ordering. Two regions may generate IDs whose lexical order does not match real-time event order. A timestamp component is not a perfect clock or a global ordering service. If the business requires strictly monotonic order, a coordination point may be necessary.

Describe how worker IDs are assigned uniquely across deployments and failover; how clock skew or rollback is handled; what happens when sequence bits are exhausted; and whether IDs reveal information. The ID generator should be tested during region loss and split brain, not only under normal startup. If the database can enforce a uniqueness constraint at the true invariant boundary, keep that as a final safety check where feasible.

### Q169. Why can time-based IDs cause problems?

Time-based identifiers often include timestamp bits to make IDs roughly sortable and decentralized to generate. The trade-offs arise because clocks are imperfect and timestamp order is not the same as a globally agreed causal order. Clock rollback or skew can cause a generator to wait, reuse a sequence range, emit out-of-order identifiers, or collide if worker/sequence bits are not correctly managed.

Sequential or timestamp-leading keys can also concentrate database index writes toward the newest B-tree pages or newest range partition. This is not always bad—locality can improve cache behavior—but it may create a hot write target at high throughput. Randomized suffixes, alternate index strategies, or hash distribution can help, each with its own range-query and locality cost.

The exact risk depends on the ID algorithm: worker identity allocation, timestamp resolution, sequence-bit capacity, clock rollback behavior, and uniqueness assumptions. A generator needs a safe response if its clock moves backward or its worker identity is duplicated after failover. Random UUIDs avoid time coordination but provide different sorting/index properties and collision probability.

Do not use IDs as a substitute for ordering or authorization. IDs can reveal approximate creation time or volume, and lexical order may not reflect causal order across regions. In an interview, state whether you need unique, sortable, monotonic, or globally ordered IDs; these are distinct properties and may require different mechanisms.

### Q170. How should timestamps and event ordering be represented?

A timestamp records a clock reading; it does not by itself establish a strict order across distributed writers. Machines can have clock skew, time synchronization can step or drift, and an event may be created at one time but published or observed much later. Two events with identical timestamps or timestamps that appear reversed may still have a clear causal order.

Use the field that matches the question: event occurrence time, database commit time, publication time, ingestion time, or processing time are different values. Use UTC and an explicit machine-readable format for wall-clock timestamps, and add a stable sequence/version when per-entity order matters. Database log positions, Kafka offsets within a partition, and aggregate versions can provide order within their defined scope, but none is automatically a global order across partitions or regions.

For state updates, use conditional version checks or compare-and-set rather than “latest timestamp wins” if clock skew could overwrite the newer business state. For event-time analytics, late arrivals require a watermark/lateness policy; processing-time ordering may give a different result. For lease and expiry decisions, define which clock is authoritative and use fencing or conditional transitions as needed.

In an interview, label the timestamp semantics and ordering scope explicitly. If the question is “which event happened first?”, clarify whether the business means real-world occurrence, causal order, commit order, or ingestion order. A single timestamp field cannot answer all four reliably.

### Q171. How do you prevent stale updates from overwriting newer state?

A stale update can occur when a slow request or delayed event arrives after a newer change and overwrites it. Example: event version 12 updates a search document, then delayed version 10 arrives and writes the older state. Sorting by wall-clock timestamp may not be safe because clocks can skew and event creation, commit, and delivery times differ.

Use a monotonic version/sequence per entity and make writes conditional: apply an update only if its version is newer than the stored version. Database optimistic locking can use a version column; event consumers can reject stale events; caches can compare versioned values. The version must be assigned at the authoritative state transition and preserved through propagation.

For deletes, retain a versioned tombstone long enough that old updates cannot recreate the entity. For multi-writer data, define conflict resolution explicitly—last-write-wins is safe only where overwriting based on its chosen timestamp semantics is acceptable. For business invariants, a version check may need to be part of a transaction or conditional update.

Monitor out-of-order arrivals, rejected stale writes, version gaps, and reconciliation mismatch. In an interview, walk through a concrete interleaving and show the atomic comparison that prevents stale state from winning. Explain how the design behaves if the update source has been compacted, replayed, or migrated.

### Q172. What is a reconciliation loop, and why is it useful?

A reconciliation loop repeatedly compares observed state with desired or authoritative state and takes idempotent actions to close the gap. It is useful when distributed systems cannot guarantee every update is delivered once and in order, or when failures can interrupt a multi-step workflow. Controllers in orchestration systems are a common example: they inspect actual state and drive it toward the declared desired state.

For business data, reconciliation might compare internal payment records against provider status/settlement, verify that a search index matches the source database, or find reservations whose holds expired but inventory was not released. The loop should identify the authoritative side and must not blindly overwrite a valid newer state with an older observation.

Make each repair action idempotent, bounded, auditable, and safe to retry. Include version/sequence checks, stable operation IDs, rate limits, checkpoints, and explicit escalation for ambiguous conflicts. A reconciler that continually flips state back and forth can amplify an incident; add convergence rules, retry backoff, and alerts for repeated oscillation.

Measure mismatch count, oldest unresolved item, repair rate, and repeated failures. Reconciliation complements transactional correctness; it does not justify ignoring an atomic invariant that can be enforced locally. In an interview, define desired state, observed state, safe repair action, and the mechanism that prevents the repair process from causing new corruption.

### Q173. How do you decide between synchronous replication and asynchronous replication?

With asynchronous replication, the primary can acknowledge a write before all replicas have applied it. This reduces latency and permits writes to continue when replicas are slow or disconnected, but a primary failure can expose a gap between acknowledged state and the replica promoted for recovery. The actual loss risk depends on the database's commit/flush/replication configuration and failover policy.

Synchronous replication waits for one or more replicas according to a configured acknowledgement rule before the write is considered committed. This can reduce the risk of losing acknowledged writes, but increases write latency and may reject/block writes when the required replicas cannot respond. “Synchronous” must be qualified: acknowledgement may mean received in memory, flushed to durable storage, or applied; a quorum protocol may have other semantics. Read the product's guarantee precisely.

Choose from the business RPO, geographic distance, latency budget, and partition behavior. A cross-region synchronous commit may add a network round trip to every write and reduce write availability during a link failure. A system can instead keep a single write leader in a region, replicate asynchronously, and accept that a regional failover may require an explicit data-loss decision—or use stronger commit semantics for the critical dataset.

Test promotion after a node/zone/region failure and verify which writes are present. Monitor replication lag/position, not just a green “replica connected” status. In an interview, state the failure case you promise to tolerate, whether acknowledged writes can be lost, and how the chosen topology meets latency and RPO requirements.

### Q174. How do you reason about database connection-pool size?

A connection pool caps how many concurrent database sessions an application can hold and avoids paying connection setup cost for every request. It is also a queueing boundary: when all connections are checked out, new requests wait in the application. Increasing the pool only helps if the database can use the extra concurrency; otherwise it shifts contention into the database.

Calculate the fleet-wide connection budget, not just one service instance. If 20 application instances each configure a maximum pool size of 50, they can potentially open 1,000 connections, before batch workers, migrations, monitoring, and administrative sessions. Reserve capacity for operations and failover, and consider connection limits per database cluster, shard, and tenant. Autoscaling can multiply connections suddenly.

Size from measured query/transaction duration, needed concurrency, and the database's safe throughput under the real workload. Long queries hold connections longer; short queries may need fewer. Use connection acquisition wait, pool utilization, hold duration, database active sessions, lock waits, and query latency to diagnose. If the pool is saturated because transactions wait on locks, making it larger may increase the number of blocked sessions and worsen tail latency.

Bound acquisition time and fail quickly when the request deadline cannot be met. Avoid holding a connection while doing remote HTTP calls or expensive CPU work. For large fleets, use a proxy/pooler where appropriate, but understand transaction/session semantics and limits. In an interview, explain how you prevent connection storms during rollout/failover and how you distinguish a small pool from a saturated database.

### Q175. What is the difference between a database lock wait and connection-pool wait?

A **connection-pool wait** happens before a request obtains a database connection. It can be caused by a pool that is too small for the workload, connections held too long, a leak, slow acquisition, or a surge in request concurrency. The query may not have reached the database yet. Increasing pool size can help only if the database has remaining capacity.

A **lock wait** occurs after a transaction has a connection and tries to access data that another transaction holds or conflicts with. Causes include long transactions, hot rows, lock ordering, a missing/poor index that expands the locked work, or a transaction that waits on a remote call while holding locks. The remedy is usually to reduce contention or transaction duration, not to create more connections.

Instrument pool acquisition time, connection hold time, active/idle/waiting pool counts, database sessions, lock wait graphs, deadlocks, slow queries, and transaction age. Traces should distinguish waiting for a connection from executing SQL. A service can show low CPU while all request threads wait for a pool slot or database lock.

Use the symptom to test the hypothesis. If acquisition wait rises while the database has few active queries, inspect leaks, connection holds, and pool limits. If connections are busy with blocked transactions, inspect blocking sessions and transaction paths. In an interview, clearly explain where the request is waiting and how the proposed fix reduces the queue without pushing overload to another layer.

### Q176. How do you choose a storage model for time-series data?

Start with the query and retention shape: append rate, points per second, number of distinct series, time-range queries, downsampling, late arrivals, and how often old data is updated or deleted. Time-series data often benefits from partitioning by time so queries can prune irrelevant ranges and retention can remove old data efficiently. If all writes target the newest range on one node, combine time partitioning with a distribution strategy that avoids a hot write target.

Choose the storage engine based on ingestion rate, compression, cardinality, aggregation/query patterns, and operational needs. A relational database can be sufficient at moderate scale, particularly with appropriate indexes and time partitions. A specialized time-series database or columnar store may be better for high-ingest metrics and range aggregations; object storage can retain raw historical data cheaply but generally has different query latency and access patterns.

Series cardinality can dominate storage and index cost. Labels such as unique user IDs or request IDs may create millions of distinct series, overwhelming a metrics-oriented store. Define allowed dimensions, aggregation/downsampling, and separate detailed event logs from high-cardinality metrics. For late or corrected samples, decide whether to update historical buckets, write correction records, or accept a watermark/lateness window.

Retention is part of the design: raw data may stay for days, downsampled aggregates for months, and long-term archives for years. Include compression, replication, indexes, ingestion buffers, backups, and query concurrency in estimates. In an interview, map the access pattern to storage layout and explain how partitions, cardinality, late events, and retention affect cost and performance.

### Q177. How do you design a search index rebuild without losing live updates?

A search index is usually a derived projection, so rebuilding it must account for writes and deletes that happen while the bulk load is running. Deleting the old index and loading a new one directly can expose an empty or partially populated search experience and can miss concurrent updates.

A safer pattern is to build a **new versioned index** beside the live one. Establish a consistent source boundary, such as a snapshot tied to a database log position or CDC offset. Bulk-load the snapshot in bounded batches, then replay changes after that boundary in the correct per-entity order. Track consumer lag and retain enough source-log history to finish the rebuild. If the log expires before catch-up completes, the process needs a new snapshot or a defined restart.

Validate record counts, sampled documents, key fields, analyzer behavior, versions, and deletes/tombstones. Shadow queries can compare the new index against the old index without changing user-visible responses. During catch-up, use version-aware writes so an older bulk-load record cannot overwrite a newer event. Then switch an alias or routing pointer atomically where supported, monitor search latency and result quality, and retain the old index for a bounded rollback window.

Do not confuse infrastructure success (index exists and responds) with search correctness. A rebuild can silently omit documents or resurrect deleted records. Plan for schema/analyzer version changes, rollback after new updates, capacity for two indexes during migration, and privacy/deletion requirements. In an interview, identify the source of truth, snapshot boundary, live-change capture, validation gates, and cutover mechanism.

### Q178. How do you make pagination stable while records are changing?

Offset pagination can skip or repeat records when rows are inserted or deleted before the offset between page requests. Keyset/cursor pagination avoids the growing cost of large offsets by continuing from the last sort key, but it is only stable if the ordering is deterministic and the cursor includes enough information to resume correctly.

Use a stable order with a unique tie-breaker, for example `(created_at, id)`, and encode the last observed key in an opaque cursor. Protect or sign the cursor if clients must not alter filters, tenant context, or page position; validate it against the request's sort/filter schema. The query should use a matching index so it can seek from the cursor rather than scan from the beginning.

Cursor pagination does not automatically provide a snapshot. If rows change their sort key, records can move across page boundaries. For a live feed, that may be acceptable; for an audit export or billing report, use a snapshot/version boundary, a database snapshot transaction where operationally practical, or an asynchronous export built from a fixed dataset. Avoid long-running database snapshots that retain old versions or lock resources for too long.

Define deletion behavior, filtering changes, maximum page size, cursor expiry, and what happens when the underlying resource is deleted. A cursor is a continuation token, not an authorization grant. Revalidate authorization on each page. In an interview, explain the ordering/index choice and whether the API promises a stable snapshot or simply efficient continuation over a changing dataset.

### Q179. How do you design a webhook delivery system?

Treat each webhook as durable outbound work. When a business event occurs, record the delivery intent transactionally with the source change—often through an outbox—then enqueue or publish the delivery. The delivery worker signs the payload, sends it with a bounded timeout, records the response and attempt, and retries eligible failures with exponential backoff and jitter. A slow customer endpoint must not block the business transaction.

Use a stable event/delivery ID so receivers can deduplicate repeats. At-least-once delivery is common because the sender can time out after the receiver processes the event but before it receives the acknowledgement. Sign payloads with a timestamp and a secret or asymmetric key, use TLS, support secret rotation, and document replay protection. The receiver should verify the signature and use the event ID to make its handler idempotent.

Classify responses: success codes acknowledge delivery; timeouts, connection failures, and some 5xx/429 responses may be retryable; permanent 4xx failures often require pausing, alerting, or endpoint correction. Respect retry-after where appropriate, but keep a bounded retry horizon. Make delivery order explicit: per-destination order may be needed for state changes, but global order across all customers is usually expensive.

Protect the platform with per-customer concurrency and rate limits, bounded queues, dead-letter/failed-delivery views, manual redrive, and endpoint disablement after sustained failure. Monitor success ratio, oldest undelivered event, attempts, latency, and customer-specific failures. Provide a delivery log and event replay interface with audit controls. Never assume a webhook is delivered exactly once or that HTTP 2xx proves the customer completed its downstream business workflow.

### Q180. How should a system handle poison records without violating ordering?

A poison record cannot be processed successfully under the current consumer code or data contract. Moving it to a dead-letter queue lets the partition progress, but may violate the business requirement if later records depend on the missing state transition. The design must choose between blocking that key/partition, quarantining the record and accepting a gap, or applying a domain-specific skip/repair.

For per-entity ordering, isolate by key where possible. Pause or defer only the affected key/partition rather than every unrelated workload if the broker and consumer architecture allows it. Preserve the original record, offset/sequence, schema version, failure reason, and trace context. Fix the cause, then replay the record before later dependent events. If skipping is permitted, record an explicit gap/compensation decision and reconcile the resulting state.

Automatic retries should be bounded for deterministic failures; repeated deserialization or validation failure will not improve with faster retry. Use delayed retries for transient dependencies and a quarantine/DLQ for poison payloads, while protecting the retry topic from the same failure. Manual replay must be rate-limited and idempotent, and it should not change event identity unless the business action is genuinely new.

Measure blocked partition time, oldest poison record, DLQ age, and entities affected. In an interview, identify whether ordering is global, per partition, or per entity; then explain why your chosen remediation preserves or deliberately relaxes that guarantee.

### Q181. What is the difference between a queue's depth and age, and which is more useful?

**Depth** tells how many items are waiting (sometimes also measured in bytes). **Age** tells how long the oldest or typical waiting item has been pending. Both matter, but they answer different questions. A large queue may be draining quickly with enough capacity; a small queue can still violate an SLA if one critical item is stuck behind a poison message or blocked partition.

For example, a queue contains 500,000 events and workers complete 40,000/s while arrivals are 20,000/s. It has high depth but strong net drain. Another queue contains only 2,000 jobs, but the oldest job is 45 minutes old and the completion target is five minutes; the service objective is already violated. Track both with successful completion rate, arrival rate, retry count, and per-partition skew.

For deadlines, record enqueue time and, where relevant, the business expiry time. Avoid relying solely on consumer clocks for ordering or lease decisions; define timestamp semantics and use monotonic/versioned identifiers when needed. If old work is no longer useful, expire it deliberately and report the reason rather than processing it indefinitely.

Use depth to plan capacity and storage, age to detect customer impact and stuck work. Alert on time-to-drain versus the business deadline, not just one absolute item count. In an interview, explain what each metric tells you and how it changes the response—scale, throttle, repair a blocked consumer, or expire stale work.

### Q182. How do you prevent one slow consumer from affecting others?

Isolation starts with understanding what the consumers share: a process/thread pool, broker partition, queue, database connection pool, provider quota, or a shared lock. A single slow consumer can block others if work is serialized behind one partition, if all messages use one ordered key, or if one destination monopolizes shared concurrency. Adding more workers may not help if the ordering or external dependency is the actual limit.

Use separate consumer groups for independent subscribers, bounded concurrency and queues, per-tenant or per-destination quotas, and dedicated pools for high-cost work where justified. For a shared queue, weighted fair scheduling can prevent one tenant from consuming all available workers. For Kafka-style partitioning, parallelism is bounded by partitions in a group and ordering is per partition; a hot key cannot simply be spread across consumers without changing semantics.

Use deadlines and circuit breakers around slow external calls, and avoid holding broker partitions or database transactions unnecessarily while waiting on network operations. Some brokers use visibility timeouts or per-message leases; make lease duration and redelivery semantics safe for long jobs. Consumers should commit/ack only according to the durability boundary promised by the workflow.

Observe per-consumer throughput, per-partition lag, queue age, active concurrency, throttling, and error rates. In an interview, name the shared resource and explain which isolation boundary prevents one slow consumer from delaying critical unrelated work without breaking ordering or fairness.

### Q183. How do you choose a message retention period?

Retention is determined by recovery, replay, audit, and business-expiry requirements—not by picking an arbitrary number of days. Ask how long consumers may be unavailable, how long a schema bug might go unnoticed, whether new consumers need historical events to build projections, what audit/regulatory obligations apply, and how much storage the retained volume requires.

For an event stream, retention must be long enough for the worst credible consumer outage plus detection, repair, and catch-up time, with margin. If a consumer can be offline for seven days but the broker retains only three, it may need a full snapshot/rebuild from another source. For a work queue, retention also interacts with expiry: processing a notification two weeks after its deadline may be worse than dropping it with an explicit outcome.

Estimate storage from incoming bytes per second, retention duration, replication factor, compression, segment/index overhead, and peak bursts. Longer retention increases disk and recovery/replay volume. Backups, object-storage archives, and compacted topics may offer different trade-offs but have distinct guarantees. Tombstones and compaction rules must be understood if deletes or latest-state reconstruction matter.

Document what happens on expiry, who owns replay, how schemas stay compatible over the retention window, and whether payloads contain data subject to deletion or residency requirements. Monitor retention headroom and consumer lag so the system does not silently delete the history a slow consumer still needs. In an interview, connect retention directly to the recovery plan and the system's rebuild capability.

### Q184. How do you avoid retrying permanent failures forever?

Classify failure categories and give each a policy. Network timeouts, temporary service unavailability, and throttling may be retryable; malformed payloads, schema incompatibility, invalid credentials, and business-rule violations generally need correction rather than repeated attempts. Some responses—such as 429 or 5xx—require reading the specific provider contract rather than applying one universal rule.

Bound attempts and total retry age, use exponential backoff and jitter, honor retry-after signals, and stop retrying when the operation's business deadline expires. Send exhausted or deterministic failures to a DLQ/quarantine with enough context to investigate and safely replay. Alert on increasing DLQ age/count and repeated failure reasons; a DLQ that no one owns is silent data loss from the business's perspective.

Do not discard events without understanding ordering and state impact. A poison event may block dependent events for the same entity. A human or automated remediation path should repair the schema/data or mark an explicit compensation, then replay with the original event identity so deduplication still works.

External calls require idempotency because a timeout can hide a successful side effect. Before retrying a payment or webhook delivery, use a stable downstream operation ID or query/reconcile the provider state. In an interview, define transient versus permanent failure, the retry budget, escalation path, and what evidence proves a failed operation was safely resolved.

### Q185. What should happen when a circuit breaker opens?

When a breaker opens, new calls to the dependency should fail fast or use a documented fallback instead of waiting on a dependency that is currently failing or too slow. The desired behavior is route-specific: an optional recommendation call may return an empty result, while a payment authorization cannot safely claim success without confirmation.

A breaker needs defined failure categories, a measurement window/minimum sample count, thresholds, open duration, and a half-open probe policy. Network/timeouts and selected 5xx/slow calls may count; client validation errors should not normally be treated as dependency unavailability. Thresholds that are too sensitive can turn transient noise into avoidable outage, while thresholds that are too lax let requests pile up.

Opening the breaker is only one protection. Bound concurrent calls with a bulkhead, use deadlines, and cap retries so callers do not continue consuming resources while the breaker evaluates failures. At scale, synchronize-free or per-instance breakers may transition at different times; half-open probes should not overwhelm a recovering dependency. Do not add a fallback that returns semantically incorrect or stale data without defining the tolerance.

Monitor breaker state transitions, rejected calls, fallback frequency, dependency latency/error rate, and the time to recovery. Test open, half-open, and recovery states. In an interview, explain what users receive while the breaker is open and how the system determines that the dependency is safe to receive traffic again.

### Q186. How do you choose between fail-open and fail-closed behavior?

**Fail-open** means continuing the operation when a control or dependency is unavailable; **fail-closed** means rejecting or pausing the operation because the required decision cannot be made. The correct choice depends on what an incorrect allow or incorrect deny costs. Neither is universally safer.

For a payment or sensitive authorization check, allowing access because the policy service is unavailable can violate security or financial invariants, so fail-closed may be appropriate. For noncritical analytics or a recommendation service, failing open by omitting the optional capability may preserve the main user journey. A rate limiter can have mixed requirements: coarse abuse protection may use a bounded local fallback, while a hard account-wide billing quota may require strict enforcement.

Specify the failure mode and scope. “Fail-open” should not mean unbounded work; local concurrency caps or conservative cached policy can provide a controlled degraded mode. “Fail-closed” should not unintentionally take down all routes when only one optional feature needs the dependency. Use cached decisions only with explicit freshness and revocation rules.

For every dependency, define the safe default, maximum time allowed in degraded mode, monitoring, and recovery behavior. In an interview, map the choice to the business invariant, explain abuse/correctness risks, and state how clients observe rejection or degraded responses. The policy should be tested during dependency outages, not decided during an incident.

### Q187. How should a feature degrade during overload?

Start by separating the system's core promise from optional work. Under overload, it may be acceptable to remove recommendations, delay analytics, reduce image resolution, or reject bulk exports so that checkout, authentication, or other critical journeys remain usable. It is not acceptable to return a fabricated success for a payment or reserve inventory based on stale data simply to preserve the HTTP success rate.

Define the degraded mode in advance. Set workload priorities, per-route concurrency budgets, queue bounds, feature flags/kill switches, and explicit fallback semantics. A request should fail quickly with a meaningful retry policy when it cannot finish within its deadline, rather than sitting in a queue until it times out. Use load shedding and admission control before the system exhausts thread pools or database connections.

Fallbacks can create a second incident: serving stale data can violate permissions or inventory rules, and redirecting all traffic to an alternate provider can exceed its quota. Define which dependencies are required, how stale the data may be, and when a fallback is disabled. For asynchronous work, durably accept only what can be stored and processed before its business deadline.

Measure successful critical operations, p99 latency, rejected work, queue age, and business correctness during overload. Test degradation at planned load and beyond capacity. A senior answer states what is preserved, what is sacrificed, and why the degraded behavior remains correct and understandable to users.

### Q188. How do you design an operational kill switch?

A kill switch disables a specific risky behavior quickly without requiring a full deployment—for example, stop a new recommendation path, pause a replay job, disable an external integration, or prevent a new type of write while preserving safe reads. It is an emergency control, not a substitute for rollback, admission control, or data repair.

Design its scope narrowly and define the safe state. A switch that disables a feature should not accidentally bypass authentication, skip a required inventory check, or leave a workflow permanently half-complete. For a partially completed business operation, the kill switch must determine whether to reject new operations, finish existing ones, compensate, or let a durable state machine resume later.

The switch should have authenticated, audited changes; explicit owner and reason; observable current state; and a tested propagation path to all instances. If configuration is distributed asynchronously, measure how long it takes to become effective and what happens when the control plane is unreachable. For high-risk actions, use a fail-safe default and avoid letting arbitrary application users change the control.

Test activation and recovery, including stale caches and in-flight requests. A switch that is not exercised can fail precisely when needed. Monitor effects after activation and define clear criteria for re-enabling the feature. In an interview, identify what gets stopped, what remains safe, how existing work is handled, who can act, and how the team verifies that the switch took effect.

### Q189. How do you measure the quality of an asynchronous workflow?

Measure from the caller's durable acceptance to the final business outcome. HTTP response time often reports only how quickly the job was queued, not whether it was completed correctly. Track acceptance success, end-to-end completion success, time-to-complete percentiles, timeout/expiry rate, oldest pending age, retries, dead-lettered items, and operations stuck in intermediate states.

Define the terminal states and completion semantics. A webhook being accepted by a broker is not the same as the receiver processing it; a payment command being accepted is not the same as settlement. State transitions should be durable and correlated by a stable operation ID. For external systems, use receipts, status queries, or reconciliation to identify outcomes that remain unknown after a timeout.

Measure both system throughput and business validity. A worker can process 100,000 records/s yet fail to apply them correctly, or complete events after their useful deadline. Track duplicates, ordering gaps, version conflicts, compensation outcomes, and manual intervention. Segment metrics by workflow type, priority, tenant tier, and provider without using every workflow ID as a metric label.

Choose alerts from user-facing objectives: for example, “99% of accepted exports complete within 10 minutes” or “no payment remains unknown for more than 5 minutes without investigation.” Combine metrics with traces/logs for diagnosis and a durable state history for audit. In an interview, explain how you detect silent loss, stuck workflows, and incorrect completion—not just worker crashes.

### Q190. How do you test a system's capacity before launch?

Translate requirements into a realistic workload model: request mix, read/write ratio, payload sizes, key/tenant skew, burst patterns, cache hit rate, downstream latency, and arrival rates. A constant stream of identical tiny requests is rarely representative. Include background work, event consumers, retries, and administrative operations that compete for resources in production.

Increase offered load in stages while measuring successful throughput, p50/p95/p99 latency, errors, saturation, queue age, and database behavior. The goal is not merely to find a request rate that returns responses; it is to identify the maximum sustainable load within the SLO and understand what breaks next. Test with production-like data volume and indexes, since query plans and cache behavior can change with data size.

Test failure conditions as well: loss of an instance or zone, cache cold start, increased database latency, provider throttling, broker lag, connection churn, and recovery after the burst. Verify scale-out time and headroom after a failure, not only the all-instances-healthy path. Protect downstream systems and third-party environments from load tests through explicit limits and coordination.

Use a controlled baseline, compare one change at a time, and keep the test repeatable. Validate that the system's business invariants still hold under load—for example, no duplicate booking or lost event. A senior answer should explain the target SLO, the workload assumptions, the bottleneck found, the capacity margin, and the corrective action taken before launch.

### Q191. What is the difference between load, stress, spike, and soak testing?

**Load testing** checks performance under the expected workload and planned peak, including latency, throughput, errors, and resource use. **Stress testing** pushes beyond the expected operating range to find limits and observe failure behavior: does the system reject safely, collapse unpredictably, or recover? **Spike testing** introduces a rapid demand increase to test cold-start, autoscaling, connection, cache, and admission-control behavior. **Soak testing** sustains a representative load over a longer period to uncover leaks, gradual queue growth, compaction problems, resource drift, and periodic-task interference.

A meaningful test needs a workload model and an explicit pass/fail criterion. The target may be “p99 below 300 ms and fewer than 0.1% errors at 4,000 requests/s,” not just “the test completed.” Also track successful business operations, database contention, external quotas, and the time required to drain a backlog after a spike.

Use production-like data volumes and dependency behavior, while protecting real users and providers. Test result interpretation should separate offered load from successful completions: high attempts with rising errors do not demonstrate capacity. Repeat tests after meaningful schema, query, library, instance, or configuration changes.

Each test answers a different question, so passing a steady-state load test does not prove spike recovery or long-duration stability. In an interview, choose a test that targets the suspected failure mode and explain what evidence would change the architecture or capacity plan.

### Q192. How do you design chaos experiments safely?

A chaos experiment deliberately introduces a controlled failure to test a resilience hypothesis—for example, “the service remains within its SLO when one application instance disappears” or “the consumer recovers after broker connectivity is interrupted.” It should test a specific property, not randomly break production systems without a way to judge the result.

Begin with a baseline, a written hypothesis, measurable steady-state indicators, a bounded blast radius, and a stop condition. Start in a test environment or small canary cohort; inject one failure at a time before testing correlated failures. Use realistic failure modes such as latency, packet loss, process termination, dependency errors, leader loss, or zone isolation. Ensure the team can stop the experiment and restore the environment quickly.

Protect data integrity first. Experiments involving payment ledgers, destructive writes, failover ownership, or production customer data require additional safeguards and carefully scoped test entities. Avoid experiments that cause uncontrolled third-party calls, exceed provider quotas, expose secrets, or create non-reversible side effects. Have a responder and rollback/cleanup plan, and coordinate with incident/on-call teams.

A successful experiment is not merely that the service stayed up; it confirms expected behavior and surfaces limitations. Record what happened, whether alerts fired, whether operators followed the runbook, and how long recovery took. If the experiment violates the hypothesis, fix the design or update the SLO/runbook before increasing blast radius. In an interview, distinguish a disciplined fault-injection test from uncontrolled production disruption.

### Q193. How do you define useful alerts from SLOs?

Start from a user-facing SLI and SLO, then alert when the system is consuming its error budget fast enough that action is needed. A service can violate a 30-day SLO only gradually, or burn most of its monthly budget in minutes; these situations need different alert urgency. Burn-rate alerts help detect fast deterioration without waiting for the full window to close.

An alert should be actionable. It should identify the affected journey/region, the symptom, the likely diagnostic links, and the runbook or owner. CPU at 80% may be a useful dashboard signal but not necessarily a page unless it predicts user impact or imminent failure. Conversely, a low-volume critical workflow can violate a business invariant while aggregate availability and CPU look healthy.

Use multiple windows or thresholds to balance fast detection against noise. Alert on end-to-end successful completion, p99 latency, queue age, provider failure, replication lag relative to RPO, and error-budget burn where they correspond to a clear response. Avoid high-cardinality labels and duplicate alerts that all fire for the same root cause.

Review alerts after incidents: did the alert fire early enough, was it noisy, and did the action restore service? A senior answer should connect alert design to the SLO, an operator decision, and a tested runbook—not merely list every metric that can be monitored.

### Q194. How do you reason about a dependency's error budget?

A service's SLO may depend on another service, but the dependency's published availability does not automatically equal its contribution to your user-visible success. The impact depends on whether the dependency is required for a specific operation, how its failure modes correlate with yours, and whether there is a safe fallback or cache.

If an endpoint requires a payment authorization dependency, its unavailability directly limits successful payment requests. If an endpoint can omit recommendations when the recommender fails, the recommender may affect feature completeness but not the core page-availability SLI. Track dependency success and latency by route, and distinguish failures the caller can safely handle from failures that break the business invariant.

Set an internal reliability target for dependencies based on the user journey. A downstream SLO can be stricter or looser than the upstream service's need, but the upstream team must not assume the dependency team will always meet an idealized target. Include timeout, retry, quota, maintenance, and correlated failure behavior in the integration contract.

When the dependency's error budget burns quickly, reduce optional traffic or use an approved degraded mode; do not blindly retry and increase pressure. Coordinate ownership, escalation, capacity plans, and incident reviews across teams. In an interview, draw the dependency graph, identify critical edges, and explain how the upstream system protects its own SLO when the downstream is degraded.

### Q195. What is a failure domain, and how do you reduce correlated failures?

A failure domain is a set of components likely to fail together: one process, host, rack, availability zone, region, shared network, common database, credential provider, deployment, or control plane. Placing two replicas on different hosts helps against host failure; it does not protect against a zone outage if both hosts share that zone, or a bad release if both run the same artifact and configuration.

Map the dependencies and failure causes, not just the deployment topology. Place replicas across independent domains where the recovery objective warrants it; avoid shared credentials, storage, network paths, and quotas that erase the benefit of geographic redundancy. Stagger deployments and retry schedules, reserve capacity for failover, and ensure control-plane outages do not immediately stop all data-plane traffic.

Redundancy can create correlated overload. If one zone fails and all traffic moves to the survivors, the survivors must have sufficient headroom or shed lower-priority work. Health-based routing that removes every slightly slow node can also reduce capacity and trigger a cascading failure. Model capacity after a failure, not only under normal balancing.

Test representative failures and inspect what was actually independent. A “multi-region” architecture can still share a single global database, identity provider, KMS, DNS control plane, or external provider account. In an interview, name the failure being tolerated, the isolation boundary that helps, the shared dependencies that remain, and the observed evidence that failover works.

### Q196. How do you evaluate a fallback's correctness?

A fallback is correct only if the behavior remains within the business contract when the preferred path fails. A response can be technically available but incorrect—for example, returning stale inventory that leads to an oversell, treating a failed authorization check as allowed, or claiming a payment succeeded without provider confirmation.

Classify the dependency's role. If it provides optional enrichment, omission may be safe. If it provides authoritative inventory, access permission, or payment state, stale cached results may not be safe for the final mutation. Define permitted staleness, which operations may proceed, which must fail closed, and how users are told that a result is partial or pending.

Evaluate fallback data freshness and provenance; a cache may be behind, an alternate provider may have different semantics, and a secondary region may not contain the latest acknowledged write. Verify that fallback capacity can absorb the traffic and that switching does not cause a provider quota spike. Ensure fallback does not skip audit logging or security controls.

Test fallback behavior under partial response, timeout-after-success, and recovery. Measure fallback frequency and business error indicators, and provide a reconciliation path for uncertain outcomes. In an interview, explain not just what the fallback returns but why it is safe, how stale it may be, what it costs, and when the system should stop serving it.

### Q197. How should a design account for data privacy in observability?

Logs, traces, metrics, crash dumps, and audit events can copy sensitive data into systems with different access, retention, and residency policies from the primary application. Avoid capturing passwords, tokens, payment credentials, full request bodies, or unnecessary personal data by default. Prefer allowlisted fields and structured error codes over blanket payload logging.

Where identifiers are useful for correlation, consider stable pseudonymous IDs or hashes, but recognize that pseudonymous data may still be sensitive and that hashes can be reversible by enumeration for low-entropy values. Redaction should happen before data is exported to the telemetry backend, not only in the dashboard. Prevent secrets from appearing in exception messages, URLs, baggage, or high-cardinality labels.

Use role-based access, least privilege, encryption, audit logs, and retention limits for observability platforms. Separate operational logs from immutable business audit trails: debugging data may be sampled or deleted, while regulated audit evidence may need stronger controls and integrity guarantees. Cross-region telemetry must respect residency and legal restrictions even when production data itself remains in-region.

Review the privacy impact of tracing propagation and tenant-specific diagnostic tools. Sample or aggregate high-volume data while preserving sufficient incident evidence. In an interview, state what is captured, where it is stored, who can access it, how long it lives, and how an operator can debug a failure without exposing customer secrets.

### Q198. What is the difference between technical success and business success?

Technical success means an infrastructure or component operation completed according to its local contract—an HTTP response returned, a database transaction committed, a broker accepted a message, or a consumer acknowledged an event. Business success means the intended user outcome actually occurred correctly. These can diverge at distributed boundaries.

For example, an order API may return HTTP 202 because a durable job was accepted, but the order is not yet fulfilled. A payment request may time out after a provider charge succeeded, so the client sees an error although money moved. A consumer can acknowledge a malformed event after sending it to a DLQ, while the workflow remains incomplete. A green health check does not prove data correctness.

Define business-level states and SLIs alongside technical metrics. Track order completion, payment reconciliation, reservation conflicts, message delivery within deadline, and unresolved unknown outcomes. Link them with a stable operation ID and durable transition history so engineers can explain what happened. For async workflows, distinguish accepted, processing, succeeded, failed, expired, and manual-review states.

This distinction affects API semantics, monitoring, incident response, and compensation. Do not return “success” merely because a message was enqueued unless the API contract says that acceptance is the promised outcome. In an interview, clarify exactly what “success” means to the caller and how the system proves that result.

### Q199. How do you estimate cache hit ratio before implementation?

There is no reliable hit ratio from total request volume alone. It depends on the distribution of key popularity, the size of the working set, TTL/invalidation behavior, cache capacity, access locality, and whether the cache key includes high-cardinality dimensions such as user or tenant. A public product catalogue may have strong reuse; a personalized endpoint whose key includes a unique request ID may have almost none.

Start from access logs or a representative workload trace. Count requests per canonical cache key, estimate the popularity distribution, and simulate the chosen cache capacity and eviction policy. A rough upper bound can be estimated by asking what fraction of reads target the most popular keys that fit in memory, but measured/simulated workload is better than assuming uniform access. Test cold start, expiry bursts, and changing popularity after a campaign.

Calculate the weighted hit ratio only across requests eligible for caching, and distinguish hit ratio by request count from bytes served. A cache can have a high request hit ratio but still provide little bandwidth relief if large responses are misses. Also track stale-result rate, origin latency, evictions, memory, and refresh cost.

Finally, convert hit ratio into origin load. If the eligible origin-facing traffic would be 10,000 requests/s and the cache serves 90% of those requests, the origin would see about 1,000 requests/s at steady state, before refreshes and unrelated traffic. But a cache outage can send the full load to origin. Capacity-plan and protect that failure mode rather than treating expected hit ratio as a guaranteed safety margin.

### Q200. How do you calculate storage for a replicated database?

Separate logical data size from total physical footprint. Estimate records per day × average serialized row size × retention, then add actual database row/page overhead and index sizes. Include updates and deletes, write-ahead logs/binlogs, temporary space for vacuum/compaction/reindexing, and headroom for uneven growth. For an indexed transactional database, the index footprint can be a meaningful fraction of the table or larger depending on key width and number of indexes.

Then account for physical copies. If one primary dataset occupies 8 TB including its indexes and the topology stores three full copies (one primary plus two replicas), the data fleet needs roughly 24 TB before logs, snapshots, backups, and temporary migration copies. The replication factor may apply differently to logical shards or storage blocks; use the product's model rather than multiplying blindly.

Backups are additional capacity unless the product uses storage-efficient incremental snapshots. Point-in-time recovery logs can add substantial volume, especially for high write churn. Search indexes, object storage exports, and caches may be additional derived copies. Retention and regulatory deletion requirements can affect how long each copy persists.

Estimate growth and recovery too: a full restore must be able to stream data at a rate that meets RTO, while replication consumes network and disk bandwidth continuously. Use observed logical-to-physical storage ratios from realistic data and include a failure margin for free space. In an interview, show assumptions and identify which overheads are measured versus estimated.

### Q201. How do you estimate connection demand across a microservice fleet?

Estimate potential concurrent connections at the fleet level, then compare with every downstream limit. If 30 application instances each have a maximum database pool of 40, the application tier alone can open 1,200 database connections. Add background workers, scheduled jobs, migrations, monitoring, other services, and failover headroom. Autoscaling can multiply connections faster than the database can serve them.

Request rate alone does not determine connection demand. Concurrency depends on how long each operation holds a connection and whether work waits on locks or external calls while holding it. As a first approximation, Little's Law relates average in-flight work to arrival rate × average time in the system; use actual connection hold time, not end-to-end request time, when estimating pool usage.

Calculate budgets per database, shard, and connection class. A single service may have multiple pools to different dependencies; a fan-out request can occupy several pools at once. Connection pooling/proxying can reduce connection overhead but may introduce transaction/session constraints or a new central bottleneck. Reserve sessions for failover, admin work, and migrations rather than consuming every available connection in normal traffic.

Monitor pool wait, active/idle counts, hold duration, database session count, transaction age, and lock waits. Increasing pool size is not a universal fix; it can increase database contention. The design should cap downstream concurrency, use short acquisition timeouts, avoid holding connections across remote calls, and test reconnect storms after a database failover.

### Q202. How do you estimate the impact of a downstream fan-out ratio?

Model how many internal operations each incoming request creates. If an API receives 2,000 requests/s and 60% of them call three downstream services, those calls alone create (2{,}000 × 0.60 × 3 = 3{,}600) downstream attempts/s, before retries or background work. If some paths call a service more than once, count those calls explicitly rather than using a vague “fan-out factor.”

The average fan-out ratio is not enough if a small request class triggers expensive fan-out or a tenant causes skew. Break down by endpoint and dependency, and include retries, cache misses, and optional enrichments. Parallel calls reduce serial latency but increase peak concurrency and cause the response to depend on the slowest required dependency. If all downstreams must succeed, end-to-end availability falls as required dependencies are added.

Use traces to validate actual call graphs and call counts under representative load. Apply per-dependency concurrency limits, deadlines, and bulkheads; separate optional calls so they can be skipped under pressure. Precompute or materialize repeated compositions when they are reused often enough to justify freshness and rebuild complexity.

During overload, fan-out can amplify traffic exactly when a downstream is least able to handle it. Budget downstream QPS and concurrency for peak, retry, cold-cache, and failover scenarios. In an interview, show the request-to-dependency arithmetic, name the likely bottleneck, and explain how you prevent retries or one slow dependency from consuming all capacity.

### Q203. How do you calculate a service's error budget from a request-based SLO?

For a request-based SLO, the error budget is the allowed fraction of eligible requests that may fail during the measurement window. If the target is 99.9% success across 20 million eligible requests, the permitted failure fraction is 0.1%, so the budget is 20,000 failed eligible requests. If 5,000 have already failed, 15,000 remain for that window, assuming the definition and denominator remain fixed.

Define success precisely. A network response may be HTTP 200 while the business operation failed, so the SLI should reflect the user-facing contract where possible. Define whether client errors, cancelled requests, internal retries, synthetic probes, or requests rejected before admission count. Do not change exclusions during an incident to improve the reported result.

Request-based and time-based availability differ. A service can be unavailable for a short period during a high-traffic window and consume a large request-based budget; a low-traffic outage may consume little of that budget but still be severe for affected users. For multi-step workflows, consider separate journey-level indicators.

Use burn rate to detect the speed of budget consumption, with alert windows that balance urgency and noise. Pair availability with latency, correctness, and asynchronous completion SLIs. In an interview, calculate the budget, state what is counted, and explain what operational action follows when the budget burns too fast.

### Q204. How do you estimate object-storage cost at a high level?

Model storage cost as a set of separate dimensions: bytes stored by storage class and region, request counts (PUT/GET/LIST and multipart operations), data retrieval/restore charges, egress, replication, lifecycle transitions, and any minimum storage-duration or small-object overhead. The exact billing model varies by provider and tier, so use current official pricing for the intended region and storage class once the workload is known.

Estimate object count and byte volume over time. If 2 million files/day average 4 MB and are retained for 90 days, new logical data is about 8 TB/day and steady retained data can approach 720 TB before compression, versioning, replication, and deletion lag. That example may be unrealistic for a particular workload, but it shows why the upload rate and retention duration matter as much as API request rate.

Consider lifecycle policies: hot objects may need low-latency access, older data may move to a cheaper tier, and archival tiers may have retrieval delay and fee implications. Versioning and soft deletion can preserve recoverability but increase retained storage. Cross-region replication improves resilience or residency options but multiplies storage and transfer costs.

Large uploads should usually use direct or pre-signed uploads so application servers do not proxy every byte. Track incomplete multipart uploads and configure safe cleanup; otherwise abandoned uploads accrue cost. In an interview, enumerate the cost dimensions, model retention and access frequency, and explain how the chosen tier and replication policy satisfy performance, compliance, and recovery requirements.

### Q205. How do you estimate how much capacity a cache outage removes?

Start from the origin workload that exists with the cache healthy. If the application receives 10,000 eligible reads/s and the cache hit ratio is 90%, the steady-state origin load attributable to those reads is about 1,000 misses/s (plus refreshes and other traffic). If the cache becomes unavailable, origin load can jump toward 10,000 reads/s—a tenfold increase for that request class.

That simple estimate assumes the measured hit ratio applies to the outage workload. It may be optimistic: cache failures can create concurrent stampedes, clients may retry failed cache calls, and hot keys may dominate origin capacity. A warm local cache or stale-while-revalidate policy may reduce the sudden shift, but only where freshness allows. If a cache stores correctness-critical coordination state, bypass behavior may be unsafe.

Capacity-test the origin at the cold-cache rate, including realistic query plans, data volume, pool sizes, and writes that still occur. Add request coalescing, bounded origin concurrency, circuit breaking, and load shedding so the entire fleet does not overwhelm the database. Stagger cache warm-up and autoscaling to avoid everyone loading the same keys simultaneously.

Monitor cache availability, miss rate, origin QPS, database saturation, connection wait, and p99. In an interview, quantify the expected traffic multiplier and state what protects the origin during cache failure, instead of assuming the cache is always available.

### Q206. How do you estimate the bandwidth needed for replication?

Estimate the volume of changes that must be shipped, not just the final row size. If the database generates 40 MB/s of write-log/change data on average, one remote replica needs roughly that throughput before protocol overhead, compression effects, metadata, and bursts. Two full remote copies can require approximately twice that replication traffic, though the actual pattern depends on topology and whether replicas cascade through other nodes.

Average bandwidth is not sufficient. Replication must catch up after network interruption or maintenance. If 2 TB of change data accumulates during an outage and the link can deliver 200 MB/s while new changes continue at 40 MB/s, the net catch-up rate is about 160 MB/s; ideal catch-up takes roughly 12,500 seconds, or 3.5 hours. Actual time may be longer due to apply/flush bottlenecks and variance.

Measure log generation rate, replica apply rate, network throughput, lag in bytes/time, and write amplification/compaction. Synchronous replication places network and remote durability latency on the commit path; asynchronous replication decouples commit latency but permits lag and a defined risk of losing recent writes on failover. Cross-region links can also carry snapshots, rebuilds, backups, and application traffic.

In an interview, connect bandwidth to RPO, failover eligibility, and recovery time. Ensure the replica can apply changes as fast as they are generated during normal conditions and faster than generation during catch-up. Validate the model using real write patterns, compression, and a planned link outage.

### Q207. How do you estimate partition count for an event stream such as Kafka?

Partition count affects parallelism, ordering, throughput, and operational cost. It is not just a throughput knob. Within a consumer group, a partition is assigned to one consumer at a time, so the number of partitions places an upper bound on useful consumer parallelism for that group. Extra consumers beyond the partition count generally sit idle for that topic, although a consumer can own multiple partitions.

Estimate partition count from at least three constraints:

- **Throughput:** measure the per-partition producer and consumer rate for your payload size, batching, compression, broker storage/network, and actual processing cost. For target throughput (T) and safe per-partition throughput (t), a starting point is (N geq T/t), then add measured headroom.
- **Parallelism and latency:** estimate the number of independent consumers needed to keep up with peak arrival rate. If each event takes 20 ms of CPU-bound work, the effective processing rate per worker is very different from a worker that mostly waits on I/O.
- **Ordering and key skew:** ordering is guaranteed within a partition, not across the topic as a whole. Events that must be processed in order for one account or entity should consistently map to the same partition. If one key produces most of the traffic, adding partitions does not split that key unless the data model changes.

Partitions carry costs: broker metadata, file handles/log segments, leader placement, replication traffic, rebalances, and recovery work. Adding partitions may also change key-to-partition mapping for some producer partitioners, which can affect ordering assumptions for historical and new events. Treat partition-count changes as a design decision, not a harmless toggle.

For example, if load testing shows each partition can safely sustain 2,000 events/s for the real payload and consumer logic, a 40,000 events/s target suggests at least 20 partitions on throughput alone. If you need 32 active consumer workers and strict per-key order, you may choose more partitions—but validate the actual broker and consumer limits, hot-key distribution, replication factor, and failure headroom before finalizing.

### Q208. How do you estimate whether a rate limiter can sustain traffic?

Estimate the number of limit checks per second, not just incoming HTTP requests. A request may be checked at the IP, user, tenant, route, and provider-account levels, leading to multiple lookups or atomic updates. If 50,000 requests/s each require three globally coordinated checks, the limiter may need to sustain 150,000 checks/s plus bursts and retries.

Measure the algorithm's operation cost, key cardinality, hot-key distribution, store latency, and consistency needs. A single popular account's counter can be a hot key even when total traffic is distributed. Fixed/sliding windows, token buckets, and concurrency-based admission controls have different memory, accuracy, and burst behavior. Local counters are fast but may overshoot the aggregate quota when traffic moves across instances; centralized atomic counters are more accurate but add latency and can become a bottleneck.

Consider hierarchical allocation: a global controller grants bounded token budgets to local instances, allowing most requests to be decided locally while limiting overshoot. This can reduce coordination but may leave unused capacity stranded on one instance or allow a defined temporary variance. For strict provider quotas, allocate centrally at the scope the provider enforces.

Test the limiter during autoscaling, region loss, store failover, and adversarial key cardinality. Define fail-open/closed behavior, monitoring, and what the client receives when limited. In an interview, explain required check rate, per-key skew, accuracy-versus-latency trade-off, and how the limiter's own failure does not bring down the service.

### Q209. When should large files bypass application servers?

For large uploads/downloads, the application server should usually authorize the operation and manage metadata, while the client transfers file bytes directly to object storage or a dedicated transfer service. Proxying every byte through application instances consumes bandwidth, connections, memory buffers, CPU, and autoscaling capacity that would be better used for business logic.

A common upload flow is: the client requests an upload session; the application authenticates the caller, checks tenant quota and allowed content; it creates a scoped pre-signed URL or multipart-upload capability; the client transfers directly to object storage; then the application validates final size/checksum and marks the object ready. Keep the capability short-lived and restricted to a specific object, operation, and permitted headers. Do not expose broad bucket credentials.

Direct upload does not remove security requirements. Validate file size/type, scan for malware, treat filenames and metadata as untrusted, and avoid serving active content from a privileged application origin. For private downloads, use authorization before issuing a short-lived signed link or proxy through an access-controlled path. Large-object downloads should support range requests where the client/use case benefits.

Handle incomplete multipart uploads, duplicate finalize requests, client retries, checksums, quotas, encryption, retention, and object deletion. An object may be uploaded but not yet approved/available to downstream workflows; represent that state explicitly. In an interview, separate control-plane operations (authorize, create, complete, validate) from the data plane (transfer bytes) and discuss security, cost, and failure recovery.

### Q210. How do you design resumable uploads?

A resumable upload divides a large object into independently transferred parts so the client can retry failed parts rather than restarting the whole file. The server or object store maintains an upload session with an ID, destination, expected size, part list/checksums, expiry, and state. Multipart object-storage APIs are commonly used, but their exact guarantees and limits are provider-specific.

The API should create the session idempotently, authorize every part/finalize operation, and enforce maximum object size, part count, content type, and tenant quota. Each part should have a checksum or integrity validation. The client can query which parts were accepted after a connection loss and retry missing or failed parts. Finalization must be idempotent so two concurrent completion requests do not publish the object twice or expose a partially assembled file.

Treat session IDs and pre-signed URLs as capabilities; keep them scoped and short-lived, and avoid leaking them in logs. A session may be abandoned, so clean up incomplete multipart uploads after a safe expiry. For security, validate the completed object, scan it if necessary, and keep it in a quarantined or pending state until validation succeeds.

Account for duplicate callbacks, object-store temporary failures, checksum mismatch, and a client that attempts to finalize a different size/content. Use an operation state machine and background reconciliation for uploads whose completion status is uncertain. In an interview, discuss resumability, idempotent finalize, resource-abuse protection, data integrity, and cleanup—not just dividing a file into chunks.

### Q211. How do you choose between polling and push notifications for status updates?

Choose based on how quickly the client needs updates, how many clients are concurrently connected, and whether updates must be delivered while the client is offline. **Polling** is simple: the client requests current status at an interval. It works well for low-frequency changes and simple infrastructure, but creates repeated requests even when nothing changes. If every client polls every second, the steady request rate can be large and polling timers can synchronize into periodic spikes.

**Push** mechanisms such as Server-Sent Events (SSE), WebSockets, or mobile push notifications reduce repeated checks. SSE is useful for server-to-browser updates over an HTTP stream; WebSockets support bidirectional, interactive communication; mobile push is useful for waking or alerting an offline device but is not a reliable substitute for application state. Persistent connections need heartbeats, reconnect handling, connection distribution, authentication refresh, and fan-out capacity.

A common design is hybrid: the client submits a job, receives an operation ID, and polls a status endpoint with backoff until it completes; clients that need near-real-time updates subscribe to push events, then fetch durable status after reconnect. Persist status transitions and event IDs so a dropped push does not lose the result. Support ETags/conditional requests or long polling to avoid returning unchanged payloads repeatedly.

Do not assume a transport provides durable delivery. Define reconnect cursors, event retention, offline behavior, authorization, and the source of truth. Pick polling when frequency is low and simplicity matters; use push when the latency/cost benefit justifies connection management. Measure active connections, polling QPS, update latency, reconnect storms, and completion freshness.

### Q212. How do you design a long-running operation API?

Do not keep an HTTP request open while work may take minutes or hours. Accept the command durably, create an operation resource with a stable ID, and return a response such as HTTP 202 Accepted with a link to query its status. Acceptance means the system has recorded responsibility for the operation; it must not imply that the final business action has completed.

Represent the operation as a durable state machine—for example, QUEUED, RUNNING, SUCCEEDED, FAILED, CANCEL_REQUESTED, CANCELLED, EXPIRED, or NEEDS_REVIEW. Persist progress when useful, the initiating principal/tenant, input reference, timestamps, deadline, retry count, result reference, and failure reason. Use a database transaction/outbox or another durable handoff so the API does not record “accepted” and then lose the job before enqueueing it.

Make the create request idempotent. If a client times out after acceptance, retrying with the same key should return the same operation instead of creating duplicate exports or payments. Provide polling with backoff, SSE/WebSocket/webhook updates where justified, and authorization checks on every operation lookup. Large results should use a controlled download URL with explicit expiry and access scope.

Define cancellation precisely. It may stop not-yet-started work, request cancellation at a safe checkpoint, or be impossible once an irreversible external action has begun. Include operation retention and expiry, DLQ/reconciliation, progress semantics, and how clients learn about a final unknown result. Monitor completion latency and oldest pending age, not just API acceptance latency.

### Q213. How should cancellation work in a distributed workflow?

Cancellation is a business command that races with execution; it is not a guarantee that all work has instantly stopped. Persist the cancellation request and let the workflow state machine decide which transitions remain legal. Workers should check cancellation at safe boundaries, such as before starting a new stage or before committing a reversible local change.

Define the race between cancellation and completion. If cancellation arrives before a payment is submitted, the workflow may stop cleanly. If the provider has already charged the customer, cancellation may require a refund or void rather than erasing the charge. Some steps may be irreversible or have an uncertain outcome after a timeout. The API should then return an explicit outcome such as cancellation requested, cancelled, completed, or too late to cancel—not a misleading universal “cancelled” result.

Propagate cancellation to queued and active downstream calls where supported, but do not rely on network cancellation to roll back a committed database transaction or external side effect. Use durable operation IDs, idempotent cancellation requests, compensating actions, and reconciliation for uncertain steps. A cancellation can also expire while a worker is paused, so workers need version-checked state transitions.

For batch operations, define partial cancellation: which items completed, which were stopped, and which require repair. Record an audit trail and ensure the workflow reaches a terminal state eventually. In an interview, draw the cancellation path alongside the happy path, identify irreversible boundaries, and explain what users see when cancellation races with final completion.

### Q214. What is the difference between a timeout, a deadline, and a lease?

A **timeout** bounds how long one operation or wait may continue—for example, acquiring a database connection or waiting for a downstream response. A **deadline** is the absolute time by which a complete request or workflow is expected to finish. A **lease** grants temporary ownership or permission until it expires unless renewed. They are used for different problems and should not be substituted for one another.

Suppose an API has a one-second end-to-end deadline. If it spends 150 ms in authentication and 200 ms querying the database, a downstream call should receive only the remaining budget, with time reserved to commit and return the response. Giving every hop an independent one-second timeout can cause the overall request to run several seconds. Retries consume that same budget.

A lease is about ownership, not merely latency. A worker may acquire a 30-second lease, pause for 45 seconds, then resume after another worker has acquired the lease. The old worker may still execute unless the protected resource validates an epoch/fencing token or conditional transition. Expiry does not forcibly terminate a process.

Use monotonic clocks for local elapsed-time measurement; absolute wall-clock deadlines across hosts require careful propagation and clock assumptions. Define what happens when a deadline expires after a side effect may have occurred. In an interview, explain the time budget, the ownership contract, and what prevents an expired owner from performing stale writes.

### Q215. How should timeouts be chosen for downstream calls?

Choose timeouts from the caller's end-to-end latency objective, measured downstream latency distributions, network conditions, and the cost of a false timeout. A timeout that is too short rejects healthy but slower requests and triggers retries; one that is too long ties up threads, connections, memory, and queued work while the caller waits. Avoid selecting a value only because it “feels safe.”

Measure each stage: DNS/resolution, connection-pool acquisition, connection/TLS setup, request transfer, server processing, and response transfer. Client libraries differ in which stages a configured timeout covers. A connection timeout, read timeout, and whole-call deadline may be separate settings. Reuse connections where appropriate, but still bound how long an individual operation may wait.

Propagate an end-to-end deadline or remaining budget to downstream calls. A service should not start a retry when too little time remains for the retry to succeed and return the result. For asynchronous work, use separate task deadlines and retry schedules; do not accidentally inherit a short HTTP deadline for an hours-long background workflow.

Use timeouts with bounded retries, jitter, concurrency limits, and idempotency for mutations. Tune from p95/p99 data and acceptable false-timeout rates, then validate under load and partial failure. A timeout is not cancellation of a remote effect: a timed-out payment call may still succeed. See [AWS guidance on timeouts, retries, and jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).

### Q216. How do you decide whether a fallback is safe?

Write down what result the fallback returns and the invariant it must preserve. Fallback is safe only when that result remains within the product's correctness and security contract. Serving a cached product description may be acceptable; serving stale authorization, account balance, or final inventory availability may not be safe for a write decision.

Separate optional dependencies from authoritative ones. If a recommendation service fails, the API can omit recommendations and mark the response partial. If a payment provider times out, the system cannot claim payment failed or succeeded without evidence; it should preserve an unknown/pending state and query or reconcile. An alternate provider may have different semantics, quotas, supported currencies, or idempotency behavior, so failover does not mean transparently sending the same command anywhere.

Define the fallback's freshness limit, activation rule, capacity, security checks, and recovery procedure. Cached fallback data can become dangerously stale during a long outage. A fallback that depends on another shared service may fail at the same time, and switching every request to a secondary provider can overwhelm that provider. Preserve audit events and stable operation identities.

Test both partial and prolonged dependency failures, including the return to normal operation. Measure fallback frequency and business outcomes. In interviews, distinguish “technically available” from “correct”: a controlled error is better than a plausible response that violates the invariant.

### Q217. How do you avoid cascading failure across a dependency graph?

A cascading failure occurs when a component's slowdown causes callers to accumulate work, time out, retry, and overload neighboring components until the failure spreads. Draw the dependency graph and identify shared bottlenecks: databases, identity providers, rate-limit stores, brokers, DNS/control planes, and external quotas. Multiple services may look independent while all depend on the same constrained resource.

Bound work at each layer. Use end-to-end deadlines, short and bounded queues, per-dependency concurrency limits, bulkheads, and admission control. Shed optional traffic before critical pools are exhausted. Circuit breakers can stop calls to a failing dependency, but must be paired with a safe fallback or explicit failure behavior. Retries should be limited, jittered, idempotent where necessary, and owned by a clear layer so policies do not multiply.

Scale based on sustainable throughput and downstream capacity. Adding application replicas can multiply connection pools and provider calls. Cache failures can create origin stampedes. When a dependency recovers, ramp traffic and queued work gradually instead of releasing every retry at once. Preserve capacity for health, control, and recovery operations.

Monitor queue age, successful throughput, retry amplification, connection wait, dependency p99, saturation, and SLO burn. Design for the one-zone-down or cache-cold case, not just normal steady state. In an interview, explain the feedback loop, the first resource that saturates, and the mechanism that interrupts the loop without compromising business correctness.

### Q218. What is a retry budget, and how is it different from a per-request attempt limit?

A per-request attempt limit bounds how many times one logical request may be attempted. A **retry budget** limits aggregate retry-generated work over a time window or relative to original traffic, protecting the system during broad failures. Both are needed: a request might have only two retries while millions of requests retry simultaneously.

For example, a client may allow at most two retries per call, while a service-wide token bucket allows retries only up to a bounded fraction of recent original requests. Retry tokens are consumed by extra attempts and replenished according to a defined rate. Once the budget is exhausted, the system should fail fast or defer eligible asynchronous work rather than keep amplifying pressure.

Budgets should be applied at the scope of the bottleneck—per dependency, region, tenant, or service. A global budget may let one noisy tenant consume all retry allowance; a per-instance budget can multiply with autoscaling. Choose a policy that matches the dependency's actual quota and failure domain.

A retry budget does not make a non-idempotent mutation safe, fix a bad timeout, or guarantee the dependency is healthy. Combine it with deadlines, backoff with jitter, error classification, circuit breaking, and concurrency limits. Track original logical requests separately from attempts, plus retry success rate, budget exhaustion, and downstream saturation. Explain what happens when the budget runs out so the mechanism does not become an undocumented source of dropped work.

### Q219. How do you choose the right level of redundancy?

Choose redundancy from the failure model, required availability, RTO/RPO, geographic needs, and budget. Multiple application instances can protect against process/host failures; deployment across availability zones can tolerate some zone failures; cross-region replicas can support regional recovery. But every extra copy adds cost, coordination, and operational work, and only protects against failures independent of the other copies.

Ask what is replicated and how it behaves. Stateless compute is relatively easy to duplicate. Databases need a replication and promotion protocol; message brokers need a durability/replica acknowledgement policy; caches may lose recent state on failover; search indexes may be rebuildable from source. Replicas sharing a single network, credential provider, storage system, or broken release remain correlated.

Plan capacity during loss of a failure domain. Two regions that each use 90% of capacity normally cannot necessarily carry all traffic when one region fails. The surviving region may need reserved headroom, priority shedding, or a reduced service contract. Redundancy also needs tested failover/failback, restore procedures, and fencing to prevent split brain.

Use reliability modeling as a first estimate, but include common-cause failures and operational mistakes. In an interview, state which failures are tolerated, what data can be lost, how long recovery takes, how the surviving tier handles the load, and whether the team can afford to maintain and test the added complexity.

### Q220. How do you design graceful shutdown for a worker or service?

Graceful shutdown gives the system a controlled transition from serving work to stopping, but it must not rely on the process having unlimited time to finish. On receiving a termination signal, stop accepting new traffic, mark the instance unready, drain in-flight requests within a defined deadline, and close dependencies cleanly. The platform's termination grace period must be long enough for the intended drain behavior but still bounded.

For HTTP/RPC servers, stop new requests and allow safe in-flight calls to finish. Long-lived WebSockets may need a reconnect signal and a maximum drain duration. For message consumers, stop fetching new records, finish or safely abandon in-flight work, and commit acknowledgements/offsets only after the processing side effect is durable. If shutdown is forced, assume the message will be redelivered.

For workers using leases, stop renewing ownership when appropriate and release or hand off safely; do not let a second worker act before the first can no longer commit. Ensure shutdown does not wait forever on a stuck dependency. Use deadlines, cancellation tokens, and idempotent side effects. A service should remain able to receive termination signals even when its main worker pool is saturated.

Test rolling deployment, node loss, forced kill, and slow downstream behavior. Watch rejected requests, in-flight duration, duplicate work, and termination timeout. In an interview, describe readiness/draining order, message acknowledgement, external side-effect safety, and what the recovery mechanism does if graceful shutdown is interrupted.

### Q221. How should a system handle duplicate job scheduling?

Assume the scheduler can trigger a job more than once. A scheduler may crash after enqueuing a job but before recording that it did so; a lease may expire while the worker is paused; a retry can arrive after the first execution succeeded. “Only one scheduler instance” does not remove all duplicate windows.

Give each logical occurrence a deterministic identity, such as job ID plus scheduled instant or business period, and enforce uniqueness durably. Claim a job with a conditional state transition or lease and record its status/progress. The worker should be idempotent at the business-effect boundary; a dedupe row alone cannot atomically cover an unrelated email, payment provider, or third-party action.

Define misfire policy: should missed runs be skipped, coalesced into one run, or caught up individually? For periodic jobs, blindly replaying every missed interval after a long outage can create an overload burst and may execute work whose deadline has passed. Keep a bounded catch-up rate and preserve per-entity ordering where required.

Monitor schedule lag, duplicate occurrence attempts, execution duration, lease expirations, and oldest pending job. Recovery tools should allow safe re-drive with the original job identity. In an interview, explain the crash window between schedule creation, queue publication, job execution, and final status recording—and how the system remains correct through each retry.

### Q222. How do you prevent stale leaders from committing writes?

A stale leader is a process that once owned a resource but has lost authority and may still be running after a pause, network partition, or delayed shutdown. A lease or health check alone cannot force that process to stop. The protected resource must be able to reject stale writes.

Use a monotonically increasing **epoch** or fencing token assigned when leadership changes. The leader includes the token in every protected write, and the database/resource records the highest accepted token and rejects older epochs. If the resource supports compare-and-set, conditional updates, or term/version checks, enforce the ownership rule there. A token that is generated but never validated by the resource provides no protection.

Leadership transfer must be coordinated: establish the new epoch, ensure the new writer has the data it needs, fence the previous writer, and only then route authoritative writes. For external side effects, the external provider may not understand your fencing token; use provider idempotency, a single durable command owner, or reconciliation to reduce duplicate effects.

Test a process that pauses past lease expiry and then resumes, a region that loses control-plane connectivity but can still reach the database, and a slow old leader during promotion. In an interview, name the resource that checks fencing, the atomicity of that check with the write, and how you avoid a window where old and new leaders can both commit.

### Q223. How do you choose a data consistency guarantee for each API?

Choose the guarantee from the operation's business meaning and failure cost. “Consistency” is too vague without a named property. Options may include linearizable reads/writes, read-your-writes, monotonic reads, bounded staleness, or eventual convergence. These guarantees differ in what a caller can observe and what the system must coordinate.

For example, a product browsing endpoint may tolerate slightly stale descriptive data; a user's immediate profile update may need read-your-writes; a balance or inventory reservation may require an authoritative transactional operation. Even a strongly consistent read is not sufficient if the subsequent mutation is not protected atomically—checking that a seat is available and then inserting a booking still races unless the write enforces the invariant.

Write the contract per operation: which source is authoritative, whether the result can be stale, how much, and what happens during replica lag or regional partition. Choose cache and replica routing accordingly. If the requirement cannot be met in a failure state, return a controlled error or pending status rather than silently weakening the promise.

A single system can use several consistency levels for different endpoints. Avoid describing an entire database as simply “strong” or “eventual” without checking its actual API and configuration. In an interview, connect the guarantee to the data model, transaction boundary, latency, availability, and user-visible behavior.

### Q224. What does bounded staleness mean?

Bounded staleness promises that a read is no more than a specified amount behind authoritative state, under the system's documented assumptions. The bound may be expressed as elapsed time (for example, at most five seconds behind) or as a number of versions/operations. It is stronger than eventual consistency, which promises convergence if updates stop but gives no maximum time for convergence.

A practical design must define what is being measured: source commit to replica visibility, event publication to projection update, or cache age. If source and consumer clocks differ, wall-clock timestamps can mislead; a replication log position or monotonic entity version may be a better basis when available. The promise must also specify what happens if the system cannot currently meet the bound.

A cache TTL does not automatically guarantee bounded staleness. Invalidation may be delayed, a stale reader may refill the cache after a write, and a replica can already be behind before the value is cached. If the system cannot prove the bound, describe it as a target or best-effort behavior rather than a guarantee.

Bounded staleness can suit dashboards, catalogues, and some status views. It is usually insufficient for authorizing a financial write or reserving the final inventory unit. When lag exceeds the bound, the system might wait within the deadline, route to an authoritative source, return an explicit stale/pending state, or reject the operation. Choose based on the user contract, not simply on the database's advertised consistency mode.

### Q225. How do you prevent lost updates in a read-modify-write flow?

A lost update occurs when two requests read the same value, compute independent changes, and one overwrites the other. If a counter starts at 10, both requests read 10 and each writes 11, the final value becomes 11 rather than 12. The fix is to make the intended update atomic or detect that the value has changed since it was read.

Prefer a database operation that expresses the change directly, such as an atomic increment or conditional decrement. For more complex changes, use a transaction, row lock, unique constraint, or optimistic compare-and-set with a version column. A versioned update succeeds only if the stored version still matches the version read by the caller; on conflict, reread and make an explicit retry/merge decision.

The right method depends on contention and invariant scope. Optimistic control works when conflicts are relatively uncommon; pessimistic locking can be appropriate for hot records but may block and deadlock. Multi-row rules may require serializable isolation or another explicit mechanism; a single row version does not automatically protect a cross-row invariant.

Do not hold a database lock while calling a remote provider, and do not place an external side effect inside a transaction that may be retried unless that effect is idempotent. In an interview, demonstrate a concurrent interleaving and show which atomic mechanism prevents the incorrect final state, including what happens when the competing update wins.

### Q226. How do you design inventory for oversell-sensitive products?

The authoritative inventory store must enforce a condition like “available quantity never becomes negative” at the write boundary. A safe pattern is an atomic conditional update that decrements stock only when sufficient quantity remains, checking the affected-row count. A prior read of availability is not sufficient because concurrent requests can observe the same stock.

For scarce inventory, model explicit reservations/holds with unique IDs, ownership, expiry, and state transitions. Decide when a hold becomes committed, how it interacts with payment authorization, and what happens if payment succeeds after the hold expires. Release and confirmation must be idempotent, and expiry correctness cannot depend solely on a cleanup job running at the exact time.

A cache or search index can show approximate availability for browsing, but the authoritative reservation must revalidate current state. Partitioning by SKU or event can localize updates, yet one highly popular item can become a hot key. Inventory buckets or preallocated stock slices can increase throughput, but allocation and reconciliation must prove that aggregate available stock never exceeds real inventory.

If the business explicitly allows overselling or backorders, encode that as a deliberate policy instead of accidentally allowing negative stock. Monitor reservation conflicts, expired holds, stock reconciliation, and queue delay. In an interview, state the invariant, atomic enforcement mechanism, hot-key strategy, and failure behavior across the inventory/payment boundary.

### Q227. How do you model temporary holds and expiry safely?

Represent a hold as a durable record containing a stable hold ID, resource ID, owner/order ID, expiry instant, and state such as ACTIVE, CONFIRMED, RELEASED, or EXPIRED. Enforce state transitions atomically. A hold should be confirmable only while it remains valid and has not already been released or consumed.

Do not rely on a background job deleting expired rows precisely on time. Schedulers can lag or run twice. The confirmation command should check current state and expiry in the same transaction or conditional update that consumes the hold. The cleanup worker can then release inventory idempotently for expired holds. Use the datastore's time semantics where practical and handle clock skew explicitly.

Expiry races with external actions. Payment may complete just as the hold expires; the workflow must decide whether to revalidate inventory, retry reservation, refund/void the payment, or send the order for review. Do not assume the expiry transition and external provider action are one atomic transaction. Use stable operation IDs and reconciliation to resolve uncertain outcomes.

Hold IDs and owner IDs should be unique under the intended business scope; duplicate client requests must not create multiple holds. Measure active holds, expired-but-not-released records, confirmation conflicts, and oldest cleanup backlog. In interviews, explain the exact transition that wins when confirmation and expiry race, what the user sees, and how a stuck or duplicate cleanup job remains safe.

### Q228. How do you keep a payment ledger correct across retries and reconciliation?

A ledger should preserve a durable, auditable history of financial movements rather than overwriting history to make the latest status look simple. Model stable IDs for the logical payment, provider attempt, authorization, capture, refund, reversal, and settlement reference as required by the domain. Enforce uniqueness and legal state transitions transactionally.

Treat provider calls as uncertain distributed operations. A timeout can occur after the provider processed a charge; a webhook can be duplicated or delayed; a settlement file may arrive later than an API response. Use idempotency keys within the provider's documented scope and retention, store provider references, and record each state transition. Do not mark an operation definitively failed merely because an HTTP call timed out.

Reconciliation compares internal records with provider status or settlement data using merchant/reference IDs, currency, amount, fees, and expected state. Mismatches need classification: missing callback, duplicate request, amount mismatch, delayed settlement, refund discrepancy, or an unknown external effect. Safe cases can be repaired automatically with an auditable action; ambiguous/high-impact cases should be held for review rather than “last write wins.”

Keep accounting entries balanced and immutable where the domain requires it; model refunds/reversals as new entries instead of silently editing a prior ledger event. Protect reconciliation tooling with access control, rate limits, and audit trails. In an interview, explain how the ledger handles retries, how you find an unknown outcome, and how corrections preserve the evidence of what originally happened.

### Q229. How do you decide whether to use a saga or a single database transaction?

If all required state changes live in one transactional datastore and the transaction can be short, a local ACID transaction is usually simpler and easier to reason about. It can enforce invariants atomically using constraints, conditional updates, and appropriate isolation. Splitting a transaction into remote calls merely because the code has several services or classes adds failure modes without necessarily providing a benefit.

Use a **saga** when the workflow spans independently owned data stores or external systems and cannot share one atomic transaction. A saga records progress across local transactions and defines compensation or recovery after failure. Intermediate states are observable; compensation is a new business action and may fail or be impossible. The design therefore needs durable workflow state, idempotent steps, deadlines, retry policy, and reconciliation.

Example: reserving inventory and creating an order in the same database may fit a local transaction if ownership and scaling allow it. Charging a payment provider, booking a shipment, and reserving a stock item across distinct systems usually require a workflow that handles partial completion. Avoid a distributed transaction unless the participants and operational constraints genuinely support it and the coordination cost is justified.

Ask which invariant must be atomic and where it can be enforced. If the system can make the invariant local by adjusting ownership boundaries, a simpler transaction may be preferable. In an interview, compare the failure windows, user-visible intermediate states, operational tooling, and compensation semantics rather than saying “sagas are for microservices.”

### Q230. What is a transactional inbox pattern?

A transactional inbox lets a consumer safely handle at-least-once delivery by recording that a message was received and applying the local database effect in the same transaction. The inbox usually stores a stable message/event ID and may also record consumer identity, source, timestamp, or processing status. A uniqueness constraint prevents the same event ID from applying its database effect twice after redelivery.

The important crash window is between the side effect and acknowledgement. If the consumer commits its database change and crashes before acknowledging the broker, the message is delivered again. Because the inbox record and business update committed together, the redelivery sees the existing event ID and avoids applying the local mutation again. If the transaction rolls back, neither the inbox marker nor the local effect is committed, so the consumer can retry.

The inbox only covers effects inside that local atomic boundary. If processing also sends an email or calls a payment provider, the inbox transaction cannot make the remote effect atomic. Use an outbox for outgoing messages, provider idempotency keys, or reconciliation for unknown outcomes. Also define dedupe key scope: two event IDs may represent the same business action, while one event ID may be reused across different sources if producers are poorly specified.

Retention must cover the realistic replay/redelivery window; pruning inbox rows too early can permit duplicates during old-event replay. In an interview, draw the process → local transaction (inbox + state) → acknowledgement sequence and explain each crash point.

### Q231. How do you handle schema changes when old and new application versions run simultaneously?

Assume a rolling deployment will run old and new code at the same time, while background workers, delayed messages, analytics jobs, and rollback versions may continue using old assumptions. A schema migration that works for the newest binary can still break these callers.

Use expand–migrate–contract. First add a backward-compatible column/table/index without removing the old representation. Deploy code that can read both old and new forms and, if necessary, write both in a controlled manner. Backfill existing rows in bounded batches while capturing concurrent changes. Validate counts and business invariants, switch reads to the new representation, observe it under production traffic, and only then remove old fields in a later release.

Mixed-version writes require care. Dual-writing two fields in one local transaction can preserve local consistency, but it adds write cost and demands a clear authority rule. Two services writing separate stores is a dual-write problem and needs an outbox/CDC, workflow coordination, or reconciliation. Do not allow the new version to write data the old version cannot interpret until rollback is no longer required.

Test rollback with data written by the new code, queued events, cached schemas, and scheduled jobs. The compatibility window should be explicit: which code versions and payload versions must continue to work, and for how long. In an interview, describe the phases, the safe rollback point, and how you prove the backfill is complete despite concurrent writes.

### Q232. How do you prevent cache poisoning or cache-key confusion?

Cache poisoning occurs when untrusted request input causes a cache to store a response that is incorrect or unsafe for later users. Cache-key confusion occurs when two requests that should produce different responses map to the same key, or equivalent requests map inconsistently and bypass intended controls. These are correctness and security issues, not merely cache-efficiency concerns.

Build keys from canonical, validated inputs and every dimension that changes the representation: tenant, authorization scope, locale, content encoding, query parameters, API version, and feature cohort where relevant. Be careful with request headers that influence content; a CDN must honor the correct cache variation and cache-control rules. A shared cache should not store a personalized or privileged response unless the cache partition and policy are designed to make that safe.

Validate hostnames, forwarded headers, URL normalization, encodings, and redirects before caching. Do not trust arbitrary client-supplied identity headers. Ensure cache invalidation endpoints are authenticated and properly scoped so an attacker cannot poison or purge other tenants' entries. Apply size limits and avoid caching error responses or redirects unless their behavior is intentional.

Test with adversarial combinations: same URL under different tenants, languages, authorization states, query orderings, and content types. Verify that responses never cross trust boundaries. Monitor suspicious cache-key cardinality, unexpected cache hits for private data, and mismatches between origin and cached responses. In interviews, explain both the key construction and the cache-control contract across browser, CDN, gateway, and application cache layers.

### Q233. How do you decide between local cache and distributed cache?

A **local cache** is held inside each application process. Lookups are very fast and need no network call, but each instance has its own copy and can see a different value until expiry or invalidation. A **distributed cache** is shared by many instances and can hold a larger common working set, but adds network latency, serialization, capacity planning, key-hotspot risk, and an outage mode. Neither one is automatically more consistent.

Use local caching for small, frequently reused data that changes infrequently or can be versioned, such as immutable configuration bundles or public reference metadata. Use a distributed cache when the working set is shared and too large to duplicate in every process, or when a centrally managed cache reduces repeated source reads. A two-level cache can reduce latency, but the design must coordinate invalidation/versioning across the local and shared layers.

Freshness requirements decide the invalidation method. Time-to-live is simple but permits staleness. Pub/sub invalidation can reduce staleness but messages may be delayed or missed; versioned entries and periodic refresh can provide additional safety. A local cache can also survive a distributed cache outage, but stale authorization or tenant-specific content must not be served beyond its contract.

Calculate memory multiplied by instance count, hit ratio, network costs, and origin offload. Measure p99 latency, hit/miss rate, local-cache divergence, eviction, and origin load during cold start. In an interview, explain which copy is authoritative, how an update becomes visible, and what happens during a cache outage.

### Q234. How do you plan a CDN cache invalidation strategy?

For immutable assets, prefer content-hashed or versioned URLs: a new asset has a new cache key, so clients can safely cache it for a long time. For mutable content, define TTL, purge scope, propagation expectations, and what the user sees while different edge locations may still hold old data. Purging can be rate-limited or asynchronous, and invalidating too much can produce an origin traffic spike.

The cache key and headers must reflect every dimension that changes the response. Personalized or authorization-sensitive content should generally use private/non-cacheable semantics unless the cache explicitly partitions it safely. Do not rely on a purge to correct an unsafe shared cache key; fix the policy first. For HTML that references versioned assets, use different TTLs for the HTML document and immutable JavaScript/CSS files.

Plan for purge failure or delay. A deployment may need to keep old objects available until stale HTML is unlikely to reference them. If an origin change is security-critical, such as revoked content, consider whether CDN invalidation latency is an acceptable risk or whether a stronger authorization check is needed at the edge/origin.

Monitor cache hit ratio, origin offload, purge latency/error rate, stale-content incidents, and origin p99 during rollout. Use gradual traffic release and warm high-value keys if appropriate. In an interview, distinguish cache expiry from invalidation, explain versioned URLs, and state how correctness and cost behave when edge locations converge at different times.

### Q235. How do you prevent an origin stampede when a CDN key expires?

A CDN does not guarantee that every cache node or region will coordinate the same miss. When a popular object expires or is purged, many edge locations can request it from the origin at once. An origin stampede can overwhelm the application even though the CDN normally shields it from most reads.

Use origin request collapsing/coalescing where supported so concurrent misses for one key share a fetch, plus stale-while-revalidate for content whose staleness is acceptable. Add TTL jitter where applicable, protect the origin with concurrency/rate limits, and warm only high-value objects before a large rollout. If using multiple CDN layers, understand whether their miss coordination works across points of presence or only locally.

Protect correctness and security in the fallback. Stale product descriptions may be acceptable; stale authorization or private content may not be. Purging an object that has just been revoked should not trigger a stale response that bypasses the intended policy. Use explicit cache-control and avoid leaking credentials or tenant data in cache keys/headers.

Load-test cold-cache, global purge, regional CDN failure, and origin slowdown. Observe origin request rate, collapsed requests, cache fill latency, 5xx/timeout rates, and cache hit ratio during recovery. The origin should have enough protection that a temporary CDN miss storm results in bounded delay or deliberate load shedding rather than a cascading outage.

### Q236. How do you design tenant-aware data partitioning?

Start with tenant size and traffic distribution, not only the number of tenants. A `tenant_id` shard key can make tenant-scoped queries efficient and help isolate data placement, but one very large or hot tenant can still dominate a shard. Conversely, partitioning only by record ID may distribute writes well while making tenant-level queries expensive.

Choose between shared tables with tenant predicates, schema-per-tenant, database-per-tenant, or a hybrid based on scale, isolation, residency, compliance, and operations. Shared storage is efficient but requires defense-in-depth tenant filtering and resource quotas. Dedicated databases/shards can isolate large or regulated tenants but increase deployment, backup, restore, capacity, and schema-migration costs.

For a shared-shard design, include tenant identity in relevant indexes and cache keys, and ensure async jobs, search queries, object storage, logs, exports, and administrative tools preserve tenant scope. For geographic residency, routing must keep primary data and potentially replicas/backups in permitted regions; telemetry and control-plane metadata need separate analysis.

Design a placement map and migration process before tenants outgrow the initial layout. Use per-tenant metrics for storage, QPS, latency, queue use, and expensive queries without creating unbounded monitoring cardinality. In an interview, explain the access patterns served by the key, the response to tenant skew, and how a tenant can be moved without split ownership or cross-tenant exposure.

### Q237. How do you handle a tenant that outgrows its original shard?

A tenant outgrowing its shard is a data placement migration and a load-isolation problem. First verify what is hot: storage volume, write rate, a few popular keys, long-running queries, or cross-tenant shared dependencies. A larger dedicated shard helps a tenant that is too big for the shared shard, but it may not help a single hot key or global lock.

Keep a routing abstraction that maps tenant IDs to a shard and supports versioned ownership. Prepare the destination schema/indexes, copy a consistent baseline, capture concurrent changes through CDC or another reliable change stream, and validate counts, checksums, deletes, and business invariants. Then switch writes with an ownership epoch or routing version so stale application instances cannot keep writing to the old shard.

Avoid naive dual-writes to two shards without a defined authority and repair protocol. Read shadowing can compare results safely; shadow writes may create real side effects. The cutover plan must cover background jobs, caches, search indexes, tenant-specific quotas, backups, and administrative tools. Retain the old copy until reconciliation and rollback criteria are met.

After migration, verify that the former shard's hot spot is resolved and that tenant-level latency and resource use improved. A dedicated shard can still become hot, so sub-sharding may be required for some access patterns. In an interview, explain the trigger for migration, live-write capture, routing change, rollback boundary, and long-term monitoring.

### Q238. How do you know when a system needs sharding rather than a larger database?

Treat sharding as a response to a measured constraint, not a milestone that follows a certain user count or row count. Identify what is actually saturated: CPU, memory, storage IOPS, write-ahead log throughput, index size, connection slots, lock contention, replication lag, maintenance windows, or a hard limit in the database service. Different bottlenecks imply different remedies.

Before sharding, evaluate query plans and indexes, remove N+1 calls, shorten transactions, batch writes, archive expired data, use native partitioning where it helps pruning/retention, add read replicas for eligible read traffic, cache repeated reads, and scale the database vertically where viable. Some of these will not increase primary write capacity, but they may remove the specific bottleneck at much lower complexity.

Sharding is justified when the workload must be distributed across independent database resources—for example, the primary cannot meet write throughput or storage objectives at a feasible size, or tenants need stronger capacity isolation. The design then needs a shard key, routing layer, rebalancing strategy, global ID/uniqueness plan, backup/restore model, monitoring by shard, and a decision about cross-shard transactions, joins, reports, and consistency. Sharding by a key that is missing from common queries may make those queries fan out to every shard.

Use load testing and growth projections to compare the simpler option and the sharded option on both cost and failure behavior. Include the operational capacity to run, patch, restore, and rebalance all shards. A well-tuned single database can be a better senior-level design than a prematurely sharded system; equally, refusing to shard when a proven hard limit has been reached is not pragmatism.

### Q239. How do DNS-based global routing and application-level routing differ?

DNS-based global routing returns one or more endpoints according to policy such as geography, health, latency, or failover priority. It is useful for steering new connections toward a region, but it is not an instantaneous per-request switch. Resolver/client caching and existing TCP, HTTP/2, WebSocket, or gRPC connections can continue using an old endpoint after the DNS record changes.

Application-level routing makes a decision at a gateway, proxy, or application for each request or connection. It can consider identity, path, tenant, route policy, or current health and can shift traffic more precisely. The trade-off is an extra routing layer, operational state, and a component that needs capacity and resilience of its own. It only helps when the routing layer has correct and sufficiently fresh information.

Neither method guarantees safe data ownership. During regional failover, steering traffic to a new region can produce conflicting writes if the old region still accepts them or if the new region's replica is behind. Stateful services need a leader/epoch/fencing and promotion protocol, not merely DNS health checks.

Use DNS/global routing for coarse regional choice and failover where its convergence delay is acceptable; use application routing for finer-grained traffic decisions. Define health signals, TTL/convergence expectations, connection draining, session/reconnect behavior, and how failback is handled. In an interview, distinguish traffic routing from data replication and write-authority transfer.

### Q240. What does connection draining accomplish during deployment?

Connection draining stops assigning new requests or connections to an instance while allowing existing in-flight work to finish within a configured grace period. The instance should generally become unready before shutdown, so load balancers and service discovery stop selecting it. Draining reduces deployment-induced errors, but cannot guarantee that every request finishes; the deadline eventually forces termination.

Different protocols need different handling. Short HTTP requests may drain quickly. Long-running HTTP responses, gRPC streams, and WebSockets can remain open much longer, so clients may need a reconnect signal and a resume cursor. Requests blocked on a dependency should not be allowed to hold the instance forever; propagate deadlines and enforce a maximum drain duration.

For message consumers, stop fetching new records, finish work that can complete within the grace period, and acknowledge/commit only after effects are durable. Otherwise the broker may redeliver work after termination, so idempotency is necessary. For leased jobs, coordinate ownership so a second worker cannot take over while the old worker still commits stale effects.

Test rolling deploys, forced termination, node loss, long-lived connections, and slow dependencies. Observe termination timeouts, in-flight cancellation, retry amplification, and duplicate operations. In an interview, describe the sequence of readiness update, drain, safe progress/acknowledgement, and shutdown—and explain how the service behaves when the graceful window expires.

### Q241. What is the architectural role of TLS termination?

TLS termination is the point where encrypted transport is decrypted, often at a CDN, load balancer, ingress, or API gateway. Centralizing termination can simplify certificate provisioning, renewal, protocol policy, and edge inspection. It also creates a security boundary: after termination, the proxy can see plaintext unless it re-encrypts traffic downstream.

Decide whether the proxy-to-service hop requires TLS or mutual TLS based on the network trust model, threat model, regulatory obligations, and whether traffic crosses shared or untrusted infrastructure. In zero-trust designs, internal network location is not treated as sufficient proof of identity. mTLS can authenticate workloads, but authorization still must decide whether that workload may perform the action.

If a proxy forwards client IP, scheme, or identity through headers, strip untrusted incoming copies and generate trusted headers at the edge. Otherwise clients can spoof their address or authentication context. Define who validates user tokens, how the authenticated identity is propagated, and whether downstream services can verify it. Plan certificate rotation and key revocation so a control-plane problem does not expire every connection simultaneously.

TLS also has capacity costs: handshake CPU, connection churn, session resumption, and certificate validation. HTTP/2 connection reuse can reduce setup overhead but changes multiplexing behavior. In an interview, draw the trust boundaries, explain where plaintext exists, and state how transport identity, end-user authentication, and resource authorization differ.

### Q242. When does HTTP/2 help, and what are its limits?

HTTP/2 supports multiplexing multiple request/response streams over one TCP connection and compresses headers, reducing repeated connection setup and header overhead for many API patterns. It can be useful for browser traffic and service-to-service calls with many concurrent requests. Connection reuse can reduce TLS handshakes and socket count.

Multiplexing does not remove server-side queueing, database latency, payload serialization, or downstream fan-out. Because HTTP/2 runs over TCP, packet loss can block progress for multiple streams at the transport layer until lost data is recovered. One long-lived connection can also concentrate traffic onto a particular backend, so connection-level load balancing may not evenly distribute individual requests.

Client and proxy limits matter: max concurrent streams, flow control, buffer sizes, connection lifetimes, and server worker capacity affect performance. A single connection can carry many outstanding streams, so counting connections alone may understate concurrency. Conversely, opening excessive connections can increase handshake and resource costs.

Benchmark end-to-end behavior with realistic request sizes, concurrency, loss, and proxy configuration. Measure p95/p99, successful throughput, active streams, connection reuse, and CPU. In an interview, explain the specific overhead HTTP/2 reduces and the bottlenecks it does not fix, rather than saying it is automatically faster than HTTP/1.1.

### Q243. What problem does HTTP/3 attempt to address?

HTTP/3 maps HTTP semantics onto QUIC, which runs over UDP and integrates transport security. One motivation is to reduce transport head-of-line blocking across independent streams: packet loss affecting one QUIC stream does not necessarily stop delivery progress for other streams, unlike multiple HTTP/2 streams sharing one TCP byte stream. QUIC can also support connection migration and faster handshakes in supported cases.

These properties can improve behavior on lossy or changing networks, especially mobile clients. They do not make application code, DNS, database queries, or downstream calls faster by themselves. Performance depends on RTT, loss patterns, payload, congestion control, implementation, proxy support, and the client population. UDP connectivity and network policy can also affect adoption.

HTTP/3 introduces operational considerations: observability and packet capture differ from TCP-based traffic, network middleboxes may behave differently, and deployments need fallback/negotiation behavior. Security and load balancing must understand connection migration and the relevant QUIC/TLS configuration.

Choose the protocol from measurements on the actual path and clients rather than assuming a protocol upgrade is a general latency fix. In an interview, identify the transport-level limitation it is intended to address, the workload likely to benefit, and what metrics or experiment would validate the change.

### Q244. How should an API gateway avoid becoming a single point of failure or a bottleneck?

Run gateway capacity across independent failure domains, but do not stop at “deploy multiple replicas.” Plan the entire request path: DNS/global routing, public load balancer, gateway instances, certificate and configuration distribution, authentication, rate-limit storage, logging, and downstream routing. A fleet of redundant gateways can still fail together if every request synchronously depends on one overloaded policy store, identity provider, or shared limiter.

Keep the hot path lean. Expensive business orchestration is usually better owned by backend services; the gateway should focus on edge concerns such as routing, coarse quotas, TLS policy, request size limits, external authentication integration, and API compatibility. If the gateway performs token validation, know whether it verifies signed tokens locally or makes a remote introspection call on every request. Local verification reduces dependency latency but requires clear key rotation, revocation, audience, issuer, and expiry rules.

Capacity-plan CPU and memory for TLS handshakes, connection churn, HTTP/2 multiplexing, payload buffering, compression, logging, and worst-case headers/bodies—not merely request count. Establish connection/request limits, bounded queues, per-route concurrency budgets, upstream deadlines, and backpressure. If overload occurs, reject cheaply before consuming expensive downstream capacity.

The gateway also needs safe configuration changes and rollout: version configuration, validate it before activation, roll it out progressively, and retain a known-good state. Define fail-open/fail-closed behavior for authentication, quota, and policy dependencies separately; “fail open” may be unacceptable for access control, while a bounded fallback may be appropriate for an approximate traffic-shaping policy. Instrument per-route latency, upstream latency, rejection reason, retries, active connections, queueing, and configuration age. Finally, test gateway failure and regional failover as part of end-to-end load tests rather than treating the gateway as invisible plumbing.

### Q245. What is service discovery, and what failure modes matter?

Service discovery translates a logical service identity—such as `orders.internal`—into one or more reachable endpoints. Discovery may be provided by DNS, a registry, a container orchestrator, or a proxy's dynamic endpoint configuration. It answers “where might I send traffic?”; load balancing chooses among eligible endpoints, and health/readiness signals determine whether a target should receive new work. Products may combine these roles, but the responsibilities remain distinct.

There are two broad patterns. In **client-side discovery**, the client obtains the endpoint list and chooses a target; this gives the client routing control but requires consistent client libraries, refresh behavior, and health tracking. In **server-side discovery**, the client sends traffic to a proxy or load balancer that discovers and selects the backend; this centralizes policy and simplifies clients but adds a hop and a routing layer that must itself be operated reliably.

Important failure modes include stale endpoint caches, endpoint removal racing with long-lived connections, slow or flapping health checks, registry/control-plane outages, resolver caching, bad configuration pushes, and uneven balancing because endpoint capacities differ. A discovery record is not proof the application can serve the intended request: an endpoint might accept TCP but fail every database-backed operation.

Design endpoint freshness and failure behavior deliberately. Cache last-known-good endpoints for a bounded period when safe; use connection and request deadlines; drain connections before planned removal; refresh discovery after failures without having every client refresh simultaneously; and include jitter/backoff to avoid a discovery stampede. For long-lived gRPC or WebSocket connections, account for the fact that changing discovery does not move an established connection; reconnect and missed-message behavior belong in the application protocol.

Monitor endpoint age, resolution errors, connection failures, per-endpoint load, health transitions, and route convergence. For stateful writes, discovery must not decide the authoritative owner on its own: use an explicit leader/epoch/fencing mechanism so stale routing cannot produce two active writers.

### Q246. When should sticky sessions be used for WebSockets?

A WebSocket is a long-lived connection, so once the handshake is upgraded, that TCP connection remains attached to one backend until it closes or fails. The load balancer does not normally redistribute individual messages from an established connection. Sticky routing is therefore often a consequence of connection lifetime rather than an application requirement.

Affinity can be useful when connection state—subscriptions, room membership, or unacknowledged messages—is kept in process memory. However, instance failure loses that local state, and uneven connection duration or activity can create imbalance. Prefer keeping authoritative chat/session state in shared or partitioned storage and treating the connection as a transport, not the durable owner of the user session.

Scale by tracking active connections, messages per connection, fan-out, outbound buffers, and per-instance memory/CPU—not just requests per second. Use connection draining and reconnect signals during deployment; clients need backoff/jitter so an instance failure does not cause a reconnect storm. After reconnecting, the client should authenticate again, re-establish subscriptions, and resume from a durable cursor or fetch a state snapshot.

If affinity is required, choose a stable routing key and define what happens when the target disappears. Service discovery updates will not move established sockets. In an interview, distinguish connection affinity from state ownership and explain reconnect recovery, missed-message handling, backpressure, and how one noisy room or tenant is isolated.

### Q247. How do you scale presence tracking in a chat system?

Presence—online, typing, last seen, or active session count—is usually best-effort, ephemeral state rather than a permanent source of truth. A client can disappear without a clean logout, so presence needs heartbeat/lease expiry and a definition of how long a user may be idle before being considered offline. “Online” should not imply that message delivery is guaranteed.

Partition presence by user ID or another stable key, and store connection/session metadata so multiple devices can be represented. Heartbeats should be coalesced and rate-limited; broadcasting every heartbeat to all contacts would create an enormous fan-out. A presence event stream can publish meaningful transitions (online/offline) rather than every refresh. TTL/leases help remove stale entries, but expiry and reconnect can race, so versioned updates or session IDs prevent an old disconnect event from marking a newly connected session offline.

Avoid a single global presence counter or lock. Use regional aggregation or sharded counters for counts that can be approximate. Privacy settings should determine who may see presence, and the fan-out path needs authorization and abuse controls.

On failover, presence may briefly be stale or reset; the client should be able to re-register. Measure heartbeat rate, stale-session count, propagation delay, fan-out backlog, and per-tenant connections. In an interview, define the semantics, partitioning strategy, handling of multiple devices and stale disconnects, and why this state can tolerate different consistency from durable chat messages.

### Q248. How should a real-time system handle reconnects and missed events?

Treat the live connection as a delivery channel, not the durable record of what happened. A connection can drop after the server sends an event but before the client processes it, or the client can be offline while important events are produced. To recover, persist relevant events or maintain a durable state model and give the client a sequence/cursor from which it can resume.

On reconnect, authenticate the client again, validate the cursor and authorization, and replay retained events after the last acknowledged position. Stable event IDs allow the client to deduplicate repeats; per-entity sequence numbers or stream offsets can reveal gaps. If the cursor has expired or history is too large, send a state snapshot and then resume from a known position, avoiding a race between snapshot creation and new event delivery.

Exactly-once delivery to the client is generally not something a socket alone can guarantee. Design for at-least-once replay and idempotent client application where possible. For state-changing actions, the server's authoritative state must remain correct even if clients miss or reorder notifications. Do not use a push event as the only record of a payment or message.

Connection recovery can create a surge in subscriptions and replay. Apply jitter, pagination/batching, rate limits, bounded outbound buffers, and per-tenant quotas. Monitor reconnect rate, replay backlog, cursor expiry, duplicate/gap count, and event delivery lag. In an interview, explain what the cursor means, how the snapshot/replay boundary is chosen, and what happens when history is no longer available.

### Q249. What is the difference between a CDN cache and an application cache?

A **CDN cache** sits near clients and typically caches HTTP assets or responses at edge locations. It reduces geographic latency and origin traffic for content that can be safely shared under HTTP cache rules. An **application cache** sits closer to service logic and stores domain objects, query results, or computed data—often in process memory or a distributed cache such as Redis. Its key schema and invalidation behavior are controlled by the application.

The layers can be stacked. A browser may cache an asset, a CDN may cache the response, an API gateway may cache a route, and the application may cache the underlying product. Each layer can have a different TTL and invalidation mechanism; purging one does not necessarily invalidate the others. Define which layer owns freshness and how a change propagates through every layer.

CDNs are excellent for versioned static assets and carefully designed public responses. Application caches are more flexible for domain-specific reuse and may be needed when source queries are expensive. Neither should share private or tenant-specific results under an incomplete key. A CDN misconfigured to cache authorization-sensitive content can leak data across users; an application cache can make the same mistake if it omits identity/tenant dimensions.

Measure hit ratio, origin offload, staleness, request coalescing, and origin behavior during a purge or cold cache. In an interview, explain the hierarchy, cache-control/key rules, the source of truth, and how the architecture prevents stale or private responses from surviving longer than permitted.

### Q250. How should you cache personalized responses safely?

First ask whether the personalized response should be shared-cacheable at all. If it depends on user identity, tenant, role, permissions, locale, currency, feature flag, or entitlement, every dimension that changes the result must be represented in the cache policy/key—or the response should be private and not shared. Do not assume the URL uniquely identifies a response.

Authentication and authorization must not be bypassed because an entry exists. A key should be constructed from trusted, canonicalized identity/context rather than arbitrary client-supplied headers. For CDN caching, configure cache-control and variation correctly and be especially careful with cookies, Authorization headers, query parameters, and redirects. A cache-key omission can become a cross-user or cross-tenant data leak.

Revocation and permission changes make freshness a security property. Define how quickly a cached decision must expire and whether high-risk requests must re-check authoritative policy. Short TTLs can reduce, but do not eliminate, stale authorization; event-based invalidation can be delayed or lost without a reliable mechanism. Sensitive data should not appear in cache keys or telemetry in plaintext unnecessarily.

Test cross-user, cross-tenant, privilege-change, locale, and feature-cohort combinations. Monitor suspicious cache hits, stale authorization incidents, and key cardinality. In an interview, distinguish content caching from authorization caching and explain how the design proves that the same cache entry cannot be returned to an unauthorized principal.

### Q251. How do you prevent cache invalidation from racing with a refill?

A common race is: reader A misses the cache and reads old value (v1) from the database; writer B commits (v2) and invalidates the cache; reader A then writes (v1) into the now-empty cache. The cache is stale immediately after a successful write even though the writer did invalidate it. A simple “update database, then delete cache” sequence is not atomic across two systems.

One mitigation is versioned cache values. Every source record has a monotonically increasing version, and a cache fill is accepted only if it is not older than the version already known for the key. Conditional writes or compare-and-set can reject stale refill. A delayed refill or single-flight mechanism can reduce races and stampedes but may add coordination and still need version correctness.

Reliable change propagation (outbox/CDC) can deliver invalidation after commit, while TTL acts as a safety net. However, the propagation delay remains a staleness window. If deletion events can arrive after older updates, retain tombstones/versions long enough to prevent resurrection. For critical decisions, bypass cache rather than trying to eliminate every cross-system race.

Test interleavings explicitly: stale read, concurrent write, invalidation delivery delay, reader refill, cache node restart, and out-of-order events. Measure invalidation lag, stale-read detection, and origin load. In an interview, describe the concrete race and identify which mechanism closes it—not just that you use a shorter TTL.

### Q252. How do you choose between time-based and event-based cache invalidation?

**Time-based expiry (TTL)** is simple: an entry stops being reused after a configured interval. It bounds ordinary cache residence, but a value can remain stale until expiry and many keys can expire together. TTL alone does not prevent stale refill races or guarantee a global staleness bound if source replication is already delayed.

**Event-based invalidation** reacts to a source change and can reduce staleness. But the invalidation event can be delayed, duplicated, lost, or processed out of order; therefore, delivery and consumer behavior need a reliability design. An outbox/CDC pipeline can make the change intent durable, and versioned invalidations can prevent an older event from removing a newer value.

A hybrid is common: events provide prompt invalidation, while TTL bounds the effect of missed or delayed invalidations. Add jitter to prevent synchronized expiry, request coalescing for hot keys, and reconciliation/periodic refresh for important projections. For a high-risk authorization or inventory decision, neither TTL nor event invalidation should replace the authoritative check required by the business invariant.

Set a measurable freshness objective and instrument it. The key question is not “which technique is modern?” but “what is the maximum stale window, what happens if the event pipeline is down, and what data can safely be served during that period?” In an interview, compare latency, origin load, correctness, and operational complexity.

### Q253. What does a Bloom filter help with in a high-volume lookup service?

A Bloom filter is a space-efficient probabilistic structure that answers whether an item is **definitely absent** or **possibly present** in a set. Under the standard design, inserted items do not produce false negatives, but an item not in the set can produce a false positive. A positive result must still be verified against the authoritative store.

It is useful when negative lookups are common and each miss would otherwise trigger an expensive database or distributed-store request—for example, checking whether a URL short code exists or whether a username might already be allocated. The filter can sit in front of the datastore: if it says absent, skip the lookup; if it says maybe present, do the real lookup.

The false-positive rate depends on filter size and number of inserted items. A filter sized too small becomes less selective as the set grows, so capacity planning and rebuild/update strategy matter. Standard Bloom filters do not support arbitrary deletion because clearing bits could incorrectly affect other elements; counting Bloom filters or rebuilds have different memory/correctness trade-offs.

A filter can become stale after inserts or deletes. If a newly inserted item is not reflected in the filter, a “definitely absent” result could be wrong, so update sequencing and rebuild consistency are critical. Use it only where that false-negative risk is controlled and where an extra positive lookup is safe. In an interview, state the probability trade-off, the authoritative verification step, and how the filter stays synchronized with the data source.

### Q254. How should you choose between normalized data and a denormalized read model in an interview?

Start with the write-side invariant and which component owns each fact. Normalize authoritative transactional data where duplication would create update anomalies or inconsistent business state. Then add a denormalized read model when a high-value read path repeatedly joins/aggregates data or needs a shape that is expensive to compute from the write model.

The read model can be updated synchronously, through an outbox/domain event, through CDC, or by periodic batch. Synchronous maintenance adds work to the write path and may couple availability; asynchronous maintenance makes the read model eventually consistent and needs lag monitoring, idempotent upserts, ordering/version checks, delete handling, replay, and rebuild. A denormalized copy should not quietly become independently writable authority.

Example: an order table remains authoritative for order state, while a customer order-history projection combines order status, product summary, and shipment status for fast reads. If the projection is behind, the UI may show a stale status; checkout or refund decisions should consult the authoritative owner where necessary.

Use measurements: query latency, read/write ratio, join cost, cacheability, update frequency, and the acceptable freshness window. Avoid denormalizing just because “reads are more common”; the cost of propagation and repair may outweigh the benefit. In an interview, identify the source of truth, projection update path, staleness contract, rebuild strategy, and what happens when the projection disagrees with the source.

### Q255. How do database isolation levels influence HLD?

Isolation defines what concurrent transactions may observe and which anomalies are possible. Common phenomena include dirty reads, non-repeatable reads, phantoms, lost updates, and write skew, but the exact behavior depends on the database engine and implementation. Names such as READ COMMITTED, REPEATABLE READ, and SERIALIZABLE should not be assumed to mean identical behavior across every product.

Choose isolation from the invariant. A seat booking needs an atomic uniqueness/conditional transition; a balance update may need a ledger or serialized account-level operation; a reporting query may tolerate a snapshot that is slightly behind. A transaction alone does not guarantee correctness if the rule spans rows or the application performs a non-atomic read-modify-write outside the transaction.

Stronger isolation may block or abort concurrent work, increasing lock waits, deadlocks, and retry pressure. Weaker isolation can improve concurrency but requires explicit constraints, version checks, or a merge policy. Keep transactions short, avoid network calls while holding locks, and define bounded retry behavior for deadlocks/serialization failures. Retries must replay the whole local transaction safely and should not duplicate external side effects.

Test concurrent interleavings, not just sequential unit tests. In an interview, name the anomaly you are preventing, the database mechanism that prevents it, the expected behavior for the losing transaction, and the throughput/latency cost. For vendor-specific behavior, verify documented semantics and test the actual isolation level in use.

### Q256. When is a unique constraint better than application-level validation?

Application validation improves user experience, but a check such as “does this username already exist?” followed by an insert is vulnerable to a race: two transactions can both observe no row and then both attempt to create it. A unique constraint or unique index at the authoritative database boundary serializes the conflict and guarantees that duplicates cannot commit under that key.

Use constraints for invariants such as unique usernames, one booking per seat, one idempotency key per scope, or one event inbox record per consumer/event ID. The application should catch the constraint violation and translate it into a meaningful response. Do not expose raw database errors, and do not blindly retry the same insert when the conflict is definitive.

The key must represent the real business scope. For example, username uniqueness may be global or only within a tenant; including tenant ID in the index changes the guarantee. Null/case-insensitive behavior, collation, soft deletes, and partial/filtered unique indexes vary by engine and must be considered. A unique constraint across shards may not be enforceable with only a local index, so global uniqueness may require a routing scheme or dedicated coordination.

Keep the application-level check if it avoids an unnecessary failed write, but treat it as an optimization, not the final correctness mechanism. In an interview, explain the race, define the uniqueness scope, and state what happens when two requests compete.

### Q257. How should an API handle idempotency-key expiration?

Idempotency-key retention is part of the API contract. Keep the key and operation record longer than the realistic retry window, including mobile offline retries, delayed network responses, provider uncertainty, and reconciliation. If the record expires too early, a late retry can be interpreted as a new command and duplicate an operation. Keeping every request forever, however, creates storage and privacy costs.

Separate the idempotency cache from durable business uniqueness. For a low-impact operation, a documented retention period may be sufficient. For payments or bookings, the underlying business object and provider reference should often preserve enough history to prevent duplicate execution after the idempotency key expires. A repeated key after expiry should have a deliberate response, not accidental behavior.

Scope keys by principal/tenant and operation, and store a request fingerprint. A repeated key with the same payload should return the existing operation or its status; the same key with a different payload should be rejected. Coordinate the record with the business transition in a transaction or durable workflow. If the operation outcome is unknown, do not remove the record merely because the request timed out.

Document the retention window and status-query behavior, and test retries just before and after expiration. In an interview, explain what storage persists, what guarantee ends at expiry, and what additional business key prevents a late retry from creating a duplicate high-impact side effect.

### Q258. How do you design API pagination when the underlying data changes frequently?

Use deterministic ordering and a unique tie-breaker, such as `(created_at, id)`, so two records with the same timestamp still have a clear order. A cursor can encode the last sort key plus any filter/sort context required to continue. Make it opaque or integrity-protected so clients cannot alter internal query state, and revalidate authorization on each page.

Cursor/keyset pagination is usually more efficient than a large offset because the database can seek from an indexed position. It does not automatically provide a snapshot. Rows inserted after the cursor may appear later; deleted or updated rows can move between pages if the sort key changes. This is acceptable for a live activity feed in many cases but not necessarily for audit, billing, or export jobs.

If the caller needs a stable view, define a snapshot boundary—such as a dataset version, immutable cutoff, or database snapshot where practical—and ensure all pages use it. For large exports, an asynchronous job can create a consistent output artifact without holding a long-running database transaction. Long-lived snapshots can retain old row versions and create operational cost, so consider that trade-off.

Define max page size, cursor lifetime, behavior when the underlying record is deleted, and what happens when the filter or API schema changes. Cursors are continuation tokens, not authorization grants. In an interview, state whether pagination is live or snapshot-consistent and how the index/query plan supports it.

### Q259. What is an API compatibility contract?

An API compatibility contract includes more than JSON fields. It defines request/response shape and semantics, required versus optional fields, error codes, authentication, pagination, idempotency, ordering, rate limits, timeout behavior, and which guarantees clients may rely on. A change can be breaking even if the schema still parses—for example, changing a field's meaning from gross amount to net amount.

Prefer additive, backward-compatible changes where possible. New optional fields should have defined defaults; clients should handle unknown response fields where the format permits it, and servers should not silently reinterpret an existing field. For incompatible changes, define a version/deprecation strategy, a support window, usage telemetry, and a migration path for clients that are not under your control.

Contract tests can check structural and behavior compatibility between consumers and providers. Rolling deployments require old and new versions to coexist, including messages or long-running workflows that may call older behavior later. Do not remove a field until supported callers and stored data no longer need it. Rollback is safe only if the previous version can interpret the data written by the new version.

Include SLO/error semantics and async states: does HTTP 202 mean durable acceptance or full completion, and how does a client retrieve the result? Document retry safety and unknown-outcome behavior. In an interview, describe how the contract evolves without synchronized deployment and how you detect remaining clients before removing the old behavior.

### Q260. How should a system manage external API quotas across retries and failover?

Model the provider's quota exactly: per account, credential, endpoint, region, operation type, concurrent request count, or time window. A per-instance rate limiter is insufficient when the provider enforces a shared account-wide budget; autoscaling from 5 to 20 instances can multiply outbound attempts even though the provider's quota has not changed.

Coordinate outbound work with a global limiter, a durable scheduler, or hierarchical token allocations. Reserve capacity for critical traffic, coalesce duplicate calls where safe, and use bounded queues for deferrable work. Count retry attempts against the same budget and honor retry-after signals. If the provider has multiple quota classes, allocate by operation cost rather than treating every request equally.

Failover can make this worse: traffic from several regions may suddenly use the same credential/account, and a replayed backlog may compete with live requests. Share quota state or allocate budgets so regional recovery does not reset the counter. If exact coordination is too expensive, define the allowed overshoot and use conservative local budgets.

Monitor request/attempt rate, 429s, latency, queue age, quota utilization, and provider success rate. Define what happens when the quota is exhausted: defer, reject, use an approved fallback provider, or return pending status. A fallback provider may have different semantics and may not support the same idempotency key, so verify before switching critical writes. In an interview, connect quota enforcement to fleet-wide concurrency and outage recovery.

### Q261. How should a service mesh affect application design, and what should remain in the application?

A service mesh can move repeated service-to-service networking concerns into a shared infrastructure layer: workload identity, mutual TLS, traffic routing, telemetry, and standardized timeout/retry policies. Depending on the implementation, it may use sidecar proxies, node-level proxies, or a different data-plane model. It can make policies more consistent across teams, but it adds configuration, resource consumption, debugging layers, and another system that must be understood during incidents.

A mesh does **not** know whether a business operation is safe to repeat. A proxy can retry a GET-like request under an appropriate policy, but retrying a payment, inventory reservation, or non-idempotent POST may duplicate a side effect unless the application implements idempotency or a durable operation identity. Likewise, transport mTLS authenticates a workload identity; it does not prove that a particular end user is allowed to refund a specific order. Business authorization must remain at the resource-owning service.

Avoid layered policy multiplication. If a client library makes three attempts and the mesh independently makes three attempts for each one, the downstream might receive up to nine attempts per original request. Establish a single clear owner for retries where possible, propagate absolute deadlines, cap attempts and per-request time, and distinguish transport errors from application responses. Check actual effective policies rather than assuming defaults.

Before adopting a mesh, name the operational problem it solves—such as consistent workload identity across many teams, traffic shifting, or common telemetry—and compare it with simpler options such as a library, gateway, or platform-native networking. Validate tail latency, CPU/memory overhead, failure behavior during control-plane disruption, certificate rotation, and the team's debugging capability. The goal is consistent networking policy, not moving every concern out of application code.

### Q262. What should happen if service discovery returns stale endpoints?

Treat discovery data as a potentially stale routing hint, not proof that an endpoint is usable. A client or proxy should use bounded connection/request timeouts, detect connection failures, refresh endpoint data with jitter, and avoid repeatedly sending traffic to a known-dead target. If all clients refresh simultaneously, they can stampede the registry/control plane during the same outage.

The platform's readiness and endpoint-removal behavior matters. A target may be removed from new routing while existing keep-alive, HTTP/2, gRPC, or WebSocket connections still exist. Draining is needed for planned shutdown; long-lived connections need reconnect and resume semantics. Discovery freshness and health-check intervals affect how long traffic may continue to reach a failing instance.

Do not retry indefinitely or blindly retry non-idempotent writes. A connection failure before a request is sent differs from a timeout after the remote service may have processed it. Preserve operation IDs and use idempotency/status lookup for mutations. For stateful leaders, stale discovery must not confer write authority; use an explicit leadership epoch/fencing check at the resource.

Monitor endpoint age, connection failures by target, resolution errors, route convergence, retries, and per-endpoint imbalance. In an interview, explain how quickly the stale endpoint is detected, how clients refresh without amplification, and how correctness survives when routing metadata is out of date.

### Q263. How do you design a health endpoint that does not amplify outages?

Separate **liveness** (would restarting the process plausibly help?), **readiness** (should new traffic be sent here?), and **diagnostics** (what dependencies are degraded?). A single endpoint that recursively checks every dependency can cause all instances to become unready during a shared database or identity-provider outage. Restarting every process because the same dependency is down can turn a partial failure into a restart storm.

Keep probe work cheap, bounded, and non-destructive. A readiness check can include the ability to serve critical traffic, but should not execute expensive queries or wait without timeout. Optional dependencies may degrade only one route rather than removing the whole instance from service. Use thresholds and hysteresis to prevent brief latency spikes from causing readiness flapping.

A green process probe is not proof that user journeys work. The service may respond to /health while business requests fail due to a wrong schema, authorization regression, exhausted downstream quota, or incorrect data. Use synthetic checks and service-level indicators for end-to-end correctness while keeping probe semantics simple.

During shutdown, mark readiness false before stopping new work, then drain connections and in-flight requests. For leaders/writers, health endpoints cannot transfer ownership safely; promotion requires a proper protocol and fencing. In interviews, state what each probe controls, how shared dependency failures are reported, and why probes should not generate more load during an outage.

### Q264. How should graceful degradation be represented in an API contract?

A degraded response should remain truthful and machine-readable. If a capability is optional, the API can omit it or return a partial result only when the contract clearly distinguishes “no data exists” from “data unavailable.” For example, an empty recommendations array means no recommendations were produced; it should not ambiguously mean that the recommendation service timed out unless the response includes an availability/freshness indicator.

Define which fields are authoritative, which may be stale, and whether partial responses include warnings, status metadata, or per-section errors. If clients must make a decision based on completeness, include an explicit completeness state or a separate endpoint. Do not return HTTP 200 with misleading values merely to satisfy availability metrics.

For writes, distinguish accepted/pending from succeeded. A request accepted into a durable queue may return 202 and an operation ID; it should not claim that payment, booking, or export completed. During a dependency failure, reject or mark the operation pending if the invariant cannot safely be established. Avoid degrading authentication or authorization controls to keep a route online.

Version the degraded behavior and test clients against it. Document retry semantics and whether the user can recover omitted data later. In an interview, describe the contract seen by the caller, the freshness guarantee, and how the system prevents an optional fallback from changing the meaning of a successful response.

### Q265. How do you choose between blue-green and canary deployment?

A **canary** sends a small, controlled share of traffic or a selected cohort to the new version, then expands based on measurable health and business outcomes. It reduces blast radius and can detect real-world regressions, but needs representative traffic, reliable comparison, and enough volume to reveal rare failures. A low-volume service may need synthetic tests or longer observation windows.

A **blue-green** deployment runs old and new environments side by side and switches traffic after the new environment is validated. It can make the traffic cutover and application rollback straightforward, but temporarily duplicates capacity. It does not automatically roll back database writes, messages, or external side effects created by the new version.

Choose based on traffic control, cost, statefulness, service criticality, and how quickly regression signals become statistically useful. Define automated gates for latency/error rate/saturation and business correctness, not just instance health. A canary should include tenant and request-type diversity; a 5% traffic slice that excludes the biggest tenants may miss the real failure.

Both strategies require backward-compatible schema/event evolution and connection/session handling. Plan expand–migrate–contract, identify the point where rollback is no longer safe, and test old code against new data before release. In an interview, explain rollout steps, success criteria, abort conditions, traffic shift, and what rollback cannot reverse.

### Q266. What is a safe rollback strategy for a database migration?

Design rollback before the migration starts. Prefer additive, backward-compatible changes so the previous application can continue to read/write during the migration window. A destructive schema change or a new write representation may make a simple binary rollback unsafe once the new version has committed data the old version cannot understand.

Use expand–migrate–contract: add the new schema, deploy compatible code, backfill and validate, switch traffic/reads, then remove the old representation in a later release after the rollback window expires. Take a verified recovery point before high-risk changes, but do not rely on restoring the entire database as the default rollback—it may discard legitimate writes made after the snapshot.

For data transformations, keep versioned/checkpointed migration logic and define whether rollback means reverse transformation or forward repair. Reverse transforms may be lossy if the new schema stores information the old schema cannot represent. If both representations are written, define which is authoritative and how conflicts are reconciled. Avoid uncontrolled dual-writes to separate stores.

Test the rollback with mixed application versions and representative data, including nulls, large rows, constraints, and concurrent writes. Record the point of no return and who can approve a risky reversal. In an interview, distinguish rolling back code, reverting schema, and restoring data; they are different operations with different loss risks.

### Q267. How do feature flags interact with data migrations?

A feature flag controls whether behavior is active, but it does not make an incompatible schema safe. Deploying code with a flag off can still run startup migrations, emit events, write new fields through background jobs, or cache a new data format. Schema and event compatibility must be designed independently from the flag.

A safe rollout often adds the compatible schema first, deploys code that can read old and new representations, backfills and validates, then gradually enables the new path for a cohort. During rollout, monitor behavior and business outcomes. If the feature is disabled, the old path must still cope with data written by the new path; otherwise flag-off is not a real rollback.

For a flag that changes write semantics, decide whether old and new code may run concurrently. If they mutate the same invariant differently, use versioned transitions or serialize the cutover. Keep flag targeting server-side and ensure callers cannot escalate access by setting a client-controlled flag. High-impact operational flags need access control, audit logs, owner, expiry, and tested propagation.

Remove obsolete flags after the safe migration window so the code does not accumulate untested combinations. In an interview, describe schema rollout, backfill, cohort enablement, metrics, abort threshold, and how you recover if the new path has already written incompatible data.

### Q268. How do you plan a canary for a low-volume service?

A percentage-based canary may receive too few requests to expose a regression, so “no errors observed at 5%” can be weak evidence. Estimate the expected event/request count and the failure rate you need to detect. If the service sees only a few dozen requests per hour, an aggregate latency/error dashboard may not provide meaningful confidence within a short rollout window.

Use synthetic transactions, representative contract/integration tests, targeted requests for critical flows, and a longer bake period. Validate business invariants such as “no duplicate booking,” “no missing event,” or “authorization rules unchanged” in addition to technical health. For low-volume asynchronous systems, seed controlled test events and track end-to-end completion time.

Keep the canary representative of tenant and request classes, but protect real customers and external providers. Route selected internal or low-risk tenants first where possible, and make the fallback/rollback path reliable. If traffic is low but the change is high risk, prefer stronger preproduction test coverage and staged capability enablement rather than relying on insufficient canary data.

Define abort criteria before the release and measure whether the observations are enough to detect the expected failure mode. In an interview, explain the statistical limitation, what extra signals make the canary informative, and why absence of observed failure is not proof of safety.

### Q269. What is the difference between horizontal scaling and workload isolation?

Horizontal scaling adds instances or nodes to increase total capacity. Workload isolation prevents one workload class from consuming resources needed by another. They solve different problems: adding more shared workers may increase total throughput while still allowing a bulk export or one noisy tenant to starve an interactive payment path.

Isolation can be implemented with separate queues, worker pools, concurrency limits, database pools, priority classes, quotas, dedicated shards, or even separate infrastructure for high-criticality workloads. The boundary should match the constrained resource. A separate application pool does not isolate a shared database from expensive queries, while a tenant rate limit may not address one global lock.

Isolation has costs: capacity can sit idle in one pool while another is overloaded, and separate pools increase operational complexity. Weighted fair scheduling and bounded borrowing can improve utilization while preserving reservations for critical work. Define how priorities are assigned and prevent clients from escalating their own priority.

Measure per-class latency, queue age, rejection rate, resource usage, and starvation. Test a heavy background batch alongside interactive traffic. In an interview, identify the shared bottleneck, explain why more nodes alone do not isolate it, and state how capacity is reserved, fairly shared, and recovered after a burst.

### Q270. How should a design handle bursty batch workloads alongside interactive traffic?

Treat batch and interactive requests as different workload classes. Interactive operations usually have tight latency deadlines; batch jobs often need high total throughput but can be deferred or throttled. If both share the same worker pool, database connections, or provider quota, a batch spike can inflate interactive p99 even when the total system has enough average capacity.

Use separate queues or worker pools, per-class concurrency limits, weighted fair scheduling, and reserved capacity for critical paths. Schedule batch work during lower-demand windows only if deadlines permit. Make jobs checkpointable and restartable, use idempotent chunks, and ensure a failed batch does not restart from zero or duplicate side effects.

Isolation must reach the bottleneck. Separate application workers do not protect a shared database if the batch query performs a large scan; use indexes, replicas/analytical stores, query resource controls, or a separate data pipeline as needed. Bulk processing should respect provider quotas and network/egress limits. Bound backlog and define expiry or prioritization when work cannot finish by its deadline.

Monitor interactive SLOs and batch completion SLOs together, plus database I/O, pool wait, queue age, and per-class throughput. In an interview, explain which work can be deferred, what capacity is reserved, and how the scheduler prevents starvation without letting batch jobs monopolize critical resources.

### Q271. How do you estimate whether a cache saves money?

Compare total system cost with and without the cache, not just the cost of database calls avoided. Include cache memory/instances, network requests, replication, egress, invalidation/publishing infrastructure, cache-warming jobs, engineering/on-call burden, and the origin capacity it allows you to avoid. A cache may be worth using for latency or resilience even if the direct cost reduction is small, but that benefit should be explicit.

Estimate eligible read volume, expected hit ratio from representative key distribution, average value size, origin compute/I/O cost, and request rate during cache failure. A 95% hit ratio on a cheap indexed lookup may save little; a lower hit ratio on an expensive aggregation may save a lot. Personalized cache keys can lower reuse and increase memory requirements.

Include freshness and correctness costs. Aggressive TTLs may raise miss load; long TTLs increase stale-data risk. Event invalidation adds infrastructure and operational failure modes. High-cardinality keys can increase eviction and memory costs, while a hot key can require coalescing or replication.

Measure cost per successful business operation, end-to-end p99, origin offload, stale-result rate, and cold-cache behavior. Test the no-cache/outage scenario to ensure the origin remains safe. In an interview, make the economic comparison quantitative and explain the cache's purpose—cost reduction, latency, origin protection, or some combination.

### Q272. How do you choose a retention policy for logs and traces?

Choose retention by data purpose: incident debugging, product analytics, security investigation, audit, or regulatory evidence. High-volume debug logs and ordinary traces may need sampling and short retention, while security or business audit records may require stricter integrity, access control, and longer retention. These should not automatically share the same store or deletion policy.

Estimate volume from events per second × average record size × retention, then include indexing/replication overhead and query costs. Traces can be sampled by request class, error, latency, or tail-based criteria so rare slow failures are more likely to be retained. Metrics often use aggregation and longer storage at lower resolution. Preserve enough context to investigate an incident without capturing every payload.

Privacy matters: strip credentials and sensitive fields before export, limit access, encrypt data, and handle residency constraints. A log's retention can accidentally outlive the underlying customer data's approved lifecycle. Define how deletion requests, legal holds, backups, and telemetry copies interact.

Test investigations against the retention window: can the team reconstruct a multi-service failure or an async workflow that took several days? Retention is a trade-off between storage cost and diagnostic value. In an interview, explain the purpose of each signal, expected data volume, sampling strategy, access controls, and why the selected window meets operational and legal needs.

### Q273. How should an architecture handle sensitive data in backups?

Backups are part of the production trust boundary because they may contain full database snapshots, transaction logs, credentials, personal information, and deleted records. Encrypt them at rest and in transit, restrict write/read/restore permissions, protect key management separately, and audit both backup access and restore operations. Immutable or access-separated backup copies can reduce the impact of an attacker who compromises production credentials.

Key management must support disaster recovery. If all encryption keys are stored only in the failed region or are deleted with the production account, backups may be unusable. But broadly accessible keys undermine the backup's security. Define key replication, recovery access, rotation, revocation, and separation of duties.

Retention must balance recovery objectives, legal obligations, privacy, and cost. A deletion request may not mean every immutable backup is immediately rewritten, but the policy must define when copies expire and how restored data is prevented from reintroducing records that should remain deleted. Consider cryptographic erasure where appropriate and legally valid, while preserving required audit information.

Test restores in an isolated environment, validate permissions and data integrity, and verify that sensitive data remains protected during the process. Include object-storage versions, snapshots, WAL/binlogs, exports, search indexes, and offsite copies. In an interview, explain who can access backups, how keys survive a regional failure, and how RPO/RTO align with the retention design.

### Q274. What is a data residency boundary in a multi-region system?

A data residency boundary defines where certain data may be stored, processed, transferred, or accessed. It must cover more than the primary database: replicas, backups, logs, traces, search indexes, analytics extracts, object storage, support exports, and third-party processors may all contain regulated or tenant-specific information.

Classify data by sensitivity and jurisdiction, then map each data flow and copy. A global control plane might store limited identifiers or routing metadata, but that must be deliberate and approved rather than assumed harmless. Support access from another region may itself count as processing or access under the applicable policy. The exact legal requirements depend on jurisdiction and contract, so architecture decisions need legal/security review rather than a generic “regional deployment” checkbox.

Implement placement with enforceable mechanisms: region-aware tenant routing, separate regional stores or keys, replication policies, access controls, deployment guardrails, and audit evidence. Prevent accidental cross-region telemetry or backups. Derived datasets must preserve the same restrictions unless transformation genuinely changes the classification under an approved rule.

Plan for failover conflicts. A region that is prohibited from receiving certain data cannot automatically become the failover target for that tenant. Residency can constrain disaster recovery and RTO, so the requirement must be included in the availability plan. In an interview, trace data from ingestion to backup/observability and explain how placement is enforced and audited.

### Q275. How do you design a safe administrative or repair tool?

Administrative and repair tools can bypass normal application workflows and alter production state, so treat them as privileged production software rather than harmless scripts. Require strong authentication, least privilege, scoped targets, approval for high-impact actions, rate limits, and immutable audit records. Separate routine service credentials from operator privileges and use short-lived elevation where possible.

Provide a dry-run mode that shows the target records and intended changes, with counts and sampled output. Make long-running repairs checkpointable, idempotent, and resumable; include a unique repair/job ID and log before/after state or a safe reference to it. Use transactions or conditional/version checks to avoid overwriting newer live changes. For destructive work, require explicit scoping and a clear rollback/forward-repair plan.

Do not let repair jobs ignore tenant boundaries, authorization, data residency, retention, or event ordering. A script that updates the primary but does not publish events can leave search/cache/analytics inconsistent. Define whether the repair should emit domain events, update projections, or trigger reconciliation—avoid accidentally replaying external side effects.

Test on a snapshot or limited cohort, monitor production impact, and pause automatically when mismatch counts or latency exceed thresholds. Protect repair credentials and outputs from leaking secrets. In an interview, describe safeguards against wrong-scope execution, partial failure, duplicate execution, and reintroduction of deleted or stale data.

### Q276. How do you protect a system from a retry storm caused by clients outside your control?

Assume some clients will retry immediately, ignore retry-after, or generate new request IDs for the same operation. Server-side protection must not depend on every client implementing a polite policy. Start with edge rate limits, connection/request budgets, per-identity or per-tenant quotas, request size limits, and concurrency caps before expensive work begins.

For clients that can be given feedback, return clear retry semantics and use 429/503 plus Retry-After where appropriate. Jittered server-directed delays can reduce synchronized retries, but a malicious or broken client may ignore them. Use idempotency keys for mutation APIs, detect repeated logical operations, and avoid treating every repeated payment/booking request as independent work.

During overload, shed low-priority traffic, cap retries internally, and protect shared databases/providers with bulkheads and bounded pools. Do not rely on a circuit breaker alone; the API still needs admission control. Use per-tenant enforcement and fair scheduling so one client cannot consume the retry budget of others. DDoS/bot protection may be required when traffic is distributed across identities or IPs.

Monitor attempt rate versus unique operations, retry amplification, repeated payloads, queue age, 429/503 rates, and successful business throughput. In an interview, explain what the server does when clients ignore backoff and how it prevents duplicate side effects while preserving capacity for legitimate critical work.

### Q277. How should a system handle overload when requests have different business priorities?

Define priority from business outcomes and service contracts, not only endpoint names. A checkout operation may be critical, while a bulk export is deferrable; the same API endpoint could contain both depending on tenant tier, job size, or operation type. Authenticate/authorize priority so clients cannot label every request “critical.”

Use admission control, priority queues, per-class concurrency budgets, weighted fair scheduling, and reserved capacity for critical work. Strict priority alone can starve low-priority tasks forever, so use aging or weighted service guarantees. A separate worker pool helps only if it also isolates the database/provider resource that is actually saturated.

During overload, reject or delay low-priority work before shared pools are exhausted. Return a clear deferred/pending status for durable work, or a retryable rejection with guidance for requests that were not accepted. Do not accept more work into a queue that cannot complete before its business deadline. Protect the recovery path and ensure retries do not jump ahead of new high-priority work uncontrollably.

Measure latency and rejection by class, tenant, and age; verify that low priority receives eventual service when capacity returns. In an interview, explain the scheduling policy, resource boundaries, fairness rules, and behavior when critical traffic alone exceeds capacity.

### Q278. How do you model a workflow that can remain pending for hours?

Persist a durable workflow state machine instead of holding an HTTP connection, thread, or process memory for the duration. Store the operation ID, current state, next action/time, retry count, deadline/expiry, correlation IDs, and references to inputs/results. Each transition should be conditional and idempotent so the workflow can resume after a worker crash.

Use durable timers or scheduled work for delayed steps. Do not create one sleeping thread per long-running operation. A state change and the event/command that wakes the next step should be recorded reliably, often through an outbox or workflow engine. Workers should claim due work, process bounded batches, and update state with version checks.

Define cancellation, manual intervention, expiry, compensation, and uncertain external outcomes. If a workflow waits for a provider callback for six hours, it needs a timeout/reconciliation path for callbacks that never arrive. Events can be duplicated or arrive out of order, so use stable IDs and legal state transitions rather than trusting event sequence alone.

Expose accepted/pending/completed states to clients, with polling/push and authorization controls. Monitor oldest pending age, time spent in each state, retries, and workflows requiring review. In an interview, explain how the workflow survives restarts, how timers are recovered, and how it avoids duplicate external effects after retries.

### Q279. How do you avoid duplicate side effects when replaying historical events?

Separate **rebuilding derived state** from **re-executing business commands**. Replaying events into a search index or projection is often intended and should be idempotent; replaying a historical “charge payment” or “send email” handler can repeat irreversible effects. A generic replay flag is insufficient unless every side-effecting handler respects it.

Use stable event IDs and entity versions, and make projection updates conditional/idempotent. Rebuild into a new index or table where practical, capture concurrent live events from a known snapshot/log position, validate, then switch the read pointer. During projection rebuild, external side effects should usually be disabled or routed to a safe no-op/simulation path.

For business replay, determine whether the event represents a past fact or a command that should execute now. If a repair truly must call an external provider, use the original operation identity/idempotency key where the provider contract permits it and reconcile status first. Maintain an inbox/deduplication record long enough to cover the replay window; but remember deduplication cannot stop two different event IDs representing the same business operation unless there is a business-level uniqueness rule.

Rate-limit replay, preserve per-entity ordering, and audit the operator, input range, and outcome. In an interview, name the target store, specify whether the replay may trigger external calls, explain the dedupe scope, and describe how you prove the rebuilt projection matches the authoritative state.

### Q280. How do you design a poison-message remediation workflow?

A poison-message workflow must preserve evidence and avoid silently losing business progress. Store the original message or an immutable reference, source topic/queue and partition/offset, event ID, schema version, failure category, stack/error context, attempts, and correlation/tenant identifiers with appropriate sensitive-data handling. Distinguish deterministic payload/schema errors from transient dependency failures.

Assign an owner and an explicit disposition: fix consumer code/schema and replay; repair invalid data through an audited tool; skip with a documented business decision; or compensate if downstream state is already inconsistent. If later events depend on this event, moving it to a DLQ and continuing can break per-entity ordering. Pause the affected partition/key where needed or isolate unrelated work while the gap is repaired.

Replay should be controlled, rate-limited, and idempotent. Preserve event identity so inbox/deduplication logic still applies; if a corrected payload needs a new version, keep the original and record the transformation. Prevent replay from re-sending external side effects unintentionally. Track the repair attempt, old/new state, and validation result.

Monitor DLQ age, recurrence rate, affected entities, blocked partitions, and unresolved business impact. A DLQ that accumulates records without an operational owner is delayed data loss. In an interview, explain how the message is diagnosed, corrected, replayed, and proven safe without silently violating ordering or invariants.

### Q281. How do you select a partitioning strategy for multi-tenant data?

Choose the partitioning scheme from tenant size/skew, access patterns, ordering requirements, residency constraints, and promised isolation. If most operations are tenant-scoped, `tenant_id` can localize queries and writes, but it does not automatically balance load: one large tenant may dominate a shard, and global reporting may require scatter-gather.

Range partitioning can support lifecycle tasks and time-window queries, but sequential time keys may create a hot newest partition. Hashing tenant IDs distributes tenants, but cannot split one huge tenant by itself. A hybrid can keep ordinary tenants in shared shards and move large tenants to dedicated shards or tenant-specific buckets. This requires a placement map, routing version, and live migration plan.

All access paths must preserve tenant ownership: background jobs, search indexes, cache keys, event partitions, object storage, exports, backup/restore, and administrative tools. Residency requirements may also constrain replicas and backups. Isolation includes resource fairness; a correctly tenant-filtered query can still create a noisy-neighbor incident.

Measure per-tenant storage, query/write rates, p99 latency, and queue use. Plan for a tenant growing beyond its current placement and prove migration can run during live writes. In an interview, justify the key with common queries and invariants, explain skew mitigation, and state how routing changes without split ownership.

### Q282. How do you prevent a global counter from limiting write throughput?

A single global counter is a serialization point: every increment competes for the same row, key, or coordinator. As concurrency grows, lock contention and coordination latency can dominate throughput. First ask whether the product truly needs an exact, immediately visible count or only an estimate.

For analytics, maintain sharded counters or per-partition aggregates and sum them asynchronously. This spreads writes and improves throughput but permits slight staleness. For ID allocation, reserve ranges from a coordinator so workers generate IDs locally until their range is exhausted; this reduces coordination at the cost of gaps and more complicated range ownership.

If the counter protects a hard invariant—inventory, quota, or a globally unique sequence—an approximate count is not sufficient. Use an authoritative atomic transaction, a central sequencer, or reserved capacity slices with a clearly defined overshoot/ordering model. A sharded display counter may coexist with a separate atomic enforcement mechanism.

Measure hot-key throughput, lock wait, aggregation lag, and coordinator failure behavior. In an interview, distinguish an approximate metric from a correctness-critical counter and explain which component still enforces the invariant after the hotspot is removed.

### Q283. How do you handle clock skew in expiration and lease logic?

Clock skew means two machines' wall clocks disagree; clocks can also drift or jump after synchronization. Logic that assumes identical time can expire a lease too early, retain it too long, or order events incorrectly. Use monotonic clocks for measuring elapsed time within a process, and define an authoritative clock or datastore timestamp for cross-node expiry when possible.

A lease needs more than accurate clocks. A worker may pause past expiry and resume after another worker acquires the lease. The protected resource must validate a fencing token/epoch or conditional version so the stale owner cannot commit. Lease expiry makes takeover possible; it does not terminate the old process.

For expiry jobs, make the state transition conditional on both the stored state and expiry rather than blindly deleting because a worker's clock says a row is old. For event ordering, prefer aggregate versions or log positions to timestamps when causal order matters. A timestamp is not a globally reliable ordering service.

Test clock jumps, delayed workers, renewal failures, and overlapping ownership. In an interview, explain which clock is authoritative, what happens if a worker resumes late, and how the target resource rejects stale writes.

### Q284. What is the difference between a lease and a lock with an explicit release?

A lock with explicit release remains held until the owner releases it or a recovery mechanism intervenes; it can block indefinitely if the owner crashes. A lease expires automatically unless renewed, which improves recovery but creates a risk that the old owner continues working after expiry.

Neither guarantees exclusive side effects by itself. A process may pause, lose connectivity to the lock service, or resume after a new owner has acquired the resource. Use a monotonically increasing fencing token and require the target resource to reject writes from older ownership generations. If an external provider cannot validate fencing, use stable idempotency identities, a single durable command owner, or reconciliation.

A lease needs renewal policy, bounded acquisition time, behavior on renewal failure, and fencing. A non-expiring lock needs crash recovery and proof that the old owner can no longer act before force-release. Do not hold a global lock across slow network calls if a narrower transaction or conditional update can protect the invariant.

For short local critical sections, database locks may be simpler. In an interview, state what resource is protected, who validates ownership, and what happens if an owner pauses or loses contact with the coordinator.

### Q285. How should a design handle a cache or broker that is itself replicated?

Understand what the system acknowledges, when data becomes durable, whether replication is synchronous or asynchronous, how leaders are selected, and what data may be lost or unavailable during failover. A cluster can improve node-failure handling while still losing recent data, serving stale values, or requiring manual recovery under some configurations.

For a cache, recent entries may be lost during promotion and repopulated from an older source. That can be acceptable if the origin remains protected during cold start. If the cache stores sessions, workflow coordination, or rate-limit state, define how lost/stale state affects security and correctness rather than blindly bypassing checks.

For a message broker, durability depends on producer acknowledgements, replication factor, in-sync replica policy, persistence, retention, and consumer-offset handling. A producer success means the configured acknowledgement policy was met; it does not mean every consumer processed the message. Consumers still need idempotency and replay safety.

Validate actual product/configuration behavior under node, zone, and network failures; do not infer guarantees from the word “replicated.” Monitor lag, leader changes, rejected writes, reconciliation, and recovery time. In an interview, distinguish cluster availability from data durability and consistency.

### Q286. How do you choose between synchronous and asynchronous audit logging?

The decision depends on whether the audit record is part of the operation's correctness or compliance boundary. If a high-risk action must not occur without a durable audit trail, record the audit intent in the same local transaction as the business state change where possible. For operational debug logs, asynchronous buffering and sampling may be acceptable.

Synchronous logging to a remote service on every request adds latency and can make the logging service a critical dependency. Decide fail-open versus fail-closed by action: a privileged administrative change may need to fail if no durable audit intent can be recorded, while noncritical debug telemetry should not bring down checkout.

A transactional outbox can preserve the audit event's intent atomically with the business change, then deliver it asynchronously. The publisher needs durable retention, retries, integrity protection, access controls, and monitoring. Do not confuse ordinary logs—which may be sampled or pruned—with the tamper-resistant business audit trail needed for investigations.

Audit records should identify actor/workload, action, target, timestamp semantics, operation ID, result, and relevant policy context while minimizing sensitive payloads. In an interview, state which operations require durable audit before commit, how delivery is guaranteed, and what happens during audit-store failure.

### Q287. How do you make an API safe for clients that retry after losing the response?

Assume the client can lose a response after the server commits a mutation. Retries may come from network timeouts, proxies, browser refreshes, or mobile reconnects. Without a stable operation identity, the same logical command can execute twice.

Accept an idempotency key or client operation ID scoped to the authenticated principal/tenant and operation. Atomically claim the key and store a request fingerprint so concurrent first attempts cannot start independent operations. A repeated key with the same payload should return the existing operation/status; reusing the key with different input should be rejected.

Coordinate the idempotency record with the business transition in one transaction or durable state machine. A separate “remember key” write can still leave a crash window. For external effects, pass the same identity downstream if supported and reconcile uncertain outcomes. Long-running operations should expose an operation resource with explicit pending/success/failure states.

Define key retention and the behavior after expiry; high-impact operations may also need business-level uniqueness that outlives the retry window. In interviews, walk through concurrent duplicate requests and a crash after commit but before response, then show exactly what the retry returns.

### Q288. How should a service expose partial completion?

Partial completion should be explicit and machine-readable, not an ambiguous HTTP success. If a batch contains 100 independent items and 97 succeed, expose which items succeeded, failed, or remain pending, with stable item IDs and actionable reasons. A single top-level 200 without item-level state may cause callers to retry completed work and duplicate side effects.

For long-running operations, return an operation ID and expose progress/terminal states. Distinguish accepted from completed and technical success from business success. Define whether cancellation stops only pending items or compensates completed items. If items share a transaction or ordering invariant, partial completion may be invalid and should not be invented merely for convenience.

Persist each item result durably so aggregate status can be reconstructed after a crash. Make retries idempotent at both workflow and item level. For large outputs, use a result resource or export artifact with explicit authorization and retention. Include correlation IDs and a repair path for unresolved items.

Tell clients whether to retry failed items only, poll pending items, or treat the operation as requiring manual review. In an interview, explain partial-success semantics, how retries avoid repeating completed effects, and what consistency boundary applies across items.

### Q289. How do you decide between one large service and several smaller services?

Choose service boundaries around cohesive business capabilities, data ownership, change patterns, team ownership, and distinct operational needs—not source-file count. One service is often better when components change together, share transaction invariants, and benefit from low-latency calls. Separate services are useful when capabilities need independent scaling, deployment, security, fault isolation, or ownership.

Splitting introduces network latency and failure, timeout/retry policy, API/event compatibility, observability, deployment pipelines, secrets, discovery, duplicated data, and cross-service workflows. A local transaction may become a saga with partial progress and compensation. If module boundaries are unclear, moving tangled dependencies over HTTP tends to create a distributed monolith.

A modular monolith can enforce internal boundaries while retaining one deployment and simpler transactions. It is a strong option while domain boundaries are changing or the team cannot operate many services reliably. Conversely, an unstructured monolith can create release coupling and a shared failure domain.

Use evidence such as independent scaling profiles, release contention, security isolation, or an operational bottleneck. Before extraction, define data ownership, contract, failure behavior, migration, and rollback. In an interview, explain what coupling the split removes and what new coupling it introduces; “microservices scale better” is not sufficient.

### Q290. What is the role of a backend-for-frontend (BFF) in HLD?

A backend-for-frontend is a server-side layer tailored to a particular client experience, such as web, mobile, or an administration UI. It can aggregate data from several services, adapt payloads, reduce chatty client calls, and isolate client-specific API evolution from domain services. It is not automatically a replacement for an API gateway or a place to duplicate business rules.

A mobile client may need a compact response and different offline/reconnect behavior than a web client. Without a BFF, the client may make many calls and own orchestration; a BFF can simplify this but adds another hop and a component to deploy and operate. Aggregation also creates fan-out latency and partial-failure questions.

Keep domain invariants in the owning services. The BFF may compose and transform data, but should not independently decide that inventory is available, authorize a refund, or own a second source of truth. Define which downstream services are required, how deadlines propagate, which fields may be omitted, and how responses are cached safely.

Avoid one BFF per screen unless the operational cost is justified. Use it when client-specific composition is stable and valuable. In an interview, distinguish client adaptation from domain ownership and explain how the BFF handles a failed downstream without hiding correctness errors.

### Q291. How do you design a system for graceful regional failback?

Failback is the controlled return to a preferred region after it recovers. It is not simply reversing DNS or routing back to the old primary. The recovered region may be behind, may contain outdated configuration, or may not have enough capacity to serve peak traffic. Returning too soon can create split brain or lose writes accepted in the surviving region.

First verify that the original region is healthy at the business-path level, not just reachable. Synchronize or reconcile missing data, confirm replication position and schema compatibility, and establish exclusive write ownership using the appropriate leader/fencing protocol. Ensure queues, timers, and background jobs will not run twice when ownership changes.

Shift traffic gradually and monitor end-to-end SLOs, replication lag, errors, latency, connection churn, queue backlog, and business invariants. Maintain a rollback option if the recovered region destabilizes. Avoid automatic oscillation by requiring sustained health and using cooldown/hysteresis. Long-lived connections may remain in the surviving region until clients reconnect.

Define what happens to in-flight and unknown operations that began during the outage. Reconcile payment/booking status before replaying or moving them. In an interview, distinguish restoring infrastructure from restoring authoritative ownership and user traffic, and explain how failback is tested with realistic data volume.

### Q292. How do you decide whether to queue a write during a regional outage?

Queue a write only if the system can durably accept the command, define its ordering and expiry semantics, and execute it later without violating the business rule. An in-memory queue or a 202 response before durable recording does not establish reliable acceptance. The API must communicate pending status and provide an operation ID.

Some work is safe to defer: analytics events, report generation, or an email that remains useful later. Other work is time-sensitive or conflict-prone: reserving the last seat, buying an expiring offer, or authorizing a payment may not remain valid when the region recovers. Revalidate business state before execution and define whether conflict means rejection, compensation, or manual review.

If both regions accept writes during a partition, the design needs a conflict-free model or a deliberate consistency trade-off. For a strongly consistent balance, inventory, or globally unique name, it may be safer to reject writes where authoritative ownership cannot be established. Queuing must not become an implicit promise that every request will eventually succeed.

Use durable workflow state, idempotency, deadlines, fair backlog processing, and reconciliation. When service returns, throttle replay so it does not overwhelm the recovered region or provider. In an interview, explain the operation's deadline, authority model, acceptance guarantee, and user-visible result for a write that cannot be safely completed during the outage.

### Q293. How should a system respond when a replica is too far behind?

Define a freshness threshold based on the operation's contract. A catalogue read may tolerate seconds of lag; a financial or inventory decision may require the authoritative source. Monitor lag as time and, where possible, log/sequence position. A green connectivity status does not mean the replica has applied recent writes.

When lag exceeds the threshold, route freshness-sensitive reads to the primary or another eligible replica, wait within a bounded deadline, or return an explicit stale/pending response if the API permits it. Do not sleep for an arbitrary duration and assume the replica has caught up. If the primary is overloaded, redirecting every read to it may worsen the incident; preserve capacity for critical writes and use controlled load shedding.

Find the cause: write rate exceeding apply capacity, large transactions, replica CPU/IO, network saturation, long-running queries, schema/index maintenance, or storage contention. Fix the bottleneck before simply adding replicas; read replicas can compete with replication for resources.

Failover is a separate decision. Promoting a lagging replica may lose acknowledged writes depending on commit policy. Verify RPO and election rules before promotion and prevent the old primary from continuing to write. In an interview, describe both read routing under lag and the data-safety criteria for failover.

### Q294. How do you design an API gateway's rate limiting for authenticated and anonymous traffic?

Anonymous traffic needs coarse limits and abuse signals such as IP/network, connection rates, request size, and bot patterns. Authenticated traffic can use account, API key, tenant, route, and operation-cost budgets. IP-only limits are often unfair behind shared NAT and insufficient against distributed attackers; identity-only limits can be bypassed if identity creation is cheap.

Apply inexpensive limits at the edge before authentication or expensive parsing, then enforce stricter resource-aware quotas after identity is known. A search request, bulk export, and simple read consume different capacity, so use operation-cost or concurrency limits in addition to requests per second. Prevent clients from selecting another tenant's quota key or claiming elevated priority.

The gateway fleet must enforce limits at the scope promised. Per-instance local counters can allow aggregate traffic to multiply with instance count; a shared atomic limiter is more accurate but adds latency and a potential hot-key/store bottleneck. Hierarchical token allocation may trade exactness for lower hot-path coordination. Define behavior if the limiter or identity provider is unavailable.

Return clear 429 responses and retry guidance where safe, but assume clients may ignore it. Measure allow/reject rates, limiter latency, key cardinality, and quota overshoot during autoscaling/failover. In an interview, define quota scope, fairness policy, abuse model, failure behavior, and how the gateway avoids becoming the bottleneck.

### Q295. How do you evaluate a fallback provider for critical traffic?

A fallback provider must be evaluated for semantic compatibility, not just availability and price. Check whether it supports the same operation types, currencies/regions, idempotency behavior, status lookup, error model, rate limits, security requirements, and reconciliation interfaces. The alternate provider may define accepted/succeeded differently or settle at a different time.

Capacity matters. If the primary provider fails and all traffic moves instantly to the secondary, that provider may be overwhelmed or hit account-wide quotas. Use pre-agreed capacity, health/circuit-breaker criteria, traffic limits, and controlled ramp-up. A health endpoint is not proof that the provider can process production volume of critical writes.

For a mutation such as payment, a timeout outcome may be unknown at the primary. Do not automatically send the same logical payment to another provider unless you can establish that the primary did not process it or both providers share a safe operation-identity protocol. Otherwise failover can double-charge. Maintain a durable attempt record and reconcile status across providers before retrying.

Define activation/deactivation, data mapping, audit trail, customer experience, and rollback. Test timeout-after-success, partial outages, quota exhaustion, and duplicate callbacks. In an interview, explain which requests are safe to fail over transparently and which require status resolution or an explicit pending state.

### Q296. How do you prevent queue starvation across priorities?

Strict priority always processes high-priority work first, but sustained high-priority traffic can leave lower-priority work waiting forever. Pure FIFO creates the opposite problem when slow bulk work blocks urgent tasks. Choose a scheduling policy that matches the business contract and define the fairness objective.

Common approaches include weighted fair queues, reserved capacity per class, round-robin among priorities, aging (raising priority as work waits), and separate pools for critical workloads. A weighted policy can guarantee low-priority work receives a fraction of capacity while allowing critical work to borrow unused capacity. The right values depend on deadlines, resource consumption, and service objectives.

Priority must be based on trusted metadata and cannot be freely chosen by clients. Consider tenant fairness as well as task priority; otherwise one tenant can fill the high-priority queue. Work with variable costs may need cost-aware scheduling rather than counting messages equally.

Monitor queue age and completion SLO by class, not just aggregate depth. Ensure retries and replays do not jump ahead forever and starve new work. In an interview, explain what each priority receives under sustained load and what happens when critical demand alone exceeds capacity—because no scheduler can make an overloaded system complete every task on time.

### Q297. How do you distinguish a capacity problem from a correctness problem during an incident?

A capacity problem means the system cannot complete enough correct work within its resource or latency limits; a correctness problem means it is producing or accepting the wrong state/result. They can coexist: overload causes timeouts and retries, which may expose duplicate side effects or stale updates. Incident response must protect business invariants even while restoring throughput.

For capacity, look for queue growth, saturation, connection-pool wait, CPU/memory/IO pressure, rising p95/p99, throttling, and successful throughput flattening while offered traffic rises. For correctness, inspect invariant violations, duplicate IDs/effects, conflicting versions, ledger imbalances, missing/duplicated events, authorization bypass, or divergence between authoritative and derived stores. HTTP 200 and healthy CPU do not prove correct business behavior.

Mitigate differently. Capacity may require admission control, load shedding, reducing fan-out, pausing backfills, or fixing a query. Correctness may require stopping a write path, disabling a feature, fencing an owner, preserving evidence, and running reconciliation. Do not increase throughput on a path that is corrupting data; do not perform a risky failover before establishing write authority.

Use traces/logs for context and durable operation IDs/state transitions for reconstruction. In an interview, state which signals distinguish the failure classes, which action limits damage first, and what validates recovery. Business invariants should be included in alerts and runbooks, not checked only after customers report a problem.

### Q298. How do you decide what should be included in a design diagram?

Include components and boundaries that affect correctness, capacity, latency, security, failure recovery, or team ownership. A useful HLD diagram normally shows clients/edge, gateways or proxies, stateless services, authoritative stores, caches/read models, brokers/workers, external providers, region/failure-domain boundaries, and the main synchronous/asynchronous flows. Mark where data is persisted and which component owns each important fact.

Do not draw every class or cloud product. Add detail when it explains a decision: primary/replica relationships, shard routing, outbox/event flow, partition key, writer ownership, or trust boundary. Use separate diagrams for system context, request/data flow, deployment/failure domains, and a deep-dive workflow when one drawing becomes unreadable.

Label arrows with semantics where it matters: HTTP/RPC, event, replication, cache invalidation, or provider callback. Distinguish synchronous request paths from asynchronous work and show what durable acceptance means. For multi-region designs, mark which data is replicated, which writes have one owner, and how failover changes routing.

A diagram is not complete without a verbal walkthrough of the happy path and at least one failure path. In an interview, keep it legible, state assumptions, and use the diagram to explain trade-offs rather than treating boxes as an architecture by themselves.

### Q299. How should a senior candidate respond when an interviewer challenges a technology choice?

Treat the challenge as an invitation to expose assumptions, not as a debate to win. Restate the requirement the choice was meant to satisfy, then explain the mechanism and trade-off. For example: “I chose a relational database because this workflow relies on transactional uniqueness and joins. If write throughput exceeds a single primary's capacity, I would measure the bottleneck and consider sharding, which makes cross-shard transactions and global uniqueness harder.”

A strong defense names the best alternative and explains why it was not chosen under the current assumptions. Avoid popularity arguments (“Kafka is standard,” “NoSQL scales better”) and avoid pretending the design has no weaknesses. If the interviewer changes a requirement, trace its impact on data model, latency, consistency, cost, capacity, operations, and migration.

You do not need every vendor-specific detail from memory. State which guarantee needs verification and how you would test it: load test, failure injection, restore exercise, query-plan check, or proof of the concurrency invariant. Be precise about what is known, assumed, and unresolved.

In an interview, revise a decision when the facts change. Senior-level judgment is not always choosing the right component on the first attempt; it is making a defensible choice, recognizing the conditions under which it fails, and evolving the design without losing correctness.

### Q300. What is the final checklist for defending an HLD design?

Before calling an HLD complete, be able to explain:

1. **Requirements and success:** primary user journeys, business invariants, measurable latency/availability/freshness targets, retention, geography, security, and cost assumptions.
2. **Capacity:** average/peak QPS, traffic mix, storage growth, network bandwidth, fan-out, cache hit/miss behavior, queue drain, connection budgets, provider quotas, and headroom after a failure.
3. **Architecture and ownership:** component responsibilities, authoritative owner for each fact, API/event contracts, synchronous versus asynchronous boundaries, trust boundaries, and failure domains.
4. **Data and consistency:** schema/index/query model, partition/shard key, hot-key strategy, transaction isolation, replication semantics, uniqueness, ordering scope, cache consistency, and read-model rebuild.
5. **Failure behavior:** timeouts, deadlines, retries, idempotency, partial failure, overload, backpressure, load shedding, DLQ/poison messages, compensation, and reconciliation.
6. **Operations:** SLI/SLO and alerts, tracing/logging, load/failure tests, schema rollout/rollback, backup restore, RTO/RPO, regional failover/failback, access controls, and runbooks.
7. **Trade-offs and evolution:** why this is the simplest viable design, what breaks first, which assumptions could change the architecture, and how to migrate safely as load or requirements grow.

Finally, walk through one successful request and at least two failure paths—such as a timeout after commit and a regional partition. State what the caller sees, which component owns recovery, and how you verify no business invariant was violated. A senior design is not one with no failure; it is one whose failure behavior is deliberate, observable, and recoverable.

## 18. Senior-level scenario bank

For every scenario, state assumptions, identify the invariant, analyse evidence, recommend a design, explain alternatives, describe failure behavior, and name operational signals. These are connected design discussions, not technology flashcards.

### Scenario 1 — 100,000 requests/second, one hot database shard

**Problem:** The API receives 100,000 requests/second, but one database shard has high CPU and p99 latency while other shards are relatively idle. Adding application instances does not improve successful throughput.

**Clarify the shape of the hotspot:** Separate read hotness from write hotness, and distinguish one popular key, one oversized tenant, poor shard-key distribution, expensive queries, lock contention, and an undersized shard. Ask which request paths touch the shard, what fraction is cacheable, whether operations require per-key ordering, and whether the shard is saturated by CPU, storage, connections, or locks.

**Diagnose before changing topology:** Inspect per-shard QPS, bytes, CPU, lock waits, queue depth, query plans, connection-pool wait, and key/tenant skew. Compare rows stored with work served; row-count balance does not guarantee load balance. Check if retries or a cache miss storm are sending extra load. Establish whether the bottleneck is one logical key or a whole shard.

**Design options depend on the cause:**
- For a hot read key, use request coalescing, local/distributed caching, or replicated read paths if the freshness contract permits them. A cache reduces repeated reads but does not solve write contention.
- For a large tenant, move it to dedicated capacity or split that tenant into buckets while keeping ordinary tenants shared. This needs a routing map and safe migration.
- For a monotonically increasing range hotspot, consider hash/bucket distribution or a composite key—but confirm that range scans and ordering can tolerate the change.
- For an expensive query, fix the plan, index, pagination, or data model before introducing more shards.

**Migration and correctness:** Sharding more broadly is a live data migration: copy a consistent baseline, capture concurrent changes, validate counts/checksums, switch routing with a version/epoch, and retain a defined rollback or forward-repair path. If operations must be atomic per account/order, colocate those records where practical. Avoid changing modulo-based routing without a controlled transition.

**Trade-offs and signals:** Key salting spreads writes but complicates reads and aggregation; caching increases freshness/invalidation complexity; dedicated tenant placement costs more. Track per-shard p99, hot-key share, origin/cache hit ratio, lock waits, rebalance lag, and business errors. A good follow-up answer explains why adding shards alone may not divide a single hot key and how the design handles migration failures.

### Scenario 2 — Adding a cache increases latency

**Problem:** Redis reduces database CPU, but p99 latency rises after the cache is introduced.

**Clarify:** Is the cache remote or local? What are hit ratio by endpoint, value size, serialization cost, connection-pool wait, Redis p99, origin query latency, miss/refill concurrency, and invalidation behavior? Did the rollout change request mix or only add the cache layer?

**Diagnosis:** Compare traces before and after and break latency into cache lookup, pool acquisition, serialization, database miss path, and downstream calls. A high hit ratio does not prove the cache improves end-to-end latency: a small indexed database read might be faster than a network round trip. Large values, hot-key contention, connection saturation, cache stampedes, and a miss path that performs duplicate reads can make p99 worse. Also verify whether cache failures are being retried inside the client library.

**Design:** Cache only values with demonstrated reuse and a suitable freshness contract. Add request coalescing for hot misses, bounded origin concurrency, connection-pool sizing, TTL jitter, and an explicit stale-while-revalidate policy only for data that can safely be stale. Keep authorization and transactional inventory decisions on their authoritative paths. Use versioned values or safe invalidation for data that changes frequently.

**Failure behavior:** Redis outage should not fan every request straight to the database without limits. Use bounded fallback, load shedding for noncritical reads, and gradual warm-up after recovery. Avoid unlimited retries and do not make every new application instance preload the entire keyspace simultaneously.

**Signals and follow-up:** Track p50/p95/p99 by endpoint, hit ratio, cache latency, pool wait, origin QPS, refill concurrency, stale-read rate, and database saturation. Test cold cache, mass expiry, hot keys, and complete cache outage. The decision is based on net user latency and origin work saved—not hit ratio alone.

### Scenario 3 — Read replicas return stale order status

**Problem:** A client successfully creates an order, then immediately calls GET /orders/{id}. The write transaction committed on the primary, but the read is routed to an asynchronous replica that has not applied it yet, so the API returns “not found” or an old state.

**First distinguish failure from lag:** Check the primary's durable record and the replica's replay/commit position. Confirm whether this is normal replication lag or a failover where the new primary may be behind. Also verify that the read path is not using a stale cache or search index instead of the database replica.

**Choose a consistency contract:** If the product promises read-your-writes, route the immediate read to the primary, carry a database-supported commit/consistency token, or wait until the chosen replica has applied the needed log position. If the status page can tolerate a documented delay, expose a pending/processing state and refresh within the expected propagation window. An arbitrary fixed sleep is not a guarantee because lag is variable.

**Do not make every read primary-bound without reason:** That is a simple fix but may erase the read-scaling benefit of replicas. Classify reads by correctness: post-create order lookup and payment status may need stronger freshness, while catalogue descriptions or analytics may tolerate staleness. Enforce the same rule in caches and derived read models, not just SQL routing.

**Failover consideration:** A replica promoted after primary loss may lack recently acknowledged writes depending on replication guarantees. The service must establish a single authority, understand RPO, and handle in-flight write outcomes as uncertain. Read-your-writes routing cannot recover data that was lost during failover.

**Signals and follow-up:** Track replica replay lag, stale-read rate, reads routed to the primary, cache age, and post-write request traces. Test normal lag and promotion under load. Be ready to explain how a consistency token works in the chosen datastore and why “use the primary” is a correctness policy with capacity implications.

### Scenario 4 — Payment provider times out after charging

**Problem:** The payment provider may have successfully charged a customer, but the provider response timed out before the payment service received it. The client retries, creating a risk of duplicate charge or a false failure state.

**Invariant:** One logical payment operation must not produce multiple unintended charges, and the platform must not report a definite failure when the outcome is unknown.

**Design:** Persist a payment operation and stable internal/provider idempotency identity before calling the provider. Atomically bind the client idempotency key to a canonical request fingerprint and payment record so concurrent duplicate requests converge on the same operation. Use explicit states such as CREATED, SUBMITTING, AUTHORIZED/CAPTURED, FAILED, and UNKNOWN/RECONCILIATION_REQUIRED; state transitions must be legal and durable.

After a timeout, do not create a new payment attempt with a new identity. Query provider status if supported, reuse the same provider idempotency key within its documented scope/retention window, process signed callbacks idempotently, and reconcile unresolved records against provider transaction/settlement reports. Provider idempotency is a useful guarantee, not a replacement for local operation identity or reconciliation.

**Crash windows:** The service may crash before sending the provider request, after the provider charges but before the response, after receiving the response but before updating the database, or after committing state but before replying to the client. Each window needs a repeatable action: resend the same logical request, query status, consume a callback, or reconcile. Never assume a timeout means rollback.

**API behavior:** Return a stable payment/operation ID and expose pending/unknown status where appropriate. Do not tell the user the payment failed until evidence supports that conclusion. A compensating refund is a new financial action, not erasure of the original charge.

**Signals and follow-up:** Track unknown outcomes by age, provider timeouts, duplicate callbacks, retry counts, reconciliation mismatches, and charge/refund anomalies. Explain which guarantees depend on the provider and how support staff trace one payment end to end.

### Scenario 5 — Kafka consumers fall behind

**Problem:** Producer rate has increased, consumer lag is growing, and events are now processed after their business deadline. Adding consumers has limited impact.

**Diagnose the bottleneck:** Compare arrival and successful completion rates, oldest-record age, lag by partition, processing duration, rebalance frequency, retries/DLQ volume, downstream database latency, and provider throttling. Determine whether the problem is a temporary burst or sustainable throughput deficit. A hot key can serialize work in one partition; adding consumers cannot parallelize that key without changing the ordering model.

**Quantify recovery:** If the backlog is 1.8 million records, workers complete 11,000 records/s, and new events arrive at 6,000/s, the net drain rate is 5,000/s and the ideal time to catch up is 360 seconds. If completion rate is at or below arrival rate, the backlog will not shrink. Use successful, committed processing—not dequeued attempts—as the service rate.

**Design:** Fix the slow stage first. Increase consumer parallelism only if there are enough partitions and the database/provider can accept more concurrent work. Consider batching to reduce overhead, improving query plans, isolating slow event types, adding partitions where ordering and remapping rules allow it, or moving expensive processing to a separate topic/worker pool. Use bounded retry delay and DLQ/quarantine handling for deterministic poison events.

**Recovery behavior:** Once the dependency recovers, ramp up consumption gradually to avoid overwhelming it with replay traffic. Preserve per-key ordering where required, keep processing idempotent, and check that event retention covers credible downtime plus repair and catch-up. If queued work has expired, follow the business policy rather than delivering obsolete notifications or applying stale commands.

**Signals and follow-up:** Track per-partition lag, oldest age, successful completion, retry ratio, rebalance count, downstream saturation, and time-to-drain. Explain why increasing consumer count can worsen a downstream bottleneck and how you would prove the new capacity is sustainable.

### Scenario 6 — Autoscaling overloads the database

**Problem:** CPU-based autoscaling adds application instances as traffic rises, but database connections, lock waits, and query latency spike. The system has more application capacity but lower successful throughput.

**Root cause hypothesis:** Every application instance may bring its own database pool. If 30 instances each permit 40 connections, the fleet could open 1,200 connections before batch workers or other services are counted. More instances also increase cache misses and simultaneous downstream calls. The database may be the limiting resource, not the application CPU.

**Diagnose:** Compare offered requests with successful completions; inspect pool-acquisition wait, pool hold time, connection count, database CPU/IOPS, slow-query plans, transaction age, lock waits, replication lag, and request fan-out. Determine whether the database is overloaded by legitimate work, retries, N+1 queries, a hot row, or long transactions holding connections during network calls.

**Design:** Set a fleet-wide connection budget and account for rolling-deployment surge and failover. Bound concurrent requests per route/dependency, shorten transactions, remove unnecessary queries, batch where safe, and use read replicas only for reads that tolerate their consistency model. If the database is already saturated, reject/defer low-priority work, apply backpressure, and stop scaling the application tier beyond the safe concurrency limit. Scale the database only if its true bottleneck and supported scaling path justify it.

**Trade-offs:** Smaller pools can shift queueing to the application, where it is bounded and observable; larger pools may multiply blocked sessions. Aggressive autoscaling may never react quickly enough to a spike and may create a connection storm on recovery. Any pooler/proxy needs testing against transaction/session semantics.

**Signals and follow-up:** Monitor successful throughput, pool wait, DB queue/lock waits, p99, connection counts per instance and per database, and rejected work. Load-test scaling, node loss, and rolling deploys. Explain why “add replicas” or “increase pool size” is not a safe default.

### Scenario 7 — Notification provider unavailable for 45 minutes

**Problem:** An external provider is unavailable for 45 minutes while new email, SMS, and push requests arrive. The design must avoid losing accepted notifications, flooding the provider on recovery, and delivering obsolete or duplicate messages.

**Clarify semantics:** Distinguish “API request durably accepted,” “provider accepted the message,” and “recipient device received it.” Define per-message deadline and priority. A security OTP or fraud alert has different latency and expiry requirements from a promotional notification.

**Design:** Persist the notification intent, preferably using an outbox with the triggering business state, then publish to durable queues partitioned by channel/priority. Maintain provider-specific quotas, concurrency limits, backoff with jitter, and circuit-breaker behavior. Record attempt state and use stable notification IDs so retries and consumer redelivery do not create multiple logical sends. If the provider offers an idempotency mechanism or delivery-status API, incorporate its actual scope and limits.

**During outage:** Keep queue growth within storage and business-retention constraints. Expire work that can no longer be useful, retain urgent work within its deadline, and provide a visible pending status rather than claiming delivery. Separate channels so one provider outage does not block unrelated channels; failover to another provider only if content, consent, deliverability, and quota semantics are compatible.

**Recovery:** Do not release the entire backlog at once. Ramp delivery rate according to provider quotas and reserve capacity for live urgent messages. Reconcile unknown outcomes where the provider may have accepted a request before the timeout. Preserve idempotency during redrive and avoid retrying permanent address/template/auth failures.

**Signals and follow-up:** Track queue depth and oldest age by priority/channel, delivery success, attempt count, provider throttling, expiry, cost per successful delivery, and estimated drain time. Explain what the API promises and how the system behaves when a queued message becomes stale.

### Scenario 8 — Redis outage overloads the database

**Problem:** Redis becomes unavailable or extremely slow. The application falls back to the origin database, which was sized for cache misses rather than full read traffic, and the fallback itself causes a cascading failure.

**Diagnose:** Estimate uncached QPS and compare it with tested database capacity. Determine whether Redis has a total outage, partial network issue, connection exhaustion, hot-key problem, or elevated command latency. Inspect cache client timeouts/retries, miss/refill concurrency, application thread pools, DB pool wait, and whether all instances began retrying together.

**Design:** Treat cache failure as an explicit operating mode. Bound cache timeouts, cap concurrent origin reads, coalesce identical misses, open a circuit for a failing cache path where appropriate, and shed low-priority reads before the database is overwhelmed. Serve stale values only when the data's freshness, privacy, and correctness contract permits it. A cache holding session or lock state cannot simply be bypassed like a public catalogue cache.

**Recovery:** When Redis returns, avoid warming every key from every instance simultaneously. Ramp traffic gradually, prioritize hot keys, use TTL jitter, and monitor eviction/fill rates. If Redis failover can lose recent writes under the chosen persistence/replication settings, ensure the application does not treat lost coordination or workflow state as authoritative. Business state should remain in a durable source of truth when correctness matters.

**Trade-offs and signals:** Fail-open to the database is safe only if the database can sustain it or the system has bounded degradation. Track Redis latency/errors, cache hit ratio, origin QPS, DB saturation, queue age, rejected requests, stale responses, and p99. Test the full outage and cold-cache path ahead of time. Explain why the cache is optional acceleration or, if not, how its semantics change the fallback decision.

### Scenario 9 — One tenant dominates the platform

**Problem:** One customer sends a traffic burst or runs expensive queries, increasing latency for other customers in a shared SaaS platform. A global rate limit exists but does not sufficiently isolate tenants.

**Diagnose:** Attribute requests, CPU/query cost, concurrency, storage growth, queue depth, cache-hit rate, and errors by tenant. Determine whether the load is abusive, an expected burst within the contract, a query bug, a hot key, or a customer large enough to outgrow shared capacity. Count both request rate and cost per request: ten expensive exports can be worse than thousands of cache hits.

**Design:** Enforce tenant identity at the resource-owning service. Apply per-tenant rate and concurrency limits, bounded queues, weighted fair scheduling, query budgets, and separate pools for bulk jobs. Global limits protect platform capacity, while per-tenant budgets prevent a single customer from consuming all of it. Honor contractual burst allowances and reserve capacity for critical tasks such as recovery or security events.

**Isolation tiers:** Shared database/shards are cost-efficient for ordinary tenants. Persistently large tenants may need dedicated shards, pools, or worker capacity; this requires a routing map, migration strategy, cost model, and a plan for tenant-wide queries. Splitting one tenant across buckets helps hot writes but complicates transactions and aggregation.

**Security:** Tenant filtering must not depend only on client-supplied parameters. Include tenant and authorization scope in cache keys and ensure events, search indexes, object storage, exports, and administrative tools preserve the boundary. Test that one tenant cannot read or overwrite another tenant's data.

**Signals and follow-up:** Track per-tenant p99, resource cost, throttling, queue age, and cross-tenant error rates. Explain how you decide between throttling, isolation, and a paid dedicated tier without creating unfairness or data leakage.

### Scenario 10 — Zero-downtime migration of a heavily written table

**Problem:** A large, heavily written table must change schema or move to a new representation without blocking production traffic, losing updates, or leaving old and new application versions incompatible.

**Plan:** Use expand–migrate–contract. First add the new column/table/index in a backward-compatible way and deploy code that can coexist with both representations. Do not remove or reinterpret the old field while any supported process still reads it. For indexes or constraints that may lock large tables, use the database's online/concurrent facilities where supported and verify their operational limitations.

**Backfill with a boundary:** Take a consistent snapshot or capture a high-water mark/log position. Copy records in bounded, checkpointed, idempotent batches, then apply concurrent changes through CDC or a controlled dual-write protocol. A naive dual write can commit to one side and fail on the other, so its repair and authority model must be explicit. Use versions/conditional writes so an old backfill value cannot overwrite a newer production update. Capture deletes too.

**Validate:** Compare counts and sampled/full checksums where feasible; check nulls, constraints, version mismatches, application-visible results, and replication lag. Shadow-read the new representation if practical. Rate-limit migration work to protect production query latency and define pause thresholds.

**Cut over and rollback:** Shift reads to the new representation gradually, monitor correctness and p99, then shift writes once compatibility is proven. Keep the old representation until the rollback window is complete. After the new representation has accepted writes that the old one cannot understand, rollback may require reverse replication or forward repair rather than simply reverting application code.

**Signals and follow-up:** Track backfill progress/lag, lock time, DB load, mismatch counts, errors, and versions still writing the old shape. “Zero downtime” means no planned user-visible interruption, not zero risk; explain how each crash window resumes safely.

### Scenario 11 — Distributed lock expires while work continues

**Problem:** Worker A acquires a 30-second lease to process an account. It pauses for 45 seconds during a long runtime pause or network partition. Worker B sees the lease expire and acquires ownership, then A resumes and attempts to commit stale work. Both workers may believe they are acting legitimately.

**Invariant:** Only the current owner may commit a mutation to the protected resource. A lock service that records ownership is insufficient if the target database or external system still accepts writes from a stale owner.

**Design:** Where possible, express the invariant directly in the authoritative store using a conditional update, unique constraint, or transaction. If a lease is required, allocate a monotonically increasing fencing token/epoch on each ownership change. Every write carries that token, and the protected resource rejects tokens older than the latest accepted one. Checking the lease locally just before writing is not enough because ownership can change between the check and the write.

**Crash and external-effect windows:** A may have already called an external provider before its lease expired. If that provider cannot validate fencing tokens, use a stable idempotency key, durable operation state, and status query/reconciliation. A fencing token cannot undo an email, payment, or external mutation already accepted. Avoid holding distributed locks across long network calls; split work into durable, resumable steps where possible.

**Recovery and signals:** Define lease duration/renewal, owner identity, behavior when the lock service is partitioned, and safe cleanup of abandoned work. Track lease expiry, duplicate processing, stale-token rejection, lock contention, and jobs stuck in progress.

**Follow-up:** Explain exactly where the stale write is rejected. “We use Redis locks” is incomplete unless the resource being protected can distinguish the current owner from an expired one.

### Scenario 12 — p99 latency is poor but averages look healthy

**Problem:** Average request latency is 80 ms, but p99 is 2.5 seconds. Most users see a fast response, while a small but important fraction experience stalls or timeouts. The mean is hiding the tail.

**Diagnose:** Confirm the percentile's request population, sampling, time window, endpoint, region, and success/error treatment. Break the latency timeline into queue wait, connection-pool acquisition, application execution, database time, downstream calls, and network/serialization. Segment by tenant, payload size, cache hit/miss, request route, and peak versus off-peak periods. Averages can hide a hot key or one slow dependency serving only a subset of requests.

Check for queueing near saturation, database lock contention, slow query plans, thread/connection pool starvation, garbage-collection pauses, retries, network retransmissions, and synchronous fan-out. In a fan-out endpoint, the slowest required parallel dependency often determines the tail. For a serial chain, downstream latency accumulates.

**Design response:** Fix the dominant cause rather than optimizing average latency. Bound concurrency and queue depth, propagate an end-to-end deadline, prevent retries from exceeding it, isolate expensive tenant/work classes, and remove optional dependencies from the critical path when the API contract permits degradation. Optimize a query or partition key when a specific database path dominates. Parallelize only independent calls and cap fan-out so it does not create a new downstream overload.

**Validation:** Reproduce the load and traffic mix, then compare p50/p95/p99, successful throughput, error rate, queue/pool waits, and business completion. Validate with one controlled change at a time and include degraded-dependency tests.

**Follow-up:** Low CPU does not rule out saturation. Many threads can be waiting on locks, sockets, or connections. Explain which metric tells you where time is spent, not just that you would “check logs.”

### Scenario 13 — Multi-region partition during financial writes

**Problem:** Regions A and B lose connectivity, but clients can still reach both. Each region receives debit/refund requests for the same accounts. If both regions accept writes independently, the ledger may diverge or the same logical payment may be applied twice.

**Clarify the contract:** Which store is authoritative for a ledger entry? Does an acknowledged response promise durable commit, or merely acceptance for later processing? What RPO is allowed? Can the system determine current write ownership during the partition?

**Design:** Use a datastore/protocol that provides the required cross-region transaction guarantee, or route writes for each account/ledger to one authoritative region or partition. Replicate reads to other regions where the freshness contract permits. Ownership changes require an explicit quorum/leader protocol and fencing so the old writer cannot resume after a pause. Health checks and DNS changes do not establish exclusive write authority.

If the owner is unreachable and ownership cannot safely transfer, reject the write or durably record a **pending command** without claiming the debit/refund has committed. Do not queue arbitrary financial mutations indefinitely: the command may be stale or invalid when connectivity returns. Preserve the original idempotency identity and validate current account state before executing it. For external payment effects, query/reconcile the provider before retrying.

**Trade-off:** Strong coordination can reduce write availability during partitions and add cross-region latency. Allowing both sides to accept writes improves availability only if the data model can safely merge the operations; financial debits generally cannot be treated as a simple last-write-wins register.

**Recovery:** Fence one owner, reconcile logs/provider transactions, repair derived read models, and expose unresolved outcomes for investigation. Monitor owner changes, rejected/pending writes, unknown payment outcomes, and ledger mismatches.

**Follow-up:** Distinguish active-active compute, multi-region reads, single-writer-per-entity, and true multi-writer writes. They provide different guarantees.

### Scenario 14 — New microservice makes requests slower

**Problem:** A feature previously made one in-process call but now calls three new services. P99 latency rises, timeouts increase, and failures in one service affect the entire user journey. The extraction may have improved ownership without improving runtime performance.

**Diagnose:** Compare distributed traces before and after the change. Separate network/serialization cost, DNS/service discovery, TLS, connection-pool acquisition, sequential versus parallel calls, duplicated database queries, retries, and the downstream processing time. Check whether the new service actually owns its data or only adds a network hop to a shared database. Identify whether the tail comes from one required dependency or many small sequential costs.

**Architectural choices:** Coarsen an API to return the data needed for one user operation instead of making a chatty series of calls. Reuse persistent connections, propagate deadlines, enforce concurrency limits, and parallelize independent calls only where semantics allow. For a high-volume composite read, consider a materialized read model to avoid repeated fan-out, accepting its propagation lag and rebuild complexity. Keep business authorization at the resource-owning service; do not bypass ownership merely to shave latency.

If the service boundary provides no independent scaling, deployment, security, or ownership benefit, a module within the same deployment may be better. Alternatively, keep the service boundary but move noncritical work off the synchronous path. Avoid solving latency by indiscriminately increasing timeouts; that often ties up more threads and connections.

**Trade-offs and validation:** Measure end-to-end latency and availability, not each service's local metric alone. Load-test with network delays and one slow dependency. Monitor fan-out ratio, pool wait, retry multiplier, p99, and resource consumption.

**Follow-up:** Explain why microservices are not automatically faster and which boundary benefit justifies the additional partial failures and operational overhead.

### Scenario 15 — Retry policy creates an outage

**Problem:** The client, gateway, service, SDK, and worker each retry a slow dependency. Three attempts at multiple layers can multiply downstream load instead of merely adding a few extra requests. The dependency slows further, causing more timeouts and a retry storm.

**Quantify the amplification:** If four layers each permit three attempts and all attempts reach the next layer, one original call can theoretically cause up to (3^4 = 81) attempts at the deepest dependency. Real paths may stop early, but the example shows why independent retry policies cannot be considered in isolation. Include background retries and replay traffic in the capacity model.

**Diagnose:** Compare original logical request rate with physical attempt rate, timeout and latency distributions, per-layer retry counters, concurrency, queue age, and dependency saturation. Determine whether timeouts are due to a transient network issue, overload, or uncertain mutation outcome. A timeout does not imply that the remote side effect failed.

**Design:** Choose a clear retry owner where possible, propagate an overall deadline, cap attempts, and use exponential backoff with jitter. Respect retry-after instructions and use a retry budget during incidents. Retry only plausible transient failures, and require idempotency or stable operation identity before retrying mutations. Apply bulkheads, admission control, circuit breakers, and load shedding. Do not allow all retries to immediately resume at full rate after recovery.

**Recovery:** Reduce retry traffic first, recover the dependency, then increase traffic gradually. Reconcile uncertain external effects before replaying. Ensure DLQ/redrive work cannot compete without bounds with live business traffic.

**Validation:** Inject dependency latency/outage and test recovery, not merely the first failure. Track attempts per logical operation, retry-generated QPS, successful throughput, error budget burn, and time-to-recover.

**Follow-up:** A policy can improve individual-request success probability while making system-wide availability worse. Explain that system-level trade-off.

### Scenario 16 — API returns success before work is durable

**Problem:** The API returns `201 Created` or `202 Accepted` before the database commit finishes or before a job is durably accepted by the broker. The process crashes, and the client believes work exists when it has been lost.

**Clarify semantics:** `201 Created` normally represents creation of a resource; `202 Accepted` means processing was accepted but may not be complete. Neither status code itself proves durability. The contract should name the exact boundary: persisted command, committed business state, broker acceptance, provider acceptance, or completed business outcome.

**Design for synchronous creation:** Commit the record and required state transition before returning success. If an event must be emitted, use a transactional outbox so the business state and publication intent commit together. Publishing to a broker before the database transaction commits can expose a non-existent entity; publishing afterward in a separate step can lose the event if the process crashes.

**Design for asynchronous work:** Persist an operation/job record (and outbox row when needed) in a durable transaction, then return an operation ID and pending state. A worker claims the durable job, processes idempotently, records progress, and exposes succeeded/failed/expired states. In-memory task submission alone is not durable acceptance. If the broker itself is the acceptance boundary, document its acknowledgement and replication guarantees.

**Failure windows:** Test crash before commit, after commit but before publish, after broker publish but before marking the outbox row complete, and after durable acceptance but before the HTTP response reaches the client. Use idempotency keys so a client retry returns the existing operation rather than creating a duplicate. Expect duplicate publication and make consumers idempotent.

**Follow-up:** Explain what a success response promises and how you prove the operation can survive a process, node, or broker failure.

### Scenario 17 — Strongly consistent payments and eventually consistent analytics

**Problem:** The payment domain needs correct state transitions, ledger entries, and idempotent processing, while dashboards and analytical reports can tolerate delayed updates. The design should not couple payment authorization latency to the reporting stack, nor let a lagging report decide a financial action.

**Source of truth:** Keep payment/order financial state in an authoritative transactional store with constraints and explicit state transitions. The payment path should atomically update its durable business state and record publication intent, commonly via a transactional outbox. A publisher or CDC pipeline sends changes to a broker; consumers update analytical storage and report projections.

**Consistency contract:** The outbox prevents the database change and event intent from diverging, but it does not give exactly-once end-to-end processing. Publish can be repeated after a crash, so consumers must deduplicate/idempotently apply events. Version checks and ordered per-entity processing prevent stale status updates. Reports should surface an “as of” timestamp/freshness SLO and not be used to approve refunds or authorize debits.

**Failure handling:** Monitor outbox backlog, oldest unpublished event, consumer lag, projection errors, and ledger-to-report reconciliation. If analytics is behind, payment operations continue while the dashboard shows delayed freshness. If the source transaction fails, no committed outbox event should claim the payment change happened. If the event is published twice, the projection should not double-count it.

**Recovery:** A corrupted projection should be rebuildable from authoritative data or retained events using a versioned snapshot boundary, with concurrent changes captured before cutover. Preserve delete and correction semantics; a financial correction should be represented as an auditable adjustment rather than silently rewriting history.

**Follow-up:** Explain why strong consistency is scoped to a critical business boundary rather than imposed on every downstream consumer.

### Scenario 18 — Incompatible event schema deployment

**Problem:** A producer begins emitting a new event shape that an older consumer cannot parse or interpret. The consumer retries, lag grows, and a partition may become blocked. The failure can be syntactic (deserialization) or semantic (the same field now means something different).

**Diagnose:** Identify schema versions in flight, producer/consumer deployment order, registry compatibility mode, replayed historical payloads, DLQ growth, and which partition/key is affected. A schema registry can enforce structural rules, but it cannot prove that a field's business meaning is unchanged.

**Safe rollout:** Prefer additive optional fields and stable semantics. Deploy consumers that tolerate both old and new forms before producing the new form. Set safe defaults and ensure old consumers can ignore unknown fields if the serialization contract permits it. For a breaking change, create a new event type/version and migrate consumers deliberately rather than repurposing a field in place.

**Recovery:** Preserve the original payload, schema ID/version, error, and event identity. Do not retry deterministic schema errors forever. Quarantine malformed messages and replay only when a compatible consumer is deployed; maintain ordering if downstream events for that entity depend on the missing event. If a projection was partially updated, make its replay idempotent and reconcile.

**Rollback:** The old producer may be safe to restore only if new consumers accept old events and any data written by the new version remains readable. Keep mixed-version compatibility through the rollback window. Test old-producer/new-consumer and new-producer/old-consumer combinations, including historical replay.

**Signals:** Monitor deserialization failures, schema-version distribution, DLQ age, per-partition lag, and completion SLO. In an interview, explain rollout order and rollback compatibility—not just “use schema registry.”

### Scenario 19 — Rate limiter becomes the bottleneck

**Problem:** Every incoming request calls a shared rate-limit store or service, adding a network round trip and contending on hot counters. The limiter's p99 becomes the API's p99 or the limiter itself becomes unavailable during traffic spikes.

**Diagnose:** Measure checks per request, limiter latency, hot-key contention, key cardinality, store CPU/network, atomic operation cost, and the fraction of end-to-end latency spent waiting for quota decisions. Confirm whether the quota must be exact globally, exact per tenant, or only approximate abuse protection.

**Options:**
- **Central atomic limiter:** simple global semantics and accurate shared accounting, but it adds latency and can create a hot store/key.
- **Local token bucket:** fast and resilient, but each instance can independently grant tokens; autoscaling can multiply the aggregate allowance.
- **Hierarchical budget allocation:** a global controller leases bounded token budgets to local instances. Requests are then decided locally while total overshoot is bounded by the distributed allocation, but some capacity may temporarily be stranded or unfairly distributed.
- **Edge plus business quotas:** coarse IP/network limits reject obvious abuse early; authenticated tenant/account limits and expensive-operation budgets are enforced closer to the API owner.

**Correctness and failure:** Define behavior on store outage and partition. A hard billing quota may need fail-closed behavior; an approximate anti-abuse limit may permit a conservative local fallback. Make the limit scope match the provider's actual quota and include burst semantics. Rate limiting controls requests per time; concurrency admission controls simultaneous expensive work and can be needed even with an acceptable rate.

**Signals and follow-up:** Track allowed/rejected traffic, per-key skew, store latency/errors, quota overshoot, and autoscaling changes. Load-test hot tenants and limiter failover. Explain how the design avoids a limiter dependency taking down every endpoint.

### Scenario 20 — 10× spike with a limited budget

**Problem:** A campaign may create ten times normal traffic, but continuously provisioning every component for the peak is too expensive. The design must absorb bursts without letting optional work or one hot dependency overwhelm critical journeys.

**Clarify the shape:** Ask how quickly demand ramps, how long the spike lasts, whether requests are read-heavy or write-heavy, what percentage can be deferred, which journeys are revenue- or safety-critical, and what rejection/latency target is acceptable. Ten times traffic for 30 seconds is different from ten times traffic for six hours.

**Capacity and design:** Separate baseline from burst capacity. Cache immutable or safely stale content, precompute expensive reads, use a CDN for eligible public/static responses, and avoid proxying large object bytes through application servers. Queue deferrable work with bounded retention and priority separation. Reserve capacity for critical writes, put quotas around expensive tenants/routes, and use admission control and load shedding when the safe concurrency limit is reached.

**Autoscaling limits:** Provisioning is not instant. New instances need startup, readiness, connection pools, and cache warm-up; a database or provider quota may not scale with application instances. Pre-warm minimum capacity for known campaigns, use gradual ramping, and test cold-cache/dependency-degraded conditions. More app instances can actually create more DB connections and downstream traffic.

**Trade-off:** Overprovisioning reduces rejection risk but increases cost. Load shedding protects critical flows while some work is rejected/deferred. Precomputation reduces read-time cost but introduces freshness and rebuild responsibilities. Do not accept work durably if storage, quota, or business deadlines make completion implausible.

**Signals:** Monitor offered versus successful traffic, p99, cache misses, queue age, per-dependency saturation, rejected work by priority, error-budget burn, and cost per successful operation. Use a representative load test to prove the peak path, recovery, and rollback—not only steady-state autoscaling.

### Scenario 21 — Millions of queued events after an outage

**Problem:** A downstream dependency was unavailable for several hours. It has recovered, but millions of retryable events are waiting. Replaying at full speed could create a second outage, exceed an external quota, or starve fresh interactive traffic.

**Clarify the workload:** Establish backlog size and age, current arrival rate, successful completion rate, provider quota, event deadlines, and which event types are critical. Determine whether the queue contains only retryable work or also permanent failures and records whose business meaning has expired.

**Quantify net drain:** If the worker can successfully complete 18,000 events/s and new arrivals continue at 7,000/s, the net drain is 11,000/s. A 3.3-million backlog takes about 300 seconds under stable conditions. If completion rate is at or below ongoing arrivals, backlog cannot shrink; extending retention or adding consumers alone does not solve the deficit.

**Recovery design:** Ramp replay gradually and cap worker concurrency against downstream capacity. Reserve a portion of provider quota for new urgent requests, separate priorities/channels, and process oldest or most deadline-sensitive work according to the business contract. Use exponential backoff with jitter and respect provider retry-after signals. Ensure retries use stable operation IDs/idempotency so events that may already have succeeded are not repeated as new actions.

**Data and ordering:** Preserve per-key ordering where required, quarantine poison messages, and expire obsolete work explicitly. If the downstream side effect is uncertain, query status or reconcile before replay. Do not blindly replay the entire backlog after a consumer reset.

**Signals and follow-up:** Track oldest-event age, arrivals/completions, net drain rate, provider throttling, retries, duplicates, and completion SLO. Explain how you will protect live traffic and what you will do when the predicted drain time exceeds the event's useful deadline.

### Scenario 22 — Duplicate notifications after consumer rebalance

**Problem:** A consumer group rebalance occurs during deployment or a slow processing interval, and users receive the same notification twice. Rebalances change partition ownership; they do not provide exactly-once semantics for arbitrary provider side effects.

**Trace the operation:** Follow one stable notification ID from the source transaction through outbox publication, broker topic/partition/offset, consumer attempt, provider request, and delivery callback. Determine whether the duplicate came from producer retry, offset reset, a crash after provider acceptance but before offset commit, reprocessing of a partition, or an HTTP timeout after the provider sent the message.

**Design for idempotent handling:** Persist the logical notification ID and processing state in a durable inbox/state table. Coordinate the local state transition and processed-event record in one database transaction, then commit the broker offset according to the consumer model. If the process crashes after local commit but before offset commit, redelivery is recognized as already processed. If the provider accepts a send and the consumer crashes before recording that result, the local transaction alone cannot prevent a duplicate external side effect.

Pass a stable provider idempotency key where supported; record provider message IDs, response state, and callback evidence. If the provider does not support idempotency and a timeout leaves the outcome uncertain, query status or reconcile when possible and document the residual duplicate window. Avoid acknowledging before the notification is durably recorded, but also avoid holding a Kafka partition while performing unbounded slow provider calls.

**Operational controls:** Use bounded provider concurrency, deadlines, retry jitter, and per-destination quotas. Monitor rebalance frequency, processing duration, duplicate rate, unknown provider outcomes, and attempt counts. Test crashes before/after provider send and before/after offset commit.

**Follow-up:** Distinguish deduplication of the event, idempotency of the database transition, and exactly-once external delivery. Those are three different guarantees.

### Scenario 23 — Search shows stale price after product update

**Problem:** A product price changes, but search results continue to show the old price while the customer attempts checkout.

**Clarify:** Is the search index a derived projection? What freshness SLO is promised? Are updates published from the transaction through an outbox/CDC, and can events be duplicated or reordered? Is the displayed price informational, or is it reused as the amount charged?

**Design:** Keep the authoritative price and checkout validation in the owning transactional store/service. Publish price changes reliably through an outbox or CDC pipeline; make index updates idempotent and version-aware. A projection update should not overwrite a newer document with an older event. Monitor outbox age, consumer lag, failed indexing, and source-to-index freshness.

**Correctness:** Treat search price as a discovery/display value, not authorization to charge. At checkout, revalidate current price and inventory against the authoritative owner. If the price changed, return an explicit price-change result and require confirmation when the product contract demands it; do not silently charge a different amount. If the business needs a quote that remains valid for a period, create an explicit versioned price quote or reservation with expiry rather than relying on cached search data.

**Recovery:** Rebuild or replay the index from authoritative data if updates were lost or the schema changed. Use a consistent snapshot boundary, capture concurrent changes, validate document counts and representative queries, then switch index generations. Tombstones/version checks prevent delayed updates from resurrecting deleted or replaced products.

**Signals:** Track index lag, stale-price reports, version regressions, update failures, source/index mismatches, and checkout price conflicts. In an interview, distinguish projection freshness from transaction correctness and explain how customers are protected while the index catches up.

### Scenario 24 — Regional failover creates a routing loop

**Problem:** Region A forwards traffic to Region B after a health check fails, while B's local routing rule forwards the request back to A. Requests bounce between regions, increasing latency and potentially multiplying traffic during an outage.

**Clarify routing ownership:** Determine which layer makes the global region decision (DNS/global traffic manager, edge proxy, or application router), whether regional routing is allowed to forward across regions, and how the system represents a region's state: healthy, degraded, draining, failed, or recovering. A local service should not independently override global ownership without a defined rule.

**Design:** Separate global traffic policy from regional/service-level routing. Make the failover decision explicit and idempotent, and ensure a region marked failed/draining is not selected again by another region's fallback policy. Include a hop count or visited-region context as a safety guard, but treat that as defense in depth rather than the primary routing design. Use request IDs and region-routing metadata to detect repeated forwarding and stop loops before exhausting resources.

Distinguish ingress health from the ability to complete a user journey. A region may accept TCP connections but fail to reach its identity provider, database, or external dependency. Conversely, a partial failure may justify degraded local service rather than redirecting every request globally. Avoid health rules that react to transient noise and create failover oscillation.

**Stateful data:** Routing is not write ownership. Before sending financial writes to another region, fence the previous writer, verify replica eligibility/RPO, and ensure one authoritative owner. Existing connections and WebSockets may not move when DNS changes.

**Recovery and signals:** Make failback a separate controlled transition after data is current. Track cross-region hops, forwarding loops, decision reasons, route convergence, region states, and retries. Test partial connectivity and failback, not only complete region shutdown.

### Scenario 25 — Retention doubles storage requirements

**Problem:** A policy change doubles retention from 90 to 180 days, and the original storage budget no longer fits.

**Estimate the impact:** Recalculate steady-state growth for each data class using arrival rate × average physical bytes × retention, then account for indexes, replicas, snapshots, logs/WAL, derived read models, object versions, backups, and temporary space during migration. For example, doubling retention may nearly double raw retained payload, but total physical cost can grow by more or less depending on compression, indexes, compaction, and copy count. Measure actual bytes per record rather than assuming payload size equals disk usage.

**Separate storage tiers:** Identify which records must remain low-latency and queryable, which can be time-partitioned and archived, and which can move to cheaper object/archive storage. Include retrieval latency, minimum storage duration, egress, restore time, and the effect of archive access on product/API promises.

**Correctness and compliance:** Confirm retention and deletion rules for primary data, replicas, caches, search indexes, analytics, logs, and backups. Legal holds may override ordinary expiry. Do not simply delete old audit records or shorten retention to fit budget without approval. For privacy deletion, define how data in backups eventually expires and how a restore avoids reintroducing data that should remain deleted.

**Migration plan:** Estimate the one-time capacity needed to reindex, copy, or repartition data; throttling may be needed to protect production. Define checkpoints, validation, rollback, and the order in which lifecycle rules are activated.

**Signals and follow-up:** Track storage growth forecast, partition/archive lag, retrievals, cost per retained GB, restore time, and policy exceptions. A senior answer balances storage cost with searchability, auditability, residency, and recoverability; raw storage is only one part of the decision.

### Scenario 26 — Global service needs low latency on multiple continents

**Problem:** Users are globally distributed, and round trips to one region's database dominate latency.

**Clarify:** Which data is public/cacheable? Which reads need freshness? Which entities are written globally? What data-residency restrictions apply, and what RTO/RPO is required for regional failure? Does the workload need truly concurrent multi-region writes or only active-active compute?

**Design:** Use CDN/edge caching for eligible content and stateless compute near users. Read-only or derived regional replicas can serve local requests when their freshness contract permits it. Assign an authoritative writer for each entity/partition or choose a datastore with documented global-consistency semantics. Avoid synchronous cross-region calls on every request unless the invariant requires coordination; those round trips add latency and tie availability to inter-region links.

**Write architecture:** Active-active compute can route writes for a particular account or inventory item to one owner region, preserving a single serialization point while allowing nearby reads. True multi-writer requires explicit conflict prevention/resolution and is unsafe for every domain. During a partition, the system may have to reject a payment/inventory write rather than accept conflicting state on both sides.

**Failover:** DNS/global routing affects new connections and converges according to TTL/client behavior; it does not transfer write authority. Verify replica position, fence the old writer, promote a safe candidate, refresh routing, and reconcile uncertain in-flight operations. Failback must ensure the recovered region is current before ownership returns.

**Trade-offs and signals:** Regional copies improve read latency but introduce lag, residency, cost, and recovery complexity. Measure p99 by region, cross-region traffic, replication lag, error budget by journey, routing convergence, and failover/failback correctness. In an interview, distinguish global traffic routing from data ownership and explain the consistency guarantee for each operation.

### Scenario 27 — External provider imposes a strict quota

**Problem:** A provider permits 500 calls/second per account, while the application scales across dozens of instances. A per-instance limiter appears healthy but the aggregate exceeds the provider quota, causing 429 responses and retry amplification.

**Clarify the actual limit:** Is the quota per account, API key, endpoint, region, or resource? Are bursts allowed? Are read and write calls counted together? Does the provider publish rate-limit headers or retry-after guidance? Which calls are urgent and which can wait?

**Design:** Enforce a shared budget at the same scope as the provider's quota. A central limiter is straightforward but adds request-path latency and can become a hot dependency. A hierarchical design can allocate bounded token budgets to instances/workers, reducing per-call coordination while limiting aggregate overshoot. Reconcile budgets when instances scale in/out so unused tokens are not permanently stranded and new instances do not start with a full quota.

Queue deferrable calls with bounded depth, use priority classes, coalesce duplicate requests, and cache only data the provider contract permits to be cached. Bound concurrency as well as calls/second: a slow provider can exhaust in-flight work even when the rate is within quota. Respect retry-after responses and use jitter; immediate retries convert throttling into more throttling.

**Failure behavior:** Define what happens if the quota service is unavailable. Strict contractual limits may require fail-closed behavior; a conservative local fallback may suit an approximate protective guard. Persist operation identity for non-idempotent provider calls and do not retry an uncertain mutation with a new ID.

**Signals and follow-up:** Track aggregate call rate, 429s, quota consumption, limiter latency, queue age, attempts per logical operation, and per-operation completion. Load-test autoscaling and region failover; the quota must remain global even when traffic shifts between instances or regions.

### Scenario 28 — Infrastructure is healthy but data is inconsistent

**Problem:** CPU, memory, availability dashboards, and health probes are green, but users report missing or contradictory records.

**Start from a concrete example:** Pick one affected entity and trace its operation ID/version through the authoritative database, transaction/outbox, broker, consumer, replica, cache, search index, and API response. Identify the expected invariant and the actual conflicting states. Do not begin by restarting services without evidence; a restart may hide a stuck queue or create more duplicate processing.

**Likely causes:** Replica lag, a stuck outbox publisher, consumer offsets committed before the side effect, duplicate/out-of-order events, stale backfills, missing tombstones, schema-version mismatch, cache refill races, or an unsafe failover with split write ownership. Check whether the API promised strong/read-your-writes behavior or allowed a bounded staleness window.

**Containment and repair:** If the issue threatens payments, authorization, or inventory correctness, stop the unsafe decision path first. Identify the authoritative source and repair projections from it using version-aware idempotent updates. Reconcile unknown external outcomes rather than choosing whichever copy has the newest wall-clock timestamp. Preserve audit evidence and avoid re-running irreversible side effects during replay.

**Signals:** Add source-to-read freshness, outbox age, oldest consumer event, replication position/lag, duplicate/conflict counts, version regressions, and reconciliation outcomes. Infrastructure health proves that components respond; it does not prove business state is correct.

**Follow-up:** Explain which invariant was violated, how you found the first divergence, how the repair avoided overwriting newer state, and what alert/test would catch the problem earlier.

### Scenario 29 — Team wants a new database because it is popular

**Problem:** A team proposes adopting a popular database because it is widely discussed or used by another company, but has not shown that the current datastore cannot meet the workload requirements.

**Start with evidence:** Describe access patterns, joins/aggregations, transaction boundaries, consistency needs, data size and growth, peak read/write rate, key distribution, p95/p99 objective, retention, RPO/RTO, security, and residency. Use production query plans, lock/pool metrics, storage/IOPS, replication lag, and load tests to establish the current limitation. A large table alone does not prove that the data needs a new engine.

**Compare alternatives fairly:** First consider tuning queries/indexes, correcting N+1 behavior, shortening transactions, partition pruning, archival, read replicas for eligible reads, cache or read-model design, connection management, or vertical scale. Then compare candidate stores with representative data and key skew. Test not only throughput but consistency, failover, backup/restore, schema evolution, hot-key behavior, observability, upgrades, and operational effort.

**Model migration cost:** Moving data requires dual-read/write decisions, snapshot and change capture, reconciliation, rollback, compatibility, and downtime or consistency trade-offs. A faster benchmark can be a net loss if it makes recovery slower or demands skills the team cannot operate safely. Include total cost: service pricing, replicas, network/egress, licenses, on-call, migration, and eventual exit.

**Decision rule:** Adopt the new store only when the measured benefit matters enough to justify the added operational and correctness burden. If the real issue is a missing index, hot key, or overloaded connection pool, solve that first.

**Follow-up:** Define a benchmark workload, success thresholds, failure tests, and rollback/exit criteria before committing to a proof of concept.

### Scenario 30 — Interviewer asks “What breaks first?”

This is a capacity and failure-model question, not an invitation to guess one component. Start with a workload model and explicitly state assumptions: peak request rate, request mix, fan-out, read/write ratio, payload sizes, cache hit ratio, and downstream/provider quotas.

**Estimate each tier:** Calculate edge/API requests, application concurrency, database queries and connections, write throughput, cache memory and origin misses, event production/consumption, storage growth, and external API calls. Use safe tested capacity rather than theoretical maximum. For a queue, compare arrivals and successful completions; for a database, consider lock contention and hot partitions as well as aggregate CPU.

**Rank bottlenecks:** Identify the component with the lowest sustainable margin under expected peak. A shared database or provider quota may saturate before stateless app CPU. A cache outage may multiply origin traffic, and retries may turn a modest latency increase into a traffic spike. If all tiers look safe in isolation, trace fan-out and connection multiplication across the whole system.

**Consider one failure:** Recalculate capacity after one instance/zone/region is lost and include deployment surge, cache cold start, or consumer replay. A system that handles peak only while every instance is healthy lacks resilience headroom. Explain which optional work is degraded, which requests are rejected early, and how critical correctness is preserved.

**Validate:** Name the load/stress/spike tests and metrics that would prove or disprove the hypothesis: successful throughput, p99, pool wait, per-shard skew, cache misses, queue age/drain time, dependency saturation, error-budget burn, and business invariants. State what you would optimize, isolate, cache, queue, shard, or throttle only after the evidence points to that bottleneck.

**Follow-up:** Senior-level reasoning is conditional: “Given these assumptions, database write contention is the leading risk; I would validate it with this test, and if it is not the bottleneck, I would revisit the next constraint.”

### Scenario 31 — Requirement changes from eventual to strong consistency

**Problem:** A product requirement changes from “updates usually appear within a few seconds” to “after a write succeeds, reads must return that write.” The existing cache, replica, or search architecture may not provide that guarantee.

**Clarify the contract:** “Strong” is ambiguous. Does the product need read-your-writes for the same user/session, monotonic reads, bounded staleness, or linearizability across all clients? Does the rule apply to every field or only a specific operation such as payment state or inventory allocation? Stronger guarantees have different latency and availability costs.

**Map the current data path:** Identify the authority, replicas, caches, event consumers, and derived read models. Determine where stale results can enter: asynchronous replication, delayed event propagation, cache invalidation, search refresh, or an API that reads from multiple stores. Do not assume switching a single query to the primary resolves a multi-service consistency contract.

**Design options:** Route read-after-write requests to the authority, use a database-provided session/commit token, or carry an entity version that downstream reads must meet. For cross-service workflows, expose explicit pending state or query the owner rather than inferring completion from an eventually consistent projection. Keep analytics/search eventually consistent where business requirements allow it; do not force all consumers onto the strongest model.

**Failure behavior:** If strong consistency requires coordination across regions, quantify extra round-trip latency and what happens during a network partition. The system may need to reject/block some writes instead of accepting divergent state. Confirm the selected datastore's actual semantics and failover guarantees; a “primary” label alone does not prove no acknowledged write can be lost.

**Validation:** Add read-after-write and concurrent-write tests, inject replica lag, and test failover. Monitor stale-read incidents, token/replication lag, primary-read volume, and p99. Update the API contract and capacity plan.

**Follow-up:** Explain which operation requires the stronger guarantee and why the rest of the system can remain eventually consistent.

### Scenario 32 — Global username uniqueness during a regional partition

**Problem:** Two regions lose connectivity but both can serve registration requests. The same normalized username is submitted in each region. Both sides locally see the name as available and attempt to create an account.

**Invariant:** No two committed accounts may own the same normalized username. An application-level “check then insert” is a race; uniqueness must be enforced at the authoritative write boundary. Be precise about normalization as well: case folding, Unicode normalization, whitespace, reserved names, and locale rules must be identical across regions.

**Design choices:**
- **Single uniqueness owner:** Route username claims to one authoritative region/service that enforces a unique constraint. This is simple and clear but adds cross-region latency and can reduce registration availability during a partition.
- **Globally consistent datastore/coordination:** Use a protocol that provides the required global uniqueness guarantee and understand its quorum/partition behavior. Do not assume global replication alone provides uniqueness.
- **Namespace partitioning:** Allocate disjoint name ranges/prefixes to regions if the product can accept that contract; ordinary user-chosen usernames often cannot use this approach without UX changes.
- **Pending registration:** During a partition, durably accept a registration request as pending only if the API explicitly promises deferred processing, not confirmed username ownership. Finalize after the authority can check uniqueness.

**Recovery:** Reconcile pending claims and ensure only one owner can commit. If an unsafe system has already accepted duplicates, there is no perfect technical merge: define conflict resolution, customer communication, and audit trail. Prevent dual writes through fencing and a unique index at the authoritative boundary.

**Signals and tests:** Track uniqueness conflicts, deferred registrations, owner changes, and duplicate detection. Test partitions with simultaneous same-name requests. In an interview, explain why choosing “availability” is not free when the invariant requires global uniqueness.

### Scenario 33 — Connection pool saturated while database CPU is low

**Problem:** Application requests wait several seconds for a database connection, but the database CPU dashboard looks low. Increasing the connection pool appears to help briefly, then creates more blocked work.

**Diagnose the wait stage:** Separate pool-acquisition delay from query execution and database lock waits. Inspect active/idle/waiting pool counts, connection hold duration, transaction age, leaked/unclosed connections, slow queries, blocked threads, and whether requests call remote services while holding a transaction. Low CPU may mean requests are waiting on locks/network or connections are held idle rather than performing work.

**Calculate the fleet-wide budget:** If 25 instances have a maximum pool of 40, the application can potentially create 1,000 connections before counting workers, migration jobs, other services, and rolling-deployment surge. Compare that number with the database's safe active-session capacity and reserve space for administration/failover. Autoscaling can multiply the pool cap quickly.

**Fix the underlying queue:** If connections are held by long transactions, shorten the transaction and move remote calls outside it. If one query holds locks, inspect its plan and contention. If there is a connection leak, fix lifecycle management. Bound acquisition time and per-route concurrency. A larger pool is justified only if the database can make productive use of more sessions; otherwise it simply moves the queue downstream.

**Validation:** Load-test with realistic query mix and failure/redeploy behavior. Monitor pool wait, hold time, active transaction age, database lock waits, open sessions, successful throughput, and p99. Test connection storms during failover, since many clients may reconnect at once.

**Follow-up:** Explain why database CPU, connection utilization, and lock contention measure different bottlenecks, and how your traces locate the actual wait.

### Scenario 34 — Search indexing falls behind during a campaign

**Problem:** A campaign creates a large burst of catalogue updates. The search index falls behind, so listings show stale price, inventory metadata, or product attributes even though the source database is current.

**Diagnose:** Compare event arrival rate with successful indexing rate, oldest-event age, per-partition skew, bulk-request latency, retries, shard/segment/merge pressure, and search-cluster CPU/disk. A high aggregate indexing rate can conceal one hot key or a poisoned partition. Adding consumers only helps if indexing capacity and partition parallelism are available.

**Architecture:** Keep transactional product/price state authoritative. Publish updates through an outbox or CDC pipeline so committed changes are not lost. Make index updates idempotent and version-aware; a delayed old event must not overwrite a newer document. Route high-impact changes (price/availability) separately or give them priority over low-impact descriptions if the business demands different freshness SLAs.

**Protect correctness:** Search is a discovery/read model, not the final authority for checkout. Checkout must revalidate current price and inventory and return a clear result if the data changed. If the user needs a price guaranteed for a window, create a versioned quote/hold with explicit expiry instead of relying on a search document.

**Recovery:** Scale indexing only within cluster capacity, batch where appropriate, reduce optional index work if needed, and drain the backlog gradually. Rebuild from authoritative records if change history was lost; establish a snapshot boundary and replay concurrent changes before switching generations. Include tombstones for deletes and version checks during replay.

**Signals:** Monitor source-to-index freshness, outbox age, indexing failures, version regressions, mismatch counts, p99 search latency, and checkout conflicts. In an interview, distinguish search freshness from transaction correctness and explain the user's behavior while indexing catches up.

### Scenario 35 — A scheduled job runs twice after failover

**Problem:** A scheduler triggers a billing, report, or cleanup job, then fails over. The old worker may still be running while a new scheduler believes the occurrence was missed and starts it again. Both can perform the same external side effect.

**Invariant:** Each logical scheduled occurrence should produce at most one intended business effect, even though scheduler delivery/execution may be duplicated. Define the occurrence identity precisely, usually from job name, scheduled instant, tenant/partition, and relevant business period—not a random worker attempt ID.

**Design:** Persist a durable job-run record with a unique constraint on the logical occurrence. A worker atomically claims or transitions the record to running, and records progress and terminal status. A lease can recover abandoned work, but it does not stop a paused old worker from resuming after expiry; use fencing if the protected resource can enforce the token. Make individual steps idempotent and record their effect identity.

**External effects:** If a job charges an account or calls a provider, the local unique run record cannot make that remote effect atomic. Pass a stable provider idempotency key where supported, persist the provider reference, and reconcile unknown outcomes before repeating the call. For non-idempotent legacy operations, design a compensation/manual-review path.

**Misfires and catch-up:** Decide whether missed runs are skipped, coalesced into one run, or replayed individually. A missed hourly report may be safely coalesced; missed billing periods may need each period processed. Rate-limit catch-up so it does not overwhelm the database or provider.

**Signals:** Track duplicate claims, stale fencing-token rejection, overdue/missed runs, run duration, retries, and ambiguous external outcomes. Test rolling deployment, process pause, scheduler failover, and crash after effect but before completion record.

**Follow-up:** A lock or cron setting alone is not an exactly-once guarantee; explain where duplicate business effects are prevented.

### Scenario 36 — Queue retention expires before consumers recover

**Problem:** A consumer outage lasts longer than the broker retention period. Some events expire before processing resumes, so the system may have missing state changes even though the broker itself is healthy.

**Clarify what the stream represents:** Are events the only durable record of business facts, a change feed for a rebuildable projection, or merely work instructions? If a message represents a required financial or operational action, silently losing it is very different from losing an old cache-refresh event. Identify the authoritative source and the business meaning of event expiry.

**Retention design:** Estimate outage detection, repair, and catch-up time, not just expected downtime. Retention must cover the worst credible recovery window plus margin if consumers rely on the stream for replay. Calculate bytes from event rate, payload size, compression, replication factor, partition/segment overhead, and expected burst. Check whether producer/consumer lag could exceed retention before a human notices.

**Recovery if history is missing:** Do not resume as though no gap occurred. If a durable source of truth or archive exists, take a consistent snapshot, rebuild the projection, and capture concurrent changes after the snapshot boundary. Validate counts, versions, tombstones, and business aggregates. For irreversible actions that cannot be reconstructed from current state, reconcile with an external ledger/provider or escalate the gap for manual repair.

**Operational controls:** Alert when the oldest unprocessed event approaches retention expiry; track per-partition lag and estimated time to expiry. Run catch-up capacity tests so backlog drains before the next retention boundary. Retention extensions during an incident may help only if storage capacity and broker behavior permit it.

**Follow-up:** Explain which data can be rebuilt from a snapshot and which requires event history. Retention is a recovery and correctness decision, not just a broker cost setting.

### Scenario 37 — Cached authorization remains valid after revocation

**Problem:** A user's permission is revoked, but a cached authorization decision or long-lived token still allows a sensitive operation. The cache improves latency but violates the security contract.

**Invariant and threat model:** Define the maximum acceptable revocation delay and the consequence of a stale allow. For a low-risk feature flag, a short delay may be tolerable; for account disablement, financial action, or sensitive data access, a stale allow may be unacceptable. Caching cannot be evaluated without the policy freshness requirement.

**Design options:** Use short-lived cached decisions, version permission state, publish reliable invalidation events, or perform an authoritative check for high-risk operations. A token with embedded claims remains valid until expiry unless revocation/introspection or a version check invalidates it; changing a database permission does not retroactively alter a signed token. Use cache keys that include user, tenant, resource, action, and relevant policy version.

Event-driven invalidation reduces delay but can be lost or delayed, so combine it with a defined TTL/safety net or authoritative recheck. If the invalidation channel is unavailable and freshness cannot be established, fail closed for sensitive routes. Do not invalidate only the local cache while a CDN, API gateway, or downstream service still holds the old result.

**Recovery and audit:** Track revocation-to-enforcement latency, permission-version lag, cache age, invalidation errors, and denied/allowed decisions after revocation. Test cross-tenant requests, user disablement, key rotation, and a cache/broker outage. Ensure audit logs capture the actor, resource, action, policy version, and decision without leaking secrets.

**Follow-up:** Explain how the revocation guarantee changes the cache strategy and where enforcement ultimately happens: at the resource-owning service, not just the UI or gateway.

### Scenario 38 — Rolling deployment duplicates scheduled work

**Problem:** During a rolling deployment, old and new application versions overlap. Both run the same periodic billing or cleanup task, causing duplicate effects or inconsistent output.

**Root cause:** A process-local timer is not a distributed scheduler, and deployment overlap is normal. Even leader election or a lock can leave a failure window if an old worker pauses, loses its lease, and later resumes after the new worker has claimed the task.

**Design:** Represent each logical occurrence with a deterministic identity, such as `(job_name, scheduled_instant, tenant_or_partition)`, and persist it under a unique constraint. The worker claims a state transition atomically. Use leases for recovery and fencing tokens if the side-effect target can reject stale owners. Each step should be idempotent so a crash after an effect but before recording completion does not repeat the logical action.

For external providers, pass the same stable idempotency key across retries and reconcile uncertain outcomes. If the effect cannot be repeated safely, do not replay blindly; use a status lookup or manual decision. Keep job state durable and observable rather than relying on in-memory timers.

**Misfire policy:** Decide what happens when a scheduled occurrence is missed during downtime: skip, run once now, or catch up each occurrence. Billing, data retention, and report generation may need different policies. Throttle catch-up and protect interactive traffic.

**Deployment and validation:** Drain old workers gracefully, prevent them from acquiring new jobs after readiness changes, and test the case where the old version remains alive but partitioned. Monitor duplicate occurrence claims, fencing rejections, job duration, missed intervals, and terminal failures.

**Follow-up:** Explain why “single scheduler instance” or “distributed lock” alone does not guarantee at-most-once business effects.

### Scenario 39 — Writes slow after adding indexes

**Problem:** A new index improves one read query, but write latency and replication lag increase. The database now performs extra index maintenance for inserts, updates, and deletes, and may also have different cache/I/O behavior.

**Diagnose:** Compare query plans and read latency before/after, but also inspect write p95/p99, index size, buffer/cache hit, page splits, storage I/O, lock waits, transaction duration, WAL/binlog volume, and replication apply rate. Verify that the new index is actually used and that it is not redundant with an existing composite index. Poor selectivity or a query that returns a large fraction of rows may not benefit much.

**Trade-off:** An index is a maintained data structure, not a free read shortcut. Each affected write updates the table and relevant indexes; wide keys and multiple indexes can add storage and write amplification. The extra index can be worth it if it prevents a much more expensive scan on critical reads, but the business workload includes both read and write cost.

**Approach:** Test representative data size and mixed read/write load. Use the actual execution plan and rows examined to decide whether the index helps. Consider a narrower or better-ordered composite index, query rewrite, covering index only if the included columns justify the size, or a different read model/search engine if the query semantics demand it. Remove an index only after verifying constraints and production query usage.

**Safe rollout:** For large tables, use online/concurrent index creation where supported and understand locking and disk-space requirements. Monitor the build, replication lag, and production p99. Define rollback and verify whether removing the index would break a uniqueness constraint.

**Follow-up:** Explain why “more indexes means faster database” is false and how you would measure net benefit under the actual workload.

### Scenario 40 — Fan-out request calls 20 downstream services

**Problem:** One API request calls 20 downstream services. P99 latency and availability are poor, and transient dependency failures cause many requests to fail despite the top-level service appearing healthy.

**Map the dependency graph:** Identify mandatory versus optional calls, serial versus parallel edges, shared failure domains, per-dependency latency, and the average number of attempts after retries. Serial calls accumulate latency; parallel calls finish near the slowest required branch, but consume more instantaneous concurrency. If every one of 20 independent dependencies is required and each is 99.9% available, the simplified combined availability is (0.999^{20} approx 98.0%). Correlated failures can make the actual result worse or otherwise differ from this model.

**Reduce fan-out:** Remove data that is not needed for the core response, create a coarser-grained API, cache reusable values with a safe freshness contract, or build a materialized view for frequently requested composites. Precomputation reduces runtime coupling but introduces projection lag, replay, deletion, and rebuild concerns. Do not put a derived read model in charge of a correctness-critical authorization or financial decision.

**Control the remaining calls:** Parallelize only independent calls under concurrency limits. Propagate one end-to-end deadline and allocate sub-budgets to required dependencies. Apply bulkheads and bounded queues, and avoid retry multiplication across clients, gateways, meshes, and services. Use a fallback only when the response remains correct and the API contract explicitly permits partial data.

**Validation:** Trace each downstream branch and track p99, timeout rate, attempts per logical request, concurrency, and fallback frequency. Test one slow/failed dependency and a peak-load scenario.

**Follow-up:** Explain which calls can be optional, which are true correctness dependencies, and whether a read model or simpler service boundary is worth the freshness and operational trade-off.

### Scenario 41 — HTTP 200 responses hide business failures

**Problem:** Dashboards report a high HTTP success rate, but customers report orders stuck in PENDING and downstream work is not completing.

**Clarify the contract:** Does HTTP 200 mean the business operation finished, or merely that the request was accepted? For a long-running task, durable acceptance with HTTP 202 may be correct, but the workflow must expose a status and completion semantics. A message broker acknowledgement, worker acknowledgement, and business completion are three different milestones.

**Diagnose:** Trace a stable operation ID through the request, database transaction, outbox, broker, consumer, external provider, and final state transition. Inspect the age/count of nonterminal states, state-transition failures, consumer lag, retries, DLQ, and outbox age. Check whether a consumer commits its offset before committing the database effect or whether a publisher is stuck.

**Repair:** Reconcile accepted operations that did not advance. Do not blindly mark them complete or replay them with a new identity. Query authoritative state and external provider status, then apply an idempotent transition or escalate ambiguous cases. If the failure affects correctness, pause the unsafe path before increasing throughput.

**Observability:** Add acceptance-to-completion latency, terminal-state success rate, age by workflow state, repeated transition failures, and unresolved unknown outcomes. Keep transport-level SLIs but pair them with business SLIs.

**Interview point:** State exactly what the API promises, which durable event represents acceptance, and which metric measures the promised outcome. A green infrastructure dashboard cannot prove that the business process is healthy.

### Scenario 42 — Consumer offset commits before database transaction

**Problem:** A consumer commits its broker offset before its database transaction commits. If the database transaction then fails or the process crashes, the broker considers the record handled and may not deliver it again. The business effect is lost even though the message is no longer pending.

**Correctness boundary:** The consumer's durable business update and its record of having processed the event must commit together. Merely disabling automatic offset commits does not solve every failure window; it only gives the application control over ordering.

**Preferred design:** Validate the message, then in one local database transaction apply the business change and insert the stable event ID into an inbox/deduplication table protected by a uniqueness constraint. Commit the transaction first, then commit/acknowledge the broker offset. If the consumer crashes after the database commit but before the offset commit, the broker redelivers the record; the inbox detects it and the consumer can safely acknowledge without repeating the effect.

**External side effect caveat:** If processing calls a payment/email provider, the database transaction cannot atomically include that provider. Persist an operation/outbox record and let a separate worker perform the external call using a stable idempotency key where available. Reconcile uncertain outcomes. Do not keep the Kafka transaction open while waiting on a slow provider unless the architecture explicitly supports and bounds that behavior.

**Operational concerns:** Preserve partition ordering, make batch processing atomic or track per-record outcomes carefully, and avoid advancing the offset beyond an unprocessed earlier record when order matters. Monitor commit failures, redelivery, inbox duplicates, and consumer lag.

**Follow-up:** Explain why database-first plus offset-second chooses duplicate delivery over message loss, and how idempotency makes that safer.

### Scenario 43 — Consumer crashes after processing but before acknowledgement

**Problem:** The consumer commits its database update successfully, then crashes before acknowledging the broker message. When it restarts or rebalances, the record is delivered again. This is normal at-least-once behavior, not necessarily a broker bug.

**Local database effect:** Use a stable event ID and apply both the inbox/deduplication record and the business state transition in one database transaction. The uniqueness constraint prevents a redelivered event from applying the same logical update again. If different event IDs can represent the same business action, also enforce domain-level uniqueness—for example, one inventory reservation per order line.

**External side effects:** A database inbox alone cannot prove whether an unrelated provider sent an email, charged a card, or accepted a request. If the crash happens after the provider accepts the request but before local completion is recorded, the consumer must use provider idempotency or query/reconcile by a stable operation ID. If the provider offers neither, the design should document the residual duplicate/uncertain-outcome window and provide manual or financial reconciliation where needed.

**Acknowledgement rule:** Acknowledge only after the durable processing boundary is committed. Acknowledging first risks loss; acknowledging later means duplicates are possible. Avoid holding an unbounded broker transaction while waiting for remote work. For batched records, be precise about whether partial processing succeeded and which offsets are safe to commit.

**Signals and tests:** Track redelivery and duplicate-detection rates, processing latency, offset-commit failures, and unresolved external outcomes. Inject crashes before the database commit, after commit, and after provider response to validate each case.

**Follow-up:** Distinguish exactly-once broker processing from exactly-once business effect. The latter usually requires atomic local state changes plus idempotency/reconciliation across external boundaries.

### Scenario 44 — Cache invalidation events arrive out of order

**Problem:** Product version 12 is cached, then a delayed version 10 event arrives and overwrites it with stale data—or an older invalidation clears a newer value and creates a refill storm.

**Clarify:** Are events ordered by entity key? Do cache values carry a source version? Is the cache populated on read or updated from the event stream? Can events arrive from several partitions or through both CDC and an application event path?

**Design:** Prefer version-aware cache writes: only accept an update if its source version is newer than the version currently stored. If safe conditional writes are unavailable, use events to invalidate and refill from the authoritative source, while guarding against a stale reader racing with a fresh write. A timestamp alone may be unreliable if clocks skew; use a monotonic entity version or source log position where available.

**Ordering caveat:** Broker ordering is usually scoped to a partition and helps only if related events use a consistent key and processing is sequential. It does not solve reordered deliveries across independent pipelines or a backfill racing with live updates. For deletes, a versioned tombstone prevents older events from resurrecting the record.

**Recovery and signals:** Track event age, per-key version regressions, invalidation lag, cache/source mismatches, duplicate events, and refill concurrency. Rebuild from a known snapshot plus subsequent changes when the cache/projection becomes suspect. Test delayed, duplicated, and reordered events explicitly.

### Scenario 45 — A distributed botnet bypasses IP rate limits

**Problem:** Attack traffic comes from thousands of IP addresses, bypassing a per-IP limit. Meanwhile, overly aggressive IP blocking risks denying service to legitimate customers behind corporate networks or mobile carrier NAT.

**Layered defense:** Use coarse edge/network limits for obvious spikes, but also rate-limit by authenticated account, API key, tenant, operation, and estimated resource cost. Expensive search, password-reset, signup, export, and payment-initiation endpoints need different protections from cheap cacheable reads. Apply concurrency limits and request-size/time budgets so attackers cannot consume unlimited work with a low request count.

**Identity and behavior:** Account-based controls can be evaded by account farms; IP limits can be bypassed by distributed sources; behavioral detection can produce false positives. Combine signals such as request velocity, account age, endpoint sequence, reputation, device/session signals where lawful, and repeated failed operations. Use progressive throttling or challenges where appropriate, and provide a safe recovery path for real users.

**Protect downstream capacity:** Reject suspicious traffic before it reaches expensive database or provider operations. Use per-tenant budgets, quotas for external calls, and a kill switch for especially costly operations. A shared limiter can become a target itself; design its capacity and fallback behavior, and avoid unbounded key cardinality from attacker-controlled IDs.

**Operations and privacy:** Ensure rate-limit policy changes are authenticated and auditable. Avoid storing more personal/device data than necessary, and control access to abuse telemetry. Coordinate emergency blocking rules with false-positive monitoring.

**Signals and follow-up:** Track attack and allowed traffic by endpoint/identity, throttling/challenge outcomes, downstream cost, false-positive reports, and limiter latency. Explain why a distributed attack requires several identity and cost dimensions rather than one global rate threshold.

### Scenario 46 — Backups exist but restore is untested

**Problem:** Backup jobs report success, but the team has never restored a production-sized dataset or verified that the application can operate from the restored data. The backup may be incomplete, encrypted with an unavailable key, incompatible with the current schema, or too slow to restore within the RTO.

**Restore exercise:** On a recurring schedule, restore into an isolated environment and verify integrity, row/partition counts, key business invariants, schema compatibility, encryption keys, secrets, access policies, deployment artifacts, and required configuration. Test recovery of logs/WAL/binlogs if point-in-time recovery is part of the promise. Include dependent systems such as identity/KMS, DNS, network policies, and object storage.

**RPO/RTO:** Record the restored data point and total time to provide an acceptable user journey. A backup might meet the RPO but miss RTO because loading several terabytes, replaying logs, rebuilding indexes, warming caches, or reconciling external workflows takes hours. Measure detection, authorization, restore, validation, routing, and recovery—not only database startup.

**Safety:** Define who approves a restore, how to keep the recovery environment isolated, and how to avoid overwriting healthy production with an old copy. Replication may have copied corruption or accidental deletion, so preserve independent recovery points and consider immutable/offline retention according to threat model. Understand how a restore handles data that should remain deleted under current privacy requirements.

**Signals:** Track last known-good restore test, backup age, backup integrity, restore duration, recovery point age, and failed validations. Run game days for accidental deletion, regional loss, and corrupted writes.

**Follow-up:** Explain which incidents replication handles and which require backup restoration. A green backup job is evidence that bytes were written, not proof that the service is recoverable.

### Scenario 47 — Multiple layers retry the same request

**Problem:** The SDK, gateway, application service, service mesh, and background worker all retry the same dependency. During a slowdown, one logical request creates many physical attempts, overwhelming the dependency and delaying recovery.

**Draw the attempt tree:** Write down the maximum attempts at each layer, whether each layer retries the entire downstream call or only its local attempt, and whether broker redelivery or client retries can occur outside that tree. If four layers each allow three attempts and every attempt reaches the deepest dependency, the theoretical upper bound is (3^4=81) attempts for one logical call. Actual behavior depends on short-circuiting, but the multiplication risk is real.

**Diagnose:** Measure original request IDs versus attempt IDs, retry reason, timeouts, queue redelivery, downstream active concurrency, and successful throughput. Determine whether the first attempt may have committed a side effect; a timeout is uncertainty, not proof of failure.

**Design:** Choose a clear retry owner where possible. Propagate a single end-to-end deadline and ensure inner timeouts fit inside it. Use capped exponential backoff with jitter, a retry budget, concurrency limits, circuit breaking, and load shedding. Retry only plausible transient errors and only repeat mutations through idempotency/operation identity. Apply a separate durable retry policy to asynchronous jobs rather than making a synchronous request wait indefinitely.

**Recovery:** Suppress noncritical retries during overload, recover the dependency, then restore traffic gradually. Ensure DLQ redrive cannot compete with live traffic without bounds.

**Follow-up:** A retry policy should improve successful user outcomes without destroying system-level availability. Track attempts per logical operation, retry-generated QPS, timeout rate, and time-to-recover.

### Scenario 48 — New service must extract data from a monolith safely

**Problem:** A team extracts a domain from a monolith, but both old and new paths still read/write the same tables. The migration risks conflicting updates, data loss, and an apparent service boundary that preserves the old coupling.

**Define ownership:** Choose one authoritative owner for each business fact and enumerate all writers: APIs, scheduled jobs, reports, admin tools, batch imports, and other services. A new deployment is not independent if every writer still modifies its tables directly. Identify the commands and invariants that must remain atomic.

**Migration plan:** Start with a cohesive domain and stable contract. Add a façade/adapter, backfill the new store from a consistent snapshot, capture ongoing changes with CDC or another controlled feed, and validate versions, counts, and business outcomes. Avoid naive dual writes because one side can commit while the other fails. If dual-write is temporarily unavoidable, define authority, retries, repair, and a cutover/rollback boundary explicitly.

**Incremental cutover:** Shadow reads can compare old/new results without exposing the new output. Route a small cohort or operation type to the new owner, compare correctness and p99, then expand. For writes, ensure only one system is authoritative for the data at any time. Once the new service accepts writes the monolith cannot understand, rollback may require reverse replication or forward repair rather than simply switching routing back.

**Operational details:** Update permissions so old code can no longer write after cutover, migrate consumers and scheduled jobs, and retain an audit/repair path. Monitor migration lag, source/destination mismatches, duplicate effects, error rate, latency, and hidden legacy dependencies.

**Follow-up:** Explain how the design prevents two owners, how concurrent changes are captured, and what rollback means after new writes occur.

### Scenario 49 — Tenant requires data residency

**Problem:** An enterprise tenant requires regulated data to remain within a specified jurisdiction, including disaster-recovery copies and potentially operational telemetry.

**Clarify scope:** Identify protected data classes and what the contract/regulation covers: primary records, replicas, backups, object storage, search indexes, caches, analytics, logs/traces, exports, support access, encryption keys, and control-plane metadata. Data residency is not automatically equivalent to data sovereignty or a complete access-control model; legal interpretation should be confirmed with the responsible compliance team.

**Architecture:** Route tenant traffic to an approved region and enforce placement through deployment, storage, and data-pipeline policies. Use tenant-aware data ownership and region tags in metadata; prevent cross-region replication for restricted classes unless explicitly permitted. Ensure derived systems inherit the same restrictions, not only the primary database. Keep backups and snapshots in approved locations and specify where encryption keys are held/used.

**Operational access:** Support engineers, global operators, and telemetry systems can accidentally move or expose data outside the region. Use least-privilege, audited, time-bounded access and redact/minimize cross-region logs. Verify exports, data-science pipelines, search rebuilds, and support diagnostics.

**Failure trade-off:** Residency can constrain disaster-recovery placement and failover options. If the approved region is unavailable, the system may need to remain unavailable rather than fail over to an unauthorized region. Define RTO/RPO and tested recovery plan consistent with that requirement.

**Validation:** Automate placement checks, data-flow inventory, access audits, backup location tests, and regional-outage scenarios. Track exceptions with owners/expiry. In an interview, explain the full lifecycle of the data and the constrained failover behavior; “database is local” is not sufficient.

### Scenario 50 — Availability conflicts with correctness during a partition

**Problem:** A network partition prevents one region from confirming the latest state, but users continue submitting writes. The service must decide whether to stay responsive or refuse operations whose correctness cannot be guaranteed.

**Reason from the invariant:** Ask what damage a conflicting write could cause and whether the business explicitly tolerates it. A payment ledger, exclusive seat reservation, or globally unique username cannot necessarily merge two independently accepted conflicting writes. A view counter or some independent metadata may tolerate local updates followed by reconciliation.

**Design:** For correctness-critical writes, keep a single authoritative owner per entity/partition or use a datastore with documented global transaction/consistency semantics. If the owner or latest state cannot be established, reject the write or durably record a pending command only if delayed execution is still valid and the API contract says it is pending. Never claim the mutation committed merely because a queue accepted a command.

**Degraded mode:** Preserve safe reads, local cached content, or unrelated operations where their freshness contract permits it. Separate “read service is healthy” from “write ownership is safe.” If the system uses active-active writes, define conflict resolution for each field and prove the business invariant is mergeable; last-write-wins is not safe for all ledgers or inventory.

**Recovery:** Re-establish one owner, fence the old writer, reconcile accepted commands and external effects, and verify invariants before resuming normal writes. Preserve operation IDs and audit evidence so clients can query uncertain outcomes.

**Trade-off:** Strong coordination can reduce write availability during partitions. State the chosen policy per operation, rather than saying simply “CAP means choose two.”

**Follow-up:** Explain which operations are rejected, which can proceed safely, and how the system identifies/repairs divergence after connectivity returns.

### Scenario 51 — A dashboard endpoint fans out to many services

**Problem:** The endpoint calls account, billing, usage, permissions, and recommendations services. Its p99 latency is poor even though most dependencies look healthy.

**Clarify:** Which fields are required to render the page? Which data has a freshness requirement? What is the end-to-end latency objective, and do all calls execute serially?

**Analysis and recommendation:** Draw the dependency graph and use traces to measure each span, including queue and connection wait. Parallelize independent calls under a bounded concurrency limit, but do not assume parallelism removes the slowest dependency's tail. Return a partial response only if the API contract explicitly permits it. For expensive, frequently requested combinations, consider a materialized read model with a freshness SLO. Keep authorization and other correctness-critical checks authoritative.

**Trade-offs and signals:** A read model improves predictable latency but adds projection lag and operational work. Monitor per-dependency p95/p99, fan-out count, timeout rate, queue wait, and read-model age.

**Follow-up:** If one field is optional, isolate its timeout and render the rest rather than allowing it to consume the entire request deadline.

### Scenario 52 — A tenant exceeds its contracted API quota

**Problem:** A tenant exceeds its contractual request allowance and threatens shared capacity. The API needs to enforce the quota without letting one tenant starve others or incorrectly penalizing legitimate burst behavior.

**Clarify the quota:** Is the limit a hard billing/contract ceiling or a soft fairness target? Is it per second, minute, concurrent request, byte volume, or expensive operation? What burst allowance exists? Do all endpoints cost the same, or should exports/searches consume more of the budget than cheap reads?

**Design:** Scope enforcement to the authenticated tenant and relevant endpoint/cost class. A distributed/shared limiter can coordinate a global budget; hierarchical token allocation can let instances make local decisions with bounded overshoot. Account for autoscaling and regional routing so adding instances does not multiply the permitted allowance. Combine rate limits with concurrency/admission limits because slow expensive requests can exhaust resources even below the rate limit.

Use fair scheduling and per-tenant queues for deferrable work. Reserve capacity for critical recovery/security operations if the contract allows it. Return clear 429/quota responses with retry semantics; do not enqueue unlimited work simply because the API rejected synchronous execution.

**Correctness and security:** Derive tenant identity from trusted authentication/authorization context, not a caller-controlled query parameter. Ensure tenant limits cannot be bypassed through alternate routes, background jobs, or service-to-service APIs.

**Signals and trade-offs:** Track allowed/rejected traffic, actual cost per tenant, throttling, burst use, limiter errors, queue age, and per-tenant p99. A strict billing limit may need fail-closed behavior; approximate abuse limits may permit a bounded fallback. Explain how you honor contractual limits while preserving platform fairness.

### Scenario 53 — An outbox publisher is stuck but API requests succeed

**Problem:** Business records commit and the API returns success, but downstream consumers stop receiving new events.

**Analysis:** The transactional outbox ensures that event intent commits with the business change; it does not guarantee that the publisher keeps making progress. Inspect oldest unpublished row, publish throughput/errors, broker availability, CDC connector state, database load, schema/serialization errors, and whether cleanup accidentally removes unpublished records.

**Containment:** If downstream projections or workflows depend on these events, identify which user journeys are now stale or stuck. The API may continue accepting business writes only while the outbox has sufficient durable capacity and the business contract tolerates downstream delay. Alert or throttle admission if backlog threatens database growth or the maximum completion deadline.

**Recovery:** Retry transient failures with bounded backoff; isolate poison records so one bad payload does not block all progress; maintain idempotent publication and consumer processing because a crash after publish but before marking sent may duplicate an event. Do not delete or skip an event merely to make the queue look healthy. Replay after repair at a controlled rate and reconcile downstream projections.

**Operations:** Alert on outbox age and backlog, not just API success. Track events produced versus business changes committed, publisher errors, consumer lag, and end-to-end freshness. The outbox table can itself become a bottleneck: batch reads, appropriate indexes, partition/retention strategy, and safe cleanup may be required at scale.

**Follow-up:** Explain why the database transaction remains correct while the event pipeline is unhealthy and what condition would cause you to stop accepting more work.

### Scenario 54 — A provider callback arrives before the synchronous response

**Problem:** A payment provider processes a request quickly and sends a webhook before the application receives or persists the synchronous HTTP response.

**Why this happens:** Network paths are independent and delivery order is not guaranteed. The callback may arrive before the local request handler updates the payment row, or the synchronous request may time out even though the provider completed the operation. Do not assume wall-clock arrival order is the business order.

**Design:** Create a durable payment attempt with a stable internal operation ID and idempotency identity before calling the provider. Include a correlation reference the callback can resolve. The callback handler should authenticate the signature, deduplicate provider event IDs, validate the state transition, and apply updates transactionally. If the local state is not yet ready to accept the callback, durably store it as unmatched/pending and retry correlation rather than dropping it.

**State handling:** Allow only legal transitions; for example, a late “processing” callback must not move a terminal CAPTURED payment back to PENDING. Use provider versions/timestamps only according to documented semantics, and query the provider if current status is unclear. The synchronous response handler and webhook handler must converge on the same operation state idempotently.

**Recovery:** Reconcile unmatched callbacks and unresolved operations using provider status/settlement reports. Track callback-before-response frequency, unmatched-event age, duplicate callback count, invalid signatures, and state-transition conflicts.

**Interview point:** Explain how both handlers race safely, how one payment operation is correlated, and why the callback may be the first reliable proof of completion even if the synchronous call has not returned.

### Scenario 55 — A database failover causes a burst of connection errors

**Problem:** A database promotes a new primary, but application requests continue using stale pooled connections or reconnect simultaneously. The database may be healthy while the application fleet experiences a connection storm.

**Diagnose:** Verify promotion and write authority first: is the new primary writable, and is the old primary fenced? Then inspect error types, stale DNS/service-discovery records, connection-pool lifetime, pool acquisition wait, active sessions, transaction outcomes, and per-instance reconnect rate. Distinguish connection failures from queries that executed but whose response was lost.

**Design:** Ensure the database driver/proxy can discover the current authority, discard broken connections, and establish new ones with bounded connect/acquisition timeouts. Use exponential backoff with jitter so all instances do not reconnect in lockstep. Keep fleet-wide pool caps within database capacity, including rolling deployment surge, background workers, and administrative sessions. Limit request concurrency while the database is recovering.

**Uncertain mutations:** A failed socket or timeout does not prove the transaction rolled back. For writes, use idempotency/business operation IDs and query status or reconcile before repeating. Do not retry every write with a new identity. If promotion may have lost acknowledged writes, validate the RPO and reconcile affected transactions.

**Recovery:** Restore traffic gradually after verifying the primary, not all at once. Warm pools under controlled concurrency and watch query throughput, replication state, lock waits, and business-level errors.

**Signals and tests:** Track connection churn, pool wait, authentication/connection errors, write timeout/unknown outcomes, successful commit rate, and time to recover. Test failover under peak traffic and a simultaneous rollout; “managed database failover” does not remove client-side reconnection design.

### Scenario 56 — A data backfill competes with live production traffic

**Problem:** A large backfill competes with live reads/writes, increases p99 latency, creates replication lag, and may overwrite data updated by production while the backfill is running.

**Diagnose:** Measure backfill throughput, batch size, transaction duration, lock waits, database CPU/IOPS, replication lag, production p99, WAL/binlog volume, and query-plan changes. Determine whether the job holds a long snapshot, performs large writes, or shares a connection pool with interactive traffic.

**Design:** Use bounded, checkpointed, idempotent batches with configurable throttling and pause/resume controls. Give the job a separate, limited concurrency budget. Process keys in predictable ranges to avoid repeated full scans, but ensure range traversal handles concurrent inserts and deletes. Use a consistent snapshot/high-water mark and capture later changes with CDC or another reliable log. Version-aware conditional writes prevent older snapshot values from overwriting newer live updates.

**Isolation:** Run from a replica or snapshot only if the source is consistent enough and replication lag/freshness remains acceptable. Do not offload to a read replica blindly if the backfill produces writes to the same target or if long reads slow replication. Schedule windows help but are not a substitute for throttling when traffic spikes unexpectedly.

**Validation and rollback:** Check counts, checksums, nullability, version mismatches, and business invariants. Reconcile deletes and failed batches. Define a safe stop point and whether partial destination data can be ignored or must be cleaned up. Do not switch reads until the backfill and concurrent-change stream have converged.

**Signals:** Monitor progress/lag, mismatch counts, production p99, lock duration, replication delay, and backfill retries. A successful job exit code is not proof that the migrated data is correct.

### Scenario 57 — A retry storm begins after a brief dependency outage

**Problem:** The dependency has recovered, but request volume remains elevated and latency does not normalize.

**Diagnose the feedback loop:** Inspect attempts per logical request, retry policies at clients/gateways/services/meshes, synchronized schedules, current dependency saturation, queue age, and timeout rates. A request may time out while the original attempt is still running, so retries can increase concurrent work even after the initiating fault is gone. Multiple retry layers can multiply traffic.

**Immediate response:** Protect the dependency with concurrency limits, temporary load shedding, and a retry budget. Reduce or pause retries that are not necessary for critical work, and let noncritical asynchronous work wait or expire under a documented policy. Do not abruptly replay every deferred request at full rate when the dependency first appears healthy.

**Long-term design:** Use capped exponential backoff with jitter, end-to-end deadlines, bounded attempts, error classification, and idempotency for mutations. Give retry ownership to a clear layer where possible. Make circuit-breaker thresholds and half-open probes conservative enough not to flood recovery. For client behavior outside your control, enforce server-side admission control and rate limits.

**Signals:** Track original requests separately from attempts, attempts per operation, retry success rate, downstream p99, active concurrency, queue age, and successful throughput. Timeouts bound resource holding; retries can mask transient errors; backoff/jitter spread load but do not make unsafe operations idempotent.

**Follow-up:** Explain the first control you would activate during the incident and how you know the system can return to normal without a second spike. See [AWS guidance on timeouts, retries, and jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).

### Scenario 58 — A feature flag service becomes unavailable

**Problem:** A feature-flag service becomes unavailable during rollout. Applications cannot fetch new configuration, and different instances may have different flag values. A generic “default false” is unsafe if some flags control security, data migration, or emergency shutdown.

**Classify flags:** Cosmetic/optional flags, gradual rollout flags, business decision flags, migration switches, and kill switches have different failure semantics. Decide which can safely continue using last-known-good configuration and which require fail-closed behavior. A flag that gates access or a money movement should not share the same fallback logic as a recommendation widget.

**Design:** Avoid a remote flag lookup on every request. Fetch/version configuration, validate it, and cache it locally with an explicit freshness limit and source version. Roll out configuration changes progressively and expose current version/age. Define fallback per flag and test the behavior when configuration is missing, malformed, stale, or partially propagated. For emergency kill switches, establish a tested control path and a safe degraded mode if the control plane is unreachable.

**Consistency and deployment:** Flag changes across instances are not instantaneous; decide whether a feature can operate with mixed versions. Do not use a flag to hide an incompatible schema migration or to bypass a required authorization check. Maintain clear ownership, audit all changes, expire unused flags, and prevent unauthorized operators from changing production policy.

**Recovery:** When the service returns, reconcile local versions and roll out changes safely rather than causing a synchronized burst of configuration fetches. Track fetch failures, config age, propagation lag, flag evaluation errors, and behavior by version.

**Follow-up:** Explain which flags use last-known-good values, which fail closed, and how the system detects that an emergency control did not reach every instance.

### Scenario 59 — A CDN serves private content to the wrong user

**Problem:** A CDN returns one user's private response to another user because the cache key or cache-control policy did not include an identity/tenant dimension, or because authorization was assumed to happen at the origin even on a cache hit.

**Immediate containment:** Disable shared caching for the affected response class or bypass cache for protected routes, then purge affected entries according to the incident plan. Preserve evidence without copying sensitive payloads into logs or incident notes. Assess the exposure scope and follow the security/privacy incident process; an origin authorization check may never run on a shared cache hit.

**Corrective design:** Treat CDN cache key and cache-control behavior as security controls. Personalized or sensitive responses should be private/non-cacheable unless an explicit safe caching design exists. For shared responses, ensure every representation-changing dimension is included and untrusted request headers cannot poison or fragment the cache unexpectedly. Validate the interaction of cookies, Authorization, Vary, query parameters, and CDN-specific normalization rules.

Do not solve the issue solely by adding a user ID to every cache key without checking whether the CDN actually uses it in its cache policy. Signed URLs/tokens must be scoped to a particular object, operation, and expiry; they should not grant broad bucket access. Avoid caching authorization decisions longer than the revocation contract permits.

**Validation:** Add cross-user and cross-tenant tests against the real CDN configuration, including different roles, locales, query parameters, cookies, and revoked permissions. Review origin logs, cache hit/miss logs, purge propagation, and cache-control headers. Ensure response bodies and tokens are not logged. Track cache policy changes with code review and staged rollout.

**Follow-up:** Explain why “the backend is secure” does not protect against a cache serving a stored response without reaching the backend.

### Scenario 60 — A design review challenges a new microservice

**Problem:** A team proposes extracting a service mainly to “make it scalable,” but the proposed service adds several network calls and continues sharing the same database tables.

**Clarify the motivation:** Which measured bottleneck or ownership problem does the extraction solve? Does this capability require an independent release cadence, security boundary, scaling profile, or failure domain? Is the domain boundary stable, and can the service own its data and contract?

**Evaluate the trade-off:** Estimate the extra network latency, timeout/retry complexity, observability and on-call burden, service discovery/configuration, deployment coordination, and cross-service transaction cost. If both services directly write the same tables, the split may not reduce coupling; it can add network failure while preserving shared ownership.

**Recommendation:** If the boundary is not cohesive or independently valuable, keep a module in the monolith and strengthen internal interfaces/ownership. If extraction is justified, define the API/event contract, authoritative data owner, migration/CDC or outbox strategy, version compatibility, dashboards, runbook, rollout plan, and rollback/forward-repair boundary. Extract one capability at a time and compare old/new behavior where safe.

**Evidence and follow-up:** Use release lead time, incident blast radius, capacity isolation, and team ownership as measurable outcomes—not just count of services. A service split that reduces one bottleneck but introduces a cross-service saga may be justified, but the new workflow and compensation need explicit design.

**Interview point:** “Microservices scale better” is not a sufficient rationale. Explain what coupling the boundary removes, what new coupling it introduces, and how you will prove the split improved the system.

### Scenario 61 — A search-index rebuild must not lose live updates

**Problem:** The search index is being rebuilt to change analyzers, mappings, or storage layout while product writes continue. If the new index is populated from a moving source without capturing live changes, updates can be lost or overwritten.

**Establish a snapshot boundary:** Build a new index generation rather than deleting the live index. Capture a consistent snapshot and the corresponding database log/CDC position. Bulk-load the snapshot, then replay all later changes in order for each entity. If the change log expires before catch-up, restart from a new consistent boundary rather than cutting over with a gap.

**Prevent stale overwrites:** Include entity versions or source sequence numbers. A document loaded from the snapshot must not overwrite a newer update already applied from the change stream. Preserve delete tombstones long enough so a replayed older create/update cannot resurrect a deleted document. Make retries and upserts idempotent.

**Validate before cutover:** Compare document counts, versions, required fields, domain aggregates, deletion behavior, and representative queries/relevance—not only that the index returns HTTP 200. Shadow queries can compare new/old results without exposing the candidate to users. Track indexing lag, error rate, and query p99.

**Cutover and rollback:** Switch an alias/routing pointer only after the new index reaches the required freshness boundary. Keep the old generation for a bounded rollback period, but define how writes occurring after cutover are reflected if rollback is needed. Account for the temporary storage/compute cost of two indexes.

**Follow-up:** Explain the snapshot/CDC boundary, how deletes and late events are handled, and why “bulk load completed” is not proof of a correct rebuild.

### Scenario 62 — A deleted account reappears in a downstream system

**Problem:** A deleted account reappears in search, analytics, cache, or another downstream service because a delayed create/update event was applied after the delete. Deletion has propagated to some stores but not all.

**Root cause:** Removing a row from the source does not automatically erase derived copies, and a consumer that treats every event as an unconditional upsert can resurrect the entity. An old event may also be replayed during a rebuild after the tombstone was discarded.

**Design:** Publish a durable deletion/tombstone event with the entity identity and a version or lifecycle epoch. Consumers should apply an update only if its version is newer than the last applied version, and retain deletion metadata long enough to reject delayed updates and replayed history. Use explicit deletion semantics for search indexes, caches, analytics, object storage, exports, and downstream processors. A boolean soft-delete alone is not enough if the event pipeline later treats old records as active.

**Privacy and retention:** Determine whether the requirement is logical hiding, physical erasure, or deletion from all derived copies within a stated time. Backups may retain data until their retention expires, so restore procedures must also reapply current deletion records or tombstones before exposing restored data. Preserve only audit evidence permitted by policy.

**Recovery:** Reconcile every required downstream location against the authoritative deletion state. Repair with version-aware operations and verify the data does not reappear after a consumer replay or index rebuild.

**Signals and tests:** Monitor deletion propagation latency, resurrection attempts, failed downstream deletes, tombstone expiry, and per-store mismatches. Test out-of-order events, full rebuild, replay, and restore.

**Follow-up:** Explain the tombstone retention window and how it exceeds the maximum credible event/replay delay.

### Scenario 63 — Global ID generation fails during a network partition

**Problem:** A central ID allocator is unreachable during a network partition, while services in multiple regions need to create records. A naive failover can allocate the same sequence range or worker identity twice.

**Clarify requirements:** Is the system requiring uniqueness only, global monotonic order, approximate time ordering, compact IDs, or the ability to generate IDs while disconnected? These goals conflict. A globally monotonic sequence requires coordination; a decentralized scheme generally provides uniqueness with weaker ordering properties.

**Design choices:** Preallocate disjoint sequence ranges, use region/worker bits with durable exclusive worker assignment, or generate sufficiently large random identifiers. A central sequence is simple but adds latency and becomes an availability dependency. Range allocation reduces per-ID coordination but must ensure a failover node cannot reuse an active range. Time/worker/sequence schemes need rules for clock rollback, worker-ID reuse, sequence exhaustion, and startup after a pause.

**Prevent split-brain allocation:** Do not promote a standby based only on a heartbeat timeout if the old allocator could still issue IDs. Use epoch/fencing or exclusive lease ownership that the allocation resource enforces. Persist range/worker assignments and quarantine uncertain assignments after failover. A local check alone cannot prevent an isolated old process from continuing.

**Validation and repair:** Test process restart, node replacement, region loss, clock skew/rollback, and concurrent allocator promotion. Enforce a unique constraint at the destination where feasible; if collisions occur, preserve referential integrity and define a repair path rather than silently overwriting records.

**Signals:** Monitor duplicate-key violations, worker-ID reuse, range allocation history, clock drift, and generator failover. In an interview, state clearly whether ID uniqueness is guaranteed by protocol or only probabilistically.

### Scenario 64 — A data backfill silently overwrites current values

**Problem:** A backfill reads an old snapshot, then writes it after a newer live update, silently reverting current state. The job completes successfully and row counts match, but the resulting data is incorrect.

**Root cause:** The backfill is a concurrent writer. If its writes are unconditional, the order in which operations reach the destination—not the age of the data—determines the final value.

**Design:** Define a snapshot/high-water mark or source version, and apply records with conditional writes so an older version cannot overwrite a newer live value. For a large migration, copy a consistent baseline and capture all later changes through CDC/log replay. If dual writes are used, define which side is authoritative and how partial success is repaired; naive writes to both stores can diverge.

**Deletes and ordering:** Capture deletes/tombstones as well as updates, preserve per-entity ordering, and ensure replayed events cannot resurrect a deleted record. A wall-clock timestamp can be unsafe if hosts have skew; database-generated versions or source log positions are preferable when available.

**Operational control:** Run bounded, checkpointed, idempotent batches and throttle them against production p99, database I/O, lock waits, and replication lag. Pause automatically when mismatches or lag cross a threshold. Keep progress and failure records so a restarted worker does not repeat unsafe work.

**Validation:** Compare versions and domain-level values, not only row counts; use checksums where they are meaningful. Do not switch reads until concurrent updates have converged and discrepancy rules are understood. Keep rollback or forward repair for partially migrated data.

**Follow-up:** Explain how the migration identifies and rejects a stale write, including the crash window when the worker restarts.

### Scenario 65 — A provider returns throttling errors after scaling workers

**Problem:** Increasing worker instances reduces successful throughput because a third-party provider enforces a fixed account-wide rate limit.

**Diagnose:** Compare the provider's quota scope with the limiter's scope. If every instance independently permits the full account limit, aggregate outbound rate rises with instance count and triggers throttling. Retries may consume the same quota and reduce useful throughput further. Verify whether the provider also limits concurrency, per-endpoint calls, or burst size.

**Design:** Coordinate a shared account-level budget or allocate bounded token budgets from a global allowance. Queue deferrable work, separate urgent and bulk jobs, coalesce duplicate calls, and honor Retry-After. Count retries against the same budget. Scale workers only until they can sustain the permitted provider rate; extra workers may prepare requests but cannot raise the external limit.

**Recovery:** If throttling created a backlog, estimate net drain rate and ramp replay gradually. Preserve capacity for live critical work; do not reset quota accounting during regional failover or autoscaling. If a second provider is proposed, verify semantic compatibility, idempotency behavior, capacity, and reconciliation before failing over critical mutations.

**Signals:** Track provider 429s, attempts per logical operation, quota utilization, queue age, successful completion rate, and expected drain time. Use a single place to observe fleet-wide outbound attempts.

**Interview point:** More consumers only help when consumer capacity is the bottleneck. If the external quota is fixed, the correct response is quota-aware scheduling and priority, not unbounded scale-out.

### Scenario 66 — An emergency kill switch is needed during a rollout

**Problem:** A newly released feature produces incorrect business results, but the feature-flag control plane is partially unavailable and some instances cannot fetch the latest flag value. Turning a flag off everywhere is not instantaneous, and already completed effects may remain.

**Define safe behavior first:** Classify the feature. An optional recommender can often be disabled safely; a feature involved in a database migration, payment state, or authorization may require a specific compatibility path. Do not assume “default false” is safe if old and new code interpret persisted data differently.

**Control path:** Keep a validated, versioned last-known-good configuration local and define a maximum freshness age. Emergency controls need a tested propagation objective, authenticated and audited changes, and a fallback if the control plane cannot be reached. For high-risk behavior, use a conservative safe default or fail-closed policy where feasible. Avoid a design where every request synchronously calls the flag service.

**In-flight operations:** Disabling a feature stops future work; it does not undo side effects already performed. Identify workflows that started under the new behavior, decide whether to finish, cancel, compensate, or reconcile them, and ensure the disabled path can still read records created by the enabled path.

**Rollout and recovery:** Stage the change, verify configuration version/age across instances, pause further release, and monitor business outcomes after the switch. Re-enable only after the root cause is addressed and tests cover the failure.

**Signals and follow-up:** Track flag propagation delay, stale instances, active workflow state, and errors by version. Explain how the kill switch works during control-plane failure and how you prevent already-started work from becoming orphaned.

### Scenario 67 — A backup restore meets RPO but misses RTO

**Problem:** A restore recovers data from within the allowed loss window, satisfying RPO, but the full service takes longer than the business can tolerate to return, violating RTO.

**Separate the objectives:** RPO is the age of the recovery point/data loss the business tolerates. RTO is the time to restore an acceptable user journey. A database restore that finishes in two hours can meet a six-hour RPO but still miss a one-hour RTO. The entire path matters, not just database startup.

**Measure the recovery timeline:** Record time for detection, decision/approval, snapshot retrieval, data restore, log replay, schema migrations, index rebuild, key/secret retrieval, application deployment, DNS/routing, cache/read-model warm-up, and business validation. Find the dominant stage; optimizing backup frequency may improve RPO without improving RTO.

**Options:** Increase restore bandwidth and parallelism, use incremental/PITR techniques, retain prebuilt artifacts/indexes where safe, automate deployment/configuration recovery, pre-provision a warm standby, or keep a replica for faster node failover. A replica is not a substitute for backups after logical corruption because corruption can replicate. A warm standby costs more and must be tested for actual capacity and data freshness.

**Validate end to end:** Run periodic restores at production-like volume, verify business invariants and application dependencies, and record measured RPO/RTO rather than target-only values. Include KMS/identity/DNS/operator access outages and the possibility that the restored data requires reconciliation with external providers.

**Follow-up:** Explain which step dominates RTO, how your chosen mitigation changes the full recovery timeline, and how you prove the target has been met.

### Scenario 68 — A fallback returns stale inventory during an outage

**Problem:** The authoritative inventory service is unavailable, and a fallback cache displays stock that may already have been sold. The system wants to remain available but must not confirm a reservation based on stale inventory.

**Define the invariant:** If overselling is prohibited, only an authoritative atomic reservation/hold can confirm stock. A cached count may be acceptable for browse/display, but it cannot prove the unit is available at commit time. If the business intentionally allows oversell, that must be a stated product policy with bounded limits and remediation—not an accidental consequence of fallback.

**Degraded design:** Keep safe catalogue reads available, but disable or defer the final reservation decision while authoritative state is unreachable. Queue a command only if delayed execution remains valid and the API explicitly reports pending rather than confirmed; a seat or flash-sale item may be gone before the queue drains. For a pending operation, persist the operation ID and idempotency key durably, then revalidate stock before any commit when service returns.

**Avoid misleading status:** Do not return “reserved” or “order confirmed” unless the invariant is durably enforced. If payment has already been captured but reservation cannot be confirmed, define a refund/compensation workflow and reconciliation. Avoid charging before inventory commitment unless the business workflow can handle the failure safely.

**Recovery:** Reconcile holds, reservations, payment attempts, and inventory before resuming normal writes. Use conditional transitions and idempotency to avoid duplicates. Monitor stale-stock age, pending reservations, rejected attempts, oversell/compensation counts, and inventory service recovery.

**Follow-up:** Explain what can remain available during the outage, what must be rejected, and how the API represents uncertain or pending outcomes.

### Scenario 69 — A low-volume service violates its SLO but dashboards show no obvious issue

**Problem:** A low-volume service violates its SLO, but dashboards look normal because aggregates and percentiles are based on a small number of requests or a long observation window.

**Diagnose carefully:** Verify the SLI denominator, window, exclusions, and whether the service measures business success or only HTTP response codes. With few requests, one failure can move a percentage sharply; conversely, an hour-long outage between synthetic checks may be hidden by a 30-day aggregate. Inspect individual traces and structured events, dependency errors, state transitions, and the affected user journey.

**Design indicators at the right level:** Use request-based SLOs where eligible request counts are meaningful, plus synthetic checks for critical workflows and business-level indicators for low-volume high-impact operations. A single failed payment or broken administrative workflow may merit immediate action even when statistical significance is low. Set alert policies that combine burn rate with minimum signal and clear severity rather than suppressing all low-sample alarms.

**Avoid false confidence:** Health probes can be green while a required provider, credential, or data path is broken. Test real actions safely—for example, synthetic read/auth flows that do not create irreversible effects. Preserve per-operation traces and correlation IDs for manual diagnosis.

**Signals:** Track successful user journeys, per-operation error category, latency, dependency outcomes, synthetic success, and remaining error budget. Compare deployment/change markers and region/tenant context. Do not use high-cardinality identifiers as metric labels; keep detailed records in logs/traces.

**Follow-up:** Explain what alert is actionable despite low traffic, and how you distinguish a genuine issue from percentile noise without ignoring a critical single failure.

### Scenario 70 — A team proposes active-active writes to reduce latency

**Problem:** A team proposes active-active writes in two regions to reduce write latency, but both regions can update the same account, inventory unit, or username. During a network partition, both may accept conflicting mutations.

**Clarify mergeability:** Can the data be partitioned so each entity has one home writer? Are updates independent and commutative, or does each operation depend on the latest global state? Counters and append-only telemetry may allow merge rules; debiting a balance, reserving the last item, or claiming a unique username generally needs stronger coordination or a deliberate product trade-off.

**Design options:** Active-active compute with a single authoritative writer per entity can preserve a serialization point while letting users read locally from replicas. Use global consistency/consensus semantics if the business requires a globally ordered transaction. True multi-writer designs need an explicit conflict model, versioning, merge semantics, and a plan for deletes, uniqueness, and idempotency; last-write-wins based on wall-clock timestamps is unsafe for many business facts.

**Partition behavior:** State which writes are rejected, deferred, or accepted locally during a partition. If accepted writes can conflict, explain how that conflict is resolved and why the business invariant remains valid. Do not claim a payment or reservation committed when the region cannot establish authority. Failover requires fencing the former owner; DNS routing alone is not a write-ownership protocol.

**Trade-offs:** Coordination increases latency and can reduce availability. Local writes reduce latency but may require reconciliation and customer-visible conflicts. Model cost, replication lag, residency, and operational complexity.

**Follow-up:** Distinguish active-active stateless servers, regional read replicas, single-writer-per-entity, and true multi-writer state. They are different architecture choices.

### Scenario 71 — Object uploads overload application servers

**Problem:** Customers upload large files through application instances, consuming connections and bandwidth needed for normal API traffic.

**Clarify:** Must the application inspect or transform the file inline? What are the size/type limits, checksum requirements, authorization rules, malware scanning needs, and retention policy? Which operations must be synchronous and which can happen after transfer?

**Recommendation:** Authenticate and authorize the upload request, enforce tenant quota and allowed object path, then issue a short-lived, scoped pre-signed URL or multipart upload session. The client transfers bytes directly to object storage. On completion, validate object size/checksum and metadata, run malware/content inspection asynchronously, and mark the file visible/usable only after required checks pass.

**Data model:** Keep an upload/session record with owner, tenant, object key, expected size, status, expiry, checksum, and multipart state. Make create/finalize idempotent. A timeout after storage has accepted the object should be resolved by checking the object/session state rather than creating a second object accidentally.

**Security and cleanup:** Signed URLs are capabilities; restrict them to a specific key, operation, headers, and expiry. Do not expose bucket credentials. Clean up abandoned multipart uploads and quarantine untrusted files so they cannot be served from a privileged application origin before scanning.

**Signals and trade-offs:** Monitor incomplete multipart uploads, finalization failures, checksum mismatch, scan backlog/age, egress, object count, and storage growth. Direct upload reduces application bandwidth but moves critical policy to storage permissions and the completion workflow. In an interview, distinguish the control plane (authorize/create/complete/validate) from the data plane (transfer bytes).

### Scenario 72 — A long-running operation is cancelled after external work starts

**Problem:** A client cancels a data export or workflow, but workers and an external provider continue processing. The cancellation request races with work that may already have committed or cannot be reversed.

**Contract:** Define whether cancellation means “stop starting new steps,” “request stop at the next safe checkpoint,” or “compensate effects already performed.” Persist cancellation intent in the workflow state machine and return explicit states such as CANCEL_REQUESTED, CANCELLED, COMPLETED, or TOO_LATE rather than immediately claiming all work has stopped.

**Implementation:** Workers check cancellation before each stage and use version-checked/idempotent transitions. Propagate cancellation to queued work and downstream APIs where supported, but do not assume aborting an HTTP call undoes an external action. If an export has written temporary objects, delete them safely; if payment or shipment has already occurred, cancellation may require compensation or manual review.

**Race handling:** If completion and cancellation occur concurrently, one durable transition should win according to documented rules. Use a version/conditional update and record completed steps. For partial batch work, show which items finished, which were cancelled, and which require repair.

**Recovery and signals:** Track cancellation latency, workflows stuck in CANCEL_REQUESTED, compensation failures, external effects after cancellation, and oldest pending state. In an interview, explain what the caller sees if cancellation arrives after the side effect but before the local status commit.

### Scenario 73 — A database pool becomes exhausted during graceful deployment

**Problem:** New instances start before old instances drain, multiplying connection pools and exhausting the database during a rolling deployment.

**Capacity model:** Calculate peak connections, not only steady-state connections. If 20 instances normally each have a maximum pool of 40, they can open up to 800 connections; a 50% rollout surge can add 400 more before accounting for workers, scheduled jobs, migrations, monitoring, and admin sessions. Compare this with database limits and reserve capacity for failover.

**Design:** Set a fleet-wide connection budget, coordinate max surge/rollout batch size, mark old instances unready before draining, and let in-flight work finish within a bounded window. Ensure new instances do not immediately warm every cache or execute a large startup query. Avoid increasing pool size without checking database saturation; more connections may create more blocked transactions rather than more throughput.

**Diagnosis:** Separate pool-acquisition wait from database lock wait. Inspect active/idle/waiting pool counts, connection hold time, long transactions, lock graphs, query plans, connection churn, and database CPU/IO. A service can show low application CPU while all request threads are waiting for a connection.

**Recovery:** Pause rollout if connection pressure or p99 crosses thresholds, shed noncritical work, and restore capacity gradually. Test rollout, node loss, and database failover with realistic fleet size.

**Signals:** Total database sessions, per-instance pool utilization, acquisition time, transaction duration, waiters, database saturation, retry rate, and business p99. The key is to control aggregate concurrency across the fleet, not tune each pool in isolation.

### Scenario 74 — A cache key omission leaks tenant data

**Problem:** A cache key uses only a resource ID, omitting tenant or authorization context. Two tenants that request the same logical ID can receive the same cached response, potentially exposing private records.

**Containment:** Disable shared caching or bypass the affected route immediately, then purge vulnerable entries under the incident procedure. Preserve logs/metadata without copying sensitive response bodies into debug output. Determine the exposure window and affected tenants, and follow security/privacy response requirements rather than treating it as a mere cache correctness bug.

**Root cause analysis:** Reconstruct how the cache key was built, whether tenant identity was trusted, and whether cache hits bypass origin authorization. Review application cache, CDN, gateway, local cache, and read-model keys—one correct layer does not fix another unsafe layer. Check alternate representations where locale, role, permissions, query filters, or feature flags affect the response.

**Corrective design:** Build keys from canonical, validated dimensions that change the representation: tenant, user/permission scope where required, resource, locale, filters, and schema version. Do not trust caller-controlled headers as authorization evidence. Personalized responses should generally be private/non-cacheable unless an explicit safe caching design exists. Authorization must still be enforced at the resource-owning service, and sensitive cached values need suitable TTL, encryption/access control, and invalidation semantics.

**Validation:** Add automated cross-tenant and cross-role tests against the real cache/CDN configuration. Test anonymous versus authenticated requests, differing cookies/Authorization headers, cache-control/Vary behavior, and revocation. Stage policy changes and verify purge propagation.

**Follow-up:** Explain why “the backend is secure” is insufficient when a cache hit can return a stored response without reaching the backend.

### Scenario 75 — A distributed ID generator returns duplicates after failover

**Problem:** A distributed ID generator returns duplicates after failover because two nodes reuse the same worker ID or sequence range. The collision may cause overwritten records, broken references, or a unique-constraint failure well after the original incident.

**Root cause:** The uniqueness guarantee depended on exclusive allocation that was not preserved across restart, promotion, or network partition. An unreachable old generator may still be alive and issuing IDs. Promoting a standby based only on a heartbeat timeout can create split-brain allocation.

**Design:** For range allocation, persist each range and owner, mark it consumed/retired, and never reuse a range while an old owner might still issue from it. For time/worker/sequence schemes, allocate worker IDs durably and define fencing/epoch semantics. Handle clock rollback, skew, sequence exhaustion, and restart within the same time window. If using random IDs, size the namespace for the collision probability and keep uniqueness constraints where feasible.

**Containment and repair:** Stop unsafe generation or route allocations through the single proven authority until ownership is restored. A unique constraint can detect duplicates but does not decide which already-created record should win; preserve audit data and define referential repair. Do not silently overwrite an existing entity because an ID collided.

**Validation:** Run tests for concurrent allocator promotion, process pauses, VM/host cloning, worker-ID reuse, clock rollback, and multi-region partition. Monitor duplicate-key violations, allocation leases/epochs, clock drift, and generator failover.

**Follow-up:** Distinguish guaranteed uniqueness by coordination/exclusive allocation from probabilistic uniqueness by a large random namespace. Also explain why uniqueness does not imply a global chronological order.

### Scenario 76 — A read model rebuild disagrees with production results

**Problem:** A rebuilt read model has different totals, statuses, or search results from the current production projection. The rebuild job completes and the index responds, but the data may be incomplete or semantically different.

**Diagnose from one entity outward:** Select an affected record and trace its authoritative state, event/outbox record, broker offset, schema version, consumer logic, and projected document. Check the snapshot boundary, replay offsets, tombstones, duplicate handling, event ordering, timezone/currency semantics, and transformation-code version. Row counts alone will not detect a double-counted aggregate or a missing delete.

**Rebuild safely:** Use a consistent snapshot and capture changes that occur after it. Apply version-aware, idempotent projection writes so older snapshot data cannot overwrite newer events. Keep the candidate projection separate from the active one, replay up to a defined position, and ensure historical events remain readable by the current consumer. If CDC/log retention expires during rebuild, restart from a new snapshot rather than silently skip the gap.

**Validate before cutover:** Compare counts, checksums where meaningful, domain totals, representative entities, deleted records, and critical queries. Define an acceptance threshold and require explanations for all mismatches. Shadow-read the candidate and compare results without exposing it to users. Do not cut over while the discrepancy is unexplained.

**Recovery and rollback:** Preserve the old projection until the candidate is verified. If the candidate has already received new updates, rollback must account for those updates. Track rebuild progress, replay lag, mismatch counts, projection freshness, and errors.

**Follow-up:** Explain the exact snapshot/event boundary and how you prove the new projection includes every required update and delete.

### Scenario 77 — A rate limiter allows bursts after instance scaling

**Problem:** A rate limiter allows a sudden burst after autoscaling because every new instance starts with a full local token bucket. What was intended as a global 1,000 requests/second limit becomes the sum of local allowances.

**Diagnose:** Confirm the quota scope (process, tenant, account, provider, or platform) and compare accepted request rate with instance count. If each instance can independently grant the full quota, adding ten instances can permit roughly ten times the configured allowance. Repeated restarts can also mint fresh local budgets, and uneven load balancing may cause unfairness between tenants.

**Design options:** A shared atomic limiter can enforce a more accurate global budget but adds latency and a central dependency. A hierarchical design allocates bounded token budgets/leases from a global controller to local instances; this reduces per-request coordination but trades exactness for bounded overshoot and potentially stranded tokens. A local limiter is still useful as a per-instance safety cap, but do not treat it as the global contractual limiter.

Define refill rate, burst capacity, token lease lifetime, allocation on scale-out, and how unused tokens are reclaimed safely. Ensure rate semantics and concurrency admission are separate: a request rate may be acceptable while long-running operations exhaust workers. Decide behavior if the shared limiter fails according to the abuse/billing risk.

**Validation:** Test scale-out/in, restarts, region failover, limiter partition, and hot tenant keys. Monitor aggregate accepted rate, quota overshoot, provider throttling, limiter latency, rejection rate, and token allocation.

**Follow-up:** Explain how the design prevents each newly created instance from independently resetting the platform's global budget.

### Scenario 78 — A scheduled workflow misses several intervals

**Problem:** A scheduler was unavailable for two hours and must decide what to do with missed runs. Blindly replaying every missed interval can cause duplicate work or overwhelm dependencies; blindly skipping them can violate business requirements.

**Define a policy per job type:** A dashboard snapshot may be coalesced into one current run, cache refreshes may be skipped, and billing or compliance workflows may require every logical period to be processed or explicitly reconciled. State whether a missed occurrence expires, catches up, coalesces, or requires manual intervention. Do not apply one global scheduler policy to every job.

**Durable identity and safety:** Give each occurrence a deterministic ID such as job name + scheduled instant + tenant/partition. Record started/progress/completed/failed/skipped states durably and enforce uniqueness for the occurrence. Make steps idempotent and use fencing where stale workers could continue after lease expiry. For external side effects, preserve operation IDs and reconcile unknown outcomes before retrying.

**Catch-up control:** Bound concurrency and rate-limit replay. Prioritize jobs by business deadline and dependencies; catch-up must not starve live interactive traffic or exceed a provider quota. For time-windowed jobs, define whether calculations use the original scheduled interval or the current time.

**Auditability:** Keep a record of every interval and why it ran, was coalesced, skipped, or failed. Use versioned configuration to make the misfire policy visible and reviewed.

**Signals and tests:** Track missed-run count, oldest overdue occurrence, catch-up rate, duplicate effects, lease/fencing errors, and duration. Test scheduler downtime, rolling deployment, time-zone/DST changes, and concurrent failover.

**Follow-up:** Explain how the product chooses between skip, coalesce, and replay based on the business meaning of the task.

### Scenario 79 — A region is healthy but cannot reach the identity provider

**Problem:** Compute, database, and basic network health checks pass in one region, but authentication fails because the region cannot reach the identity provider or refresh signing keys. A basic health check says the region is healthy even though the user journey is broken.

**Diagnose:** Separate process health, dependency reachability, and end-to-end authentication. Determine whether the service verifies signed tokens locally using cached public keys, whether it requires remote token introspection on every request, and whether identity/policy changes must take effect immediately. Also check whether key rotation is expected during the outage and whether cached keys have exceeded their permitted freshness window.

**Design:** For self-contained signed tokens, cache validated public keys with a documented refresh/expiry policy and validate issuer, audience, signature, algorithm, and expiration. This can keep the hot path available during a brief identity-provider outage, but revocation and policy changes may be delayed. Remote introspection can provide fresher status but creates a synchronous dependency and latency/failure coupling. Choose per operation's risk; privileged/high-risk actions may need to fail closed if identity cannot be established safely.

**Regional routing:** Do not automatically send all requests to another region unless that region has capacity and can reach the required identity/data services. Routing cannot correct stale token policy by itself. Ensure fallback does not cross data-residency boundaries or bypass tenant authorization.

**Recovery:** Re-establish key/config freshness, handle tokens that expired during the outage, and restore routing gradually. Test whether local caches can safely serve existing tokens without accepting revoked or forged credentials.

**Signals:** Monitor authentication success, identity-provider latency/errors, key age/rotation failures, authorization denial changes, and synthetic user journeys. In an interview, distinguish authentication availability from general infrastructure health and explain the fail-closed boundary.

### Scenario 80 — A new storage engine benchmarks faster but raises operational risk

**Problem:** A new storage engine benchmarks faster, but the team lacks operational experience and the restore process is not proven.

**Evaluate:** Compare representative throughput and p95/p99 latency, data skew, transaction guarantees, failover, backup/restore, observability, security, migration, and on-call expertise. A benchmark that tests only the happy path can miss recovery behavior, compaction pressure, degraded performance, and operational toil.

**Decision:** Keep the existing store if the measured gain does not justify migration risk and ongoing support cost. If adoption is justified, define success metrics, a reversible migration, data-validation checks, and explicit exit criteria before moving critical traffic. Run failure and restore exercises using production-like data and load.

**Interview takeaway:** A technology choice is an investment in both runtime behavior and the team's ability to operate it. Faster benchmarks alone are not sufficient evidence.

### Multi-stage mock interview: Design a notification platform

Use this as one evolving discussion rather than separate trivia questions.

**Stage 1 — Clarify the contract.** Ask which channels are required, whether delivery is best-effort or deadline-bound, how user preferences and consent work, what “delivered” means, and how many notifications arrive at peak. Distinguish accepted, queued, provider-accepted, delivered, and read states.

**Stage 2 — Establish the baseline architecture.** Persist the business event and outbox record together, publish to a durable broker, route by channel and priority, and let workers call providers. Store attempt state and stable notification IDs. Keep provider calls off the user request path unless the product explicitly requires synchronous delivery.

```mermaid
flowchart LR
  S[Business Service] --> D[(Business DB + Outbox)]
  D --> P[Outbox Publisher / CDC]
  P --> B[Durable Broker]
  B --> R[Priority Router]
  R --> E[Email Workers]
  R --> M[SMS / Push Workers]
  E --> V[Email Provider]
  M --> W[SMS / Push Providers]
  E --> T[(Delivery State)]
  M --> T
```

**Stage 3 — Traffic grows 10×.** Recalculate event rate and provider quotas. Scale consumers only within broker partition and downstream capacity, coalesce redundant updates where semantics allow, reserve capacity for urgent alerts, and shed or defer low-priority work according to policy. Add bounded queues and queue-age alerts.

**Stage 4 — One provider fails for 45 minutes.** Open a provider-specific circuit, retain retryable work durably, use capped backoff with jitter, and route to an alternate provider only if the channel, consent, cost, and delivery contract permit it. Expire time-sensitive marketing notifications rather than delivering them hours late; security alerts may need a different policy.

**Stage 5 — Users report duplicate notifications.** Trace notification IDs through outbox, broker, consumer attempts, provider request IDs, and callbacks. A crash after provider acceptance but before local acknowledgement can cause a duplicate. Use durable deduplication and provider idempotency where available; if the provider offers no idempotency, document the remaining duplicate window and reconcile.

**Stage 6 — Senior trade-off.** Explain that broker durability does not guarantee end-user delivery. The system can guarantee durable acceptance and controlled retry attempts, but provider outages, invalid destinations, and downstream behavior still affect final delivery. Track queue age, attempt rate, provider success, delivery latency percentiles, duplicate reports, and cost per successful delivery.

### Multi-stage mock interview: Ticket booking under contention

**Stage 1 — Define the invariant.** Clarify whether seats are exclusive, whether temporary holds exist, how long a hold lasts, and when payment is captured. The core invariant is that at most one active reservation owns a seat for a given event.

**Stage 2 — Design the critical path.** Use an authoritative database operation with a unique constraint or conditional state transition to reserve a seat. Store hold expiry and reservation state durably. A cache can display approximate availability but must not authorize the reservation.

**Stage 3 — Traffic spikes at onsale.** Use a waiting room or admission control, per-event concurrency budgets, and queue-based load leveling. Prioritize fairness and protect the database. Autoscaling application instances alone may amplify database contention.

**Stage 4 — Payment times out after the seat is held.** Keep the reservation pending while the payment outcome is unknown. Query the provider or reconcile using the stable payment operation ID. Release or confirm the hold according to explicit deadlines and state transitions; do not create a second payment on a blind retry.

**Stage 5 — Hold expiry races with confirmation.** Make confirmation conditional on the hold still being valid and in the expected state. The expiry worker must also perform an atomic transition. Only one transition should win; delayed cleanup must not allow an expired hold to be confirmed.

**Stage 6 — Region failure.** If the system cannot establish the authoritative owner for a seat, reject or queue new reservations rather than accept conflicting ownership. Recover the authority, reconcile payment and reservation state, then reopen writes.

### Multi-stage mock interview: Payment processing with uncertain outcomes

**Stage 1 — Define business guarantees.** Clarify whether the system authorizes or captures funds, whether partial refunds exist, what status the client receives, and what provider idempotency features are available. Distinguish request acceptance from a confirmed financial outcome.

**Stage 2 — Durable operation model.** Persist an operation ID, idempotency key, request fingerprint, amount/currency, and state before calling the provider. Enforce uniqueness for the logical operation and return the same operation for retries using the same key. Reject a key reused with a different payload.

**Stage 3 — Provider times out.** Mark the result unknown or pending. Query provider status or wait for a verified webhook; do not assume timeout means failure. If the provider cannot reliably answer, reconcile against settlement data and keep the operation unresolved until evidence is available.

**Stage 4 — Duplicate or out-of-order webhooks.** Authenticate signatures, persist event IDs, deduplicate, and apply only legal state transitions. Use provider references and stable operation IDs to correlate callbacks. Do not allow an old “pending” event to overwrite a confirmed terminal state.

**Stage 5 — Regional failover.** Preserve one write authority per operation or use a store that guarantees the required consistency. Fence old writers, maintain a durable audit trail, and reconcile before resuming ambiguous operations. Avoid accepting independent financial writes in two partitioned regions.

**Stage 6 — Senior trade-off.** Explain that exactly-once network delivery is not the same as exactly-once business effect. The design combines durable state, idempotency, constraints, authenticated callbacks, and reconciliation to make repeated attempts safe and resolve uncertain outcomes. Measure unknown-outcome age, duplicate-key conflicts, provider latency, callback lag, and settlement mismatches.


## 19. End-to-end case-study index

This section includes eight worked end-to-end designs for frequently used senior-level interview prompts, followed by an index of additional systems to practise. These are reference designs, not one-size-fits-all answers. In an interview, clarify the prompt, state assumptions, adapt the capacity model, and explain which requirement would change each major choice.

### Worked design 1: URL shortener

### Requirements and assumptions

A URL shortener accepts a long URL and returns a short URL. A redirect request resolves the code and sends the client to the destination. Before designing, clarify whether links are public or private, whether users can choose aliases, whether links expire, whether click analytics must be real time, and whether abusive destinations need moderation. For the baseline design, assume anonymous creation, random short codes, optional expiration, high read-to-write ratio, and analytics that may be delayed.

The correctness properties are simple but important: a code must map to one canonical destination during its lifetime; code generation must handle collisions; disabled/expired links must not redirect; and redirects should be low-latency. Analytics must not make the redirect path depend on a slow analytics store.

### Capacity estimate

Use explicit assumptions rather than presenting the following values as universal. Suppose the service creates 10 million links/day and handles 100 million redirects/day. That is roughly 116 creates/s and 1,157 redirects/s on average. If peak redirects are 10 times average, plan initially for about 11,600 redirects/s, then account for hot links, bots, retries, and regional skew.

If each link record averages 300 bytes of logical payload and metadata, one year of 10 million new links/day is about 1.095 TB of raw logical data before indexes, database/page overhead, replicas, backups, and analytics. Use measured row/index sizes before sizing production storage. The service is likely read-heavy, so cache effectiveness and hot-link behavior will matter more than create QPS.

### Architecture and request flows

```mermaid
flowchart LR
  C[Client] --> E[Edge / Rate Limit]
  E --> G[API Service]
  G --> D[(Link Database)]
  G --> O[Outbox / Event]
  O --> B[Analytics Stream]
  B --> A[Analytics Workers]
  A --> W[(Analytics Store)]
  R[Redirect Request] --> E
  E --> L[Redirect Service]
  L --> K[(Cache)]
  K -->|Miss| D
  K -->|Hit| H[HTTP Redirect]
  D --> H
```

**Create flow:** authenticate or rate-limit the caller, validate and canonicalize the destination, generate a candidate code, insert the mapping under a unique constraint, and return the short URL only after the mapping is durably committed. If the code collides, generate another candidate and retry within a bounded limit. If a user-selected alias is supported, reserve it atomically and return a conflict if it already belongs to another active link.

**Redirect flow:** extract and validate the code, check cache, then read the authoritative mapping on a miss. Check expiry/status and apply redirect policy before returning the redirect response. Emit click telemetry asynchronously; if analytics is unavailable, the redirect should normally still work. Do not block every redirect on a synchronous analytics write.

### Data model, APIs, and code generation

A link record may include `code` (unique), `destination`, `owner_id` (nullable for anonymous links), `created_at`, `expires_at`, `status`, and a version. The main API can be `POST /links`, returning `201` with the code and short URL, plus `GET /{code}` for redirect resolution and `DELETE /links/{code}` to disable a link when the caller is authorized.

Code-generation approaches have different trade-offs. A central numeric ID encoded in Base62 produces compact codes but requires safe ID allocation and can reveal approximate creation order. Random codes avoid centralized sequencing but require enough entropy and a unique constraint to handle collisions. A distributed time/worker ID can improve coordination characteristics but adds worker-ID and clock assumptions. Do not rely on “collision is unlikely” without a datastore uniqueness check.

### Scaling, caching, and correctness

Cache popular code-to-destination mappings with TTLs aligned to expiry and revocation requirements. Hot links may need local caching or request coalescing. A disable/delete must invalidate caches and consider CDN/browser caching: a permanent 301 may be cached by clients and difficult to revoke, while a 302/307-style temporary redirect gives more control but may add repeat load. The right HTTP status depends on product semantics and client behavior.

Prevent abuse with creation quotas, destination validation, malware/phishing checks where appropriate, enumeration protection, safe redirect policy, and limits on response amplification. Do not allow an open redirector to become a security problem. Store click events in an event stream partitioned by a suitable key, then aggregate asynchronously. Define whether analytics can be approximate and how lost telemetry is detected.

### Failure modes and trade-offs

If the cache is unavailable, protect the database with bounded concurrency and coalesced fills; do not allow all traffic to stampede the origin. If the database is unavailable, redirects may be served from a still-valid cache only if the revocation/expiry contract permits it. If code generation fails, do not return a URL until its mapping is durable. If analytics backs up, measure queue age and apply retention or sampling policy without blocking redirects.

In an interview, explain what happens when two requests choose the same code, a link expires while cached, a popular code receives a burst, or a user asks to revoke a link that was returned as a permanent redirect.

### Worked design 2: News feed / social timeline

### Requirements and consistency boundary

Clarify whether the system needs a home feed, profile timeline, ranked recommendations, or all three; how many users one account can follow; whether privacy/block rules must apply immediately; how fresh posts must be; and whether the feed is strictly chronological or ranked. The baseline design returns a personalized, ranked feed with eventual freshness, but privacy and deletion decisions must not be bypassed by stale precomputed data.

The important split is between **candidate generation** and **final eligibility/ranking**. Candidate posts can be precomputed for speed, while current visibility, deletion, block, and authorization state must be checked before returning content when the product requires it.

### Architecture

```mermaid
flowchart LR
  U[Client] --> A[Feed API]
  A --> T[(Timeline Cache)]
  T -->|Miss| R[Timeline Read Service]
  R --> C[(Candidate Timeline Store)]
  R --> P[Post / Visibility Service]
  P --> D[(Authoritative Post Store)]
  W[Post Write API] --> D
  D --> O[Outbox / CDC]
  O --> B[Event Stream]
  B --> F[Fan-out Workers]
  F --> C
  A --> S[Ranking Service]
  S --> A
```

A post-write path durably stores the post and publishes a post-created event through an outbox or equivalent reliable mechanism. Fan-out workers place the post ID into followers' candidate timelines for users where write-time fan-out is reasonable. A feed-read path loads candidate IDs, filters current visibility/deletion constraints, fetches necessary post data, ranks candidates, and returns a page cursor.

### Fan-out on write versus fan-out on read

**Fan-out on write** pre-populates a follower's timeline when a post is created. Reads are fast because candidate lists already exist, but one post by a celebrity with tens of millions of followers can generate enormous write amplification and queue delay. It also means a follower's timeline can temporarily lag the source post.

**Fan-out on read** stores posts by author and merges recent posts from followed authors when the feed is requested. It avoids massive fan-out work for celebrities but makes reads more expensive, especially for users following many high-volume authors. A hybrid is common: precompute for ordinary authors, but include high-follower authors at read time or through a separate candidate path.

Estimate the trade-off using follower distribution, post rate, feed read rate, and desired freshness—not only total users. If a normal post goes to 500 followers but a celebrity post goes to 20 million, the system needs an explicit strategy for the skewed tail.

### Data model and APIs

Typical entities include users/follow relationships, posts, visibility rules, and candidate timeline entries keyed by `user_id` and ordered by score/time plus a stable tie-breaker. APIs might include `POST /posts`, `GET /feed?cursor=...`, `POST /users/{id}/follow`, and mutation endpoints for delete/privacy changes. Cursor pagination should be stable and opaque; offset pagination becomes costly and unstable at scale.

Timeline entries should reference post IDs rather than duplicate every field if minimizing fan-out storage is important. Denormalized content snapshots can reduce reads but require update/delete propagation. Decide which fields must be fresh at read time and which may be stale.

### Ordering, privacy, and deletion

A feed often needs order per user timeline, not one global order across all users. Use a score/time key and unique tie-breaker. Wall-clock timestamps alone do not establish strict causal order; events can be delayed or replayed. Versioned/tombstone updates prevent stale events from restoring deleted posts or old visibility. If a user blocks another user or makes a post private, the read path must prevent unauthorized content from being returned even if an old timeline entry remains.

Feed freshness is usually eventual: the post may appear after some delay. Make this visible in telemetry through outbox lag, fan-out queue age, and timeline propagation delay. Reconciliation can repair missing candidate entries by comparing source posts/follow relationships or replaying retained events.

### Scaling, failure, and trade-offs

Cache timeline slices for active users, but use short/appropriate freshness rules and avoid caching a visibility decision that can become invalid. Separate write fan-out from feed reads, apply backpressure, and make workers idempotent. A queue backlog after an outage may take time to drain; do not replay it without capacity planning. Ranking can degrade to a simpler chronological feed if a noncritical ranking service is unavailable, but privacy filtering cannot be skipped.

In an interview, be ready to discuss celebrity skew, follow/unfollow consistency, delete propagation, a feed read during fan-out lag, timeline rebuild, cursor stability, and how a viral event changes capacity.

### Worked design 3: Chat / messaging platform

### Requirements and delivery semantics

Clarify one-to-one versus group chat, group-size limits, message history, online presence, media attachments, search, encryption, delivery/read receipts, retention, and offline delivery. A useful baseline supports one-to-one and group messages, durable history, real-time delivery to online users, and replay after reconnect. Distinguish the states **accepted**, **persisted**, **delivered to a device**, and **read**; they are not synonymous.

The key invariant is that an acknowledged durable message must remain retrievable within the promised retention, and per-conversation ordering must be well defined. Do not promise globally ordered messages across all conversations unless the product truly needs that.

### Architecture

```mermaid
flowchart LR
  C[Clients] <-->|WebSocket| G[Connection Gateways]
  G --> R[Session / Routing Service]
  R --> M[Message Service]
  M --> D[(Durable Message Store)]
  M --> O[Outbox / Event Stream]
  O --> B[Conversation Partitions]
  B --> W[Delivery Workers]
  W --> G
  W --> P[Push Notification Service]
  G --> X[Presence Service]
```

Gateways maintain connections, authenticate clients, and route outbound messages. The message service validates sender membership and conversation permissions, assigns a stable message ID and per-conversation sequence/version where needed, persists the message, and publishes delivery work. The durable store remains the source of history; gateways and presence are transient routing aids.

### Data model and write flow

A message record may contain `conversation_id`, `message_id`, `sender_id`, `sequence` or version, `created_at`, `payload_reference`, and status/retention metadata. Store large media in object storage and keep metadata/reference in the message database. Access control must be checked for reads and writes, not only when the socket is opened.

The sender submits a message with an idempotency/client message ID. The service validates access, writes the message durably, and only then acknowledges persistence. It then dispatches to recipient gateways and offline push mechanisms. If the publisher crashes after persistence but before dispatch, an outbox or CDC pipeline eventually resumes delivery. If the recipient is offline, the message stays in durable storage for later sync.

For duplicate sends, the client message ID plus sender/conversation scope should resolve to the original message. Consumer redelivery must not create duplicate message rows or repeated side effects. A delivered receipt means a device acknowledged receipt according to the protocol; a read receipt is a separate user/application signal.

### Ordering, scaling, and presence

Partitioning messages by `conversation_id` can preserve per-conversation order and make history queries efficient, but a very large group can become a hot partition. A message sequence may be allocated by a conversation owner or transactional store; define how sequence assignment behaves during failover. Global total order is usually unnecessary and costly.

Connection gateways scale based on active sockets, messages per second, outbound buffer size, and fan-out—not just HTTP QPS. Maintain a session map from user/device to gateway with leases/heartbeats, but treat it as ephemeral. Presence should tolerate stale status after disconnect and should not broadcast every heartbeat to all friends. Group messages may need fan-out-on-write or read-time delivery depending on group size.

Reconnect should authenticate again, resume from a durable cursor/sequence, replay retained messages, and fall back to a recent-history snapshot if the cursor expired. Deduplicate replayed message IDs. Use bounded outbound queues and backpressure so a slow client cannot consume unbounded gateway memory.

### Security and failure modes

Use TLS, secure authentication, conversation-level authorization, rate limits, spam/abuse controls, and careful media access. If end-to-end encryption is required, key management and multi-device history become major requirements that change the architecture. Presence, typing indicators, and read receipts may be best-effort; message persistence and authorization cannot be silently degraded.

If a gateway fails, clients reconnect with jitter and recover history. If the broker is delayed, messages remain persisted but delivery latency rises; alert on oldest undelivered age. If a message is stored but response lost, idempotency returns the same message on retry. In an interview, explain connection churn, hot group scaling, ordering scope, reconnect replay, and how online status differs from durable delivery.

### Worked design 4: Ticket booking with seat holds and payment

### Requirements and invariants

Clarify seat inventory, hold duration, payment flow, refund behavior, queue/waiting-room requirements, ticket issuance, cancellation, and traffic burst. The critical invariant is **one seat cannot be sold to two orders**. A high-traffic ticket drop also needs fairness and overload control; simply autoscaling the API can overwhelm the database before it can safely allocate seats.

Separate browsing/search from the authoritative booking path. Event pages and seat maps can be cached and slightly stale; the final hold/booking decision must be made atomically against authoritative inventory.

### Architecture and flow

```mermaid
flowchart TD
  C[Client] --> W[Waiting Room / Admission]
  W --> A[Booking API]
  A --> I[Inventory Service]
  I --> D[(Authoritative Inventory DB)]
  A --> O[(Order / Workflow State)]
  O --> P[Payment Orchestrator]
  P --> X[Payment Provider]
  P --> E[Outbox / Event Bus]
  E --> N[Ticket / Notification Workers]
  S[Search / Seat Map Read Model] --> C
```

A typical flow: the client passes admission control, requests a hold, the inventory service atomically changes a seat from AVAILABLE to HELD with a unique hold/owner/expiry, then the order workflow requests payment. After confirmed payment, the workflow commits the booking and issues the ticket. If payment fails definitively, it releases the hold. If provider status is unknown after a timeout, it queries/reconciles before retrying or releasing inventory.

### Data correctness and API design

A table might include `event_id`, `seat_id`, `state`, `hold_id`, `order_id`, `expires_at`, and version. Enforce uniqueness/conditional state transitions in the database; an application check that reads “available” then inserts is vulnerable to concurrent races. Use idempotency keys for hold/order/payment commands. The hold confirmation path must atomically verify the hold is still valid and owned by the order.

Potential APIs include `POST /holds`, `POST /orders`, `POST /payments`, `GET /operations/{id}`, and cancellation/confirmation endpoints. Long-running payment should return pending state and a durable operation ID where needed rather than keeping an HTTP request open indefinitely.

Expiry jobs should release holds idempotently, but correctness must not rely on a cleanup timer executing exactly on time. A delayed job should not release a seat already confirmed; use conditional version/state checks. Payment may succeed after a hold expires, so the workflow needs a defined compensation—reserve alternative inventory, refund/void, or escalate for review.

### Peak-load strategy and failure modes

Use a virtual waiting room or admission-control token to bound concurrent checkout requests. Apply fair limits per identity/device/account, while guarding against bots and account abuse. Cache event pages and seat-map reads, but never treat cached availability as final booking authority. Shard or partition inventory by event/venue when it fits access patterns, but one very hot event can still overload its shard; partitioning alone cannot split a strict per-seat invariant arbitrarily.

Monitor hold conflicts, inventory lock wait, payment unknown outcomes, queue age, admission rejection, ticket issuance delay, and reconciliation mismatches. If the payment provider times out after charging, keep the payment operation UNKNOWN/PENDING and reconcile with a stable provider key; do not create a new charge just because the response was lost. If the database fails, reject or queue only those operations that can be durably accepted without violating inventory correctness.

In an interview, walk through two users requesting the same seat, payment success with a lost response, hold expiry racing with confirmation, and a 10× traffic burst. Explain which checks are cached and which must be authoritative.

### Worked design 5: Multi-channel notification platform

### Requirements and workload model

Clarify supported channels (email, SMS, push, in-app), priority classes, delivery deadlines, personalization/templates, user preferences, quiet hours, provider quotas, deduplication, and delivery-status semantics. Define what “delivered” means: durably accepted by your API, sent to the provider, accepted by a carrier, or acknowledged by the recipient's device. These are different milestones.

A useful baseline durably accepts notification requests, processes them asynchronously, respects preferences and provider quotas, retries transient failures, and exposes status. Notifications vary in urgency: a security alert should not be trapped behind a bulk campaign.

### Architecture

```mermaid
flowchart LR
  C[Product Services] --> A[Notification API]
  A --> D[(Notification DB + Outbox)]
  D --> O[Outbox Publisher]
  O --> Q[Priority / Channel Queues]
  Q --> W[Workers]
  W --> P[Preference / Template Service]
  W --> L[Provider Quota Limiter]
  L --> X[Email / SMS / Push Providers]
  X --> R[Receipt / Callback Handler]
  R --> D
  Q --> DLQ[DLQ / Quarantine]
```

The API validates request, tenant, template, and idempotency identity, then commits a notification record and outbox event in a local transaction. It returns an operation/notification ID only after durable acceptance. Workers select the correct channel and priority, check preferences/quiet hours, render templates, send through the provider limiter, and persist attempts and outcome. Provider callbacks are verified and processed idempotently.

### Data model and state machine

A notification record may include `notification_id`, `business_event_id`, `recipient`, `channel`, `priority`, `template_version`, `scheduled_at`, `expires_at`, `status`, and attempt history/reference. Model states such as ACCEPTED, QUEUED, SENDING, PROVIDER_ACCEPTED, DELIVERED, FAILED, EXPIRED, and NEEDS_REVIEW according to channel semantics. A single mutable status without attempt history can hide important retries or uncertain outcomes.

Deduplication should be scoped to a business notification identity—for example, event + recipient + channel + template/purpose—so retrying the same upstream event does not send duplicate messages. Do not deduplicate two genuinely distinct reminders just because the recipient and template match. Idempotency depends on business semantics, not simply identical payloads.

### Quotas, retries, and recovery

The provider may enforce an account-wide quota across all workers/regions. Use a coordinated limiter or hierarchical token budget; a per-instance limiter can exceed the contract as autoscaling increases instance count. Reserve capacity for critical notifications, split bulk and urgent queues, use backoff/jitter for transient failure, honor Retry-After, and send exhausted or invalid payloads to a monitored DLQ. Avoid retrying permanent errors such as invalid recipient or template schema forever.

A provider timeout can be an uncertain outcome: it may have accepted the message even if the service received no response. Use provider idempotency/delivery references where supported, query status if available, and reconcile unknown attempts. Avoid automatically retrying with a new identity when duplicate messages would be harmful.

Track queue depth/age, accepted-to-provider latency, delivery confirmation latency, provider throttling, attempt count, DLQ age, expiration, and cost per successful delivery. If provider capacity returns after a long outage, throttle catch-up so the backlog does not cause another quota incident. In an interview, explain priority fairness, durability before 202 acceptance, duplicate handling, and the precise guarantee represented by each delivery status.

### Trade-offs and follow-up probes

The platform trades fast API acceptance for asynchronous completion. Separate per-channel adapters isolate provider-specific policies, but increase operational surface. A multi-provider strategy improves resilience only if the secondary has compatible semantics, capacity, and idempotency behavior. The interviewer may ask what happens when preferences change after queueing, the provider callback arrives before the synchronous response, the retry budget is exhausted, or the user requests deletion while messages remain queued.

### Worked design 6: Distributed API rate limiter

### Requirements and policy

Clarify whether the limiter protects against abuse, enforces a customer contract, protects a provider quota, or controls system concurrency. The required guarantee differs: approximate burst protection can use local state, while a strict tenant/account quota requires coordination at the scope where that quota is enforced. Define keys (IP, API key, user, tenant, route, provider account), rate, burst, reset semantics, and fail-open/closed behavior.

A rate limiter should return allow/deny plus retry guidance, be low latency, and remain available enough not to become the gateway's bottleneck. It should also protect against unbounded key cardinality and spoofed identity.

### Architecture

```mermaid
flowchart LR
  C[Client] --> E[Edge / Gateway]
  E --> L[Rate Limit Service]
  L --> K[(Sharded Counter / Token State)]
  L --> P[Policy / Quota Control Plane]
  E --> S[Backend Service]
  P --> L
```

The data plane evaluates a request using a trusted identity/route key and a configured policy. The control plane owns limits and rollout/versioning. Cache validated policy locally so every request does not need a synchronous configuration lookup, but define how quickly changed or revoked policies take effect.

### Algorithm and consistency choices

A **token bucket** stores a refill rate, capacity, and current token balance; it allows controlled bursts while bounding long-run rate. A **fixed window** is simple but may permit a double burst around a window boundary. A **sliding window** smooths this effect but generally needs more state/work. A **leaky-bucket shaper** queues work for a smooth output rate; policing variants reject overflow. A concurrency limiter answers a different question—how many operations may be in flight—and may be needed alongside a rate limiter.

A local limiter is fast and remains available during a shared-store outage, but each instance has its own budget and total allowance can multiply with autoscaling. A shared atomic store enforces a more global view but adds network latency, hot keys, and a dependency. Hierarchical token allocation lets a global controller allocate bounded budgets to instances; it trades exact instantaneous enforcement for lower hot-path coordination and potentially stranded tokens.

For distributed counters, the algorithm must update state atomically. A read-then-write sequence can race and allow excess traffic; use a datastore atomic script/transaction or a dedicated rate-limit primitive. Avoid one global counter key that all requests contend on. Partition by the quota scope and analyze the hottest tenant/route.

### Failure handling, overload, and security

Define what happens if the limiter state store is slow or unavailable. A strict billing quota may need fail-closed behavior; coarse abuse protection may permit a conservative local fallback. Do not fail open without bounds. Ensure authenticated identity comes from trusted authentication context rather than arbitrary headers, and do not rely on IP address alone when clients share NAT or attackers rotate addresses.

Apply inexpensive edge limits before expensive authentication/parsing, followed by identity- and operation-cost limits. A bulk export or search request should not necessarily consume the same budget as a cheap read. Bound key cardinality and TTL for anonymous identifiers. Use 429 and Retry-After when clients can act on it, but assume hostile clients may ignore advice.

### Capacity, observability, and follow-ups

Estimate checks per second at all policy scopes. If 50,000 requests/s require three global checks each, the limiter may see 150,000 state updates/s before retries or bursts. Measure hot-key skew, state-store latency, rejected requests, quota overshoot, key cardinality, and limiter-induced p99. Test autoscaling, region failover, store latency, provider quota exhaustion, and an attacker generating many new keys.

In an interview, be ready to explain why a token bucket fits the policy, how the counter update is atomic, how quotas behave when the fleet doubles, and whether correctness or availability takes priority if the limiter backend fails.

### Worked design 7: Search and autocomplete service

### Requirements and query model

Clarify whether the product needs full-text search, prefix autocomplete, typo tolerance, relevance ranking, filters/facets, personalization, language support, and freshness. Search is often a derived read model: the authoritative catalogue/content store owns the original data, while the search index is optimized for retrieval and may lag.

Define measurable targets: query p95/p99, index freshness, update/delete propagation, result count, and acceptable behavior if ranking or suggestions are unavailable. Autocomplete needs especially low latency and may have very high request volume because clients request suggestions on every keystroke.

### Architecture

```mermaid
flowchart LR
  C[Client] --> G[Search API / Gateway]
  G --> A[Autocomplete Index]
  G --> S[Search Cluster]
  W[Content Write API] --> D[(Authoritative Store)]
  D --> O[Outbox / CDC]
  O --> I[Indexing Pipeline]
  I --> S
  I --> A
  G --> R[Ranking / Filters]
  R --> C
```

A write commits to the authoritative store, then publishes a reliable change through an outbox/CDC stream. Indexing workers transform the record into search documents and update the full-text and autocomplete indexes. Query services validate filters and authorization, perform retrieval/ranking, and return results with a freshness or partial-result indicator when needed.

### Index and query design

For full-text search, the index may store analyzed tokens, fields, filters, and ranking features. For autocomplete, common choices include a prefix trie/FST, edge n-grams, or a dedicated suggestion index; each trades memory, update cost, typo handling, and ranking flexibility. Avoid querying the full search index in an unrestricted scan on every keystroke. Debounce requests client-side, cap query length/page size, cache high-frequency public prefixes, and use per-user/IP abuse controls.

Use explicit field mappings and analyzer versions. Schema/analyzer changes often require a new index generation and reindexing. Keep source documents or events sufficient to rebuild the index; do not treat the search index as the sole source of truth unless that is an explicit datastore design.

### Freshness, correctness, and privacy

Make document updates idempotent and version-aware so a delayed older event cannot overwrite a newer version. Use tombstones/version checks to prevent old events from resurrecting deleted content. Index lag should be measured from authoritative commit to searchable version. If the product displays a price or inventory result, the checkout/write path must revalidate current values with the authoritative owner.

Authorization and tenant filtering must be enforced on the search path. A shared cache key that omits tenant, visibility, locale, or entitlement can leak data. Search queries must not bypass private-content checks just because a document remains in an index after revocation. Define deletion and privacy-erasure behavior across replicas, caches, suggestions, and analytics.

### Scaling, recovery, and testing

Partition the index according to query locality and data volume; more shards can increase parallelism but add coordination, merge cost, metadata, and fan-out latency. A query across many shards may be dominated by the slowest shard. Use routing where queries have a reliable tenant/region key, but avoid a hot shard from a dominant tenant.

To rebuild, create a new index generation, load a consistent snapshot, replay changes from the snapshot/log position, validate counts/samples/deletes, shadow-query, then switch an alias or routing pointer. Track index lag, indexing errors, shard health, query p99, cache hit ratio, zero-result rate, and result-quality metrics. In an interview, explain why a database index is not enough, how prefix search differs from full-text search, how deletes/updates propagate, and how a rebuild avoids losing live changes.

### Worked design 8: Large file storage and synchronization

### Requirements and metadata model

Clarify maximum file size, object count, version history, sharing/access control, synchronization, chunking, deduplication, retention, encryption, geographic durability, and recovery targets. Separate the **control plane** (metadata, authorization, upload sessions, versions, sharing) from the **data plane** (large object bytes). Sending every byte through application servers often wastes compute and connection capacity.

The metadata store may hold `file_id`, `owner/tenant`, `object_key`, size, checksum, MIME type, version, ACL, created/updated time, retention state, and upload status. Object storage holds the bytes. Avoid using a user-supplied filename as the storage key or assuming that an upload is available merely because bytes exist.

### Architecture and upload flow

```mermaid
flowchart LR
  C[Client] --> A[Metadata / Upload API]
  A --> M[(Metadata DB)]
  A --> U[Scoped Upload URL / Session]
  C --> O[(Object Storage)]
  O --> V[Validation / Scan Workers]
  V --> M
  C --> D[Download API]
  D --> M
  D --> S[Scoped Download URL / CDN]
  S --> O
```

The client requests an upload session. The API authenticates the caller, checks tenant quotas, creates a scoped object key/session, and returns a short-lived pre-signed URL or multipart-upload capability. The client uploads directly to object storage, then calls a finalize endpoint. Finalization verifies the object exists, expected size/checksum matches, and required validation has completed before the metadata state becomes AVAILABLE.

For large objects, multipart/resumable upload lets clients retry failed parts without retransferring the entire file. Persist session state and part checksums; make create/finalize idempotent. Clean up incomplete multipart uploads after a safe timeout. If validation includes malware scanning or transformation, expose UPLOADED/PROCESSING/AVAILABLE/REJECTED states and do not serve the object before policy checks finish.

### Download, consistency, and security

Authorize every download using trusted identity/ACL and issue a short-lived scoped URL or proxy the request where policy requires. If access is revoked, signed URL lifetime and CDN cache behavior define the exposure window. Do not place sensitive tokens in logs or public URLs unnecessarily. Separate user-controlled metadata from internal object keys, validate content type/size, and protect against active-content execution.

Metadata and object storage are separate systems, so partial failures matter. An object may upload successfully while finalization fails; metadata can remain pending until a reconciliation worker detects the object and completes or removes the orphan. Conversely, metadata must not point to an object that was never durably stored. Use checksums, versioned object IDs, idempotent finalize, and reconciliation for orphaned data.

Versioning and soft delete help recovery but increase storage and interact with privacy deletion. Lifecycle rules can move cold data to cheaper tiers, while restore delay and minimum storage-duration costs may affect the product contract. Backups must preserve metadata and any version references required to recover objects.

### Capacity, reliability, and trade-offs

Estimate new bytes/day × retention before adding replication, version history, multipart overhead, object metadata, backups, and cross-region copies. Include request rates for metadata separately from byte throughput. CDN caching helps downloads of eligible content but not private content unless cache policy correctly respects authorization. Direct upload reduces application bandwidth; it shifts work to object-store policy, signed-link security, finalization, scanning, and cleanup.

Track upload completion rate, incomplete multipart sessions, checksum failures, scan backlog, object/metadata mismatches, access-denied rates, egress, storage growth, and restore time. In an interview, explain why the app server should not proxy bytes, how resumability and finalize work after a lost response, and how private objects remain private through CDN/cache/URL layers.



### Core systems

| System | Level / concepts | References |
|---|---|---|
| URL shortener | Intermediate: IDs, redirects, cache, hot keys | [URL shortener design walkthrough](https://systemdesign.cloud/part3-design-problems/url-shortener) |
| Pastebin | Intermediate: expiry, storage, read-heavy traffic | [System Design Primer — Pastebin / Bitly](https://github.com/donnemartin/system-design-primer#design-pastebincom-or-bitly) |
| Distributed unique ID generator | Advanced: uniqueness, time, coordination | [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| Distributed rate limiter | Advanced: token bucket, atomicity, quotas | [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| Notification service | Advanced: queues, priorities, retries | [AWS Builders’ Library](https://aws.amazon.com/builders-library/) |
| Distributed job scheduler | Advanced: durable scheduling, leases | [Google SRE Books](https://sre.google/books/) |

### Social and communication systems

| System | Level / concepts | References |
|---|---|---|
| Chat/messaging platform | Advanced: WebSockets, ordering, delivery | [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| News feed | Advanced: fan-out, ranking, caching | [System Design Primer — Twitter timeline / Facebook feed](https://github.com/donnemartin/system-design-primer#design-the-twitter-timeline-and-search-or-facebook-feed-and-search) |
| Photo-sharing platform | Advanced: object storage, metadata, CDN | [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| Email service | Advanced: queues, abuse controls, delivery status | [Google SRE Books](https://sre.google/books/) |
| Real-time collaboration | Advanced: concurrent edits, conflicts | [Designing Data-Intensive Applications](https://dataintensive.net/) |

### Commerce and transactions

| System | Level / concepts | References |
|---|---|---|
| E-commerce platform | Advanced: catalogue, cart, orders, inventory | [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| Shopping cart and checkout | Advanced: idempotency, workflow | [AWS Builders’ Library](https://aws.amazon.com/builders-library/) |
| Payment processing | Senior: unknown outcomes, ledger, reconciliation | [Stripe Engineering](https://stripe.com/blog/engineering) |
| Ticket booking | Senior: seat holds, contention, expiry | [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| Hotel booking | Senior: date ranges, inventory, overbooking | [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| Inventory reservation | Senior: atomic updates, holds, compensation | [AWS Builders’ Library](https://aws.amazon.com/builders-library/) |
| Food delivery | Senior: geospatial matching, workflows | [Uber Engineering](https://www.uber.com/blog/engineering/) |

### Media and storage

| System | Level / concepts | References |
|---|---|---|
| File storage and synchronization | Advanced: object storage, metadata, versions | [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| Video streaming | Senior: transcoding, CDN, adaptive bitrate | [Netflix TechBlog](https://netflixtechblog.com/) |
| Image-processing pipeline | Intermediate/Advanced: async jobs, retries | [AWS Builders’ Library](https://aws.amazon.com/builders-library/) |
| Search and autocomplete | Advanced: indexes, prefixes, freshness | [Elastic Blog](https://www.elastic.co/blog) |
| Content delivery platform | Advanced: edge caching, origin protection | [Cloudflare Blog](https://blog.cloudflare.com/) |

### Infrastructure and data platforms

| System | Level / concepts | References |
|---|---|---|
| Distributed cache | Senior: sharding, eviction, consistency | [System Design Primer](https://github.com/donnemartin/system-design-primer) |
| Key-value store | Senior: replication, partitioning, quorum | [System Design Primer — key-value store](https://github.com/donnemartin/system-design-primer#design-a-key-value-store-for-a-search-engine) · [Designing Data-Intensive Applications](https://dataintensive.net/) |
| Metrics and logging platform | Senior: ingestion, cardinality, retention | [Google SRE Books](https://sre.google/books/) |
| Event ingestion and analytics | Senior: partitioning, backpressure | [Confluent Developer](https://developer.confluent.io/) |
| Distributed task execution | Senior: scheduling, leases, retries | [Temporal documentation](https://docs.temporal.io/) |
| Feature-flag platform | Advanced: evaluation, rollout, consistency | [LaunchDarkly Blog](https://launchdarkly.com/blog/) |
| API gateway/load balancer | Senior: routing, quotas, resilience | [Envoy documentation](https://www.envoyproxy.io/docs) |

### Location and real-time systems

| System | Level / concepts | References |
|---|---|---|
| Ride-booking platform | Senior: matching, geospatial indexes, surge | [Uber Engineering](https://www.uber.com/blog/engineering/) |
| Location tracking | Advanced: high-write streams, privacy | [Uber Engineering](https://www.uber.com/blog/engineering/) |
| Geospatial search | Advanced: spatial indexes, radius queries | [PostGIS documentation](https://postgis.net/documentation/) |
| Live leaderboard | Advanced: ranking, hot keys, approximate counts | [Redis documentation](https://redis.io/docs/latest/) |

Additional problems worth practising: distributed lock service; webhook delivery platform; configuration service; multi-tenant SaaS control plane; fraud detection pipeline; CDC platform; workflow engine; backup and restore platform.

Reference links are starting points, not claims that every linked page contains a complete solution for the exact prompt. Prefer a specific article when available and verify access before relying on it.

## 20. Cross-references and authoritative reading

The repository's focused guides provide implementation depth; this handbook retains the architectural reasoning.

- [Java Interview Handbook](../Java/Java-Interview-QA.md) — concurrency, JVM, collections, runtime behavior.
- [Design Patterns & LLD](../Design-Patterns-LLD/Design-Patterns-LLD-Interview-QA.md) — object collaboration and component design.
- [Spring Framework + Spring Boot](../Spring/Spring-Interview-QA.md) — application framework and transaction implementation.
- [Microservices & REST APIs](../Microservices-REST-APIs/Microservices-REST-APIs-Interview-QA.md) — service boundaries and API contracts.
- [Distributed Systems](../Distributed-Systems/Distributed-Systems-Interview-QA.md) — consistency, coordination, distributed failure.
- [Hibernate + JPA](../Hibernate-JPA/Hibernate-JPA-Interview-QA.md) — ORM and transaction behavior.
- [SQL & Database Engineering](../SQL-Database/SQL-Database-Interview-QA.md) — query plans, indexes, transaction isolation.
- [Redis & Caching](../Redis-Caching/Redis-Caching-Interview-QA.md) — cache behavior and Redis/database consistency.
- [Apache Kafka](../Kafka/KAFKA_INTERVIEW_GUIDE.md) — broker mechanics, partitioning, delivery semantics.

### Authoritative and practical references

- [System Design Primer](https://github.com/donnemartin/system-design-primer) — concepts and practice problems; its [interview approach](https://github.com/donnemartin/system-design-primer#how-to-approach-a-system-design-interview-question) is a useful companion to this handbook.
- [Google SRE Books](https://sre.google/books/) — reliability, monitoring, incident response.
- [AWS Builders’ Library](https://aws.amazon.com/builders-library/) — operational lessons and distributed design; see also [timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).
- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework) — reliability, security, cost.
- [Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/) — architecture patterns and reference designs.
- [Apache Kafka documentation](https://kafka.apache.org/documentation/) — broker/client guarantees.
- [PostgreSQL documentation](https://www.postgresql.org/docs/) — transaction, isolation, replication behavior.
- [Redis documentation](https://redis.io/docs/latest/) — data structures, persistence, replication.
- [Envoy documentation](https://www.envoyproxy.io/docs) — proxy and traffic management.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — foundational data-system architecture.

### Revision checklist

Before finishing a practice design, ask:
- What are the key functional requirements and business invariant?
- Which quality attributes are targets, and how are they measured?
- What are average/peak rates, data growth, and payload sizes?
- Which component owns each authoritative data item?
- What happens when a request times out after success?
- Which reads may be stale, and which need read-your-writes?
- What happens when cache, database, broker, or provider fails?
- How are retries bounded and duplicate effects prevented?
- What breaks first at 10× traffic?
- What are the RTO/RPO and data-loss assumptions?
- How are tenant isolation, abuse prevention, and sensitive data handled?
- What are the main operating costs and simplest viable alternative?
- Which metrics and traces prove the design is working?

---

*This is a living study resource. Technical guarantees vary by database, broker, service, configuration, and failure mode. Verify product-specific claims against official documentation and test behavior that matters to the design.*
