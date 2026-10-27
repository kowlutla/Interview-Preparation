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

Each question below describes a concrete situation you might be given in a Spark interview. A strong answer should connect the observed symptom to evidence, explain why the chosen fix addresses the cause, and state how correctness will be checked. Do not jump straight to a configuration change before identifying the bottleneck.

### Scenario 1 — A Spark job suddenly becomes slow

**Scenario:** Your production orders pipeline has processed about 500 GB of data and usually finishes in 22 minutes. Today it takes 75 minutes. The code and cluster size have not changed, and the business deadline is in one hour.

**Question:** You are on call. What do you inspect first, how do you narrow down the cause, and how do you decide whether a change is safe to make before the deadline?

**Model answer:** I would compare this run with a recent successful run in the Spark UI. I would first find the stage whose duration increased, then compare input bytes, task-duration distribution, shuffle read/write, spill, GC time, and executor failures. A larger scan points me toward source growth or lost partition pruning; a slow join stage points me toward changed join cardinality or skew; a few long tasks suggest skew or unhealthy executors; long write time suggests the sink is limiting progress. I would verify the suspected cause against logs and the physical plan, make one targeted change, and compare both runtime and business totals. I would not add executors just because the job is slow: more workers do not fix an inefficient scan, hot key, or saturated sink.

**Likely follow-up:** What evidence would make you suspect data skew rather than a general increase in input volume?

### Scenario 2 — One task is much slower than all the others

**Scenario:** In a customer aggregation stage, 399 tasks finish in 45–70 seconds, while one task runs for 18 minutes. That task reads far more shuffle data than the median task and spills heavily.

**Question:** What is your leading hypothesis? How would you confirm it, and which remedies would you compare?

**Model answer:** My leading hypothesis is a skewed key or partition, but I would confirm it rather than guessing from duration alone. In the Spark UI, I would compare the slow task's shuffle bytes, records, spill, and host with the other tasks. I would also measure key frequencies before the shuffle to see whether one customer or tenant dominates. If the stage is a supported skewed join, I would test AQE skew-join handling. For an aggregation, I might pre-aggregate or use a carefully designed two-stage aggregation; for a join, salting can help only if the other side is replicated or partitioned correctly. If the slow task has ordinary input, I would investigate executor health, disk, and GC. I would then verify counts and totals; merely increasing the partition count does not automatically split one hot key.

**Likely follow-up:** How would you tell a hot-key problem apart from a slow executor?

### Scenario 3 — A join creates a huge shuffle

**Scenario:** A 400 GB orders table is joined to a customer dimension expected to contain one row per customer. After a new dimension load, the join output triples and shuffle writes jump sharply.

**Question:** How would you identify why the join expanded, choose a join strategy, and prove the fix has not changed business results?

**Model answer:** I would begin with the physical plan and row counts before and after the join. Because the dimension is expected to have one row per customer, I would check whether customer IDs have become duplicated. A many-to-many join can multiply orders, revenue, and shuffle data even when Spark completes successfully. I would validate the join predicate, filter early, and select only the columns needed. If the filtered dimension is genuinely small enough to fit in executor memory, broadcasting it may remove a large-side shuffle; otherwise a shuffle-based join may be safer. I would compare the old and new row counts, unmatched-key counts, and revenue totals, and separately confirm that the dimension uniqueness constraint holds. I would not force broadcast based only on the table's compressed file size.

**Likely follow-up:** Which data-quality check would you add to catch this before the next production run?

### Scenario 4 — Executors run out of memory repeatedly

**Scenario:** A job started failing after adding a 6 GB dimension to a join. Executor logs show memory pressure; in some attempts the container is killed, while in others the JVM reports an out-of-memory error.

**Question:** How would you distinguish the failure types, find the operation responsible, and decide whether to increase memory or redesign the query?

**Model answer:** I would separate JVM heap exhaustion from a container-level OOM kill. I would inspect the executor logs and Kubernetes or cluster-manager events, then identify the stage that fails. If the problem starts during a broadcast exchange, I would check the actual in-memory broadcast size and whether it is being copied to every executor. If only a small number of tasks fail, I would compare their partition sizes and check for skew. I would reduce unnecessary columns, filter the dimension before joining, remove unused caches, or choose a shuffle join if the broadcast is too large. Only if the working set is legitimate and the container has appropriate memory overhead would I consider increasing resource limits. I would rerun with realistic data and monitor peak memory, spill, GC, and output correctness.

**Likely follow-up:** Why can a container be killed even when the JVM heap has not reached its configured maximum?

### Scenario 5 — The driver crashes during collect()

**Scenario:** A developer adds collect() after filtering a 300-million-row DataFrame so that a Java method can loop through all results. The executors appear healthy, but the driver crashes with an out-of-memory error.

**Question:** Explain the failure and propose alternatives for three different needs: sampling rows, calculating a total, and exporting every row.

**Model answer:** collect() transfers every result row to the driver and materializes the results locally. The driver must hold those objects, so healthy executor memory does not prevent a driver OOM. If the code needs only a small sample, I would use take(n) or limit(n) and collect that bounded result. If it needs a total or other summary, I would calculate it in Spark and collect only the small aggregate. If it needs all rows, I would write the DataFrame to distributed storage or process it with a distributed sink rather than bringing it into one JVM. I would also remove accidental diagnostic actions from production code because show(), count(), and a final write can each trigger separate work. I would verify that the replacement preserves the required output and does not silently truncate data.

