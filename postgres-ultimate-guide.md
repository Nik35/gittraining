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

PostgreSQL provides two mechanisms for storing large binary data: **Large Objects** (using the `pg_largeobject` system table) and **BYTEA columns**. Large Objects are designed for files that exceed the maximum size of a BYTEA column (1GB) or when you need streaming access to binary data.

### Large Objects vs BYTEA

| Feature | Large Objects | BYTEA Column |
|---------|--------------|--------------|
| Maximum size | Unlimited (up to 4TB per object) | 1GB per value |
| Storage | Stored in `pg_largeobject` table | Stored inline in table |
| Access | Stream-based (chunks) | All-at-once |
| Transactions | Separate transaction handling | Standard transactional |
| Backup | Requires special handling | Standard pg_dump |
| Performance | Better for very large files | Better for smaller files (<1MB) |

### Using Large Objects

#### Creating and Storing Large Objects

```sql
-- Method 1: Using lo_import (from file on server)
SELECT lo_import('/path/to/file.pdf');  -- Returns OID
-- Returns: 16385 (example OID)

-- Method 2: Using lo_create and lo_put
SELECT lo_create(0);  -- Creates empty large object, returns OID
-- Returns: 16386

-- Method 3: From application (using libpq functions)
-- In application code, use lo_creat(), lo_open(), lo_write()

-- Store the OID in a table
CREATE TABLE documents (
    doc_id SERIAL PRIMARY KEY,
    filename VARCHAR(255),
    content_type VARCHAR(100),
    file_size BIGINT,
    large_object_id OID,  -- Reference to large object
    uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert reference
INSERT INTO documents (filename, content_type, large_object_id)
VALUES ('report.pdf', 'application/pdf', 16385);
```

#### Reading Large Objects

```sql
-- Method 1: Using lo_export (to file on server)
SELECT lo_export(16385, '/tmp/exported_file.pdf');

-- Method 2: Using lo_get (read entire content)
SELECT lo_get(16385);  -- Returns bytea

-- Method 3: Stream-based reading (in application)
-- Open for reading (mode: 262144 = INV_READ)
SELECT lo_open(16385, 262144);  -- Returns file descriptor
-- Returns: 0 (file descriptor)

-- Read chunk (in application using libpq)
-- lo_read(fd, buffer, length)

-- Close (in application)
-- lo_close(fd)

-- Example: Retrieve document content
SELECT 
    filename,
    content_type,
    lo_get(large_object_id) AS content
FROM documents
WHERE doc_id = 1;
```

#### Updating Large Objects

```sql
-- Open for read/write (mode: 393216 = INV_READ | INV_WRITE)
SELECT lo_open(16385, 393216);

-- Seek to position (in application)
-- lo_lseek(fd, offset, whence)

-- Write data (in application)
-- lo_write(fd, buffer, length)

-- Truncate large object
SELECT lo_truncate(16385, 1024);  -- Truncate to 1024 bytes

-- Close
-- lo_close(fd)
```

#### Deleting Large Objects

```sql
-- Delete a large object
SELECT lo_unlink(16385);

-- Important: Delete from your reference table AND the large object
BEGIN;
-- Get the OID first
SELECT large_object_id INTO v_oid FROM documents WHERE doc_id = 1;

-- Delete the reference
DELETE FROM documents WHERE doc_id = 1;

-- Delete the large object
SELECT lo_unlink(v_oid);
COMMIT;

-- Bulk cleanup: Delete orphaned large objects
SELECT lo_unlink(l.oid)
FROM pg_largeobject_metadata l
LEFT JOIN documents d ON d.large_object_id = l.oid
WHERE d.large_object_id IS NULL;
```

### The pg_largeobject System Table

Large objects are stored in the `pg_largeobject` system table, which stores data in 2KB chunks.

```sql
-- View large object metadata
SELECT * FROM pg_largeobject_metadata;
-- Columns: oid, lomowner, lomacl

-- View large object data (stored in chunks)
SELECT * FROM pg_largeobject WHERE loid = 16385;
-- Columns: loid (large object ID), pageno (chunk number), data (bytea chunk)

-- Check large object size
SELECT 
    loid,
    count(*) AS num_pages,
    count(*) * 2048 AS approx_bytes,
    pg_size_pretty(count(*) * 2048) AS approx_size
FROM pg_largeobject
GROUP BY loid
ORDER BY count(*) DESC;

-- List all large objects with ownership
SELECT 
    lom.oid,
    lom.lomowner,
    pg_get_userbyid(lom.lomowner) AS owner_name,
    count(lo.pageno) AS pages,
    pg_size_pretty(count(lo.pageno) * 2048) AS size
FROM pg_largeobject_metadata lom
LEFT JOIN pg_largeobject lo ON lom.oid = lo.loid
GROUP BY lom.oid, lom.lomowner
ORDER BY count(lo.pageno) DESC;
```

### Large Object Permissions

```sql
-- Grant permissions on a large object
GRANT SELECT ON LARGE OBJECT 16385 TO user_readonly;
GRANT UPDATE ON LARGE OBJECT 16385 TO user_readwrite;

-- Revoke permissions
REVOKE ALL ON LARGE OBJECT 16385 FROM user_readonly;

-- Check permissions
SELECT 
    oid,
    lomowner,
    lomacl
FROM pg_largeobject_metadata
WHERE oid = 16385;
```

### Practical Example: Document Management System

```sql
-- Create document table with large object references
CREATE TABLE documents (
    doc_id SERIAL PRIMARY KEY,
    filename VARCHAR(255) NOT NULL,
    content_type VARCHAR(100),
    file_size BIGINT,
    large_object_id OID NOT NULL,
    description TEXT,
    uploaded_by INTEGER REFERENCES users(user_id),
    uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    version INTEGER DEFAULT 1
);

-- Create index on large_object_id for faster lookups
CREATE INDEX idx_documents_loid ON documents(large_object_id);

-- Function to safely insert a document
CREATE OR REPLACE FUNCTION insert_document(
    p_filename VARCHAR,
    p_content_type VARCHAR,
    p_data BYTEA,
    p_uploaded_by INTEGER
) RETURNS INTEGER AS $$
DECLARE
    v_oid OID;
    v_fd INTEGER;
    v_doc_id INTEGER;
BEGIN
    -- Create large object
    v_oid := lo_create(0);
    
    -- Open for writing
    v_fd := lo_open(v_oid, 131072);  -- INV_WRITE
    
    -- Write data
    PERFORM lo_write(v_fd, p_data);
    
    -- Close
    PERFORM lo_close(v_fd);
    
    -- Insert metadata
    INSERT INTO documents (filename, content_type, file_size, large_object_id, uploaded_by)
    VALUES (p_filename, p_content_type, length(p_data), v_oid, p_uploaded_by)
    RETURNING doc_id INTO v_doc_id;
    
    RETURN v_doc_id;
END;
$$ LANGUAGE plpgsql;

-- Function to safely delete a document
CREATE OR REPLACE FUNCTION delete_document(p_doc_id INTEGER) RETURNS BOOLEAN AS $$
DECLARE
    v_oid OID;
BEGIN
    -- Get the large object ID
    SELECT large_object_id INTO v_oid FROM documents WHERE doc_id = p_doc_id;
    
    IF NOT FOUND THEN
        RETURN FALSE;
    END IF;
    
    -- Delete the document record
    DELETE FROM documents WHERE doc_id = p_doc_id;
    
    -- Delete the large object
    PERFORM lo_unlink(v_oid);
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Function to retrieve document content
CREATE OR REPLACE FUNCTION get_document_content(p_doc_id INTEGER) RETURNS BYTEA AS $$
DECLARE
    v_oid OID;
    v_data BYTEA;
BEGIN
    SELECT large_object_id INTO v_oid FROM documents WHERE doc_id = p_doc_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Document not found';
    END IF;
    
    -- Read the entire large object
    SELECT lo_get(v_oid) INTO v_data;
    
    RETURN v_data;
END;
$$ LANGUAGE plpgsql;
```

### Maintenance and Monitoring

```sql
-- Find orphaned large objects (not referenced by any table)
-- This query checks the 'documents' table; adjust for your schema
SELECT l.oid, pg_size_pretty(count(*) * 2048) AS size
FROM pg_largeobject_metadata l
WHERE NOT EXISTS (
    SELECT 1 FROM documents WHERE large_object_id = l.oid
)
GROUP BY l.oid;

-- Cleanup orphaned large objects
-- Create a function for safety
CREATE OR REPLACE FUNCTION cleanup_orphaned_large_objects() RETURNS INTEGER AS $$
DECLARE
    v_count INTEGER := 0;
    v_oid OID;
BEGIN
    FOR v_oid IN 
        SELECT l.oid
        FROM pg_largeobject_metadata l
        WHERE NOT EXISTS (
            SELECT 1 FROM documents WHERE large_object_id = l.oid
        )
    LOOP
        PERFORM lo_unlink(v_oid);
        v_count := v_count + 1;
    END LOOP;
    
    RETURN v_count;
END;
$$ LANGUAGE plpgsql;

-- Run cleanup
SELECT cleanup_orphaned_large_objects();

-- Monitor large object storage usage
SELECT 
    'Total Large Objects' AS metric,
    count(*) AS count,
    pg_size_pretty(sum(count(*) * 2048)) AS total_size
FROM pg_largeobject
GROUP BY loid;

-- Check large objects by owner
SELECT 
    pg_get_userbyid(lomowner) AS owner,
    count(*) AS num_objects,
    pg_size_pretty(sum(pages * 2048)) AS total_size
FROM pg_largeobject_metadata lom
LEFT JOIN (
    SELECT loid, count(*) AS pages FROM pg_largeobject GROUP BY loid
) lo ON lom.oid = lo.loid
GROUP BY lomowner
ORDER BY sum(pages * 2048) DESC;
```

### Backup and Restore

```bash
# Large objects require special handling in backups

# Backup with large objects (pg_dump automatically includes them)
pg_dump -U postgres -d mydb -F c -b -f mydb_backup.dump

# -b or --blobs: Include large objects in the dump
# -F c: Custom format (recommended)

# Restore with large objects
pg_restore -U postgres -d mydb_restored -F c mydb_backup.dump

# For plain SQL format:
pg_dump -U postgres -d mydb -b -f mydb_backup.sql

# Note: Large objects are included automatically in pg_dump 
# unless you specifically exclude them with --no-blobs
```

### Best Practices

1. **Always use transactions** when creating/deleting large objects and their references
2. **Cleanup orphaned large objects** regularly to prevent storage waste
3. **Consider BYTEA for small files** (<1MB) for simpler management
4. **Use TOAST for medium files** (1MB-1GB) with BYTEA columns
5. **Use Large Objects for very large files** (>1GB) or streaming access
6. **Implement proper cleanup triggers** to automatically delete large objects when references are deleted
7. **Monitor storage usage** regularly
8. **Include `-b` flag in backups** to ensure large objects are backed up

