# How databases actually work — notes with diagrams and SQL

Same map as before: an in-memory dictionary is a database until three constraints hit.

1. **Bigger than RAM** → pages, cache, indexes, joins, pagination, partitions  
2. **Many users at once** → MVCC, isolation, locks, pooling, replicas  
3. **Power dies mid-write** → WAL, checkpoints, durable replication  

Running query (every layer):

```sql
SELECT *
FROM orders
WHERE customer_id = $1
  AND created_at >= now() - interval '30 days'
ORDER BY created_at DESC
LIMIT 20;
```

Assume: ~200M rows, ~40 columns, ~800 GB, ~50k reads/s, ~2k writes/s, 11 indexes.

```
  ┌─────────────────────────────────────────────────────────┐
  │  7 Schema    access patterns, ALTER, expand/contract    │
  │  6 Scale     pool, replica, cache, partition, shard     │
  │  5 Tx        snapshots, RR, SSI, SKIP LOCKED, xmin      │
  │  4 Execute   nested / hash / merge, EXPLAIN, work_mem   │
  │  3 Page      OFFSET vs keyset, COUNT, cursors           │
  │  2 Index     B-tree, INCLUDE, bitmap, CONCURRENTLY      │
  │  1 Storage   8KB page, TOAST, buffers, WAL, MVCC, vac   │
  └─────────────────────────────────────────────────────────┘
         ↑ watch bottom four first
```

---

# Module 1 — Storage

## Heap = file of 8 KB pages

Postgres always I/Os **whole pages**. Ask for 40 bytes → still 8192 bytes.

```
heap file (orders)
┌──────────┬──────────┬──────────┬────┬──────────┐
│ page 0   │ page 1   │ page 2   │ …  │ page N   │  each 8192 B
└──────────┴──────────┴──────────┴────┴──────────┘
800 GB ≈ 100 million pages. Goal: touch fewer.
```

### One page in cross-section

```
 offset 0                                              8191
┌────────────┬──────────────────┬─────────────┬────────────┐
│ page hdr   │ line pointers →  │  FREE GAP   │ ← tuples   │
│ 24 bytes   │ 4 B each         │  both eat   │  ~23 B hdr │
└────────────┴──────────────────┴─────────────┴────────────┘
     pd_lower ──────────────────┘             └─ pd_upper
```

**Line pointer** = where a tuple starts + length.  
**ctid** = `(page, slot)`. Slot stays put when the tuple slides during compact. **UPDATE changes ctid** — use PK, never ctid.

```sql
SELECT ctid, customer_id, created_at
FROM orders
WHERE customer_id = 42
LIMIT 5;

-- After UPDATE this row, ctid usually changes:
UPDATE orders SET note = 'x' WHERE id = 1;
SELECT ctid FROM orders WHERE id = 1;
```

A tuple **cannot span two pages**.

## TOAST (~2 KB whole-row line)

```
  main heap page                         toast table
┌─────────────────────┐                ┌──────────────────┐
│ row header + cols   │                │ chunk 1          │
│ big_col → TOAST ptr ├── 2nd fetch ─► │ chunk 2          │
└─────────────────────┘                └──────────────────┘
```

- Compress in place first; leftover goes out-of-line.  
- **EXTENDED** (default): compress + out. **EXTERNAL**: out, no compress (cheap substring).  
- Extra I/O **only if you SELECT that column**. `SELECT *` always pays.

```sql
ALTER TABLE orders ALTER COLUMN body SET STORAGE EXTERNAL;

-- Avoid SELECT * so you don't pull TOAST
SELECT id, customer_id, created_at FROM orders WHERE id = 1;
```

## Seq vs random I/O + buffer pool

```
seq scan:  [p0][p1][p2][p3]  cheap consecutive
random:    [p7]     [p2]        [p91]  planner: random_page_cost ≈ 4× seq
```

```
RAM
┌─────────────────────────────┐
│ shared_buffers (page slots) │  start ~25% RAM, rarely >40%
└──────────────▲──────────────┘
               │ miss
┌──────────────┴──────────────┐
│ OS page cache               │
└──────────────▲──────────────┘
               │ miss → disk
```

