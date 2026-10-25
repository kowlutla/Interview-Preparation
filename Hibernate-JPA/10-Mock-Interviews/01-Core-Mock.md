# Mock Interview 1 — Core JPA and Hibernate

Keep initial answers to 2–3 minutes, then defend assumptions.

## Q1. Explain JPA versus Hibernate.
JPA is the Jakarta Persistence specification; Hibernate implements it and adds provider-specific features. Portability does not imply identical SQL or performance.

**Follow-up:** What would you isolate to ease a provider migration?

## Q2. Explain the persistence context.
It tracks managed entities, enforces identity within the context, supports dirty checking, and synchronizes changes at flush.

**Follow-up:** Is it the same as a database transaction?

## Q3. Explain entity lifecycle states.
Transient, managed, detached, and removed describe the relationship between an object and a persistence context. Detachment is not deletion.

**Follow-up:** What happens if a detached object changes?

## Q4. Compare persist and merge.
Persist makes a new entity managed. Merge copies state to a managed instance and returns it; the original may remain detached.

**Follow-up:** Which instance should be used after merge?

## Q5. Flush versus commit?
Flush executes pending SQL; commit completes the transaction. Flushed statements can still roll back.

**Follow-up:** When can a flush happen automatically?

## Q6. Explain owning side and mappedBy.
The owning side controls relationship updates. mappedBy points to the owning attribute on the inverse side.

**Follow-up:** Why can changing only the collection fail?

## Q7. Cascade versus orphan removal?
Cascade propagates selected operations; orphan removal deletes privately owned children removed from the relationship.

**Follow-up:** Why is cascading REMOVE dangerous for shared entities?

## Q8. Why is dirty checking useful?
Managed changes are detected at flush, so explicit update calls are unnecessary. It depends on managed state and successful transaction completion.

**Follow-up:** How can dirty checking affect batch jobs?

## Q9. Explain lazy loading.
Associations load on access if the context can initialize them. Access after the context closes can fail.

**Follow-up:** Why isn't EAGER the universal fix?

## Q10. Explain N+1.
One root query is followed by association queries for each root. Diagnose counts and select a use-case fetch plan.

**Follow-up:** Compare fetch joins, entity graphs, and batch fetching.

## Q11. What does Spring Data JPA add?
Repository abstractions, query derivation, projections, pagination, and Spring integration; the provider still controls ORM behavior.

**Follow-up:** What does save() do internally?

## Q12. Why are entity equals/hashCode tricky?
Generated IDs and mutable fields can destabilize hash collections. Choose an identity strategy and test lifecycle/proxy cases.

**Follow-up:** How can proxies affect equality?

## Evaluation checklist
- Explains mechanisms rather than memorized definitions.
- Separates specification requirements from Hibernate behavior.
- Describes SQL timing and transaction boundaries accurately.
- Gives practical trade-offs without overgeneralizing.
