# Strings and Text Processing

**Target level:** Beginner → Senior  
**Scope:** Java 8–21

String handling appears in almost every Java system: identifiers, JSON, logs, database queries, HTTP headers, user-visible text, and security-sensitive validation. Distinguish immutable text values from mutable builders, UTF-16 code units from Unicode code points, and textual representation from semantic meaning.

## 1. String fundamentals

### Q1. Why is String immutable in Java? *(Beginner)*

A String's contents cannot be changed after construction. Operations that appear to modify a string return a new String or another result rather than mutating the original. Immutability simplifies sharing, caching, hashing, and use as a map key, and it helps make strings safe to share across threads.

Immutability does not mean every operation is allocation-free, nor does it mean a String is suitable for every security purpose. Avoid retaining sensitive values longer than necessary.

### Q2. What is the difference between String, StringBuilder, and StringBuffer? *(Beginner)*

String is immutable. StringBuilder is a mutable character sequence designed for efficient local construction. StringBuffer is mutable and synchronizes its methods for legacy thread-safe use.

Prefer concatenation for simple expressions and StringBuilder for repeated appends in loops when that improves clarity or measured performance. Synchronizing StringBuffer methods does not make a multi-step sequence atomic.

### Q3. What is the String pool? *(Intermediate)*

The String pool canonicalizes interned strings so equal string literals and strings returned by intern() can share a canonical instance. This can reduce duplicate storage for selected strings.

Do not use reference identity as a general string-equality rule. Strings read from files, network requests, or constructed dynamically are not guaranteed to be the same reference as a literal. Use equals for content equality.

### Q4. What is the difference between == and equals for strings? *(Beginner)*

For references, == compares identity, while String.equals compares character sequence contents.

    String a = new String("java");
    String b = new String("java");
    System.out.println(a == b);      // false
    System.out.println(a.equals(b)); // true

Literal pooling can make some == examples print true, but that is not a safe basis for content comparison. Use equals, or Objects.equals when null-safe comparison is required.

### Q5. What does String.intern() do, and should it be used broadly? *(Advanced)*

intern() returns the canonical pooled representation for a string value. It can be useful when an application intentionally canonicalizes a bounded set of repeated values.

Using intern() indiscriminately on high-cardinality or attacker-controlled input can create memory and performance problems. Measure before using it as an optimization.

### Q6. Why is String concatenation inside a loop potentially inefficient? *(Intermediate)*

Because String is immutable, repeated concatenation may create many intermediate strings and repeatedly copy accumulated content. Modern compilers and runtimes optimize many concatenation expressions, but repeatedly extending a result in a loop can still do unnecessary work.

    StringBuilder result = new StringBuilder();
    for (String part : parts) {
        result.append(part);
    }
    String joined = result.toString();

For a simple expression such as combining a first and last name, ordinary concatenation is clear and appropriate. Choose based on operation shape and measurements, not a blanket rule.

## 2. Equality, searching and manipulation

### Q7. What is the difference between equals and equalsIgnoreCase? *(Beginner)*

equals compares the exact sequence of UTF-16 code units. equalsIgnoreCase performs case-insensitive comparison using Java's case-mapping rules, but it is not a locale-sensitive linguistic comparison and should not be assumed to implement every language's collation rules.

For human text, use appropriate Locale and Collator APIs. For identifiers or protocol tokens, define the required normalization and case policy explicitly.

### Q8. Why can String.length() differ from the number of visible characters? *(Intermediate)*

String.length() returns the number of UTF-16 code units, not the number of Unicode code points or user-perceived grapheme clusters. A supplementary code point uses two char code units, and a displayed character can consist of multiple code points, such as a base letter plus combining marks.

Use codePointCount when code-point count is desired. User-perceived grapheme counting can require Unicode text segmentation rules beyond code points.

### Q9. How do you iterate over Unicode code points? *(Intermediate)*

Use String.codePoints() or advance with codePointAt and Character.charCount rather than incrementing a char index by one for every code point.

    String text = "A\uD83D\uDE00B";
    text.codePoints().forEach(cp -> System.out.printf("%X ", cp));
    // 41 1F600 42

This processes the supplementary character as one code point. If a task needs grapheme clusters, such as user-visible cursor positions, code points alone may not be enough.

### Q10. What is the difference between substring and split? *(Intermediate)*

substring extracts a range using begin and end indexes, with the end index exclusive. split divides text using a regular-expression delimiter and returns an array. The split argument is a regex, not necessarily a literal string.

    "a.b.c".split("\\."); // dot must be escaped in the regex