Eviction: **clock sweep** (usage counters). Dirty written first.

```sql
SHOW shared_buffers;
EXPLAIN (ANALYZE, BUFFERS)  -- PG18 ANALYZE implies BUFFERS
SELECT * FROM orders WHERE id = 1;
-- look at shared hit vs read
```

PG18: **async I/O** queues many reads (defaults concurrency 16). Waiting shrinks; **page count unchanged**.

## WAL: commit is the log

```
memory:  page dirty ──► append WAL record ──► fsync WAL ──► COMMIT ACK
                              │
                              ▼ later checkpoint (~5 min or ~1GB WAL)
                         flush dirty heap pages
```

Crash: replay WAL after last checkpoint. 10 scattered rows at commit → **one sequential WAL append**, not 10 random heap fsyncs. `full_page_writes`: first change after checkpoint logs **whole page**.

```sql
SHOW wal_level;
SHOW full_page_writes;
SHOW checkpoint_timeout;
```

## MVCC: UPDATE never overwrites

```
BEFORE UPDATE
 slot 3:  xmin=100  xmax=0   {id=1, note='a'}   ← live

AFTER UPDATE (same page if room)
 slot 3:  xmin=100  xmax=200 {id=1, note='a'}   ← old version
 slot 7:  xmin=200  xmax=0   {id=1, note='b'}   ← new version
```

Reader started at xid 150: xmax 200 is “future” → still sees `'a'`.  
Readers don’t wait for writers.

```sql
SELECT xmin, xmax, ctid, * FROM orders WHERE id = 1;
```

## Vacuum, HOT, wraparound

Dead tuples stay until **VACUUM**. File **rarely shrinks** (bloat). Autovacuum jobs: dead tuples, **stats**, **visibility map** (index-only), **freeze** (32-bit xid wrap).

**HOT:** non-indexed column + new version **fits on page** → indexes not updated.

```
11 indexes
UPDATE note (not indexed, space on page)  →  0 index writes  (HOT)
UPDATE created_at (indexed)               →  11 index writes
page full                                 →  11 anyway
```

```sql
ALTER TABLE orders SET (fillfactor = 80);  -- leave room for HOT

SELECT n_live_tup, n_dead_tup, n_tup_hot_upd, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

Wraparound: freeze or at ~3M xids left **writes stop** (read-only).

## B-tree vs LSM

```
B-tree (Postgres)
        [root]
       /  |  \
    [ ]  [ ]  [ ]
    /|\
 leaves linked in key order  → range + sort free
 ~4 page reads to any key among billions
 write = random at leaf

LSM
 memtable → sorted SSTs → compact (write amplification)
 sequential writes; lookups may check many files
```

```sql
SELECT * FROM orders WHERE id = 42;           -- ~4 index + 1 heap
SELECT * FROM orders ORDER BY id LIMIT 20;    -- walk 20 leaves, no Sort
```

---

# Module 2 — Indexes

## Which query is slow?

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;  -- often needs restart
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
-- TOTAL = where time went; MEAN = loud rare reports
```

## Two hops (no clustered PK)

```
  B-tree descent (~4 pages)          heap
  [root]→[inner]→[leaf: key|ctid] ──► [page with tuple]
  1 row ≈ 5 pages. 20 rows cheap; 40_000 heap hops not.
```

**Index-only:** columns ⊆ index **and** visibility map bit. Vacuum sets; writes clear **whole page**.

```sql
CREATE INDEX ON orders (customer_id, created_at DESC)
  INCLUDE (status);  -- payload: covering, not for seek

EXPLAIN SELECT customer_id, created_at, status
FROM orders
WHERE customer_id = 42 AND created_at >= now() - interval '30 days';
-- Index Only Scan if VM is set
```

## Composite rule (the whole game)

```
Equality on leading keys  →  one contiguous band
THEN one range on next    →  trim both ends
Further right             →  in-index filter, walk NOT shorter
```