**Likely follow-up:** When is collect() acceptable in production code?

### Scenario 6 — A broadcast join causes long GC pauses

**Scenario:** After a developer adds a broadcast hint, the join stage gets faster initially, but total job runtime increases. Executors now spend 35% of task time in garbage collection and some tasks spill.

**Question:** How would you determine whether broadcast is still the right strategy and what would you change?

**Model answer:** I would inspect the executed plan, the size of the broadcast relation, executor memory pressure, GC time, and the stage timeline. Compressed files on storage can expand substantially after decoding and building the in-memory relation, and that relation is distributed to each executor. I would check whether the dimension can be filtered and projected more aggressively before the join. If the resulting broadcast is still too large or memory pressure outweighs the saved shuffle, I would remove the hint and benchmark a shuffle-based join. I would compare end-to-end runtime, spill, GC, and the final row counts instead of judging by the join operator name. I would also put monitoring around dimension growth so the same hint does not become unsafe later.

**Likely follow-up:** Why is source-file size a poor estimate of a broadcast relation's memory footprint?

### Scenario 7 — The job produces thousands of tiny files

**Scenario:** A daily pipeline writes 180,000 Parquet files, many below 1 MB. The write finishes, but downstream jobs spend several minutes listing and opening files before processing much data.

**Question:** What could be causing the small-file problem, how would you fix it, and why is coalesce(1) usually a poor production solution?

**Model answer:** I would inspect output partition columns, task count, file sizes, and how data is distributed across partitions. Small input partitions can create many output files, and partitioning by a high-cardinality column can create a large number of tiny directories. I would adjust the output partitioning and, if necessary, compact existing files with a distributed rewrite. I would choose a practical target file size based on downstream scan parallelism and storage behavior, not simply try to create the fewest files. coalesce(1) forces the final write through one partition and can make a large dataset slow, memory-intensive, and vulnerable to one-task failure. After changing the layout, I would measure file count, average size, listing time, and downstream scan time.

**Likely follow-up:** What is the trade-off between fewer large files and enough files for parallel reads?

### Scenario 8 — Increasing partitions makes the job slower

**Scenario:** A 200 GB shuffle job used 200 shuffle partitions. The team changes the setting to 2,000, expecting faster execution, but runtime increases and the UI shows many tasks finishing in under a second.

**Question:** Why might more partitions make a job slower, and how would you tune the value with or without AQE?

**Model answer:** More partitions can expose parallelism when tasks are too large, but they also add task scheduling and bookkeeping overhead. I would inspect bytes per task, task duration, scheduler delay, spill, executor slots, and the final adaptive plan if AQE is enabled. If tasks are already tiny, I would reduce unnecessary task count; if tasks are huge and spill heavily, a larger partition count may help. With AQE enabled, Spark can coalesce eligible small post-shuffle partitions using runtime statistics, so I would verify the executed plan rather than tuning only from the initial plan. I would benchmark representative data and compare both runtime and output. There is no universally correct partition count: it depends on data volume, skew, available cores, and operator cost.

**Likely follow-up:** How can AQE coalescing help without reducing useful parallelism too much?

### Scenario 9 — A large input has only one effective task

**Scenario:** A raw-data stage is expected to read 600 GB, but the Spark UI shows only one task doing almost all the work. A later repartition does not improve the total runtime.

**Question:** What would you check to find the stage that limits parallelism, and what fixes apply to different causes?

**Model answer:** I would inspect the scan plan and input file layout before changing shuffle settings. The input may be one large file that the format or codec cannot split, the DataFrame may have been reduced to one partition by earlier code, or the apparent issue may actually be a hot key in a later shuffle stage. I would remove an unnecessary coalesce(1), rewrite a non-splittable source into an appropriate set of files, or treat a downstream hot key with a skew-specific strategy. A repartition after the scan cannot make the original scan parallel retroactively. I would confirm the fix by checking scan-task count and bytes per task, then measure total pipeline runtime—not just the duration of the repartition stage.

**Likely follow-up:** What evidence in the UI differentiates scan parallelism problems from post-shuffle skew?

### Scenario 10 — The job works locally but fails in the cluster

**Scenario:** A Java Spark application runs in local mode but fails on Kubernetes. One deployment reports a ClassNotFoundException; another starts but cannot write to cloud storage.

**Question:** How would you investigate these two failures without treating them as one generic Spark issue?

**Model answer:** I would classify each error separately. A class-loading failure often means the packaged application has a missing or incompatible Spark, storage, or connector dependency; I would verify Spark and JDK versions, Scala binary compatibility where relevant, and the actual runtime artifact. A storage failure points toward credentials, service-account permissions, endpoint configuration, or network access. I would inspect driver and executor logs, pod events, dependency trees, and the deployed identity. Local mode often hides these problems because it uses one machine and may rely on local credentials or classpaths. I would test a small job in staging with the same container image, service account, and storage paths used in production, then verify both data access and output permissions.

**Likely follow-up:** Why can code pass in local mode but fail during distributed task serialization?

### Scenario 11 — AQE changes the plan unexpectedly

