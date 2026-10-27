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

It sets the default partition count for many Spark SQL shuffle operations. The current Spark 4.2 configuration reference lists 200 as the default, but verify the deployed version and effective application settings. Too high creates tiny tasks; too low creates large tasks and insufficient parallelism. AQE can coalesce shuffle partitions or alter eligible join decisions at runtime. A Java application can override it for a session with `spark.conf().set("spark.sql.shuffle.partitions", "400")`, but choose the value from shuffle volume and task metrics rather than copying a magic number.

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

AQE uses runtime statistics to revise eligible parts of a query plan. In the current Spark 4.2 configuration reference, `spark.sql.adaptive.enabled` defaults to true; verify the effective setting for the actual Spark distribution/application. Depending on version and query shape, AQE can coalesce shuffle partitions, handle qualifying skewed joins, and revise certain join strategies. Inspect the final adaptive plan rather than assuming an adaptive change was beneficial. AQE cannot fix incorrect join logic, huge output cardinality, or every skew pattern.

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

A processing-time trigger requests periodic micro-batches. If a batch finishes before the interval, Spark waits for the interval boundary; if it takes longer, the next batch starts when the prior one completes. An available-now trigger processes data available at query start in one or more micro-batches and then terminates, provided the source supports it. It is useful for scheduled catch-up jobs. The default trigger starts another micro-batch as soon as the prior one completes and data is available.

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

A trigger interval is not a batch-completion SLA. If processing takes longer than the interval, the query cannot catch up unless sustainable throughput exceeds input rate. Continuous processing is separate, documented as experimental, and has different operation and sink constraints.

### Q73. What is a checkpoint directory?

It stores query progress and, for stateful queries, state required for recovery. For Kafka, Spark records offsets in the checkpoint; Kafka consumer-group committed offsets are not the sole restart authority. The location must be durable, accessible, and dedicated to a compatible query lifecycle. Deleting/reusing it can cause replay, lost state, or incompatible recovery.

### Q74. Query ID versus run ID?

A query identifier is associated with the checkpointed query identity; a run ID identifies a particular execution instance. Restarting from the same checkpoint can preserve query identity while creating a new run. Correlate progress, batch IDs, offsets, and sink records. IDs are not business idempotency keys.

### Q75. Does Structured Streaming guarantee exactly-once processing?

It depends on source replayability, checkpointing, the operation, and sink commit behavior. Structured Streaming can recover query progress/state for supported sources and operations, but sink guarantees are connector-specific. The official Spark Kafka integration guide explicitly documents Kafka writes from Spark as at-least-once: a retried write can duplicate a record even when the broker had accepted the earlier attempt. A custom callback or arbitrary database write can also repeat side effects after failure. Separate input replay, state recovery, sink commit, and business effect. Use stable event IDs, upserts, unique constraints, or a transactional/idempotent sink protocol where needed.

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

A watermark is derived from the maximum event time observed by the query minus the configured delay threshold. It lets supported stateful operations decide when older windows/state can be finalized or cleaned. It does not advance just because wall-clock time passes; the source must make event-time progress. For aggregation cleanup, the watermark must be applied before the aggregation to the same event-time column (or a window derived from it), and output mode must support cleanup (Append or Update, not Complete). Without effective state cleanup, a long-running query can retain state indefinitely.

For supported windowed aggregations, Spark documents a one-sided guarantee: records less than the configured delay behind the latest event time processed will not be dropped as too late. Records delayed beyond that threshold may or may not be processed; the watermark is not a promise that every older record is immediately discarded. Longer thresholds usually retain state longer and tolerate more event-time lateness, at additional memory/checkpoint cost. Multiple streams and stateful operator types introduce further semantics, so validate the exact query shape against the deployed version.

### Q79. What happens to records later than the watermark?

For supported window aggregations, an event that is less than the configured watermark delay behind the latest event time processed is guaranteed not to be dropped as late. That is an event-time relationship, not simply the elapsed wall-clock time since the event happened. Records beyond the threshold are not guaranteed to be dropped: Spark may still aggregate them if the relevant state has not been cleaned. The watermark is therefore neither a universal pre-filter nor a promise that every very late event is ignored. Choose the delay from business requirements and provide correction/reconciliation if very late updates matter.

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

Confirm output-mode compatibility and connector documentation for your Spark version. Important guarantee: the official Spark Kafka integration documents Kafka sink writes as at-least-once, so a retry can create duplicate records even if the broker accepted an earlier attempt. Checkpointing supports recovery of query progress; it does not turn the Kafka sink into an exactly-once business-effect protocol. Use a stable event key and make downstream consumers idempotent or deduplicate where required.

### Q86. Are Spark tasks the same as Kafka consumer-group members?

Not exactly. By default, each Kafka TopicPartition maps to a Spark input partition for a micro-batch, so Kafka partition count is an important parallelism limit. The connector also provides a minPartitions option that can split large Kafka offset ranges into more Spark input partitions; treat it as a desired minimum, not a promise of an exact task count. Downstream shuffles may have a different task count. Spark tracks source offsets in its checkpointed query progress rather than relying on an ordinary consumer group's committed offsets as the sole restart position.

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

Use a JDK supported by the exact Spark distribution and matching Spark/Scala connector artifacts. The current Spark 4.2 documentation lists Java 17, 21, and 25 and Scala 2.13; Spark 4.0 removed Scala 2.12 support. Do not apply that matrix blindly to older Spark 3.x deployments—check the documentation for the version you actually run. Managed environments often provide Spark libraries, so avoid bundling conflicting copies. Kafka connector coordinates must match Spark and Scala binary versions. Test the packaged artifact in a representative cluster, not only in an IDE.

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

The following scenarios model how a senior engineer should reason in an interview. Start by clarifying the expected behavior and scope; use runtime evidence to narrow the cause; make the smallest safe change; then verify both performance and business correctness. Spark tuning should be evidence-driven: “add executors,” “increase memory,” and “repartition” are not diagnoses on their own.

### Scenario 1. A Spark job suddenly becomes much slower.

**Interview problem:** A daily orders pipeline used to finish in 18 minutes but now takes 75 minutes, although the code appears unchanged.

