# Java Collections Framework: Overview and Contracts

**Target level:** Beginner → Senior  
**Scope:** Java 8–21

The Collections Framework provides interfaces and implementations for storing and processing groups of objects. The key design choice is usually not a class name alone; it is the required ordering, uniqueness, lookup complexity, mutation model, concurrency behavior, and null policy.

## 1. Core interfaces

### Q1. What is the Java Collections Framework? *(Beginner)*

The Collections Framework is a set of interfaces, implementations, and algorithms for representing and manipulating groups of objects. Core interfaces include Collection, List, Set, Queue, Deque, and Map. Map is part of the framework but does not extend Collection because it models key-value associations rather than a single sequence of elements.

Program against the narrowest interface that expresses the contract needed. This allows implementation changes and communicates semantics more clearly than exposing a concrete implementation everywhere.

### Q2. What is the difference between Collection and Collections? *(Beginner)*

Collection is an interface representing a group of elements. Collections is a utility class with static methods for operations such as sorting, searching, wrapping collections as unmodifiable views, and creating synchronized wrappers.

The similar names are a frequent interview trap. List, Set, and Queue extend Collection, whereas Map is separate. Collections utility methods do not make every input thread-safe or immutable; understand the specific wrapper's contract.

### Q3. What is the difference between List, Set, Queue, and Map? *(Beginner)*

List represents an ordered sequence that permits duplicates. Set represents uniqueness according to its equality or ordering contract. Queue represents elements waiting for processing under a queue discipline; Deque supports operations at both ends. Map associates keys with values and normally requires unique keys.

Choose based on domain semantics first. For example, a list of purchase line items may preserve order and duplicates; a set of permissions may enforce uniqueness; a map of user IDs to profiles supports key lookup.

### Q4. What is the difference between ArrayList and LinkedList? *(Intermediate)*

ArrayList uses a resizable array and provides fast indexed access, with amortized constant-time append and linear-time insertion/removal in the middle due to shifting. LinkedList is a doubly linked list, with linear-time indexed access and constant-time insertion/removal at a known node, but finding that node can itself be linear.

ArrayList is the default choice for most list workloads because of locality and lower per-element overhead. LinkedList is useful only when its specific deque/list operations fit the workload; Big-O alone does not capture cache and allocation costs.

### Q5. What is the difference between HashSet and TreeSet? *(Intermediate)*

HashSet uses hashing and equality for membership, typically providing expected constant-time basic operations when hashes distribute well. TreeSet uses ordering, typically providing logarithmic-time operations and sorted iteration.

TreeSet's comparator or natural ordering defines which values are treated as equivalent for set operations; this may differ from equals. The comparator should be consistent with equals when the set is expected to obey the general Set contract predictably.

### Q6. What is the difference between HashMap and TreeMap? *(Intermediate)*

HashMap is hash-based and does not guarantee iteration order. TreeMap maintains keys in sorted order according to natural ordering or a Comparator. HashMap typically offers expected constant-time get/put, while TreeMap offers logarithmic-time operations.

Use TreeMap when ordered navigation, range views, or sorted traversal is required. Do not rely on HashMap's observed iteration order; it can change with data, capacity, implementation, or JDK version.

### Q7. Why is Map not a subtype of Collection? *(Intermediate)*

A Collection models individual elements, while Map models key-value entries and exposes distinct views for keys, values, and entries. These views are collections, but the map as a whole has different semantics for insertion, lookup, and uniqueness.

The API separates them to avoid pretending that a map is a simple bag of elements. Use entrySet when processing key-value pairs efficiently rather than repeatedly looking up each key.

## 2. Equality, ordering and nulls

### Q8. Why do collections depend on equals and hashCode? *(Intermediate)*

Hash-based collections use hashCode to select candidate buckets and equals to distinguish keys or elements. If equal objects have different hash codes, lookups and uniqueness checks may fail. Tree-based collections rely on comparator or natural ordering for placement and equivalence.

A collection's correctness depends on stable equality/ordering behavior while elements are stored. Mutable keys and inconsistent comparators can produce surprising lookup behavior even when the collection implementation is correct.

### Q9. What does fail-fast iteration mean? *(Intermediate)*