**Scenario:** EXPLAIN shows a sort-merge join before execution. During execution, the final adaptive plan uses a broadcast join. A teammate believes Spark has ignored the team's tuning.

**Question:** Explain why this plan change can be expected, what evidence you would inspect, and when you would intervene.

**Model answer:** With Adaptive Query Execution enabled, Spark can use runtime statistics to revise eligible parts of a query plan. The build-side relation may turn out to be small enough to broadcast, or shuffle partitions may be coalesced or treated as skewed. I would inspect the final adaptive plan in the Spark UI or SQL tab, relation sizes, shuffle metrics, and stage runtimes. If the final plan is faster and correct, the change is desirable. If it is slower, I would look for inaccurate estimates, unusual data distribution, memory pressure, or a poor threshold decision and benchmark a targeted configuration change. I would not disable AQE just because the plan differs from EXPLAIN. AQE cannot fix an incorrect join predicate or eliminate a genuinely huge result.

**Likely follow-up:** Name one problem AQE can improve and one it cannot fix.

### Scenario 12 — A DataFrame pipeline is computed more than once

**Scenario:** A service builds an expensive DataFrame with two joins. It calls show() for debugging, count() for logging, and then writes the DataFrame. The Spark UI shows three separate jobs with similar upstream stages.

**Question:** Why is Spark recomputing the pipeline, and how would you reduce repeated work without introducing unnecessary caching?

**Model answer:** DataFrame transformations are lazy. Each action can trigger execution of the upstream plan, so show(), count(), and write may each recompute the joins. I would remove diagnostic actions from the production path if their results are not required. If several consumers genuinely reuse the same expensive result, I would consider persist(), materialize it with an action, and unpersist it after the last consumer finishes. Caching is not automatically beneficial: it consumes memory or disk and can cause eviction or GC pressure. I would inspect the Jobs and Stages pages to confirm recomputation, then compare runtime and resource use after the change. I would also distinguish normal re-execution from task retries, which repeat failed work, and streaming replay, which follows checkpoint recovery rules.

**Likely follow-up:** How would you decide whether an expensive intermediate DataFrame is worth persisting?

### Scenario 13 — Kafka lag grows continuously

**Scenario:** A Structured Streaming query consumes order events from Kafka. It starts with low lag, but over four hours the backlog grows from 50,000 to 4 million records. Micro-batches take 90 seconds while the trigger interval is 30 seconds.

**Question:** How would you determine whether the source, transformations, state, or sink is the bottleneck, and what can you do to catch up safely?

**Model answer:** I would compare the incoming rate with the processed rate, then inspect batch duration, input per Kafka partition, shuffle and spill, state-store time, and sink latency. If parsing or joins dominate, I would simplify those transformations; if source parallelism is insufficient, I would review Kafka partition count and whether the connector's minPartitions option is appropriate for large offset ranges. If the sink dominates, I would improve its bulk-write path or capacity before adding more Spark executors. maxOffsetsPerTrigger can cap each batch while catching up to protect the sink, but it does not increase sustainable throughput. I would estimate catch-up time from backlog and measured excess processing capacity, then alert on lag and batch duration.

**Likely follow-up:** If the pipeline processes 12,000 records per second and receives 9,000, what determines its catch-up time?

### Scenario 14 — Restarting a stream appears to duplicate output

**Scenario:** A database sink receives a micro-batch, commits its transaction, and then the driver crashes before the query records successful progress. On restart, the same source data is processed again and duplicate business rows appear.

**Question:** Why did checkpointing not prevent this, and what design would make recovery safe?

**Model answer:** A checkpoint tracks streaming progress and state, but it cannot automatically make an arbitrary database write atomic with Spark's progress recording. In this failure window, the database may have committed even though Spark must retry the batch. I would make sink writes idempotent: use a stable event ID with an upsert or uniqueness constraint, or write the batch output and a processed-batch marker in the same database transaction. I would test crashes before the transaction, during it, and after commit but before Spark records progress. I would preserve the checkpoint during recovery and reconcile existing duplicates. Deleting the checkpoint is not a duplicate-removal strategy; it changes recovery history and may replay much more data.

**Likely follow-up:** What must be in the same transaction if you use a batch marker for idempotency?

### Scenario 15 — Late events are missing from a window result

**Scenario:** A five-minute revenue aggregation uses event time and a ten-minute watermark. Customer support reports that some orders arrive 12–20 minutes late and are absent from the finalized totals.

**Question:** Explain how the watermark affects this result, how you would choose a new delay, and what you would do about events later than that delay.

**Model answer:** A watermark is based on observed event-time progress and provides a bound for state cleanup and late-data handling; it is not a timer that waits exactly ten minutes after each record arrives. A record arriving beyond the accepted lateness may no longer update the relevant window. I would measure the distribution of event lateness and confirm timestamp parsing and timezone assumptions before changing the delay. A longer watermark can accept more late data but keeps more state and increases recovery cost. I would agree the threshold with the business and, where correctness requires it, create a reconciliation path for very late records instead of assuming one finite streaming watermark will cover all delays.

**Likely follow-up:** What measurements would you use to choose a watermark delay instead of guessing?

### Scenario 16 — The watermark does not seem to reduce state

**Scenario:** A windowed aggregation has a watermark, but its state-store row count remains high and checkpoint commits keep getting slower. The query uses Complete output mode.

