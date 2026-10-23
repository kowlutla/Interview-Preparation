# Java Stream API

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Streams express data-processing pipelines using source, intermediate operations, and terminal operations. They are not collections and do not store elements themselves. Correct use depends on laziness, stateless behavior, encounter order, reduction contracts, and the difference between sequential and parallel execution.

## 1. Stream fundamentals

### Q1. What is a Stream? *(Beginner)*

A Stream is a sequence of elements supporting aggregate operations such as filter, map, sorted, reduce, and collect. A pipeline typically has a source, zero or more intermediate operations, and a terminal operation.

Streams do not generally store data and are consumed by a terminal operation. They are designed to describe computation over data sources, not to replace every loop or mutable data structure.

### Q2. What is the difference between intermediate and terminal operations? *(Beginner)*

Intermediate operations return another stream and are generally lazy, such as filter, map, and sorted. Terminal operations produce a result or side effect and trigger pipeline evaluation, such as collect, reduce, count, forEach, or findFirst.

A pipeline without a terminal operation usually does not process its elements. This is important when debugging why a lambda has not executed or why a side effect inside an intermediate operation did not happen.

### Q3. Why are streams single-use? *(Intermediate)*

A stream represents a pipeline over a source, and terminal evaluation consumes it. Reusing a stream after a terminal operation is invalid and can throw IllegalStateException, though implementations may detect reuse only in some cases.

Create a fresh stream from the source for each independent computation. If the source itself is one-shot, such as an I/O stream, its lifecycle must also be managed correctly.

### Q4. What is the difference between map and flatMap? *(Intermediate)*

map transforms each input element into one output element. flatMap transforms each input into a stream or other supported stream-like result and flattens the nested results into one stream.

    List<String> words = List.of("ab", "cd");
    List<String> letters = words.stream()
        .flatMap(word -> word.chars().mapToObj(c -> String.valueOf((char) c)))
        .toList();

Use flatMap for one-to-many transformations. Avoid using it just to make code look functional if a direct loop communicates the algorithm more clearly.

### Q5. What is laziness in the Stream API? *(Intermediate)*

Intermediate operations are lazy: they describe work that is performed when a terminal operation consumes the pipeline. The implementation may fuse operations and avoid evaluating elements that are not needed, for example when findFirst finds a match early.

Do not rely on side effects inside intermediate operations for correctness. Optimization and short-circuiting can mean that not every element or operation is evaluated.

### Q6. What is short-circuiting in streams? *(Intermediate)*

Short-circuiting operations can finish without processing all elements. Examples include anyMatch, allMatch, noneMatch, findFirst, and findAny. Limit can also bound downstream processing.

For an infinite stream, short-circuiting may make a result possible where a full traversal would never finish. The result still depends on the operation's semantics and whether the source and preceding operations can produce needed elements.

## 2. Intermediate operations

### Q7. What is the difference between filter and map? *(Beginner)*

filter keeps or discards elements based on a predicate. map transforms each element into another value. Filtering changes which elements remain; mapping changes their representation.

    List<String> result = names.stream()
        .filter(name -> !name.isBlank())
        .map(String::trim)
        .toList();

Order matters: trimming before filtering can produce a different result from filtering before trimming. Choose the order based on the intended data contract.

### Q8. What is the difference between sorted and distinct? *(Intermediate)*

sorted orders elements by natural ordering or a comparator. distinct removes duplicates according to equals. Both are stateful intermediate operations because they may need information about multiple elements before emitting the final sequence.

Stateful operations can require buffering and can reduce parallel-stream efficiency. Use them when semantics require them, and consider whether the source can provide uniqueness or ordering already.

### Q9. What is peek intended for? *(Intermediate)*

peek is an intermediate operation that applies an action to elements as they are consumed, often for debugging. Because streams are lazy and operations can short-circuit or be optimized, peek is not a reliable mechanism for required business side effects.

For business actions, use an explicit terminal operation or collect results and perform the action in a separate stage. Logging through peek can be useful during diagnosis but should not be essential to correctness.

### Q10. What is the difference between limit and skip? *(Intermediate)*

limit(n) restricts the pipeline to at most n elements; skip(n) discards the first n elements in encounter order where that order is defined. They can be useful for basic pagination over an in-memory source, but they are not a substitute for database pagination for large datasets.

Negative values are invalid. On ordered parallel streams, skip and limit may require coordination and can be expensive. For large result sets, use a data source that supports paging or cursor-based traversal.

### Q11. What is the difference between findFirst and findAny? *(Advanced)*

findFirst respects encounter order when the stream has one. findAny allows any element and can provide more flexibility for parallel execution. Both return Optional and are short-circuiting terminal operations.

If the requirement is “the first item according to business order,” ensure the source has a defined encounter order or sort it explicitly. Do not use findAny when deterministic selection is part of correctness.

## 3. Reduction and collection

