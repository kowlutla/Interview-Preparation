# SQL & Database Engineering Interview Q&A Handbook

A focused interview handbook for Java backend engineers preparing for mid-level and senior software engineering interviews. Examples use broadly supported SQL and call out Oracle Database and MySQL differences where they matter.

> **Compatibility note:** Unless marked otherwise, examples use common SQL supported by modern Oracle Database and MySQL 8.x. Check the target database/version and schema constraints before using a query in production. The examples are explanatory; they have not been executed against a live database as part of this handbook.

## Table of contents

- [Top 15 Questions to Master First](#top-15-questions-to-master-first)
- [SQL Fundamentals and Query Logic](#sql-fundamentals-and-query-logic)
- [Joins, Aggregation, and Query Composition](#joins-aggregation-and-query-composition)
- [Window Functions and Practical SQL](#window-functions-and-practical-sql)
- [Indexes and Query Performance](#indexes-and-query-performance)
- [Transactions and Concurrency](#transactions-and-concurrency)
- [Oracle and MySQL in Practice](#oracle-and-mysql-in-practice)
- [Backend Production Scenarios](#backend-production-scenarios)
- [References](#references)

## Top 15 Questions to Master First

1. [Q1. What is the difference between WHERE and HAVING?](#q1-what-is-the-difference-between-where-and-having)
2. [Q2. What is the difference between DELETE, TRUNCATE, and DROP?](#q2-what-is-the-difference-between-delete-truncate-and-drop)
3. [Q3. How do NULL values behave in SQL?](#q3-how-do-null-values-behave-in-sql)
4. [Q8. What is the difference between INNER JOIN and LEFT JOIN, and how can ON versus WHERE change results?](#q8-what-is-the-difference-between-inner-join-and-left-join-and-how-can-on-versus-where-change-results)
5. [Q13. How do you find the second-highest distinct salary?](#q13-how-do-you-find-the-second-highest-distinct-salary)
6. [Q12. How do ROW_NUMBER, RANK, and DENSE_RANK differ?](#q12-how-do-rownumber-rank-and-denserank-differ)
7. [Q14. How do you find the highest-paid employee in each department?](#q14-how-do-you-find-the-highest-paid-employee-in-each-department)
8. [Q19. How do you identify duplicates and remove them safely?](#q19-how-do-you-identify-duplicates-and-remove-them-safely)
9. [Q21. How do indexes work, and what are their trade-offs?](#q21-how-do-indexes-work-and-what-are-their-trade-offs)
10. [Q22. How do composite indexes work, and why does column order matter?](#q22-how-do-composite-indexes-work-and-why-does-column-order-matter)
11. [Q25. How do you investigate and optimize a slow SQL query?](#q25-how-do-you-investigate-and-optimize-a-slow-sql-query)
12. [Q27. What are ACID properties?](#q27-what-are-acid-properties)
13. [Q28. Explain transaction isolation levels and common anomalies.](#q28-explain-transaction-isolation-levels-and-common-anomalies)
14. [Q30. What causes deadlocks, and how should an application handle them?](#q30-what-causes-deadlocks-and-how-should-an-application-handle-them)
15. [Q33. How should pagination be implemented in Oracle and MySQL?](#q33-how-should-pagination-be-implemented-in-oracle-and-mysql)

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

Do not assume the three commands have identical transaction semantics. In Oracle, DDL statements such as `TRUNCATE` and `DROP` cause implicit commits. In MySQL, `TRUNCATE TABLE` is DDL and causes an implicit commit; transactional rollback should not be expected to restore the table contents. `DELETE` on an InnoDB table can be rolled back before commit. Check foreign keys, triggers, privileges, and backup/recovery implications before destructive operations.

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

Moving `o.status = 'PAID'` into `WHERE` filters out the NULL-extended rows and effectively turns this condition into an inner-style result for that predicate. Put match criteria in `ON` when unmatched left rows must remain; put final row filters in `WHERE` when excluding rows is intended. Oracle and MySQL support these join forms; FULL OUTER JOIN availability differs, and MySQL does not directly support the standard `FULL OUTER JOIN` syntax.

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

### Q11. When would you use a subquery or a common table expression (CTE)?

A subquery expresses a query inside another query. A CTE, introduced with `WITH`, names a query expression so it can be read and referenced more clearly. CTEs are useful for breaking a multi-step transformation into understandable stages, including ranking then filtering.

```sql
WITH department_totals AS (
  SELECT department_id, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department_id
)
SELECT e.employee_id, e.salary
FROM employees e
JOIN department_totals d
  ON d.department_id = e.department_id
WHERE e.salary > d.avg_salary;
```

A CTE is not automatically faster than a subquery or join. Whether it is inlined, merged, or materialized depends on the database, version, and query shape. Use the clearest correct form first, then inspect the execution plan. Recursive CTE support and syntax details also vary by database/version.

## Window Functions and Practical SQL

### Q12. How do ROW_NUMBER, RANK, and DENSE_RANK differ?

These window functions assign ordering numbers within a result set or partition. `ROW_NUMBER()` assigns a unique sequential number to each row, even when sort values tie. `RANK()` gives tied rows the same rank and leaves gaps afterward. `DENSE_RANK()` gives tied rows the same rank without gaps.

For salaries `100, 90, 90, 80`, the ranks are:

| Salary | ROW_NUMBER (one possible order) | RANK | DENSE_RANK |
|---:|---:|---:|---:|
| 100 | 1 | 1 | 1 |
| 90 | 2 | 2 | 2 |
| 90 | 3 | 2 | 2 |
| 80 | 4 | 4 | 3 |

Use a deterministic tie-breaker with `ROW_NUMBER` when exactly one row must be selected. If the business requirement is to return all tied top earners, use `RANK` or filter by the maximum value instead. A window's `ORDER BY` defines ranking order; it does not guarantee the final display order, so add a query-level `ORDER BY` if required.

### Q13. How do you find the second-highest distinct salary?

Rank distinct salary values or use the second row of a distinct descending list. A window function makes the intent explicit and avoids the common mistake of treating two employees tied for the highest salary as first and second.

```sql
WITH salary_ranks AS (
  SELECT salary,
         DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
  FROM employees
  WHERE salary IS NOT NULL
)
SELECT DISTINCT salary
FROM salary_ranks
WHERE salary_rank = 2;
```

This returns the second distinct salary, and returns no row if fewer than two distinct non-NULL salaries exist. If the requirement is the second employee after sorting (not the second distinct salary), use `ROW_NUMBER()` over employees with a deterministic tie-breaker. Clarify the requirement before coding.

### Q14. How do you find the highest-paid employee in each department?

Use a window function partitioned by department. Choose `RANK` if every employee tied for the highest salary should be returned; choose `ROW_NUMBER` if exactly one winner is required and define a tie-breaker.

```sql
WITH ranked AS (
  SELECT employee_id, department_id, salary,
         RANK() OVER (
           PARTITION BY department_id
           ORDER BY salary DESC
         ) AS salary_rank
  FROM employees
  WHERE salary IS NOT NULL
)
SELECT employee_id, department_id, salary
FROM ranked
WHERE salary_rank = 1
ORDER BY department_id, employee_id;
```

The example excludes NULL salaries. If departments with no non-NULL salary must also appear, start from the department table and left join an appropriately ranked employee result. Be explicit about whether NULL salaries count and whether all ties should be returned.

### Q15. How do you find the top N records per group?

Use a ranking function with `PARTITION BY` and filter the result in an outer query or CTE. For example, to return the three most recent orders per customer:

```sql
WITH ranked_orders AS (
  SELECT order_id, customer_id, created_at,
         ROW_NUMBER() OVER (
           PARTITION BY customer_id
           ORDER BY created_at DESC, order_id DESC
         ) AS rn
  FROM orders
)
SELECT order_id, customer_id, created_at
FROM ranked_orders
WHERE rn <= 3
ORDER BY customer_id, created_at DESC, order_id DESC;
```

The unique `order_id` tie-breaker makes the choice deterministic when timestamps match. `ROW_NUMBER` returns at most N rows per group; `RANK` or `DENSE_RANK` can return more than N rows when ties qualify. Window functions are available in modern Oracle and MySQL 8.0+; older MySQL versions need a different approach.

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

Rank records per customer by timestamp and a unique tie-breaker, then keep rank one.

```sql
WITH ranked AS (
  SELECT e.*,
         ROW_NUMBER() OVER (
           PARTITION BY customer_id
           ORDER BY updated_at DESC, event_id DESC
         ) AS rn
  FROM customer_events e
)
SELECT customer_id, event_id, updated_at, event_type
FROM ranked
WHERE rn = 1;
```

Define what “latest” means: event time, creation time, or update time can differ. If timestamps can be NULL, specify where NULLs belong using database-appropriate syntax or a normalized expression. Without a stable tie-breaker, the selected row among equal timestamps is not guaranteed. If the table is large, an index beginning with `customer_id` and followed by the ordering columns may help, depending on the access pattern and optimizer.

### Q18. How do you calculate a running total?

Use a windowed aggregate. Unlike `GROUP BY`, a window function keeps each detail row and adds a cumulative value.

```sql
SELECT account_id, transaction_id, transaction_time, amount,
       SUM(amount) OVER (
         PARTITION BY account_id
         ORDER BY transaction_time, transaction_id
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_balance_change
FROM account_transactions
ORDER BY account_id, transaction_time, transaction_id;
```

The explicit `ROWS` frame and deterministic ordering avoid ambiguity from peer rows with equal sort values. This calculates a cumulative sum of transaction amounts, not necessarily a bank balance; an opening balance would need to be added. Consider how NULL amounts and reversals should be treated.

### Q19. How do you identify duplicates and remove them safely?

First define a duplicate by its business key, not by assuming every identical-looking row is invalid. Find duplicate groups with `GROUP BY` and `HAVING COUNT(*) > 1`.

```sql
SELECT email, COUNT(*) AS occurrences
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

Before deleting anything, preview the rows and decide which record should survive. The following examples assume `user_id` is a unique, non-NULL key and that only repeated, non-NULL email addresses count as duplicates. Change the survivor rule if the business requirement differs.

**Preview the rows that would be removed:**

```sql
WITH ranked AS (
  SELECT user_id, email, created_at,
         ROW_NUMBER() OVER (
           PARTITION BY email
           ORDER BY created_at, user_id
         ) AS rn
  FROM users
  WHERE email IS NOT NULL
)
SELECT user_id, email, created_at
FROM ranked
WHERE rn > 1
ORDER BY email, rn;
```

**Oracle — delete by `ROWID` after reviewing the preview:**

```sql
DELETE FROM users
WHERE ROWID IN (
  SELECT rid
  FROM (
    SELECT ROWID AS rid,
           ROW_NUMBER() OVER (
             PARTITION BY email
             ORDER BY created_at, user_id
           ) AS rn
    FROM users
    WHERE email IS NOT NULL
  )
  WHERE rn > 1
);
```

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

Compute the average per department, then compare each employee with that department's average. A CTE keeps the aggregation separate from row-level filtering.

```sql
WITH department_average AS (
  SELECT department_id, AVG(salary) AS avg_salary
  FROM employees
  WHERE salary IS NOT NULL
  GROUP BY department_id
)
SELECT e.employee_id, e.department_id, e.salary
FROM employees e
JOIN department_average d
  ON d.department_id = e.department_id
WHERE e.salary > d.avg_salary;
```

An equivalent window-function solution uses `AVG(salary) OVER (PARTITION BY department_id)`. The CTE makes the two grains easy to reason about. Departments containing only NULL salaries have no average group here; clarify whether those employees should be excluded or reported separately.

## Indexes and Query Performance

### Q21. How do indexes work, and what are their trade-offs?

An index is an auxiliary data structure that helps the database locate rows without scanning every row. Common B-tree indexes support equality lookups, ranges, and ordered access. The optimizer decides whether an index is cheaper than alternatives based on selectivity, table size, statistics, required columns, and estimated I/O.

Indexes speed up some reads but consume storage and add work to inserts, updates, and deletes because index entries must be maintained. Too many indexes can hurt write throughput and increase maintenance cost. In MySQL InnoDB, the primary key is the clustered index and secondary indexes include the primary-key value; Oracle's heap-table organization and index structures differ. An index is not automatically beneficial: a full scan can be cheaper when a query needs a large fraction of the table.

### Q22. How do composite indexes work, and why does column order matter?

A composite index stores multiple columns in a defined order. For a B-tree index on `(status, created_at, customer_id)`, predicates that constrain the leftmost column and then a range on the next column are often well supported. A query filtering only by `customer_id` generally cannot use the leading prefix as efficiently as an index that begins with that column.

Choose column order from real query patterns: equality predicates, range predicates, join conditions, sorting requirements, and selectivity all matter. Do not rely on a simplistic “most selective column first” rule for every workload. A covering index contains all columns needed by a query and may avoid table-row lookups, but larger indexes consume more space and make writes more expensive. Confirm the benefit with the execution plan and representative data.

### Q23. Why might the optimizer ignore an index that exists?

An index may be ignored because a large proportion of rows qualifies, the table is small, the index does not match the predicate or leading-column pattern, statistics are stale, implicit type conversion occurs, or a function/expression prevents a normal index range lookup. The optimizer may correctly choose a table scan if that is cheaper.

For example, wrapping an indexed timestamp in a function can make range access harder:

```sql
-- Often less index-friendly
WHERE DATE(created_at) = '2026-01-15'

-- Range predicate; adapt date literal syntax to the database
WHERE created_at >= :start_time
  AND created_at <  :next_day_start
```

Use correctly typed bind parameters and half-open time ranges. Function-based indexes or generated-column indexes can be appropriate when the expression is intentional and supported. Refresh statistics using the database's supported procedure when evidence suggests they are stale; do not force an index hint before understanding the plan.

### Q24. What is a sargable predicate, and why does it matter?

A sargable predicate lets the optimizer use an index access path to narrow candidate rows efficiently. Equality and range conditions directly on an indexed column are common examples. Applying a function or arithmetic operation to the column can prevent ordinary index use unless a matching expression index or optimizer transformation applies.

Prefer `created_at >= :start AND created_at < :end` over converting every timestamp to a date in the predicate. Avoid leading wildcards such as `LIKE '%term'` when expecting a conventional B-tree index to perform a prefix lookup. Sargability is not a guarantee of index use; selectivity and cost still decide the plan.

### Q25. How do you investigate and optimize a slow SQL query?

Start with evidence: identify the exact SQL, bind values, frequency, latency, rows returned, and whether time is spent in the database, network, or application. Compare the slow query with a known-good baseline and inspect the execution plan. Look for estimated-versus-actual row mismatches where available, full scans over large selective workloads, expensive sorts, repeated lookups, poor join order, and unexpectedly large intermediate results.

A quick syntax reminder:

```sql
-- MySQL: estimated plan
EXPLAIN
SELECT order_id
FROM orders
WHERE customer_id = ?;

-- Oracle: estimated plan
EXPLAIN PLAN FOR
SELECT order_id
FROM orders
WHERE customer_id = :customer_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY);
```

The placeholder syntax shown is illustrative: the actual bind syntax depends on the database client or framework. A practical sequence is:

1. Reproduce with representative data and parameter values.
2. Inspect the plan and database metrics; check table/index statistics.
3. Verify join cardinality and filtering logic before changing indexes.
4. Test a targeted index or rewrite, considering write cost and storage.
5. Measure before and after under comparable conditions.
6. Roll out safely and monitor latency, load, and regressions.

Oracle tools include `EXPLAIN PLAN`, `DBMS_XPLAN`, and runtime plan information where privileges and configuration allow. MySQL supports `EXPLAIN` and, in modern versions including 8.0.18+, `EXPLAIN ANALYZE` for supported statements; it executes the statement to collect actual timing and row data. Be careful with `EXPLAIN ANALYZE` on statements that modify data. An execution plan is evidence to interpret, not a score where any full scan is automatically bad.

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

Oracle's usual Read Committed behavior gives each statement a consistent view as of that statement; Oracle also supports Serializable and Read Only transaction modes. InnoDB's default isolation level in MySQL 8.4 is Repeatable Read, using MVCC for consistent nonlocking reads and locks for writes; locking and predicate behavior matters. The textbook anomaly table is not a complete description of every engine's behavior. For a concurrency bug, reason about the exact engine, isolation level, transaction boundaries, and whether the query is a locking read.

### Q29. What is MVCC, and how is it different from locking?

Multi-Version Concurrency Control (MVCC) keeps enough row versions or undo information for readers to observe a consistent snapshot while writers modify data. This can reduce reader-writer blocking, but it does not eliminate locks: writes still need coordination, and some reads explicitly request locks.

Oracle uses undo-based consistent reads. InnoDB combines MVCC with row-level locking; ordinary consistent reads and locking reads have different semantics. Long-running transactions can retain old versions longer, increase cleanup work, and contribute to storage pressure. Do not assume that “MVCC means no blocking” or that a normal SELECT always sees the newest committed value at the instant it finishes.

### Q30. What causes deadlocks, and how should an application handle them?

A deadlock occurs when transactions form a cycle of dependencies—for example, transaction A locks row 1 and waits for row 2, while transaction B locks row 2 and waits for row 1. The database detects a cycle and aborts a participant to release resources.

Reduce risk by acquiring rows in a consistent order, keeping transactions short, indexing predicates so updates lock fewer rows where applicable, and avoiding user/network calls while a transaction is open. Applications should treat deadlock victims as failed transactions and may retry the entire transaction with bounded retries and jitter when the operation is safe to repeat. Do not retry indefinitely or retry only the last statement if earlier work was rolled back. A unique business key or idempotency mechanism can help prevent duplicate side effects during retries.

### Q31. What is the difference between optimistic and pessimistic locking?

Optimistic locking assumes conflicts are uncommon. A typical implementation adds a version column and updates only if the version has not changed:

```sql
UPDATE inventory
SET quantity = :new_quantity,
    version = version + 1
WHERE item_id = :item_id
  AND version = :expected_version;
```

If the affected-row count is zero, the row was changed or no longer exists; reload and resolve the conflict. In JPA, a `@Version` field can provide this pattern.

Pessimistic locking acquires locks before changes are made, such as a supported `SELECT ... FOR UPDATE`, preventing competing transactions from modifying the locked rows until completion. It can be appropriate for high-contention workflows but increases blocking and deadlock risk. Choose based on contention, latency, correctness requirements, and database support; keep the transaction brief.

### Q32. Why are long-running transactions dangerous for a backend service?

They hold locks longer, increase the chance of blocking and deadlocks, and can delay cleanup of old row versions. In a connection-pooled Java service, a transaction that waits on remote HTTP calls or performs lengthy computation may also occupy a database connection for too long, contributing to pool exhaustion.

Keep database transactions around the smallest unit of work that must be atomic. Do not call slow external services while holding database locks unless the design explicitly requires it. Monitor transaction age, lock waits, connection acquisition time, pool active/idle counts, and query latency. Increasing the connection pool alone may overload the database or worsen contention; identify whether connections are slow, leaked, or held by overly broad transaction boundaries.

## Oracle and MySQL in Practice

### Q33. How should pagination be implemented in Oracle and MySQL?

Always use a deterministic `ORDER BY`; pagination without stable ordering can return inconsistent pages. Offset pagination is straightforward, but large offsets may require the database to walk and discard many rows. Concurrent inserts or deletes can also shift records between requests.

**MySQL 8.x:**

```sql
SELECT order_id, created_at
FROM orders
ORDER BY created_at DESC, order_id DESC
LIMIT ? OFFSET ?;
```

**Oracle 12c and later:**

```sql
SELECT order_id, created_at
FROM orders
ORDER BY created_at DESC, order_id DESC
OFFSET :offset ROWS FETCH NEXT :page_size ROWS ONLY;
```

Bind-marker syntax depends on the client or framework; `?` and `:name` are examples, not interchangeable raw-SQL syntax across all drivers.

For deep pagination, consider keyset (seek) pagination. Pass the last row's sort keys and request rows after that position, using a predicate consistent with the ordering. Assuming `created_at` is non-NULL, the next-page predicate for descending `(created_at, order_id)` is:

```sql
-- MySQL 8.x; use the bind syntax supported by your driver
SELECT order_id, created_at
FROM orders
WHERE created_at < ?
   OR (created_at = ? AND order_id < ?)
ORDER BY created_at DESC, order_id DESC
LIMIT ?;
```

For Oracle 12c and later, use the same predicate and ordering with `FETCH NEXT :page_size ROWS ONLY` instead of `LIMIT ?`. Bind the last row's timestamp twice and its `order_id` once. The `order_id` tie-breaker is essential when timestamps match. If the sort column can be NULL, define explicit NULL ordering and build a predicate that matches it. Keyset pagination works best with a compatible index and avoids scanning/discarding a large offset, but does not provide arbitrary page-number jumps.

### Q34. What is the difference between Oracle ROWNUM and ROW_NUMBER()?

Oracle's `ROWNUM` is a pseudocolumn assigned as rows are selected, before a same-query `ORDER BY` is applied in the classic query shape. Therefore, `WHERE ROWNUM <= 10 ORDER BY salary DESC` does not mean “the ten highest salaries”; it can select ten rows first and then sort them.

`ROW_NUMBER()` is an analytic function that assigns sequence numbers according to an explicit window ordering and can partition by groups. For top-N results, use `ROW_NUMBER()` in a subquery/CTE or Oracle's row-limiting clause. `ROWNUM` remains useful in legacy Oracle SQL, but its evaluation order must be understood. MySQL has `ROW_NUMBER()` in version 8.0+, but does not have Oracle's `ROWNUM` pseudocolumn.

### Q35. What are the differences between NVL and COALESCE?

Oracle's `NVL(expr1, expr2)` returns `expr2` when `expr1` is NULL; otherwise it returns `expr1`. `COALESCE(expr1, expr2, ...)` returns the first non-NULL expression and is supported by both Oracle and MySQL.

```sql
SELECT COALESCE(nickname, preferred_name, 'Unknown') AS display_name
FROM users;
```

`COALESCE` is more portable and handles more than two alternatives. Oracle's type conversion rules for `NVL` and `COALESCE` differ in some cases, so mixed types can produce unexpected conversions or errors. Use explicit casts where appropriate and avoid relying on implicit conversion. Do not assume every function has identical type and evaluation semantics across database engines.

### Q36. How do Oracle sequences differ from MySQL AUTO_INCREMENT?

Oracle sequences are independent database objects that generate numeric values using `NEXTVAL`; they can be used by multiple tables or callers and are not inherently tied to one table. Sequence values can have gaps because values may be consumed by rolled-back transactions, caching, or concurrent activity. A sequence is not a guarantee of gapless business numbering.

MySQL commonly uses `AUTO_INCREMENT` on a table column, with generated values associated with inserts into that table. Allocation and persistence behavior depends on engine and version. Neither mechanism should be used as proof of transaction order or as a gapless invoice number without a separately designed business process. Use generated IDs for identity, and use constraints to enforce uniqueness.

### Q37. How do you inspect execution plans in Oracle versus MySQL?

Both databases have optimizers that choose access paths, join order, join algorithms, and operations such as sorting and aggregation. Read the plan to understand the strategy, not merely to look for an index name.

- **Oracle:** `EXPLAIN PLAN` records the estimated plan; `DBMS_XPLAN` can display plan output. Where permissions and runtime statistics are available, inspect the cursor's actual plan and row counts because the explained plan may differ from the plan used during execution.
- **MySQL:** `EXPLAIN` shows the optimizer's plan. `EXPLAIN ANALYZE` (available in modern MySQL 8.0 versions, introduced in 8.0.18) executes supported statements and reports actual iterator timing and row information.

Look for estimated versus actual row counts, access paths, join order, repeated loops, sorting, and whether the plan matches the query's expected cardinality. Plan syntax, columns, and instrumentation differ by version. Running an ANALYZE form can execute the statement, so use care with modifying queries and production environments.

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
WHERE item_id = :item_id
  AND quantity > 0;
```

Check the affected-row count: one means the reservation succeeded; zero means the item was missing or insufficient stock remained. If the application must distinguish those cases, perform a follow-up read or use a workflow designed for that distinction. Keep the update inside the transaction that includes any other database changes required for the reservation. Use the bind syntax supported by the database client.

Do not read a value into Java, calculate a new value, and overwrite it without protecting against concurrent updates. Use constraints where possible and make retries bounded and safe. Verify that the isolation level and ORM behavior match the chosen approach.

### Q41. The connection pool is exhausted, but the database CPU is not high. What might be wrong?

Pool exhaustion means application threads cannot acquire a connection quickly; it does not necessarily mean the database is CPU-saturated. Connections may be held by long transactions, slow queries waiting on locks or I/O, leaked connections, remote calls performed inside transactions, or an undersized pool relative to legitimate concurrency.

Inspect pool active/idle/pending counts, connection acquisition latency, transaction duration, slow query and lock-wait data, and application traces. Verify connections are always closed or returned to the pool and transaction boundaries are narrow. Increasing the pool can increase database contention and is not a substitute for diagnosing why connections are retained. Add timeouts and alerts so overload fails predictably rather than cascading.

### Q42. A query works in Oracle but behaves differently in MySQL. How do you troubleshoot it?

Do not assume SQL dialects share every detail. Check function availability and return types, implicit conversions, NULL semantics, collation and case sensitivity, date/time types and time zones, pagination syntax, identifier rules, grouping behavior, transaction isolation, and engine-specific optimizer choices.

Reduce the issue to a small reproducible query and compare results as well as plans. Consult documentation for the deployed database versions, and use explicit casts, stable ordering, and unambiguous predicates. Avoid depending on non-standard behavior such as selecting ungrouped columns without a well-defined aggregate rule. Test migrations against representative data, including NULLs, duplicate values, and boundary timestamps.

## References

The question selection reflects recurring themes across multiple interview-preparation resources—joins, filtering and aggregation, NULL behavior, ranking/window functions, practical query tasks, indexes, transactions, and performance troubleshooting. These sources are useful for identifying repeated themes, not for proving that any specific company asks a question or for estimating interview probabilities.

### Interview-topic cross-checks

- [DataCamp — Top SQL Interview Questions and Answers](https://www.datacamp.com/blog/top-sql-interview-questions-and-answers-for-beginners-and-intermediate-practitioners)
- [Coddy — SQL Interview Questions](https://coddy.tech/interview-questions/sql)
- [Weekday — SQL Interview Questions and Answers](https://www.weekday.works/post/sql-interview-questions)
- [buggedbrains — SQL Interview Questions](https://buggedbrains.com/interview/sql-interview)

### Official Oracle documentation

- [Oracle Database SQL Language Reference](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/)
- [Oracle — ROWNUM pseudocolumn](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/ROWNUM-Pseudocolumn.html)
- [Oracle — ROW_NUMBER analytic function](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/ROW_NUMBER.html)
- [Oracle — COALESCE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/COALESCE.html)
- [Oracle Database 21c — Generating and Displaying Execution Plans](https://docs.oracle.com/en/database/oracle/oracle-database/21/tgsql/generating-and-displaying-execution-plans.html)
- [Oracle Database SQL Tuning Guide](https://docs.oracle.com/en/database/oracle/oracle-database/23/tgsql/sql-tuning-guide.pdf)

### Official MySQL documentation

- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/)
- [MySQL — Optimization and Indexes](https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html)
- [MySQL — Multiple-Column Indexes](https://dev.mysql.com/doc/refman/8.4/en/multiple-column-indexes.html)
- [MySQL — EXPLAIN and EXPLAIN ANALYZE](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [MySQL — InnoDB Transaction Model and Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-model.html)
- [MySQL — InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)

---

## Final self-check

Before considering this handbook mastered, make sure you can do each of the following without looking at the answers:

- [ ] Explain `WHERE` versus `HAVING`, NULL logic, and why `NOT EXISTS` can be safer than `NOT IN`.
- [ ] Write a join and explain how one-to-many relationships can inflate aggregates.
- [ ] Solve second-highest salary, top-N-per-group, latest-row-per-customer, and customers-with-no-orders problems.
- [ ] Explain index trade-offs and walk through a measured slow-query investigation.
- [ ] Describe a lost update, a deadlock, and how optimistic locking or an atomic update addresses them.
- [ ] Explain why pagination needs a deterministic order and when keyset pagination is preferable.
- [ ] Name at least two Oracle/MySQL differences that could affect correctness.

## Revision strategy

For each question, practise explaining the core idea aloud, writing the SQL without looking, and then naming one edge case. For performance and concurrency questions, explain the investigation steps and trade-offs before jumping to a fix. Prefer understanding query behavior and data cardinality over memorizing a single syntax pattern. Treat all destructive SQL examples as patterns to adapt: preview affected rows, verify keys and constraints, and test in a safe environment before production use.



For each question, practise explaining the core idea aloud, writing the SQL without looking, and then naming one edge case. For performance and concurrency questions, explain the investigation steps and trade-offs before jumping to a fix. Prefer understanding query behavior and data cardinality over memorizing a single syntax pattern.
