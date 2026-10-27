# Elasticsearch & Search Systems Interview Handbook

**Audience:** Java backend engineers preparing for mid-level and senior backend interviews.  
**Approach:** 40 high-value questions with explanatory answers, 12 production scenarios, and 18 practical Query DSL patterns. The aim is understanding and technical reasoning, not maximizing the question count.

> **Version note:** REST examples use current Elasticsearch Query DSL style. Some settings and capabilities differ by release and deployment model. The Java example uses the official Java API Client style; align the client major version with your Elasticsearch server and the Java version supported by that client. Treat examples as patterns to test against your actual mappings and environment.

## How to study this handbook

Start with mapping and Query DSL. Many real Elasticsearch bugs come from a field mapped as the wrong type, using a term query on analyzed text, putting a required business condition in the wrong clause, or relying on unstable pagination. Then study search visibility, shards, aggregations, and operational diagnosis. Finish with the scenarios: explain what evidence you would collect, how you would prove the cause, what safe change you would make, and how you would prevent a recurrence.

## Contents

1. [Elasticsearch fundamentals](#1-elasticsearch-fundamentals)
2. [Indexing, Lucene, and mappings](#2-indexing-lucene-and-mappings)
3. [Query DSL: the questions interviewers ask most](#3-query-dsl-the-questions-interviewers-ask-most)
4. [Modern search, aggregations, and pagination](#4-modern-search-aggregations-and-pagination)
5. [Shards, performance, and production operations](#5-shards-performance-and-production-operations)
6. [Index lifecycle and safe migrations](#6-index-lifecycle-and-safe-migrations)
7. [Consistency, concurrency, and recovery](#7-consistency-concurrency-and-recovery)
8. [Java integration](#8-java-integration)
9. [SQL, Kafka, and search projections](#9-sql-kafka-and-search-projections)
10. [Production scenarios](#10-production-scenarios)
11. [Query DSL cookbook: 18 useful patterns](#11-query-dsl-cookbook-18-useful-patterns)
12. [Revision checklist and official references](#12-revision-checklist-and-official-references)

---

## 1. Elasticsearch fundamentals

### Q1. What is Elasticsearch, and how is it different from a relational database?

Elasticsearch is a distributed search and analytics engine built on Apache Lucene. It stores JSON documents and is especially good at full-text retrieval, relevance ranking, structured filtering, faceted navigation, and aggregations. An online store can let a user search for “wireless headphones”, tolerate a small typo, filter by brand and price, rank useful matches, and show category counts next to the results.

A relational database is usually the better source of truth for transactions, relational constraints, and business rules that must hold across records. A product catalogue might be copied into Elasticsearch as a denormalized document so search can return product name, brand, category, price, and availability together without performing joins for each request. The trade-off is duplicated data and asynchronous synchronization.

A common design is to keep business state in SQL and maintain Elasticsearch as a search-oriented projection. The application must define what happens when this projection is stale or unavailable. Search can help a customer discover a product, but checkout should revalidate authoritative stock, price, and eligibility in the transactional system.

### Q2. Explain cluster, node, index, document, field, mapping, and shard.

A cluster is a coordinated Elasticsearch deployment, and a node is one running member. An index is a logical collection of documents. A document is a JSON record, a field is one property inside that record, and a mapping defines how fields are interpreted and indexed.

A shard is a piece of an index. Elasticsearch distributes documents across primary shards, and each shard is a Lucene index with its own data structures. Searching an index can therefore mean searching several shards and merging their partial results. This is why shard layout affects both parallelism and coordination overhead.

In a product index, productId may be a keyword, name and description text, price and rating numeric, inStock boolean, and createdAt a date. The mapping is not just a schema description: it determines which queries make sense and which operations are efficient. If the query assumes the wrong mapping, syntactically valid DSL can still return no results or the wrong results.

### Q3. What are primary shards and replicas? How do you choose shard count?

Primary shards partition an index's documents. Replica shards are copies of primaries; they improve resilience and can serve search requests, but consume storage and also need to process writes. A primary and its replica copies are allocated to different nodes so that one node failure does not remove every copy of the same shard.

Choose shard count from expected indexed data size, write throughput, query concurrency, growth, and recovery requirements. There is no universally correct shard size. Too many tiny shards increase metadata, file-handle, heap, and coordination overhead; too few very large shards can limit parallelism and increase recovery time. Benchmark realistic indexing and search traffic rather than blindly applying a fixed rule.

Primary-shard topology is not a casual tuning knob after index creation. Changing it may require split or shrink operations, or a new index and reindexing. Replica count is more flexible, but adding replicas does not automatically solve a poor query or unsuitable mapping.

### Q4. What happens when a search request reaches Elasticsearch?

The node receiving a search acts as coordinator, even if it is not a dedicated coordinating node. It selects shard copies, sends shard-level requests, gathers partial hits and aggregation results, and reduces them into a final response. Each participating shard runs a local Lucene search. The coordinator merges the returned candidates into the requested sort order.

This fan-out model explains why one slow shard can dominate tail latency and why searching too many small shards can become expensive. Inspect timeout and shard-failure information in the response instead of assuming that an HTTP success guarantees a complete search. A request that returned partial results may not satisfy an API contract that promises complete results.

Writes follow a separate path. A document is routed to its primary shard, and the operation is replicated according to Elasticsearch's replication protocol. Acknowledgement, durability, and search visibility are related but distinct; a strong answer explains the differences rather than calling all three “consistency”.

### Q5. What is an inverted index, and why does Lucene use it?

A full-text search engine needs to answer the question “which documents contain this term?” without scanning every document. An inverted index maps terms to the documents—and often token positions—in which they occur. It makes text retrieval efficient because the query can locate candidate documents from the terms that were indexed.

Text is usually analyzed before it is indexed. For example, a text field containing “Wireless Headphones” may index lowercase tokens such as “wireless” and “headphones”. A keyword field generally indexes the whole value as one term, unless a normalizer transforms it. That difference is central to understanding match versus term queries.

Lucene also uses immutable segments and doc values. Segments are created as data is indexed and later merged. Doc values provide a column-oriented representation used by many sorting, aggregation, and scripting operations. Therefore, a field optimized for full-text retrieval is not automatically the right field for sorting or grouping.

---

## 2. Indexing, Lucene, and mappings

### Q6. What do indexing, refresh, translog, and flush mean? Why can a successful write be missing from search?

When Elasticsearch accepts a write, it records the operation in the translog for recovery and indexes the document into its internal indexing structures. A successful indexing response means the operation was accepted according to the request's acknowledgement semantics; it does not necessarily mean a search can retrieve the new document immediately.

A refresh opens newly written Lucene data for search. This is why Elasticsearch is near-real-time rather than strictly real-time for search visibility. The default refresh interval depends on deployment and index settings: it is 1 second in Elastic Stack and 5 seconds in Elastic Cloud Serverless by default. An index can override this setting. The refresh request option can force immediate visibility, wait for a scheduled refresh, or leave normal behavior alone. Forcing refresh on every write can create many small segments and reduce throughput, so use it only when the freshness requirement justifies the cost.

A flush is different: it commits Lucene changes and rolls the translog to a new generation so older translog data can eventually be removed. Do not use refresh, flush, and durability as synonyms. A concise explanation is: acknowledgement concerns the response to the write, translog and replication concern recovery, and refresh determines when search can see the change.

### Q7. Why use the Bulk API, and how should bulk indexing be tuned?

The Bulk API sends many index, create, update, or delete operations in one HTTP request. It reduces per-request overhead and is normally much more efficient than sending one request per document. It is useful for initial imports and high-volume event consumers.

Bulk is not an all-or-nothing transaction. A request can return successfully while individual documents fail. Inspect the response's item-level errors. Retry transient overload with bounded exponential backoff and jitter; a permanent mapping or parsing error will not fix itself through retries, so capture it in an observable dead-letter path for correction and replay.

Tune batch size and concurrency together. Increase batch size gradually until throughput stops improving or request latency and memory pressure become unhealthy. Huge batches can cause spikes, and excessive concurrent workers can saturate CPU, disk, merges, or network. Track documents per second, bulk duration, item failures, rejected requests, and live search latency during ingestion. The right settings come from a representative workload test, not a universal magic number.

### Q8. What is a mapping, and why explicitly map important fields?

A mapping defines each field's type and indexing behavior. Dynamic mapping can be convenient, but it infers types from early documents. A date string in an unexpected format, an identifier that looks numeric, or an arbitrary JSON property can lead to an inferred mapping that does not fit future data or query requirements.

Explicitly map identifiers, prices, dates, statuses, searchable text, and fields used for sorting or aggregations. For example, productId and brand typically need exact-match semantics, name needs full-text semantics, and price needs a numeric type for ranges and sorting. If a value must support both full-text and exact operations, use a multi-field instead of expecting one field mapping to do everything well.

Mapping errors often explain “zero results” or sort failures. A template change affects indices created with that template in the future; it does not rewrite the mapping of an existing production index. When staging succeeds but production fails, compare the actual mappings, templates, aliases, and indexed sample documents before changing the query.

### Q9. Explain analyzers, tokenizers, and token filters. How do you debug analysis?

An analyzer turns text into indexed or searched terms. Character filters can alter text before tokenization; a tokenizer splits it into tokens; token filters can lowercase, stem, remove stop words, normalize characters, or apply synonyms. The analyzer used at index time determines what terms are stored, while the search analyzer determines what terms a query asks for.

Suppose “Running Shoes” is indexed with an analyzer that lowercases text. It might be stored as the terms “running” and “shoes”. A term query using the original phrase does not analyze the input the same way a match query does, so the two queries can return very different results. This is one of the most common sources of confusion in interviews and production.

Use the Analyze API to see the actual output of an analyzer. Test punctuation, hyphens, plurals, accents, product identifiers, and synonyms. Avoid changing analyzers based on one example alone: define expected behavior with real query examples, choose the analyzer to produce those terms, and reindex when an index-time change requires rebuilding the indexed representation.

### Q10. What is the difference between text, keyword, multi-fields, object, and nested?

A text field is analyzed for full-text search. A keyword field treats a value as one term and is commonly used for exact filtering, sorting, and aggregations. A multi-field indexes the same source value in more than one way, for example name as text and name.keyword as keyword.

Object fields can be a trap with arrays of objects because their subfields are flattened for indexing. Imagine a product with variants red/size S and blue/size L. A query requiring color red and size L on an ordinary object mapping may match the parent document even though no single variant has both values.

A nested field stores each child object as a separate hidden document and lets a nested query require both predicates to match the same child object. Nested indexing and queries cost more, so use it only when same-object matching matters. Changing an existing field from object to nested usually requires creating a new index and reindexing.

---

## 3. Query DSL: the questions interviewers ask most

### Q11. What is the difference between match and term?

Use match for analyzed text. Elasticsearch analyzes the query input with the field's search analyzer and searches the resulting terms. This is the normal starting point for a product name, description, article body, or title.

Use term when you want to match an exact indexed term, commonly on keyword, numeric, boolean, or other structured fields. A term query does not analyze the query value. A term query for “Wireless Headphones” on a text field may not match because the field was indexed as separate lowercase tokens. Use match for the text field or term for an appropriate keyword sub-field, depending on the intended meaning.

Do not memorize the rule as “match is fuzzy and term is exact”. Match is not fuzzy by default; fuzziness must be requested. The important distinction is analysis. When results are surprising, inspect the field mapping and Analyze output, then select the query based on the indexed representation.

### Q12. Explain bool queries: must, filter, should, must_not, and minimum_should_match.

A bool query combines conditions. must is required and contributes to scoring. filter is required but does not contribute to relevance scoring. should represents optional scoring matches or alternatives that can be made mandatory. must_not excludes documents and runs in filter context.

For product search, put the user’s words in scoring clauses and conditions such as stock, brand, price, tenant, and product status in filter clauses. The user wants the most relevant matching product, but stock is an eligibility rule rather than a relevance signal. Mixing the two makes queries harder to reason about and can produce unintuitive ranking.

One subtle default is worth knowing: if a bool query has should clauses but no must or filter clauses, at least one should is required by default. If it has a must or filter clause, should clauses are optional by default. Set minimum_should_match explicitly when the business rule requires some of the alternatives. This prevents a common bug where adding a filter accidentally turns previously required alternatives into optional boosts.

### Q13. What is query context versus filter context? Are filters always cached?

In query context Elasticsearch asks how well a document matches and calculates a relevance score. Full-text matching usually belongs here. In filter context Elasticsearch asks only whether the document meets a condition. Price ranges, exact status, tenant scope, stock eligibility, and date windows usually belong in filter context.

Filters can be faster for structured conditions because scoring is skipped, and they may benefit from the query cache. But filter context does not mean every filter is cached. Cache behavior depends on query type, usage, segment lifecycle, and implementation details. Saying “filters are always cached” is inaccurate.

Combine the two contexts: score candidates by the text relevance that matters to users, then apply mandatory structured filters. Choose filter context because a condition should not affect the score, not because every filter is guaranteed to be faster.

### Q14. How do term, terms, range, exists, and ids queries differ?

term matches one exact indexed value; terms matches any value from a set; range matches values within numeric or date bounds; exists finds documents with an indexed value for a field; and ids targets Elasticsearch document IDs. For example, a product search can combine full-text matching with filters for brand, price range, stock, and a list of allowed categories.

exists does not always mean “the JSON property appears in the original source”. Null values, empty arrays, and values excluded from indexing can affect whether Elasticsearch considers a field to have an indexed value. Range queries also depend on field mapping and date semantics, including boundary inclusivity and timezone handling.

These clauses normally define which documents are eligible; they are not usually the main relevance signal. Put them in filter context unless their score is deliberately meaningful. If one appears to behave incorrectly, inspect the source document, mapping, and indexed representation rather than immediately adding scripts.

### Q15. When should you use match_phrase, multi_match, and field boosts?

A match query searches analyzed terms. match_phrase is stricter about the ordered phrase and token positions. multi_match searches multiple fields with configurable combination behavior. Field boosts express that a match in a product name may be stronger evidence of relevance than a match buried in a long description.

A practical baseline might search name with a higher boost and description with a lower boost, then add a phrase clause for exact name-like matches. Do not make every clause mandatory without a clear product requirement: overly strict queries may improve precision for one query while hiding useful results for another.

Boosts are tuning hints, not guaranteed multipliers of business relevance. The final score depends on the query structure, term statistics, similarity algorithm, and other matching clauses. Collect real search queries and judged good results; adjust boosts based on observed ranking quality rather than intuition alone.

### Q16. When should you use fuzzy, wildcard, regexp, or query_string queries?

Fuzzy matching helps with typos but expands nearby terms and costs more than exact matching. Use it selectively, especially for short inputs and high-volume autocomplete. It is not a replacement for good mappings, analyzers, or a deliberate suggestion strategy.

Wildcard and regexp queries can be expensive when they expand to many terms. Patterns beginning with a wildcard are particularly risky at scale. If the requirement is prefix search, a prefix-aware mapping or autocomplete approach is often better than broad wildcard matching. Do not expose unrestricted patterns to a public endpoint without strict limits.

query_string supports an explicit query language, including operators and field syntax. It can be useful for advanced search, but arbitrary input can produce parse errors or surprising semantics. simple_query_string handles invalid syntax more leniently but still exposes query-language behavior. For a normal search box, building a match or multi_match query in backend code is generally safer than passing raw user input as a query string.

### Q17. How does BM25 scoring work, and what does the score tell you?

BM25 is a common default similarity algorithm for full-text fields. It considers how often query terms occur in a document, how informative the terms are across the collection, and how long the field is relative to typical field length. In practical terms, a rare and meaningful term often provides stronger evidence than a very common term; repeated occurrences have diminishing influence.

The score is a ranking signal within a query, not a universal quality score or percentage. A document may get a different score with a different query or after corpus statistics change. Do not compare raw scores across unrelated queries as if they were directly comparable. Matching must and should clauses can add score contributions, while ordinary filters do not.

When ranking looks wrong, inspect the query, mapping, analyzer, boosts, synonyms, phrase matches, and documents that rank above the expected result. Explain helps understand one document's score. Use a small set of real queries with judged expected results to evaluate a change, because syntactically correct DSL does not guarantee useful relevance.

### Q18. How do you debug irrelevant results or a search that returns zero hits?

First reproduce the exact request and a small known set of documents. Check the target index or alias, the existence of the document, actual mappings, and the analyzed terms. A term query on text, a wrong field name, a changed analyzer, or a filter that is too restrictive can explain zero hits without any cluster failure.

If too many weak results appear, inspect default OR behavior, boosts, phrase requirements, and whether should clauses have become optional. Use Explain for a misranked document and Profile selectively for query execution cost; they answer different questions. Explain helps with scoring, while Profile helps understand where a representative search spends time and adds overhead itself.

Make the fix measurable. Keep regression queries with expected relevant results, compare rankings before and after the change, and monitor search quality signals where appropriate. Do not tune relevance using one example alone.

### Q19. How should autocomplete be designed?

Choose the mechanism based on user experience. For suggestions of known entities, the completion suggester may fit. For matching prefixes across multiple words, search_as_you_type or edge n-grams may be more suitable. A phrase-prefix query can be enough for a simpler experience. These options differ in indexed data, memory use, matching behavior, and typo support.

Autocomplete should return a small result set quickly. Debounce keystrokes, limit results, avoid unnecessary total-hit counts, and query only the fields needed for suggestions. Typo tolerance can help but can also multiply work and produce noisy results when applied to every keystroke.

Evaluate with realistic partial input, including short prefixes, multiple words, misspellings, and identifiers. Test tail latency and whether the top suggestions are useful. Choose the design from measured relevance and latency, not because one demo query looked good.

---

## 4. Modern search, aggregations, and pagination

### Q20. When should you use vector search or hybrid search instead of traditional full-text search?

Traditional lexical search, commonly ranked with BM25, is strong when the words themselves matter: product IDs, error messages, names, exact phrases, and terms users expect to match. Vector search represents content as embeddings and retrieves documents by semantic similarity. It can find conceptually related content even when the query and document use different wording, but it may be weaker for exact identifiers or strict wording.

Hybrid search combines lexical and vector retrieval. One practical method is reciprocal rank fusion (RRF), which combines the rankings from the two retrieval methods without requiring their raw scores to be directly comparable. Elasticsearch's supported hybrid-search APIs and syntax depend on the server release and deployment, so verify the exact implementation against the target version.

Do not add vectors just because they are fashionable. Embeddings introduce model choice, inference cost, indexing/update workflows, latency, storage, and another relevance dimension to evaluate. Keep eligibility rules such as tenant, access control, and availability as explicit filters, and compare lexical-only, vector-only, and hybrid results on a judged query set. For many backend search applications, a strong lexical baseline plus measured hybrid improvements is safer than replacing lexical search wholesale.

### Q21. What are aggregations? Explain bucket versus metric aggregation.

Aggregations summarize the documents matched by a search request. A bucket aggregation groups documents, such as one bucket per brand or category. A metric aggregation calculates values such as average price, minimum, maximum, sum, or statistics. Aggregations can be nested to answer questions such as “for the matching products, show the top categories and average rating in each category”.

This is how search facets work: a user searches for headphones and the response also reports how many matching products belong to each brand or price band. Put filters in the query when they should constrain both hits and aggregations; use a filter aggregation when only one aggregation needs a narrower condition.

Aggregations consume resources. High-cardinality fields, deeply nested buckets, or huge result sets can cause memory pressure and expensive coordinator reductions. Bound the number of buckets and return only what the interface needs. When you must enumerate many distinct buckets, consider composite aggregation rather than setting terms size to a huge number.

### Q22. Why can a terms aggregation return approximate counts? When would you use composite aggregation?

A terms aggregation often returns the top buckets by document count. Each shard sends candidate buckets to the coordinating node, which merges them. A term that sits just below the local candidate cutoff on several shards could still have a large global count; if those shards do not return it, the merged count may be incomplete. Increasing shard_size can improve candidate coverage for the default count-descending order, but costs network traffic and coordinator memory and does not guarantee accuracy in every ordering.

Inspect response metadata such as count-error information and sum_other_doc_count instead of treating every facet as an exact group-by over all possible values. If exact business reporting is required, validate the aggregation's guarantees and error behavior for that workload.

Composite aggregation is designed to paginate through many distinct buckets or combinations, such as every category-and-brand pair. For a small top-N facet on a search page, terms aggregation is usually simpler and more appropriate.

### Q23. How do from/size, search_after, PIT, and scroll differ?

from/size is easiest for shallow page-number pagination. For a deep page, each shard must collect candidates for skipped hits as well as the page being returned, which increases CPU and memory. The default result window is commonly 10,000 hits. Raising it is not the general fix for deep pagination.

For deep sequential traversal, use search_after. Sort the hits, then send the complete sort values from the last hit with the next request. Keep the query and sort unchanged, and use deterministic ordering. If refreshes occur between requests, documents can move between pages. A point in time (PIT) preserves a stable view during paging; use the latest returned PIT ID and preserve the full sort array, including the tie-breaker Elasticsearch returns when PIT is used.

Scroll is primarily for batch processing of large result sets, not interactive page navigation. Current Elasticsearch guidance favors PIT plus search_after for deep pagination requiring a consistent view. Keep the PIT lifetime bounded and close it when finished.

### Q24. Why does deterministic sorting matter?

If multiple documents have identical values for all sort fields, their order can change between requests or across shard copies. That can create duplicates or omissions when continuing pagination. Ordinary search_after pagination should therefore end with a unique tie-breaker, such as a sortable productId keyword field.

When using PIT, Elasticsearch adds a tie-breaker based on shard document order. Pass the entire sort array from the last returned hit rather than reconstructing it. The query, sort, and PIT view must stay consistent across pages.

This is also relevant when ranking by business criteria. Decide whether relevance, price, popularity, or recency has priority, then express that ordering explicitly. A cursor only makes sense if the ordering contract stays stable.

---

## 5. Shards, performance, and production operations

### Q25. How do you choose a sensible shard strategy for a production index?

Estimate primary data size after indexing, document size, growth, write throughput, search concurrency, and recovery requirements. Add replica storage and operational headroom to estimate physical disk needs. Test plausible shard layouts with realistic indexing and queries instead of relying on document count alone.

Too many small shards add coordination and metadata overhead. Too few very large shards can limit parallelism and make recovery expensive. Replica allocation also needs enough space on other eligible nodes and ideally across failure domains. The cluster should retain enough capacity to recover a failed node without pushing survivors into disk, heap, or latency trouble.

Review shard distribution, per-shard load, recovery times, and query patterns. A hot tenant or time partition may create a hotspot even when cluster-wide averages look fine. Add shards or replicas only after telemetry identifies the bottleneck; they are not interchangeable solutions.

### Q26. What makes an Elasticsearch query expensive?

Common causes include broad wildcard or regexp patterns, fuzzy expansions, scripts, high-cardinality aggregations, deep from/size pagination, wide time ranges, nested queries, exact total-hit counts when unnecessary, and searches that fan out to too many shards. Mapping matters too: a keyword field with doc values has different costs and capabilities from an analyzed text field that would require fielddata for aggregation.

Bound user-controlled input. Limit page sizes, time windows, pattern complexity, searched fields, aggregation buckets, and concurrent requests. Avoid letting public clients submit arbitrary Query DSL, scripts, or unrestricted sort field names. These safeguards protect availability as well as correctness.

Use slow logs to identify recurring outliers, Profile to examine a representative query, and node/index metrics to distinguish query complexity from CPU saturation, disk latency, heap pressure, or rejected work. Profile adds overhead and is a diagnostic tool, not a setting to enable blindly for every production request.

### Q27. How do heap, filesystem cache, CPU, disk, and network affect search?

Heap holds JVM objects and metadata. Excessive heap use can cause garbage-collection pauses and reduce memory left for the operating system. Lucene benefits from the filesystem cache, so giving all memory to the JVM can hurt search performance. The right balance depends on mappings, workload, and cluster size.

CPU is used for analysis, scoring, scripts, aggregation reduction, and concurrent requests. Disk latency and throughput affect segment reads, merges, translog writes, and shard recovery. Network latency and bandwidth matter during shard fan-out, replication, and recovery. These resources interact: more concurrency can improve throughput until it saturates a resource, after which latency and queueing often worsen.

Diagnose with heap and post-GC heap, GC pauses, CPU saturation, disk latency, filesystem-cache behavior, thread-pool rejections, merge pressure, network, and per-shard load. Do not increase heap or shard count just because latency rose; first establish which resource is under pressure.

### Q28. How do you investigate slow searches and production timeouts?

Establish the scope first: are all searches slow, only one query shape, one tenant, one index, or only deep pages? Compare p50/p95/p99 latency, request rate, timeouts, shard failures, rejections, and indexing load. Also check whether the application timed out before Elasticsearch did, and whether the response contains partial results.

Use slow logs for recurring requests and Profile for a representative query. Explain answers why one document scored as it did; Profile shows query execution details. They are different diagnostic tools. Inspect range size, expensive clauses, aggregation cardinality, shard allocation, heap/GC, disk, and hot nodes before changing topology.

A good mitigation targets the evidence: narrow an unbounded range, reduce bucket counts, correct mapping, throttle bulk imports, or add capacity to the saturated resource. Increasing client timeouts can hide overload while more work piles up. Verify the fix against the same workload and add a regression test or alert.

### Q29. What do green, yellow, and red cluster health mean, and how do you respond to node failure?

Green means all primary and replica shards are allocated. Yellow means all primaries are allocated but one or more replicas are not, so the index can remain searchable with reduced redundancy. Red means at least one primary shard is unassigned, so some data may be unavailable. The color describes the symptom, not the cause.

After a node failure, Elasticsearch may promote an available replica and recover or relocate missing replicas when capacity and valid shard copies permit. Recovery competes with live queries and writes for disk, network, CPU, and cache. A cluster sized only for normal operation may struggle to recover because surviving nodes cannot handle both production traffic and recovery.

Inspect health, allocation explanations, failed-node logs, disk watermarks, and recovery progress. Preserve valid copies and snapshots. Avoid force-allocating stale primaries or invoking data-loss commands before understanding the consequences and getting approval. Resilience means having enough remaining capacity to restore redundancy, not merely having configured one replica.

### Q30. What are disk watermarks and circuit breakers?

Disk watermarks guide shard allocation when nodes approach their disk capacity. At flood stage, Elasticsearch can apply a read-only-allow-delete block to indices with shards on the affected node. This is protection against running out of disk, not simply an application permission problem. Exact thresholds depend on configuration and deployment.

Inspect free space, shard placement, retention and rollover, old indices, and the space needed for merges and recovery. Free or add disk safely, correct the retention cause, and verify that the block has cleared according to the cluster's behavior. Deleting business data without an approved retention decision is not an acceptable first response.

Circuit breakers estimate memory use and reject operations before they exhaust the JVM. Large aggregations, oversized requests, fielddata, scripts, and high concurrency can trigger them. Inspect breaker stats, memory after garbage collection, and request patterns before raising a limit; raising it without adding capacity can turn a safe rejection into an out-of-memory failure.

---

## 6. Index lifecycle and safe migrations

### Q31. What are aliases, templates, ILM, rollover, and data streams?

An alias is a logical name that can point to one or more indices. Applications can search an alias rather than embedding a physical index name in code. Aliases can also support an atomic switch from an old index to a new one. Write aliases must be configured deliberately so writes reach the intended destination.

Index templates define settings and mappings for indices created in the future; component templates allow reusable parts to be composed. A template change does not rewrite an existing index's mapping. This distinction explains why a fix can work for a newly created test index but not for old production data.

ILM can automate rollover and configured lifecycle actions such as moving data through supported tiers or deleting expired data. Data streams provide a named abstraction over backing indices for append-oriented time-series data. Capabilities vary by deployment. Design retention, rollover, and shard sizing together so the cluster does not accumulate thousands of tiny shards.

### Q32. How do you change an incompatible field type or analyzer without downtime?

You generally cannot change an existing field's type or re-index its old values with a new analyzer in place. Create a versioned destination index with the correct settings and mapping, populate it, validate it, and then switch the application-facing alias. Updating a template alone only affects future indices.

The hard part is keeping the destination current while writes continue. Use a durable change stream or source-versioned events, establish a clear backfill and catch-up strategy, replay changes after the chosen backfill point, and include deletes. Validate counts, field distributions, representative queries, and indexing failures before cutover. The alias switch can be atomic, but it does not solve missing data or out-of-order updates.

Keep the old index available for a defined rollback window, and understand how writes made after cutover would be reconciled if rollback is needed. Practice the migration and rollback rather than assuming changing the alias is the entire plan.

---

## 7. Consistency, concurrency, and recovery

### Q33. Explain write acknowledgement, durability, and search visibility. Are they the same?

No. A write acknowledgement describes how an operation was accepted under the request's replication and acknowledgement conditions. Durability concerns recovery after a failure. Search visibility is when refresh makes the new indexed data available to search. These are separate dimensions.

An indexing request can return successfully while an immediate search still misses the document because refresh has not occurred. If an API contract requires the write to be searchable before returning, evaluate refresh=wait_for for the relevant operation or batch. Avoid forcing immediate refresh on every write because it can damage indexing and search efficiency.

State the contract clearly: “saved”, “searchable”, and “committed to the authoritative source” are different promises. A local test may pass by chance due to timing; test the required visibility behavior under realistic traffic and refresh settings.

### Q34. What is optimistic concurrency control in Elasticsearch?

Optimistic concurrency control prevents an update based on an old document version from silently overwriting newer state. Elasticsearch exposes sequence number and primary term metadata for conditional writes. If two callers read the same version and both update, a conditional write lets one proceed while detecting the stale write from the other caller.

The application decides whether to retry, reload, merge, or reject a conflict. Do not retry conflicts forever without asking whether the operation is safe to repeat. Setting a field to a particular value is often easier to retry than applying an increment without an event identity or concurrency check.

For a search projection, source versions can prevent an older replayed event from replacing newer state, but only if versioning is enforced by the write path. A JSON field named sourceVersion does not automatically protect the document. Choose a documented versioning strategy and test duplicate and out-of-order events.

### Q35. How should snapshots and recovery objectives be designed?

A snapshot is a repository-managed backup using Elasticsearch's supported snapshot API. It is not the same as copying a live data directory. Configure a supported repository, monitor snapshot success and age, and retain snapshots according to recovery, compliance, and cost requirements.

RPO (recovery point objective) is how much data loss in time the business can tolerate; RTO (recovery time objective) is how quickly service must return. A snapshot schedule alone proves neither. Restore time depends on data size, repository throughput, cluster capacity, and index reconstruction work.

Run restore drills and validate the recovered indices, required settings, aliases, and search behavior. If Elasticsearch is a rebuildable projection, the authoritative source and event stream may help recover changes; if it is the only copy of data, recovery requirements are different. Replicas are not backups because accidental deletes and bad writes can be replicated to every copy.

---

## 8. Java integration

### Q36. Which Java client should a new application use, and what should the integration layer handle?

Use the official Elasticsearch Java API Client for new Java applications instead of starting new work with the deprecated High Level REST Client. The official client offers typed request and response objects, synchronous and asynchronous APIs, and object-mapper integration. Match the client major version to the server using Elastic's compatibility guidance; compatibility does not mean an older client automatically supports every feature introduced by a newer server.

Create the client as an application-scoped bean rather than constructing one per request. Configure TLS and authentication, connection resources, timeouts, and orderly shutdown for the target environment. Put Elasticsearch query construction behind a repository or search-service boundary so business logic does not spread raw DSL throughout the application.

Validate user input before building queries. Derive tenant identity from authenticated server context, allowlist sort fields, cap page sizes and date ranges, and never accept arbitrary raw DSL or scripts from public clients. Translate failures into clear application behavior without exposing credentials, internal hostnames, or sensitive request details.

### Q37. How should Java code search and index documents safely?

Prefer the typed Java API Client builders for application-owned queries because the request is easier to review and many structural mistakes become more visible in code. This example shows the important design choice: text relevance is a scoring clause, while inventory eligibility is a filter. Exact generated builder signatures should be verified against the client dependency version used by the application.

~~~java
SearchResponse<Product> response = client.search(s -> s
        .index("products-v1")
        .query(q -> q.bool(b -> b
            .must(m -> m.match(mm -> mm
                .field("name")
                .query("wireless headphones")))
            .filter(f -> f.term(t -> t
                .field("inStock")
                .value(FieldValue.of(true))))
        )),
    Product.class
);
~~~

For bulk indexing, inspect item-level responses and record failed document IDs, causes, and source versions. Retry only transient failures with bounded backoff; send permanent mapping or validation errors to a dead-letter process. A timeout can leave the outcome unknown, so deterministic IDs and idempotent writes matter.

Unit tests can verify query construction, but integration tests catch analyzer, mapping, serialization, and version-specific behavior. Test missing fields, special characters, sorting, pagination, partial bulk failures, and the response contract, not only the happy path.

---

## 9. SQL, Kafka, and search projections

### Q38. Why is dual-writing SQL and Elasticsearch unsafe? How do outbox and CDC patterns help?

Suppose a service commits an order to SQL and then writes its search document to Elasticsearch. If SQL commits but Elasticsearch fails, the two stores disagree. If Elasticsearch is written first and SQL later fails, search exposes a change that never committed in the source database. This dual-write gap is a common consistency bug.

A transactional outbox writes both the business change and an event record in the same SQL transaction. A relay or change-data-capture process publishes the event to a durable stream such as Kafka, and a consumer updates Elasticsearch. This closes the gap between committing source state and creating the event, but delivery and indexing remain asynchronous and still require retry, replay, and reconciliation.

Define freshness explicitly. If search may lag by seconds, measure event age and consumer lag. If a critical decision needs current stock or payment state, re-check the authoritative system. Elasticsearch is a search projection, not the authority for a transaction that may not yet have been indexed.

### Q39. How do you handle duplicate, out-of-order, and delete events?

Consumers may retry after timeouts or restart from an offset, so duplicate delivery is normal. Use deterministic document IDs and make projection updates idempotent where possible. If events can be out of order, carry a source version or sequence and enforce a policy that prevents an older event from replacing newer state.

Deletes need the same care as creates and updates. If a delete event is lost, a stale search document can remain visible. Include entity IDs and versions in delete events, make handling repeatable, monitor failures, and support replay or reconciliation against the authoritative source. A permanent mapping error should go to a dead-letter workflow rather than retrying forever.

“Exactly once” is not a magic end-to-end guarantee obtained by enabling a Kafka option. Think through producer guarantees, consumer offset commits, idempotent Elasticsearch writes, replay, deletes, and reconciliation. The practical objective is a projection that converges to source state and makes drift visible early.

### Q40. How do you prevent cross-tenant data leakage in search?

Apply tenant restrictions on the server using authenticated identity, not an untrusted tenantId provided by a caller. Centralize query construction so every path includes tenant scope: normal search, direct document lookup, aggregations, autocomplete, exports, and pagination cursors. A single forgotten path can leak data even when the main search endpoint is correct.

Application-level filtering should be supported by authorization and defensive tests. Depending on threat model and deployment, document-level security or separate indices may provide additional boundaries. Routing by tenant can improve locality but is not authorization and must never be used as a security control by itself.

Test negative cases: a user from tenant A cannot retrieve tenant B's document by ID, infer counts through aggregations, reuse another tenant's cursor, or access suggestions and exports. Tenant isolation is part of correctness and security, not merely a performance feature.

---

## 10. Production scenarios

Read each prompt as an interview case: understand the user-visible symptom, explain the likely causes, then describe how you would investigate, fix the issue, and prevent it from returning. The model answers are intentionally written as explanations you could give aloud, rather than as checklists of keywords.

### Scenario 1. Your product exists in Elasticsearch, but a search for its full name returns zero hits. How would you find the cause?

A product named “Wireless Headphones” exists in the index, but the API uses a term query on the name field and returns no result. I would first inspect the mapping and the stored document, then run the Analyze API using the field's analyzer. If name is mapped as text, the value may be indexed as separate lowercase terms such as wireless and headphones. A term query searches for an exact indexed term and does not analyze the supplied phrase the way a match query does, so it may look for one term that was never indexed. I would use match for full-text search or the name.keyword multi-field when the requirement is exact whole-value matching. I would not change the whole name field to keyword just to make this one query work, because that would undermine normal text search and might require reindexing. If the correct query still returns nothing, I would check the target index or alias, other filters, routing, and whether the document is searchable yet.

### Scenario 2. Customers search for a specific product, but broad or less relevant products appear above it. How would you improve relevance without hiding useful results?

Imagine a customer searches for “Sony WH-1000XM5 headphones”. The exact product is returned, but generic headphones with a long description containing some of the same words rank above it. I would capture real search examples and the products that should rank near the top, then inspect the request, field mappings, analyzers, bool-query defaults, and current boosts. A name match should usually carry more weight than a description match, so a multi_match query can boost name and a match_phrase clause can give additional weight to the complete model name. Availability, tenant, and other eligibility rules should remain filters rather than relevance signals. I would use Explain to understand one surprising score, but would not tune ranking from that document alone; I would compare the top results for a small set of representative queries before and after each change. Finally, I would preserve those examples as a relevance regression suite so future analyzer, synonym, or mapping changes do not quietly reduce search quality.

### Scenario 3. Your API returns success after indexing a new product, but the next search request cannot find it. What would you check, and what guarantee should the API provide?

A successful indexing response does not necessarily mean the document is searchable immediately. Elasticsearch refreshes recent Lucene changes for search periodically, so a search performed right after a write can miss a document even though the indexing operation succeeded. I would first make sure the write and read target the same index or alias, and verify the document ID, routing, tenant scope, and other filters. If the requirement is specifically that the product be visible to search before the write request returns, I would consider refresh=wait_for for the relevant indexing operation or bulk request. It waits for a refresh rather than usually forcing an immediate refresh, while refresh=true on every write can reduce indexing throughput. If the API only needs to return the saved product to the caller, returning the authoritative write result may be simpler than issuing another search. I would document whether the API promises “saved” or “searchable now” and test the actual requirement under load.

### Scenario 4. A nightly catalogue import used to finish quickly, but now it causes product searches to time out. How would you diagnose and mitigate the problem?

I would correlate the start of the latency increase with the import's bulk size and concurrency, indexing throughput, segment merges, CPU, disk latency, heap and garbage collection, thread-pool rejections, and search p95/p99. I would also check whether the slow requests share a query shape or concentrate on particular shards; the import may be contributing without being the only cause. If live search is suffering from ingestion pressure, I would first throttle the importer or reduce its concurrency to restore service, then tune batch size gradually while measuring both import throughput and search latency. Very large batches can increase memory use and request duration, while too many concurrent writers can saturate disk or merging capacity. Adding replicas might distribute some read traffic, but it also consumes disk and adds replicated write work, so it is not a universal fix. For future imports, I would run mixed search-and-ingest tests, define an ingestion rate limit, and alert on rejected requests, bulk item failures, and search tail latency.

### Scenario 5. One data node fails, the cluster becomes yellow, and the application is still serving searches. What does this mean, and how would you restore resilience?

Yellow means all primary shards are allocated but one or more replicas are not. The data may still be searchable, but some shards have lost their redundant copy, so another failure could cause an outage or data unavailability. I would inspect the allocation explanation, remaining eligible nodes, disk watermarks, placement or awareness rules, and recovery progress to learn why Elasticsearch cannot assign the missing replicas. The normal fix is to restore the failed capacity or resolve the allocation constraint. Lowering the replica count merely to turn the status green may hide the redundancy problem and should only be a deliberate, approved trade-off. After recovery, I would confirm that the replicas are allocated and that the remaining cluster has enough headroom for a node or zone failure. The incident should also prompt a review of capacity planning and whether recovery traffic can coexist with normal search traffic.

### Scenario 6. Several searches suddenly return incomplete data and cluster health is red. How would you recover without making data loss worse?

Red means at least one primary shard is unassigned, so documents from that shard may be unavailable to search. I would identify the affected index and shard, inspect its allocation explanation and node logs, and establish whether a valid copy exists but cannot be allocated or whether all usable copies are lost. Disk watermarks, a failed node, allocation constraints, and storage corruption have different remedies. I would preserve existing shard copies and snapshots, fix the underlying infrastructure or allocation issue where possible, and recover from a valid copy or approved snapshot when necessary. I would not force-allocate a stale or empty primary merely to make the status green, because doing so can discard acknowledged data. After recovery, I would verify document availability and search behavior, test snapshot restoration, and update the runbook. The correct recovery decision depends on the valid copies available and the business recovery-point objective; the health color alone is not enough to choose a recovery command.

### Scenario 7. Your Bulk API request returns HTTP 200, but the import has fewer indexed products than expected. How would you handle the failed records and avoid losing them?

The Bulk API is not an all-or-nothing transaction. The top-level request can return successfully while individual operations fail, so I would inspect the response's errors flag and each failed item's status, ID, and error details. Mapping or parsing failures require a data or schema correction; temporary overload or availability failures may be retryable. I would retry only transient failures, using bounded exponential backoff and jitter, rather than repeating the entire batch blindly. Permanent failures should be sent to a durable dead-letter workflow with the source event, document ID, source version, and cause so they can be corrected and replayed. If the import is driven by Kafka, I would also check that the consumer does not commit offsets in a way that loses failed records before they are safely recorded for retry. I would track item-level failure rates and reconcile source IDs against indexed IDs so a successful HTTP status is never treated as proof that every product was loaded.

### Scenario 8. A customer filters for a blue, size-L product variant, but Elasticsearch returns a product that has only a blue size-S variant and a black size-L variant. Why can this happen, and how would you fix it?

This can happen when variants are mapped as an ordinary array of object fields. Elasticsearch flattens the values for indexing, so a color condition may match one child object and the size condition another, allowing the parent product to match even though no variant has both attributes. I would inspect the mapping and reproduce the issue with a deliberately invalid color/size combination. If the application needs the two conditions to belong to the same variant, I would map variants as nested and put both term conditions inside the same nested query. Nested fields preserve that relationship, but add storage and query cost. Changing an existing field from object to nested generally requires creating a new index and reindexing, so I would validate the mapping and query on a test index before migrating. I could use inner_hits during debugging to see which variant matched and add regression tests for both valid and invalid combinations.

### Scenario 9. Your API offers page-number pagination. At page 500, requests become slow and users sometimes see duplicate products across pages. How would you redesign it?

Deep from/size pagination is expensive because each shard has to collect candidates for the skipped hits as well as the requested page. Duplicate or missing results can also occur when the sort is not deterministic or the index changes between page requests. For shallow pages, from/size is simple and appropriate. For deep sequential browsing, I would use search_after with the same query and sort on every request, and include a unique sortable tie-breaker so equal prices or timestamps do not leave the order ambiguous. If the user must page through a consistent view while refreshes occur, I would open a point in time (PIT) and use the latest PIT ID plus the complete sort array returned by the last hit. I would expose this through a short-lived cursor scoped to the authenticated user, tenant, filters, and sort contract rather than letting clients invent arbitrary sort values. PIT contexts consume resources, so I would bound their lifetime and close them when paging finishes.

### Scenario 10. A mapping change passed in staging but fails after production deployment. You need to change a field type without downtime. What steps would you take?

I would compare the actual index mappings and settings, index and component templates, aliases and write index, analyzer output, sample documents, and server/client versions. A template update affects future index creation; it does not rewrite an existing production index. Dynamic mapping can also infer different types depending on which values arrive first, explaining why two environments differ. If the field type or index-time analyzer change is incompatible, I would create a versioned destination index with the intended mapping, backfill the current data, and replay concurrent updates and deletes from a durable change stream. I would validate counts, mapping errors, and representative search results, wait until the destination catches up, and then switch the application-facing aliases atomically. I would keep the old index for a planned rollback window, but also define how post-cutover writes would be reconciled if a rollback is needed. The alias switch makes routing atomic; it does not guarantee the destination is complete or make rollback automatically safe.

### Scenario 11. After a Kafka consumer restarts, an older product event overwrites a newer product document in Elasticsearch. How would you prevent stale data from winning?

A deterministic document ID prevents duplicate logical documents, but it does not stop an older event from overwriting a newer one. For example, if version 42 is indexed and a replayed version 41 is processed afterward using an ordinary index operation, version 41 can become the current document. I would inspect event ordering, retries, consumer offset commits, and how the source system identifies versions. Then I would use a monotonically increasing source version with a supported versioning or conditional-write strategy so the write path rejects stale events. The handling of equal versions and version conflicts must be deliberate; blindly retrying every conflict can create a loop. I would test duplicate and out-of-order updates, including deletes, and make sure failed events remain recoverable. Finally, I would monitor consumer lag and reconcile the search projection against the authoritative source, because Kafka delivery settings and deterministic IDs alone do not guarantee the projection always reflects the newest state.

### Scenario 12. A customer from tenant A can retrieve a known product belonging to tenant B. How would you contain the issue and prevent it from recurring?

I would treat this as a security incident and first contain the exposure, then determine its scope. I would verify how authenticated identity becomes tenant scope and audit every retrieval path: normal search, direct document lookup, aggregations, autocomplete, exports, and cursor continuation. The server must derive tenant scope from trusted authentication and authorization context; a tenant ID supplied by a caller is not proof of authorization. I would centralize tenant filtering so it cannot be accidentally omitted by a new endpoint and consider defense in depth, such as document-level security or separate indices, according to the threat model. Routing by tenant may improve locality, but it is not an access-control mechanism. I would add negative integration tests proving tenant A cannot fetch tenant B's known ID, infer protected counts, reuse another tenant's cursor, or access suggestions and exports. I would also assess the data exposure, invalidate affected cursors where practical, and follow the incident-response process without logging secrets.

---

## 11. Query DSL cookbook: 18 useful patterns

The examples assume a products index with productId, brand, category, status, and tenantId as keyword fields; name and description as text; price as numeric; inStock as boolean; createdAt as a date; rating as numeric; and variants mapped as nested. Adapt field names and types to the actual mapping. A query can be valid JSON and still be wrong for the data model, so each example explains what its mapping assumptions mean.

### Pattern 1. Full-text search on a product name

~~~json
GET products-v1/_search
{
  "query": {
    "match": {
      "name": "wireless headphones"
    }
  }
}
~~~

Use match when the user is searching words in a text field. Elasticsearch analyzes the query input and matches the resulting tokens against the analyzed field. With the default operator, matching either term can be sufficient, so this can return documents that contain only “wireless” or only “headphones”. That is often useful for recall, but it may be too broad for some products. If the business needs more terms to match, set operator or minimum_should_match explicitly and evaluate the impact on recall. If this returns nothing, inspect the field mapping and Analyze output before changing the query.

### Pattern 2. Find an exact product identifier

~~~json
GET products-v1/_search
{
  "query": {
    "term": {
      "productId": "P1001"
    }
  }
}
~~~

This assumes productId is mapped as keyword or another exact-match type. term looks for an exact indexed term and does not analyze the query value as a full-text match query would. It is a good fit for SKUs, codes, status values, and identifiers that must not be tokenized. If productId is the Elasticsearch document ID rather than a source field, GET by ID is usually more direct. Keep the source identifier type and normalization consistent; an accidental numeric mapping can cause leading zeroes to be lost.

### Pattern 3. Combine relevance with stock, brand, and price filters

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "wireless headphones",
            "fields": ["name^4", "description"]
          }
        }
      ],
      "filter": [
        { "term": { "inStock": true } },
        { "term": { "brand": "Acme" } },
        { "range": { "price": { "gte": 50, "lte": 250 } } }
      ]
    }
  }
}
~~~

This is one of the most useful production patterns. The full-text clause decides how matching products are ranked, while stock, brand, and price determine which products are eligible. Putting these structured conditions in filter context means they do not add relevance score. Confirm that brand is keyword and price is numeric; if brand is a text field, its indexed terms may not support the exact semantics you expect. The inclusive gte/lte range includes both endpoints, which is a business decision worth making explicit.

### Pattern 4. Search multiple text fields with boosts

~~~json
GET products-v1/_search
{
  "query": {
    "multi_match": {
      "query": "noise cancelling headphones",
      "fields": ["name^5", "description"]
    }
  }
}
~~~

This says a name match is usually stronger evidence of relevance than a description match. The boost changes score contribution; it is not a fixed multiplier of business quality and does not guarantee that every name hit outranks every description hit. Start with a simple field list, gather representative queries, then compare ranking quality after changing boosts. Avoid including fields whose values are not analyzed as text. If exact phrase matches deserve additional weight, add a phrase clause deliberately and test that it does not suppress useful results.

### Pattern 5. Search an exact phrase or ordered phrase

~~~json
GET products-v1/_search
{
  "query": {
    "match_phrase": {
      "name": "noise cancelling headphones"
    }
  }
}
~~~

match_phrase analyzes the query text and checks token positions so terms must occur in the expected order by default. It is stricter than a simple match query, but it is not the same as comparing the original source string character-for-character. Analyzer choices, stop words, and token positions affect what counts as a phrase. Use it for names, titles, and important expressions, often as an additional relevance signal. If you require exact whole-field equality, use a keyword field with term instead.

### Pattern 6. Require a minimum number of matching alternatives

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "name": "wireless" } },
        { "match": { "name": "bluetooth" } },
        { "match": { "name": "headphones" } }
      ],
      "minimum_should_match": 2
    }
  }
}
~~~