### Q12. What is reduce? *(Intermediate)*

reduce combines stream elements into a single result using an associative accumulation operation, with overloads for identity and accumulator and for a combiner. It is suitable for reductions such as sum, product, or combining immutable values.

For parallel execution, the operation must obey the required associativity and identity properties. A subtraction-based reduction, for example, is not associative and can produce different results when partitioned.

### Q13. Why must a parallel reduction be associative? *(Advanced)*

Parallel execution can partition elements, compute partial results, and combine those results in a different grouping. Associativity means grouping does not change the result: (a op b) op c equals a op (b op c).

Floating-point addition is not mathematically associative due to rounding, so parallel and sequential sums can differ slightly. For exact financial calculations, choose numeric representations and reduction policies that satisfy the required correctness constraints.

### Q14. What is the difference between reduce and collect? *(Advanced)*

reduce is intended to combine values into a single result using an associative operation. collect is a mutable reduction that accumulates elements into a result container using a supplier, accumulator, and combiner, or a Collector.

Do not use reduce with a shared mutable identity such as a mutable list. Use collect for building collections, maps, grouping, or other mutable result containers.

### Q15. What does Collectors.groupingBy do? *(Intermediate)*

groupingBy classifies elements by a key and accumulates elements into groups, commonly returning a Map<K, List<T>>.

    Map<String, List<Employee>> byTeam = employees.stream()
        .collect(Collectors.groupingBy(Employee::team));

The default map implementation and result details follow the collector contract, not an assumption of stable order. If order or a specific map type matters, provide a map factory and downstream collector explicitly.

### Q16. What does partitioningBy do? *(Intermediate)*

partitioningBy groups elements into two Boolean-keyed groups based on a predicate. Unlike groupingBy, it always models the true/false partition and can return both groups, including an empty group.

Use it when the domain is inherently binary. For more than two categories, groupingBy is generally clearer.

### Q17. What is a downstream collector? *(Advanced)*

A downstream collector performs an additional reduction within each group produced by groupingBy or partitioningBy. Examples include counting elements per key, summing a numeric property, mapping values, or collecting each group into a set.

    Map<String, Long> countByTeam = employees.stream()
        .collect(Collectors.groupingBy(Employee::team, Collectors.counting()));

Downstream collectors avoid creating intermediate lists when only an aggregate is required. Ensure the chosen collector matches ordering, mutability, and null requirements.

### Q18. What is the difference between toList and Collectors.toList? *(Advanced)*

Stream.toList was added in Java 16 and returns an unmodifiable list. Collectors.toList does not guarantee a particular implementation type, mutability, serializability, or thread safety. Do not assume either operation returns an ArrayList.

If a mutable ArrayList is required, use Collectors.toCollection(ArrayList::new). If an unmodifiable result is required, choose an API whose contract guarantees it.

## 4. Parallel streams and correctness

### Q19. When should you use parallelStream? *(Advanced)*

Parallel streams can help CPU-bound, sufficiently large workloads with splittable sources and low coordination cost. They can hurt when tasks are small, work is blocking, the source splits poorly, shared state is contended, or ordering constraints are expensive.

Benchmark with realistic data and production-like CPU limits. Parallelism is not a free performance switch and may use shared ForkJoinPool resources that interact with other application tasks.

### Q20. Why are side effects dangerous in parallel streams? *(Advanced)*

Parallel operations may execute on multiple threads and in an unspecified order. Mutating a shared ArrayList or counter without proper synchronization creates data races, while synchronized mutation can serialize the workload and erase the benefit.

Prefer collectors and stateless transformations. If external side effects are necessary, define ordering, failure, cancellation, and idempotency semantics explicitly.

### Q21. What is the difference between forEach and forEachOrdered? *(Advanced)*

forEach does not guarantee encounter order for parallel streams. forEachOrdered preserves encounter order where one exists, potentially introducing coordination and reducing parallel throughput.

If output order is a requirement, make it explicit. For concurrent logging or external side effects, order and exactly-once behavior may require a separate design beyond selecting forEachOrdered.

### Q22. Why can a stream pipeline run slower than a loop? *(Senior)*

Streams add abstraction and can incur lambda, pipeline, boxing, allocation, and dispatch costs. For primitive data, using Stream<Integer> can box values, whereas IntStream avoids much of that overhead. The JIT may optimize many patterns, so results depend on the workload.

Use streams when they clarify transformations and composition. For a hot path, benchmark the real operation with JMH and compare equivalent implementations; avoid optimizing based on tiny ad hoc tests.

### Q23. How do primitive streams help? *(Intermediate)*

IntStream, LongStream, and DoubleStream provide primitive-specialized operations and avoid boxing each value into Integer, Long, or Double for many pipeline steps. They include numeric operations such as sum, average, and range construction where applicable.

Convert with mapToInt/mapToLong/mapToDouble when numeric processing dominates. Be aware that conversion back to object streams can reintroduce boxing.