For a literal delimiter, Pattern.quote(delimiter) prevents regex metacharacters from changing the meaning. Also consider split's limit argument: the default drops trailing empty strings, while a negative limit preserves them.

### Q11. Why does split sometimes lose trailing empty fields? *(Intermediate)*

String.split(regex) behaves like split(regex, 0), and a zero limit discards trailing empty strings. If trailing empty fields are meaningful, use a negative limit.

    System.out.println(Arrays.toString("a,b,,".split(",")));     // [a, b]
    System.out.println(Arrays.toString("a,b,,".split(",", -1))); // [a, b, , ]

For CSV or other structured formats, splitting on commas is insufficient because quoting, escaped delimiters, and line breaks may be part of the format. Use a parser that follows the actual specification.

### Q12. What is the difference between trim() and strip()? *(Intermediate)*

trim() removes leading and trailing characters whose code points are at most U+0020 under its historical definition. strip(), added in Java 11, uses Unicode whitespace classification via Character.isWhitespace.

The difference matters for non-ASCII whitespace. If targeting Java 8, strip() is unavailable. For security-sensitive input, define exactly which characters are allowed rather than assuming a generic whitespace method performs validation.

### Q13. What do isEmpty() and isBlank() mean? *(Intermediate)*

isEmpty() is true only when the string length is zero. isBlank(), added in Java 11, is true when the string is empty or consists entirely of characters recognized as whitespace by Java's definition.

A string containing spaces can be non-empty but blank. For protocol identifiers, passwords, or security tokens, do not silently normalize values unless the protocol explicitly permits it.

### Q14. How should strings be compared lexicographically? *(Intermediate)*

String.compareTo compares strings lexicographically according to UTF-16 code-unit values and returns a negative integer, zero, or a positive integer. It does not promise to return only -1, 0, or 1.

Lexicographic code-unit order is not the same as locale-aware alphabetical order. For user-facing sorting, use a Collator configured for the intended locale; for stable machine identifiers, define a consistent comparison contract.

## 3. Formatting, conversion and parsing

### Q15. What are common pitfalls with String.format? *(Intermediate)*

String.format uses format specifiers and locale-sensitive formatting behavior. A mismatch between a specifier and its argument can cause IllegalFormatException, and decimal separators or other output can vary by locale.

Specify Locale when output must be stable, especially for machine-readable data. Do not build SQL or shell commands by formatting untrusted strings; use parameterized APIs that separate data from syntax.

### Q16. What is the difference between valueOf and toString? *(Intermediate)*

String.valueOf has overloads for primitive values and objects. For a non-null object it typically delegates to toString; for a null reference it returns the text "null". Calling obj.toString() directly on null throws NullPointerException.

Neither approach is a serialization format unless the type explicitly defines it as one. toString is commonly intended for human-readable diagnostics and its output can change as implementations evolve.

### Q17. How do you parse numeric strings safely? *(Intermediate)*

Use an appropriate parse method, such as Integer.parseInt, and handle NumberFormatException at the boundary where malformed input is expected. Validate range and domain constraints separately; syntactically valid numbers may still be invalid for the business operation.

Avoid broad catch blocks that turn every parsing failure into a silent default such as zero. That can transform bad input into plausible but incorrect data.

### Q18. Why is StringBuilder generally preferable for repeated mutation? *(Intermediate)*

StringBuilder maintains a mutable character buffer and avoids creating a new immutable String for every append. It is usually appropriate for loops, serializers, and generated text when many pieces are appended.

It is not automatically faster for every expression, and it does not make concurrent access safe. For joining a collection, String.join or Collectors.joining may express intent more clearly than a manual builder.

### Q19. What are text blocks, and when were they introduced? *(Intermediate)*

Text blocks became permanent in Java 15 after preview releases. They allow multiline string literals using triple quotes and defined indentation and escape-processing rules, improving readability for embedded SQL, JSON examples, and templates.

Text blocks are ordinary String values; they do not parse or validate the embedded language. Indentation and trailing-newline behavior follow language rules, so test exact output when whitespace is significant. They are unavailable under Java 8 source compatibility.

### Q20. What are String.repeat, lines, and strip, and which Java version introduced them? *(Intermediate)*

These methods were added in Java 11. repeat creates a string by repeating the sequence; lines provides a stream of lines; strip removes leading/trailing Unicode whitespace according to Java's whitespace definition. They are convenient APIs, not language syntax.

If a project targets Java 8, these calls will not compile against the Java 8 API. The build's --release setting is useful for enforcing both source language and target platform API compatibility.

## 4. Security and production concerns

### Q21. Why is string concatenation dangerous for SQL? *(Advanced)*