**Clarify first:** Did input volume or file count increase? Did a producer change the schema or key distribution? Did cluster capacity, Spark version, source layout, or sink latency change? Is the slowdown in one stage or end-to-end?

**Diagnosis and resolution:** Compare a known-good run with the slow run. In the Spark UI, identify the stage with the largest increase and compare input bytes, shuffle read/write, spill, GC time, task-duration distribution, executor loss, and failed/retried tasks. Inspect the formatted physical plan and look for a new scan, exchange, join strategy, or lost partition filter. If the input scan grew, verify partition pruning and file counts. If shuffle exploded, compare row counts and join cardinality. If only the tail is slow, inspect skew and executor health. If processing finishes but the application waits, inspect the sink.

A disciplined experiment changes one variable: restore a useful filter, correct a join key, compact small files, address skew, or tune partitions based on the evidence. Compare output counts and totals as well as runtime.

**Trade-offs and prevention:** Keep run-level baselines, representative performance tests, and alerts for input volume, duration, failed tasks, and data freshness.

**Follow-ups:** Why is increasing executor count not the first response? What metrics distinguish a shuffle bottleneck from a slow sink?

### Scenario 2. One task runs much longer than every other task.

**Interview problem:** Most tasks finish within 40 seconds, but one task runs for 15 minutes.

**Clarify first:** Is the slow task processing much more input or shuffle data? Does the same executor repeatedly appear slow? Is the task spilling heavily or spending time in GC?

**Diagnosis and resolution:** Compare the slow task with the median task in the same stage: input records/bytes, shuffle read, memory and disk spill, GC time, executor, and host. If it processes much more data, inspect the partitioning key and key-frequency distribution. A hot customer or tenant can put a disproportionate share of rows into one partition. Use AQE skew handling when the query and join type qualify, pre-aggregate before the expensive operation when semantics permit, or salt the hot key with a correct second aggregation/join step. If data sizes are similar but one host is consistently slow, investigate CPU throttling, disk, network, JVM pauses, and container events instead.

Do not randomly increase partitions and assume the hot logical key will split; a normal key-based aggregation still needs to combine that key correctly.

**Trade-offs and prevention:** AQE has operator-specific limits; salting adds complexity and can change join multiplicity if implemented incorrectly. Track max/median task duration and hot-key distribution.

**Follow-ups:** How would you distinguish key skew from a slow node? Why does adding executors not necessarily help one skewed reduce task?

### Scenario 3. A join causes a massive shuffle and runtime increase.

**Interview problem:** Joining 800 GB of order events with customer data introduces a multi-terabyte shuffle.

**Clarify first:** How large are the inputs after filters and projections? Is one side actually small enough to broadcast after materialization? Are join keys normalized and unique? Is a many-to-many result intended?

**Diagnosis and resolution:** Inspect the physical plan for exchanges and the selected join operator. Record input row counts, distinct key counts, null-key rates, duplicate keys, and output row count. A missing or incomplete join predicate can produce a legitimate but enormous Cartesian-like expansion. Apply filters and select only needed columns before joining. If one side is genuinely small, a broadcast hash join may avoid shuffling the large side; otherwise a sort-merge join may be appropriate. If a small number of keys dominate, test AQE skew-join handling or a targeted salting design.

~~~java
orders.filter(col("status").equalTo("PAID"))
    .select("customer_id", "order_id", "amount")
    .join(broadcast(customerDimension.select("customer_id", "segment")),
          new String[] {"customer_id"}, "left");
~~~

Use a broadcast hint only after confirming size and executor memory. Validate business totals and row multiplicity after the change.

**Trade-offs and prevention:** Broadcasting trades shuffle cost for memory/network overhead on each executor. Add join-cardinality checks and review plans for critical data products.

**Follow-ups:** Why can a small compressed Parquet file still become a large broadcast relation? When is sort-merge preferable?

### Scenario 4. An executor repeatedly runs out of memory.

**Interview problem:** The application fails with executor OOMs after a new dimension table is added.

**Clarify first:** Is it a JVM heap OOM, a container kill, or a native/off-heap memory failure? Does it happen to all executors or only a few tasks? Did a broadcast, cache, partition size, or data skew change?

**Diagnosis and resolution:** Read executor logs and cluster events, then correlate the failure with task metrics, GC time, spill, cached blocks, and broadcast size. A container can exceed its total memory limit even when the JVM heap has not produced a conventional Java heap OOM. If failures cluster around one task, investigate skew or oversized partition state. If all executors fail when a broadcast arrives, re-evaluate the join strategy and relation size. Remove unnecessary caches, prune columns, filter earlier, or reduce per-task working sets before increasing resources. Adjust executor heap/overhead only after identifying the memory domain and checking cluster capacity.

**Trade-offs and prevention:** Bigger heaps can reduce eviction but also lengthen GC pauses; more executors can increase overhead and shuffle traffic. Load-test representative peak data and alert on GC, spill, executor loss, and container memory kills.

**Follow-ups:** How do you distinguish heap OOM from memory overhead exceeded? Why might one executor fail when others succeed?

### Scenario 5. The driver crashes during collect.

**Interview problem:** A developer calls collect to pass all transformed orders into a Java collection and the driver crashes.

**Clarify first:** Does the consumer really require all rows in one JVM, or only a sample, aggregate, or persisted output?

**Diagnosis and resolution:** Confirm the driver OOM and the size of the returned result. collect materializes every result row in driver memory and transfers it over the network; executor memory does not protect the driver from this. Keep large processing distributed: write to a distributed sink, aggregate before collecting, or use a bounded sample. For example, use take(20) for debugging rather than collecting the entire Dataset. A count may still scan a large input but returns only a scalar, so it has a very different driver-memory profile.

**Trade-offs and prevention:** A bounded action is safer but still triggers a job. Add code-review guidance against unbounded collect and inspect driver/executor logs separately.

**Follow-ups:** Why is show safer but not free? When is collect appropriate?

### Scenario 6. A broadcast join unexpectedly causes memory pressure.

**Interview problem:** A join was changed to broadcast a dimension table and executors begin experiencing long GC pauses.

