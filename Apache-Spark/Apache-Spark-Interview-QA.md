# Apache Spark & Structured Streaming Interview Handbook

**Audience:** Java backend engineers with 5+ years of experience preparing for senior backend interviews.  
**Focus:** Spark fundamentals, Spark SQL/DataFrames, execution internals, partitioning and shuffles, performance, Java APIs, Structured Streaming, Kafka, and production reliability.  
**Version note:** Examples use established Spark 3.x/4.x DataFrame and Structured Streaming APIs. Connector coordinates, defaults, supported operations, AQE rules, and state behavior vary by release; validate against the Spark distribution and connector version deployed.

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

In client mode, the driver runs in the submitting environment; convenient for interactive work, but the client is often unsuitable as a production host. In cluster mode, the driver runs in the cluster-managed environment. The choice affects logs, network reachability, credentials, dependency distribution, and restart behavior, not the logical meaning of DataFrame expressions.

---

## 3. RDDs, DataFrames, and Datasets

### Q14. What is an RDD?

An RDD (Resilient Distributed Dataset) is an immutable, partitioned collection with lineage information. Transformations create new RDDs; lineage describes how lost partitions can be recomputed.

~~~java
JavaRDD<String> lines = sc.textFile("s3a://analytics/logs/*.txt");
JavaRDD<String> errors = lines.filter(line -> line.contains("ERROR"));
long errorCount = errors.count();
~~~

The filter is lazy; count triggers execution. Collecting all matching lines would move them to driver memory and may cause an out-of-memory failure.

### Q15. Compare RDDs, DataFrames, and Datasets.

A DataFrame is a distributed table with named columns and a schema; in Java it is represented as Dataset<Row>. Typed Datasets use JVM types and encoders and are particularly common in Scala. RDDs expose lower-level record operations but less relational optimization.

Prefer DataFrames for structured ETL and SQL because Spark can inspect expressions and optimize them. Use RDDs when a specialized unstructured transformation genuinely needs the lower-level API. Do not switch to RDDs merely because a query is slow; first inspect the plan and data distribution.

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

In production, the submission environment normally supplies cluster configuration. Avoid hard-coding a local master. Use explicit schemas for CSV/JSON when correctness depends on types.

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

These are immutable transformations; they describe a plan rather than mutating the original Dataset. The aggregation produces one row per customer. Import functions from org.apache.spark.sql.functions.

### Q20. Explain select, filter, withColumn, and drop.

Select projects or derives output columns; filter retains rows matching a predicate; withColumn adds or replaces a column; drop removes a column. Each returns a new Dataset. Hundreds of chained withColumn calls can create an unwieldy plan; a select with several expressions can be clearer.

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

GroupBy collapses each group into aggregate rows. A window computes over related rows while preserving individual rows, for example ranking orders within a customer.

~~~java
WindowSpec w = Window.partitionBy("customer_id")
    .orderBy(col("order_ts").asc());
Dataset<Row> ranked = orders.withColumn("order_rank", row_number().over(w));
~~~

Windows can require repartitioning and sorting and may be costly for high-cardinality or skewed keys. Use deterministic tie-breakers if ordering matters.

### Q24. union versus unionByName?

Union combines rows by column position and does not deduplicate. UnionByName matches columns by name and can fill missing columns with null where the API supports that option. Normalize types and schemas before combining evolving datasets; explicitly request deduplication if required.

### Q25. How do you query a DataFrame using SQL?

~~~java
orders.createOrReplaceTempView("orders");
Dataset<Row> result = spark.sql(
    "SELECT customer_id, SUM(amount) AS revenue " +
    "FROM orders WHERE status = 'PAID' GROUP BY customer_id");
~~~

SQL and DataFrame expressions generally use Spark SQL planning and optimization. Choose the syntax that communicates the logic most clearly.

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

CSV is text and needs typing; JSON handles nested records; Parquet is columnar and well suited to analytical reads because it can skip unused columns and use statistics. partitionBy creates directory-level partitioning; it does not promise one file per partition or one file overall. Write atomicity depends on the filesystem and committer.

### Q27. How do nested types and explode work?

Spark supports arrays, maps, and structs. Dot notation selects nested fields; explode turns array elements into rows. Exploding large arrays multiplies row counts and can become a major cost. Preserve parent identifiers and verify that the downstream operation truly needs one row per element.

### Q28. How should dates and timestamps be handled?

Distinguish an instant from local wall-clock time. Standardize time zones and parsing formats, and be explicit about session timezone when correctness depends on it. In streaming, event time controls windows and watermarks; processing time is when Spark handles a record. Incorrect timezone parsing can put records in the wrong day while the query still succeeds.

---

## 5. Lazy evaluation and execution plans

### Q29. Transformations versus actions?

Transformations describe new datasets, such as filter, select, map, and groupBy. Actions request a result or side effect, such as count, collect, show, or write. A write is an action too, and can process terabytes despite looking like one short method call.

### Q30. Why is collect dangerous?

Collect transfers every output row to driver memory. Use it only when the result is known to be small. For large results, aggregate in Spark or write to distributed storage; for inspection use take(n) or a bounded sample. Show displays a limited number of rows by default but still triggers work. Foreach executes on distributed rows/partitions and is not a transactional sink by itself.

### Q31. How do you inspect a query plan?

Use explain(true) or explain("formatted") where supported. Inspect scans, filters, exchanges, sorts, join operators, and adaptive-plan details. The plan shows intended execution, not actual task performance; pair it with runtime metrics in the Spark UI.

### Q32. Why can a transformation pipeline execute multiple times?

Each action may require Spark to execute the upstream plan. Calling count and then writing the same uncached DataFrame can scan and transform the source twice. Remove unnecessary actions or persist expensive results that are reused. Cache population itself costs resources, so measure the benefit.

---

## 6. Partitions, shuffles, and skew

### Q33. What is a partition, and how does it relate to parallelism?

A partition is a chunk of distributed data processed as a task unit for many operations. A stage with 100 partitions can expose about 100 tasks of parallel work, subject to dependencies and resources. Executor count is not partition count: executors can run multiple task slots over time.

Too few partitions limit parallelism and create large tasks; too many tiny partitions increase scheduling overhead and small files. The right count depends on data size, operation, cluster capacity, and source layout.

### Q34. What causes a shuffle and why is it expensive?

