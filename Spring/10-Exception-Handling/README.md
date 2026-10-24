# Exception Handling and API Error Design

## Q&A

### Q1. What is the role of @ControllerAdvice?

It centralizes cross-controller concerns such as exception-to-response mapping and binding behavior. `@RestControllerAdvice` combines advice semantics with response-body handling. Keep handlers specific enough to preserve meaningful status codes and avoid turning unexpected defects into misleading client errors.

### Q2. When should an exception become HTTP 400, 404, 409, or 500?

Use 400 for invalid request syntax or input according to the API contract, 404 when the requested resource is not found, 409 for a conflict with current resource state, and 500 for unexpected server failures. The correct status depends on the semantics and must remain consistent across endpoints.

### Q3. What is ProblemDetail?

In supported Spring Framework versions, `ProblemDetail` represents an RFC 7807-style problem response. It can provide a standard type/title/status/detail/instance shape with extensions. Verify version support and serialization behavior, and do not expose stack traces or internal exception messages to clients.

### Q4. Should controllers catch every exception?

No. Catch exceptions when the layer can meaningfully recover, translate, or add context. A global boundary can map known failures consistently, while unexpected failures should be logged with correlation context and returned as safe generic errors. Swallowing exceptions or returning HTTP 200 for failed operations breaks client behavior and observability.

### Q5. How should validation errors be represented?

Return a stable structure with a general error code and field-level messages that are safe to disclose. Avoid returning rejected secrets or full request bodies. Keep validation messages suitable for API clients and do not make clients parse framework exception class names.

### Q6. How do downstream service failures affect API error mapping?

Distinguish invalid client input from temporary dependency failures, timeouts, and unavailable downstream systems. Map them according to the API contract, preserve retryability information where appropriate, and avoid exposing internal hostnames. Pair error responses with bounded timeouts, metrics, and trace correlation.

### Q7. How do you preserve diagnostic context without leaking sensitive details?

Log the exception and relevant safe identifiers at the boundary where it can be understood; include a trace or correlation ID in logs and responses. Redact tokens and personal data, avoid logging the same exception at every layer, and keep client-facing details separate from internal diagnostics.

## Official references

- [Spring MVC exception handling](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html)
- [Problem Details](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)
