# Senior Production Scenarios 66–75

## Scenario 66 — Application reports ready before required warmup

**Symptoms:** First requests fail or are very slow after deployment.

**Investigation:** Check startup runners, lazy beans, cache warmup, and readiness probe timing.

**Resolution:** Make readiness reflect the minimum state needed to serve safely; keep warmup bounded and optional work asynchronous where appropriate.

**Prevent:** Test first-request behavior on cold instances.

**Follow-up:** Why can lazy initialization move failure into the request path?

## Scenario 67 — Session state breaks behind load balancer

**Symptoms:** Users intermittently lose login state across replicas.

**Investigation:** Check session persistence, load-balancer affinity, session fixation handling, and whether the architecture assumes local in-memory state.

**Resolution:** Use a deliberate shared session strategy or stateless token design where appropriate, and secure session cookies.

**Prevent:** Test across multiple replicas and failover.

**Follow-up:** What are the trade-offs of sticky sessions?

## Scenario 68 — Authorization differs between controller and service

**Symptoms:** One route is protected but another path reaches the same operation without the expected rule.

**Investigation:** Compare request matcher coverage, method security, proxy boundaries, and direct internal callers.

**Resolution:** Enforce authorization at the appropriate service/resource boundary and keep request-layer rules coherent.

**Prevent:** Test every route and direct service invocation paths.

**Follow-up:** Why is UI hiding not authorization?

## Scenario 69 — Clock skew causes token rejection

**Symptoms:** Valid tokens are rejected near expiry or before not-before time.

**Investigation:** Compare host clocks, token claims, validator clock-skew policy, and identity-provider time.

**Resolution:** Synchronize clocks and apply narrowly justified skew tolerance; do not disable expiry validation.

**Prevent:** Monitor time synchronization and test boundary claims.

**Follow-up:** What security risk comes from excessive clock-skew tolerance?

## Scenario 70 — Log volume overwhelms storage

**Symptoms:** Storage costs rise and useful events are difficult to find.

**Investigation:** Identify duplicate exception logging, verbose SQL/request logging, high-volume success logs, and unbounded payloads.

**Resolution:** Use appropriate levels, structured fields, redaction, sampling where safe, and metrics for high-volume counts.

**Prevent:** Load-test logging volume and define retention policy.

**Follow-up:** Why should an exception not be logged at every layer?

## Scenario 71 — Heap dump contains secrets

**Symptoms:** Diagnostic artifacts expose credentials or personal data.

**Investigation:** Review who can access dumps, artifact retention, crash collection, and support bundle permissions.

**Resolution:** Restrict access and encryption, minimize retention, and follow incident procedures if exposed.

**Prevent:** Treat heap dumps as sensitive production data.

**Follow-up:** Why is a diagnostic artifact not automatically safe to share?

## Scenario 72 — Consumer lag grows continuously

**Symptoms:** Messages arrive faster than the service processes them.

**Investigation:** Inspect lag, processing duration, partition count, concurrency, retry delays, database bottlenecks, and poison-message behavior.

**Resolution:** Fix slow work, bound retries, scale within partition and downstream capacity, or batch where semantics permit.

**Prevent:** Alert on lag age and processing throughput.

**Follow-up:** Why can adding consumer threads fail to improve throughput?

## Scenario 73 — Message order is violated

**Symptoms:** Updates arrive in an unexpected order.

**Investigation:** Check partition key, concurrent processing, retry/replay behavior, and whether the domain requires global or per-entity order.

**Resolution:** Partition by the relevant key and serialize processing where required; avoid assuming order across partitions.

**Prevent:** Test retries and rebalances with ordered updates.

**Follow-up:** What throughput trade-off comes with strict ordering?

## Scenario 74 — Outbox backlog grows

**Symptoms:** Business writes succeed but event delivery is delayed.

**Investigation:** Inspect oldest pending age, relay errors, broker health, database query plan, row locking, and cleanup strategy.

**Resolution:** Restore relay capacity, retry safely, and scale only within broker/database limits.

**Prevent:** Alert on backlog age and verify replay procedures.

**Follow-up:** Why is pending count alone sometimes less useful than oldest-event age?

## Scenario 75 — Canary deployment regresses p99 latency

**Symptoms:** Error or latency SLO worsens only on new replicas.

**Investigation:** Compare dependency versions, configuration, startup/warmup state, pool settings, traces, and resource limits between canary and stable versions.

**Resolution:** Roll back or halt rollout while preserving diagnostics; fix the measured regression and rerun a representative test.

**Prevent:** Use automated SLO-based rollout gates and version-correlated dashboards.

**Follow-up:** Why can averages hide a canary regression?