```sql
-- GOOD for our query
CREATE INDEX idx_orders_cust_time
  ON orders (customer_id, created_at DESC);

-- BAD: date first → 30 days of ALL customers
CREATE INDEX idx_orders_time_cust
  ON orders (created_at DESC, customer_id);
```

```
(customer, date)  descent → one customer → trim 30d → LIMIT 20 STOP
(date, customer)  30d of everyone (~8M) then filter; sparse customer walks ALL
two singles       rows yes, order no → Sort; LIMIT cannot stop inner walk
```

**PG18 skip scan:** missing leading equality → one hop **per distinct leading value** (6 statuses OK; 1M customers not).

Docs: past **3 key columns** rarely worth it (fanout + WAL).

## Planner ignores a “correct” index

Seq cost **flat**; index cost **rises with matches**. No magic 5%.

```sql
-- correlated: CAD always CAD$
CREATE STATISTICS st_country_currency
  ON country, currency FROM orders;
ANALYZE orders;

SHOW effective_cache_size;     -- default 4GB is a GUESS, allocates nothing
SHOW random_page_cost;         -- 4 already assumes cache; fully cached → equalize seq/random
```

**EXPLAIN habits**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

1. **Rows:** estimate vs actual — **ratio**; deepest blow-up; `loops × rows`  
2. **BUFFERS:** hit / read (counts accesses)  
3. **Memory:** `quicksort` vs `external merge` + temp blocks  

## Bitmap (two weak indexes)

```
index A ──► bitmap     AND/OR     heap in PHYSICAL order (order LOST)
index B ──► bitmap  ──────────►  then Sort if you asked ORDER BY
```

LIMIT cannot early-exit a Sort. `work_mem` too small → **lossy** bits per page → **recheck**.

## Build without locking writes

```sql
-- BLOCKS WRITERS (reads continue). Queue of inserts can exhaust the pool.
CREATE INDEX idx_orders_cust_time ON orders (customer_id, created_at DESC);

-- Two scans; not in a transaction; failure → INVALID (still maintained!)
CREATE INDEX CONCURRENTLY idx_orders_cust_time
  ON orders (customer_id, created_at DESC);

SELECT indexrelid::regclass, indisvalid
FROM pg_index JOIN pg_class ON pg_class.oid = indexrelid
WHERE relname LIKE 'idx_orders%';
```

## Splits and UUIDv4

```
random keys     split middle     leaves ~65–70% full
monotonic keys  pack right edge  ~90% full (fillfactor 90)
```

UUIDv4: no locality. **UUIDv7**: timestamp prefix.

```sql
-- PG18
SELECT uuidv7();
```

**BRIN** only if column **correlates with heap order** (`pg_stats.correlation`). GIN for jsonb/arrays/FTS. Partial index for `WHERE deleted_at IS NULL`. **FK does not create an index on the child.**

Eleven indexes = eleven writes per non-HOT update. Unused index = tax.

---

# Module 3 — Pagination

```
OFFSET n     walk n+20, throw away n     cost ∝ page number
KEYSET       seek to last key, walk 20   page 1 ≈ page 50_000
```

```sql
-- OFFSET: skipped rows still computed
SELECT * FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 10000;

-- KEYSET (row comparison ≠ two ANDs)
SELECT * FROM orders
WHERE customer_id = 42
  AND (created_at, id) < ($last_ts, $last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Need **unique tail** (`id`) or ties duplicate/skip with **no error**. Nulls in compare → unknown → **dropped rows**. Cursor = those keys in a signed token. Fetch 21 for `has_next`.

**COUNT(*)** is a separate full-filter scan. `reltuples` is an estimate. Server cursor pins **xmin** cluster-wide.

```sql
SELECT reltuples FROM pg_class WHERE relname = 'orders';
```

---

# Module 4 — Execution

```
Nested loop   for each outer row, probe inner (index inner + SMALL outer = fastest)
Hash join     build hash on smaller (equality); spill batches if > work_mem*hash_mem_multiplier
Merge join    both sorted on join keys; two pointers; index can supply order
```

Planner **prices all three from row estimates**. Errors **multiply** up the tree. 12 FROM items → GEQO.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.*, c.email
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.customer_id = 42
ORDER BY o.created_at DESC
LIMIT 20;
```