Here a document must match at least two of the three clauses. This illustrates minimum_should_match on the bool query itself, where each clause is an alternative. Do not confuse this with minimum_should_match inside a match query, where it applies to analyzed query terms. Another common trap is the bool default: should clauses are required by default only when there is no must or filter clause. Set the value explicitly whenever these alternatives are part of the business rule rather than optional boosts.

### Pattern 7. Filter multiple accepted values and exclude discontinued products

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "inStock": true } },
        { "terms": { "brand": ["Acme", "Globex"] } }
      ],
      "must_not": [
        { "term": { "status": "discontinued" } }
      ]
    }
  }
}
~~~

terms means the brand can equal any value in the supplied list; it is an exact-value alternative, not a full-text OR query. All filter clauses must match, while must_not removes discontinued documents. This is useful for eligibility rules and filters selected in a UI. Make sure brand and status are mapped for exact matching, and validate empty or very large input lists in application code. Do not let a client pass arbitrary status restrictions that bypass the server's own business rules.

### Pattern 8. Filter a date range with clear boundaries

~~~json
GET products-v1/_search
{
  "query": {
    "range": {
      "createdAt": {
        "gte": "2026-01-01T00:00:00Z",
        "lt": "2027-01-01T00:00:00Z"
      }
    }
  }
}
~~~