### Automatic Cleanup with Triggers

```sql
-- Trigger to automatically delete large object when document is deleted
CREATE OR REPLACE FUNCTION delete_document_large_object() RETURNS TRIGGER AS $$
BEGIN
    PERFORM lo_unlink(OLD.large_object_id);
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_delete_document_large_object
BEFORE DELETE ON documents
FOR EACH ROW
EXECUTE FUNCTION delete_document_large_object();

-- Trigger to delete old large object when updating reference
CREATE OR REPLACE FUNCTION update_document_large_object() RETURNS TRIGGER AS $$
BEGIN
    IF OLD.large_object_id != NEW.large_object_id THEN
        PERFORM lo_unlink(OLD.large_object_id);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_document_large_object
BEFORE UPDATE ON documents
FOR EACH ROW
WHEN (OLD.large_object_id IS DISTINCT FROM NEW.large_object_id)
EXECUTE FUNCTION update_document_large_object();
```

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/largeobjects.html">PostgreSQL Large Objects Documentation</a>
  - <a href="https://www.postgresql.org/docs/18/lo-funcs.html">Large Object Functions</a>
  - <a href="https://www.postgresql.org/docs/18/catalog-pg-largeobject.html">pg_largeobject System Catalog</a>

## 7. Postmaster Process

The **Postmaster** is the first process started when PostgreSQL is launched. It's the master control process responsible for managing all other PostgreSQL processes, handling connection requests, and maintaining the overall health of the database cluster.

### Role and Responsibilities

The Postmaster process:
1. **Listens for connections** on the configured port (default: 5432)
2. **Spawns backend processes** for each client connection
3. **Manages background worker processes** (autovacuum, WAL writer, checkpointer, etc.)
4. **Monitors child processes** and performs cleanup if they crash
5. **Handles shutdown** and startup of the database cluster
6. **Enforces connection limits** and authentication

### Process Architecture

```
Postmaster (Main Process)
    ├── Backend Process 1 (Client connection 1)
    ├── Backend Process 2 (Client connection 2)
    ├── Backend Process N (Client connection N)
    ├── Background Writer
    ├── WAL Writer
    ├── Checkpointer
    ├── Autovacuum Launcher
    │   ├── Autovacuum Worker 1
    │   ├── Autovacuum Worker 2
    │   └── Autovacuum Worker N
    ├── WAL Sender (for replication)
    ├── WAL Receiver (on standby)
    ├── Stats Collector
    └── Logical Replication Workers
```

### Starting the Postmaster

```bash
# Method 1: Using pg_ctl (recommended)
pg_ctl -D /var/lib/postgresql/data start

# Method 2: Direct invocation
postgres -D /var/lib/postgresql/data

# Method 3: Using system service (systemd)
systemctl start postgresql

# Check if postmaster is running
pg_isready -h localhost -p 5432

# Or check process list
ps aux | grep postgres
```

**Example output:**
```
postgres   1234  0.0  0.1  ... postgres -D /var/lib/postgresql/data
postgres   1235  0.0  0.0  ... postgres: checkpointer
postgres   1236  0.0  0.0  ... postgres: background writer
postgres   1237  0.0  0.0  ... postgres: walwriter
postgres   1238  0.0  0.0  ... postgres: autovacuum launcher
postgres   1239  0.0  0.0  ... postgres: stats collector
postgres   1240  0.0  0.0  ... postgres: logical replication launcher
postgres   1241  0.0  0.1  ... postgres: user dbname [local] idle
```

### Connection Management

#### Connection Process Flow

1. **Client initiates connection** to port 5432
2. **Postmaster receives connection request**
3. **Postmaster authenticates client** (using pg_hba.conf rules)
4. **Postmaster forks a new backend process** dedicated to this client
5. **Backend process handles all queries** from this client
6. **Connection ends**: Backend process terminates, Postmaster cleans up

#### Connection Configuration

```conf
# In postgresql.conf

# Maximum number of connections
max_connections = 100

# Reserved connections for superusers
superuser_reserved_connections = 3

# Maximum time to complete connection (seconds)
authentication_timeout = 60

# TCP/IP settings
listen_addresses = '*'              # Listen on all interfaces
port = 5432                         # Default port

# Connection limits per user/database
# Set in CREATE ROLE or ALTER ROLE:
# ALTER ROLE username CONNECTION LIMIT 10;
```

#### Monitoring Connections

```sql
-- View all active connections
SELECT 
    pid,                          -- Process ID
    usename,                      -- User name
    application_name,             -- Application name
    client_addr,                  -- Client IP address
    client_hostname,              -- Client hostname
    backend_start,                -- Connection start time
    state,                        -- Connection state
    wait_event_type,              -- What the process is waiting for
    wait_event,                   -- Specific wait event
    query                         -- Current/last query
FROM pg_stat_activity
ORDER BY backend_start;

-- Count connections by state
SELECT 
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;

-- Count connections by user
SELECT 
    usename,
    count(*) AS connections
FROM pg_stat_activity
WHERE usename IS NOT NULL
GROUP BY usename
ORDER BY count(*) DESC;

-- Count connections by database
SELECT 
    datname,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY datname
ORDER BY count(*) DESC;

-- Check if approaching max_connections
SELECT 
    count(*) AS current_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections,
    count(*) * 100.0 / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS percent_used
FROM pg_stat_activity;
```

#### Terminating Connections

```sql
-- Cancel a query (gentle, lets transaction rollback cleanly)
SELECT pg_cancel_backend(12345);  -- Replace with actual PID

-- Terminate a connection (forceful, immediate rollback)
SELECT pg_terminate_backend(12345);

-- Terminate all connections for a user
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'problem_user'
AND pid != pg_backend_pid();  -- Don't terminate your own connection

-- Terminate all connections to a database (e.g., before dropping it)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'target_database'
AND pid != pg_backend_pid();

-- Terminate idle connections older than 30 minutes
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
AND state_change < current_timestamp - interval '30 minutes';
```

### Background Processes

The Postmaster spawns several background worker processes:

#### 1. Background Writer
Writes dirty pages from shared buffers to disk gradually to reduce checkpoint I/O spikes.

```conf
# Configuration in postgresql.conf
bgwriter_delay = 200ms              # Delay between rounds
bgwriter_lru_maxpages = 100         # Max pages to write per round
bgwriter_lru_multiplier = 2.0       # Multiplier for average usage
```

#### 2. WAL Writer
Writes WAL records from WAL buffers to WAL files on disk.

```conf
wal_writer_delay = 200ms            # Delay between WAL writes
wal_writer_flush_after = 1MB        # Flush after this much data
```

#### 3. Checkpointer
Performs checkpoints - writes all dirty pages to disk and creates a checkpoint record in WAL.

```conf
checkpoint_timeout = 5min           # Time between checkpoints
checkpoint_completion_target = 0.9  # Fraction of interval for checkpoint I/O
checkpoint_warning = 30s            # Warn if checkpoints too frequent
```

#### 4. Autovacuum Launcher and Workers
Automatically vacuums tables to reclaim space and prevent transaction ID wraparound.

```conf
autovacuum = on                     # Enable autovacuum
autovacuum_max_workers = 3          # Number of worker processes
autovacuum_naptime = 1min           # Time between runs
```

```sql
-- Monitor autovacuum activity
SELECT 
    pid,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE backend_type = 'autovacuum worker';
```

#### 5. Stats Collector
Collects statistics about database activity (queries, table access, etc.).

```sql
-- View collected statistics
SELECT * FROM pg_stat_user_tables;
SELECT * FROM pg_stat_statements;  -- Requires pg_stat_statements extension
```

#### 6. WAL Sender and Receiver
Handle streaming replication to standby servers.

```sql
-- On primary: Monitor WAL senders
SELECT * FROM pg_stat_replication;

-- On standby: Check WAL receiver status
SELECT * FROM pg_stat_wal_receiver;
```

### Postmaster Crash Recovery

If the Postmaster crashes or is killed unexpectedly:

1. **Postmaster will not restart automatically** (unless managed by systemd or similar)
2. **All child processes are terminated**
3. **On next startup, crash recovery is performed**
4. **WAL is replayed** to restore consistency
5. **Database returns to normal operation**

**Safe shutdown methods:**
```bash
# Smart shutdown (wait for all clients to disconnect)
pg_ctl -D /var/lib/postgresql/data stop -m smart

# Fast shutdown (terminate connections, checkpoint)
pg_ctl -D /var/lib/postgresql/data stop -m fast

# Immediate shutdown (abort immediately, requires crash recovery on restart)
pg_ctl -D /var/lib/postgresql/data stop -m immediate
```

### Process Monitoring and Debugging

```sql
-- List all backend processes
SELECT 
    pid,
    backend_type,          -- Type of backend process
    backend_start,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
ORDER BY backend_type, backend_start;

-- Backend types include:
-- - client backend
-- - autovacuum worker
-- - background writer
-- - checkpointer
-- - walwriter
-- - walsender
-- - walreceiver

-- Check for processes waiting on locks
SELECT 
    pid,
    usename,
    pg_blocking_pids(pid) AS blocked_by,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';

-- Memory usage per backend (requires Linux, may need superuser)
SELECT 
    pid,
    usename,
    application_name,
    pg_size_pretty((pg_backend_memory_contexts()).*) AS memory_info
FROM pg_stat_activity
WHERE backend_type = 'client backend';
```

### Connection Pooling (External)

The Postmaster creates one process per connection, which can be resource-intensive. For high-connection scenarios, use a connection pooler like **PgBouncer** or **Pgpool-II**.

```ini
# PgBouncer configuration example
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
```

**Benefit:** 1000 client connections share a pool of 25 database connections.

### Best Practices

1. **Set appropriate max_connections** based on workload and available memory
2. **Use connection pooling** for high-connection scenarios (web applications)
3. **Monitor connection usage** regularly to avoid hitting limits
4. **Set connection timeouts** to prevent hung connections
5. **Use smart shutdown** for planned maintenance when possible
6. **Monitor background worker processes** to ensure they're functioning properly
7. **Configure autovacuum properly** to prevent table bloat and wraparound
8. **Keep the Postmaster process healthy** - it's a single point of failure
9. **Use systemd or similar** for automatic restart on crash
10. **Monitor for crashed backends** and investigate root causes

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/app-postgres.html">PostgreSQL Postmaster Documentation</a>
  - <a href="https://www.postgresql.org/docs/18/runtime-config-connection.html">Connection Configuration</a>
  - <a href="https://www.postgresql.org/docs/18/monitoring-stats.html">Monitoring Statistics</a>

## 8. Transaction ID Wraparound

Transaction ID wraparound is one of the most critical maintenance concerns in PostgreSQL. Understanding and preventing it is essential for maintaining database health. Failure to address wraparound can lead to database shutdown to prevent data loss.

### Understanding Transaction IDs (XIDs)

