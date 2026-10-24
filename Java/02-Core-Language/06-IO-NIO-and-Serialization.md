# I/O, NIO and Serialization

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Java I/O APIs range from byte and character streams to files, paths, channels, buffers, and serialization. Correct usage requires explicit resource ownership, character encoding, bounded input handling, and a clear decision about whether Java object serialization is appropriate for the system.

## 1. Byte and character streams

### Q1. What is the difference between InputStream and Reader? *(Beginner)*

InputStream reads bytes. Reader reads characters after decoding bytes according to a character set. OutputStream writes bytes, while Writer writes characters and encodes them as bytes where applicable.

Use byte streams for binary data and character readers/writers for text. Always specify the intended charset at file/network boundaries rather than relying on platform defaults.

### Q2. Why should StandardCharsets.UTF_8 be explicit? *(Intermediate)*

In Java 17 and earlier, the default charset could vary with the operating system and environment. Starting with Java 18, UTF-8 is the default charset for Java SE APIs by default (with a compatibility mode available through runtime configuration). If a producer and consumer use different encodings, text can be corrupted or interpreted incorrectly. Explicitly choosing the format's required charset remains the clearest way to make a file or protocol contract unambiguous.

    // Java 11+
    Files.readString(path, StandardCharsets.UTF_8);
    Files.writeString(path, text, StandardCharsets.UTF_8);

Explicit encoding makes the protocol reproducible. For machine-readable formats, follow the format's encoding rules rather than the local operating system's preferences.

### Q3. Why use try-with-resources for streams? *(Beginner)*

Streams, readers, writers, channels, and many other I/O objects hold operating-system or runtime resources. Try-with-resources closes AutoCloseable resources even when the block throws, and closes multiple resources in reverse initialization order.

    // InputStream.readAllBytes() is Java 9+.
    try (InputStream in = Files.newInputStream(path)) {
        return in.readAllBytes();
    }

For untrusted or very large inputs, readAllBytes can consume excessive memory. Choose streaming or bounded reads based on input size and resource limits.

### Q4. What is buffering, and why does it help? *(Intermediate)*

Buffered streams/readers/writers reduce the number of low-level I/O operations by collecting data in memory and transferring it in larger chunks. This can improve throughput when individual reads or writes are expensive.

Buffering does not guarantee data is durable on disk; flush and filesystem durability semantics are separate concerns. Choose buffer sizes through measurement and understand when data must be flushed.

### Q5. What is the difference between flush and close? *(Intermediate)*

flush requests that buffered output be passed to the underlying destination. close releases the resource and usually performs required final flushing as part of closing. A closed stream generally cannot be used again.

A successful flush does not necessarily mean data has reached durable storage hardware. For durability-sensitive applications, understand the filesystem and database guarantees beyond Java's stream API.

## 2. Files and NIO

### Q6. What is the difference between File and Path? *(Intermediate)*

File is a legacy abstraction for file-system paths and operations. Path, introduced with NIO.2 in Java 7, represents a path in a file system and integrates with Files for richer operations.

Prefer Path and Files for new code. A Path is not necessarily an existing file, and path syntax and case sensitivity can depend on the file system.

### Q7. What does Files.copy or move guarantee? *(Advanced)*

The exact behavior depends on options, providers, and the underlying file system. Copy and move can fail due to permissions, missing parents, cross-filesystem constraints, or concurrent changes. ATOMIC_MOVE requests an atomic move where supported; it is not universally guaranteed.

Design for partial failure and test the actual deployment filesystem. Do not assume an atomic move is available across file systems or cloud-mounted storage.

### Q8. How should file paths from user input be validated? *(Senior)*

Resolve user-supplied paths against a trusted base directory, normalize them, and verify the result remains within the permitted root. Consider symbolic links and time-of-check/time-of-use races, not just string prefixes.

Avoid simple checks such as startsWith on an unnormalized raw string. For security-sensitive file access, define symlink policy and use filesystem-aware checks with appropriate permissions and race handling.

