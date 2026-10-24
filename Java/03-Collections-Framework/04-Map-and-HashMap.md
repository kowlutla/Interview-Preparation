# Map and HashMap Internals

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Map is one of the most important Java APIs in backend systems. Correctness depends on stable key equality and hashing; performance depends on hash distribution, capacity, and access patterns; concurrency requires atomic map operations or external coordination rather than assuming individual calls compose into a transaction.

## 1. Map contract

### Q1. What does Map represent? *(Beginner)*

Map associates keys with values and normally permits at most one mapping per key according to its equality or ordering semantics. put replaces the previous value for an existing key, and get retrieves the value associated with a key.

Map is not a subtype of Collection. Its keySet, values, and entrySet are collection views backed by the map. Choose Map when lookup by key is central to the data model.

### Q2. How does HashMap work at a high level? *(Intermediate)*

HashMap computes a hash from the key's hashCode, uses it to select a bucket, and then compares candidate keys to find a matching mapping. Collisions can be stored in the same bucket, with implementation-specific structures and resizing behavior.

Expected average get/put time is constant under suitable hashing assumptions, but worst-case and practical performance depend on collision patterns, key equality cost, resizing, and memory locality. Do not treat expected O(1) as an unconditional guarantee.

### Q3. What happens when two keys have the same hash code? *(Intermediate)*

They collide into the same bucket region, but HashMap still uses equality to distinguish different keys. A hash collision does not mean the keys are equal, and unequal keys may legally have the same hash code.

Good hashCode implementations distribute keys well. The equality contract is more important than trying to produce unique hashes, because uniqueness is not possible for arbitrary key spaces.

### Q4. What is the difference between capacity and size in HashMap? *(Intermediate)*

Size is the number of mappings. Capacity refers to the backing table's bucket count. Load factor controls the threshold at which the table is resized as mappings are added.

Pre-sizing can reduce resizing when the approximate number of entries is known, but oversizing wastes memory and can affect iteration cost. Exact growth and sizing details are implementation-specific; use public constructors and benchmark rather than depending on internal constants.

### Q5. What is the default load factor, and what does it trade off? *(Advanced)*

OpenJDK HashMap traditionally uses a default load factor of 0.75. A lower load factor generally uses more buckets and can reduce collisions at the cost of memory; a higher load factor saves table space but can increase collision work.

The default is an implementation/API behavior to verify for the target JDK, not a reason to tune every map. Measure with the expected key distribution and memory budget.

### Q6. How does HashMap handle collisions in modern OpenJDK? *(Advanced)*

OpenJDK HashMap buckets can contain linked nodes and, under specified conditions, tree bins to improve behavior with heavy collisions. Treeification depends on implementation thresholds, table capacity, and comparable/order-related behavior.

These internal thresholds are not a stable public contract. Interview answers should explain why collision handling matters and how tree bins reduce severe collision costs without claiming every bucket is always a tree.

### Q7. Why must a HashMap key be immutable or stable? *(Advanced)*

HashMap determines where a mapping belongs based on the key's hash when inserted. If the key's hash/equality state changes, later lookup may search a different bucket or fail to match the stored key.

Use immutable value keys or a stable ID. Mutability is safe only when it does not affect equality/hashCode while the key is stored. This is one of the most common causes of confusing map bugs.

## 2. Nulls, iteration and equality

### Q8. Does HashMap allow null keys and values? *(Beginner)*

HashMap permits one null key and multiple null values. This differs from ConcurrentHashMap, which rejects null keys and values. The distinction matters because get returning null in HashMap can mean either no mapping or a mapping to null.

Use containsKey when that distinction matters, or avoid null values if absence and null would be ambiguous in the domain. Null policy is part of the concrete implementation's contract.

### Q9. What is the difference between containsKey and get? *(Intermediate)*

get returns the mapped value or null if there is no mapping. Since HashMap permits null values, get alone cannot distinguish absence from a present mapping to null. containsKey can make that distinction.

If null values are not meaningful for the application, prohibit them by convention or use a value type that represents absence explicitly. Avoid doing two lookups unnecessarily when an atomic method better fits the operation.

### Q10. What is Map.Entry, and why use entrySet? *(Intermediate)*

