# SQL & Database Engineering Interview Q&A Handbook

A focused interview handbook for Java backend engineers preparing for mid-level and senior software engineering interviews. Runnable SQL examples use broadly compatible MySQL syntax, targeting MySQL 5.7+ unless a feature explicitly states otherwise. The emphasis is on clear, practical queries rather than requiring the newest MySQL features.

> **Compatibility note:** Examples aim for MySQL 5.7+ compatibility and avoid requiring CTEs, window functions, or `EXPLAIN ANALYZE` where a simpler alternative is practical. Check the deployed version, SQL mode, schema constraints, and bind-parameter conventions before production use. The examples are explanatory and have not been executed against a live database as part of this handbook.

## Table of contents

- [Top 15 Questions to Master First](#top-15-questions-to-master-first)
- [SQL Fundamentals and Query Logic](#sql-fundamentals-and-query-logic)
- [Joins, Aggregation, and Query Composition](#joins-aggregation-and-query-composition)
- [Practical SQL Query Patterns](#practical-sql-query-patterns)
- [Indexes and Query Performance](#indexes-and-query-performance)
- [Transactions and Concurrency](#transactions-and-concurrency)
- [MySQL in Practice](#mysql-in-practice)
- [Backend Production Scenarios](#backend-production-scenarios)
- [Additional High-Value Backend Questions](#additional-high-value-backend-questions)
- [Essential SQL Interview Questions: Joins, Keys, and Core Query Patterns](#essential-sql-interview-questions-joins-keys-and-core-query-patterns)
- [References](#references)

## Top 15 Questions to Master First

1. [Q1. What is the difference between WHERE and HAVING?](#q1-what-is-the-difference-between-where-and-having)
2. [Q45. How do prepared statements prevent SQL injection, and what can they not parameterize?](#q45-how-do-prepared-statements-prevent-sql-injection-and-what-can-they-not-parameterize)
3. [Q3. How do NULL values behave in SQL?](#q3-how-do-null-values-behave-in-sql)
4. [Q8. What is the difference between INNER JOIN and LEFT JOIN, and how can ON versus WHERE change results?](#q8-what-is-the-difference-between-inner-join-and-left-join-and-how-can-on-versus-where-change-results)
5. [Q13. How do you find the second-highest distinct salary?](#q13-how-do-you-find-the-second-highest-distinct-salary)
6. [Q12. How do ROW_NUMBER, RANK, and DENSE_RANK differ?](#q12-how-do-row_number-rank-and-dense_rank-differ)
7. [Q14. How do you find the highest-paid employee in each department?](#q14-how-do-you-find-the-highest-paid-employee-in-each-department)
8. [Q19. How do you identify duplicates and remove them safely?](#q19-how-do-you-identify-duplicates-and-remove-them-safely)
9. [Q21. How do indexes work, and what are their trade-offs?](#q21-how-do-indexes-work-and-what-are-their-trade-offs)
10. [Q22. How do composite indexes work, and why does column order matter?](#q22-how-do-composite-indexes-work-and-why-does-column-order-matter)
11. [Q25. How do you investigate and optimize a slow SQL query?](#q25-how-do-you-investigate-and-optimize-a-slow-sql-query)
12. [Q27. What are ACID properties?](#q27-what-are-acid-properties)
13. [Q28. Explain transaction isolation levels and common anomalies.](#q28-explain-transaction-isolation-levels-and-common-anomalies)
14. [Q30. What causes deadlocks, and how should an application handle them?](#q30-what-causes-deadlocks-and-how-should-an-application-handle-them)
15. [Q33. How should pagination be implemented in MySQL?](#q33-how-should-pagination-be-implemented-in-mysql)

## SQL Fundamentals and Query Logic

### Q1. What is the difference between WHERE and HAVING?

`WHERE` filters individual rows before grouping and aggregation. `HAVING` filters groups after `GROUP BY` has formed them. Use `WHERE` for row-level conditions and `HAVING` for aggregate conditions.

```sql
SELECT department_id, COUNT(*) AS employee_count
FROM employees
WHERE salary > 50000
GROUP BY department_id
HAVING COUNT(*) >= 5;
```

Only employees earning more than 50,000 contribute to each count; departments with fewer than five qualifying employees are then excluded. A common mistake is trying to put an aggregate such as `COUNT(*)` in `WHERE`. Filtering early can also reduce the rows that need to be grouped.

### Q2. What is the difference between DELETE, TRUNCATE, and DROP?

- `DELETE` removes selected rows (or all rows if no `WHERE` is supplied). It is a DML operation and row-level triggers may fire, depending on the database.
- `TRUNCATE` removes all rows using a database-specific DDL operation; it generally cannot filter rows and has different logging, trigger, identity, and rollback behavior from `DELETE`.
- `DROP` removes the table object itself, including its definition and associated dependent objects subject to database rules.

Do not assume the three commands have identical transaction semantics. In MySQL, `TRUNCATE TABLE` is DDL and causes an implicit commit; transactional rollback should not be expected to restore the table contents. `DELETE` on an InnoDB table can be rolled back before commit. Check foreign keys, triggers, privileges, and backup/recovery implications before destructive operations.

### Q3. How do NULL values behave in SQL?

`NULL` represents missing or unknown information; it is not equal to zero, an empty string in all databases, or another `NULL`. Comparisons such as `column = NULL` evaluate to UNKNOWN rather than TRUE. Use `IS NULL` or `IS NOT NULL`.

SQL predicates can evaluate to TRUE, FALSE, or UNKNOWN. A `WHERE` clause keeps only rows for which the condition is TRUE. This is why `NOT IN` can surprise you if its list or subquery contains a `NULL`: the result may be UNKNOWN rather than TRUE.

```sql
SELECT COUNT(*) AS all_rows,
       COUNT(manager_id) AS rows_with_manager
FROM employees;
```

`COUNT(*)` counts rows, while `COUNT(manager_id)` counts only non-NULL values. Most aggregates ignore NULL inputs; `COUNT(column)` is the key common exception to remember. Use `COALESCE` to supply a fallback when appropriate, but do not replace NULL with a made-up value if that changes the meaning of the data.

### Q4. What is the difference between primary keys, foreign keys, and unique constraints?

A primary key uniquely identifies each row and cannot contain NULL. A foreign key enforces a relationship to a referenced key (subject to the database's supported constraints and configuration). A unique constraint prevents duplicate key values, with NULL handling varying by database and constraint definition.

Use constraints to enforce data integrity at the database boundary, not only in Java validation: concurrent requests can pass application checks at the same time. A unique constraint is the final guard against duplicate business identifiers. A foreign key prevents orphan references, but it introduces write-time checks and must be considered in deletion and migration plans. Composite keys are possible, but a surrogate primary key plus appropriate unique business constraints is often easier to reference in application code.

### Q5. What is the difference between UNION and UNION ALL?

Both combine the results of compatible `SELECT` statements. `UNION` removes duplicate result rows; `UNION ALL` preserves them. Duplicate elimination usually requires additional work, such as sorting or hashing, so prefer `UNION ALL` when duplicates are meaningful or impossible by design.

The corresponding columns must have compatible types and appear in the same order. An `ORDER BY` for the combined result belongs at the end of the compound query. Do not add `UNION` merely to hide unexpected duplicates caused by a faulty join; find the underlying cardinality problem first.

### Q6. When would you use EXISTS instead of IN or a JOIN?

Use `EXISTS` when the question is whether at least one matching row exists, especially for correlated existence checks. Use a join when columns from both tables are needed. `IN` is often clear for membership in a set. Optimizers can transform these forms into similar plans, so correctness and measured execution plans matter more than a universal rule that one is always faster.

```sql
SELECT c.customer_id
FROM customers c
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.customer_id
);
```

This returns each customer row that has at least one order, without multiplying a customer by the number of orders. For customers without orders, use `NOT EXISTS`; it is often safer than `NOT IN` when the subquery can return NULL. Ensure correlated predicates compare the intended keys.

### Q7. What is normalization, and when is denormalization justified?

Normalization organizes data to reduce avoidable duplication and update anomalies. For example, storing a department name on every employee row can create inconsistent names when a department is renamed; a department table and foreign key avoid that duplication. First, second, and third normal forms progressively address repeating groups, partial dependencies on composite keys, and non-key dependencies on other non-key attributes.

Denormalization deliberately duplicates or precomputes data to improve a measured read workload, such as storing a summary count. The trade-off is more complicated writes, consistency management, and storage. Consider it after examining query plans, indexes, workload patterns, and caching—not as the first response to every slow query.

## Joins, Aggregation, and Query Composition

### Q8. What is the difference between INNER JOIN and LEFT JOIN, and how can ON versus WHERE change results?

An `INNER JOIN` returns matching row pairs only. A `LEFT JOIN` returns every left-side row, adding NULLs for right-side columns when no match exists. One left row can still produce multiple output rows when several right rows match.

```sql
-- Keep every customer; attach only paid orders when available.
SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o
  ON o.customer_id = c.customer_id
 AND o.status = 'PAID';
```

Moving `o.status = 'PAID'` into `WHERE` filters out the NULL-extended rows and effectively turns this condition into an inner-style result for that predicate. Put match criteria in `ON` when unmatched left rows must remain; put final row filters in `WHERE` when excluding rows is intended. MySQL supports `INNER JOIN` and `LEFT JOIN`, but does not directly support the standard `FULL OUTER JOIN` syntax. To produce a full-outer-join-like result in MySQL, combine a left join with the unmatched rows from the other side using `UNION ALL`, taking care not to duplicate matched rows.

### Q9. Why can a join unexpectedly multiply rows or inflate aggregates?

A join returns one output row for each matching pair. If one customer has three orders and each order has four line items, joining customers, orders, and items can produce twelve rows for that customer. A subsequent `SUM(order_total)` can count each order total multiple times.

Check the relationship cardinalities and keys at every join. Aggregate a many-side table to the required grain before joining when that is logically correct, or use `EXISTS` if only existence matters. `COUNT(DISTINCT ...)` can be valid when counting unique entities, but using `DISTINCT` as a blanket repair can conceal a data-model or join-condition defect.

### Q10. How do GROUP BY and conditional aggregation work?

`GROUP BY` produces one result row per group, and aggregate functions summarize the rows in each group. In a portable query, selected expressions should either be grouped or aggregated. Conditional aggregation uses `CASE` inside an aggregate to compute multiple metrics in one pass.

```sql
SELECT customer_id,
       COUNT(*) AS total_orders,
       SUM(CASE WHEN status = 'PAID' THEN 1 ELSE 0 END) AS paid_orders,
       SUM(CASE WHEN status = 'CANCELLED' THEN amount ELSE 0 END) AS cancelled_amount
FROM orders
GROUP BY customer_id;
```

Remember that `SUM` and `AVG` generally ignore NULL input values, and `SUM` over no input rows may return NULL rather than zero. Use `COALESCE` if zero is the correct business meaning. Also verify whether an aggregate is computed before or after a join; the join's row grain affects the result.

### Q11. When would you use a subquery?

A subquery expresses a query inside another query. It can keep a multi-step operation readable without requiring newer syntax such as common table expressions (CTEs), which are not available in MySQL 5.7.

```sql
SELECT e.employee_id, e.salary
FROM employees e
JOIN (
  SELECT department_id, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department_id
) d ON d.department_id = e.department_id
WHERE e.salary > d.avg_salary;
```

The derived table calculates one average per department, then the outer query compares each employee with that average. A subquery is not automatically faster than a join or another equivalent form. Prefer the clearest correct query, then inspect its execution plan and measure it with representative data.

## Practical SQL Query Patterns

### Q12. How do ROW_NUMBER, RANK, and DENSE_RANK differ?

These ranking functions assign ordering numbers within a result set or partition. `ROW_NUMBER()` assigns a unique sequential number to each row, even when sort values tie. `RANK()` gives tied rows the same rank and leaves gaps afterward. `DENSE_RANK()` gives tied rows the same rank without gaps. They are available in newer MySQL releases; the practical query examples in this handbook use alternatives that work on older MySQL versions.

For salaries `100, 90, 90, 80`, the ranks are:

| Salary | ROW_NUMBER (one possible order) | RANK | DENSE_RANK |
|---:|---:|---:|---:|
| 100 | 1 | 1 | 1 |
| 90 | 2 | 2 | 2 |
| 90 | 3 | 2 | 2 |
| 80 | 4 | 4 | 3 |

Use a deterministic tie-breaker with `ROW_NUMBER` when exactly one row must be selected. If the business requirement is to return all tied top earners, use `RANK` or filter by the maximum value instead. A window's `ORDER BY` defines ranking order; it does not guarantee the final display order, so add a query-level `ORDER BY` if required.

### Q13. How do you find the second-highest distinct salary?

Sort the distinct, non-NULL salary values in descending order and skip the highest one. This simple query works without window functions or CTEs and correctly handles ties for the highest salary.

```sql
SELECT DISTINCT salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

The query returns one row containing the second-highest distinct salary, or no rows if fewer than two distinct non-NULL salaries exist. If the requirement is instead the second employee after sorting, do not use `DISTINCT`; define a deterministic tie-breaker such as `employee_id`.

### Q14. How do you find the highest-paid employee in each department?

Compare each employee's salary with the maximum non-NULL salary in that employee's department. This returns all employees tied for the highest salary and avoids requiring window functions.

```sql
SELECT e.employee_id, e.department_id, e.salary
FROM employees e
WHERE e.salary IS NOT NULL
  AND e.salary = (
    SELECT MAX(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
  )
ORDER BY e.department_id, e.employee_id;
```

The correlated subquery calculates the maximum salary for each employee's department. If a department has no non-NULL salaries, it contributes no rows. If exactly one winner is required even when salaries tie, define a business tie-breaker and use a different query.

### Q15. How do you find the top N records per group without window functions?

For MySQL versions without window functions, a correlated subquery can count how many rows rank ahead of the current row. The following returns up to three most recent orders per customer, using `order_id` to break timestamp ties:

```sql
SELECT o.order_id, o.customer_id, o.created_at
FROM orders o
WHERE (
  SELECT COUNT(*)
  FROM orders newer
  WHERE newer.customer_id = o.customer_id
    AND (newer.created_at > o.created_at
      OR (newer.created_at = o.created_at AND newer.order_id > o.order_id))
) < 3
ORDER BY o.customer_id, o.created_at DESC, o.order_id DESC;
```

The count is the number of rows ordered ahead of the current row; fewer than three means the row belongs in the top three. The unique `order_id` tie-breaker makes the ordering deterministic. This is easy to understand but can be expensive on large tables without a suitable index. If timestamps or grouping keys can be NULL, define the intended NULL behavior and adjust the predicate.

### Q16. How do you find customers who have never placed an order?

Use an anti-join pattern, most simply `NOT EXISTS`.

```sql
SELECT c.customer_id
FROM customers c
WHERE NOT EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.customer_id
);
```

This expresses the requirement directly and is not confused by NULLs in an unrelated selected column. An alternative is a `LEFT JOIN` followed by `WHERE o.order_id IS NULL`, provided `order_id` is guaranteed non-NULL for real orders. Make sure the join predicate uses the correct customer key and that indexes support the lookup when the tables are large.

### Q17. How do you retrieve the latest record for each customer?

Find the row for which no newer row exists for the same customer. Use a unique tie-breaker so the query returns one row even when timestamps match.

```sql
SELECT e.customer_id, e.event_id, e.updated_at, e.event_type
FROM customer_events e
LEFT JOIN customer_events newer
  ON newer.customer_id = e.customer_id
 AND (newer.updated_at > e.updated_at
      OR (newer.updated_at = e.updated_at AND newer.event_id > e.event_id))
WHERE newer.event_id IS NULL;
```

Define what “latest” means: event time, creation time, or update time can differ. This example assumes `customer_id` and `updated_at` are non-NULL, and `event_id` is unique and non-NULL. If a grouping key or timestamp can be NULL, define the intended behavior and adjust the comparison accordingly. An index beginning with `customer_id` and including the ordering columns may help; measure the plan on representative data.

### Q18. How do you calculate a running total without window functions?

In older MySQL versions, a correlated subquery can sum all transactions up to the current transaction for the same account. This is more verbose and may be slower than a window function, but it avoids requiring MySQL 8.0.

```sql
SELECT t.account_id, t.transaction_id, t.transaction_time, t.amount,
       (
         SELECT SUM(t2.amount)
         FROM account_transactions t2
         WHERE t2.account_id = t.account_id
           AND (t2.transaction_time < t.transaction_time
             OR (t2.transaction_time = t.transaction_time
                 AND t2.transaction_id <= t.transaction_id))
       ) AS running_balance_change
FROM account_transactions t
ORDER BY t.account_id, t.transaction_time, t.transaction_id;
```

The ordering key must be deterministic. This computes the cumulative sum of transaction amounts, not necessarily a bank balance; an opening balance may need to be added. Consider NULL amounts, NULL timestamps, and reversals, and index the account and ordering columns for larger tables.

### Q19. How do you identify duplicates and remove them safely?

First define a duplicate by its business key, not by assuming every identical-looking row is invalid. Find duplicate groups with `GROUP BY` and `HAVING COUNT(*) > 1`.

```sql
SELECT email, COUNT(*) AS occurrences
FROM users
WHERE email IS NOT NULL
GROUP BY email
HAVING COUNT(*) > 1;
```

Before deleting anything, preview the rows and decide which record should survive. The following examples assume `user_id` is a unique, non-NULL key and that only repeated, non-NULL email addresses count as duplicates. Change the survivor rule if the business requirement differs.

**Preview the rows that would be removed (keeping the smallest `user_id` per email):**

```sql
SELECT u1.user_id, u1.email, u1.created_at
FROM users u1
WHERE u1.email IS NOT NULL
  AND EXISTS (
    SELECT 1
    FROM users u2
    WHERE u2.email = u1.email
      AND u2.user_id < u1.user_id
  )
ORDER BY u1.email, u1.user_id;
```

This preview and the DELETE statement both keep the row with the smallest `user_id` for each non-NULL email. The `created_at` column is displayed for inspection only; it does not determine which row survives. If the business rule is to keep the earliest-created record, change both the preview and deletion logic to use the same ordering rule.

**MySQL — keep the row with the smallest `user_id`:**

```sql
DELETE u1
FROM users u1
JOIN users u2
  ON u1.email = u2.email
 AND u1.user_id > u2.user_id
WHERE u1.email IS NOT NULL;
```

The MySQL self-join is appropriate only when keeping the smallest key is the intended rule. Review the affected-row count and verify the result before committing; use a transaction where supported and appropriate, and ensure backups and dependent records are handled. A unique constraint can prevent recurrence, but confirm the intended NULL and collation/case-sensitivity rules first. Never run a destructive delete from an example without adapting and validating it for the actual schema.

### Q20. How do you find employees earning more than their department's average salary?

Use a correlated subquery to compare each employee's salary with the average for the same department.

```sql
SELECT e.employee_id, e.department_id, e.salary
FROM employees e
WHERE e.salary IS NOT NULL
  AND e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
      AND e2.salary IS NOT NULL
  );
```

The subquery calculates the department average for each employee's department. Departments with no non-NULL salaries contribute no matching employees. This form avoids CTEs and window functions; compare execution plans and consider indexing `department_id` for large tables.

## Indexes and Query Performance

### Q21. How do indexes work, and what are their trade-offs?

An index is an auxiliary data structure that helps the database locate rows without scanning every row. Common B-tree indexes support equality lookups, ranges, and ordered access. The optimizer decides whether an index is cheaper than alternatives based on selectivity, table size, statistics, required columns, and estimated I/O.

Indexes speed up some reads but consume storage and add work to inserts, updates, and deletes because index entries must be maintained. Too many indexes can hurt write throughput and increase maintenance cost. In MySQL InnoDB, the primary key is the clustered index and secondary indexes include the primary-key value. An index is not automatically beneficial: a full table scan can be cheaper when a query needs a large fraction of the table.

### Q22. How do composite indexes work, and why does column order matter?

A composite index stores multiple columns in a defined order. For a B-tree index on `(status, created_at, customer_id)`, predicates that constrain the leftmost column and then a range on the next column are often well supported. A query filtering only by `customer_id` generally cannot use the leading prefix as efficiently as an index that begins with that column.

Choose column order from real query patterns: equality predicates, range predicates, join conditions, sorting requirements, and selectivity all matter. Do not rely on a simplistic “most selective column first” rule for every workload. A covering index contains all columns needed by a query and may avoid table-row lookups, but larger indexes consume more space and make writes more expensive. Confirm the benefit with the execution plan and representative data.

### Q23. Why might the optimizer ignore an index that exists?

An index may be ignored because a large proportion of rows qualifies, the table is small, the index does not match the predicate or leading-column pattern, statistics are stale, implicit type conversion occurs, or a function/expression prevents a normal index range lookup. The optimizer may correctly choose a table scan if that is cheaper.

For example, wrapping an indexed timestamp in a function can make range access harder. In MySQL, prefer a half-open range rather than applying `DATE()` to every row.

```sql
-- Often less index-friendly
WHERE DATE(created_at) = '2026-01-15';

-- Prefer a half-open range; bind the start and next-day boundary
WHERE created_at >= ?
  AND created_at <  ?;
```

The two positional placeholders represent the start timestamp and the exclusive next-day boundary, respectively.

Use correctly typed bind parameters and half-open time ranges. Function-based indexes or generated-column indexes can be appropriate when the expression is intentional and supported. Refresh statistics using the database's supported procedure when evidence suggests they are stale; do not force an index hint before understanding the plan.

### Q24. What is a sargable predicate, and why does it matter?

A sargable predicate lets the optimizer use an index access path to narrow candidate rows efficiently. Equality and range conditions directly on an indexed column are common examples. Applying a function or arithmetic operation to the column can prevent ordinary index use unless a matching expression index or optimizer transformation applies.

Prefer `created_at >= ? AND created_at < ?` over converting every timestamp to a date in the predicate. Avoid leading wildcards such as `LIKE '%term'` when expecting a conventional B-tree index to perform a prefix lookup. Sargability is not a guarantee of index use; selectivity and cost still decide the plan.

### Q25. How do you investigate and optimize a slow SQL query?

Start with evidence: identify the exact SQL, bind values, frequency, latency, rows returned, and whether time is spent in the database, network, or application. Compare the slow query with a known-good baseline and inspect the execution plan. Look for estimated-versus-actual row mismatches where available, full scans over large selective workloads, expensive sorts, repeated lookups, poor join order, and unexpectedly large intermediate results.

A quick MySQL syntax reminder:

```sql
EXPLAIN
SELECT order_id
FROM orders
WHERE customer_id = 123;
```

Use a representative value and inspect the estimated access type, chosen index, join order, and estimated rows. A practical sequence is:

1. Reproduce with representative data and parameter values.
2. Inspect the plan and database metrics; check table/index statistics.
3. Verify join cardinality and filtering logic before changing indexes.
4. Test a targeted index or rewrite, considering write cost and storage.
5. Measure before and after under comparable conditions.
6. Roll out safely and monitor latency, load, and regressions.

Use MySQL `EXPLAIN` to inspect the optimizer's estimated plan, then compare it with query latency, the slow query log, and application/database metrics. An execution plan is evidence to interpret, not a score where any full scan is automatically bad. Avoid repeating the same diagnosis steps: establish a baseline, make one evidence-based change, and measure the result under comparable conditions.

### Q26. What are covering indexes and index selectivity?

Selectivity describes how effectively a predicate narrows a set of rows; a value that matches very few rows is often more selective than a low-cardinality value such as a boolean status. A covering index contains all columns needed by a query, so the database may satisfy it from the index without visiting the base table for each matching row.

A low-selectivity index can still help in a composite index, for sorting, or when combined with other predicates. Conversely, a highly selective index can be a poor choice if a query returns most columns for many rows and would cause many random lookups. Index design is workload-specific: evaluate query frequency, read/write mix, data distribution, clustering, and actual plans rather than applying one heuristic blindly.

## Transactions and Concurrency

### Q27. What are ACID properties?

- **Atomicity:** A transaction's changes are committed as a unit or rolled back as a unit.
- **Consistency:** A transaction preserves defined integrity rules when starting from a valid state and completing successfully.
- **Isolation:** Concurrent transactions are controlled so their interactions follow the database's isolation semantics.
- **Durability:** Once committed, changes survive failures within the guarantees of the database configuration.

ACID is not a replacement for correct business logic. For example, a transfer must debit and credit within the same transaction, and the application must still validate that the transfer is allowed. Durability can depend on configuration and failure assumptions, so do not promise more than the database's documented guarantees.

### Q28. Explain transaction isolation levels and common anomalies.

Isolation levels trade concurrency for how much concurrent activity a transaction can observe. The common SQL names are Read Uncommitted, Read Committed, Repeatable Read, and Serializable, but implementation details vary.

- **Dirty read:** Reading another transaction's uncommitted changes.
- **Non-repeatable read:** Reading the same row twice and seeing a committed update between reads.
- **Phantom:** Repeating a predicate query and seeing a changed set of matching rows.
- **Lost update:** Two writers overwrite one another's changes if the application/database pattern does not protect the update.

InnoDB's default isolation level in MySQL 8.4 is Repeatable Read, using MVCC for consistent nonlocking reads and locks for writes; locking and predicate behavior matters. Isolation semantics are engine-specific, so use the documentation for the deployed database and version. The textbook anomaly table is not a complete description of every engine's behavior. For a concurrency bug, reason about the exact engine, isolation level, transaction boundaries, and whether the query is a locking read.

### Q29. What is MVCC, and how is it different from locking?

Multi-Version Concurrency Control (MVCC) keeps enough row versions or undo information for readers to observe a consistent snapshot while writers modify data. This can reduce reader-writer blocking, but it does not eliminate locks: writes still need coordination, and some reads explicitly request locks.

InnoDB combines MVCC with row-level locking; ordinary consistent reads and locking reads have different semantics. Long-running transactions can retain old versions longer, increase cleanup work, and contribute to storage pressure. Do not assume that “MVCC means no blocking” or that a normal SELECT always sees the newest committed value at the instant it finishes.

### Q30. What causes deadlocks, and how should an application handle them?

A deadlock occurs when transactions form a cycle of dependencies—for example, transaction A locks row 1 and waits for row 2, while transaction B locks row 2 and waits for row 1. The database detects a cycle and aborts a participant to release resources.

Reduce risk by acquiring rows in a consistent order, keeping transactions short, indexing predicates so updates lock fewer rows where applicable, and avoiding user/network calls while a transaction is open. Applications should treat deadlock victims as failed transactions and may retry the entire transaction with bounded retries and jitter when the operation is safe to repeat. Do not retry indefinitely or retry only the last statement if earlier work was rolled back. A unique business key or idempotency mechanism can help prevent duplicate side effects during retries.

### Q31. What is the difference between optimistic and pessimistic locking?

Optimistic locking assumes conflicts are uncommon. A typical implementation adds a version column and updates only if the version has not changed:

```sql
UPDATE inventory
SET quantity = ?,
    version = version + 1
WHERE item_id = ?
  AND version = ?;
```

If the affected-row count is zero, the row was changed or no longer exists; reload and resolve the conflict. In JPA, a `@Version` field can provide this pattern.

Pessimistic locking acquires locks before changes are made, such as a supported `SELECT ... FOR UPDATE`, preventing competing transactions from modifying the locked rows until completion. It can be appropriate for high-contention workflows but increases blocking and deadlock risk. Choose based on contention, latency, correctness requirements, and database support; keep the transaction brief.

### Q32. Why are long-running transactions dangerous for a backend service?

They hold locks longer, increase the chance of blocking and deadlocks, and can delay cleanup of old row versions. In a connection-pooled Java service, a transaction that waits on remote HTTP calls or performs lengthy computation may also occupy a database connection for too long, contributing to pool exhaustion.

Keep database transactions around the smallest unit of work that must be atomic. Do not call slow external services while holding database locks unless the design explicitly requires it. Monitor transaction age, lock waits, connection acquisition time, pool active/idle counts, and query latency. Increasing the connection pool alone may overload the database or worsen contention; identify whether connections are slow, leaked, or held by overly broad transaction boundaries.

## MySQL in Practice

### Q33. How should pagination be implemented in MySQL?

Always use a deterministic `ORDER BY`; pagination without stable ordering can return inconsistent pages. Offset pagination is straightforward, but large offsets may require the database to walk and discard many rows. Concurrent inserts or deletes can also shift records between requests.

```sql
SELECT order_id, created_at
FROM orders
ORDER BY created_at DESC, order_id DESC
LIMIT ? OFFSET ?;
```

In a JDBC prepared statement, bind the page size and offset to the two positional placeholders. Use a deterministic order, including a unique tie-breaker.

For deep pagination, consider keyset (seek) pagination. Pass the last row's sort keys and request rows after that position, using a predicate consistent with the ordering. Assuming `created_at` is non-NULL, the next-page predicate for descending `(created_at, order_id)` is:

```sql
-- MySQL; bind values using your database driver
SELECT order_id, created_at
FROM orders
WHERE created_at < ?
   OR (created_at = ? AND order_id < ?)
ORDER BY created_at DESC, order_id DESC
LIMIT ?;
```

Bind the last row's timestamp twice, its `order_id`, and the page size to the four positional placeholders in the MySQL example. The `order_id` tie-breaker is essential when timestamps match. These examples assume `created_at` is non-NULL. If the sort column can be NULL, define explicit NULL ordering and build a predicate that matches it. Keyset pagination works best with a compatible index and avoids scanning/discarding a large offset, but does not provide arbitrary page-number jumps.

### Q34. How do you retrieve the top three salaries per department in MySQL?

For compatibility with older MySQL versions, use a correlated subquery to count employees in the same department who have a higher salary, plus a tie-breaker. This returns up to three employees per department.

```sql
SELECT e.employee_id, e.department_id, e.salary
FROM employees e
WHERE e.salary IS NOT NULL
  AND (
    SELECT COUNT(*)
    FROM employees higher
    WHERE higher.department_id = e.department_id
      AND higher.salary IS NOT NULL
      AND (higher.salary > e.salary
        OR (higher.salary = e.salary AND higher.employee_id < e.employee_id))
  ) < 3
ORDER BY e.department_id, e.salary DESC, e.employee_id;
```

The query gives higher salaries priority and uses the smaller `employee_id` to break ties, returning at most three rows per department—not necessarily the top three distinct salary levels. If the requirement is to include every employee tied within the top three salary ranks, the query needs different logic. This example assumes `department_id` is non-NULL. It may be more expensive than a window-function query on large tables, so inspect the execution plan and consider an index on `(department_id, salary, employee_id)`.

### Q35. How does COALESCE handle NULL values in MySQL?

`COALESCE(expr1, expr2, ...)` returns the first non-NULL expression. In MySQL, it is useful when a result should fall back through several optional values. It does not make `NULL = NULL` evaluate to true; SQL's three-valued logic still applies.

```sql
SELECT COALESCE(nickname, preferred_name, 'Unknown') AS display_name
FROM users;
```

The expressions should have compatible result types; MySQL may perform type conversion when their types differ, so cast explicitly when that conversion could be surprising. In this example, `nickname` is used when present, otherwise `preferred_name`, and finally the literal `Unknown`. Note that `COALESCE` handles SQL NULL, not blank strings: if empty or whitespace-only names should count as missing, normalize or test those values explicitly.

### Q36. How does AUTO_INCREMENT work in MySQL, and why can IDs have gaps?

In MySQL, an integer column can be declared `AUTO_INCREMENT` so inserts can generate identifiers without the application supplying each value. Generated IDs can have gaps because transactions may roll back, rows may be deleted, and allocation behavior is designed for concurrency rather than gapless numbering. Do not use an auto-increment ID as proof of commit order or as a gapless invoice number. Use it for identity, and use constraints to enforce uniqueness.

### Q37. How do you inspect MySQL execution plans?

MySQL's `EXPLAIN` shows the optimizer's chosen plan, including access methods, possible and chosen indexes, join order, and estimated rows. It is available in older MySQL versions and is the baseline tool for understanding how a query is executed.

```sql
EXPLAIN
SELECT order_id
FROM orders
WHERE customer_id = ?;
```

Compare estimated row counts, access types, selected indexes, join order, and use of temporary tables or filesort. A full table scan is not automatically bad; it may be cheaper when a query needs a large fraction of the table. Measure actual latency separately with representative parameters and data, and use the slow query log or application metrics to understand real workload behavior.

## Backend Production Scenarios

### Q38. A query became slow after production data grew. What would you do?

First establish what changed: row counts, data distribution, query frequency, parameter values, schema/index changes, database version, statistics, or workload contention. Compare the current plan and timings with a prior baseline. A query that was fast on a small table may now scan a large table or produce a much larger join intermediate.

Validate that predicates are selective and sargable, joins use the right keys, and pagination or sorting is not doing excessive work. Check statistics and wait events as well as CPU and I/O. Test a targeted index or query rewrite against representative data and measure both read improvement and write overhead. Avoid speculative index additions or optimizer hints before confirming the bottleneck.

### Q39. Pagination sometimes skips or repeats records. Why?

If the query orders by a non-unique column, rows tied on that column can move between pages because their relative order is unspecified. Inserts and deletes between page requests can also shift offset-based pages. Without `ORDER BY`, SQL does not promise a stable order at all.

Add a unique tie-breaker to the sort order. For feeds or large datasets, keyset pagination using the last-seen sort key is often more stable and efficient. If the product requires a consistent snapshot across multiple page requests, consider the cost and feasibility of snapshot-based reads or an explicit as-of/version boundary; do not keep a long database transaction open across a user's browsing session.

### Q40. Concurrent requests update the same record and one update is lost. How do you fix it?

Choose a concurrency-control strategy based on the invariant. Optimistic locking adds a version predicate and checks the affected-row count; pessimistic locking locks the row during the critical transaction. For inventory, an atomic conditional update can prevent a negative quantity without a read-then-write race:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE item_id = ?
  AND quantity > 0;
```

Check the affected-row count: one means the reservation succeeded; zero means the item was missing or insufficient stock remained. If the application must distinguish those cases, perform a follow-up read or use a workflow designed for that distinction. Keep the update inside the transaction that includes any other database changes required for the reservation. Use the bind syntax supported by the database client.

Do not read a value into Java, calculate a new value, and overwrite it without protecting against concurrent updates. Use constraints where possible and make retries bounded and safe. Verify that the isolation level and ORM behavior match the chosen approach.

### Q41. The connection pool is exhausted, but the database CPU is not high. What might be wrong?

Pool exhaustion means application threads cannot acquire a connection quickly; it does not necessarily mean the database is CPU-saturated. Connections may be held by long transactions, slow queries waiting on locks or I/O, leaked connections, remote calls performed inside transactions, or an undersized pool relative to legitimate concurrency.

Inspect pool active/idle/pending counts, connection acquisition latency, transaction duration, slow query and lock-wait data, and application traces. Verify connections are always closed or returned to the pool and transaction boundaries are narrow. Increasing the pool can increase database contention and is not a substitute for diagnosing why connections are retained. Add timeouts and alerts so overload fails predictably rather than cascading.

### Q42. How do you troubleshoot unexpected MySQL query results or behavior?

Unexpected MySQL results often come from implicit conversions, NULL semantics, collation and case sensitivity, date/time types and time zones, grouping rules, transaction isolation, or assumptions about ordering. Reduce the issue to a small reproducible query and inspect both its results and execution plan. Use explicit casts, stable ordering, and unambiguous predicates. Avoid selecting ungrouped columns without a well-defined aggregate rule. Test with representative data, including NULLs, duplicates, and boundary timestamps.


## Additional High-Value Backend Questions

These questions extend the core SQL revision set with security, schema design, engine behavior, and safe operational practices that commonly arise in Java backend interviews.

### Q43. What is the logical order in which a SQL query is processed?

A useful conceptual order is:

1. `FROM` and `JOIN` establish the input row set.
2. `WHERE` filters individual rows.
3. `GROUP BY` forms groups and aggregates summarize them.
4. `HAVING` filters groups.
5. The `SELECT` list computes output expressions.
6. `DISTINCT` removes duplicate output rows when requested.
7. `ORDER BY` sorts the result.
8. MySQL applies the row-limiting operation, commonly `LIMIT`, after sorting the result with `ORDER BY`.

This is a logical model for understanding query semantics, not a promise about the physical execution plan. The optimizer can reorder joins, push predicates, and choose indexes while preserving the query's required result.

It explains why a SELECT alias is not generally available to the same query block's `WHERE` clause: the alias belongs to the output projection, which is logically later. It also explains why filtering on an aggregate belongs in `HAVING`, and why a window-function result is commonly filtered in an outer query or CTE. Some dialects add features such as `QUALIFY`; support varies by database and version.

### Q44. What is the difference between a clustered index and a secondary index in MySQL InnoDB?

In InnoDB, the table's rows are organized by the clustered index, normally the primary key. The clustered index leaf pages contain the row data. A secondary index stores its indexed columns plus the primary-key value used to locate the corresponding clustered row. A query that finds a secondary-index entry may therefore need an additional lookup in the clustered index unless the secondary index covers all required columns.

This has practical design implications. A wide primary key is copied into secondary-index entries, increasing their size. A stable, reasonably narrow primary key can reduce that overhead. Randomly distributed primary-key inserts can also create different page-split and locality behavior from append-friendly keys, although the right choice depends on workload and requirements.

This clustered-index description is specific to MySQL InnoDB and should not be assumed for other database engines. Confirm the actual table and index definitions and measure the workload rather than choosing a key based on a single rule.

### Q45. How do prepared statements prevent SQL injection, and what can they not parameterize?

SQL injection occurs when untrusted input is treated as part of SQL syntax. The core defense is to keep SQL structure separate from data: use prepared statements or parameter binding so values are transmitted as parameters rather than concatenated into the SQL string.

~~~java
String sql = "SELECT user_id, display_name FROM users WHERE email = ?";
try (PreparedStatement statement = connection.prepareStatement(sql)) {
    statement.setString(1, email);
    try (ResultSet resultSet = statement.executeQuery()) {
        // Map the result while the JDBC resources are open.
    }
}
~~~

Never build a predicate by concatenating user-provided values. Bind parameters represent values, not SQL identifiers or keywords. If a request allows sorting by a column, map the requested sort option to a fixed allowlist of known column names and directions; do not insert arbitrary text into `ORDER BY`. Also apply authorization, least-privilege database accounts, and careful error handling. Parameter binding prevents injection through the bound values, but it does not fix insecure authorization logic or make an unsafe dynamic SQL design safe.

### Q46. How do you model a many-to-many relationship correctly?

Use a junction (association) table that stores one row per relationship. For example, `student_course(student_id, course_id)` connects students and courses. The two columns can form a composite primary key, or the table can use a surrogate key plus a unique constraint on `(student_id, course_id)` to prevent duplicate relationships.

Add foreign keys to both parent tables when their integrity and lifecycle requirements call for them. Indexes should support the actual access paths: a primary key on `(student_id, course_id)` supports lookups by student and course pair, while an additional index beginning with `course_id` may be needed for listing students by course. The order of a composite index matters.

If the relationship has attributes such as enrollment date, role, quantity, or status, those attributes belong on the junction table. Choose delete behavior deliberately: cascading deletion may be appropriate for pure association rows, but should not accidentally delete independently meaningful business records. In Java/JPA, model the association explicitly when it has its own lifecycle or attributes rather than hiding important business behavior behind a bare many-to-many mapping.

### Q47. How do you implement an upsert in MySQL, and what concurrency issue remains?

An upsert attempts to insert a row and update an existing row when a defined uniqueness conflict occurs. In MySQL, use `INSERT ... ON DUPLICATE KEY UPDATE`. The operation relies on a primary-key or unique-key conflict; the application should still handle its exact retry and update semantics intentionally.

The conflict must be based on a real database uniqueness rule, such as a primary key or unique business key. An application-level “SELECT first, then INSERT” is racy: two requests can both observe that the row is absent. A unique constraint is the durable arbiter under concurrency; the application should handle a duplicate-key outcome intentionally.

Be explicit about what “update” means. Should a retry overwrite all fields, preserve fields already set, increment a counter, or return the result of the first request? Those are business semantics, not syntax details. Upsert is not automatically idempotent—for example, an update that increments a value may apply twice on retry. Use an idempotency key or operation record when retries must return the same logical result.

### Q48. How do you safely update or delete millions of rows?

Start by defining the selection criteria, expected row count, business impact, and recovery plan. Test the predicate with a SELECT, inspect the access path, and ensure the batch cannot unintentionally affect the entire table. A single huge transaction can hold locks for a long time, generate substantial undo/redo or binary-log activity, delay replication, and make rollback expensive.

For suitable workloads, process bounded, deterministic batches—often by a stable primary-key range—committing each batch and recording progress. Make the operation resumable and idempotent, and ensure concurrent inserts or updates cannot cause rows to be skipped. Throttling and pauses may be necessary to protect latency-sensitive traffic. The exact batch size depends on row size, indexes, replication, lock contention, and service-level objectives.

For schema changes, use an expand–migrate–contract rollout when old and new application versions must coexist. “Online” DDL does not mean every phase is lock-free. Monitor lock waits, transaction age, replica lag, error rates, and application latency. Never assume that a rollback will be cheap after a large data change; prepare and rehearse a recovery or roll-forward plan.

### Q49. When should you use SELECT ... FOR UPDATE instead of optimistic locking?

A locking read such as `SELECT ... FOR UPDATE` requests locks on selected rows so another transaction cannot modify the locked records incompatibly until the transaction completes. It is useful when the application must inspect current state and then make a related decision and update within the same short transaction—for example, reserving inventory based on the current quantity.

Optimistic locking instead lets concurrent work proceed and detects a conflict using a version column or conditional update. It is often a good fit when conflicts are relatively rare and the application can reload or report a conflict. Pessimistic locking may be preferable when a critical section needs to serialize competing writers, but it increases waiting and deadlock risk.

Lock scope and behavior depend on the engine, isolation level, index access path, and predicate. In InnoDB, locking reads can take record and range-related locks depending on the query and isolation level; an appropriate index helps avoid locking more rows than intended. Keep the transaction short, access rows in a consistent order, and never hold database locks while waiting on user input or slow remote calls.

### Q50. What is the difference between a view and a materialized view?

A regular view stores a query definition and presents its result when queried; it does not normally persist a separate copy of every result row. It can simplify complex joins, provide a stable interface over tables, and help restrict which columns are exposed, but it is not automatically a performance optimization. The optimizer may merge or otherwise transform the view query, depending on the database and query shape.

MySQL provides regular views but not the same native materialized-view feature offered by some other database engines. For a precomputed result, teams commonly implement a summary table or read model and maintain it with application logic, jobs, or events.

The trade-off is freshness and operational complexity versus faster repeated reads. Before introducing a materialized result, define refresh frequency, acceptable staleness, refresh failure handling, and how inserts, updates, and deletes affect the result. For many backend workloads, a well-indexed query or Redis cache may be simpler; choose based on measured requirements.

## Essential SQL Interview Questions: Joins, Keys, and Core Query Patterns

This section covers frequently tested fundamentals that deserve direct, focused answers. Examples use broadly compatible MySQL syntax and avoid requiring CTEs or window functions.

### Q51. What are the different types of SQL joins?

- **INNER JOIN:** returns only matching rows from both tables.
- **LEFT JOIN / LEFT OUTER JOIN:** returns all left-table rows and matching right-table rows; unmatched right-side columns are NULL.
- **RIGHT JOIN / RIGHT OUTER JOIN:** returns all right-table rows and matching left-table rows. It can usually be rewritten as a LEFT JOIN by swapping table order.
- **CROSS JOIN:** returns every possible pair of rows from both tables.
- **SELF JOIN:** joins a table to itself using aliases; it is a pattern rather than a separate join operator.
- **FULL OUTER JOIN:** returns matched rows plus unmatched rows from both sides. MySQL does not directly support this syntax.

Example:

~~~sql
SELECT e.employee_id, e.name, d.department_name
FROM employees e
INNER JOIN departments d
  ON d.department_id = e.department_id;
~~~

Choose the join based on which rows must appear. Do not assume one join type is inherently faster; verify the execution plan for performance questions.

### Q52. What is the difference between INNER JOIN and LEFT JOIN?

INNER JOIN excludes rows without a match. LEFT JOIN preserves every row from the left table and returns NULL for right-side columns when no match exists.

~~~sql
SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o
  ON o.customer_id = c.customer_id;
~~~

This includes customers who have never placed an order. A customer with multiple orders appears multiple times—one row per matching order. If you only need customers who have at least one order, EXISTS may be clearer and avoids multiplying customer rows.

### Q53. What is the difference between LEFT JOIN and RIGHT JOIN?

LEFT JOIN preserves all rows from the left input; RIGHT JOIN preserves all rows from the right input. A RIGHT JOIN can generally be rewritten as a LEFT JOIN by swapping the table order and keeping the join condition equivalent. Teams often standardize on LEFT JOIN because reading from the primary entity outward is easier to follow. The key interview point is identifying which side must be preserved, not memorizing a performance rule.

### Q54. What is a CROSS JOIN, and when is it useful?

CROSS JOIN returns the Cartesian product. If one table has four rows and another has three, the result contains twelve rows. It can be useful for generating combinations, such as every store for every calendar date.

~~~sql
SELECT s.store_id, d.calendar_date
FROM stores s
CROSS JOIN calendar_dates d;
~~~

The output grows multiplicatively. An accidentally omitted join condition can create a huge result and cause serious performance problems, so estimate the expected row count before running the query.

### Q55. What is a SELF JOIN? Give a practical example.

A self join joins a table to itself. It is commonly used for hierarchies, such as employees referring to their managers through manager_id.

~~~sql
SELECT e.employee_id,
       e.name AS employee_name,
       m.name AS manager_name
FROM employees e
LEFT JOIN employees m
  ON m.employee_id = e.manager_id;
~~~

The LEFT JOIN retains employees without a manager. Aliases distinguish the employee row from the manager row. For editable hierarchies, also consider how cycles or invalid manager references are prevented.

### Q56. Does MySQL support FULL OUTER JOIN? How can it be emulated?

MySQL does not directly support FULL OUTER JOIN syntax. A common workaround is to return all left-side rows with a LEFT JOIN, then append only unmatched right-side rows using UNION ALL.

~~~sql
SELECT a.id, a.value AS a_value, b.value AS b_value
FROM table_a a
LEFT JOIN table_b b ON b.id = a.id

UNION ALL

SELECT b.id, a.value AS a_value, b.value AS b_value
FROM table_b b
LEFT JOIN table_a a ON a.id = b.id
WHERE a.id IS NULL;
~~~

This assumes the join keys are non-NULL for real rows. The second query adds only right-side rows without a match, so UNION ALL preserves matched results from the first query without unnecessary duplicate elimination.

### Q57. What is the difference between ON and WHERE with an outer join?

ON defines which rows match during the join; WHERE filters the resulting rows. Moving a right-table predicate from ON to WHERE can remove NULL-extended rows and effectively turn a LEFT JOIN into an inner-style result.

~~~sql
SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o
  ON o.customer_id = c.customer_id
 AND o.status = 'PAID';
~~~

This keeps every customer and attaches paid orders when present. If o.status = 'PAID' is put in WHERE instead, customers without a paid order are filtered out. Explain whether unmatched rows must remain before choosing where the predicate belongs.

### Q58. What are semi joins and anti joins? How do EXISTS and NOT EXISTS help?

A semi join returns rows from one side when a match exists without multiplying each left row for every match; SQL commonly expresses this using EXISTS. An anti join returns rows with no match and is commonly expressed using NOT EXISTS.

~~~sql
SELECT c.customer_id
FROM customers c
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.customer_id
);
~~~

For customers without orders, use NOT EXISTS instead. NOT EXISTS is often safer than NOT IN if the subquery can return NULL, because NULL can make a NOT IN predicate evaluate to UNKNOWN.

### Q59. Is JOIN faster than a subquery?

Neither is always faster. A JOIN is natural when columns from both tables are needed. A subquery is useful for existence checks, scalar values, or isolating an aggregate. The optimizer may transform equivalent forms into similar plans. Select the form that most accurately expresses the requirement, then use EXPLAIN and representative measurements. Also consider result cardinality: a join may return multiple rows per parent, while EXISTS can return each qualifying parent row once.

### Q60. Why can a JOIN produce duplicate rows?

A join produces one output row for every matching pair. If a customer has five orders, joining customers to orders returns five rows for that customer. This is expected one-to-many behavior, not necessarily bad duplicate data.

Determine the required output grain first. If you need one row per customer, consider EXISTS, aggregating orders before joining, or selecting one order according to a defined business rule. Do not add DISTINCT automatically; it may hide a wrong join condition or an aggregate inflated by row multiplication.

### Q61. What is the difference between a primary key, unique key, and foreign key?

- **Primary key:** uniquely identifies a row and cannot be NULL. A composite primary key can contain multiple columns.
- **Unique key:** prevents duplicate key values. MySQL generally permits multiple NULL values in a unique index, so nullable-column behavior must be considered.
- **Foreign key:** enforces a reference to a key in another table or the same table, subject to engine and schema rules.

Database constraints protect integrity under concurrency. An application-side check followed by INSERT is racy because two requests may both see that a value is absent. Use a unique constraint as the final guard for a unique business identifier.

### Q62. What is a composite primary key?

A composite primary key uses two or more columns together to identify a row. It is common in junction tables where the combination represents a unique relationship.

~~~sql
CREATE TABLE student_course (
  student_id BIGINT NOT NULL,
  course_id BIGINT NOT NULL,
  enrolled_at DATETIME NOT NULL,
  PRIMARY KEY (student_id, course_id)
);
~~~

This prevents duplicate student-course pairs. The key supports lookups starting with student_id; queries filtering only by course_id may need another index beginning with course_id. Consider a surrogate key plus a unique constraint when other tables need a simpler reference.

### Q63. Explain 1NF, 2NF, and 3NF.

Normalization reduces redundant data and update anomalies.

- **1NF:** values are atomic for the chosen model; avoid storing repeating lists in one field.
- **2NF:** the table is in 1NF and non-key attributes depend on the whole candidate key, not just part of a composite key.
- **3NF:** the table is in 2NF and non-key attributes do not depend transitively on a candidate key through another non-key attribute.

For example, storing department_name on every employee row can create inconsistent values when a department is renamed. A separate departments table and foreign key avoid this duplication. Normalize for integrity first; denormalize only when measured workload needs justify the trade-off.

### Q64. What is denormalization, and when is it justified?

Denormalization deliberately duplicates or precomputes data to make frequent reads cheaper, such as maintaining a daily sales summary rather than repeatedly aggregating millions of transactions. The cost is extra storage, write complexity, and possible stale or inconsistent values. Before denormalizing, inspect query plans, indexes, cache effectiveness, and read/write patterns. Define how changes propagate, how failures are repaired, and what staleness is acceptable.

### Q65. What is the difference between WHERE and HAVING?

WHERE filters individual rows before grouping. HAVING filters groups after aggregation.

~~~sql
SELECT department_id, COUNT(*) AS employee_count
FROM employees
WHERE status = 'ACTIVE'
GROUP BY department_id
HAVING COUNT(*) >= 5;
~~~

This counts active employees per department and returns only departments with at least five active employees. COUNT(*) belongs in HAVING because it is an aggregate. Use WHERE for row-level predicates so unnecessary rows are filtered before grouping.

### Q66. What is the difference between COUNT(*), COUNT(column), and COUNT(DISTINCT column)?

- COUNT(*) counts result rows, including rows where some columns are NULL.
- COUNT(column) counts rows where that expression is not NULL.
- COUNT(DISTINCT column) counts distinct non-NULL values of the expression.

~~~sql
SELECT COUNT(*) AS total_rows,
       COUNT(manager_id) AS employees_with_manager,
       COUNT(DISTINCT department_id) AS departments_represented
FROM employees;
~~~

A join can multiply rows before aggregation. If a count unexpectedly increases after adding a join, check relationship cardinality and whether the requirement is to count rows, entities, or distinct business keys.

### Q67. What is the difference between UNION and UNION ALL?

Both combine compatible SELECT results. UNION removes duplicate result rows, while UNION ALL preserves them and usually avoids the cost of duplicate elimination. Use UNION ALL when duplicates are meaningful or the input sets are known to be disjoint. Use UNION only when deduplication is required. The SELECTs must return the same number of columns in compatible positions and types. Do not use UNION to conceal an incorrect join.

### Q68. What is the difference between DELETE, TRUNCATE, and DROP in MySQL?

- DELETE removes selected rows and supports WHERE. InnoDB changes can be rolled back before commit.
- TRUNCATE TABLE removes all rows, does not support a row filter, is DDL, and causes an implicit commit.
- DROP TABLE removes the table object and its definition.

Check foreign keys, triggers, privileges, and recovery requirements before destructive operations. Do not assume TRUNCATE can be rolled back like an ordinary InnoDB DELETE. Preview the target set and use the least destructive operation that meets the requirement.

### Q69. How do you find duplicate values safely?

Group by the business key and retain groups containing more than one row.

~~~sql
SELECT email, COUNT(*) AS occurrences
FROM users
WHERE email IS NOT NULL
GROUP BY email
HAVING COUNT(*) > 1;
~~~

Define duplicate semantics before deleting anything: case sensitivity, whitespace, and NULL handling may matter. After cleaning existing data, a unique constraint can prevent recurrence if the database collation and constraint behavior match the business rule.

### Q70. How do you find the Nth-highest distinct salary without window functions?

Sort distinct, non-NULL salaries descending and skip the first N-1 values. For the second-highest distinct salary:

~~~sql
SELECT DISTINCT salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
~~~

For the Nth-highest distinct salary, use LIMIT 1 OFFSET N-1, replacing N with the desired rank minus one. OFFSET is zero-based. If fewer than N distinct non-NULL salaries exist, the query returns no rows. Validate any application-supplied offset as a non-negative integer.

### Q71. What is the difference between CHAR and VARCHAR?

CHAR is fixed-length character storage, while VARCHAR stores variable-length strings up to the declared limit. CHAR may suit fixed-width values; VARCHAR is generally natural for names, emails, and labels. Actual storage and comparison behavior also depend on character set, collation, row format, and MySQL version. Choose a length based on valid business data and do not select a type based only on a presumed speed difference.

### Q72. What is the difference between DATE, DATETIME, and TIMESTAMP in MySQL?

DATE stores a calendar date without a time. DATETIME stores date and time without automatic time-zone conversion. TIMESTAMP uses session-time-zone conversion between the stored UTC instant and the session representation, and its supported range differs from DATETIME.

For day filtering, use a half-open range rather than applying DATE() to an indexed timestamp column:

~~~sql
SELECT order_id, created_at
FROM orders
WHERE created_at >= '2026-01-15 00:00:00'
  AND created_at <  '2026-01-16 00:00:00';
~~~

Define whether the value represents an instant or local wall-clock time. Standardizing instants to UTC and handling display time zones at application boundaries is common, but the application and connection configuration must agree.

### Q73. What is the difference between a normal index and a unique index?

A normal index helps locate or order rows but permits duplicate indexed values. A unique index also enforces uniqueness. Both consume storage and increase write maintenance. MySQL unique-index behavior with NULL values may surprise people who assume NULLs compare equal, so declare columns NOT NULL when that matches the business rule. The optimizer may still choose a table scan if that is cheaper.

### Q74. Explain the leftmost-prefix rule for composite indexes.

For an index on (customer_id, status, created_at), queries filtering by customer_id or by customer_id plus status can commonly use the leading prefix. A query filtering only by status generally cannot use the leading prefix as effectively because it skips customer_id.

~~~sql
CREATE INDEX idx_orders_customer_status_created
ON orders (customer_id, status, created_at);
~~~

Choose column order based on real predicates, range conditions, sorting, and join patterns. A range condition can limit how later index columns narrow the scan. Verify the plan and write overhead rather than applying a universal “most selective first” rule.

### Q75. What is a transaction, and why does ACID matter?

A transaction groups operations into a logical unit. ACID means Atomicity (all-or-nothing), Consistency (preserve declared invariants), Isolation (control interference between concurrent transactions), and Durability (committed changes survive failures within system guarantees).

For a money transfer, debit and credit should be in one transaction so only one side cannot commit. A transaction does not make a multi-service workflow automatically atomic; distributed workflows may need idempotency, an outbox, or a saga.

### Q76. What are transaction isolation levels and common anomalies?

The standard levels are Read Uncommitted, Read Committed, Repeatable Read, and Serializable. Textbook anomalies include dirty reads, non-repeatable reads, and phantom reads, but actual behavior depends on the storage engine, isolation level, locking, and MVCC implementation. MySQL InnoDB commonly defaults to Repeatable Read, though deployments can configure this differently. For correctness-critical operations, reason about the exact engine and transaction boundaries, and consider constraints, conditional updates, optimistic version checks, and locking reads.

### Q77. What causes deadlocks, and how should a Java backend handle them?

A deadlock occurs when transactions form a cycle of waiting—for example, transaction A holds row 1 and waits for row 2 while transaction B holds row 2 and waits for row 1. InnoDB detects deadlocks and rolls back one transaction.

Reduce risk by accessing rows in a consistent order, keeping transactions short, indexing predicates, and avoiding remote calls while holding locks. The application should retry the entire transaction a bounded number of times for retryable deadlock failures, with backoff. Retrying only the last statement is generally incorrect if the transaction has been rolled back.

### Q78. How do you prevent SQL injection in Java/JDBC?

Use prepared statements with bound values instead of concatenating untrusted input into SQL.

~~~java
String sql = "SELECT user_id, display_name FROM users WHERE email = ?";
try (PreparedStatement ps = connection.prepareStatement(sql)) {
    ps.setString(1, email);
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            // Map each row while the ResultSet is open.
        }
    }
}
~~~

Bind parameters represent values, not table names, column names, or keywords. For dynamic sorting, map the requested option to an allowlist of column names and directions. Prepared statements do not replace authorization checks or least-privilege database accounts.

### Q79. How do you investigate a slow MySQL query?

Identify the exact SQL, parameter values, frequency, latency, rows returned, and whether time is spent in the database or application. Use EXPLAIN to inspect access types, chosen indexes, join order, estimated rows, and sorting or temporary-table work. Check slow-query logs, lock waits, statistics, and production metrics. Make one evidence-based change at a time and measure it on representative data. An index is not always beneficial, and a full scan is not always wrong.

### Q80. What is offset pagination versus keyset pagination?

Offset pagination uses LIMIT and OFFSET and is simple for numbered pages, but large offsets may be expensive and concurrent inserts/deletes can shift results. Keyset pagination starts after the last row seen and is often more efficient for feeds.

Offset example:

~~~sql
SELECT order_id, created_at
FROM orders
ORDER BY created_at DESC, order_id DESC
LIMIT 20 OFFSET 40;
~~~

Keyset example, assuming created_at is NOT NULL:

~~~sql
SELECT order_id, created_at
FROM orders
WHERE created_at < ?
   OR (created_at = ? AND order_id < ?)
ORDER BY created_at DESC, order_id DESC
LIMIT 20;
~~~

The unique order_id tie-breaker is essential. Keyset pagination is not ideal when users must jump directly to arbitrary page numbers; choose based on navigation requirements, indexes, and data-change behavior.


## References

The question selection reflects recurring themes across multiple interview-preparation resources—joins, filtering and aggregation, NULL behavior, ranking/window functions, practical query tasks, indexes, transactions, and performance troubleshooting. These sources are useful for identifying repeated themes, not for proving that any specific company asks a question or for estimating interview probabilities.

### Interview-topic cross-checks

- [DataCamp — Top SQL Interview Questions and Answers](https://www.datacamp.com/blog/top-sql-interview-questions-and-answers-for-beginners-and-intermediate-practitioners)
- [Coddy — SQL Interview Questions](https://coddy.tech/interview-questions/sql)
- [Weekday — SQL Interview Questions and Answers](https://www.weekday.works/post/sql-interview-questions)
- [buggedbrains — SQL Interview Questions](https://buggedbrains.com/interview/sql-interview)

### Official MySQL documentation

- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/)
- [MySQL — Optimization and Indexes](https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html)
- [MySQL — Multiple-Column Indexes](https://dev.mysql.com/doc/refman/8.4/en/multiple-column-indexes.html)
- [MySQL — EXPLAIN and EXPLAIN ANALYZE](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [MySQL — TRUNCATE TABLE](https://dev.mysql.com/doc/refman/8.4/en/truncate-table.html)
- [MySQL — InnoDB Transaction Model and Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-model.html)
- [MySQL — InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- [MySQL — Prepared Statements](https://dev.mysql.com/doc/refman/8.4/en/sql-prepared-statements.html)
- [MySQL — Locking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)
- [MySQL — INSERT ... ON DUPLICATE KEY UPDATE](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html)
- [MySQL — Views](https://dev.mysql.com/doc/refman/8.4/en/views.html)

---

## Final self-check

Before considering this handbook mastered, make sure you can do each of the following without looking at the answers:

- [ ] Explain `WHERE` versus `HAVING`, NULL logic, and why `NOT EXISTS` can be safer than `NOT IN`.
- [ ] Write a join and explain how one-to-many relationships can inflate aggregates.
- [ ] Solve second-highest salary, top-N-per-group, latest-row-per-customer, and customers-with-no-orders problems using basic MySQL queries.
- [ ] Explain index trade-offs and walk through a measured slow-query investigation.
- [ ] Describe a lost update, a deadlock, and how optimistic locking or an atomic update addresses them.
- [ ] Explain why pagination needs a deterministic order and when keyset pagination is preferable.
- [ ] Explain MySQL-specific behavior for `LIMIT`, `AUTO_INCREMENT`, InnoDB indexes, and transaction boundaries.

## Revision strategy

For each question, practise explaining the core idea aloud, writing the SQL without looking, and then naming one edge case. For performance and concurrency questions, explain the investigation steps and trade-offs before jumping to a fix. Prefer understanding query behavior and data cardinality over memorizing a single syntax pattern. Treat all destructive SQL examples as patterns to adapt: preview affected rows, verify keys and constraints, and test in a safe environment before production use.


