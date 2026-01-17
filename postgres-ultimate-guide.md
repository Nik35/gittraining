# PostgreSQL Ultimate Guide

## Introduction
This guide covers essential PostgreSQL topics for database enthusiasts and professionals.

## 1. Vacuuming

Vacuuming is a crucial maintenance task in PostgreSQL that helps reclaim storage and maintain performance. PostgreSQL uses Multi-Version Concurrency Control (MVCC), which means that when rows are updated or deleted, the old versions (dead tuples) are not immediately removed. Vacuuming cleans up these dead tuples to prevent table bloat and transaction ID wraparound.

### Types of Vacuuming

#### 1.1 Routine VACUUM
Regular `VACUUM` reclaims storage occupied by dead tuples and makes it available for reuse within the same table. It does not return space to the operating system.

```sql
-- Vacuum a specific table
VACUUM my_table;

-- Vacuum all tables in the current database
VACUUM;

-- Vacuum with verbose output to see details
VACUUM VERBOSE my_table;
```

**When to use:** Run regularly on tables with frequent UPDATEs and DELETEs to prevent table bloat.

#### 1.2 VACUUM FULL
`VACUUM FULL` rewrites the entire table and indexes, reclaiming space back to the operating system. However, it requires an exclusive lock on the table.

```sql
-- Perform a full vacuum (use cautiously in production)
VACUUM FULL my_table;

-- Full vacuum with analyze
VACUUM FULL ANALYZE my_table;
```

**Warning:** This locks the table exclusively and can be very slow for large tables. Use only during maintenance windows.

#### 1.3 Autovacuum
Autovacuum is a background process that automatically performs VACUUM and ANALYZE operations. It monitors table activity and triggers vacuum when thresholds are exceeded.

**Configuration in `postgresql.conf`:**
```conf
# Enable autovacuum (enabled by default)
autovacuum = on

# Number of autovacuum worker processes
autovacuum_max_workers = 3

# Naptime between autovacuum runs
autovacuum_naptime = 1min

# Minimum number of updated/deleted tuples before vacuum
autovacuum_vacuum_threshold = 50

# Scale factor for threshold (as fraction of table size)
autovacuum_vacuum_scale_factor = 0.2

# Minimum number of inserted tuples before vacuum (for insert-only tables)
autovacuum_vacuum_insert_threshold = 1000
```

**Monitor autovacuum activity:**
```sql
-- Check last autovacuum time
SELECT 
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

#### 1.4 VACUUM FREEZE
`VACUUM FREEZE` is a special type of vacuum that prevents transaction ID wraparound by "freezing" old transaction IDs. PostgreSQL uses 32-bit transaction IDs, which wrap around after about 2 billion transactions.

```sql
-- Freeze transaction IDs in a table
VACUUM FREEZE my_table;

-- Check age of oldest transaction ID
SELECT 
    relname,
    age(relfrozenxid) as xid_age,
    pg_size_pretty(pg_total_relation_size(oid)) as size
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 10;

-- Configure freeze parameters in postgresql.conf
-- vacuum_freeze_min_age = 50000000          -- Minimum age before freezing
-- vacuum_freeze_table_age = 150000000       -- Age at which to scan entire table
-- autovacuum_freeze_max_age = 200000000     -- Maximum age before forced autovacuum
```

**Best Practice:** Monitor the age of the oldest transaction ID and ensure autovacuum is running properly to prevent wraparound issues.

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/sql-vacuum.html">PostgreSQL VACUUM Documentation</a>
  - <a href="https://www.postgresql.org/docs/18/routine-vacuuming.html">Routine Vacuuming</a>

## 2. Indexing

Indexes are one of the most powerful tools for improving query performance in PostgreSQL. They allow the database to find rows much faster than scanning the entire table. However, indexes also have costs: they consume disk space and slow down writes (INSERT, UPDATE, DELETE). Choosing the right index type and strategy is crucial.

### Index Types

#### 2.1 B-Tree Indexes (Default)
B-Tree indexes are the default and most commonly used index type. They work well for equality and range queries on sortable data.

```sql
-- Create a simple B-Tree index
CREATE INDEX idx_users_email ON users(email);

-- Multi-column (composite) index
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Index with sort order
CREATE INDEX idx_products_price_desc ON products(price DESC);

-- Use cases:
-- WHERE email = 'user@example.com'
-- WHERE customer_id = 123 AND order_date > '2024-01-01'
-- ORDER BY price DESC
```

**Best for:** Equality comparisons (`=`), range queries (`<`, `>`, `BETWEEN`), sorting (`ORDER BY`), and pattern matching with leading wildcards (`LIKE 'abc%'`).

#### 2.2 GiST (Generalized Search Tree) Indexes
GiST indexes support complex data types and are commonly used for geometric data, full-text search, and range types.

```sql
-- Create GiST index for geometric data
CREATE INDEX idx_locations_point ON locations USING gist(coordinates);

