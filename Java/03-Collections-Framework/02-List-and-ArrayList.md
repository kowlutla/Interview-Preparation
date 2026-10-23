# List and ArrayList

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

List models an ordered sequence with index-based access and permits duplicates. ArrayList is the common resizable-array implementation. Interview questions often focus on capacity growth, array copying, iterator behavior, subList views, equality, and the difference between fixed-size and mutable lists.

## 1. List contract

### Q1. What guarantees does List provide? *(Beginner)*

List is an ordered collection whose elements have positions indexed from zero. It permits duplicates and usually supports positional access, insertion, replacement, and removal, subject to the implementation's optional-operation support.

The interface does not guarantee a particular backing data structure or complexity. ArrayList, LinkedList, and specialized immutable or unmodifiable lists can all implement List with different operational trade-offs.

### Q2. What is the difference between size and capacity in ArrayList? *(Intermediate)*

Size is the number of elements currently stored. Capacity is the length of the internal backing array available before a growth operation is needed. Capacity is an implementation detail and is not exposed as the List size.

An ArrayList may reserve more storage than it currently uses to make appends efficient. Clearing the list removes elements but does not necessarily shrink its backing storage. If memory retention matters, choose a deliberate lifecycle or create a compact copy when appropriate.

### Q3. How does ArrayList append achieve amortized constant time? *(Advanced)*

Appending is constant-time while spare capacity exists. When capacity is exhausted, the implementation allocates a larger backing array and copies existing references, making that particular append linear. With a geometric growth strategy, the total copying over many appends is linear, so average cost per append is amortized constant time.

Exact growth factors are implementation details and can change between JDK versions. Do not write application logic that assumes a specific capacity multiplier.

### Q4. Why is insertion at index zero expensive in ArrayList? *(Intermediate)*

ArrayList uses a contiguous backing array. Inserting near the beginning requires shifting existing element references to make room, so the work is linear in the number of elements after the insertion point. Removal from the middle similarly shifts following elements.

If frequent insertion/removal at both ends is central, consider ArrayDeque. If frequent indexed reads dominate, ArrayList is usually a better fit than LinkedList.

### Q5. Does ArrayList store objects directly inside its backing array? *(Intermediate)*

For reference elements, the backing array stores references, not inline copies of the objects. The objects may live elsewhere in the heap, and the array holds references to them. Primitive values are boxed when stored in a generic List such as List<Integer>.

This affects memory footprint and locality. Large collections of boxed numbers can use substantially more memory than primitive arrays or specialized primitive collections.

### Q6. Is ArrayList thread-safe? *(Beginner)*

No. Concurrent structural mutation without coordination can race and produce incorrect behavior. Concurrent reads can be safe only when the list is safely published and no thread mutates it concurrently in an unsafe way.

Use synchronization, a suitable concurrent structure, or immutable snapshots. Collections.synchronizedList requires external synchronization during iteration as documented; merely wrapping the list does not make an entire multi-step workflow atomic.

## 2. Iteration and modification

### Q7. What is a structural modification? *(Advanced)*

A structural modification changes the size of a collection or otherwise changes its structure as defined by the implementation. Replacing an element with set generally is not structural for ArrayList, while add and remove are.

ArrayList iterators may use a modification count to detect some unsupported structural changes. The fail-fast behavior is best effort and must not be used as a correctness or concurrency guarantee.

### Q8. Why does ConcurrentModificationException occur? *(Intermediate)*

An ArrayList iterator can detect a structural modification made outside the iterator's supported mutation methods and throw ConcurrentModificationException. This commonly happens when code removes from the list directly inside an enhanced for loop.

Use Iterator.remove or removeIf for safe local mutation. The exception does not necessarily mean multiple threads are involved; it can occur in one thread.

### Q9. Is it safe to call set while iterating an ArrayList? *(Advanced)*

Replacing an existing element with set is not a structural modification for ArrayList, so it normally does not invalidate its iterator. However, changing an element can still affect algorithmic assumptions or shared mutable state. The contract differs by collection implementation.

Distinguish replacing a value from changing the collection's structure. When code needs to mutate elements, use an explicit index loop or iterator and document the intended semantics.

### Q10. What does subList return? *(Advanced)*