**Question:** What would you inspect before lowering the watermark delay, and how can output mode affect cleanup?

**Model answer:** I would inspect event-time values, whether the watermark is advancing, whether the watermark is applied to the relevant event-time column before the stateful aggregation, and the state metrics exposed by the query. I would also review the output mode and operation semantics: Complete mode requires the complete aggregate result to be emitted and does not allow old aggregation state to be removed in the same way as a watermark-based Append/Update aggregation. In multi-input queries, the global watermark policy and the slowest input can affect progress. I would first establish whether the query's semantics permit state eviction, then change output mode or query design only if it preserves the required results. Simply shortening the delay can make state smaller by dropping records the business expects to count.

**Likely follow-up:** What dashboard signals would indicate that event-time progress is stalled?

### Scenario 17 — A streaming state store grows every day

**Scenario:** A long-running stream keeps one state entry per customer ID. The number of distinct IDs grows daily, and batch duration gradually increases even though event volume per minute is stable.

**Question:** How would you decide whether this is a leak, an unbounded-state design, or ordinary growth, and what are the options?

**Model answer:** I would graph state rows and bytes, commit duration, checkpoint duration, key cardinality, and watermark progress across several days. If the query retains one entry for every key ever seen and has no valid state-removal rule, growing state may be expected rather than a leak. I would inspect whether a bounded event-time window, deduplication horizon, or explicit timeout is compatible with the business requirement. If every customer's lifetime aggregate must remain available indefinitely, I would plan for that retained state or move durable historical aggregation to a store designed for it. Adding executor memory alone cannot remove information the query is required to retain. I would test recovery duration as the state store grows, not only steady-state throughput.

**Likely follow-up:** What business requirement would justify retaining state for every customer indefinitely?

### Scenario 18 — The sink cannot keep up with Spark

**Scenario:** A streaming query transforms events in 8 seconds per micro-batch, but writing 100,000 rows to a relational database takes 55 seconds. The trigger is 20 seconds, so lag increases.

**Question:** How would you confirm the sink is the bottleneck and improve throughput without overwhelming the database?

**Model answer:** I would compare source and processed rates and measure sink duration, database CPU and I/O, lock waits, connection count, transaction size, and retry rate. I would prefer batched or bulk writes, reuse connections safely, and limit concurrent writers to what the database can sustain. A staging table followed by a database-side merge may be more efficient than many individual row operations. I would set a throughput target and test with production-like row sizes and concurrency. Adding Spark executors can worsen the situation by increasing concurrent connections and write pressure. If the permanent incoming rate exceeds what the database can handle, the system needs more sink capacity, a different write design, or a durable decoupling layer—not just Spark tuning.

**Likely follow-up:** What would you monitor to determine the database's sustainable write concurrency?

### Scenario 19 — The checkpoint directory becomes unavailable

**Scenario:** A production stream fails after a storage permission change. The checkpoint path is temporarily inaccessible, and the on-call engineer proposes deleting it and restarting the query with a new checkpoint.

**Question:** Why is that risky, what would you do first, and what is the recovery plan if the checkpoint is genuinely lost?

**Model answer:** A checkpoint stores committed progress and, for stateful queries, state needed for recovery. Creating a new checkpoint does not resume the old query from its previous offsets and state. I would first preserve logs and determine whether this is a transient credentials, permission, or storage issue. If the original checkpoint is intact, I would restore access and restart the compatible query from it. If it is lost, I would define a controlled replay boundary from retained Kafka offsets or a durable archive, rebuild state where possible, ensure the sink is replay-safe, and reconcile the output. I would also test checkpoint-storage permissions and retention policies regularly. I would never delete a production checkpoint as the first troubleshooting step.

**Likely follow-up:** What if the Kafka offsets needed to rebuild state have already expired?

### Scenario 20 — A producer changes the schema and the stream fails

**Scenario:** A producer changes the amount field from a JSON number to a string and adds a new optional field. The streaming query begins failing on some records and silently returns null for amount on others.

**Question:** How would you isolate the incompatible change, protect the metrics, and roll out a safe schema migration?

**Model answer:** I would capture representative payloads and the failing topic, partition, and offset, then compare them with the declared schema. Adding an optional field is usually straightforward; changing a field's type requires explicit conversion and validation. I would parse into a canonical schema, validate the amount field, and quarantine invalid records with a rejection reason. For a metric where dropping or nulling amounts would be harmful, I would fail or pause the affected processing rather than silently substituting zero. I would deploy a compatibility plan that accounts for old and new producers/readers, test both versions together, and only then roll out the change. If the stateful query's schema change is incompatible with its checkpoint, I would plan a controlled migration rather than deleting the checkpoint casually.

**Likely follow-up:** How would you support old and new producer versions during a gradual rollout?

### Scenario 21 — One Kafka message cannot be parsed

**Scenario:** A stream processes valid order events until one malformed JSON record repeatedly causes the micro-batch to fail. Restarting the query does not help because it encounters the same record again.

**Question:** How would you keep the pipeline moving without silently losing business data, and how would you make the bad-record path observable and replayable?

