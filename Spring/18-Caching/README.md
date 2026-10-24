# Spring Caching

## Q&A

### Q1. What does @Cacheable do?

It checks the configured cache for a key and, on a miss, invokes the method and stores the result according to the cache configuration. The annotation relies on caching infrastructure and commonly proxy interception, so self-invocation may bypass it. Cache keys and value semantics must be designed explicitly.

### Q2. How do @CachePut and @CacheEvict differ?

`@CachePut` always invokes the method and updates the cache with the result; `@CacheEvict` removes entries. Use them when the data-change semantics are clear. Combining cache annotations without understanding evaluation and ordering can produce stale or contradictory results.

### Q3. Why is cache invalidation difficult?

The source of truth can change through multiple paths, and distributed caches introduce propagation delay and concurrent updates. Define which writes invalidate which keys, consider TTL as a safety bound rather than perfect consistency, and test concurrent read/write scenarios.

### Q4. What is a cache stampede?

A large number of callers miss or expire the same key and simultaneously recompute the value, potentially overwhelming the backing service. Depending on the cache, use request coalescing, controlled refresh, jittered TTLs, or stale-while-revalidate patterns. Measure whether complexity is justified.

### Q5. When should Redis be used instead of an in-memory cache?

A local cache is fast and simple but is per-process and can diverge across replicas. Redis can share entries across instances but adds network latency, operational dependency, and its own consistency/failure modes. Select based on freshness, sharing, availability, and capacity requirements.

### Q6. What happens to caching when the backing cache is unavailable?

Behavior depends on the cache provider and application configuration. Decide whether to fail open by invoking the underlying method or fail closed, and ensure the fallback cannot overwhelm the source of truth. Monitor cache errors and hit ratio rather than assuming the cache is always available.

## Official references

- [Spring cache abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)