Map.Entry represents one key-value mapping. Iterating entrySet lets code access each key and value together without a separate lookup for each key.

    for (Map.Entry<String, Integer> entry : counts.entrySet()) {
        System.out.println(entry.getKey() + "=" + entry.getValue());
    }

Entry objects obtained from a map's entry view have contract-specific behavior. Do not retain mutable entry references beyond the operation unless the API guarantees the required lifetime and semantics.

### Q11. Does HashMap preserve insertion order? *(Beginner)*

No. HashMap makes no iteration-order guarantee. An observed order may appear stable in a small test but can change after resizing, key changes, implementation changes, or a JDK upgrade.

Use LinkedHashMap when insertion order is part of the requirement, or TreeMap for sorted order. Never write tests or business logic that depend on an unspecified HashMap iteration sequence.

### Q12. What is the difference between HashMap and IdentityHashMap? *(Advanced)*

HashMap compares keys using equals and hashCode. IdentityHashMap uses reference identity (==) for keys and identity hash behavior. It is intended for specialized cases such as object-graph processing, not as a drop-in map for ordinary domain keys.

Using IdentityHashMap changes the meaning of key uniqueness. Two equal but distinct objects are separate keys. Document this clearly because callers often assume normal Map equality semantics.

## 3. Atomic map operations

### Q13. What does putIfAbsent do? *(Intermediate)*

putIfAbsent associates a value when the key is absent or, in Map's default-method semantics, when the current mapping is null. It returns the previous value according to the API contract. Exact concurrency guarantees depend on the implementation.

For concurrent code, use an implementation that documents atomicity for the operation. A sequence of containsKey followed by put is not atomic and can race with another thread.

### Q14. What is the difference between computeIfAbsent and get-then-put? *(Advanced)*

computeIfAbsent performs a mapping operation according to the Map contract and, for concurrent implementations such as ConcurrentHashMap, provides stronger atomic behavior than an unsynchronized get-then-put sequence. It is useful for lazy initialization of a value associated with a key.

The mapping function should be short, should not recursively update the same map in prohibited ways, and should avoid side effects that may be repeated in some implementations or failure cases. Read the specific implementation's contract before using it for expensive work.

### Q15. What are merge and compute useful for? *(Advanced)*

merge combines an incoming value with an existing non-null value using a remapping function, while compute and computeIfPresent provide more general remapping behavior. They can simplify counters and aggregation.

    counts.merge(word, 1, Integer::sum);

Remember that returning null from certain remapping functions removes the mapping. Handle overflow for numeric counters when the range can be exceeded, and avoid side effects that make retries or concurrent execution unsafe.

### Q16. Why is containsKey followed by put unsafe in concurrent code? *(Advanced)*

Another thread can change the map between the check and the update. The pair is a compound operation, and thread-safe individual methods do not automatically make a sequence atomic.

Use an atomic operation such as putIfAbsent, computeIfAbsent, or a lock covering the entire invariant. The right method depends on whether you need to create, update, or compare-and-set state.

## 4. Production scenarios

### Q17. How would you count word frequencies using a Map? *(Intermediate)*

Use merge or compute to update each word's count. Normalize text according to the domain's case and Unicode requirements before counting; splitting text by whitespace is not a complete natural-language tokenizer.

    Map<String, Integer> counts = new HashMap<>();
    for (String word : words) {
        counts.merge(word, 1, Integer::sum);
    }

For very large counts, consider long values and checked overflow if wraparound is unacceptable. For parallel aggregation, choose a concurrency strategy based on contention and required consistency.

### Q18. How do you choose initial capacity for a large HashMap? *(Advanced)*

Estimate expected mappings and choose a capacity that avoids repeated resizing under the expected load factor, while avoiding excessive memory use. Some JDKs provide sizing helpers in specific collections APIs, but these are version-dependent.

Do not blindly set capacity equal to expected entry count because resizing thresholds depend on load factor. Validate with the target JDK and measure allocation peaks when building large maps.

### Q19. How can HashMap cause memory problems? *(Senior)*

A map retains its keys and values while mappings remain reachable. Large maps can retain entire object graphs through values, and a long-lived static map can become an accidental memory leak if entries are never evicted. Oversized tables and many boxed objects add overhead.