**Model answer:** I would capture safe identifying metadata—topic, partition, offset, Kafka timestamp, and event ID if available—and inspect the payload to determine whether this is malformed JSON, a schema mismatch, or a producer bug. If business policy allows quarantine, I would write the raw payload and a rejection reason to a durable dead-letter location, with idempotent keys so retries do not create endless duplicates. If the invalid data could make a critical metric wrong, I would pause or fail visibly rather than silently defaulting amount to zero. I would monitor rejection counts and alert on a sudden increase. After fixing or transforming the record, I would replay it using stable identifiers and reconcile the resulting aggregates.

**Likely follow-up:** How would you replay quarantined records without processing the same corrected event twice?

### Scenario 22 — Batch and streaming totals disagree

**Scenario:** The live stream reports daily revenue of ₹98 lakh. A nightly batch recomputation from the same business source reports ₹1.02 crore. Both jobs complete successfully.

**Question:** How would you investigate the discrepancy systematically, and how would you prevent the same class of mismatch from recurring?

**Model answer:** I would first make the comparison fair: use the same source boundary, date/timezone definition, currency and null rules, deduplication key, and business filters. Then I would reconcile row counts and amounts at each stage: raw events, parsed rows, filtered rows, post-join rows, deduplicated records, and final windows. I would inspect late events that the batch includes but the finalized streaming window no longer accepts, duplicate IDs, join multiplicity, and window-boundary handling. Spark is not necessarily inconsistent; the two computations may be applying different rules. I would formalize the business contract in shared transformations or tests, add edge cases for late and duplicate events, and create a reconciliation job that explains variance before publishing numbers.

**Likely follow-up:** Which three test cases would you add first to validate batch/stream parity?

### Scenario 23 — Database writes must be idempotent

**Scenario:** A foreachBatch writer sends 20,000 rows to a database. The database commits them, but the client loses the response and Spark retries the batch. The table must not contain duplicate events.

**Question:** Design a retry-safe write pattern and explain how you would handle updates that can arrive out of order.

**Model answer:** I would give each business event a stable unique identifier and enforce it with a primary key or unique constraint, then use an upsert or merge so a retry does not create a second row. If I use a batch marker, the data changes and marker must commit in the same transaction; otherwise a crash between them can still cause duplicate effects. For mutable current-state rows, I would include a monotonic version or source sequence and update only when the incoming version is newer. I would test failures before commit, after the commit, and when the commit succeeds but the client times out. A Spark batch ID can help with batch-level idempotency, but a stable event key is still useful for partial writes and replay.

**Likely follow-up:** What should the database do if the same event ID arrives with a different payload?

### Scenario 24 — One hot key dominates a join

**Scenario:** A multi-tenant analytics job joins usage events to tenant metadata. One enterprise tenant contributes 35% of the events, and a few join tasks run 20 times longer than the rest.

**Question:** How would you prove the skew is caused by a hot tenant, and what approaches would you consider before using salting?

**Model answer:** I would compare task-level shuffle sizes and durations, then measure tenant frequencies on both join inputs and validate key uniqueness. If the large tenant produces far more rows, AQE skew-join handling may split eligible skewed partitions. Depending on the semantics, I might process the hot tenant separately, pre-aggregate events before joining, or use salting. Salting requires care: the salted join key must be created consistently, and the matching smaller-side rows may need to be replicated so no valid matches are lost. I would avoid salting until I know why the skew exists and whether earlier aggregation is possible. After the change, I would compare row counts, key-level totals, and the distribution of task durations.

**Likely follow-up:** What changes if both sides of the join have a highly skewed distribution?

### Scenario 25 — Garbage collection consumes much of the runtime

**Scenario:** A Spark application has enough executor cores, but a job that used to run in 15 minutes now runs in 40 minutes. The UI shows heavy garbage collection on every executor, especially during a join and aggregation stage.

**Question:** What evidence would you collect and which code or plan changes would you try before increasing heap size?

**Model answer:** I would correlate GC time with the stages, heap occupancy, spill, broadcast size, cache usage, and task input sizes. An oversized broadcast or cache can put pressure on every executor; object-heavy RDD transformations and repeated conversion to Java objects can create allocation overhead. I would reduce columns and rows early, prefer built-in DataFrame expressions where they fit, remove unused persisted data, and address skew that creates very large task working sets. I would consider heap or container changes only after measuring the actual memory requirement. A larger heap may reduce collection frequency but can also produce longer pauses and will not fix unbounded state or excessive data movement. I would compare runtime, GC time, spill, and output correctness after each targeted change.

**Likely follow-up:** Why might increasing executor heap make pause times longer without solving the root cause?

### Scenario 26 — Shuffle fetch failures repeat

**Scenario:** Several reduce tasks fail with FetchFailedException. Retrying sometimes succeeds, but failures return whenever the job runs on the same worker node.

**Question:** How would you isolate whether the failure is transient or infrastructure-related, and what should be fixed before changing retry settings?

**Model answer:** I would correlate failed fetches with executor IDs, worker hosts, disk space, network errors, and cluster events. A source executor may have disappeared and taken local shuffle files with it, or the worker may have disk, network, or stability problems. If errors repeatedly point to one host, I would investigate and remove or repair that host rather than merely increasing retries. I would review how the deployment handles executor loss and shuffle data, then rerun the same workload to see whether failures follow the node. Retry settings can help with transient failures, but cannot restore missing data if the underlying failure repeats. I would verify that the job completes without repeated fetch errors and that the final output is complete.