-- Create GiST index for range types
CREATE INDEX idx_events_timerange ON events USING gist(event_period);

-- Create GiST index for full-text search
CREATE INDEX idx_documents_fts ON documents USING gist(to_tsvector('english', content));

-- Use cases:
-- WHERE coordinates && ST_MakeEnvelope(...)  -- geometric overlap
-- WHERE event_period @> '2024-01-15'::timestamp  -- range contains
```

**Best for:** Geometric data (PostGIS), range types, full-text search, and custom data types with overlapping concepts.

#### 2.3 GIN (Generalized Inverted Index) Indexes
GIN indexes are optimized for indexing composite values where each element can appear in multiple rows, such as arrays, JSONB, and full-text search.

```sql
-- Create GIN index for array column
CREATE INDEX idx_tags_array ON articles USING gin(tags);

-- Create GIN index for JSONB
CREATE INDEX idx_metadata_jsonb ON products USING gin(metadata);

-- Create GIN index for full-text search (faster than GiST for FTS)
CREATE INDEX idx_content_fts ON documents USING gin(to_tsvector('english', content));

-- Create GIN index with jsonb_path_ops for better performance on @> operator
CREATE INDEX idx_metadata_path_ops ON products USING gin(metadata jsonb_path_ops);

-- Use cases:
-- WHERE tags @> ARRAY['postgresql', 'database']  -- array contains
-- WHERE metadata @> '{"category": "electronics"}'  -- JSONB contains
-- WHERE to_tsvector('english', content) @@ to_tsquery('postgresql & database')
```

**Best for:** Arrays, JSONB documents, full-text search, and any data type where you need to search within composite values.

#### 2.4 BRIN (Block Range Index) Indexes
BRIN indexes are very small and efficient for large tables where data has natural ordering (e.g., timestamp columns in append-only tables).

```sql
-- Create BRIN index for time-series data
CREATE INDEX idx_logs_timestamp_brin ON logs USING brin(created_at);

-- Create BRIN index with specific pages_per_range
CREATE INDEX idx_events_date_brin ON events USING brin(event_date) 
WITH (pages_per_range = 128);

-- Use cases:
-- WHERE created_at > '2024-01-01'  -- on large, chronologically ordered tables
-- Very effective for time-series data, logs, and IoT sensor data
```

**Best for:** Very large tables with natural ordering (timestamps, sequential IDs). BRIN indexes are extremely small compared to B-Tree.

**Trade-off:** Less precise than B-Tree, so may scan more rows, but the tiny size makes them ideal for massive tables.

#### 2.5 Hash Indexes
Hash indexes support only equality comparisons and are generally not recommended since B-Tree indexes can do the same job with more features.

```sql
-- Create hash index (rarely needed)
CREATE INDEX idx_codes_hash ON codes USING hash(code);

-- Use case:
-- WHERE code = 'ABC123'  -- equality only
```

**Note:** Hash indexes are now WAL-logged (since PostgreSQL 10) but still offer limited advantages over B-Tree.

### Advanced Indexing Strategies

#### 2.6 Partial Indexes
Partial indexes only index rows that match a specific condition, making them smaller and faster for queries targeting that subset.

```sql
-- Index only active users
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- Index only recent orders
CREATE INDEX idx_recent_orders ON orders(order_date) 
WHERE order_date > '2024-01-01';

-- Index only non-null values
CREATE INDEX idx_optional_phone ON users(phone) WHERE phone IS NOT NULL;

-- Query that uses the partial index:
SELECT * FROM users WHERE status = 'active' AND email = 'user@example.com';
```

**Benefits:** Smaller index size, faster updates, reduced maintenance overhead.

#### 2.7 Expression Indexes
Expression indexes allow you to index the result of a function or expression rather than raw column values.

```sql
-- Index lowercase version of email for case-insensitive searches
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Index extracted JSON field
CREATE INDEX idx_metadata_category ON products((metadata->>'category'));

-- Index date part for grouping by year
CREATE INDEX idx_orders_year ON orders(EXTRACT(YEAR FROM order_date));

-- Query that uses the expression index:
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
```

#### 2.8 Unique Indexes
Unique indexes enforce uniqueness constraints while also providing index benefits.

```sql
-- Create unique index
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Multi-column unique constraint
CREATE UNIQUE INDEX idx_unique_username_tenant ON users(username, tenant_id);

-- Partial unique index (unique only for specific subset)
CREATE UNIQUE INDEX idx_unique_active_email ON users(email) 
WHERE status = 'active';
```

#### 2.9 Covering Indexes (Index-Only Scans)
Covering indexes include all columns needed by a query, allowing PostgreSQL to satisfy the query entirely from the index without accessing the table.

```sql
-- Create covering index using INCLUDE
CREATE INDEX idx_orders_covering ON orders(customer_id) 
INCLUDE (order_date, total_amount);

