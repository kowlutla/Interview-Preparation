# Entity Mapping and Relationships

## Q1. What makes a class a JPA entity?
**Must Know**

An entity is declared with @Entity and follows the selected Jakarta Persistence version's requirements, including a persistent identifier and suitable construction rules. Entities represent persistent identity, not merely API payloads. Use explicit table/column mappings when schema conventions differ.

## Q2. How should identifier generation be selected?
**High Value**

IDENTITY, SEQUENCE, TABLE, and AUTO have different trade-offs. Sequence generation can support efficient allocation and batching on suitable databases; identity generation may require immediate inserts to obtain keys and constrain batching. AUTO is provider/dialect-dependent. Verify the actual schema and generated SQL.

## Q3. What is the owning side of a relationship?
**Must Know**

The owning side controls the relationship update in the database. In a common bidirectional one-to-many/many-to-one mapping, the many-to-one side owns the foreign key, while the collection side uses mappedBy. Updating only the inverse collection may not persist the relationship.

## Q4. What does mappedBy mean?
**Must Know**

mappedBy names the association attribute on the owning entity, not a database column. It marks the inverse side. The attribute name must match the Java mapping, and the owning side should be set before flush.

## Q5. How do cascade and orphanRemoval differ?
**Must Know**

Cascade propagates selected operations such as persist, merge, or remove to related entities. orphanRemoval schedules deletion when a privately owned child is removed from its relationship. Cascade is not orphan removal. Avoid cascading REMOVE through shared many-to-many relationships.

## Q6. When should each relationship type be used?
**High Value**

ManyToOne commonly models a child foreign key; OneToMany models the parent's collection; OneToOne represents a unique association; ManyToMany is suitable only when the join has no meaningful attributes and lifecycle is genuinely shared. If the join has role, status, or timestamps, model it as an association entity.

## Q7. Why is LAZY often safer than EAGER?
**Must Know**

LAZY enables use-case-specific loading. EAGER requires data to be available but does not guarantee a single join; secondary selects can still cause N+1 behavior. Prefer lazy mappings with explicit fetch plans, and load required data before the persistence boundary closes.

## Q8. What makes entity equals() and hashCode() difficult?
**High Value**

Generated identifiers may be null before persistence and assigned later, so naïve ID-based hash codes can change while an entity is inside a hash collection. Equality based on mutable fields is also unstable. Choose a deliberate identity strategy and test transient, managed, detached, and proxy cases; no single implementation suits every model.

## Q9. How should composite identifiers be mapped?
**Advanced**

Use EmbeddedId with an embeddable key or IdClass. The key type must implement equality and hash code consistently with database key semantics. Composite IDs affect repository signatures and associations; use a surrogate key if it fits the domain and schema.

## Q10. What is the difference between JoinColumn and JoinTable?
**High Value**

JoinColumn maps a foreign-key column, commonly in the owning table. JoinTable maps an intermediate table connecting entities. Choose based on the relational schema and cardinality; one-to-many does not always require a join table.

## Q11. Why might a bidirectional relationship look correct in Java but not persist?
**Must Know**

The application may have updated only the inverse side. Set the owning-side reference and keep both in-memory sides consistent with helper methods. If a new child is involved, also check cascade configuration and inspect generated SQL.

## Q12. Should entities be returned directly from REST controllers?
**High Value**

Usually not. Serialization can trigger lazy loads, expose internal fields, create recursive graphs, and couple API contracts to persistence mappings. Map to DTOs inside a deliberate fetch/transaction boundary; projections can avoid loading full entities.

## Official references
- [Jakarta Persistence](https://jakarta.ee/specifications/persistence/)
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