**Likely follow-up:** When are task retries useful, and when do they merely hide a recurring infrastructure defect?

### Scenario 27 — The Spark UI shows too many tiny tasks

**Scenario:** A SQL stage creates 25,000 tasks. Most finish in less than one second, and scheduler delay is a significant fraction of total stage time.

**Question:** How do you determine whether these are caused by input files or shuffle partitioning, and how do you tune the correct layer?

**Model answer:** I would identify the stage and inspect its input and physical plan. Many tiny scan tasks can come from a large number of small source files; many tiny reduce tasks can come from excessive shuffle partitions. These are different controls. For the file problem, I would compact files or review input file-packing behavior. For the shuffle problem, I would tune the shuffle partition setting and inspect AQE coalescing. I would compare task size, scheduler delay, spill, and downstream parallelism before and after the change. I would not collapse everything into one task, because the next run could create a small number of oversized tasks and lose parallelism. The goal is balanced task sizes for the workload, not the minimum possible task count.

**Likely follow-up:** Which UI clues tell you whether the excess tasks belong to the scan or the shuffle stage?

### Scenario 28 — A high-cardinality aggregation exhausts memory

**Scenario:** A telemetry job groups 2 TB of events by request ID. Nearly every row has a different request ID, and the aggregation spills heavily while building tens of millions of output groups.

**Question:** Why may increasing heap or shuffle partitions not be enough, and what alternatives would you discuss with the product team?

**Model answer:** The key issue may be the number of distinct groups, not simply a poor Spark setting. Partial aggregation helps when many input records share keys, but offers little reduction when almost every request ID is unique. I would inspect cardinality, skew, shuffle volume, and spill, then filter or project earlier and choose partitions that distribute the data sensibly. If the report only needs counts by service and minute, grouping by request ID is unnecessary detail and a coarser aggregation can reduce work dramatically. If every request-level output is truly required, the cost is real and the output/storage design must accommodate it. I would explain this trade-off to the product team and validate that any reduced-granularity result still answers the business question.

**Likely follow-up:** Why can a groupBy return almost as many rows as the input?

### Scenario 29 — A stream must recover after several hours offline

**Scenario:** A Kafka-to-Spark stream is down for six hours. Kafka retention is 24 hours, the checkpoint is available, and the downstream database can sustain only 1.5 times the normal incoming rate.

**Question:** How would you plan the recovery so events are not lost and the database is not overwhelmed?

**Model answer:** I would first confirm the checkpoint is intact and that Kafka still retains every required offset. I would estimate backlog size and calculate catch-up capacity from measured processing rate minus incoming rate. Since the database can sustain only 1.5 times normal traffic, I would not assume Spark can catch up faster than that without overloading it. I would use controlled per-trigger input, scale only the parts that are actually limiting throughput, and ensure writes are idempotent because recovery can replay data. If the required offsets have expired, I would recover from a durable archive or backfill and reconcile the missing range. I would monitor lag, processing rate, sink health, and estimated time to clear the backlog throughout recovery.

**Likely follow-up:** Why must Kafka retention cover both the maximum outage and the time needed to catch up?

### Scenario 30 — Kafka volume increases tenfold

**Scenario:** A stream normally receives 8,000 events per second. A product launch raises traffic to 80,000 per second; batch duration grows beyond the trigger interval and the backlog grows even after adding executors.

**Question:** How would you find the sustainable throughput limit and decide whether more Kafka partitions, Spark capacity, or sink capacity is needed?

**Model answer:** I would measure the incoming and processed rates and profile each part of the pipeline: Kafka partition distribution, parsing, joins and shuffles, state updates, and sink latency. If only a few Kafka partitions are active, source parallelism may be limited; adding Kafka partitions can help but can change key distribution and ordering assumptions. If transformations dominate, I would optimize their plans or increase usable parallelism. If the sink is saturated, more executors can make it worse by creating more concurrent writes. I would load-test sustained traffic as well as short spikes and measure the spare capacity available for backlog recovery. The design should process consistently faster than the expected arrival rate, not merely survive one short benchmark.

**Likely follow-up:** What ordering assumptions could be affected by increasing a Kafka topic's partition count?

### Scenario 31 — Events must be deduplicated reliably

**Scenario:** The producer retries publishing after a timeout, so the same event appears twice in Kafka. The dashboard counts duplicate order-status changes, but a single order can legitimately have several different status events.

**Question:** What deduplication key would you use, how would you bound state, and what guarantee would you explain to the business?

**Model answer:** I would first confirm that each logical event has a stable event ID. Order ID alone is insufficient when one order can emit multiple legitimate status changes. I would deduplicate by the event ID and configure a watermark or other supported state-retention boundary based on the maximum duplicate arrival delay the business is willing to handle. Keeping IDs longer consumes more state, while bounded deduplication cannot promise to suppress a duplicate that arrives after the retention horizon. I would also make the sink idempotent as a second layer of protection. I would measure duplicate rates and late arrivals, state the guarantee in business terms, and add a reconciliation path for out-of-horizon duplicates when accuracy requirements demand it.

**Likely follow-up:** What exactly does the system promise for duplicates that arrive after the watermark horizon?

### Scenario 32 — A historical backfill must transition to live events

