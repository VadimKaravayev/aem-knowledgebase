# Database index types: B-tree, Hash, GiST/GIN, partial, covering

An index is a separate, sorted lookup structure that lets the database jump to matching rows instead of reading the whole table, like the index at the back of a book. **Indexes speed up reads and slow down writes**: every `INSERT`, `UPDATE` or `DELETE` also updates every index on the table, inside the same transaction. A well-chosen index often fixes a slow query faster than any code change.

Syntax and names below are PostgreSQL; MySQL, SQL Server and Oracle have most of the same ideas under slightly different names. For composite-index column order and the general read/write trade-off, see [[data-storage]].

## Example table

```sql
CREATE TABLE users (
  id         BIGINT PRIMARY KEY,
  email      TEXT,
  name       TEXT,
  country    TEXT,
  is_active  BOOLEAN,
  created_at TIMESTAMP,
  tags       TEXT[],
  bio        TEXT
);
```

## Which type for which query

| Index | Good for | Not for |
|---|---|---|
| **B-tree** (default) | `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'abc%'` | `LIKE '%abc'`, full-text, arrays |
| **Hash** | `=` only | ranges, sorting, uniqueness, multi-column |
| **GIN** | "contains" searches: full-text, arrays, JSONB | ranges, sorting |
| **GiST** | geometry, geo data, ranges, nearest-neighbour | plain equality on simple values |
| **Partial** (any type + `WHERE`) | queries that always filter the same subset | queries outside that subset |
| **Covering** (B-tree + `INCLUDE`) | queries answerable from the index alone | wide or often-changing extra columns |

## B-tree: the default

A sorted tree, kept balanced, so lookups take a few steps even for millions of rows. Because it's sorted, it handles equality, ranges and ordering.

```sql
CREATE INDEX users_created_at_idx ON users (created_at);

SELECT * FROM users WHERE created_at >= now() - interval '7 days';
SELECT * FROM users ORDER BY created_at DESC LIMIT 20;  -- reads the index in order, no sort
```

Gotchas:

- `LIKE 'abc%'` (prefix) can use a B-tree; `LIKE '%abc'` (leading wildcard) can't. In PostgreSQL the prefix case works only with the `C` collation or an index built with `text_pattern_ops`: `CREATE INDEX ON users (email text_pattern_ops);`.
- A function on the column bypasses the index: `WHERE lower(email) = 'a@example.com'` needs an expression index, `CREATE INDEX ON users (lower(email));`.
- Primary keys and `UNIQUE` constraints create B-tree indexes automatically.

## Hash: equality only

Stores a hash of each value, so it answers only "is it exactly this?".

```sql
CREATE INDEX users_email_hash_idx ON users USING hash (email);

SELECT * FROM users WHERE email = 'a@example.com';  -- can use it
SELECT * FROM users WHERE email LIKE 'a%';          -- can't
```

It is **rarely worth choosing** over a B-tree. The speed gain on `=` is small in practice, and B-tree does everything Hash does and more. Where it can win: very long values (long URLs, tokens), because the index stores a fixed-size hash rather than the value. Limits in PostgreSQL: no `UNIQUE`, no multi-column, no sorting. Before PostgreSQL 10, hash indexes were not crash-safe and not replicated, so old advice says never to use them.

## GIN and GiST: for values with inner structure

A B-tree compares whole values. Some questions are about what's *inside* a value: words in a text, items in an array, a point on a map. Those need different index types.

**GIN** (Generalized Inverted Index) works like a book index: for each word or element, a list of rows that contain it. Best for "does this row contain X?".

```sql
-- full-text search
CREATE INDEX users_bio_fts_idx ON users USING gin (to_tsvector('english', bio));
SELECT * FROM users WHERE to_tsvector('english', bio) @@ to_tsquery('database & index');

-- arrays: users tagged 'admin'
CREATE INDEX users_tags_idx ON users USING gin (tags);
SELECT * FROM users WHERE tags @> ARRAY['admin'];
```

GIN is fast to search but slow to update: one new row can touch many entries, one per word.

**GiST** (Generalized Search Tree) groups values into nested "boxes" (bounding regions), so it can answer "what overlaps this?" or "what's nearest?". It's the standard index for geo data (PostGIS), ranges, and nearest-neighbour search.

```sql
-- requires PostGIS; location is a geography column
CREATE INDEX shops_location_idx ON shops USING gist (location);
SELECT * FROM shops ORDER BY location <-> ST_MakePoint(0, 0)::geography LIMIT 5;

-- ranges: bookings that overlap a time window
CREATE INDEX bookings_period_idx ON bookings USING gist (period);
SELECT * FROM bookings WHERE period && tsrange(now()::timestamp, (now() + interval '1 day')::timestamp);
```

GiST also supports full-text search, but it's lossy (it re-checks rows), so it searches more slowly than GIN while updating faster. Rule of thumb: **GIN for text, arrays and JSONB; GiST for geometry, ranges and "nearest"**.

## Partial index: only some of the rows

An index with a `WHERE` clause: only rows matching it are indexed.

```sql
CREATE INDEX users_active_email_idx ON users (email) WHERE is_active = true;

SELECT * FROM users WHERE email = 'a@example.com' AND is_active = true;  -- uses it
SELECT * FROM users WHERE email = 'a@example.com';                       -- can't
```