A shuffle redistributes records between partitions, often to bring matching keys together. Joins, groupBy, sorting, and repartitioning commonly trigger one. Shuffle data may be serialized, written to disk, transferred over the network, fetched, and deserialized. Reduce data before the shuffle, choose appropriate join strategies, and address skew rather than trying to eliminate every required exchange.

### Q35. repartition versus coalesce?

Repartition normally shuffles data and can increase or decrease partitions while redistributing records. Coalesce can reduce partitions without a full shuffle, but may create uneven partitions. Use coalesce after filtering when reducing task count is sensible; use repartition when even distribution is needed. Neither guarantees balanced work for a hot key.

### Q36. What does spark.sql.shuffle.partitions control?

It sets the default partition count for many Spark SQL shuffle operations; defaults vary by Spark release. Too high creates tiny tasks; too low creates large tasks and insufficient parallelism. AQE can coalesce shuffle partitions or alter eligible join decisions at runtime. Tune using shuffle volume and task metrics, not a universal magic number.

### Q37. What is data skew and how do you diagnose it?

Skew is uneven data distribution. A hot customer ID might represent 40% of events, leaving one task much slower than peers. Compare task durations, input bytes, shuffle read, spill, and key frequencies in the Spark UI. First identify whether the cause is a hot key, one large file, or an unhealthy node.

Mitigations include filtering, pre-aggregation when semantics allow, AQE skew-join handling, or salting. Adding executors will not fix a hot key if work still concentrates in one partition.

### Q38. Explain salting and its risks.

Salting appends an artificial bucket to a hot key so work is distributed across multiple keys. Aggregations commonly need a first aggregation by (key, salt) and a second aggregation by original key. Joins may require replicating the smaller side across salt values. This adds work and can multiply rows; incorrect replication or merging produces wrong results. Use it only after measuring skew and designing the merge correctly.

### Q39. File partitioning versus Spark partitions?

Directory partitioning such as year=2026/month=10 is a storage layout that can support partition pruning. Spark execution partitions are task-sized chunks. One directory partition can contain many files, and Spark may split or combine file inputs. Repartitioning in memory does not automatically reorganize source directories.

### Q40. What is partition pruning?

Partition pruning skips storage directories whose partition-column values cannot satisfy a filter. Predicate pushdown can filter rows within files. Functions or casts on partition columns can prevent pruning in some query shapes; inspect scan details and actual input bytes to verify.

---

## 7. Joins and aggregations

### Q41. Which join strategies should you know?

Broadcast hash join distributes a sufficiently small relation to executors, avoiding a shuffle of the large side. Sort-merge join commonly shuffles and sorts both sides by key and is common for large equi-joins. Shuffled hash join redistributes data by key and builds a hash table for one side of each partition when the planner considers it suitable.

Spark's choice depends on statistics, join type, hints, configuration, and AQE. Inspect the physical and final adaptive plans rather than memorizing a simplistic rule.

### Q42. When is broadcast useful, and what can go wrong?

Broadcast a small dimension table when the materialized relation can safely fit on executors. It can remove a large shuffle. However, the in-memory broadcast size is not the compressed source file size. A too-large broadcast can consume memory, cause GC, or fail during exchange. Check actual size and executor pressure; a hint is not a substitute for evidence.

### Q43. What is Adaptive Query Execution?

AQE uses runtime statistics to revise eligible parts of a query plan. Depending on Spark version and configuration, it can coalesce shuffle partitions, handle certain skewed joins, and change some join strategies. It cannot fix incorrect join logic, huge output cardinality, or every skew pattern. Verify it is enabled and inspect the final adaptive plan.

### Q44. Why can distinct be expensive?

Deduplication requires identifying equal rows across partitions and often causes a shuffle. dropDuplicates(keys) better expresses key-based deduplication but still requires distributed coordination. In streaming, deduplication also needs state and a policy for how long IDs are retained.

### Q45. Why can an aggregation run out of memory?

High-cardinality grouping creates many distinct keys and intermediate state. Skew can make one task much larger than peers. Spark often performs partial aggregation before shuffle, but that cannot eliminate the cost of tracking many unique groups. Inspect cardinality, spill, and skew; reduce rows earlier or change the aggregation grain only if business semantics allow it.

---

## 8. Catalyst and query optimization

### Q46. Explain Spark SQL's planning pipeline.

DataFrame expressions or SQL become a logical plan. Spark resolves names/types and validates operations, then Catalyst applies eligible optimizations such as predicate pushdown, projection pruning, and constant folding. The planner chooses a physical plan with concrete scans, exchanges, joins, and aggregates. AQE may revise parts of the physical plan using runtime statistics. The exact rules depend on query, source, version, and configuration.

### Q47. Predicate pushdown versus projection pruning?

Predicate pushdown tries to filter close to the source; projection pruning avoids reading unused columns. Parquet can benefit because Spark may read only required column chunks and skip data using metadata. Not all predicates can be pushed down, and a UDF can block optimization. Verify the plan and input metrics.

### Q48. What are Tungsten and whole-stage code generation?

Tungsten describes Spark's execution-efficiency work, including memory management and binary processing. Whole-stage code generation combines eligible operators into generated JVM code to reduce per-row overhead. Internals evolve and not every query is fully code-generated. Opaque UDFs or unsupported operators can limit optimization.

### Q49. What can the optimizer not fix?

It cannot infer missing business semantics, eliminate unavoidable output volume, guarantee balanced keys, or make a slow external sink fast. It may not optimize opaque UDF logic or repair an incorrect join. Expose intent through built-in expressions, design sensible data layouts, and validate actual execution.

---

## 9. Memory, serialization, and garbage collection

### Q50. Driver memory versus executor memory?

Driver memory supports planning, scheduling metadata, task results, and objects returned to the application. Executor memory supports task execution, shuffle, cached blocks, and broadcasts. Container memory can include off-heap and native use beyond JVM heap. A driver OOM from collect is not fixed by increasing executor memory; a skewed join OOM needs task-size and join-strategy investigation.

### Q51. Execution memory versus storage memory?

Execution memory supports joins, sorts, and aggregations; storage memory supports cached data. Spark's unified memory manager allows these uses to share a region, and cached blocks may be evicted under pressure. Caching competes with execution memory and can increase spill. Cache only data that is expensive to recompute and reused.