subList returns a view backed by the original list over a range, not an independent copy. Changes through the view affect the backing list and vice versa. Structural modifications to the backing list outside the view can make the view's behavior undefined by the view's documented contract or trigger fail-fast exceptions.

If an independent list is needed, create a copy such as new ArrayList<>(list.subList(from, to)). Also be aware that retaining a small subList view can retain the backing list's larger storage.

### Q11. Why can retaining a subList cause unexpected memory retention? *(Advanced)*

Because subList is a view, it can retain a reference to the backing list and its underlying storage even if the view contains only a few elements. A long-lived small view may therefore keep a much larger data structure reachable.

Create a copy when the smaller subset must outlive the original collection. Diagnose suspected retention with heap dumps and object-reference paths rather than assuming the visible list size equals retained memory.

### Q12. What is the difference between Arrays.asList and new ArrayList? *(Intermediate)*

Arrays.asList returns a fixed-size list backed by the given array. It supports get and set but not structural add/remove. new ArrayList<>(source) creates a separately resizable list containing the source elements.

The backing relationship can surprise callers: changing the array changes the Arrays.asList view. Choose an explicit copy when ownership and independent mutation are required.

## 3. List equality, sorting and algorithms

### Q13. How does List.equals work? *(Intermediate)*

List equality is sequence-based: two lists are equal when they have the same size and corresponding elements are equal in order. The implementation class does not need to be the same, so an ArrayList and LinkedList can be equal when their contents match.

This means order matters for List equality, unlike the general set equality model. Element equals implementations therefore influence list equality.

### Q14. What is the difference between Collections.sort and List.sort? *(Intermediate)*

Collections.sort is a utility method that sorts a list; List.sort is a default method on List. Both use natural ordering or a supplied Comparator, with stable sorting specified by the APIs for object lists.

Choose a comparator that is consistent with the desired ordering and handles nulls deliberately. If the comparator is inconsistent or violates its contract, sorting can produce unexpected results or throw IllegalArgumentException in some circumstances.

### Q15. What is a stable sort, and why does it matter? *(Intermediate)*

A stable sort preserves the relative order of elements that compare as equal under the comparator. This matters when sorting records by one key after they have already been ordered by another key.

For example, sorting employees by department can preserve their earlier name order within each department if the sort is stable. If multiple ordering criteria are intended, an explicit chained comparator is often clearer than relying on previous sorting passes.

### Q16. How do you avoid accidental aliasing when returning a list? *(Advanced)*

If a method returns its mutable internal list directly, callers can mutate internal state without going through validation. Return an unmodifiable view or a defensive copy depending on the desired contract.

An unmodifiable view still reflects changes made through the original backing reference. List.copyOf creates an unmodifiable snapshot of the element references and rejects nulls, but it does not deeply copy mutable elements. Document ownership and mutation semantics.

### Q17. How should List be used in API design? *(Senior)*

Expose List when order and duplicates are meaningful, and document whether the result is mutable, unmodifiable, a live view, or a snapshot. Accept the interface when callers can supply different implementations, but do not promise random-access performance unless the API contract needs it.

Avoid leaking implementation assumptions such as ArrayList capacity. For large data, consider pagination, streaming, or iterators rather than returning an unbounded in-memory list.

### Q18. How would you diagnose high memory usage in a large ArrayList? *(Senior)*

Check both element count and retained object graph. The backing array stores references, while each boxed value or object can add separate object overhead. A list can also retain capacity after elements are removed, and views may retain backing collections.

Use heap histograms and heap dumps, inspect dominator trees and reference paths, and compare retained size to shallow size. Consider primitive arrays, pagination, streaming, compaction, or shorter object lifetimes when justified by measurement.

## Official references

1. [Java SE 21 API — List](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/List.html)
2. [Java SE 21 API — ArrayList](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayList.html)
3. [Java SE 21 API — Iterator](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Iterator.html)
4. [Java SE 21 API — Collections](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Collections.html)

## Chapter checklist

- [ ] Explain size versus capacity and amortized append.
- [ ] Distinguish structural changes from element replacement.
- [ ] Understand fail-fast iterators and subList views.
- [ ] Choose fixed-size, mutable, unmodifiable, and copied lists deliberately.
- [ ] Diagnose retained memory rather than judging by list size alone.
