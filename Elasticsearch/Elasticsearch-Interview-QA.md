# Elasticsearch & Search Systems Interview Handbook

**Audience:** Java backend engineers with 5–10 years of experience targeting senior backend roles.  
**Coverage:** 140 numbered questions and answers, including 40 production scenarios, a 30-pattern Query DSL cookbook, architecture diagrams, Java integration, mock interviews, and revision checklists.

> **Compatibility note:** Elasticsearch behavior and APIs vary by major version and deployment model. REST examples use current-style Elasticsearch Query DSL. Java examples assume the official Java API Client 9.x style and Java 17+; select versions using the official compatibility matrix and the Spring Data release train used by your application. Examples are practical templates, not a substitute for running integration tests against your actual mappings and release.

## How to use this handbook

Learn the fundamentals before memorizing query patterns. For each production design, be ready to state the invariant, trace the data/request path, identify bottlenecks and failure modes, and describe how you would validate or roll back the change. Prioritize Query DSL practice: many Elasticsearch problems come from the wrong field mapping, a misunderstanding of query/filter context, or an unbounded query.

## Contents

1. [Search fundamentals](#1-search-fundamentals)
2. [Architecture and internals](#2-architecture-and-internals)
3. [Indexing and Lucene](#3-indexing-and-lucene)
4. [Mappings and analyzers](#4-mappings-and-analyzers)
5. [Query DSL deep dive](#5-query-dsl-deep-dive)
6. [Query DSL cookbook: 30 patterns](#6-query-dsl-cookbook-30-patterns)
7. [Relevance and autocomplete](#7-relevance-and-autocomplete)
8. [Pagination, sorting, aggregations](#8-pagination-sorting-aggregations)
9. [Index lifecycle and migrations](#9-index-lifecycle-and-migrations)
10. [Distributed behavior and consistency](#10-distributed-behavior-and-consistency)
11. [Performance and capacity](#11-performance-and-capacity)
12. [Reliability and troubleshooting](#12-reliability-and-troubleshooting)
13. [Java and Spring Boot](#13-java-and-spring-boot)
14. [SQL, Kafka, and microservices](#14-sql-kafka-and-microservices)
15. [Security and senior decisions](#15-security-and-senior-decisions)
16. [Frequently asked questions](#16-frequently-asked-questions)
17. [Scenario bank](#17-scenario-bank)
18. [Mock interviews and revision](#18-mock-interviews-and-revision)
19. [Official references](#19-official-references)

---

## 1. Search fundamentals

### Q1. What is Elasticsearch, and what problems does it solve?

Elasticsearch is a distributed search and analytics engine built on Apache Lucene. It indexes JSON documents and provides full-text search, structured filters, relevance ranking, aggregations, and near-real-time data exploration. An e-commerce site can use it to search product names, tolerate typos, filter by price/brand/stock, rank useful matches, and calculate facets. An observability platform can search logs and aggregate errors by service or time.

Elasticsearch is a search system, not a replacement for every database. Keep transactional invariants such as payment state and inventory reservation in an authoritative database. Elasticsearch is often a denormalized projection, which means indexing lag, replay, and reconciliation must be designed deliberately.

### Q2. How is Elasticsearch different from a relational database?

Relational databases excel at transactions, constraints, joins, and authoritative updates. Elasticsearch is optimized for text retrieval, distributed search, ranking, and aggregations over denormalized documents. Both use indexes, but the index structures and query behavior differ.

A product document might copy brand, category, price, rating, and stock into one search record to avoid joins while serving a product-search request. This improves retrieval but duplicates data and creates synchronization work. Use SQL for payment and inventory decisions; use Elasticsearch for discovery and search UX. If search can be stale by a few seconds, make that an explicit product contract rather than an accidental side effect.

### Q3. How is Elasticsearch different from a traditional database index?

A database B-tree index helps locate rows by structured predicates and ordering. Lucene's inverted index maps terms to documents and positions, enabling full-text retrieval; doc values support sorting and many aggregations. An analyzed text field can match tokens in a longer phrase, while a keyword field matches the whole indexed value.

An exact SKU lookup and a relevance-ranked description search are different access patterns. Elasticsearch handles both, but a normal primary-key lookup may be simpler in the source database. Choose based on the query, consistency, and operational requirements—not just because Elasticsearch is available.

### Q4. How does Elasticsearch compare with Solr and OpenSearch?

Elasticsearch and Apache Solr both build on Lucene and support full-text search and faceting, but have different APIs, operational models, and integrations. OpenSearch is a separate project that forked from an older Elasticsearch codebase; it has its own release and compatibility path. Do not assume every feature, setting, security capability, or query behavior is identical across products.

Compare deployment, licenses, managed services, plugins, upgrade path, security requirements, team expertise, and migration cost. An Elasticsearch Query DSL example should be labelled as Elasticsearch-specific unless it has been tested on the other distribution.

### Q5. When is Elasticsearch appropriate, and when is it the wrong tool?

It is appropriate when relevance-ranked text search, flexible filters, facets, log exploration, autocomplete, or search over denormalized data materially improves the product. It is less suitable as the primary store for relational constraints, multi-record transactions, payments, or inventory correctness. It may be unnecessary for a small app that only needs exact lookups.

A cluster adds mappings, indexing lag, JVM/disk operations, recovery, security configuration, and a second copy of data. A senior design should justify these costs with concrete search requirements and define the freshness and failure contract.

### Q6. Explain cluster, node, index, document, field, and mapping.

A cluster is a coordinated group of Elasticsearch nodes. Nodes hold shards and run search/index operations according to their roles. An index is a logical collection of documents. A document is a JSON object, a field is one named value, and a mapping defines the field's type and indexing behavior.

In a product catalogue, productId may be keyword, name and description text, category and brand keyword, price numeric, inStock boolean, and createdAt date. Mapping controls what queries are meaningful and efficient. A valid query can return no results if it assumes a field has the wrong type or analyzer.

---

## 2. Architecture and internals

### Q7. What are primary shards and replicas?

An index is split into primary shards, each owning a subset of its documents. Replica shards are copies of primary shards that support resilience and can serve search traffic. A primary and its replica copies cannot be allocated to the same node.

Primary-shard topology is chosen when creating an index; changing it generally requires a supported split/shrink workflow or reindexing. Replica count is more flexible but trades storage, indexing work, and recovery load for resilience and possible read capacity. More shards are not always faster because every shard adds overhead and many queries fan out to them.

### Q8. Which node roles matter?

Master-eligible nodes participate in cluster coordination and metadata management. Data nodes store shards and execute searches and writes. Ingest nodes run ingest pipelines. Dedicated coordinating nodes can receive requests, route work, and reduce results. Modern deployments also have specialized data tiers and other roles.

Every node can coordinate requests it receives. Dedicated coordination may isolate fan-out/reduction traffic at sufficient scale, but it adds a hop and needs capacity. Keep expensive workloads from starving cluster coordination. The right role layout depends on deployment, cluster size, and the mix of search, indexing, and analytics traffic.

### Q9. What happens when a search request reaches the cluster?

The receiving node coordinates the search. It routes shard-level work to eligible copies, gathers partial hits and aggregation results, merges them, and returns the final ordered response. Each shard runs Lucene search locally; the coordinator reduces the partial results.

~~~mermaid
flowchart TD
    A[Application] --> B[Coordinating node]
    B --> C[Shard group A]
    B --> D[Shard group B]
    B --> E[Shard group C]
    C --> F[Local Lucene search]
    D --> G[Local Lucene search]
    E --> H[Local Lucene search]
    F --> I[Partial results]
    G --> I
    H --> I
    I --> J[Coordinator reduction]
    J --> K[Response]
~~~

This fan-out explains why one hot or slow shard can dominate tail latency. Check timeout and shard-failure metadata, not only the HTTP status. A request that returns partial results may be unacceptable for an API that promises complete results.

### Q10. What do green, yellow, and red cluster health mean?

Green means all primary and replica shards are allocated. Yellow means all primaries are allocated but one or more replicas are unassigned. Red means at least one primary is unassigned, so some indexed data may be unavailable.

A single-node test cluster with replicas configured may remain yellow by design because a replica cannot colocate with its primary. In production, yellow after a node loss signals reduced redundancy; red calls for urgent investigation. Use allocation explain, node status, disk watermarks, allocation constraints, and recovery logs. Do not modify replica settings just to improve the colour without understanding the cause.

### Q11. How do writes and searches use primary and replica shards?

A write is routed to the primary shard, which processes it and replicates the operation to assigned replicas. A search may be served by eligible primary or replica copies. Acknowledgement depends on active-shard requirements, replication outcomes, and durability settings.

Replicas can improve fault tolerance and search capacity, but consume storage and receive indexing work. The wait-for-active-shards option controls the number of shard copies that must be active before a write proceeds; it is not a guarantee that the new document is instantly visible to search. Separate the concepts of accepted write, durability, and refresh visibility.

### Q12. How does shard routing work?

By default, Elasticsearch hashes the routing value, normally the document ID, to choose a primary shard. Custom routing can colocate related documents and reduce fan-out, but reads, updates, and deletes must use the compatible routing value. A routing mismatch can make a document appear missing.

Routing by tenant can make tenant searches cheaper, but a large tenant may create a hot shard. Routing is not authorization. Measure data skew and per-shard traffic before adopting it, and consider routing partitions or a separate data model when one tenant dominates the workload.

---

## 3. Indexing and Lucene

### Q13. What happens between an index request and search visibility?

Elasticsearch validates the document against mapping and ingest rules, routes it to the primary shard, and Lucene indexes term and field structures. The operation is recorded in the translog for recovery. A refresh opens recent segments for search. Background merges later compact smaller segments.

~~~mermaid
flowchart TD
    A[JSON write] --> B[Mapping and ingest validation]
    B --> C[Primary shard]
    C --> D[Lucene indexing]
    D --> E[Translog operation]
    D --> F[Refresh]
    F --> G[Search-visible document]
    D --> H[Background segment merge]
~~~

A real-time GET by ID can see a recent change before ordinary search does. A successful index request does not always mean the next search immediately sees the change. For bulk jobs, delaying refresh may improve throughput if the product can tolerate delayed search visibility.

### Q14. What is an inverted index?

An inverted index maps a term to documents and, when positional information is retained, the locations of that term within each document. Searching for “wireless” can identify candidate documents without scanning every JSON source. Analysis determines the indexed terms; positions support phrase search.

Lucene also uses other structures, such as doc values for sorting and aggregations, points for many numeric/date operations, and stored source data. These structures make common searches efficient, but broad wildcard expansions, expensive scripts, high-cardinality aggregations, and broad fan-out can still be costly.

### Q15. What is refresh, and why is search near-real-time?

Refresh exposes recent indexed operations to search by opening new Lucene segments. It is distinct from the write itself. Defaults vary by release and deployment, so application correctness should not depend on an assumed exact refresh interval.

Refreshing every write can reduce throughput. During a planned bulk import, lengthening the refresh interval and performing a refresh after the load may improve throughput when delayed visibility is acceptable. For write-then-read flows, consider returning the saved entity or using real-time GET by ID rather than forcing a refresh on every write.

### Q16. Explain refresh, flush, and translog durability.

Refresh makes changes searchable. Flush performs a Lucene commit and advances translog generation, allowing old translog files to be removed once safe. The translog holds operations needed to recover changes not yet represented by a Lucene commit. Translog durability settings affect what acknowledged writes survive a crash.

On standard self-managed deployments, request-based durability is commonly the default, while asynchronous durability may lose recent acknowledged operations after a failure. Verify the actual setting and deployment. Do not manually flush every request: flush does not replace refresh for search visibility and Elasticsearch performs flushes automatically.

### Q17. How do updates and deletes work with Lucene?

Lucene segments are immutable. An Elasticsearch update generally locates the old document and indexes a replacement version; a delete marks a document deleted and space is reclaimed later during merges. Frequent updates create merge and deleted-document overhead.

For high-churn entities, evaluate document size and update frequency. Sometimes a smaller search projection is better than rewriting a large denormalized document. If events can arrive out of order, use source versions or concurrency controls so an older event does not overwrite newer state.

### Q18. Why use Bulk API and how should it be tuned?

Bulk sends multiple index/create/update/delete actions in one request, reducing per-request overhead. The response reports each action separately: HTTP success does not mean all items succeeded. Inspect item errors, retry only retryable failures, and route permanent data errors to a dead-letter path.

Tune batch size and concurrency from measured throughput, bytes, latency, heap, disk I/O, merges, and rejected work. Huge batches create memory pressure and long retries; tiny batches waste overhead. Increase concurrency gradually and stop when throughput plateaus or latency/rejections rise.

---

## 4. Mappings and analyzers

### Q19. What is a mapping, and why make it explicit?

A mapping determines how fields are interpreted and indexed. Explicit mappings prevent an early malformed document from inferring the wrong type. Dynamic mapping is convenient for exploration but can lead to mapping conflicts, especially for dates, IDs, and fields with inconsistent types.

This example is the assumed product schema for the rest of the cookbook:

~~~json
PUT products-v1
{
  "settings": {
    "analysis": {
      "normalizer": {
        "folded_lowercase": {
          "type": "custom",
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "productId": { "type": "keyword" },
      "name": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "description": { "type": "text" },
      "brand": { "type": "keyword", "normalizer": "folded_lowercase" },
      "category": { "type": "keyword" },
      "price": { "type": "scaled_float", "scaling_factor": 100 },
      "inStock": { "type": "boolean" },
      "rating": { "type": "float" },
      "createdAt": { "type": "date" },
      "tags": { "type": "keyword" },
      "discontinued": { "type": "boolean" },
      "tenantId": { "type": "keyword" },
      "variants": {
        "type": "nested",
        "properties": {
          "color": { "type": "keyword" },
          "size": { "type": "keyword" },
          "sku": { "type": "keyword" }
        }
      }
    }
  }
}
~~~

Strict dynamic mapping rejects unknown fields until the contract is updated. The mapping is illustrative; choose numeric precision, normalization, and nested structures for the actual domain. Templates must be updated before new index generations are created.

### Q20. What is the difference between text and keyword?

Text is analyzed into terms and used for natural-language search. Keyword indexes the whole value as one term and is generally used for IDs, statuses, tags, filtering, sorting, and aggregations.

A match query on name analyzes the search phrase. A term query against text does not analyze its input and often fails when the input string is not itself one indexed term. Use a multi-field when the same source value needs both full-text search and exact sorting/facets, such as name and name.keyword. Always inspect the real mapping rather than assuming a subfield exists.

### Q21. What are analyzers, tokenizers, and token filters?

An analyzer transforms input during indexing and usually transforms query text in a corresponding way. Character filters can normalize text before tokenization; a tokenizer splits text into tokens; token filters lowercase, stem, remove stop words, expand synonyms, or transform terms.

A standard analyzer may turn “Wireless Bluetooth Headphones” into lowercase tokens. A keyword analyzer keeps the whole value as a single token. Language analyzers may improve recall but can be wrong for product names, brands, and SKUs. Test representative data with the Analyze API before changing analysis.

### Q22. What are multi-fields and normalizers?

Multi-fields index one source value in multiple ways, such as analyzed name for search and name.keyword for sorting. Each representation consumes storage and indexing work. A keyword normalizer transforms case/accent representation without splitting the value into separate terms, useful for case-insensitive exact matching.

Do not indiscriminately apply stemming or accent folding to identifiers and brand names. Choose transformations from the desired retrieval behavior and test precision, phrase matching, highlighting, and facets.

### Q23. How should dynamic templates be used?

Dynamic mapping infers types for newly encountered fields. Dynamic templates let the team apply field-name or detected-type rules, such as mapping fields ending in _id to keyword. This is useful for predictable log schemas, but uncontrolled field names can create mapping explosion.

Templates apply during index creation; updating a template does not rewrite existing index mappings. Use explicit mappings for core application fields and templates for known patterns. Monitor field counts and rejected documents, and treat incompatible type changes as migrations.

### Q24. What is the difference between object and nested?

Ordinary object arrays are flattened during indexing. If a document has one red-small variant and one blue-large variant, a query for red and large can incorrectly combine fields from different variants. Nested maps each object as a separate hidden document and lets a nested query require conditions to match the same child object.

Nested fields preserve relationships at additional indexing/query cost. Use them when the association between attributes matters; otherwise ordinary objects may be sufficient. For huge or independently updated child collections, consider a separate document model.

### Q25. How do you change a field's type safely?

An incompatible field type or analyzer usually needs a new index because existing indexed values were built using the old mapping. Create a versioned index, backfill, validate, then switch aliases. Changing a template alone does not fix an existing index.

When writes continue, capture a durable change-stream position, populate the new index from a consistent snapshot, replay changes after that position, validate counts and sample records, catch up, and then switch aliases. Keep the old index and understand how to reconcile writes after cutover before claiming rollback is safe.

### Q26. Explain doc values, stored fields, _source filtering, and fielddata.

_source stores the original JSON and is useful for retrieval/reindexing. Doc values are column-oriented structures used by many field types for sorting, aggregations, and scripts. Stored fields are values stored separately when configured. Source filtering restricts response fields but is not an authorization boundary.

Keyword and numeric types generally support doc values by default. Text does not; enabling fielddata on text can consume substantial heap. If sorting or aggregating text, prefer a suitable keyword multi-field instead of enabling fielddata without a measured reason.

---

## 5. Query DSL deep dive

### Q27. What is Query DSL?

Query DSL is Elasticsearch's JSON-based language for matching and scoring documents, combining clauses, filtering structured values, and running aggregations. Leaf queries include match, term, range, and exists. Compound queries include bool, dis_max, boosting, constant_score, and function_score.

A syntactically correct query can still return nothing if its field, mapping, analyzer, nested path, or date boundaries are wrong. Confirm the mapping and a sample document, then add one clause at a time. Use query context for relevance and filter context for binary constraints.

A match_all query is useful as a baseline when checking whether the index contains searchable documents:

~~~json
GET products-v1/_search
{
  "query": {
    "match_all": {}
  }
}
~~~

It matches every document in the targeted index; use a bounded size and avoid using it as an unfiltered production endpoint on a large index.

### Q28. Compare match and term.

Use match on analyzed text. It analyzes the query terms and matches documents according to operator and other settings:

~~~json
GET products-v1/_search
{
  "query": {
    "match": {
      "name": {
        "query": "wireless headphones",
        "operator": "and"
      }
    }
  }
}
~~~

Use term for exact indexed values like keyword IDs or status:

~~~json
GET products-v1/_search
{
  "query": {
    "term": { "brand": "acme" }
  }
}
~~~

This mapping normalizes brand to lowercase. A term query does not run the text analyzer; don't use it for a natural-language phrase in a text field. A match query is usually wrong for an identifier where tokenization could alter the value.

### Q29. Compare match_phrase and match_phrase_prefix.

match normally matches analyzed terms, often with OR behavior unless configured otherwise. match_phrase requires terms in the given order and positional relationship. match_phrase_prefix treats the final analyzed term as a prefix expansion and can support phrase autocomplete.

~~~json
GET products-v1/_search
{
  "query": {
    "match_phrase": {
      "description": "noise cancelling headphones"
    }
  }
}
~~~

Phrase-prefix can be expensive when expansions are broad. For high-QPS autocomplete, evaluate search_as_you_type, edge n-grams, or completion suggesters rather than assuming one query type will scale for every UX.

### Q30. Explain term, terms, range, exists, and ids.

term matches one exact indexed value; terms matches any value in a provided set; range matches numeric/date bounds; exists requires an indexed value; ids matches document IDs. Example:

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "filter": [
        { "terms": { "category": ["audio", "computers"] } },
        { "range": { "price": { "gte": 500, "lte": 5000 } } },
        { "term": { "inStock": true } },
        { "exists": { "field": "rating" } }
      ]
    }
  }
}
~~~

Structured predicates belong in filter context when they express eligibility rather than relevance. Nulls, empty arrays, ignore_above, and ignore_malformed can affect exists behavior. Confirm mapped field types and unit conventions.

The ids query matches Elasticsearch document IDs (_id), not an arbitrary business field. Use it only when the indexing pipeline deliberately sets _id to the product identifier:

~~~json
GET products-v1/_search
{
  "query": {
    "ids": {
      "values": ["P1001", "P1002"]
    }
  }
}
~~~

If productId is stored in _source but is not the document's _id, use a terms query against productId instead.

### Q31. When should prefix, wildcard, and regexp be used?

prefix matches terms starting with a prefix. wildcard supports * and ? patterns. regexp uses Lucene regex syntax, which is not identical to every programming language's engine. Broad expansions, especially leading wildcards, can be expensive.

~~~json
GET products-v1/_search
{
  "query": {
    "prefix": { "productId": "P10" }
  }
}
~~~

~~~json
GET products-v1/_search
{
  "query": {
    "wildcard": {
      "productId": { "value": "P*01" }
    }
  }
}
~~~

For substring or autocomplete needs, consider edge n-grams, search-as-you-type, a completion suggester, or a specialized wildcard field.

A regular-expression example is:

~~~json
GET products-v1/_search
{
  "query": {
    "regexp": {
      "productId": {
        "value": "P10[0-9][0-9]"
      }
    }
  }
}
~~~

This pattern matches indexed IDs that start with P10 and end with two digits. Regex patterns can be expensive when they expand broadly. Bound user patterns, monitor latency, and check expensive-query controls for the deployed version.

### Q32. How does fuzzy matching work?

Fuzzy matching retrieves terms within a configured edit distance, helping with spelling mistakes. It is not semantic search, and short tokens can produce excessive expansions and poor precision.

~~~json
GET products-v1/_search
{
  "query": {
    "match": {
      "name": {
        "query": "bluetooh headphons",
        "fuzziness": "AUTO",
        "prefix_length": 1,
        "max_expansions": 25
      }
    }
  }
}
~~~

Use fuzziness selectively, especially for user-entered text rather than identifiers. A prefix length can reduce expansion work. Validate behavior with representative queries, because typo tolerance can increase false positives and CPU cost.

### Q33. How does multi_match and field boosting work?

multi_match searches several fields using a combination type. A title often deserves more weight than a description:

~~~json
GET products-v1/_search
{
  "query": {
    "multi_match": {
      "query": "wireless noise cancelling headphones",
      "type": "best_fields",
      "fields": ["name^4", "description^1.5"],
      "tie_breaker": 0.1
    }
  }
}
~~~

best_fields tends to favor the strongest field match; other types such as most_fields and cross_fields combine evidence differently. A boost is relative scoring influence, not a guaranteed business-relevance multiplier. Judge results with real queries, and avoid searching every field by default.

### Q34. query_string versus simple_query_string?

query_string supports rich operators and can fail on malformed syntax. simple_query_string is more forgiving and can ignore invalid syntax, making it suitable for a constrained user search box. Neither should be treated as an authorization boundary.

For example, query_string supports a richer parser syntax, whereas simple_query_string is more forgiving:

~~~json
GET products-v1/_search
{
  "query": {
    "query_string": {
      "query": "wireless AND headphones",
      "default_field": "name"
    }
  }
}
~~~

~~~json
GET products-v1/_search
{
  "query": {
    "simple_query_string": {
      "query": "wireless +headphones -wired",
      "fields": ["name^3", "description"]
    }
  }
}
~~~

Use query_string only when its richer syntax is intentional; invalid syntax can fail the request. The simple form ignores invalid syntax, but both still need field allowlists and resource limits.

Limit query length, searchable fields, supported operators, aggregation size, and execution budget. Build structured filters with Query DSL instead of concatenating arbitrary user JSON, and derive security-sensitive filters from trusted server context.

### Q35. Explain bool, must, should, filter, and must_not.

must requires a match and contributes to scoring. filter requires a match without scoring. must_not excludes documents in filter context. should contributes optional relevance or required alternatives according to minimum_should_match and the other clauses.

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "must": [
        { "multi_match": {
            "query": "wireless headphones",
            "fields": ["name^4", "description"]
        }}
      ],
      "filter": [
        { "term": { "inStock": true } },
        { "range": { "price": { "gte": 1000, "lte": 8000 } } }
      ],
      "must_not": [
        { "term": { "discontinued": true } }
      ],
      "should": [
        { "range": { "rating": { "gte": 4.5, "boost": 2 } } }
      ]
    }
  }
}
~~~

This query ranks text matches, requires the stock/price criteria, excludes discontinued products, and boosts highly rated products. Explain which conditions are mandatory and which only affect ranking; confusing them creates surprising result sets.

### Q36. How does minimum_should_match work?

If bool has should clauses but no must/filter, the default minimum is 1. If must or filter exists, the default is 0, so should clauses usually become optional boosts. Set minimum_should_match explicitly when a business rule requires one or more alternatives.

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "must": [{ "match": { "name": "wireless headphones" } }],
      "should": [
        { "term": { "brand": { "value": "acme", "boost": 2 } } },
        { "range": { "rating": { "gte": 4.5, "boost": 1.5 } } }
      ],
      "minimum_should_match": 0
    }
  }
}
~~~

Counts and percentages need care for short queries. If a match query asks for two terms but receives a one-term query, no document can satisfy the rule. Adapt the query policy to the analyzed input length.

### Q37. Query context versus filter context; is caching guaranteed?

Query context answers whether a document matches and how well it matches; filter context answers only whether it matches. Use query context for text relevance and filter context for structured eligibility such as tenant, category, state, price, and time.

Some filter results may be cached by Elasticsearch's node query cache, but caching is heuristic and not guaranteed for every filter. Never claim a particular filter is always cached. If the request sorts only by price, price determines order; relevance is no longer the primary sort unless _score is explicitly part of the sort.

### Q38. Explain constant_score, boosting, and dis_max.

constant_score wraps a filter and gives matching documents a constant score. boosting combines a positive query with a negative query whose matches are demoted rather than excluded. dis_max returns the highest subquery score and optionally adds part of other matching scores using tie_breaker.

~~~json
GET products-v1/_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "name": { "query": "wireless headphones", "boost": 4 } } },
        { "match": { "description": { "query": "wireless headphones", "boost": 1 } } }
      ],
      "tie_breaker": 0.15
    }
  }
}
~~~

Use dis_max when the strongest field match should dominate, such as title over body. Use bool/more additive strategies when evidence across fields should accumulate. Choose from relevance testing rather than syntax preference.

A constant_score query gives all documents passing its filter the same score for that clause:

~~~json
GET products-v1/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "brand": "acme"
        }
      },
      "boost": 1.5
    }
  }
}
~~~

The boosting query can reduce the score of documents matching a negative query without excluding them:

~~~json
GET products-v1/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "name": "wireless headphones"
        }
      },
      "negative": {
        "term": {
          "discontinued": true
        }
      },
      "negative_boost": 0.2
    }
  }
}
~~~