**Clarify first:** What is the actual materialized broadcast size? Is the dimension filtered and projected? Did statistics or row counts change?

**Diagnosis and resolution:** Inspect the initial and final physical plans, broadcast exchange metrics, executor storage/execution pressure, and GC. The compressed file size is not the in-memory join representation size. Remove the hint or use a shuffle join if the table cannot safely fit on every executor. If broadcasting remains appropriate, reduce rows and columns before broadcast and correct stale statistics. Run the test at realistic dimension size, not with a tiny development sample.

**Trade-offs and prevention:** A shuffle join spends network/disk bandwidth; a broadcast join replicates data and uses per-executor memory. Establish a size budget and monitor broadcast metrics as dimension tables grow.

**Follow-ups:** What is the difference between a broadcast hint and a guarantee? Can AQE change the chosen join at runtime?

### Scenario 7. A job creates thousands of tiny output files.

**Interview problem:** A daily ETL creates 60,000 small Parquet files, and downstream queries spend more time listing/opening files than scanning data.

**Clarify first:** How many tasks ran? How many values exist for the output partition columns? Is the data volume itself small, or is the dataset fragmented across many input files?

**Diagnosis and resolution:** Inspect file count and size distribution, output partition columns, task count, and downstream scan metrics. Too many small tasks or too many directory partition values can create many files. Adjust the write partitioning to match the data volume and query patterns; compact existing files with a distributed rewrite when required. Avoid forcing a large dataset through one task with coalesce(1). That creates a serial bottleneck and may exhaust resources.

**Trade-offs and prevention:** Larger files improve scan efficiency but oversized files can limit parallelism; high-cardinality directory partitioning creates excessive directories and tiny files. Pick layout using actual query predicates and expected volume, then monitor file-size percentiles.

**Follow-ups:** Why is the number of output files not always equal to the input DataFrame partition count? What risks does coalesce(1) introduce?

### Scenario 8. Increasing partitions makes the job slower.

**Interview problem:** A team increases shuffle partitions from 200 to 2,000 and the runtime gets worse.

**Clarify first:** Did per-task work fall? Did scheduling overhead, tiny files, or shuffle metadata increase? Is the problem present in every stage or only a particular operation?

**Diagnosis and resolution:** Compare tasks per stage, median task duration, scheduler delay, shuffle bytes, spill, and executor slot utilization. Too few partitions create oversized tasks; too many tiny partitions create scheduling and file overhead. The best count varies by stage and volume. Start with a measured baseline, adjust the relevant shuffle setting, and consider AQE's post-shuffle coalescing where enabled. A setting that improves a large join may be poor for a smaller aggregation.

**Trade-offs and prevention:** AQE may merge small partitions after runtime statistics become available, but cannot make every stage optimal. Record configurations and compare equivalent datasets while checking output correctness.

**Follow-ups:** What is the difference between input partition sizing and shuffle partition count? How would you know whether the workload lacks parallelism?

### Scenario 9. A large input produces a single bottleneck partition.

**Interview problem:** A large source takes hours because the first stage has only one effective task.

**Clarify first:** Is the file unsplittable, did the code call coalesce(1), or is a key-based operation concentrating all rows for one key?

**Diagnosis and resolution:** Inspect the source format, physical plan, input partition count, and earlier transformations. A single compressed non-splittable input file can limit scan parallelism; split or rewrite it into splittable files. Remove unnecessary single-partition operations. If the issue is a hot key after a shuffle, improve the key distribution or use a skew-aware design; simply repartitioning the source cannot split the final logical group of a normal aggregation.

**Trade-offs and prevention:** Rewriting source data costs I/O and storage, but may pay off across repeated queries. Validate source layout and track input files/partition sizes.

**Follow-ups:** Why can repartition improve scan parallelism only after the scan has already completed? How does key skew differ from one large source file?

### Scenario 10. A Spark job works locally but fails in a cluster.

**Interview problem:** A Java Spark application runs on a laptop but fails on Kubernetes with class errors or storage access errors.

**Clarify first:** Are the JDK, Spark version, Scala binary version, and connector artifacts compatible? Can the driver and executors reach the data source? Are credentials and service permissions available to both?

**Diagnosis and resolution:** Separate code errors from environment errors. Compare driver/executor logs, classpaths, dependency scopes, image versions, network/DNS, identity permissions, and container limits. Local mode often hides serialization and distributed-network issues. If a managed cluster supplies Spark libraries, bundling conflicting Spark jars in the application artifact can cause linkage errors. Include connector dependencies compatible with the exact Spark/Scala distribution, and test the packaged artifact in a representative cluster.

**Trade-offs and prevention:** Local tests are fast but do not prove cluster readiness. Add deployment smoke tests for source access, sink access, serialization, and checkpoint permissions.

**Follow-ups:** Why does a driver-only classpath fix not necessarily fix executor failures? What differs between client and cluster deployment modes?

### Scenario 11. AQE changes a query plan unexpectedly.

**Interview problem:** The initial plan shows a sort-merge join, but the running query switches strategies and performance changes.

**Clarify first:** Is AQE enabled? What runtime statistics were observed? Did actual relation size differ from estimates or was a shuffle partition skewed?

**Diagnosis and resolution:** Compare the initial plan, final adaptive plan, runtime statistics, and stage metrics. AQE can coalesce shuffle partitions, handle qualifying skewed joins, and revise certain join strategies based on runtime information. Determine whether the new plan is actually worse before changing settings. If the estimate was wrong, improve statistics or reduce data before the join; if adaptive coalescing causes inadequate parallelism, benchmark a targeted configuration adjustment.

**Trade-offs and prevention:** Disabling AQE may remove useful protections and should be a controlled diagnostic, not the first fix. Store representative plan/metric baselines for important queries.

**Follow-ups:** What does AQE optimize? Why can it not fix an incorrect join predicate or enormous output cardinality?

### Scenario 12. A transformation appears to run twice.

**Interview problem:** A costly upstream scan occurs during both a preview and a final write.

**Clarify first:** How many actions occur? Is the query being retried, or is this simply separate actions over the same uncached lineage?