PostgreSQL uses 32-bit transaction IDs (XIDs) to implement Multi-Version Concurrency Control (MVCC). Each transaction gets a unique XID, which is used to determine row visibility.

**The problem:** 32-bit integers can only represent about 4.2 billion values (2^32). After that, they wrap around to 0.

**The consequence:** Without proper handling, old transactions could suddenly appear to be "in the future," making old data invisible and potentially causing data loss.

### How PostgreSQL Prevents Wraparound

PostgreSQL uses **transaction ID freezing** to prevent wraparound:

1. **Freezing**: Old transaction IDs are replaced with a special "frozen" transaction ID (FrozenXID = 2)
2. **Frozen XIDs appear in the past** to all current and future transactions
3. **VACUUM FREEZE** performs this freezing operation
4. **Autovacuum** automatically freezes old rows based on age thresholds

### Monitoring Transaction Age

```sql
-- Check the age of the oldest transaction ID in each table
SELECT 
    c.oid::regclass AS table_name,
    age(c.relfrozenxid) AS xid_age,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS table_size,
    c.relfrozenxid
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind IN ('r', 't')  -- Regular tables and TOAST tables
AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;

-- Check database-wide oldest frozen XID
SELECT 
    datname,
    age(datfrozenxid) AS xid_age,
    datfrozenxid,
    2147483647 - age(datfrozenxid) AS xids_until_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Warning threshold: age > 1 billion is concerning
-- Critical threshold: age > 1.5 billion requires immediate action
-- Emergency: age > 2 billion (autovacuum_freeze_max_age default)

-- Check for tables approaching wraparound
SELECT 
    schemaname,
    tablename,
    age(relfrozenxid) AS xid_age,
    CASE 
        WHEN age(relfrozenxid) > 1500000000 THEN 'CRITICAL - IMMEDIATE ACTION REQUIRED'
        WHEN age(relfrozenxid) > 1000000000 THEN 'WARNING - Schedule maintenance soon'
        WHEN age(relfrozenxid) > 500000000 THEN 'NOTICE - Monitor closely'
        ELSE 'OK'
    END AS status
FROM pg_stat_user_tables
ORDER BY age(relfrozenxid) DESC;
```

### Configuration Parameters

```conf
# In postgresql.conf

# --- Transaction ID Wraparound Prevention ---

# Vacuum automatically freezes tuples older than this many transactions
# Default: 50 million transactions
vacuum_freeze_min_age = 50000000

# Force a table-scanning vacuum if relfrozenxid is older than this
# Default: 150 million transactions
vacuum_freeze_table_age = 150000000

# Force autovacuum if age exceeds this (last resort protection)
# Default: 200 million transactions (about 2 billion total before wraparound)
autovacuum_freeze_max_age = 200000000

# Similar settings for multixact IDs (used for row-level locks)
vacuum_multixact_freeze_min_age = 5000000
vacuum_multixact_freeze_table_age = 150000000
autovacuum_multixact_freeze_max_age = 400000000
```

**Understanding the parameters:**

- `vacuum_freeze_min_age`: Minimum age before VACUUM freezes a tuple (balances I/O vs safety)
- `vacuum_freeze_table_age`: Age at which VACUUM scans the entire table (even index-only pages)
- `autovacuum_freeze_max_age`: Emergency threshold that forces autovacuum even if disabled

### Manual Freezing

```sql
-- Freeze a specific table
VACUUM FREEZE my_table;

-- Freeze with verbose output
VACUUM FREEZE VERBOSE my_table;

-- Freeze all tables in a database
VACUUM FREEZE;

-- Aggressive vacuum freeze (for emergency situations)
-- Note: This can take a very long time and increase I/O significantly
VACUUM (FREEZE, VERBOSE) my_table;

-- Check what VACUUM FREEZE would do without actually doing it
-- (Requires PostgreSQL 12+)
VACUUM (FREEZE, VERBOSE, DRY_RUN) my_table;
```

### Autovacuum and Freezing

Autovacuum automatically handles freezing, but you should monitor it:

```sql
-- Check when tables were last autovacuumed
SELECT 
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    age(relfrozenxid) AS xid_age,
    n_dead_tup,
    n_live_tup
FROM pg_stat_user_tables
ORDER BY age(relfrozenxid) DESC
LIMIT 20;

-- Check autovacuum activity right now
SELECT 
    pid,
    usename,
    datname,
    state,
    wait_event_type,
    wait_event,
    query_start,
    query
FROM pg_stat_activity
WHERE query LIKE '%autovacuum%'
AND pid != pg_backend_pid();

-- Check if autovacuum is keeping up
SELECT 
    relname,
    age(relfrozenxid) AS xid_age,
    last_autovacuum,
    autovacuum_count,
    CASE 
        WHEN last_autovacuum IS NULL THEN 'NEVER autovacuumed!'
        WHEN last_autovacuum < now() - interval '1 week' AND age(relfrozenxid) > 100000000 
            THEN 'Autovacuum falling behind'
        ELSE 'OK'
    END AS autovacuum_status
FROM pg_stat_user_tables
WHERE age(relfrozenxid) > 100000000
ORDER BY age(relfrozenxid) DESC;
```

### Preventing Wraparound Issues

#### 1. Ensure Autovacuum is Enabled and Working

```sql
-- Check if autovacuum is enabled globally
SHOW autovacuum;

-- Check autovacuum configuration
SELECT 
    name,
    setting,
    unit,
    short_desc
FROM pg_settings
WHERE name LIKE 'autovacuum%'
ORDER BY name;

-- Check if any tables have autovacuum disabled
SELECT 
    n.nspname,
    c.relname,
    c.reloptions
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
AND EXISTS (
    SELECT 1 FROM unnest(c.reloptions) opt 
    WHERE opt LIKE 'autovacuum_enabled=false'
);
```

#### 2. Monitor XID Age Regularly

```sql
-- Create a monitoring query you can run regularly (daily)
CREATE OR REPLACE VIEW wraparound_monitoring AS
SELECT 
    schemaname || '.' || tablename AS table_name,
    age(relfrozenxid) AS xid_age,
    2000000000 - age(relfrozenxid) AS xids_remaining,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS table_size,
    last_autovacuum,
    last_vacuum,
    CASE 
        WHEN age(relfrozenxid) > 1500000000 THEN 'CRITICAL'
        WHEN age(relfrozenxid) > 1000000000 THEN 'WARNING'
        WHEN age(relfrozenxid) > 500000000 THEN 'NOTICE'
        ELSE 'OK'
    END AS status
FROM pg_stat_user_tables
ORDER BY age(relfrozenxid) DESC;

-- Query the view
SELECT * FROM wraparound_monitoring WHERE status != 'OK';
```

#### 3. Schedule Preventive Maintenance

```sql
-- For very large tables, schedule regular VACUUM FREEZE during low-activity periods
-- Example maintenance script:

DO $$
DECLARE
    table_rec RECORD;
BEGIN
    FOR table_rec IN 
        SELECT schemaname, tablename
        FROM pg_stat_user_tables
        WHERE age(relfrozenxid) > 800000000
        ORDER BY age(relfrozenxid) DESC
    LOOP
        RAISE NOTICE 'Freezing %.%', table_rec.schemaname, table_rec.tablename;
        EXECUTE format('VACUUM FREEZE %I.%I', table_rec.schemaname, table_rec.tablename);
    END LOOP;
END $$;
```

#### 4. Tune Autovacuum for Large Tables

For tables that grow very large, tune autovacuum settings per-table:

```sql
-- Make autovacuum more aggressive for a specific large table
ALTER TABLE large_table SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- Vacuum at 1% dead tuples (default: 20%)
    autovacuum_vacuum_threshold = 1000,
    autovacuum_freeze_min_age = 1000000,
    autovacuum_freeze_max_age = 150000000
);

-- For append-only tables (like logs), tune insert-based autovacuum
ALTER TABLE log_table SET (
    autovacuum_vacuum_insert_threshold = 10000,
    autovacuum_vacuum_insert_scale_factor = 0.1
);

-- Allocate more memory to autovacuum for faster operation
ALTER TABLE large_table SET (
    autovacuum_work_mem = '1GB'
);
```

### Emergency Response: Wraparound Shutdown Prevention

If a database is approaching wraparound (age > 2 billion), PostgreSQL will:

1. **Emit warnings** in logs when age > 1.5 billion
2. **Stop accepting new transactions** when age approaches 2^31 - 1000000
3. **Shut down** to prevent data loss

**Emergency response procedure:**

```sql
-- 1. Check the severity
SELECT 
    datname,
    age(datfrozenxid) AS xid_age,
    2147483647 - age(datfrozenxid) AS xids_until_emergency_shutdown
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- 2. If critical, immediately freeze the oldest tables
-- Start with smallest tables first for quick wins
SELECT 
    schemaname || '.' || tablename AS table_name,
    age(relfrozenxid) AS xid_age,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_stat_user_tables
WHERE age(relfrozenxid) > 1500000000
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) ASC;

-- 3. Freeze tables one by one
VACUUM FREEZE VERBOSE small_table_1;
VACUUM FREEZE VERBOSE small_table_2;
-- Continue until age is below danger threshold

-- 4. For very large tables, consider using pg_repack or similar tools
-- to avoid long-running VACUUM operations
```

### Best Practices

1. **Enable autovacuum** and ensure it's running (it's on by default)
2. **Monitor XID age regularly** (at least weekly for large databases)
3. **Set up alerts** for tables with age > 1 billion transactions
4. **Never disable autovacuum** on production databases (or only very temporarily)
5. **Tune autovacuum** for large, frequently-updated tables
6. **Schedule preventive VACUUM FREEZE** for very large tables
7. **Monitor autovacuum logs** to ensure it's completing successfully
8. **Plan for growth** - more transactions = faster XID consumption
9. **Test recovery procedures** before an emergency occurs
10. **Keep statistics up to date** to help autovacuum make good decisions

### Common Causes of Wraparound Issues

1. **Autovacuum disabled** globally or on specific tables
2. **Long-running transactions** blocking autovacuum
3. **Prepared transactions** left open (2PC)
4. **Replication slots** preventing old WAL removal
5. **Insufficient autovacuum workers** for workload
6. **I/O constraints** making vacuum too slow
7. **Very large tables** taking too long to vacuum

```sql
-- Find long-running transactions blocking autovacuum
SELECT 
    pid,
    usename,
    datname,
    state,
    backend_xid,
    backend_xmin,
    now() - xact_start AS duration,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
AND state != 'idle'
ORDER BY xact_start
LIMIT 10;

-- Find prepared transactions
SELECT * FROM pg_prepared_xacts;

-- Find replication slots
SELECT 
    slot_name,
    slot_type,
    database,
    active,
    xmin,
    catalog_xmin,
    restart_lsn
FROM pg_replication_slots;
```

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND">PostgreSQL Transaction ID Wraparound</a>
  - <a href="https://www.postgresql.org/docs/18/routine-vacuuming.html">Routine Vacuuming</a>
  - <a href="https://wiki.postgresql.org/wiki/Wraparound">PostgreSQL Wiki: Wraparound</a>

