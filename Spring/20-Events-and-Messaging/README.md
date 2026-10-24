# Spring Events and Messaging

## Q&A

### Q1. Are Spring application events synchronous by default?

The standard event multicaster invokes listeners synchronously unless configured otherwise. A slow listener can therefore increase the publisher's latency. Asynchronous event delivery changes error handling and transaction/context assumptions and is not a durable message queue.

### Q2. What does @TransactionalEventListener provide?

It binds listener execution to a transaction phase, commonly after commit. If no transaction is active, behavior depends on the configured fallback option. It helps coordinate in-process reactions with transaction outcomes but does not guarantee durable delivery if the process crashes after commit.

### Q3. Why is an outbox pattern useful?

The application writes business state and an event record in the same local database transaction. A separate publisher relays the outbox record to the broker and marks progress. This avoids the dual-write gap, but requires handling retries, duplicate publication, cleanup, ordering, and operational monitoring.

### Q4. Why must Kafka consumers be idempotent?

At-least-once processing and retries can deliver the same logical event more than once. Use a durable deduplication key or design side effects to be idempotent. Acknowledgment timing and consumer transactions affect delivery behavior, but they do not remove the need to reason about external side effects.

### Q5. How should listener retries and dead-letter handling be designed?

Classify retryable and non-retryable failures, bound attempts, use backoff, and send poison messages to a monitored dead-letter destination when appropriate. Define replay procedures and preserve enough metadata to diagnose failures without exposing secrets. Ensure retry behavior does not block partitions indefinitely.

### Q6. What is the difference between an application event and a broker message?

An application event is an in-process communication mechanism within an application context. A broker message crosses a process boundary and can provide durable retention and independent consumers, depending on configuration. Use local events for local decoupling, not as a substitute for cross-service reliability.

## Official references

- [Spring application events](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html)
- [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/)