-- This query can be answered entirely from the index:
SELECT order_date, total_amount FROM orders WHERE customer_id = 123;
```

### Index Maintenance and Monitoring

```sql
-- Check index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Find unused indexes (candidates for removal)
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelname NOT LIKE '%_pkey';

-- Check index bloat
SELECT 
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild a bloated index
REINDEX INDEX CONCURRENTLY idx_name;  -- Doesn't lock the table

-- Rebuild all indexes on a table
REINDEX TABLE CONCURRENTLY my_table;
```

### Index Best Practices

1. **Create indexes on foreign keys** - Essential for JOIN performance
2. **Use composite indexes wisely** - Order columns by selectivity (most selective first)
3. **Consider index size vs benefit** - Don't over-index; each index has a cost
4. **Use partial indexes for subsets** - More efficient than full table indexes
5. **Monitor and remove unused indexes** - They slow down writes without benefit
6. **Use INCLUDE for covering indexes** - Enable index-only scans
7. **REINDEX periodically** - Especially for heavily updated tables

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/indexes.html">PostgreSQL Indexes Documentation</a>
  - <a href="https://www.postgresql.org/docs/18/indexes-types.html">Index Types</a>
  - <a href="https://www.postgresql.org/docs/18/indexes-partial.html">Partial Indexes</a>

## 3. Write-Ahead Logging (WAL)

Write-Ahead Logging (WAL) is the foundation of data durability and crash recovery in PostgreSQL. The core principle is simple: all changes to data files must be logged before they are written to disk. This ensures that even if the system crashes, PostgreSQL can replay the log to recover to a consistent state.

### How WAL Works

1. **Change occurs**: When a transaction modifies data, the change is first written to the WAL buffer in memory
2. **WAL write**: At commit time (or when the buffer fills), WAL records are written to WAL files on disk
3. **Data file write**: Later, at checkpoints, the actual data pages are written to data files
4. **Crash recovery**: If a crash occurs, PostgreSQL replays WAL records since the last checkpoint to reconstruct the database state

This mechanism ensures **durability** (the "D" in ACID) - committed transactions are never lost, even in crashes.

### WAL Configuration

Key WAL-related parameters in `postgresql.conf`:

```conf
# --- WAL Settings ---

# WAL level determines how much information is written to WAL
# minimal: only crash recovery
# replica: supports WAL archiving and replication (default)
# logical: supports logical decoding/replication
wal_level = replica

# Number of 16MB WAL segment files
# Higher values reduce checkpoint frequency but use more disk space
# Example: 32 = 512MB of WAL
min_wal_size = 80MB
max_wal_size = 1GB

# WAL buffers in shared memory
# -1 means auto-tune to 1/32 of shared_buffers (up to 16MB)
wal_buffers = -1

# Commit delay for group commit optimization (microseconds)
# Allows multiple transactions to share fsync cost
commit_delay = 0
commit_siblings = 5

# --- WAL Archiving (for Point-in-Time Recovery) ---

# Enable continuous archiving
archive_mode = on

# Command to archive completed WAL segments
# %p = path of file to archive, %f = filename only
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'

# Alternative using rsync:
# archive_command = 'rsync -a %p backup_server:/wal_archive/%f'

# Timeout for archive command (0 = no timeout)
archive_timeout = 300  # Archive every 5 minutes even if not full

# --- Checkpoints ---

# Maximum time between automatic checkpoints
checkpoint_timeout = 5min

# Checkpoint completion target (fraction of checkpoint_timeout)
# 0.9 means spread checkpoint I/O over 90% of checkpoint interval
checkpoint_completion_target = 0.9

# Warning threshold for checkpoint frequency (seconds)
# Warns if checkpoints happen more frequently than this
checkpoint_warning = 30s
```

### Monitoring WAL

```sql
-- Check current WAL location
SELECT pg_current_wal_lsn();

-- Check current WAL insert location
SELECT pg_current_wal_insert_lsn();

-- Calculate WAL distance between two LSNs (in bytes)
SELECT pg_wal_lsn_diff('0/3000000', '0/2000000');

-- View WAL files in pg_wal directory
SELECT * FROM pg_ls_waldir() ORDER BY modification DESC LIMIT 10;

-- Check WAL archiving status
SELECT 
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;

-- Monitor checkpoint activity
SELECT 
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    maxwritten_clean,
    buffers_backend,
    buffers_backend_fsync
FROM pg_stat_bgwriter;

-- Check replication lag (on primary)
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

### WAL and Crash Recovery

When PostgreSQL starts after a crash, it automatically performs crash recovery:

1. **Reads last checkpoint location** from `pg_control` file
2. **Replays WAL records** from checkpoint to end of WAL
3. **Reconstructs database state** to last committed transaction
4. **Rolls back uncommitted transactions**

**Example recovery process:**