Concatenating untrusted values into SQL mixes data with executable query syntax, enabling SQL injection when values are interpreted as part of the query. Use PreparedStatement parameters or a framework's parameterized query mechanism.

    PreparedStatement ps = connection.prepareStatement(
        "SELECT id FROM users WHERE email = ?");
    ps.setString(1, email);

Parameters represent values, not arbitrary SQL identifiers or keywords. Dynamic table/column names require a separate allowlist or safe query-construction strategy.

### Q22. Why is string concatenation dangerous for shell commands or HTML? *(Advanced)*

Shell commands and HTML have their own parsing and escaping rules. Concatenating untrusted text into those contexts can change the command or markup structure, leading to command injection or cross-site scripting.

Prefer APIs that pass arguments separately rather than invoking a shell with one assembled command string. For HTML, use context-appropriate output encoding and safe templating. A generic escape function is not correct for every context: HTML text, attributes, JavaScript, URLs, and CSS have different rules.

### Q23. What is Unicode normalization, and when is it important? *(Advanced)*

Visually identical text can have different code-point sequences, such as a precomposed accented character versus a base character followed by a combining mark. Unicode normalization maps text into a defined form, such as NFC or NFKC.

Whether normalization is appropriate depends on the domain. It can matter for search, identifiers, deduplication, and security, but normalizing passwords or opaque protocol tokens without a clear specification can change meaning. Case folding, normalization, and locale-sensitive comparison are distinct operations.

### Q24. Why should you avoid using String as a mutable shared buffer? *(Intermediate)*

String is immutable, so updates create new values. If several threads need to assemble shared text, define ownership or synchronization rather than trying to mutate a String. A local StringBuilder is normally safe when confined to one thread, but sharing it requires coordination.

For concurrent logs, prefer a logging framework that accepts structured arguments rather than manually assembling shared message buffers.

### Q25. Is String safe for storing passwords? *(Senior)*

String immutability and ordinary garbage collection mean an application's reference cannot explicitly overwrite the String's character storage. This can make secret lifetime harder to control. For APIs that permit it, char[] or byte[] can be cleared after use, though copies may still exist and clearing is not a complete guarantee.

Follow the credential library's recommended type and lifecycle, avoid logging secrets, minimize copies, and use dedicated password-hashing mechanisms for stored passwords. Do not invent a custom encryption or hashing scheme because String is convenient.

### Q26. How can normalization bugs become security bugs? *(Senior)*

If validation checks one representation while a later component interprets another, attackers may exploit differences in Unicode normalization, case handling, path decoding, or percent-encoding. Two strings that look alike to a person may not compare equal, and differently encoded inputs may be interpreted as the same identifier downstream.

Define canonicalization at the correct trust boundary and ensure validation and interpretation use compatible rules. Do not normalize blindly: specify allowed characters and a canonical representation, then test alternate encodings.

### Q27. How would you safely truncate text for a user interface? *(Senior)*

First decide whether the limit is bytes, UTF-16 code units, Unicode code points, or user-perceived grapheme clusters. These limits are not interchangeable. Truncating in the middle of a surrogate pair or grapheme cluster can produce invalid or visually broken output.

For storage and protocol limits, obey the exact byte or character-count rule. For UI truncation, use grapheme-aware segmentation when required and make clear whether the original full value remains available.

### Q28. How do you benchmark string concatenation strategies? *(Senior)*

Use JMH or another suitable microbenchmark harness rather than a single wall-clock loop. Warmup, JIT compilation, dead-code elimination, allocation rate, input sizes, and runtime version can dominate results. Compare realistic workloads and consume results so the benchmark cannot eliminate the work.

Prefer the clearest implementation first, then measure a representative bottleneck. Modern Java concatenation can be optimized aggressively, so folklore is not a substitute for measurement.

## Official references

1. [Java SE 21 API — String](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/String.html)
2. [Java SE 21 API — StringBuilder](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/StringBuilder.html)
3. [Java SE 21 API — Character](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Character.html)
4. [JEP 378 — Text Blocks](https://openjdk.org/jeps/378)
5. [Java SE 21 API — Normalizer](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/text/Normalizer.html)

## Chapter checklist

- [ ] Explain immutability, pooling, identity, equality, and builder use.
- [ ] Distinguish UTF-16 code units, code points, and grapheme clusters.
- [ ] Handle split limits, regex delimiters, blank strings, and locale.
- [ ] Use parameterized APIs instead of concatenating SQL, shell, or HTML.
- [ ] Treat normalization and secret handling as explicit domain/security decisions.
- [ ] Benchmark string strategies with a proper harness before optimizing.
