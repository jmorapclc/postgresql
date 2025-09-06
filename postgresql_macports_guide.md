# PostgreSQL Installation Guide via MacPorts for Apple Silicon Macs

## Prerequisites

- macOS on Apple Silicon (M1/M2/M3)
- MacPorts installed and updated
- Administrative access (sudo)

## Installation Instructions

### Step 1: Update MacPorts

```bash
sudo port selfupdate
sudo port upgrade outdated
```

### Step 2: Choose Your PostgreSQL Version

Replace `XX` with your desired version (e.g., 15, 16, 17):

```bash
# Check available PostgreSQL versions
port search postgresql | grep "^postgresql[0-9]"

# Set your version (example: 16)
PG_VERSION=16
```

### Step 3: Install PostgreSQL

```bash
# Install PostgreSQL without problematic variants
sudo port install postgresql${PG_VERSION} -perl -python -universal

# Install the server component
sudo port install postgresql${PG_VERSION}-server

# Set as default PostgreSQL
sudo port select --set postgresql postgresql${PG_VERSION}
```

### Step 4: Initialize the Database Cluster

```bash
# Create data directory
sudo mkdir -p /opt/local/var/db/postgresql${PG_VERSION}/defaultdb
sudo chown postgres:postgres /opt/local/var/db/postgresql${PG_VERSION}
sudo chown postgres:postgres /opt/local/var/db/postgresql${PG_VERSION}/defaultdb

# Initialize database
sudo -u postgres /opt/local/lib/postgresql${PG_VERSION}/bin/initdb \
    -D /opt/local/var/db/postgresql${PG_VERSION}/defaultdb \
    -E UTF8 \
    --locale=en_US.UTF-8 \
    --auth-local=trust \
    --auth-host=md5
```

### Step 5: Configure PostgreSQL for Apple Silicon Performance

Create optimized configuration based on your RAM:

```bash
# Add performance optimizations (adjust values based on your RAM)
# For 96GB RAM M2 Max example:
cat << 'EOF' | sudo -u postgres tee -a /opt/local/var/db/postgresql${PG_VERSION}/defaultdb/postgresql.conf > /dev/null

# === Apple Silicon Optimizations ===
# Memory Settings (adjust for your RAM)
# For 96GB: use values below
# For 64GB: shared_buffers = 16GB, effective_cache_size = 48GB
# For 32GB: shared_buffers = 8GB, effective_cache_size = 24GB
# For 16GB: shared_buffers = 4GB, effective_cache_size = 12GB

shared_buffers = 24GB              # 25% of RAM
effective_cache_size = 72GB        # 75% of RAM
maintenance_work_mem = 4GB
work_mem = 256MB
wal_buffers = 512MB
huge_pages = off

# CPU Settings (M2 Max example - adjust for your chip)
max_parallel_workers_per_gather = 8
max_parallel_workers = 16
max_parallel_maintenance_workers = 8
max_worker_processes = 16

# SSD Optimizations
random_page_cost = 1.1
effective_io_concurrency = 0       # Must be 0 on macOS
checkpoint_completion_target = 0.9
wal_compression = on
jit = on

# Connection Settings
max_connections = 200
listen_addresses = 'localhost'
port = 5432

# Logging
logging_collector = on
log_directory = '/opt/local/var/log/postgresql${PG_VERSION}'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_line_prefix = '%t [%p]: '
EOF
```

### Step 6: Configure Authentication

```bash
# Setup authentication (adjust usernames as needed)
cat << EOF | sudo tee /opt/local/var/db/postgresql${PG_VERSION}/defaultdb/pg_hba.conf > /dev/null
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                trust
local   all             $(whoami)                               trust
local   all             all                                     md5
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
EOF

# Fix permissions
sudo chown postgres:postgres /opt/local/var/db/postgresql${PG_VERSION}/defaultdb/pg_hba.conf
sudo chmod 600 /opt/local/var/db/postgresql${PG_VERSION}/defaultdb/pg_hba.conf
```

### Step 7: Create Log Directory and Start PostgreSQL

```bash
# Create log directory
sudo mkdir -p /opt/local/var/log/postgresql${PG_VERSION}
sudo chown postgres:postgres /opt/local/var/log/postgresql${PG_VERSION}

# Start PostgreSQL using MacPorts service
sudo port load postgresql${PG_VERSION}-server

# Wait for startup
sleep 3

# Verify it's running
/opt/local/lib/postgresql${PG_VERSION}/bin/pg_isready
```

### Step 8: Create Your User and Database

```bash
# Create superuser for your account
sudo -u postgres /opt/local/lib/postgresql${PG_VERSION}/bin/createuser --superuser $(whoami)

# Create your personal database
/opt/local/lib/postgresql${PG_VERSION}/bin/createdb $(whoami)

# Test connection
/opt/local/lib/postgresql${PG_VERSION}/bin/psql -c "SELECT version();"
```

### Step 9: Install Extensions (Optional)

```bash
# Install PostGIS (if needed)
sudo port install pg${PG_VERSION}-postgis3

# Install other tools
sudo port install pgbadger

# Enable extensions in your database
/opt/local/lib/postgresql${PG_VERSION}/bin/psql -d $(whoami) << 'SQL'
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
-- If PostGIS was installed:
-- CREATE EXTENSION IF NOT EXISTS postgis;
SQL
```