A range query on a date field is useful for reports, log exploration, and time-bounded search. This example uses a half-open interval: start is included and the next period's start is excluded. That avoids double-counting a record that sits exactly on a boundary when adjacent periods are queried. Confirm that the field is mapped as date and that the application has an explicit timezone policy. For high-volume log search, constrain the time range to a reasonable window instead of accepting an unbounded query.

### Pattern 9. Require color and size to match the same nested variant

~~~json
GET products-v1/_search
{
  "query": {
    "nested": {
      "path": "variants",
      "query": {
        "bool": {
          "filter": [
            { "term": { "variants.color": "blue" } },
            { "term": { "variants.size": "L" } }
          ]
        }
      }
    }
  }
}
~~~

This only works as intended when variants is mapped as nested. The nested query evaluates both conditions against the same child variant. With an ordinary array of objects, flattened indexing may allow color to match one variant and size to match another, producing a false positive at the parent product level. Nested fields have additional storage and query cost, so use them when the relationship must be preserved. During debugging, inner_hits can show which child object matched the nested query.

### Pattern 10. Sort by price with a stable tie-breaker

~~~json
GET products-v1/_search
{
  "size": 20,
  "sort": [
    { "price": "asc" },
    { "productId": "asc" }
  ],
  "query": {
    "term": {
      "inStock": true
    }
  }
}
~~~