**Diagnosis and resolution:** Inspect jobs in the Spark UI and trace all actions in application code. Calling show(), count(), then write() can trigger separate executions. Remove diagnostic actions from the production path or persist the expensive intermediate if it will genuinely be reused.

~~~java
Dataset<Row> reusable = expensiveTransform(source)
    .persist(StorageLevel.MEMORY_AND_DISK());
reusable.count(); // populate cache only if reuse justifies it
// Use reusable in multiple downstream actions.
reusable.unpersist();
~~~

Persistence occupies storage and may evict data or increase spill, so it is not automatically a win. Also distinguish recomputation from task retries and streaming replay; the remedies differ.

**Trade-offs and prevention:** Keep actions purposeful and benchmark cache population versus recomputation.

**Follow-ups:** Does cache eagerly execute the pipeline? Why does an uncached DataFrame not represent a materialized result?

### Scenario 13. Kafka lag grows continuously in a streaming query.

**Interview problem:** Source lag rises for hours even though the query remains active.

**Clarify first:** Is arrival rate greater than processing rate? Is batch duration increasing? Did Kafka volume, state size, join cost, or sink latency change?

**Diagnosis and resolution:** Track input rate, processing rate, batch duration, trigger delay, source partition distribution, shuffle/spill, state-store commit time, and sink duration. If input processing is limiting, optimize parsing/joins and provide adequate task parallelism. If Kafka is the source bottleneck, remember that source partitions constrain parallelism by default, though the Kafka connector's minPartitions option can split large offset ranges into more Spark tasks. If the sink is slow, improve bulk writes or scale the sink rather than adding Spark executors blindly.

For bounded admission, the Kafka source supports options such as maxOffsetsPerTrigger; this controls the amount admitted per trigger, not the query's fundamental throughput.

~~~java
.option("maxOffsetsPerTrigger", "500000")
~~~

**Trade-offs and prevention:** Rate limits can protect a sink but make backlog grow if sustained capacity is insufficient. Alert on lag, batch duration, and time-to-catch-up.

**Follow-ups:** Why might more executors fail to reduce Kafka lag? How would you calculate catch-up capacity after an outage?

### Scenario 14. A streaming restart appears to duplicate output.

**Interview problem:** After a failure, the stream resumes from its checkpoint but a database or downstream topic contains duplicate events.

**Clarify first:** Are duplicates in Spark's internal aggregate, at the sink, or merely in a downstream view? Did the sink commit before Spark durably recorded the batch as complete?

**Diagnosis and resolution:** Correlate batch IDs, source offsets, query/run IDs, sink transaction records, and event IDs around the failure timestamp. Replay can be expected when a failure occurs between an external side effect and progress recording. Preserve the checkpoint. Make the sink idempotent using a stable event key, unique constraint/upsert, transactional batch marker, or supported transactional protocol. Test crashes before, during, and after sink commit.

**Trade-offs and prevention:** Event-level upserts can protect individual records but require conflict/version rules for out-of-order updates. A batchId marker alone is insufficient if partial writes can occur without an atomic transaction.

**Follow-ups:** What exactly is meant by exactly once at this sink boundary? Why is deleting the checkpoint not a safe duplicate-removal strategy?

### Scenario 15. Late events are missing from a window aggregation.

**Interview problem:** Five-minute order totals are lower than expected because some events arrive several minutes late.

**Clarify first:** What is the event-time distribution and business lateness SLA? Is the event timestamp parsed in the right timezone? Are late records rejected at ingestion or too late for retained state?

**Diagnosis and resolution:** Inspect sample event timestamps, watermark progress, window boundaries, output mode, and late-event metrics/logging. Use a watermark only after defining the acceptable delay. A watermark uses observed event-time progress and the configured delay; it is not a fixed timer for every row. With supported window aggregations, data arriving too late to update state may not affect final results. A larger delay retains state longer and tolerates more lateness; a shorter delay reduces state but can exclude legitimate events. For financial or audit-sensitive metrics, consider a correction/reconciliation path for very late data.

~~~java
events.withWatermark("event_time", "20 minutes")
    .groupBy(window(col("event_time"), "5 minutes"), col("merchant_id"))
    .agg(sum("amount").alias("revenue"));
~~~

**Trade-offs and prevention:** Validate event clocks and monitor late-event distribution against the chosen delay. The precise Spark guarantee compares event-time progress, not a fixed wall-clock arrival stopwatch: within-threshold data is guaranteed not to be dropped as late for supported window aggregations; beyond-threshold data may still be processed if state remains.

**Follow-ups:** What is the difference between event-time delay and real elapsed arrival delay? Why can an event older than the watermark still appear in an aggregate?

### Scenario 16. Watermarks do not reduce state as expected.

**Interview problem:** The state store grows despite a configured watermark.

**Clarify first:** Is event time advancing? Are timestamps null or far in the future/past? Does the stateful operation use the watermarked column? Is this a multi-stream query?

**Diagnosis and resolution:** Inspect current watermark, state rows/bytes, update/commit duration, null timestamp rate, and input event-time distribution. A watermark does not advance merely because wall-clock time passes; it depends on observed event-time progress. In a multi-stream query, global watermark behavior and lagging inputs can affect cleanup. Also confirm the operation and query shape support state eviction. Fix malformed timestamps or semantic issues before shortening the threshold.

**Trade-offs and prevention:** Shorter thresholds can clean state earlier but change late-data correctness. Alert on growing state coupled with stalled watermark progress, not just on absolute state size.

**Follow-ups:** What is the difference between processing-time passage and event-time progress? Why may a watermark not bound state for every possible stateful query?

### Scenario 17. The streaming state store grows excessively.

**Interview problem:** State size and batch duration rise every day in a keyed aggregation.

**Clarify first:** Is the key cardinality unbounded? Are windows defined? Does the watermark advance? Is the application retaining all customer IDs by design?

**Diagnosis and resolution:** Track state rows/bytes per operator, state update/commit durations, watermark, key cardinality, and duplicate horizon. If the query is an unwindowed aggregation by an ever-growing key, all unique keys may need to be retained, so a watermark may not make that state expire as expected. Redesign around bounded windows or a domain-specific retention rule only if business semantics permit. Reduce key cardinality or process a coarser grain where acceptable. Increasing executor memory cannot solve logically unbounded state.

