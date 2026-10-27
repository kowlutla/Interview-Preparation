# Apache Spark & Structured Streaming Interview Handbook

**Audience:** Java backend engineers with 5+ years of experience preparing for senior backend interviews.  
**Focus:** Spark fundamentals, Spark SQL/DataFrames, execution internals, partitioning and shuffles, performance, Java APIs, Structured Streaming, Kafka, and production reliability.  
**Version note:** Examples focus on established Java DataFrame and Structured Streaming APIs used in late Spark 3.x and Spark 4.x; they are focused snippets, not a promise that every API/configuration is identical across every release. The official links below currently resolve to Spark 4.2.0 documentation, which lists Java 17/21/25 and Scala 2.13; Spark 4.0 moved Structured Streaming guides under the /streaming/ path and removed Scala 2.12 support. Spark 3.x has a different compatibility matrix. Connector coordinates, JDK requirements, defaults, supported operations, AQE rules, and state behavior vary by release. Validate against the exact Spark, JDK, Scala-binary, Kafka connector, and deployment versions in use.

This handbook starts with big-data basics and builds toward senior-level production reasoning. It assumes limited prior Hadoop knowledge and emphasizes understanding over memorizing definitions.

## Contents

1. [Big-data fundamentals](#1-big-data-fundamentals)
2. [Architecture and execution](#2-architecture-and-execution)
3. [RDDs, DataFrames, and Datasets](#3-rdds-dataframes-and-datasets)
4. [Spark SQL and DataFrame operations](#4-spark-sql-and-dataframe-operations)
5. [Lazy evaluation and execution plans](#5-lazy-evaluation-and-execution-plans)
6. [Partitions, shuffles, and skew](#6-partitions-shuffles-and-skew)
7. [Joins and aggregations](#7-joins-and-aggregations)
8. [Catalyst and query optimization](#8-catalyst-and-query-optimization)
9. [Memory, serialization, and garbage collection](#9-memory-serialization-and-garbage-collection)
10. [Caching and checkpointing](#10-caching-and-checkpointing)
11. [Performance tuning and Spark UI](#11-performance-tuning-and-spark-ui)
12. [Files and storage](#12-files-and-storage)
13. [Structured Streaming](#13-structured-streaming)
14. [Event time, watermarks, windows, and state](#14-event-time-watermarks-windows-and-state)
15. [Kafka integration and delivery guarantees](#15-kafka-integration-and-delivery-guarantees)
16. [Java implementation patterns](#16-java-implementation-patterns)
17. [Production reliability](#17-production-reliability)
18. [Deployment and operations](#18-deployment-and-operations)
19. [Frequently asked questions](#19-frequently-asked-questions)
20. [Production scenario bank](#20-production-scenario-bank)
21. [Mock interviews and revision](#21-mock-interviews-and-revision)
22. [Official references](#22-official-references)

---

## 1. Big-data fundamentals

### Q1. What does big data mean in practical engineering?

Big data is not a universal size threshold. It describes data volume, arrival rate, variety, or processing requirements that exceed what a single machine or a simple database workflow can meet economically and within the required time. A few terabytes may be manageable for an overnight job but difficult under a five-minute SLA. A smaller event stream may still be challenging if it must be enriched continuously and retain state.

A single machine has finite CPU, memory, disk, and network bandwidth. Vertical scaling buys a larger machine but has cost and hardware limits. Horizontal scaling distributes work across machines. This adds network transfer, serialization, coordination, skew, and partial failures, so a distributed system is justified when its parallelism benefits exceed its operational costs.

### Q2. Explain Hadoop, HDFS, and MapReduce.

Hadoop is an ecosystem for distributed storage and processing. HDFS stores large files as blocks across machines, with fault-tolerance mechanisms such as replication. MapReduce is a programming model: map transforms input records into intermediate key/value pairs, and reduce groups values by key and combines them. Traditional MapReduce workflows often write intermediate results to disk between jobs.

Spark is a distributed compute engine that can read HDFS and many other sources. It builds a directed acyclic graph (DAG) of work and can reuse data in memory, reducing overhead for multi-step analytics. Spark does not require HDFS or MapReduce; it can run with object storage, Kafka, Kubernetes, YARN, or its standalone cluster manager.

### Q3. What is Apache Spark, and when is it appropriate?

Spark is a distributed engine for batch processing, SQL/DataFrame workloads, and streaming through Structured Streaming. Developers describe transformations such as filtering events, joining customer data, and aggregating revenue; Spark plans and schedules the work across executors.

It suits large ETL, log analytics, lake processing, feature preparation, and near-real-time aggregations. It is usually unnecessary for a Spring Boot endpoint that reads a few rows from a transactional database. Spark has startup, cluster, shuffle, and operational overhead; use a database query or ordinary Java service when it meets the requirement more simply.

### Q4. How do Spark, Kafka, a database, and a data lake differ?

A relational database manages tables, indexes, transactions, and concurrent operational queries. Kafka is a durable distributed event log, ordered within each partition. A data lake stores files or table formats on distributed/object storage for analytical datasets. Spark is a compute engine that transforms data from those systems.

For example, an order service publishes events to Kafka, Spark enriches and aggregates them, and results are written to a lake for analytics or Elasticsearch for search. A Spark write to an external database does not automatically make all distributed writes one transaction.

### Q5. Batch, streaming, and micro-batch: what is the difference?

Batch processing handles bounded input, such as yesterday's orders. Streaming processes newly arriving data continuously. Structured Streaming commonly uses micro-batches: each trigger processes newly available records and advances query progress. The programming model is streaming, but the default execution is incremental batches rather than one callback per record.

Latency depends on trigger policy, backlog, scheduling, transformation cost, and sink commit time. Spark documents continuous processing separately, with limitations and different semantics; do not assume it supports every Structured Streaming operation.

### Q6. What is data locality?

Data locality means scheduling computation near the storage containing the input to reduce network transfer. With HDFS, a scheduler may prefer a node holding a block. Object storage usually requires network reads, so locality works differently. Locality is a preference, not a guarantee; joins and aggregations may still shuffle data across the network.

---

## 2. Architecture and execution

### Q7. Describe Spark's architecture.

The driver runs the application, builds plans, coordinates execution, and tracks progress. SparkSession is the main entry point for DataFrame/SQL work and uses SparkContext underneath. Executors are processes on worker nodes that run tasks and may cache data. A cluster manager allocates resources; common choices include Spark Standalone, YARN, and Kubernetes.

~~~mermaid
flowchart TB
  App[Java application] --> Driver[Driver: SparkSession and SparkContext]
  Driver --> Scheduler[DAG scheduler and task scheduler]
  Scheduler --> CM[Cluster manager]
  CM --> E1[Executor 1]
  CM --> E2[Executor 2]
  E1 --> T1[Tasks over partitions]
  E2 --> T2[Tasks over partitions]
  E1 <--> Shuffle[Shuffle files and network]
  E2 <--> Shuffle
  E1 --> Data[(Files, Kafka, databases)]
  E2 --> Data
~~~

The driver coordinates; executors process partitions. Collecting an entire dataset to the driver is usually a design error. Executor loss may cause task retries; driver loss usually ends the application unless deployment supervision restarts it and the application can recover.

### Q8. Explain jobs, stages, and tasks.

An action such as count or a write triggers execution and commonly creates a job. Spark divides work into stages at boundaries that require data exchange, especially shuffles. A task is work for one partition in a stage. A stage with 200 partitions commonly has about 200 tasks, subject to the execution plan.

A filter and projection may be pipelined in one stage; a groupBy often shuffles data and introduces a stage boundary. Not every high-level operation maps to exactly one job or stage. Use the UI to understand the actual execution.

### Q9. What happens when Java code calls count() on a DataFrame?

Spark analyzes and optimizes the query, chooses a physical plan, and schedules jobs and stages. Executors read relevant partitions, apply operators, exchange data if required, and return partial counts. Spark combines partial results and returns the count to the driver. The plan depends on source capabilities, filters, partition pruning, statistics, and Spark version.

Count does not necessarily materialize every row, but it can still read a large dataset. If the intent is only to see whether data exists, a bounded action may be cheaper.

### Q10. What do the DAG scheduler and task scheduler do?

The DAG scheduler divides the dependency graph into stages, typically around shuffle dependencies. The task scheduler launches tasks, coordinates placement with available resources, and handles task-level failures. Executors report status and metrics. Retry can recover transient failures; it cannot fix deterministic bad input, a repeatedly failing operation, or driver loss.

### Q11. Narrow versus wide dependencies?

A narrow dependency means a child partition can be computed from a small, fixed set of parent partitions without redistributing the full dataset; filter and many map operations are examples. A wide dependency requires records from multiple partitions to be reorganized, commonly by key; groupBy, many joins, sorting, and repartitioning often cause a shuffle.

Shuffles introduce serialization, network transfer, disk I/O, and stage boundaries. Do not try to eliminate every shuffle: a distributed join or aggregation often needs one. Reduce unnecessary data before it and inspect the plan.

### Q12. What is lazy evaluation?

Transformations such as filter, select, and withColumn describe work without immediately processing all rows. Spark can combine operations, push eligible filters toward a source, prune columns, and choose a plan before execution. An action triggers the required computation.

Lazy does not mean free. Calling show, then count, then write may recompute the upstream pipeline three times unless data is reused through caching or another mechanism. Cache only when repeated computation justifies the storage and population cost.

### Q13. What are client and cluster deployment modes?

In client mode, the driver runs in the environment that submitted the application; this is convenient for development and interactive work, but a laptop or CI worker is rarely a resilient production driver host. In cluster mode, the driver is launched in the cluster-managed environment. Spark Standalone is a simple Spark-native cluster manager, YARN integrates with Hadoop resource management, and Kubernetes schedules the driver/executors as pods with container resource and identity controls. The deployment mode affects where logs live, whether executors can reach the driver, how credentials/dependencies are supplied, and how failed applications are restarted—not the meaning of a DataFrame expression.

---

## 3. RDDs, DataFrames, and Datasets

### Q14. What is an RDD?

An RDD (Resilient Distributed Dataset) is an immutable collection split into partitions so Spark can process different pieces in parallel. Each transformation returns a new RDD rather than modifying the old one. Spark tracks lineage—the dependencies and operations used to produce each partition—so it can often recompute a lost partition instead of storing a full replica of every intermediate result. RDDs are flexible for record-oriented work, but Spark has less knowledge of the business meaning inside arbitrary functions than it has for DataFrame column expressions.

~~~java
JavaRDD<String> lines = sc.textFile("s3a://analytics/logs/*.txt");
JavaRDD<String> errors = lines.filter(line -> line.contains("ERROR"));
long errorCount = errors.count();
~~~

The filter is lazy; count triggers execution. Collecting all matching lines would move them to driver memory and may cause an out-of-memory failure.

### Q15. Compare RDDs, DataFrames, and Datasets.

A DataFrame is a distributed table with named columns and a schema; in Java it is represented as `Dataset<Row>`. A typed Dataset instead exposes JVM domain objects through an `Encoder`. Java can create one from a bean class, for example `spark.createDataset(orderBeans, Encoders.bean(Order.class))`, provided the class meets the bean-encoder requirements. Typed Datasets are especially ergonomic in Scala, while Java teams often prefer `Dataset<Row>` for SQL-style ETL.

DataFrames are usually the best default for structured transformations because Spark can inspect column expressions and optimize them. Typed Datasets add type safety when working with domain objects, but opaque custom functions can reduce the optimizer's visibility into the transformation. Use RDDs when a specialized record-oriented operation genuinely needs the lower-level API; do not switch to RDDs merely because a query is slow.

### Q16. Why can DataFrames outperform hand-written RDD code?

For a DataFrame filter and projection, Spark can reason about the predicate and required columns. It may push filtering into Parquet, read only necessary columns, reorder eligible operations, and generate efficient code. Spark generally cannot inspect the business logic hidden inside an arbitrary RDD lambda or opaque UDF.

DataFrames are not automatically fast: skew, huge joins, and poor data layout remain expensive. Expose intent through built-in SQL expressions and measure the plan.

### Q17. How do you convert between RDDs and DataFrames?

A Java RDD of Row values can be converted using a StructType schema and SparkSession.createDataFrame(rdd, schema). A DataFrame can be converted using javaRDD(), but lower-level processing can lose relational optimization opportunities. Use explicit schemas when types, field names, nullability, or timestamp semantics matter; inference from a sample may not describe all future records.

---

## 4. Spark SQL and DataFrame operations

### Q18. How do you create a SparkSession and read data?

~~~java
SparkSession spark = SparkSession.builder()
    .appName("DailyOrderMetrics")
    .getOrCreate();

Dataset<Row> orders = spark.read()
    .format("parquet")
    .load("s3a://company-lake/orders/");
~~~

In production, the submission configuration normally supplies the cluster manager, resources, and deployment settings, so application code does not need to hard-code `master("local[*]")`. The `SparkSession` is the entry point for DataFrame and SQL work. Parquet files carry schema metadata; CSV and JSON often benefit from explicit schemas because type inference can be expensive and can infer the wrong type from limited or inconsistent input. The path, permissions, and connector must be available from the executors—not merely from the machine where the Java code was written.

### Q19. Show common DataFrame transformations.

~~~java
Dataset<Row> paid = orders
    .filter(col("status").equalTo("PAID"))
    .select("order_id", "customer_id", "amount")
    .withColumn("amount_with_tax", col("amount").multiply(lit(1.18)));

Dataset<Row> totals = paid.groupBy("customer_id")
    .agg(sum("amount").alias("total_amount"),
         count(lit(1)).alias("order_count"));
~~~

These expressions build a plan; they do not immediately loop through every row or mutate `orders`. The filter keeps paid orders, the projection narrows the data to the needed columns, and `withColumn` derives a new value. The aggregation produces one row per customer and calculates both revenue and row count. Keeping the logic in built-in column expressions lets Spark optimize the query and may allow source-level filtering or column pruning. In Java, these examples assume static imports from `org.apache.spark.sql.functions`.

### Q20. Explain select, filter, withColumn, and drop.

`select` controls which columns appear in the result; `filter` keeps only rows satisfying a condition; `withColumn` adds or replaces one named column; `drop` removes columns. Each method returns a new Dataset, so the original variable still refers to its original plan. For example, filtering paid orders before selecting `order_id` and `amount` reduces what later operators need to process. Long chains of `withColumn` are valid, but a single `select` with several expressions can be easier to read and can avoid building an unnecessarily deep plan.

### Q21. How should nulls be handled?

SQL null means unknown/missing, not ordinary equality. Use isNull and isNotNull rather than comparing to null. Null-safe equality is available through Spark SQL's <=> and relevant DataFrame APIs. Count(*) counts rows; count(column) counts non-null values; sum generally ignores null inputs.

Use fill/drop/default operations only when they match domain meaning. Replacing a missing payment amount with zero can create a false financial fact.

### Q22. Explain join types and null behavior.

Inner join returns matching rows. Left/right outer joins preserve unmatched rows from one side and fill the other side's columns with null; full outer preserves unmatched rows from both. Left semi returns left rows that have a match without right columns; left anti returns left rows without a match.

~~~java
Dataset<Row> enriched = orders.join(
    customers,
    orders.col("customer_id").equalTo(customers.col("customer_id")),
    "left");
~~~

Check key uniqueness: if the right side has three rows per customer, an order can become three output rows. Ordinary equality does not match null keys to each other. Correct semantics come before performance tuning.

### Q23. How do window functions differ from groupBy?

`groupBy` reduces each group to aggregate rows, so a customer-level `sum(amount)` returns one result per customer. A window function calculates over related rows while keeping the original rows, which is useful for ranking orders or computing running totals.

~~~java
WindowSpec w = Window.partitionBy("customer_id")
    .orderBy(col("order_ts").asc(), col("order_id").asc());
Dataset<Row> ranked = orders.withColumn("order_rank", row_number().over(w));
~~~

The window definition says to rank each customer's orders by timestamp, using `order_id` as a tie-breaker so the order is deterministic when timestamps match. The physical plan may need to repartition by customer and sort within each partition, which can be expensive for high-cardinality or skewed keys. Choose a window only when row-level results are required; an aggregate is usually cheaper if the output can be collapsed.

### Q24. union versus unionByName?

`union` combines two compatible datasets by column position; if the column order differs, values can land under the wrong names even when both inputs have the same column names. `unionByName` aligns columns by name and, in supported versions, can fill missing columns with null when `allowMissingColumns` is enabled. Neither operation removes duplicates. Normalize data types and decide how missing fields should be represented before combining evolving event schemas; add explicit deduplication only if the business semantics require it.

### Q25. How do you query a DataFrame using SQL?

Registering a temporary view makes the DataFrame queryable by SQL within that SparkSession; it does not copy all rows into a separate database table. The SQL and DataFrame APIs generally feed the same Spark SQL optimizer, so one syntax is not automatically faster than the other.

~~~java
orders.createOrReplaceTempView("orders");
Dataset<Row> result = spark.sql(
    "SELECT customer_id, SUM(amount) AS revenue " +
    "FROM orders WHERE status = 'PAID' GROUP BY customer_id");
~~~

Use SQL when joins and aggregations are easier to review as a query, or the DataFrame API when composing expressions programmatically. Inspect the plan when performance matters; readability and optimizer visibility are more important than the syntax choice.

### Q26. How do you read/write CSV, JSON, and Parquet?

~~~java
Dataset<Row> csv = spark.read()
    .option("header", "true")
    .schema(orderSchema)
    .csv(inputPath);

csv.write().mode(SaveMode.Append)
    .partitionBy("order_date")
    .parquet(outputPath);
~~~

CSV is text, so numeric/date columns need parsing and schema decisions; JSON supports nested objects and arrays but may cost more to parse; Parquet is a typed columnar format that can skip unneeded columns and use file/row-group statistics to reduce reads. Compression is usually effective for analytical files, but the codec and row-group/file layout still affect CPU, I/O, and parallelism. `partitionBy("order_date")` writes a directory layout such as `order_date=2026-10-27/`; it does not promise one file per date, per task, or for the whole dataset. For evolving schemas, validate types rather than relying on inference.

### Q27. How do nested types and explode work?

Spark represents nested data with types such as `struct`, `array`, and `map`. Dot notation can project a nested field—for example, `col("customer.address.city")`—while `explode(col("items"))` creates one output row for each array element. This is useful when an order contains multiple line items, but it can multiply row counts substantially and change aggregation totals if the parent order is then counted once per item. Keep the parent identifier, decide whether null/empty arrays should produce zero or no rows, and explode only when downstream logic needs element-level records.

### Q28. How should dates and timestamps be handled?

Distinguish an instant from local wall-clock time. Standardize time zones and parsing formats, and be explicit about session timezone when correctness depends on it. In streaming, event time controls windows and watermarks; processing time is when Spark handles a record. Incorrect timezone parsing can put records in the wrong day while the query still succeeds.

---

## 5. Lazy evaluation and execution plans

### Q29. Transformations versus actions?

A transformation describes how to derive a dataset; examples include `filter`, `select`, `withColumn`, and `groupBy`. An action asks Spark to produce a result or perform an output operation—such as `count`, `collect`, `show`, or `write`. This distinction explains why defining a long chain of transformations is usually cheap while the final write may run for many minutes: the action triggers the necessary scan, shuffle, and executor work. Some metadata/schema checks can occur earlier, so “lazy” does not mean every API call is free.

### Q30. Why is collect dangerous?

Collect transfers every output row to driver memory. Use it only when the result is known to be small. For large results, aggregate in Spark or write to distributed storage; for inspection use take(n) or a bounded sample. Show displays a limited number of rows by default but still triggers work. Foreach executes on distributed rows/partitions and is not a transactional sink by itself.

### Q31. How do you inspect a query plan?

`df.explain(true)` prints several logical-plan stages and the physical plan; `df.explain("formatted")` gives a more structured view in versions that support it. Look for the scan, pushed and partition filters, `Exchange` nodes that indicate data redistribution, sort operators, join strategy, and aggregate operators. If AQE is enabled, inspect the final adaptive plan after runtime statistics are available. A plan shows what Spark intends to execute, but it cannot tell you whether one executor is unhealthy or a key is skewed; correlate it with task-level metrics in the Spark UI.

### Q32. Why can a transformation pipeline execute multiple times?

DataFrames are lazy descriptions, not automatically materialized results. If code calls `show()` to preview, `count()` to check size, and then `write()`, Spark can execute the scan and transformations separately for each action. That may be correct but needlessly expensive for a large source. Remove actions used only for temporary debugging, or persist the intermediate if it is genuinely reused by several downstream operations. Caching also costs time and memory, so compare its population cost with the recomputation it avoids.

---

## 6. Partitions, shuffles, and skew

### Q33. What is a partition, and how does it relate to parallelism?

A partition is a logical chunk of distributed data that a task processes for an operation. A stage with 100 runnable partitions can expose about 100 tasks of parallel work, but concurrency is limited by available executor task slots; partitions and executors are different quantities. Input-file partitions, shuffle partitions, and output directory partitions are also different concepts.

For DataFrames, `repartition(n, col("customer_id"))` redistributes rows using hash partitioning on the supplied expression(s), which is useful when equal keys need to be brought together. `repartitionByRange(n, col("event_time"))` assigns rows to sampled ordered ranges, which can help range-oriented workloads but has its own sampling and shuffle cost. It does not guarantee that rows are sorted within each output partition; use an explicit sort when row order is required. Because range boundaries are estimated by sampling, exact boundaries can vary. Neither hash nor range partitioning guarantees even work if the key distribution is skewed. Too few partitions limit parallelism; too many tiny partitions add scheduling overhead and may create small files. Choose based on stage metrics, row sizes, and cluster capacity.

### Q34. What causes a shuffle and why is it expensive?

A shuffle redistributes records between partitions, often to bring matching keys together. Joins, groupBy, sorting, and repartitioning commonly trigger one. Shuffle data may be serialized, written to disk, transferred over the network, fetched, and deserialized. Reduce data before the shuffle, choose appropriate join strategies, and address skew rather than trying to eliminate every required exchange.

### Q35. repartition versus coalesce?

Repartition normally shuffles data and can increase or decrease partitions while redistributing records. Coalesce can reduce partitions without a full shuffle, but may create uneven partitions. Use coalesce after filtering when reducing task count is sensible; use repartition when even distribution is needed. Neither guarantees balanced work for a hot key.

### Q36. What does spark.sql.shuffle.partitions control?

It sets the default partition count for many Spark SQL shuffle operations. The current Spark 4.2 configuration reference lists 200 as the default, but verify the deployed version and effective application settings. Too high creates tiny tasks; too low creates large tasks and insufficient parallelism. AQE can coalesce shuffle partitions or alter eligible join decisions at runtime. A Java application can override it for a session with `spark.conf().set("spark.sql.shuffle.partitions", "400")`, but choose the value from shuffle volume and task metrics rather than copying a magic number.

### Q37. What is data skew and how do you diagnose it?

Skew is uneven data distribution. A hot customer ID might represent 40% of events, leaving one task much slower than peers. Compare task durations, input bytes, shuffle read, spill, and key frequencies in the Spark UI. First identify whether the cause is a hot key, one large file, or an unhealthy node.

Mitigations include filtering, pre-aggregation when semantics allow, AQE skew-join handling, or salting. Adding executors will not fix a hot key if work still concentrates in one partition.

### Q38. Explain salting and its risks.

Salting appends an artificial bucket to a hot key so work is distributed across multiple keys. Aggregations commonly need a first aggregation by (key, salt) and a second aggregation by original key. Joins may require replicating the smaller side across salt values. This adds work and can multiply rows; incorrect replication or merging produces wrong results. Use it only after measuring skew and designing the merge correctly.

### Q39. File partitioning versus Spark partitions?

Storage partitioning and Spark execution partitioning solve different problems. A layout such as `year=2026/month=10/` stores files under partition-column directories and can help Spark skip whole directories when a filter matches those columns. An execution partition is a chunk processed by tasks; one storage directory may contain many files, and Spark can split large files or pack small files into input partitions. Calling `repartition` changes the execution distribution for the resulting plan—it does not rewrite the original source directory layout.

### Q40. What is partition pruning?

Partition pruning skips whole storage partitions when Spark can prove their partition-column values cannot match the predicate—for example, reading one `order_date` from a dataset partitioned by date. Predicate pushdown is related but different: it tries to apply supported filters inside the source or file reader. A query can prune directory partitions and still read many row groups within the selected files. Expressions or casts on the partition column can sometimes make pruning harder, so verify `PartitionFilters`, `PushedFilters`, file counts, and input bytes in the plan/UI rather than assuming it occurred.

---

## 7. Joins and aggregations

### Q41. Which join strategies should you know?

Broadcast hash join distributes a sufficiently small relation to executors, avoiding a shuffle of the large side. Sort-merge join commonly shuffles and sorts both sides by key and is common for large equi-joins. Shuffled hash join redistributes data by key and builds a hash table for one side of each partition when the planner considers it suitable.

Spark's choice depends on statistics, join type, hints, configuration, and AQE. Inspect the physical and final adaptive plans rather than memorizing a simplistic rule.

### Q42. When is broadcast useful, and what can go wrong?

Broadcast a small dimension table when the materialized relation can safely fit on executors. It can remove a large shuffle. However, the in-memory broadcast size is not the compressed source file size. A too-large broadcast can consume memory, cause GC, or fail during exchange. Check actual size and executor pressure; a hint is not a substitute for evidence.

### Q43. What is Adaptive Query Execution?

Adaptive Query Execution (AQE) lets Spark revise eligible parts of a physical query plan after it has measured runtime statistics, rather than committing to every decision from estimates alone. Common capabilities include coalescing small post-shuffle partitions, splitting qualifying skewed join partitions, and converting a planned sort-merge join to a broadcast hash join when runtime sizes justify it. The current Spark 4.2 documentation enables AQE by default through `spark.sql.adaptive.enabled`; verify the effective configuration and supported rules for the version actually deployed.

A useful workflow is to compare the initial and final adaptive plans and correlate the change with stage metrics. AQE can make a better decision when estimates were wrong, but it cannot repair a missing join predicate, make a huge result set small, or automatically fix every skewed aggregation.

### Q44. Why can distinct be expensive?

`distinct()` deduplicates complete rows, so Spark must compare records across partitions; that often means a shuffle and can move wide rows unnecessarily. If the business rule is “one record per `event_id`,” use `dropDuplicates("event_id")` to state that rule explicitly, while deciding what should happen if two records share the ID but contain different payloads. This is still distributed work. In streaming, Spark must remember previously seen keys across triggers; without a bounded duplicate horizon/watermark, state can grow with the history of unique IDs.

### Q45. Why can an aggregation run out of memory?

High-cardinality grouping creates many distinct keys and intermediate state. Skew can make one task much larger than peers. Spark often performs partial aggregation before shuffle, but that cannot eliminate the cost of tracking many unique groups. Inspect cardinality, spill, and skew; reduce rows earlier or change the aggregation grain only if business semantics allow it.

---

## 8. Catalyst and query optimization

### Q46. Explain Spark SQL's planning pipeline.

Spark SQL moves through several planning steps. It first parses the SQL or builds a logical plan from DataFrame expressions. Analysis resolves column names, data types, functions, and table references; unresolved or invalid expressions fail before execution. Catalyst then applies eligible logical optimizations—such as constant folding, predicate and projection pushdown, and some join reordering—and the planner selects a physical plan with concrete scans, exchanges, joins, sorts, and aggregates. Adaptive Query Execution (AQE) can revise eligible parts of the physical plan after runtime statistics become available. Not every rule applies to every query or data source, so inspect `explain("formatted")` and the final adaptive plan rather than assuming an optimization occurred.

### Q47. Predicate pushdown versus projection pruning?

Predicate pushdown tries to filter close to the source; projection pruning avoids reading unused columns. Parquet can benefit because Spark may read only required column chunks and skip data using metadata. Not all predicates can be pushed down, and a UDF can block optimization. Verify the plan and input metrics.

### Q48. What are Tungsten and whole-stage code generation?

Tungsten is the name historically used for Spark's work on more efficient execution, including memory use and binary representation; it is not a single feature that can be toggled to make a query faster. Whole-stage code generation can fuse eligible operators into generated JVM code and reduce per-row virtual-call overhead. Spark SQL also uses columnar processing in supported scans and operators, especially with columnar file formats such as Parquet. The exact execution path depends on the operators, source, and version: a UDF, unsupported expression, or transition between row and columnar representations can limit these optimizations. Use the physical plan and runtime metrics to judge the effect.

### Q49. What can the optimizer not fix?

It cannot infer missing business semantics, eliminate unavoidable output volume, guarantee balanced keys, or make a slow external sink fast. It may not optimize opaque UDF logic or repair an incorrect join. Expose intent through built-in expressions, design sensible data layouts, and validate actual execution.

---

## 9. Memory, serialization, and garbage collection

### Q50. Driver memory versus executor memory?

Driver memory supports planning, scheduling metadata, task results, and objects returned to the application. Executor memory supports task execution, shuffle, cached blocks, and broadcasts. Container memory can include off-heap and native use beyond JVM heap. A driver OOM from collect is not fixed by increasing executor memory; a skewed join OOM needs task-size and join-strategy investigation.

### Q51. Execution memory versus storage memory?

Execution memory is used by work such as joins, sorts, and aggregations; storage memory holds cached/persisted blocks. Under Spark's unified memory manager, the execution/storage region can be shared, and cached blocks may be evicted when execution needs room. This is why caching a large intermediate can make a later join slower rather than faster: it competes with the memory the join needs and can increase spill or recomputation. Cache expensive data only when it will be reused enough to repay materialization and memory costs, and unpersist it when the reuse period ends.

### Q52. Why does Spark spill to disk?

When an operation such as a sort, join, or aggregation cannot keep all of its working data in memory, Spark may spill intermediate structures to local disk. That can let the task finish instead of failing, but spill adds read/write I/O and can dominate runtime when it happens heavily. Check the stage's memory/disk spill metrics alongside partition sizes, key cardinality, skew, and GC. Small amounts of spill are not automatically a bug; repeated large spill is a signal to investigate the workload and memory distribution before simply increasing heap.

### Q53. Java serialization versus Kryo?

Java serialization is convenient but can be verbose and CPU-intensive for object-heavy RDD workloads. Kryo often produces smaller, faster serialized representations when configured and tested correctly, though custom classes and registrations can matter. The benefit is most visible when Spark repeatedly serializes application objects for shuffles or network transfer. Many DataFrame SQL operators use Spark's internal row/columnar representations, so changing `spark.serializer` is not a universal fix for a slow SQL query. Measure serialization time, shuffle size, and task metrics before changing it.

### Q54. How do you diagnose excessive garbage collection?

Start with executor metrics and logs: is GC taking a large fraction of task time, and is the problem isolated to one stage or spread across the application? Correlate GC with heap occupancy, spill, broadcast size, cached blocks, and task-size skew. Object-heavy RDD code, repeated conversions to Java collections, and large broadcast structures can create allocation pressure. Prefer built-in DataFrame expressions where they express the same logic, reduce unnecessary rows/columns, remove unused caches, and fix skew before adjusting heap settings. A larger heap may postpone the next collection but can also make individual pauses longer.

### Q55. What is executor memory overhead?

In a containerized deployment, executor memory overhead accounts for process memory outside the configured JVM heap, including native memory, direct buffers, and other runtime overhead. The scheduler/container limit applies to the whole process footprint, not just Java heap usage. As a result, a pod can be OOM-killed without a `Java heap space` error in the executor log. Check both JVM logs and the cluster manager's exit reason/events before deciding whether to tune `spark.executor.memory`, `spark.executor.memoryOverhead`, task size, or native-memory usage.

---

## 10. Caching and checkpointing

### Q56. cache versus persist?

Cache requests the default persistence level for that abstraction/version; persist lets you select a storage level. For DataFrames, modern Spark SQL versions commonly use memory-and-disk persistence by default, but verify the deployed API docs. Cache is populated when an action requires the data.

~~~java
Dataset<Row> cleaned = raw.filter(col("event_id").isNotNull())
    .persist(StorageLevel.MEMORY_AND_DISK());
cleaned.count(); // materializes cache
// Reuse for multiple actions.
cleaned.unpersist();
~~~

Cache only if reuse offsets population and storage cost. Unpersist when no longer needed.

### Q57. Checkpoint versus cache?

Cache stores computed partitions for reuse; lost partitions can generally be recomputed from lineage. Checkpoint materializes data to a checkpoint location and truncates lineage, useful for long dependency chains and stateful streaming recovery. Reliable checkpoints require durable storage. Local checkpointing uses executor-local storage and has weaker fault tolerance.

Structured Streaming checkpoints track progress and state; they are not equivalent to caching. Protect them and do not casually reuse a checkpoint path for a different logical query.

### Q58. What is lineage?

Lineage records the dependency graph Spark uses to produce an RDD partition or a derived dataset. If a cached partition is lost, Spark may be able to recompute it by replaying the required transformations from its parents. This is a key recovery mechanism, but recomputing a long pipeline can be expensive, and recovery is impossible if the needed source data is no longer available. Caching can speed up repeated access; reliable checkpointing materializes a recovery point and truncates lineage. Those mechanisms solve related but different problems.

---

## 11. Performance tuning and Spark UI

### Q59. How should you investigate a slow job?

Identify the slow job/stage, compare task durations, inspect input bytes, shuffle, spill, GC, executor time, and query plan, then validate key distribution and file layout. Change one likely cause and compare equivalent inputs. A job may be slow due to skew, full scans, tiny files, shuffle, resource contention, or a slow sink; each needs a different remedy.

### Q60. What does the Spark UI tell you?

The Spark UI is where you turn a vague complaint like “the job is slow” into a specific bottleneck. Jobs and Stages show task counts, duration distributions, shuffle read/write, spill, and failures; Executors show task activity, cached storage, GC time, and executor loss; the SQL page connects query plans to runtime metrics when available. Compare median, tail, and maximum task times. A completed application's UI may disappear unless event logging and a history server are configured, so enable those for production troubleshooting.

### Q61. Why are small files harmful?

Thousands of tiny files impose costs before Spark has processed much data: listing and metadata work, opening each file, building input partitions, and scheduling many small tasks. This can make a query spend more time on overhead than on scanning records. A distributed compaction/rewrite can combine files, but the target size depends on query parallelism, data volume, codec, and storage system. Do not solve the problem by writing a huge dataset through `coalesce(1)`; that replaces many small tasks with one serial bottleneck and can create a very large file.

### Q62. What is dynamic allocation?

Dynamic allocation lets Spark request additional executors when work is queued and release idle executors, subject to the cluster manager and configuration. It can improve utilization for workloads whose demand changes over time, but scaling is not instantaneous and executor removal interacts with shuffle-data preservation. AQE is different: it adjusts eligible query-plan decisions using runtime statistics, while dynamic allocation changes available compute resources. Neither fixes a single hot partition, an incorrect query, or a sink that can accept only a limited write rate.

### Q63. How should tuning experiments be performed?

Choose a representative dataset and record a baseline before changing a setting: wall-clock runtime, stage duration, input bytes, shuffle, spill, GC, task variance, and output correctness. Change one factor at a time so you can attribute the result. Small samples often hide the very problems that dominate production—skew, large join cardinality, state growth, and sink saturation—so verify at realistic scale where feasible. Compare end-to-end time and resource cost, not only one stage. Keep a rollback path for production settings and confirm that totals and row counts remain correct.

---

## 12. Files and storage

### Q64. Why is Parquet common for analytics?

Parquet stores values by column rather than as a whole row, so a query that needs `customer_id` and `amount` can often avoid reading unrelated columns. Column statistics and row-group metadata can help skip data when predicates are selective, while column-friendly compression can reduce storage and I/O. The trade-off is that parsing/writing and row-oriented access patterns may differ from text formats. JSON is handy for raw or nested events; CSV is widely interoperable but has weak type information; ORC is another analytical columnar format. Parquet helps most when schemas, partition layout, and file sizes are designed for the query workload.

### Q65. Why does a Spark write create a directory of part files?

Each writing task normally produces its own output file for the data it handled, so a Spark dataset is commonly represented as a directory of `part-...` files rather than one monolithic file. The final file count depends on execution partitions, dynamic partition values, writer behavior, and the output committer—not simply on how many executors exist. Downstream Spark readers treat the directory as a dataset. If a consumer needs a single export file, create it as a deliberate bounded export step after confirming the data is small enough; do not funnel a large production dataset through one task.

### Q66. What is schema evolution?

Schema evolution means a dataset's contract changes over time—for example, a new optional field is added, a field is renamed, or a number becomes a string. An additive optional field is often easier to support than a type change or rename, because old files may not contain the new field and different files may carry different schema metadata. Directory reads do not guarantee automatic reconciliation of every incompatible file. Define compatibility rules, normalize inputs to a canonical schema, validate casts/nulls, and test queries against both historical and new partitions.

### Q67. What should you know about object storage?

Object storage scales well for durable lake data, but its request, listing, and commit behavior differs from a local filesystem or HDFS. Many small objects increase listing/open overhead, and a rename is not universally an atomic filesystem operation on object stores. A failed Spark write can leave temporary or partial objects depending on the connector and committer. Use the deployment's supported filesystem connector and output committer, and test failed writes and retries. Plain Parquet files do not, by themselves, provide the same table-level transaction protocol as a transactional lakehouse table format; choose a table format when atomic table commits, concurrent writers, or schema evolution are requirements.

---

## 13. Structured Streaming

### Q68. What is Structured Streaming?

Structured Streaming lets you describe a continuous computation with familiar DataFrame/Dataset operations, while Spark manages source progress and incremental execution. Conceptually, an unbounded stream is a table that keeps receiving rows; the query defines how the result should be updated as new input arrives. By default, Spark runs it as micro-batches and stores the progress—and state for stateful operators—in a checkpoint. This model simplifies development, but streaming queries have constraints around output modes, state cleanup, late data, and sink behavior that do not apply in exactly the same way to bounded batch queries.

### Q69. Describe a streaming query lifecycle.

~~~mermaid
flowchart LR
  Kafka[Kafka source] --> Read[Streaming DataFrame]
  Read --> Transform[Parse and enrich]
  Transform --> State[Window and state]
  State --> Sink[Kafka, files, database]
  Read -. offsets .-> CP[(Checkpoint: progress and state)]
  State -. recovery state .-> CP
~~~

Build a streaming query with `readStream`, apply supported transformations, and call `writeStream.start()` to obtain a `StreamingQuery` handle. On each trigger, Spark determines the source offsets for the batch, executes the incremental plan, updates state when needed, writes output, and records query progress through the checkpoint protocol. A restart using the same compatible checkpoint can recover the recorded progress and state. The order and guarantees at the sink boundary depend on the source/sink implementation; a checkpoint cannot make arbitrary external side effects atomic or prevent duplicates in a non-idempotent sink.

### Q70. What are sources and sinks?

A source is the system Spark reads incrementally, such as Kafka or a directory of newly arriving files. A sink is where the query writes its result, such as a file sink or Kafka topic. Connector packages and options must match the Spark version, and the sink's fault-tolerance contract matters as much as the query's checkpoint. `foreachBatch` is a useful escape hatch for a custom batch writer, but Spark cannot infer whether arbitrary code is transactional or idempotent; the application must design safe retries and partial-failure recovery.

### Q71. Explain append, update, and complete output modes.

Append emits rows that are new/final under the query's semantics. For a windowed event-time aggregation, that often means Spark waits until the watermark passes the window before emitting its final result. Update emits rows changed since the previous trigger, so a sink must handle repeated updates to the same key/window. Complete emits the entire result table after each trigger; because it promises to preserve all aggregate rows, Spark cannot use watermarking to discard old aggregation state in Complete mode. This can make state and output grow over time. The supported modes depend on both the query operation and sink, so check the compatibility matrix before choosing a mode.

### Q72. What does a trigger control?

A processing-time trigger requests micro-batches on a fixed interval. If a batch finishes early, Spark waits for the next interval; if it runs long, the next batch cannot begin until the current one finishes. `Trigger.AvailableNow()` is useful for scheduled catch-up jobs: for sources that support it, Spark processes all data available at the start of that query execution, possibly across multiple batches, and then terminates. It also processes pending batches from a previous run and may run a final no-data batch to advance watermarks. A source that does not support AvailableNow can cause fallback behavior, so check connector support. The default trigger starts another micro-batch as soon as the previous one finishes and data is available.

~~~java
import org.apache.spark.sql.streaming.Trigger;

// Option A: long-running query with periodic micro-batches.
StreamingQuery query = stream.writeStream()
    .trigger(Trigger.ProcessingTime("10 seconds"))
    .option("checkpointLocation", checkpointPath)
    .start();

// Alternative for a scheduled catch-up run: use this instead of Option A,
// not concurrently with it. The source must support AvailableNow.
// StreamingQuery catchUp = stream.writeStream()
//     .trigger(Trigger.AvailableNow())
//     .option("checkpointLocation", checkpointPath)
//     .start();
~~~

A trigger interval is not a batch-completion SLA. If processing takes longer than the interval, batches cannot finish faster than the work allows; a backlog shrinks only when sustainable processing throughput exceeds the incoming rate. Continuous processing is a separate experimental mode intended for lower latency, with a restricted set of supported operations and at-least-once semantics in the documented model. It does not support `foreachBatch`, because that API operates on micro-batches. Use the default micro-batch model unless the continuous-mode latency/semantics trade-off and query compatibility are explicitly justified.

### Q73. What is a checkpoint directory?

A checkpoint directory stores the query's progress log and, for stateful queries, the state needed to continue from a prior micro-batch. For Kafka, Spark records the offsets it has processed in the checkpoint rather than relying only on a conventional consumer-group commit. The directory must be durable, writable/readable by the runtime identity, and kept with the query it belongs to. Do not reuse it for a logically different query or delete it to “unstick” a stream: you may lose state or start a new replay history. Checkpoint compatibility must be assessed when the query changes.

### Q74. Query ID versus run ID?

`StreamingQuery.id()` identifies the query across restarts from checkpoint data, while `runId()` identifies a particular execution and changes when the query is started again. This distinction helps correlate logs and progress across a restart: the query may be the same logical stream even though the run has changed. Use batch IDs and source offsets to investigate processing progress, and a separate stable event/business key for idempotent output; query/run IDs do not deduplicate business records.

### Q75. Does Structured Streaming guarantee exactly-once processing?

It depends on source replayability, checkpointing, the operation, and sink commit behavior. Structured Streaming can recover query progress/state for supported sources and operations, but sink guarantees are connector-specific. The official Spark Kafka integration guide explicitly documents Kafka writes from Spark as at-least-once: a retried write can duplicate a record even when the broker had accepted the earlier attempt. A custom callback or arbitrary database write can also repeat side effects after failure. Separate input replay, state recovery, sink commit, and business effect. Use stable event IDs, upserts, unique constraints, or a transactional/idempotent sink protocol where needed.

### Q76. Event time versus processing time?

Event time is when an event happened; processing time is when Spark handles it. A mobile event created at 10:00 and delivered at 10:08 has event time 10:00 but later processing time. Event-time windows fit business analytics with delayed arrival; processing-time windows answer a different operational question. Validate clocks, time zones, and timestamp parsing.

---

## 14. Event time, watermarks, windows, and state

### Q77. Tumbling versus sliding windows?

A tumbling window has a fixed duration and does not overlap, such as 10:00–10:05 and 10:05–10:10. A sliding window can overlap; for example, a 10-minute window sliding every 5 minutes can count one event in two windows, increasing work and state. Spark also supports session windows: instead of a fixed window end, a session grows while events for a key arrive within the configured inactivity gap. Session-window behavior and supported combinations should be checked for the deployed Spark version.

~~~java
Dataset<Row> perWindow = events
    .withWatermark("event_time", "10 minutes")
    .groupBy(window(col("event_time"), "5 minutes"), col("customer_id"))
    .agg(sum("amount").alias("total_amount"));

// Session windows: sessions for a customer extend when activity continues
// within the gap duration. The session_window function is available since Spark 3.2.
Dataset<Row> sessions = events
    .withWatermark("event_time", "10 minutes")
    .groupBy(expr("session_window(event_time, '5 minutes')"), col("customer_id"))
    .agg(count(lit(1)).alias("event_count"));
~~~

The watermark communicates an event-time lateness threshold; it does not mean that each event gets a ten-minute wall-clock timer. In current Structured Streaming semantics, session-window aggregation does not support Update output mode, and the grouping key must include at least one column in addition to the session window. Check the session-window restrictions for your Spark version.

### Q78. What does a watermark do?

A watermark is derived from the maximum event time observed by the query minus the configured delay threshold. It lets supported stateful operations decide when older windows/state can be finalized or cleaned. It does not advance just because wall-clock time passes; the source must make event-time progress. For aggregation cleanup, the watermark must be applied before the aggregation to the same event-time column (or a window derived from it), and output mode must support cleanup (Append or Update, not Complete). Without effective state cleanup, a long-running query can retain state indefinitely.

For supported windowed aggregations, Spark documents a one-sided guarantee: records less than the configured delay behind the latest event time processed will not be dropped as too late. Records delayed beyond that threshold may or may not be processed; the watermark is not a promise that every older record is immediately discarded. Longer thresholds usually retain state longer and tolerate more event-time lateness, at additional memory/checkpoint cost. Multiple streams and stateful operator types introduce further semantics, so validate the exact query shape against the deployed version.

### Q79. What happens to records later than the watermark?

For supported window aggregations, an event that is less than the configured watermark delay behind the latest event time processed is guaranteed not to be dropped as late. That is an event-time relationship, not simply the elapsed wall-clock time since the event happened. Records beyond the threshold are not guaranteed to be dropped: Spark may still aggregate them if the relevant state has not been cleaned. The watermark is therefore neither a universal pre-filter nor a promise that every very late event is ignored. Choose the delay from business requirements and provide correction/reconciliation if very late updates matter.

### Q80. How do stateful aggregation and deduplication work?

A stateful aggregation keeps intermediate values keyed by grouping columns and, for event-time aggregates, by window. On a new micro-batch, Spark updates that state rather than recomputing the aggregate from all historical input. Streaming deduplication similarly remembers keys that have already appeared. State is checkpointed for recovery and consumes memory, disk, and I/O as cardinality grows. A watermark can bound state only where the operator and query shape support cleanup; without a realistic duplicate horizon, deduplication may need to retain every seen ID. Use stable event IDs and choose the retention policy from business semantics.

### Q81. Stream-static versus stream-stream joins?

A stream-static join enriches a streaming row using a bounded DataFrame and is not stateful, so it does not retain stream-side join state for future matches. However, do not assume it is a point-in-time slowly changing dimension lookup: freshness and snapshot behavior depend on how the static source is read and planned. If business correctness requires matching an event to the dimension version valid at that event's time, model effective-from/effective-to or version information explicitly.

A stream-stream join must buffer rows from both unbounded inputs because a future event may match an earlier row. To bound state, add watermarks on the event-time columns and an event-time constraint that tells Spark when old rows cannot match future records. For example, an impression can be joined to a click only when the click follows the impression and arrives within a defined hour-long window. Outer joins have stricter requirements because Spark must know when it is safe to emit a no-match result.

~~~java
Dataset<Row> impressions = impressionStream
    .withWatermark("impression_time", "2 hours");
Dataset<Row> clicks = clickStream
    .withWatermark("click_time", "3 hours");
Dataset<Row> matched = impressions.join(clicks,
    expr("impression_id = click_impression_id AND " +
         "click_time >= impression_time AND " +
         "click_time <= impression_time + INTERVAL 1 HOUR"),
    "inner");
~~~

This is a pattern, not a complete application; valid join types and state cleanup rules depend on the chosen query and Spark release.

### Q82. What is the state store?

The state store is the mechanism Spark uses to read and update an operator's keyed state between micro-batches. It enables incremental aggregations, deduplication, and stream-stream joins, but the size and update cost depend on key cardinality, window/lateness policy, and how much state changes in each batch. A growing state store can increase batch duration and checkpoint cost even if the input rate is unchanged. Track state rows/bytes, update and commit time, watermark progress, and recovery duration; do not assume more executor heap can fix state that the query never evicts.

### Q83. Why might a watermark not reduce state?

Check whether timestamps are valid and advancing, whether the watermark is attached to the relevant input and operation, whether null timestamps exist, and whether a multi-stream watermark policy is holding progress back. Confirm that the query supports the intended cleanup. Do not shorten the delay until the late-data correctness trade-off is understood.

---

## 15. Kafka integration and delivery guarantees

### Q84. How do you read Kafka in Structured Streaming?

~~~java
Dataset<Row> kafka = spark.readStream()
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker1:9092,broker2:9092")
    .option("subscribe", "orders")
    .option("startingOffsets", "latest")
    .load();

Dataset<Row> parsed = kafka.selectExpr(
    "CAST(key AS STRING) AS message_key",
    "CAST(value AS STRING) AS json_value",
    "timestamp AS kafka_timestamp", "topic", "partition", "offset");
~~~

Include the Kafka connector artifact compatible with the Spark and Scala binary versions. Kafka `key` and `value` arrive as binary columns, so cast to string only when the payload encoding is text. Parse JSON with an explicit schema rather than assuming every record is valid or has the same shape. For example, `from_json(col("value").cast("string"), eventSchema).alias("event")` lets you select nested fields such as `col("event.order_id")`; rows where parsing returns null can be routed to quarantine with topic/partition/offset metadata. Set `.option("includeHeaders", "true")` when the query needs Kafka headers; otherwise the headers field is not included by default. `startingOffsets` initializes a new query; an existing checkpoint's recorded progress governs restart.

### Q85. How do you write to Kafka?

~~~java
Dataset<Row> output = parsed.selectExpr(
    "CAST(order_id AS STRING) AS key",
    "to_json(struct(order_id, status, event_time)) AS value");

StreamingQuery query = output.writeStream()
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker1:9092")
    .option("topic", "order-enriched")
    .option("checkpointLocation", "s3a://company-checkpoints/order-enriched")
    .outputMode("append")
    .start();
~~~

Confirm output-mode compatibility and connector documentation for your Spark version. Important guarantee: the official Spark Kafka integration documents Kafka sink writes as at-least-once, so a retry can create duplicate records even if the broker accepted an earlier attempt. Checkpointing supports recovery of query progress; it does not turn the Kafka sink into an exactly-once business-effect protocol. Use a stable event key and make downstream consumers idempotent or deduplicate where required.

### Q86. Are Spark tasks the same as Kafka consumer-group members?

Not exactly. By default, each Kafka TopicPartition maps to a Spark input partition for a micro-batch, so Kafka partition count is an important parallelism limit. The connector also provides a minPartitions option that can split large Kafka offset ranges into more Spark input partitions; treat it as a desired minimum, not a promise of an exact task count. Downstream shuffles may have a different task count. Spark tracks source offsets in its checkpointed query progress rather than relying on an ordinary consumer group's committed offsets as the sole restart position.

### Q87. How do offsets and restart work?

Spark's Kafka source records offset progress in the Structured Streaming checkpoint; Kafka consumer-group commits are not the sole restart authority. `startingOffsets` applies when a new query starts without prior checkpoint progress and does not override offsets already recorded in an existing checkpoint. This is why deleting a checkpoint is not a routine restart technique: it creates a new query history and can replay data or lose prior state. Also check Kafka retention. If the consumer falls behind until required offsets have expired, Spark cannot recreate the missing records from the checkpoint. The `failOnDataLoss` option can make the query fail when Spark detects that offsets may no longer be available; investigate before disabling it. Even correct replay of source offsets can repeat external side effects unless the sink is idempotent.

### Q88. How do you make database writes idempotent?

Use stable event identifiers and upserts/unique constraints so a retry converges to the same final state. With foreachBatch, batchId helps identify a batch only if the sink safely records and commits it; it cannot prevent duplicates if a write partially succeeds before the marker. Use transactions where possible, deterministic keys, and version checks for out-of-order updates.

### Q89. What is backpressure and how do you handle Kafka lag?

Lag grows when sustained arrival rate exceeds sustainable processing throughput. Compare input and processing rates, batch duration, per-partition traffic, shuffle/spill, state-store time, and sink latency. The Kafka connector supports `maxOffsetsPerTrigger` to cap how many offsets enter a micro-batch; this can protect a fragile sink during recovery, but it does not increase the pipeline's long-term capacity. The connector's `minPartitions` option can split large Kafka offset ranges into more Spark tasks when default source parallelism is too low, but it's a hint and should be benchmarked. Scale or optimize the component that is actually limiting throughput. If the sink is slow, more executors can increase pressure rather than throughput.

### Q90. How do you handle malformed messages and schema evolution?

Treat payloads as untrusted input. Parse with an explicit schema, preserve event ID and source metadata, and route malformed records to quarantine with a reason. Fail fast when continuing would corrupt critical results. Define schema compatibility and coordinate producer/consumer changes; a new optional field differs from changing an existing field's type. Alert on malformed-record rates.

---

## 16. Java implementation patterns

### Q91. What does a basic Java batch application look like?

~~~java
import static org.apache.spark.sql.functions.*;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SaveMode;
import org.apache.spark.sql.SparkSession;

public final class DailyOrderMetrics {
  public static void main(String[] args) {
    SparkSession spark = SparkSession.builder()
        .appName("DailyOrderMetrics").getOrCreate();
    try {
      Dataset<Row> orders = spark.read().parquet(args[0]);
      Dataset<Row> metrics = orders
          .filter(col("status").equalTo("PAID"))
          .groupBy(to_date(col("order_ts")).alias("order_date"))
          .agg(sum("amount").alias("revenue"));
      metrics.write().mode(SaveMode.Overwrite).parquet(args[1]);
    } finally {
      spark.stop();
    }
  }
}
~~~

This is a teaching skeleton. Validate schema, timezone, output overwrite scope, and commit behavior before production use. Never overwrite a shared path without understanding the impact.

### Q92. How do you shut down a streaming query gracefully?

Keep the StreamingQuery returned by start(), wait with awaitTermination(), log failure details, and stop the query through a controlled shutdown path. The deployment supervisor should have an explicit restart policy. A process that remains alive after its query terminates is not healthy.

~~~java
try {
  query.awaitTermination();
} catch (StreamingQueryException e) {
  throw new RuntimeException("Streaming query failed", e);
} finally {
  spark.stop();
}
~~~

Use the appropriate output mode and add shutdown hooks and query-health metrics according to your runtime.

### Q93. How do you use foreachBatch safely?

`foreachBatch` supplies a bounded DataFrame and a unique micro-batch ID (`batchId`) to custom write logic. The callback is retried as part of streaming recovery, so an arbitrary append is not automatically exactly once. Spark's guide notes that the batch ID can support exactly-once output only when the sink uses it to deduplicate safely. For a database, a robust design might write records and a processed-batch marker in the same database transaction, with a uniqueness constraint on the batch ID; if the marker and records are committed separately, a partial failure can still duplicate output. A stable event ID/upsert is useful when deduplication must work at record level.

~~~java
stream.writeStream()
    .foreachBatch((batch, batchId) -> {
      // Application-specific: atomically write/upsert the batch and record batchId,
      // or make every record write idempotent. Plain append can duplicate rows.
      writeBatchIdempotently(batch, batchId);
    })
    .option("checkpointLocation", checkpointPath)
    .start();
~~~

`writeBatchIdempotently` is a placeholder for your sink protocol, not a Spark method. If you perform multiple actions or write the same micro-batch to multiple sinks, persist the batch DataFrame during the callback and unpersist it afterward to avoid recomputing a stateful micro-batch multiple times. Keep credentials outside source code.

### Q94. What should observability and exception handling include?

Log the application ID, query ID/run ID, batch context, and failure cause so an incident can be traced across driver, executor, and sink logs. Export input and processing rates, batch duration, sink latency, lag, state growth, malformed-record counts, and output freshness. Avoid putting credentials or sensitive payloads into logs. A process can remain alive after its streaming query has failed, so health checks should verify the query is active and making progress rather than relying on process liveness alone.

---

## 17. Production reliability

### Q95. What happens if an executor is lost?

When an executor disappears, its running tasks fail and Spark may retry them on surviving executors. Cached partitions held only there can be recomputed from lineage. Shuffle output is more subtle: if the required shuffle files are no longer available, downstream tasks can fail with fetch errors and Spark may need to rerun upstream work. Repeated loss is not normal noise to hide with retries. Correlate the executor exit reason with GC, memory/container events, disk capacity, network errors, and the stage/task that was running.

### Q96. What happens if the driver fails?

The driver owns the application's scheduling and query coordination, so losing it generally ends that application. Whether a new driver is launched is a deployment/orchestration decision; Spark does not guarantee every environment automatically restarts the process. A restarted Structured Streaming application can recover recorded offsets and state when it reuses the correct compatible checkpoint, but the sink may still have duplicate or partial effects depending on its commit protocol. Batch jobs commonly need resubmission, and output paths/transactions must be designed so resubmitting does not corrupt or duplicate results.

### Q97. How do you investigate shuffle fetch failures?

A shuffle fetch failure means a downstream task asked for map-side shuffle data and could not retrieve it. Common causes include an executor/node disappearing after producing shuffle files, local disk failure/full disks, network timeouts, corruption, or resource pressure. Use the exception details and executor/worker logs to see whether failures concentrate on one host or block across the cluster. Retries can recover a transient network blip; if shuffle files keep disappearing or a node is unhealthy, fix the infrastructure or ensure shuffle data can be preserved before increasing retry counts.

### Q98. What if a streaming checkpoint is unavailable or corrupted?

Preserve logs and investigate storage connectivity, permissions, deletion, and compatibility before changing anything. Do not point to an empty checkpoint and assume it will resume correctly; it may start a new query and replay input or lose prior state. Restore a valid checkpoint if possible. Otherwise plan a controlled replay from a known boundary with idempotent output and state reconstruction.

### Q99. How should corrupt input be handled?

Agree on what the pipeline does with malformed input before deploying it. For recoverable bad records, preserve safe source identifiers such as topic/partition/offset or file path, record a rejection reason, and write the record to a quarantine path for later repair. For critical fields such as payment amount or event time, failing fast may be safer than silently substituting zero or null and publishing incorrect metrics. Quarantine writes should be replay-safe, and alerts/runbooks should prevent a poison record from either blocking progress indefinitely or disappearing unnoticed.

### Q100. What should a production Spark pipeline monitor?

Monitor application/query health, batch and stage duration, input/processing rates, Kafka lag, task failures, executor loss, shuffle, spill, GC, state size, watermark progress, sink latency, rejected records, and data freshness. In Java, the running `StreamingQuery` exposes `status()`, `lastProgress()`, and `recentProgress()` so the application can publish query status and batch metrics; the exact progress fields depend on source and Spark version. Correlate these with Spark UI/event logs and sink/database metrics. Alert on service-level objectives and trends, not only process uptime. Reconcile input/output counts and business totals because green infrastructure metrics do not prove correct results.

---

## 18. Deployment and operations

### Q101. What changes on Kubernetes?

Kubernetes schedules the driver and executor pods and enforces their resource requests and limits. Choose executor memory, memory overhead, and cores with both Spark's task behavior and the pod's total memory limit in mind; JVM heap is not the whole container footprint. Executor cores influence concurrent task slots, while more executors add capacity only when the workload has enough independent partitions and the cluster has resources. Configure the service account, network access, secrets, compatible image and connector dependencies, and durable checkpoint storage. Use pod events plus driver/executor logs to tell a Java heap OOM from a container OOM kill. Kubernetes manages processes and resources, but application-level sink idempotency and checkpoint recovery remain application design responsibilities.

### Q102. What matters when packaging a Java Spark application?

Use a JDK supported by the exact Spark distribution and matching Spark/Scala connector artifacts. The current Spark 4.2 documentation lists Java 17, 21, and 25 and Scala 2.13; Spark 4.0 removed Scala 2.12 support. Do not apply that matrix blindly to older Spark 3.x deployments—check the documentation for the version you actually run. Managed environments often provide Spark libraries, so avoid bundling conflicting copies. Kafka connector coordinates must match Spark and Scala binary versions. Test the packaged artifact in a representative cluster, not only in an IDE.

### Q103. How do you debug local success but cluster failure?

Start by classifying the symptom: class-not-found/linkage errors suggest dependency mismatch; serialization errors point to closures or unsupported objects; permission errors point to identity/configuration; executor loss or OOM points to runtime resources; timeouts may indicate networking or storage. Compare Spark/JDK/connector versions, deployment mode, classpath, credentials, DNS, file-system configuration, and container limits. Local mode often hides executor serialization, network reachability, and resource isolation issues. Collect driver, executor, and cluster-manager logs using the application ID and timestamps, then reproduce the smallest failing operation in a staging cluster.

---

## 19. Frequently asked questions

### Q104. How do file-read partitions differ from shuffle partitions?

File-read partitions are formed when Spark splits or packs input files for scanning. The settings `spark.sql.files.maxPartitionBytes` and `spark.sql.files.openCostInBytes` influence how file-based inputs are packed; the former limits bytes per file-read partition, while the latter estimates the cost of opening a file. Shuffle partitions are a separate set of partitions created for operations such as joins, sorts, and aggregations, with `spark.sql.shuffle.partitions` providing a default count for many SQL shuffles.

This distinction matters when the UI shows thousands of tiny input tasks but a reasonable shuffle stage—or the reverse. Fix file layout/input packing for a scan bottleneck; tune shuffle distribution for a shuffle bottleneck. Confirm the effective settings and input sizes before changing them.

### Q105. How do table and data-source statistics affect query planning?

Spark uses available estimates such as relation size and row counts to choose among eligible plans—for example, whether a relation is small enough to broadcast or which join order is likely to be cheaper. If statistics are missing or stale, a plan can make a poor decision even when the SQL is logically correct. The optimizer cannot estimate well what the source does not expose.

Inspect plan estimates and, where supported for the table/catalog, refresh statistics using the appropriate catalog/`ANALYZE TABLE` workflow. Join reordering specifically depends on eligible query shapes and relevant cost-based optimizer settings; basic size statistics can still affect join-strategy selection even when broader cost-based reordering is not applied. Statistics are inputs to planning, not guarantees: runtime sizes can differ, which is one reason AQE is useful. Do not make a broadcast decision from compressed file size alone.

### Q106. What is a broadcast nested-loop join, and when can it appear?

A broadcast hash join relies on equality join keys so Spark can build a hash table for the broadcast side. If the join condition is non-equi (for example, a range comparison) or the join type/condition does not support a broadcast hash join, Spark may choose a broadcast nested-loop join when a relation is broadcast. Conceptually, it tests rows from one side against rows from the broadcast side, so its work can grow roughly with the product of input sizes.

This is sometimes reasonable when one side is genuinely tiny, but dangerous when both sides are large. Inspect the physical plan for `BroadcastNestedLoopJoin`, verify output cardinality, and consider whether the business logic can be expressed as an equi-join plus a residual filter. A broadcast hint can prioritize a strategy but cannot make an inherently expensive join cheap.

### Q107. What is dynamic partition pruning?

Dynamic partition pruning is a Spark SQL optimization that can use values produced by one side of a join to skip irrelevant partitions on the other side at runtime. For example, if a small customer dimension filters the query to a subset of customers, Spark may avoid reading fact-table partitions that cannot match, provided the physical layout and query shape make pruning possible.

It is different from ordinary static partition pruning, where the filter values are already known from the query predicate. Dynamic pruning depends on eligible joins, partitioning, source capabilities, and configuration; do not assume every dimension join will skip fact data. Inspect the plan and scan metrics to see whether partitions were actually pruned.

### Q108. Are Spark file writes atomic?

Do not assume a plain DataFrame write provides universal transaction semantics. Spark writes through tasks and an output committer; the failure behavior depends on the filesystem/connector and committer implementation. A failed job can leave temporary or uncommitted objects, and object storage does not provide every local-filesystem rename behavior. A destination directory containing part files is not automatically a transactionally committed table.

For critical tables, use the recommended committer and storage connector, write to a controlled temporary path when appropriate, validate output, and publish with a supported table-format commit protocol if atomic table visibility is required. Test failures during task output and job commit. A successful Spark action is not the same thing as a business transaction spanning arbitrary external systems.

### Q109. When should you use `foreach` versus `foreachBatch`?

`foreach` gives custom logic at the row level and is useful when the sink requires per-record handling or when using continuous processing, where `foreachBatch` is not supported. `foreachBatch` passes the output of each micro-batch as a bounded DataFrame plus a batch ID. It lets you reuse batch writers, perform batch operations, or write the same batch to multiple sinks.

The trade-off is where you manage side effects. `foreach` can create per-row overhead and requires careful connection lifecycle/concurrency handling. `foreachBatch` is easier to batch, but it is retried with streaming recovery, so writes need idempotency or a transaction/deduplication protocol. Multiple actions on a stateful batch can recompute it; persist/unpersist inside the callback when repeated actions are genuinely needed.

### Q110. How do you monitor a running Structured Streaming query from Java?

Keep the `StreamingQuery` returned by `start()`. Its `status()` reports what the query is doing now; `lastProgress()` returns the most recent `StreamingQueryProgress`, including available source/processing rates, batch duration, input rows, and operator metrics supported by that query. `recentProgress()` provides recent progress records. These metrics can be exported to your monitoring system and correlated with Spark UI/event logs and sink-side telemetry.

No single metric explains every delay. Rising batch duration with stable input can indicate a newly expensive transform or sink; growing Kafka lag with processing rate below arrival rate indicates capacity imbalance; rising state size and long state commits point toward stateful-operation pressure. Interpret metrics together and alert on freshness/backlog SLOs, not just process liveness.

### Q111. Why can an AvailableNow query run a final batch with no new input?

For supported sources, `Trigger.AvailableNow()` processes the data available at execution start, possibly in multiple batches, then terminates. For stateful queries, Spark may run a final no-data batch when needed to advance the watermark and emit results that become final after that advancement. This is one reason AvailableNow is preferable to the deprecated one-time trigger for many scheduled catch-up jobs.

It is not supported identically by every source; check connector support and any documented fallback behavior. It is a bounded catch-up execution, not a promise that the query will stay running to process future arrivals.

### Q112. What is required to bound state in a stream-stream join?

Spark must know when an old row can no longer match future rows on the other side. Watermarks communicate expected event-time lateness, while an event-time range or window constraint establishes the latest possible matching time. For example, if a click can match an impression only from the impression time through one hour afterward, Spark can reason about when old impressions/clicks are no longer eligible for matches once watermark progress passes the relevant bound.

Adding watermarks without a usable time constraint may not be enough to bound join state. Outer joins have stricter requirements because Spark must know when to emit an unmatched row with nulls. Verify the documented join-type/time-constraint support matrix for the Spark version and test out-of-order/late-event cases.

### Q113. How do multiple input streams affect the global watermark?

A query with multiple streaming inputs tracks event-time progress for each input and uses a global watermark policy for stateful operations. The default `min` policy generally waits for the slowest stream, which is safer when late data from that stream must be retained but can delay cleanup if one input stalls. A `max` policy advances with the fastest stream, allowing state to be cleaned sooner but increasing the chance that records from a lagging stream arrive too late to be incorporated.

Choose the policy based on the business lateness guarantee, not simply memory pressure. The setting is `spark.sql.streaming.multipleWatermarkPolicy`; the current documentation defaults it to `min`. Monitor each input's event-time progress and state size, and test what happens during a source outage or a stream with very old timestamps. Switching to `max` can clean state sooner but may aggressively drop late data from the slower stream, so make the trade-off explicit before changing it.

### Q114. What happens when Kafka offsets have expired before a stream recovers?

A checkpoint can remember which offsets the query intended to process, but it cannot recreate records that Kafka has already removed through retention or topic changes. Inspect the last committed/available offsets, Kafka retention settings, and Spark's checkpoint progress. The Kafka source's `failOnDataLoss` option can ask Spark to fail when it detects that required offsets may be unavailable; disabling it may allow progress while silently skipping a data gap.

If required records have expired, recover from a durable archive/backfill if one exists and reconcile the affected output. Prevention includes retention sized for the maximum outage plus catch-up time, lag alerts, and recovery drills. Do not claim exactly-once recovery across a source gap that no longer exists in retained storage.

### Q115. How should JSON Kafka values be parsed safely?

Kafka's `value` column is binary. When the producer sends UTF-8 JSON, cast it to string and use `from_json` with an explicit `StructType` schema. Select fields from the parsed struct, but also detect rows where parsing failed or required fields are null. Preserve topic, partition, offset, timestamp, and a safe record identifier so a bad event can be traced and replayed.

For example, `from_json(col("value").cast("string"), eventSchema).alias("event")` creates a structured column from the payload. Route malformed or schema-incompatible records to quarantine with a reason, and monitor rejection rates. Do not silently convert missing payment amounts or invalid timestamps into plausible business values.

### Q116. Do all Structured Streaming sinks have the same delivery guarantee?

No. The official Spark guide lists different semantics by sink: the built-in file sink can provide exactly-once output under its documented checkpoint/committer conditions, while the Kafka sink is at-least-once and may duplicate records on retry. `foreachBatch` guarantees depend on the code and sink protocol implemented by the application. A process successfully restarting from a checkpoint is not proof that every external business effect occurred once.

Write down the boundary of the guarantee: source offsets, Spark state, sink commit, or business record. For a sink that can repeat writes, use deterministic keys/upserts, uniqueness constraints, deduplication, or a transaction that atomically records both the output and its processed batch marker.

### Q117. What is the difference between SparkSession and SparkContext?

`SparkSession` is the high-level entry point for DataFrame/Dataset and Spark SQL operations, readers/writers, catalog access, and session SQL configuration. `SparkContext` is the lower-level connection to Spark's execution environment and is associated with core RDD functionality and application coordination. A SparkSession uses a SparkContext underneath; most structured Java applications should begin with a SparkSession rather than manually creating multiple contexts.

A session can have SQL-specific configuration and catalog state. Do not assume creating another session means creating a new cluster application, and do not create multiple independent SparkContexts in one application as a normal way to isolate work.

### Q118. How are executor cores different from partitions and task parallelism?

A partition is a unit of data; a task is work scheduled for a partition in a stage. Executor cores (or configured task slots) determine how many tasks an executor can run concurrently, subject to scheduler/resource constraints. For example, 400 runnable tasks on 40 available task slots will execute in waves; adding slots may reduce queueing only if there is enough CPU, memory, network, and independent work.

More partitions increase potential parallelism but also scheduling overhead. More cores per executor can increase concurrency and memory pressure within that executor, while more executors distribute processes across nodes. Tune from task duration, executor utilization, skew, and resource headroom rather than assuming a fixed partition-to-core ratio is optimal.

### Q119. Why can a Java Spark lambda fail with a serialization error?

Spark sends task logic and the objects it captures from the driver to executor processes. A lambda that captures a non-serializable field, a large enclosing object, a database connection, or a service client can fail when Spark serializes the task closure—or can capture far more state than intended. Even if an object is serializable, a live network connection should not be assumed to work on an executor.

Keep task closures small and capture only the values each task needs. For external writes, create and close connections on the executor/partition side using the sink's recommended lifecycle, bound concurrency, and make writes retry-safe. Never capture a live Spring service graph or connection pool and assume Spark will distribute it correctly.

### Q120. Can a stateful streaming query always reuse its checkpoint after a code change?

No. Checkpoint compatibility depends on the query's stateful operators, state schema, source/sink configuration, and the nature of the change. Some non-semantic changes may be safe; changing grouping keys, state encoding, event-time logic, or operator structure can make the saved state incompatible or change results even if the query starts. Spark's recovery guide documents which changes are supported, so evaluate the specific change rather than using a blanket rule.

For incompatible changes, plan a migration: preserve the old checkpoint, define a replay boundary, rebuild state from retained source data if required, make sink writes idempotent, and reconcile the new result. Never delete a production checkpoint as a casual fix for a restart error.

---

## 20. Production scenario bank

These examples use straightforward interview language, but the answers show the reasoning expected from an experienced engineer. The key habit is to look for evidence first, make a targeted change, and verify that the output remains correct.

### Scenario 1 — A Spark job suddenly becomes slow

A daily orders job that normally finishes in 20 minutes now takes more than an hour. The code has not changed.

I would first compare the latest run with a known-good run in the Spark UI. I would look for the stage whose duration increased and compare input bytes, shuffle, spill, garbage collection, and task durations. If the scan reads much more data, I would check whether partition pruning stopped working or the source now contains many more files. If a join stage grew, I would check join cardinality and key distribution. If most tasks finish quickly but a few take much longer, I would investigate skew or an unhealthy executor. If Spark finishes its transformations but waits during output, I would examine the sink.

I would change one thing that matches the evidence and compare both runtime and output totals. Adding executors without knowing the bottleneck can increase cost without fixing anything.

### Scenario 2 — One task is much slower than all the others

Most tasks finish in under a minute, but one task runs for 15 minutes.

I would compare that task with the median task in the same stage: input and shuffle bytes, spill, GC time, and executor/host. If the slow task has far more data, the likely cause is skew—a key such as one very active customer may dominate the partition. I would check key frequencies, then consider AQE skew-join handling, a pre-aggregation, or a carefully designed salting strategy. If its input size is ordinary but the task is still slow, I would investigate the executor, disk, network, and GC instead.

I would not assume that adding partitions will split one hot key automatically. Keyed aggregations still need to combine values for that key correctly.

### Scenario 3 — A join creates a huge shuffle

Joining orders to a customer dimension suddenly creates a very large shuffle.

I would inspect the physical plan and count rows before and after the join. First I would verify that the join condition is correct and that the customer key is unique on the dimension side if the business model expects one customer row. Duplicate dimension keys can multiply output rows and make the shuffle much larger than expected. I would filter early and select only the columns needed. If the dimension is genuinely small after filtering, a broadcast join may avoid shuffling the large side. For two large inputs, a sort-merge join may be appropriate; skew may need a separate treatment.

I would validate row counts and business totals after changing the plan. A faster join that changes row multiplicity is not a successful fix.

### Scenario 4 — Executors run out of memory repeatedly

A Spark job fails with executor memory errors after a new dimension table is added.

I would first determine whether the JVM reports heap exhaustion or whether the cluster manager killed the container for exceeding its total memory limit. Then I would inspect the failing stage for a large broadcast, skewed partition, excessive cache, spill, and GC pressure. If the failures start at the broadcast, I would reconsider whether the relation is actually small enough to fit on every executor. If only one or two tasks fail, I would look for unusually large partitions.

I would reduce unnecessary columns and rows, remove unused caches, and address skew or the join strategy before increasing memory. Increasing heap may help a legitimate working set, but it will not solve an oversized broadcast or logically unbounded state.

### Scenario 5 — The driver crashes during collect()

A developer calls `collect()` on a large DataFrame and the driver runs out of memory.

`collect()` sends every result row to the driver, where all those objects must fit in memory. Executor memory does not protect the driver from this. I would ask what the code actually needs: if it needs a small sample, use `take(n)`; if it needs a total, aggregate in Spark and collect only the small result; if it needs to preserve all records, write to distributed storage.

I would also remove temporary `show()` or `count()` calls from a production path if they cause the same expensive pipeline to run again. The key lesson is to keep large datasets distributed instead of converting them into a local Java collection.

### Scenario 6 — A broadcast join causes long GC pauses

After forcing a broadcast join, executor GC time increases and the job becomes slower.

I would inspect the actual broadcast exchange size and executor memory use. The compressed source-file size can be much smaller than the in-memory relation used by the join. Broadcasting duplicates that relation across executors, so it can create memory and network pressure. I would filter and project the dimension before the join, remove the hint if it is too large, or let Spark use a shuffle-based strategy when that is safer. I would compare the final plan and stage metrics rather than judging the strategy by its name.

Broadcasting is useful only when the saved shuffle cost outweighs the broadcast and memory cost. I would monitor the dimension size because a join that was safe last month can become unsafe as the table grows.

### Scenario 7 — The job produces thousands of tiny files

A batch writes tens of thousands of small Parquet files, and downstream jobs spend a long time opening them.

I would inspect file sizes, task count, and the output partition columns. Too many small execution partitions can create many files, and partitioning by a very high-cardinality column can create many tiny directories. I would adjust the write layout and compact existing files with a distributed rewrite. I would not force a large dataset through `coalesce(1)`, because that turns a distributed write into a single-task bottleneck.

The goal is a practical file size and enough files for downstream parallelism—not always the fewest possible files. I would check downstream scan time after compaction to confirm the change helped.

### Scenario 8 — Increasing partitions makes the job slower

The team increases shuffle partitions from 200 to 2,000 and the job takes longer.

More partitions can improve parallelism when tasks are too large, but every task has scheduling and bookkeeping cost. I would compare task duration, scheduler delay, bytes per task, shuffle volume, and executor slots. If tasks were already tiny, the increase likely added overhead. If tasks were large and spilling, more partitions might help. Where AQE is enabled, Spark may coalesce small post-shuffle partitions using runtime statistics.

I would tune the stage that is actually slow, not assume one setting is best for every operation. The right value depends on data size, skew, and available resources, and a test should compare equal input and output correctness.

### Scenario 9 — A large input has only one effective task

The first stage reads a large input, but only one task does most of the work.

I would check whether the source is a single non-splittable file, whether an earlier operation reduced the data to one partition, or whether I am confusing a scan problem with a hot key after a shuffle. If it is one unsplittable input file, I would rewrite the source into a suitable set of splittable files. If the code calls `coalesce(1)` unnecessarily, I would remove it. If one key dominates a later aggregation, I would use a skew-specific approach.

A repartition that occurs after a slow scan cannot make the original scan parallel. I would fix the bottleneck at the stage where it originates.

### Scenario 10 — The job works locally but fails in the cluster

A Java Spark app works on a laptop but fails on Kubernetes with class errors or storage access failures.

I would classify the error before changing code. Class-loading errors often point to incompatible Spark or connector dependencies; permission errors point to identity or configuration; serialization errors can come from objects captured by task lambdas; OOM kills point to resource limits. Local mode does not test executor-to-storage connectivity or distributed task serialization properly.

I would compare the Spark, JDK, Scala-binary, and connector versions, then inspect driver/executor logs, pod events, credentials, and network access. I would run a small staging test using the packaged artifact and the same identity/storage paths as production. Passing an IDE run is not proof that the cluster deployment is configured correctly.

### Scenario 11 — AQE changes the plan unexpectedly

A query begins with a sort-merge join in the initial plan, but the final adaptive plan uses a different strategy.

That change is not automatically a bug. AQE uses runtime statistics, so it may decide that a relation is small enough to broadcast or that a shuffle partition is skewed. I would compare the initial and final plans and look at the runtime sizes and stage metrics. If the new strategy is slower, I would check whether estimates or data shape changed and benchmark a targeted adjustment. I would not disable AQE reflexively.

AQE can improve eligible plan decisions, but it cannot fix a wrong join condition or eliminate a genuinely enormous result. The important question is whether the final plan is appropriate for the actual data.

### Scenario 12 — A DataFrame pipeline is computed more than once

The application calls `show()`, then `count()`, then writes the same transformed DataFrame.

Each action can cause Spark to execute the upstream plan again. I would inspect the Jobs page and remove temporary diagnostic actions from the production path. If the same expensive result is genuinely used by several downstream operations, I would consider `persist()`, trigger cache population with an action, and `unpersist()` when finished. Caching is worthwhile only if reuse saves more work than cache population and memory pressure cost.

I would distinguish this from task retries, which repeat failed work, and from streaming replay, which has different recovery semantics. The same symptom—work appearing twice—can have different causes.

### Scenario 13 — Kafka lag grows continuously

A Structured Streaming query is running, but Kafka lag keeps increasing.

I would compare input rate with processing rate and examine batch duration, per-partition input, shuffle/spill, state-store time, and sink latency. If parsing or joins are expensive, I would optimize those operations. If source parallelism is too low, I would review Kafka partition count and, for large offset ranges, the connector's `minPartitions` option. If the sink is slow, I would improve bulk writes or scale the sink rather than adding Spark executors blindly.

`maxOffsetsPerTrigger` can limit the amount admitted to a micro-batch and protect a sink during catch-up, but it does not increase long-term throughput. I would alert on lag and estimate how long recovery takes at the measured processing rate.

### Scenario 14 — Restarting a stream appears to duplicate output

A streaming query restarts from its checkpoint, but the sink contains duplicate business records.

I would check where the failure happened relative to the sink commit and Spark's progress recording. A batch may have partially or fully affected an external system before Spark retries it. The checkpoint helps recover source progress and state; it does not make every external write exactly once. I would use a stable event ID with an upsert or uniqueness constraint, or a sink transaction that commits output and a processed-batch marker together.

I would preserve the checkpoint and test crashes before, during, and after sink commits. Deleting the checkpoint is not a safe way to remove duplicates; it changes the query's recovery history and may replay much more data.

### Scenario 15 — Late events are missing from a window result

A five-minute revenue window is lower than expected because some orders arrive late.

I would first inspect the event-time values, timezone parsing, watermark progress, and the business distribution of arrival delay. A watermark is based on observed event-time progress, not a timer that starts when each record arrives. A longer delay lets Spark retain window state longer and tolerate more late data, but increases state size and recovery cost. A shorter delay cleans state sooner but can prevent late records from affecting finalized results.

I would choose the threshold from measured lateness and business requirements, then monitor late-event counts. For financial reporting, I would also consider a correction or reconciliation job for records that arrive beyond the streaming window's accepted lateness.

### Scenario 16 — The watermark does not seem to reduce state

The query has a watermark configured, but its state store keeps growing.

I would check whether event-time values are valid and advancing, whether the watermark is applied before the relevant aggregation, and whether the selected output mode allows state cleanup. Complete mode retains the whole aggregation result, so it cannot discard old aggregate state through watermark cleanup in the same way as Append/Update. In a query with multiple streams, the global watermark policy can also be held back by a lagging input.

I would inspect watermark progress and state metrics before reducing the delay. Shortening a watermark without understanding late data can make the state look healthier while changing the result.

### Scenario 17 — A streaming state store grows every day

A keyed streaming aggregate becomes slower over time because state size and commit duration keep increasing.

I would look at key cardinality, windowing, duplicate horizon, and watermark progress. If the query keeps a separate state entry for every ever-seen key and has no valid cleanup rule, growing state may be expected rather than a memory leak. I would use bounded windows or retention only if the business can accept those semantics; otherwise, I would plan capacity and state maintenance for the required history.

Adding executor memory alone does not solve state that the query is required to retain indefinitely. I would monitor state rows/bytes, commit time, checkpoint time, and recovery duration.

### Scenario 18 — The sink cannot keep up with Spark

A streaming job transforms data quickly, but writing to a relational database takes most of each micro-batch.

I would measure sink latency and inspect database CPU, locks, connection count, transaction size, and retry rate. I would prefer batch or bulk writes, limit concurrent writers to a sustainable level, and consider staging followed by a database-side merge. More Spark executors may create more concurrent writes and make the database less stable.

I would set a sink throughput budget and test it with realistic data. If incoming traffic permanently exceeds what the sink can accept, tuning Spark alone cannot close the gap; the sink or architecture needs more capacity.

### Scenario 19 — The checkpoint directory becomes unavailable

A streaming query fails because the checkpoint path is inaccessible or its contents appear corrupted.

I would first preserve the error logs and determine whether the issue is temporary access/credentials or actual checkpoint loss. If the original checkpoint is intact, I would restore access and restart the same compatible query from it. If it is lost, I would plan controlled replay from a known offset or archive, rebuild state if possible, and make sink writes idempotent. I would not create a fresh checkpoint and claim it resumes the previous query.

A checkpoint contains progress and possibly state, so it is operationally important data. I would monitor access failures, protect retention policies, and regularly test the recovery procedure.

### Scenario 20 — A producer changes the schema and the stream fails

A producer adds a field or changes the type of an existing JSON field; the streaming query then fails or starts producing nulls.

I would capture the failing topic/partition/offset and inspect the actual payload against the declared schema. Adding an optional field is usually easier to accommodate than changing a numeric amount to a string or renaming a required field. I would normalize events to a canonical schema, quarantine records that cannot be safely parsed, and fail fast if continuing would corrupt important business metrics.

If the change affects stateful query compatibility, I would not delete the checkpoint casually. I would plan a controlled migration or replay and test old and new producer versions together. Schema contracts and rejection-rate alerts make these incidents easier to catch.

### Scenario 21 — One Kafka message cannot be parsed

A malformed event repeatedly triggers errors or blocks the pipeline.

I would preserve safe metadata such as topic, partition, offset, timestamp, and a record identifier, then compare the payload with the producer's schema and encoding. If the record can be quarantined safely, I would store it with a rejection reason for later repair. If the bad data could make critical calculations incorrect, I would fail the pipeline rather than silently substituting a zero or null.

The quarantine path must also be replay-safe, and malformed-record counts should be monitored. This keeps one poison message from either disappearing unnoticed or causing an endless unobserved retry loop.

### Scenario 22 — Batch and streaming totals disagree

A batch recomputation reports a different daily revenue total from the live streaming aggregate.

I would compare the two implementations using the same source boundary, timezone, null policy, event-ID rules, and window definitions. Then I would inspect duplicate events, late records, join multiplicity, and watermark behavior. A batch job over full history may include late events that the streaming query can no longer incorporate after its state has been finalized. Those differences must be understood rather than dismissed as Spark inconsistency.

I would define one business contract, add test cases around window boundaries and duplicate/late events, and reconcile live output against backfills. A successful Spark job proves execution completed; it does not prove the business logic is correct.

### Scenario 23 — Database writes must be idempotent

A micro-batch partially writes to a database and then fails. On retry, the same records must not create duplicate rows.

I would use a stable event ID and a database upsert or uniqueness constraint. If using a batch marker, the marker and the batch's data changes should be committed in the same database transaction; otherwise, a failure between those operations can still duplicate output. If updates can arrive out of order, I would store a version or sequence and prevent older updates from overwriting newer state.

I would test failure before commit, after data writes, and after the commit response. `batchId` is useful for batch-level idempotency, but record-level event keys are still important when records can be partially written or replayed independently.

### Scenario 24 — One hot key dominates a join

A tenant ID represents a large fraction of the rows, making one join task much slower than the rest.

I would confirm skew by comparing task shuffle sizes and measuring key frequencies on both sides. I would also check whether the join is many-to-many, because unexpected duplicate keys can inflate the output. Depending on the shape, I might use AQE skew-join handling, handle the hot key separately, pre-aggregate, or salt the key and replicate the matching smaller-side rows carefully.

Salting is not a generic random repartition: the replicated side and final result must be combined correctly. I would compare row counts and business totals after the change and monitor hot-key distribution over time.

### Scenario 25 — Garbage collection consumes much of the runtime

The Spark UI shows that executors spend a large fraction of task time in GC.

I would check whether the problem is concentrated in one stage and correlate GC with heap occupancy, broadcasts, cached data, spill, and task size. Object-heavy RDD code, repeated conversion to Java objects, or an oversized broadcast can cause allocation pressure. I would reduce unnecessary data, prefer built-in DataFrame expressions where possible, remove unused caches, and fix skew before changing heap settings.

A larger heap may lower the frequency of collections but can make some pauses longer. I would measure after each change and track GC time in future performance runs.

### Scenario 26 — Shuffle fetch failures repeat

Tasks fail because they cannot fetch shuffle data, and Spark keeps retrying the stage.

I would inspect the exact fetch error and determine whether failures follow one executor or host. Shuffle files may have been lost when an executor disappeared, or the issue may be disk capacity, network timeout, or node instability. I would use executor and worker logs, disk metrics, and cluster events to identify the source. Retries can recover from a transient failure, but they cannot restore shuffle data that repeatedly disappears.

I would fix the failing node/storage/network issue and verify how shuffle data is preserved in the deployment. Only after confirming that the infrastructure is healthy would I consider retry-related tuning.

### Scenario 27 — The Spark UI shows too many tiny tasks

A stage schedules tens of thousands of tasks, each reading very little data.

I would determine whether the tasks came from tiny input files or an excessive shuffle partition count. File reads are influenced by input file sizes and file-packing settings; shuffle tasks are controlled separately by SQL shuffle settings and AQE. If the problem is tiny files, I would compact them. If it is a shuffle, I would tune the relevant partitioning or let AQE coalesce eligible partitions. I would check scheduler delay and task durations to verify the overhead is material.

I would avoid reducing partitions so aggressively that the next run has a few enormous tasks. The goal is a balanced task size, not simply the smallest task count.

### Scenario 28 — A high-cardinality aggregation exhausts memory

A groupBy over events spills heavily or fails when there are many unique keys.

I would check expected distinct-key count, task skew, spill, and whether partial aggregation reduces the data before the shuffle. Partial aggregation helps when many rows share keys, but provides little reduction when nearly every row has a unique key. I would filter and project early, pre-aggregate when the calculation allows it, and address skew. If the output truly contains an enormous number of unique groups, that output has a real cost; a bigger heap cannot remove it.

If possible, I would discuss whether the report needs that level of detail or could use a coarser grain. That is a business decision, not merely a tuning trick.

### Scenario 29 — A stream must recover after several hours offline

The query has been down for six hours, and the team needs to catch up without losing events or overloading the sink.

I would verify that the checkpoint is intact and Kafka still retains the required offsets. A checkpoint records progress but cannot restore messages already removed by Kafka retention. I would estimate backlog and measure sustainable processing throughput, then decide whether to allocate extra capacity or limit per-trigger input to protect the sink during catch-up. If required offsets have expired, I would use a durable archive/backfill and reconcile the gap.

I would also confirm that the sink is safe for replay. Recovery planning needs source retention, catch-up time, sink capacity, and idempotency to work together.

### Scenario 30 — Kafka volume increases tenfold

Traffic spikes, and the stream's batches take longer than the trigger interval.

I would measure whether the bottleneck is source parallelism, parsing, joins/shuffles, state updates, or sink writes. I would also check whether traffic is evenly spread across Kafka partitions. More partitions may help when the source has insufficient parallelism, but changing Kafka partition count has key-distribution and ordering implications. If the sink is already saturated, adding Spark executors may increase write pressure rather than throughput.

I would load-test the expected peak and sustained rate separately, then monitor how much faster processing runs than incoming traffic. That excess capacity determines how quickly the pipeline can recover from a backlog.

### Scenario 31 — Events must be deduplicated reliably

Producer retries send the same event more than once, and the duplicates inflate order metrics.

I would first verify that the producer supplies a stable event ID. An order ID alone may not be unique if the order has multiple legitimate status events. I would then configure streaming deduplication with a watermark or other supported state boundary that matches how late duplicates may arrive, and make the sink idempotent as a second line of defense.

There is a real trade-off: remembering IDs for longer consumes more state; bounded deduplication cannot promise to suppress duplicates that arrive outside its retention horizon. I would measure duplicate rates and agree on that horizon with the business team.

### Scenario 32 — A historical backfill must transition to live events

The team must backfill old orders and then continue consuming live Kafka events without a gap.

I would define a precise handoff boundary using a source offset or trusted timestamp and ensure both paths use the same schema and transformation rules. A practical design often allows some overlap and deduplicates it using stable event IDs; overlap is safer than silently missing events if output is idempotent. Simply starting the live stream after the backfill finishes can leave a race window if new records arrive during the handoff.

I would document the cutover and reconcile counts around the boundary. The exact procedure depends on the historical source and whether Kafka still retains the required offsets.

### Scenario 33 — The Parquet schema changes over time

New files contain a field or type that older files do not have, and a query fails or returns nulls.

I would inspect the schema of old and new partitions rather than assuming the whole directory has one schema. Adding an optional field can be handled by mapping both versions into a canonical schema; changing a field's type needs explicit validation and conversion rules. Blind schema merging or permissive casts can hide bad values and add planning cost.

I would add schema compatibility tests and validate row counts/null rates after the change. Historical data must remain readable under the chosen contract, not just the new partition.

### Scenario 34 — A Spark application must run on Kubernetes

Executors are killed or cannot access checkpoint storage after deployment.

I would inspect pod events and driver/executor logs, then check image and dependency compatibility, service-account permissions, network access, and total container memory. A pod can exceed its memory limit even without a Java heap OOM because native and direct-buffer memory also count. I would set resource requests/limits and memory overhead based on the observed workload, then test storage/checkpoint access using the deployed identity.

I would validate the packaged application in staging. Kubernetes schedules and supervises pods, but it does not make the Spark query, checkpoint, or external sink automatically correct.

### Scenario 35 — One sink succeeds while another fails

A pipeline writes to Kafka and Elasticsearch, but Elasticsearch fails after Kafka has accepted the batch.

Those writes are not one atomic transaction. A Spark checkpoint does not make independent systems commit together. I would choose a durable source of truth and make each sink independently replayable where possible. Elasticsearch writes should use deterministic document IDs or version-aware upserts; Kafka consumers should use stable event IDs and handle duplicates. I would monitor and reconcile the two sinks separately.

If the application itself writes to both systems, it needs an explicit partial-failure strategy. The simpler architecture is often to write once to a durable log/table and let independent sink pipelines retry safely.

### Scenario 36 — The database becomes overloaded after scaling Spark

After adding executors, Spark writes faster briefly, but database CPU, locks, connections, and retries rise sharply.

I would compare concurrent Spark write tasks with database connection count, transaction size, lock waits, CPU, and I/O. The database has a finite sustainable write rate. I would batch writes, limit concurrent writers, or stage data and merge it in the database. I would also set bounded retries with backoff so failed writes do not amplify overload.

More Spark parallelism is not automatically more end-to-end throughput. I would scale the sink or reduce the incoming workload if the database is the confirmed bottleneck.

### Scenario 37 — A report's totals do not match the source

The Spark job succeeds, but daily revenue differs from the source system.

I would reconcile counts and amounts at each boundary: raw input, successfully parsed events, filtered rows, post-join rows, deduplicated events, and final aggregates. I would inspect null amounts, timezone boundaries, duplicate IDs, join-key uniqueness, and late records. A duplicate dimension key or incorrect join predicate can inflate totals even though no task fails.

I would fix the business logic and replay affected data idempotently, then compare totals before publishing. For important financial data, reconciliation should be an explicit pipeline check, not a manual step after an incident.

### Scenario 38 — A UDF makes a query slower

Runtime doubles after custom logic is moved into a Java UDF.

I would compare the query plan and CPU metrics, then check whether built-in Spark SQL functions can express the same logic. Spark can inspect built-in expressions, but it generally cannot simplify the code inside an opaque UDF. Applying the UDF to fewer rows and fewer columns can also help. If the UDF is genuinely necessary, I would benchmark it with realistic data and check allocation/serialization cost.

I would not remove a UDF just for style; I would keep it when it expresses specialized logic that cannot be handled adequately with built-ins, and test its null/type behavior carefully.

### Scenario 39 — A Parquet query scans far too much data

A query filtered to one date reads hundreds of gigabytes from a date-partitioned dataset.

I would inspect the physical scan for partition filters, pushed filters, selected columns, and actual bytes/files read. The filter may be expressed in a way that prevents partition pruning, or the data layout may not match the query. I would use a compatible predicate directly on the partition column and project only the required columns. If many queries need a different layout, I would consider a controlled rewrite rather than repeatedly scanning unnecessary data.

I would compare the scan metrics before and after the change. A correct filter does not guarantee that the source can skip all irrelevant bytes.

### Scenario 40 — Updates arrive out of order

An order emits several status events, but an older event sometimes overwrites a newer status.

I would not use event time alone as the ordering contract; producer clocks can be wrong and events can be delayed. I would ask whether every event has a monotonic version or sequence number. The sink can then accept an update only when its version is newer than the version already stored. If the business also needs the full history, I would preserve every event separately rather than treating the current-state table as the audit log.

I would test duplicate and out-of-order updates and define what to do when two records have the same version. Watermarks help with event-time state cleanup; they do not automatically resolve business conflicts.

### A simple way to explain a production incident

In an interview, say what you would measure first, explain what each likely metric would tell you, and then connect the evidence to a fix. Finish by stating how you would check that the fix did not change the result. This sounds more credible than listing Spark settings without explaining why they are relevant.

~~~mermaid
flowchart TD
  A[Define the expected result and SLA] --> B[Find the slow stage or delayed batch]
  B --> C[Check task times, input, shuffle, spill, GC, and sink latency]
  C --> D{What does the evidence show?}
  D -->|Too much data read| E[Check file layout and pruning]
  D -->|Uneven task work| F[Check skew, keys, and join plan]
  D -->|Memory pressure| G[Check cache, broadcast, heap, and partition size]
  D -->|Growing stream backlog| H[Check rates, state, and sink capacity]
  E --> I[Make one targeted change]
  F --> I
  G --> I
  H --> I
  I --> J[Compare runtime and verify output correctness]
~~~


## 21. Mock interviews and revision

Use these as realistic practice rounds, not as a script to memorize. In a real interview, state your assumptions, talk through the evidence you would collect, and explain why you choose a solution. The model answers below show the level of reasoning expected; they are not the only valid designs.

### Mock interview A — Spark fundamentals and execution

**Interviewer prompt:** You receive a 500 GB dataset of order events. Remove failed orders, join the remaining events to customer data, and aggregate revenue by customer. Explain how Spark would execute the job.

**Model answer:** I would read the input as a DataFrame and express the filter, join, and aggregation using built-in Spark SQL expressions. Those transformations build a logical plan; they do not immediately process the whole dataset. When I trigger an action such as writing the aggregate, Spark analyzes and optimizes the plan, selects physical operators, and schedules tasks over partitions.

The filter may reduce the rows before the join. If the customer data is genuinely small, Spark may broadcast it; if both sides are large, a shuffle-based join is more likely. The aggregation may also shuffle so rows for each customer can be combined. In the UI I would look at the scan size, exchanges, task-duration spread, shuffle, spill, and output cardinality. I would not say “500 GB means 500 executors”: the appropriate resources depend on data distribution, row width, shuffle volume, target runtime, and the cluster.

**Follow-up:** Why can one job have several stages? What evidence would make you suspect skew rather than simply large input?

### Mock interview B — Spark SQL/DataFrame coding round

**Interviewer prompt:** Given orders with `order_id`, `customer_id`, `order_ts`, `status`, and `amount`, calculate paid revenue per customer per day. Then rank each customer's days by revenue.

**Model answer (focused Java snippet):**

~~~java
Dataset<Row> daily = orders
    .filter(col("status").equalTo("PAID"))
    .groupBy(
        col("customer_id"),
        to_date(col("order_ts")).alias("order_date"))
    .agg(sum("amount").alias("revenue"));

WindowSpec byRevenue = Window.partitionBy("customer_id")
    .orderBy(col("revenue").desc());

Dataset<Row> ranked = daily.withColumn(
    "revenue_rank", dense_rank().over(byRevenue));
~~~

I would explain that the filter removes unpaid orders, the groupBy produces one row per customer/day, and sum ignores null amounts. If every amount in a group is null, the sum is null; I would replace it with zero only if the data contract explicitly defines missing amount as zero. I used `dense_rank` ordered only by revenue so two days with equal revenue receive the same rank. Adding the date as a second ranking expression would break that tie. If deterministic display order is needed, sort the final result separately by customer, rank, and date.

I would also clarify whether `order_ts` is an instant or a local timestamp and which timezone defines the business day. Finally, I would validate row counts and totals against the filtered source data.

**Follow-up:** How would you return the top three days per customer? What changes if the requirement is a running total rather than a rank?

### Mock interview C — Performance troubleshooting

**Interviewer prompt:** A large join runs for 40 minutes. Most tasks finish in two minutes, but a small number run for 25 minutes. What do you do?

**Model answer:** I would start with the stage that contains the long-running tasks and compare input bytes, shuffle read, spill, GC time, executor, and host. If the slow tasks handle much more data, I would profile the join keys and check whether a few keys dominate the data. I would also check whether the join output count is unexpectedly large because of duplicate keys.

If the evidence shows skew, I would see whether AQE's skew-join handling applies. If not, I might aggregate before joining when correct, isolate hot keys, or salt them with a carefully designed second phase. If task sizes are similar but one executor is slow, I would investigate node health, GC, disk, and network rather than applying a skew fix. After the change, I would compare both runtime and output counts/totals.

**Follow-up:** Why might raising executor count not help? Why is salting a correctness-sensitive change?

### Mock interview D — Structured Streaming and Kafka

**Interviewer prompt:** A Kafka-to-database pipeline restarts after a failure. Some rows appear twice. Is this a Spark bug, and how would you fix it?

**Model answer:** Not necessarily. The checkpoint allows Spark to recover its recorded source progress and state, but the database write can succeed before Spark has durably recorded the batch as complete. On retry, that external write may be attempted again. The Kafka sink also has documented at-least-once write semantics, so duplicates are possible there too.

I would find the failure boundary using batch progress, source offsets, application logs, and database commit records. I would make database writes idempotent, for example by upserting on a stable event ID and enforcing uniqueness. If using a batch marker, I would commit the marker and the data writes in one database transaction. I would test failure before the write, during the write, and after commit. I would preserve the checkpoint and avoid pretending that starting from a fresh checkpoint fixes duplicates.

**Follow-up:** What guarantee does the checkpoint provide? How does your design behave if a batch partly commits?

### Mock interview E — Senior architecture and reliability

**Interviewer prompt:** Design a pipeline that consumes order events from Kafka, enriches them with customer data, calculates five-minute event-time revenue, and writes results to Kafka and Elasticsearch.

**Model answer:** I would first clarify the latency target, expected event volume, acceptable lateness, replay horizon, schema ownership, and whether late orders must correct previously published metrics. I would read Kafka with Structured Streaming, parse events with an explicit schema, quarantine invalid records, enrich using a suitable customer dimension, and calculate revenue with a five-minute event-time window and a watermark chosen from observed/business lateness. I would monitor source lag, batch duration, state size, watermark progress, rejected records, and sink latency.

I would not assume writes to Kafka and Elasticsearch form one atomic transaction. I would choose a durable source of truth and make each sink replayable and idempotent. For Elasticsearch, deterministic document IDs or version-aware upserts can avoid duplicates; the Kafka output still needs a stable key and downstream duplicate handling because the Spark Kafka sink is at-least-once. If independent sink pipelines are possible, they usually make partial failure and replay easier to reason about. I would finish by defining reconciliation and recovery procedures for sink outages and events that arrive beyond the watermark.

**Follow-up:** What happens when Kafka retention expires before recovery? How would you reprocess a day without double-counting revenue?

### Top 50 must-know questions — navigation to detailed answers

Use these links to jump to the full explanation rather than rereading a duplicate list of questions.

1. [Spark's purpose and when to use it — Q3](#q3-what-is-apache-spark-and-when-is-it-appropriate)
2. [Hadoop, HDFS, and MapReduce — Q2](#q2-explain-hadoop-hdfs-and-mapreduce)
3. [Driver, executor, and Spark architecture — Q7](#q7-describe-sparks-architecture)
4. [Jobs, stages, and tasks — Q8](#q8-explain-jobs-stages-and-tasks)
5. [What triggers a job and how count executes — Q9](#q9-what-happens-when-java-code-calls-count-on-a-dataframe)
6. [Lazy evaluation — Q12](#q12-what-is-lazy-evaluation)
7. [Narrow and wide dependencies — Q11](#q11-narrow-versus-wide-dependencies)
8. [RDDs, DataFrames, and Datasets — Q15](#q15-compare-rdds-dataframes-and-datasets)
9. [Why DataFrames can be faster — Q16](#q16-why-can-dataframes-outperform-hand-written-rdd-code)
10. [Reading the physical plan with explain — Q31](#q31-how-do-you-inspect-a-query-plan)
11. [Shuffles and their cost — Q34](#q34-what-causes-a-shuffle-and-why-is-it-expensive)
12. [Partitions and parallelism — Q33](#q33-what-is-a-partition-and-how-does-it-relate-to-parallelism)
13. [repartition versus coalesce — Q35](#q35-repartition-versus-coalesce)
14. [Diagnosing data skew — Q37](#q37-what-is-data-skew-and-how-do-you-diagnose-it)
15. [Salting hot keys — Q38](#q38-explain-salting-and-its-risks)
16. [Partition pruning — Q40](#q40-what-is-partition-pruning)
17. [spark.sql.shuffle.partitions — Q36](#q36-what-does-sparksqlshufflepartitions-control)
18. [Broadcast, sort-merge, and other joins — Q41](#q41-which-join-strategies-should-you-know)
19. [Adaptive Query Execution — Q43](#q43-what-is-adaptive-query-execution)
20. [Join types and duplicate rows — Q22](#q22-explain-join-types-and-null-behavior)
21. [Why distinct can be expensive — Q44](#q44-why-can-distinct-be-expensive)
22. [Window functions versus groupBy — Q23](#q23-how-do-window-functions-differ-from-groupby)
23. [Catalyst and query planning — Q46](#q46-explain-spark-sqls-planning-pipeline)
24. [Predicate pushdown and projection pruning — Q47](#q47-predicate-pushdown-versus-projection-pruning)
25. [Driver memory versus executor memory — Q50](#q50-driver-memory-versus-executor-memory)
26. [Spill to disk — Q52](#q52-why-does-spark-spill-to-disk)
27. [Garbage collection diagnosis — Q54](#q54-how-do-you-diagnose-excessive-garbage-collection)
28. [cache versus persist — Q56](#q56-cache-versus-persist)
29. [Checkpoint versus cache — Q57](#q57-checkpoint-versus-cache)
30. [How to investigate a slow job — Q59](#q59-how-should-you-investigate-a-slow-job)
31. [The small-file problem — Q61](#q61-why-are-small-files-harmful)
32. [Why Parquet suits analytics — Q64](#q64-why-is-parquet-common-for-analytics)
33. [Why output is a directory of part files — Q65](#q65-why-does-a-spark-write-create-a-directory-of-part-files)
34. [Structured Streaming fundamentals — Q68](#q68-what-is-structured-streaming)
35. [Triggers and micro-batches — Q72](#q72-what-does-a-trigger-control)
36. [Append, Update, and Complete modes — Q71](#q71-explain-append-update-and-complete-output-modes)
37. [Checkpoint directories — Q73](#q73-what-is-a-checkpoint-directory)
38. [Event time versus processing time — Q76](#q76-event-time-versus-processing-time)
39. [Tumbling, sliding, and session windows — Q77](#q77-tumbling-versus-sliding-windows)
40. [Watermarks and state cleanup — Q78](#q78-what-does-a-watermark-do)
41. [Events that arrive after the watermark — Q79](#q79-what-happens-to-records-later-than-the-watermark)
42. [Stateful aggregation and deduplication — Q80](#q80-how-do-stateful-aggregation-and-deduplication-work)
43. [Kafka offsets and restart behavior — Q87](#q87-how-do-offsets-and-restart-work)
44. [Spark source tasks versus Kafka partitions — Q86](#q86-are-spark-tasks-the-same-as-kafka-consumer-group-members)
45. [Exactly-once boundaries — Q75](#q75-does-structured-streaming-guarantee-exactly-once-processing)
46. [Idempotent database writes — Q88](#q88-how-do-you-make-database-writes-idempotent)
47. [Kafka lag and throughput — Q89](#q89-what-is-backpressure-and-how-do-you-handle-kafka-lag)
48. [Executor loss and recovery — Q95](#q95-what-happens-if-an-executor-is-lost)
49. [Checkpoint loss and controlled replay — Q98](#q98-what-if-a-streaming-checkpoint-is-unavailable-or-corrupted)
50. [Multiple sinks, partial failures, and replay — Scenario 35](#scenario-35--one-sink-succeeds-while-another-fails)

### Top 25 Spark SQL/DataFrame patterns

1. Select only needed columns.
2. Filter before expensive joins/aggregations.
3. `withColumn` for derived values.
4. `when/otherwise` for conditions.
5. `coalesce` for deliberate null fallback.
6. `isNull/isNotNull` for null predicates.
7. `groupBy/agg` for distributed aggregation.
8. `count(*)` versus `count(column)`.
9. `row_number` for ranking.
10. `dense_rank` for ties.
11. Left join for enrichment.
12. Semi join for existence.
13. Anti join for missing keys.
14. Broadcast only safely small tables.
15. `unionByName` for schema alignment.
16. `dropDuplicates(keys)` for key deduplication.
17. Parse dates/timestamps with timezone awareness.
18. `explode` for array expansion.
19. Temporary view plus Spark SQL.
20. Explicit schema for CSV/JSON.
21. Parquet projection and predicates.
22. Repartition only when redistribution is justified.
23. Coalesce output tasks when unevenness is acceptable.
24. `explain("formatted")` for plan inspection.
25. Persist reused expensive data and unpersist afterward.

### Top 20 Structured Streaming concepts

1. Unbounded input table.
2. Micro-batch and trigger cadence.
3. Source offsets and checkpoint.
4. Query ID versus run ID.
5. Output modes.
6. Event versus processing time.
7. Tumbling/sliding/session windows.
8. Watermark and state cleanup.
9. Late-data trade-offs.
10. Stateful aggregation.
11. Deduplication horizon.
12. Stream-static versus stream-stream joins.
13. State-store metrics and growth.
14. Kafka offset recovery.
15. Kafka partition parallelism.
16. `foreachBatch` retry implications.
17. Idempotent sinks.
18. Kafka lag and throughput.
19. Schema evolution/quarantine.
20. Multi-sink partial failure.

### Common mistakes

- Treating Spark as a transactional database or replacement for every backend service.
- Assuming transformations execute immediately.
- Collecting large results to the driver.
- Assuming more executors always improve performance.
- Assuming every shuffle is avoidable.
- Confusing storage partitions with execution partitions.
- Using a single output partition for huge data.
- Broadcasting based only on compressed file size.
- Assuming AQE fixes every skew/cardinality issue.
- Caching everything without measuring reuse.
- Treating a watermark as a timer or universal hard filter.
- Treating a checkpoint as an exactly-once transaction for arbitrary sinks.
- Assuming consumer-group offsets alone govern Spark Kafka restart.
- Assuming Kafka and Elasticsearch writes are atomic.
- Silently replacing malformed business values with defaults.
- Treating successful job status as proof of business correctness.

### Last-minute revision checklist

- [ ] Explain driver/executor/job/stage/task and shuffle.
- [ ] Read a physical plan and relate it to Spark UI metrics.
- [ ] Diagnose skew using task metrics and key distribution.
- [ ] Compare broadcast and sort-merge joins and memory risks.
- [ ] Distinguish driver OOM, executor heap OOM, and container kill.
- [ ] Explain cache, lineage, and checkpointing.
- [ ] Explain event time, windows, watermarks, state, and late data.
- [ ] Explain Kafka offset recovery and sink idempotency.
- [ ] Discuss replay, retention, backpressure, schema change, and partial writes.
- [ ] State the Spark/JDK/connector versions assumed.


## 22. Official references

Use documentation matching the deployed Spark version; the latest URLs change as releases publish.

- [Apache Spark overview and current Java/Scala runtime requirements](https://spark.apache.org/docs/latest/index.html)
- [Spark Java API documentation (Javadoc)](https://spark.apache.org/docs/latest/api/java/index.html)
- [Apache Spark documentation](https://spark.apache.org/docs/latest/)
- [RDD programming guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html)
- [Spark SQL, DataFrames, and Datasets](https://spark.apache.org/docs/latest/sql-programming-guide.html)
- [Structured Streaming overview](https://spark.apache.org/docs/latest/streaming/index.html)
- [Structured Streaming quick start](https://spark.apache.org/docs/latest/streaming/getting-started.html)
- [Structured Streaming APIs, query lifecycle, monitoring, and checkpoint recovery](https://spark.apache.org/docs/latest/streaming/apis-on-dataframes-and-datasets.html)
- [Structured Streaming Kafka integration](https://spark.apache.org/docs/latest/streaming/structured-streaming-kafka-integration.html)
- [Spark tuning guide](https://spark.apache.org/docs/latest/tuning.html)
- [Spark on Kubernetes](https://spark.apache.org/docs/latest/running-on-kubernetes.html)
- [Spark configuration](https://spark.apache.org/docs/latest/configuration.html)
- [Spark SQL performance tuning](https://spark.apache.org/docs/latest/sql-performance-tuning.html)

**Validation note:** This is a documentation-focused handbook; snippets are illustrative patterns, not a claim that every example was compiled or run in a live cluster. Verify API signatures, connector coordinates, default settings, supported streaming operations, AQE rules, and state behavior against your deployed Spark/JDK/connector versions.
