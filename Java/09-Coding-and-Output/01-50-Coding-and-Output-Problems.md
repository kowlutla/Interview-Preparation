# Java Coding and Output Practice — 50 Problems

Use this bank after studying the handbook chapters. For each coding problem, state assumptions, complexity, edge cases, and Java-version requirements. For output problems, identify compile-time errors separately from runtime output.

## A. String problems (1–10)

### Problem 1. Reverse a string

**Task:** Reverse a string without using a built-in reverse method.

**Reference approach:** Convert to char array, swap from both ends, create a new String. O(n) time and O(n) output space. Clarify UTF-16 code-unit versus Unicode code-point reversal.

### Problem 2. Check a palindrome

**Task:** Ignore spaces and punctuation, then check case-insensitive palindrome behavior.

**Reference approach:** Two pointers, skip non-alphanumeric code units, compare normalized characters. For full Unicode behavior, iterate code points and define normalization/case rules.

### Problem 3. Count character frequency

**Task:** Return each character and its count.

**Reference approach:** One pass with HashMap.merge. O(n) expected time and O(k) space. State whether characters mean UTF-16 code units or Unicode code points.

### Problem 4. First non-repeating character

**Task:** Return the first character with frequency one.

**Reference approach:** Count in a LinkedHashMap or do two passes (frequency map, then scan input). O(n) expected time.

### Problem 5. Check anagrams

**Task:** Determine whether two strings contain the same characters.

**Reference approach:** Count character frequencies or sort character arrays. Define case sensitivity, whitespace, and Unicode normalization.

### Problem 6. Longest substring without repeated characters

**Task:** Return the maximum length of a substring with unique characters.

**Reference approach:** Sliding window plus last-seen index map. O(n) expected time.

### Problem 7. Group anagrams

**Task:** Group words that are anagrams.

**Reference approach:** Canonical key from sorted characters or a frequency signature; collect into Map<Key,List<String>>. Discuss alphabet and Unicode assumptions.

### Problem 8. Longest common prefix

**Task:** Find the longest prefix shared by all strings.

**Reference approach:** Vertical scan to the shortest string's length. O(total characters inspected).

### Problem 9. Balanced brackets

**Task:** Validate (), [], and {} nesting.

**Reference approach:** ArrayDeque as a stack, push opening symbols, match each closer, require an empty stack at the end. O(n) time and space.

### Problem 10. Run-length encode a string

**Task:** Convert repeated runs such as aaabb into a3b2.

**Reference approach:** Scan each maximal run and append character plus run length. O(n). Discuss escaping when input itself contains digits.

## B. Arrays and algorithms (11–22)

### Problem 11. Find the missing number in 0..n

**Reference approach:** XOR all expected indices and values, or use a long sum to avoid int overflow. Validate the uniqueness/range assumption.

### Problem 12. Find the second-largest distinct value

**Reference approach:** One pass tracking largest and second largest with explicit presence flags. O(n) time, O(1) extra space. Test all-negative arrays and duplicates.

### Problem 13. Rotate an array by k positions

**Reference approach:** Three reversals, normalizing k with floorMod. O(n) time, O(1) extra space.

### Problem 14. Move zeroes to the end

**Reference approach:** Stable compaction of nonzero elements followed by zero fill. O(n) time, O(1) extra space.

### Problem 15. Merge two sorted arrays

**Reference approach:** Two pointers, selecting the smaller current value and then copying the remainder. O(n+m) time.

### Problem 16. Binary search

**Reference approach:** Maintain inclusive or half-open bounds consistently; compute midpoint as lo + (hi-lo)/2. O(log n). Input must be sorted by the searched ordering.

### Problem 17. Two-sum

**Reference approach:** Scan once with a map from seen value to index; look up target-current before inserting. O(n) expected time. Consider integer overflow and duplicate policy.

### Problem 18. Maximum subarray sum

**Reference approach:** Kadane's algorithm; track best subarray ending here and global best. O(n) time. Use long if sum bounds exceed int.

### Problem 19. Majority element

**Reference approach:** Boyer–Moore candidate pass, then verify the candidate occurs more than n/2 times unless the input guarantees a majority. O(n), O(1).

### Problem 20. Merge overlapping intervals

**Reference approach:** Sort by start, then extend the current interval while the next start overlaps its end. O(n log n). Define whether touching endpoints overlap.

### Problem 21. Top K frequent values

**Reference approach:** Frequency map plus min-heap of size k, O(n + u log k). Define tie-breaking for equal frequencies.

### Problem 22. Longest consecutive sequence

**Reference approach:** Put values in a HashSet and start a run only where x-1 is absent. O(n) expected time. Handle integer boundary overflow carefully.

## C. Collections and streams (23–32)

### Problem 23. Remove duplicates preserving encounter order

**Reference approach:** LinkedHashSet or stream distinct on an ordered stream. Discuss equality contract and null policy.

### Problem 24. Group employees by department

**Reference approach:** groupingBy(Employee::department). If only counts are needed, use a downstream counting collector instead of materializing every group list.

### Problem 25. Find second-highest distinct salary

**Reference approach:** distinct then descending sort and skip one, or maintain top two in one pass. Explain O(n log n) versus O(n).

### Problem 26. Convert a list to a map with duplicate keys

**Reference approach:** Collectors.toMap with an explicit merge function. Never let accidental duplicate handling remain unspecified.

### Problem 27. Count word frequencies

**Reference approach:** Map.merge(word, 1, Integer::sum). Define tokenization, case folding, and Unicode normalization.

