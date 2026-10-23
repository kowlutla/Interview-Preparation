# Networking and Sockets

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Java networking spans URI/URL handling, sockets, HTTP clients, timeouts, TLS, and asynchronous I/O. Production networking requires explicit deadlines, resource limits, error classification, and security boundaries. A timeout or connection reset does not always tell the caller whether a remote side effect occurred.

## 1. Network fundamentals

### Q1. What is the difference between URI and URL? *(Intermediate)*

URI is a general identifier syntax; URL is a locator that identifies a resource by how it can be accessed. In Java, URI supports structured parsing and validation, while URL has protocol-handler behavior and may perform resolution or connection-related operations in some methods.

Use URI for parsing and constructing identifiers and convert to URL when a URL-specific API requires it. Avoid treating string concatenation as safe URI construction; components require correct encoding.

### Q2. What is a Socket? *(Beginner)*

Socket represents one endpoint of a network connection, commonly TCP. Socket APIs provide input/output streams and connection lifecycle operations. ServerSocket accepts incoming connections in traditional blocking server designs.

Network reads and writes can block, return partial data, or fail. Define framing, timeouts, connection lifecycle, and how partial messages are handled.

### Q3. What is the difference between TCP and UDP from Java's perspective? *(Intermediate)*

TCP provides a reliable ordered byte stream; application code still needs message framing because TCP does not preserve message boundaries. UDP sends datagrams without TCP's delivery/order guarantees and has packet-size and network-loss considerations.

Choose based on protocol requirements, not merely performance assumptions. Retries, ordering, congestion, and idempotency must be handled at the appropriate layer.

### Q4. Why does a TCP read not necessarily return a complete message? *(Intermediate)*

TCP is a byte stream, not a message protocol. A read can return fewer bytes than requested, even when more bytes will arrive later. A protocol must define framing, such as fixed lengths, delimiters with escaping, or length-prefixed messages.

Implement loops that read the full frame or detect end-of-stream. Do not assume one read call corresponds to one application request.

## 2. HTTP and timeouts

### Q5. What is Java's standard HttpClient? *(Intermediate)*

java.net.http.HttpClient became a standard API in Java 11. It supports synchronous send, asynchronous sendAsync, HTTP/2, WebSocket, and configurable client behavior.

Production clients need connect/request timeouts, TLS policy, redirect behavior, body handling, cancellation, and observability. Async requests still consume resources and need concurrency limits.

### Q6. What is the difference between connect timeout and request timeout? *(Intermediate)*

Connect timeout bounds the time spent establishing a connection. Request timeout bounds the duration of a request according to the client API's semantics. A request may also spend time waiting for connection-pool resources or response-body consumption depending on the library.

Use an end-to-end deadline across retries and downstream calls. Independent per-hop timeouts can accidentally exceed the caller's total latency budget.

### Q7. Why is retrying an HTTP request not always safe? *(Advanced)*

A timeout or connection reset can happen after the server commits a side effect but before the client receives the response. Retrying a non-idempotent request can create duplicate orders, payments, or messages.

Use idempotency keys, deduplication, or a reconciliation protocol. Retry only classified transient failures with bounded attempts, backoff, jitter, and a total deadline.

### Q8. What is TLS certificate validation? *(Advanced)*

TLS authenticates the server (and optionally the client) using certificates and trust validation, while providing confidentiality and integrity for the connection. Certificate validation includes chain trust, validity, and hostname verification according to the client configuration.

Do not disable hostname verification or trust all certificates to “fix” connectivity. Configure the trust store and certificates correctly, and log enough diagnostic information without exposing secrets.

## 3. Blocking, asynchronous and NIO

### Q9. What is the difference between blocking and non-blocking I/O? *(Advanced)*

Blocking I/O waits for operations to complete from the caller's perspective. Non-blocking channels can return immediately when no data or capacity is available, requiring readiness selection or an event-driven state machine.

Non-blocking I/O can scale to many connections but adds complexity. Frameworks often handle the state machine; avoid blocking calls on event-loop threads.

### Q10. What is Selector in NIO? *(Advanced)*

Selector allows a thread to monitor multiple selectable channels for readiness events such as accept, connect, read, or write. It underpins many event-driven networking designs.

Correct selector code must handle partial reads/writes, key cancellation, interest-op updates, wakeups, and connection lifecycle. Use a mature networking framework unless implementing the event loop is a deliberate requirement.

### Q11. How do virtual threads change blocking network code? *(Advanced)*

Virtual threads can make many blocking network tasks cheaper by allowing the runtime to unmount the virtual thread during supported blocking operations. This can simplify thread-per-request code without manually composing callbacks for every I/O wait.

The network, remote service, and connection pool remain finite. Use timeouts, backpressure, and resource limits; verify behavior of native libraries and drivers.

## 4. Production scenarios

### Q12. How do you design a resilient HTTP client? *(Senior)*

Define connect and request deadlines, bounded retries for safe operations, exponential backoff with jitter, idempotency behavior, response-body limits, connection reuse, TLS verification, and metrics. Decide which errors are permanent and which may be transient.

Do not retry authentication/validation failures blindly. Keep a total deadline so retries do not turn a short request into a long-running one.

### Q13. How should large HTTP response bodies be handled? *(Senior)*

Stream the body or process it in bounded chunks when the content can be large. Reading the entire response into memory can cause high allocation and OOM under load. Enforce maximum body sizes for untrusted endpoints.

Ensure the response body is fully consumed or closed according to the client API so connections can be reused. Handle cancellation and partial content explicitly.

### Q14. What metrics help diagnose network latency? *(Senior)*

Measure DNS resolution where applicable, connection acquisition, connect/TLS handshake, time to first byte, response-body duration, retries, timeout rate, connection pool utilization, and downstream status codes. Correlate with tracing and server-side metrics.

A single “HTTP duration” metric may hide where time is spent. Instrument the stages that distinguish local saturation from network or remote-service delay.

### Q15. How should URL or URI input be validated? *(Senior)*

Parse into structured components, allowlist schemes and hosts where appropriate, and validate resolved addresses and redirect behavior for SSRF-sensitive requests. Avoid relying on string-prefix checks. Consider DNS rebinding, IPv6 forms, user-info, alternate encodings, and redirects.

Do not let untrusted URLs reach internal metadata endpoints or privileged services without a deliberate policy. Validation must align with the actual network connection target, not just the initial string.

## Official references

1. [Java SE 21 API — java.net](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/net/package-summary.html)
2. [Java SE 21 API — java.net.http](https://docs.oracle.com/en/java/javase/21/docs/api/java.net.http/java/net/http/package-summary.html)
3. [Java SE 21 API — Socket](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/net/Socket.html)
4. [Java SE 21 API — Selector](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/channels/Selector.html)

## Chapter checklist

- [ ] Understand TCP stream framing and partial reads/writes.
- [ ] Configure deadlines and retry policies deliberately.
- [ ] Preserve TLS verification and validate untrusted destinations.
- [ ] Distinguish blocking and non-blocking I/O.
- [ ] Bound response sizes and instrument network stages.