Use must_not instead when discontinued items must never appear. Boosting is a ranking preference, not an eligibility filter.

### Q39. When should function_score or script_score be used?

function_score combines text scores with functions such as field-value factors, weights, or recency decay. score_mode defines how functions combine; boost_mode defines how the combined function score interacts with the base query score.

~~~json
GET products-v1/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "name": "wireless headphones" } },
      "field_value_factor": {
        "field": "rating",
        "modifier": "sqrt",
        "factor": 0.3,
        "missing": 1
      },
      "boost_mode": "sum",
      "score_mode": "multiply",
      "max_boost": 3
    }
  }
}
~~~

Test how values and missing fields change ranking; a function can swamp text relevance if poorly scaled. script_score is flexible but may be expensive per matching document. Prefer built-in functions when they express the rule, and profile scripts under realistic load.

For example, this scripted multiplier keeps the base text score and applies a modest rating adjustment, with a fallback when rating is missing:

~~~json
GET products-v1/_search
{
  "query": {
    "script_score": {
      "query": {
        "match": {
          "name": "wireless headphones"
        }
      },
      "script": {
        "source": "_score * (doc['rating'].size() == 0 ? 1.0 : 1.0 + doc['rating'].value / 10.0)"
      }
    }
  }
}
~~~

This script assumes rating is numeric and values are in the expected range. Scripts run for matching documents and can increase CPU, so prefer field_value_factor or decay functions if they express the business rule. Test missing-value behavior and cap query scope.