## 9. Explain Plans

Understanding query execution plans is essential for optimizing PostgreSQL performance. The `EXPLAIN` command shows how PostgreSQL will execute a query, while `EXPLAIN ANALYZE` actually runs the query and shows real execution statistics.

### Basic EXPLAIN

```sql
-- Show query plan without executing
EXPLAIN 
SELECT * FROM users WHERE email = 'user@example.com';

-- Sample output:
-- Seq Scan on users  (cost=0.00..35.50 rows=1 width=100)
--   Filter: (email = 'user@example.com'::text)
```

**Reading the output:**
- **Seq Scan**: Sequential scan (reads entire table)
- **cost=0.00..35.50**: Startup cost..total cost (in arbitrary units)
- **rows=1**: Estimated number of rows returned
- **width=100**: Estimated average row size in bytes

### EXPLAIN ANALYZE

```sql
-- Execute query and show actual statistics
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';

-- Sample output:
-- Seq Scan on users  (cost=0.00..35.50 rows=1 width=100) 
--                    (actual time=0.123..2.456 rows=1 loops=1)
--   Filter: (email = 'user@example.com'::text)
--   Rows Removed by Filter: 9999
-- Planning Time: 0.234 ms
-- Execution Time: 2.567 ms
```

**Additional information:**
- **actual time**: Real startup time..total time in milliseconds
- **rows=1**: Actual number of rows returned
- **loops=1**: Number of times this node executed
- **Rows Removed by Filter**: Rows scanned but not returned
- **Planning Time**: Time to create the query plan
- **Execution Time**: Time to execute the query

### EXPLAIN Options

```sql
-- VERBOSE: Show additional details
EXPLAIN (VERBOSE, ANALYZE)
SELECT * FROM users WHERE email = 'user@example.com';

-- BUFFERS: Show buffer cache statistics (requires ANALYZE)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'user@example.com';
-- Shows: Buffers: shared hit=45 read=12

-- COSTS: Show or hide cost estimates
EXPLAIN (COSTS OFF)
SELECT * FROM users WHERE email = 'user@example.com';

-- FORMAT: Output format (TEXT, JSON, YAML, XML)
EXPLAIN (ANALYZE, FORMAT JSON)
SELECT * FROM users WHERE email = 'user@example.com';

-- TIMING: Enable/disable timing (disabling can reduce overhead)
EXPLAIN (ANALYZE, TIMING OFF)
SELECT * FROM users WHERE email = 'user@example.com';

-- WAL: Show WAL generation statistics (PostgreSQL 13+)
EXPLAIN (ANALYZE, WAL)
INSERT INTO logs (message) VALUES ('test');

-- SETTINGS: Show configuration parameters that affect planning
EXPLAIN (SETTINGS)
SELECT * FROM users WHERE email = 'user@example.com';

-- Combination (most comprehensive for tuning)
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, TIMING)
SELECT u.*, o.* 
FROM users u 
JOIN orders o ON u.user_id = o.customer_id
WHERE u.created_at > '2024-01-01';
```

### Common Scan Types

#### 1. Sequential Scan (Seq Scan)
Reads entire table from start to finish.

```sql
EXPLAIN ANALYZE
SELECT * FROM large_table WHERE status = 'active';

-- Seq Scan on large_table  (cost=0.00..18334.00 rows=1000 width=100)
--   Filter: (status = 'active'::text)
```

**When it occurs:** No suitable index, or planner estimates seq scan is faster (e.g., selecting >10% of rows)

**Fix if inappropriate:** Add an index on the filtered column

#### 2. Index Scan
Uses an index to find specific rows, then fetches from table.

```sql
CREATE INDEX idx_users_email ON users(email);

EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';

-- Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=100)
--   Index Cond: (email = 'user@example.com'::text)
```

**When it occurs:** Fetching small percentage of rows with good selectivity

#### 3. Index Only Scan
Query satisfied entirely from index without accessing table.

```sql
CREATE INDEX idx_users_email_name ON users(email) INCLUDE (name);

EXPLAIN ANALYZE
SELECT email, name FROM users WHERE email = 'user@example.com';

-- Index Only Scan using idx_users_email_name on users  (cost=0.42..8.44 rows=1 width=64)
--   Index Cond: (email = 'user@example.com'::text)
--   Heap Fetches: 0
```

**Best case:** Heap Fetches: 0 means no table access needed

**Fix if Heap Fetches is high:** Run VACUUM to update visibility map

#### 4. Bitmap Index Scan
Combines multiple indexes or handles many scattered rows efficiently.

```sql
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_created ON users(created_at);

EXPLAIN ANALYZE
SELECT * FROM users 
WHERE status = 'active' AND created_at > '2024-01-01';

-- Bitmap Heap Scan on users  (cost=25.42..250.00 rows=100 width=100)
--   Recheck Cond: ((status = 'active') AND (created_at > '2024-01-01'))
--   ->  BitmapAnd  (cost=25.42..25.42 rows=100 width=0)
--         ->  Bitmap Index Scan on idx_users_status
--         ->  Bitmap Index Scan on idx_users_created
```

**When it occurs:** Multiple indexes combined, or many scattered rows to fetch

### Join Types

#### 1. Nested Loop Join
For each row in outer table, scan inner table.

```sql
EXPLAIN ANALYZE
SELECT * FROM users u JOIN orders o ON u.user_id = o.customer_id
WHERE u.user_id = 123;

-- Nested Loop  (cost=0.85..24.45 rows=5 width=200)
--   ->  Index Scan using users_pkey on users u  (cost=0.42..8.44 rows=1 width=100)
--   ->  Index Scan using idx_orders_customer on orders o  (cost=0.42..16.00 rows=5 width=100)
```

**Best for:** Small number of rows, good indexes on join keys

**Worst for:** Large row counts without indexes

#### 2. Hash Join
Builds hash table of smaller relation, probes with larger relation.

```sql
EXPLAIN ANALYZE
SELECT * FROM users u JOIN orders o ON u.user_id = o.customer_id;

-- Hash Join  (cost=145.00..2345.00 rows=10000 width=200)
--   Hash Cond: (o.customer_id = u.user_id)
--   ->  Seq Scan on orders o  (cost=0.00..1500.00 rows=50000 width=100)
--   ->  Hash  (cost=95.00..95.00 rows=4000 width=100)
--         Buckets: 4096  Batches: 1  Memory Usage: 256kB
--         ->  Seq Scan on users u  (cost=0.00..95.00 rows=4000 width=100)
```

**Best for:** Large tables being joined, equality conditions

**Issue if Memory Usage is high:** May spill to disk (increase work_mem)

#### 3. Merge Join
Both inputs sorted, then merged together.

```sql
EXPLAIN ANALYZE
SELECT * FROM users u JOIN orders o ON u.user_id = o.customer_id
ORDER BY u.user_id;

-- Merge Join  (cost=0.85..2845.12 rows=10000 width=200)
--   Merge Cond: (u.user_id = o.customer_id)
--   ->  Index Scan using users_pkey on users u
--   ->  Index Scan using idx_orders_customer on orders o
```

**Best for:** Both sides already sorted (or can use indexes for sorting)

### Aggregation Methods

```sql
-- HashAggregate: Hash-based grouping (faster, requires memory)
EXPLAIN ANALYZE
SELECT customer_id, count(*) FROM orders GROUP BY customer_id;

-- HashAggregate  (cost=1250.00..1350.00 rows=10000 width=12)
--   Group Key: customer_id
--   Batches: 1  Memory Usage: 1024kB

-- GroupAggregate: Sort-based grouping (used when work_mem insufficient)
SET work_mem = '1MB';
EXPLAIN ANALYZE
SELECT customer_id, count(*) FROM orders GROUP BY customer_id;

-- GroupAggregate  (cost=5000.00..5500.00 rows=10000 width=12)
--   Group Key: customer_id
--   ->  Sort  (cost=5000.00..5125.00 rows=50000 width=4)
```

### Sorting

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users ORDER BY created_at DESC LIMIT 100;

-- Limit  (cost=850.23..852.73 rows=100 width=100)
--   ->  Sort  (cost=850.23..900.23 rows=20000 width=100)
--         Sort Key: created_at DESC
--         Sort Method: top-N heapsort  Memory: 45kB  -- Good!
--         ->  Seq Scan on users

-- If sorting to disk (bad):
--         Sort Method: external merge  Disk: 15360kB  -- Need more work_mem!
```

**Fix for disk sorting:**
1. Increase work_mem: `SET work_mem = '16MB';`
2. Add index on sort column: `CREATE INDEX idx_users_created ON users(created_at DESC);`
3. Use LIMIT to reduce rows sorted

### Performance Tuning Tips

#### 1. Look for Sequential Scans on Large Tables

```sql
-- Problem: Sequential scan on 10M row table
EXPLAIN ANALYZE
SELECT * FROM large_table WHERE rare_value = 'specific';

-- Seq Scan on large_table  (cost=0.00..200000.00 rows=1 width=100)
--   Filter: (rare_value = 'specific'::text)
--   Rows Removed by Filter: 9999999

-- Solution: Add index
CREATE INDEX idx_large_table_rare_value ON large_table(rare_value);
```

#### 2. Check for High "Rows Removed by Filter"

```sql
-- Problem: Reading many rows, returning few
-- Rows Removed by Filter: 999999

-- Solution: Add more selective indexes or partial indexes
CREATE INDEX idx_active_recent_users ON users(email) 
WHERE status = 'active' AND created_at > '2024-01-01';
```

#### 3. Watch for Nested Loop with Many Rows

```sql
-- Problem: Nested Loop with 1000 outer rows × 1000 inner rows = 1M iterations
-- Nested Loop  (actual time=...  rows=500000 loops=1)
--   ->  Seq Scan on table1  (actual time=... rows=1000 loops=1)
--   ->  Seq Scan on table2  (actual time=... rows=1000 loops=1000)  -- Ouch!

-- Solution 1: Add index on join key
CREATE INDEX idx_table2_join_key ON table2(join_column);

-- Solution 2: Encourage hash join
SET enable_nestloop = off;  -- For testing only
SET work_mem = '64MB';      -- Allow larger hash tables
```

#### 4. Monitor Buffer Usage

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'user@example.com';

-- Buffers: shared hit=1234 read=56 dirtied=0 written=0
-- 
-- shared hit: Found in cache (good)
-- read: Had to read from disk (slower)
-- dirtied: Modified pages
-- written: Written to disk

-- If many reads: Data not in cache
-- Solution: Increase shared_buffers, or warm cache
SELECT pg_prewarm('users');
```

#### 5. Check Estimated vs Actual Rows