### Q52. Why does Spark spill to disk?

When an operation cannot keep working data in memory, Spark may spill intermediate data to disk rather than immediately failing. Spill avoids some OOMs but adds I/O and often signals large partitions, skew, high cardinality, or memory pressure. Some spill can be normal; large repeated spill merits investigation.

### Q53. Java serialization versus Kryo?

Java serialization is convenient but can be costly for object-heavy RDD workloads. Kryo can reduce serialization overhead when correctly configured. DataFrame SQL execution often uses internal representations, so changing the serializer is not a universal fix for slow SQL. Measure serialization time and size before tuning.

### Q54. How do you diagnose excessive garbage collection?

Use executor metrics/logs to identify GC time as a share of task runtime. Check object-heavy transformations, broadcasts, cached data, skew, and repeated allocation. Remedies may include SQL expressions instead of object-heavy RDD code, reducing data, fixing skew, or removing unnecessary cache. Increasing heap without understanding allocation can merely delay failure and lengthen pauses.

### Q55. What is executor memory overhead?

In containerized deployments, overhead covers non-heap/native memory in addition to JVM heap, including direct buffers and process overhead. A container can be killed for exceeding its total limit without a conventional Java heap OOM. Review JVM logs and platform events to distinguish the failure types.

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

Lineage records the transformations and dependencies used to compute data. If a partition is lost, Spark can recompute it from ancestors rather than replicating every intermediate. Long lineage can make recomputation expensive, and source availability matters. Checkpointing cuts lineage; caching reduces recomputation during normal execution.

---

## 11. Performance tuning and Spark UI

### Q59. How should you investigate a slow job?

Identify the slow job/stage, compare task durations, inspect input bytes, shuffle, spill, GC, executor time, and query plan, then validate key distribution and file layout. Change one likely cause and compare equivalent inputs. A job may be slow due to skew, full scans, tiny files, shuffle, resource contention, or a slow sink; each needs a different remedy.

### Q60. What does the Spark UI tell you?

Jobs and Stages show progress, task distribution, failures, and shuffle metrics. Executors show active tasks, storage, GC, and failures. SQL pages can connect query plans with runtime metrics. Compare median and maximum task duration, not just overall runtime. Configure event logging/history server for completed applications.

### Q61. Why are small files harmful?

Many tiny files increase listing, open, metadata, and task scheduling overhead. Compaction or a controlled rewrite can help. Output size depends on partitioning, volume, compression, and writer behavior; forcing one partition for a large dataset can create a bottleneck.

### Q62. What is dynamic allocation?

Dynamic allocation requests/releases executors based on workload demand when supported and configured. It affects available resources, while AQE adjusts eligible query plans. Executor churn and shuffle preservation matter; dynamic allocation does not solve skew or a single-threaded sink.

### Q63. How should tuning experiments be performed?

Change one factor at a time, use representative input, record baseline and after metrics, and verify result correctness. Compare end-to-end runtime and cost, not only one stage. Small samples can hide skew and cardinality effects. Keep a rollback plan for production changes.

---

## 12. Files and storage

### Q64. Why is Parquet common for analytics?

Parquet is columnar. Queries needing only a few columns can avoid reading others, and metadata can help skip data. Compression is often effective for typed column values. JSON is useful for raw/nested input, CSV is interoperable but weakly typed, and ORC is another columnar format. Layout and schema discipline remain important.

### Q65. Why does a Spark write create a directory of part files?

Spark writes distributed output from multiple tasks, so the destination commonly contains multiple part files and possibly metadata/commit files. File count depends on tasks, writer behavior, and partitioned output. Downstream readers generally read the directory as a dataset. Avoid coalesce(1) for large output because it funnels data through one task.

### Q66. What is schema evolution?

Schema evolution is a change to fields or types over time. File formats and table formats differ in how they reconcile schemas; a directory of mixed-schema files does not guarantee automatic correct merging. Define compatibility rules, use explicit schemas at critical boundaries, and test old and new data together.

### Q67. What should you know about object storage?

Object storage is scalable and durable, but listing and commit behavior differ from local filesystems/HDFS. Many small objects can be expensive, and atomic rename assumptions may not hold. Use the recommended connector and committer for the environment and test failure behavior.

---

## 13. Structured Streaming

### Q68. What is Structured Streaming?

Structured Streaming is Spark's scalable stream-processing engine built on DataFrame/Dataset APIs. You describe a query over an unbounded input table; Spark incrementally processes new records and tracks progress using checkpoints. Stateful operations maintain recoverable state. The default engine uses micro-batches, and streaming has constraints that do not apply identically to batch queries.

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

Create a streaming DataFrame with readStream, transform it, and call writeStream.start(). Each trigger processes available input, updates progress and state, and commits according to source/sink protocols. Restart with the same compatible checkpoint to recover progress/state. This alone does not make arbitrary external effects exactly once.

### Q70. What are sources and sinks?

Sources include Kafka, files, and connectors available in the chosen Spark distribution. Sinks include files, Kafka, and supported external systems. Options, dependencies, and guarantees are version-specific. foreachBatch gives control over each micro-batch but makes the application responsible for safe external writes.

### Q71. Explain append, update, and complete output modes.

Append emits rows considered new/final under the query's semantics; event-time aggregations may need watermarks to finalize windows. Update emits rows changed since the prior trigger for supported queries. Complete outputs the whole result table each trigger and can become expensive as results grow. Supported modes depend on the operation and sink; verify compatibility and sink semantics.

### Q72. What does a trigger control?

A processing-time trigger requests periodic micro-batches. Available-now (where supported) processes currently available data in one or more batches and terminates, useful for catch-up workloads. Default processing starts a new batch when possible. A trigger interval is not a guarantee of batch completion time. Continuous processing is separate and limited in documented versions.

### Q73. What is a checkpoint directory?

It stores query progress and, for stateful queries, state required for recovery. For Kafka, Spark records offsets in the checkpoint; Kafka consumer-group committed offsets are not the sole restart authority. The location must be durable, accessible, and dedicated to a compatible query lifecycle. Deleting/reusing it can cause replay, lost state, or incompatible recovery.

### Q74. Query ID versus run ID?

A query identifier is associated with the checkpointed query identity; a run ID identifies a particular execution instance. Restarting from the same checkpoint can preserve query identity while creating a new run. Correlate progress, batch IDs, offsets, and sink records. IDs are not business idempotency keys.

