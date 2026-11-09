# Database transactions and isolation levels

A **transaction** is a group of statements the database treats as one unit: either all of them take effect or none do. **Isolation levels** decide how much transactions running at the same time can see of each other's unfinished or freshly committed work. Higher levels prevent more surprises and cost more waiting or more retries.

Examples use PostgreSQL syntax; differences in MySQL, SQL Server and Oracle are called out where they matter.

## Transactions

### The classic example: a money transfer

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

If the database crashes, or the second `UPDATE` fails, between the two statements, neither change survives. Without a transaction, 100 could leave account 1 and never arrive in account 2.

- `COMMIT` makes the changes permanent and visible to others.
- `ROLLBACK` throws away everything since `BEGIN`.
- Without `BEGIN`, most databases run in **autocommit** mode: each statement is its own transaction.
- `SAVEPOINT name` / `ROLLBACK TO name` undoes part of a transaction without abandoning all of it.

### ACID in plain words

| Letter | Means | Plain words |
|---|---|---|
| **A**tomicity | all or nothing | The transfer happens completely or not at all. |
| **C**onsistency | rules hold | Constraints (`CHECK balance >= 0`, foreign keys, `UNIQUE`) are true before and after. |
| **I**solation | others don't interfere | Concurrent transactions don't see each other's half-done work. *How much* they're isolated is the isolation level. |
| **D**urability | committed means saved | After `COMMIT` returns, the change survives a crash (it's in the write-ahead log on disk). |

Atomicity and durability are basically all-or-nothing. Isolation is the one with settings, and the rest of this doc is about it.

## What can go wrong when transactions overlap

Each example has two sessions, A and B, running at the same time. Read the steps top to bottom.

### Dirty read: seeing uncommitted data

| Step | Session A | Session B |
|---|---|---|
| 1 | `UPDATE accounts SET balance = 0 WHERE id = 1;` | |
| 2 | | `SELECT balance FROM accounts WHERE id = 1;` → **0** |
| 3 | `ROLLBACK;` | |

B acted on a value that never officially existed.

### Non-repeatable read: same row, different answer

| Step | Session A | Session B |
|---|---|---|
| 1 | `SELECT balance ... WHERE id = 1;` → 500 | |
| 2 | | `UPDATE accounts SET balance = 200 WHERE id = 1; COMMIT;` |
| 3 | `SELECT balance ... WHERE id = 1;` → **200** | |

Within one transaction, A read the same row twice and got two values. A report that sums accounts on one line and checks the total on another can disagree with itself.

### Phantom read: new rows appear

| Step | Session A | Session B |
|---|---|---|
| 1 | `SELECT count(*) FROM orders WHERE customer = 'C1';` → 3 | |
| 2 | | `INSERT INTO orders (customer) VALUES ('C1'); COMMIT;` |
| 3 | `SELECT count(*) FROM orders WHERE customer = 'C1';` → **4** | |

Same as non-repeatable read, but for the *set* of rows matching a condition rather than one row's value.

### Lost update: one write silently overwrites another

| Step | Session A | Session B |
|---|---|---|
| 1 | `SELECT stock FROM products WHERE id = 7;` → 10 | |
| 2 | | `SELECT stock FROM products WHERE id = 7;` → 10 |
| 3 | app computes 10 − 1, `UPDATE ... SET stock = 9;` `COMMIT;` | |
| 4 | | app computes 10 − 2, `UPDATE ... SET stock = 8;` `COMMIT;` |

Three items were sold, but stock went from 10 to 8. A's update is lost. This is the most common real-world concurrency bug: read a value into the application, change it, write it back.

### Write skew: two valid changes, an invalid result

Rule: at least one doctor must be on call. Doctors A and B are both on call.

| Step | Session A (doctor A) | Session B (doctor B) |
|---|---|---|
| 1 | `SELECT count(*) FROM on_call WHERE on_duty;` → 2 | |
| 2 | | `SELECT count(*) FROM on_call WHERE on_duty;` → 2 |
| 3 | "2 > 1, I can leave": `UPDATE on_call SET on_duty = false WHERE doctor = 'A'; COMMIT;` | |
| 4 | | "2 > 1, I can leave": `UPDATE on_call SET on_duty = false WHERE doctor = 'B'; COMMIT;` |

Nobody is on call. Each transaction was correct given what it saw, and they changed *different* rows, so no row-level conflict happened. Only full serializability catches this.

## The four isolation levels

The SQL standard defines four levels by which of the first three problems they allow:

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| **Read Uncommitted** | possible | possible | possible |
| **Read Committed** | prevented | possible | possible |
| **Repeatable Read** | prevented | prevented | possible |
| **Serializable** | prevented | prevented | prevented |

**Serializable** means more than the table shows: the result must be the same as if the transactions had run one after another in *some* order. That rules out lost updates and write skew as well.

In plain words:

- **Read Uncommitted**: you may see others' unfinished work. Almost never what you want.
- **Read Committed**: each *statement* sees data committed before it started. Two statements in the same transaction can see different data.
- **Repeatable Read**: the whole *transaction* sees one snapshot taken at its first statement. Others' commits are invisible until you finish.
- **Serializable**: as if nobody else were running. The database either makes you wait or aborts one transaction, which you then retry.

Set it per transaction:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- ...
COMMIT;
```

## What real databases actually do

The standard sets minimums; real behavior differs, and the differences cause bugs when code moves between databases.

| | PostgreSQL | MySQL (InnoDB) | SQL Server | Oracle |
|---|---|---|---|---|
| **Default** | Read Committed | **Repeatable Read** | Read Committed | Read Committed |
| **Read Uncommitted** | behaves as Read Committed | real dirty reads | real dirty reads (`NOLOCK`) | not offered |
| **Repeatable Read** | snapshot; also prevents phantoms; **detects lost updates** (aborts) | snapshot for plain `SELECT`; **lost updates still possible** | lock-based; phantoms possible | not offered |
| **Serializable** | true serializable (SSI), aborts on conflict | lock-based: plain `SELECT`s take shared locks | lock-based (range locks) | actually snapshot isolation: **allows write skew** |

Points that bite:

- **PostgreSQL never shows dirty reads**, at any level.
- **MySQL's default is higher than everyone else's**, but in Repeatable Read a plain `SELECT` reads the snapshot while `UPDATE` and `SELECT ... FOR UPDATE` read the *latest* data. The lost-update example above still happens.
- **Oracle's "Serializable" isn't**: it's snapshot isolation, so the write-skew example succeeds.
- **SQL Server** can switch Read Committed to snapshot reads with the `READ_COMMITTED_SNAPSHOT` database option (on by default in Azure SQL Database), which changes blocking behavior noticeably.
- "Snapshot isolation" is what most databases' Repeatable Read really is: consistent reads, but not full serializability.

## Fixing the common problems

### Lost update

Pick one, in order of preference:

1. **Do the math in the database**, as one atomic statement:
   ```sql
   UPDATE products SET stock = stock - 1 WHERE id = 7 AND stock >= 1;
   ```
   Check the affected-row count: 0 means out of stock.
2. **Lock the row while you work** (pessimistic locking):
   ```sql
   BEGIN;
   SELECT stock FROM products WHERE id = 7 FOR UPDATE;  -- others wait here
   UPDATE products SET stock = 9 WHERE id = 7;
   COMMIT;
   ```
3. **Version column** (optimistic locking), good when the read and the write are in separate requests, such as a form a user edits:
   ```sql
   UPDATE products SET stock = 9, version = version + 1
   WHERE id = 7 AND version = 3;   -- 0 rows updated = someone changed it; reload and retry
   ```
4. **Repeatable Read or Serializable** in PostgreSQL, plus retry on error (not enough in MySQL Repeatable Read).

### Write skew

- Run the transaction as **Serializable** (PostgreSQL) and retry on failure, or
- **lock the rows the decision is based on**: `SELECT ... FROM on_call WHERE on_duty FOR UPDATE;`, or
- turn the rule into a **constraint** the database enforces, where possible.

## In practice

- **Read Committed plus targeted fixes** (atomic updates, `FOR UPDATE`, version columns) is the common default and works well when you know where the hot spots are.
- **Serializable plus a retry loop** is simpler to reason about: write the code as if it were alone, retry when the database says no. It costs some throughput and requires every transaction to be safe to retry.
- **Retry on the right errors**: serialization failure (SQLSTATE `40001`) and deadlock (`40P01` in PostgreSQL, error 1213 in MySQL). Retry the *whole* transaction, not the failed statement, with a small random delay.
- **Keep transactions short.** No network calls, user input or file uploads between `BEGIN` and `COMMIT`. Long transactions hold locks, block others, and in PostgreSQL stop `VACUUM` from cleaning up old row versions. Watch for sessions shown as `idle in transaction`.
- **Deadlocks** happen when two transactions lock the same rows in opposite order. The database kills one; avoid them by always locking in a consistent order, for example by ascending `id`.
- An **ORM or framework** often opens transactions for you (per request, per method annotation). Know where they start and end; an accidental transaction around a slow external call is a common cause of lock pile-ups.

Related: [[database-normalization]] for where each fact lives, [[database-index-types]] for making the queries inside transactions fast.
