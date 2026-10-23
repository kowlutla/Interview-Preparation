# Iterators, Spliterators and Collection Internals

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Iteration contracts determine whether data can be safely traversed while changes occur, how streams split work, and whether order is preserved. Understanding these contracts prevents common ConcurrentModificationException bugs and helps explain the behavior of parallel streams and concurrent collections.

## 1. Iterators

### Q1. What is an Iterator? *(Beginner)*

Iterator provides sequential traversal through hasNext and next, and may support removal through remove depending on the implementation. It abstracts the collection's storage representation so callers do not need to know whether the data is array-backed, linked, hashed, or tree-based.

The iterator's contract defines which modifications are supported during traversal. Do not assume every iterator supports remove or that it is safe to use concurrently.

### Q2. What is the difference between Iterator and ListIterator? *(Intermediate)*

Iterator supports forward traversal and optional removal. ListIterator is specific to lists and supports bidirectional traversal, index queries, and optional add/set operations.

Use ListIterator when an algorithm genuinely needs to traverse or mutate a list around a cursor. For simple traversal, Iterator or enhanced for is easier to read.

### Q3. What is fail-fast behavior? *(Intermediate)*

Some iterators detect structural modifications outside the iterator and throw ConcurrentModificationException. This is intended to detect programming mistakes, not to provide synchronization or guarantee detection of every race.

Fail-fast behavior is best effort. Code must remain correct even if an invalid concurrent modification is not detected, so never rely on the exception as a locking mechanism.

### Q4. What is weakly consistent iteration? *(Advanced)*

Weakly consistent iterators are designed to tolerate concurrent changes and may reflect some updates without throwing ConcurrentModificationException. Their exact visibility behavior depends on the concurrent collection.

They are not necessarily snapshots and do not guarantee every element is observed exactly once under all concurrent modifications. Use explicit snapshot or synchronization when the algorithm requires a point-in-time view.

### Q5. What is a snapshot iterator? *(Advanced)*

A snapshot iterator traverses a stable copy or version of a collection rather than the live structure. CopyOnWriteArrayList iterators, for example, observe the array snapshot that existed when the iterator was created and do not reflect later mutations.

Snapshot semantics simplify traversal but can consume memory and make updates expensive. Choose them when reads greatly outnumber writes and a stable traversal is valuable.

## 2. Spliterator and streams

### Q6. What is Spliterator? *(Advanced)*

Spliterator combines traversal with the ability to split a source into parts, supporting efficient sequential or parallel processing. It can report characteristics such as ORDERED, SIZED, SORTED, DISTINCT, and CONCURRENT.

Characteristics are promises that help the stream implementation optimize processing. A custom Spliterator must report them accurately; false characteristics can cause incorrect assumptions or results.

### Q7. How does Spliterator support parallel streams? *(Advanced)*

The stream framework repeatedly splits the source into subranges or partitions, processes them in parallel, then combines results. Splitting quality and balance strongly affect performance.

Array-backed sources often split efficiently, while linked or I/O-bound sources may be less effective. Parallelism can be slower if split costs, coordination, or combination work exceed the useful computation.

### Q8. What does ORDERED mean for a Spliterator? *(Intermediate)*

ORDERED means traversal has a defined encounter order. Stream operations such as findFirst and forEachOrdered can use that order. A collection such as HashSet does not promise encounter order, while List generally does.

Do not infer ORDERED from observed output in a test. If deterministic order matters, use an ordered source or explicitly sort the elements.

### Q9. What does CONCURRENT mean in Spliterator characteristics? *(Advanced)*

CONCURRENT indicates that the source can be safely modified concurrently under the source's documented semantics during traversal. It does not mean every operation on the source is atomic or that the traversal is a snapshot.

Only report this characteristic when the source truly supports it. For ordinary ArrayList or HashMap, concurrent structural modification remains unsafe.

## 3. Mutation and view semantics

### Q10. Why can modifying a collection during enhanced for fail? *(Beginner)*

Enhanced for uses an iterator or equivalent array traversal. Removing structurally from the collection directly while an iterator is active can invalidate the iterator and trigger ConcurrentModificationException.

Use Iterator.remove where supported or collection operations such as removeIf. For concurrent data, select an appropriate concurrent collection and understand its iterator semantics.

### Q11. What is the difference between a live view and a snapshot? *(Intermediate)*

A live view reflects changes to the backing collection and often allows supported changes to propagate back. A snapshot captures a state at a point in time and does not reflect later mutations. subList and Map views are examples of backed views; copy constructors often produce independent structure copies.

A snapshot is generally shallow unless elements are also copied. Decide whether the caller needs a stable membership view or independent element state.

### Q12. How do unmodifiable wrappers affect iterators? *(Advanced)*

An unmodifiable wrapper rejects mutation through that wrapper, including iterator removal. But if the backing collection is mutable and reachable elsewhere, its changes can still be visible through the wrapper. Iteration behavior may still be affected by backing modifications.

For a stable unmodifiable result, use a copy or an API that returns an unmodifiable snapshot, and account for shallow element mutability. Unmodifiable is not the same as deeply immutable or thread-safe.

### Q13. What is CopyOnWriteArrayList and when is it appropriate? *(Advanced)*

CopyOnWriteArrayList creates a new backing array for mutations, while readers and iterators can operate on stable snapshots. It is suitable for read-heavy collections with relatively infrequent updates, such as listener registries.

Writes can be expensive in memory and copying time, so it is a poor choice for frequent mutation or very large lists. Snapshot iterators do not support mutation and do not see changes made after iterator creation.

## 4. Senior-level diagnosis

### Q14. How do you debug ConcurrentModificationException? *(Senior)*

Find the iterator and identify structural modifications that occur outside its supported methods. The modification may be in a helper method, callback, listener, or nested loop rather than on the visible line that throws.

Replace mutation with iterator.remove, collect changes for a second phase, or use a collection with suitable concurrent semantics. Do not “fix” the exception by catching it and retrying without understanding the invalidated traversal.

### Q15. How would you implement a custom Spliterator? *(Senior)*

Define the source's traversal rules, splitting strategy, estimateSize, and characteristics. Ensure tryAdvance advances at most one element and returns false only when exhausted; trySplit should return a valid disjoint partition when possible.

Test sequential traversal, repeated splitting, ordering, size estimates, and parallel stream results. Report only characteristics that are guaranteed, and benchmark whether the splitting strategy improves the workload.

### Q16. Why can parallel streams behave unexpectedly with a mutable source? *(Senior)*

A non-concurrent source must not be structurally modified during pipeline execution. Concurrent modification can cause exceptions, inconsistent results, or undefined behavior under the source contract. Even a concurrent source may provide weakly consistent rather than snapshot semantics.

Create a stable input snapshot or use a source designed for concurrent traversal. Separate data acquisition and mutation from the computation pipeline when possible.

## Official references

1. [Java SE 21 API — Iterator](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Iterator.html)
2. [Java SE 21 API — ListIterator](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ListIterator.html)
3. [Java SE 21 API — Spliterator](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Spliterator.html)
4. [Java SE 21 API — CopyOnWriteArrayList](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CopyOnWriteArrayList.html)

## Chapter checklist

- [ ] Distinguish fail-fast, weakly consistent, and snapshot iterators.
- [ ] Understand iterator mutation contracts.
- [ ] Explain Spliterator characteristics and parallel splitting.
- [ ] Choose live views, copies, and unmodifiable wrappers deliberately.