**Scenario:** The team loads six months of order history from object storage and then enables a Kafka stream for new events. Orders continue arriving during the backfill, so a simple “finish batch, then start stream” sequence may miss events.

**Question:** Design a cutover that avoids gaps, and explain how you would handle overlap and prove the transition is complete.

**Model answer:** I would define an explicit handoff boundary using source offsets or another durable, well-defined position. I would ensure the historical and live paths use the same schema and transformation rules. Where practical, I would allow controlled overlap and deduplicate using stable event IDs; replayable overlap is usually safer than an unobserved gap when writes are idempotent. I would not rely only on wall-clock time unless the source guarantees make that boundary reliable. I would record the boundary, reconcile counts and key-level totals on both sides of it, and test failure/restart cases during cutover. The exact procedure depends on the historical source and Kafka retention, but the principle is to make the boundary explicit and verifiable.

**Likely follow-up:** What evidence would convince you that no events were lost at the handoff boundary?

### Scenario 33 — The Parquet schema changes over time

**Scenario:** A date-partitioned Parquet dataset contains older files where amount is an integer and newer files where amount is a string. A query either fails during schema resolution or produces null values after casting.

**Question:** How would you restore compatibility without hiding invalid data, and what tests would you add?

**Model answer:** I would inspect schemas and representative records from both old and new partitions rather than assuming the directory has one consistent schema. I would define a canonical schema and apply explicit, validated conversion rules when reading each version. Valid numeric strings can be converted, while invalid values should be counted or quarantined rather than silently treated as zero or null. Schema merging is not a replacement for a data contract and can add planning cost or mask a producer problem. I would test old and new partitions together, check row counts and null rates, and verify aggregate totals before publishing. I would also add a compatibility test to the ingestion process so future type changes are caught before a large historical query fails.

**Likely follow-up:** Why is a successful permissive cast not proof that schema evolution was safe?

### Scenario 34 — A Spark application must run on Kubernetes

**Scenario:** A Spark job runs in Kubernetes. Several executors are OOM-killed, and after a pod restart the stream cannot access its checkpoint path. Increasing the JVM heap makes the container kills more frequent.

**Question:** What would you inspect, and how would you separate container sizing from checkpoint-access issues?

**Model answer:** I would treat this as two issues. For the OOM kills, I would inspect pod events, container memory usage, executor logs, heap occupancy, and memory overhead. Container usage includes native and direct-buffer memory as well as the JVM heap, so raising heap without increasing the container limit can leave less headroom and trigger more kills. I would size the requests, limits, heap, and overhead using measurements and realistic workloads. For checkpoint access, I would check the service account, storage credentials, network path, and permissions using the identity actually assigned to the pod. I would verify that restarts resume from the existing checkpoint before calling the recovery complete. A successful local run does not validate Kubernetes resource limits or service identity.

**Likely follow-up:** Which evidence distinguishes a Java heap OOM from a Kubernetes container OOM kill?

### Scenario 35 — One sink succeeds while another fails

**Scenario:** One Spark micro-batch writes order events to Kafka and then indexes them into Elasticsearch. Kafka accepts the batch, but Elasticsearch times out after only some documents are indexed. The next retry risks duplicates.

**Question:** How would you design partial-failure recovery and keep the two destinations consistent enough for the business?

**Model answer:** I would not claim these two independent writes are one atomic transaction. A Spark checkpoint does not commit Kafka and Elasticsearch together, so the application needs a partial-failure strategy. I would assign deterministic document IDs or use version-aware upserts in Elasticsearch, and make downstream Kafka consumers idempotent using stable event IDs. I would record per-sink progress where appropriate, retry the failed destination safely, and monitor and reconcile the sinks independently. For a simpler architecture, I would write once to a durable source of truth such as Kafka or a reliable table and let separate consumers build each destination. I would also test the case where Kafka succeeds and Elasticsearch partially succeeds, then verify retries converge without missing or duplicating business events.

**Likely follow-up:** How would you rebuild Elasticsearch after a long outage without stopping the Kafka pipeline?

### Scenario 36 — The database becomes overloaded after scaling Spark

**Scenario:** A team doubles Spark executors to reduce batch duration. Spark writes become faster initially, but database CPU reaches 95%, lock waits increase, connection pools saturate, and retries cause even more load.

**Question:** How would you restore stability and increase throughput without losing reliability?

**Model answer:** I would confirm that the database is now the bottleneck by correlating Spark sink duration with database CPU, I/O, lock waits, active sessions, and retry rates. I would limit concurrent writers to a measured sustainable level, use batched or bulk writes, and consider a staging table followed by a database-side merge. Retries should be bounded and use backoff so they do not amplify overload. If the sustained incoming rate is greater than the database can handle, the long-term answer is to increase sink capacity, change the write pattern, or decouple ingestion—not keep adding executors. I would restore service first, then run controlled load tests to find the throughput/concurrency point where database latency and error rates remain healthy.

**Likely follow-up:** How would you choose a maximum number of concurrent Spark writers?

### Scenario 37 — A report's totals do not match the source

**Scenario:** The Spark job exits successfully, but its daily revenue is 3% higher than the order system. No task failed, and the output files are readable.

**Question:** How would you find where the extra revenue enters the pipeline and prevent publishing incorrect totals again?

