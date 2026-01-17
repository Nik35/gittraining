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
Transactions ensure a sequence of database operations is executed in a coherent way.

- **Example**: `BEGIN; INSERT INTO my_table VALUES (1); COMMIT;`
- **Further Reading**: [PostgreSQL Transactions Documentation](https://www.postgresql.org/docs/18/sql-begin.html)

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