### Q24. How should you handle exceptions inside a stream pipeline? *(Advanced)*

Standard functional interfaces generally do not declare checked exceptions, so checked exceptions must be handled, translated, or modeled by a custom functional interface. Do not catch exceptions and silently drop elements unless that is the defined business policy.

For pipelines that combine I/O with transformation, a conventional loop may be clearer because it makes resource lifetime and error handling explicit. Streams are a tool, not a requirement for every collection operation.

## 5. Senior-level design

### Q25. Why should a stream source not be modified during pipeline execution? *(Advanced)*

Modifying a non-concurrent source while it is being traversed can violate the source's behavioral assumptions, produce unexpected results, or trigger ConcurrentModificationException. Stream behavioral parameters should be non-interfering with the source.

Create a separate result or use a supported concurrent source with its documented semantics. Do not add/remove elements from the same collection inside its own stream pipeline.

### Q26. What is non-interference? *(Advanced)*

Non-interference means the pipeline's behavioral parameters do not modify the stream source in ways that invalidate traversal. It supports predictable execution and allows implementations to optimize pipelines.

If a transformation must change the source, separate the phases: compute a result, then apply updates. This is often easier to test and safer than mutating the source from inside a lambda.

### Q27. How do you debug a stream that produces an unexpected result? *(Senior)*

Reduce the pipeline to stages, inspect the source and encounter order, check lazy execution and short-circuiting, and verify equals/comparator behavior. Look for side effects, accidental boxing/conversion, nulls, and collector semantics. Use a terminal operation to trigger execution.

A small equivalent loop can help reveal the intended algorithm. Avoid leaving required behavior hidden inside peek or relying on the execution order of a parallel pipeline.

### Q28. How do you decide between a stream and a loop? *(Senior)*

Use a stream when the operation is a clear sequence of transformations and reductions, especially when standard collectors express the result concisely. Use a loop when complex control flow, checked exceptions, early exits, stateful algorithms, or detailed debugging make it clearer.

Readability and correctness outrank style preference. A senior engineer should be comfortable reviewing both and should measure performance only where the workload justifies it.

## Official references

1. [Java SE 21 API — Stream](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Stream.html)
2. [Java SE 21 API — Collectors](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Collectors.html)
3. [Java SE 21 API — IntStream](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/IntStream.html)
4. [Java SE 21 API — Spliterator](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Spliterator.html)
5. [Java SE 21 API — java.util.stream package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)

## Chapter checklist

- [ ] Explain laziness, short-circuiting, and single-use pipelines.
- [ ] Distinguish map, flatMap, reduce, collect, and downstream collectors.
- [ ] Understand associativity and side-effect hazards in parallel streams.
- [ ] Use primitive streams when they improve clarity and avoid unnecessary boxing.
- [ ] Choose streams or loops based on clarity, correctness, and measured performance.

## Worked examples: stream contracts and pitfalls

### Example 1. Avoid side effects in intermediate operations

\`\`\`java
List<Integer> seen = new ArrayList<>();
long count = Stream.of(1, 2, 3)
        .peek(seen::add)
        .count();
\`\`\`

Do not rely on `seen` being populated as a required business effect. Implementations can sometimes compute a terminal result directly when the source size is known and intermediate operations do not change it, so traversal side effects may be elided. Use an explicit terminal operation or a normal loop when side effects are part of the requirement.

### Example 2. Reduce values with an associative operation

\`\`\`java
int sum = Stream.of(1, 2, 3, 4)
        .parallel()
        .reduce(0, Integer::sum);
System.out.println(sum); // 10
\`\`\`

Addition of these integers is associative under Java's defined `int` arithmetic (including wraparound), and zero is an identity. A subtraction reduction is not associative, so parallel partitioning can change the result. For mutable containers, prefer `collect` rather than using `reduce` with a shared mutable identity.

### Example 3. Make list mutability explicit

\`\`\`java
List<String> result = Stream.of("a", "b").toList(); // unmodifiable, Java 16+
List<String> mutable = Stream.of("a", "b")
        .collect(Collectors.toCollection(ArrayList::new));
\`\`\`

Do not assume `Collectors.toList()` promises an `ArrayList` or a specific mutability contract. Choose `toList()` when an unmodifiable result is desired on Java 16+, or `toCollection(ArrayList::new)` when the contract requires a mutable `ArrayList`.

### Example 4. Keep parallel side effects out of shared mutable state

\`\`\`java
List<Integer> values = IntStream.range(0, 1_000)
        .boxed()
        .collect(Collectors.toList());
\`\`\`

A collector expresses the accumulation to the stream implementation. Replacing this with `parallel().forEach(sharedList::add)` on an ordinary `ArrayList` is unsafe because concurrent mutations race. Use a proper collector or a concurrent data structure only when its semantics match the requirement.
