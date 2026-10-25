# Mock Interview 1 — Core JPA and Hibernate

Use these prompts to practise speaking naturally. Aim for a two-minute initial answer, then handle the follow-up without turning the response into a memorized definition.

## Q1. Explain JPA versus Hibernate.
I'd say JPA is the standard contract for persistence in Java, while Hibernate is an ORM provider that implements that contract and offers extra features. Using JPA APIs improves portability, although generated SQL and performance can still vary.

**Follow-up:** What would you isolate to make a provider migration easier?

## Q2. Explain the persistence context.
I think of it as Hibernate's managed workspace. It tracks entities, preserves identity within the context, and detects changes so they can be synchronized at flush. It is related to a transaction but isn't the same thing as the database transaction.

**Follow-up:** What happens to dirty checking when an entity becomes detached?

## Q3. Explain entity lifecycle states.
I distinguish the states by whether the persistence context tracks the object: transient objects are new and unmanaged; managed objects are tracked; detached objects are no longer tracked; removed objects are scheduled for deletion.

**Follow-up:** Does detaching an entity delete its database row?

## Q4. Compare persist and merge.
I use persist to make a new entity managed. Merge copies state into a managed instance and returns that instance; the object passed to merge doesn't itself become managed just because I called it.

**Follow-up:** Which instance would you continue working with after merge?

## Q5. Flush versus commit?
Flush sends pending SQL to the database within the transaction. Commit completes the transaction. If the transaction later rolls back, flushed changes can still be undone.

**Follow-up:** When might Hibernate flush automatically?

## Q6. Explain owning side and mappedBy.
The owning side controls the relationship update in the database. mappedBy points from the inverse side to the owning association attribute. In a common parent-child mapping, setting only the parent's collection may not update the foreign key.

**Follow-up:** How would you prevent inconsistent references in memory?

## Q7. Cascade versus orphan removal?
Cascade propagates selected entity operations from one entity to related entities. Orphan removal schedules deletion when a privately owned child is removed from the relationship. I only use orphan removal when the child really has no independent lifecycle.

**Follow-up:** Why can cascading REMOVE be dangerous for shared objects?

## Q8. Why is dirty checking useful?
It lets Hibernate detect changes to managed entities at flush, so I don't need to call update explicitly after every setter. The key requirement is that the entity remains managed and the transaction completes successfully.

**Follow-up:** How could dirty checking affect a large batch job?

## Q9. Explain lazy loading.
A lazy association is loaded when it is accessed, provided the persistence context can initialize it. If the context has closed, Hibernate may throw LazyInitializationException. I plan the fetch strategy around each use case rather than making everything EAGER.

**Follow-up:** How would you load the required data before returning a REST response?

## Q10. Explain N+1.
I get N+1 when one query loads the roots and additional queries load related data for individual roots. I confirm it by inspecting statement counts, then choose a fetch join, entity graph, batch fetching, or DTO projection depending on the relationship and response shape.

**Follow-up:** What makes collection fetch joins risky with pagination?

## Q11. What does Spring Data JPA add?
It reduces repository boilerplate with query derivation, projections, pagination, sorting, and Spring integration. It still relies on the JPA provider for persistence behavior, so I need to understand what Hibernate is doing underneath.

**Follow-up:** What does save() decide internally?

## Q12. Why are entity equals() and hashCode() tricky?
Generated IDs can change from null to assigned, and mutable fields can destabilize hash collections. I choose an identity strategy based on the domain and test transient, managed, detached, and proxy cases rather than copying a generic implementation.

**Follow-up:** How might Hibernate proxies affect equality?

## Evaluation checklist
- Answers sound conversational and explain mechanisms, not just labels.
- The candidate distinguishes JPA specification behavior from Hibernate-specific behavior.
- SQL timing and transaction boundaries are described accurately.
- Examples are practical and trade-offs are acknowledged without overgeneralizing.