**Trade-offs and prevention:** State cleanup changes which future late events can update results. Load-test with peak cardinality and include state growth, checkpoint time, and recovery duration in capacity planning.

**Follow-ups:** Why is a watermark not a universal state TTL? Which metrics distinguish high input volume from ineffective cleanup?

### Scenario 18. The sink is slower than the incoming source.

**Interview problem:** Spark processes records quickly until it writes them to a relational database; backlog increases.

**Clarify first:** What fraction of each batch is spent in the sink? Are database connections saturated, locks high, or writes one row at a time? Does adding parallel tasks increase database pressure?

**Diagnosis and resolution:** Measure sink latency and batch duration separately. Compare Spark input and compute metrics against database CPU, I/O, lock waits, connection count, and transaction duration. Use bulk/batched writes, bounded parallelism, staging tables plus a merge, or a sink-specific bulk API. Tune connection pools and concurrency in coordination with database capacity. Do not let every Spark task create unconstrained connections.

**Trade-offs and prevention:** More parallel writes may improve throughput until the database saturates, after which retries and locks amplify the incident. Set a sink throughput budget and alert on latency, backlog, connection saturation, and retry rate.

**Follow-ups:** Why can more Spark executors make the database less stable? How would you preserve idempotency when using a staging table?

### Scenario 19. The checkpoint directory becomes unavailable.

**Interview problem:** A streaming query cannot read or write its checkpoint because object storage credentials expired or the location was removed.

**Clarify first:** Is this a temporary permission/network issue, or is checkpoint data permanently lost or corrupted?

**Diagnosis and resolution:** Stop uncontrolled restart loops, preserve driver logs and checkpoint metadata, and inspect storage health, IAM/credentials, lifecycle policies, and path access. If the original checkpoint is intact, restore access and restart with the same compatible query/checkpoint. If checkpoint state is truly lost, treat it as a recovery incident: select a known source boundary, assess source retention, reconstruct state by replay/backfill if feasible, and use idempotent sinks plus reconciliation.

Do not create an empty checkpoint at the same path and claim the stream will resume from its old progress. A fresh checkpoint is a new query history and may replay or omit expected history depending on source options and retention.

**Trade-offs and prevention:** Checkpoints are production state, not disposable logs. Protect them with least-privilege durable storage, retention controls, monitoring, backups/restore strategy where applicable, and recovery drills.

**Follow-ups:** Can Spark recover records Kafka has already expired? What evidence is needed before deciding to replay?

### Scenario 20. A streaming query fails after a schema change.

**Interview problem:** A producer deploys a new payload field or changes a field type, and the downstream query fails or silently produces nulls.

**Clarify first:** Was a field added, removed, renamed, or changed type? Is the format parsed against an explicit schema? Is the failure in parsing, analysis, state schema compatibility, or sink schema?

**Diagnosis and resolution:** Inspect the failing batch, schema version, parse modes, and error samples with source topic/partition/offset. A new optional JSON field is not equivalent to changing amount from a number to a string. Prefer an explicit schema and a controlled compatibility policy. Quarantine malformed records where safe; fail fast if continuing would corrupt critical metrics. If a stateful query's state schema or logical plan is incompatible with its existing checkpoint, plan a deliberate migration or replay rather than deleting the checkpoint casually.

**Trade-offs and prevention:** Flexible parsing can preserve availability but risk silently dropping business data. Add producer/consumer contract tests, schema compatibility checks, staged rollout, and malformed-record alerts.

**Follow-ups:** Why can syntactically valid JSON fail schema parsing? Why is checkpoint compatibility especially sensitive for stateful queries?

### Scenario 21. A Kafka event cannot be deserialized.

**Interview problem:** A single bad message repeatedly fails parsing and blocks progress or inflates error logs.

**Clarify first:** Is the payload invalid JSON, wrong encoding, an unknown schema version, or missing a required business field? Can the record be safely skipped?

**Diagnosis and resolution:** Preserve topic, partition, offset, event ID if available, schema/version headers, and a sanitized sample. Compare the actual payload with producer contract and parser schema. Route invalid messages to a quarantine/dead-letter dataset with a reason and replay metadata, or fail the pipeline when continuing could produce incorrect business output. Do not log credentials or sensitive full payloads. Make quarantine itself replay-safe so a retry does not flood duplicate error records.

**Trade-offs and prevention:** Skipping poison records protects availability but creates an explicit data-quality gap that must be measured and repaired. Alert on rejected rates and keep a controlled replay mechanism.

**Follow-ups:** How do you avoid infinite retries for a poison record? Which source metadata must be preserved for replay?

### Scenario 22. Batch and streaming implementations produce different results.

**Interview problem:** A batch recomputation reports a different daily total from the live streaming aggregate.

**Clarify first:** Do both use the same source boundary, timezone, null policy, deduplication key, event-time windows, and late-data rules?

**Diagnosis and resolution:** Compare a small deterministic set of events at window boundaries, including duplicates, missing timestamps, late arrivals, and out-of-order updates. A batch query over all history can include records that the streaming query has already finalized and removed from state under its watermark policy. Also inspect differences in schema parsing and join cardinality. Define business semantics first, then make both implementations follow the same rule or document why they intentionally differ. Reconcile live totals with periodic backfills where the business requires correction.

**Trade-offs and prevention:** A streaming result may prioritize bounded latency/state while a backfill computes from complete retained history. Differential tests should verify known edge cases, not just happy-path records.

**Follow-ups:** Why isn't batch dropDuplicates operationally equivalent to streaming deduplication? How would you reconcile late corrections?

### Scenario 23. A pipeline requires idempotent database writes.

**Interview problem:** A micro-batch can partially write records, fail, and retry; repeated inserts must not create duplicates.

**Clarify first:** What uniquely identifies a business event? Can the database enforce uniqueness? Are updates versioned or can events arrive out of order?

**Diagnosis and resolution:** Map the failure window between database side effects and Spark's recorded batch progress. Design a write that converges when repeated: unique event key plus upsert, uniqueness constraint, version-aware merge, or a transactional protocol that commits output and batch marker together. With foreachBatch, batchId is useful for identifying a batch, but it does not itself make record writes idempotent if a partial commit can happen before the marker is recorded.