### Q75. Does Structured Streaming guarantee exactly-once processing?

It depends on source replayability, checkpointing, operations, and sink commit behavior. Supported source/sink combinations can provide documented exactly-once semantics for query processing, but custom callbacks or arbitrary external writes may repeat side effects after failure. Separate input replay, state recovery, sink commit, and business effect. Use stable event IDs, upserts, unique constraints, or a transactional protocol for external systems.

### Q76. Event time versus processing time?

Event time is when an event happened; processing time is when Spark handles it. A mobile event created at 10:00 and delivered at 10:08 has event time 10:00 but later processing time. Event-time windows fit business analytics with delayed arrival; processing-time windows answer a different operational question. Validate clocks, time zones, and timestamp parsing.

---

## 14. Event time, watermarks, windows, and state

### Q77. Tumbling versus sliding windows?

A tumbling window has fixed duration and does not overlap, such as 10:00–10:05 and 10:05–10:10. Sliding windows overlap; a 10-minute window sliding every 5 minutes includes an event in multiple windows, increasing work and state.

~~~java
Dataset<Row> perWindow = events
    .withWatermark("event_time", "10 minutes")
    .groupBy(window(col("event_time"), "5 minutes"), col("customer_id"))
    .agg(sum("amount").alias("total_amount"));
~~~

The watermark communicates expected lateness relative to event-time progress; it is not a promise that every event within ten minutes is guaranteed to arrive.

### Q78. What does a watermark do?

A watermark advances based on observed event-time progress and a configured delay. It lets supported stateful operations finalize eligible windows and clean old state. Without bounded cleanup, a long-running aggregation can retain state indefinitely.

Shorter thresholds reduce state sooner but may exclude late updates; longer thresholds tolerate delay at higher state and recovery cost. Multiple streams and stateful operators have additional semantics, so verify the rules for the deployed Spark version.

### Q79. What happens to records later than the watermark?

For a window aggregation, records too late to affect retained/finalized state may no longer update results. Exact behavior depends on operation and output mode. A watermark is not a universal filter that drops every old record immediately. Choose the lateness policy from business requirements and provide correction/reconciliation if very late updates matter.

### Q80. How do stateful aggregation and deduplication work?

Stateful aggregation retains partial results by key/window across triggers. Streaming deduplication remembers IDs for a supported state lifetime. State is checkpointed so a restarted query can recover it. Cost depends on key cardinality, window length, lateness threshold, and input rate. Define the duplicate horizon and use stable event IDs.

### Q81. Stream-static versus stream-stream joins?

A stream-static join enriches each streaming record with a bounded relation, often simpler but with deliberate handling needed for refreshed dimension data. A stream-stream join matches events from two unbounded streams and must retain state because matching records can arrive at different times. It requires careful event-time constraints and watermarks for bounded state in supported patterns. Verify join-type compatibility.

### Q82. What is the state store?

The state store persists intermediate state for stateful streaming operators between micro-batches. State growth, checkpoint I/O, key cardinality, and implementation affect latency. Monitor state rows/bytes, update/commit duration, and watermark progress where available.

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

Include the Kafka connector version compatible with Spark and its Scala binary version. Kafka key/value are binary columns; parse explicitly. startingOffsets applies when initializing a new query/checkpoint; an existing checkpoint's progress governs recovery.

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

Confirm output-mode compatibility and connector documentation for your Spark version. Checkpointing supports Spark recovery; downstream consumers still need duplicate-safe business handling where required.

### Q86. Are Spark tasks the same as Kafka consumer-group members?

No. Kafka topic partitions constrain source parallelism, while Spark schedules tasks for source offset ranges and downstream operations. Do not model the source as an ordinary consumer application with the same manual commit workflow. More executors do not create more Kafka partitions. Downstream shuffles may have a different task count.

### Q87. How do offsets and restart work?

Spark's Kafka source stores offset progress in its checkpoint and can replay according to recorded progress. Kafka consumer-group committed offsets are not the sole authority for query restart. startingOffsets matters for a new checkpoint, not for overriding an established query's progress. Replay may be correct for Spark state but can repeat an external side effect if the sink is not idempotent.

### Q88. How do you make database writes idempotent?

Use stable event identifiers and upserts/unique constraints so a retry converges to the same final state. With foreachBatch, batchId helps identify a batch only if the sink safely records and commits it; it cannot prevent duplicates if a write partially succeeds before the marker. Use transactions where possible, deterministic keys, and version checks for out-of-order updates.

### Q89. What is backpressure and how do you handle Kafka lag?

Lag grows when arrival rate exceeds sustainable processing and sink throughput. Diagnose input/processing rate, batch duration, topic partition count, shuffle, state-store time, and sink latency. Optimize the measured bottleneck, add source partitions when appropriate, and tune admission/trigger settings according to version docs. If the sink is slow, more executors may increase pressure rather than throughput.

### Q90. How do you handle malformed messages and schema evolution?

Treat payloads as untrusted input. Parse with an explicit schema, preserve event ID and source metadata, and route malformed records to quarantine with a reason. Fail fast when continuing would corrupt critical results. Define schema compatibility and coordinate producer/consumer changes; a new optional field differs from changing an existing field's type. Alert on malformed-record rates.

---

## 16. Java implementation patterns

### Q91. What does a basic Java batch application look like?

~~~java
import static org.apache.spark.sql.functions.*;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
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

foreachBatch provides a bounded DataFrame for each micro-batch, useful for custom sinks. It does not make arbitrary code exactly once. The callback may be retried after a failure, so writes must be idempotent or transactional.

~~~java
stream.writeStream()
    .foreachBatch((batch, batchId) -> {
      // Use a stable event key and idempotent upsert/transaction protocol.
      // Plain append may duplicate rows after a retry.
      writeBatchIdempotently(batch, batchId);
    })
    .option("checkpointLocation", checkpointPath)
    .start();
~~~

The method writeBatchIdempotently is intentionally application-specific, not a Spark API. Design it around unique event keys and the sink's transactional capabilities. Keep credentials in a secret manager.

### Q94. What should observability and exception handling include?

Log query/run identity, batch context, and failure cause; export input/processing rate, batch duration, sink latency, lag, state size, malformed records, and data freshness. Do not log secrets or sensitive payloads. Health checks should distinguish process health from active query health.