**JOIN vs IN vs EXISTS**

```sql
-- JOIN duplicates (6 items × order = 6 output rows)
SELECT o.* FROM orders o JOIN items i ON i.order_id = o.id;

-- EXISTS: true/false, stop at first match
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM items i WHERE i.order_id = o.id);

-- NOT IN + NULL in subquery → ALL rows vanish, no error
SELECT * FROM orders
WHERE warehouse_id NOT IN (SELECT id FROM warehouses);  -- if any id is NULL, empty

-- Use this
SELECT * FROM orders o
WHERE NOT EXISTS (
  SELECT 1 FROM warehouses w WHERE w.id = o.warehouse_id AND w.congested
);
```

**work_mem is per operator**, not per query. N sessions × M hashes can exceed RAM.

**N+1:** 500 PK lookups never appear as one plan. `pg_stat_statements`: huge `calls`, tiny mean.

---

# Module 5 — Transactions

```
Read Committed     new snapshot EVERY statement     nonrepeatable read
Repeatable Read    one snapshot (first real stmt)   snapshot isolation
                   same-row conflict → ROLLBACK
                   WRITE SKEW still allowed
Serializable SSI   predicate locks, no wait, error 40001 → RETRY whole tx
```

**Write skew (hospital):** two tx, two different rows, both “other is on call,” both commit at RR → rota empty.

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- check count of on_call = 1, then UPDATE self off_call; COMMIT;
-- both succeed at RR; SERIALIZABLE cancels one with 40001
```

```sql
-- queue workers
SELECT * FROM jobs
WHERE status = 'queued'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

Idle `BEGIN` pins **global xmin** → vacuum **everywhere** stalls.

```sql
SELECT pid, state, xact_start, query
FROM pg_stat_activity
WHERE state = 'idle in transaction';

SET idle_in_transaction_session_timeout = '30s';
```

---

# Module 6 — Scale

One client connection = **one OS process**. Idle costs like busy. **PgBouncer transaction pooling:** next statement may be another backend — prepared statements / temp tables / LISTEN **die**.

Replication = **WAL replay**.

```
async: COMMIT on primary ──gap── replica   crash = documented data loss = lag
sync:  wait remote_write / on / remote_apply   RTT is a FLOOR
replica is NOT a backup (replays DELETE)
```

Read-your-writes: write primary, read replica → comment missing.

```sql
-- after write, capture LSN
SELECT pg_current_wal_lsn();          -- primary
SELECT pg_last_wal_replay_lsn();      -- replica
-- app: poll until replica >= write LSN, deadline, else read primary
```

Partition prune is **proof**, not a guess:

```sql
CREATE TABLE orders (
  id bigint NOT NULL,
  customer_id bigint NOT NULL,
  created_at timestamptz NOT NULL,
  PRIMARY KEY (id, created_at)          -- MUST include partition key
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_03 PARTITION OF orders
  FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

ALTER TABLE orders DETACH PARTITION orders_2025_01;  -- cheap vs DELETE
```

`DELETE` 6M rows = dead versions + WAL + no file shrink. **Shard** = extra machines; sequential shard key = one hot shard.

Redis = third buffer pool. **You** invalidate. 4% hit rate = long-tail + sliding window, not “need more RAM.”

---

# Module 7 — Schema

Design from **queries**, not nouns.

```sql
-- UNIQUE + NULL: many NULLs allowed unless:
CREATE UNIQUE INDEX ON payments (idempotency_key) NULLS NOT DISTINCT;  -- PG15

-- partial index for soft delete
CREATE INDEX ON orders (customer_id, created_at DESC)
  WHERE deleted_at IS NULL;

-- constant default = metadata; now() is VOLATILE = rewrite table+indexes
ALTER TABLE orders ADD COLUMN flag int DEFAULT 0;           -- fast
ALTER TABLE orders ADD COLUMN ts timestamptz DEFAULT now(); -- rewrite trap
-- better: add NULL, then
ALTER TABLE orders ALTER COLUMN ts SET DEFAULT now();
-- batched backfill
```