```sql
-- Problem: Planner estimate way off
-- Hash Join  (cost=... rows=10 width=...)       -- Estimated 10 rows
--            (actual time=... rows=100000 loops=1)  -- Actually 100,000!

-- This causes poor plan choices
-- Solution: Update statistics
ANALYZE users;
ANALYZE orders;

-- Or increase statistics target for specific columns
ALTER TABLE users ALTER COLUMN status SET STATISTICS 1000;
ANALYZE users;
```

#### 6. Look for Subplan Execution

```sql
-- Problem: Subplan executed many times
-- Nested Loop  (actual time=... rows=1000 loops=1)
--   ->  Seq Scan on users
--   ->  SubPlan 1  (actual time=... rows=1 loops=1000)  -- Executed 1000 times!
--         ->  Seq Scan on orders
--               Filter: (customer_id = users.user_id)

-- Solution: Rewrite as JOIN instead of correlated subquery
-- Before:
SELECT u.*, (SELECT count(*) FROM orders WHERE customer_id = u.user_id) 
FROM users u;

-- After:
SELECT u.*, count(o.order_id) 
FROM users u 
LEFT JOIN orders o ON u.user_id = o.customer_id 
GROUP BY u.user_id;
```

### Practical Tuning Workflow

```sql
-- 1. Get baseline
EXPLAIN (ANALYZE, BUFFERS, TIMING)
SELECT ... FROM ... WHERE ...;

-- Note:
-- - Total execution time
-- - Buffer hits vs reads
-- - Scan types
-- - Join methods
-- - Rows estimated vs actual

-- 2. Identify bottlenecks
-- Look for:
--   - Seq Scans on large tables (add index?)
--   - High "Rows Removed by Filter" (add better index?)
--   - Nested Loop with high loop count (add index on inner relation?)
--   - Sorting to disk (increase work_mem?)
--   - Many buffer reads (data not cached?)
--   - Large estimation errors (run ANALYZE?)

-- 3. Make ONE change at a time
CREATE INDEX idx_fix ON table(column);

-- 4. Retest
EXPLAIN (ANALYZE, BUFFERS, TIMING)
SELECT ... FROM ... WHERE ...;

-- 5. Compare before/after
-- Execution time improved? Buffer usage better? 

-- 6. Repeat until satisfactory
```

### Using EXPLAIN in Production

```sql
-- Enable auto_explain to log slow queries
-- In postgresql.conf:
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '1000ms'  -- Log queries > 1 second
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_timing = off  -- Reduce overhead
auto_explain.log_format = 'json'

-- Check logs for slow queries with their plans
```

### Query Plan Visualization

```sql
-- Get JSON format for visualization tools
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM users WHERE email = 'user@example.com';

-- Use visualization tools:
-- - https://explain.depesz.com/
-- - https://explain.dalibo.com/
-- - PgAdmin's graphical EXPLAIN
-- - pgMustard (commercial)
```

### Common Anti-Patterns

1. **Missing indexes on foreign keys**
   ```sql
   -- Always index foreign keys for JOIN performance
   CREATE INDEX idx_orders_customer_id ON orders(customer_id);
   ```

2. **Functions in WHERE preventing index use**
   ```sql
   -- Bad: Function prevents index use
   WHERE LOWER(email) = 'user@example.com'
   
   -- Good: Use expression index
   CREATE INDEX idx_users_email_lower ON users(LOWER(email));
   ```

3. **SELECT * when only few columns needed**
   ```sql
   -- Bad: Fetches all columns
   SELECT * FROM large_table WHERE id = 123;
   
   -- Good: Fetch only what's needed (may enable index-only scan)
   SELECT id, name FROM large_table WHERE id = 123;
   ```

4. **LIMIT without ORDER BY on paginated queries**
   ```sql
   -- Bad: Non-deterministic results
   SELECT * FROM users LIMIT 100 OFFSET 1000;
   
   -- Good: Explicit ordering
   SELECT * FROM users ORDER BY user_id LIMIT 100 OFFSET 1000;
   ```

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/sql-explain.html">PostgreSQL EXPLAIN Documentation</a>
  - <a href="https://www.postgresql.org/docs/18/using-explain.html">Using EXPLAIN</a>
  - <a href="https://www.postgresql.org/docs/18/planner-optimizer.html">Query Planning</a>

## 10. Deadlocks

A deadlock occurs when two or more transactions are waiting for each other to release locks, creating a circular dependency that prevents any of them from proceeding. PostgreSQL automatically detects deadlocks and resolves them by aborting one of the transactions.

### What is a Deadlock?

**Simple example:**

| Time | Transaction 1 | Transaction 2 |
|------|---------------|---------------|
| T1 | BEGIN; | BEGIN; |
| T2 | UPDATE accounts SET balance = 100 WHERE id = 1; | UPDATE accounts SET balance = 200 WHERE id = 2; |
| T3 | -- Holds lock on row 1 | -- Holds lock on row 2 |
| T4 | UPDATE accounts SET balance = 150 WHERE id = 2; | UPDATE accounts SET balance = 250 WHERE id = 1; |
| T5 | -- Waits for lock on row 2 | -- Waits for lock on row 1 |
| T6 | **DEADLOCK DETECTED** | (One transaction is rolled back) |

Transaction 1 holds a lock on row 1 and waits for row 2.
Transaction 2 holds a lock on row 2 and waits for row 1.
Neither can proceed → Deadlock!

### Deadlock Detection

PostgreSQL checks for deadlocks periodically:

```conf
# In postgresql.conf
deadlock_timeout = 1s  # How long to wait before checking for deadlock
```

When a deadlock is detected, PostgreSQL:
1. Selects one transaction as the "victim"
2. Aborts that transaction with an error
3. Allows other transactions to proceed

**Error message:**
```
ERROR:  deadlock detected
DETAIL:  Process 12345 waits for ShareLock on transaction 67890;
         blocked by process 12346.
         Process 12346 waits for ShareLock on transaction 67889;
         blocked by process 12345.
HINT:  See server log for query details.
```

### Monitoring Deadlocks

#### 1. Check Deadlock Count

```sql
-- View deadlock statistics
SELECT 
    datname,
    deadlocks,
    conflicts,
    temp_files,
    temp_bytes
FROM pg_stat_database
WHERE datname = current_database();

-- Deadlocks over time (requires pg_stat_statements)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT 
    query,
    calls,
    deadlocks,
    deadlocks::float / NULLIF(calls, 0) * 100 AS deadlock_rate_percent
FROM pg_stat_statements
WHERE deadlocks > 0
ORDER BY deadlocks DESC
LIMIT 20;
```

#### 2. View Current Locks

```sql
-- See all current locks
SELECT 
    locktype,
    database,
    relation::regclass,
    page,
    tuple,
    virtualxid,
    transactionid,
    classid,
    objid,
    objsubid,
    virtualtransaction,
    pid,
    mode,
    granted,
    fastpath
FROM pg_locks
ORDER BY pid;

-- Simplified view of blocked queries
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement,
    blocked_activity.application_name AS blocked_application,
    blocking_activity.application_name AS blocking_application
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Using pg_blocking_pids (PostgreSQL 9.6+) - simpler
SELECT 
    pid,
    usename,
    pg_blocking_pids(pid) AS blocked_by,
    query AS blocked_query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

#### 3. Create a Deadlock Monitoring View

```sql
CREATE OR REPLACE VIEW lock_monitor AS
SELECT 
    COALESCE(blockingl.relation::regclass::text, blockingl.locktype) AS locked_item,
    now() - blockeda.query_start AS waiting_duration,
    blockeda.pid AS blocked_pid,
    blockeda.query AS blocked_query,
    blockedl.mode AS blocked_mode,
    blockinga.pid AS blocking_pid,
    blockinga.query AS blocking_query,
    blockingl.mode AS blocking_mode
FROM pg_locks blockedl
JOIN pg_stat_activity blockeda ON blockedl.pid = blockeda.pid
JOIN pg_locks blockingl ON (
    blockingl.transactionid = blockedl.transactionid
    OR blockingl.relation = blockedl.relation AND blockingl.locktype = blockedl.locktype
) AND blockedl.pid != blockingl.pid
JOIN pg_stat_activity blockinga ON blockingl.pid = blockinga.pid
WHERE NOT blockedl.granted
AND blockinga.datname = current_database();

-- Use the view
SELECT * FROM lock_monitor;
```

### Lock Types and Modes

PostgreSQL uses various lock types:

```sql
-- View lock types in use
SELECT DISTINCT mode FROM pg_locks;

-- Common lock modes (from least to most restrictive):
-- AccessShareLock       - SELECT
-- RowShareLock          - SELECT FOR UPDATE/SHARE
-- RowExclusiveLock      - INSERT, UPDATE, DELETE
-- ShareUpdateExclusiveLock - VACUUM, CREATE INDEX CONCURRENTLY
-- ShareLock             - CREATE INDEX
-- ShareRowExclusiveLock - (rare)
-- ExclusiveLock         - REFRESH MATERIALIZED VIEW CONCURRENTLY
-- AccessExclusiveLock   - DROP TABLE, TRUNCATE, VACUUM FULL, LOCK TABLE

-- Check locks by type
SELECT 
    locktype,
    mode,
    count(*) AS count
FROM pg_locks
WHERE granted = true
GROUP BY locktype, mode
ORDER BY count DESC;
```

### Common Causes of Deadlocks

#### 1. Inconsistent Lock Order

```sql
-- Transaction 1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Transaction 2 (running concurrently with opposite order)
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;  -- Different order!
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
COMMIT;

-- Solution: Always acquire locks in the same order
-- Both transactions should lock id=1 first, then id=2
```

#### 2. Missing Indexes on Foreign Keys

```sql
-- Without index on orders.customer_id:
-- DELETE from customers can cause deadlocks with INSERT into orders

-- Solution: Index all foreign keys
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

#### 3. Long-Running Transactions

```sql
-- Long transactions hold locks longer, increasing deadlock chance
-- Solution: Keep transactions as short as possible
BEGIN;
-- Do work quickly
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Don't do this:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Call external API, complex processing, user interaction ...
COMMIT;
```

#### 4. Concurrent Updates to the Same Rows

```sql
-- Multiple transactions updating popular rows
-- Example: Updating a "points_balance" for active users

-- Solution: Use advisory locks or SELECT FOR UPDATE NOWAIT
BEGIN;
SELECT * FROM users WHERE id = 123 FOR UPDATE NOWAIT;
-- If lock fails, handle gracefully instead of waiting
UPDATE users SET points = points + 10 WHERE id = 123;
COMMIT;
```

### Deadlock Prevention Strategies

#### 1. Consistent Lock Ordering

```sql
-- BAD: Random order
UPDATE accounts SET balance = balance + amount WHERE id = to_id;
UPDATE accounts SET balance = balance - amount WHERE id = from_id;

-- GOOD: Always lock in ascending ID order
BEGIN;
IF from_id < to_id THEN
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
ELSE
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
END IF;
COMMIT;
```

#### 2. Use SELECT FOR UPDATE