The sort returns the cheapest eligible products first and uses productId to order products with the same price. This assumes productId is sortable and unique within the searched result set. Stable sorting matters for pagination; without a unique tie-breaker, equal-price hits can move between requests and produce duplicates or gaps. When paginating deeply, use search_after and copy the last hit's actual sort values. When a PIT is used, preserve the full sort array returned by Elasticsearch, including its implicit tiebreaker.

### Pattern 11. Return category facets without fetching hits

~~~json
GET products-v1/_search
{
  "size": 0,
  "query": {
    "match": {
      "name": "headphones"
    }
  },
  "aggs": {
    "categories": {
      "terms": {
        "field": "category",
        "size": 10
      }
    }
  }
}
~~~

This returns the top ten category buckets for documents matching the text query, without returning the regular hit list. It is a common search-page facet. category should normally be a keyword field or another suitable aggregatable field; aggregating a text field can fail or require an expensive and usually discouraged fielddata configuration. A terms aggregation returns selected buckets, not an unlimited exact catalogue of all unique values, and distributed document counts can be approximate in some cases.

### Pattern 12. Build price-band facets

~~~json
GET products-v1/_search
{
  "size": 0,
  "aggs": {
    "price_bands": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 50 },
          { "from": 50, "to": 150 },
          { "from": 150 }
        ]
      }
    }
  }
}
~~~