---

## 17. Production reliability

### Q95. What happens if an executor is lost?

Tasks running there may fail and be retried elsewhere. Cached partitions may be recomputed from lineage. Shuffle files can become unavailable depending on executor loss and shuffle configuration. Repeated executor loss can indicate memory pressure, node instability, container limits, or disk/network problems. Inspect executor logs, platform events, GC, and task failures before increasing retries.

### Q96. What happens if the driver fails?

The application usually terminates because the driver coordinates execution. A cluster manager may restart it, but streaming recovery depends on the application restarting with the correct checkpoint and compatible query. Batch jobs may need resubmission, and output correctness depends on sink commit behavior. Checkpointing does not make arbitrary batch side effects resumable.

### Q97. How do you investigate shuffle fetch failures?

A downstream task could not retrieve shuffle output. Causes include executor loss, unavailable shuffle files, disk issues, network timeouts, or resource exhaustion. Determine whether failures cluster around one node and inspect executor/worker logs, disk, network, and cluster events. Retries can recover transient problems but do not fix unstable infrastructure.

### Q98. What if a streaming checkpoint is unavailable or corrupted?

Preserve logs and investigate storage connectivity, permissions, deletion, and compatibility before changing anything. Do not point to an empty checkpoint and assume it will resume correctly; it may start a new query and replay input or lose prior state. Restore a valid checkpoint if possible. Otherwise plan a controlled replay from a known boundary with idempotent output and state reconstruction.

### Q99. How should corrupt input be handled?

Define schema and parsing policy. Quarantine malformed records with source metadata and reason, or fail fast when invalid input threatens correctness. Avoid silently replacing business-critical fields with defaults. Ensure a poison record does not create an endless retry loop without an alert and remediation path.

### Q100. What should a production Spark pipeline monitor?

Monitor application/query health, batch/stage duration, input/processing rates, Kafka lag, failed tasks, executor loss, shuffle, spill, GC, state size, watermark progress, sink latency, output counts, malformed records, and data freshness. Alert on SLOs and trends. Reconcile source/sink counts or business totals because green infrastructure metrics do not prove correct output.

---

## 18. Deployment and operations

### Q101. What changes on Kubernetes?

Kubernetes schedules driver/executor pods and enforces resource limits. Configure compatible images/dependencies, service-account permissions, network access, credentials, and checkpoint storage. Container memory includes more than JVM heap; pod events help distinguish Java OOM from a container kill. Kubernetes is a deployment environment, not an automatic solution to application-level recovery.

### Q102. What matters when packaging a Java Spark application?

Use a Spark-compatible JDK and matching Spark/Scala connector artifacts. Managed environments often provide Spark libraries, so avoid bundling conflicting copies. Kafka connector coordinates must match Spark and Scala binary versions. Test the packaged artifact in a representative cluster, not only in an IDE.

### Q103. How do you debug local success but cluster failure?

Compare Spark/JDK/connector versions, deployment mode, classpath, credentials, DNS, storage access, executor logs, and container limits. Local mode hides distributed serialization and resource constraints. Collect driver, executor, and cluster-manager logs with application ID and timestamps.

---

## 19. Frequently asked questions

### Q104. Why avoid collecting large results to the driver?

The full result must fit in driver memory and transfer over the network. Keep computation distributed, write to distributed storage, aggregate before collecting, or request a bounded sample.

### Q105. Does increasing executor count always speed up a job?

No. It helps only when parallel work exists and compute resources are the bottleneck. It does not fix one skewed partition, a serial sink, insufficient Kafka partitions, or an unavoidable serial step. More executors can increase network pressure and contention.

### Q106. repartition versus partitionBy?

Repartition changes in-memory execution partitions and usually shuffles. partitionBy in a writer organizes output into directory partitions by column values. They solve different problems and can both be used when justified.

### Q107. Is Spark a database?

No. Spark is a distributed compute engine with SQL capabilities, not a general transactional OLTP database.

### Q108. Checkpoint versus offset?

An offset identifies a source position, such as a Kafka partition offset. A checkpoint stores query progress and may also hold state required for recovery.

### Q109. Does a watermark wait exactly that long for each event?

No. It is derived from event-time progress and a configured delay. It governs cleanup/finalization for supported stateful operations; it is not a timer per record.

### Q110. Can foreachBatch guarantee exactly-once database writes?

Not automatically. The callback can be retried and a write can partially succeed. Exactly-once business effects require idempotent/transactional sink behavior.

### Q111. Input rate versus processing rate?

Input rate measures records arriving/admitted; processing rate measures records processed. Sustained input above processing causes backlog or lag. Interpret both with batch duration and admission settings.

### Q112. Why might a filter not reduce input I/O?

The predicate may not be pushable, the source may not support skipping data, or an expression/UDF may block optimization. It can reduce downstream rows while Spark still reads all source bytes.

### Q113. What is the small-file problem?

Too many small files increase listing, metadata, open, and scheduling overhead. Compact files and design output partitioning sensibly; do not funnel a huge dataset through one task.

### Q114. Why can streaming state grow forever?

Unbounded key cardinality, absent/ineffective watermarking, non-advancing event time, or semantics that require retaining all keys can keep state growing. Inspect state and watermark metrics before changing retention.

### Q115. Spark or a regular Kafka consumer service?

Spark fits distributed joins, aggregations, large-scale transformations, and backfills. A regular consumer can be simpler for low-latency per-message business logic and small state. Choose based on latency, scale, correctness, and operations.

### Q116. What does exactly-once mean?

Define the boundary. It may mean input offsets contribute once to recoverable Spark state, or that a business row changes once. Those are not equivalent; source replay, state checkpointing, sink commits, and idempotency must all be considered.

### Q117. First response to a slow stage?

Compare task duration, input/shuffle metrics, and the plan. Determine whether one or all tasks are slow. Do not immediately increase memory or executor count; distinguish skew, large input, spill, GC, and external I/O.

### Q118. Why can broadcast be slower than expected?

The materialized relation may be larger than estimated, broadcast consumes memory/network, or join output cardinality is huge. Validate actual size and final plan.

### Q119. Why use an explicit schema?

It clarifies types and expected fields, avoids unreliable inference, and makes schema changes visible. It does not eliminate the need to validate records.

### Q120. How do you prevent duplicate effects after restart?

