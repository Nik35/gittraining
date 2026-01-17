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
│  │  •Manages background workers                        │ │
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