# Java 8–21 Feature and Status Matrix

Use this as a quick index, not as a replacement for the release chapters or the relevant JEP/API specification. “Final” means part of the standard platform for that release; “preview” requires the matching release's preview enablement and can change; “incubator” is an evolving API/module and is not a finalized Java SE contract.

## Release-by-release highlights

| Release | Important additions and changes | Status notes |
|---|---|---|
| Java 8 | Lambda expressions, method references, functional interfaces, Stream API, default/static interface methods, Optional, java.time, CompletableFuture, Base64 API | Standard/final in Java 8 |
| Java 9 | JPMS/modules, JShell, collection factory methods, private interface methods, Process API improvements, Flow API | Standard/final in Java 9 |
| Java 10 | Local-variable type inference with var, APIs for unmodifiable collection copies | Standard/final in Java 10; var applies only to local-variable declarations allowed by the language rules |
| Java 11 | Standard HTTP Client, new String/file methods, var in lambda parameters | Standard/final in Java 11 |
| Java 12 | Switch expressions first preview, teeing collector, Shenandoah available in some builds | Switch expressions were preview; collector API is standard in Java 12 |
| Java 13 | Text blocks first preview, switch expressions second preview | Preview features are not stable Java SE contracts in this release |
| Java 14 | Switch expressions final, records first preview, pattern matching for instanceof first preview, helpful NullPointerException messages | Status differs by feature: switch expressions final; records and instanceof pattern matching preview |
| Java 15 | Text blocks final, records second preview, sealed classes first preview, hidden classes | Text blocks final; records/sealed classes preview; hidden classes are a JVM feature |
| Java 16 | Records final, pattern matching for instanceof final, Stream.toList(), strong encapsulation changes | Records and instanceof pattern matching final; Stream.toList() is not available on Java 8 |
| Java 17 | Sealed classes final, always-strict floating-point semantics, stronger encapsulation of JDK internals, enhanced pseudo-random number generators | Sealed classes final; pattern matching for switch remained preview |
| Java 18 | UTF-8 as the default charset, simple web server, code snippets in Javadoc, finalization deprecated for removal | Standard platform/runtime changes; finalization is deprecated for removal, not removed in Java 18 |
| Java 19 | Virtual threads first preview, structured concurrency first incubator, record patterns first preview, pattern matching for switch fourth preview | Preview/incubator status; not final Java 19 APIs |
| Java 20 | Virtual threads second preview, structured concurrency second incubator, scoped values incubator, record patterns second preview, pattern matching for switch fifth preview, FFM API second preview | Preview/incubator status; exact JEP matters |
| Java 21 | Virtual threads, record patterns, pattern matching for switch, sequenced collections, generational ZGC, KEM API; string templates, unnamed patterns/variables, unnamed classes/instance main methods, FFM API, scoped values and structured concurrency; Vector API | Virtual threads, record patterns, pattern switch, sequenced collections, generational ZGC and KEM API are final in Java 21. String templates, unnamed patterns/variables, unnamed classes/instance main methods, FFM, scoped values and structured concurrency are preview; Vector API is incubating. JEP 451 prepares to disallow dynamic agent loading; JEP 449 deprecates the Windows 32-bit x86 port for removal. |

## Java 21 status details

The Java SE 21 release includes more than language syntax. Key JEPs include:

| JEP | Feature | Java 21 status |
|---|---|---|
| 431 | Sequenced Collections | Final |
| 439 | Generational ZGC | Final |
| 440 | Record Patterns | Final |
| 441 | Pattern Matching for switch | Final |
| 444 | Virtual Threads | Final |
| 452 | Key Encapsulation Mechanism API | Final |
| 430 | String Templates | Preview |
| 442 | Foreign Function & Memory API | Third preview |
| 443 | Unnamed Patterns and Variables | Preview |
| 445 | Unnamed Classes and Instance Main Methods | Preview |
| 446 | Scoped Values | Preview |
| 448 | Vector API | Sixth incubator |
| 453 | Structured Concurrency | Preview |
| 451 | Prepare to Disallow the Dynamic Loading of Agents | Runtime/operational change preparing for a future restriction |
| 449 | Deprecate the Windows 32-bit x86 Port for Removal | Deprecation/platform lifecycle change |

Do not assume the API shape of a Java 21 preview matches the later version. When studying newer JDKs, check the JEP history and the API documentation for that exact release.

## Important cross-release interview reminders

- Records became final in Java 16, not Java 14 or 15.
- Pattern matching for instanceof became final in Java 16.
- Sealed classes became final in Java 17.
- Switch expressions became final in Java 14.
- Text blocks became final in Java 15.
- Virtual threads became final in Java 21.
- Structured concurrency and scoped values were still preview features in Java 21.
- Java 21's String Templates were preview; do not teach them as finalized syntax.
- The Java 21 Vector API was incubating, not a standard finalized API.
- A feature's current status in a newer JDK must not be retroactively assigned to an older JDK.

## Official references

- [Java SE 21 specifications and API index](https://docs.oracle.com/en/java/javase/21/docs/specs/)
- [Java SE 21 API documentation](https://docs.oracle.com/en/java/javase/21/docs/api/)
- [Java language changes for Java SE 21](https://docs.oracle.com/en/java/javase/21/language/java-language-changes-release.html)
- [JDK 21 release notes](https://www.oracle.com/java/technologies/javase/21all-relnotes.html)
- [OpenJDK JEP index](https://openjdk.org/jeps/0)
