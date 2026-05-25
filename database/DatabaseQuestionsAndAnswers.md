# Database Interview Questions and Answers for DevOps

A focused collection of database interview questions for DevOps engineers — covering MySQL, MongoDB, PostgreSQL, DynamoDB, and operational topics like backup, replication, scaling, and troubleshooting.

---

## Table of Contents

- [Database Fundamentals](#database-fundamentals)
- [MySQL](#mysql)
- [MongoDB](#mongodb)
- [PostgreSQL](#postgresql)
- [DynamoDB](#dynamodb)
- [Database Operations for DevOps](#database-operations-for-devops)

---

## Database Fundamentals

### 1. What is the difference between SQL and NoSQL databases?

**Answer:**

| Feature | SQL (Relational) | NoSQL (Non-Relational) |
|---------|-----------------|----------------------|
| **Schema** | Fixed schema (tables, rows, columns) | Flexible/schema-less (documents, key-value, graph) |
| **Data model** | Structured, normalized | Semi-structured or unstructured |
| **Query language** | SQL | Varies (MongoDB query, DynamoDB API, etc.) |
| **Scaling** | Primarily vertical (scale up) | Horizontal (scale out / sharding) |
| **Transactions** | Strong ACID guarantees | Varies (eventual consistency common) |
| **Joins** | Native JOIN support | Typically no joins (embed or reference) |
| **Best for** | Structured data, complex queries, transactions | High volume, flexible schema, distributed systems |

| Use Case | Best Choice |
|----------|------------|
| Financial transactions | MySQL / PostgreSQL |
| User profiles, catalogs | MongoDB |
| Session storage, caching | Redis / DynamoDB |
| Real-time analytics | ClickHouse / DynamoDB Streams |
| Relationships/graph | Neo4j / PostgreSQL |

---

### 2. What are ACID properties?

**Answer:**

| Property | Meaning | Example |
|----------|---------|---------|
| **Atomicity** | All or nothing — transaction fully completes or fully rolls back | Transfer $100: debit + credit both succeed or both fail |
| **Consistency** | Database moves from one valid state to another | Foreign key constraints enforced |
| **Isolation** | Concurrent transactions don't interfere | Two users updating same row don't see partial data |
| **Durability** | Committed data survives crashes | Data written to disk/WAL before confirming |

**MySQL** — Full ACID with InnoDB engine
**PostgreSQL** — Full ACID with MVCC
**MongoDB** — ACID for single-document (always); multi-document transactions since v4.0
**DynamoDB** — ACID via DynamoDB Transactions (across up to 100 items)

---

### 3. What is the CAP theorem and how does it apply to databases?

**Answer:** The CAP theorem states a distributed system can guarantee only **two of three** properties:

| Property | Meaning |
|----------|---------|
| **Consistency** | Every read returns the most recent write |
| **Availability** | Every request gets a response (not necessarily the latest) |
| **Partition tolerance** | System works despite network partitions |

Since network partitions are inevitable in distributed systems, the real choice is **CP vs AP**:

| Database | CAP choice | Behavior during partition |
|----------|-----------|-------------------------|
| **MySQL** (single node) | CA (no partitions) | N/A — single node |
| **PostgreSQL** (streaming replication) | CP | Replica may lag, primary serves reads |
| **MongoDB** (replica set) | CP | Primary handles writes, secondaries catch up |
| **DynamoDB** | AP (eventual) or CP (strong) | Configurable per read |
| **Cassandra** | AP | Eventual consistency, tunable |

---

## MySQL

### 4. What are MySQL storage engines and when do you use each?

**Answer:**

| Engine | Key Features | Use Case |
|--------|-------------|----------|
| **InnoDB** (default) | ACID, row-level locking, foreign keys, crash recovery | General purpose, transactions |
| **MyISAM** | Table-level locking, full-text search, no transactions | Read-heavy, legacy apps |
| **MEMORY** | Data in RAM, very fast, lost on restart | Temp tables, caching |
| **Archive** | Compressed, insert/select only, no indexes | Log archival |

```sql
-- Check engine of a table
SHOW TABLE STATUS LIKE 'users'\G

-- Create table with specific engine
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    total DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB;

-- Convert engine
ALTER TABLE old_table ENGINE=InnoDB;
```

---

### 5. How does MySQL replication work?

**Answer:**

**Types of replication:**

| Type | How it works | Use case |
|------|-------------|----------|
| **Async (default)** | Primary writes binlog → replica reads asynchronously | Read scaling, backups |
| **Semi-sync** | Primary waits for at least one replica ACK | Better durability |
| **Group Replication** | Multi-primary or single-primary consensus | HA, auto-failover |

```sql
-- Check replication status (on replica)
SHOW REPLICA STATUS\G

-- Key fields to monitor:
-- Seconds_Behind_Source: 0 (caught up) or lag in seconds
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes
-- Last_Error: (should be empty)
```

```bash
# Setup async replication
# On primary:
# my.cnf:
# server-id = 1
# log_bin = mysql-bin
# binlog_format = ROW

# On replica:
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='primary-host',
  SOURCE_USER='repl_user',
  SOURCE_PASSWORD='password',
  SOURCE_LOG_FILE='mysql-bin.000001',
  SOURCE_LOG_POS=4;
START REPLICA;
```

---

### 6. How do you optimize MySQL query performance?

**Answer:**

```sql
-- 1. Use EXPLAIN to analyze queries
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'active'\G

-- Key EXPLAIN fields:
-- type: ALL (bad) → index → range → ref → eq_ref → const (best)
-- rows: Estimated rows scanned
-- Extra: "Using index" (good), "Using filesort" (bad), "Using temporary" (bad)

-- 2. Create proper indexes
CREATE INDEX idx_user_status ON orders (user_id, status);  -- Composite index

-- 3. Avoid SELECT *
SELECT id, total, created_at FROM orders WHERE user_id = 100;

-- 4. Use LIMIT for large result sets
SELECT * FROM logs ORDER BY created_at DESC LIMIT 100;

-- 5. Optimize JOINs — index join columns
SELECT o.id, u.name
FROM orders o
JOIN users u ON o.user_id = u.id  -- Ensure user_id and u.id are indexed
WHERE o.status = 'active';

-- 6. Check slow queries
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- Log queries > 1 second
SHOW VARIABLES LIKE 'slow_query_log_file';
```

**Index best practices:**
- Index columns used in `WHERE`, `JOIN`, `ORDER BY`
- Composite indexes: left-most prefix rule (`(a, b, c)` covers `a`, `a,b`, `a,b,c`)
- Avoid over-indexing — each index slows `INSERT`/`UPDATE`
- Use `EXPLAIN` before and after adding indexes

---

### 7. What are MySQL transactions and isolation levels?

**Answer:**

```sql
-- Transaction
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- or ROLLBACK on error
```

**Isolation levels (from weakest to strongest):**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|-----------|-------------------|-------------|-------------|
| **READ UNCOMMITTED** | Yes | Yes | Yes | Fastest |
| **READ COMMITTED** | No | Yes | Yes | Fast |
| **REPEATABLE READ** (MySQL default) | No | No | Yes (InnoDB prevents) | Good |
| **SERIALIZABLE** | No | No | No | Slowest |

```sql
-- Check current isolation level
SELECT @@transaction_isolation;

-- Set isolation level
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

### 8. How do you perform backup and recovery in MySQL?

**Answer:**

```bash
# Logical backup (mysqldump)
mysqldump -u root -p --all-databases --single-transaction \
  --routines --triggers > full_backup.sql

# Backup specific database
mysqldump -u root -p mydb --single-transaction > mydb_backup.sql

# Restore
mysql -u root -p mydb < mydb_backup.sql

# Physical backup (Percona XtraBackup — for large DBs)
xtrabackup --backup --target-dir=/backup/full
xtrabackup --prepare --target-dir=/backup/full
xtrabackup --copy-back --target-dir=/backup/full

# Point-in-time recovery using binlog
mysqlbinlog --start-datetime="2024-01-15 10:00:00" \
  --stop-datetime="2024-01-15 11:00:00" mysql-bin.000003 | mysql -u root -p
```

| Method | Speed | Lock | Use Case |
|--------|-------|------|----------|
| **mysqldump** | Slow (large DBs) | `--single-transaction` (no lock for InnoDB) | Small-medium DBs, portability |
| **XtraBackup** | Fast (physical copy) | No lock | Large production DBs |
| **MySQL Enterprise Backup** | Fast | Minimal | Enterprise customers |
| **Binlog** | Incremental | N/A | Point-in-time recovery |

---

### 9. How do you troubleshoot MySQL performance issues?

**Answer:**

```sql
-- Check active connections and queries
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;  -- Full query text

-- Kill long-running query
KILL <process_id>;

-- Check InnoDB status (deadlocks, buffer pool, I/O)
SHOW ENGINE INNODB STATUS\G

-- Check table sizes
SELECT table_name, 
       ROUND(data_length / 1024 / 1024, 2) AS data_mb,
       ROUND(index_length / 1024 / 1024, 2) AS index_mb,
       table_rows
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY data_length DESC;

-- Check for missing indexes
SELECT * FROM sys.statements_with_full_table_scans
ORDER BY no_index_used_count DESC LIMIT 10;

-- Buffer pool hit ratio (should be > 99%)
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
-- Hit ratio = 1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)

-- Connection stats
SHOW GLOBAL STATUS LIKE 'Threads_%';
SHOW VARIABLES LIKE 'max_connections';
```

---

## MongoDB

### 10. How does MongoDB differ from MySQL and when should you choose it?

**Answer:**

| Feature | MongoDB | MySQL |
|---------|---------|-------|
| **Data model** | Document (JSON/BSON) | Table (rows/columns) |
| **Schema** | Flexible (schema-less) | Fixed (DDL required) |
| **Scaling** | Native horizontal (sharding) | Primarily vertical |
| **Joins** | `$lookup` (limited) | Full JOIN support |
| **Transactions** | Multi-doc since v4.0 | Full ACID (InnoDB) |
| **Query** | JSON query language | SQL |
| **Best for** | Flexible schema, rapid iteration, hierarchical data | Structured data, complex queries, strict consistency |

**Choose MongoDB when:**
- Schema evolves frequently (startup, MVP)
- Hierarchical/nested data (product catalogs, user profiles)
- High write throughput needed
- Horizontal scaling is required
- Document-oriented data (JSON APIs)

**Choose MySQL when:**
- Data is highly relational
- Complex JOINs and aggregations needed
- Strong ACID transactions required
- Team has SQL expertise

---

### 11. Explain MongoDB replica sets and how failover works.

**Answer:** A **replica set** is a group of MongoDB instances that maintain the same data set for high availability.

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Primary  │────►│Secondary │     │Secondary │
│ (writes) │     │ (replica)│     │ (replica)│
└──────────┘     └──────────┘     └──────────┘
                      ↑                ↑
                      └────Replication─┘
```

**Components:**
- **Primary** — Receives all write operations
- **Secondary** — Replicates data from primary, serves read queries (if configured)
- **Arbiter** — Votes in elections but holds no data (tie-breaking)

**Failover:**
1. Secondaries continuously ping the primary (heartbeat every 2s)
2. If primary is unreachable for `electionTimeoutMillis` (default 10s), an election starts
3. Secondaries vote; the one with the most recent oplog wins
4. New primary is elected (typically < 12 seconds)
5. Application drivers auto-reconnect to the new primary

```javascript
// Connection string with replica set
mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=rs0

// Read preference — control where reads go
db.collection.find().readPref("secondary")       // Read from secondary
db.collection.find().readPref("secondaryPreferred") // Prefer secondary
db.collection.find().readPref("primaryPreferred")   // Prefer primary

// Check replica set status
rs.status()
rs.conf()
```

---

### 12. What is MongoDB sharding and how does it work?

**Answer:** Sharding distributes data across multiple servers for horizontal scaling.

```
                    ┌──────────┐
                    │  mongos  │ (query router)
                    └────┬─────┘
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         ┌─────────┐ ┌─────────┐ ┌─────────┐
         │ Shard 1 │ │ Shard 2 │ │ Shard 3 │
         │(replica │ │(replica │ │(replica │
         │  set)   │ │  set)   │ │  set)   │
         └─────────┘ └─────────┘ └─────────┘
```

**Shard key strategies:**

| Strategy | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Ranged** | Sequential ranges per shard | Range queries efficient | Hot spots if key is sequential |
| **Hashed** | Hash of shard key | Even distribution | Range queries hit all shards |
| **Zone/Tag** | Route data to specific shards by region | Data locality | Manual management |

```javascript
// Enable sharding on database
sh.enableSharding("mydb")

// Shard a collection
sh.shardCollection("mydb.orders", { "user_id": "hashed" })  // Hashed
sh.shardCollection("mydb.logs", { "timestamp": 1 })          // Ranged

// Check sharding status
sh.status()
db.orders.getShardDistribution()
```

**Choosing a shard key:**
- High cardinality (many distinct values)
- Even write distribution
- Supports your query patterns
- **Cannot be changed** after sharding (choose carefully!)

---

### 13. How do you perform backup and restore in MongoDB?

**Answer:**

```bash
# mongodump — logical backup
mongodump --uri="mongodb://user:pass@host:27017/mydb" --out=/backup/$(date +%Y%m%d)

# Backup specific collection
mongodump --db=mydb --collection=users --out=/backup/

# mongorestore — restore from dump
mongorestore --uri="mongodb://user:pass@host:27017" /backup/20240115/

# Restore specific collection
mongorestore --db=mydb --collection=users /backup/20240115/mydb/users.bson

# MongoDB Atlas — automated snapshots
# Continuous backup with point-in-time restore (configurable retention)
```

| Method | Type | Use Case |
|--------|------|----------|
| **mongodump/mongorestore** | Logical | Small-medium DBs, selective restore |
| **Filesystem snapshot** | Physical | Large DBs, fast backup |
| **Atlas Backups** | Managed | Cloud-hosted MongoDB |
| **Ops Manager** | Enterprise | On-prem enterprise |

---

### 14. How do you optimize MongoDB query performance?

**Answer:**

```javascript
// 1. Use explain() to analyze queries
db.orders.find({ user_id: 100, status: "active" }).explain("executionStats")

// Key fields:
// executionStats.totalDocsExamined — docs scanned (lower = better)
// executionStats.nReturned — docs returned
// winningPlan.stage — COLLSCAN (bad) vs IXSCAN (good)

// 2. Create proper indexes
db.orders.createIndex({ user_id: 1, status: 1 })  // Compound index
db.orders.createIndex({ created_at: 1 }, { expireAfterSeconds: 86400 })  // TTL

// 3. Use projection — return only needed fields
db.orders.find({ user_id: 100 }, { _id: 1, total: 1, status: 1 })

// 4. Aggregation pipeline optimization
db.orders.aggregate([
  { $match: { status: "active" } },    // Filter FIRST (uses index)
  { $group: { _id: "$user_id", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])

// 5. Check index usage
db.orders.aggregate([{ $indexStats: {} }])

// 6. Monitor slow queries
db.setProfilingLevel(1, { slowms: 100 })  // Log queries > 100ms
db.system.profile.find().sort({ ts: -1 }).limit(5)
```

---

## PostgreSQL

### 15. What is PostgreSQL and how does it differ from MySQL?

**Answer:**

| Feature | PostgreSQL | MySQL |
|---------|-----------|-------|
| **Standards compliance** | Highly SQL-compliant | Partially compliant |
| **Advanced types** | JSON/JSONB, arrays, hstore, ranges, custom types | JSON (basic), limited |
| **Extensions** | PostGIS, pg_trgm, TimescaleDB, Citus | Limited plugin system |
| **MVCC** | Full MVCC with vacuum | MVCC (InnoDB) |
| **Replication** | Streaming, logical, built-in | Async, semi-sync, group |
| **Performance** | Better for complex queries, analytics | Better for simple reads, high concurrency |
| **License** | PostgreSQL License (very permissive) | GPL (dual-licensed, Oracle) |
| **Best for** | Complex queries, GIS, analytics, JSONB | Web apps, simple CRUD, read-heavy |

```sql
-- PostgreSQL unique features

-- JSONB (binary JSON with indexing)
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    data JSONB NOT NULL
);
INSERT INTO events (data) VALUES ('{"type": "click", "page": "/home", "user_id": 123}');
SELECT data->>'type' as event_type FROM events WHERE data @> '{"page": "/home"}';
CREATE INDEX idx_events_data ON events USING GIN (data);

-- Array columns
CREATE TABLE tags (
    id SERIAL PRIMARY KEY,
    name TEXT,
    labels TEXT[]
);
SELECT * FROM tags WHERE 'urgent' = ANY(labels);

-- CTEs (Common Table Expressions)
WITH recent_orders AS (
    SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days'
)
SELECT user_id, COUNT(*) FROM recent_orders GROUP BY user_id;

-- Window functions
SELECT user_id, amount,
       ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) as rank
FROM orders;
```

---

### 16. How does PostgreSQL replication and high availability work?

**Answer:**

| Type | Description | Use Case |
|------|-------------|----------|
| **Streaming (physical)** | Binary replication of WAL to standby | HA, read replicas |
| **Logical** | Replicate specific tables/databases | Selective replication, upgrades |
| **Synchronous** | Primary waits for standby ACK | Zero data loss |
| **Asynchronous** | Primary doesn't wait | Better performance, slight lag |

```bash
# postgresql.conf (primary)
wal_level = replica
max_wal_senders = 5
synchronous_standby_names = 'standby1'

# pg_hba.conf (primary)
host replication repl_user standby_ip/32 md5

# Create base backup on standby
pg_basebackup -h primary_host -D /var/lib/postgresql/data -U repl_user -P --wal-method=stream
```

**HA solutions:**
- **Patroni** — Cluster management with etcd/ZooKeeper (most popular for K8s)
- **PgBouncer** — Connection pooler (reduces connection overhead)
- **pgpool-II** — Load balancing, connection pooling, failover
- **AWS RDS/Aurora** — Managed HA with auto-failover

---

### 17. How do you perform backup and restore in PostgreSQL?

**Answer:**

```bash
# Logical backup (pg_dump)
pg_dump -h localhost -U postgres mydb > mydb_backup.sql
pg_dump -Fc mydb > mydb_backup.dump           # Custom format (compressed)
pg_dumpall > all_databases.sql                  # All databases + roles

# Restore
psql -U postgres mydb < mydb_backup.sql
pg_restore -d mydb mydb_backup.dump

# Physical backup (pg_basebackup)
pg_basebackup -h primary -D /backup/base -Ft -z -P

# Point-in-time recovery (PITR)
# 1. Restore base backup
# 2. Configure recovery.conf / postgresql.conf:
#    restore_command = 'cp /archive/%f %p'
#    recovery_target_time = '2024-01-15 10:30:00'
# 3. Start PostgreSQL — it replays WAL to target time

# Automated backup with WAL archiving
archive_mode = on
archive_command = 'aws s3 cp %p s3://pg-backups/wal/%f'
```

---

## DynamoDB

### 18. What is DynamoDB and when should you use it?

**Answer:** DynamoDB is AWS's fully managed **NoSQL key-value and document database**, designed for single-digit millisecond performance at any scale.

**Key concepts:**

| Concept | Description |
|---------|-------------|
| **Table** | Collection of items (no fixed schema) |
| **Item** | Single record (like a row, max 400KB) |
| **Partition key** | Primary key — determines data distribution |
| **Sort key** | Optional — enables range queries within a partition |
| **GSI (Global Secondary Index)** | Alternate partition + sort key for different query patterns |
| **LSI (Local Secondary Index)** | Same partition key, different sort key |

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# Put item
table.put_item(Item={
    'order_id': 'ORD-123',
    'user_id': 'USR-456',
    'amount': 99.99,
    'status': 'active',
    'created_at': '2024-01-15T10:30:00Z'
})

# Get item (by primary key — very fast)
response = table.get_item(Key={'order_id': 'ORD-123'})

# Query (partition key + sort key condition)
response = table.query(
    KeyConditionExpression='user_id = :uid AND created_at > :date',
    ExpressionAttributeValues={
        ':uid': 'USR-456',
        ':date': '2024-01-01'
    }
)

# Scan (full table — expensive, avoid in production)
response = table.scan(FilterExpression='status = :s',
                      ExpressionAttributeValues={':s': 'active'})
```

**When to use DynamoDB:**
- Predictable, single-digit ms latency at any scale
- Serverless / event-driven architecture (Lambda + DynamoDB)
- Simple access patterns (key-value lookups, narrow queries)
- Auto-scaling, zero operational overhead

**When NOT to use:**
- Complex queries with JOINs
- Ad-hoc analytics (use Athena or Redshift instead)
- Small, cost-sensitive workloads (minimum throughput costs)

---

### 19. How does DynamoDB capacity and pricing work?

**Answer:**

| Mode | How it works | Best for |
|------|-------------|----------|
| **On-demand** | Pay per request (no provisioning) | Unpredictable traffic, new apps |
| **Provisioned** | Set RCU/WCU (can auto-scale) | Predictable traffic, cost optimization |

**Capacity units:**
- **RCU (Read Capacity Unit):** 1 strongly consistent read/sec (up to 4KB) or 2 eventually consistent
- **WCU (Write Capacity Unit):** 1 write/sec (up to 1KB)

```bash
# Create table with provisioned capacity
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
      AttributeName=order_id,AttributeType=S \
  --key-schema \
      AttributeName=order_id,KeyType=HASH \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=50

# Enable auto-scaling
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --min-capacity 5 --max-capacity 1000
```

**Cost optimization tips:**
- Use **on-demand** for unpredictable workloads
- Use **provisioned + auto-scaling** for steady traffic
- Use **eventually consistent reads** (2x cheaper) where possible
- Use **TTL** to auto-delete expired items (free)
- Use **DynamoDB Accelerator (DAX)** for caching hot keys

---

### 20. What are DynamoDB Streams and how are they used?

**Answer:** DynamoDB Streams captures a time-ordered sequence of item-level changes (insert, update, delete) in a table.

```
Table change → DynamoDB Stream → Lambda trigger → Process event
```

**Use cases:**
- **Event-driven:** Trigger Lambda on data change (send email on new order)
- **Replication:** Sync DynamoDB to OpenSearch, S3, or another table
- **Audit trail:** Log all changes for compliance
- **Aggregation:** Update materialized views / counters

```python
# Lambda handler for DynamoDB Stream
def handler(event, context):
    for record in event['Records']:
        event_name = record['eventName']  # INSERT, MODIFY, REMOVE
        new_image = record['dynamodb'].get('NewImage', {})
        old_image = record['dynamodb'].get('OldImage', {})
        
        if event_name == 'INSERT':
            order_id = new_image['order_id']['S']
            print(f"New order: {order_id}")
            # send_notification(order_id)
```

---

## Database Operations for DevOps

### 21. How do you set up MySQL/PostgreSQL on Kubernetes?

**Answer:**

**Options (from simple to production-grade):**

| Approach | Complexity | Use Case |
|----------|-----------|----------|
| **StatefulSet + PVC** | Medium | Dev/staging |
| **Helm chart (Bitnami)** | Low | Quick setup |
| **Operator (CloudNativePG, Percona)** | Low-Medium | Production |
| **Managed (RDS, Cloud SQL)** | Lowest | Production (recommended) |

```bash
# PostgreSQL via CloudNativePG Operator (production-recommended)
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.0.yaml

cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: mydb
spec:
  instances: 3
  storage:
    size: 50Gi
    storageClass: gp3
  backup:
    barmanObjectStore:
      destinationPath: s3://pg-backups/mydb
      s3Credentials:
        accessKeyID:
          name: aws-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-creds
          key: SECRET_ACCESS_KEY
  monitoring:
    enablePodMonitor: true
EOF

# MySQL via Percona Operator
helm repo add percona https://percona.github.io/percona-helm-charts/
helm install mysql-cluster percona/pxc-db
```

---

### 22. How do you monitor database performance?

**Answer:**

**Key metrics to monitor:**

| Metric | MySQL | PostgreSQL | MongoDB |
|--------|-------|-----------|---------|
| **Connections** | `Threads_connected` | `numbackends` | `current_connections` |
| **Query latency** | Slow query log | `pg_stat_statements` | Profiler |
| **Replication lag** | `Seconds_Behind_Source` | `pg_stat_replication` | `rs.status()` |
| **Cache hit ratio** | Buffer pool hit ratio | `blks_hit / (blks_hit + blks_read)` | WiredTiger cache |
| **Locks/deadlocks** | `SHOW ENGINE INNODB STATUS` | `pg_stat_activity` | `db.currentOp()` |
| **Disk usage** | `information_schema.tables` | `pg_database_size()` | `db.stats()` |
| **Connections** | `max_connections` | `max_connections` | `net.maxIncomingConnections` |

**Prometheus exporters:**
```yaml
# MySQL
- job_name: 'mysql'
  static_configs:
    - targets: ['mysql-exporter:9104']

# PostgreSQL
- job_name: 'postgres'
  static_configs:
    - targets: ['postgres-exporter:9187']

# MongoDB
- job_name: 'mongodb'
  static_configs:
    - targets: ['mongodb-exporter:9216']
```

---

### 23. How do you handle database migrations in a CI/CD pipeline?

**Answer:**

**Tools:**

| Tool | Language | Databases |
|------|----------|-----------|
| **Flyway** | Java (CLI available) | MySQL, PostgreSQL, Oracle |
| **Liquibase** | Java (CLI available) | MySQL, PostgreSQL, MongoDB |
| **Alembic** | Python | PostgreSQL, MySQL |
| **golang-migrate** | Go | PostgreSQL, MySQL, MongoDB |
| **mongomigrate** | Various | MongoDB |

```bash
# Flyway example
flyway -url=jdbc:mysql://localhost:3306/mydb \
  -user=root -password=pass \
  migrate

# Migration files (versioned)
# V1__create_users_table.sql
# V2__add_email_column.sql
# V3__create_orders_table.sql
```

**CI/CD integration best practices:**
- **Version all migrations** in Git alongside application code
- **Never modify** an already-applied migration — create a new one
- **Run migrations before deployment** (pre-deploy step)
- **Test migrations** against a staging copy of production data
- **Always have rollback migrations** (down scripts)
- **Use zero-downtime patterns** — add column, then backfill, then enforce NOT NULL

```yaml
# GitHub Actions — migration step
- name: Run database migrations
  run: |
    flyway -url=$DB_URL -user=$DB_USER -password=$DB_PASS migrate
  env:
    DB_URL: jdbc:mysql://${{ secrets.DB_HOST }}:3306/mydb
```

---

### 24. How do you implement database high availability?

**Answer:**

| Database | HA Solution | RPO | RTO |
|----------|------------|-----|-----|
| **MySQL** | Group Replication, InnoDB Cluster, Percona XtraDB Cluster | ~0 | < 30s |
| **PostgreSQL** | Patroni + etcd, CloudNativePG, Aurora PostgreSQL | ~0 | < 30s |
| **MongoDB** | Replica Set (3+ nodes) | ~0 | < 12s |
| **DynamoDB** | Built-in (multi-AZ, global tables) | 0 | 0 |

**RPO** = Recovery Point Objective (how much data can you lose)
**RTO** = Recovery Time Objective (how fast must you recover)

```
                    ┌─────────────┐
                    │ Load Balancer│
                    │ / ProxySQL   │
                    └──────┬──────┘
                           │
               ┌───────────┼───────────┐
               ▼           ▼           ▼
          ┌─────────┐ ┌─────────┐ ┌─────────┐
          │Primary  │ │Replica 1│ │Replica 2│
          │(writes) │ │(reads)  │ │(reads)  │
          │ AZ-1    │ │ AZ-2    │ │ AZ-3    │
          └─────────┘ └─────────┘ └─────────┘
```

---

### 25. How do you handle database security?

**Answer:**

```sql
-- MySQL — principle of least privilege
CREATE USER 'app_user'@'10.0.%' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'app_user'@'10.0.%';
-- Never GRANT ALL PRIVILEGES to application users

-- PostgreSQL
CREATE ROLE app_readonly;
GRANT CONNECT ON DATABASE mydb TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;

CREATE USER readonly_user WITH PASSWORD 'strong_password';
GRANT app_readonly TO readonly_user;
```

**Security checklist:**
- **Network isolation** — DB in private subnet, no public access
- **Encryption at rest** — Enable TDE or disk encryption
- **Encryption in transit** — TLS/SSL for all connections
- **Least privilege** — App user gets only needed permissions
- **Rotate credentials** — Use AWS Secrets Manager / Vault
- **Audit logging** — Enable query audit logs
- **Backup encryption** — Encrypt backups at rest
- **Patch regularly** — Apply security patches promptly

---

### 26. What is connection pooling and why is it important?

**Answer:** Connection pooling reuses database connections instead of creating new ones for each request, reducing overhead.

**Why it matters:**
- Each DB connection consumes memory (~5-10MB for PostgreSQL)
- Connection creation is expensive (TCP handshake + auth)
- Databases have max connection limits
- Microservices with many replicas can exhaust connections

| Pooler | Database | Type |
|--------|----------|------|
| **PgBouncer** | PostgreSQL | External (recommended) |
| **pgpool-II** | PostgreSQL | External (pooling + HA) |
| **ProxySQL** | MySQL | External (routing + pooling) |
| **HikariCP** | Any | Application-side (JVM) |
| **MongoDB Driver** | MongoDB | Built into driver (default pool) |

```ini
# PgBouncer config
[databases]
mydb = host=postgres port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
pool_mode = transaction     # transaction (recommended) | session | statement
max_client_conn = 1000
default_pool_size = 25
```

---

### 27. How do you handle data migration between databases?

**Answer:**

**MySQL → PostgreSQL migration:**
```bash
# Using pgloader (recommended)
pgloader mysql://user:pass@mysql-host/mydb postgresql://user:pass@pg-host/mydb

# Using AWS DMS (Database Migration Service)
# Source: MySQL → DMS Replication Instance → Target: PostgreSQL
# Supports full load + CDC (Change Data Capture)
```

**MongoDB schema design migration (embedded → referenced):**
```javascript
// Migrate embedded addresses to separate collection
db.users.find().forEach(function(user) {
    if (user.address) {
        db.addresses.insertOne({
            user_id: user._id,
            ...user.address
        });
        db.users.updateOne(
            { _id: user._id },
            { $unset: { address: "" } }
        );
    }
});
```

**Best practices:**
- **Test on staging** with production-like data volume
- **Dual-write** during migration for zero downtime
- **Validate data** — row counts, checksums, sample queries
- **Have rollback plan** — keep old database running until confident
- **Monitor performance** — new database may need different indexes

---

### 28. What is database sharding and when should you use it?

**Answer:** Sharding horizontally partitions data across multiple database instances.

| Strategy | How it works | Example |
|----------|-------------|---------|
| **Range-based** | Partition by value range | users 1-1M → shard1, 1M-2M → shard2 |
| **Hash-based** | Hash of shard key → shard | hash(user_id) % num_shards |
| **Directory-based** | Lookup table maps key → shard | Region → specific shard |
| **Geographic** | Data in region-specific shard | EU users → EU shard |

**When to shard:**
- Single database can't handle write throughput
- Data exceeds single server storage
- Need geographic data locality
- Read replicas aren't enough

**When NOT to shard:**
- Read replicas can solve the problem
- Vertical scaling is still possible
- Application needs cross-shard JOINs
- Data volume < 500GB (usually)

**Native sharding:**
- **MongoDB** — Built-in sharding with `mongos` router
- **DynamoDB** — Automatic partitioning (transparent)
- **MySQL** — Vitess, ProxySQL, or application-level
- **PostgreSQL** — Citus extension, or application-level

---

### 29. How does MongoDB compare to DynamoDB?

**Answer:**

| Feature | MongoDB | DynamoDB |
|---------|---------|----------|
| **Type** | Document database | Key-value + document |
| **Hosting** | Self-hosted or Atlas (managed) | Fully managed (AWS only) |
| **Query** | Rich query language, aggregation pipeline | Limited — primary key + GSI queries |
| **Schema** | Flexible | Flexible |
| **Scaling** | Sharding (manual config) | Automatic partitioning |
| **Transactions** | Multi-document ACID | Transaction API (up to 100 items) |
| **Cost** | Infrastructure cost (or Atlas pricing) | Per-request or provisioned |
| **Latency** | Low (ms) | Consistent single-digit ms |
| **Cloud lock-in** | None (run anywhere) | AWS only |
| **Best for** | Complex queries, aggregation, flexibility | Simple access patterns, serverless, massive scale |

**Choose MongoDB when:** Complex queries, aggregation, need to run anywhere (multi-cloud/on-prem)
**Choose DynamoDB when:** AWS-native, serverless architecture, simple key-value patterns, need zero ops

---

### 30. What are common database troubleshooting scenarios for DevOps?

**Answer:**

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| **High CPU** | Slow queries, missing indexes | `EXPLAIN` queries, add indexes, optimize queries |
| **High connections** | Connection leaks, no pooling | Add PgBouncer/ProxySQL, fix app connection handling |
| **Replication lag** | Heavy writes, slow network, large transactions | Tune `binlog` format, upgrade replica, parallel replication |
| **Disk full** | Large tables, binlogs, WAL accumulation | Purge old binlogs, archive data, increase disk |
| **Deadlocks** | Concurrent conflicting transactions | Review transaction order, reduce lock scope, add indexes |
| **OOM** | Buffer pool/shared_buffers too large, too many connections | Tune memory settings, add connection pooler |
| **Slow failover** | No automated failover | Implement Patroni (PG), Orchestrator (MySQL), or use managed DB |
| **Backup failure** | Disk space, long-running transactions, permissions | Monitor backup jobs, test restores regularly |

```bash
# Quick health check script
# MySQL
mysql -e "SHOW GLOBAL STATUS LIKE 'Threads_connected';"
mysql -e "SHOW GLOBAL STATUS LIKE 'Slow_queries';"
mysql -e "SHOW REPLICA STATUS\G" | grep -E "Seconds_Behind|Running"

# PostgreSQL
psql -c "SELECT count(*) FROM pg_stat_activity;"
psql -c "SELECT * FROM pg_stat_replication;"
psql -c "SELECT pg_database_size('mydb');"

# MongoDB
mongosh --eval "rs.status()"
mongosh --eval "db.serverStatus().connections"
mongosh --eval "db.currentOp({'active': true})"
```