### Step 10: Setup Shell Environment

Add to your `~/.zshrc` or `~/.bash_profile`:

```bash
cat >> ~/.zshrc << 'EOF'

# PostgreSQL Configuration (adjust version number)
export PATH="/opt/local/lib/postgresqlXX/bin:$PATH"
export PGUSER="$(whoami)"
export PGDATABASE="$(whoami)"

# PostgreSQL Aliases (adjust version number in paths)
alias pg='psql'
alias pgstart='sudo port load postgresqlXX-server'
alias pgstop='sudo port unload postgresqlXX-server'
alias pgstatus='pg_isready && echo "✓ PostgreSQL running" || echo "✗ PostgreSQL stopped"'
alias pglog='sudo tail -f /opt/local/var/log/postgresqlXX/*.log'
alias pgbackup='pg_dumpall > ~/PostgreSQL_Backups/backup_$(date +%Y%m%d_%H%M).sql'
alias pgsize='psql -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database ORDER BY 2 DESC;"'
EOF

# Remember to replace XX with your version number, then:
source ~/.zshrc
```

## Troubleshooting

### If PostgreSQL Won't Start

1. Check the logs:
```bash
sudo tail -50 /opt/local/var/log/postgresql${PG_VERSION}/startup.log
```

2. Common issues and fixes:

**Permission Errors:**
```bash
# Run commands from /tmp to avoid permission issues
cd /tmp
sudo -u postgres /opt/local/lib/postgresql${PG_VERSION}/bin/pg_ctl \
    -D /opt/local/var/db/postgresql${PG_VERSION}/defaultdb \
    -l /opt/local/var/log/postgresql${PG_VERSION}/startup.log \
    start
cd ~
```

**Configuration Errors:**
- On macOS, `effective_io_concurrency` must be 0
- Ensure shared_buffers doesn't exceed available memory

### Managing Multiple PostgreSQL Versions

To run multiple versions simultaneously:

1. Use different ports for each version:
```bash
# In postgresql.conf for version 15:
port = 5432

# In postgresql.conf for version 16:
port = 5433
```

2. Connect to specific versions:
```bash
psql -p 5432  # Connect to version 15
psql -p 5433  # Connect to version 16
```

3. Switch default version:
```bash
sudo port select --set postgresql postgresql15
# or
sudo port select --set postgresql postgresql16
```

## Backup Strategy

Create a backup system:

```bash
# Create backup directory
mkdir -p ~/PostgreSQL_Backups

# Create backup script
cat > ~/PostgreSQL_Backups/backup.sh << 'SCRIPT'
#!/bin/bash
BACKUP_DIR="$HOME/PostgreSQL_Backups"
DATE=$(date +%Y%m%d_%H%M)
pg_dumpall | gzip > "$BACKUP_DIR/backup_$DATE.sql.gz"
find "$BACKUP_DIR" -name "backup_*.sql.gz" -mtime +7 -delete
echo "Backup completed: $BACKUP_DIR/backup_$DATE.sql.gz"
SCRIPT

chmod +x ~/PostgreSQL_Backups/backup.sh
```

## Uninstalling a Version

To completely remove a PostgreSQL version:

```bash
# Stop the server
sudo port unload postgresql${PG_VERSION}-server

# Backup your data first!
pg_dumpall > ~/PostgreSQL_Backups/final_backup_pg${PG_VERSION}.sql

# Uninstall
sudo port uninstall postgresql${PG_VERSION}-server postgresql${PG_VERSION}

# Remove data directory (after backing up!)
sudo rm -rf /opt/local/var/db/postgresql${PG_VERSION}
sudo rm -rf /opt/local/var/log/postgresql${PG_VERSION}
```

## Performance Tuning Guidelines

### Memory Allocation by System RAM:

| System RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem |
|------------|---------------|---------------------|----------|---------------------|
| 8GB        | 2GB           | 6GB                 | 64MB     | 512MB              |
| 16GB       | 4GB           | 12GB                | 128MB    | 1GB                |
| 32GB       | 8GB           | 24GB                | 256MB    | 2GB                |
| 64GB       | 16GB          | 48GB                | 256MB    | 4GB                |
| 96GB       | 24GB          | 72GB                | 256MB    | 4GB                |
| 128GB      | 32GB          | 96GB                | 512MB    | 8GB                |

### CPU Settings by Apple Silicon Chip:

| Chip          | max_parallel_workers | max_parallel_workers_per_gather |
|---------------|---------------------|----------------------------------|
| M1/M2         | 8                   | 4                                |
| M1/M2 Pro     | 12                  | 6                                |
| M1/M2 Max     | 16                  | 8                                |
| M1/M2 Ultra   | 24                  | 12                               |

## Verification Commands

After installation, verify your setup:

```bash
# Check version
psql --version

# Check status
pg_isready

# List databases
psql -l

# Check configuration
psql -c "SHOW shared_buffers;"
psql -c "SHOW max_connections;"

# Check extensions
psql -c "\dx"
```

## Notes

- This guide is optimized for Apple Silicon Macs (M1/M2/M3)
- Always backup before major changes
- Adjust memory settings based on your system RAM
- Multiple PostgreSQL versions can coexist using different ports
- The `-universal` flag should be avoided on Apple Silicon to prevent build issues