Range aggregation groups documents into explicit numeric intervals. The lower and upper boundary semantics matter: ranges are commonly inclusive at the lower bound and exclusive at the upper bound, so the example creates bands below 50, from 50 up to but not including 150, and 150 or more. Use business-friendly labels in the UI and test values exactly on the boundaries. Apply the same search query or filters if the facets should describe only currently eligible products.

### Pattern 13. Find documents that lack an indexed rating

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "must_not": [
        { "exists": { "field": "rating" } }
      ]
    }
  }
}
~~~

exists asks whether Elasticsearch has an indexed value for the field. It does not simply check whether the raw JSON object visually contains the property. For example, null values and empty arrays are commonly treated as having no indexed value, while a configured null_value or indexing settings can affect the result. This query is useful for finding products that need enrichment, but validate the field mapping and ingestion rules before using the result as proof that the source database has no rating.

### Pattern 14. Implement type-ahead with search_as_you_type

~~~json
GET products-v1/_search
{
  "query": {
    "multi_match": {
      "query": "wireless head",
      "type": "bool_prefix",
      "fields": [
        "name",
        "name._2gram",
        "name._3gram"
      ]
    }
  }
}
~~~

This assumes name is mapped as search_as_you_type, which creates the shown shingle sub-fields. The bool_prefix query lets terms match flexibly and scores documents higher when terms appear together in the shingle fields. It is useful when the user types part of a multi-word product name. It is not a drop-in query for a plain text field, and the field's analyzer and indexed subfields affect the behavior. Compare it with phrase-prefix or a completion suggester based on the UX, typo requirements, and index-size budget.

