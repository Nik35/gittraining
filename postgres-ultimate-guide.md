# PostgreSQL Ultimate Guide

## Introduction
This guide covers essential PostgreSQL topics for database enthusiasts and professionals.

## 1. Vacuuming
Vacuuming is a crucial maintenance task in PostgreSQL that helps reclaim storage and maintain performance. It removes dead tuples from tables and indexes.

- **Example**: `VACUUM my_table;`
- **Further Reading**: [PostgreSQL Vacuuming Documentation](https://www.postgresql.org/docs/18/sql-vacuum.html)

## 2. Indexing
Indexes improve the speed of data retrieval operations. Proper use of indexes can significantly enhance query performance.

- **Example**: `CREATE INDEX idx_name ON my_table (column_name);`
- **Further Reading**: [PostgreSQL Indexing Documentation](https://www.postgresql.org/docs/18/indexes.html)

## 3. Write-Ahead Logging (WAL)
WAL is a standard method for ensuring data integrity. It logs all changes made to the database before they are applied.

- **Example**: `SELECT pg_current_wal_lsn();`
- **Further Reading**: [PostgreSQL WAL Documentation](https://www.postgresql.org/docs/18/wal.html)

## 4. Caching
PostgreSQL uses shared buffers for caching data in memory, improving performance for read-heavy workloads.

- **Example**: Set `shared_buffers` in `postgresql.conf` for optimal performance.
- **Further Reading**: [PostgreSQL Caching Documentation](https://www.postgresql.org/docs/18/runtime-config-resource.html#RUNTIME-CONFIG-RESOURCE-SHARED-BUFFERS)

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

- **Example**: `SELECT * FROM pg_stat_replication;`
- **Further Reading**: [PostgreSQL Replication Documentation](https://www.postgresql.org/docs/18/warm-standby.html)

## 13. PostgreSQL Installation

Installing PostgreSQL varies by operating system. This section provides step-by-step installation instructions for Linux, Windows, and Mac.

### Linux Installation

#### Ubuntu/Debian
```bash
# Update package lists
sudo apt update

# Install PostgreSQL
sudo apt install postgresql postgresql-contrib

# Start PostgreSQL service
sudo systemctl start postgresql

# Enable PostgreSQL to start on boot
sudo systemctl enable postgresql

# Check PostgreSQL status
sudo systemctl status postgresql

# Switch to postgres user and access PostgreSQL
sudo -i -u postgres
psql
```

#### Red Hat/CentOS/Fedora
```bash
# Install PostgreSQL repository
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable built-in PostgreSQL module (if on CentOS 8+)
sudo dnf -qy module disable postgresql

# Install PostgreSQL
sudo dnf install -y postgresql16-server

# Initialize database
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# Start and enable PostgreSQL service
sudo systemctl start postgresql-16
sudo systemctl enable postgresql-16

# Access PostgreSQL
sudo -u postgres psql
```

### Windows Installation

1. **Download the Installer**
   - Visit [PostgreSQL Downloads](https://www.postgresql.org/download/windows/)
   - Download the latest PostgreSQL installer for Windows

2. **Run the Installer**
   - Double-click the downloaded `.exe` file
   - Click "Next" through the setup wizard

3. **Select Components**
   - PostgreSQL Server (required)
   - pgAdmin 4 (GUI tool - recommended)
   - Stack Builder (optional, for additional tools)
   - Command Line Tools (recommended)

4. **Choose Installation Directory**
   - Default: `C:\Program Files\PostgreSQL\16\`

5. **Set Password**
   - Enter a strong password for the `postgres` superuser
   - **Important**: Remember this password!

6. **Configure Port**
   - Default port: `5432`
   - Change only if port 5432 is already in use

7. **Select Locale**
   - Choose your preferred locale (default is usually fine)

8. **Complete Installation**
   - Click "Finish" to complete the installation

9. **Verify Installation**
   - Open Command Prompt or PowerShell
   - Navigate to PostgreSQL bin directory:
     ```cmd
     cd "C:\Program Files\PostgreSQL\16\bin"
     ```
   - Connect to PostgreSQL:
     ```cmd
     psql -U postgres
     ```

### Mac Installation

#### Using Homebrew (Recommended)
```bash
# Install Homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Update Homebrew
brew update

# Install PostgreSQL
brew install postgresql@16

# Start PostgreSQL service
brew services start postgresql@16

# Create default database
createdb `whoami`

# Access PostgreSQL
psql postgres
```

#### Using Postgres.app
1. Download Postgres.app from [postgresapp.com](https://postgresapp.com/)
2. Move the app to your Applications folder
3. Double-click to start PostgreSQL
4. Click "Initialize" to create a new database cluster
5. Configure your PATH (add to `~/.zshrc` or `~/.bash_profile`):
   ```bash
   export PATH="/Applications/Postgres.app/Contents/Versions/latest/bin:$PATH"
   ```
6. Reload your shell configuration:
   ```bash
   source ~/.zshrc  # or source ~/.bash_profile
   ```

### Post-Installation Steps

After installation on any platform:

1. **Create a New User**
   ```sql
   CREATE USER myuser WITH PASSWORD 'mypassword';
   ```

2. **Create a Database**
   ```sql
   CREATE DATABASE mydatabase OWNER myuser;
   ```

3. **Grant Privileges**
   ```sql
   GRANT ALL PRIVILEGES ON DATABASE mydatabase TO myuser;
   ```

4. **Test Connection**
   ```bash
   psql -U myuser -d mydatabase -h localhost
   ```

- **Further Reading**: [PostgreSQL Installation Documentation](https://www.postgresql.org/docs/current/installation.html)

## 14. Replication Configuration

PostgreSQL supports multiple replication strategies for high availability and disaster recovery. This section covers both Write-Ahead Log (WAL) based streaming replication and logical replication.

### Understanding Replication Types

- **Physical Replication (WAL-based)**: Replicates the entire database cluster at the byte level. All databases are replicated together.
- **Logical Replication**: Replicates data changes at the logical level (individual tables/databases). Provides more flexibility but with some limitations.

### Full WAL-Based Streaming Replication

WAL-based replication creates an exact copy of the primary server on one or more standby servers. It's the most common replication method.

#### Prerequisites
- Two or more PostgreSQL servers (Primary and Standby)
- Network connectivity between servers
- Same PostgreSQL version on all servers
- Sufficient disk space on standby server

#### Step 1: Configure Primary Server

**Edit `postgresql.conf` on Primary Server:**
```conf
# Replication Settings
wal_level = replica                    # Minimal WAL level for replication
max_wal_senders = 10                   # Maximum concurrent connections from standby servers
wal_keep_size = 1GB                    # Amount of WAL to retain for standby servers
hot_standby = on                       # Allow read queries on standby
archive_mode = on                      # Enable WAL archiving
archive_command = 'test ! -f /var/lib/postgresql/archive/%f && cp %p /var/lib/postgresql/archive/%f'
synchronous_commit = on                # Wait for WAL to be written to disk
max_replication_slots = 10             # Maximum replication slots

# Ensure these are also set appropriately
listen_addresses = '*'                 # Listen on all interfaces
port = 5432
```

**Create Archive Directory on Primary:**
```bash
sudo mkdir -p /var/lib/postgresql/archive
sudo chown postgres:postgres /var/lib/postgresql/archive
sudo chmod 700 /var/lib/postgresql/archive
```

**Edit `pg_hba.conf` on Primary Server:**
```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Replication connections from standby server
# Replace 192.168.1.100/32 with your standby server's IP
host    replication     replicator      192.168.1.100/32        scram-sha-256
host    replication     replicator      192.168.1.101/32        scram-sha-256

# Allow regular connections
host    all             all             0.0.0.0/0               scram-sha-256
```

**Create Replication User on Primary:**
```sql
-- Connect to PostgreSQL as superuser
sudo -u postgres psql

-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'StrongPassword123!';

-- Verify user creation
\du replicator
```

**Restart Primary Server:**
```bash
sudo systemctl restart postgresql
```

#### Step 2: Configure Standby Server

**Stop PostgreSQL on Standby (if running):**
```bash
sudo systemctl stop postgresql
```

**Remove Existing Data Directory on Standby:**
```bash
# Backup existing data first (if needed)
sudo rm -rf /var/lib/postgresql/16/main/*
```

**Create Base Backup from Primary:**
```bash
# Run as postgres user on standby server
sudo -u postgres pg_basebackup \
    -h 192.168.1.10 \
    -D /var/lib/postgresql/16/main \
    -U replicator \
    -P \
    -v \
    -R \
    -X stream \
    -C -S standby_slot

# Flags explained:
# -h: Primary server hostname/IP
# -D: Data directory on standby
# -U: Replication user
# -P: Show progress
# -v: Verbose output
# -R: Create standby.signal and append connection settings to postgresql.auto.conf
# -X stream: Stream WAL while backing up
# -C: Create replication slot
# -S: Replication slot name
```

**Edit `postgresql.conf` on Standby (if needed):**
```conf
# These should be automatically configured by pg_basebackup with -R flag
# But verify or manually add if needed:
hot_standby = on
```

**Verify `standby.signal` File Exists:**
```bash
# This file indicates the server should start in standby mode
ls -l /var/lib/postgresql/16/main/standby.signal
```

**Check `postgresql.auto.conf` for Connection Settings:**
```bash
cat /var/lib/postgresql/16/main/postgresql.auto.conf
```

Should contain:
```conf
primary_conninfo = 'user=replicator password=StrongPassword123! host=192.168.1.10 port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
primary_slot_name = 'standby_slot'
```

**Start Standby Server:**
```bash
sudo systemctl start postgresql
```

#### Step 3: Verify Replication

**On Primary Server:**
```sql
-- Check replication status
SELECT * FROM pg_stat_replication;

-- Expected output should show:
-- - application_name
-- - client_addr (standby IP)
-- - state: 'streaming'
-- - sync_state: 'async' or 'sync'

-- Check replication slots
SELECT * FROM pg_replication_slots;

-- Check WAL sender processes
SELECT * FROM pg_stat_wal_receiver;
```

**On Standby Server:**
```sql
-- Check if in recovery mode (should return 't')
SELECT pg_is_in_recovery();

-- Check replication lag
SELECT 
    CASE 
        WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() 
        THEN 0 
        ELSE EXTRACT(EPOCH FROM now() - pg_last_xact_replay_timestamp()) 
    END AS replication_lag_seconds;

-- Check WAL receiver status
SELECT * FROM pg_stat_wal_receiver;
```

**Test Replication:**
```sql
-- On primary
CREATE TABLE replication_test (id SERIAL PRIMARY KEY, data TEXT);
INSERT INTO replication_test (data) VALUES ('Test data');

-- On standby (should see the table after a moment)
SELECT * FROM replication_test;
```

### Logical Replication

Logical replication allows selective replication of specific tables or databases. It's useful for:
- Replicating between different PostgreSQL versions
- Replicating specific tables instead of entire cluster
- Consolidating data from multiple databases
- Zero-downtime upgrades

#### Step 1: Configure Publisher (Source)

**Edit `postgresql.conf` on Publisher:**
```conf
# Logical Replication Settings
wal_level = logical                    # Must be 'logical' for logical replication
max_replication_slots = 10
max_wal_senders = 10
```

**Edit `pg_hba.conf` on Publisher:**
```conf
# Allow replication connections from subscriber
host    all             replicator      192.168.1.200/32        scram-sha-256
host    replication     replicator      192.168.1.200/32        scram-sha-256
```

**Restart PostgreSQL on Publisher:**
```bash
sudo systemctl restart postgresql
```

**Create Publication on Publisher:**
```sql
-- Connect to the database you want to replicate
psql -U postgres -d mydatabase

-- Create a table for testing
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create publication for specific tables
CREATE PUBLICATION my_publication FOR TABLE products;

-- Or create publication for all tables in database
CREATE PUBLICATION all_tables_publication FOR ALL TABLES;

-- View publications
SELECT * FROM pg_publication;

-- View tables in publication
SELECT * FROM pg_publication_tables WHERE pubname = 'my_publication';
```

#### Step 2: Configure Subscriber (Destination)

**Edit `postgresql.conf` on Subscriber:**
```conf
# Ensure these are set
max_replication_slots = 10
max_logical_replication_workers = 4
max_worker_processes = 8
```

**Restart PostgreSQL on Subscriber:**
```bash
sudo systemctl restart postgresql
```

**Create Subscription on Subscriber:**
```sql
-- Connect to the destination database
psql -U postgres -d mydatabase

-- Create the same table structure (must match publisher)
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create subscription
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=192.168.1.10 port=5432 user=replicator password=StrongPassword123! dbname=mydatabase'
    PUBLICATION my_publication;

-- View subscriptions
SELECT * FROM pg_subscription;

-- View subscription status
SELECT * FROM pg_stat_subscription;
```

#### Step 3: Verify Logical Replication

**On Publisher:**
```sql
-- Insert test data
INSERT INTO products (name, price) VALUES 
    ('Laptop', 999.99),
    ('Mouse', 29.99),
    ('Keyboard', 79.99);

-- View replication stats
SELECT * FROM pg_stat_replication;
```

**On Subscriber:**
```sql
-- Check if data was replicated
SELECT * FROM products;

-- Check replication status
SELECT 
    subname,
    pid,
    received_lsn,
    latest_end_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_time
FROM pg_stat_subscription;
```

**Test Real-time Replication:**
```sql
-- On publisher, update data
UPDATE products SET price = 899.99 WHERE name = 'Laptop';

-- On subscriber, verify update (should appear within seconds)
SELECT * FROM products WHERE name = 'Laptop';

-- On publisher, delete data
DELETE FROM products WHERE name = 'Mouse';

-- On subscriber, verify deletion
SELECT * FROM products WHERE name = 'Mouse';
```

### Synchronous vs. Asynchronous Replication

**Asynchronous Replication (Default):**
- Faster writes on primary
- Risk of data loss if primary fails before standby receives WAL
- Better for performance-critical applications

**Synchronous Replication:**

**Edit `postgresql.conf` on Primary:**
```conf
synchronous_commit = on
synchronous_standby_names = 'standby1,standby2'  # Names of synchronous standbys
```

Benefits:
- Zero data loss
- Guaranteed consistency
- Slower writes (waits for standby confirmation)

### Monitoring and Maintenance

**Check Replication Lag:**
```sql
-- On standby server
SELECT 
    CASE 
        WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() 
        THEN 0 
        ELSE pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) 
    END AS replication_lag_bytes,
    EXTRACT(EPOCH FROM now() - pg_last_xact_replay_timestamp()) AS replication_lag_seconds;
```

**Common Issues and Solutions:**

1. **Replication Slot Full:**
   ```sql
   -- Check slot status
   SELECT slot_name, active, restart_lsn FROM pg_replication_slots;
   
   -- Drop inactive slot if needed
   SELECT pg_drop_replication_slot('slot_name');
   ```

2. **Authentication Failed:**
   - Verify pg_hba.conf allows connections from standby IP
   - Check replication user password
   - Ensure scram-sha-256 authentication is properly configured

3. **WAL Files Accumulating:**
   - Increase wal_keep_size
   - Use replication slots to prevent WAL deletion
   - Set up WAL archiving

- **Further Reading**: [PostgreSQL Replication Documentation](https://www.postgresql.org/docs/current/high-availability.html)

## 15. PostgreSQL Binary Files and Utilities

PostgreSQL includes numerous utility programs and important configuration files in its installation. Understanding these files is crucial for database administration.

### Important Configuration Files

#### postgresql.conf
**Location:** 
- Linux: `/etc/postgresql/16/main/postgresql.conf` or `/var/lib/postgresql/data/postgresql.conf`
- Windows: `C:\Program Files\PostgreSQL\16\data\postgresql.conf`
- Mac: `/usr/local/var/postgresql@16/postgresql.conf`

**Purpose:** Main configuration file for PostgreSQL server. Controls all aspects of database behavior.

**Key Settings:**

```conf
# Connection Settings
listen_addresses = 'localhost'         # Addresses to listen on (* = all)
port = 5432                            # Port number
max_connections = 100                  # Maximum concurrent connections
superuser_reserved_connections = 3     # Reserved for superuser

# Memory Settings
shared_buffers = 128MB                 # Memory for caching data (25% of RAM recommended)
effective_cache_size = 4GB             # OS cache estimate (50-75% of RAM)
work_mem = 4MB                         # Memory per query operation
maintenance_work_mem = 64MB            # Memory for maintenance operations

# WAL Settings
wal_level = replica                    # minimal, replica, or logical
fsync = on                             # Force synchronous writes
synchronous_commit = on                # Wait for WAL write confirmation
wal_buffers = 16MB                     # WAL buffer size
checkpoint_timeout = 5min              # Time between checkpoints
max_wal_size = 1GB                     # Max WAL size before checkpoint

# Query Tuning
random_page_cost = 4.0                 # Disk random access cost (lower for SSD)
effective_io_concurrency = 200         # Concurrent I/O operations
default_statistics_target = 100        # Statistics detail level

# Logging
logging_collector = on                 # Enable log collection
log_directory = 'log'                  # Log directory
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'none'                 # none, ddl, mod, all
log_duration = off                     # Log query duration
log_line_prefix = '%m [%p] %u@%d '     # Log line format

# Locale and Formatting
datestyle = 'iso, mdy'
timezone = 'UTC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
```

**Reload Configuration Without Restart:**
```bash
# Method 1: Using pg_ctl
pg_ctl reload -D /var/lib/postgresql/data

# Method 2: Using SQL
psql -c "SELECT pg_reload_conf();"

# Method 3: Using systemctl (Linux)
sudo systemctl reload postgresql
```

**Note:** Some settings require a full restart (e.g., shared_buffers, max_connections).

#### pg_hba.conf
**Location:** Same directory as `postgresql.conf`

**Purpose:** Controls client authentication - who can connect from where and how.

**Format:** Each line specifies a connection type and authentication method.
```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections (Unix socket)
local   all             all                                     peer

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256

# IPv6 local connections
host    all             all             ::1/128                 scram-sha-256

# Remote connections
host    all             all             0.0.0.0/0               scram-sha-256
host    all             all             ::/0                    scram-sha-256

# Specific user from specific IP
host    mydatabase      myuser          192.168.1.100/32        scram-sha-256

# Replication connections
host    replication     replicator      192.168.1.0/24          scram-sha-256

# SSL connections only
hostssl all             all             0.0.0.0/0               scram-sha-256

# Reject specific user
host    all             baduser         0.0.0.0/0               reject
```

**Authentication Methods:**
- `trust`: Allow connection without password (dangerous for remote)
- `reject`: Reject connection
- `scram-sha-256`: Encrypted password (recommended)
- `md5`: MD5 encrypted password (legacy, less secure)
- `password`: Plain text password (not recommended)
- `peer`: Use OS username (local connections only)
- `ident`: Use ident protocol (local or remote)
- `cert`: SSL certificate authentication

**Reload After Changes:**
```bash
sudo systemctl reload postgresql
# or
psql -c "SELECT pg_reload_conf();"
```

#### pg_ident.conf
**Purpose:** Maps system usernames to PostgreSQL usernames for ident authentication.

```conf
# MAPNAME       SYSTEM-USERNAME         PG-USERNAME
mymap          john                     jsmith
mymap          admin                    postgres
```

### Critical Binary Utilities in /bin

PostgreSQL's `bin` directory contains essential command-line utilities. Located at:
- Linux: `/usr/lib/postgresql/16/bin/` or `/usr/bin/`
- Windows: `C:\Program Files\PostgreSQL\16\bin\`
- Mac: `/usr/local/opt/postgresql@16/bin/` or `/Applications/Postgres.app/Contents/Versions/latest/bin/`

#### pg_ctl
**Purpose:** Control PostgreSQL server (start, stop, restart, reload).

**Common Commands:**
```bash
# Start PostgreSQL server
pg_ctl start -D /var/lib/postgresql/data

# Stop PostgreSQL server (smart shutdown)
pg_ctl stop -D /var/lib/postgresql/data -m smart

# Restart PostgreSQL server
pg_ctl restart -D /var/lib/postgresql/data

# Reload configuration
pg_ctl reload -D /var/lib/postgresql/data

# Check server status
pg_ctl status -D /var/lib/postgresql/data

# Stop modes:
# -m smart: Wait for all clients to disconnect (default)
# -m fast: Disconnect clients and rollback transactions
# -m immediate: Force immediate shutdown (may require recovery)
```

#### psql
**Purpose:** Interactive PostgreSQL command-line client.

**Common Usage:**
```bash
# Connect to database
psql -U username -d database -h hostname -p 5432

# Connect to local database
psql mydatabase

# Execute SQL from file
psql -f script.sql

# Execute single command
psql -c "SELECT * FROM users;"

# Useful meta-commands inside psql:
# \l - List databases
# \c dbname - Connect to database
# \dt - List tables
# \d tablename - Describe table
# \du - List users
# \q - Quit
```

#### pg_dump
**Purpose:** Backup PostgreSQL databases.

**Common Usage:**
```bash
# Dump entire database
pg_dump -U postgres mydatabase > backup.sql

# Dump with custom format (compressed)
pg_dump -U postgres -Fc mydatabase > backup.dump

# Dump specific tables
pg_dump -U postgres -t table1 -t table2 mydatabase > tables.sql

# Dump database schema only
pg_dump -U postgres -s mydatabase > schema.sql

# Dump data only (no schema)
pg_dump -U postgres -a mydatabase > data.sql

# Dump with verbose output
pg_dump -U postgres -v mydatabase > backup.sql
```

#### pg_dumpall
**Purpose:** Backup all databases in a cluster (including roles and tablespaces).

**Common Usage:**
```bash
# Dump entire cluster
pg_dumpall -U postgres > cluster_backup.sql

# Dump only roles and tablespaces
pg_dumpall -U postgres -g > roles.sql
```

#### pg_restore
**Purpose:** Restore databases from pg_dump backups.

**Common Usage:**
```bash
# Restore from custom format
pg_restore -U postgres -d mydatabase backup.dump

# Restore with clean (drop objects first)
pg_restore -U postgres -d mydatabase -c backup.dump

# Restore specific tables
pg_restore -U postgres -d mydatabase -t table1 backup.dump

# List contents without restoring
pg_restore -l backup.dump
```

#### createdb / dropdb
**Purpose:** Create or drop databases.

**Common Usage:**
```bash
# Create database
createdb -U postgres mydatabase

# Create with specific owner
createdb -U postgres -O myuser mydatabase

# Drop database
dropdb -U postgres mydatabase
```

#### createuser / dropuser
**Purpose:** Create or drop PostgreSQL users (roles).

**Common Usage:**
```bash
# Create user
createuser -U postgres myuser

# Create user with password prompt
createuser -U postgres -P myuser

# Create superuser
createuser -U postgres -s admin

# Drop user
dropuser -U postgres myuser
```

#### pg_basebackup
**Purpose:** Create base backup for replication or backup purposes.

**Common Usage:**
```bash
# Create base backup
pg_basebackup -h primary_host -D /backup/standby_data -U replicator -P -v -R -X stream

# Flags:
# -D: Target directory
# -U: Replication user
# -P: Show progress
# -v: Verbose
# -R: Create recovery configuration
# -X stream: Stream WAL during backup
```

#### pg_isready
**Purpose:** Check PostgreSQL server connection status.

**Common Usage:**
```bash
# Check if server is ready
pg_isready

# Check specific host
pg_isready -h hostname -p 5432

# Use in scripts
if pg_isready; then
    echo "PostgreSQL is ready"
else
    echo "PostgreSQL is not ready"
fi
```

#### vacuumdb
**Purpose:** Vacuum and analyze databases.

**Common Usage:**
```bash
# Vacuum database
vacuumdb -U postgres mydatabase

# Vacuum and analyze
vacuumdb -U postgres -z mydatabase

# Vacuum all databases
vacuumdb -U postgres -a

# Full vacuum (rewrites entire table)
vacuumdb -U postgres -f mydatabase
```

#### pg_config
**Purpose:** Display PostgreSQL configuration information.

**Common Usage:**
```bash
# Show all configuration
pg_config

# Show specific information
pg_config --bindir
pg_config --configure
pg_config --version
```

#### initdb
**Purpose:** Initialize a new PostgreSQL database cluster.

**Common Usage:**
```bash
# Initialize database cluster
initdb -D /var/lib/postgresql/data

# Initialize with specific encoding and locale
initdb -D /var/lib/postgresql/data -E UTF8 --locale=en_US.UTF-8

# Initialize with specific superuser
initdb -D /var/lib/postgresql/data -U postgres
```

### Data Directory Structure

Understanding the data directory structure helps with troubleshooting and maintenance:

```
/var/lib/postgresql/data/
├── base/               # Database files
├── global/             # Cluster-wide tables
├── pg_wal/             # Write-Ahead Log files
├── pg_xact/            # Transaction commit status
├── pg_stat/            # Statistics files
├── pg_stat_tmp/        # Temporary statistics
├── pg_tblspc/          # Tablespace symbolic links
├── postgresql.conf     # Main configuration
├── pg_hba.conf         # Authentication configuration
├── pg_ident.conf       # Ident authentication map
├── postmaster.pid      # Server process ID (when running)
└── postmaster.opts     # Server command-line options
```

### Environment Variables

Useful PostgreSQL environment variables:

```bash
# Set default database
export PGDATABASE=mydatabase

# Set default user
export PGUSER=myuser

# Set default host
export PGHOST=localhost

# Set default port
export PGPORT=5432

# Set data directory
export PGDATA=/var/lib/postgresql/data

# Connection string
export DATABASE_URL="postgresql://user:pass@host:5432/dbname"
```

### Best Practices

1. **Always back up configuration files before editing**
   ```bash
   sudo cp postgresql.conf postgresql.conf.backup
   ```

2. **Use version control for configuration**
   ```bash
   git init /etc/postgresql/config
   ```

3. **Test configuration changes on non-production first**

4. **Monitor log files regularly**
   ```bash
   tail -f /var/log/postgresql/postgresql-16-main.log
   ```

5. **Keep PostgreSQL updated**
   ```bash
   sudo apt update && sudo apt upgrade postgresql
   ```

- **Further Reading**: [PostgreSQL Server Administration](https://www.postgresql.org/docs/current/admin.html)

## Conclusion
This guide provides fundamental knowledge about PostgreSQL, serving as a starting point for further exploration. For more advanced topics, refer to the official PostgreSQL documentation and community resources.