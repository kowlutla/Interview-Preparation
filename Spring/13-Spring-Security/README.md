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

## Official references

- [Spring Security reference](https://docs.spring.io/spring-security/reference/)
- [Servlet architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
- [OAuth 2.0 Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