Use stable event IDs and idempotent writes, unique constraints/upserts, and tests around partial sink commits. A Spark checkpoint alone cannot prevent arbitrary external side effects from repeating.

---

## 20. Production scenario bank

For every incident, clarify expected behavior, inspect evidence, isolate the cause, make the smallest safe change, and verify both performance and correctness.

### Scenario 1. A Spark job suddenly becomes much slower.

Ask whether input volume, file count, schema, code, cluster size, or sink changed. Compare known-good and current stage durations, task distribution, input bytes, shuffle, spill, GC, executor loss, and plan. Restore pruning, compact files, fix skew, tune partitions, or address contention based on evidence. Keep baseline metrics and alerts. Follow-up: why not add executors first?

### Scenario 2. One task runs much longer than all others.

Compare task input/shuffle bytes, spill, and duration. A hot key, large file, or unhealthy executor may explain it. Verify key distribution and node health, then use AQE skew handling, pre-aggregation, or correctly designed salting. Monitor task variance. Follow-up: how do you distinguish skew from a bad node?

### Scenario 3. A join causes a massive shuffle.

Inspect the physical plan, input sizes, key uniqueness, and output cardinality. Filter/project first, broadcast only a genuinely small side, improve statistics, or address skew. Do not force broadcast based only on compressed file size. Follow-up: why can a broadcast hint cause OOM?

### Scenario 4. An executor repeatedly runs out of memory.

Determine JVM heap OOM versus container kill; inspect GC, spill, broadcasts, cache, task size, and platform events. Reduce data, fix skew, remove unneeded cache, or change join strategy before increasing resources. Follow-up: why can a larger heap worsen GC pauses?

### Scenario 5. The driver crashes during collect.

Confirm result size and whether a full result was needed. Replace collect with distributed output, aggregation, or a bounded take/sample. Add code review rules for unbounded driver actions. Follow-up: why is show safer but still an action?

### Scenario 6. A broadcast join causes memory pressure.

Inspect materialized broadcast size, final plan, executor memory, and stale statistics. Remove the hint or use a shuffle join if the relation is not safely small; project/filter before broadcasting. Monitor dimension-table growth. Follow-up: why is compressed source size misleading?

### Scenario 7. Thousands of tiny output files are created.

Check task count, output partition values, and file sizes. Adjust partitioning and compact output in a distributed rewrite; avoid coalesce(1) for large datasets. Define file-size targets and compaction. Follow-up: why isn't file count always equal to DataFrame partition count?

### Scenario 8. Increasing partitions makes a job slower.

Inspect scheduling overhead, bytes per task, file count, executor slots, and shuffle metrics. Reduce excessive partitions or use AQE coalescing where suitable while preserving parallelism. Benchmark on realistic data. Follow-up: can one stage need a different count than another?

### Scenario 9. A large input becomes one bottleneck partition.

Determine whether it is an unsplittable file, a hot key, or explicit single-partition operation. Fix source layout, remove unnecessary coalesce, or redistribute before expensive work. Random repartition alone will not split a single hot aggregation key's logical group. Follow-up: why?

### Scenario 10. A job works locally but fails in cluster mode.

Compare versions, dependencies, credentials, networking, storage permissions, serialization, and memory limits. Read driver/executor/platform logs. Fix packaging and environment differences, then add cluster smoke tests. Follow-up: why can a fat JAR with Spark libraries conflict with cluster libraries?

### Scenario 11. AQE changes the query plan unexpectedly.

Compare initial/final adaptive plans and runtime statistics. Determine whether the new plan is actually harmful; validate thresholds and data stats, then benchmark a targeted change. Disable AQE only as a controlled diagnostic. Follow-up: what can AQE not fix?

### Scenario 12. A transformation appears to run twice.

List actions and inspect UI jobs. show followed by write can recompute the plan. Remove unnecessary actions or persist an expensive reused result, then unpersist it. For streaming side effects, make writes idempotent. Follow-up: does cache execute immediately?

### Scenario 13. Kafka lag grows continuously.

Compare input/processing rate, batch duration, source partitions, shuffle, state time, and sink latency. Optimize the actual bottleneck; add partitions/resources only where parallelism exists. Alert on lag and capacity. Follow-up: why might more executors not help source reading?

### Scenario 14. A restart appears to duplicate output.

Correlate checkpoint progress, sink commits, event IDs, and failure timing. A partial sink commit may be replayed. Preserve the checkpoint and implement upserts/unique keys or a transactional sink protocol. Test failures at each boundary. Follow-up: define exactly-once for this sink.

### Scenario 15. Late events are missing from a window.

Inspect timestamp parsing, event-time distribution, watermark progress, window boundaries, and output mode. Set lateness based on business data and provide reconciliation for very late records. Monitor late-event rates. Follow-up: does a ten-minute watermark guarantee every event less than ten minutes late is included?

### Scenario 16. Watermarks do not reduce state.

Check advancing event timestamps, nulls, watermark placement, query compatibility, and multi-stream policy. Fix data/query semantics before shortening the delay. Alert on state growth without watermark progress. Follow-up: how can a lagging input affect a multi-stream watermark?

### Scenario 17. State store grows excessively.

Inspect key cardinality, window duration, watermark progress, and state metrics. Bound state only if business semantics permit; if all historical keys are required, plan for it. Monitor state and recovery time. Follow-up: why can more executor memory only postpone the issue?

### Scenario 18. Sink throughput is below input throughput.

Measure sink latency, connection limits, locks, and write concurrency. Batch/bulk writes, bound concurrency, or scale the sink; avoid overwhelming it with more parallel tasks. Set sink SLOs. Follow-up: how can retries amplify overload?

### Scenario 19. Checkpoint storage becomes unavailable.

Preserve logs and check permissions, storage health, and checkpoint integrity. Restore access and resume from the valid checkpoint. If lost, plan controlled replay and idempotent reconciliation; never assume an empty checkpoint resumes the old state. Follow-up: can checkpoint recovery restore Kafka records already deleted by retention?

### Scenario 20. Streaming fails after a schema change.

Determine whether fields were added, renamed, removed, or type-changed. Inspect parser errors and checkpoint/query compatibility. Apply a compatible rollout or deliberate migration/replay, and test old/new versions. Follow-up: why can valid JSON fail schema parsing?

### Scenario 21. A Kafka event cannot be deserialized.

