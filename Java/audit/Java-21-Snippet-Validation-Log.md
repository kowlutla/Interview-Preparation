# Java Snippet Validation Log

**Validation date:** 2026-10-24  
**Compiler/runtime:** OpenJDK 21.0.12.1 (Debian build)  
**Command pattern:** `javac --release 21 ...`, then execute the resulting classes.

This log records only the examples explicitly listed below. It is not a claim that all handbook code blocks have been compiled.

## Executed examples

| Area | Example | Result | Notes |
|---|---|---|---|
| Collections | Mutable key used in a HashMap | Compiled and executed | Lookup printed `null` after mutating the key's equality/hash state in this run. This is illustrative, not a guaranteed output for every key/hash setup. |
| Streams | `Stream.of(...).peek(...).count()` | Compiled and executed | On this JDK, `count` returned 3 while the `peek` side-effect list remained empty. This demonstrates why required side effects must not be placed in intermediate stream operations. |
| Streams | Parallel integer sum with `reduce` | Compiled and executed | Returned 10. |
| Streams | `Stream.toList()` and `Collectors.toCollection(ArrayList::new)` | Compiled and executed | Both created the expected values; the API contracts differ on mutability. |
| Concurrency | Volatile stop flag and volatile increment example | Compiled and executed | Syntax/type check only; a single-thread run does not prove the counter increment is thread-safe. |

## Reproduction

The examples above were copied into standalone Java source files for validation. The following command compiled the three source files together:

```bash
javac --release 21 MutableKeyDemo.java StreamExamples.java ConcurrencyExamples.java
```

Run them with:

```bash
java MutableKeyDemo
java StreamExamples
java ConcurrencyExamples
```

## Important limitations

- The tests used Java 21 only; no Java 8, 11, or 17 matrix has been completed yet.
- A successful compile does not prove an example's concurrency semantics are correct.
- Some examples in the handbook are fragments that intentionally rely on surrounding context and cannot be compiled independently as pasted.
- A full scan and execution harness for all code fences is still required.
- Output that depends on implementation details or unspecified behavior must be labeled accordingly; a single observed run is not a portable guarantee.
