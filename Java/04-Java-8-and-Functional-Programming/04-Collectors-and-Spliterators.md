# Collectors and Spliterators

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Collectors describe reductions from stream elements into mutable or immutable result structures. Spliterators describe traversal and partitioning characteristics, especially for parallel processing. Correctness depends on the collector's identity/accumulator/combiner contracts and accurate source characteristics.

## 1. Collector fundamentals

### Q1. What is a Collector? *(Intermediate)*

Collector describes how to accumulate stream elements into a result container using a supplier, accumulator, combiner, and optional finisher. Standard collectors cover lists, sets, maps, grouping, partitioning, and statistics.

Collectors allow the stream implementation to manage reduction safely. Do not mutate a shared external collection from a parallel forEach when a collector expresses the result directly.

### Q2. What are supplier, accumulator, combiner, and finisher? *(Advanced)*

The supplier creates a result container; the accumulator incorporates one element; the combiner merges partial results; and the finisher converts the intermediate container to the final result if needed.

Parallel collection depends on these operations obeying the collector contract. A combiner that drops data or merges incompatible containers can produce results that differ between sequential and parallel execution.

### Q3. What is the difference between groupingBy and groupingByConcurrent? *(Advanced)*

groupingBy collects elements into groups, commonly a Map<K,List<T>>. groupingByConcurrent uses a concurrent map and is intended for concurrent reduction where its characteristics and downstream collector permit it.

Concurrent grouping does not promise ordered grouping in the same way as an ordered sequential pipeline. Use it when concurrency helps and order is not required; benchmark the contention and merge costs.

### Q4. What does Collectors.toMap do when duplicate keys occur? *(Intermediate)*

The two-argument toMap collector throws IllegalStateException if the mapped keys collide. An overload accepts a merge function that defines how duplicates are combined, and another overload allows a map factory.

Make duplicate policy explicit. Choosing first or last silently may hide data-quality problems, so validate the domain expectation.

### Q5. What is a downstream collector? *(Intermediate)*

A downstream collector performs a second reduction inside each group created by groupingBy or partitioningBy. Examples include counting, summing, mapping, filtering with Collectors.filtering (Java 9+), and collecting each group into a set.

Use downstream collectors to avoid building full lists when only an aggregate is needed. Ensure the chosen collector matches the required null, ordering, and mutability semantics.

### Q6. What is Collectors.teeing? *(Advanced)*

teeing was added in Java 12. It sends each input element to two downstream collectors and combines their results with a merger. It is useful when one traversal should compute two related aggregates.

If targeting Java 8 or 11, this collector is unavailable. Consider whether two clear passes are simpler than introducing compatibility code.

### Q7. What are collector characteristics? *(Advanced)*

Characteristics include CONCURRENT, UNORDERED, and IDENTITY_FINISH. They describe properties the stream framework may use to choose a reduction strategy. They must be declared accurately.

Incorrectly claiming concurrent or unordered behavior can cause invalid assumptions or incorrect results. Custom collectors should be tested with sequential and parallel streams and with empty/singleton inputs.

### Q8. Why should mutable reduction not use reduce with a shared list? *(Advanced)*

reduce is designed around a reduction operation with identity and associativity requirements. Using a mutable list as a shared identity can violate the reduction contract, especially under parallel execution, because the same object may be reused or combined unexpectedly.

Use collect with a supplier, accumulator, and combiner for mutable containers. Better still, use a standard collector when it matches the requirement.

## 2. Spliterator

### Q9. What is a Spliterator? *(Intermediate)*

Spliterator combines element traversal with the ability to split a source into partitions. It is used by streams to support sequential and parallel processing and can report characteristics such as ORDERED, SIZED, SORTED, DISTINCT, and CONCURRENT.

The split strategy affects parallel performance. Array-backed sources often split efficiently; linked or I/O-backed sources may be less balanced.

### Q10. What does trySplit do? *(Advanced)*

trySplit attempts to return a Spliterator covering a portion of the remaining elements while the original retains another portion. A good split divides work with low overhead and balanced sizes.

Custom spliterators must ensure partitions do not overlap or omit elements. Test repeated splitting and compare sequential and parallel results.

### Q11. What does SIZED mean? *(Advanced)*

SIZED indicates that the exact number of remaining elements is known according to the Spliterator contract. SUBSIZED indicates that splits also have exact sizes. These characteristics can enable optimizations such as efficient sizing and partitioning.

Do not claim exact size if the source can change unpredictably. Concurrent sources need characteristics consistent with their actual traversal semantics.

### Q12. What does ORDERED mean? *(Intermediate)*

ORDERED means traversal has a defined encounter order. Operations such as findFirst and forEachOrdered can rely on that order. HashSet does not guarantee order, while List generally does.

Do not infer order from one run's output. If deterministic results matter, use a source with a defined order or sort explicitly.

## 3. Senior-level reasoning

### Q13. When is a custom collector justified? *(Senior)*

Use a custom collector when a domain-specific reduction cannot be expressed clearly with standard collectors, and its supplier/accumulator/combiner contract can be specified and tested. Avoid custom collectors for simple grouping/counting that standard APIs already handle.

Test empty input, one element, multiple partitions, sequential and parallel execution, and accumulator combination order where relevant.

### Q14. How do you test a collector's parallel correctness? *(Senior)*

Compare sequential and parallel results across varied inputs, including empty data, duplicates, skewed keys, and multiple partition shapes. Verify identity and associativity properties and ensure mutable containers are not shared incorrectly.

A collector may pass simple tests and fail only when partial results combine. Property-based tests and randomized partitioning can help expose errors.

### Q15. How do you choose between a loop and a collector? *(Senior)*

Use a collector when it expresses grouping or aggregation clearly. Use a loop when state transitions, validation, exceptions, or control flow are complex enough that a collector obscures the algorithm.

The API style is secondary to correctness and maintainability. Benchmark only when performance is material and compare equivalent semantics.

## Official references

1. [Java SE 21 API — Collector](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Collector.html)
2. [Java SE 21 API — Collectors](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Collectors.html)
3. [Java SE 21 API — Spliterator](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Spliterator.html)
4. [Java SE 21 API — Stream](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Stream.html)

## Chapter checklist

- [ ] Explain collector reduction roles and characteristics.
- [ ] Choose duplicate-key policy explicitly for toMap.
- [ ] Use downstream and concurrent collectors appropriately.
- [ ] Understand Spliterator characteristics and split quality.
- [ ] Test custom collectors under parallel reduction.