### Q9. What is a FileChannel? *(Advanced)*

FileChannel supports operations on a file using channels, including positional reads/writes, mapping, and force operations. It can support high-throughput and lower-copy patterns for certain workloads.

Channel APIs add complexity around buffers, partial reads/writes, and resource ownership. Do not assume one read fills the buffer; handle the returned byte count and end-of-stream correctly.

### Q10. What is a ByteBuffer? *(Advanced)*

ByteBuffer wraps a byte storage region and maintains position, limit, and capacity. flip switches from writing data into the buffer to reading the written data; clear resets it for writing again, while compact preserves unread bytes and makes room for more input.

Buffer bugs often come from misunderstanding position/limit. A buffer's capacity is not the number of valid bytes, and direct buffers have different allocation/lifecycle trade-offs from heap buffers.

### Q11. What is memory-mapped I/O? *(Advanced)*

A mapped byte buffer maps a file region into a process address space, allowing access through memory-like operations. It can be useful for large-file access patterns, but mapping does not mean the data is guaranteed resident in physical RAM or that writes are instantly durable.

Mapped files introduce platform and lifecycle considerations, including file size, access permissions, mapping lifetime, and synchronization. Benchmark against ordinary channel or stream I/O for the actual workload.

## 3. Serialization

### Q12. What is Java native serialization? *(Intermediate)*

Java serialization converts an object graph into a byte stream using Serializable and ObjectOutputStream, and reconstructs it using ObjectInputStream. The protocol supports object references and graph structure, but it couples persisted data to Java serialization semantics and class evolution.

For new distributed APIs, JSON, Protocol Buffers, Avro, or another explicitly versioned format is often a better choice. Native serialization should be used only when its compatibility and security model are understood.

### Q13. What is serialVersionUID? *(Intermediate)*

serialVersionUID is a version identifier used by Java serialization to check compatibility between the serialized stream and the local class. If not explicitly declared, the runtime may compute one from class details, making compatibility sensitive to changes.

Declare it intentionally for serializable classes where compatibility matters. It does not guarantee that every field or semantic change is compatible; it only participates in a limited version check.

### Q14. What do transient and static mean for serialization? *(Intermediate)*

Transient instance fields are excluded from default serialization. Static fields belong to the class, not to an individual serialized object, so they are not serialized as ordinary per-instance state. Custom serialization hooks can change how state is written and restored.

Never rely on transient alone as a complete secret-management strategy. Data may still appear in custom serialization, logs, or other copies.

### Q15. Why is deserializing untrusted Java data dangerous? *(Advanced)*

Native deserialization can instantiate classes and invoke behavior as an object graph is reconstructed. Gadget chains in the available classpath can turn attacker-controlled streams into code execution or other severe effects.

Avoid native Java deserialization for untrusted input. `ObjectInputFilter` was added in Java 9; on Java 9+, apply a strict allowlist and resource limits if deserialization cannot be avoided. Java 8 deployments do not have this standard filtering API, so the safest policy is to avoid deserializing untrusted streams. Keep dependencies patched and test the actual reachable class graph. Filtering is defense in depth, not a reason to accept arbitrary serialized data.

### Q16. What is the difference between Externalizable and Serializable? *(Advanced)*

Serializable relies on default object graph serialization unless custom hooks are provided. Externalizable gives the class explicit responsibility to write and read its serialized representation through public no-argument-constructor-based mechanics.

Externalizable offers control but increases the burden of maintaining version compatibility and invariants. Both use Java native serialization's security considerations; a custom format is not automatically safe simply because the class writes fields manually.

### Q17. What is serialVersionUID's limitation? *(Advanced)*

Matching serialVersionUID does not prove semantic compatibility. Fields may be added, removed, or changed in ways that preserve the identifier but alter meaning or invariants. Deserialization can also bypass ordinary constructor validation for serializable classes, subject to serialization rules.