### Pattern 15. Allow a limited number of typing mistakes

~~~json
GET products-v1/_search
{
  "query": {
    "match": {
      "name": {
        "query": "headphnes",
        "fuzziness": "AUTO",
        "prefix_length": 1
      }
    }
  }
}
~~~

Fuzzy matching can recover a misspelling such as “headphnes”, but it expands the terms considered and may increase latency or bring in noisy results. AUTO chooses an edit-distance allowance based on term length; prefix_length keeps the first character exact in this example. Fuzziness is not equally useful for all terms, particularly very short words and identifiers. Apply it selectively and evaluate typo queries against a judged set. A keyword SKU lookup should generally remain exact rather than becoming a fuzzy text query.

### Pattern 16. Let users enter a simple query expression safely

~~~json
GET products-v1/_search
{
  "query": {
    "simple_query_string": {
      "query": "wireless + headphones - refurbished",
      "fields": ["name^4", "description"],
      "default_operator": "and"
    }
  }
}
~~~

Use this only when the product intentionally supports a small search syntax, such as required terms and exclusions. simple_query_string is more forgiving about invalid syntax than query_string, but operators still change the meaning of the query; it is not just an ordinary match query. Keep the searchable field list controlled by the server, document which operators are supported, and test punctuation and special characters. For a normal search box that should interpret everything as plain text, build match or multi_match in application code instead.