### Problem 28. Find duplicate elements and counts

**Reference approach:** frequency map, then filter entries whose count exceeds one. Choose HashMap or LinkedHashMap depending on output-order requirements.

### Problem 29. Flatten nested collections

**Reference approach:** flatMap(Collection::stream). Define how null nested collections are handled and whether encounter order matters.

### Problem 30. Implement an LRU cache

**Reference approach:** LinkedHashMap in access-order mode with removeEldestEntry for a simple bounded cache. State clearly that it is not thread-safe and has no TTL/load-coalescing policy.

### Problem 31. Sort employees by department and salary

**Reference approach:** Comparator.comparing(Employee::department).thenComparingLong(Employee::salary). Define null handling and ascending/descending rules.

### Problem 32. Find the latest event at or before a time

**Reference approach:** NavigableMap.floorEntry with a composite key if multiple events can share the same timestamp. Consider database indexing for durable large histories.

## D. Core Java output and compilation (33–42)

### Problem 33. What prints?
    byte b = 127;
    b++;
    System.out.println(b);

**Answer:** -128. Increment narrows the overflowing int result back to byte.

### Problem 34. Does this compile?
    byte b = 1;
    // b = b + 1;

**Answer:** No. Binary numeric promotion makes b + 1 an int; assignment needs a cast. The compound form b += 1 is permitted with implicit narrowing.

### Problem 35. What prints?
    System.out.println(5 / 2);
    System.out.println(5 / 2.0);

**Answer:** 2 and 2.5. The first is integral division; the second is floating-point division.

### Problem 36. What prints?
    String a = new String("x");
    String b = new String("x");
    System.out.println(a == b);
    System.out.println(a.equals(b));

**Answer:** false, then true. Identity differs from content equality.

### Problem 37. What prints?
    int x = 0;
    boolean y = false && (++x > 0);
    System.out.println(x);

**Answer:** 0. The right operand is skipped by short-circuit AND.

### Problem 38. What prints?
    int x = 0;
    boolean y = true || (++x > 0);
    System.out.println(x);

**Answer:** 0. The right operand is skipped by short-circuit OR.

### Problem 39. What prints?
    System.out.println((double) (5 / 2));
    System.out.println((double) 5 / 2);

**Answer:** 2.0 then 2.5. Cast placement changes which operation is floating point.

### Problem 40. What happens?
    System.out.println(Integer.MAX_VALUE + 1);

**Answer:** -2147483648. int overflow wraps according to Java's defined two's-complement arithmetic; no exception is thrown.

### Problem 41. What prints?
    String s = null;
    System.out.println(String.valueOf(s));

**Answer:** The text null. Calling s.toString() would throw NullPointerException.

### Problem 42. What prints?
    List<String> a = new ArrayList<>(Arrays.asList("a"));
    List<String> v = Collections.unmodifiableList(a);
    a.add("b");
    System.out.println(v);

**Answer:** [a, b]. The wrapper is an unmodifiable live view, not a snapshot.

## E. Collections, exceptions and concurrency output (43–50)

### Problem 43. What happens when a HashMap key is mutated after insertion?

**Answer:** If the mutation changes hashCode/equals, later lookup/removal may fail because the key is searched using its new hash/equality state. Use stable immutable keys.

### Problem 44. Does HashMap guarantee insertion order?

**Answer:** No. Use LinkedHashMap for insertion-order iteration or TreeMap for sorted keys.

### Problem 45. What happens when a list is structurally modified during enhanced-for iteration?
    List<Integer> xs = new ArrayList<>(Arrays.asList(1, 2, 3));
    for (Integer x : xs) {
        if (x == 1) xs.remove(x);
    }

**Answer:** With the standard ArrayList iterator, the next iteration typically detects the structural modification and throws ConcurrentModificationException. Fail-fast detection is best-effort, not a contractual synchronization guarantee. The similar-looking variant that removes `2` from `[1, 2, 3]` can finish without throwing because the iterator may not call `next()` again; never rely on the exception for correctness. Use `Iterator.remove()` or `removeIf`.

### Problem 46. What is the primary exception?
    try {
        throw new IllegalStateException("A");
    } finally {
        throw new IllegalArgumentException("B");
    }

**Answer:** IllegalArgumentException("B") propagates; the finally exception replaces the pending exception. Avoid throwing/returning from finally.

### Problem 47. Is volatile count++ atomic?

**Answer:** No. Increment is a read-modify-write sequence. Use AtomicInteger or a lock if increments must not be lost.

### Problem 48. Why can sleep not fix a visibility bug?

**Answer:** sleep does not establish a happens-before relationship for arbitrary shared state. Use volatile, synchronization, atomics, or a documented concurrency primitive.

### Problem 49. Does Future.cancel(true) guarantee a remote operation stops?

**Answer:** No. It requests cancellation/interruption but cannot undo a remote side effect that may already have committed. Use deadlines, cooperative cancellation, idempotency, and reconciliation.

### Problem 50. Does a virtual thread make a database connection unlimited?

**Answer:** No. Virtual threads reduce the cost of waiting threads but database connections and downstream capacity remain bounded. Use pool limits, timeouts, and backpressure.

## Candidate evaluation rubric

For each problem, score 0–2 for correctness, 0–2 for edge cases, 0–2 for complexity, 0–2 for Java-specific semantics, and 0–2 for clarity. A strong senior answer explains assumptions and failure modes rather than only producing a working happy-path snippet.