```sql
-- Lock rows explicitly before updating
BEGIN;
-- Lock rows in consistent order
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;

-- Now safe to update
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

#### 3. Use NOWAIT or SKIP LOCKED

```sql
-- NOWAIT: Fail immediately if row is locked
BEGIN;
SELECT * FROM queue_items WHERE id = 123 FOR UPDATE NOWAIT;
-- If locked, transaction fails immediately instead of waiting
-- Application can retry or handle differently
COMMIT;

-- SKIP LOCKED: Skip locked rows (useful for queue processing)
BEGIN;
SELECT * FROM job_queue 
WHERE status = 'pending' 
ORDER BY priority DESC 
LIMIT 1 
FOR UPDATE SKIP LOCKED;
-- Gets next available job, skips locked ones
UPDATE job_queue SET status = 'processing' WHERE id = selected_id;
COMMIT;
```

#### 4. Reduce Lock Hold Time

```sql
-- Keep transactions short
-- Do heavy computation outside transaction
-- Prepare data first
WITH data_to_update AS (
    SELECT compute_expensive_value() AS new_value
)
-- Then quick transaction
BEGIN;
UPDATE my_table SET value = (SELECT new_value FROM data_to_update) WHERE id = 123;
COMMIT;
```

#### 5. Batch Operations Carefully

```sql
-- Instead of updating rows one by one:
-- BAD (in a loop):
FOR each_id IN id_list LOOP
    UPDATE my_table SET status = 'processed' WHERE id = each_id;
END LOOP;

-- GOOD (single statement):
UPDATE my_table SET status = 'processed' WHERE id = ANY(id_array);

-- Or with chunking for very large updates:
UPDATE my_table SET status = 'processed' 
WHERE id IN (SELECT id FROM my_table WHERE status = 'pending' LIMIT 1000);
```

#### 6. Use Advisory Locks

```sql
-- Application-level locks to coordinate access
-- Transaction 1
BEGIN;
SELECT pg_advisory_lock(123);  -- Application-defined lock ID
-- Do work that might conflict
UPDATE shared_resource SET value = value + 1;
SELECT pg_advisory_unlock(123);
COMMIT;

-- Transaction 2
BEGIN;
SELECT pg_advisory_lock(123);  -- Waits for Transaction 1
-- Now safe to proceed
UPDATE shared_resource SET value = value + 1;
SELECT pg_advisory_unlock(123);
COMMIT;

-- Or use try-lock (non-blocking)
SELECT pg_try_advisory_lock(123);  -- Returns true if acquired, false if not
```

#### 7. Optimize Indexes

```sql
-- Ensure foreign key columns are indexed
-- This reduces lock duration on referenced tables
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
```

### Handling Deadlocks in Application Code

```python
# Python example with psycopg2
import psycopg2
from psycopg2 import extensions

def transfer_money(conn, from_id, to_id, amount, max_retries=3):
    """Transfer money with deadlock retry logic"""
    for attempt in range(max_retries):
        try:
            with conn.cursor() as cur:
                # Lock rows in consistent order
                cur.execute("""
                    SELECT * FROM accounts 
                    WHERE id IN (%s, %s) 
                    ORDER BY id 
                    FOR UPDATE
                """, (from_id, to_id))
                
                # Perform transfer
                cur.execute(
                    "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, from_id)
                )
                cur.execute(
                    "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to_id)
                )
                
                conn.commit()
                return True  # Success
                
        except psycopg2.extensions.TransactionRollbackError as e:
            # Deadlock detected
            conn.rollback()
            if attempt < max_retries - 1:
                # Retry with exponential backoff
                time.sleep(0.1 * (2 ** attempt))
                continue
            else:
                # Max retries exceeded
                raise
    
    return False
```

```javascript
// Node.js example with pg
async function transferMoney(client, fromId, toId, amount, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      await client.query('BEGIN');
      
      // Lock rows in consistent order
      await client.query(
        'SELECT * FROM accounts WHERE id IN ($1, $2) ORDER BY id FOR UPDATE',
        [fromId, toId]
      );
      
      // Perform transfer
      await client.query(
        'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
        [amount, fromId]
      );
      await client.query(
        'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
        [amount, toId]
      );
      
      await client.query('COMMIT');
      return true; // Success
      
    } catch (err) {
      await client.query('ROLLBACK');
      
      if (err.code === '40P01') { // deadlock_detected
        if (attempt < maxRetries - 1) {
          // Retry with exponential backoff
          await new Promise(resolve => 
            setTimeout(resolve, 100 * Math.pow(2, attempt))
          );
          continue;
        }
      }
      throw err;
    }
  }
}
```

### Analyzing Deadlocks from Logs

Enable detailed deadlock logging:

```conf
# In postgresql.conf
log_lock_waits = on              # Log slow lock acquisitions
deadlock_timeout = 1s            # Time before deadlock check
log_min_error_statement = error  # Log statements that cause errors
log_error_verbosity = verbose    # Include detailed error info
```

**Example log entry:**
```
ERROR:  deadlock detected
DETAIL:  Process 12345 waits for ShareLock on transaction 67890; blocked by process 12346.
         Process 12346 waits for ShareLock on transaction 67889; blocked by process 12345.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,123) in relation "accounts"
STATEMENT:  UPDATE accounts SET balance = balance - 100 WHERE id = 2
```

### Best Practices

1. **Always acquire locks in a consistent order** (e.g., ascending ID)
2. **Keep transactions short** - minimize lock hold time
3. **Use explicit locking** (SELECT FOR UPDATE) when order matters
4. **Index foreign keys** to reduce lock duration
5. **Implement retry logic** in application code for deadlocks
6. **Use NOWAIT or SKIP LOCKED** for non-blocking operations
7. **Monitor deadlock frequency** and investigate patterns
8. **Consider advisory locks** for application-level coordination
9. **Avoid long-running transactions** in OLTP workloads
10. **Test concurrent scenarios** during development

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/explicit-locking.html">PostgreSQL Explicit Locking</a>
  - <a href="https://www.postgresql.org/docs/18/view-pg-locks.html">pg_locks System View</a>
  - <a href="https://www.postgresql.org/docs/18/runtime-config-locks.html">Lock Configuration</a>

## 11. Metadata System Tables

PostgreSQL stores extensive metadata about database objects in system catalogs (tables starting with `pg_`). Understanding these tables is essential for monitoring, troubleshooting, and database administration.

### Core System Catalogs

#### pg_class - Tables, Indexes, Views, Sequences

`pg_class` contains information about all relations (tables, indexes, sequences, views, etc.).

```sql
-- View all user tables with sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - 
                   pg_relation_size(schemaname||'.'||tablename)) AS indexes_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- View table details from pg_class
SELECT 
    c.relname AS table_name,
    c.relkind AS type,  -- r=table, i=index, v=view, m=materialized view, S=sequence
    c.reltuples::bigint AS estimated_rows,
    pg_size_pretty(pg_relation_size(c.oid)) AS size,
    c.relpages AS pages,
    c.relallvisible AS all_visible_pages,
    age(c.relfrozenxid) AS xid_age,
    c.relhasindex AS has_indexes
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE n.nspname = 'public'
AND c.relkind = 'r'  -- Regular tables only
ORDER BY pg_relation_size(c.oid) DESC;

-- View indexes for a specific table
SELECT 
    i.relname AS index_name,
    pg_size_pretty(pg_relation_size(i.oid)) AS index_size,
    a.attname AS column_name,
    am.amname AS index_type
FROM pg_class t
JOIN pg_index ix ON t.oid = ix.indrelid
JOIN pg_class i ON ix.indexrelid = i.oid
JOIN pg_am am ON i.relam = am.oid
LEFT JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(ix.indkey)
WHERE t.relname = 'my_table'
ORDER BY pg_relation_size(i.oid) DESC;
```

#### pg_stat_activity - Current Activity

`pg_stat_activity` shows current connections and their activity.

```sql
-- View all active connections
SELECT 
    pid,                    -- Process ID
    usename,               -- User name
    application_name,      -- Application name
    client_addr,           -- Client IP
    client_hostname,       -- Client hostname
    backend_start,         -- Connection start time
    xact_start,            -- Transaction start time
    query_start,           -- Current query start time
    state_change,          -- Last state change time
    state,                 -- Current state: active, idle, idle in transaction
    wait_event_type,       -- Type of wait event
    wait_event,            -- Specific wait event
    query                  -- Current or last query
FROM pg_stat_activity
ORDER BY backend_start DESC;

-- Active queries only
SELECT 
    pid,
    usename,
    now() - query_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state = 'active'
AND pid != pg_backend_pid()  -- Exclude current query
ORDER BY duration DESC;

-- Idle in transaction (potentially problematic)
SELECT 
    pid,
    usename,
    now() - state_change AS idle_duration,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY state_change;

-- Long-running transactions
SELECT 
    pid,
    usename,
    now() - xact_start AS transaction_duration,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 10;

-- Connection counts by state
SELECT 
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY state
ORDER BY connections DESC;

-- Connection counts by user
SELECT 
    usename,
    count(*) AS connections,
    count(*) FILTER (WHERE state = 'active') AS active_connections
FROM pg_stat_activity
WHERE usename IS NOT NULL
GROUP BY usename
ORDER BY connections DESC;

-- Connection counts by application
SELECT 
    application_name,
    count(*) AS connections,
    count(*) FILTER (WHERE state = 'active') AS active_connections
FROM pg_stat_activity
WHERE application_name IS NOT NULL
GROUP BY application_name
ORDER BY connections DESC;
```

#### pg_stat_all_tables - Table Statistics

`pg_stat_all_tables` provides statistics about table access and maintenance.

```sql
-- Comprehensive table statistics
SELECT 
    schemaname,
    relname AS table_name,
    seq_scan,                   -- Number of sequential scans
    seq_tup_read,               -- Tuples read by sequential scans
    idx_scan,                   -- Number of index scans
    idx_tup_fetch,              -- Tuples fetched by index scans
    n_tup_ins,                  -- Tuples inserted
    n_tup_upd,                  -- Tuples updated
    n_tup_del,                  -- Tuples deleted
    n_tup_hot_upd,              -- HOT updates (heap-only tuples)
    n_live_tup,                 -- Estimated live tuples
    n_dead_tup,                 -- Estimated dead tuples
    n_mod_since_analyze,        -- Tuples modified since last ANALYZE
    last_vacuum,                -- Last manual vacuum time
    last_autovacuum,            -- Last autovacuum time
    last_analyze,               -- Last manual analyze time
    last_autoanalyze,           -- Last autoanalyze time
    vacuum_count,               -- Manual vacuum count
    autovacuum_count,           -- Autovacuum count
    analyze_count,              -- Manual analyze count
    autoanalyze_count           -- Autoanalyze count
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;

-- Tables with high dead tuple counts (need vacuum)
SELECT 
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_tuple_percent,
    last_autovacuum,
    last_vacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_tuple_percent DESC