---

## 6. Query DSL cookbook: 30 patterns

Examples assume the mapping from Q19. Each request is illustrative and should be tested against the target mapping. Text queries score; structured filters generally do not.

### Pattern 1 — Search by name

~~~json
GET products-v1/_search
{
  "query": {
    "match": {
      "name": { "query": "wireless headphones", "operator": "and" }
    }
  }
}
~~~

Use for analyzed text; AND requires both terms and narrows recall compared with OR.

### Pattern 2 — Exact product identifier

~~~json
GET products-v1/_search
{
  "query": { "term": { "productId": "P1001" } }
}
~~~

productId is keyword. For a single Elasticsearch document whose _id is known, GET by ID is simpler; business productId is not automatically the same as _id.

### Pattern 3 — Filter brand and category

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "brand": "acme" } },
        { "term": { "category": "audio" } }
      ]
    }
  }
}
~~~

Both are structured constraints and should not affect text relevance.

### Pattern 4 — Search name and description with boosts

~~~json
GET products-v1/_search
{
  "query": {
    "multi_match": {
      "query": "noise cancelling",
      "fields": ["name^4", "description"],
      "type": "best_fields"
    }
  }
}
~~~

The title is weighted more strongly than the description. Verify the result ranking rather than interpreting boosts as exact percentages.

### Pattern 5 — Exact phrase

~~~json
GET products-v1/_search
{
  "query": {
    "match_phrase": {
      "description": "noise cancelling over ear"
    }
  }
}
~~~

Analyzed terms must appear in order. Analyzer rules can affect positions and phrase behavior.

### Pattern 6 — Require at least two terms

~~~json
GET products-v1/_search
{
  "query": {
    "match": {
      "name": {
        "query": "bluetooth wireless headphones",
        "operator": "or",
        "minimum_should_match": "2"
      }
    }
  }
}
~~~

Require two analyzed terms; adapt the policy for a single-token input.

### Pattern 7 — Full-text plus stock and price

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "must": [
        { "multi_match": {
            "query": "wireless headphones",
            "fields": ["name^4", "description"]
        }}
      ],
      "filter": [
        { "term": { "inStock": true } },
        { "range": { "price": { "gte": 1000, "lte": 6000 } } }
      ]
    }
  }
}
~~~

Text controls ranking; numeric price and stock determine eligibility. Price must be mapped numeric and use a consistent currency/unit.

### Pattern 8 — Exclude discontinued products

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "must": [{ "match": { "name": "headphones" } }],
      "must_not": [{ "term": { "discontinued": true } }]
    }
  }
}
~~~

Missing discontinued values are not the same as true; define how missing values should behave.

### Pattern 9 — Date range

~~~json
GET products-v1/_search
{
  "query": {
    "range": {
      "createdAt": {
        "gte": "2026-01-01T00:00:00Z",
        "lt": "2026-02-01T00:00:00Z"
      }
    }
  }
}
~~~

A half-open interval avoids overlap at boundaries. Keep timezone conventions and mapped date format consistent.

### Pattern 10 — Sort by price, rating, and relevance

~~~json
GET products-v1/_search
{
  "query": { "match": { "name": "wireless headphones" } },
  "sort": [
    { "price": "asc" },
    { "rating": { "order": "desc", "missing": "_last" } },
    { "_score": "desc" },
    { "productId": "asc" }
  ]
}
~~~

The first field dominates; this example sorts primarily by price. For relevance-first ordering, put _score first or encode business signals in scoring. productId gives deterministic final ordering.

### Pattern 11 — Source filtering

~~~json
GET products-v1/_search
{
  "_source": ["productId", "name", "price", "rating"],
  "query": { "match": { "name": "wireless headphones" } }
}
~~~

Return fewer fields to reduce payload. Source filtering is not a security boundary.

### Pattern 12 — Shallow pagination

~~~json
GET products-v1/_search
{
  "from": 20,
  "size": 10,
  "query": { "match": { "name": "headphones" } }
}
~~~

Offsets are zero-based. Deep from/size is expensive because shards collect candidates through from + size. Check the index result-window limit.

### Pattern 13 — search_after

~~~json
GET products-v1/_search
{
  "size": 20,
  "query": { "match": { "name": "headphones" } },
  "sort": [
    { "createdAt": "desc" },
    { "productId": "asc" }
  ],
  "search_after": ["2026-06-10T13:45:00Z", "P1001"]
}
~~~

Use the complete sort values from the last hit of the previous response; keep the query and sort unchanged. Add a unique tie-breaker.

### Pattern 14 — Point in time (PIT)

~~~json
POST /products-v1/_pit?keep_alive=1m
~~~

~~~json
GET /_search
{
  "size": 20,
  "pit": { "id": "PIT_ID_FROM_RESPONSE", "keep_alive": "1m" },
  "query": { "match": { "name": "headphones" } },
  "sort": [{ "createdAt": "desc" }]
}
~~~

PIT stabilizes the index view for multi-page searches. Use the latest PIT ID and complete returned sort array with search_after, and close the PIT when finished. Open contexts consume resources.

### Pattern 15 — Exact total hits

~~~json
GET products-v1/_search
{
  "track_total_hits": true,
  "query": { "match": { "name": "headphones" } }
}
~~~

Exact total counting can cost more than a lower-bound count. Do not request exact totals for every autocomplete keystroke unless the UX needs them.

### Pattern 16 — Require a percentage of matching terms

~~~json
GET products-v1/_search
{
  "query": {
    "match": {
      "description": {
        "query": "wireless bluetooth noise cancelling headphones",
        "minimum_should_match": "75%"
      }
    }
  }
}
~~~

This is meaningful for OR-style analyzed matching. Stop words and analysis change how many terms exist, so test the actual analyzer output.

### Pattern 17 — Phrase-prefix autocomplete

~~~json
GET products-v1/_search
{
  "size": 10,
  "query": {
    "match_phrase_prefix": {
      "name": { "query": "wireless blue", "max_expansions": 25 }
    }
  }
}
~~~

Useful as a simple first implementation; bound expansions and benchmark. search_as_you_type or edge n-grams may scale better.

### Pattern 18 — Prefix suggestion

~~~json
GET products-v1/_search
{
  "size": 10,
  "query": { "prefix": { "brand": "ac" } }
}
~~~

The mapping normalizes brand to lowercase. Prefix cost depends on the field and index; dedicated suggesters may be better for high QPS.

### Pattern 19 — Fuzzy typo tolerance

~~~json
GET products-v1/_search
{
  "query": {
    "match": {
      "name": {
        "query": "bluetooh headphons",
        "fuzziness": "AUTO",
        "prefix_length": 1,
        "max_expansions": 25
      }
    }
  }
}
~~~

Typo tolerance can increase recall but also CPU and false positives. Apply selectively and test on representative inputs.

### Pattern 20 — Exact phrase and name boosts

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "name": { "query": "wireless headphones", "boost": 4 } } },
        { "match_phrase": { "name": { "query": "wireless headphones", "boost": 6 } } },
        { "match": { "description": { "query": "wireless headphones", "boost": 1 } } }
      ],
      "minimum_should_match": 1
    }
  }
}
~~~

At least one clause must match. Excessive overlapping boosts can double-count evidence; evaluate ranking quality.

### Pattern 21 — Recency boost

~~~json
GET products-v1/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "name": "wireless headphones" } },
      "functions": [
        {
          "gauss": {
            "createdAt": {
              "origin": "2026-10-01T00:00:00Z",
              "scale": "30d",
              "offset": "3d",
              "decay": 0.5
            }
          }
        }
      ],
      "boost_mode": "multiply",
      "score_mode": "multiply"
    }
  }
}
~~~

Decay gives recent documents more influence without a hard cutoff. Multiplication can suppress textual relevance too much; calibrate it.

### Pattern 22 — Same nested variant matches color and size

~~~json
GET products-v1/_search
{
  "query": {
    "nested": {
      "path": "variants",
      "query": {
        "bool": {
          "filter": [
            { "term": { "variants.color": "red" } },
            { "term": { "variants.size": "large" } }
          ]
        }
      },
      "score_mode": "none"
    }
  }
}
~~~