Treat serialized form as a versioned contract. Validate restored state, use serialization proxy patterns where appropriate, and consider a schema-based external format for long-lived persistence.

## 4. Production I/O design

### Q18. How do you avoid reading an unbounded file into memory? *(Senior)*

Stream incrementally, enforce byte and record limits, and process records in bounded batches. readAllBytes and readString are convenient for known small inputs but can exhaust memory when input size is attacker-controlled or unexpectedly large.

Set timeouts for network reads, apply backpressure to downstream consumers, and define behavior for partial records. Memory usage should be bounded by explicit design rather than assumptions about file size.

### Q19. What is the difference between blocking and non-blocking I/O? *(Advanced)*

Blocking I/O waits for an operation to complete before the calling thread proceeds. Non-blocking channel operations may return immediately with fewer bytes or no data, requiring a readiness/selection mechanism or repeated state-driven operations.

Non-blocking I/O can support many connections with fewer threads but increases state-machine complexity. Choose based on workload and framework support; asynchronous APIs do not eliminate the need for timeouts, cancellation, and backpressure.

### Q20. How should I/O errors be handled in a backend service? *(Senior)*

Distinguish missing resources, permission failures, invalid input, capacity exhaustion, transient network issues, timeouts, and data corruption. Preserve causes, include safe context, and decide whether retry is safe. A timeout may leave the outcome unknown if a remote write partially succeeded.

Use try-with-resources, explicit encoding, bounded input sizes, and structured logs. Test failures such as disk full, permission denied, truncated input, slow reads, and close failure rather than only the happy path.

## 5. File traversal and asynchronous I/O

### Q21. How should files be traversed with Files.walk? *(Advanced)*

Files.walk returns a lazy Stream of paths and can keep directory resources open while the stream is in use. Close it with try-with-resources, filter paths deliberately, and handle I/O failures rather than assuming every entry is accessible.

~~~java
try (Stream<Path> paths = Files.walk(root)) {
    paths.filter(Files::isRegularFile)
         .forEach(this::processPath);
}
~~~

For untrusted trees, define symlink policy, maximum depth, file-size limits, and behavior when files change during traversal. Avoid collecting an unbounded tree into a list when streaming is sufficient.

### Q22. What is WatchService, and is it a durable file-change log? *(Advanced)*

WatchService lets a program register directories for events such as entry creation, deletion, and modification. It is useful for reacting to changes, but event delivery and details can vary by file-system provider; events may be coalesced or overflow, and some network or virtual file systems have limitations.

After processing a WatchKey, call reset so it can receive further events, and handle invalid keys and OVERFLOW. Treat notifications as hints to reconcile current directory state, not as a durable, exactly-once journal of every file operation.

### Q23. What is AsynchronousFileChannel? *(Advanced)*

AsynchronousFileChannel supports file reads and writes whose completion can be observed through a Future or CompletionHandler. It can let a caller continue while an operation is pending, but the underlying implementation and executor behavior depend on the platform and provider.

Asynchronous APIs still require bounded concurrency, buffer-lifetime discipline, cancellation handling, and resource closure. Do not assume “async” means zero threads or that every storage device provides native asynchronous operations.

## Official references

1. [Java SE 21 API — java.io](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/io/package-summary.html)
2. [Java SE 21 API — java.nio.file](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/file/package-summary.html)
3. [Java SE 21 API — ByteBuffer](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/ByteBuffer.html)
4. [Java SE 21 API — ObjectInputFilter](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/io/ObjectInputFilter.html)
5. [JEP 290 — Filter Incoming Serialization Data](https://openjdk.org/jeps/290)

## Chapter checklist

- [ ] Choose byte versus character streams and specify encoding.
- [ ] Manage resources and bound input sizes.
- [ ] Understand Path, channels, and ByteBuffer state.
- [ ] Treat native deserialization of untrusted data as a serious security risk.
- [ ] Design file access and I/O retries around real failure modes.