Inspect heap dumps, dominator trees, entry counts, key cardinality, and lifecycle. Use bounded caches with explicit eviction policies instead of assuming a plain HashMap is an appropriate cache.

### Q20. How should you implement an in-memory cache with a Map? *(Senior)*

Define maximum size, expiration semantics, concurrency, loading behavior, failure caching, and metrics before implementing the cache. A plain HashMap does not provide eviction, TTL, or thread safety. A ConcurrentHashMap supports concurrency but still does not supply a complete cache policy.

Prefer a maintained caching library when requirements include eviction, refresh, statistics, or stampede prevention. If implementing a simple cache, specify how simultaneous loads are coordinated and whether stale values may be served.

### Q21. Why can HashMap iteration order change after adding one entry? *(Advanced)*

Adding an entry can trigger a resize, changing bucket placement and traversal order. The internal table size and hash distribution determine iteration order, and that order is not guaranteed by the API.

If deterministic output is needed for logs, tests, or serialization, sort keys or use an ordered map. Do not try to infer or preserve a particular HashMap layout.

### Q22. How do you diagnose poor HashMap performance? *(Senior)*

Check key cardinality, hash distribution, equality cost, map sizing, resize frequency, contention, allocation, and whether the workload needs sorted or range queries. Use profiling and representative benchmarks rather than assuming collisions are the only cause.

If keys are attacker-controlled, evaluate adversarial hash patterns and resource limits. A map with excellent average-case behavior can still be used in a denial-of-service path if input size and operation cost are unbounded.

## Official references

1. [Java SE 21 API — Map](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Map.html)
2. [Java SE 21 API — HashMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html)
3. [Java SE 21 API — LinkedHashMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedHashMap.html)
4. [Java SE 21 API — ConcurrentHashMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

## Chapter checklist

- [ ] Explain hashing, collisions, capacity, load factor, and resize trade-offs.
- [ ] Keep equality-relevant key state stable.
- [ ] Understand null ambiguity and iteration-order guarantees.
- [ ] Use atomic map methods for compound update intent.
- [ ] Treat caches as lifecycle/concurrency policies, not just maps.

## Worked examples: contracts in code

### Example 1. A key must keep stable equality and hash state

\`\`\`java
import java.util.HashMap;
import java.util.Map;

final class UserKey {
    private String email;

    UserKey(String email) { this.email = email; }

    @Override public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof UserKey)) return false;
        UserKey that = (UserKey) other;
        return java.util.Objects.equals(email, that.email);
    }

    @Override public int hashCode() {
        return java.util.Objects.hashCode(email);
    }

    void setEmail(String email) { this.email = email; }
}

class MutableKeyDemo {
    public static void main(String[] args) {
        Map<UserKey, String> map = new HashMap<>();
        UserKey key = new UserKey("a@example.com");
        map.put(key, "user");
        key.setEmail("b@example.com");
        System.out.println(map.get(key)); // typically null; do not rely on lookup
    }
}
\`\`\`

The map placed the entry using the hash value at insertion time. Mutating state used by `hashCode` or `equals` breaks the map's lookup assumptions. The safe design is an immutable key, not a workaround that attempts to reinsert keys after mutation.

### Example 2. Use a map operation for a per-key computation

\`\`\`java
Map<String, java.util.List<String>> grouped = new HashMap<>();
grouped.computeIfAbsent("team-a", ignored -> new java.util.ArrayList<>())
       .add("service-1");
\`\`\`

This is appropriate for a thread-confined `HashMap`. It does not make the list safe for concurrent mutation. With a `ConcurrentHashMap`, the mapping computation has documented atomicity constraints, but mutating a returned mutable value still requires its own concurrency design. Keep mapping functions short and avoid recursively updating the same map.

### Example 3. Choose a duplicate-key policy deliberately

\`\`\`java
Map<String, Integer> totals = new HashMap<>();
totals.merge("orders", 1, Integer::sum);
totals.merge("orders", 1, Integer::sum);
System.out.println(totals.get("orders")); // 2
\`\`\`

`merge` expresses the duplicate policy as summation. For `Collectors.toMap`, supply a merge function when duplicate keys are expected; otherwise, the two-argument overload throws for duplicate mapped keys. Do not silently choose first/last if duplicates indicate invalid domain data.
