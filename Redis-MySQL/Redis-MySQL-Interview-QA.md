
# Redis & MySQL Interview Q&A Handbook

> **Audience:** Java backend engineers with 4–6 years of experience targeting senior backend and product-company interviews.  
> **Version context:** SQL examples target MySQL 8.0/8.4. Redis behavior and available commands can vary by Redis version and managed service. Spring examples are illustrative; verify APIs and configuration against the application's Spring Boot and Spring Data Redis versions.

This is a backend developer's guide, not a Redis administrator or MySQL DBA manual. In the typical integration described here, MySQL owns durable business records and Redis is an acceleration or coordination layer with explicitly defined failure behavior.

## Table of contents

- [Top Questions to Master First](#top-questions-to-master-first)
- [Part I — Redis](#part-i--redis)
  - [Fundamentals and execution model](#fundamentals-and-execution-model)
  - [Data structures](#data-structures)
  - [Expiration, persistence and availability](#expiration-persistence-and-availability)
  - [Caching and consistency](#caching-and-consistency)
  - [Coordination and rate limiting](#coordination-and-rate-limiting)
  - [Performance and troubleshooting](#performance-and-troubleshooting)
- [Part II — MySQL](#part-ii--mysql)
  - [Data modeling and SQL](#data-modeling-and-sql)
  - [Indexes and query optimization](#indexes-and-query-optimization)
  - [Transactions and concurrency](#transactions-and-concurrency)
  - [InnoDB, replication and reliability](#innodb-replication-and-reliability)
  - [Production troubleshooting](#production-troubleshooting)
- [Part III — Redis + MySQL with Java/Spring](#part-iii--redis--mysql-with-javaspring)
- [Part IV — Real-world interview scenarios](#part-iv--real-world-interview-scenarios)
- [Redis command reference](#redis-command-reference)
- [MySQL command reference](#mysql-command-reference)
- [Official references](#official-references)

## Top Questions to Master First

1. [How does cache-aside work, and what consistency gaps remain?](#q19-how-does-cache-aside-work-and-what-consistency-gaps-remain)
2. [Why is Redis described as single-threaded?](#q2-why-is-redis-described-as-single-threaded)
3. [How do expiration and eviction differ?](#q12-what-is-the-difference-between-expiration-and-eviction)
4. [How do composite indexes work?](#q38-how-do-composite-indexes-work)
5. [How do you investigate a slow MySQL query?](#q43-how-do-you-investigate-a-slow-mysql-query)
6. [What does InnoDB REPEATABLE READ mean?](#q48-what-does-innodb-repeatable-read-mean)
7. [How do you diagnose deadlocks?](#q51-how-do-you-diagnose-and-reduce-deadlocks)
8. [Why can database update plus cache invalidation still leave stale data?](#q61-why-can-database-update-plus-cache-invalidation-still-leave-stale-data)
9. [How do you protect MySQL during a Redis outage?](#q68-redis-is-down-and-mysql-load-surges-how-do-you-respond)
10. [How do you enforce uniqueness under concurrent requests?](#q78-two-concurrent-requests-create-duplicates-how-do-you-fix-it)

## How to answer in an interview

Start with a direct answer, explain how the mechanism works, then describe a trade-off and a failure mode. For production scenarios, state what you would measure and inspect before proposing a change. The code is illustrative rather than a drop-in production library; production systems also need error classification, bounded timeouts, security, metrics, and tests.

# Part I — Redis

## Fundamentals and execution model

### Q1. What is Redis, and when should a backend service use it?

Redis is an in-memory data-structure server. A client sends commands over a network and Redis operates on values such as strings, hashes, sets, sorted sets, lists, and streams. It is commonly used for cache entries, counters, rate limiting, leaderboards, short-lived state, and lightweight event processing.

Use Redis when low-latency data-structure operations materially help the workload and the team can define expiration, memory limits, consistency, and outage behavior. Do not introduce it just because it is fast. A relational database is usually a better owner for durable transactional records, foreign-key relationships, and complex business invariants. Redis can persist data, but persistence and failover settings must be selected deliberately; “stored in Redis” does not automatically mean “durable with zero loss.”

### Q2. Why is Redis described as single-threaded?

Ordinary Redis commands are executed through a serialized command-processing path, so one command's effects are atomic relative to other commands. This helps make operations such as INCR safe from client-side read-modify-write races. It does not mean the entire Redis process never uses threads: I/O processing and background tasks can use threads depending on version and configuration.

The important consequence for application developers is that an expensive command, oversized response, or long-running script can delay unrelated clients. Avoid unbounded scans, huge collection operations, and scripts whose work grows with user-controlled input. A single command being atomic also does not make a sequence of separate client commands atomic.

### Q3. How do Redis keys, values, and TTLs work?

A key identifies a value, whose type determines the available operations. A TTL (time to live) controls how long a key is intended to remain available.

~~~text
SET product:42 '{"id":42,"name":"Keyboard"}' EX 300
GET product:42
TTL product:42
~~~

EX sets expiry in seconds; PX uses milliseconds. TTL distinguishes a missing key from a key with no expiration through different negative return values. Expiration is not a guarantee that the key is physically removed at the exact millisecond the timer reaches zero; Redis uses passive checks on access and active expiration work. Application code must treat a missing cache key as a normal cache miss.

Use namespaced keys such as product:v2:42. Do not place secrets or personal data in key names because diagnostic tools and logs may expose them.

### Q4. What determines Redis request latency?

End-to-end latency includes application scheduling, pool wait, network round trips, Redis command execution, serialization, and any fallback query. Redis can execute a command quickly while the Java request is slow because a connection is unavailable, the payload is large, or the database fallback is expensive.

Measure client-side latency and server-side latency separately. Track p95/p99, not only averages. Keep timeouts finite, avoid chatty command sequences, and keep Redis near the service when possible. A cache also adds a failure path; justify it with observed database cost or latency.

### Q5. Are Redis commands atomic? Are MULTI/EXEC transactions equivalent to SQL transactions?

A single command such as INCR is atomic relative to other commands. A client sequence such as GET, calculate in Java, then SET is not atomic: two clients can read the same value and overwrite each other's update. Use an atomic command, optimistic check with WATCH, or a bounded Lua script when the invariant requires it.

MULTI queues commands and EXEC runs the queued commands without other clients interleaving in the middle of that execution. Redis transactions do not provide SQL-style rollback of already executed commands when a command errors. A read-compute-write decision still needs correct concurrency handling; simply wrapping arbitrary commands in MULTI is not a universal fix.

## Data structures

### Q6. When should you use Strings and counters?

Strings store byte sequences and are useful for serialized cache values, flags, tokens, and counters. GET, SET, MGET, INCR, and INCRBY are common operations. Atomic increment avoids a client-side read-modify-write race.

~~~text
SET visits:article:42 0 EX 3600 NX
INCR visits:article:42
~~~

This is only a basic counter example. A real rate limiter must define the window, first-hit expiry, concurrency behavior, and boundary semantics. Strings are less attractive when one field of a large object changes frequently and rewriting the full serialized object is wasteful.

### Q7. When are Hashes useful compared with serialized Strings?

A Hash stores field/value pairs under one key, making individual field reads and updates straightforward. For example, HGET user:17 status or HSET user:17 status active. A serialized String is often simpler for an immutable DTO that is fetched and replaced as a whole.

A Redis Hash is not a relational row: it does not enforce schema, foreign keys, or multi-key relational constraints. Choose based on update pattern, payload size, serialization compatibility, and access needs. Large hashes still require memory and cardinality planning.

### Q8. How do Lists, Sets, and Sorted Sets differ?

A List is an ordered sequence that permits duplicates and is useful for simple queues or recent-item lists. A Set stores unique members and is useful for membership tests and tags. A Sorted Set stores unique members ordered by numeric score, making it useful for leaderboards, ranking, and score-range queries.

Sets do not provide ranking; Sorted Sets are not general-purpose relational indexes; Lists do not provide the same consumer-group workflow as Streams. Consider cardinality, retention, tie-breaking, and whether the application needs reliable work distribution or just a simple collection.

### Q9. When would you choose a Stream over a List?

Redis Streams act as append-oriented logs with entry IDs, range reads, consumer groups, acknowledgement, and pending-entry tracking. They are useful when multiple workers share work and operators need to inspect or reclaim unacknowledged entries after consumer failure.

Lists can support simple push/pop queues but do not provide the same event-log and consumer-group workflow. Streams still require retention policy, retry handling, poison-message strategy, idempotent processing, and recovery procedures. Acknowledging a Stream entry and committing a MySQL transaction are not one atomic operation, so duplicate processing remains possible.

### Q10. What are Bitmaps, HyperLogLogs, and geospatial structures for?

Bitmaps efficiently store boolean flags by bit position and can answer questions such as whether a user ID was active on a given day. They work best when the identifier space is bounded or reasonably dense. HyperLogLog estimates distinct cardinality with low memory but cannot provide exact counts or enumerate members. Geospatial commands support coordinate storage and nearby/radius lookups, but are not a full GIS engine.

Use approximate structures only when approximate answers are acceptable. They are inappropriate for billing, settlement, or decisions that require exact membership.

### Q11. Redis versus MySQL: how do you decide?

MySQL is a natural owner for durable relational records, constraints, joins, and multi-row transactions. Redis is useful for low-latency access to selected data structures, caching, counters, and ephemeral coordination. Many systems use both: MySQL remains canonical while Redis accelerates selected reads.

The two-system design adds consistency and operational complexity. Cache stable, frequently read data whose staleness can be bounded. Be cautious with sensitive, highly personalized, rapidly changing, or rarely reused data.

## Expiration, persistence and availability

### Q12. What is the difference between expiration and eviction?

Expiration is driven by a key's TTL. Eviction removes eligible keys to satisfy a memory policy when Redis approaches its memory limit. A key may be evicted before its TTL. Under noeviction, commands that need more memory can fail rather than removing keys.

A cache must be reconstructible after misses. If Redis stores irreplaceable state, eviction, persistence, backup, and failover require a more deliberate design. Do not assume a TTL guarantees that the key remains present until the deadline.

### Q13. Explain common eviction policies.

- allkeys-lru: approximate least-recently-used candidates from all keys.
- allkeys-lfu: approximate least-frequently-used candidates from all keys.
- volatile-lru / volatile-lfu: apply LRU/LFU only to keys with expiry set.
- volatile-ttl: prefer eligible keys with the shortest remaining TTL.
- allkeys-random / volatile-random: random eligible-key eviction.
- noeviction: reject writes that need additional memory rather than evicting keys.

LRU favors recent access; LFU can favor a stable hot set. Neither is universally best. With volatile policies, if no eligible expiring keys exist, writes may fail similarly to noeviction. Confirm the policy in the deployed service; managed Redis products may restrict configuration.

### Q14. RDB versus AOF persistence: what are the trade-offs?

RDB stores point-in-time snapshots. It can be compact and convenient for recovery, but changes since the last snapshot may be lost after a crash. AOF records write operations for replay; durability depends on the fsync policy, and the trade-off includes write and storage overhead. Redis can combine approaches or disable persistence for a disposable cache.

Persistence is not the same as a backup strategy. Backups need retention, off-host storage, access controls, and tested restoration. Replication is not a backup either: accidental deletion or corruption may propagate to replicas.

### Q15. How do replication, Sentinel, and Redis Cluster differ?

Replication copies data from a primary to replicas; it is generally asynchronous, so replicas may lag and recent writes may not yet be visible. Sentinel monitors a non-clustered primary/replica setup and coordinates failure detection and promotion. Redis Cluster shards keys across hash slots and provides per-shard replication/failover.

Sentinel is not sharding, and Cluster is not simply Sentinel with more nodes. Multi-key operations in Cluster generally require involved keys to share a hash slot; hash tags such as customer:{17}:profile can colocate related keys but can also create hot spots. Failover does not guarantee zero data loss. Applications must handle topology changes, redirects, transient errors, and retry ambiguity.

### Q16. Why might a key disappear earlier than expected?

Possible causes include TTL expiry, eviction, application deletion/replacement, failover or restore behavior, a flush operation, key-prefix changes, or querying a different endpoint. Verify the endpoint, key name, TTL, memory policy, and application write paths first.

Use TTL/PTTL, INFO memory, INFO keyspace, and carefully scoped application logs. Avoid broad KEYS * scans on production. A missing cache key does not mean the canonical MySQL row was deleted.

## Caching and consistency

### Q17. Explain cache-aside (lazy loading).

The application checks Redis. On a miss it reads MySQL, populates Redis with a bounded TTL, and returns the result. On writes it updates MySQL and invalidates the cache key.

~~~mermaid
sequenceDiagram
    participant A as Application
    participant R as Redis
    participant D as MySQL
    A->>R: GET product:42
    alt cache hit
      R-->>A: cached product
    else cache miss
      R-->>A: nil
      A->>D: SELECT product
      D-->>A: product
      A->>R: SET product:42 value EX 300
    end
    A-->>A: return product
~~~

Cache-aside is simple and widely used, but the first read after a miss pays the database cost. Concurrent fills can race with writes, so invalidation order and stale-fill windows matter. TTL bounds some stale periods but does not guarantee immediate consistency.

### Q18. What are read-through, write-through, and write-behind patterns?

With read-through, a cache abstraction loads misses from the backing store. With write-through, writes pass through a layer that synchronously updates the backing store according to its contract. With write-behind/write-back, the cache acknowledges before the database update is durably complete.

Write-behind can reduce latency and batch writes, but creates a durability risk: process failure or failed flushes can lose acknowledged changes. It needs durable buffering, replay, ordering, idempotency, and reconciliation. These names describe patterns, not guarantees; inspect the actual failure semantics.

### Q19. How does cache-aside work, and what consistency gaps remain?

A read checks Redis, falls back to MySQL on a miss, then caches the result. A write commits MySQL and invalidates Redis. The database remains authoritative.

The difficult part is the race: reader A misses and reads old value V1; writer B commits V2 and deletes the key; reader A then writes V1 into Redis after the deletion. The stale value survives until TTL or another invalidation. Other failures include Redis invalidation failing after database commit.

Mitigations include finite TTLs, versioned values, per-key refresh coordination, durable invalidation events, and bypassing the cache for read-your-writes flows. No single simple cache-aside sequence proves immediate consistency across MySQL and Redis.

### Q20. What are cache stampede, penetration, avalanche, and hot-key problems?

- Stampede: many requests miss the same key and load MySQL simultaneously. Use single-flight/request coalescing, a refresh lease where appropriate, early refresh, or stale-while-revalidate.
- Penetration: repeated requests for absent records hit MySQL. Briefly negative-cache confirmed misses, but do not cache transient database failures as not-found.
- Avalanche: many keys expire together or Redis becomes unavailable, causing a backend surge. Add TTL jitter, stagger warming, rate limits, and bounded fallback concurrency.
- Hot key: disproportionate traffic to one key overloads a shard. Consider local caching for safe data, splitting aggregates, or redesigning the access pattern.

A lock can reduce duplicate work but introduces coordination and lease-expiry failure modes. Negative caching can briefly hide a newly created record, so choose its TTL carefully.

### Q21. Why is cache invalidation difficult?

MySQL and Redis do not share a normal atomic transaction. The database commit can succeed while cache invalidation fails; invalidation can also occur before a database transaction later rolls back. A common default is commit MySQL first, then evict Redis, with TTL as a backstop.

For stronger eventual delivery, write an outbox row in the same MySQL transaction and publish it asynchronously. The consumer invalidates keys idempotently and retries failures. This closes the gap where the database commits but no durable event was recorded, though event delay still permits a stale window. If immediate read-your-writes is required, return the committed database result or bypass cache on the follow-up read.

### Q22. How should you use negative caching and cache warming?

Negative caching stores a short-lived “not found” result to protect MySQL from repeated invalid IDs. It must distinguish real absence from timeouts, authorization failures, and database errors. Keep negative TTLs short if records may be created soon after a miss.

Cache warming preloads hot keys after startup or deployment. It can reduce cold-start latency but overload MySQL if uncontrolled. Warm only measured hot data with bounded concurrency; service startup should not depend on complete warming unless the business explicitly requires it.

## Coordination and rate limiting

### Q23. How would you implement a distributed rate limiter?

A fixed-window counter is simple but can allow bursts at window boundaries. Sliding-window counters/logs smooth boundaries at extra cost. A token bucket supports a refill rate and burst capacity. Concurrent updates must be atomic, typically using a suitable Redis command or a bounded Lua script.

Choose the key identity carefully, set expiry and cardinality limits, and decide whether to fail open or closed if Redis is unavailable. For important limits, measure rejected requests and test the outage policy.

### Q24. Are Redis distributed locks safe?

A basic lease can be acquired with SET lock:resource random-token NX PX 10000. Release must compare the stored token to the owner's token before deleting; plain DEL is unsafe because the lease may expire and another owner may acquire the lock.

Token-safe release does not prevent stale work. The original process may pause beyond the lease, resume after another owner acquired it, and still modify the protected resource. For correctness-critical operations, use fencing tokens validated by the resource, database constraints/transactions, or a coordination system with guarantees that fit the requirement. Treat a Redis lease as best-effort coordination unless the full design proves otherwise.

### Q25. What are idempotency keys, and should Redis enforce uniqueness?

An idempotency key identifies retries of one logical operation, such as creating an order. A Redis key with TTL can reduce duplicate work, but eviction, expiry, and failover can allow a second execution. For business uniqueness, use a durable MySQL unique constraint and store operation status/result transactionally where feasible.

A unique key such as (client_id, idempotency_key) protects concurrent requests that both pass an application-level “does it exist?” check. Define how to handle the same key with a different request body and how in-progress operations are represented.

## Performance and troubleshooting

### Q26. Why avoid KEYS in production?

KEYS pattern can scan the keyspace as one operation and delay other work. SCAN iterates incrementally and is safer for operational inspection, but is not a snapshot: callers must tolerate duplicates, empty batches before cursor zero, and keys changing during iteration. Use bounded iterations and do not turn global scanning into a frequent application lookup.

### Q27. What Redis commands or patterns cause latency spikes?

Large collection operations, unbounded scripts, huge responses, broad scans, and synchronous cleanup of large values can cause latency spikes. Big-O matters, but payload size and constants matter too. Verify complexity in the command reference.

Pipelining reduces network round trips but does not make a batch transactional and can increase memory pressure if batches are oversized. Use bounded batches and track tail latency.

### Q28. How do you investigate Redis memory growth?

Check INFO memory, INFO keyspace, INFO stats, eviction counters, key count, and maxmemory policy. Compare dataset memory with process memory; allocator fragmentation and buffers can make them differ. Inspect representative keys with MEMORY USAGE and use sampling carefully.

Likely causes include missing TTLs, unbounded key cardinality, duplicated namespaces, oversized serialized objects, retained Stream entries, or a changed workload. Do not delete data blindly; identify whether it is disposable first.

### Q29. How should a Java service configure Redis connections and timeouts?

Use the client and connection factory supported by the application's Spring Data Redis version and configure finite connect/command timeouts. A small pool can create wait time; an excessively large pool can overload the service or Redis. Size based on measured concurrency, command latency, deployment replica count, and Redis capacity.

Bound retries and avoid retry storms. A timeout does not prove a command failed to execute; retrying a non-idempotent operation can apply it twice.

### Q30. How should you handle large keys and payloads?

Large values increase network transfer, serialization cost, memory use, replication traffic, and latency. Cache only fields needed by common requests, paginate collections, and avoid serializing an entire aggregate to update one field. Set explicit cardinality and retention limits.

Measure key size, access frequency, command latency, and memory impact in the real workload. Splitting a hot aggregate can help, but complicates reads and atomic updates.

# Part II — MySQL

## Data modeling and SQL

### Q31. Explain primary keys, foreign keys, unique constraints, and check constraints.

A primary key uniquely identifies a row and cannot be NULL. A foreign key enforces a relationship to a referenced key when configured and enforced. A unique constraint prevents duplicate key combinations, with MySQL-specific NULL behavior for unique indexes. A check constraint validates a row predicate in supported versions; confirm the target version and enforcement behavior.

Constraints protect correctness when requests race or retries occur. Application validation improves error messages but cannot replace a database uniqueness constraint for a hard invariant.

### Q32. How do normalization and denormalization differ?

Normalization stores each fact in an appropriate place to reduce update anomalies. For example, customer data generally belongs in a customer table, while an order may intentionally snapshot a shipping address as historical transaction data. Denormalization can speed common reads or preserve historical facts, but introduces synchronization and write-amplification costs.

Normalize the source model first, then denormalize based on measured query patterns and explicit consistency rules. Summary tables, materialized read models, and caches all require a repair or refresh strategy.

### Q33. Explain WHERE, GROUP BY, HAVING, and aggregates.

WHERE filters rows before grouping. GROUP BY forms groups; aggregate functions compute values per group; HAVING filters groups after aggregation.

~~~sql
SELECT customer_id, COUNT(*) AS order_count
FROM orders
WHERE created_at >= '2026-01-01'
GROUP BY customer_id
HAVING COUNT(*) >= 3
ORDER BY order_count DESC;
~~~

This returns customers with at least three orders in the selected period. A one-to-many join before aggregation can multiply rows and inflate counts. Aggregate at the intended grain or use COUNT(DISTINCT ...) only when that matches the business meaning.

### Q34. How do INNER, LEFT, RIGHT, and self joins differ?

INNER JOIN returns matching row pairs. LEFT JOIN retains every left-side row and produces NULLs for unmatched right-side columns. RIGHT JOIN is the corresponding right-preserving form and can usually be rewritten by swapping tables. A self-join relates a table to itself, such as employees to managers.

For a LEFT JOIN, a right-table predicate in WHERE can remove unmatched rows and effectively turn the result into an inner join. If all customers must remain while only paid orders are joined, put the paid-order condition in ON. Always reason about cardinality and NULL-extended rows.

### Q35. How do NULL, IN, EXISTS, and NOT IN behave?

NULL means unknown/missing, not zero or an empty string. Use IS NULL, not = NULL. SQL uses three-valued logic, and rows where a WHERE predicate evaluates to unknown are filtered out.

EXISTS tests whether a subquery returns any row; IN tests membership. The optimizer may transform either into similar plans, so neither is universally faster. NOT IN can behave unexpectedly if the subquery includes NULL; NOT EXISTS is often safer when nullable values are possible.

### Q36. What are CTEs and window functions useful for?

A common table expression (CTE) names a subquery for readability and reuse within a statement. Optimizer behavior, including merging or materialization, depends on query shape and version. Window functions compute over related rows without collapsing each group to one row.

~~~sql
SELECT employee_id, department_id, salary,
       DENSE_RANK() OVER (
         PARTITION BY department_id ORDER BY salary DESC
       ) AS salary_rank
FROM employees;
~~~

This ranks employees by salary within each department and preserves employee rows. ROW_NUMBER assigns unique sequence numbers, RANK leaves gaps after ties, and DENSE_RANK does not. Add a deterministic tie-breaker when stable row ordering matters.

### Q37. How would you solve common SQL interview problems?

For the second-highest distinct salary:

~~~sql
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
~~~

It returns NULL if no lower distinct salary exists. For the highest-paid employee(s) in each department, use DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) and keep rank 1 to preserve ties. To find duplicates, GROUP BY the business key and HAVING COUNT(*) > 1. Deleting duplicates requires deciding which row survives and protecting the operation with a transaction/backup and a unique constraint afterward.

## Indexes and query optimization

### Q38. How do composite indexes work?

An index on (tenant_id, status, created_at) is ordered by tenant, then status within tenant, then date within that combination. It is generally useful for predicates beginning with tenant_id and often tenant_id plus status. It is not equivalent to three independent indexes. This is the leftmost-prefix principle, with nuances for range predicates and optimizer choices.

Choose column order based on actual filters, equality/range conditions, joins, and sorting. Equality predicates often precede range columns in a useful index, but no single rule replaces query-plan evidence. A composite index can cover a query if all needed columns are present. Extra indexes increase storage and write cost.

### Q39. What is a clustered index in InnoDB?

InnoDB stores table rows in the leaf pages of the clustered index, usually the primary key. Secondary index entries contain the secondary key plus primary-key value to find the clustered row. A wide primary key is therefore repeated in secondary indexes and increases their size.

Choose a stable, reasonably compact primary key. A secondary-index lookup may require an additional clustered-index lookup unless the secondary index covers the query. An index lookup does not always mean one disk read.

### Q40. What is a covering index, and why can indexes slow writes?

A covering index contains the columns needed for a query so it can be satisfied from the index without some table-row lookups. It can help frequent read paths, but including every selected column makes indexes wide. Each maintained index adds work for inserts, updates, and deletes.

Create indexes from observed query patterns, not from every column mentioned in WHERE clauses. Validate with EXPLAIN and workload measurements. Remove redundant indexes cautiously after checking query patterns and constraints.

### Q41. Why might MySQL ignore an index?

The optimizer estimates the cost of candidate plans. It may prefer a table scan when a predicate matches many rows, statistics are stale, selectivity is poor, a function or implicit conversion is applied to the indexed column, or index order does not support the predicate. LIKE '%word' generally cannot use an ordinary B-tree index for a prefix seek.

Inspect EXPLAIN, predicates, parameter types, table size, and statistics. ANALYZE TABLE may refresh statistics, but do not force an index before understanding the optimizer's choice. A forced plan can become worse as data distribution changes.

### Q42. How do you use EXPLAIN and EXPLAIN ANALYZE?

EXPLAIN shows the optimizer's planned access paths, join order, estimated rows, and chosen indexes. In MySQL 8.0.18 and later, EXPLAIN ANALYZE executes the query and reports actual timing and row information for supported statements; it is not merely a harmless plan description.

Compare estimated and actual rows, look for large scans, repeated inner-loop work, temporary tables or sorts, and filters that discard most rows late. A fix may be a better index, more selective predicates, a rewritten join, less data fetched, or updated statistics. Treat the plan as evidence to interpret, not an automatic instruction to add indexes.

### Q43. How do you investigate a slow MySQL query?

Identify the endpoint/query, onset time, p95/p99 latency, traffic volume, and whether the time is query execution or connection-pool wait. Capture SQL and representative parameter values; inspect EXPLAIN and, where safe, EXPLAIN ANALYZE. Check row estimates, indexes, lock waits, concurrency, buffer-pool/disk behavior, and recent schema/data/application changes.

Form a hypothesis and change one thing at a time: index, predicate shape, projection, pagination, batching, or query design. Validate correctness and compare under representative load. EXPLAIN ANALYZE runs the query, so do not use it on a potentially expensive production statement without understanding the impact.

### Q44. How do functions, implicit casts, N+1 queries, and pagination affect performance?

A predicate like WHERE DATE(created_at) = ? may prevent a normal range lookup on created_at. Prefer created_at >= ? AND created_at < ?. Comparing unlike types can introduce conversions that affect index usage.

An ORM N+1 problem issues one initial query and then one query per result row, often through lazy loading. Use deliberate fetch plans, projections, batch fetching, or bounded secondary queries without creating huge Cartesian result sets. Offset pagination can be costly for deep pages because the database walks past many rows; keyset pagination uses the last seen ordering key and a matching index.

~~~sql
SELECT id, created_at
FROM orders
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 50;
~~~

The order must be deterministic and aligned with the cursor predicate and index.

## Transactions and concurrency

### Q45. Explain ACID in practical terms.

Atomicity means a transaction's changes commit or roll back together. Consistency means constraints and application invariants hold before and after a successful transaction. Isolation controls interactions between concurrent transactions. Durability means committed changes survive failures according to database and storage configuration.

ACID does not automatically enforce every application invariant. A balance check performed in Java without a lock or conditional update can race. Use constraints, conditional updates, and correctly scoped transactions.

### Q46. What are MySQL isolation levels?

InnoDB supports READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, and SERIALIZABLE. READ UNCOMMITTED may expose uncommitted data. READ COMMITTED gives each consistent read a fresh snapshot. InnoDB REPEATABLE READ normally reuses a snapshot for nonlocking consistent reads within a transaction. SERIALIZABLE adds stronger conflict behavior and can reduce concurrency.

Describe the actual InnoDB context, not only a generic textbook table. InnoDB combines MVCC consistent reads with locking reads, and locks depend on the statement, index, predicate, and isolation level. SELECT ... FOR UPDATE is not the same as a plain consistent read.

### Q47. What are MVCC, undo logs, and consistent reads?

MVCC lets a transaction read an appropriate older row version without blocking every concurrent writer. InnoDB uses undo information to reconstruct versions needed by consistent reads. Old versions cannot be purged while active transactions still need them.

Long-running transactions can retain undo history, delay purge, and increase work. Keep transactions short, avoid remote calls while holding a transaction open, and do not leave read transactions open while processing a large result set or waiting for user input.

### Q48. What does InnoDB REPEATABLE READ mean?

For ordinary nonlocking consistent reads, InnoDB REPEATABLE READ generally reuses the snapshot established by the transaction's first consistent read. Repeating the same query can return the same snapshot even if another transaction commits an update. Locking reads and writes use current data and locks, so mixing them with snapshot reads can surprise developers.

InnoDB uses record, gap, and next-key locks in relevant locking operations to prevent certain conflicts and phantom inserts. Exact locks depend on the index and predicate. Do not assume every SELECT locks rows or every range query locks the same range regardless of index choice.

### Q49. Explain lost updates and optimistic versus pessimistic locking.

A lost update can occur when two requests read a value, independently calculate a new value, and overwrite one another. Optimistic locking uses a version column and updates only if the version still matches; zero affected rows signals a conflict. Pessimistic locking obtains a database lock, for example SELECT ... FOR UPDATE, before changing the row.

Often a conditional atomic update expresses the invariant cleanly:

~~~sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = ? AND quantity > 0;
~~~

Check the affected-row count. Pessimistic locks can serialize hot rows and deadlock; optimistic locking can cause retries under contention.

### Q50. Explain record, gap, next-key, and intention locks.

Record locks protect index records. Gap locks protect gaps against inserts in relevant contexts. Next-key locks combine a record lock with the preceding gap and help prevent phantom inserts for applicable locking range operations. Intention locks signal a transaction's intention to acquire row-level locks within a table.

InnoDB locks index records/ranges, not an abstract business predicate independently of the access path. A missing or unsuitable index can cause a locking statement to scan and lock many records. Design the index and transaction scope together and inspect deadlock reports instead of guessing.

### Q51. How do you diagnose and reduce deadlocks?

A deadlock occurs when transactions wait in a cycle, each holding a resource needed by another. InnoDB detects many deadlocks and rolls back a victim. Retrieve the deadlock report, identify both statements, indexes, lock modes, and conflicting rows/ranges. Correlate with query-plan changes and transaction ordering.

Reduce deadlocks by updating rows in a consistent order, keeping transactions short, indexing predicates to limit locked rows, and avoiding unnecessary work inside transactions. Retry the whole transaction with bounded backoff only when the operation is safe to repeat. Do not retry only the last statement after the earlier transaction has rolled back.

## InnoDB, replication and reliability

### Q52. What are the buffer pool, redo log, undo log, and binary log?

The InnoDB buffer pool caches table and index pages in memory. The redo log records changes needed for crash recovery. Undo logs retain previous row versions for rollback and MVCC. The binary log records database changes for replication and point-in-time recovery workflows. They serve different purposes.

Durability depends on configuration and storage guarantees. Replication is not a backup; accidental changes may replicate. Backup and restore need retention, tested procedures, and a recovery-point/recovery-time plan.

### Q53. What is replication lag and how can it affect an API?

Writes usually go to the primary and changes are applied to replicas asynchronously or according to the configured topology. A read from a lagging replica may not show a just-committed write, breaking read-after-write behavior.

Route immediate follow-up reads to the primary or use a consistency mechanism supported by the architecture. Other endpoints may tolerate stale reads. Do not add arbitrary sleeps; lag varies and sleeping does not guarantee freshness. Monitor replication lag and define endpoint-specific consistency needs.

### Q54. What should a backend developer know about backups and schema migrations?

Backups matter only if restoration is tested. Understand retention, point-in-time recovery, recovery time objectives, recovery point objectives, and ownership of restore procedures. Replicas do not protect against all accidental data changes.

Prefer backward-compatible expand–migrate–contract releases: add compatible schema, deploy code that supports both versions, backfill in bounded batches, switch reads/writes, and remove obsolete schema later. DDL, backfills, metadata locks, and old/new application versions can cause outages. Test migration time and rollback/roll-forward plans on realistic data.

### Q55. Partitioning versus indexing: what is the difference?

Partitioning divides a table according to a partitioning expression and can help lifecycle management or queries that prune irrelevant partitions. It does not automatically make every query faster. An index helps locate rows within a table or partition. Partitioning is useful when predicates align with the partition key and there is a real operational/query benefit; it is not a substitute for indexing.

### Q56. Why does a database connection pool become exhausted?

Causes include slow queries, long transactions, leaked connections, too many application replicas, or a pool larger than the database can sustain. Increasing pool size may worsen contention. Track active/idle connections, wait time, timeout count, query and transaction duration, and total fleet-wide pool capacity.

Release connections promptly, avoid network calls inside transactions, and apply backpressure. Pool wait can be a symptom of slow queries rather than the root cause.

## Production troubleshooting

### Q57. An index exists but the optimizer ignores it. What do you check?

Verify predicate shape, parameter types, leftmost-prefix use, selectivity, functions/casts, and whether the query returns a large fraction of the table. Inspect EXPLAIN and statistics. A table scan can be the correct plan for a low-selectivity query. Refresh statistics only when justified and avoid forcing an index without evidence.

### Q58. A composite index did not improve the query. Why?

The query may not use the leftmost prefix, a range condition may limit later index use, the order may not support filtering/sorting, or many table rows may still need fetching. The bottleneck could instead be a join, sort, lock wait, or connection-pool delay. Compare actual rows and timing, not merely the index definition.

### Q59. A join suddenly returns duplicate entities. What do you do?

A one-to-many join legitimately repeats parent columns. If one order has five items, joining orders to items returns five joined rows. Aggregate at the correct grain, use EXISTS when only existence matters, or fetch parent and child collections deliberately. DISTINCT can hide the symptom and add cost without fixing the underlying query logic.

### Q60. CPU and disk reads are high on MySQL. How do you investigate?

Find top queries by total time and execution count. Distinguish CPU-heavy scans/sorts from I/O-heavy reads and lock contention. Inspect plans, rows examined, buffer-pool behavior, query frequency, and recent traffic changes. Reduce unnecessary data, fix access paths, batch requests, and cache suitable read-heavy data only after defining consistency requirements.

### Q61. Why can database update plus cache invalidation still leave stale data?

The database commit can succeed while invalidation fails. A concurrent reader may read the old row before the update and populate Redis after invalidation. These are different failure windows and need different mitigations. Use finite TTLs, durable outbox events with retries, versioning or refresh coordination where justified, and database reads for flows requiring read-your-writes. Do not claim a simple ordering makes two independent systems atomic.

### Q62. A schema migration threatens availability. What is your response?

Understand the exact DDL algorithm and lock behavior for the deployed MySQL version, table size, indexes, and platform. Test on realistic data, use expand–migrate–contract, monitor metadata locks and replica lag, and define pause and roll-forward/rollback plans. “Online DDL” does not mean zero impact or no locks in every phase.

# Part III — Redis + MySQL with Java/Spring

### Q63. How would you implement cache-aside with Spring Data Redis?

An explicit implementation makes the miss and fallback behavior visible. This example is illustrative and omits production policy details such as metrics, negative caching, single-flight, and typed exception classification.

~~~java
@Service
public class ProductService {
    private final ProductRepository repository;
    private final StringRedisTemplate redis;
    private final ObjectMapper objectMapper;

    public ProductService(ProductRepository repository,
                          StringRedisTemplate redis,
                          ObjectMapper objectMapper) {
        this.repository = repository;
        this.redis = redis;
        this.objectMapper = objectMapper;
    }

    public ProductDto getProduct(long id) {
        String key = "product:v1:" + id;
        try {
            String cached = redis.opsForValue().get(key);
            if (cached != null) {
                try {
                    return objectMapper.readValue(cached, ProductDto.class);
                } catch (JsonProcessingException invalidValue) {
                    redis.delete(key); // emit a metric in real code
                }
            }
        } catch (DataAccessException cacheUnavailable) {
            // Continue to the source of truth only if fallback policy allows it.
        }

        Product product = repository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        ProductDto dto = ProductDto.from(product);
        try {
            redis.opsForValue().set(key, objectMapper.writeValueAsString(dto),
                                    Duration.ofMinutes(5));
        } catch (Exception cacheWriteFailure) {
            // Log/measure safely; do not log the full payload.
        }
        return dto;
    }
}
~~~

Production code should not swallow all exceptions indiscriminately: programming errors must not be misclassified as cache outages. Use DTOs rather than ORM entities with lazy state. Configure finite timeouts, stable serialization, key versioning, metrics, and a database-protection policy for cache outages.

### Q64. How should writes update MySQL and invalidate Redis?

Commit the database transaction first, then invalidate the key. Invalidation should occur only after successful commit; an after-commit callback or transaction-aware event avoids evicting for a rolled-back write. However, invalidation can fail after commit, so this is not a complete reliability guarantee.

~~~java
@Transactional
public ProductDto updateProduct(long id, UpdateProductRequest request) {
    Product product = repository.findById(id)
        .orElseThrow(() -> new ProductNotFoundException(id));
    product.setName(request.name());
    product.setPrice(request.price());
    Product saved = repository.save(product);

    // Publish an application event. Handle it after commit.
    // Use a transactional outbox when durable event delivery is required.
    events.publishEvent(new ProductChanged(saved.getId()));
    return ProductDto.from(saved);
}
~~~

An in-process event may be lost if the process dies between database commit and handling. A transactional outbox stores the event in the same database transaction; a publisher retries and a consumer invalidates idempotently. This supports eventual invalidation, not zero staleness during delivery delay.

### Q65. How should serialization and cache schema evolution be handled?

Use namespaced/versioned keys such as product:v2:42 if incompatible formats may coexist. Prefer stable DTO schemas over native Java serialization of arbitrary object graphs, which couples cache values to class details and can create compatibility/security risks. During rolling deployment, old and new application versions may run simultaneously; ensure they can coexist or deliberately switch namespaces and protect MySQL from the cold-cache surge.

Do not cache secrets by default. Authorization and tenant isolation must still be enforced on cache hits; a cache key is not proof that a caller can access a value.

### Q66. What should happen when Redis is unavailable?

The answer depends on the operation. For an optional cache, a bounded fallback to MySQL may preserve correctness but can cause a traffic surge. For rate limiting or coordination, fail-open may be unsafe; fail-closed may reduce availability. Define policy per feature.

Use finite timeouts, bounded concurrency, circuit breaking where appropriate, request coalescing, and database load shedding. A cache outage can become a database outage if fallback is unbounded. Monitor Redis errors and MySQL pool utilization together.

### Q67. How do you prevent cache stampede in Spring?

Use per-key single-flight/request coalescing so concurrent misses share one database load. Use a distributed refresh lease only when cross-instance coordination is needed and stale-owner behavior is acceptable. TTL jitter reduces synchronized expiration. Where stale data is acceptable, serve a stale value while refreshing asynchronously.

Cap database concurrency and protect MySQL with backpressure. A lock alone is not enough: if the holder pauses or the lease expires, duplicate loaders can still occur. Measure miss rate, load count per key, database query rate, and fallback latency.

# Part IV — Real-world interview scenarios

For each scenario, distinguish immediate mitigation, investigation, root cause, and long-term prevention.

### Q68. Redis is down and MySQL load surges. How do you respond?

Protect MySQL first: cap concurrent fallback reads, apply endpoint-specific load shedding and rate limits, use finite timeouts, and prioritize critical operations. Then inspect Redis health, connection errors, network/deployment changes, and hit ratio. Recover Redis safely and warm only hot keys with bounded concurrency. Long term, test cache-outage behavior and set explicit database capacity limits.

### Q69. A user updates a record but still sees stale data. How do you investigate?

Trace database commit, invalidation event, key/TTL, and subsequent read route. Confirm the endpoint and key namespace, and check whether a lagging MySQL replica or local cache serves the response. Reproduce the miss-fill/update race if possible. Mitigate critical read-after-write by returning the committed result or bypassing cache; add durable invalidation and metrics if event loss is responsible.

### Q70. One Redis key overloads a Cluster node. What are the options?

Determine whether it is a large value, frequent counter, or giant collection. A single key maps to one hash slot and cannot simply be spread across nodes by normal hashing. Consider local caching for safe immutable data, splitting a logical aggregate into independently addressable keys, or sharding a counter and combining results. Each option changes consistency, atomicity, or read cost.

### Q71. Redis memory grows continuously. What do you inspect?

Check maxmemory, eviction policy, key count, TTL distribution, representative key sizes, Stream retention, and key cardinality. Look for missing TTLs, per-request keys, duplicate namespaces, or workload changes. Fix lifecycle/data shape before blindly increasing capacity or changing eviction.

### Q72. A distributed lock expires while the original process is still running. What can go wrong?

A second process can acquire the lease while the original process is paused; both may then perform work. Token-checked release prevents one owner deleting another's lock but does not prevent stale work. Use fencing tokens checked by the protected resource, database constraints/conditional writes, or a coordination system with appropriate guarantees.

### Q73. A query that was fast at small scale is now slow. What is the process?

Capture SQL and parameters; inspect execution plan, estimated/actual rows, selectivity, index coverage, pagination offset, and data distribution. Check query volume and concurrency too. Change indexes only after confirming the access path and write overhead. Consider keyset pagination, narrower projections, batching, or a read model if the workload changed fundamentally.

### Q74. A read replica returns stale data after a write. What should the API do?

Decide whether read-your-writes is required. Route immediate reads to the primary or use a supported consistency mechanism. Other endpoints may tolerate stale reads. Avoid arbitrary sleeps and monitor lag.

### Q75. Database connections are exhausted after adding application replicas. Why?

Potential connections grow with instance count multiplied by pool size, plus jobs and other clients. The database may be overloaded by concurrency, causing queries to take longer and hold connections longer. Calculate fleet-wide connection budget, inspect pool wait and query latency, and reduce unnecessary concurrency. Increasing each pool independently can worsen the incident.

### Q76. An invalidation event is lost or processed twice. What design survives?

Use a transactional outbox so the business update and event record commit together. Retry publication and make consumers idempotent; duplicate cache deletes should be harmless. Monitor outbox age and retry counts. Eventual invalidation still has a delay window, so TTL and read-your-writes policies remain important.

### Q77. Cache hit ratio drops after deployment. What do you check?

Check key prefixes/version, serialization changes, TTL configuration, cache endpoint, code path, cache flushes, traffic mix, and errors. A deliberate key-version change creates cold misses. Protect MySQL with bounded warming and fallback concurrency. Do not use ad hoc deserialization fallbacks that silently return incorrect business data.

### Q78. Two concurrent requests create duplicates. How do you fix it?

Enforce uniqueness in MySQL with a unique constraint and handle duplicate-key errors intentionally. An application-level SELECT-before-INSERT is insufficient because both requests may see no row. Redis can reduce duplicate work but should not replace the durable constraint. Make retries return a stable result rather than creating another entity.

### Q79. An API needs rate limiting across many instances. What should be designed?

Choose identity and algorithm (fixed window, sliding window, token bucket), update the counter/tokens atomically, set expiry, and cap key cardinality. Decide fail-open/closed behavior during Redis failure, return appropriate retry information, and monitor limit decisions. Ensure clients cannot spoof the identity used for limiting.

### Q80. Too many indexes slow a write-heavy table. How do you decide what to remove?

Inventory indexes and constraints, inspect production query frequency and plans, and identify duplicate/overlapping indexes. Test candidate removals in staging; where supported, invisible indexes can help assess read impact. Preserve uniqueness/foreign-key requirements and validate representative read/write workloads before removal.

### Q81. Counts became wrong after adding a join. How do you investigate?

Identify the intended row grain and inspect relationship cardinality. One-to-many joins multiply parent rows. Pre-aggregate, use EXISTS for existence-only logic, or count at the correct grain. Validate with small data containing multiple children and missing relationships; do not add DISTINCT reflexively.

### Q82. Deadlocks start after a release. What do you do?

Capture deadlock reports and compare lock order, query plans, indexes, and transaction scope before/after deployment. A new code path may touch tables in a different order or hold locks longer. Mitigate with safe bounded retries while fixing the conflict pattern. Monitor deadlock and retry rates after rollout.

### Q83. Old and new pods use different cached object formats. What is safe?

Use versioned keys or a backward-compatible schema rollout. During mixed-version deployment, ensure both versions can coexist or deliberately switch namespaces and expect cold misses. Keep TTL bounded and protect MySQL from warming traffic. Do not silently return defaults for deserialization failures.

### Q84. A cache contains sensitive customer information. What risks matter?

Minimize cached fields, set TTLs, use appropriate authentication/TLS/access controls, avoid secrets in keys and logs, and consider encryption/data-residency requirements. Enforce authorization on cache hits and invalidate on permission changes or account closure.

### Q85. Design a product-read API that is fast but reasonably fresh.

Keep MySQL authoritative, index the canonical lookup, and use cache-aside with a finite TTL for hot product DTOs. Invalidate after committed writes and use durable events if the stale window warrants the complexity. Add single-flight loading, TTL jitter, metrics, and bounded database fallback. Return the committed update result for read-after-write and avoid shared caching of permission-dependent data.

# Redis command reference

Use diagnostic commands with awareness of production impact. Some managed services restrict them.

~~~text
# Strings and counters
GET product:42
SET product:42 '{"id":42}' EX 300
MGET product:42 product:43
INCR rate:user:17

# Hashes / sets / sorted sets
HGET user:17 status
HSET user:17 status active
SADD product:42:tags keyboard office
SISMEMBER product:42:tags office
ZADD leaderboard 1200 user:17
ZREVRANGE leaderboard 0 9 WITHSCORES

# Expiry
TTL product:42
PTTL product:42
EXPIRE product:42 300
PERSIST product:42

# Incremental scan; tolerate duplicates and changing keyspace
SCAN 0 MATCH 'product:*' COUNT 100

# Diagnostics
INFO memory
INFO stats
INFO keyspace
MEMORY USAGE product:42
SLOWLOG GET 20
LATENCY LATEST
~~~

Avoid KEYS * on a busy production instance. MONITOR can be expensive and expose command arguments; use it briefly and under controlled conditions. Verify command availability in the deployed version/service.

# MySQL command reference

~~~sql
-- Schema and index inspection
SHOW CREATE TABLE orders;
SHOW INDEX FROM orders;
DESCRIBE orders;

-- Plan inspection
EXPLAIN SELECT id, created_at
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 20;

-- MySQL 8.0.18+: executes the query, so use with care
EXPLAIN ANALYZE
SELECT id FROM orders WHERE customer_id = 42;

-- Transaction boundaries
START TRANSACTION;
UPDATE inventory SET quantity = quantity - 1
WHERE product_id = 42 AND quantity > 0;
COMMIT;
-- Use ROLLBACK if the transaction must be abandoned.

-- Sessions and InnoDB diagnostics
SHOW FULL PROCESSLIST;
SHOW ENGINE INNODB STATUS;

-- Lock tables; availability depends on version and privileges
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
~~~

Do not kill sessions or change global settings without understanding transaction consequences and the operational procedure.

# Official references

Use official documentation for version-sensitive details and check the version deployed by the application.

## Redis
- [Redis data types](https://redis.io/docs/latest/develop/data-types/)
- [Redis command reference](https://redis.io/docs/latest/commands/)
- [EXPIRE command](https://redis.io/docs/latest/commands/expire/)
- [Redis eviction policies](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
- [Redis Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
- [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)

## MySQL
- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/)
- [InnoDB introduction](https://dev.mysql.com/doc/refman/8.4/en/innodb-introduction.html)
- [Indexes and optimization](https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html)
- [Using EXPLAIN](https://dev.mysql.com/doc/refman/8.4/en/using-explain.html)
- [InnoDB locking and transaction model](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-transaction-model.html)
- [MySQL replication](https://dev.mysql.com/doc/refman/8.4/en/replication.html)
- [MySQL backup and recovery](https://dev.mysql.com/doc/refman/8.4/en/backup-and-recovery.html)

## Java and Spring
- [Spring Data Redis reference](https://docs.spring.io/spring-data/redis/reference/)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)
- [Spring transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [JDBC basics](https://docs.oracle.com/javase/tutorial/jdbc/basics/)

---

**Final revision tip:** Be prepared to explain what happens when the cache is stale, a replica lags, a transaction deadlocks, a connection pool is exhausted, or a dependency fails. Those failure paths often distinguish memorized definitions from senior-level engineering judgment.