**ACCESS EXCLUSIVE** blocks **reads**. Waiting lock **queues everyone behind it**.

```sql
BEGIN;
SET LOCAL lock_timeout = '5s';
ALTER TABLE ...;
COMMIT;
```

Expand/contract: add column → dual-write → batched backfill → flip reads → drop old.

UUIDv4 PK = random leaf splits for life. EAV/jsonb-as-EAV starves planner. `SELECT *` + wide rows → TOAST.

---

# Incident (finale)

p99 80ms → 3.5s, replica 45s behind, autovacuum 9 days.

```
standby long query ──► replay stall (lag)
        └── hot_standby_feedback ──► xmin on primary ──► bloat, vacuum “runs” but reclaims nothing
index (customer_id) only ──► cannot trim time; LIMIT walks history
CREATE INDEX without CONCURRENTLY ──► write lock
12th index ──► write tax on 2k/s
Redis 4% ──► keys never repeat
nested loop 15 vs 40k ──► correlated stats + ANALYZE never ran
partition by time ──► prune + drop month; shard is the wrong machine-count answer
```

---

# Three tutorials (SQL + diagrams + memo)

## Easy — see the page

```sql
CREATE TABLE demo (
  id bigserial PRIMARY KEY,
  customer_id int NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  note text
);
INSERT INTO demo (customer_id, note)
SELECT (random()*1000)::int, 'x' FROM generate_series(1, 100000);

EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM demo WHERE id = 1;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM demo WHERE customer_id = 42;
SELECT pg_relation_size('demo') / 8192 AS pages;
SELECT ctid, * FROM demo LIMIT 3;
```

```
query → shared_buffers → OS cache → 8KB disk page
```

**Memo:** Postgres never fetches “a column.” It fetches **pages**. `ctid` is location, not identity.

---

## Medium — 20 recent orders

```sql
CREATE INDEX ON demo (customer_id);
CREATE INDEX ON demo (created_at DESC);
CREATE INDEX ON demo (customer_id, created_at DESC);

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM demo
WHERE customer_id = 42
  AND created_at >= now() - interval '30 days'
ORDER BY created_at DESC
LIMIT 20;

-- OFFSET tax vs keyset
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM demo WHERE customer_id = 42
ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 5000;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM demo
WHERE customer_id = 42 AND (created_at, id) < (now(), 0)
ORDER BY created_at DESC, id DESC LIMIT 20;
```

```
(customer, time) → one band → already sorted → STOP at 20
OFFSET 5000      → compute 5020, throw 5000
```

**Memo:** Equality first, range last, ORDER BY = leaf order. OFFSET still **computes** skips. Indexing the column you UPDATE kills HOT.

---

## Very hard — incident week

```sql
-- 1 write skew
-- session A and B, RR, two different doctors, both UPDATE off_call

-- 2 idle xmin
BEGIN;
UPDATE demo SET note = 'held' WHERE id = 1;
-- leave open; watch n_dead_tup on OTHER tables

-- 3 LSN wait (run replay LSN on replica)
SELECT pg_current_wal_lsn();

-- 4 correlation
CREATE STATISTICS s0 ON country, currency FROM orders;
ANALYZE orders;

-- 5 online index
CREATE INDEX CONCURRENTLY ...;
SELECT indisvalid FROM pg_index WHERE indexrelid = 'idx'::regclass;

-- 6 lock_timeout + constant default
BEGIN;
SET LOCAL lock_timeout = '5s';
ALTER TABLE demo ADD COLUMN flag int DEFAULT 0;
COMMIT;

-- 7 partition + drop vs delete
```

```
40001 ──► retry whole transaction from first read
INVALID index ──► drop; still paid on every write until you do
```

**Memo:** One standby query can look like three outages. Cache hit rate tests **repeat keys**. Serializable is **retry in the app**. Never ACCESS EXCLUSIVE without `lock_timeout`. Partition to **drop time slices**; shard only when you need **more write machines**.