```bash
# After a crash, PostgreSQL logs show:
LOG:  database system was interrupted; last known up at 2024-01-15 10:30:45 UTC
LOG:  database system was not properly shut down; automatic recovery in progress
LOG:  redo starts at 0/1A2B3C4D
LOG:  invalid record length at 0/1B2C3D4E: wanted 24, got 0
LOG:  redo done at 0/1B2C3D4E
LOG:  last completed transaction was at log time 2024-01-15 10:35:22.123456+00
LOG:  database system is ready to accept connections
```

You can also perform manual recovery using archived WAL for Point-in-Time Recovery (PITR).

### WAL and Replication

WAL is the foundation for both physical and logical replication in PostgreSQL:

#### Physical Replication (WAL Streaming)
The primary server streams WAL records to standby servers in near real-time:

```sql
-- On primary: Create replication slot (prevents WAL deletion)
SELECT pg_create_physical_replication_slot('standby_slot');

-- On primary: Check replication status
SELECT * FROM pg_stat_replication;

-- On standby: View recovery status
SELECT pg_is_in_recovery();  -- Returns true on standby

SELECT 
    pg_last_wal_receive_lsn(),  -- Last WAL received from primary
    pg_last_wal_replay_lsn(),   -- Last WAL applied
    pg_last_xact_replay_timestamp();  -- Time of last replayed transaction
```

**Configuration for standby server (standby.signal file):**
```conf
# standby.signal file indicates this is a standby server
# (file presence, no content needed)

# In postgresql.conf:
primary_conninfo = 'host=primary_host port=5432 user=replicator password=secret'
primary_slot_name = 'standby_slot'
restore_command = 'cp /mnt/wal_archive/%f %p'
```

#### Logical Replication
Uses WAL to decode changes into logical replication messages:

```sql
-- Enable logical replication (set wal_level = logical)
-- Create publication on source database
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- Create subscription on target database
CREATE SUBSCRIPTION my_sub 
CONNECTION 'host=source_host dbname=sourcedb user=repl_user password=secret'
PUBLICATION my_pub;

-- Monitor logical replication
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_replication_slots;
```

### WAL Retention and Cleanup

WAL files accumulate over time. PostgreSQL manages them based on several factors:

```sql
-- Check replication slots (prevent WAL removal)
SELECT 
    slot_name,
    slot_type,
    database,
    active,
    restart_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes
FROM pg_replication_slots;

-- Remove unused replication slot
SELECT pg_drop_replication_slot('old_slot_name');

-- Manually switch to new WAL file (useful before backup)
SELECT pg_switch_wal();

-- Force checkpoint (triggers WAL file cleanup)
CHECKPOINT;
```

**WAL cleanup rules:**
- Keeps WAL needed for replication (based on replication slots)
- Keeps WAL for archiving (if archive_mode is on)
- Removes old WAL after successful archiving and when beyond `min_wal_size`

### Best Practices

1. **Set appropriate wal_level**: Use `replica` for most cases, `logical` only if needed
2. **Configure archiving for production**: Essential for Point-in-Time Recovery
3. **Monitor archive failures**: Unarchived WAL can fill disk space
4. **Use replication slots carefully**: They prevent WAL removal; drop unused slots
5. **Tune checkpoint frequency**: Balance I/O smoothness vs recovery time
6. **Keep WAL on fast storage**: WAL writes are synchronous and latency-sensitive
7. **Monitor replication lag**: Use `pg_stat_replication` to track standby servers

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/wal.html">PostgreSQL WAL Documentation</a>
  - <a href="https://www.postgresql.org/docs/18/wal-configuration.html">WAL Configuration</a>
  - <a href="https://www.postgresql.org/docs/18/continuous-archiving.html">Continuous Archiving and PITR</a>

## 4. Caching

PostgreSQL's caching system is critical for performance. It works at multiple levels to minimize expensive disk I/O operations. Understanding and properly configuring these caches can dramatically improve query performance.

### Shared Buffers

Shared buffers is PostgreSQL's main cache for data pages. It's shared memory where PostgreSQL caches table and index blocks.

#### Configuration

```conf
# In postgresql.conf

# Shared buffers - main data cache
# Default: 128MB
# Recommended: 25% of total RAM (for dedicated database server)
# Maximum practical: ~40% of RAM (beyond this, OS cache is more efficient)
shared_buffers = 4GB

# Example sizing:
# 8GB RAM  -> shared_buffers = 2GB
# 32GB RAM -> shared_buffers = 8GB
# 64GB RAM -> shared_buffers = 16GB
```

**How it works:**
1. When PostgreSQL needs a data page, it first checks shared buffers
2. If found (cache hit), data is returned immediately
3. If not found (cache miss), page is loaded from disk into shared buffers
4. Least Recently Used (LRU) algorithm evicts old pages when buffer is full

#### Monitoring Shared Buffers

