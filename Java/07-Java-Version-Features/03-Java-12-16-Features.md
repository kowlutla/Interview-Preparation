# Java 12–16 Features

**Target level:** Intermediate → Senior  
**Releases:** Java 12–16

This period introduced switch expressions, text blocks, records, pattern matching for instanceof, and other language/runtime changes. Several features were previewed before becoming permanent, so version-aware answers must state the finalization release.

## 1. Switch expressions

### Q1. What is a switch expression? *(Intermediate)*

A switch expression returns a value and can be used in an assignment or other expression context. It became permanent in Java 14 after preview releases.

    int days = switch (month) {
        case APRIL, JUNE, SEPTEMBER, NOVEMBER -> 30;
        case FEBRUARY -> 28;
        default -> 31;
    };

Arrow rules avoid accidental fall-through. The expression must produce a value on all paths according to the language rules, making missing cases more visible.

### Q2. What is yield in a switch expression? *(Advanced)*

yield returns a value from a block-style switch-expression rule. It is used when a case needs multiple statements before producing the result.

Do not confuse yield with return: yield returns from the switch expression, while return exits the enclosing method. Use arrow expressions for simple cases and block rules when validation or multiple statements are needed.

## 2. Text blocks and records

### Q3. What are text blocks? *(Intermediate)*

Text blocks became permanent in Java 15. They provide multiline string literals with defined indentation and escape processing, useful for embedded SQL, JSON, and other readable text.

They do not parse or validate the embedded format. Whitespace and trailing newline behavior can matter, so test the resulting string when it is used in a protocol or exact comparison.

### Q4. What are records? *(Intermediate)*

Records became permanent in Java 16 after preview releases in Java 14 and 15. They model transparent carriers of fixed components and generate accessors, canonical construction, equals, hashCode, and toString according to record rules.

Record components are final references, but nested mutable objects are not automatically made immutable. Use validation and defensive copies when the value contract requires them.

### Q5. Can records implement interfaces? *(Intermediate)*

Yes. A record can implement interfaces, but it implicitly extends java.lang.Record and cannot extend an arbitrary class. Its instance state is defined by its record components.

Records work well with interfaces and sealed hierarchies for explicit value variants. They are not a replacement for every mutable entity or class with hidden lifecycle state.

## 3. Pattern matching

### Q6. What is pattern matching for instanceof? *(Intermediate)*

Pattern matching for instanceof became permanent in Java 16. It combines a type check with a variable binding and avoids a separate cast when the pattern succeeds.

    if (obj instanceof String text) {
        System.out.println(text.length());
    }

Pattern-variable scope follows flow analysis. This is a final feature in Java 16 and later, unlike switch pattern matching, which became final in Java 21.

### Q7. What is the difference between a preview feature and a final feature? *(Intermediate)*

A preview feature is released for evaluation and may change or be removed, typically requiring preview flags for compilation and execution. A final feature is part of the permanent language/API contract for that release.

State the version in which a feature became final, not just the first release where it appeared. This matters for build flags and compatibility guarantees.

## 4. Other important changes

### Q8. What is the Vector API? *(Advanced)*

The Vector API was introduced as an incubator API in Java 16 and continued to evolve in later releases. It expresses vector computations that can map to SIMD instructions while remaining within Java's type system.

Incubator status means the API is not a stable permanent Java SE contract. Use it only when the target JDK and maintenance policy permit incubator modules, and benchmark against ordinary loops and library implementations.

### Q9. What is helpful about JFR in modern Java? *(Intermediate)*

Java Flight Recorder is a low-overhead event recording system for JVM and application diagnostics. It supports analysis of CPU, allocation, GC, locks, and other runtime events.

Use JFR to connect symptoms to time-correlated evidence. It complements, rather than replaces, thread dumps, heap dumps, logs, and service-level metrics.

### Q10. What changed in Java 16 for packaging and runtime? *(Advanced)*

Java 16 included language and runtime changes such as records and pattern matching for instanceof becoming final, along with platform/tooling evolution. Exact changes should be checked against the release notes and JEP list for the target distribution.

Do not memorize a release as a single feature. Maintain a version matrix and verify source/API/runtime compatibility for every feature used.

## Official references

1. [JEP 361 — Switch Expressions](https://openjdk.org/jeps/361)
2. [JEP 378 — Text Blocks](https://openjdk.org/jeps/378)
3. [JEP 395 — Records](https://openjdk.org/jeps/395)
4. [JEP 394 — Pattern Matching for instanceof](https://openjdk.org/jeps/394)
5. [JEP 338 — Vector API (Incubator)](https://openjdk.org/jeps/338)
6. [JDK 16 Documentation](https://docs.oracle.com/en/java/javase/16/)

## Chapter checklist

- [ ] Identify the release where switch expressions became final.
- [ ] Explain text blocks and records.
- [ ] Distinguish pattern matching for instanceof from switch pattern matching.
- [ ] Track incubator and preview status accurately.