This works because variants is mapped nested. A plain object array can cross-match fields from different variants.

### Pattern 23 — Logs by severity, service, and time

~~~json
GET application-logs-*/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "service.name": "checkout" } },
        { "terms": { "log.level": ["error", "fatal"] } },
        { "range": { "@timestamp": { "gte": "now-1h", "lt": "now" } } }
      ]
    }
  },
  "sort": [{ "@timestamp": "desc" }]
}
~~~

Requires keyword mappings for service and level and a date mapping for timestamp. Bound the time range.

### Pattern 24 — Tenant-scoped search

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "must": [{ "match": { "name": "headphones" } }],
      "filter": [{ "term": { "tenantId": "tenant-42" } }]
    }
  }
}
~~~

Derive tenantId from authenticated identity on the server, not an untrusted caller input. Apply the scope to IDs, facets, suggestions, and exports too.

### Pattern 25 — Brand and price facets

~~~json
GET products-v1/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [{ "match": { "name": "headphones" } }],
      "filter": [{ "term": { "inStock": true } }]
    }
  },
  "aggs": {
    "brands": { "terms": { "field": "brand", "size": 20 } },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 1000 },
          { "from": 1000, "to": 3000 },
          { "from": 3000 }
        ]
      }
    }
  }
}
~~~

size 0 returns aggregations without hits. Terms returns top buckets, not necessarily every possible value.

### Pattern 26 — Top products in each category

~~~json
GET products-v1/_search
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": { "field": "category", "size": 10 },
      "aggs": {
        "top_products": {
          "top_hits": {
            "size": 3,
            "sort": [{ "rating": "desc" }],
            "_source": ["productId", "name", "rating"]
          }
        }
      }
    }
  }
}
~~~

top_hits returns a few hits per bucket. Keep both category bucket count and hits per bucket bounded.

### Pattern 27 — Find missing ratings

~~~json
GET products-v1/_search
{
  "query": {
    "bool": {
      "must_not": [{ "exists": { "field": "rating" } }]
    }
  }
}
~~~

This finds documents without an indexed rating value. Nulls, empty arrays, and ignored values can affect exists semantics.

### Pattern 28 — Highlight matches

~~~json
GET products-v1/_search
{
  "query": { "match": { "description": "noise cancelling" } },
  "highlight": { "fields": { "description": {} } }
}
~~~

Highlight fragments help explain matches in a UI. Check cost on large fields, and escape returned content safely in the client.

### Pattern 29 — Practical search endpoint query

~~~json
GET products-v1/_search
{
  "from": 0,
  "size": 20,
  "_source": ["productId", "name", "brand", "price", "rating"],
  "query": {
    "bool": {
      "must": [
        { "multi_match": {
            "query": "bluetooth headphones",
            "fields": ["name^4", "description"]
        }}
      ],
      "filter": [
        { "term": { "tenantId": "tenant-42" } },
        { "term": { "inStock": true } },
        { "range": { "price": { "gte": 1000, "lte": 10000 } } }
      ],
      "must_not": [{ "term": { "discontinued": true } }],
      "should": [{ "range": { "rating": { "gte": 4.5, "boost": 1.5 } } }]
    }
  },
  "sort": [
    { "_score": "desc" },
    { "rating": { "order": "desc", "missing": "_last" } },
    { "productId": "asc" }
  ]
}
~~~

This combines text relevance with server-controlled tenant/stock/price constraints. The score sort is primary, rating breaks ties, and productId gives deterministic ordering.

### Pattern 30 — Debug a zero-result query

~~~json
GET products-v1/_mapping
GET products-v1/_doc/P1001
POST products-v1/_analyze
{
  "field": "name",
  "text": "Wireless Bluetooth Headphones"
}
~~~

Then test the broad match first and add filters one at a time. Check field names, mapping types, casing/normalizer behavior, dates/timezones, nested paths, and missing values. Use Explain for an individual document's match/score reasoning and Profile for targeted execution details; profiling adds overhead and should not run on every production request.

---

## 7. Relevance and autocomplete

### Q40. Explain BM25.

BM25 is the common default text similarity. Its score depends on term frequency, inverse document frequency, field length, and query structure. Repeating a term can increase relevance but with diminishing influence; rare terms are often more discriminative than common ones; and long fields are normalized differently from short fields.

BM25 is a baseline, not a business-quality guarantee. Analyze output, field boosts, synonyms, phrase clauses, filters, and collection statistics all influence results. Use real search queries and judged results, inspect Explain for individual documents, and compare relevance regressions when mappings or corpus size change. Scores are best used to rank hits within a query, not compared as absolute scores across unrelated queries.

### Q41. How should synonyms, stemming, and stop words be handled?

Synonyms improve recall for equivalent terms; stemming merges word variants; stop-word filters may remove frequent words. These transformations can help or hurt precision and phrase behavior. Index-time changes generally require reindexing; search-time synonym approaches may allow easier updates but depend on analyzer and query compatibility.

Test representative product names, multi-word synonyms, phrase queries, highlighting, and negative examples. Do not stem every identifier or brand. A relevance test set should be part of analyzer releases.

### Q42. How do you design autocomplete?

Choose a primitive based on the UX: completion suggester for navigational suggestions, search_as_you_type for incremental text matching, edge n-grams for indexed prefixes, or match_phrase_prefix for a simple phrase-like start. Fuzzy matching can help with typos but adds expansion cost and noise.

Autocomplete is high-QPS. Restrict fields and result count, debounce requests, avoid exact hit totals, limit query length, and set useful minimum input length. Measure latency and relevance together: a fast suggestion that is not useful is still a failure.

### Q43. How do you combine text relevance, rating, popularity, and recency?

Keep text relevance as the base query and place hard business requirements in filters. Use phrase/field boosts and bounded function_score features for rating or recency. Decide how to treat missing values, new products, and outliers. Recency decay often works better than an abrupt cutoff.

The correct ranking is a product decision. Evaluate with representative queries and judged results, monitor downstream outcomes such as no-result rate or conversion, version ranking rules, and preserve rollback.

### Q44. How do you debug unexpectedly high or low scores?

Check field selection, analyzer, query operator, boosts, overlapping clauses, missing-value behavior, and corpus changes. Use Analyze to inspect tokens and Explain to see why a particular document matched and how its score was composed. Profile is for execution cost rather than the primary explanation of score semantics.

A score is not a universal quality label; scores from separate queries are not necessarily comparable. If a threshold is required, calibrate it against real result judgements rather than choosing a number intuitively.

---

## 8. Pagination, sorting, aggregations

### Q45. When should from/size, search_after, or scroll be used?

from/size works well for shallow page-number pagination. Each shard may need to collect up to from + size candidates, so deep offsets cost more. The default result window is commonly 10,000 unless changed. search_after advances using the last hit's sort values and is better for deep sequential traversal. PIT preserves a stable view during multi-page search and should be closed when done.

Scroll is designed for batch processing rather than interactive pagination. Current Elasticsearch guidance generally favors PIT plus search_after for deep pagination requiring a consistent view. Keep query and sort identical across pages and include a stable tie-breaker.

### Q46. Why is deterministic sorting important?

If multiple hits share sort values, order may change across shard copies or refreshes. Without a unique tie-breaker, pagination can duplicate or skip results. For products, add a unique keyword like productId as a final sort key. With PIT, Elasticsearch includes an implicit _shard_doc tie-breaker; send the full returned sort array with search_after and use the latest PIT ID.

### Q47. What are aggregations? Bucket versus metric?

Aggregations summarize matching documents. Bucket aggregations group by values/ranges/time intervals; metric aggregations calculate values such as sum, avg, min, max, value_count, stats, and extended_stats.

~~~json
GET products-v1/_search
{
  "size": 0,
  "query": { "term": { "category": "audio" } },
  "aggs": {
    "price_stats": { "stats": { "field": "price" } },
    "average_rating": { "avg": { "field": "rating" } },
    "brands": { "terms": { "field": "brand", "size": 10 } }
  }
}
~~~

stats returns count/min/max/avg/sum; average_rating returns a value; brands returns buckets. Interpret missing values and multi-valued fields according to mapping and business semantics.

### Q48. Explain terms, filter, filters, range, histogram, and date_histogram.

terms groups distinct values. filter makes one bucket for a predicate, filters makes several named predicate buckets, range defines numeric/date intervals, histogram creates fixed numeric bins, and date_histogram groups timestamps by calendar or fixed intervals.

~~~json
GET application-logs-*/_search
{
  "size": 0,
  "query": {
    "range": { "@timestamp": { "gte": "now-24h", "lt": "now" } }
  },
  "aggs": {
    "per_hour": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      },
      "aggs": {
        "errors": {
          "filter": {
            "terms": { "log.level": ["error", "fatal"] }
          }
        }
      }
    }
  }
}
~~~

Use fixed_interval for fixed durations and calendar_interval for calendar boundaries. Time zone and daylight saving can affect buckets.

### Q49. Explain value_count, cardinality, stats, extended_stats.

value_count counts field values and can count multiple values in one document. cardinality estimates distinct values and is generally approximate. stats calculates count, min, max, avg, and sum; extended_stats adds variance and standard deviation. Do not use an approximate cardinality metric for a requirement that demands exact accounting.

High-cardinality aggregations can consume memory and CPU. Understand missing-value and multi-value semantics before interpreting metrics.

### Q50. Why can distributed terms aggregations be approximate or expensive?

Shards send local candidate buckets to the coordinator, which merges them. A bucket omitted from a shard's local top candidates can make global counts approximate. Increasing shard_size can improve accuracy at additional cost. High-cardinality values and nested bucket combinations increase memory and response size.

Bound bucket counts, use top terms for interactive facets, and use composite aggregation to paginate through many buckets. Do not confuse terms with a guaranteed exhaustive exact GROUP BY over every possible value.

### Q51. How do nested aggregations and top_hits work?

Nested aggregations operate on nested child documents and preserve child relationships. top_hits returns sample documents per bucket, such as three top-rated products in each category.

~~~json
GET products-v1/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 10 },
      "aggs": {
        "top_rated": {
          "top_hits": {
            "size": 3,
            "sort": [{ "rating": "desc" }],
            "_source": ["productId", "name", "rating"]
          }
        }
      }
    }
  }
}
~~~

These structures can be expensive when both bucket count and hit count are large. Cap both, and consider a separate search for a complete category list.

### Q52. When should composite aggregation be used?

Composite aggregation paginates buckets through an after_key. Use it to enumerate many unique values or combinations such as brand/category without returning all buckets at once. This is different from search_after, which paginates document hits.

~~~json
GET products-v1/_search
{
  "size": 0,
  "aggs": {
    "brand_category": {
      "composite": {
        "size": 500,
        "sources": [
          { "brand": { "terms": { "field": "brand" } } },
          { "category": { "terms": { "field": "category" } } }
        ]
      }
    }
  }
}
~~~

Use the response after_key in the next request and preserve the aggregation definition. For a repeatable enumeration on changing data, design a suitable snapshot strategy.

---

## 9. Index lifecycle and migrations

### Q53. What are aliases, index templates, and component templates?

Aliases are logical names that decouple applications from concrete indices and enable atomic cutover. Index templates define mappings, settings, and aliases for new matching indices. Component templates provide reusable pieces. Use a stable product-search alias for versioned indices such as products-v1 and products-v2.

Updating a template doesn't retrofit an existing index. Inspect mapping/settings after changes, and test a newly created index. A write alias pointing to multiple indices should clearly identify the write index where required.

### Q54. How do you plan a large reindex?

Create the destination mapping/settings first. Estimate storage for source, destination, replicas, translog, and temporary recovery overhead. Load-test and rate-limit the job based on cluster health, disk, heap, merges, and query latency. Inspect task status and failures.

If writes continue, reindex alone is not a synchronization protocol. Establish a durable change-stream position, backfill a consistent snapshot, replay later mutations, validate counts and critical queries, catch up, and cut over aliases. Keep the old index until rollback risk is understood.

### Q55. Explain ILM, rollover, data streams, and data tiers.

