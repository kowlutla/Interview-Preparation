# System Design HLD

> Numbered learning checklist. Use the existing handbook in this repository for detailed explanations and interview Q&A.

### 31. System Design Interview Method
- [ ] Problem Clarification and Scope
- [ ] Functional Requirements
- [ ] Non-Functional Requirements
- [ ] User Journeys and Critical Business Invariants
- [ ] API and Data Contract Definition
- [ ] Capacity Estimation
  - [ ] DAU/MAU and Request Rates
  - [ ] Average and Peak RPS
  - [ ] Read/Write Ratio
  - [ ] Storage Growth and Retention
  - [ ] Bandwidth and Network Egress
  - [ ] Replication and Redundancy Overhead
  - [ ] Queue Backlog and Drain Time
  - [ ] Connection and Thread Capacity
- [ ] Architecture Diagram and Data Flow
- [ ] Bottleneck Identification
- [ ] Trade-Off Analysis
- [ ] Failure Modes and Recovery
- [ ] Security, Observability, and Cost
- [ ] Architecture Evolution and Migration
- [ ] Presenting a 45–60 Minute Design

### 32. System Design Building Blocks
- [ ] DNS, CDN, and Edge Caching
- [ ] Reverse Proxy and Load Balancing
- [ ] L4 vs L7 Load Balancing
- [ ] API Gateway and WAF
- [ ] Stateless and Stateful Services
- [ ] Horizontal and Vertical Scaling
- [ ] Autoscaling and Capacity Headroom
- [ ] Service Discovery
- [ ] Synchronous and Asynchronous Calls
- [ ] REST, gRPC, WebSockets, and SSE
- [ ] Database Selection
- [ ] Replication, Partitioning, and Sharding
- [ ] Caching and Cache Invalidation
- [ ] Queues, Streams, and Event Logs
- [ ] Search Indexes and Read Models
- [ ] Object Storage and Content Delivery
- [ ] Batch, Stream, and Change-Data-Capture Pipelines
- [ ] Service Mesh Concepts
- [ ] Secrets, Identity, and Policy Enforcement

### 33. Data Architecture and Distributed Storage
- [ ] Logical and Physical Data Models
- [ ] Read/Write Patterns and Access Locality
- [ ] ACID vs BASE
- [ ] Relational vs NoSQL Trade-Offs
- [ ] Index Selection and Query Shapes
- [ ] Read Replicas and Replication Lag
- [ ] Shard Key Selection
- [ ] Consistent Hashing
- [ ] Hot Shards and Resharding
- [ ] Denormalization and Materialized Views
- [ ] CQRS
- [ ] Event Sourcing
- [ ] Data Ownership Across Services
- [ ] Data Retention and Archival
- [ ] Online Schema Changes
- [ ] Backup, Restore, and Point-in-Time Recovery

### 34. Scalability, Consistency, and Resilience in HLD
- [ ] Load Distribution and Hotspot Management
- [ ] Horizontal Scale-Out
- [ ] Rate Limiting and Quotas
- [ ] Backpressure and Queue-Based Load Leveling
- [ ] Caching Strategies and Invalidation
- [ ] Strong and Eventual Consistency
- [ ] Quorum Reads and Writes
- [ ] Leader/Follower Replication
- [ ] Idempotency and Deduplication
- [ ] Exactly-Once Claims and Their Boundaries
- [ ] Distributed Transactions and Sagas
- [ ] Transactional Outbox and CDC
- [ ] Retry Budgets and Deadline Propagation
- [ ] Circuit Breakers and Bulkheads
- [ ] Graceful Degradation
- [ ] Disaster Recovery and Regional Failover

### 35. HLD Case Studies
- [ ] URL Shortener
- [ ] Pastebin
- [ ] Rate Limiter
- [ ] API Gateway
- [ ] Notification Service
- [ ] Email Delivery Platform
- [ ] Chat / Messaging System
- [ ] News Feed / Activity Feed
- [ ] Social Graph / Follow System
- [ ] Search Autocomplete
- [ ] Search Platform
- [ ] File Storage and Sharing
- [ ] Photo / Video Upload Platform
- [ ] Video Streaming Platform
- [ ] Ride-Hailing Platform
- [ ] Food Delivery Platform
- [ ] E-Commerce Catalog and Inventory
- [ ] Shopping Cart
- [ ] Order Management
- [ ] Payment Processing
- [ ] Ticket Booking / Seat Reservation
- [ ] Hotel Booking
- [ ] Banking Ledger / Money Transfer
- [ ] Job Scheduler
- [ ] Distributed Task Queue
- [ ] Web Crawler
- [ ] Log Aggregation Platform
- [ ] Metrics and Monitoring System
- [ ] Audit Logging Platform
- [ ] Feature Flag Service
- [ ] Distributed Cache
- [ ] Leaderboard
- [ ] Recommendation Platform
- [ ] Multi-Tenant SaaS Platform
- [ ] Data Ingestion Platform
- [ ] Event Streaming Platform
- [ ] Analytics Dashboard
- [ ] Multi-Region SaaS Application
- [ ] Cloud Data Governance / Data Marketplace