~~~sql
MERGE INTO order_events AS target
USING staged_events AS source
ON target.event_id = source.event_id
WHEN NOT MATCHED THEN INSERT (event_id, order_id, status)
VALUES (source.event_id, source.order_id, source.status);
~~~

This is illustrative SQL; MERGE support/syntax depends on the target database or table format. Test failures before, during, and after commit.

**Trade-offs and prevention:** Unique constraints protect correctness but can create contention; upserts need explicit conflict/version rules. Monitor duplicate-key conflicts and replay outcomes.

**Follow-ups:** Why is batchId alone not a sufficient per-event key? What if two versions of the same order arrive out of order?

### Scenario 24. One hot key dominates join runtime.

**Interview problem:** A tenant ID represents 35% of the data and dominates a shuffle join.

**Clarify first:** Is the skew on one side or both? Is the join many-to-many? Can the hot-key records be aggregated or treated separately?

**Diagnosis and resolution:** Profile key frequencies on both inputs and compare shuffle read for tasks. Validate expected output multiplicity before changing the algorithm. Use AQE skew-join handling when eligible. Otherwise, handle the hot key separately, salt the join key and replicate the smaller-side matches across salt buckets as designed, or pre-aggregate if semantics permit. For aggregation, combine partial totals by (key, salt), then sum by the original key.

**Trade-offs and prevention:** Salting adds data and another phase; an incorrect salt assignment can lose matches or multiply output. Prefer measured targeted handling over random repartitioning and monitor key distribution over time.

**Follow-ups:** Why is salting aggregation different from salting a join? What validation proves no rows were lost or multiplied?

### Scenario 25. Most runtime is garbage collection.

**Interview problem:** Executor task time is high, and the UI shows a large fraction spent in GC.

**Clarify first:** Is this isolated to a stage or all stages? Is the workload object-heavy, caching too much, broadcasting a large relation, or processing skewed partitions?

**Diagnosis and resolution:** Compare GC time, heap occupancy, task duration, cache size, broadcast metrics, and spill. Inspect user code for object creation and conversions, including conversion from optimized DataFrames to collections/RDDs. Prefer built-in SQL expressions where possible, reduce row/column volume, remove unused caches, and fix skew or oversized broadcasts. Only tune heap/GC after measuring allocation and resource limits.

**Trade-offs and prevention:** A larger heap may reduce frequency but increase pause duration. Add GC and spill to performance baselines and test with production-like object/key distributions.

**Follow-ups:** Why can a DataFrame expression allocate less than a Java object-heavy RDD pipeline? What other symptoms distinguish GC pressure from network wait?

### Scenario 26. Shuffle fetch failures recur.

**Interview problem:** Downstream tasks repeatedly fail to fetch shuffle blocks, causing stage retries and eventual job failure.

**Clarify first:** Are errors tied to a particular executor, node, local disk, network path, or all workers? Does the upstream executor disappear after writing shuffle output?

**Diagnosis and resolution:** Read the exact fetch error, executor-loss events, disk utilization, node logs, and network timeouts. A shuffle consumer may fail because the producer's local shuffle files disappeared after executor/node loss or storage failure. Fix the underlying disk/network/node instability and check the cluster's shuffle preservation configuration. Retry settings may help transient failures, but raising them cannot restore a repeatedly deleted shuffle block.

**Trade-offs and prevention:** Robust shuffle preservation has operational/resource costs. Monitor disk space, executor churn, fetch failures, and host-level health, and test recovery after executor loss.

**Follow-ups:** Why can a downstream stage fail after upstream tasks reported success? When is retry tuning appropriate?

### Scenario 27. The Spark UI shows many tiny tasks.

**Interview problem:** A stage schedules tens of thousands of tasks, each processing only a few megabytes.

**Clarify first:** Are tiny input files causing the task count, or is the shuffle partition count unnecessarily high? Are tasks empty, and does scheduler delay dominate compute time?

**Diagnosis and resolution:** Inspect stage task count, input size per task, scheduler delay, source file sizes, and shuffle metrics. For file input, Spark uses file-splitting/packing settings; shuffle partition count is controlled separately. Compact tiny files, adjust relevant settings based on measurement, or allow AQE to coalesce eligible shuffle partitions. Preserve enough parallelism for genuinely large stages.

**Trade-offs and prevention:** Over-coalescing creates long tasks and poor parallelism; compaction costs a rewrite and should align with downstream access patterns. Track task-size distributions.

**Follow-ups:** Why are file input partitions different from shuffle partitions? Which metric tells you scheduling overhead is material?

### Scenario 28. A large aggregation exhausts memory.

**Interview problem:** A groupBy over event data spills heavily or fails on high-cardinality dimensions.

**Clarify first:** How many distinct groups are expected? Is one key skewed? Can the business report use a coarser grain or filter before aggregation?

**Diagnosis and resolution:** Inspect aggregate operators, spill, task distribution, and key cardinality. Spark can perform partial aggregation before the shuffle, but a high-cardinality key space still requires large intermediate state and output. Reduce rows/columns earlier, pre-aggregate in a semantics-preserving way, fix skew, or adjust partitioning. If the required output itself contains an enormous number of distinct groups, acknowledge that output and state costs are fundamental; memory tuning alone will not make it cheap.

**Trade-offs and prevention:** Coarser aggregation changes the product's granularity and needs business agreement. Estimate group cardinality and test peak loads.

**Follow-ups:** Why can partial aggregation fail to reduce data much? What is the difference between input volume and output cardinality?

### Scenario 29. A streaming pipeline must recover after several hours of downtime.

**Interview problem:** The stream has been down for six hours and must catch up without losing events or overloading the sink.

**Clarify first:** Is the checkpoint intact? Does Kafka retention still cover the required offsets? How much backlog exists, and can the sink sustain catch-up traffic?