Some iterators detect certain structural modifications outside the iterator and may throw ConcurrentModificationException. This is a best-effort bug-detection mechanism, not a synchronization guarantee and not a reliable way to coordinate threads.

Do not build correctness around the exception always being thrown. For single-threaded mutation, use iterator.remove or collection-specific operations. For concurrency, use proper synchronization or concurrent collections.

### Q10. What is the difference between unmodifiable and immutable collections? *(Advanced)*

An unmodifiable view rejects mutations through that view, but changes to a mutable backing collection can still be visible. An immutable collection has no externally observable mutable backing state under its contract. The term immutable is often used informally, so inspect the API.

    List<String> backing = new ArrayList<>(Arrays.asList("a"));
    List<String> view = Collections.unmodifiableList(backing);
    backing.add("b");
    System.out.println(view); // [a, b]

List.of and List.copyOf return unmodifiable lists, but immutability is shallow: mutable elements can still change. A read-only reference does not automatically make the whole object graph immutable.

### Q11. What is the difference between List.of and Arrays.asList? *(Intermediate)*

List.of was added in Java 9 and returns an unmodifiable list that rejects null elements. Arrays.asList returns a fixed-size list backed by the supplied array, supports set, and reflects changes between the list and array. It does not support structural add/remove.

    String[] data = {"a", "b"};
    List<String> list = Arrays.asList(data);
    list.set(0, "x");
    System.out.println(data[0]); // x

Use new ArrayList<>(...) when you need a structurally mutable copy. Be careful with null behavior and Java version compatibility.

### Q12. How do null policies differ among common collections? *(Advanced)*

ArrayList permits null elements. HashMap permits one null key and null values. ConcurrentHashMap does not permit null keys or values, which avoids ambiguity in concurrent retrieval. List.of and Set.of reject null elements.

Null policy is part of the API contract, not a universal Collections Framework rule. If null has business meaning, model that meaning deliberately instead of assuming every implementation accepts it.

## 3. Complexity and selection

### Q13. How should you choose a collection for a workload? *(Senior)*

Start with required semantics: duplicates, order, sorted traversal, range queries, key lookup, nulls, concurrency, and mutation. Then estimate read/write frequency, size, access pattern, and memory pressure. Finally, measure realistic workloads if performance matters.

A useful default is ArrayList for general sequences, HashSet for membership, HashMap for key lookup, ArrayDeque for queue/stack operations, and TreeMap/TreeSet for sorted navigation. These are starting points, not universal rules.

### Q14. Why is Big-O not enough to choose a collection? *(Advanced)*

Big-O describes asymptotic growth but hides constants, cache locality, allocation, resizing, hashing quality, comparator cost, and contention. ArrayList can outperform LinkedList for many operations even when a linked list has attractive theoretical insertion complexity.

Measure with representative data distributions and access patterns. Consider memory footprint and tail latency as well as average throughput. Avoid microbenchmarks that ignore warmup, allocation, and realistic object sizes.

### Q15. What is the difference between a stack and a queue, and what Java type should be preferred? *(Intermediate)*

A stack is last-in-first-out; a queue is typically first-in-first-out. Deque supports operations at both ends and can implement either pattern. ArrayDeque is often preferred over legacy Stack for stack behavior because it has a clearer deque API and avoids Stack's inheritance from Vector.

ArrayDeque does not permit null elements. If thread safety is required, use an appropriate concurrent queue/deque or external synchronization rather than assuming ArrayDeque is safe for multiple writers.

### Q16. What are collection views such as keySet, values, and entrySet? *(Intermediate)*

Map keySet, values, and entrySet are backed views of the map, not independent snapshots. Changes to the map are reflected in the views, and supported removals through the views affect the map. Structural changes during iteration follow the map's iterator contract.

Use entrySet when both key and value are needed; it avoids a separate map lookup for every key. Create a copy if a stable snapshot is required, and use concurrency-safe snapshot semantics when multiple threads may mutate the source.

## 4. Concurrency and API behavior

### Q17. Are ArrayList and HashMap thread-safe? *(Beginner)*

No. They do not provide general thread safety for concurrent mutation. Concurrent reads are only safe under appropriate safe-publication and no-racing-mutation conditions; a data race can make behavior incorrect.

Use synchronization around a shared collection, a synchronized wrapper with a correctly synchronized iteration protocol, or a suitable concurrent collection. A synchronized wrapper does not mean every compound workflow becomes atomic.