```sql
-- Check shared buffer settings
SHOW shared_buffers;

-- View buffer cache hit ratio (should be > 99% for steady-state workload)
SELECT 
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100 AS cache_hit_ratio
FROM pg_statio_user_tables;

-- Per-table cache hit ratio
SELECT 
    schemaname,
    relname,
    heap_blks_read,
    heap_blks_hit,
    CASE 
        WHEN heap_blks_hit + heap_blks_read = 0 THEN 0
        ELSE round(100.0 * heap_blks_hit / (heap_blks_hit + heap_blks_read), 2)
    END AS cache_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY cache_hit_ratio ASC
LIMIT 20;

-- Check what's in shared buffers (requires pg_buffercache extension)
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

SELECT 
    c.relname,
    count(*) AS buffers,
    pg_size_pretty(count(*) * 8192) AS size_in_cache
FROM pg_buffercache b
INNER JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 20;

-- Check buffer usage by database
SELECT 
    CASE 
        WHEN d.datname IS NULL THEN 'shared objects'
        ELSE d.datname 
    END AS database,
    count(*) AS buffers,
    pg_size_pretty(count(*) * 8192) AS size_in_cache
FROM pg_buffercache b
LEFT JOIN pg_database d ON b.reldatabase = d.oid
GROUP BY d.datname
ORDER BY count(*) DESC;
```

### Effective Cache Size