### Pattern 17. Scope search to the authenticated tenant

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "headphones" } }
      ],
      "filter": [
        { "term": { "tenantId": "tenant-123" } }
      ]
    }
  }
}
~~~

The tenant filter is a mandatory security boundary, not a preference supplied by the user. In a real API, derive tenantId from validated authentication and authorization context on the server; do not trust tenant-123 from a query parameter. Ensure the same scope is applied to direct reads, autocomplete, aggregations, exports, and cursor continuation. Test cross-tenant negative cases. Depending on the threat model, application filters may need defense-in-depth from document-level security or separate indices. Routing can improve locality but is not authorization.

### Pattern 18. Deep pagination with a point in time and search_after

First open a point in time and keep the returned ID:

~~~json
POST /products-v1/_pit?keep_alive=1m
~~~

Use the returned PIT ID for the first search. Keep the query and sort unchanged for later pages, and use the latest PIT ID returned by Elasticsearch. A PIT automatically adds a _shard_doc tie-breaker to the sort, so copy the complete sort array from the last hit rather than reconstructing it.

~~~json
POST /_search
{
  "size": 50,
  "pit": {
    "id": "REPLACE_WITH_RETURNED_PIT_ID",
    "keep_alive": "1m"
  },
  "query": {
    "term": {
      "inStock": true
    }
  },
  "sort": [
    { "createdAt": { "order": "desc", "format": "epoch_millis" } }
  ]
}
~~~