Preserve topic/partition/offset and a safe sample; compare encoding/schema versions. Quarantine malformed events or fail fast when correctness requires it. Avoid endless poison-record retries and sensitive payload logs. Follow-up: how do you avoid repeatedly processing the same poison message?

### Scenario 22. Batch and streaming produce different totals.

Compare time zones, window boundaries, null handling, duplicates, late events, and deduplication horizons. Streaming state cleanup may differ from an unbounded batch calculation. Define shared semantics, test boundary cases, and reconcile with a backfill. Follow-up: why is streaming distinct operationally different?

### Scenario 23. Database writes must be idempotent.

Map the failure window between database commit and Spark progress commit. Use stable event keys, unique constraints/upserts, version checks, and transactional batch markers where supported. Test partial commits. Follow-up: why is batchId alone not enough for individual records?

### Scenario 24. One hot key dominates join runtime.

Profile key counts and task shuffle reads; check for many-to-many expansion. Pre-aggregate when valid, use AQE skew handling, or salt with careful replication and merge. Follow-up: what correctness risks does salting introduce?

### Scenario 25. Most runtime is garbage collection.

Correlate GC with heap occupancy, spill, cache, broadcasts, and task size. Reduce unnecessary objects, prefer SQL expressions where possible, fix skew, and tune heap only with evidence. Track GC in load tests. Follow-up: why is a bigger heap not always a fix?

### Scenario 26. Shuffle fetch failures recur.

Determine whether errors follow one executor, disk, node, or network. Inspect fetch causes, executor-loss events, and worker logs. Fix unstable infrastructure before tuning retries. Follow-up: why can downstream tasks fail after upstream tasks succeeded?

### Scenario 27. The UI shows many tiny tasks.

Check tiny input files, excessive shuffle partitions, empty partitions, and scheduling overhead. Compact files or tune partition counts/AQE. Preserve sufficient parallelism for large stages. Follow-up: why are file input partitions and shuffle partitions different?

### Scenario 28. A large aggregation exhausts memory.

Inspect cardinality, skew, spill, and partial aggregation. Reduce input early, pre-aggregate where valid, fix hot keys, or revise reporting grain if business needs allow. Follow-up: why can groupBy still be expensive with partial aggregation?

### Scenario 29. A stream must recover after hours of downtime.

Check whether Kafka retention still contains required offsets, whether checkpoint/state is intact, and whether sink can absorb catch-up throughput. Resume from checkpoint and provision catch-up capacity; backfill gaps from durable history if retention expired. Follow-up: can checkpoint restore deleted Kafka records?

### Scenario 30. Kafka volume grows tenfold.

Measure event size, partition skew, batch duration, state, shuffle, and sink capacity. Scale the constrained layer, increase Kafka partitions only after considering ordering/keying, and test downstream limits. Follow-up: what ordering changes can partition expansion cause?

### Scenario 31. The pipeline must deduplicate reliably.

Define globally stable event IDs and the maximum duplicate delay. Use streaming deduplication with a justified state horizon plus sink idempotency. Measure duplicate rate and test replay. Follow-up: why does deduplication state need a lifecycle policy?

### Scenario 32. Historical backfill must transition to live events.

Define a common schema and exact offset/time boundary. Prevent gaps/overlap with a documented handoff and deterministic keys; reconcile overlap idempotently. Follow-up: how can a race during cutover lose events?

### Scenario 33. Parquet schema changes over time.

Inspect schemas per file/partition. Normalize compatible additions and deliberately migrate incompatible type changes; test old and new data together. Add schema contracts. Follow-up: what are the cost and risk of schema merging?

### Scenario 34. A Spark app must run on Kubernetes.

Check image/dependency compatibility, service-account permissions, storage/network access, resource requests/limits, and memory overhead. Inspect pod events and driver/executor logs; stage-test before production. Follow-up: why can a pod be OOM-killed without Java heap OOM?

### Scenario 35. A pipeline writes to Kafka and Elasticsearch and one sink fails.

Independent writes are not atomic: Kafka may succeed while Elasticsearch fails. Choose a source of truth, use replayable/idempotent sinks, deterministic Elasticsearch document IDs, and reconciliation. Consider an outbox or independent sink consumers instead of pretending one checkpoint coordinates a distributed transaction. Follow-up: why can't Spark checkpointing make unrelated systems atomic?

### Scenario 36. The database is overloaded after Spark scaling.

Compare write concurrency, connection counts, locks, database CPU/I/O, and retries. Bound parallel writes, batch or stage/merge data, or scale the sink. Set throughput budgets. Follow-up: how can retries amplify overload?

### Scenario 37. A report's totals differ from the source.

Reconcile counts/sums at each boundary; inspect duplicates, null amounts, currency, timezone, joins, and late events. Correct semantic bugs before tuning and backfill idempotently. Add data-quality assertions. Follow-up: why can a successful job still produce incorrect results?

### Scenario 38. A UDF makes the query slower.

Compare plan, CPU, and code generation. Replace UDF with built-in expressions where possible, reduce rows before applying it, or precompute when justified. Benchmark at scale. Follow-up: why is a UDF not automatically more efficient just because Spark distributes it?

### Scenario 39. A Parquet query scans too much data.

Check partition filters, pushed filters, casts/functions on partition columns, and selected columns. Express eligible predicates clearly and align layout with common access patterns. Follow-up: distinguish partition pruning from predicate pushdown.

### Scenario 40. An upstream producer emits out-of-order updates.

Ask for business version/sequence and rules for stale updates. Compare event/ingestion time and version; watermarks do not define business ordering. Use deterministic versioned upserts and a correction path. Follow-up: why is event time not a substitute for entity version?

---

## 21. Mock interviews and revision

### Mock interview A — Fundamentals

1. What problem does Spark solve that a single machine cannot meet economically?
2. Explain driver, executor, job, stage, and task using a groupBy.
3. Why prefer DataFrames for structured data?
4. What happens when count is called?
5. Explain a shuffle and two operations that cause one.
6. Diagnose a single slow task.

Strong answers connect definitions to execution and trade-offs.

### Mock interview B — Spark SQL/DataFrame coding

**Prompt:** Given orders with order_id, customer_id, order_ts, status, and amount, calculate paid revenue per customer/day and rank each customer's days by revenue.

