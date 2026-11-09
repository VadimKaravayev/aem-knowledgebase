# Database data models: document, columnar, graph, vector

A data model is the shape a database stores data in, and that shape decides which questions are cheap to ask. The relational model (tables, rows, joins) is the baseline. Document, columnar, graph and vector databases each give up some of its generality to make one kind of access much faster or simpler.

The running example is a small shop: customers, products and orders, shown in each model.

## Relational: the baseline

Tables with fixed columns, linked by keys, queried with SQL and joins ([[database-normalization]]).

- **Strong at:** any query you didn't plan for, joins, transactions across many rows ([[database-transactions-isolation]]), constraints the database enforces.
- **Weak at:** very large write-heavy scale on one key, deeply nested or highly variable data, many-hop relationship queries, similarity search.
- **Engines:** PostgreSQL, MySQL, SQL Server, Oracle.

Every model below is measured against this.

## Document model

Data is stored as self-contained documents, usually JSON (BSON in MongoDB). One document holds an entity *together with* the data you usually read with it.

```json
{
  "_id": "order-1001",
  "customer": { "id": "C1", "name": "Customer A" },
  "status": "paid",
  "items": [
    { "product_id": "P1", "name": "Pen", "qty": 3, "price": 1.50 },
    { "product_id": "P2", "name": "Ink", "qty": 1, "price": 4.00 }
  ]
}
```

The whole order, with its lines, is read and written in one operation. The relational version needs three tables and two joins.

**Engines:** MongoDB, Couchbase, Firestore, Azure Cosmos DB; DynamoDB is key-value but often used the same way.

**Pros**

- Matches how applications think: one object in code, one document in the database.
- Flexible schema: documents in one collection can have different fields, so adding a field needs no migration.
- A write to one document is atomic, so embedding related data avoids multi-document transactions.
- Scales out well by sharding on a document key.

**Cons**

