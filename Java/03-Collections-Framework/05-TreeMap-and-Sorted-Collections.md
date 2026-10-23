# TreeMap and Sorted Collections

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Sorted collections are useful when order, ranges, predecessor/successor queries, and ordered traversal are part of the requirement. TreeMap and TreeSet use comparison semantics rather than hash buckets, so comparator correctness is central to their behavior.

## 1. Sorted collection contracts

### Q1. What is a SortedMap? *(Beginner)*

SortedMap is a Map whose keys are ordered according to natural ordering or a Comparator. It provides operations such as firstKey, lastKey, and range views. NavigableMap extends this contract with navigation methods such as lowerKey, floorKey, ceilingKey, and higherKey.

Use sorted maps when ordered traversal or range lookup is required. If only exact key lookup is needed, HashMap is often simpler and typically offers lower overhead.

### Q2. How does TreeMap work internally? *(Intermediate)*

OpenJDK TreeMap is based on a balanced search tree, specifically a red-black tree. It keeps keys in sorted order and supports lookup, insertion, and removal in logarithmic time. The exact internal implementation is not a requirement of the Map API, though it is the relevant OpenJDK implementation detail for interviews.

TreeMap avoids hash distribution concerns but pays comparison and tree-node overhead. Comparator cost can dominate for complex keys.

### Q3. What is the difference between lowerKey, floorKey, ceilingKey, and higherKey? *(Intermediate)*

For a query key k, lowerKey returns the greatest key strictly less than k; floorKey returns the greatest key less than or equal to k; ceilingKey returns the least key greater than or equal to k; higherKey returns the least key strictly greater than k.

These methods are useful for time ranges, nearest-prior records, and interval indexes. Their behavior depends on the comparator's ordering, not necessarily numeric intuition.

### Q4. What is a range view in TreeMap? *(Intermediate)*

Methods such as subMap, headMap, and tailMap return backed views over a key range. Changes through the view affect the original map, and changes to the backing map can be visible in the view.

Be explicit about inclusive/exclusive boundaries and whether the view should be live or copied. If a snapshot is needed, create a new map from the view.

### Q5. Why must a Comparator be consistent? *(Advanced)*

A comparator should satisfy antisymmetry/sign reversal, transitivity, and consistent comparison results for equal ordering relationships. If compare(a,b) returns zero, a sorted map/set treats the keys/elements as equivalent for its operations, even if equals says they are different.

When comparator equivalence differs from equals, a TreeMap may appear to overwrite distinct keys or a TreeSet may reject distinct values. Document the ordering and ideally keep it consistent with equals.

### Q6. What happens if a key changes after insertion into TreeMap? *(Advanced)*

If fields used by the comparator change, the key's ordering position may no longer match the tree structure. Lookups and removals can fail or behave unexpectedly because the tree follows comparisons based on the new state.

Use immutable keys or immutable ordering fields. A sorted collection assumes the ordering relation remains stable while an element is stored.

## 2. Navigation and algorithms

### Q7. When would you use TreeMap instead of sorting a list? *(Advanced)*

Use TreeMap when you need repeated ordered insertions/removals, nearest-key navigation, or range views. Sort a list when you mostly collect data and then need one ordered traversal; sorting once can be cheaper than maintaining tree order on every update.

Consider the full workload: number of writes, reads, range queries, and required memory. There is no universal winner based only on asymptotic complexity.

### Q8. What are firstEntry and pollFirstEntry? *(Intermediate)*

firstEntry returns an entry for the lowest key, while pollFirstEntry returns and removes that entry. Similar methods exist for the last entry. Returned entry mutability and snapshot details follow the API contract.

Use polling methods to consume ordered work, but remember TreeMap itself is not a concurrent priority queue. For concurrent ordered processing, choose a suitable concurrent structure or coordinate access explicitly.

### Q9. How do null keys work in TreeMap? *(Intermediate)*

Natural ordering generally cannot compare null with ordinary keys, so a TreeMap using natural ordering rejects null keys. A custom comparator may define a null ordering, depending on its implementation. Null values are allowed by TreeMap.

Do not assume that all sorted collections support null uniformly. The comparator and concrete implementation determine the behavior.

### Q10. What is the difference between TreeSet and PriorityQueue? *(Intermediate)*

TreeSet stores unique elements in sorted order and supports set membership and navigation. PriorityQueue exposes the least (or highest, with a reversed comparator) element at the head, but its iterator does not promise sorted traversal and it allows duplicate elements.

Use PriorityQueue for repeated access to the next priority item. Use TreeSet when you need uniqueness and sorted iteration or range operations.

## 3. Production scenarios

### Q11. How would you find the latest event before a timestamp? *(Senior)*

Store events keyed by a comparable timestamp or a composite key that breaks ties, then use floorEntry or lowerEntry depending on whether an event exactly at the requested time should be included. Ensure that multiple events at the same timestamp are represented without overwriting each other if required.

For large event volumes, consider database indexes or time-series storage rather than an unbounded in-memory TreeMap. Define clock source, time zone, tie-breaking, and retention policies.

### Q12. How do you represent an interval index with a sorted map? *(Senior)*

A NavigableMap can locate the nearest preceding boundary, but interval overlap and equal endpoints require a carefully defined model. A simple floorEntry lookup works only when the intervals and key structure satisfy assumptions such as non-overlap or a suitable start-key index.

For overlapping intervals, one map keyed by start time may not be sufficient. Use an interval tree, database range index, or a domain-specific structure if overlap queries are central.

### Q13. How should composite keys be ordered? *(Senior)*

Define a comparator that orders by the primary field and then deterministic tie-breakers, often with Comparator.comparing(...).thenComparing(...). Ensure that compare returns zero only when the entries should be treated as equivalent by the sorted map.

Test boundary values, null policies, and transitivity. Avoid subtraction-based integer comparators such as a.id - b.id because overflow can violate ordering; use Integer.compare or comparingInt.

## Official references

1. [Java SE 21 API — NavigableMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/NavigableMap.html)
2. [Java SE 21 API — TreeMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/TreeMap.html)
3. [Java SE 21 API — Comparator](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Comparator.html)
4. [Java SE 21 API — NavigableSet](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/NavigableSet.html)

## Chapter checklist

- [ ] Explain sorted ordering and navigation methods.
- [ ] Understand backed range views and boundary inclusivity.
- [ ] Keep comparator behavior stable and consistent.
- [ ] Choose TreeMap versus list sorting based on workload.