ILM can roll over indices, force-merge, move data across supported hot/warm/cold/frozen tiers, and delete data by policy. Data streams provide a named abstraction over backing indices for append-oriented time-series data. Exact capabilities differ between self-managed, Elastic Cloud, and Serverless deployments.

Use rollover based on size/document count as well as age to avoid thousands of tiny shards. Retention must reflect compliance, search demand, recovery, and replay requirements. Validate the current deployment's lifecycle options before using settings from a different environment.

### Q56. How do aliases enable a zero-downtime mapping migration?

Create a new versioned index, backfill it, replay concurrent changes, validate counts/results, and atomically change aliases. The alias update is atomic metadata routing; it does not solve data synchronization by itself.

~~~mermaid
flowchart LR
    A[Application] --> B[Stable aliases]
    B --> C[products-v1]
    D[Snapshot plus change stream] --> E[products-v2]
    C --> F[Validation]
    E --> F
    F --> G[Atomic alias cutover]
    G --> H[Aliases point to v2]
~~~

Keep old data until rollback is safe. Writes made after cutover may not exist in v1, so rollback may require reconciliation or reverse synchronization.

### Q57. Explain update_by_query, delete_by_query, and snapshots.

Update/delete by query affect matching documents and can conflict with concurrent updates. Throttling and slicing can control resource usage, but inspect task status and version conflicts. Test destructive predicates carefully.

Snapshots are the supported backup mechanism for indices and cluster metadata. They are not useful until restore is tested. Define RPO/RTO, secure the repository, retain backups appropriately, and practise restore. Replicas are not backups because accidental deletes and corrupt writes can propagate to replicas.

---

## 10. Distributed behavior and consistency

### Q58. Distinguish write acknowledgement, durability, and search visibility.

A successful write follows the configured acknowledgement path. Translog durability controls recovery of operations not yet in a Lucene commit, while refresh makes recently indexed changes visible to search. These are distinct guarantees.

Real-time GET often sees recent changes before a search does. For read-after-write, return the saved entity, use a real-time GET where suitable, wait for refresh when justified, or tolerate bounded lag. Elasticsearch does not provide arbitrary multi-document transactions; preserve critical invariants in the authoritative database.

### Q59. Explain optimistic concurrency control.

Elasticsearch sequence numbers and primary terms can be used to condition updates on the version the caller last read. If the document changed, the write conflicts rather than silently overwriting another update. Handle conflict as a concurrency outcome: reread, merge, retry with a new version, or return a conflict to the user.

For event projections, an authoritative source version can help prevent stale replay events from replacing a newer document. Blindly retrying stale content is not a correct concurrency strategy.

### Q60. What happens during node failure or shard relocation?

Cluster coordination updates state and Elasticsearch recovers shards from replicas or peers where possible. A replica can be promoted if the primary is unavailable and a suitable copy exists. Lost copies may require snapshot or source-system recovery.

Failure often creates concurrent pressure from relocation, recovery I/O, rebalancing, reduced search capacity, and node overload. Reserve capacity for a node/zone failure. Modern coordination uses voting/quorum mechanisms; investigate the actual cluster version and logs rather than repeating outdated split-brain explanations.

### Q61. How should retries, idempotency, and partial failures work?

A timeout does not prove an operation failed; the response may be lost after the operation was applied. Use deterministic IDs, source event IDs, source versions, and optimistic concurrency where required. Retry transient errors with exponential backoff and jitter. For bulk requests, retry only retryable item failures and route permanent mapping errors to a DLQ.

Track consumed, successful, retried, dead-lettered, and reconciled events. At-least-once delivery is safe only when duplicate effects are explicitly handled.

---

## 11. Performance and capacity

### Q62. How should shard count and size be chosen?

Estimate primary data size, document growth, retention, query parallelism, nodes, and failure headroom. Very large shards recover slowly; too many tiny shards increase metadata, heap, file-handle, and fan-out overhead. There is no universal ideal shard size: benchmark the real document mapping and workload.

Plan space for replicas, merges, translog, recovery, and temporary source/destination copies during reindex. Use rollover for growing time series and monitor size distribution.

### Q63. How do you tune bulk throughput?

Tune request bytes and worker concurrency experimentally. Monitor rejections, indexing pressure, heap, CPU, disk write latency, merge time, refresh overhead, and search SLOs. High concurrency may increase queueing rather than throughput.

For planned offline imports, a longer refresh interval may help. Do not remove replicas or disable refresh in production without an explicit availability/durability plan. Retry partial failures, not whole successful batches blindly.

### Q64. How do heap, filesystem cache, CPU, disk, and network affect search?

Heap holds runtime structures, aggregation state, caches, and metadata. Lucene also relies heavily on filesystem cache; oversized heap can starve index reads. CPU affects parallel query work; disk affects reads, writes, merges, and recovery; network affects fan-out, replication, and relocation.

Observe post-GC heap, GC pauses, disk latency, filesystem cache pressure, network, per-node CPU, hot threads, queue rejections, and per-shard load. Don't blindly increase heap or cache. Find the workload and bottleneck first.

### Q65. What makes a query expensive?

Broad wildcard/regexp and fuzzy expansions, scripts, high-cardinality aggregations, unbounded time ranges, deep offsets, large nested queries, and broad shard fan-out can be expensive. Query cost also depends on the mapping and index structures.

Use slow logs for recurring outliers and Profile for a representative query; profiling adds overhead. Bound patterns, time ranges, buckets, page sizes, and fields. Request validation and rate limits protect cluster availability just as much as code security.

### Q66. What should production observability cover?

Track search/index latency percentiles, throughput, timeouts, partial results, shard failures, query rejections, bulk item failures, consumer lag, index freshness, heap/GC, disk, filesystem cache, recoveries, and shard distribution. Use cluster/node/index stats, allocation explain, task status, slow logs, Profile and Explain selectively.

High p99 is a symptom, not a root cause. Connect request IDs and sanitized query shapes to shard/cluster metrics so the team can distinguish queueing, hot shards, expensive aggregations, network issues, and indexing lag.

### Q67. How do you estimate capacity?

For an initial estimate, documents × average indexed bytes gives only a rough primary-store baseline. For example, 50 million documents averaging 2 KB of payload represent around 100 GB before index structures, replicas, translog, merges, and headroom. Measure actual storage per indexed document; analyzers and mappings substantially affect it.

For logs, estimate events/sec × indexed bytes/event × retention, then add replicas and operating headroom. Load-test the real query mix under concurrent indexing and node-failure conditions. Capacity math is a starting hypothesis, not proof.

---

## 12. Reliability and troubleshooting

### Q68. How do you diagnose yellow or red health?

Find unassigned shards and use allocation explain. Check failed nodes, disk watermarks, allocation filters, awareness rules, available distinct nodes, and recovery logs. Yellow means primaries are allocated but redundancy is reduced. Red means at least one primary is unavailable.

Recover the node or allocate from valid copies using supported procedures. Do not force a stale primary without understanding data loss. After recovery, fix the capacity or allocation root cause and monitor unassigned shard age.

### Q69. What causes unassigned shards?

Causes include disk watermarks, failed nodes, allocation constraints, awareness rules, shard limits, and insufficient distinct nodes for replicas. Allocation explain reports which deciders block a shard. Immediate fixes may be restoring a node, adding capacity, freeing disk safely, or correcting an accidental rule.

If only replicas are unassigned, primaries may still serve traffic but redundancy is reduced. Don't force-allocate as a first response; identify whether a valid copy exists and the data-loss consequences.

### Q70. How do disk watermarks and read-only blocks happen?

When disk usage crosses configured watermarks, Elasticsearch can stop allocation and may apply a read-only-allow-delete block. Exact behavior varies by release. Inspect filesystem use, shard distribution, retained indices, old logs, snapshots and reindex temporary space. Free/add disk safely, fix retention/rollover, and verify whether the block clears automatically or requires an explicit unblock in that version.

Alert before watermarks and reserve space for merges and recovery. Deleting data without an approved retention decision can turn a disk incident into data loss.

### Q71. What are circuit breakers?

Circuit breakers estimate or limit memory-intensive operations to reduce the risk of JVM OOM. An exception may indicate high-cardinality aggregation, request size, fielddata, script usage, or concurrency pressure. Check breaker stats, post-GC heap, requests, and mappings before raising limits.

Prefer reducing bucket count/concurrency, fixing fielddata misuse, and bounding expensive requests. Raising a breaker without understanding memory demand can turn a controlled request rejection into node instability.

### Q72. How do you investigate slow searches/timeouts?

Compare client timing with server-reported took, timeout status, shard failures, queue rejection, node saturation, and slow logs. Profile a representative query, then reduce broad ranges, wildcard expansions, aggregation buckets, response fields, and unnecessary exact total counts.

Scale only when metrics indicate capacity pressure. Client retries need deadlines, budgets, and jitter; otherwise a timeout storm can amplify overload. Clarify whether partial results are acceptable to the API.

### Q73. How do slow logs, Explain, and Profile differ?

Slow logs find recurring slow query/fetch work at shard level. Explain shows why a document matched and how its score was built. Profile details query execution components but adds overhead. Use each for the relevant question, with cluster metrics to identify queueing or hardware issues.

### Q74. How should mapping exceptions and bulk failures be handled?

Mapping exceptions usually mean data violates the mapped type, dynamic mapping policy, date format, or ingest pipeline. Fix the producer/schema contract rather than silently converting a numeric field into text. If the type is wrong, create a versioned index and reindex.

Bulk responses have per-item status. Retry transient failures with bounded backoff and send permanent invalid records to a DLQ. Monitor item failure rates, not just HTTP status, and reconcile expected versus indexed document IDs.

### Q75. How do you manage slow shard recovery?

Monitor recovery progress/bytes, source and target nodes, disk I/O, network, concurrent recoveries, search p99, and disk headroom. A large shard, overloaded source, slow network, or excessive concurrent recovery can be responsible. Increasing recovery concurrency blindly can hurt production search; restore redundancy without starving workload and retain enough capacity for another failure.

---

## 13. Java and Spring Boot

### Q76. Which client and versions should a new Java application use?

Use the official Elasticsearch Java API Client for typed API access. Current client 9.x documentation requires Java 17+ and uses the built-in REST 5 transport by default; the older REST client is an optional legacy path. Match the client/server major versions and check the exact compatibility matrix. Spring Data Elasticsearch is bound to its release train and Elasticsearch baseline, so do not upgrade it independently of Spring Boot and the server without compatibility tests.

### Q77. Show a minimal official Java API Client example.

The example assumes current Java API Client 9.x style and an Elastic Cloud endpoint/API key. Keep credentials in a secret manager or environment configuration, not source code.

~~~java
import co.elastic.clients.elasticsearch.ElasticsearchClient;

public final class SearchClientExample {
    public static void main(String[] args) throws Exception {
        String serverUrl = System.getenv("ES_URL");
        String apiKey = System.getenv("ES_API_KEY");

        ElasticsearchClient client = ElasticsearchClient.of(builder -> builder
            .host(serverUrl)
            .apiKey(apiKey)
        );

        try {
            var health = client.cluster().health();
            System.out.println("Cluster status: " + health.status());
        } finally {
            client.close();
        }
    }
}
~~~

The client is thread-safe and normally application-scoped. In Spring, define a singleton bean and close it at shutdown. Self-managed TLS/basic authentication needs configuration appropriate to the exact transport/version.

### Q78. How do you index, get, and search Java objects?

~~~java
import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch.core.SearchResponse;

public class ProductSearchService {
    private final ElasticsearchClient client;

    public ProductSearchService(ElasticsearchClient client) {
        this.client = client;
    }

    public void index(Product product) throws java.io.IOException {
        client.index(request -> request
            .index("products-v1")
            .id(product.productId())
            .document(product)
        );
    }

    public Product getById(String id) throws java.io.IOException {
        var response = client.get(request -> request
                .index("products-v1")
                .id(id),
            Product.class
        );
        return response.found() ? response.source() : null;
    }

    public SearchResponse<Product> search(String text) throws java.io.IOException {
        return client.search(request -> request
                .index("products-v1")
                .query(query -> query.match(match -> match
                    .field("name")
                    .query(text)
                )),
            Product.class
        );
    }
}
~~~