LIMIT 20;

-- Tables with many sequential scans (candidates for indexing)
SELECT 
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    CASE 
        WHEN seq_scan + idx_scan = 0 THEN 0
        ELSE round(100.0 * seq_scan / (seq_scan + idx_scan), 2)
    END AS seq_scan_percent,
    pg_size_pretty(pg_relation_size(schemaname||'.'||relname)) AS table_size
FROM pg_stat_user_tables
WHERE seq_scan > 100
ORDER BY seq_tup_read DESC
LIMIT 20;

-- Tables needing ANALYZE (statistics outdated)
SELECT 
    schemaname,
    relname,
    n_mod_since_analyze,
    n_live_tup,
    round(100.0 * n_mod_since_analyze / NULLIF(n_live_tup, 0), 2) AS percent_modified,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_mod_since_analyze > 1000
ORDER BY n_mod_since_analyze DESC
LIMIT 20;
```

#### pg_stat_user_indexes - Index Statistics

```sql
-- Index usage statistics
SELECT 
    schemaname,
    tablename,
    indexrelname AS index_name,
    idx_scan,                   -- Number of index scans
    idx_tup_read,               -- Tuples read from index
    idx_tup_fetch,              -- Tuples fetched from table
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Unused indexes (candidates for removal)
SELECT 
    schemaname,
    tablename,
    indexrelname AS index_name,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelname NOT LIKE '%_pkey'  -- Exclude primary keys
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index efficiency (how many tuples fetched per scan)
SELECT 
    schemaname,
    tablename,
    indexrelname AS index_name,
    idx_scan,
    CASE 
        WHEN idx_scan = 0 THEN 0
        ELSE round(idx_tup_fetch::numeric / idx_scan, 2)
    END AS avg_tuples_per_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan > 0
ORDER BY idx_scan DESC
LIMIT 20;
```

#### pg_stat_database - Database-Wide Statistics

```sql
-- Database-wide statistics
SELECT 
    datname AS database_name,
    numbackends AS connections,
    xact_commit AS transactions_committed,
    xact_rollback AS transactions_rolled_back,
    round(100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback, 0), 2) AS rollback_percent,
    blks_read AS blocks_read_from_disk,
    blks_hit AS blocks_read_from_cache,
    round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted,
    conflicts,
    temp_files,
    temp_bytes,
    deadlocks,
    pg_size_pretty(pg_database_size(datname)) AS database_size,
    stats_reset AS statistics_reset_time
FROM pg_stat_database
WHERE datname = current_database();
```

#### pg_statio_user_tables - I/O Statistics

```sql
-- Table I/O statistics
SELECT 
    schemaname,
    relname AS table_name,
    heap_blks_read,             -- Blocks read from disk
    heap_blks_hit,              -- Blocks read from cache
    CASE 
        WHEN heap_blks_hit + heap_blks_read = 0 THEN 0
        ELSE round(100.0 * heap_blks_hit / (heap_blks_hit + heap_blks_read), 2)
    END AS cache_hit_ratio,
    idx_blks_read,              -- Index blocks read from disk
    idx_blks_hit,               -- Index blocks read from cache
    toast_blks_read,            -- TOAST blocks read from disk
    toast_blks_hit              -- TOAST blocks read from cache
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY heap_blks_read DESC
LIMIT 20;

-- Tables with poor cache hit ratio
SELECT 
    schemaname,
    relname,
    heap_blks_read,
    heap_blks_hit,
    round(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS cache_hit_ratio,
    pg_size_pretty(pg_relation_size(schemaname||'.'||relname)) AS table_size
FROM pg_statio_user_tables
WHERE heap_blks_hit + heap_blks_read > 1000
ORDER BY cache_hit_ratio ASC
LIMIT 20;
```

### Other Useful System Tables

#### pg_settings - Configuration Parameters

```sql
-- View all configuration settings
SELECT 
    name,
    setting,
    unit,
    category,
    short_desc,
    source,  -- How setting was determined (default, file, override, etc.)
    context  -- When setting can be changed (postmaster, sighup, user, etc.)
FROM pg_settings
WHERE name LIKE '%vacuum%'
ORDER BY name;

-- View non-default settings
SELECT 
    name,
    setting,
    source,
    short_desc
FROM pg_settings
WHERE source != 'default'
ORDER BY name;

-- View settings that require restart
SELECT 
    name,
    setting,
    short_desc
FROM pg_settings
WHERE context = 'postmaster'
AND source != 'default'
ORDER BY name;
```

#### pg_stat_statements - Query Statistics

```sql
-- Enable extension (requires configuration)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top queries by total time
SELECT 
    queryid,
    left(query, 100) AS query_snippet,
    calls,
    total_exec_time / 1000 AS total_seconds,
    mean_exec_time AS avg_ms,
    min_exec_time AS min_ms,
    max_exec_time AS max_ms,
    stddev_exec_time AS stddev_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Slowest queries on average
SELECT 
    queryid,
    left(query, 100) AS query_snippet,
    calls,
    mean_exec_time AS avg_ms,
    max_exec_time AS max_ms
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Most frequently called queries
SELECT 
    queryid,
    left(query, 100) AS query_snippet,
    calls,
    total_exec_time / 1000 AS total_seconds,
    mean_exec_time AS avg_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;

-- Queries with most rows returned
SELECT 
    queryid,
    left(query, 100) AS query_snippet,
    calls,
    rows,
    rows / calls AS avg_rows_per_call
FROM pg_stat_statements
WHERE calls > 0
ORDER BY rows DESC
LIMIT 20;
```

#### pg_tables - Table Information

```sql
-- View table ownership and settings
SELECT 
    schemaname,
    tablename,
    tableowner,
    tablespace,
    hasindexes,
    hasrules,
    hastriggers
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;
```

#### pg_indexes - Index Information

```sql
-- View all indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    indexdef  -- CREATE INDEX statement
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;

-- Find duplicate indexes
SELECT 
    a.schemaname,
    a.tablename,
    a.indexname AS index1,
    b.indexname AS index2,
    a.indexdef
FROM pg_indexes a
JOIN pg_indexes b 
    ON a.schemaname = b.schemaname
    AND a.tablename = b.tablename
    AND a.indexdef = b.indexdef
    AND a.indexname < b.indexname
WHERE a.schemaname NOT IN ('pg_catalog', 'information_schema');
```

### Practical Monitoring Queries

#### Database Health Check

```sql
-- Comprehensive database health check
SELECT 
    'Database Size' AS metric,
    pg_size_pretty(pg_database_size(current_database())) AS value
UNION ALL
SELECT 
    'Cache Hit Ratio',
    round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit + blks_read), 0), 2)::text || '%'
FROM pg_stat_database
WHERE datname = current_database()
UNION ALL
SELECT 
    'Active Connections',
    count(*)::text
FROM pg_stat_activity
WHERE state = 'active'
UNION ALL
SELECT 
    'Idle in Transaction',
    count(*)::text
FROM pg_stat_activity
WHERE state = 'idle in transaction'
UNION ALL
SELECT 
    'Deadlocks',
    deadlocks::text
FROM pg_stat_database
WHERE datname = current_database()
UNION ALL
SELECT 
    'Tables Needing Vacuum',
    count(*)::text
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000;
```

#### Table Bloat Estimation

```sql
-- Estimate table bloat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS bloat_percent,
    CASE 
        WHEN n_dead_tup > n_live_tup * 0.2 THEN 'High bloat - vacuum recommended'
        WHEN n_dead_tup > n_live_tup * 0.1 THEN 'Moderate bloat - monitor'
        ELSE 'OK'
    END AS recommendation
FROM pg_stat_user_tables
WHERE n_live_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### Best Practices

1. **Monitor pg_stat_activity regularly** to understand connection patterns
2. **Reset statistics after major changes** using `pg_stat_reset()`
3. **Use pg_stat_statements** for query performance analysis
4. **Check cache hit ratios** - aim for >99% for steady workloads
5. **Monitor table and index bloat** to prevent performance degradation
6. **Track autovacuum activity** to ensure maintenance is occurring
7. **Identify unused indexes** to reduce write overhead
8. **Watch for sequential scans** on large tables
9. **Monitor deadlock frequency** and investigate patterns
10. **Set up automated monitoring** for key metrics

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/monitoring-stats.html">PostgreSQL Statistics Collector</a>
  - <a href="https://www.postgresql.org/docs/18/catalogs.html">System Catalogs</a>
  - <a href="https://www.postgresql.org/docs/18/pgstatstatements.html">pg_stat_statements Extension</a>

## 12. Replication Types

PostgreSQL offers two main types of replication: **Physical Replication** (also known as WAL streaming or binary replication) and **Logical Replication**. Each has distinct characteristics, advantages, and use cases.

### Physical Replication (WAL Streaming)

Physical replication creates an exact byte-for-byte copy of the entire database cluster. The standby server replays WAL (Write-Ahead Log) records from the primary to maintain an identical copy.

#### How Physical Replication Works

1. **Primary server** generates WAL records for all changes
2. **WAL sender process** on primary streams WAL to standby
3. **WAL receiver process** on standby receives WAL
4. **Startup/recovery process** on standby replays WAL
5. **Standby is read-only** (can accept SELECT queries)

#### Advantages of Physical Replication

- **Complete database copy**: All databases, tables, and objects are replicated
- **Very low overhead**: Minimal impact on primary server performance
- **Crash recovery**: Standby can be promoted to primary quickly
- **Guaranteed consistency**: Exact copy of primary at block level
- **Simple to set up**: Fewer configuration options
- **High availability**: Foundation for automatic failover
- **No version restrictions**: Works with all PostgreSQL data types and extensions

#### Disadvantages of Physical Replication

- **All-or-nothing**: Cannot replicate specific tables or databases
- **Same PostgreSQL version**: Standby must be same major version as primary
- **Read-only standby**: Cannot write to standby (except hot_standby_feedback)
- **Entire cluster replicated**: All databases replicated together
- **Cannot filter data**: All data is replicated

#### Setting Up Physical Replication

**On Primary Server:**

```bash
# 1. Configure postgresql.conf
# Enable WAL archiving and replication
wal_level = replica                  # replica or higher
max_wal_senders = 5                  # Number of concurrent WAL sender processes
wal_keep_size = 1GB                  # Amount of WAL to keep for standby (PostgreSQL 13+)
# For older versions: wal_keep_segments = 64

# Optional: Synchronous replication
# synchronous_standby_names = 'standby1'  # Name of standby for synchronous replication
# synchronous_commit = on

# Archive WAL files (optional but recommended for PITR)
archive_mode = on
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
```

```bash
# 2. Configure pg_hba.conf to allow replication connections
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    replication     replicator      192.168.1.0/24          scram-sha-256
```

```sql
-- 3. Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password';

-- 4. Create replication slot (prevents WAL deletion while standby is down)
SELECT pg_create_physical_replication_slot('standby_slot');

-- Check replication slots
SELECT * FROM pg_replication_slots;
```