`effective_cache_size` is a planning parameter that tells PostgreSQL how much memory is available for caching data (both PostgreSQL's shared buffers and OS filesystem cache).

```conf
# In postgresql.conf

# Effective cache size - used by query planner
# Recommended: 50-75% of total RAM
# This is NOT allocated memory, just a hint to the planner
effective_cache_size = 12GB

# Example sizing:
# 16GB RAM -> effective_cache_size = 12GB
# 32GB RAM -> effective_cache_size = 24GB
# 64GB RAM -> effective_cache_size = 48GB
```

**Important:** This doesn't allocate memory; it just tells the planner how aggressive to be with index scans vs sequential scans.

### Work Memory

`work_mem` specifies the amount of memory to use for internal sort operations and hash tables before writing to temporary disk files.

```conf
# In postgresql.conf

# Work memory per operation
# Default: 4MB (often too small!)
# Recommended: Start with 16-64MB, tune based on workload
work_mem = 32MB

# Note: This is PER OPERATION, and a complex query might use multiple operations
# Example: A query with 3 sorts could use up to 3 * work_mem
# Total memory usage = connections * work_mem * operations_per_query
```

**Monitor and tune:**
```sql
-- Check current setting
SHOW work_mem;

-- Set for current session (useful for specific heavy queries)
SET work_mem = '256MB';

-- Find queries writing to temp files (sign that work_mem is too low)
SELECT 
    query,
    temp_blks_written,
    temp_blks_read
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 10;
```

### Maintenance Work Memory

`maintenance_work_mem` is used for maintenance operations like VACUUM, CREATE INDEX, and ALTER TABLE.

```conf
# In postgresql.conf

# Maintenance work memory
# Default: 64MB
# Recommended: 256MB to 1GB (or higher for large databases)
maintenance_work_mem = 512MB

# This affects:
# - VACUUM performance
# - Index creation speed
# - Foreign key checking speed
```

**Usage:**
```sql
-- Set higher for a specific maintenance operation
SET maintenance_work_mem = '2GB';
CREATE INDEX idx_large_table ON large_table(column);
RESET maintenance_work_mem;
```

### Operating System Cache

PostgreSQL also benefits from the operating system's filesystem cache. Data that doesn't fit in shared buffers may still be cached by the OS, avoiding physical disk reads.

**Best practice:** On a dedicated database server, configure shared_buffers to use ~25% of RAM, leaving the rest for OS cache, work_mem, and other operations.

### Query Result Caching

Note: PostgreSQL doesn't have a built-in query result cache like some other databases (e.g., MySQL query cache). However, there are strategies:

1. **Application-level caching**: Use Redis, Memcached, or application memory
2. **Materialized views**: Pre-compute expensive queries

```sql
-- Create materialized view for expensive aggregation
CREATE MATERIALIZED VIEW sales_summary AS
SELECT 
    DATE_TRUNC('day', order_date) AS day,
    product_id,
    sum(quantity) AS total_quantity,
    sum(amount) AS total_amount
FROM orders
GROUP BY DATE_TRUNC('day', order_date), product_id;

-- Create index on materialized view
CREATE INDEX idx_sales_summary_day ON sales_summary(day);

-- Refresh materialized view periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;  -- CONCURRENTLY doesn't block reads

-- Query the cached results
SELECT * FROM sales_summary WHERE day = '2024-01-15';
```

3. **Prepared statements**: PostgreSQL caches query plans for prepared statements

```sql
-- Prepare a statement (plan is cached)
PREPARE get_user AS SELECT * FROM users WHERE id = $1;

-- Execute multiple times (reuses cached plan)
EXECUTE get_user(123);
EXECUTE get_user(456);

-- Deallocate when done
DEALLOCATE get_user;
```

### Table and Index Bloat (Affects Cache Efficiency)

Bloated tables and indexes reduce cache efficiency because more pages are needed to store the same data.

```sql
-- Check table bloat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS external_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

-- Solutions for bloat:
-- 1. Regular VACUUM (automatic with autovacuum)
-- 2. VACUUM FULL (locks table, use with caution)
-- 3. pg_repack extension (online repack without locks)
```

### Cache Warming

After a restart, caches are cold. You can pre-warm them:

```sql
-- Pre-load specific tables into shared buffers (requires pg_prewarm extension)
CREATE EXTENSION IF NOT EXISTS pg_prewarm;

-- Load entire table into cache
SELECT pg_prewarm('users');

-- Load specific pages
SELECT pg_prewarm('large_table', 'buffer', 'main', 0, 1000);  -- First 1000 blocks

-- Automate cache warming on startup:
-- Add to postgresql.conf:
-- shared_preload_libraries = 'pg_prewarm'
-- This will automatically reload tables that were in cache before shutdown
```

### Best Practices

1. **Set shared_buffers to ~25% of RAM** for dedicated database servers
2. **Set effective_cache_size to ~75% of RAM** to help query planner
3. **Monitor cache hit ratios** - aim for >99% for steady-state workloads
4. **Tune work_mem carefully** - balance per-query performance vs memory usage
5. **Use maintenance_work_mem generously** for faster maintenance operations
6. **Keep tables and indexes compact** with regular VACUUM to improve cache efficiency
7. **Use materialized views** for frequently-accessed expensive queries
8. **Monitor temp file usage** to identify queries needing more work_mem

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/runtime-config-resource.html">PostgreSQL Resource Configuration</a>
  - <a href="https://www.postgresql.org/docs/18/pgbuffercache.html">pg_buffercache Extension</a>
  - <a href="https://www.postgresql.org/docs/18/pgprewarm.html">pg_prewarm Extension</a>

## 5. Transactions

Transactions are fundamental to database integrity. They group multiple operations into a single atomic unit that either completely succeeds or completely fails. PostgreSQL provides robust transaction support with full ACID compliance.

### ACID Properties

PostgreSQL transactions guarantee ACID properties:

#### **A - Atomicity**
All operations in a transaction succeed together or fail together. No partial updates.

```sql
-- Example: Transfer money between accounts (atomic operation)
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

-- If either UPDATE fails, ROLLBACK undoes both
-- If both succeed, COMMIT makes changes permanent
COMMIT;
```

#### **C - Consistency**
Transactions move the database from one valid state to another. All constraints and rules are enforced.

```sql
-- Constraints ensure consistency
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    total_amount DECIMAL(10,2) CHECK (total_amount >= 0),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

BEGIN;
-- This will fail if customer doesn't exist (foreign key constraint)
-- or if total_amount is negative (check constraint)
INSERT INTO orders (customer_id, total_amount) VALUES (999, -50);
ROLLBACK;  -- Transaction is rolled back, consistency maintained
```

#### **I - Isolation**
Concurrent transactions don't interfere with each other. PostgreSQL uses MVCC (Multi-Version Concurrency Control).

```sql
-- Even with concurrent transactions, each sees a consistent snapshot
-- Transaction 1
BEGIN;
SELECT balance FROM accounts WHERE account_id = 1;  -- Sees: 1000
-- ... time passes, Transaction 2 updates this account ...
SELECT balance FROM accounts WHERE account_id = 1;  -- Still sees: 1000 (snapshot isolation)
COMMIT;
```

#### **D - Durability**
Once committed, changes are permanent and survive crashes. Guaranteed by WAL (Write-Ahead Logging).

```sql
BEGIN;
INSERT INTO important_data (value) VALUES ('critical');
COMMIT;  -- After COMMIT returns, data is durable (even if server crashes immediately)
```

### Transaction Control

```sql
-- Start a transaction
BEGIN;  -- or START TRANSACTION;

-- Perform operations
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;

-- Commit the transaction (make changes permanent)
COMMIT;

-- Or rollback (undo all changes)
ROLLBACK;

-- Savepoints (partial rollback within a transaction)
BEGIN;

INSERT INTO logs (message) VALUES ('Step 1');
SAVEPOINT step1;

INSERT INTO logs (message) VALUES ('Step 2');
SAVEPOINT step2;

INSERT INTO logs (message) VALUES ('Step 3');

-- Rollback to savepoint (undoes Step 3 only)
ROLLBACK TO SAVEPOINT step2;

-- Continue transaction
INSERT INTO logs (message) VALUES ('Step 3 corrected');

COMMIT;  -- Commits Step 1, Step 2, and Step 3 corrected
```

### Isolation Levels

PostgreSQL supports four SQL standard isolation levels. Each level makes different trade-offs between consistency and concurrency.

#### 1. Read Uncommitted (Not truly implemented in PostgreSQL)
PostgreSQL treats this as Read Committed.

#### 2. Read Committed (Default)
Sees only data committed before the query begins. Most common isolation level.

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

BEGIN;
SELECT balance FROM accounts WHERE account_id = 1;  -- Sees: 1000

-- Meanwhile, another transaction commits: UPDATE accounts SET balance = 1500 WHERE account_id = 1

SELECT balance FROM accounts WHERE account_id = 1;  -- Now sees: 1500 (updated value)
COMMIT;
```

**Behavior:**
- Each query sees a snapshot as of its start time
- Different queries in the same transaction may see different data
- Prevents dirty reads (reading uncommitted data)
- Allows non-repeatable reads and phantom reads

**Use when:** Default choice for most applications. Good balance of consistency and concurrency.

#### 3. Repeatable Read
Sees a snapshot of the database as of the transaction start. Same data throughout the transaction.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN;
SELECT balance FROM accounts WHERE account_id = 1;  -- Sees: 1000

-- Meanwhile, another transaction commits: UPDATE accounts SET balance = 1500 WHERE account_id = 1

SELECT balance FROM accounts WHERE account_id = 1;  -- Still sees: 1000 (snapshot isolation)
COMMIT;
```

**Behavior:**
- All queries in the transaction see the same snapshot
- Prevents dirty reads and non-repeatable reads
- Prevents phantom reads in PostgreSQL (stricter than SQL standard)
- May cause serialization failures with concurrent updates

**Use when:** Need consistent reads throughout a transaction (e.g., generating reports, batch processing).

**Handling serialization failures:**
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN
    -- Attempt update
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
    
    -- If concurrent modification occurred, PostgreSQL raises error:
    -- ERROR: could not serialize access due to concurrent update
    
    COMMIT;
EXCEPTION
    WHEN serialization_failure THEN
        ROLLBACK;
        -- Retry transaction
END;
```

#### 4. Serializable
Strongest isolation level. Guarantees transactions execute as if they ran serially (one after another).

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

BEGIN;
-- Transaction logic
SELECT sum(amount) FROM orders WHERE customer_id = 1;
INSERT INTO order_summary (customer_id, total) VALUES (1, 5000);
COMMIT;

-- If concurrent transactions would create an anomaly,
-- PostgreSQL aborts one with:
-- ERROR: could not serialize access due to read/write dependencies among transactions
```

**Behavior:**
- Guarantees serializable execution
- May abort transactions more frequently than Repeatable Read
- Requires application retry logic
- Uses Serializable Snapshot Isolation (SSI) - efficient implementation

**Use when:** Financial transactions, inventory systems, or any case requiring absolute consistency.

**Example serializable transaction with retry:**
```sql
CREATE OR REPLACE FUNCTION transfer_money(from_account INT, to_account INT, amount DECIMAL)
RETURNS BOOLEAN AS $$
DECLARE
    retry_count INT := 0;
    max_retries INT := 5;
BEGIN
    LOOP
        BEGIN
            -- Set serializable isolation
            SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
            
            -- Check sufficient balance
            IF (SELECT balance FROM accounts WHERE id = from_account) < amount THEN
                RAISE EXCEPTION 'Insufficient funds';
            END IF;
            
            -- Perform transfer
            UPDATE accounts SET balance = balance - amount WHERE id = from_account;
            UPDATE accounts SET balance = balance + amount WHERE id = to_account;
            
            -- Success
            RETURN TRUE;
            
        EXCEPTION
            WHEN serialization_failure OR deadlock_detected THEN
                retry_count := retry_count + 1;
                IF retry_count >= max_retries THEN
                    RAISE EXCEPTION 'Transaction failed after % retries', max_retries;
                END IF;
                -- Retry after brief delay
                PERFORM pg_sleep(0.1 * retry_count);
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### Transaction Best Practices

#### Keep Transactions Short
```sql
-- BAD: Long-running transaction holds locks
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
-- ... complex processing, external API calls, user interaction ...
COMMIT;

-- GOOD: Minimize transaction time
-- Do processing outside transaction
-- ... complex processing, external API calls ...
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
COMMIT;
```

#### Explicit Locking When Needed
```sql
-- Lock row for update (prevents concurrent modifications)
BEGIN;
SELECT * FROM inventory WHERE product_id = 123 FOR UPDATE;

-- Now safe to read quantity, compute new value, and update
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 123;
COMMIT;

-- Lock with NOWAIT (fails immediately if row is locked)
SELECT * FROM inventory WHERE product_id = 123 FOR UPDATE NOWAIT;

-- Lock with SKIP LOCKED (skips locked rows, useful for queue processing)
SELECT * FROM job_queue WHERE status = 'pending' 
ORDER BY priority DESC 
LIMIT 1 
FOR UPDATE SKIP LOCKED;
```

#### Monitor Long Transactions
```sql
-- Find long-running transactions
SELECT 
    pid,
    now() - xact_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY duration DESC
LIMIT 10;

-- Kill a long-running transaction
SELECT pg_terminate_backend(pid);

-- Cancel a query (less aggressive than terminate)
SELECT pg_cancel_backend(pid);
```

#### Transaction Timeouts
```conf
# In postgresql.conf or set per-session

# Maximum time a transaction can be idle
idle_in_transaction_session_timeout = 600000  # 10 minutes

# Maximum time a statement can run
statement_timeout = 30000  # 30 seconds

# Maximum time a transaction can run
transaction_timeout = 3600000  # 1 hour (PostgreSQL 14+)
```

### Practical Examples

#### Example 1: E-commerce Order Processing
```sql
BEGIN;

-- Create order
INSERT INTO orders (customer_id, total_amount) 
VALUES (123, 99.99) 
RETURNING order_id INTO v_order_id;

-- Add order items
INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES 
    (v_order_id, 1, 2, 29.99),
    (v_order_id, 2, 1, 39.99);

-- Update inventory
UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 1;
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 2;

-- Create invoice
INSERT INTO invoices (order_id, amount, status)
VALUES (v_order_id, 99.99, 'pending');

COMMIT;
```

#### Example 2: Batch Processing with Savepoints
```sql
BEGIN;

FOR record IN SELECT * FROM staging_data LOOP
    BEGIN
        SAVEPOINT process_record;
        
        -- Process record
        INSERT INTO production_data (data) VALUES (record.data);
        
        -- Mark as processed
        UPDATE staging_data SET processed = TRUE WHERE id = record.id;
        
    EXCEPTION
        WHEN OTHERS THEN
            -- Rollback this record only, continue with others
            ROLLBACK TO SAVEPOINT process_record;
            
            -- Log error
            INSERT INTO error_log (record_id, error_message)
            VALUES (record.id, SQLERRM);
    END;
END LOOP;

COMMIT;
```

### Monitoring Transactions

```sql
-- Current transaction activity
SELECT 
    pid,
    usename,
    application_name,
    state,
    xact_start,
    query_start,
    state_change,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start;

-- Transaction ID information
SELECT 
    txid_current(),              -- Current transaction ID
    txid_current_if_assigned(),  -- Current ID (or NULL if not assigned)
    txid_snapshot_xmin(txid_current_snapshot()),  -- Oldest active transaction
    txid_snapshot_xmax(txid_current_snapshot());  -- Next transaction ID to be assigned
```

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/tutorial-transactions.html">PostgreSQL Transactions Tutorial</a>
  - <a href="https://www.postgresql.org/docs/18/transaction-iso.html">Transaction Isolation Levels</a>
  - <a href="https://www.postgresql.org/docs/18/mvcc.html">MVCC (Multi-Version Concurrency Control)</a>

## 6. Large Object Storage
PostgreSQL supports storing and manipulating large objects using the `lo` module.

- **Example**: `SELECT lo_open(large_object_id, 131072);`
- **Further Reading**: [PostgreSQL Large Object Documentation](https://www.postgresql.org/docs/18/largeobjects.html)

## 7. Postmaster Process
The Postmaster is the primary process for a PostgreSQL database cluster responsible for managing the database connections.

- **Example**: You can check the active database connections with `SELECT * FROM pg_stat_activity;`
- **Further Reading**: [PostgreSQL Postmaster Documentation](https://www.postgresql.org/docs/18/architecture.html#architecture-processes)

## 8. ID Wraparound
To prevent data loss, PostgreSQL requires regular vacuums to manage the transaction ID wraparound.

- **Example**: Check for warnings using the command `VACUUM FREEZE;`
- **Further Reading**: [PostgreSQL ID Wraparound Documentation](https://www.postgresql.org/docs/18/routine-vacuuming.html#VACUUM-FREEZE)

## 9. Explain Plans
The `EXPLAIN` command provides insight into how PostgreSQL executes queries.

- **Example**: `EXPLAIN SELECT * FROM my_table;`
- **Further Reading**: [PostgreSQL EXPLAIN Documentation](https://www.postgresql.org/docs/18/sql-explain.html)

## 10. Deadlocks
Deadlocks occur when two transactions block each other. PostgreSQL automatically detects and resolves deadlocks.

- **Example**: Configure database settings to minimize deadlocks.
- **Further Reading**: [PostgreSQL Deadlocks Documentation](https://www.postgresql.org/docs/18/locking-issues.html)

## 11. Metadata Using pg_class and pg_stat
The `pg_class` and `pg_stat` system catalogs provide useful metadata about tables and their states.

- **Example**: `SELECT * FROM pg_class WHERE relname = 'my_table';`
- **Further Reading**: [PostgreSQL pg_class Documentation](https://www.postgresql.org/docs/18/catalog-pg-class.html)

## 12. Replication Types
PostgreSQL supports both synchronous and asynchronous replication for high availability.

- **Further Reading**: [PostgreSQL Replication Documentation](https://www.postgresql.org/docs/18/warm-standby.html)

## Conclusion
This guide provides fundamental knowledge about PostgreSQL, serving as a starting point for further exploration. For more advanced topics, refer to the official PostgreSQL documentation and community resources.