For the next page, copy the exact values in the last hit's sort array into search_after. With the sort above and a PIT, that array includes the date value and the implicit tie-breaker; the following values are illustrative only.

~~~json
POST /_search
{
  "size": 50,
  "pit": {
    "id": "REPLACE_WITH_LATEST_PIT_ID",
    "keep_alive": "1m"
  },
  "query": {
    "term": {
      "inStock": true
    }
  },
  "sort": [
    { "createdAt": { "order": "desc", "format": "epoch_millis" } }
  ],
  "search_after": [1772360130000, 4294967298]
}
~~~

Replace the example sort values with the complete array returned for the actual last hit, and use the latest PIT ID from the previous response. Omit search_after on the first page. Close the PIT when traversal is complete and keep its lifetime bounded; open PIT contexts consume resources.

---

## 12. Revision checklist and official references

Use this section for last-minute revision. These are not unanswered prompts: each question has a short model answer. The earlier sections provide the fuller explanation and examples.

### Quick revision questions and answers

#### Q1. Why keep SQL as the source of truth while Elasticsearch serves search?

SQL is usually the authoritative store for transactions, constraints, and business updates that must remain correct. Elasticsearch is often a denormalized search projection built from that source so queries can retrieve text, filters, facets, and relevance-ranked results efficiently. Because projection updates are asynchronous, search can be stale or temporarily unavailable. Critical decisions such as charging a payment or confirming inventory should be checked against the authoritative system.

#### Q2. What do shards and replicas do, and why are more shards not always better?

Primary shards divide an index's documents so storage and work can be distributed. Replicas keep copies of primary shards, improve fault tolerance, and can serve search requests, but they use additional disk and must also handle replicated writes. Too few large shards can limit parallelism and increase recovery time; too many small shards create metadata, file-handle, heap, and coordination overhead. Choose shard count from realistic data volume and workload tests, not a fixed rule.

#### Q3. What is the practical difference between match and term?

The match query analyzes the query text and is normally used on text fields for full-text search. The term query looks for an exact indexed term without analyzing the supplied value, so it is commonly used with keyword, numeric, boolean, and other structured fields. If a name is analyzed into separate tokens, a term query for the whole original phrase may return no hits. Check the mapping and Analyze API output before changing the query.

#### Q4. How do must, filter, should, must_not, and minimum_should_match work?

must requires a match and contributes to scoring; filter requires a match without affecting the score; must_not excludes documents; and should adds optional scoring matches or alternatives. The important default is that should clauses are required by default when the bool query has no must or filter clauses, but become optional by default when a must or filter clause is present. Set minimum_should_match explicitly when the business rule requires some of those alternatives.

#### Q5. Why can a successful write be missing from search, and how are refresh and flush different?

A successful indexing response does not necessarily mean the document is already searchable. A refresh opens recent Lucene changes to search, which is why Elasticsearch is near-real-time. A flush is different: it commits Lucene changes and rolls the translog to a new generation so older translog data can eventually be removed. If an operation must be searchable before the request returns, consider refresh=wait_for; avoid forcing refresh on every write because it can reduce throughput.

#### Q6. How should I choose aggregations and deep pagination?

Use a terms aggregation for common top-N facets such as the most frequent brands or categories, but remember that distributed bucket counts can be approximate depending on shard candidate selection and ordering. Use composite aggregation when you need to paginate through many buckets. from/size is suitable for shallow page-number navigation; for deep sequential pagination, use search_after with a stable sort. Add a unique tie-breaker, and use a point in time (PIT) when the result set must remain stable while the user pages through it.

#### Q7. How do I diagnose a slow query instead of guessing at a fix?

First establish whether the slowdown affects every query, one query shape, one tenant, or one index. Compare p95/p99 latency, request rate, timeouts, shard failures, rejections, indexing load, CPU, heap/GC, disk, and per-shard load. Use slow logs to identify recurring outliers and Profile to inspect a representative query; Explain is for understanding why an individual document received its score. Fix the bottleneck supported by the evidence, then test the same workload again. Raising timeouts or adding shards without diagnosis can hide or worsen the issue.

#### Q8. How do I change an incompatible mapping without downtime?

Create a new versioned index with the intended mapping and settings, then backfill the data. Since writes can continue during the backfill, capture and replay changes and deletes from a durable source or change stream, and validate counts plus representative search results. Switch the application-facing alias atomically only after the destination is caught up. Keep the old index for a planned rollback window, but remember that switching the alias back is safe only if post-cutover writes can be reconciled.

#### Q9. How do I keep an Elasticsearch projection correct when updates arrive through SQL and Kafka?

Avoid relying on an application to write independently to SQL and Elasticsearch, because one write can succeed while the other fails. A transactional outbox or CDC can publish committed source changes to Kafka, and an idempotent consumer can update Elasticsearch using deterministic document IDs. If events can be duplicated or arrive out of order, enforce a source-version strategy so older events cannot silently replace newer state. Handle permanent failures in a dead-letter workflow, preserve deletes, and reconcile the search projection against the authoritative source.

#### Q10. How should tenant isolation, snapshots, RPO, and RTO be handled?

Derive tenant scope from trusted server-side authentication and apply it consistently to search, direct reads, aggregations, suggestions, exports, and pagination cursors. Routing by tenant can improve locality but is not authorization; add defensive negative tests and consider document-level security or separate indices when the threat model requires them. Snapshots protect against loss or accidental changes that replicas may not protect against, so test restoring them. RPO defines the tolerable amount of data loss in time, while RTO defines the target time to restore service; a snapshot schedule alone does not prove either target will be met.

### Official documentation used for technical cross-checking

- [Elasticsearch Java API Client and compatibility policy](https://www.elastic.co/docs/reference/elasticsearch/clients/java)
- [Boolean Query and minimum_should_match](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-bool-query)
- [Refresh API parameter and search visibility](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/refresh-parameter)
- [Paginating with search_after, PIT, and scroll](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/paginate-search-results)
- [Terms aggregation, shard_size, and count accuracy](https://www.elastic.co/docs/reference/aggregations/search-aggregations-bucket-terms-aggregation)
- [Reindex API and migration patterns](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reindex-indices)
- [Hybrid search and reciprocal rank fusion](https://www.elastic.co/docs/solutions/search/hybrid-search)
- [Disk allocation and shard allocation settings](https://www.elastic.co/docs/reference/elasticsearch/configuration-reference/cluster-level-shard-allocation-routing-settings)
- [Index blocks and read-only-allow-delete behavior](https://www.elastic.co/docs/reference/elasticsearch/index-settings/index-block)

**Final study advice:** do not memorize JSON alone. For every query, explain what the mapping must look like, why each clause belongs in query or filter context, what happens at scale, and how you would test correctness and performance.