This example assumes Product is a Java record, so product.productId() is a record accessor, and that the configured JSON mapper can serialize and deserialize it. If your application uses a JavaBean-style POJO, adapt the accessor and constructor configuration accordingly. This is representative code; compile against the exact declared client version and add domain error handling, validation, metrics, and tests. Do not create a new client per request.

### Q79. How should Java bulk indexing work?

Build a bounded batch with deterministic IDs and official BulkRequest/BulkOperation builders from the selected client release. Parse each item result. A bulk worker should convert events, submit operations, classify item failures, acknowledge source records only after the required outcome is handled, retry transient failures with backoff, and route permanent mapping errors to a DLQ.

If the process crashes after indexing but before the Kafka offset commits, replay is expected. Stable IDs make replacement operations idempotent; source versions may be needed to prevent stale events from overwriting newer state. Measure batch bytes, item count, failures, retries, and indexing lag.

### Q80. How should an application build search filters and pagination?

Validate user input and build Query DSL in the backend. Derive tenant identity from authenticated context, allowlist sort fields, cap page size, and avoid accepting arbitrary raw DSL. For deep paging, return an opaque cursor containing validated query/sort scope and PIT state rather than trusting caller-supplied sort values.

Handle timeouts and shard failures explicitly. If partial results are unacceptable, do not return them as a normal complete response. Cursors should be scoped to the authenticated tenant and query and should expire.

### Q81. What are good error-handling and integration-testing practices?

Classify mapping/client errors, authorization errors, version conflicts, transient overload, network failures, and unavailable-cluster cases. Retry only safe transient failures with bounded exponential backoff and jitter. A timeout leaves the outcome uncertain, so idempotency matters.

Unit-test query builder rules, tenant scope, and validation. Integration-test mappings, analyzers, nested queries, aggregations, serialization, and version behavior against a real compatible Elasticsearch instance; mocks cannot prove that a query is valid for the intended mapping.

---

## 14. SQL, Kafka, and microservices

### Q82. Why is dual-writing SQL and Elasticsearch unsafe?

If SQL commits but Elasticsearch fails, the stores diverge. If Elasticsearch is updated but SQL rolls back, search exposes a non-authoritative change. A normal local transaction cannot atomically commit across both systems.

Prefer SQL as the source of truth and Elasticsearch as an asynchronous projection. Write the business change and outbox event in one SQL transaction, then publish reliably and consume idempotently. CDC can also stream committed changes. This creates eventual consistency that can be replayed and reconciled.

### Q83. Explain transactional outbox and CDC-to-Kafka-to-Elasticsearch.

The outbox writes the event in the same SQL transaction as the business update. A relay/CDC connector publishes it to Kafka; a consumer updates Elasticsearch using stable IDs and version rules.

~~~mermaid
flowchart LR
    A[Application] --> B[SQL transaction]
    B --> C[Business row]
    B --> D[Outbox]
    D --> E[CDC or relay]
    E --> F[Kafka]
    F --> G[Idempotent indexer]
    G --> H[Elasticsearch projection]
    G --> I[Retry and DLQ]
    J[Reconciliation] --> C
    J --> H
~~~

Outbox addresses the lost-event window between SQL commit and publication. Duplicate delivery, ordering, schema evolution, lag, deletes, and DLQ replay still require explicit design.

### Q84. How do consumers keep Elasticsearch projections correct?

Choose Kafka keys to preserve per-entity ordering when required. Use stable document IDs, event IDs, source versions, and explicit upsert/delete semantics. Inspect individual bulk results before committing offsets according to the delivery contract. Replays since the last offset commit are normal; idempotency makes them safe.

Monitor lag, indexing delay, retry count, failed items, DLQ volume, and reconciliation. Delta events may not be idempotent by themselves; full-state projection events or version checks are often easier to recover.

### Q85. How should deletes, schema evolution, and reconciliation work?

Treat deletes as events. If a delete is lost, a stale search document can remain indefinitely. Preserve entity keys and versions, replay deletes idempotently, and reconcile extra/missing IDs against the authoritative database. Rate-limit repair jobs.

Use versioned event schemas and explicit mapping migrations. A field type change may require a new index. Reconciliation should be observable, resumable, and safe to repeat.

---

## 15. Security and senior-level decisions

### Q86. What security controls should protect Elasticsearch?

Use TLS, authentication, least-privilege roles, secret management, restricted network access, and protected administrative APIs. Keep app identities separate from operators and migration jobs. Validate security features against the exact distribution, deployment model, and license. Do not expose an unprotected cluster to the public internet or embed privileged credentials in code.

Search validation is also availability protection: limit input length, page size, time range, aggregation buckets, allowed fields, and expensive query constructs.

### Q87. How do you prevent cross-tenant data leakage?

Derive the tenant from the authenticated principal and inject the tenant filter in a server-controlled query builder. Apply policy to search hits, direct ID lookups, aggregations, suggestions, exports, background jobs, and cursors. A client must not be able to replace or bypass the scope with arbitrary DSL.

Routing is not authorization. Depending on the threat model and number of tenants, consider separate indices/clusters or supported document-level security. Test cross-tenant negative cases and audit access without logging sensitive payloads unnecessarily.

### Q88. When are nested, parent-child, runtime fields, vector search, and cross-cluster search appropriate?

Nested is suitable for bounded child arrays whose values must match together, but increases Lucene document count and query cost. Parent-child allows independently updated documents but introduces routing and join cost. Denormalize until update patterns justify complexity. Runtime fields can ease migration or infrequent calculations, but repeated computation may be slower than indexing a field.

Vector/hybrid search can improve semantic retrieval but adds embedding, ranking, recall/latency, and evaluation concerns. Cross-cluster search can query multiple clusters but introduces network and partial-failure considerations. Keep a lexical baseline and measure whether advanced features solve a real user problem.

### Q89. Explain snapshots, RPO/RTO, and multi-zone design.

Snapshots are backups and must be restored in drills. Replicas are not backups: logical deletion or corruption may propagate. Define recovery point objective (acceptable data loss) and recovery time objective before choosing snapshot cadence, retention, replicas, and restore procedures. Keep repository access secure.

Multi-zone placement helps only if replicas span separate failure domains and remaining nodes have enough capacity after a zone fails. Multi-region designs also add latency, bandwidth, and cost. Clarify whether the goal is disaster recovery, read locality, active-active serving, or isolation.

### Q90. When should Elasticsearch be avoided or reconsidered?

If the workload is only exact primary-key lookup, transactional updates, joins, and constraints, a relational database may already be the simpler fit. Search adds operational burden and eventual consistency. If already deployed, optimize mapping/query shapes, retention, shard count, and projection ownership before planning replacement. A SQL LIKE query may not reproduce analyzers, ranking, facets, and autocomplete.

---

## 16. Frequently asked questions

### Q91. Full-text search versus exact matching versus filtering?

Full-text search operates on analyzed terms and often scores results. Exact matching typically targets keyword values. Filtering determines eligibility without relevance scoring. A product query normally combines all three: text relevance plus tenant/stock/price filters and optional rating boosts. The main mistake is not matching query intent to mapping.

### Q92. Why can some fields not be sorted or aggregated?

Text fields are analyzed and generally lack doc values by default. Keyword/numeric fields support these operations efficiently. Use a keyword multi-field for sorting/facets, and avoid enabling text fielddata without a measured need because of heap cost.

### Q93. Why are results stale after a successful write?

The document may not yet be visible to search because refresh has not happened, or the upstream database/Kafka projection has not indexed it yet. Trace database commit, event publication, consumer lag, indexing success, and refresh separately. Don't treat refresh as a substitute for a healthy ingestion pipeline.

### Q94. How do you introduce a new field or analyzer?

Update mapping/template and producers in a compatible sequence. Templates affect future indices; incompatible type/analyzer changes often need a new index and reindex. Test Analyze output, queries, aggregations, missing values, and old event payloads before cutover.

### Q95. How does shard fan-out affect latency?

Queries may execute on many shard copies, then reduce at a coordinator. More shards can improve parallelism but increase coordination and overhead. One slow shard or node can dominate p99. Diagnose per-shard load and query complexity before adding shards or replicas.

### Q96. How do you make autocomplete both fast and relevant?

Choose completion suggester, search_as_you_type, edge n-grams, or phrase-prefix based on the UX. Limit fields/results, debounce requests, avoid exact total counts, and handle typos selectively. Evaluate latency and relevance on real partial inputs.

### Q97. When should analytics move outside Elasticsearch?

Elasticsearch aggregations fit interactive facets and bounded dashboards. Exhaustive high-cardinality or long-window analytics may fit composite aggregation, pre-aggregated indices, or a dedicated analytical system better. Isolate expensive dashboards from critical search workloads and enforce bucket/query budgets.

### Q98. What if indexing times out with an unknown outcome?

Treat it as uncertain. Retry with stable IDs and source-version rules; inspect bulk item status; use bounded backoff. Network failure after server-side success is possible. Make replay safe rather than assuming every timeout means no write occurred.

### Q99. How do you validate a search design?

Use production-like text/cardinality/missing values and nested data; test relevance, filters, aggregations, pagination, tenant isolation, and failure recovery. Benchmark mixed indexing/search at realistic scale and test replay, duplicate events, partial bulk failures, alias migrations, and snapshot restore.

### Q100. What makes a senior-level Elasticsearch answer?

Connect the feature to workload and correctness, explain cost and failure modes, name metrics/APIs to prove behavior, and describe rollout/rollback. For queries, state mapping assumptions and score/filter intent. For architecture, explain source of truth, idempotency, replay, freshness, and reconciliation. Diagnose before scaling.

---

## 17. Scenario bank

Each scenario should be answered with clarifying questions, diagnosis, step-by-step mitigation, prevention, and interviewer follow-ups. The short structured answers below provide a model to expand aloud.

### Q101. Product search returns irrelevant results.
Clarify query examples and expected ranking. Inspect mapping, analyzer output, operator, boosts, and relevance judgements. Narrow broad OR matching, boost name/phrase matches, keep eligibility predicates in filters, and use Explain on bad hits. Build a regression query set and roll ranking changes back if quality falls. Follow-up: how do you evaluate top-k relevance?



**Concrete example:** Run `POST /products-v1/_search` with `{"query":{"multi_match":{"query":"wireless headphones","fields":["name^4","description"]}}}` and compare the top hits against a judged query set.
### Q102. A term query returns no product-name hits.
Confirm whether the field is text or keyword. term does not analyze input like match; a whole phrase may not be one indexed token. Inspect mapping, source document, and Analyze output. Use match for text and keyword for exact values; reindex if required indexing representation is missing. Follow-up: why use multi-fields?



**Concrete example:** Compare `{"query":{"match":{"name":"Wireless Bluetooth Headphones"}}}` with a `term` query on `name.keyword`; inspect `_mapping` before deciding which field matches the intent.
### Q103. Staging works but production does not.
Compare mappings, settings, templates, aliases, sample docs, analyzers, and client/server versions. Dynamic mapping or a missing template often causes divergence. Reproduce with production-equivalent index definitions; correct template/migration and rebuild an index if the type is incompatible. Follow-up: what would you compare first under pressure?



**Concrete example:** Compare `GET /products-v1/_mapping`, `GET /_index_template/products-template`, and `_analyze` output in both environments before changing production data.
### Q104. Query latency rises with data volume.
Determine whether the problem is isolated or cluster-wide. Inspect slow logs, Profile, per-shard metrics, heap, queue rejections, aggregation cardinality, and query range. Bound expensive clauses and response sizes; change topology only when telemetry supports it. Follow-up: why can more shards worsen latency?



**Concrete example:** Use `POST /products-v1/_search` with `{"profile":true,"query":{"match":{"name":"headphones"}}}` for a targeted profile, and correlate it with slow logs and node stats.
### Q105. Cluster turns yellow after a node loss.
Check whether primaries remain allocated, then inspect allocation explain, disk, capacity, and awareness rules. Restore the node or add capacity and recover replica placement. Yellow means redundancy is reduced, not necessarily that primaries are unavailable. Follow-up: why can replica and primary not share a node?



