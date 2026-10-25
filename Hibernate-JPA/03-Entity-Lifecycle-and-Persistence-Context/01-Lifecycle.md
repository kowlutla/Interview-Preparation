# Entity Lifecycle and Persistence Context

## Q1. What is the difference between persist() and merge()?
**Must Know**

I use `persist()` when I want a new entity instance to become managed. I use `merge()` when I need the state of an entity object to be copied into a managed instance. The subtle but important point is that merge returns the managed instance; it does not make the original detached object managed again.

For example, if I merge a detached Product, I continue working with the object returned by merge. This distinction matters when changes are made after the merge call or when the returned instance contains state loaded by the provider.

## Q2. What is the difference between find() and getReference()?
**High Value**

With `find()`, I ask the EntityManager to return the entity for an identifier, or null if it does not exist. The persistence context can satisfy the request without a database query if the entity is already managed. With `getReference()`, I can obtain a reference without immediately loading all state, which is useful when I only need to assign an association by ID.

I don't treat getReference as a promise that no SQL will occur. Accessing the proxy's state can trigger a load, and behavior around a missing row depends on provider and timing.

## Q3. What do detach(), clear(), and refresh() do?
**High Value**

I use `detach(entity)` to stop tracking one entity, and `clear()` to detach everything in the persistence context. `refresh(entity)` reloads the entity's state from the database, overwriting local changes to that entity.

A common batch-processing mistake is to clear the context before flushing pending changes. If those changes must be saved, I flush first, then clear at a deliberate boundary. I also use refresh carefully because it can discard unsaved edits.

## Q4. Why can a managed entity update without calling save()?
**Must Know**

Once I load an entity into the current persistence context, Hibernate tracks its state. If I modify a field inside a transaction, dirty checking detects the change and schedules an update during flush. For that already-managed entity, calling Spring Data's save method again is usually unnecessary.

That doesn't mean the update is guaranteed regardless of context. The entity must be managed, the transaction must complete successfully, and the flush must succeed.

## Q5. What is flush mode?
**Advanced**

Flush mode controls when the persistence provider synchronizes pending changes with the database. With JPA's AUTO mode, the provider must ensure queries see relevant pending changes, which can mean flushing before a query. COMMIT generally defers synchronization toward transaction completion, but it should not be interpreted as an absolute guarantee that no earlier flush occurs.

I would tune flush behavior only after understanding the query semantics and measuring the effect. Provider-specific modes need to be checked against the Hibernate version in use.

## Q6. Why doesn't changing a detached entity update the database?
**Must Know**

Dirty checking only tracks entities managed by a persistence context. If I load an entity, close the context, and then change a field on that Java object, I've changed memory—not the database.

I can load the current managed entity and apply the intended changes, or deliberately merge detached state. I am cautious about merging partially populated objects because missing fields or relationships may unintentionally overwrite existing state.

## Q7. When would you use refresh()?
**High Value**

I use refresh when I need to discard the current in-memory state and reload an entity from the database—for example, after a database-side operation makes my managed instance stale. Refresh can also discard my unsaved changes, so I call it only when that is intended.

If the returned state still looks old, I investigate transaction isolation, context lifetime, and cache behavior rather than assuming refresh always bypasses every layer.

## Q8. What are entity lifecycle callbacks used for?
**Advanced**

Callbacks such as `@PrePersist`, `@PreUpdate`, and `@PreRemove` let me run small pieces of logic at defined lifecycle points. They can be useful for simple entity-local rules, but I avoid putting service workflows or complicated database operations inside callbacks because the behavior becomes harder to reason about and test.

I also distinguish callback timing from transaction commit: a callback can run before the transaction has successfully completed.

## Q9. Why can flush fail even when my setters work?
**Must Know**

A setter normally changes an in-memory value; it does not prove that the value satisfies every database constraint. When Hibernate flushes, it may execute SQL and encounter a uniqueness, foreign-key, not-null, or optimistic-locking failure.

When that happens, I inspect the deepest database exception and roll back the failed transaction. I don't catch the error and assume the same persistence context can safely continue.

## Q10. How can the persistence context cause memory pressure?
**High Value**

A persistence context retains managed entities and their tracking state. If a batch job loads or inserts tens of thousands of entities in one context, memory usage can grow and flush processing can become expensive.

For large jobs, I process bounded chunks and flush/clear at safe points. I first decide whether the business operation must be fully atomic, because committing each chunk changes the rollback behavior.

## Official references
- [Jakarta Persistence API](https://jakarta.ee/specifications/persistence/)
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