~~~java
Dataset<Row> daily = orders.filter(col("status").equalTo("PAID"))
    .groupBy(col("customer_id"),
             to_date(col("order_ts")).alias("order_date"))
    .agg(sum("amount").alias("revenue"));

WindowSpec w = Window.partitionBy("customer_id")
    .orderBy(col("revenue").desc(), col("order_date").asc());
Dataset<Row> ranked = daily.withColumn("day_rank", dense_rank().over(w));
~~~

Discuss null amounts, timezone, ties, shuffle/sort cost, and source reconciliation.

### Mock interview C — Performance troubleshooting

**Prompt:** A 1 TB join takes 40 minutes; most tasks finish in two minutes but a few take 25 minutes.

Compare task durations, input/shuffle metrics, plan, key frequencies, and executor health. If slow tasks process more data, investigate skew and use a semantics-preserving fix. If data sizes are similar, inspect node health, GC, and I/O. Prove the fix with output checks and representative load.

### Mock interview D — Streaming and Kafka

**Prompt:** A Kafka-to-database query restarts and sometimes duplicates rows.

Explain checkpointed offsets, micro-batch retries, partial database commits, and why a checkpoint alone cannot guarantee idempotent database effects. Propose stable event IDs, unique constraints/upserts, and failure-injection tests. Discuss lag, retention, sink capacity, and schema errors.

### Mock interview E — Senior architecture

**Prompt:** Consume order events, enrich customer data, calculate five-minute event-time metrics, and write to Kafka and Elasticsearch.

Clarify latency, lateness, replay horizon, schema ownership, and consistency. Explain offsets, checkpoint, watermark/state, sink idempotency, and why two independent sinks are not atomically updated by an ordinary Spark checkpoint. Choose a source of truth and add lag, state, sink, freshness, and reconciliation monitoring.

### Top 50 must-know questions

1. What problem does Spark solve?
2. Explain Hadoop, HDFS, and MapReduce.
3. Driver and executor responsibilities?
4. Jobs, stages, and tasks?
5. What triggers execution?
6. What is lazy evaluation?
7. Narrow versus wide dependency?
8. RDD versus DataFrame versus Dataset?
9. Why do DataFrames optimize well?
10. What does explain show?
11. What causes shuffle?
12. How does partition count affect parallelism?
13. repartition versus coalesce?
14. What is data skew?
15. How does salting work?
16. What is partition pruning?
17. What does shuffle partition configuration control?
18. Broadcast versus sort-merge join?
19. How does AQE work?
20. Why can joins multiply rows?
21. Why can distinct be expensive?
22. Window versus groupBy?
23. Catalyst and code generation?
24. Predicate pushdown versus projection pruning?
25. Driver OOM versus executor OOM?
26. Why spill to disk?
27. How does GC affect runtime?
28. Cache versus persist?
29. Checkpoint versus cache?
30. Diagnose a slow stage?
31. Why are small files harmful?
32. Why Parquet for analytics?
33. How are output files created?
34. What is Structured Streaming?
35. Explain micro-batch.
36. Append/update/complete modes?
37. What is a checkpoint directory?
38. Event time versus processing time?
39. Tumbling versus sliding windows?
40. What does a watermark do?
41. What happens to late data?
42. How does streaming state recover?
43. How are Kafka offsets recovered?
44. Spark tasks versus Kafka consumers?
45. What does exactly-once mean?
46. Idempotent database writes?
47. Diagnose growing Kafka lag?
48. What happens if an executor is lost?
49. Recover from checkpoint loss?
50. Design replay across multiple sinks?

### Top 25 Spark SQL/DataFrame patterns

1. Select only needed columns.
2. Filter before expensive joins/aggregations.
3. withColumn for derived values.
4. when/otherwise for conditions.
5. coalesce for deliberate null fallback.
6. isNull/isNotNull for null predicates.
7. groupBy/agg for distributed aggregation.
8. count(*) versus count(column).
9. row_number for ranking.
10. dense_rank for ties.
11. Left join for enrichment.
12. Semi join for existence.
13. Anti join for missing keys.
14. Broadcast only safely small tables.
15. unionByName for schema alignment.
16. dropDuplicates(keys) for key deduplication.
17. Parse dates/timestamps with timezone awareness.
18. explode for array expansion.
19. Temporary view plus Spark SQL.
20. Explicit schema for CSV/JSON.
21. Parquet projection and predicates.
22. Repartition only when redistribution is justified.
23. Coalesce output tasks when unevenness is acceptable.
24. explain("formatted") for plan inspection.
25. Persist reused expensive data and unpersist afterward.

### Top 20 Structured Streaming concepts

1. Unbounded input table.
2. Micro-batch and trigger cadence.
3. Source offsets and checkpoint.
4. Query ID versus run ID.
5. Output modes.
6. Event versus processing time.
7. Tumbling/sliding windows.
8. Watermark and state cleanup.
9. Late-data trade-offs.
10. Stateful aggregation.
11. Deduplication horizon.
12. Stream-static versus stream-stream joins.
13. State-store metrics.
14. Kafka offset recovery.
15. Kafka partition parallelism.
16. foreachBatch retry implications.
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

---

## 22. Official references

Use documentation matching the deployed Spark version; the latest URLs change as releases publish.

- [Apache Spark documentation](https://spark.apache.org/docs/latest/)
- [RDD programming guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html)
- [Spark SQL, DataFrames, and Datasets](https://spark.apache.org/docs/latest/sql-programming-guide.html)
- [Structured Streaming overview](https://spark.apache.org/docs/latest/streaming/index.html)
- [Structured Streaming programming guide](https://spark.apache.org/docs/latest/streaming/getting-started.html)
- [Structured Streaming Kafka integration](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- [Spark tuning guide](https://spark.apache.org/docs/latest/tuning.html)
- [Spark on Kubernetes](https://spark.apache.org/docs/latest/running-on-kubernetes.html)
- [Spark configuration](https://spark.apache.org/docs/latest/configuration.html)
- [Spark SQL performance tuning](https://spark.apache.org/docs/latest/sql-performance-tuning.html)

**Validation note:** This is a documentation-focused handbook; snippets are illustrative patterns, not a claim that every example was compiled or run in a live cluster. Verify API signatures, connector coordinates, default settings, supported streaming operations, AQE rules, and state behavior against your deployed Spark/JDK/connector versions.
