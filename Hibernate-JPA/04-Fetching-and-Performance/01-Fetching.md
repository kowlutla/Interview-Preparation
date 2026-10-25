# Fetching and Performance

## Q1. What is the N+1 query problem?
**Must Know**

I call it an N+1 problem when the application runs one query to load a set of root entities and then runs another query for each root to load related data. For example, I might load 100 orders in one query and then trigger 100 separate customer or item queries while building the response.

The endpoint may still look correct in a small test, but the number of database round trips grows with the result size. I confirm the problem using SQL logs, query counts, traces, and realistic data volumes.

## Q2. How would you fix an N+1 query?
**Must Know**

First, I identify which association is triggering repeated queries and what data the use case actually needs. Then I compare a fetch join, an entity graph, batch fetching, or a DTO projection. A fetch join may be ideal for a required to-one relationship; a projection is often better when the endpoint only needs a few fields.

I don't automatically join every collection, because one large join can multiply rows and consume more memory than a few bounded queries. After changing the fetch plan, I measure both statement count and total rows transferred.

## Q3. What is a fetch join, and what are its limitations?
**Must Know**

A fetch join lets me request an association as part of a JPQL query, so Hibernate can load the relationship along with the root entities. It is useful when I know the response needs that relationship immediately.

The main trap is collection fetching. If a root has multiple child rows, the SQL result contains repeated root columns. Fetching multiple collections can multiply rows, and combining a collection fetch join with pagination can make the database limit rows rather than logical root entities. I check Hibernate warnings and the exact version's behavior instead of assuming the query is safe.

## Q4. When would you use an entity graph?
**High Value**

I use an entity graph when I want to define which attributes a particular operation should fetch without embedding every fetch decision into the JPQL itself. A named graph can be reused, while a dynamic graph can be assembled for a specific use case.

An entity graph describes the fetch plan, not a guarantee that Hibernate will use exactly one SQL join. I still inspect generated SQL and verify that the graph doesn't load more data than the endpoint needs.

## Q5. Why not configure every association as EAGER?
**Must Know**

EAGER means the association must be available, not that Hibernate must fetch everything in one join. Depending on the query, Hibernate can issue secondary selects, and loading associations that a caller never uses wastes work.

I generally keep associations LAZY and explicitly fetch what each use case needs. This gives me more predictable query behavior, but it requires a clear service boundary so required data is loaded before the persistence context closes.

## Q6. What causes LazyInitializationException?
**Must Know**

With Hibernate, this exception commonly occurs when the code accesses an uninitialized proxy or collection after the Session/persistence context is no longer available to load it. A typical example is returning an entity from a service and then having JSON serialization access a lazy collection after the service transaction has ended.

I fix the fetch plan: load the required data inside the transaction and map it to a DTO before returning. Making everything EAGER can create a different performance problem, so I don't treat it as the default solution.

## Q7. What are the trade-offs of Open Session in View?
**High Value**

Open Session in View keeps the persistence context open through web request processing, which can allow lazy associations to load while a response is being serialized. That may hide missing fetch plans, but it can also trigger unexpected database calls outside the service transaction and make latency difficult to predict.

If I disable OSIV, I make sure service methods explicitly load the data the response needs. Whether OSIV is enabled by default depends on framework configuration and version, so I check the application rather than assume.

## Q8. Why can a collection fetch join break pagination?
**Must Know**

A collection join can produce several SQL rows for one root entity. Database pagination applies its limit and offset to those rows, which may cut through a root's children or return an unexpected set of roots. Some provider configurations may instead fetch a large result and paginate in memory, which can be very expensive.

A safer pattern is to page the root IDs first and then fetch the associations for those IDs in a second query. A DTO projection designed for pagination can be another good option.

## Q9. When would you choose a DTO projection over an entity?
**High Value**

If an endpoint only needs a few fields for display or reporting, I usually prefer a DTO projection. It avoids materializing a full managed entity and can avoid loading associations the endpoint never uses. Entities make more sense when I need domain behavior, relationship navigation, or managed updates.

I verify the generated SQL because the projection type alone doesn't guarantee the most efficient query, especially with nested projections or complex joins.

## Q10. What is batch fetching?
**High Value**

Batch fetching lets Hibernate load several eligible lazy associations or entities together, often using an IN-style query, instead of issuing one query for every object. It can reduce round trips without joining all collections into a single enormous result.

It is a useful option when access patterns naturally touch several related entities. I tune the batch size based on measurements and database constraints, and I verify the actual statement count.

## Q11. How do you diagnose a slow Hibernate query?
**Must Know**

I start by separating database execution time from ORM and application overhead. I inspect generated SQL, query counts, bind types where safe, tracing data, connection-pool wait time, and the database execution plan. I look for N+1 patterns, unnecessary columns, missing indexes, poor join cardinality, lock waits, and long transactions.

Then I change one likely cause at a time and compare the results under realistic data volume. I don't assume that adding a cache or changing a fetch setting will help without measuring it.

## Q12. Why can an endpoint be slow when each SQL statement is fast?
**High Value**

The endpoint may be executing hundreds of individually fast statements, waiting for a connection, transferring too many rows, serializing a large object graph, or doing non-database work while holding a transaction open. Looking only at the slowest SQL statement misses those costs.

I measure end-to-end latency, total query count, rows returned, pool wait time, and serialization time. That gives me a better basis for deciding whether to reduce round trips, change the fetch plan, or shorten the transaction.

## Official references
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