### Q18. What is the difference between synchronized collections and concurrent collections? *(Advanced)*

Synchronized wrappers generally serialize operations on a common mutex, while concurrent collections are designed for concurrent access with more specialized algorithms and semantics. ConcurrentHashMap supports high-concurrency operations and atomic methods such as computeIfAbsent, but it is not a drop-in replacement for every Map behavior.

For synchronized wrappers, iteration usually requires synchronizing on the wrapper as documented. Concurrent collections often provide weakly consistent iterators rather than fail-fast behavior. Choose based on the required consistency and atomic operations.

### Q19. What is a weakly consistent iterator? *(Advanced)*

A weakly consistent iterator can traverse a concurrent collection while other threads modify it without throwing ConcurrentModificationException as a normal consequence. It may reflect some concurrent updates and omit others, according to the collection's contract.

It is not a transactional snapshot. If an algorithm requires a consistent point-in-time view, use explicit snapshotting, synchronization, or a data model that supports versioned reads.

### Q20. What is the difference between fail-fast and weakly consistent iteration? *(Advanced)*

Fail-fast iterators may detect unsupported structural changes and throw ConcurrentModificationException; the detection is best effort and does not make the collection thread-safe. Weakly consistent iterators are designed to tolerate concurrent modifications and provide a documented, non-snapshot traversal model.

Do not use either iterator behavior as a substitute for specifying the consistency needed by the algorithm. If every element must be processed exactly once from one logical snapshot, a different protocol may be required.

## 5. Practical scenarios

### Q21. You need to preserve insertion order while ensuring uniqueness. What would you use? *(Intermediate)*

LinkedHashSet is a common choice because it combines set uniqueness with predictable insertion-order iteration. If key-value associations are required, LinkedHashMap preserves a defined iteration order for entries.

Clarify whether re-inserting an existing element should change its position and whether access-order behavior is needed. For an LRU cache, LinkedHashMap's access-order mode may be useful, but concurrency and eviction policy still need design.

### Q22. You need fast membership tests for millions of IDs. What should you consider? *(Senior)*

A HashSet is a reasonable starting point for expected constant-time membership, assuming good hash distribution. Also consider the memory cost of object wrappers, string keys, table capacity, load factor, and whether a primitive-specialized or compact representation is justified.

If IDs are dense integers, a BitSet or boolean/bit array may be dramatically more memory-efficient. If data exceeds memory, consider an indexed database or external storage. Benchmark with realistic ID distribution and account for resize peaks.

### Q23. How do you safely remove elements while iterating? *(Intermediate)*

Use the iterator's remove method when supported, or use collection methods designed for bulk removal such as removeIf. Removing structurally through the collection while an ordinary iterator is active can trigger ConcurrentModificationException.

    Iterator<String> it = values.iterator();
    while (it.hasNext()) {
        if (it.next().trim().isEmpty()) {
            it.remove();
        }
    }

For concurrent collections, follow their documented mutation semantics; iterator.remove support differs by implementation.

### Q24. What are the main collection contracts to memorize for interviews? *(Senior)*

Understand ordering and duplicate policies, equals/hashCode or comparator-based equivalence, null support, mutation and iterator behavior, complexity expectations, concurrency guarantees, and whether returned collections are views or copies.

A good answer connects each contract to a consequence: wrong equality breaks lookup, mutable keys break hash access, unsynchronized mutation creates races, and relying on unspecified iteration order creates brittle behavior. Then choose a structure based on the workload rather than memorizing only method names.

## Official references

1. [Java SE 21 Collections Framework Overview](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/doc-files/coll-overview.html)
2. [Java SE 21 API — Collection](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Collection.html)
3. [Java SE 21 API — Map](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Map.html)
4. [Java SE 21 API — ArrayList](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayList.html)
5. [Java SE 21 API — ConcurrentHashMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

## Chapter checklist

- [ ] Select collections from semantics, not habit.
- [ ] Explain equality, ordering, views, null policy, and mutability.
- [ ] Distinguish fail-fast and weakly consistent iterators.
- [ ] Understand concurrency guarantees and compound-operation hazards.
- [ ] Evaluate memory, locality, and workload when complexity is not enough.