**Diagnosis and resolution:** Validate checkpoint and source offsets, earliest retained Kafka offsets, query/state compatibility, and sink capacity. Resume from the valid checkpoint rather than choosing fresh startingOffsets. Estimate backlog divided by sustainable excess throughput to understand catch-up time. If processing can exceed current arrival rate safely, allocate capacity for catch-up; otherwise throttle admission and accept a longer recovery window. If Kafka retention has expired required data, recover from a durable archive/backfill and explicitly reconcile the gap.

**Trade-offs and prevention:** Aggressive catch-up can overwhelm downstream databases. Longer Kafka retention costs storage, but protects the recovery window. Alert on lag, time-to-catch-up, and retention risk.

**Follow-ups:** Can a checkpoint recover Kafka records already deleted by retention? How would you prevent a catch-up surge from taking down the sink?

### Scenario 30. Kafka input volume increases tenfold.

**Interview problem:** Traffic spikes by an order of magnitude and the pipeline's batch duration exceeds the trigger interval.

**Clarify first:** Is the spike a burst or sustained? Are topic partitions balanced? Is parsing, shuffle, state, or sink throughput limiting?

**Diagnosis and resolution:** Compare event size and rate, per-partition traffic, processing rate, batch duration, shuffle/spill, state-store growth, and sink metrics. Scale the layer that limits sustainable throughput. More source parallelism may require increasing Kafka partitions or using the connector's minPartitions for large offset ranges, but this has key-ordering and operational implications. Optimize parsing and joins, and ensure downstream systems can accept the resulting concurrency.

**Trade-offs and prevention:** Backpressure/rate limiting protects downstream systems but backlog grows if average input remains above processing capacity. Load-test bursts and sustained load separately and define capacity headroom.

**Follow-ups:** How do you estimate required throughput? What does changing Kafka partition count do to per-key ordering and distribution?

### Scenario 31. The pipeline must deduplicate events reliably.

**Interview problem:** Producers retry events, and duplicates must not inflate order or payment metrics.

**Clarify first:** Is there a stable globally unique event ID? How late can duplicates arrive? Does the requirement demand lifetime deduplication or only a bounded replay horizon?

**Diagnosis and resolution:** Inspect duplicate rates and the producer's event identity contract. Use a stable event ID for streaming deduplication with a deliberate state-retention/watermark policy, and make downstream sink writes idempotent as a second layer. An order ID alone may not identify an event if an order legitimately emits multiple status changes. Preserve event version or sequence when update ordering matters.

**Trade-offs and prevention:** Longer duplicate horizons require more state and checkpoint I/O; bounded deduplication cannot promise suppression of duplicates arriving outside its retention horizon. Measure late duplicates and document the guarantee.

**Follow-ups:** What determines the deduplication horizon? Why should sink idempotency still matter if Spark deduplicates upstream?

### Scenario 32. Historical backfill must transition to live events.

**Interview problem:** The team must backfill a year of orders and then continuously process live Kafka events without a gap.

**Clarify first:** What durable historical source exists? Can the Kafka boundary be specified by offsets/timestamps? Can backfill and live processing overlap safely?

**Diagnosis and resolution:** Define an explicit cutover boundary and identical schema/transformation rules. One common pattern is to capture a source boundary, backfill the historical range, then stream from the corresponding offset range, deduplicating overlap with deterministic IDs. The precise sequence depends on source availability and retention; simply starting a stream after a batch finishes can leave a race window. Reconcile counts and totals around the boundary.

**Trade-offs and prevention:** Overlap is often safer than a gap if writes are idempotent, but increases compute. Document offset/timestamp boundaries and rehearse the handoff.

**Follow-ups:** How can an event arrive between choosing the boundary and starting the stream? What makes overlapping replay safe?

### Scenario 33. A Parquet dataset's schema changes over time.

**Interview problem:** New partitions add a field or change an integer to a string, and a downstream query either errors or returns nulls.

**Clarify first:** Is the change additive and compatible, or a rename/type change? Do all files have consistent schema metadata? Does the reader infer or impose a schema?

**Diagnosis and resolution:** Inspect schema per partition/file and test old plus new data together. A new optional field can often be normalized to a canonical schema; a type change needs explicit casting/validation and a migration policy. Schema merging is not a universal fix and can add planning cost. Do not silently cast malformed business values to null without counting/rejecting them.

**Trade-offs and prevention:** A canonical schema adds ingestion work but reduces surprise downstream. Use schema contracts, compatibility checks, and staged rollout of producer and consumer changes.

**Follow-ups:** Why can a file directory contain incompatible schemas? What risks arise from permissive casting?

### Scenario 34. A Spark application must run on Kubernetes.

**Interview problem:** Executors are repeatedly OOM-killed or cannot read checkpoint storage after deployment.

**Clarify first:** Which Spark image/JDK/connectors are used? Are service-account permissions, network policies, secrets, and storage credentials correct? What are the JVM heap and total pod limits?

**Diagnosis and resolution:** Inspect pod events, driver/executor logs, resource requests/limits, image versions, identity permissions, and storage/network reachability. Container memory includes JVM heap plus non-heap/native and process overhead. Align executor resources with cluster capacity, configure memory overhead appropriately, package compatible dependencies, and verify durable checkpoint access. Spark on Kubernetes has deployment-specific requirements; a local success is not enough.

**Trade-offs and prevention:** Larger requests improve headroom but reduce cluster packing and may increase cost. Build a staging smoke test, least-privilege access, and alerts for pod restarts/OOM kills.

**Follow-ups:** Why can a pod be killed without a Java heap OOM? What should be in the application artifact versus supplied by the cluster?

### Scenario 35. A pipeline writes to Kafka and Elasticsearch, but one sink fails.

**Interview problem:** The pipeline successfully writes Kafka output but fails halfway through updating Elasticsearch.

**Clarify first:** Are both sinks required to be atomically consistent? Which system is authoritative? Can each sink be rebuilt or replayed independently?

**Diagnosis and resolution:** Ordinary independent sink writes are not one distributed transaction. A Spark checkpoint does not atomically commit unrelated Kafka and Elasticsearch side effects. Prefer a durable source of truth and independent replayable sink consumers where architecture allows. Use deterministic Elasticsearch document IDs and version-aware upserts, unique event IDs for Kafka consumers, retry tracking, and reconciliation. If both writes happen in custom code, explicitly design the partial-failure state machine and make each operation idempotent.