**Concrete example:** Ask `POST /_cluster/allocation/explain` with `{"index":"products-v1","shard":0,"primary":false}` for a replica that is not allocating; select the real unassigned shard from cluster state.
### Q106. Cluster turns red and results are missing.
Identify unassigned primaries and affected indices. Inspect allocation explanation, node logs, recovery and snapshots. Recover a valid shard copy or restore/reconcile from backup/source. Never force a stale primary without understanding possible data loss. Follow-up: when should an API reject partial results?



**Concrete example:** Start with `GET /_cat/shards?v` and `POST /_cluster/allocation/explain` for the actual unassigned primary, then verify recoverable copies and snapshot metadata.
### Q107. One shard is much hotter than others.
Compare per-shard traffic, size, routing, tenant skew, and hot threads. Custom routing may concentrate a large tenant. Consider routing partitions, a separate tenant model, or reindexing into a better topology. Add skewed-load tests and per-shard alerts. Follow-up: routing performance versus distribution trade-off?



**Concrete example:** Compare `GET /_cat/shards/products-v1?v` with `GET /_nodes/hot_threads`; check whether one routing key or tenant generates disproportionate work.
### Q108. Bulk import throughput drops.
Inspect item failures, rejections, heap/GC, disk writes, merges, refreshes, and concurrency. Use bounded batches and tune concurrency gradually; back off on rejections. During controlled offline loads, a longer refresh interval may help. Follow-up: why can bigger batches be slower?



**Concrete example:** Monitor `GET /_nodes/stats/thread_pool,indices,fs` and inspect each item in the Bulk response; reduce batch size or concurrency if rejection and disk/merge pressure rise.
### Q109. Search endpoint times out under load.
Correlate application/client timing with server took, shard failures, queue rejections, CPU/GC and slow logs. Simplify expensive queries and apply backpressure before scaling. Retry transient errors with jitter and a request deadline. Follow-up: why not just raise all timeouts?



**Concrete example:** Use a bounded request such as `{"timeout":"2s","track_total_hits":false,"query":{"bool":{"must":[{"match":{"name":"headphones"}}],"filter":[{"range":{"createdAt":{"gte":"now-1d","lt":"now"}}}]}}}` and correlate timeout/shard metadata with queue metrics.
### Q110. Deep pagination consumes resources.
Determine whether deep page numbers are truly needed. from/size requires collecting skipped hits. Use search_after and PIT with deterministic sorting, expose a scoped cursor, and avoid exact hit counts unless needed. Follow-up: PIT versus scroll?



**Concrete example:** Return a cursor based on the last hit sort array and issue the next `search_after` request with the same query and sort; use PIT when the view must stay stable.
### Q111. Autocomplete needs partial words and typo tolerance.
Clarify prefix/phrase/entity suggestion needs. Compare search_as_you_type, edge n-grams, completion suggester, and phrase-prefix. Restrict fields and count, debounce, and use fuzzy matching selectively. Test both p99 and relevance. Follow-up: what does edge n-gram cost?



**Concrete example:** Prototype `{"query":{"match_phrase_prefix":{"name":{"query":"wireless blue","max_expansions":20}}}}`; compare it with a mapped `search_as_you_type` field under realistic QPS.
### Q112. Price filters return unexpected products.
Confirm type, units, currency, scaling factor, and inclusive boundaries. Inspect source and mapping; mixed currencies or strings are common causes. Normalize to one numeric representation and reindex if type is wrong. Test boundary values. Follow-up: scaled_float versus minor-unit integers?



**Concrete example:** Check the exact numeric mapping and test `{"query":{"range":{"price":{"gte":1000,"lte":6000}}}}` against documents on both boundaries and in the expected currency.
### Q113. Variant filters cross-match colors and sizes.
Confirm the predicates must apply to one variant. An ordinary object array can flatten values and cross-match. Use nested mapping and a single nested query; if mapping is already object, create a new index and reindex. Follow-up: why isn't nested always preferable?



**Concrete example:** Use one `nested` query at path `variants` with both `variants.color=red` and `variants.size=large` in its inner bool filter so both conditions must match the same child.
### Q114. Aggregations create too many buckets.
Find field cardinality and the requested number of buckets. Reduce size; use top terms for facets and composite aggregation for exhaustive pagination. Use keyword/doc values and cap nested buckets. Follow-up: why can terms counts be approximate?



**Concrete example:** For exhaustive facets, use a composite aggregation with `size` and continue from the returned `after_key` rather than raising a terms bucket limit without bounds.
### Q115. Heap spikes and breakers trip.
Correlate post-GC heap with large aggregations, fielddata, request size, concurrency, or scripts. Fix fielddata misuse, cap buckets/concurrent queries, and isolate batch work. Don't raise breaker limits blindly. Follow-up: why does filesystem cache matter too?



**Concrete example:** Inspect `GET /_nodes/stats/jvm,breaker,indices` before changing breaker settings; identify which request and aggregation caused the memory estimate to cross its limit.
### Q116. Disk watermarks block writes.
Find full nodes and inspect indices, retention, reindex temporary copies, snapshots and open-deleted files. Add capacity or safely remove approved expired data, fix rollover/retention, and verify block state per version. Follow-up: why can reindex exhaust disk?



**Concrete example:** Use `GET /_cat/allocation?v` and inspect effective cluster/index disk watermark settings; remove only data approved by retention policy and leave recovery headroom.
### Q117. SQL changes but search is stale.
Trace the transaction, outbox/CDC relay, Kafka lag, consumer logs, bulk item results, and refresh. Fix the failing stage, replay idempotently, and reconcile. Serve immediate read-after-write from the authoritative source when needed. Follow-up: why is dual-write insufficient?



**Concrete example:** Trace one entity ID through outbox row, Kafka offset/lag, consumer result, and `GET /products-v1/_doc/P1001`; compare the indexed source version with the database version.
### Q118. Kafka restarts and replays events.
Determine whether events are full-state or deltas and whether ordering matters. Use stable IDs, event/source versions and idempotent projection updates; inspect item failures before offset commit. Monitor retries and lag. Follow-up: why doesn't Kafka exactly-once cover Elasticsearch side effects?



**Concrete example:** Index a full-state projection using a stable ID, for example `PUT /products-v1/_doc/P1001` with `{"productId":"P1001","sourceVersion":42,"name":"Wireless headphones"}`; protect it from older source versions.
### Q119. Field type changes without downtime.
Create a versioned index, capture a durable change-stream position, backfill, replay newer changes, validate, and swap aliases. Preserve rollback and account for deletes. Follow-up: how do you prevent old events overwriting new state?



**Concrete example:** Create `products-v2` with the corrected mapping, start a controlled `POST /_reindex` backfill, then replay changes after the captured source checkpoint before cutover.
### Q120. New index rollout must be safe.
Use versioned templates, build the destination, verify mappings/settings, backfill and replay changes, compare queries, then atomically move aliases. Keep a designated write index and test rollback. Follow-up: what if alias cutover happens before catch-up?



**Concrete example:** Use `POST /_aliases` with one `remove` action for the old index and one `add` action for the new index in the same request; confirm catch-up and designate the write index.
### Q121. Tenant can see other tenants' products.
Check authenticated identity, central query builder, direct-ID paths, aggregations, cursors, suggestions, and exports. Inject tenant scope server-side; consider document-level security or per-tenant indices. Routing is not authorization. Follow-up: how do you test for leakage?



**Concrete example:** Build the tenant scope server-side, e.g. `{"query":{"bool":{"must":[{"match":{"name":"headphones"}}],"filter":[{"term":{"tenantId":"tenant-42"}}]}}}`; derive tenant-42 from authenticated identity.
### Q122. Reindex while writes continue.
A reindex alone misses or races concurrent mutations. Capture a consistent snapshot/change position, backfill, replay mutations with idempotent version-aware writes, catch up and validate, then cut over. Keep the migration resumable and budget disk. Follow-up: how do you handle deletes?



**Concrete example:** Record a durable CDC/Kafka offset, backfill a snapshot into v2, replay later events with source-version checks, and cut over only after lag reaches the agreed threshold.
### Q123. Delete event was lost.
Trace source commit, outbox/CDC, Kafka, indexing consumer and DLQ. Replay delete idempotently and reconcile extra indexed IDs. Preserve version/tombstone semantics so a stale upsert cannot resurrect the document. Follow-up: how long should tombstones be retained?



**Concrete example:** Replay a delete idempotently with `POST /products-v1/_delete_by_query` and `{"query":{"ids":{"values":["P1001"]}}}` only if that deletion scope is intended; otherwise prefer a direct delete by known ID.
### Q124. Bulk HTTP 200 but some documents are missing.
Inspect the response errors flag and each item. Retry transient failures only; send permanent schema/mapping errors to DLQ and repair/replay. Alert on item failures, not only HTTP errors. Follow-up: why is whole-batch retry unsafe?



**Concrete example:** For `POST /_bulk`, inspect both the top-level `errors` flag and every `items[]` status/error; retry only transient failures and send permanent mapping errors to a DLQ.
### Q125. Balance text, rating and recency.
Clarify the ordering objective. Keep text as base score, enforce hard requirements in filters, and use calibrated field-value/decay functions for business signals. Test missing/outlier values and version the ranking rules. Follow-up: can scores from separate queries be compared?



**Concrete example:** Compare a text query plus `function_score`/`field_value_factor` against a text-only baseline, and verify how missing ratings and outliers change the ranking.
### Q126. There are thousands of tiny shards.
Inspect index count, shard size, rollover policy, retention, and per-shard overhead. Use size/count thresholds, consolidate only where supported, and plan safe reindex/topology changes. Follow-up: what happens if shards are too large?



**Concrete example:** Use `GET /_cat/shards?v&h=index,shard,prirep,state,docs,store,node` to find tiny shards, then review rollover policy before planning any consolidation.
### Q127. Leading wildcard is slow.
Clarify whether true substring search is needed. Broad wildcard/regexp patterns can expand many terms. Consider edge n-grams, search-as-you-type, completion, or a suitable wildcard field. Bound and rate-limit user patterns. Follow-up: can prefix queries also be expensive?



**Concrete example:** Replace a leading-wildcard request with a suitable `prefix` query or an autocomplete mapping; compare `{"query":{"prefix":{"productId":"P10"}}}` with the current worst-case pattern.
### Q128. Deployment introduces mapping exceptions.
Compare source payloads, mappings, templates, date formats and ingest pipelines. Roll back or fix producer data, create a new index if type is incompatible, then replay failures. Add schema contract tests and canary ingestion. Follow-up: strict versus permissive dynamic mapping?



**Concrete example:** Reproduce the rejected document against `GET /products-v1/_mapping`, inspect the per-item `mapper_parsing_exception`, fix producer/schema, then replay failed records after validation.
### Q129. More replicas lower indexing throughput.
Each replica requires storage, indexing, network and recovery work. Measure CPU, disk, network and recovery. Choose replicas from read demand and failure tolerance, not by rule of thumb. Follow-up: what does a replica help, and what does it not help?



**Concrete example:** Before `PUT /products-v1/_settings` with `{"index":{"number_of_replicas":2}}`, calculate disk and surviving-node headroom and benchmark indexing throughput; do not apply the setting just to add redundancy blindly.
### Q130. Node fails during shard recovery.
Inspect primary/replica availability, recovery state, quorum, disk and remaining capacity. Restore infrastructure and preserve valid copies; avoid data-loss commands. Throttle/prioritize recovery carefully. Follow-up: fast recovery versus keeping current search p99?



**Concrete example:** Watch `GET /_cat/recovery?v&active_only=true`, `GET /_cluster/health`, and disk/network stats; prioritize valid shard recovery without starving production search.
### Q131. Consumer retries a poison-pill document forever.
A deterministic mapping/schema error will not fix itself. Bound retries, capture event/error/version, send permanent failures to a DLQ, alert and replay after correction. Consider partition ordering consequences. Follow-up: what happens to later events if one is skipped?



**Concrete example:** Store a DLQ envelope containing `eventId`, `entityId`, `schemaVersion`, original payload, attempt count, and `mapper_parsing_exception` detail, then replay only after the contract is corrected.
### Q132. Duplicate documents appear after replay.
Check whether retries use random IDs or stable entity IDs. Use deterministic document IDs and version-aware writes, then reconcile duplicates. Follow-up: why doesn't idempotent upsert prevent stale events?



