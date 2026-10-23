# Java Rapid-Fire Revision

Use these as short oral prompts. A strong answer should be concise but precise; expand only when asked for mechanics or trade-offs.

## Language and types

1. **Is Java pass-by-reference?** No. Java passes values; an object reference value can be copied into a parameter.
2. **Can local variables use default values?** No usable default under definite-assignment rules; fields and array elements are default-initialized.
3. **Is char a Unicode code point?** No. It is a UTF-16 code unit.
4. **Does int overflow throw?** No; integer arithmetic wraps according to Java's defined rules.
5. **Does cast after integer division restore decimals?** No. The division already truncated.
6. **Can a constructor be overridden?** No; constructors can be overloaded, not inherited/overridden.
7. **Can static methods be overridden?** No; they can be hidden.
8. **Does final reference mean immutable object?** No; it prevents reassignment of the reference.
9. **Are records deeply immutable?** No; components can refer to mutable objects.
10. **What makes a functional interface?** One abstract method under JLS rules.

## OOP and contracts

11. **Why override equals and hashCode together?** Equal objects must have equal hashes for hash-based collections.
12. **Can equal objects have different hashes?** No; unequal objects may collide.
13. **What is LSP?** Subtypes honor supertype behavioral promises.
14. **When composition over inheritance?** When reuse is desired without a true behavioral subtype or when behavior should be replaceable.
15. **What does sealed mean?** It restricts permitted direct subtypes.
16. **When did records become final?** Java 16.
17. **When did sealed classes become final?** Java 17.
18. **When did switch pattern matching become final?** Java 21.
19. **Does @Inherited apply to method annotations?** No; it applies to class annotations.
20. **Does a dynamic JDK proxy proxy concrete classes?** No; it implements interfaces.

## Generics and collections

21. **Are generics covariant?** No; generic types are invariant.
22. **What does PECS mean?** Producer extends, consumer super.
23. **What is type erasure?** Generic type arguments are mostly removed from runtime executable type information.
24. **Why no generic arrays?** Runtime array store checks cannot enforce erased parameterized types.
25. **Does HashMap preserve order?** No.
26. **Does LinkedHashMap preserve order?** It provides a defined encounter order (insertion or access order by configuration).
27. **Does TreeSet use equals for uniqueness?** It uses comparator/natural-order equivalence.
28. **Is ArrayList thread-safe?** No.
29. **Is subList a copy?** No; it is a backed view.
30. **Is an unmodifiable view a snapshot?** Not necessarily.

## Streams and Optional

31. **Are intermediate stream operations eager?** Generally no; they are lazy.
32. **Can a stream be reused?** No; a stream pipeline is single-use.
33. **What is flatMap?** Transform each input to a stream-like result and flatten it.
34. **What property does parallel reduce need?** Associativity and a valid identity/combiner contract.
35. **Is peek for business side effects?** No; use it primarily for debugging.
36. **What is Optional.ofNullable(null)?** Optional.empty.
37. **Does orElse evaluate eagerly?** Yes.
38. **Does orElseGet evaluate lazily?** Only when empty.
39. **Should Optional wrap every field?** Usually not; it is primarily useful for return contracts.
40. **Does Stream.toList return a mutable list?** No; it returns an unmodifiable list (Java 16+).

## Concurrency and JVM

41. **Does volatile make increment atomic?** No.
42. **What is happens-before?** A memory-model ordering relation that guarantees visibility/order.
43. **Does sleep create a happens-before edge?** No, not for arbitrary shared state.
44. **What does synchronized provide?** Mutual exclusion and visibility/order via monitor semantics.
45. **Can cancellation undo a remote side effect?** No guarantee.
46. **Are virtual threads final in Java 21?** Yes.
47. **Are scoped values final in Java 21?** No; preview in Java 21.
48. **What is a class-loader leak?** Old loaders remain reachable, preventing unloading of associated classes/metadata.
49. **Can native memory OOM occur with free heap?** Yes.
50. **Does GC guarantee prompt cleanup?** No; use explicit resource management.

## I/O, security and operations

51. **Why PreparedStatement?** Parameterized values prevent SQL syntax injection and clarify binding.
52. **Does TLS without hostname verification authenticate the server correctly?** No.
53. **Is Base64 encryption?** No; it is an encoding.
54. **Should untrusted Java serialization be accepted?** Avoid it; if unavoidable, strictly filter and limit the graph.
55. **What is idempotency?** Repeating an operation has the same intended effect as applying it once.
56. **Why can retrying a timeout duplicate a payment?** The first operation may have committed even though the response was lost.
57. **What does a bounded queue provide?** A finite pending-work limit and an explicit overload point.
58. **What should be checked for container OOM?** Total RSS/native memory, heap, Metaspace, direct buffers, stacks, and cgroup limits.
59. **What is JFR?** A runtime event recording facility for JVM/application diagnostics.
60. **What is the first step in performance tuning?** Define the symptom and measure a baseline.

## Final oral-answer pattern

For each prompt: define the concept, state the guarantee, name one limitation, and give one practical example. This keeps answers concise without becoming superficial.