**Trade-offs and prevention:** Independent sink pipelines add operational components but make replay and isolation clearer. Monitor each sink's lag, failures, document counts, and freshness separately.

**Follow-ups:** Why can't one checkpoint create a cross-system atomic commit? How would you rebuild Elasticsearch from Kafka or a data lake?

### Scenario 36. The database is overloaded after Spark scaling.

**Interview problem:** Adding executors increased throughput briefly, then database CPU, locks, and retries exploded.

**Clarify first:** Did task concurrency and connection counts increase? Is the database limited by CPU, I/O, locking, or connection pool capacity?

**Diagnosis and resolution:** Compare Spark's number of concurrent write tasks and retry rate with database connections, lock waits, CPU, I/O, and transaction sizes. Use bounded write concurrency, batch writes, staging plus merge, and a database-appropriate bulk interface. Establish backoff and retry limits that do not amplify overload. More parallelism is useful only up to the sink's sustainable capacity.

**Trade-offs and prevention:** Lower concurrency can increase total completion time but stabilize the service and improve end-to-end throughput. Test at expected peak volume and monitor queue depth, lock waits, and write latency.

**Follow-ups:** How do retries create a positive feedback loop? Why should sink capacity influence Spark parallelism?

### Scenario 37. A daily report's totals do not match the source system.

**Interview problem:** The Spark job succeeds, but the revenue total differs from the source of truth.

**Clarify first:** Are snapshots complete? Are there duplicates, null amounts, currency conversions, late events, timezone boundaries, or non-unique join keys?

**Diagnosis and resolution:** Reconcile record counts and sums at every stage: raw input, parsed data, filtered events, post-join rows, deduplicated records, aggregate output. Sample mismatching keys, verify parsing/types, and inspect join multiplicity and event-time cutoffs. A technically successful job can still have semantically wrong logic. Correct the rule, replay affected partitions idempotently, and compare totals before publishing.

**Trade-offs and prevention:** Additional data-quality checks consume some compute but catch silent corruption. Define tolerances only where business meaning allows; money totals often need exact reconciliation.

**Follow-ups:** Why can a non-unique dimension table inflate revenue? Which checks would you fail the pipeline on?

### Scenario 38. A new UDF makes a DataFrame query slower.

**Interview problem:** Runtime doubles after a transformation is moved into a custom Java UDF.

**Clarify first:** Does built-in Spark SQL have an equivalent expression? How many rows reach the UDF? Is the UDF in a filter/join key or simply a derived output?

**Diagnosis and resolution:** Compare physical plans, CPU time, code generation, and rows processed. Opaque UDF logic limits Catalyst's ability to reason about and simplify the expression. Replace it with built-in functions where possible, filter/project before applying it, or precompute expensive deterministic values if reuse justifies storage. If a UDF is necessary, benchmark it with realistic input and consider allocation and serialization.

**Trade-offs and prevention:** A UDF can still be appropriate for specialized logic; removing it is not an end in itself. Keep tests for nulls, type conversions, and deterministic behavior.

**Follow-ups:** Why does Spark distribution not automatically make a UDF efficient? What optimizations become harder around opaque functions?

### Scenario 39. A Parquet query scans far more data than expected.

**Interview problem:** A query filtered to one date reads hundreds of gigabytes.

**Clarify first:** Is the dataset directory-partitioned by the filtered date? Is the predicate expressed on the partition column directly? Are only necessary columns selected?

**Diagnosis and resolution:** Inspect the formatted plan for PartitionFilters and PushedFilters, and compare expected versus actual input bytes/files. A function or cast on a partition column can interfere with pruning in some query shapes. Express a compatible predicate and project only required columns. If recurring workload predicates do not align with the storage layout, consider a controlled rewrite with a sensible partition design; avoid high-cardinality directory partitions without evidence.

**Trade-offs and prevention:** A partition layout optimized for one query may hurt others or create small files. Use a workload-based design and monitor scan size per query.

**Follow-ups:** Distinguish partition pruning from predicate pushdown. What if a filter is logically correct but the data source cannot push it down?

### Scenario 40. An upstream producer emits out-of-order updates.

**Interview problem:** Multiple status events for the same order arrive out of order, and an older update can overwrite a newer one.

**Clarify first:** Does every event carry a monotonic version/sequence? Is event time reliable? Should late updates be corrected, ignored, or audited?

**Diagnosis and resolution:** Compare event time, ingestion time, business sequence, entity ID, and observed output. Event time alone is not a reliable ordering contract: clocks can be skewed, and events can be delayed. Use a deterministic business version rule in the sink, such as accepting an update only when its sequence is newer than the stored version. Preserve late updates for audit/reconciliation if required. Watermarks govern certain stateful event-time computations; they do not automatically define business conflict resolution.

**Trade-offs and prevention:** Version-aware upserts protect current state but may require a separate event history to retain every transition. Add producer contracts and tests for duplicates and reordered delivery.

**Follow-ups:** What happens if two updates have the same sequence? How would you distinguish “latest status” from “complete event history”?

---

### A repeatable incident-triage sequence

Use this order in interviews and production. It prevents premature configuration changes.

~~~mermaid
flowchart TD
  A[State expected result and SLO] --> B[Identify slow job, stage, or streaming batch]
  B --> C[Compare task duration, input, shuffle, spill, GC, and sink latency]
  C --> D{Where is time or backlog accumulating?}
  D -->|Scan / files| E[Check pruning, schema, file count, input layout]
  D -->|Shuffle / skew| F[Check join plan, cardinality, key distribution, partitions]
  D -->|Memory / GC| G[Check heap, overhead, cache, broadcast, task size]
  D -->|Streaming / sink| H[Check lag, rates, watermark/state, sink capacity, commit window]
  E --> I[Change one evidence-based factor]
  F --> I
  G --> I
  H --> I
  I --> J[Verify output correctness and compare the same metrics]
  J --> K[Add prevention: alerts, tests, runbook, or capacity guardrail]
~~~

The investigation is not complete merely because runtime improves. Confirm output counts, key uniqueness, aggregates, offset coverage, and replay behavior where those properties matter.

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
