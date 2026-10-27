# Kafka and Event-Driven Architecture

> Numbered learning checklist. Use the existing handbook in this repository for detailed explanations and interview Q&A.

### 27. Messaging and Apache Kafka
- [ ] Messaging Models
  - [ ] Queue vs Publish/Subscribe
  - [ ] Event, Command, and Message
  - [ ] Event Notification vs Event-Carried State Transfer
- [ ] Kafka Architecture
  - [ ] Brokers, Topics, Partitions, and Replicas
  - [ ] Leaders, Followers, and ISR
  - [ ] Controller / KRaft Fundamentals
  - [ ] Metadata and Cluster Operations
- [ ] Partitioning and Key Design
- [ ] Producer Internals
  - [ ] Batching and Compression
  - [ ] Acknowledgements and Retries
  - [ ] Idempotent Producer
  - [ ] Transactions
- [ ] Consumer Internals
  - [ ] Consumer Groups
  - [ ] Partition Assignment
  - [ ] Rebalance and Cooperative Rebalance
  - [ ] Offsets and Commits
  - [ ] Poll Loop and Processing Time
- [ ] Delivery Guarantees and Duplicate Processing
- [ ] Ordering Guarantees
- [ ] Consumer Lag and Throughput
- [ ] Retention, Compaction, and Segment Files
- [ ] Replication and Failure Recovery
- [ ] Quotas and Backpressure
- [ ] Schema Registry
- [ ] Avro, Protobuf, and JSON Schema
- [ ] Schema Compatibility
- [ ] Dead-Letter Topics and Retry Topics
- [ ] Poison Messages and Retry Loops
- [ ] Replay and Offset Reset
- [ ] Transactional Outbox with Kafka
- [ ] Kafka Connect and CDC
- [ ] Kafka Streams Fundamentals
- [ ] Kafka Security: TLS, SASL, ACLs
- [ ] Kafka Metrics and Troubleshooting
- [ ] Spring Kafka
  - [ ] Producers and Templates
  - [ ] Listener Containers
  - [ ] Error Handlers and Retry Topics
  - [ ] Transactions and Offset Acknowledgements
  - [ ] Testing Kafka Consumers/Producers
