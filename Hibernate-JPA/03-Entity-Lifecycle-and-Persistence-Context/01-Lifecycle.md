# Entity Lifecycle and Persistence Context

## Q1. What is the difference between persist() and merge()?
**Must Know**

persist makes a new entity managed. merge copies the supplied object's state into a managed instance and returns that instance; the original object does not become managed merely because merge was called. Continue with the returned object after merge.

## Q2. What is the difference between find() and getReference()?
**High Value**

find returns the entity or null and can consult the persistence context first. getReference can return a proxy/reference without immediately loading state; access may trigger loading or an entity-not-found failure. Proxy behavior depends on provider and mapping.

## Q3. What do detach(), clear(), and refresh() do?
**High Value**

detach stops tracking one entity; clear detaches all managed entities; refresh reloads database state and overwrites local changes for that entity. Flush first if pending changes must be saved before clearing. Refresh can discard unsaved edits.

## Q4. Why can a managed entity update without save()?
**Must Know**

The persistence context tracks managed state. At flush, Hibernate detects changed values and issues SQL. In a Spring transaction, loading an entity, modifying it, and completing the transaction can be enough. A successful flush/commit is still required.

## Q5. What is flush mode?
**Advanced**

JPA defines AUTO and COMMIT semantics; Hibernate may offer provider-specific modes. AUTO requires query results to reflect relevant pending changes, potentially by flushing before a query. COMMIT defers synchronization but does not guarantee that no earlier flush occurs. Verify exact provider behavior before tuning.

## Q6. Why does modifying a detached entity not persist?
**Must Know**

Dirty checking tracks managed entities only. Reload the managed entity and apply intended changes, or merge detached state deliberately. Merging partially populated objects can overwrite fields or relationships unintentionally.

## Q7. What does refresh() solve?
**High Value**

Refresh discards local changes and reloads an entity from the database, useful when database-side operations or external updates make state stale. Related entities may need separate refreshes, and transaction isolation or caches can affect what is visible.

## Q8. What are entity lifecycle callbacks?
**Advanced**

Callbacks such as PrePersist, PostPersist, PreUpdate, PostUpdate, and PreRemove run at defined lifecycle points. Keep them small and entity-local; do not hide service workflows or complex database operations inside callbacks. Callback timing is not the same as transaction commit timing.

## Q9. Why can flush fail when setters succeeded?
**Must Know**

Setters usually change memory. SQL may execute at flush and fail due to uniqueness, foreign-key, not-null, lock, or other database errors. Once a persistence failure occurs, roll back rather than assuming the same transaction/context can safely continue.

## Q10. How can a persistence context cause memory pressure?
**High Value**

Managed entities and tracking state remain associated with the context until detached or closed. Large batch jobs can retain many entities. Process bounded chunks and flush/clear at safe points, preserving the required atomicity semantics.

## Official references
- [Jakarta Persistence API](https://jakarta.ee/specifications/persistence/)
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