If 5% of users are active, the index is about 20 times smaller, fits in memory, and inactive rows cost nothing to write. Catch: the query's `WHERE` must clearly imply the index's condition. The planner doesn't guess, and a parameter (`is_active = $1`) usually isn't enough.

A partial unique index enforces a rule on a subset, for example "email must be unique among active users":

```sql
CREATE UNIQUE INDEX ON users (email) WHERE is_active = true;
```

## Covering index: no trip to the table

Normally an index gives the database the row's location, and it then reads the table to get the other columns. A covering index holds everything the query needs, so the table is skipped (an *index-only scan*).

```sql
CREATE INDEX users_email_covering_idx ON users (email) INCLUDE (name);

SELECT name FROM users WHERE email = 'a@example.com';  -- answered from the index alone
```

`INCLUDE` columns are stored in the index but not used for searching or sorting. That's the difference from `(email, name)`, which would also sort by `name` and make the key wider. `INCLUDE` is available from PostgreSQL 11 and in SQL Server.

Gotchas:

- Every `INCLUDE` column is a copy, so it adds size and write cost. Include only small columns for a hot query.
- In PostgreSQL an index-only scan still checks the table for rows changed since the last `VACUUM`. On a heavily updated table the "covering" benefit can mostly disappear. `EXPLAIN` shows `Heap Fetches` when that happens.

## Deciding what index to create

### When an index is worth it

Start from a **slow or frequent query**, never from a column ("we search by email, so index email"). Good signals:

- The query is near the top of `pg_stat_statements`, sorted by `total_exec_time` (frequency × duration), or shows up in the slow-query log.
- `EXPLAIN ANALYZE` shows a `Seq Scan` on a large table that **returns only a few of its rows**.
- The query does `ORDER BY ... LIMIT` and the plan has a `Sort` over many rows.
- A foreign key column on a child table, such as `order_items.order_id`. PostgreSQL doesn't index it automatically, and joins and `ON DELETE CASCADE` from the parent scan the whole child table without one.
- A rule that must hold, like "email is unique": a `UNIQUE` index enforces it.

Usually not worth it:

- **Small tables**, up to a few thousand rows. Scanning them is already fast.
- **Filters that match a large share of rows**, roughly 10–20% or more, like `is_active = true` when most users are active. Reading the whole table is cheaper than jumping around through an index, so the planner ignores it. (A partial index on the *rare* value can still help.)
- **Write-heavy tables with rare reads**, such as logs and event tables, where every index slows the main job.
- **Columns already covered** by an existing index (see step 4).

### How to choose it

1. **Pick the query** and write down its filters, joins and sort, in full.
2. **Run `EXPLAIN ANALYZE`** on realistic data to see where the time goes. A plan on a 100-row dev table tells you nothing about production.
3. **Choose the columns and their order** for a composite index:
   - columns compared with `=` first,
   - then the one range column (`>`, `<`, `BETWEEN`),
   - then the `ORDER BY` columns, in the same order and direction.
   An index on `(a, b)` helps `WHERE a = ?` and `WHERE a = ? AND b > ?`, but not `WHERE b = ?`.
4. **Check the existing indexes** (`\d table` in psql). An index on `(a, b)` already serves queries on `a` alone, so a separate index on `a` is redundant. Extending an existing index usually beats adding a new one.
5. **Choose the type** from the table at the top: B-tree unless the query is full-text, array, JSONB, geo or range overlap.
6. **Consider making it smaller or doing more**: partial if the query always filters the same subset, covering (`INCLUDE`) if one hot query reads only a couple of extra small columns.
7. **Build it without blocking writes**: `CREATE INDEX CONCURRENTLY` on a live table.
8. **Verify**: rerun `EXPLAIN ANALYZE`. The plan should use the new index and the time should drop. If the planner ignores it, find out why (statistics, a function on the column, a type mismatch) before forcing anything.
9. **Watch it afterwards**: write latency on the table, and `idx_scan` in `pg_stat_user_indexes`. Drop indexes that stay at `idx_scan = 0`, since they cost a write on every insert and give nothing back.

### Worked example

The query, run thousands of times a day:

```sql
SELECT id, name FROM users
WHERE country = $1 AND is_active = true
ORDER BY created_at DESC
LIMIT 20;
```

`EXPLAIN ANALYZE` shows a `Seq Scan` on `users` plus a `Sort`. Going through the steps:

- `country` is compared with `=`, so it goes first. `created_at` is the sort, so it goes second, `DESC` to match.
- Only 5% of users are active, and the query always filters on that, so the index can be partial.
- The query also reads `id` and `name`, so `INCLUDE (id, name)` lets it skip the table. (The index's internal row pointer isn't the `id` column, so `id` must be included too.)

```sql
CREATE INDEX CONCURRENTLY users_country_recent_active_idx
  ON users (country, created_at DESC)
  INCLUDE (id, name)
  WHERE is_active = true;
```

After: the plan shows an `Index Only Scan` that stops after 20 rows, with no `Sort`. The index holds only active users, so it's small and inactive rows cost nothing to write.
