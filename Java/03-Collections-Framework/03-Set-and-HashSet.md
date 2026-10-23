# Set and HashSet

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Set represents uniqueness, but implementations differ in how they define equivalence, preserve order, and achieve membership performance. HashSet relies on hashCode and equals; TreeSet relies on ordering; LinkedHashSet maintains a predictable iteration order.

## 1. Set contract

### Q1. What does Set guarantee? *(Beginner)*

A Set contains no duplicate elements according to its equality or ordering semantics. It does not necessarily guarantee a particular iteration order, and implementations can differ in null support, performance, and concurrency behavior.

The word duplicate depends on the implementation: HashSet uses equals/hashCode, while TreeSet uses comparator or natural-order equivalence. If those definitions disagree, the set can behave in ways that surprise callers.

### Q2. How does HashSet work internally? *(Intermediate)*

In OpenJDK, HashSet is implemented on top of a HashMap, storing set elements as keys and using a shared dummy value. The map's hashing and collision-handling behavior therefore drives set membership and performance.

The exact internal implementation is not a general API guarantee. For interviews, understand the hash/equality contract, expected constant-time membership under suitable hash distribution, and the effects of collisions and resizing.

### Q3. Why must equals and hashCode be consistent for HashSet? *(Intermediate)*

HashSet uses hashCode to find a candidate bucket and equals to detect an existing equal element. Equal values with different hash codes may land in different buckets and both be stored, violating the logical uniqueness expected by callers.

Implement both methods using the same equality-relevant state. Test with two distinct but equal instances rather than only testing insertion of the same reference twice.

### Q4. What happens when an element is mutated after insertion into HashSet? *(Advanced)*

If fields used by hashCode or equals change, contains and remove may search a different bucket or fail to recognize the element. The element can remain physically stored but be difficult to locate through normal operations.

Prefer immutable set elements or immutable equality keys. Removing before mutation and reinserting afterward is fragile and should not replace a sound model where equality-relevant state is stable.

### Q5. What is the difference between HashSet and LinkedHashSet? *(Intermediate)*

HashSet makes no iteration-order guarantee. LinkedHashSet maintains a defined encounter order, normally insertion order, using additional linkage and memory. Both use hashing and equality for uniqueness.

Use LinkedHashSet when predictable iteration order is part of the contract. Do not assume insertion order is free; the additional structure has overhead.

### Q6. What is the difference between HashSet and TreeSet? *(Intermediate)*

HashSet uses hashCode and equals and typically provides expected constant-time add, remove, and contains. TreeSet maintains sorted order and typically provides logarithmic-time operations using a comparator or natural ordering.

TreeSet's comparator defines equivalence for set operations. If compare(a,b) returns zero for values that are not equal, TreeSet may treat them as duplicates, so comparator consistency matters.

### Q7. Can a Set contain null? *(Intermediate)*

It depends on the implementation. HashSet permits one null element; TreeSet support depends on its ordering and comparator; Set.of rejects null elements. Concurrent set implementations based on ConcurrentHashMap do not permit null.

Null support is part of the concrete API contract. Avoid relying on it implicitly when switching implementations or using immutable factory methods.

## 2. Set operations and ordering

### Q8. How do union, intersection, and difference work with sets? *(Beginner)*

Union contains elements from either set, intersection contains elements common to both, and difference contains elements in one set but not the other. The Set API offers methods such as addAll, retainAll, and removeAll to perform these operations by mutating a target set.

Choose whether the operation should mutate an input or produce a new set. Defensive copies are useful when callers expect original inputs to remain unchanged.

### Q9. What is a comparator's role in TreeSet? *(Advanced)*

TreeSet uses a Comparator or natural ordering to place elements and determine whether they compare as equivalent. A comparator returning zero signals equivalence for the sorted set even if equals returns false.

Ideally, comparator ordering is consistent with equals when the set is expected to obey the general Set contract. If not, document the semantics and ensure callers understand that values may be treated as duplicates based on ordering.

### Q10. What does Set.of provide? *(Intermediate)*

Set.of was introduced in Java 9. It creates an unmodifiable set and rejects null elements and duplicate arguments at creation time. It is useful for fixed constant sets.

Do not assume the iteration order of these factory-created sets. If order matters, use an implementation with a defined ordering contract. For a mutable set, create a HashSet or another appropriate copy.

### Q11. How do you preserve order and uniqueness while transforming a stream? *(Intermediate)*

A LinkedHashSet can collect unique values while preserving encounter order when the source and collection semantics support that expectation. Stream distinct also removes duplicates according to equals while preserving encounter order for ordered streams.

Be explicit about whether order matters. Parallel stream processing and unordered sources can affect encounter-order assumptions, so do not rely on incidental order from a HashSet source.

## 3. Production scenarios

### Q12. Why can a HashSet appear to contain duplicates? *(Senior)*

If equality and hashing are inconsistent, objects that should be logically equal may be placed in different buckets. If equality-relevant state mutates after insertion, the set's current lookup behavior may no longer match the state at insertion. Another possibility is that the domain's intended uniqueness differs from Java equals.

Inspect equals/hashCode, mutability, and the domain key. For uniqueness by a business identifier, a dedicated immutable key or map keyed by that identifier may be clearer than overriding equality across a complex entity.

### Q13. How would you choose a Set for a permission system? *(Senior)*

If permissions are identifiers and membership tests dominate, HashSet is a common choice. If deterministic display order is needed, LinkedHashSet or a sorted set may be appropriate. If permission objects are mutable, prefer immutable permission codes or stable equality keys.

For concurrent authorization reads, publish an immutable snapshot or use a carefully chosen concurrent design. Authorization correctness must not depend on a mutable collection being observed at a lucky moment.

### Q14. How can a poor hash function affect HashSet? *(Advanced)*

Many collisions concentrate elements into fewer buckets, increasing comparison work and degrading expected constant-time behavior. Modern JDK implementations have collision-handling strategies, but poor hash distribution still consumes CPU and may create adversarial performance risks.

Implement hashCode to distribute realistic values well, but do not attempt to hand-optimize without measurement. For untrusted keys, consider whether attacker-controlled collisions can create denial-of-service risk and use platform/library safeguards.

### Q15. How do you test Set equality and uniqueness correctly? *(Senior)*

Test that adding the same logical value twice leaves one element, that equal objects have equal hash codes, that lookup succeeds using a distinct equal instance, and that removal works. Also test ordering behavior if using TreeSet and null behavior if relevant.

Use property-based tests for equality laws where practical. Test the actual business key semantics rather than only comparing the same reference.

## Official references

1. [Java SE 21 API — Set](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Set.html)
2. [Java SE 21 API — HashSet](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashSet.html)
3. [Java SE 21 API — TreeSet](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/TreeSet.html)
4. [Java SE 21 API — LinkedHashSet](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedHashSet.html)

## Chapter checklist

- [ ] Explain HashSet's hash/equality contract.
- [ ] Distinguish HashSet, LinkedHashSet, and TreeSet.
- [ ] Understand comparator equivalence and mutable-key hazards.
- [ ] Choose null and iteration-order semantics deliberately.