**Model answer:** I would reconcile counts and amounts at each boundary: raw records, parsed records, filtered events, rows after each join, deduplicated records, and the final aggregation. I would inspect duplicate event IDs, null or defaulted amounts, timezone boundaries, join-key uniqueness, and late-arriving records. A duplicate dimension key can multiply a fact row, so I would validate expected uniqueness on dimension tables. I would add data-quality gates for critical checks, such as unique key counts, unmatched-key ratios, and acceptable total variance, and stop publication if the agreed threshold is exceeded. After fixing the cause, I would replay affected data idempotently and compare against an independent reconciliation source. A successful Spark exit proves the computation ran, not that the business logic was correct.

**Likely follow-up:** Which checks would you make blocking before publishing a financial report?

### Scenario 38 — A UDF makes a query slower

**Scenario:** A DataFrame transformation that previously ran in 12 minutes now takes 25 minutes after Java UDFs were introduced to normalize a status field and calculate a category.

**Question:** How would you prove whether the UDFs are responsible and what would you try before rewriting them?

**Model answer:** I would compare the physical plan and stage metrics before and after the change and benchmark the UDF work on representative data. Spark can optimize and reason about built-in SQL expressions, while a UDF is largely opaque to those optimizations. If built-in functions can express the same logic, I would replace the UDF and compare runtime, null behavior, and output values. I would also apply expensive logic after filters and projections so it processes fewer rows and columns. If a UDF is needed for specialized logic, I would keep it but test allocation, serialization, and error handling. I would not remove a UDF purely for style: the decision should be backed by plan evidence and a correctness-preserving benchmark.

**Likely follow-up:** Which query optimizations can become harder when a calculation is hidden inside a UDF?

### Scenario 39 — A Parquet query scans far too much data

**Scenario:** A table is partitioned by event_date. A report filters to one day, but Spark scans hundreds of gigabytes from many dates and takes 40 minutes.

**Question:** How would you check whether partition pruning and predicate pushdown are working, and what would you change?

**Model answer:** I would inspect the physical scan in the executed plan and look at PartitionFilters, PushedFilters, selected columns, files read, and bytes scanned. PartitionFilters indicate filters used to eliminate partition directories; PushedFilters indicate predicates passed to the data source when supported. A filter written through a transformation of the partition column may prevent effective pruning, so I would use a compatible predicate directly on event_date and project only required columns. I would also confirm that the actual directory layout matches the partitioning the query assumes. If many workloads need a different layout, I would consider a controlled rewrite. I would compare files and bytes read before and after the change and validate the report totals.

**Likely follow-up:** What is the difference between partition pruning and predicate pushdown?

### Scenario 40 — Updates arrive out of order

**Scenario:** An order has status version 12, “SHIPPED.” Because of a retry and delayed delivery, an older version 10, “PROCESSING,” arrives later and overwrites the current-state database row.

**Question:** How would you stop stale events from replacing newer state while still preserving the audit history?

**Model answer:** I would make ordering explicit in the event contract instead of trusting event time alone, because producer clocks can drift and messages may be delayed. Each update should carry a monotonic version or source sequence. The sink would update the current-state row only when the incoming version is greater than the stored version; duplicate versions should be handled deterministically and conflicting payloads should be surfaced. If the business needs a full audit trail, I would store all accepted events in an append-only history and maintain the current-state view separately. A watermark helps manage event-time state in Spark, but it does not define which business update wins. I would test duplicates, delayed old versions, and conflicting versions explicitly.

**Likely follow-up:** How should the system respond if two different payloads claim to have the same version?

### A simple way to explain a production incident

In an interview, structure your answer in this order: state the symptom and impact, identify the first evidence you would inspect, explain what that evidence would distinguish, select a fix that matches the diagnosis, and define how you will verify the output. Mention a trade-off whenever the fix changes memory use, latency, state retention, ordering, or delivery semantics. This is more convincing than listing Spark settings without explaining why they fit the situation.

~~~mermaid
flowchart TD
  A[State the symptom and business impact] --> B[Find the slow stage or failing batch]
  B --> C[Inspect task distribution, input, shuffle, spill, GC, state, and sink latency]
  C --> D{What does the evidence indicate?}
  D -->|Excess data read| E[Check file layout and partition pruning]
  D -->|Uneven task work| F[Check key skew and join cardinality]
  D -->|Memory pressure| G[Check broadcast, cache, and partition size]
  D -->|Growing backlog| H[Check source rate, state, and sink capacity]
  E --> I[Apply one targeted change]
  F --> I
  G --> I
  H --> I
  I --> J[Compare runtime, data quality, and business totals]
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

**Model answer:** Not necessarily. The checkpoint allows Spark to recover its recorded source progress and state, but the database write can succeed before Spark has durably recorded the batch as complete. On retry, that external write may be attempted again. I would make database writes idempotent with a stable event ID and an upsert or uniqueness constraint, or use a database transaction that commits the output and a processed-batch marker together. If the same pipeline also republishes records to Kafka, its Kafka sink is at-least-once and may duplicate output, so downstream consumers need their own idempotency or deduplication strategy.

I would find the failure boundary using batch progress, source offsets, application logs, and database commit records. If using a batch marker, I would commit the marker and the data writes in one database transaction. I would test failure before the write, during the write, and after commit. I would preserve the checkpoint and avoid pretending that starting from a fresh checkpoint fixes duplicates.

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
