# System Design HLD

## Interview Method

- [ ] Requirements clarification, functional scope, constraints, and assumptions
- [ ] Non-functional requirements: availability, latency, durability, consistency, security, and cost
- [ ] Back-of-the-envelope estimation: traffic, storage, bandwidth, and capacity
- [ ] API design, data model, core entities, and important system invariants
- [ ] Architecture diagrams, request flows, data flows, and clear trade-off communication

## System Design Building Blocks

- [ ] DNS, load balancing, reverse proxies, gateways, and traffic routing
- [ ] Stateless services, horizontal scaling, autoscaling, and backpressure
- [ ] Relational vs NoSQL storage, data access patterns, and schema choices
- [ ] Indexing, replication, partitioning, sharding, and rebalancing
- [ ] Caching layers, invalidation, hot keys, and cache consistency
- [ ] Messaging, event streams, queues, fan-out, retries, and dead-letter handling
- [ ] CDN, object storage, blob delivery, and large-file flows
- [ ] Authentication, authorization, tenant isolation, encryption, and data privacy
- [ ] Resilience patterns, multi-AZ/multi-region, failover, and disaster recovery
- [ ] Observability, SLI/SLO, alerting, capacity planning, and cost trade-offs

## Distributed Data and Reliability

- [ ] Consistency models, CAP/PACELC, quorum choices, and read/write semantics
- [ ] Idempotency, deduplication, outbox, saga, and asynchronous workflows
- [ ] Rate limiting, fairness, abuse controls, and load shedding
- [ ] Data lifecycle, retention, archival, backup, restore, and migration strategy

## Case Studies

- [ ] URL shortener and redirect service
- [ ] Rate limiter and API quota service
- [ ] News feed, timeline, and notification platform
- [ ] Chat/messaging and real-time presence
- [ ] File/object storage and content delivery
- [ ] Search/autocomplete and distributed cache
- [ ] Ride matching, booking, or order-processing system
- [ ] Multi-tenant SaaS / data-governance or data-marketplace platform
- [ ] Multi-region architecture and disaster-recovery design
