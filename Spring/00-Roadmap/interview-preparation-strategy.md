# Spring Interview Preparation Strategy

## Answering framework questions

Use this sequence for a strong answer:
1. **Definition:** State what the abstraction does.
2. **Mechanism:** Explain the container, proxy, filter, transaction manager, or runtime path involved.
3. **Example:** Give a concise code/configuration example when useful.
4. **Failure mode:** Describe a common pitfall and how it appears.
5. **Trade-off:** Explain when the approach is appropriate and what it costs.
6. **Version note:** Mention the baseline if APIs or behavior differ.

## Prioritize depth over annotation memorization

- @Transactional, @Async, @Cacheable, and many AOP features are commonly applied through proxies; self-invocation can bypass interception.
- A singleton bean is one instance per bean definition per container, not automatically thread-safe.
- Boot auto-configuration is conditional and often backs off when application-defined beans are present.
- A transaction is scoped to a transaction manager and resource model; it does not automatically make a database write and a remote message publish atomic.
- Authentication and authorization are distinct decisions.
- A reactive return type does not make blocking I/O non-blocking.
- A successful health endpoint is only as meaningful as the checks and operational semantics configured behind it.

## Practice format

For each topic, practice a 30-second summary, a 2-minute mechanism explanation, a production debugging example, an alternative design with trade-offs, and one follow-up question that changes a constraint.

## Readiness rubric

You are interview-ready when you can explain both happy and failure paths, write a small tested implementation, identify relevant official documentation, and state assumptions clearly. Do not claim hands-on experience with a subsystem merely because you can define its annotations.
