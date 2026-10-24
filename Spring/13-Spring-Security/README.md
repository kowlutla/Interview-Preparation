# Spring Security

## Q&A

### Q1. What is the difference between authentication and authorization?

Authentication establishes the identity or principal associated with a request. Authorization decides whether that principal may perform an action on a resource. A valid login or JWT does not automatically imply permission to access every endpoint or tenant's data.

### Q2. What is SecurityFilterChain?

SecurityFilterChain defines servlet filters and request-matching security rules applied to matching requests. Modern configuration commonly declares one or more SecurityFilterChain beans instead of extending the removed WebSecurityConfigurerAdapter pattern. Multiple chains require deliberate ordering and matchers because the first matching chain controls processing.

### Q3. What is the difference between 401 and 403?

401 indicates that the request lacks acceptable authentication credentials or authentication failed. 403 indicates the request was understood but access is denied for the current principal, although exact responses can be customized. A configured entry point and access-denied handler influence the response behavior.

### Q4. How should a REST API validate JWTs?

Prefer Spring Security's OAuth 2.0 Resource Server support and configure trusted issuer or JWK settings rather than hand-writing token parsing. Validate signature and relevant claims such as issuer, expiry, and audience when required by the trust model. A decoded JWT is not trustworthy until cryptographic and claim validation succeeds.

### Q5. What is SecurityContextHolder?

It provides access to the current SecurityContext, which holds authentication information. The default servlet strategy is typically thread-bound; asynchronous execution and reactive pipelines have different context-propagation considerations. Do not assume security context automatically follows arbitrary executor tasks.

### Q6. What is the difference between roles and authorities?

Authorities are granted permission strings. Role-based checks are a convention that often adds a ROLE_ prefix when using role-oriented APIs. Be consistent about prefixes and use authorities that express the actual permission needed rather than overloading broad roles for every decision.

### Q7. Why does CSRF protection matter, and when should it remain enabled?

CSRF protection mitigates cross-site request forgery when browsers automatically attach credentials such as session cookies. Stateless APIs using non-ambient bearer tokens have a different threat profile, but disabling CSRF should follow a deliberate authentication and browser-client analysis, not a blanket rule for every endpoint called REST.

### Q8. What are CORS and CSRF, and why are they not interchangeable?

CORS is a browser-enforced cross-origin access policy; CSRF is an attack in which a victim's browser is induced to submit an authenticated request. CORS is not a general server-side authorization mechanism and does not replace CSRF defenses.

### Q9. How should method-level security be enabled and used?

Enable method security using the supported configuration for the selected version and apply annotations such as @PreAuthorize at appropriate service boundaries. Because interception is proxy-based in common setups, self-invocation and proxy visibility matter. Authorization rules should be tested for both allowed and denied principals.

### Q10. How should multiple SecurityFilterChain beans be ordered?

Each chain has a request matcher and an order. The first matching chain is used, so a broad high-priority matcher can shadow a more specific chain. Keep matchers explicit and test both intended and unintended paths.

### Q11. What is the difference between UserDetailsService and AuthenticationProvider?

UserDetailsService loads user-related data for authentication flows that use it; AuthenticationProvider performs an authentication strategy and returns an authenticated result or rejects the attempt. They are extension points with different responsibilities, not interchangeable service interfaces.

### Q12. How should passwords be stored?

Store password hashes using a supported adaptive password encoder, such as the framework's delegating encoder configuration, and never store plaintext or reversible encrypted passwords as a substitute. Choose parameters appropriate to the deployment and plan migration when algorithms or parameters change.

### Q13. What is the purpose of a PasswordEncoder?

It defines how passwords are encoded and how a submitted password is checked against a stored encoded value. Use an adaptive password-hashing algorithm with cost parameters appropriate to the deployment; never decrypt a password or store plaintext. A delegating encoder can identify the algorithm for each stored hash and support gradual upgrades when the preferred algorithm or work factor changes.

### Q14. What is method security's proxy limitation?

Method security commonly uses Spring AOP interception. A direct self-call may bypass the proxy, and non-proxy-compatible methods may not be intercepted as expected. Put authorization at a clear service boundary and test the invocation path.

### Q15. Should authorization exist only at the controller layer?

No. Controller rules protect HTTP entry points, but business operations may also be invoked from jobs, messaging listeners, or other services. Enforce domain/resource permissions at a boundary that covers every relevant caller.

### Q16. How should multi-tenant authorization be enforced?

Derive tenant identity from trusted authentication data, verify resource ownership at the service/data boundary, and include tenant scope in cache keys and background-job context. Never rely only on a client-supplied tenant identifier.

### Q17. What is session fixation protection?

It prevents an attacker from retaining a known session identifier across authentication. Use the framework's session-protection mechanisms and test session ID changes and logout/session invalidation behavior for the selected authentication mode.

### Q18. How do authentication entry points differ from access-denied handlers?

An AuthenticationEntryPoint handles a request that requires authentication but lacks acceptable authentication. An AccessDeniedHandler handles an authenticated principal that is not authorized. Their response contract should be tested for API clients.

### Q19. What should security tests cover beyond a happy path?

Cover anonymous access, authenticated-but-forbidden access, authorized access, malformed/expired tokens, wrong issuer or audience, authority mapping, CSRF where relevant, and tenant isolation. A successful test alone does not establish that forbidden requests are rejected.

## Official references

- [Spring Security reference](https://docs.spring.io/spring-security/reference/)
- [Servlet architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
- [OAuth 2.0 Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
