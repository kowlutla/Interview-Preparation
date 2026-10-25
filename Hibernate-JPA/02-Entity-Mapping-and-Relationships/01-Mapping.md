# Entity Mapping and Relationships

## Q1. What makes a class a JPA entity?
**Must Know**

I use `@Entity` to tell the persistence provider that a class represents persistent data. The entity must satisfy the requirements of the Jakarta Persistence version in use, including having an identifier and meeting the relevant construction/access rules. I also try to keep entities focused on domain persistence rather than using them as REST request and response objects.

If the database schema doesn't match the default naming strategy, I make the table and column mappings explicit so the mapping remains understandable.

## Q2. How do you choose an identifier-generation strategy?
**High Value**

I choose the strategy based on the database, schema, insert volume, and batching requirements—not just because one annotation is familiar. With sequence-based generation, supported databases can allocate identifiers efficiently and still allow batching. With identity generation, the provider may need to execute an insert earlier to obtain the generated key, which can limit insert batching. `AUTO` delegates the choice to the provider and dialect, so it may not behave identically across databases.

In a high-volume insert path, I verify the generated SQL and actual batching behavior before deciding which strategy is best.

## Q3. What is the owning side of a relationship?
**Must Know**

The owning side is the side Hibernate uses to determine how the relationship is updated in the database. In a typical bidirectional one-to-many/many-to-one mapping, the `@ManyToOne` field owns the foreign key, while the parent's `@OneToMany` collection uses `mappedBy`.

For example, adding an order to a customer's collection is not enough if the owning `order.customer` reference remains null. I set the owning side and keep both Java references consistent, usually through a helper method.

## Q4. What does mappedBy mean?
**Must Know**

I use `mappedBy` on the inverse side of a bidirectional relationship to point to the association field on the owning entity. It names a Java attribute, not the database column. For example, `mappedBy = "customer"` points to the field named `customer` in the child entity.

If the value doesn't match the mapped attribute, application startup may fail. If I update only the inverse side at runtime, the in-memory graph may look right while the foreign key remains unchanged.

## Q5. How are cascade and orphanRemoval different?
**Must Know**

Cascade means that selected entity operations on a parent propagate to related entities. For example, cascading persist can make a new child persistent when I persist the parent. Orphan removal is different: when a privately owned child is removed from the relationship, the provider schedules that child for deletion.

I use orphan removal only when the child truly belongs to that parent and has no independent lifecycle. I am careful with cascade remove on shared relationships, because deleting one parent should not unexpectedly delete an object still used elsewhere.

## Q6. When would you use OneToOne, OneToMany, ManyToOne, or ManyToMany?
**High Value**

I choose the mapping that reflects the relational model and lifecycle of the domain. ManyToOne is common for a foreign key from a child to a parent; OneToMany represents the parent's collection. OneToOne represents a unique association, often enforced by a unique foreign key. ManyToMany can work when the join has no meaningful attributes and both sides share the relationship.

If the join itself has fields such as role, status, or creation time, I usually model it as a separate association entity. That makes its lifecycle and constraints explicit and is easier to extend later.

## Q7. Why is LAZY often preferable to EAGER?
**Must Know**

I generally prefer LAZY for associations because different use cases need different amounts of data. EAGER means the association must be available, but it does not guarantee that Hibernate will fetch it in one join. Depending on the query and mapping, the provider may issue secondary selects, which can cause N+1 behavior.

Lazy loading still needs planning: if I access an uninitialized association after the persistence context closes, Hibernate may throw a LazyInitializationException. I define the fetch plan for the use case instead of making everything EAGER.

## Q8. Why are equals() and hashCode() tricky for entities?
**High Value**

The main difficulty is that an entity's identity can change during its lifecycle. A generated ID may be null before persistence and assigned afterward. If I use that ID naïvely to calculate a hash code, the hash code can change after the entity has been inserted into a HashSet, making lookups unreliable. Mutable business fields create similar problems.

I choose an identity strategy deliberately and test transient, managed, detached, and proxy cases. A stable, genuinely unique natural key can work in some domains; generated-ID strategies need care. I would not apply one generic equals/hashCode implementation to every entity without considering the model.

## Q9. How do you map a composite primary key?
**Advanced**

When the relational key is genuinely composite, I can use `@EmbeddedId` with an embeddable key class or `@IdClass`. The key type needs equality and hash-code behavior consistent with the database key. Composite IDs can complicate associations, repository signatures, and query code, so I prefer a surrogate key when it fits the domain and schema rather than introducing a composite key without a reason.

## Q10. What is the difference between JoinColumn and JoinTable?
**High Value**

A `@JoinColumn` typically maps a foreign-key column in one of the related tables. A `@JoinTable` maps an intermediate table that contains join columns pointing to the participating tables. I choose based on the actual relational design and cardinality, not on a blanket rule that a particular annotation belongs to one relationship type.

When debugging, I compare the annotations with the actual schema and inspect the generated SQL to confirm which table and columns Hibernate is using.

## Q11. Why can a bidirectional relationship look correct in Java but not persist correctly?
**Must Know**

The usual cause is that the application updated only the inverse side. In a common parent-child mapping, I need to set the child's owning-side reference, not just add the child to the parent's collection. I normally create helper methods that update both sides so the in-memory object graph stays consistent.

I also check whether a new child needs a cascade operation and then inspect the SQL and foreign-key value. Cascade alone does not automatically fix an incorrectly maintained owning side.

## Q12. Should I return JPA entities directly from REST controllers?
**High Value**

I generally avoid doing that. Serializing an entity can trigger lazy loads, expose fields that should not be part of the API, create recursion through bidirectional relationships, and couple the public response format to the database model.

Instead, I fetch the data needed by the use case and map it to a DTO inside the service boundary. For read-heavy endpoints, a DTO projection can be even more efficient because it selects only the required columns.

## Official references
- [Jakarta Persistence](https://jakarta.ee/specifications/persistence/)
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