```bash
# 5. Restart PostgreSQL
pg_ctl restart -D /var/lib/postgresql/data
```

**On Standby Server:**

```bash
# 1. Stop PostgreSQL if running
pg_ctl stop -D /var/lib/postgresql/data

# 2. Remove existing data directory
rm -rf /var/lib/postgresql/data/*

# 3. Take base backup from primary
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/data \
  -P -Xs -R -S standby_slot

# Options:
# -h primary_host: Primary server hostname/IP
# -U replicator: Replication user
# -D: Target data directory
# -P: Show progress
# -Xs: Stream WAL during backup
# -R: Create standby.signal and configure postgresql.auto.conf
# -S standby_slot: Use replication slot

# Alternative: Manual configuration without -R flag
# Create standby.signal file
touch /var/lib/postgresql/data/standby.signal

# Configure postgresql.auto.conf or postgresql.conf
cat >> /var/lib/postgresql/data/postgresql.auto.conf << EOF
primary_conninfo = 'host=primary_host port=5432 user=replicator password=strong_password'
primary_slot_name = 'standby_slot'
restore_command = 'cp /mnt/wal_archive/%f %p'  # Optional: for PITR
EOF
```

```bash
# 4. Start standby server
pg_ctl start -D /var/lib/postgresql/data
```

#### Monitoring Physical Replication

**On Primary:**

```sql
-- View replication status
SELECT 
    client_addr AS standby_ip,
    application_name,
    state,
    sync_state,              -- async, potential, sync, quorum
    sent_lsn,                -- WAL position sent
    write_lsn,               -- WAL position written to standby disk
    flush_lsn,               -- WAL position flushed to standby disk
    replay_lsn,              -- WAL position replayed on standby
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
    write_lag,               -- Time between commit and write confirmation
    flush_lag,               -- Time between commit and flush confirmation
    replay_lag               -- Time between commit and replay on standby
FROM pg_stat_replication;

-- Check replication slot status
SELECT 
    slot_name,
    slot_type,
    active,
    active_pid,
    restart_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_wal_bytes
FROM pg_replication_slots;
```

**On Standby:**

```sql
-- Check if server is in recovery mode (standby)
SELECT pg_is_in_recovery();  -- Returns true

-- View recovery status
SELECT 
    pg_last_wal_receive_lsn() AS last_received,    -- Last WAL received
    pg_last_wal_replay_lsn() AS last_replayed,     -- Last WAL replayed
    pg_last_xact_replay_timestamp() AS last_replay_time,  -- Time of last replayed transaction
    pg_wal_lsn_diff(
        pg_last_wal_receive_lsn(),
        pg_last_wal_replay_lsn()
    ) AS lag_bytes;

-- Check WAL receiver status
SELECT * FROM pg_stat_wal_receiver;
```

#### Synchronous vs Asynchronous Replication

**Asynchronous (default):**
- Primary doesn't wait for standby confirmation
- Better write performance on primary
- Risk of data loss if primary fails before WAL reaches standby
- Suitable for most use cases

```conf
# postgresql.conf on primary
synchronous_commit = off  # or 'local' - don't wait for standby
```

**Synchronous:**
- Primary waits for standby to confirm WAL receipt/replay
- Guarantees zero data loss on failover
- Write latency increased by network + standby delay
- Suitable for critical data requiring no data loss

```conf
# postgresql.conf on primary
synchronous_standby_names = 'standby1'  # Name from application_name
synchronous_commit = on                  # or 'remote_write' or 'remote_apply'

# synchronous_commit options:
# - on: Wait for standby to flush WAL to disk
# - remote_write: Wait for standby to write to OS (not flushed)
# - remote_apply: Wait for standby to replay WAL (slowest, most consistent reads)
```

### Logical Replication

Logical replication replicates data changes at a higher level (rows, not WAL blocks). It allows selective replication and works across PostgreSQL versions.

#### How Logical Replication Works

1. **Publication** on source database defines what to replicate
2. **Subscription** on target database subscribes to publication
3. **Logical decoding** converts WAL to logical changes (INSERT/UPDATE/DELETE)
4. **Replication slot** tracks position in logical WAL stream
5. **Apply worker** applies changes to target database
6. **Target is writable** - can accept other changes too

#### Advantages of Logical Replication

- **Selective replication**: Replicate specific tables or databases
- **Cross-version replication**: Different PostgreSQL versions (with limits)
- **Writable subscriber**: Target can have additional tables/data
- **Data transformation**: Can apply to different schema
- **Row filtering**: Replicate subset of rows (using WHERE clause)
- **Column filtering**: Replicate subset of columns
- **Multi-master possibilities**: With conflict resolution (e.g., BDR)
- **Migration tool**: Move data between clusters with minimal downtime

#### Disadvantages of Logical Replication

- **Higher overhead**: More CPU and I/O than physical replication
- **DDL not replicated**: Schema changes must be applied manually
- **Sequences not replicated**: Must sync sequences separately
- **Large objects not supported**: Cannot replicate large objects
- **Some data types restricted**: Cannot replicate some data types
- **Initial data copy**: Full table copy at subscription start
- **Potential conflicts**: If target is writable, conflicts possible

#### Setting Up Logical Replication

**On Source Database (Publisher):**

```conf
# 1. Configure postgresql.conf
wal_level = logical              # Required for logical replication
max_replication_slots = 4        # Number of replication slots
max_wal_senders = 4              # Number of WAL sender processes
```

```sql
-- 2. Restart PostgreSQL after configuration change
-- Then create publication

-- Replicate all tables in database
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- Replicate specific tables
CREATE PUBLICATION my_pub FOR TABLE users, orders, products;

-- Replicate with row filter (PostgreSQL 15+)
CREATE PUBLICATION active_users FOR TABLE users WHERE (status = 'active');

-- Replicate with column list (PostgreSQL 15+)
CREATE PUBLICATION user_pub FOR TABLE users (user_id, email, name);

-- View publications
SELECT * FROM pg_publication;

-- View tables in publication
SELECT * FROM pg_publication_tables WHERE pubname = 'my_pub';
```

**On Target Database (Subscriber):**

```sql
-- 1. Create subscription
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=source_host port=5432 dbname=sourcedb user=replication_user password=secret'
PUBLICATION my_pub;

-- Options:
-- - copy_data: Whether to copy existing data (default: true)
-- - create_slot: Whether to create replication slot (default: true)
-- - enabled: Whether to start replication immediately (default: true)

-- Create subscription without initial data copy
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=source_host dbname=sourcedb user=replication_user password=secret'
PUBLICATION my_pub
WITH (copy_data = false);

-- View subscriptions
SELECT * FROM pg_subscription;

-- Check subscription status
SELECT 
    subname,
    subenabled,
    subslotname,
    subpublications
FROM pg_subscription;

-- Monitor replication workers
SELECT * FROM pg_stat_subscription;
```

#### Managing Logical Replication

```sql
-- On Publisher:

-- Add table to publication
ALTER PUBLICATION my_pub ADD TABLE new_table;

-- Remove table from publication
ALTER PUBLICATION my_pub DROP TABLE old_table;

-- Drop publication
DROP PUBLICATION my_pub;

-- On Subscriber:

-- Disable subscription temporarily
ALTER SUBSCRIPTION my_sub DISABLE;

-- Enable subscription
ALTER SUBSCRIPTION my_sub ENABLE;

-- Refresh subscription (re-sync table list)
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;

-- Drop subscription
DROP SUBSCRIPTION my_sub;

-- Monitor replication lag
SELECT 
    subname,
    latest_end_lsn,
    latest_end_time,
    now() - latest_end_time AS replication_lag
FROM pg_stat_subscription;
```

#### Handling Schema Changes

Logical replication does NOT replicate DDL. Schema changes must be applied to both publisher and subscriber:

```sql
-- 1. On Publisher: Add column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- 2. On Subscriber: Apply same change
-- Option A: Disable subscription, apply DDL, re-enable
ALTER SUBSCRIPTION my_sub DISABLE;
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER SUBSCRIPTION my_sub ENABLE;

-- Option B: Let subscription continue (if adding nullable column)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- Subscription continues automatically

-- 3. Refresh publication if needed
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;
```

### Comparison: Physical vs Logical Replication

| Feature | Physical Replication | Logical Replication |
|---------|---------------------|---------------------|
| **Granularity** | Entire cluster | Specific tables/databases |
| **Version Support** | Same major version | Different versions (with limits) |
| **Overhead** | Very low | Moderate to high |
| **Standby Writable** | No (read-only) | Yes |
| **DDL Replication** | Yes (automatic) | No (manual) |
| **Use Case** | HA, disaster recovery | Selective sync, migration, consolidation |
| **Data Filtering** | No | Yes (rows and columns) |
| **Initial Sync** | Full cluster copy | Table-by-table copy |
| **Conflict Handling** | N/A (read-only standby) | Manual resolution needed |
| **Sequences** | Replicated | Not replicated |
| **Large Objects** | Supported | Not supported |
| **Failover Speed** | Fast (promote standby) | Slower (manual process) |

### Use Cases

**Use Physical Replication When:**
- High availability and disaster recovery are primary goals
- Need fast failover with guaranteed consistency
- Want complete cluster replica
- Low overhead is critical
- All data should be replicated

**Use Logical Replication When:**
- Need to replicate specific tables/databases
- Upgrading PostgreSQL major version (replication during migration)
- Consolidating data from multiple sources
- Need to transform data during replication
- Target database needs other independent tables/data
- Filtering rows or columns
- Different schemas on source and target

### Hybrid Approach

You can use both types together:

```
Primary (Physical) → Standby1 (Physical) → Standby2 (Physical)
    ↓ (Logical)
Target Database (different schema/version)
```

### Best Practices

1. **For HA: Use physical replication** with automatic failover (e.g., Patroni, repmgr)
2. **Monitor replication lag** continuously
3. **Use replication slots** to prevent WAL deletion
4. **Test failover procedures** regularly
5. **For logical replication:** Keep schemas in sync manually
6. **Monitor disk space** on primary (WAL accumulation)
7. **Use synchronous replication** for zero data loss requirements
8. **Document failover procedures** clearly
9. **Consider cascading replication** for multiple standbys
10. **Monitor subscription workers** for logical replication

- **Further Reading**: 
  - <a href="https://www.postgresql.org/docs/18/warm-standby.html">PostgreSQL Physical Replication</a>
  - <a href="https://www.postgresql.org/docs/18/logical-replication.html">PostgreSQL Logical Replication</a>
  - <a href="https://www.postgresql.org/docs/18/high-availability.html">High Availability and Load Balancing</a>
  - <a href="https://www.postgresql.org/docs/18/runtime-config-replication.html">Replication Configuration</a>

## Conclusion
This guide provides fundamental knowledge about PostgreSQL, serving as a starting point for further exploration. For more advanced topics, refer to the official PostgreSQL documentation and community resources.