**Concrete example:** Use the same deterministic identifier on every replay, such as `PUT /products-v1/_doc/P1001`; add source-version checks if an older event could overwrite a newer projection.
### Q133. GET by ID succeeds, but search misses the document.
Separate upstream indexing delay from refresh visibility. Real-time GET can see recent writes before search. Trace the ingestion pipeline; if needed use the documented refresh policy sparingly or return the source entity. Follow-up: why doesn't flush solve it?



**Concrete example:** Compare `GET /products-v1/_doc/P1001` with `POST /products-v1/_search`; if the GET is current but search lags, inspect refresh and use `refresh=wait_for` only when the latency cost is justified.
### Q134. CPU is high but average latency looks fine.
Compare p50/p95/p99, per-node CPU, hot threads, merges/recovery, request rate, queue rejections, and shard skew. Average latency hides tail outliers. Isolate background and expensive workloads and alert on percentiles. Follow-up: what metrics find a hot shard?



**Concrete example:** Correlate p95/p99 with `GET /_nodes/hot_threads`, `GET /_nodes/stats/os,process,thread_pool,indices`, and merge/recovery activity rather than relying on cluster-average CPU.
### Q135. Scores change after more documents are indexed.
BM25 depends on corpus term/document frequencies and field lengths. Inspect Explain and representative relevance results; scores are not fixed quality labels. Use a judged query set for regression tests. Follow-up: why are scores across separate queries not comparable?



**Concrete example:** Run `GET /products-v1/_explain/P1001` with the exact search query to see why the document matches and how score contributions are combined; compare after data/index changes.
### Q136. A large tenant dominates traffic.
Determine whether routing concentrates that tenant on one shard or filters fan out broadly. Consider routing partitions, separate large-tenant indices, or a different model. Keep authorization separate and test skew. Follow-up: when is tenant routing harmful?



**Concrete example:** A routed request such as `POST /products-v1/_search?routing=tenant-42` may narrow shard fan-out, but keep the tenant filter in the query and derive both values server-side.
### Q137. Restore after accidental deletion.
Replicas reflect logical deletes, so they are not backups. Find a compatible snapshot, restore to a separate index when possible, validate, reconcile writes since the snapshot, then route aliases. Follow-up: why shouldn't a live index be replaced casually with an older snapshot?



**Concrete example:** Restore `POST /_snapshot/backup-repo/snapshot-name/_restore` to a separate index name first, validate records, and reconcile authoritative writes that occurred after the snapshot.
### Q138. User can request arbitrary sorts and page sizes.
This allows costly field sorts, deep offsets, expensive aggregations and resource abuse. Allowlist fields and query shapes, cap size/ranges/aggregation budgets, enforce tenant scope, and rate-limit. Follow-up: how is input validation availability engineering?



**Concrete example:** In the API layer, allowlist sort names such as `Set.of("price", "rating", "createdAt")`, cap page size and range, and reject arbitrary raw Query DSL.
### Q139. Java client upgrade causes failures.
Check server/client compatibility, dependency tree, transport, auth/TLS, serialization, and generated API changes. Read the release/migration notes, pin compatible versions, and run real integration tests for CRUD/search/bulk. Follow-up: why does forward compatibility not mean future features work automatically?



**Concrete example:** Check dependencies with `./mvnw dependency:tree -Dincludes=co.elastic.clients:elasticsearch-java`, then compile and run real-cluster integration tests against the intended server major.
### Q140. Design resilient search using SQL, Kafka and Elasticsearch.
SQL is authoritative. Outbox/CDC publishes durable changes to Kafka; an idempotent, version-aware consumer updates a versioned Elasticsearch index; aliases serve the search API; retries, DLQ, reconciliation and snapshots handle failures. Define freshness, tenant isolation, latency and recovery objectives. Critical purchase decisions recheck SQL. Follow-up: how do you migrate mappings without downtime and prevent stale replay?

**Concrete example:** Project with a deterministic ID and source version, e.g. `PUT /products-v1/_doc/P1001` with `{"productId":"P1001","sourceVersion":42,"name":"Wireless headphones"}`; a replayed older version must not replace version 42.


---

## 18. Mock interviews and revision

### Mock round A — Fundamentals
1. Inverted index versus B-tree.
2. Primaries and replicas.
3. Green/yellow/red health.
4. Text versus keyword; match versus term.
5. Refresh versus flush versus durability.
6. Shard fan-out and result reduction.
7. Object versus nested.
8. BM25 and relevance debugging.
9. Why Elasticsearch is not a transactional source of truth.
10. Alias-based migration.

### Mock round B — Query DSL practical
1. Search name and description with different boosts.
2. Filter stock, tenant, price and category.
3. Exclude discontinued products.
4. Require a phrase.
5. Enforce minimum matching terms.
6. Query one nested variant's attributes.
7. Build brand/price facets.
8. Top products by category.
9. PIT plus search_after.
10. Diagnose zero results step by step.

### Mock round C — Architecture
1. Draw indexing lifecycle and search fan-out.
2. Plan shard placement across zones.
3. Design reindex while writes continue.
4. Design SQL/outbox/Kafka/Elasticsearch sync.
5. Estimate storage for 50 million documents.
6. Plan log retention and rollover.
7. Explain routing and hot-shard risk.
8. Plan recovery after a node/zone failure.

### Mock round D — Troubleshooting
1. Red cluster.
2. Search timeout with low cluster-average CPU.
3. Circuit breaker from aggregation.
4. Bulk partial failures.
5. Disk watermark block after reindex.
6. Mapping error after deployment.
7. Stale projection after SQL update.
8. Slow shard recovery.

### Mock round E — Senior trade-offs
1. Should primary-key lookups use Elasticsearch?
2. How to balance relevance, rating, popularity and freshness?
3. How to isolate a large tenant?
4. Recover from a lost delete event.
5. Reindex without losing concurrent writes.
6. Define read-after-write.
7. Choose nested, parent-child, or denormalization.
8. Isolate analytics from critical search.
9. Plan regional recovery.
10. Choose between shards, replicas, memory and CPU.

### Top 50 must-know questions

1. Q1 Elasticsearch fundamentals
2. Q2 Elasticsearch versus databases
3. Q7 primary/replica shards
4. Q9 search fan-out
5. Q10 cluster health
6. Q12 routing
7. Q13 indexing lifecycle
8. Q14 inverted index
9. Q15 refresh
10. Q16 translog/flush
11. Q18 bulk indexing
12. Q19 mappings
13. Q20 text versus keyword
14. Q21 analyzers
15. Q24 nested versus object
16. Q25 safe mapping migration
17. Q27 Query DSL
18. Q28 match versus term
19. Q29 phrase queries
20. Q30 term/terms/range/exists/ids
21. Q31 prefix/wildcard/regexp
22. Q32 fuzzy
23. Q33 multi_match and boosts
24. Q35 bool semantics
25. Q36 minimum_should_match
26. Q37 query/filter and caching
27. Q38 constant_score/boosting/dis_max
28. Q39 function/script score
29. Q40 BM25
30. Q42 autocomplete
31. Q44 Explain/Profile
32. Q45 pagination strategies
33. Q46 deterministic sort
34. Q47 aggregations
35. Q50 aggregation accuracy
36. Q52 composite buckets
37. Q54 reindex
38. Q55 ILM/data streams
39. Q58 consistency/visibility
40. Q59 optimistic concurrency
41. Q61 idempotency/retries
42. Q62 shard sizing
43. Q64 heap/filesystem cache
44. Q68 health troubleshooting
45. Q71 circuit breakers
46. Q76 client compatibility
47. Q79 Java bulk
48. Q82 dual-write risk
49. Q83 outbox/CDC
50. Q87 tenant isolation

### Top 25 Query DSL patterns

1. match
2. term
3. match_phrase
4. match_phrase_prefix
5. terms
6. range
7. exists / missing
8. bool must/filter/must_not
9. should and minimum_should_match
10. multi_match and boosts
11. query_string versus simple_query_string
12. fuzzy with bounded expansions
13. prefix
14. wildcard risks
15. constant_score
16. dis_max
17. function_score
18. decay functions
19. nested
20. source filtering and highlighting
21. terms/range/date_histogram aggregations
22. top_hits
23. search_after
24. PIT plus search_after
25. Explain and Profile

### Common mistakes

- Treating Elasticsearch as the transactional source of truth.
- Using term on analyzed text for a phrase.
- Assuming should is always mandatory.
- Putting structured constraints in scoring context without a reason.
- Assuming every filter is cached.
- Assuming a successful write means immediate search visibility.
- Confusing refresh, flush, and durability.
- Sorting by a field while expecting relevance to remain primary.
- Using deep from/size.
- Using search_after without stable sort values and a tie-breaker.
- Querying nested fields without nested mapping/query.
- Enabling text fielddata rather than using a keyword field.
- Returning unbounded buckets or exact hit counts for every request.
- Blindly increasing heap, breaker limits, timeouts, replicas, or shards.
- Treating Bulk HTTP success as item-level success.
- Retrying permanent mapping errors forever.
- Dual-writing SQL and Elasticsearch without recoverable synchronization.
- Assuming Kafka exactly-once automatically covers Elasticsearch.
- Treating replicas as backups.
- Trusting caller-provided tenant filters.
- Assuming templates update existing indices.
- Reindexing during active writes without change-stream replay.
- Comparing scores across unrelated queries.
- Assuming Elasticsearch/OpenSearch and license-dependent features are identical.
- Copying Java APIs without checking the matching version.

### Last-minute revision checklist

- [ ] Inverted index, Lucene segments, analyzers, mappings and doc values.
- [ ] Primaries, replicas, routing, coordinating nodes and fan-out.
- [ ] Green/yellow/red, allocation explain and disk watermarks.
- [ ] Refresh versus flush versus translog durability.
- [ ] text/keyword, match/term, object/nested.
- [ ] Bool semantics, minimum_should_match, query/filter context, BM25.
- [ ] Write ten Query DSL patterns from memory and explain mappings.
- [ ] search_after/PIT, deterministic sort, aggregations and composite after_key.
- [ ] Bulk item failures, retries, idempotency and optimistic concurrency.
- [ ] Aliases, reindex/replay, schema migration and rollback.
- [ ] Java version compatibility and application-scoped client.
- [ ] SQL/outbox-or-CDC/Kafka/Elasticsearch consistency.
- [ ] Diagnose red, slow search, heap/breakers, hot shards and recovery.
- [ ] Tenant isolation, snapshots, restore and data-loss trade-offs.

---

## 19. Official references

Use the official documentation for the exact release you deploy; current documentation evolves.

- [Elasticsearch Guide](https://www.elastic.co/guide/)
- [Query DSL overview](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)
- [Search API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html)
- [Boolean query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-bool-query)
- [Nested query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html)
- [Paginate results, search_after and PIT](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/paginate-search-results)
- [Mappings](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html)
- [Analysis](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html)
- [Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)
- [Translog settings and durability](https://www.elastic.co/docs/reference/elasticsearch/index-settings/translog)
- [Index templates](https://www.elastic.co/docs/manage-data/data-store/templates)
- [Index lifecycle management](https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management)
- [ILM phases and actions](https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management/index-lifecycle)
- [Official Java API Client](https://www.elastic.co/docs/reference/elasticsearch/clients/java)
- [Java client getting started](https://www.elastic.co/docs/reference/elasticsearch/clients/java/getting-started)
- [Java client transport](https://www.elastic.co/docs/reference/elasticsearch/clients/java/transport)
- [Java client bulk operations](https://www.elastic.co/docs/reference/elasticsearch/clients/java/usage/indexing-bulk)
- [Spring Data Elasticsearch](https://docs.spring.io/spring-data/elasticsearch/reference/)
- [Spring Data Elasticsearch version matrix](https://docs.spring.io/spring-data/elasticsearch/reference/elasticsearch/versions.html)

Query snippets are interview/practice examples, not a claim of live execution. Validate syntax, mappings, security, and client calls against your deployed versions, then run integration tests and profile representative workloads before production use.
