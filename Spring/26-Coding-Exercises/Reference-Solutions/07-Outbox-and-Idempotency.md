# Reference Solution — Transactional Outbox and Idempotent Consumer

This is a design-level reference because a complete implementation depends on the chosen database, Spring Kafka version, schema, and relay strategy.

## Data model

Create an outbox table with a stable event ID, aggregate/business key, event type, payload, creation time, publication state, attempt count, and last error metadata. Add indexes for pending-record scans and a uniqueness constraint for event identity.

## Write path

Within one local database transaction:

1. Validate and persist the business state change.
2. Insert an outbox row representing the event.
3. Commit both together.

Do not publish directly to the broker as an uncoordinated second write before or after the database commit.

## Relay path

1. Select pending rows using a concurrency-safe strategy appropriate to the database.
2. Publish each event with a stable message key/event ID.
3. Mark successful publication only after the broker operation reports success.
4. Retry transient failures with bounded backoff and alert on oldest pending age.
5. Expect duplicates if the relay crashes after publish but before marking the row complete.

## Consumer idempotency

In the consumer's database transaction, record the event ID under a uniqueness constraint and apply the business side effect. If the ID already exists, treat the delivery as a duplicate according to the domain contract. If the side effect is an external API call, use that provider's idempotency support or a separate reconciliation design; a database deduplication row alone cannot atomically control an unrelated remote system.

## Failure matrix

| Failure point | Expected recovery |
|---|---|
| Before local transaction commit | Neither business state nor outbox event should commit |
| After commit, before relay reads row | Relay discovers pending row later |
| Publish fails before broker accepts event | Retry according to policy |
| Broker accepts event, relay crashes before marking sent | Duplicate publication is possible; consumer must be idempotent |
| Consumer commits business effect but acknowledgment is lost | Redelivery is possible; deduplication prevents repeated effect |
| Poison event repeatedly fails | Isolate and alert; provide a controlled replay process |

## Tests

- Business row and outbox row commit or roll back together.
- Relay retries transient broker failures.
- Simulated crash after publish can create a duplicate but not a duplicate business effect.
- Concurrent duplicate deliveries do not bypass the uniqueness constraint.
- Metrics expose pending count, oldest pending age, attempts, and dead-letter/replay outcomes.
