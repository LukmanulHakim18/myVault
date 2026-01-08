---
tags:
  - infrastructure
  - database
  - postgresql
  - patroni
  - high-availability
  - setup-guide
type: guide
created: '2026-01-08'
updated: '2026-01-08'
owner: platform-engineering
---
# Patroni High Availability PostgreSQL Setup Guide

**Purpose**: Panduan setup PostgreSQL HA menggunakan Patroni untuk menghilangkan database SPOF  
**Target RTO**: < 5 minutes (dari ~15-30 min manual failover)  
**Target RPO**: 0 (zero data loss dengan synchronous replication)

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Component Setup](#component-setup)
4. [Patroni Configuration](#patroni-configuration)
5. [PgBouncer Setup](#pgbouncer-setup)
6. [Application Integration](#application-integration)
7. [Operations Guide](#operations-guide)
8. [Monitoring](#monitoring)
9. [Disaster Recovery](#disaster-recovery)

---

## Architecture Overview

### Target Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PATRONI HA ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      APPLICATION LAYER                               │   │
│   │                                                                      │   │
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │   │
│   │   │ Order   │ │ User    │ │ Payment │ │ Session │ │ Other   │      │   │
│   │   │ Orch    │ │ Service │ │ Proc    │ │ Manager │ │ Services│      │   │
│   │   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘      │   │
│   │        │           │           │           │           │           │   │
│   └────────┼───────────┼───────────┼───────────┼───────────┼───────────┘   │
│            │           │           │           │           │               │
│            └───────────┴─────┬─────┴───────────┴───────────┘               │
│                              │                                              │
│   ┌──────────────────────────▼──────────────────────────────────────────┐  │
│   │                       PGBOUNCER CLUSTER                              │  │
│   │                                                                      │  │
│   │   ┌─────────────────────────┐  ┌─────────────────────────┐          │  │
│   │   │   PgBouncer Primary     │  │   PgBouncer Replica     │          │  │
│   │   │   (Write - :6432)       │  │   (Read - :6433)        │          │  │
│   │   │                         │  │                         │          │  │
│   │   │   → Routes to Leader    │  │   → Load balance reads  │          │  │
│   │   └────────────┬────────────┘  └────────────┬────────────┘          │  │
│   │                │                            │                        │  │
│   └────────────────┼────────────────────────────┼────────────────────────┘  │
│                    │                            │                           │
│   ┌────────────────┼────────────────────────────┼────────────────────────┐  │
│   │                │    PATRONI CLUSTER         │                        │  │
│   │                │                            │                        │  │
│   │   ┌────────────▼────────────┐  ┌───────────▼─────────────┐          │  │
│   │   │                         │  │                         │          │  │
│   │   │   PostgreSQL Primary    │◀─│   PostgreSQL Replica 1  │          │  │
│   │   │   (Leader)              │  │   (Sync Standby)        │          │  │
│   │   │                         │  │                         │          │  │
│   │   │   Patroni Agent ────────┼──│── Patroni Agent         │          │  │
│   │   │                         │  │                         │          │  │
│   │   └────────────┬────────────┘  └────────────┬────────────┘          │  │
│   │                │                            │                        │  │
│   │                │    Streaming               │                        │  │
│   │                │    Replication             │                        │  │
│   │                │         ┌──────────────────┘                        │  │
│   │                │         │                                           │  │
│   │   ┌────────────▼─────────▼──┐                                        │  │
│   │   │                         │                                        │  │
│   │   │   PostgreSQL Replica 2  │                                        │  │
│   │   │   (Async Standby)       │                                        │  │
│   │   │                         │                                        │  │
│   │   │   Patroni Agent         │                                        │  │
│   │   │                         │                                        │  │
│   │   └─────────────────────────┘                                        │  │
│   │                                                                      │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                         ETCD CLUSTER                                  │  │
│   │                                                                      │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │  │
│   │   │   etcd-1    │  │   etcd-2    │  │   etcd-3    │                 │  │
│   │   │  (Leader)   │◀─│  (Follower) │◀─│  (Follower) │                 │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘                 │  │
│   │                                                                      │  │
│   │   Functions:                                                         │  │
│   │   • Leader election coordination                                     │  │
│   │   • Cluster state storage                                            │  │
│   │   • Distributed locking                                              │  │
│   │                                                                      │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Node Layout

| Node | IP | Role | Components |
|------|-----|------|------------|
| pg-node-1 | 10.0.1.10 | Primary (initial) | PostgreSQL, Patroni, PgBouncer |
| pg-node-2 | 10.0.1.11 | Sync Replica | PostgreSQL, Patroni, PgBouncer |
| pg-node-3 | 10.0.1.12 | Async Replica | PostgreSQL, Patroni, PgBouncer |
| etcd-1 | 10.0.2.10 | etcd Leader | etcd |
| etcd-2 | 10.0.2.11 | etcd Follower | etcd |
| etcd-3 | 10.0.2.12 | etcd Follower | etcd |

---

## Prerequisites

### Hardware Requirements

| Component | CPU | Memory | Storage | Network |
|-----------|-----|--------|---------|---------|
| PostgreSQL Node | 8 cores | 32 GB | 500 GB SSD | 10 Gbps |
| etcd Node | 2 cores | 4 GB | 50 GB SSD | 1 Gbps |

### Software Requirements

```bash
# OS
Ubuntu 22.04 LTS

# Packages
PostgreSQL 15
Patroni 3.x
etcd 3.5.x
PgBouncer 1.21.x
Python 3.10+

# Network
- All nodes can communicate on required ports
- No firewall blocking between cluster nodes
```

### Port Requirements

| Port | Service | Description |
|------|---------|-------------|
| 5432 | PostgreSQL | Database connections |
| 6432 | PgBouncer | Connection pooling (write) |
| 6433 | PgBouncer | Connection pooling (read) |
| 8008 | Patroni REST API | Health checks, management |
| 2379 | etcd client | Client connections |
| 2380 | etcd peer | Peer communication |

---

## Component Setup

### Step 1: Setup etcd Cluster

**On each etcd node (etcd-1, etcd-2, etcd-3):**

```bash
# Install etcd
sudo apt update
sudo apt install -y etcd

# Create data directory
sudo mkdir -p /var/lib/etcd
sudo chown etcd:etcd /var/lib/etcd
```

**etcd-1 configuration (/etc/etcd/etcd.conf):**

```yaml
name: etcd-1
data-dir: /var/lib/etcd

# Client communication
listen-client-urls: http://0.0.0.0:2379
advertise-client-urls: http://10.0.2.10:2379

# Peer communication
listen-peer-urls: http://0.0.0.0:2380
initial-advertise-peer-urls: http://10.0.2.10:2380

# Cluster configuration
initial-cluster: etcd-1=http://10.0.2.10:2380,etcd-2=http://10.0.2.11:2380,etcd-3=http://10.0.2.12:2380
initial-cluster-state: new
initial-cluster-token: mrg-etcd-cluster

# Tuning
heartbeat-interval: 100
election-timeout: 1000
snapshot-count: 10000
```

**Repeat for etcd-2 and etcd-3** (change name and IPs accordingly)

```bash
# Start etcd on all nodes
sudo systemctl enable etcd
sudo systemctl start etcd

# Verify cluster health
etcdctl endpoint health --cluster
etcdctl member list
```

### Step 2: Install PostgreSQL and Patroni

**On each PostgreSQL node (pg-node-1, pg-node-2, pg-node-3):**

```bash
# Install PostgreSQL 15
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
sudo apt install -y postgresql-15 postgresql-contrib-15

# Stop default PostgreSQL (Patroni will manage it)
sudo systemctl stop postgresql
sudo systemctl disable postgresql

# Install Patroni
sudo apt install -y python3-pip python3-psycopg2
sudo pip3 install patroni[etcd3]

# Create Patroni directories
sudo mkdir -p /etc/patroni
sudo mkdir -p /var/lib/patroni
sudo chown postgres:postgres /var/lib/patroni
```

---

## Patroni Configuration

### Main Configuration (/etc/patroni/patroni.yml)

**pg-node-1:**

```yaml
scope: mrg-postgres-cluster
name: pg-node-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.10:8008
  authentication:
    username: patroni
    password: ${PATRONI_REST_PASSWORD}

etcd3:
  hosts:
    - 10.0.2.10:2379
    - 10.0.2.11:2379
    - 10.0.2.12:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB - won't failover if lag > 1MB
    
    # Synchronous replication
    synchronous_mode: true
    synchronous_mode_strict: false  # Allow async if no sync available
    synchronous_node_count: 1       # At least 1 sync replica
    
    postgresql:
      use_pg_rewind: true
      use_slots: true
      
      parameters:
        # Connection settings
        max_connections: 500
        superuser_reserved_connections: 5
        
        # Memory settings
        shared_buffers: 8GB
        effective_cache_size: 24GB
        maintenance_work_mem: 2GB
        work_mem: 64MB
        
        # WAL settings
        wal_level: replica
        wal_buffers: 64MB
        min_wal_size: 2GB
        max_wal_size: 8GB
        wal_keep_size: 4GB
        
        # Replication settings
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: 'on'
        hot_standby_feedback: 'on'
        
        # Synchronous replication
        synchronous_commit: 'on'
        synchronous_standby_names: 'ANY 1 (pg-node-2, pg-node-3)'
        
        # Checkpoints
        checkpoint_completion_target: 0.9
        checkpoint_timeout: 15min
        
        # Query tuning
        random_page_cost: 1.1
        effective_io_concurrency: 200
        default_statistics_target: 100
        
        # Parallel query
        max_worker_processes: 8
        max_parallel_workers_per_gather: 4
        max_parallel_workers: 8
        max_parallel_maintenance_workers: 4
        
        # Logging
        log_destination: 'stderr'
        logging_collector: 'on'
        log_directory: '/var/log/postgresql'
        log_filename: 'postgresql-%Y-%m-%d.log'
        log_rotation_age: '1d'
        log_rotation_size: '100MB'
        log_min_duration_statement: 1000  # Log queries > 1s
        log_checkpoints: 'on'
        log_connections: 'on'
        log_disconnections: 'on'
        log_lock_waits: 'on'

  initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:
    - host replication replicator 10.0.1.0/24 scram-sha-256
    - host all all 10.0.0.0/16 scram-sha-256
    - host all all 127.0.0.1/32 scram-sha-256

  users:
    admin:
      password: ${POSTGRES_ADMIN_PASSWORD}
      options:
        - createrole
        - createdb
    replicator:
      password: ${POSTGRES_REPLICATOR_PASSWORD}
      options:
        - replication
    app_user:
      password: ${POSTGRES_APP_PASSWORD}

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.10:5432
  data_dir: /var/lib/patroni/data
  bin_dir: /usr/lib/postgresql/15/bin
  config_dir: /var/lib/patroni/data
  
  pgpass: /var/lib/patroni/.pgpass
  
  authentication:
    replication:
      username: replicator
      password: ${POSTGRES_REPLICATOR_PASSWORD}
    superuser:
      username: postgres
      password: ${POSTGRES_PASSWORD}
    rewind:
      username: postgres
      password: ${POSTGRES_PASSWORD}

  callbacks:
    on_start: /etc/patroni/callbacks/on_start.sh
    on_stop: /etc/patroni/callbacks/on_stop.sh
    on_role_change: /etc/patroni/callbacks/on_role_change.sh

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

watchdog:
  mode: automatic
  device: /dev/watchdog
  safety_margin: 5
```

**pg-node-2 and pg-node-3:** Same config but change:
- `name: pg-node-2` / `name: pg-node-3`
- `connect_address: 10.0.1.11:8008` / `10.0.1.12:8008`
- `connect_address: 10.0.1.11:5432` / `10.0.1.12:5432`

### Callback Scripts

**/etc/patroni/callbacks/on_role_change.sh:**

```bash
#!/bin/bash
set -e

ACTION=$1
ROLE=$2
CLUSTER=$3

LOG_FILE="/var/log/patroni/callbacks.log"
PGBOUNCER_CONFIG="/etc/pgbouncer/pgbouncer.ini"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> $LOG_FILE
}

log "Role change callback: action=$ACTION, role=$ROLE, cluster=$CLUSTER"

case $ACTION in
    on_start)
        log "PostgreSQL started with role: $ROLE"
        ;;
    on_stop)
        log "PostgreSQL stopped"
        ;;
    on_role_change)
        if [ "$ROLE" == "master" ]; then
            log "Promoted to master"
            # Update PgBouncer to accept writes
            # Send alert
            curl -X POST "http://alertmanager:9093/api/v1/alerts" \
                -H "Content-Type: application/json" \
                -d "[{\"labels\":{\"alertname\":\"PatroniFailover\",\"severity\":\"warning\"},\"annotations\":{\"summary\":\"$(hostname) promoted to master\"}}]"
        else
            log "Demoted to replica"
        fi
        ;;
esac

exit 0
```

### Systemd Service

**/etc/systemd/system/patroni.service:**

```ini
[Unit]
Description=Patroni PostgreSQL HA Cluster Manager
After=syslog.target network.target etcd.service

[Service]
Type=simple
User=postgres
Group=postgres

# Environment
Environment=PATRONI_REST_PASSWORD=secure_password_here
Environment=POSTGRES_PASSWORD=secure_password_here
Environment=POSTGRES_ADMIN_PASSWORD=secure_password_here
Environment=POSTGRES_REPLICATOR_PASSWORD=secure_password_here
Environment=POSTGRES_APP_PASSWORD=secure_password_here

# Patroni executable
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml

# Restart policy
Restart=on-failure
RestartSec=10s

# Watchdog
WatchdogSec=120
NotifyAccess=all

# Resource limits
LimitNOFILE=65536
LimitNPROC=65536

[Install]
WantedBy=multi-user.target
```

### Start Patroni Cluster

```bash
# On pg-node-1 (will become initial leader)
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni

# Wait for bootstrap to complete
sleep 30

# On pg-node-2 and pg-node-3
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni

# Verify cluster
patronictl -c /etc/patroni/patroni.yml list
```

Expected output:
```
+ Cluster: mrg-postgres-cluster (7123456789012345678) ----+----+-----------+
| Member    | Host       | Role    | State   | TL | Lag in MB |
+-----------+------------+---------+---------+----+-----------+
| pg-node-1 | 10.0.1.10  | Leader  | running |  1 |           |
| pg-node-2 | 10.0.1.11  | Replica | running |  1 |         0 |
| pg-node-3 | 10.0.1.12  | Replica | running |  1 |         0 |
+-----------+------------+---------+---------+----+-----------+
```

---

## PgBouncer Setup

### Configuration (/etc/pgbouncer/pgbouncer.ini)

```ini
[databases]
; Write pool - always points to current leader
; HAProxy or Patroni REST API handles routing
mrg = host=127.0.0.1 port=5432 dbname=mrg auth_user=pgbouncer

; Specific database pools if needed
mrg_read = host=127.0.0.1 port=5432 dbname=mrg auth_user=pgbouncer

[pgbouncer]
; Listening
listen_addr = 0.0.0.0
listen_port = 6432

; Authentication
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1

; Pool settings
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
min_pool_size = 10
reserve_pool_size = 20
reserve_pool_timeout = 3

; Connection lifetime
server_lifetime = 3600
server_idle_timeout = 600
client_idle_timeout = 300

; Timeouts
server_connect_timeout = 3
server_login_retry = 3
query_timeout = 0
query_wait_timeout = 120

; Health check
server_check_query = SELECT 1
server_check_delay = 30

; TLS (optional)
; client_tls_sslmode = prefer
; server_tls_sslmode = prefer

; Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

; Admin
admin_users = pgbouncer_admin
stats_users = pgbouncer_stats

; Unix socket
unix_socket_dir = /var/run/pgbouncer
unix_socket_mode = 0777
```

### User List (/etc/pgbouncer/userlist.txt)

```
"pgbouncer" "SCRAM-SHA-256$4096:..."
"app_user" "SCRAM-SHA-256$4096:..."
"pgbouncer_admin" "SCRAM-SHA-256$4096:..."
```

Generate SCRAM-SHA-256 password:
```bash
psql -h localhost -U postgres -c "SELECT rolname, rolpassword FROM pg_authid WHERE rolname='app_user';"
```

### PgBouncer Systemd Service

```bash
sudo systemctl enable pgbouncer
sudo systemctl start pgbouncer

# Verify
psql -h localhost -p 6432 -U app_user -d mrg -c "SELECT 1"
```

---

## Application Integration

### Connection String Update

**Before (Direct to PostgreSQL):**
```
postgresql://app_user:password@pg-master:5432/mrg
```

**After (Via PgBouncer with HA):**
```go
// config/database.go
type DBConfig struct {
    // Write connection (via PgBouncer to current leader)
    WriteHost string `env:"DB_WRITE_HOST" default:"pgbouncer.mrg.internal"`
    WritePort int    `env:"DB_WRITE_PORT" default:"6432"`
    
    // Read connection (via PgBouncer to replicas)
    ReadHost  string `env:"DB_READ_HOST" default:"pgbouncer.mrg.internal"`
    ReadPort  int    `env:"DB_READ_PORT" default:"6433"`
    
    Database  string `env:"DB_NAME" default:"mrg"`
    User      string `env:"DB_USER" default:"app_user"`
    Password  string `env:"DB_PASSWORD"`
    
    // Pool settings
    MaxOpenConns    int           `env:"DB_MAX_OPEN_CONNS" default:"50"`
    MaxIdleConns    int           `env:"DB_MAX_IDLE_CONNS" default:"10"`
    ConnMaxLifetime time.Duration `env:"DB_CONN_MAX_LIFETIME" default:"1h"`
    
    // Timeouts
    ConnectTimeout time.Duration `env:"DB_CONNECT_TIMEOUT" default:"5s"`
    QueryTimeout   time.Duration `env:"DB_QUERY_TIMEOUT" default:"30s"`
}

func NewDBCluster(cfg DBConfig) (*DBCluster, error) {
    // Write connection
    writeDSN := fmt.Sprintf(
        "host=%s port=%d dbname=%s user=%s password=%s sslmode=require connect_timeout=%d",
        cfg.WriteHost, cfg.WritePort, cfg.Database, cfg.User, cfg.Password,
        int(cfg.ConnectTimeout.Seconds()),
    )
    
    writeDB, err := sql.Open("postgres", writeDSN)
    if err != nil {
        return nil, fmt.Errorf("failed to open write DB: %w", err)
    }
    
    writeDB.SetMaxOpenConns(cfg.MaxOpenConns)
    writeDB.SetMaxIdleConns(cfg.MaxIdleConns)
    writeDB.SetConnMaxLifetime(cfg.ConnMaxLifetime)
    
    // Read connection (can have more connections)
    readDSN := fmt.Sprintf(
        "host=%s port=%d dbname=%s user=%s password=%s sslmode=require connect_timeout=%d",
        cfg.ReadHost, cfg.ReadPort, cfg.Database, cfg.User, cfg.Password,
        int(cfg.ConnectTimeout.Seconds()),
    )
    
    readDB, err := sql.Open("postgres", readDSN)
    if err != nil {
        return nil, fmt.Errorf("failed to open read DB: %w", err)
    }
    
    readDB.SetMaxOpenConns(cfg.MaxOpenConns * 2)
    readDB.SetMaxIdleConns(cfg.MaxIdleConns * 2)
    readDB.SetConnMaxLifetime(cfg.ConnMaxLifetime)
    
    // Verify connections
    if err := writeDB.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping write DB: %w", err)
    }
    if err := readDB.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping read DB: %w", err)
    }
    
    return &DBCluster{
        Write: writeDB,
        Read:  readDB,
    }, nil
}
```

### Repository Pattern with Read/Write Split

```go
// internal/repository/order_repository.go
type OrderRepository struct {
    db *DBCluster
}

// Write operations use Write connection
func (r *OrderRepository) Create(ctx context.Context, order *Order) error {
    query := `INSERT INTO orders (id, user_id, status, ...) VALUES ($1, $2, $3, ...)`
    _, err := r.db.Write.ExecContext(ctx, query, order.ID, order.UserID, order.Status)
    return err
}

func (r *OrderRepository) UpdateStatus(ctx context.Context, orderID string, status string) error {
    query := `UPDATE orders SET status = $1, updated_at = NOW() WHERE id = $2`
    _, err := r.db.Write.ExecContext(ctx, query, status, orderID)
    return err
}

// Read operations use Read connection
func (r *OrderRepository) GetByID(ctx context.Context, orderID string) (*Order, error) {
    query := `SELECT id, user_id, status, ... FROM orders WHERE id = $1`
    row := r.db.Read.QueryRowContext(ctx, query, orderID)
    
    var order Order
    err := row.Scan(&order.ID, &order.UserID, &order.Status)
    if err != nil {
        return nil, err
    }
    return &order, nil
}

func (r *OrderRepository) ListByUser(ctx context.Context, userID string, limit int) ([]*Order, error) {
    query := `SELECT id, user_id, status, ... FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT $2`
    rows, err := r.db.Read.QueryContext(ctx, query, userID, limit)
    // ...
}

// For operations requiring read-your-writes consistency, use Write connection
func (r *OrderRepository) GetByIDConsistent(ctx context.Context, orderID string) (*Order, error) {
    query := `SELECT id, user_id, status, ... FROM orders WHERE id = $1`
    row := r.db.Write.QueryRowContext(ctx, query, orderID) // Use Write for consistency
    // ...
}
```

---

## Operations Guide

### Common Operations

#### Check Cluster Status
```bash
patronictl -c /etc/patroni/patroni.yml list
```

#### Manual Switchover (Planned)
```bash
# Graceful switchover to specific node
patronictl -c /etc/patroni/patroni.yml switchover mrg-postgres-cluster \
    --master pg-node-1 \
    --candidate pg-node-2 \
    --force

# Let Patroni choose best candidate
patronictl -c /etc/patroni/patroni.yml switchover mrg-postgres-cluster --force
```

#### Manual Failover (Emergency)
```bash
# Force failover when leader is unresponsive
patronictl -c /etc/patroni/patroni.yml failover mrg-postgres-cluster \
    --candidate pg-node-2 \
    --force
```

#### Reinitialize Failed Node
```bash
# When a replica needs to be rebuilt from scratch
patronictl -c /etc/patroni/patroni.yml reinit mrg-postgres-cluster pg-node-3
```

#### Edit Dynamic Configuration
```bash
# View current config
patronictl -c /etc/patroni/patroni.yml show-config

# Edit config
patronictl -c /etc/patroni/patroni.yml edit-config

# Changes like max_connections require restart
patronictl -c /etc/patroni/patroni.yml restart mrg-postgres-cluster
```

#### Maintenance Mode
```bash
# Pause automatic failover (for maintenance)
patronictl -c /etc/patroni/patroni.yml pause

# Resume automatic failover
patronictl -c /etc/patroni/patroni.yml resume
```

### Backup and Restore

#### WAL-G Backup Configuration

```yaml
# /etc/wal-g/wal-g.yaml
WALG_S3_PREFIX: s3://mrg-db-backups/patroni
AWS_ACCESS_KEY_ID: xxx
AWS_SECRET_ACCESS_KEY: xxx
AWS_REGION: ap-southeast-1
PGHOST: /var/run/postgresql
PGUSER: postgres
PGDATABASE: mrg
```

#### Backup Script

```bash
#!/bin/bash
# /etc/patroni/scripts/backup.sh

# Only run on leader
if patronictl -c /etc/patroni/patroni.yml list | grep "$(hostname)" | grep -q "Leader"; then
    echo "Running backup on leader..."
    wal-g backup-push /var/lib/patroni/data
else
    echo "Not leader, skipping backup"
fi
```

#### Cron Schedule

```bash
# /etc/cron.d/patroni-backup
0 2 * * * postgres /etc/patroni/scripts/backup.sh >> /var/log/patroni/backup.log 2>&1
```

---

## Monitoring

### Prometheus Metrics

**Patroni Exporter:**
```yaml
# prometheus/patroni-targets.yml
- targets:
  - pg-node-1:8008
  - pg-node-2:8008
  - pg-node-3:8008
  labels:
    job: patroni
```

### Key Metrics to Monitor

```promql
# Cluster has leader
patroni_cluster_has_leader == 1

# Replication lag (seconds)
patroni_replication_lag_seconds

# Member state (1=running, 0=stopped)
patroni_member_state

# Timeline (should be same across cluster)
patroni_timeline

# Pending restart (config changed, needs restart)
patroni_pending_restart == 1

# PgBouncer metrics
pgbouncer_pools_server_active
pgbouncer_pools_client_waiting
pgbouncer_stats_queries_total
```

### Alert Rules

```yaml
groups:
  - name: patroni_alerts
    rules:
      - alert: PatroniNoLeader
        expr: sum(patroni_cluster_has_leader) == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Patroni cluster has no leader"
          
      - alert: PatroniReplicationLag
        expr: patroni_replication_lag_seconds > 5
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag > 5 seconds on {{ $labels.instance }}"
          
      - alert: PatroniMemberDown
        expr: patroni_member_state == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Patroni member {{ $labels.instance }} is down"
          
      - alert: PatroniPendingRestart
        expr: patroni_pending_restart == 1
        for: 1h
        labels:
          severity: info
        annotations:
          summary: "Patroni member {{ $labels.instance }} needs restart"
```

### Grafana Dashboard

Import dashboard ID: **18870** (Patroni) or create custom:

```json
{
  "panels": [
    {
      "title": "Cluster Status",
      "type": "stat",
      "targets": [{"expr": "sum(patroni_cluster_has_leader)"}],
      "fieldConfig": {
        "mappings": [
          {"value": 1, "text": "HEALTHY", "color": "green"},
          {"value": 0, "text": "NO LEADER", "color": "red"}
        ]
      }
    },
    {
      "title": "Replication Lag",
      "type": "graph",
      "targets": [
        {"expr": "patroni_replication_lag_seconds", "legendFormat": "{{instance}}"}
      ]
    },
    {
      "title": "Member States",
      "type": "table",
      "targets": [
        {"expr": "patroni_member_state", "format": "table"}
      ]
    }
  ]
}
```

---

## Disaster Recovery

### Scenario 1: Single Node Failure

**Automatic recovery by Patroni:**
1. Leader fails → Patroni promotes sync replica within 30s
2. Old leader recovers → Rejoins as replica automatically

**Manual verification:**
```bash
patronictl -c /etc/patroni/patroni.yml list
# Verify new leader and old node as replica
```

### Scenario 2: Lose Quorum (2 of 3 nodes down)

**Recovery steps:**
```bash
# 1. Check remaining node
patronictl -c /etc/patroni/patroni.yml list

# 2. If etcd quorum lost, check etcd
etcdctl endpoint health --cluster

# 3. Recover etcd first if needed
# (Follow etcd disaster recovery)

# 4. Then recover Patroni nodes
sudo systemctl start patroni
```

### Scenario 3: Complete Cluster Loss

**Recovery from backup:**
```bash
# 1. On new node, restore from backup
wal-g backup-fetch /var/lib/patroni/data LATEST

# 2. Create recovery.signal
touch /var/lib/patroni/data/recovery.signal

# 3. Configure Patroni and start
sudo systemctl start patroni

# 4. Bootstrap new cluster from this node
patronictl -c /etc/patroni/patroni.yml reinit mrg-postgres-cluster --force
```

---

## 📚 Related Documents

- [[MRG SPOF Assessment & Mitigation Strategy]]
- [[RUNBOOK-Database-Failover]]
- [[PgBouncer Operations Guide]]
- [Patroni Documentation](https://patroni.readthedocs.io/)

---

**Last Updated**: 2026-01-08  
**Owner**: Platform Engineering Team  
**Review Cycle**: Quarterly