- **Joins are weak or missing** (MongoDB's `$lookup` exists but is no relational join). Data is often copied instead, like the product name in each order line, and keeping copies in sync is your job.
- "Schemaless" really means the schema lives in application code. Without validation, collections drift into many shapes.
- Multi-document transactions exist in MongoDB (4.0+) but cost more and are the exception, not the norm.
- Size limits (16 MB per MongoDB document) punish unbounded embedded arrays, such as all comments ever on a post.

**Good fit:** content and catalogs with varied attributes, user profiles and settings, event payloads, data that is read as one aggregate (an order, a cart, an article with its blocks).

**Poor fit:** data with many cross-links queried from many directions, reporting across entities, money-style invariants spanning many records.

## Columnar model: two different things share the name

"Columnar" covers two families that solve opposite problems. Mixing them up is a common design mistake.

### Wide-column stores (operational, write-heavy)

Data is grouped by a **partition key** and sorted inside the partition by **clustering columns**. Each table is designed for one query; you look rows up by key, not by arbitrary filters.

```sql
-- Cassandra CQL: one table answers one question, "orders of a customer, newest first"
CREATE TABLE orders_by_customer (
  customer_id text,
  order_date  timestamp,
  order_id    text,
  total       decimal,
  PRIMARY KEY ((customer_id), order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC);
```

Need "orders by product" too? That's a second table with the same data, written alongside.

**Engines:** Apache Cassandra, ScyllaDB, HBase, Google Bigtable.

**Pros**

- Very high write throughput and near-linear scale-out across many nodes and data centres.
- No single primary node; stays available when nodes fail, with consistency tunable per query.
- Predictable latency for key-based reads, even at huge size.

**Cons**

- **Query-first design:** you must know your queries before modelling. A new query often means a new table and a backfill.
- No joins, limited ad-hoc filtering and secondary indexes, data duplicated per query.
- Weak transactions: only single-partition atomicity; compare-and-set (Cassandra lightweight transactions) is slow.
- Deletes write "tombstones" that slow reads until compaction removes them.

**Good fit:** time-series and IoT data, event logs, messaging and activity feeds, any huge write stream read back by a known key.

### Column-oriented analytical stores (OLAP)

A normal-looking SQL table, but stored **column by column** on disk instead of row by row. A query that reads 3 of 50 columns reads only those 3, and values in one column compress extremely well.

```sql
-- scans billions of rows but only the date, product and amount columns
SELECT product_id, sum(amount)
FROM sales
WHERE sale_date >= now() - interval '30 days'
GROUP BY product_id;
```

**Engines:** ClickHouse, BigQuery, Snowflake, Amazon Redshift, DuckDB; Parquet is the same idea as a file format.

**Pros**

- Aggregations over billions of rows in seconds.
- High compression (often 5–10× or better), so cheaper storage and faster scans.
- Standard SQL, so analysts can use it directly.

**Cons**

- Slow at fetching or updating single rows; updates and deletes are expensive or batch-only.
- Prefers inserts in large batches, not one row at a time.
- Not a transactional database for an application's day-to-day writes.

**Good fit:** reporting, dashboards, product analytics, data warehouses, log analytics. Usually fed *from* the operational database, not instead of it.

## Graph model

Data is **nodes** (things) and **edges** (relationships), both with properties. Relationships are stored directly, so following them is cheap no matter how big the database is.

```
(Customer C1) -[:BOUGHT]-> (Product P1) <-[:BOUGHT]- (Customer C2) -[:BOUGHT]-> (Product P3)
```

"Customers who bought what you bought also bought…", in Cypher:

```cypher
MATCH (me:Customer {id: 'C1'})-[:BOUGHT]->(:Product)<-[:BOUGHT]-(other:Customer)-[:BOUGHT]->(rec:Product)
WHERE NOT (me)-[:BOUGHT]->(rec)
RETURN rec.id, count(*) AS score
ORDER BY score DESC LIMIT 5;
```

In SQL this is a chain of self-joins that gets slower and harder to write with every extra hop.

**Engines:** Neo4j (Cypher), Amazon Neptune (Gremlin, openCypher, SPARQL), ArangoDB, JanusGraph. Two flavours: property graphs (the above) and RDF triples (`subject – predicate – object`, used in knowledge graphs).

**Pros**

- Multi-hop queries ("friends of friends", "paths between A and B") stay fast as data grows, because each hop follows stored links instead of searching an index.
- The model matches the domain when relationships *are* the data; new relationship types need no schema change.
- Built-in path, shortest-path and graph algorithms (centrality, communities).

**Cons**

- Weak at bulk aggregation over all nodes ("total sales per month"); a columnar store does that far better.
- Hard to shard: splitting a connected graph across machines makes traversals cross the network.
- Smaller ecosystem and fewer people who know the query languages.
- Overkill when relationships are shallow: one or two joins are cheap in SQL, and PostgreSQL handles hierarchies with recursive CTEs.

**Good fit:** fraud detection (rings of linked accounts, cards, devices), recommendations, social networks, network and dependency maps, access-control rules ("can this user reach this resource through any group?"), knowledge graphs.

## Vector model

Data is stored as **embeddings**: lists of numbers (often hundreds to a few thousand) produced by a machine-learning model, where similar meaning gives nearby vectors. The database answers "what is most similar to this?" instead of "what equals this?".

```sql
-- PostgreSQL with pgvector: products described similarly to the user's search text
CREATE TABLE products (id text PRIMARY KEY, description text, embedding vector(1536));
CREATE INDEX ON products USING hnsw (embedding vector_cosine_ops);

SELECT id, description
FROM products
ORDER BY embedding <=> $1      -- $1 = embedding of "something to write with"
LIMIT 5;
```

A search for "something to write with" finds Pen even though neither word appears in its description.

**Engines:** dedicated: Pinecone, Weaviate, Milvus, Qdrant, Chroma. Add-ons: pgvector (PostgreSQL), Elasticsearch/OpenSearch kNN, MongoDB Atlas Vector Search, Redis.

**Pros**

- Search by meaning across text, images and audio; the base of semantic search and RAG (retrieval for LLM prompts).
- Approximate-nearest-neighbour indexes (HNSW, IVF) search millions of vectors in milliseconds.
- Works alongside existing data: often a column in a database you already run.

**Cons**

- **Approximate:** ANN indexes trade accuracy (recall) for speed. They can miss the true best match, and tuning that trade-off is your job.
- **Similar is not correct:** no exact matching. Product codes, names and numbers often need keyword search alongside (hybrid search).
- Combining similarity with filters ("similar, but in stock and under 20") is tricky: filtering after the search can return too few results.
- Embeddings tie you to one model. Changing models means re-embedding everything, and vectors from different models aren't comparable.
- Memory-hungry: 1 million × 1536-dimension float vectors is about 6 GB before index overhead.

**Good fit:** semantic search, RAG for chatbots and assistants, "similar items" recommendations, deduplication and near-duplicate detection, image search.

**Poor fit:** a source of truth. Store the real record elsewhere and keep the vector as a searchable copy.

## Comparison

| | Relational | Document | Wide-column | Columnar (OLAP) | Graph | Vector |
|---|---|---|---|---|---|---|
| **Unit of data** | row in a table | JSON document | row in a partition | column of values | node + edge | embedding |
| **Best question** | anything, ad hoc | "give me this whole thing" | "rows for this key, in order" | "sum/count over everything" | "how is A connected to B?" | "what is most like this?" |
| **Schema** | fixed, enforced | flexible, in the app | fixed per query table | fixed | flexible | fixed dimension |
| **Joins** | strong | weak | none | yes, but costly at scale | the whole point (traversals) | none |
| **Transactions** | strong, multi-row | single doc; multi-doc costly | single partition | weak / batch | varies (Neo4j: ACID) | usually none needed |
| **Scale-out** | harder (read replicas, sharding is work) | good | excellent | excellent for reads | hard | good |
| **Write pattern** | balanced | balanced | very write-heavy | big batches | balanced | batch (re-embed) |
| **Typical use** | the core system of record | catalogs, profiles, content | time-series, logs, feeds | analytics, dashboards | fraud, recommendations | semantic search, RAG |

## Principles for choosing

1. **Start from the access patterns, not the data.** List the real queries: how often, how fast, how much data each touches. The same customer and order data belongs in a document store if you always read whole orders, in a wide-column store if you write millions of events per second, in a columnar store if you mostly aggregate, and in a graph if you mostly follow links.
2. **Default to relational until a specific limit forces a change.** PostgreSQL alone covers documents (`JSONB` with GIN indexes), hierarchies (recursive CTEs), vectors (pgvector) and moderate analytics. Move when you can name the limit: write volume, query shape, latency, size.
3. **Name the consistency you need.** Money, stock and bookings need transactions and constraints; that pushes toward relational. Feeds, metrics and recommendations tolerate eventual consistency, which opens up wide-column and document stores.
4. **Separate operational and analytical work.** The database the app writes to (OLTP) and the one analysts scan (OLAP) want opposite storage layouts. Running big reports on the app's database slows the app; copy data to a columnar store instead.
5. **Treat search-style models as secondary indexes.** Vector and search engines hold copies built from the source of truth, which can be rebuilt. Don't make them the only place data lives.
6. **Count the cost of every extra database.** Each one adds operations, backups, monitoring, security, skills, and a consistency gap between stores that you must keep in sync (change-data-capture or an outbox; see [[data-storage]]). Two databases you run well beat four you run badly.
7. **Model for the query you can't change later.** Document and wide-column designs are cheap to read the way they were designed and expensive to read any other way. If future queries are unknown, that argues for relational.

## Quick decision guide

| If your main need is… | Choose | Before that, check whether… |
|---|---|---|
| varied entities read whole, flexible fields | **document** | PostgreSQL `JSONB` is enough |
| huge write volume read back by key and time | **wide-column** | partitioned PostgreSQL or a time-series extension copes |
| aggregations over large history | **columnar (OLAP)** | the volume really outgrows a read replica |
| many-hop relationship queries | **graph** | the queries are deeper than 2–3 joins |
| similarity / meaning-based search | **vector** | pgvector in your existing database is enough |
| none of the above clearly | **relational** | |

Related: [[database-index-types]] for indexing inside a relational database, [[database-transactions-isolation]] for what "transactions" mean in each model.
