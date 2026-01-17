# PostgreSQL Ultimate Guide

## Introduction
This comprehensive guide covers essential and advanced PostgreSQL topics for database enthusiasts, administrators, and professionals. It provides deep insights into PostgreSQL internals, practical examples, and best practices based on the official PostgreSQL 18 documentation.

## Table of Contents
1. [Vacuuming](#1-vacuuming)
2. [Indexing](#2-indexing)
3. [Write-Ahead Logging (WAL)](#3-write-ahead-logging-wal)
4. [Caching](#4-caching)
5. [Transactions](#5-transactions)
6. [Large Object Storage](#6-large-object-storage)
7. [Postmaster Process](#7-postmaster-process)
8. [Transaction ID Wraparound](#8-transaction-id-wraparound)
9. [Explain Plans](#9-explain-plans)
10. [Deadlocks](#10-deadlocks)
11. [Metadata Using System Catalogs](#11-metadata-using-system-catalogs)
12. [Replication Types](#12-replication-types)

---

## 1. Vacuuming

### Overview
Vacuuming is a critical maintenance operation in PostgreSQL that reclaims storage occupied by dead tuples (deleted or obsoleted rows due to MVCC) and updates statistics for the query planner. PostgreSQL's MVCC (Multi-Version Concurrency Control) creates new row versions on UPDATE operations, leaving old versions as "dead tuples" that must be cleaned up.

### Why Vacuuming is Necessary
- **Reclaim Disk Space**: Dead tuples occupy space until vacuumed
- **Prevent Transaction ID Wraparound**: PostgreSQL has a 4 billion transaction limit
- **Update Statistics**: Helps the query planner make optimal decisions
- **Update Visibility Map**: Enables index-only scans
- **Prevent Table Bloat**: Reduces unnecessary storage growth

### Types of Vacuum Operations

#### Standard VACUUM
```sql
-- Vacuum a specific table
VACUUM my_table;

-- Vacuum with verbose output
VACUUM VERBOSE my_table;

-- Vacuum all tables in the database
VACUUM;
```

The standard VACUUM:
- Marks dead tuples as reusable
- Does NOT return space to the operating system
- Can run concurrently with normal operations (non-blocking)
- Updates the free space map (FSM)

#### VACUUM FULL
```sql
-- Full vacuum to reclaim space
VACUUM FULL my_table;

-- Full vacuum with analyze
VACUUM FULL ANALYZE my_table;
```

VACUUM FULL:
- Rewrites the entire table, returning space to the OS
- Requires an exclusive lock (blocks reads and writes)
- Much slower than standard VACUUM
- Should be used sparingly, typically when table bloat is severe

#### VACUUM FREEZE
```sql
-- Freeze tuples to prevent transaction ID wraparound
VACUUM FREEZE my_table;

-- Check the age of tables
SELECT relname, age(relfrozenxid) as xid_age 
FROM pg_class 
WHERE relkind = 'r' 
ORDER BY age(relfrozenxid) DESC 
LIMIT 10;
```

VACUUM FREEZE:
- Freezes transaction IDs to prevent wraparound
- Critical for database health
- Automatically triggered by autovacuum when necessary

### AUTOVACUUM

PostgreSQL includes an automated vacuuming daemon called autovacuum that automatically performs VACUUM and ANALYZE operations.

#### Configuration Parameters
```sql
-- postgresql.conf settings
autovacuum = on                              # Enable autovacuum (default: on)
autovacuum_max_workers = 3                   # Max autovacuum workers
autovacuum_naptime = 1min                    # Time between autovacuum runs
autovacuum_vacuum_threshold = 50             # Min rows before vacuum
autovacuum_vacuum_scale_factor = 0.2         # Fraction of table before vacuum
autovacuum_analyze_threshold = 50            # Min rows before analyze
autovacuum_analyze_scale_factor = 0.1        # Fraction of table before analyze
autovacuum_vacuum_cost_delay = 2ms           # Delay between vacuum operations
autovacuum_vacuum_cost_limit = 200           # Cost limit for vacuum

-- Per-table autovacuum settings
ALTER TABLE my_table SET (
    autovacuum_vacuum_scale_factor = 0.1,
    autovacuum_vacuum_threshold = 100,
    autovacuum_analyze_scale_factor = 0.05
);
```

#### Monitoring Autovacuum
```sql
-- Check last vacuum and autovacuum times
SELECT 
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    n_dead_tup,
    n_live_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Monitor running autovacuum processes
SELECT 
    pid,
    now() - xact_start AS duration,
    query
FROM pg_stat_activity
WHERE query LIKE '%autovacuum%'
    AND query NOT LIKE '%pg_stat_activity%';

-- Check table bloat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Best Practices
1. **Keep autovacuum enabled**: It's crucial for database health
2. **Monitor dead tuples**: High counts indicate vacuum isn't keeping up
3. **Tune autovacuum**: Adjust parameters based on workload
4. **Schedule manual vacuums**: For maintenance windows on large tables
5. **Avoid VACUUM FULL in production**: Use it only during planned maintenance
6. **Monitor transaction ID age**: Prevent wraparound issues

### Advanced Example: Vacuum Strategy for High-Write Tables
```sql
-- For a high-write table with millions of rows
ALTER TABLE high_activity_table SET (
    autovacuum_vacuum_scale_factor = 0.05,     -- More aggressive
    autovacuum_vacuum_threshold = 1000,
    autovacuum_vacuum_cost_delay = 10ms,       -- Slower but less impact
    fillfactor = 80                            -- Leave space for HOT updates
);

-- Monitor the effectiveness
SELECT 
    schemaname,
    relname,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count
FROM pg_stat_user_tables 
WHERE relname = 'high_activity_table';
```

- **Further Reading**: [PostgreSQL Vacuuming Documentation](https://www.postgresql.org/docs/18/sql-vacuum.html) | [Routine Vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html)

## 2. Indexing

### Overview
Indexes are specialized data structures that dramatically improve query performance by providing fast data lookup paths. However, they come with trade-offs: they consume storage space and slow down write operations (INSERT, UPDATE, DELETE).

### Index Types in PostgreSQL

#### B-tree Indexes (Default)
B-tree is the default and most commonly used index type. It handles equality and range queries efficiently.

```sql
-- Create a standard B-tree index
CREATE INDEX idx_users_email ON users(email);

-- Multi-column index (composite index)
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Partial index (indexes only a subset of rows)
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Unique index
CREATE UNIQUE INDEX idx_users_username ON users(username);
```

**Use Cases**:
- Equality comparisons: `column = value`
- Range queries: `column > value`, `column BETWEEN x AND y`
- Pattern matching: `column LIKE 'prefix%'` (only prefix matching)
- Sorting: `ORDER BY column`

#### Hash Indexes
Hash indexes are optimized for simple equality comparisons.

```sql
CREATE INDEX idx_products_sku_hash ON products USING hash(sku);
```

**Use Cases**:
- Simple equality: `column = value`
- **NOT suitable for**: Range queries, sorting, or pattern matching

**Note**: Since PostgreSQL 10, hash indexes are WAL-logged and crash-safe.

#### GiST Indexes (Generalized Search Tree)
GiST indexes support various data types and operations, particularly for geometric and full-text search.

```sql
-- Geometric data indexing
CREATE INDEX idx_locations_point ON locations USING gist(coordinates);

-- Full-text search
CREATE INDEX idx_documents_search ON documents USING gist(to_tsvector('english', content));

-- Range types
CREATE INDEX idx_reservations_period ON reservations USING gist(reservation_period);
```

**Use Cases**:
- Geometric data: points, boxes, circles
- Full-text search
- Range types
- Network address types (inet, cidr)

#### GIN Indexes (Generalized Inverted Index)
GIN indexes are ideal for indexing composite values like arrays, JSONB, and full-text search.

```sql
-- Array indexing
CREATE INDEX idx_tags_array ON articles USING gin(tags);

-- JSONB indexing
CREATE INDEX idx_metadata_jsonb ON products USING gin(metadata);

-- Full-text search (preferred over GiST)
CREATE INDEX idx_documents_fts ON documents USING gin(to_tsvector('english', content));

-- Queries that benefit
SELECT * FROM articles WHERE tags @> ARRAY['postgresql', 'database'];
SELECT * FROM products WHERE metadata @> '{"category": "electronics"}';
```

**Use Cases**:
- Array containment queries
- JSONB data
- Full-text search (faster than GiST for static documents)
- Multi-key searches

#### BRIN Indexes (Block Range Index)
BRIN indexes are extremely compact and efficient for very large tables where data is naturally ordered.

```sql
-- Time-series data
CREATE INDEX idx_logs_timestamp_brin ON logs USING brin(created_at);

-- Sequential data
CREATE INDEX idx_orders_id_brin ON orders USING brin(id);

-- Configure pages per range
CREATE INDEX idx_sensor_data_brin ON sensor_data 
USING brin(timestamp) WITH (pages_per_range = 128);
```

**Use Cases**:
- Time-series data
- Naturally ordered data (e.g., auto-incrementing IDs)
- Very large tables where space is a concern
- Data with physical correlation to insertion order

**Advantages**:
- Extremely small index size (100-1000x smaller than B-tree)
- Minimal maintenance overhead

#### SP-GiST Indexes (Space-Partitioned GiST)
SP-GiST supports partitioned search trees for non-balanced data structures.

```sql
-- Phone numbers, IP addresses
CREATE INDEX idx_phone_spgist ON contacts USING spgist(phone_number);

-- Ranges
CREATE INDEX idx_ip_ranges ON network_blocks USING spgist(ip_range);
```

**Use Cases**:
- Phone numbers
- IP addresses
- Ranges with hierarchical structure

### Index Creation Strategies

#### Concurrent Index Creation
```sql
-- Create index without locking table for writes
CREATE INDEX CONCURRENTLY idx_users_created_at ON users(created_at);

-- If it fails, clean up invalid index
DROP INDEX CONCURRENTLY IF EXISTS idx_users_created_at;
```

#### Expression Indexes
```sql
-- Index on computed values
CREATE INDEX idx_users_lower_email ON users(lower(email));

-- Query must match the expression
SELECT * FROM users WHERE lower(email) = 'user@example.com';
```

#### Covering Indexes (INCLUDE clause)
```sql
-- Include additional columns for index-only scans
CREATE INDEX idx_orders_customer_include 
ON orders(customer_id) 
INCLUDE (order_date, total_amount);

-- Query can be satisfied entirely from index
SELECT order_date, total_amount 
FROM orders 
WHERE customer_id = 123;
```

### Index Maintenance

#### Monitoring Index Usage
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

-- Find unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Check index bloat
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(indexrelid) DESC;
```

#### Rebuilding Indexes
```sql
-- Standard rebuild (locks table)
REINDEX INDEX idx_users_email;

-- Rebuild entire table's indexes
REINDEX TABLE users;

-- Concurrent rebuild (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_users_email;

-- Rebuild all indexes in database
REINDEX DATABASE mydb;
```

### Best Practices

1. **Index Selectivity**: Index high-selectivity columns (many distinct values)
   ```sql
   -- Check column selectivity
   SELECT 
       COUNT(DISTINCT email) * 100.0 / COUNT(*) as selectivity_percent
   FROM users;
   ```

2. **Column Order in Composite Indexes**: Place the most selective column first
   ```sql
   -- Good: status has few values, customer_id has many
   CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
   ```

3. **Avoid Over-Indexing**: Each index adds overhead to write operations
   ```sql
   -- Monitor write overhead
   SELECT 
       relname,
       n_tup_ins,
       n_tup_upd,
       n_tup_del,
       pg_size_pretty(pg_total_relation_size(relid))
   FROM pg_stat_user_tables
   ORDER BY n_tup_upd + n_tup_del DESC;
   ```

4. **Use Partial Indexes**: Index only relevant rows
   ```sql
   -- Only index unpaid orders
   CREATE INDEX idx_unpaid_orders ON orders(customer_id) 
   WHERE status = 'unpaid';
   ```

5. **Regular Maintenance**: Keep indexes healthy
   ```sql
   -- Scheduled maintenance
   REINDEX TABLE CONCURRENTLY large_table;
   ANALYZE large_table;
   ```

### Advanced Example: Optimizing a Complex Query

```sql
-- Before: Slow query
EXPLAIN ANALYZE
SELECT 
    u.username,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.active = true
    AND o.created_at >= NOW() - INTERVAL '30 days'
GROUP BY u.id, u.username
ORDER BY total_spent DESC
LIMIT 10;

-- Create optimal indexes
CREATE INDEX CONCURRENTLY idx_users_active ON users(active) WHERE active = true;
CREATE INDEX CONCURRENTLY idx_orders_user_created ON orders(user_id, created_at) 
    INCLUDE (total);

-- After: Fast query with index-only scans
-- Run ANALYZE to update statistics
ANALYZE users;
ANALYZE orders;
```

### Visual Guide to Index Selection

```
┌─────────────────────────────────────────────────────────────┐
│ Query Type                    → Recommended Index Type      │
├─────────────────────────────────────────────────────────────┤
│ Equality (=)                  → B-tree or Hash              │
│ Range (<, >, BETWEEN)         → B-tree                      │
│ Text search (LIKE 'prefix%')  → B-tree                      │
│ Full-text search              → GIN                         │
│ Array/JSONB containment       → GIN                         │
│ Geometric/spatial queries     → GiST                        │
│ Time-series (ordered data)    → BRIN                        │
│ IP addresses/hierarchical     → SP-GiST                     │
└─────────────────────────────────────────────────────────────┘
```

- **Further Reading**: [PostgreSQL Indexes Documentation](https://www.postgresql.org/docs/18/indexes.html) | [Index Types](https://www.postgresql.org/docs/18/indexes-types.html) | [Index-Only Scans](https://www.postgresql.org/docs/18/indexes-index-only-scans.html)

## 3. Write-Ahead Logging (WAL)

### Overview
Write-Ahead Logging (WAL) is PostgreSQL's foundation for data durability and consistency. The core principle: changes are written to a persistent log **before** they're applied to data files. This ensures crash recovery and enables point-in-time recovery (PITR) and replication.

### How WAL Works

#### WAL Architecture
```
┌──────────────────────────────────────────────────────────────┐
│                     WAL Workflow                              │
├──────────────────────────────────────────────────────────────┤
│  1. Transaction modifies data                                │
│  2. Changes written to WAL buffer in memory                  │
│  3. WAL buffer flushed to WAL files on disk (at COMMIT)     │
│  4. Changes eventually written to data files (checkpoint)    │
│  5. Old WAL files archived or recycled                       │
└──────────────────────────────────────────────────────────────┘
```

#### Key Concepts
- **WAL Segment**: Fixed-size file (default 16MB) containing log records
- **LSN (Log Sequence Number)**: Unique identifier for position in WAL
- **Checkpoint**: Point where all dirty buffers are written to data files
- **WAL Record**: Individual change record in the log

### WAL Configuration

#### Essential Parameters
```sql
-- postgresql.conf settings

-- WAL Level (determines what's logged)
wal_level = replica                      # minimal, replica, or logical
                                         # replica: enables streaming replication
                                         # logical: enables logical replication

-- WAL Write Behavior
fsync = on                               # Force synchronous disk writes
synchronous_commit = on                  # Wait for WAL write before COMMIT returns
wal_sync_method = fdatasync              # OS-specific: fdatasync, fsync, open_sync

-- WAL Buffer and Files
wal_buffers = 16MB                       # WAL buffer size in shared memory
wal_writer_delay = 200ms                 # How often WAL writer flushes
max_wal_size = 1GB                       # Triggers checkpoint when exceeded
min_wal_size = 80MB                      # Minimum WAL kept between checkpoints

-- Checkpoints
checkpoint_timeout = 5min                # Maximum time between checkpoints
checkpoint_completion_target = 0.9       # Spread checkpoint over this fraction
checkpoint_warning = 30s                 # Warn if checkpoints too frequent

-- WAL Archiving
archive_mode = on                        # Enable WAL archiving
archive_command = 'cp %p /archive/%f'    # Command to archive WAL files
archive_timeout = 60s                    # Force segment switch after timeout
```

#### Understanding wal_level

```sql
-- Check current WAL level
SHOW wal_level;

-- minimal: Basic crash recovery only (no replication)
-- replica: Physical replication support (most common)
-- logical: Logical replication support (includes replica features)
```

### Working with WAL

#### Monitoring WAL Activity
```sql
-- Current WAL insert location
SELECT pg_current_wal_lsn();

-- Current WAL flush location (written to disk)
SELECT pg_current_wal_flush_lsn();

-- Calculate WAL generation rate
SELECT 
    pg_current_wal_lsn(),
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 as mb_generated;

-- Monitor WAL files
SELECT 
    name,
    setting,
    unit,
    short_desc
FROM pg_settings
WHERE name LIKE 'wal%' OR name LIKE '%checkpoint%'
ORDER BY name;

-- Check WAL directory size
SELECT 
    pg_size_pretty(
        pg_wal_directory_size()
    ) as wal_size;

-- View checkpoint statistics
SELECT 
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend
FROM pg_stat_bgwriter;
```

#### WAL File Management
```sql
-- List WAL files
SELECT 
    name,
    size,
    modification
FROM pg_ls_waldir()
ORDER BY modification DESC
LIMIT 10;

-- Calculate LSN difference (in bytes)
SELECT pg_wal_lsn_diff('1/12345678', '1/12340000') as bytes_diff;

-- Get WAL file name for LSN
SELECT pg_walfile_name('0/16B4EB8');

-- Get LSN for WAL file
SELECT pg_walfile_name_offset('000000010000000000000002');
```

### WAL Archiving

#### Setting Up WAL Archiving
```bash
# postgresql.conf
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
# Use rsync for remote archiving
# archive_command = 'rsync -a %p backup_server:/archive/%f'
```

```sql
-- Check archiving status
SELECT 
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time,
    stats_reset
FROM pg_stat_archiver;

-- Force WAL segment switch (useful for archiving)
SELECT pg_switch_wal();

-- View archive status
SELECT 
    name,
    setting
FROM pg_settings
WHERE name LIKE 'archive%';
```

#### Advanced Archiving with WAL-G or pgBackRest
```bash
# Example using wal-g (more efficient than cp)
# archive_command = 'wal-g wal-push %p'

# Example using pgBackRest
# archive_command = 'pgbackrest --stanza=main archive-push %p'
```

### WAL and Performance

#### Tuning for Performance
```sql
-- High-performance settings (with caution)
ALTER SYSTEM SET synchronous_commit = off;     -- Don't wait for WAL flush
                                                -- Risk: Can lose recent transactions

-- For data warehousing/bulk loads
ALTER SYSTEM SET wal_level = minimal;          -- Reduces WAL overhead
ALTER SYSTEM SET full_page_writes = off;       -- Risk: corruption if crash

-- Restore safe settings
ALTER SYSTEM SET synchronous_commit = on;
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET full_page_writes = on;
SELECT pg_reload_conf();
```

#### Analyzing WAL Impact
```sql
-- Monitor transaction commit timing
SELECT 
    pid,
    usename,
    application_name,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'IO' 
    AND wait_event LIKE 'WAL%';

-- Check for checkpoint spikes
SELECT 
    checkpoints_timed,      -- Scheduled checkpoints
    checkpoints_req,        -- Requested checkpoints (indicates tuning needed)
    buffers_checkpoint,
    buffers_clean,
    maxwritten_clean,
    buffers_backend,
    buffers_alloc
FROM pg_stat_bgwriter;

-- If checkpoints_req is high, increase max_wal_size
ALTER SYSTEM SET max_wal_size = '2GB';
SELECT pg_reload_conf();
```

### WAL and Recovery

#### Point-in-Time Recovery (PITR)
```bash
# 1. Ensure WAL archiving is configured
# 2. Take a base backup
pg_basebackup -D /backup/base -Fp -Xs -P

# 3. To recover to specific point:
# Create recovery.signal file
touch /data/recovery.signal

# Configure postgresql.conf or recovery.conf
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
# Or: recovery_target_xid = '12345'
# Or: recovery_target_lsn = '1/12345678'

# 4. Start PostgreSQL - it will replay WAL until target
```

```sql
-- Monitor recovery progress
SELECT 
    pg_is_in_recovery(),
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn(),
    pg_last_xact_replay_timestamp();

-- Calculate recovery lag
SELECT 
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

#### Crash Recovery
When PostgreSQL crashes, it automatically performs crash recovery on startup:
1. Reads the last checkpoint from control file
2. Replays WAL records from checkpoint to end
3. Ensures database consistency

```sql
-- Check last checkpoint info
SELECT 
    checkpoint_lsn,
    redo_lsn,
    timeline_id
FROM pg_control_checkpoint();
```

### WAL Internals

#### WAL Record Structure
```
┌────────────────────────────────────────────────────────┐
│ WAL Record Components                                  │
├────────────────────────────────────────────────────────┤
│ • Header: Record type, transaction ID, LSN            │
│ • Main Data: Actual data changes                      │
│ • Backup Blocks: Full page images (for torn pages)   │
└────────────────────────────────────────────────────────┘
```

#### Full Page Writes
```sql
-- View full page write setting
SHOW full_page_writes;

-- When enabled, first modification after checkpoint writes entire page
-- Protects against partial page writes (torn pages)
-- Increases WAL volume but essential for reliability
```

### Advanced Examples

#### Implementing WAL-Based Replication
```sql
-- On primary server
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 10;
ALTER SYSTEM SET wal_keep_size = '1GB';  -- PostgreSQL 13+
SELECT pg_reload_conf();

-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_password';

-- On standby server
-- Use pg_basebackup to initialize
-- Then configure primary_conninfo in postgresql.conf
```

#### Monitoring WAL Performance
```sql
-- Create monitoring view
CREATE OR REPLACE VIEW wal_stats AS
SELECT 
    pg_current_wal_lsn() as current_lsn,
    pg_walfile_name(pg_current_wal_lsn()) as current_wal_file,
    pg_size_pretty(pg_wal_directory_size()) as wal_dir_size,
    (SELECT count(*) FROM pg_ls_waldir()) as wal_file_count,
    (SELECT setting FROM pg_settings WHERE name = 'wal_level') as wal_level,
    (SELECT setting FROM pg_settings WHERE name = 'max_wal_size') as max_wal_size;

-- Query the view
SELECT * FROM wal_stats;
```

### Best Practices

1. **Always keep wal_level = replica**: Enables replication without restart
2. **Monitor checkpoint frequency**: Tune max_wal_size if checkpoints_req is high
3. **Archive WAL regularly**: Essential for PITR and disaster recovery
4. **Test recovery procedures**: Regularly verify backups and WAL archives
5. **Use synchronous_commit wisely**: Balance durability vs. performance
6. **Monitor WAL directory size**: Prevent disk space issues
7. **Keep full_page_writes = on**: Essential for crash safety

- **Further Reading**: [PostgreSQL WAL Documentation](https://www.postgresql.org/docs/18/wal.html) | [WAL Configuration](https://www.postgresql.org/docs/18/wal-configuration.html) | [WAL Internals](https://www.postgresql.org/docs/18/wal-internals.html)

## 4. Caching

### Overview
PostgreSQL uses sophisticated caching mechanisms to minimize disk I/O and improve query performance. Understanding and tuning these caches is crucial for optimal database performance.

### Cache Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                PostgreSQL Memory Architecture               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Shared Memory (shared_buffers)              │  │
│  │  • Database pages cached for all connections          │  │
│  │  • Shared across all PostgreSQL processes             │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Per-Connection Memory (work_mem)              │  │
│  │  • Sort operations                                    │  │
│  │  • Hash tables for joins                             │  │
│  │  • Separate for each operation in each connection    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │          Maintenance Memory (maintenance_work_mem)    │  │
│  │  • VACUUM operations                                  │  │
│  │  • CREATE INDEX                                       │  │
│  │  • ALTER TABLE operations                            │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
         ↓ Cache miss → OS Page Cache → Disk
```

### Key Memory Parameters

#### shared_buffers
The most important memory parameter - caches database pages in shared memory.

```sql
-- View current setting
SHOW shared_buffers;

-- Recommended: 25% of system RAM (up to 40% for dedicated servers)
-- postgresql.conf
shared_buffers = 4GB          # For 16GB RAM system
shared_buffers = 8GB          # For 32GB RAM system

-- Monitor buffer cache hit ratio (should be > 99%)
SELECT 
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit)  as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100 
        as cache_hit_ratio
FROM pg_statio_user_tables;

-- Detailed cache statistics
SELECT 
    schemaname,
    tablename,
    heap_blks_read,
    heap_blks_hit,
    CASE 
        WHEN heap_blks_hit + heap_blks_read = 0 THEN 0
        ELSE round(heap_blks_hit::numeric / (heap_blks_hit + heap_blks_read) * 100, 2)
    END as cache_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY heap_blks_read DESC
LIMIT 20;
```

#### effective_cache_size
Informs the query planner about OS and PostgreSQL cache size (doesn't allocate memory).

```sql
-- View current setting
SHOW effective_cache_size;

-- Recommended: 50-75% of total system RAM
-- postgresql.conf
effective_cache_size = 12GB    # For 16GB RAM system
effective_cache_size = 24GB    # For 32GB RAM system

-- This affects query planning decisions
-- Higher values make index scans more likely to be chosen
```

#### work_mem
Memory for sort and hash operations per operation per connection.

```sql
-- View current setting
SHOW work_mem;

-- Recommended: Start with 4MB, adjust based on workload
-- postgresql.conf
work_mem = 16MB               # Conservative default
work_mem = 64MB               # For complex queries

-- WARNING: Multiple operations can use work_mem simultaneously!
-- Total usage = work_mem × operations × connections

-- Set per-session for specific queries
SET work_mem = '256MB';
-- Complex query here
RESET work_mem;

-- Monitor if queries are spilling to disk
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    temp_blks_written,
    temp_blks_read
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 10;
```

#### maintenance_work_mem
Memory for maintenance operations (VACUUM, CREATE INDEX, etc.).

```sql
-- View current setting
SHOW maintenance_work_mem;

-- Recommended: 5-10% of RAM (up to 1-2GB)
-- postgresql.conf
maintenance_work_mem = 1GB

-- Can be increased for one-time operations
SET maintenance_work_mem = '2GB';
CREATE INDEX CONCURRENTLY idx_large_table ON large_table(column_name);
RESET maintenance_work_mem;
```

### Buffer Management

#### Monitoring Buffer Usage
```sql
-- View buffer usage by relation
SELECT 
    c.relname,
    pg_size_pretty(pg_relation_size(c.oid)) as size,
    count(*) as buffers,
    round(count(*) * 8192.0 / pg_relation_size(c.oid) * 100, 2) as percent_cached
FROM pg_buffercache b
INNER JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname, c.oid
ORDER BY count(*) DESC
LIMIT 20;

-- Overall buffer cache usage
SELECT 
    CASE
        WHEN c.relname IS NULL THEN 'empty'
        ELSE c.relname
    END as relation,
    count(*) as buffers,
    pg_size_pretty(count(*) * 8192) as size
FROM pg_buffercache b
LEFT JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 10;

-- Check for buffer contention
SELECT 
    buffers_alloc,
    buffers_clean,
    maxwritten_clean,
    buffers_backend,
    buffers_backend_fsync
FROM pg_stat_bgwriter;
```

#### pg_prewarm Extension
Preload tables into buffer cache on startup or manually.

```sql
-- Enable extension
CREATE EXTENSION pg_prewarm;

-- Manually prewarm a table
SELECT pg_prewarm('frequently_accessed_table');

-- Prewarm with specific buffer type
SELECT pg_prewarm('critical_table', 'main');  -- main, fsm, or vm

-- Save buffer cache to disk (auto-loads on restart)
SELECT autoprewarm_dump_now();

-- Check autoprewarm status
SELECT * FROM pg_prewarm_info;
```

### Query Execution Memory

#### Analyzing Memory Usage in Queries
```sql
-- Use EXPLAIN BUFFERS to see cache behavior
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders WHERE customer_id = 12345;

-- Output shows:
-- Buffers: shared hit=123 read=45
-- hit: pages found in cache
-- read: pages read from disk

-- Example of query with memory spills
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, COUNT(*), SUM(amount)
FROM large_orders
GROUP BY customer_id;

-- If you see "Sort Method: external merge Disk: XXXkB"
-- Increase work_mem for this query type
```

#### Monitoring Disk Spills
```sql
-- Queries causing disk spills (requires pg_stat_statements)
SELECT 
    query,
    calls,
    mean_time,
    temp_blks_written,
    temp_blks_read,
    pg_size_pretty(temp_blks_written * 8192) as temp_space
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 20;

-- Current temp file usage
SELECT 
    datname,
    temp_files,
    temp_bytes,
    pg_size_pretty(temp_bytes) as temp_size
FROM pg_stat_database
WHERE temp_files > 0;
```

### Operating System Cache

PostgreSQL relies heavily on the OS page cache in addition to shared_buffers.

#### Linux-Specific Tuning
```bash
# View OS cache statistics
free -h
# Or more detailed
cat /proc/meminfo | grep -E 'Cached|Buffers|Dirty'

# Drop OS caches (for testing only!)
sync; echo 3 > /proc/sys/vm/drop_caches

# Tune vm.swappiness (reduce swapping)
# Add to /etc/sysctl.conf:
vm.swappiness = 1
# Apply: sysctl -p

# Transparent Huge Pages (disable for PostgreSQL)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### Caching Strategies

#### Hot Data Caching
```sql
-- Identify hot tables/indexes
SELECT 
    schemaname,
    tablename,
    heap_blks_read,
    heap_blks_hit,
    idx_blks_read,
    idx_blks_hit,
    round((heap_blks_hit + idx_blks_hit)::numeric / 
          NULLIF(heap_blks_hit + idx_blks_hit + heap_blks_read + idx_blks_read, 0) * 100, 2) 
          as total_cache_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 1000
ORDER BY heap_blks_read + idx_blks_read DESC
LIMIT 20;

-- Prewarm frequently accessed data
SELECT pg_prewarm('hot_table');
SELECT pg_prewarm('hot_index');
```

#### Partitioning for Cache Efficiency
```sql
-- Partition large tables to improve cache locality
CREATE TABLE orders (
    id BIGSERIAL,
    customer_id INTEGER,
    order_date DATE,
    amount DECIMAL
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Queries accessing recent data benefit from caching fewer partitions
```

### Advanced Tuning Examples

#### High-Concurrency OLTP System
```sql
-- postgresql.conf
shared_buffers = 8GB                    # 25% of 32GB RAM
effective_cache_size = 24GB             # 75% of RAM
work_mem = 16MB                         # Conservative for many connections
maintenance_work_mem = 1GB
wal_buffers = 16MB
max_connections = 200

-- Monitor connection memory usage
SELECT 
    COUNT(*) as connections,
    pg_size_pretty(SUM(pg_backend_memory_contexts.total_bytes)) as total_memory
FROM pg_backend_memory_contexts
GROUP BY pid;
```

#### Data Warehouse / Analytics System
```sql
-- postgresql.conf
shared_buffers = 16GB                   # 40% of 40GB RAM  
effective_cache_size = 30GB             # 75% of RAM
work_mem = 256MB                        # Higher for complex queries
maintenance_work_mem = 2GB              # Higher for large indexes
wal_buffers = 16MB
max_connections = 50                    # Fewer connections

-- Increase for specific analytical queries
SET work_mem = '1GB';
SET temp_buffers = '256MB';
```

### Monitoring Dashboard Queries

```sql
-- Comprehensive cache monitoring view
CREATE OR REPLACE VIEW cache_stats_summary AS
SELECT 
    'Buffer Cache Hit Ratio' as metric,
    round((sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100)::numeric, 2)::text || '%' as value
FROM pg_statio_user_tables
UNION ALL
SELECT 
    'Shared Buffers Size',
    pg_size_pretty((SELECT setting::bigint * 8192 FROM pg_settings WHERE name = 'shared_buffers'))
UNION ALL
SELECT 
    'Effective Cache Size',
    pg_size_pretty((SELECT setting::bigint * 8192 FROM pg_settings WHERE name = 'effective_cache_size'))
UNION ALL
SELECT 
    'Work Mem',
    pg_size_pretty((SELECT setting::bigint * 1024 FROM pg_settings WHERE name = 'work_mem'))
UNION ALL
SELECT 
    'Maintenance Work Mem',
    pg_size_pretty((SELECT setting::bigint * 1024 FROM pg_settings WHERE name = 'maintenance_work_mem'));

-- Run it
SELECT * FROM cache_stats_summary;
```

### Best Practices

1. **Set shared_buffers appropriately**: 25% of RAM is a good starting point
2. **Don't over-allocate shared_buffers**: Beyond 40% shows diminishing returns
3. **Configure effective_cache_size accurately**: Helps query planner make better decisions
4. **Monitor cache hit ratios**: Aim for > 99% for steady-state workloads
5. **Adjust work_mem carefully**: Remember it's per-operation, not per-connection
6. **Use connection pooling**: Reduces per-connection memory overhead
7. **Prewarm critical data**: Use pg_prewarm for predictable hot data
8. **Monitor disk spills**: Tune work_mem if seeing excessive temp file usage

- **Further Reading**: [PostgreSQL Resource Consumption](https://www.postgresql.org/docs/18/runtime-config-resource.html) | [pg_buffercache](https://www.postgresql.org/docs/18/pgbuffercache.html) | [pg_prewarm](https://www.postgresql.org/docs/18/pgprewarm.html)

## 5. Transactions

### Overview
Transactions are the cornerstone of database reliability, ensuring that a sequence of operations either completes entirely or has no effect at all. PostgreSQL implements transactions using MVCC (Multi-Version Concurrency Control), providing high concurrency without locking readers.

### ACID Properties

PostgreSQL guarantees ACID properties for all transactions:

```
┌────────────────────────────────────────────────────────────┐
│                    ACID Properties                         │
├────────────────────────────────────────────────────────────┤
│ A - Atomicity:   All or nothing execution                 │
│ C - Consistency: Database rules are never violated        │
│ I - Isolation:   Concurrent transactions don't interfere  │
│ D - Durability:  Committed changes survive crashes        │
└────────────────────────────────────────────────────────────┘
```

### Basic Transaction Operations

```sql
-- Start a transaction
BEGIN;
-- Or: START TRANSACTION;

-- Perform operations
INSERT INTO accounts (user_id, balance) VALUES (1, 1000.00);
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

-- Commit the transaction (make changes permanent)
COMMIT;

-- Or rollback (undo all changes)
ROLLBACK;
```

#### Savepoints
```sql
BEGIN;

INSERT INTO orders (customer_id, total) VALUES (123, 500.00);

-- Create a savepoint
SAVEPOINT order_items;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 10, 2);
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 20, 1);

-- Oops, need to undo item inserts
ROLLBACK TO SAVEPOINT order_items;

-- Continue with different items
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 15, 3);

COMMIT;
```

### Transaction Isolation Levels

PostgreSQL supports four isolation levels defined by SQL standard (though READ UNCOMMITTED behaves like READ COMMITTED).

```sql
-- View current isolation level
SHOW transaction_isolation;

-- Set isolation level for current transaction
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Set default for session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

#### Isolation Level Comparison

```
┌──────────────────────┬─────────────┬─────────────┬──────────────┐
│ Isolation Level      │ Dirty Read  │ Non-Repeat  │ Phantom Read │
├──────────────────────┼─────────────┼─────────────┼──────────────┤
│ READ UNCOMMITTED*    │ No (PG)     │ Possible    │ Possible     │
│ READ COMMITTED       │ No          │ Possible    │ Possible     │
│ REPEATABLE READ      │ No          │ No          │ No (PG)**    │
│ SERIALIZABLE         │ No          │ No          │ No           │
└──────────────────────┴─────────────┴─────────────┴──────────────┘
* PostgreSQL treats READ UNCOMMITTED as READ COMMITTED
** PostgreSQL's REPEATABLE READ prevents phantoms (stronger than SQL standard)
```

#### READ COMMITTED (Default)
```sql
-- Session 1
BEGIN;
SELECT balance FROM accounts WHERE user_id = 1;  -- Returns 1000

-- Session 2
UPDATE accounts SET balance = 1500 WHERE user_id = 1;
COMMIT;

-- Session 1 (continued)
SELECT balance FROM accounts WHERE user_id = 1;  -- Returns 1500 (changed!)
COMMIT;
```

**Characteristics**:
- Sees committed changes from other transactions
- Each statement sees a fresh snapshot
- Most commonly used isolation level

#### REPEATABLE READ
```sql
-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE user_id = 1;  -- Returns 1000

-- Session 2
UPDATE accounts SET balance = 1500 WHERE user_id = 1;
COMMIT;

-- Session 1 (continued)
SELECT balance FROM accounts WHERE user_id = 1;  -- Still returns 1000
COMMIT;
```

**Characteristics**:
- Sees a consistent snapshot from transaction start
- Protects against non-repeatable reads
- Can fail with serialization errors on conflicts

#### SERIALIZABLE
```sql
-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(balance) FROM accounts WHERE account_type = 'checking';  -- Returns 10000

-- Session 2
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
INSERT INTO accounts (account_type, balance) VALUES ('checking', 500);
COMMIT;

-- Session 1 (continued)
INSERT INTO audit_log (total_balance) 
    VALUES ((SELECT SUM(balance) FROM accounts WHERE account_type = 'checking'));
COMMIT;  -- ERROR: could not serialize access due to read/write dependencies
```

**Characteristics**:
- Strongest isolation level
- Prevents all anomalies
- Uses predicate locking (Serializable Snapshot Isolation)
- May cause more serialization failures

### MVCC (Multi-Version Concurrency Control)

PostgreSQL's MVCC implementation allows high concurrency by keeping multiple versions of rows.

#### How MVCC Works

```
┌────────────────────────────────────────────────────────────┐
│                    MVCC Row Versioning                     │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Transaction 100: UPDATE users SET name = 'Alice'          │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ Old Version (visible to txn < 100)                  │  │
│  │ xmin: 50  xmax: 100  name: 'Bob'                    │  │
│  └─────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ New Version (visible to txn >= 100)                 │  │
│  │ xmin: 100  xmax: 0  name: 'Alice'                   │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

#### Hidden System Columns
```sql
-- View system columns (PostgreSQL adds these to every row)
SELECT 
    xmin,          -- Transaction ID that inserted this row
    xmax,          -- Transaction ID that deleted/updated this row (0 if current)
    cmin,          -- Command ID within inserting transaction
    cmax,          -- Command ID within deleting transaction
    ctid,          -- Physical location of row
    *
FROM accounts;

-- Example output:
-- xmin  | xmax | cmin | cmax | ctid  | user_id | balance
-- ------+------+------+------+-------+---------+---------
-- 1234  |    0 |    0 |    0 | (0,1) |       1 | 1000.00
```

#### Visibility Rules
```sql
-- A row version is visible to transaction T if:
-- 1. xmin is committed and xmin < T
-- 2. xmax is 0 OR (xmax is not committed OR xmax >= T)

-- Check transaction ID
SELECT txid_current();

-- Check if transaction is in progress
SELECT txid_current_if_assigned();

-- Transaction ID status
SELECT txid_status(12345);  -- committed, aborted, or in progress
```

### Transaction Management

#### Long-Running Transactions
```sql
-- Monitor long-running transactions
SELECT 
    pid,
    now() - xact_start AS duration,
    state,
    query,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE state != 'idle'
    AND xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 10;

-- Kill a long-running transaction
SELECT pg_cancel_backend(pid);    -- Gentle cancellation
SELECT pg_terminate_backend(pid); -- Forceful termination

-- Long transactions are problematic because:
-- 1. Block VACUUM from cleaning up dead tuples
-- 2. Increase table bloat
-- 3. Can cause transaction ID wraparound issues
```

#### Transaction ID Management
```sql
-- Current transaction ID
SELECT txid_current();

-- Transaction ID usage
SELECT 
    datname,
    age(datfrozenxid) as xid_age,
    pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Check for wraparound danger (age > 200 million is concerning)
SELECT 
    datname,
    age(datfrozenxid),
    CASE 
        WHEN age(datfrozenxid) > 1000000000 THEN 'CRITICAL'
        WHEN age(datfrozenxid) > 200000000 THEN 'WARNING'
        ELSE 'OK'
    END as status
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

#### Prepared Transactions (Two-Phase Commit)
```sql
-- Prepare a transaction for commit
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
PREPARE TRANSACTION 'payment_tx_001';

-- Later, commit or rollback the prepared transaction
COMMIT PREPARED 'payment_tx_001';
-- Or: ROLLBACK PREPARED 'payment_tx_001';

-- View prepared transactions
SELECT * FROM pg_prepared_xacts;

-- Configuration
max_prepared_transactions = 100  -- postgresql.conf
```

### Advanced Transaction Patterns

#### Optimistic Locking
```sql
-- Add version column
ALTER TABLE accounts ADD COLUMN version INTEGER DEFAULT 1;

-- Application-level optimistic locking
BEGIN;

SELECT balance, version FROM accounts WHERE user_id = 1;
-- balance: 1000, version: 5

-- Try to update with version check
UPDATE accounts 
SET balance = balance - 100, version = version + 1
WHERE user_id = 1 AND version = 5;

-- If UPDATE returns 0 rows, someone else modified it
-- Application should retry or abort

COMMIT;
```

#### Advisory Locks
```sql
-- Session-level advisory lock
SELECT pg_advisory_lock(12345);
-- Critical section
SELECT pg_advisory_unlock(12345);

-- Transaction-level advisory lock
BEGIN;
SELECT pg_advisory_xact_lock(12345);
-- Work
COMMIT;  -- Lock automatically released

-- Try lock (non-blocking)
SELECT pg_try_advisory_lock(12345);  -- Returns true/false

-- Check held advisory locks
SELECT 
    locktype,
    objid,
    mode,
    granted,
    pid
FROM pg_locks
WHERE locktype = 'advisory';
```

#### Deferred Constraints
```sql
-- Create deferrable constraint
CREATE TABLE parent (
    id INTEGER PRIMARY KEY
);

CREATE TABLE child (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER,
    FOREIGN KEY (parent_id) REFERENCES parent(id)
        DEFERRABLE INITIALLY DEFERRED
);

-- Within transaction, constraint checked at COMMIT
BEGIN;
INSERT INTO child (id, parent_id) VALUES (1, 10);  -- parent_id 10 doesn't exist yet
INSERT INTO parent (id) VALUES (10);                 -- Now it exists
COMMIT;  -- Succeeds because constraint checked at end

-- Set constraint checking mode
SET CONSTRAINTS ALL IMMEDIATE;
SET CONSTRAINTS ALL DEFERRED;
```

### Transaction Performance

#### Monitoring Transaction Activity
```sql
-- Transaction throughput
SELECT 
    datname,
    xact_commit,
    xact_rollback,
    xact_commit + xact_rollback as total_xacts,
    round(xact_rollback::numeric / NULLIF(xact_commit + xact_rollback, 0) * 100, 2) as rollback_ratio
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY total_xacts DESC;

-- Commits per second
SELECT 
    (xact_commit - lag(xact_commit) OVER (ORDER BY stats_reset))::float / 
    EXTRACT(EPOCH FROM (stats_reset - lag(stats_reset) OVER (ORDER BY stats_reset)))
    as commits_per_second
FROM pg_stat_database
WHERE datname = current_database();

-- Blocking transactions
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement,
    blocked_activity.application_name AS blocked_application
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

#### Transaction Best Practices

1. **Keep Transactions Short**: Long transactions block VACUUM and cause bloat
```sql
-- Bad: Long-running transaction
BEGIN;
SELECT pg_sleep(300);  -- 5 minutes!
UPDATE accounts SET balance = balance + 10 WHERE user_id = 1;
COMMIT;

-- Good: Short transaction
BEGIN;
UPDATE accounts SET balance = balance + 10 WHERE user_id = 1;
COMMIT;
```

2. **Avoid Mixing DDL and DML**: DDL takes strong locks
```sql
-- Better to separate these
BEGIN;
ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT false;
COMMIT;

BEGIN;
UPDATE users SET email_verified = true WHERE email IS NOT NULL;
COMMIT;
```

3. **Use Appropriate Isolation Levels**
```sql
-- Use READ COMMITTED for most OLTP workloads
-- Use REPEATABLE READ for reports requiring consistency
-- Use SERIALIZABLE only when necessary (higher conflict rate)
```

4. **Handle Serialization Failures**
```python
# Python example with retry logic
import psycopg2
from psycopg2 import extensions

def execute_with_retry(conn, max_retries=3):
    for attempt in range(max_retries):
        try:
            cursor = conn.cursor()
            cursor.execute("BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE")
            # Your operations here
            cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE user_id = 1")
            cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE user_id = 2")
            conn.commit()
            return True
        except psycopg2.extensions.TransactionRollbackError:
            conn.rollback()
            if attempt == max_retries - 1:
                raise
            # Retry with exponential backoff
            time.sleep(0.1 * (2 ** attempt))
    return False
```

### Practical Examples

#### Bank Transfer (Money Transfer Pattern)
```sql
-- Atomic money transfer between accounts
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Lock accounts in consistent order to prevent deadlocks
SELECT balance FROM accounts 
WHERE user_id IN (1, 2) 
ORDER BY user_id 
FOR UPDATE;

-- Check sufficient balance
SELECT balance FROM accounts WHERE user_id = 1;
-- Application checks: balance >= 100

-- Perform transfer
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

-- Log the transfer
INSERT INTO transfer_log (from_user, to_user, amount, timestamp)
VALUES (1, 2, 100, NOW());

COMMIT;
```

#### Inventory Management
```sql
-- Reserve inventory with optimistic locking
BEGIN;

-- Read current quantity with lock
SELECT quantity, version 
FROM inventory 
WHERE product_id = 123 
FOR UPDATE;

-- Check availability
-- quantity: 50, version: 10

-- Reserve items
UPDATE inventory 
SET quantity = quantity - 5,
    version = version + 1
WHERE product_id = 123 
    AND version = 10
    AND quantity >= 5;

GET DIAGNOSTICS rows_affected = ROW_COUNT;

IF rows_affected = 0 THEN
    ROLLBACK;
    -- Handle conflict (item sold out or version mismatch)
ELSE
    COMMIT;
END IF;
```

- **Further Reading**: [PostgreSQL Transactions](https://www.postgresql.org/docs/18/tutorial-transactions.html) | [Transaction Isolation](https://www.postgresql.org/docs/18/transaction-iso.html) | [MVCC](https://www.postgresql.org/docs/18/mvcc.html) | [Explicit Locking](https://www.postgresql.org/docs/18/explicit-locking.html)

## 6. Large Object Storage

### Overview
PostgreSQL provides two mechanisms for storing large binary data: Large Objects (LOB) and BYTEA columns. Each has distinct characteristics, use cases, and trade-offs.

### Large Objects vs BYTEA

```
┌─────────────────────┬──────────────────────┬─────────────────────┐
│ Feature             │ Large Objects (LOB)  │ BYTEA Column        │
├─────────────────────┼──────────────────────┼─────────────────────┤
│ Max Size            │ 4TB (2GB per chunk)  │ 1GB (TOAST limit)   │
│ Storage             │ Separate system      │ In table (TOASTed)  │
│ Streaming Access    │ Yes                  │ No (all or nothing) │
│ Random Access       │ Yes (seek support)   │ No                  │
│ Permissions         │ Row-level only       │ Full SQL control    │
│ VACUUM Impact       │ Separate cleanup     │ Normal VACUUM       │
│ Backup/Restore      │ Requires --blobs     │ Automatic           │
│ Transaction Safety  │ Yes                  │ Yes                 │
└─────────────────────┴──────────────────────┴─────────────────────┘
```

### Large Object API

#### Creating and Using Large Objects

```sql
-- Create a large object (returns OID)
SELECT lo_create(0);  -- 0 = auto-assign OID
-- Returns: 16385

-- Import file to large object
SELECT lo_import('/path/to/file.pdf');
-- Returns: 16386

-- Create table to reference large objects
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    filename VARCHAR(255),
    mime_type VARCHAR(100),
    content_oid OID,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert reference to large object
INSERT INTO documents (filename, mime_type, content_oid)
VALUES ('report.pdf', 'application/pdf', 16386);
```

#### Reading Large Objects

```sql
-- Open large object for reading (mode 262144 = INV_READ)
SELECT lo_open(16386, 262144);
-- Returns file descriptor: 0

-- Read from large object
SELECT lo_read(0, 1024);  -- Read 1024 bytes from fd 0
-- Returns bytea data

-- Seek to position
SELECT lo_lseek(0, 1000, 0);  -- Seek to byte 1000 (SEEK_SET = 0)

-- Get current position
SELECT lo_tell(0);

-- Close large object
SELECT lo_close(0);

-- Export large object to file
SELECT lo_export(16386, '/path/to/output.pdf');

-- Get large object size
SELECT lo_get(16386);  -- Returns NULL if doesn't exist
```

#### Writing to Large Objects

```sql
-- Create new large object
SELECT lo_create(0) AS new_oid \gset

-- Open for writing (mode 131072 = INV_WRITE)
SELECT lo_open(:new_oid, 131072) AS fd \gset

-- Write data
SELECT lo_write(:fd, '\xDEADBEEF'::bytea);

-- Close
SELECT lo_close(:fd);

-- Or use lowrite for positioned writes
SELECT lo_open(:new_oid, 393216) AS fd \gset  -- INV_READ | INV_WRITE
SELECT lo_lseek(:fd, 1000, 0);  -- Seek to position
SELECT lo_write(:fd, 'Some text'::bytea);
SELECT lo_close(:fd);
```

#### Deleting Large Objects

```sql
-- Delete a large object
SELECT lo_unlink(16386);

-- Clean up orphaned large objects (no table references)
-- First, create a function to find orphans
CREATE OR REPLACE FUNCTION find_orphaned_large_objects()
RETURNS TABLE(oid OID) AS $$
    SELECT lo.oid
    FROM pg_largeobject_metadata lo
    LEFT JOIN documents d ON lo.oid = d.content_oid
    WHERE d.content_oid IS NULL;
$$ LANGUAGE SQL;

-- View orphaned large objects
SELECT * FROM find_orphaned_large_objects();

-- Delete orphaned large objects
SELECT lo_unlink(oid) FROM find_orphaned_large_objects();

-- Use vacuumlo utility (command line)
-- vacuumlo -v -n database_name  -- Dry run
-- vacuumlo -v database_name     -- Actually delete
```

### Large Object Permissions

```sql
-- Grant access to specific large object
SELECT lo_open(16386, 262144);  -- Will fail if no permission

-- Create large object with specific permissions
-- Note: Permissions are limited - typically manage at row level

-- Better approach: Use table-level permissions
GRANT SELECT ON documents TO readonly_user;
GRANT INSERT, UPDATE, DELETE ON documents TO editor_user;
```

### BYTEA Columns

#### Creating and Using BYTEA

```sql
-- Create table with BYTEA column
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    filename VARCHAR(255),
    mime_type VARCHAR(100),
    content BYTEA,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert binary data
INSERT INTO files (filename, mime_type, content)
VALUES ('logo.png', 'image/png', pg_read_binary_file('/path/to/logo.png'));

-- Or insert with escape format
INSERT INTO files (filename, mime_type, content)
VALUES ('data.bin', 'application/octet-stream', '\xDEADBEEF'::bytea);

-- Select data (hex format by default)
SELECT encode(content, 'hex') FROM files WHERE id = 1;

-- Select as base64
SELECT encode(content, 'base64') FROM files WHERE id = 1;

-- Get size
SELECT filename, pg_size_pretty(length(content)) as size
FROM files;
```

#### TOAST (The Oversized-Attribute Storage Technique)

PostgreSQL automatically uses TOAST for large BYTEA values.

```sql
-- TOAST compresses and stores large values externally
-- Check TOAST statistics
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - 
                   pg_relation_size(schemaname||'.'||tablename)) as toast_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Configure TOAST behavior
ALTER TABLE files ALTER COLUMN content SET STORAGE EXTENDED;  -- Compress + external (default)
-- Options: PLAIN, EXTERNAL, EXTENDED, MAIN

-- EXTENDED: Compress, then move to TOAST if still large
-- MAIN: Compress, prefer inline
-- EXTERNAL: No compression, move to TOAST
-- PLAIN: No compression, always inline (not for BYTEA)
```

### Practical Examples

#### Document Management System with Large Objects

```sql
-- Schema for document management
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    mime_type VARCHAR(100),
    file_oid OID NOT NULL,
    file_size BIGINT,
    uploaded_by INTEGER REFERENCES users(id),
    uploaded_at TIMESTAMP DEFAULT NOW(),
    version INTEGER DEFAULT 1
);

-- Function to upload document
CREATE OR REPLACE FUNCTION upload_document(
    p_title VARCHAR,
    p_description TEXT,
    p_mime_type VARCHAR,
    p_data BYTEA,
    p_user_id INTEGER
) RETURNS INTEGER AS $$
DECLARE
    v_oid OID;
    v_fd INTEGER;
    v_doc_id INTEGER;
BEGIN
    -- Create large object
    SELECT lo_create(0) INTO v_oid;
    
    -- Open for writing
    SELECT lo_open(v_oid, 131072) INTO v_fd;
    
    -- Write data
    PERFORM lo_write(v_fd, p_data);
    
    -- Close
    PERFORM lo_close(v_fd);
    
    -- Insert document record
    INSERT INTO documents (title, description, mime_type, file_oid, file_size, uploaded_by)
    VALUES (p_title, p_description, p_mime_type, v_oid, length(p_data), p_user_id)
    RETURNING id INTO v_doc_id;
    
    RETURN v_doc_id;
END;
$$ LANGUAGE plpgsql;

-- Function to download document
CREATE OR REPLACE FUNCTION download_document(p_doc_id INTEGER)
RETURNS BYTEA AS $$
DECLARE
    v_oid OID;
    v_fd INTEGER;
    v_data BYTEA;
BEGIN
    -- Get large object OID
    SELECT file_oid INTO v_oid FROM documents WHERE id = p_doc_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Document not found';
    END IF;
    
    -- Open for reading
    SELECT lo_open(v_oid, 262144) INTO v_fd;
    
    -- Read all data (for large files, read in chunks)
    SELECT string_agg(chunk, ''::bytea)
    INTO v_data
    FROM (
        SELECT lo_read(v_fd, 8192) as chunk
        FROM generate_series(1, 1000)  -- Adjust for file size
        WHERE lo_read(v_fd, 0) IS NOT NULL
    ) chunks;
    
    -- Close
    PERFORM lo_close(v_fd);
    
    RETURN v_data;
END;
$$ LANGUAGE plpgsql;

-- Cleanup trigger for large objects
CREATE OR REPLACE FUNCTION cleanup_large_object()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM lo_unlink(OLD.file_oid);
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER document_cleanup
BEFORE DELETE ON documents
FOR EACH ROW
EXECUTE FUNCTION cleanup_large_object();
```

#### Image Storage with BYTEA

```sql
-- Schema for image storage
CREATE TABLE images (
    id SERIAL PRIMARY KEY,
    filename VARCHAR(255),
    thumbnail BYTEA,        -- Small, inline
    full_image BYTEA,       -- Large, TOASTed
    width INTEGER,
    height INTEGER,
    mime_type VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Set storage strategy
ALTER TABLE images ALTER COLUMN thumbnail SET STORAGE MAIN;      -- Keep inline
ALTER TABLE images ALTER COLUMN full_image SET STORAGE EXTENDED;  -- Compress + TOAST

-- Insert image
INSERT INTO images (filename, thumbnail, full_image, width, height, mime_type)
VALUES (
    'photo.jpg',
    decode('FFD8FFE0...', 'hex'),  -- Small thumbnail
    pg_read_binary_file('/path/to/photo.jpg'),
    1920,
    1080,
    'image/jpeg'
);

-- Efficient query (only fetch thumbnail)
SELECT id, filename, thumbnail FROM images WHERE id = 1;

-- Query with image metadata only (no binary data)
SELECT id, filename, width, height, 
       pg_size_pretty(length(full_image)) as size
FROM images;
```

### Performance Considerations

#### Comparing Performance

```sql
-- Benchmark BYTEA vs Large Objects
-- BYTEA: Simpler, better for small-medium files (< 100MB)
-- Large Objects: Better for very large files, streaming access

-- Monitor table bloat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - 
                   pg_relation_size(schemaname||'.'||tablename)) as toast_and_indexes,
    (SELECT count(*) FROM pg_largeobject_metadata) as large_object_count
FROM pg_tables
WHERE tablename IN ('documents', 'images')
GROUP BY schemaname, tablename;
```

### Backup and Restore

```bash
# Backup database with large objects
pg_dump -Fc --blobs mydb > mydb.dump

# Backup without large objects
pg_dump -Fc --no-blobs mydb > mydb_no_blobs.dump

# Restore
pg_restore -d mydb mydb.dump

# For BYTEA columns, standard backup includes them automatically
pg_dump mydb > mydb.sql
```

### Best Practices

1. **Choose the Right Storage Method**:
   - Use BYTEA for files < 100MB
   - Use Large Objects for files > 100MB or when streaming is needed
   - Consider external storage (S3, filesystem) for very large files

2. **Cleanup Orphaned Large Objects**:
```sql
-- Schedule regular cleanup
SELECT lo_unlink(oid) 
FROM pg_largeobject_metadata lo
WHERE NOT EXISTS (
    SELECT 1 FROM documents WHERE file_oid = lo.oid
);
```

3. **Monitor Storage Usage**:
```sql
-- Check large object storage
SELECT 
    count(*) as lo_count,
    pg_size_pretty(sum(length(data))) as total_size
FROM pg_largeobject;

-- Check BYTEA storage
SELECT 
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass))
FROM pg_tables
WHERE schemaname = 'public';
```

4. **Use Appropriate Access Patterns**:
```sql
-- For large objects: streaming access
-- For BYTEA: batch access, cache results
```

5. **Consider Compression**:
```sql
-- BYTEA with explicit compression
CREATE EXTENSION IF NOT EXISTS pg_compression;

INSERT INTO files (filename, content)
VALUES ('data.txt', compress(pg_read_binary_file('/path/to/data.txt')));

SELECT uncompress(content) FROM files WHERE filename = 'data.txt';
```

- **Further Reading**: [PostgreSQL Large Objects](https://www.postgresql.org/docs/18/largeobjects.html) | [Binary Data Types](https://www.postgresql.org/docs/18/datatype-binary.html) | [TOAST](https://www.postgresql.org/docs/18/storage-toast.html)

## 7. Postmaster Process

### Overview
The Postmaster is PostgreSQL's main supervisory process. It's the first process started when PostgreSQL starts and is responsible for managing all other PostgreSQL processes, handling connections, and maintaining database cluster integrity.

### PostgreSQL Process Architecture

```
┌────────────────────────────────────────────────────────────┐
│                PostgreSQL Process Architecture             │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Postmaster (Main Process)                │ │
│  │  • Listens for connections                           │ │
│  │  • Spawns backend processes                          │ │
│  │  • Manages background workers                        │ │
│  └────────────┬─────────────────────────────────────────┘ │
│               │                                             │
│      ┌────────┴────────┬──────────────┬──────────────┐    │
│      ▼                 ▼              ▼              ▼    │
│  ┌──────────┐    ┌──────────┐   ┌──────────┐  ┌────────┐│
│  │ Backend  │    │ Backend  │   │ Backend  │  │Backend ││
│  │ Process  │    │ Process  │   │ Process  │  │Process ││
│  │ (Client1)│    │ (Client2)│   │ (Client3)│  │...     ││
│  └──────────┘    └──────────┘   └──────────┘  └────────┘│
│                                                             │
│  Background Processes:                                     │
│  ┌──────────────┬──────────────┬──────────────────────┐  │
│  │ Background   │ WAL Writer   │ Checkpointer         │  │
│  │ Writer       │              │                       │  │
│  ├──────────────┼──────────────┼──────────────────────┤  │
│  │ Autovacuum   │ Stats        │ Logical Replication  │  │
│  │ Launcher     │ Collector    │ Workers              │  │
│  └──────────────┴──────────────┴──────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Postmaster Responsibilities

#### 1. Connection Management
```sql
-- View connection information
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    backend_start,
    state,
    query
FROM pg_stat_activity
ORDER BY backend_start;

-- Connection limits
SHOW max_connections;  -- Default: 100

-- Reserved connections for superusers
SHOW superuser_reserved_connections;  -- Default: 3

-- Check current connection count
SELECT 
    count(*) as current_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') as max_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') - count(*) as available_connections
FROM pg_stat_activity;
```

#### 2. Backend Process Spawning
```bash
# Each client connection gets its own backend process
# View processes
ps aux | grep postgres

# Example output:
# postgres 1234 ... postgres: postmaster
# postgres 1235 ... postgres: checkpointer
# postgres 1236 ... postgres: background writer
# postgres 1237 ... postgres: walwriter
# postgres 1238 ... postgres: autovacuum launcher
# postgres 1239 ... postgres: stats collector
# postgres 1240 ... postgres: user mydb 192.168.1.100(5432) idle
```

### Background Worker Processes

#### Background Writer
Writes dirty buffers to disk to reduce checkpoint load.

```sql
-- Monitor background writer
SELECT 
    checkpoints_timed,
    checkpoints_req,
    buffers_checkpoint,
    buffers_clean,        -- Written by bgwriter
    maxwritten_clean,     -- Bgwriter stopped due to max writes
    buffers_backend,      -- Backends wrote directly
    buffers_backend_fsync,
    buffers_alloc
FROM pg_stat_bgwriter;

-- Configuration
SHOW bgwriter_delay;              -- 200ms default
SHOW bgwriter_lru_maxpages;       -- 100 default
SHOW bgwriter_lru_multiplier;     -- 2.0 default

-- Tune bgwriter
ALTER SYSTEM SET bgwriter_delay = '100ms';
ALTER SYSTEM SET bgwriter_lru_maxpages = '200';
SELECT pg_reload_conf();
```

#### WAL Writer
Flushes WAL buffers to disk periodically.

```sql
-- Configuration
SHOW wal_writer_delay;  -- 200ms default
SHOW wal_writer_flush_after;  -- 1MB default

-- Monitor WAL activity
SELECT 
    pg_current_wal_lsn(),
    pg_walfile_name(pg_current_wal_lsn())
FROM pg_stat_activity
LIMIT 1;
```

#### Checkpointer
Performs checkpoints to ensure data durability.

```sql
-- Checkpoint configuration
SHOW checkpoint_timeout;              -- 5min default
SHOW checkpoint_completion_target;    -- 0.9 default
SHOW max_wal_size;                   -- 1GB default

-- Monitor checkpoints
SELECT 
    checkpoints_timed,    -- Scheduled
    checkpoints_req,      -- Requested (tune if high)
    checkpoint_write_time,
    checkpoint_sync_time
FROM pg_stat_bgwriter;

-- Manual checkpoint
CHECKPOINT;
```

#### Autovacuum Launcher
Manages autovacuum worker processes.

```sql
-- Configuration
SHOW autovacuum;                  -- on/off
SHOW autovacuum_max_workers;     -- 3 default
SHOW autovacuum_naptime;         -- 1min default

-- Monitor autovacuum workers
SELECT 
    pid,
    now() - xact_start AS duration,
    query
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%'
ORDER BY xact_start;

-- Check autovacuum launcher
SELECT pid, backend_start 
FROM pg_stat_activity 
WHERE backend_type = 'autovacuum launcher';
```

#### Stats Collector
Collects statistics about database activity.

```sql
-- Stats collector configuration
SHOW track_activities;           -- on
SHOW track_counts;              -- on
SHOW track_io_timing;           -- off (enable for detailed I/O stats)
SHOW track_functions;           -- none/pl/all

-- Enable detailed statistics
ALTER SYSTEM SET track_io_timing = on;
ALTER SYSTEM SET track_functions = 'all';
SELECT pg_reload_conf();

-- View collected statistics
SELECT * FROM pg_stat_database;
SELECT * FROM pg_stat_user_tables;
SELECT * FROM pg_stat_user_indexes;
```

#### Logical Replication Workers
Handle logical replication subscriptions.

```sql
-- Monitor logical replication workers
SELECT 
    pid,
    application_name,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;

-- View subscription workers
SELECT 
    subname,
    pid,
    leader_pid,
    relid,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time
FROM pg_stat_subscription;
```

### Postmaster Configuration

#### Connection Settings
```sql
-- postgresql.conf
listen_addresses = '*'                    -- Which interfaces to listen on
port = 5432                              -- Port number
max_connections = 200                    -- Maximum concurrent connections
superuser_reserved_connections = 3       -- Reserved for superusers
unix_socket_directories = '/var/run/postgresql'  -- Unix socket location

-- Connection pooling (pg_bouncer recommended for high connection count)
-- Instead of many direct connections, use connection pooler
```

#### Process Management
```sql
-- postgresql.conf
max_worker_processes = 8                 -- Max background workers
max_parallel_workers_per_gather = 2      -- Parallel query workers
max_parallel_workers = 8                 -- Max parallel workers total
max_parallel_maintenance_workers = 2     -- For CREATE INDEX, VACUUM
```

### Monitoring Postmaster and Processes

#### Active Processes
```sql
-- View all backend processes
SELECT 
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    backend_start,
    state_change,
    state,
    backend_type,
    wait_event_type,
    wait_event
FROM pg_stat_activity
ORDER BY backend_start;

-- Background processes
SELECT 
    pid,
    backend_type,
    backend_start
FROM pg_stat_activity
WHERE backend_type != 'client backend'
ORDER BY backend_type;

-- Count by backend type
SELECT 
    backend_type,
    count(*) as count
FROM pg_stat_activity
GROUP BY backend_type
ORDER BY count DESC;
```

#### Process Resource Usage
```sql
-- Requires pg_stat_statements extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top queries by CPU time
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Blocked queries
SELECT 
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE state != 'idle'
    AND wait_event IS NOT NULL
ORDER BY duration DESC;
```

#### System Catalog Queries
```sql
-- Database cluster information
SELECT * FROM pg_control_system();

-- Checkpoint information
SELECT * FROM pg_control_checkpoint();

-- Recovery status
SELECT 
    pg_is_in_recovery(),
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

### Handling Process Issues

#### Terminating Processes
```sql
-- Cancel a query (gentle)
SELECT pg_cancel_backend(12345);  -- PID

-- Terminate a backend (forceful)
SELECT pg_terminate_backend(12345);

-- Terminate all connections to a database
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'mydb'
    AND pid != pg_backend_pid();

-- Terminate idle connections
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
    AND state_change < now() - interval '1 hour'
    AND pid != pg_backend_pid();
```

#### Preventing Connection Exhaustion
```sql
-- Set connection limits per database
ALTER DATABASE mydb CONNECTION LIMIT 50;

-- Set connection limits per role
ALTER ROLE app_user CONNECTION LIMIT 10;

-- Monitor connection usage by database
SELECT 
    datname,
    count(*) as connections,
    max(backend_start) as newest_connection
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;

-- Use connection pooler (pg_bouncer)
-- External tool, not part of PostgreSQL
```

### Postmaster Crash Recovery

When the postmaster crashes or is killed:

```bash
# PostgreSQL performs automatic crash recovery on restart
# 1. Postmaster reads last checkpoint from control file
# 2. Replays WAL from checkpoint to end
# 3. Ensures database consistency

# Check PostgreSQL logs
tail -f /var/log/postgresql/postgresql-18-main.log

# Safe shutdown
pg_ctl stop -D /var/lib/postgresql/data -m smart   # Wait for clients
pg_ctl stop -D /var/lib/postgresql/data -m fast    # Terminate clients
pg_ctl stop -D /var/lib/postgresql/data -m immediate  # Abort (requires recovery)

# Start PostgreSQL
pg_ctl start -D /var/lib/postgresql/data
```

### Best Practices

1. **Monitor Connection Count**: Prevent exhaustion
```sql
SELECT count(*) * 100.0 / 
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections')
    as connection_usage_percent
FROM pg_stat_activity;
```

2. **Use Connection Pooling**: For high-concurrency applications
   - Use pg_bouncer or pgpool-II
   - Reduces overhead of connection creation

3. **Monitor Background Workers**: Ensure they're running
```sql
SELECT backend_type, count(*) 
FROM pg_stat_activity 
GROUP BY backend_type;
```

4. **Configure Appropriate Limits**: Based on workload
```sql
-- High-concurrency OLTP
max_connections = 500
max_worker_processes = 8

-- Low-concurrency analytics
max_connections = 50
max_parallel_workers = 16
```

5. **Regular Maintenance**: Keep system healthy
```bash
# Monitor logs
tail -f /var/log/postgresql/*.log

# Check system resources
top -p $(pgrep -d',' -f postgres)
```

- **Further Reading**: [PostgreSQL Server Process Architecture](https://www.postgresql.org/docs/18/tutorial-arch.html) | [Background Worker Processes](https://www.postgresql.org/docs/18/bgworker.html) | [Managing Connections](https://www.postgresql.org/docs/18/runtime-config-connection.html)

## 8. Transaction ID Wraparound

### Overview
Transaction ID (XID) wraparound is a critical PostgreSQL maintenance concern. PostgreSQL uses 32-bit transaction IDs, which means after approximately **4 billion transactions**, the IDs wrap around. Without proper maintenance, this can lead to data loss as old data becomes "invisible" to new transactions.

### Understanding Transaction IDs

```
┌────────────────────────────────────────────────────────────┐
│             PostgreSQL Transaction ID Space               │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  32-bit Transaction IDs: 0 to 4,294,967,295 (4 billion)   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  0-2: Reserved (Bootstrap, Frozen, Invalid)          │ │
│  │  3+: Regular transaction IDs                         │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  Special XIDs:                                             │
│  • 0: InvalidTransactionId                                 │
│  • 1: BootstrapTransactionId                               │
│  • 2: FrozenTransactionId (older than all XIDs)           │
│  • 3+: Normal transaction IDs                              │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### The Wraparound Problem

```
┌────────────────────────────────────────────────────────────┐
│               Circular Transaction ID Space                │
├────────────────────────────────────────────────────────────┤
│                                                             │
│     Past ← [Current XID] → Future                          │
│     2 billion XIDs    2 billion XIDs                       │
│                                                             │
│  Without freezing, after 2 billion transactions:           │
│  • Old transactions appear to be "in the future"           │
│  • Rows become invisible to new transactions              │
│  • Data appears lost (but still on disk)                  │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Monitoring Transaction Age

#### Check Database Age
```sql
-- Check age of all databases
SELECT 
    datname,
    age(datfrozenxid) as xid_age,
    2147483647 - age(datfrozenxid) as xids_until_wraparound,
    CASE 
        WHEN age(datfrozenxid) > 1500000000 THEN 'CRITICAL - Immediate action required!'
        WHEN age(datfrozenxid) > 1000000000 THEN 'WARNING - Schedule maintenance soon'
        WHEN age(datfrozenxid) > 200000000 THEN 'CAUTION - Monitor closely'
        ELSE 'OK'
    END as status
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- More detailed view
SELECT 
    datname,
    datfrozenxid,
    age(datfrozenxid) as age,
    pg_size_pretty(pg_database_size(datname)) as size,
    (SELECT setting FROM pg_settings WHERE name = 'autovacuum_freeze_max_age')::int as freeze_max_age
FROM pg_database
WHERE datallowconn
ORDER BY age(datfrozenxid) DESC;
```

#### Check Table Age
```sql
-- Check age of all tables
SELECT 
    schemaname,
    relname,
    age(relfrozenxid) as xid_age,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) as size,
    last_vacuum,
    last_autovacuum,
    n_dead_tup,
    CASE 
        WHEN age(relfrozenxid) > 1500000000 THEN 'CRITICAL'
        WHEN age(relfrozenxid) > 1000000000 THEN 'WARNING'
        WHEN age(relfrozenxid) > 200000000 THEN 'CAUTION'
        ELSE 'OK'
    END as status
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
LEFT JOIN pg_stat_user_tables s ON c.oid = s.relid
WHERE c.relkind = 'r'  -- Regular tables only
    AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(relfrozenxid) DESC
LIMIT 20;

-- Check for tables at risk
SELECT 
    n.nspname as schema,
    c.relname as table,
    age(c.relfrozenxid) as xid_age,
    pg_size_pretty(pg_total_relation_size(c.oid)) as size
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
    AND age(c.relfrozenxid) > 200000000  -- Customize threshold
ORDER BY age(c.relfrozenxid) DESC;
```

### VACUUM FREEZE

#### How Freezing Works
```sql
-- VACUUM FREEZE marks old tuples as "frozen"
-- Frozen tuples have xmin = FrozenTransactionId (2)
-- Frozen tuples are visible to all transactions

-- Perform VACUUM FREEZE
VACUUM FREEZE my_table;

-- View frozen tuples
SELECT 
    relname,
    age(relfrozenxid) as age,
    relfrozenxid,
    relminmxid
FROM pg_class
WHERE relkind = 'r'
    AND relname = 'my_table';
```

#### Configuration Parameters
```sql
-- View freeze-related settings
SELECT name, setting, unit, short_desc
FROM pg_settings
WHERE name LIKE '%freeze%' OR name LIKE '%vacuum%'
ORDER BY name;

-- Key parameters (postgresql.conf)
vacuum_freeze_min_age = 50000000           -- Min age before freezing
vacuum_freeze_table_age = 150000000        -- Age for aggressive freeze
autovacuum_freeze_max_age = 200000000      -- Max age before forced autovacuum
vacuum_multixact_freeze_min_age = 5000000
vacuum_multixact_freeze_table_age = 150000000
autovacuum_multixact_freeze_max_age = 400000000

-- Tune for high-transaction environments
ALTER SYSTEM SET autovacuum_freeze_max_age = 500000000;
ALTER SYSTEM SET vacuum_freeze_min_age = 50000000;
SELECT pg_reload_conf();
```

### Preventing Wraparound

#### Strategy 1: Ensure Autovacuum is Working
```sql
-- Check autovacuum is enabled
SHOW autovacuum;  -- Must be 'on'

-- Monitor autovacuum activity
SELECT 
    schemaname,
    relname,
    last_autovacuum,
    autovacuum_count,
    n_dead_tup,
    n_live_tup
FROM pg_stat_user_tables
WHERE last_autovacuum IS NOT NULL
ORDER BY last_autovacuum DESC;

-- Check for tables that haven't been autovacuumed
SELECT 
    schemaname,
    relname,
    now() - last_autovacuum as time_since_autovacuum,
    n_dead_tup
FROM pg_stat_user_tables
WHERE last_autovacuum IS NULL
    OR last_autovacuum < now() - interval '7 days'
ORDER BY n_dead_tup DESC;
```

#### Strategy 2: Manual VACUUM for Problem Tables
```sql
-- Identify tables needing vacuum
WITH table_ages AS (
    SELECT 
        n.nspname as schema,
        c.relname as table,
        age(c.relfrozenxid) as xid_age,
        pg_total_relation_size(c.oid) as size_bytes
    FROM pg_class c
    JOIN pg_namespace n ON c.relnamespace = n.oid
    WHERE c.relkind = 'r'
        AND n.nspname NOT IN ('pg_catalog', 'information_schema')
)
SELECT 
    schema,
    table,
    xid_age,
    pg_size_pretty(size_bytes) as size,
    'VACUUM FREEZE ' || schema || '.' || table || ';' as vacuum_command
FROM table_ages
WHERE xid_age > 200000000
ORDER BY xid_age DESC;

-- Execute manual vacuum
VACUUM FREEZE VERBOSE my_large_table;
```

#### Strategy 3: Scheduled Maintenance
```bash
# Cron job for proactive freezing
# /etc/cron.daily/postgres-freeze-maintenance.sh

#!/bin/bash
psql -U postgres -d mydb << EOF
-- Freeze old tables
SELECT 'VACUUM FREEZE ' || schemaname || '.' || tablename || ';'
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
\gexec

-- Verify ages
SELECT datname, age(datfrozenxid) FROM pg_database;
EOF
```

### Wraparound Emergency Response

#### Emergency Procedure
```sql
-- 1. Check severity
SELECT 
    datname,
    age(datfrozenxid) as age,
    2147483647 - age(datfrozenxid) as xids_remaining
FROM pg_database
ORDER BY age DESC;

-- If age > 1.5 billion, immediate action required!
-- Note: 1.5 billion (1,500,000,000) is the critical threshold
-- PostgreSQL will enter emergency mode at ~2 billion to prevent wraparound

-- 2. Identify problematic tables
SELECT 
    schemaname,
    relname,
    age(relfrozenxid) as xid_age,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) as size
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
LEFT JOIN pg_stat_user_tables s ON c.oid = s.relid
WHERE c.relkind = 'r'
    AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(relfrozenxid) DESC
LIMIT 10;

-- 3. Vacuum oldest tables first
VACUUM FREEZE VERBOSE oldest_table;

-- 4. Monitor progress
SELECT 
    schemaname,
    relname,
    age(relfrozenxid),
    now() - query_start as vacuum_duration,
    query
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
JOIN pg_stat_activity a ON a.query LIKE '%' || c.relname || '%'
WHERE a.query LIKE 'VACUUM%'
    AND c.relkind = 'r';

-- 5. If database won't start due to wraparound:
-- PostgreSQL will enter single-user mode
-- Run from command line:
postgres --single -D /var/lib/postgresql/data mydb
-- Then: VACUUM FREEZE;
```

#### Preventing Shutdown
```sql
-- PostgreSQL will refuse connections when:
-- age(datfrozenxid) > 2 billion - 3 million (safety margin)

-- Check how close you are
SELECT 
    datname,
    age(datfrozenxid),
    2000000000 - age(datfrozenxid) as xids_until_shutdown,
    CASE 
        WHEN age(datfrozenxid) > 2000000000 THEN 'DATABASE SHUTDOWN IMMINENT!'
        WHEN age(datfrozenxid) > 1800000000 THEN 'CRITICAL - Hours remaining'
        WHEN age(datfrozenxid) > 1500000000 THEN 'URGENT - Days remaining'
        ELSE 'Safe'
    END as urgency
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

### Monitoring Dashboard

```sql
-- Create comprehensive monitoring view
CREATE OR REPLACE VIEW xid_wraparound_monitor AS
SELECT 
    'Database' as level,
    datname as name,
    age(datfrozenxid) as xid_age,
    2147483647 - age(datfrozenxid) as xids_remaining,
    CASE 
        WHEN age(datfrozenxid) > 1500000000 THEN 'CRITICAL'
        WHEN age(datfrozenxid) > 1000000000 THEN 'WARNING'
        WHEN age(datfrozenxid) > 200000000 THEN 'CAUTION'
        ELSE 'OK'
    END as status,
    pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
WHERE datallowconn

UNION ALL

SELECT 
    'Table' as level,
    schemaname || '.' || relname as name,
    age(relfrozenxid) as xid_age,
    2147483647 - age(relfrozenxid) as xids_remaining,
    CASE 
        WHEN age(relfrozenxid) > 1500000000 THEN 'CRITICAL'
        WHEN age(relfrozenxid) > 1000000000 THEN 'WARNING'
        WHEN age(relfrozenxid) > 200000000 THEN 'CAUTION'
        ELSE 'OK'
    END as status,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) as size
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
LEFT JOIN pg_stat_user_tables s ON c.oid = s.relid
WHERE c.relkind = 'r'
    AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY xid_age DESC
LIMIT 50;

-- Query the view
SELECT * FROM xid_wraparound_monitor WHERE status != 'OK';
```

### Best Practices

1. **Monitor Regularly**: Check XID age weekly
```sql
-- Add to monitoring system
SELECT max(age(datfrozenxid)) FROM pg_database;
-- Alert if > 200 million
```

2. **Keep Autovacuum Enabled**: Never disable autovacuum
```sql
-- Verify it's running
SELECT count(*) FROM pg_stat_activity 
WHERE query LIKE 'autovacuum:%';
```

3. **Tune Autovacuum Aggressively**: For high-transaction systems
```sql
ALTER SYSTEM SET autovacuum_freeze_max_age = 500000000;
ALTER SYSTEM SET autovacuum_naptime = '30s';
ALTER SYSTEM SET autovacuum_max_workers = 5;
```

4. **Schedule Periodic VACUUM FREEZE**: During maintenance windows
```bash
# Weekly freeze of all tables
psql -c "VACUUM FREEZE VERBOSE;" mydb
```

5. **Plan for Maintenance**: Large tables take time to freeze
```sql
-- Estimate vacuum time
SELECT 
    relname,
    pg_size_pretty(pg_total_relation_size(relname::regclass)) as size,
    age(relfrozenxid),
    pg_total_relation_size(relname::regclass) / (1024*1024) / 100 as estimated_minutes
FROM pg_class
WHERE relkind = 'r'
    AND age(relfrozenxid) > 200000000
ORDER BY pg_total_relation_size(relname::regclass) DESC;
```

6. **Use Monitoring Tools**: Set up alerts
   - Prometheus + postgres_exporter
   - Nagios/Icinga with XID age checks
   - Custom scripts with pg_monitor role

- **Further Reading**: [PostgreSQL Routine Vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND) | [Transaction ID Wraparound](https://www.postgresql.org/docs/18/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND)

## 9. Explain Plans

### Overview
EXPLAIN shows the execution plan PostgreSQL's query planner generates for a query. Understanding EXPLAIN output is crucial for query optimization and performance tuning.

### EXPLAIN Basics

```sql
-- Basic EXPLAIN
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- EXPLAIN with execution (actually runs query)
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- Detailed output with all options
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS, TIMING, SUMMARY)
SELECT * FROM users WHERE email = 'user@example.com';
```

### EXPLAIN Options

```
┌─────────────────────┬──────────────────────────────────────┐
│ Option              │ Description                          │
├─────────────────────┼──────────────────────────────────────┤
│ ANALYZE             │ Execute query and show actual times  │
│ VERBOSE             │ Show additional details              │
│ COSTS               │ Show estimated costs (default: on)   │
│ BUFFERS             │ Show buffer usage statistics         │
│ TIMING              │ Show actual timing (default: on)     │
│ SUMMARY             │ Show summary statistics              │
│ FORMAT              │ Output format: TEXT, JSON, XML, YAML │
│ SETTINGS            │ Show modified configuration params   │
│ WAL                 │ Show WAL generation statistics       │
└─────────────────────┴──────────────────────────────────────┘
```

### Reading EXPLAIN Output

#### Basic Plan Structure
```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- Output:
-- Seq Scan on orders  (cost=0.00..1234.56 rows=100 width=50)
--   Filter: (customer_id = 123)

-- Components:
-- • Node Type: Seq Scan (sequential scan)
-- • Table: orders
-- • Startup Cost: 0.00 (cost to return first row)
-- • Total Cost: 1234.56 (cost to return all rows)
-- • Estimated Rows: 100
-- • Width: 50 bytes per row
```

### Common Scan Types

#### Sequential Scan
```sql
-- Reads entire table
EXPLAIN ANALYZE
SELECT * FROM large_table WHERE status = 'active';

-- Seq Scan on large_table  (cost=0.00..1829.00 rows=50000 width=100)
--   Filter: (status = 'active'::text)

-- Used when:
-- • No suitable index
-- • Table is small
-- • Query returns large percentage of rows
```

#### Index Scan
```sql
-- Uses index to find rows
CREATE INDEX idx_orders_customer ON orders(customer_id);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 123;

-- Index Scan using idx_orders_customer on orders
--   (cost=0.29..8.31 rows=1 width=100)
--   Index Cond: (customer_id = 123)

-- Characteristics:
-- • Random I/O to read index then table
-- • Efficient for selective queries
-- • Can be slow for many rows
```

#### Index Only Scan
```sql
-- Satisfies query entirely from index
CREATE INDEX idx_orders_customer_date 
ON orders(customer_id) INCLUDE (order_date);

EXPLAIN ANALYZE
SELECT customer_id, order_date 
FROM orders 
WHERE customer_id = 123;

-- Index Only Scan using idx_orders_customer_date
--   (cost=0.29..4.31 rows=1 width=12)
--   Index Cond: (customer_id = 123)
--   Heap Fetches: 0

-- Requirements:
-- • All columns in index
-- • Table has been vacuumed (visibility map updated)
```

#### Bitmap Scan
```sql
-- Two-phase scan: build bitmap, then scan table
EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE status = 'pending' AND priority = 'high';

-- Bitmap Heap Scan on orders  (cost=24.75..829.23 rows=500 width=100)
--   Recheck Cond: ((status = 'pending') AND (priority = 'high'))
--   ->  Bitmap Index Scan on idx_status  (cost=0.00..24.62 rows=500)
--         Index Cond: ((status = 'pending') AND (priority = 'high'))

-- Advantages:
-- • Combines multiple indexes efficiently
-- • Sequential I/O on table
-- • Good for moderate selectivity
```

### Join Types

#### Nested Loop Join
```sql
EXPLAIN ANALYZE
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending';

-- Nested Loop  (cost=0.58..1234.56 rows=100 width=150)
--   ->  Seq Scan on orders o  (cost=0.00..829.00 rows=100)
--         Filter: (status = 'pending')
--   ->  Index Scan using users_pkey on users u  (cost=0.29..4.05 rows=1)
--         Index Cond: (id = o.user_id)

-- Characteristics:
-- • Efficient for small outer relation
-- • Uses index on inner relation
-- • O(N * M) complexity
```

#### Hash Join
```sql
EXPLAIN ANALYZE
SELECT o.*, p.name
FROM orders o
JOIN products p ON o.product_id = p.id;

-- Hash Join  (cost=123.45..2345.67 rows=10000 width=200)
--   Hash Cond: (o.product_id = p.id)
--   ->  Seq Scan on orders o  (cost=0.00..1829.00 rows=100000)
--   ->  Hash  (cost=100.00..100.00 rows=1000 width=50)
--         ->  Seq Scan on products p  (cost=0.00..100.00 rows=1000)

-- Characteristics:
-- • Build hash table from smaller relation
-- • Probe with larger relation
-- • Efficient for large joins
-- • Requires work_mem
```

#### Merge Join
```sql
EXPLAIN ANALYZE
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
ORDER BY o.user_id;

-- Merge Join  (cost=234.56..3456.78 rows=100000 width=150)
--   Merge Cond: (o.user_id = u.id)
--   ->  Index Scan using idx_orders_user_id on orders o
--   ->  Index Scan using users_pkey on users u

-- Characteristics:
-- • Requires sorted input
-- • Very efficient when inputs are pre-sorted
-- • O(N + M) complexity
```

### Aggregation and Sorting

#### Aggregate Functions
```sql
EXPLAIN ANALYZE
SELECT customer_id, COUNT(*), SUM(total)
FROM orders
GROUP BY customer_id;

-- HashAggregate  (cost=2345.00..2545.00 rows=10000 width=20)
--   Group Key: customer_id
--   ->  Seq Scan on orders  (cost=0.00..1829.00 rows=100000)

-- Or with sorted input:
-- GroupAggregate  (cost=1234.56..2234.56 rows=10000 width=20)
--   Group Key: customer_id
--   ->  Index Scan using idx_orders_customer on orders
```

#### Sorting
```sql
EXPLAIN ANALYZE
SELECT * FROM orders ORDER BY created_at DESC LIMIT 100;

-- Limit  (cost=1234.56..1234.81 rows=100 width=100)
--   ->  Sort  (cost=1234.56..1484.56 rows=100000 width=100)
--         Sort Key: created_at DESC
--         Sort Method: top-N heapsort  Memory: 123kB
--         ->  Seq Scan on orders

-- Sort Methods:
-- • quicksort: In-memory sort
-- • top-N heapsort: For LIMIT queries
-- • external merge: Disk-based (increase work_mem if frequent)
```

### EXPLAIN ANALYZE with BUFFERS

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 123;

-- Index Scan using idx_orders_customer on orders
--   (cost=0.29..8.31 rows=1 width=100)
--   (actual time=0.025..0.027 rows=1 loops=1)
--   Index Cond: (customer_id = 123)
--   Buffers: shared hit=4
-- Planning:
--   Buffers: shared hit=8
-- Planning Time: 0.123 ms
-- Execution Time: 0.045 ms

-- Buffer statistics:
-- • shared hit: Pages found in cache
-- • shared read: Pages read from disk
-- • shared dirtied: Pages modified
-- • shared written: Pages written to disk
-- • temp read/written: Temporary file I/O
```

### Query Optimization Techniques

#### Technique 1: Add Missing Indexes
```sql
-- Before: Sequential scan
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending';

-- Seq Scan on orders  (cost=0.00..1829.00 rows=5000)
--   Filter: (status = 'pending')
--   Rows Removed by Filter: 95000
-- Execution Time: 125.234 ms

-- Add index
CREATE INDEX idx_orders_status ON orders(status);
ANALYZE orders;

-- After: Index scan
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending';

-- Index Scan using idx_orders_status on orders
--   (cost=0.29..234.56 rows=5000)
--   Index Cond: (status = 'pending')
-- Execution Time: 12.345 ms
```

#### Technique 2: Use Covering Indexes
```sql
-- Before: Index scan + heap fetches
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, order_date, total
FROM orders
WHERE customer_id = 123;

-- Index Scan using idx_orders_customer
--   Buffers: shared hit=45 read=10

-- Create covering index
CREATE INDEX idx_orders_customer_covering 
ON orders(customer_id) 
INCLUDE (order_date, total);
ANALYZE orders;

-- After: Index only scan
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, order_date, total
FROM orders
WHERE customer_id = 123;

-- Index Only Scan using idx_orders_customer_covering
--   Heap Fetches: 0
--   Buffers: shared hit=3
```

#### Technique 3: Increase work_mem for Sorts
```sql
-- Before: External merge (disk-based)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table ORDER BY created_at;

-- Sort Method: external merge  Disk: 12345kB

-- Increase work_mem for session
SET work_mem = '64MB';

-- After: In-memory sort
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table ORDER BY created_at;

-- Sort Method: quicksort  Memory: 45678kB
```

#### Technique 4: Partition Large Tables
```sql
-- Before: Scan entire large table
EXPLAIN ANALYZE
SELECT * FROM logs WHERE log_date = '2024-01-15';

-- Seq Scan on logs  (cost=0.00..500000.00 rows=1000)
--   Filter: (log_date = '2024-01-15')

-- Create partitioned table
CREATE TABLE logs_partitioned (
    id BIGSERIAL,
    log_date DATE,
    message TEXT
) PARTITION BY RANGE (log_date);

CREATE TABLE logs_2024_01 PARTITION OF logs_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- After: Scan only relevant partition
EXPLAIN ANALYZE
SELECT * FROM logs_partitioned WHERE log_date = '2024-01-15';

-- Seq Scan on logs_2024_01  (cost=0.00..15000.00 rows=1000)
--   Filter: (log_date = '2024-01-15')
```

#### Technique 5: Use Partial Indexes
```sql
-- Before: Large index on all rows
CREATE INDEX idx_orders_all ON orders(status);
-- Index size: 500 MB

-- After: Partial index on subset
DROP INDEX idx_orders_all;
CREATE INDEX idx_orders_pending ON orders(status) 
WHERE status IN ('pending', 'processing');
-- Index size: 50 MB

-- Query must match WHERE clause
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending';

-- Index Scan using idx_orders_pending
```

### Advanced Analysis

#### Comparing Plans
```sql
-- Compare different approaches
-- Approach 1: Subquery
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE customer_id IN (
    SELECT id FROM customers WHERE active = true
);

-- Approach 2: JOIN
EXPLAIN ANALYZE
SELECT o.*
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.active = true;

-- Approach 3: EXISTS
EXPLAIN ANALYZE
SELECT *
FROM orders o
WHERE EXISTS (
    SELECT 1 FROM customers c 
    WHERE c.id = o.customer_id AND c.active = true
);

-- Compare execution times and choose best plan
```

#### Analyzing Parallel Queries
```sql
-- Enable parallel query
SET max_parallel_workers_per_gather = 4;

EXPLAIN ANALYZE
SELECT COUNT(*) FROM large_table;

-- Finalize Aggregate  (cost=123456.78..123456.79 rows=1)
--   ->  Gather  (cost=123456.12..123456.78 rows=4)
--         Workers Planned: 4
--         Workers Launched: 4
--         ->  Partial Aggregate  (cost=122456.12..122456.13 rows=1)
--               ->  Parallel Seq Scan on large_table
--                     (cost=0.00..112345.67 rows=4044444)

-- Shows parallel execution with 4 workers
```

#### Using JSON Format for Programmatic Analysis
```sql
EXPLAIN (ANALYZE, FORMAT JSON)
SELECT * FROM orders WHERE customer_id = 123;

-- Returns JSON structure:
-- {
--   "Plan": {
--     "Node Type": "Index Scan",
--     "Relation Name": "orders",
--     "Index Name": "idx_orders_customer",
--     "Startup Cost": 0.29,
--     "Total Cost": 8.31,
--     "Plan Rows": 1,
--     "Actual Rows": 1,
--     "Actual Total Time": 0.045
--   }
-- }

-- Useful for monitoring tools and automation
```

### Practical Examples

#### Optimizing a Complex Query
```sql
-- Initial slow query
EXPLAIN ANALYZE
SELECT 
    u.name,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent,
    AVG(o.total) as avg_order
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.active = true
    AND u.created_at >= '2024-01-01'
    AND (o.status = 'completed' OR o.status IS NULL)
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 5
ORDER BY total_spent DESC
LIMIT 100;

-- Analyze the plan and optimize:
-- 1. Add index on users(active, created_at)
CREATE INDEX idx_users_active_created ON users(active, created_at);

-- 2. Add index on orders(user_id, status) INCLUDE (total)
CREATE INDEX idx_orders_user_status 
ON orders(user_id, status) INCLUDE (total);

-- 3. Update statistics
ANALYZE users;
ANALYZE orders;

-- 4. Re-run EXPLAIN ANALYZE and compare
```

#### Debugging Slow Aggregation
```sql
-- Problem: Slow GROUP BY
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    date_trunc('day', created_at) as day,
    COUNT(*),
    SUM(amount)
FROM transactions
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY day
ORDER BY day;

-- Solution: Index on date expression
CREATE INDEX idx_transactions_day 
ON transactions(date_trunc('day', created_at))
WHERE created_at >= NOW() - INTERVAL '90 days';  -- Partial index

-- Or: Materialized view for aggregates
CREATE MATERIALIZED VIEW daily_transactions AS
SELECT 
    date_trunc('day', created_at) as day,
    COUNT(*) as count,
    SUM(amount) as total
FROM transactions
GROUP BY day;

CREATE UNIQUE INDEX ON daily_transactions(day);
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_transactions;
```

### Best Practices

1. **Always use EXPLAIN ANALYZE for optimization**
   - Shows actual vs estimated rows
   - Reveals performance bottlenecks

2. **Monitor buffer usage**
   - High "read" values indicate disk I/O
   - Tune shared_buffers or add indexes

3. **Watch for row estimate mismatches**
   - Large differences indicate stale statistics
   - Run ANALYZE regularly

4. **Look for sequential scans on large tables**
   - Usually indicates missing indexes
   - Acceptable for small tables or batch operations

5. **Check for expensive operations**
   - Sort operations with disk spills
   - Nested loops with many iterations
   - Full table scans

6. **Use pg_stat_statements for query patterns**
```sql
CREATE EXTENSION pg_stat_statements;

-- Find slowest queries
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```

- **Further Reading**: [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/18/sql-explain.html) | [Using EXPLAIN](https://www.postgresql.org/docs/18/using-explain.html) | [Query Planning](https://www.postgresql.org/docs/18/planner-optimizer.html)

## 10. Deadlocks

### Overview
A deadlock occurs when two or more transactions are waiting for each other to release locks, creating a circular dependency. PostgreSQL automatically detects and resolves deadlocks by aborting one of the transactions.

### Understanding Deadlocks

```
┌────────────────────────────────────────────────────────────┐
│                    Deadlock Scenario                       │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Transaction 1:                Transaction 2:              │
│  BEGIN;                        BEGIN;                      │
│  UPDATE accounts               UPDATE accounts             │
│    SET balance = balance - 100   SET balance = balance + 50│
│    WHERE id = 1;                 WHERE id = 2;             │
│  -- Locks row 1               -- Locks row 2               │
│                                                             │
│  UPDATE accounts               UPDATE accounts             │
│    SET balance = balance + 100   SET balance = balance - 50│
│    WHERE id = 2;                 WHERE id = 1;             │
│  -- Waits for row 2           -- Waits for row 1           │
│  ⚠️ DEADLOCK DETECTED!                                     │
│                                                             │
│  PostgreSQL aborts one transaction (the "victim")          │
└────────────────────────────────────────────────────────────┘
```

### Deadlock Detection

PostgreSQL's deadlock detector runs periodically (default: every 1 second).

```sql
-- View deadlock detection timeout
SHOW deadlock_timeout;  -- Default: 1s

-- Configure (postgresql.conf)
deadlock_timeout = 1s

-- When deadlock detected, one transaction receives error:
-- ERROR: deadlock detected
-- DETAIL: Process 12345 waits for ShareLock on transaction 67890;
--         blocked by process 12346.
--         Process 12346 waits for ShareLock on transaction 67889;
--         blocked by process 12345.
```

### Monitoring Deadlocks

#### Check Deadlock Statistics
```sql
-- View deadlock count per database
SELECT 
    datname,
    deadlocks,
    conflicts,
    temp_files,
    temp_bytes
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY deadlocks DESC;

-- Reset statistics to track new deadlocks
SELECT pg_stat_reset();

-- Monitor specific database
SELECT 
    deadlocks,
    blks_read,
    blks_hit,
    tup_returned,
    tup_fetched
FROM pg_stat_database
WHERE datname = current_database();
```

#### Identify Blocking Queries
```sql
-- View currently blocked queries
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS current_statement_in_blocking_process,
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

-- Simplified blocking query view
SELECT 
    waiting.pid AS waiting_pid,
    waiting.query AS waiting_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity AS waiting
JOIN pg_stat_activity AS blocking 
    ON blocking.pid = ANY(pg_blocking_pids(waiting.pid))
WHERE waiting.state = 'active';
```

#### View Lock Information
```sql
-- Current locks in the database
SELECT 
    locktype,
    database,
    relation::regclass,
    page,
    tuple,
    transactionid,
    mode,
    granted,
    pid
FROM pg_locks
WHERE NOT granted
ORDER BY pid;

-- Locks by relation
SELECT 
    c.relname,
    l.locktype,
    l.mode,
    l.granted,
    l.pid,
    a.query
FROM pg_locks l
JOIN pg_class c ON l.relation = c.oid
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE c.relname = 'accounts'
ORDER BY l.pid;
```

### Common Deadlock Scenarios

#### Scenario 1: Update Order Mismatch
```sql
-- Transaction 1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Locks account 1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Waits for account 2
COMMIT;

-- Transaction 2 (concurrent)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;   -- Locks account 2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- Waits for account 1
COMMIT;

-- Result: DEADLOCK!
```

**Solution**: Always acquire locks in consistent order
```sql
-- Both transactions lock accounts in same order (by ID)
BEGIN;
UPDATE accounts SET balance = balance - 100 
WHERE id = 1;  -- Lower ID first
UPDATE accounts SET balance = balance + 100 
WHERE id = 2;  -- Higher ID next
COMMIT;

-- Or use explicit locking with ordered SELECT
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Perform updates
COMMIT;
```

#### Scenario 2: Lock Upgrade Deadlock
```sql
-- Transaction 1
BEGIN;
SELECT * FROM orders WHERE id = 100;  -- Acquires AccessShareLock

-- Transaction 2
BEGIN;
SELECT * FROM orders WHERE id = 100;  -- Acquires AccessShareLock

-- Transaction 1 (continued)
UPDATE orders SET status = 'completed' WHERE id = 100;  -- Needs RowExclusiveLock, waits

-- Transaction 2 (continued)
UPDATE orders SET status = 'completed' WHERE id = 100;  -- Needs RowExclusiveLock, waits

-- Result: DEADLOCK!
```

**Solution**: Use SELECT FOR UPDATE to acquire write lock immediately
```sql
BEGIN;
SELECT * FROM orders WHERE id = 100 FOR UPDATE;  -- Acquires RowShareLock + RowExclusiveLock
UPDATE orders SET status = 'completed' WHERE id = 100;
COMMIT;
```

#### Scenario 3: Foreign Key Deadlock
```sql
-- Parent-child tables with foreign keys
-- Transaction 1
BEGIN;
INSERT INTO orders (id, customer_id) VALUES (1, 100);
-- Locks customer 100 in customers table

-- Transaction 2
BEGIN;
INSERT INTO orders (id, customer_id) VALUES (2, 101);
-- Locks customer 101 in customers table

-- Transaction 1 (continued)
UPDATE customers SET last_order_date = NOW() WHERE id = 101;
-- Waits for customer 101

-- Transaction 2 (continued)
UPDATE customers SET last_order_date = NOW() WHERE id = 100;
-- Waits for customer 100

-- Result: DEADLOCK!
```

**Solution**: Use consistent locking order or redesign schema
```sql
-- Option 1: Lock parents first
BEGIN;
SELECT * FROM customers WHERE id IN (100, 101) ORDER BY id FOR UPDATE;
INSERT INTO orders (id, customer_id) VALUES (1, 100);
UPDATE customers SET last_order_date = NOW() WHERE id = 100;
COMMIT;

-- Option 2: Use triggers to handle updates consistently
-- Option 3: Denormalize to avoid cross-table locks
```

### Preventing Deadlocks

#### Strategy 1: Consistent Lock Ordering
```sql
-- Bad: Random order
UPDATE accounts SET balance = balance + amount 
WHERE id = from_account;
UPDATE accounts SET balance = balance - amount 
WHERE id = to_account;

-- Good: Ordered by ID
BEGIN;
SELECT * FROM accounts 
WHERE id IN (from_account, to_account)
ORDER BY id
FOR UPDATE;

-- Now perform updates safely
UPDATE accounts SET balance = balance - amount WHERE id = from_account;
UPDATE accounts SET balance = balance + amount WHERE id = to_account;
COMMIT;
```

#### Strategy 2: Keep Transactions Short
```sql
-- Bad: Long transaction
BEGIN;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 123;
-- ... long-running computation ...
-- ... API call ...
-- ... user interaction ...
UPDATE orders SET status = 'completed' WHERE id = 456;
COMMIT;

-- Good: Short transaction
-- Do computation outside transaction
result = compute_something();

-- Quick transaction
BEGIN;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 123;
UPDATE orders SET status = 'completed' WHERE id = 456;
COMMIT;
```

#### Strategy 3: Use Lower Isolation Levels
```sql
-- Serializable isolation has higher deadlock risk
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Operations
COMMIT;

-- Read Committed has lower deadlock risk (default)
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Operations
COMMIT;
```

#### Strategy 4: Use Advisory Locks
```sql
-- Application-level locking to serialize critical sections
BEGIN;

-- Acquire advisory lock (blocks until available)
SELECT pg_advisory_xact_lock(hashtext('transfer:' || from_id || ':' || to_id));

-- Perform transfer
UPDATE accounts SET balance = balance - 100 WHERE id = from_id;
UPDATE accounts SET balance = balance + 100 WHERE id = to_id;

COMMIT;  -- Advisory lock automatically released
```

#### Strategy 5: Implement Retry Logic
```python
# Python example with deadlock retry
import psycopg2
import time

def transfer_with_retry(conn, from_id, to_id, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            cursor = conn.cursor()
            cursor.execute("BEGIN")
            
            # Lock accounts in consistent order
            cursor.execute("""
                SELECT * FROM accounts 
                WHERE id IN (%s, %s) 
                ORDER BY id 
                FOR UPDATE
            """, (from_id, to_id))
            
            # Perform transfer
            cursor.execute(
                "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                (amount, from_id)
            )
            cursor.execute(
                "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                (amount, to_id)
            )
            
            cursor.execute("COMMIT")
            return True
            
        except psycopg2.extensions.TransactionRollbackError as e:
            conn.rollback()
            if 'deadlock detected' in str(e):
                if attempt < max_retries - 1:
                    # Exponential backoff
                    time.sleep(0.1 * (2 ** attempt))
                    continue
            raise
        except Exception as e:
            conn.rollback()
            raise
    
    return False
```

### Analyzing Deadlocks from Logs

```bash
# PostgreSQL logs deadlock details
# postgresql.conf
log_lock_waits = on              # Log slow lock acquisitions
deadlock_timeout = 1s            # How long to wait before checking
log_min_duration_statement = 1000  # Log queries > 1 second

# Example log entry:
# ERROR: deadlock detected
# DETAIL: Process 12345 waits for ShareLock on transaction 67890; 
#         blocked by process 12346.
#         Process 12346 waits for ShareLock on transaction 67889; 
#         blocked by process 12345.
# HINT: See server log for query details.
# CONTEXT: while updating tuple (0,42) in relation "accounts"
# STATEMENT: UPDATE accounts SET balance = balance + 100 WHERE id = 2;

# Parse logs for deadlocks
grep "deadlock detected" /var/log/postgresql/postgresql-*.log
```

### Deadlock Monitoring Query

```sql
-- Create monitoring view
CREATE OR REPLACE VIEW deadlock_monitor AS
SELECT 
    d.datname,
    d.deadlocks,
    d.deadlocks - COALESCE(LAG(d.deadlocks) OVER (ORDER BY d.stats_reset), 0) as deadlocks_since_last,
    d.stats_reset,
    now() - d.stats_reset as time_since_reset
FROM pg_stat_database d
WHERE d.datname NOT IN ('template0', 'template1')
ORDER BY d.deadlocks DESC;

-- Query it
SELECT * FROM deadlock_monitor;

-- Alert if deadlocks increasing
SELECT 
    datname,
    deadlocks
FROM pg_stat_database
WHERE deadlocks > 10  -- Adjust threshold
    AND datname = current_database();
```

### Best Practices

1. **Design to avoid deadlocks**:
   - Use consistent lock ordering
   - Keep transactions short
   - Minimize lock contention

2. **Monitor deadlock frequency**:
```sql
SELECT datname, deadlocks 
FROM pg_stat_database 
ORDER BY deadlocks DESC;
```

3. **Implement retry logic**: Handle deadlock errors gracefully

4. **Use appropriate isolation levels**: Lower isolation reduces deadlock risk

5. **Log and analyze deadlocks**:
```sql
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = '1s';
SELECT pg_reload_conf();
```

6. **Consider application-level locking**: For complex scenarios

7. **Test under load**: Simulate concurrent access to find deadlock scenarios

8. **Use connection pooling**: Reduces connection overhead, allows better control

- **Further Reading**: [PostgreSQL Deadlocks](https://www.postgresql.org/docs/18/explicit-locking.html#LOCKING-DEADLOCKS) | [Lock Management](https://www.postgresql.org/docs/18/explicit-locking.html) | [Monitoring Locks](https://www.postgresql.org/docs/18/monitoring-locks.html)

## 11. Metadata Using System Catalogs

### Overview
PostgreSQL maintains extensive metadata about the database cluster in system catalogs. These catalogs provide information about tables, indexes, columns, functions, users, and more. Understanding system catalogs is essential for database administration, monitoring, and troubleshooting.

### Key System Catalogs

```
┌────────────────────────────────────────────────────────────┐
│              PostgreSQL System Catalogs                    │
├────────────────────────────────────────────────────────────┤
│ pg_class       → Tables, indexes, views, sequences         │
│ pg_attribute   → Table columns and attributes              │
│ pg_index       → Index definitions                         │
│ pg_constraint  → Constraints (PK, FK, CHECK)              │
│ pg_type        → Data types                                │
│ pg_proc        → Functions and procedures                  │
│ pg_namespace   → Schemas                                   │
│ pg_database    → Databases in the cluster                 │
│ pg_tablespace  → Tablespaces                              │
│ pg_roles       → Database roles (users/groups)            │
│ pg_stat_*      → Statistics views                         │
│ pg_settings    → Configuration parameters                  │
└────────────────────────────────────────────────────────────┘
```

### pg_class - Tables and Relations

```sql
-- View all user tables
SELECT 
    n.nspname as schema,
    c.relname as name,
    CASE c.relkind
        WHEN 'r' THEN 'table'
        WHEN 'i' THEN 'index'
        WHEN 'S' THEN 'sequence'
        WHEN 'v' THEN 'view'
        WHEN 'm' THEN 'materialized view'
        WHEN 'f' THEN 'foreign table'
        WHEN 'p' THEN 'partitioned table'
    END as type,
    pg_size_pretty(pg_total_relation_size(c.oid)) as total_size,
    c.reltuples::bigint as estimated_rows,
    c.relpages as pages
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
    AND c.relkind IN ('r', 'p')  -- Regular and partitioned tables
ORDER BY pg_total_relation_size(c.oid) DESC;

-- Get detailed table information
SELECT 
    c.relname as table_name,
    c.relnamespace::regnamespace as schema,
    c.relowner::regrole as owner,
    c.relpersistence as persistence,  -- p=permanent, u=unlogged, t=temporary
    c.relkind as kind,
    c.relhasindex as has_indexes,
    c.relhasrules as has_rules,
    c.relhastriggers as has_triggers,
    c.relrowsecurity as row_security,
    c.relforcerowsecurity as force_row_security,
    c.relreplident as replica_identity,
    pg_size_pretty(pg_relation_size(c.oid)) as table_size,
    pg_size_pretty(pg_total_relation_size(c.oid)) as total_size
FROM pg_class c
WHERE c.relname = 'users'
    AND c.relkind = 'r';

-- Check table statistics
SELECT 
    schemaname,
    tablename,
    n_live_tup as live_rows,
    n_dead_tup as dead_rows,
    n_tup_ins as inserts,
    n_tup_upd as updates,
    n_tup_del as deletes,
    n_tup_hot_upd as hot_updates,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE tablename = 'users';
```

### pg_attribute - Column Information

```sql
-- List columns for a table
SELECT 
    a.attnum as position,
    a.attname as column_name,
    pg_catalog.format_type(a.atttypid, a.atttypmod) as data_type,
    a.attnotnull as not_null,
    a.atthasdef as has_default,
    pg_catalog.pg_get_expr(d.adbin, d.adrelid) as default_value,
    col_description(c.oid, a.attnum) as description
FROM pg_catalog.pg_attribute a
JOIN pg_catalog.pg_class c ON a.attrelid = c.oid
LEFT JOIN pg_catalog.pg_attrdef d ON (a.attrelid = d.adrelid AND a.attnum = d.adnum)
WHERE c.relname = 'users'
    AND a.attnum > 0  -- Exclude system columns
    AND NOT a.attisdropped  -- Exclude dropped columns
ORDER BY a.attnum;

-- Find all columns of a specific type
SELECT 
    n.nspname as schema,
    c.relname as table,
    a.attname as column,
    pg_catalog.format_type(a.atttypid, a.atttypmod) as type
FROM pg_attribute a
JOIN pg_class c ON a.attrelid = c.oid
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE pg_catalog.format_type(a.atttypid, a.atttypmod) LIKE '%json%'
    AND n.nspname NOT IN ('pg_catalog', 'information_schema')
    AND a.attnum > 0
    AND NOT a.attisdropped
ORDER BY n.nspname, c.relname, a.attnum;
```

### pg_index - Index Information

```sql
-- List all indexes with details
SELECT 
    n.nspname as schema,
    t.relname as table,
    i.relname as index,
    a.amname as index_type,
    ix.indisunique as is_unique,
    ix.indisprimary as is_primary,
    ix.indisvalid as is_valid,
    pg_size_pretty(pg_relation_size(i.oid)) as index_size,
    pg_get_indexdef(ix.indexrelid) as index_definition
FROM pg_index ix
JOIN pg_class t ON t.oid = ix.indrelid
JOIN pg_class i ON i.oid = ix.indexrelid
JOIN pg_namespace n ON n.oid = t.relnamespace
JOIN pg_am a ON a.oid = i.relam
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(i.oid) DESC;

-- Find unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    round(100.0 * idx_scan / NULLIF(
        (SELECT SUM(idx_scan) FROM pg_stat_user_indexes WHERE tablename = psu.tablename), 0
    ), 2) as scan_percentage
FROM pg_stat_user_indexes psu
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;
```

### pg_stat_* Views - Statistics

#### pg_stat_database
```sql
-- Database-level statistics
SELECT 
    datname,
    numbackends as connections,
    xact_commit as commits,
    xact_rollback as rollbacks,
    blks_read as disk_blocks_read,
    blks_hit as cache_blocks_hit,
    round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) as cache_hit_ratio,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted,
    conflicts,
    temp_files,
    pg_size_pretty(temp_bytes) as temp_size,
    deadlocks,
    blk_read_time,
    blk_write_time,
    stats_reset
FROM pg_stat_database
WHERE datname = current_database();
```

#### pg_stat_user_tables
```sql
-- Table-level statistics
SELECT 
    schemaname,
    relname,
    seq_scan,  -- Sequential scans
    seq_tup_read,  -- Rows read by sequential scans
    idx_scan,  -- Index scans
    idx_tup_fetch,  -- Rows fetched by index scans
    n_tup_ins,  -- Inserts
    n_tup_upd,  -- Updates
    n_tup_del,  -- Deletes
    n_tup_hot_upd,  -- HOT (Heap-Only Tuple) updates
    n_live_tup,  -- Estimated live rows
    n_dead_tup,  -- Estimated dead rows
    n_mod_since_analyze,  -- Rows modified since last ANALYZE
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- Tables needing attention
SELECT 
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
    AND n_live_tup > 0
ORDER BY dead_ratio DESC;
```

#### pg_stat_activity
```sql
-- Current database activity
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    client_port,
    backend_start,
    xact_start,
    query_start,
    state_change,
    wait_event_type,
    wait_event,
    state,
    backend_type,
    LEFT(query, 100) as query_preview
FROM pg_stat_activity
WHERE state != 'idle'
    AND pid != pg_backend_pid()
ORDER BY xact_start NULLS LAST;

-- Long-running queries
SELECT 
    pid,
    now() - xact_start AS duration,
    usename,
    query
FROM pg_stat_activity
WHERE state != 'idle'
    AND xact_start IS NOT NULL
    AND now() - xact_start > interval '5 minutes'
ORDER BY duration DESC;

-- Blocked queries
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted
    AND blocking_locks.granted;
```

### pg_settings - Configuration

```sql
-- View all configuration settings
SELECT 
    name,
    setting,
    unit,
    category,
    short_desc,
    context,  -- When setting can be changed
    vartype,  -- bool, integer, real, string, enum
    source,   -- Where setting came from
    min_val,
    max_val,
    enumvals,  -- For enum types
    boot_val,  -- Default value
    reset_val  -- Value at session start
FROM pg_settings
WHERE name LIKE '%work_mem%'
ORDER BY name;

-- Modified settings
SELECT 
    name,
    setting,
    unit,
    source,
    sourcefile,
    sourceline
FROM pg_settings
WHERE source != 'default'
    AND source != 'override'
ORDER BY name;

-- Settings that require restart
SELECT 
    name,
    setting,
    unit,
    short_desc
FROM pg_settings
WHERE context = 'postmaster'
ORDER BY name;
```

### pg_constraint - Constraints

```sql
-- List all constraints
SELECT 
    n.nspname as schema,
    t.relname as table,
    c.conname as constraint_name,
    CASE c.contype
        WHEN 'p' THEN 'PRIMARY KEY'
        WHEN 'f' THEN 'FOREIGN KEY'
        WHEN 'u' THEN 'UNIQUE'
        WHEN 'c' THEN 'CHECK'
        WHEN 'x' THEN 'EXCLUSION'
    END as constraint_type,
    pg_get_constraintdef(c.oid) as definition
FROM pg_constraint c
JOIN pg_class t ON c.conrelid = t.oid
JOIN pg_namespace n ON t.relnamespace = n.oid
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, t.relname, c.contype;

-- Foreign key relationships
SELECT 
    tc.table_schema,
    tc.table_name,
    kcu.column_name,
    ccu.table_schema AS foreign_table_schema,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name,
    rc.update_rule,
    rc.delete_rule
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
    AND tc.table_schema = kcu.table_schema
JOIN information_schema.constraint_column_usage AS ccu
    ON ccu.constraint_name = tc.constraint_name
    AND ccu.table_schema = tc.table_schema
JOIN information_schema.referential_constraints AS rc
    ON tc.constraint_name = rc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
    AND tc.table_schema = 'public'
ORDER BY tc.table_name;
```

### Practical Monitoring Queries

#### Database Size and Growth
```sql
-- Database sizes
SELECT 
    datname,
    pg_size_pretty(pg_database_size(datname)) as size,
    age(datfrozenxid) as xid_age
FROM pg_database
WHERE datallowconn
ORDER BY pg_database_size(datname) DESC;

-- Table sizes with indexes
SELECT 
    n.nspname as schema,
    c.relname as table,
    pg_size_pretty(pg_table_size(c.oid)) as table_size,
    pg_size_pretty(pg_indexes_size(c.oid)) as indexes_size,
    pg_size_pretty(pg_total_relation_size(c.oid)) as total_size,
    round(100.0 * pg_indexes_size(c.oid) / NULLIF(pg_total_relation_size(c.oid), 0), 2) as index_ratio
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
    AND c.relkind = 'r'
ORDER BY pg_total_relation_size(c.oid) DESC
LIMIT 20;
```

#### Cache Hit Ratios
```sql
-- Overall cache hit ratio
SELECT 
    'Cache Hit Ratio' as metric,
    round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0), 2) || '%' as value
FROM pg_stat_database
WHERE datname = current_database();

-- Per-table cache hit ratio
SELECT 
    schemaname,
    tablename,
    heap_blks_read,
    heap_blks_hit,
    round(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY heap_blks_read DESC;
```

#### Index Bloat Estimation
```sql
-- Estimate index bloat
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    CASE 
        WHEN idx_scan = 0 THEN 'Unused'
        WHEN idx_tup_read = 0 THEN 'Never read'
        ELSE 'In use'
    END as usage_status
FROM pg_stat_user_indexes
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Creating Custom Monitoring Views

```sql
-- Comprehensive database health view
CREATE OR REPLACE VIEW database_health AS
SELECT 
    'Connections' as metric,
    current_setting('max_connections')::int - 
        (SELECT count(*) FROM pg_stat_activity) as available,
    (SELECT count(*) FROM pg_stat_activity) as used,
    current_setting('max_connections') as maximum
UNION ALL
SELECT 
    'Cache Hit Ratio',
    round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0), 2),
    NULL,
    '95%+ is good'
FROM pg_stat_database
WHERE datname = current_database()
UNION ALL
SELECT 
    'Deadlocks',
    sum(deadlocks),
    NULL,
    'Should be 0'
FROM pg_stat_database
WHERE datname = current_database()
UNION ALL
SELECT 
    'Transaction ID Age',
    max(age(datfrozenxid)),
    NULL,
    '< 200M is safe'
FROM pg_database;

-- Query it
SELECT * FROM database_health;
```

### Best Practices

1. **Regular Statistics Updates**: Keep statistics current
```sql
ANALYZE;  -- Update statistics for all tables
```

2. **Monitor Key Metrics**: Set up alerts
   - Cache hit ratio < 95%
   - Deadlocks > 0
   - XID age > 200 million
   - Connection usage > 80%

3. **Use pg_stat_statements**: Track query performance
```sql
CREATE EXTENSION pg_stat_statements;
```

4. **Reset Statistics Periodically**: For baseline comparisons
```sql
SELECT pg_stat_reset();
```

5. **Document Schema**: Use comments
```sql
COMMENT ON TABLE users IS 'Application user accounts';
COMMENT ON COLUMN users.email IS 'Unique email address';
```

6. **Automate Monitoring**: Use tools like pgAdmin, Prometheus, or custom scripts

- **Further Reading**: [PostgreSQL System Catalogs](https://www.postgresql.org/docs/18/catalogs.html) | [Statistics Views](https://www.postgresql.org/docs/18/monitoring-stats.html) | [pg_stat_activity](https://www.postgresql.org/docs/18/monitoring-stats.html#MONITORING-PG-STAT-ACTIVITY-VIEW)

## 12. Replication Types

### Overview
PostgreSQL supports multiple replication methods, each designed for different use cases. Replication provides high availability, load balancing, and disaster recovery capabilities.

### Replication Architecture

```
┌────────────────────────────────────────────────────────────┐
│           PostgreSQL Replication Types                     │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Physical Replication (WAL-based)                          │
│  ├─ Streaming Replication                                  │
│  │  ├─ Synchronous                                         │
│  │  └─ Asynchronous                                        │
│  └─ Log Shipping (File-based)                              │
│                                                             │
│  Logical Replication                                       │
│  ├─ Publish/Subscribe                                      │
│  ├─ Selective replication (tables/schemas)                │
│  └─ Cross-version replication                             │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Physical Replication (Streaming)

Physical replication replicates the entire database cluster at the block level by streaming WAL records.

#### Characteristics
- **Replicates**: Entire database cluster
- **Granularity**: Block-level (binary replication)
- **Standby Access**: Read-only queries (hot standby)
- **Version**: Same major version required
- **Consistency**: Byte-for-byte identical
- **Performance**: Very efficient, minimal overhead

#### Setting Up Streaming Replication

**On Primary Server**:
```sql
-- postgresql.conf
wal_level = replica                    -- Enable WAL for replication
max_wal_senders = 10                   -- Max replication connections
wal_keep_size = 1GB                    -- Keep WAL for standbys (PG 13+)
max_replication_slots = 10             -- Replication slots
hot_standby = on                       -- Allow queries on standby
archive_mode = on                      -- Enable WAL archiving
archive_command = 'cp %p /archive/%f'  -- Archive WAL files

-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_password';

-- pg_hba.conf - Allow replication connections
# host    replication     replicator      standby_ip/32      scram-sha-256

-- Create replication slot (recommended)
SELECT pg_create_physical_replication_slot('standby_slot');

-- Reload configuration
SELECT pg_reload_conf();
```

**On Standby Server**:
```bash
# Stop PostgreSQL on standby
pg_ctl stop -D /var/lib/postgresql/data

# Take base backup from primary
# Security Note: Use .pgpass file or PGPASSWORD environment variable
# to avoid exposing passwords in command history
# .pgpass format: hostname:port:database:username:password
pg_basebackup -h primary_host -D /var/lib/postgresql/data -U replicator -P -Xs -R

# The -R flag automatically creates standby.signal and configures recovery
```

```sql
-- postgresql.conf on standby
hot_standby = on                       -- Allow read-only queries
max_standby_streaming_delay = 30s      -- Max delay before canceling queries
wal_receiver_status_interval = 10s     -- Report status to primary
hot_standby_feedback = on              -- Prevent query conflicts

-- postgresql.auto.conf (created by pg_basebackup -R)
primary_conninfo = 'host=primary_host port=5432 user=replicator password=secure_password'
primary_slot_name = 'standby_slot'     -- Use replication slot

# Start standby
pg_ctl start -D /var/lib/postgresql/data
```

#### Monitoring Streaming Replication

**On Primary**:
```sql
-- View replication status
SELECT 
    pid,
    usesysid,
    usename,
    application_name,
    client_addr,
    client_hostname,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    sync_state,  -- async, potential, sync, quorum
    pg_wal_lsn_diff(sent_lsn, replay_lsn) as replication_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

-- Check replication slots
SELECT 
    slot_name,
    slot_type,  -- physical or logical
    database,
    active,
    xmin,
    catalog_xmin,
    restart_lsn,
    confirmed_flush_lsn,
    wal_status,  -- reserved, extended, unreserved
    safe_wal_size,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) as retained_wal
FROM pg_replication_slots;

-- WAL sender processes
SELECT 
    pid,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

**On Standby**:
```sql
-- Check if in recovery mode
SELECT pg_is_in_recovery();  -- Should return true

-- Replication status
SELECT 
    pg_is_in_recovery(),
    pg_last_wal_receive_lsn() as receive_lsn,
    pg_last_wal_replay_lsn() as replay_lsn,
    pg_last_xact_replay_timestamp() as last_replay_time,
    now() - pg_last_xact_replay_timestamp() as replication_lag;

-- WAL receiver status
SELECT 
    pid,
    status,
    receive_start_lsn,
    receive_start_tli,
    received_lsn,
    received_tli,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time,
    slot_name,
    sender_host,
    sender_port,
    conninfo
FROM pg_stat_wal_receiver;
```

### Synchronous vs Asynchronous Replication

#### Asynchronous Replication (Default)
```
┌────────────────────────────────────────────────────────────┐
│              Asynchronous Replication                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Primary                           Standby                 │
│  ├─ COMMIT returns immediately    ├─ Receives WAL later   │
│  ├─ No wait for standby           ├─ May lag behind        │
│  └─ Best performance               └─ Potential data loss  │
│                                                             │
│  Use case: High performance, acceptable data loss risk    │
└────────────────────────────────────────────────────────────┘
```

```sql
-- Configure asynchronous (default)
-- postgresql.conf on primary
synchronous_commit = off  -- Or 'on' for async replication
synchronous_standby_names = ''  -- Empty = async
```

#### Synchronous Replication
```
┌────────────────────────────────────────────────────────────┐
│              Synchronous Replication                       │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Primary                           Standby                 │
│  ├─ COMMIT waits for standby      ├─ Confirms WAL receipt │
│  ├─ Ensures durability            ├─ Always up-to-date    │
│  └─ Higher latency                └─ No data loss         │
│                                                             │
│  Use case: Zero data loss, critical transactions           │
└────────────────────────────────────────────────────────────┘
```

```sql
-- Configure synchronous
-- postgresql.conf on primary
synchronous_commit = on                          -- or 'remote_apply'
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'  -- Wait for 1

-- Levels of synchronous_commit:
-- • on: Wait for WAL flush to disk on primary
-- • remote_write: Wait for standby to write WAL (not flush)
-- • remote_apply: Wait for standby to apply WAL (strongest)
-- • local: Flush on primary only
-- • off: Don't wait (fastest, least safe)

-- Multiple standbys with quorum
synchronous_standby_names = 'ANY 2 (standby1, standby2, standby3)'  -- Wait for any 2

-- Check sync state
SELECT application_name, sync_state FROM pg_stat_replication;
-- async: Asynchronous
-- potential: Could become sync
-- sync: Currently synchronous
-- quorum: Part of quorum group
```

### Replication Slots

Replication slots prevent WAL deletion until all subscribers have received it.

```sql
-- Create physical replication slot
SELECT pg_create_physical_replication_slot('slot_name');

-- Create logical replication slot
SELECT pg_create_logical_replication_slot('slot_name', 'pgoutput');

-- List replication slots
SELECT * FROM pg_replication_slots;

-- Drop replication slot
SELECT pg_drop_replication_slot('slot_name');

-- Advance replication slot (skip WAL)
SELECT pg_replication_slot_advance('slot_name', 'target_lsn');

-- Monitor slot lag
SELECT 
    slot_name,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) as lag
FROM pg_replication_slots
WHERE active = true;
```

### Logical Replication

Logical replication replicates data changes at the logical level (SQL statements), allowing selective replication.

#### Characteristics
- **Replicates**: Selective tables/schemas
- **Granularity**: Row-level changes
- **Standby Access**: Read-write (separate database)
- **Version**: Cross-version compatible
- **Consistency**: Logical consistency
- **Flexibility**: Transform data during replication

#### Use Cases
- Replicating subset of database
- Consolidating multiple databases
- Upgrading PostgreSQL versions
- Real-time analytics
- Multi-master scenarios (with extensions)

#### Setting Up Logical Replication

**On Publisher (Source)**:
```sql
-- postgresql.conf
wal_level = logical                    -- Enable logical replication
max_replication_slots = 10
max_wal_senders = 10

-- Reload configuration
SELECT pg_reload_conf();

-- Create publication (all tables in schema)
CREATE PUBLICATION my_publication FOR ALL TABLES;

-- Or specific tables
CREATE PUBLICATION orders_publication 
FOR TABLE orders, order_items;

-- Or with filtering (PostgreSQL 15+)
CREATE PUBLICATION active_users_pub 
FOR TABLE users WHERE (active = true);

-- View publications
SELECT * FROM pg_publication;

-- View publication tables
SELECT * FROM pg_publication_tables;

-- Grant replication permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;
```

**On Subscriber (Target)**:
```sql
-- Create matching table structure
-- (Must have same columns or use column_list)
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date DATE,
    total DECIMAL(10,2)
);

-- Create subscription
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=publisher_host port=5432 dbname=sourcedb user=replication_user password=secure_pass'
PUBLICATION my_publication;

-- View subscriptions
SELECT * FROM pg_subscription;

-- View subscription relations
SELECT * FROM pg_subscription_rel;

-- Monitor subscription
SELECT 
    subname,
    pid,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_subscription;
```

#### Managing Logical Replication

```sql
-- Refresh subscription (sync structure changes)
ALTER SUBSCRIPTION my_subscription REFRESH PUBLICATION;

-- Disable subscription temporarily
ALTER SUBSCRIPTION my_subscription DISABLE;

-- Enable subscription
ALTER SUBSCRIPTION my_subscription ENABLE;

-- Drop subscription
DROP SUBSCRIPTION my_subscription;

-- Add table to publication
ALTER PUBLICATION my_publication ADD TABLE new_table;

-- Remove table from publication
ALTER PUBLICATION my_publication DROP TABLE old_table;

-- Change publication to all tables
ALTER PUBLICATION my_publication SET ALL TABLES;
```

### Cascading Replication

Standbys can replicate to other standbys, creating a replication chain.

```
┌────────────────────────────────────────────────────────────┐
│              Cascading Replication                         │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Primary → Standby 1 → Standby 2 → Standby 3              │
│                                                             │
│  Reduces load on primary                                   │
│  Enables geographically distributed replicas               │
└────────────────────────────────────────────────────────────┘
```

```sql
-- On Standby 1 (becomes cascading standby)
-- postgresql.conf
max_wal_senders = 10
hot_standby = on

-- Standby 2 connects to Standby 1
-- primary_conninfo points to Standby 1
primary_conninfo = 'host=standby1_host port=5432 user=replicator'
```

### Replication Comparison

```
┌──────────────────┬───────────────────┬──────────────────────┐
│ Feature          │ Physical          │ Logical              │
├──────────────────┼───────────────────┼──────────────────────┤
│ Granularity      │ Cluster-wide      │ Table-level          │
│ Standby writes   │ Read-only         │ Read-write           │
│ Version compat   │ Same major        │ Cross-version        │
│ Performance      │ Very fast         │ Moderate             │
│ Disk usage       │ Full copy         │ Selective            │
│ DDL replication  │ Yes               │ No (manual)          │
│ Sequence sync    │ Yes               │ No                   │
│ Partial replica  │ No                │ Yes                  │
│ Failover         │ Simple            │ Complex              │
│ Conflict resolv  │ N/A               │ Required             │
└──────────────────┴───────────────────┴──────────────────────┘
```

### Space and Time Efficiency

#### Physical Replication Efficiency
```sql
-- WAL is efficiently streamed (compressed possible)
-- No duplicate data processing
-- Minimal CPU overhead on primary

-- Monitor WAL generation rate
SELECT 
    pg_current_wal_lsn(),
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')
    ) as total_wal;

-- With wal_compression (PostgreSQL 14+)
-- postgresql.conf
wal_compression = on  -- Can reduce WAL size by 50-90%
```

#### Logical Replication Efficiency
```sql
-- Replicates only specified tables
-- More CPU overhead (decode WAL, apply changes)
-- Network efficiency: Only changed data

-- Monitor logical replication lag
SELECT 
    slot_name,
    database,
    pg_size_pretty(
        pg_wal_lsn_diff(confirmed_flush_lsn, pg_current_wal_lsn())
    ) as lag_bytes
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

### Failover and High Availability

#### Promoting Standby to Primary
```bash
# Automatic promotion with pg_auto_failover or Patroni

# Manual promotion:
# 1. Stop primary (if possible)
pg_ctl stop -D /var/lib/postgresql/data -m fast

# 2. Promote standby
pg_ctl promote -D /var/lib/postgresql/data

# Or using SQL
SELECT pg_promote();
```

```sql
-- Verify promotion
SELECT pg_is_in_recovery();  -- Should return false

-- Reconfigure other standbys to follow new primary
-- Update primary_conninfo on remaining standbys
```

#### pg_auto_failover (Automated HA)
```bash
# Modern HA solution for PostgreSQL
# Automatic failover, monitoring, and orchestration

# Install pg_auto_failover
# Configure monitor node
pg_autoctl create monitor --hostname monitor_host --pgdata /data/monitor

# Configure primary
pg_autoctl create postgres --hostname primary_host --pgdata /data/primary \
    --monitor postgresql://autoctl_node@monitor_host/pg_auto_failover

# Configure standbys
pg_autoctl create postgres --hostname standby_host --pgdata /data/standby \
    --monitor postgresql://autoctl_node@monitor_host/pg_auto_failover
```

### Best Practices

1. **Use Replication Slots**: Prevent WAL deletion
```sql
SELECT pg_create_physical_replication_slot('standby_slot');
```

2. **Monitor Replication Lag**: Set alerts
```sql
SELECT 
    application_name,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as lag_bytes,
    replay_lag
FROM pg_stat_replication;
```

3. **Choose Appropriate Replication Type**:
   - Physical: Full replica, high availability, disaster recovery
   - Logical: Selective replication, upgrades, analytics

4. **Test Failover Procedures**: Regular drills
```bash
# Practice promoting standby
# Verify application can reconnect
# Test recovery time objectives (RTO)
```

5. **Use Synchronous for Critical Data**:
```sql
ALTER SYSTEM SET synchronous_commit = 'remote_apply';
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (standby1)';
```

6. **Archive WAL**: For point-in-time recovery
```sql
ALTER SYSTEM SET archive_mode = on;
ALTER SYSTEM SET archive_command = 'cp %p /archive/%f';
```

7. **Monitor Slot Disk Usage**: Prevent disk fill
```sql
SELECT 
    slot_name,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) as retained_wal
FROM pg_replication_slots
WHERE active = false;  -- Alert if inactive slots retain WAL
```

8. **Use Connection Pooling**: For standbys accepting read traffic
   - pgBouncer or pgpool-II
   - Load balance read queries

9. **Consider HA Solutions**: For production
   - Patroni
   - pg_auto_failover
   - Stolon
   - repmgr

10. **Document Topology**: Keep replication architecture documented

- **Further Reading**: [PostgreSQL Replication](https://www.postgresql.org/docs/18/high-availability.html) | [Streaming Replication](https://www.postgresql.org/docs/18/warm-standby.html) | [Logical Replication](https://www.postgresql.org/docs/18/logical-replication.html) | [Replication Slots](https://www.postgresql.org/docs/18/warm-standby.html#STREAMING-REPLICATION-SLOTS)

---

## Conclusion

This comprehensive guide has covered the essential and advanced topics of PostgreSQL, providing you with:

### Key Takeaways

1. **Vacuuming** is critical for database health - keep autovacuum enabled and monitor dead tuples
2. **Indexing** strategies can dramatically improve query performance - use the right index type for your use case
3. **WAL (Write-Ahead Logging)** ensures durability and enables replication and recovery
4. **Caching** with proper memory configuration (shared_buffers, work_mem) significantly impacts performance
5. **Transactions** with MVCC provide powerful concurrency control - understand isolation levels
6. **Large Object Storage** vs BYTEA - choose based on size and access patterns
7. **Postmaster Process** architecture and background workers maintain database operations
8. **Transaction ID Wraparound** requires monitoring - prevent the 4 billion transaction limit issue
9. **EXPLAIN Plans** are essential for query optimization - learn to read and interpret them
10. **Deadlocks** can be prevented with careful design - use consistent lock ordering
11. **System Catalogs** (pg_class, pg_stat_*) provide extensive metadata for monitoring
12. **Replication** (Physical vs Logical) enables high availability - choose based on requirements

### Next Steps

- **Practice**: Set up a test PostgreSQL environment and experiment with these concepts
- **Monitor**: Implement monitoring for your production databases using the queries provided
- **Optimize**: Use EXPLAIN ANALYZE to optimize your slow queries
- **Secure**: Follow PostgreSQL security best practices
- **Scale**: Plan for replication and high availability before you need it
- **Stay Updated**: Follow PostgreSQL release notes and documentation

### Additional Resources

- **Official Documentation**: [PostgreSQL 18 Documentation](https://www.postgresql.org/docs/18/index.html)
- **Community**: [PostgreSQL Mailing Lists](https://www.postgresql.org/list/)
- **Books**: "PostgreSQL: Up and Running" by Regina Obe and Leo Hsu
- **Conferences**: PGCon, PostgreSQL Conference Europe, PGConf
- **Tools**: pgAdmin, DBeaver, pg_stat_statements, pg_auto_failover

### Contributing

PostgreSQL is open-source and welcomes contributions. Consider:
- Reporting bugs
- Contributing code
- Writing documentation
- Helping in community forums
- Sharing your knowledge

### Final Words

PostgreSQL is a powerful, feature-rich database system with decades of development. Mastering these topics will help you build reliable, performant, and scalable applications. Remember:

- **Monitor regularly**: Use the system catalogs and statistics views
- **Plan for growth**: Design with scalability in mind
- **Test thoroughly**: Especially replication and recovery procedures
- **Document everything**: Your future self will thank you
- **Stay curious**: PostgreSQL is constantly evolving with new features

Happy PostgreSQL administration! 🐘

---

*This guide was created based on PostgreSQL 18 documentation and best practices as of January 2024. Always refer to the official PostgreSQL documentation for the most up-to-date information.*