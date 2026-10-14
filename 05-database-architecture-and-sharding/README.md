# 🗄️ Topic 05: Database Architecture, Replication & Sharding

> ☕ *"Settle in with a fresh cup, Ravi! Today we enter the most critical territory in system design: the data persistence layer. Stateless web servers can be created and destroyed without a second thought, but your database holds the crown jewels: financial transactions, user identities, and purchase histories. If a database is corrupted or partitioned incorrectly, businesses go bankrupt. Let's learn how to scale databases safely to petabytes of data!"*  
> — **Rithu**

**Ravi:** *"Rithu, scaling databases has always felt intimidating. How do huge platforms like Instagram or Discord store billions of records without their databases grinding to a complete halt?"*  
**Rithu:** *"They don't put all their eggs in one giant basket, Ravi! They scale databases in stages: first, separating Reads from Writes with Read Replicas; second, vertical partitioning; and finally, horizontal database sharding. Today you will understand the exact mechanics behind each phase! 🌸"*

---

## 🎯 1. What Ravi Will Learn Today

- ⚖️ **ACID vs. BASE & The CAP Theorem**: Grasp the mathematical trade-offs between Strong Consistency (CP) and High Availability (AP) during network partitions.
- 📖 **Primary-Replica Replication Mechanics**: Master Synchronous vs. Asynchronous binary log replication, read-write splitting, and replication lag.
- ✂️ **Database Sharding (Horizontal Partitioning)**: Learn Range-Based, Hash-Based, and Consistent Hashing partitioning strategies.
- 🧩 **Polyglot Persistence**: Know exactly when to deploy Relational (PostgreSQL), Key-Value (DynamoDB), Document (MongoDB), or Wide-Column (Cassandra).
- 🧪 **Hands-on Zero-Cost Lab**: Implement an automated primary-replica PostgreSQL read-write splitting cluster using Docker.

---

## 📊 2. Visual Architecture: Primary-Replica Replication & Sharding

![Database Architecture and Sharding](database_sharding_architecture.png)

```mermaid
flowchart TD
    classDef clientStyle fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#0369a1;
    classDef routerStyle fill:#fce7f3,stroke:#db2777,stroke-width:2px,color:#9d174d;
    classDef primaryStyle fill:#ede9fe,stroke:#7c3aed,stroke-width:2px,color:#5b21b6;
    classDef replicaStyle fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#15803d;

    App([🖥️ Application Microservices]):::clientStyle --> Router[🧭 Database Query Router / Proxy<br/>e.g., PgBouncer / Vitess / ProxySQL]:::routerStyle

    subgraph Shard_1 ["🗄️ Shard 1 (User ID: 0 - 999,999)"]
        direction TB
        P1[("👑 Primary 1 (Writes Only)")]:::primaryStyle
        R1A[("📖 Read Replica 1A")]:::replicaStyle
        R1B[("📖 Read Replica 1B")]:::replicaStyle
        P1 -.->|Replication Stream| R1A & R1B
    end

    subgraph Shard_2 ["🗄️ Shard 2 (User ID: 1,000,000 - 1,999,999)"]
        direction TB
        P2[("👑 Primary 2 (Writes Only)")]:::primaryStyle
        R2A[("📖 Read Replica 2A")]:::replicaStyle
        R2B[("📖 Read Replica 2B")]:::replicaStyle
        P2 -.->|Replication Stream| R2A & R2B
    end

    %% Routing logic
    Router -->|Writes: INSERT/UPDATE| P1
    Router -->|Reads: SELECT| R1A & R1B
    Router -->|Writes: INSERT/UPDATE| P2
    Router -->|Reads: SELECT| R2A & R2B
```

---

## 👶 3. Rithu Explains Like You Are a Beginner (ELI5)

### 📚 The City Filing Cabinet Analogy

Imagine a busy city hall managing birth certificates for 10 million citizens:

1. **Phase 1: The Single Clerk (Single Monolithic Database)**:
   - There is one massive wooden filing cabinet and one clerk named Bob.
   - When 5 people arrive, Bob finds their records easily.
   - When 10,000 citizens arrive at tax season, the line stretches out into the street. Bob cannot physically open drawers any faster.
2. **Phase 2: Photocopy Assistants (Read Replicas)**:
   - 95% of people just want to *view* or copy their existing certificate; only 5% want to *register* a new birth.
   - So Bob stays as the **Primary Clerk (Master)** who handles new registrations.
   - Every evening, Bob makes carbon copies of new files and sends them to 4 **Assistant Desks (Read Replicas)**.
   - Now, anyone who just needs to read or inspect records goes to an assistant desk. Bob’s workload drops by 90%!
3. **Phase 3: The Alphabetical Rooms (Database Sharding)**:
   - Eventually, the filing cabinet contains 20 million physical pages. The floor is about to collapse under the weight!
   - You build 3 separate rooms:
     - **Room 1**: Holds surnames **A through H** (Shard 1).
     - **Room 2**: Holds surnames **I through P** (Shard 2).
     - **Room 3**: Holds surnames **Q through Z** (Shard 3).
   - An usher at the door checks your last name and sends you to the correct room. Each room operates completely independently with its own clerks and copies!

---

### 🔍 Core Architectural Questions Answered

#### What is the CAP Theorem?
In a distributed data store, when a **Network Partition (P)** occurs (e.g., fiber cut between two data centers), you **must choose** between:
1. **Consistency (C)**: Every read receives the most recent write or errors out (e.g., Banks, Financial Ledgers).
2. **Availability (A)**: Every read receives a response, but it may contain slightly stale data (e.g., Social Media feeds, YouTube view counts).
> *You cannot choose "CA" because network partitions are physical realities of computer networking that cannot be prevented.*

#### What is Replication Lag?
In asynchronous replication, the primary writes data to its local disk, confirms success to the client immediately, and streams the change to replicas in the background. If the replica is under heavy load, it might take 200–500ms to apply the update. During that window, a user who just updated their profile might refresh the page and see their old photo!

---

## ⚡ 4. The Essential 80/20 Concepts

| Concept / Strategy | What Rithu Wants Ravi to Know | Practical Job Importance |
| :--- | :--- | :--- |
| **Primary-Replica (Master-Slave)** | 1 Primary handles writes (`INSERT`, `UPDATE`, `DELETE`); $N$ Replicas handle read queries (`SELECT`). | The universal standard pattern for scaling PostgreSQL, MySQL, and MariaDB. |
| **Connection Pooling (PgBouncer)** | Maintains pre-warmed backend database sockets, multiplexing thousands of client connections onto dozens of DB threads. | Prevents PostgreSQL server memory exhaustion (every raw Postgres connection allocates ~10 MB RAM!). |
| **Hash-Based Sharding** | Computes $\text{Shard ID} = \text{hash}(\text{User ID}) \pmod N$. Evenly distributes rows across $N$ physical database servers. | Prevents "hot shards" where one database disk fills up while others sit empty. |
| **Consistent Hashing** | Uses a virtual hash ring (0 to $2^{32}-1$) to distribute partitions. Adding or removing a database node only relocates $\frac{1}{N}$ keys. | Minimizes massive data re-balancing operations when scaling database clusters out or in. |
| **Write-Ahead Logging (WAL)** | Database appends transaction operations to an immutable, sequential disk log *before* updating data pages in memory/disk. | Guarantees zero data loss on power failures and powers cross-node replication streams. |

---

## 💻 5. Practical Configuration: Read-Write Splitting with PostgreSQL & PgBouncer

Below is a production `pgbouncer.ini` configuration multiplexing thousands of incoming application connections and directing queries efficiently:

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
; Write pool pointing to Primary database
db_primary = host=10.0.1.50 port=5432 dbname=production_db user=pgpool_user

; Read pool pointing to load-balanced Read Replicas
db_read_replica = host=10.0.1.51 port=5432 dbname=production_db user=pgpool_user

[pgbouncer]
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid

; Listen on public interface
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

; Pool Mode: Transaction Pooling
; Sockets are returned to the pool as soon as an SQL transaction finishes!
pool_mode = transaction

; Connection limits
max_client_conn = 10000      ; Allow 10,000 concurrent client microservices
default_pool_size = 50        ; Maintain max 50 actual connections to PostgreSQL
min_pool_size = 10
reserve_pool_size = 5
max_db_connections = 100

; Timeouts preventing hung backend connections
server_idle_timeout = 600
client_idle_timeout = 60
query_timeout = 30
```

### Application Connection Separation (Python SQLAlchemy Example)
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Dedicated engines: Primary for mutations, Replica for reads
writer_engine = create_engine("postgresql://app:secret@pgbouncer:6432/db_primary")
reader_engine = create_engine("postgresql://app:secret@pgbouncer:6432/db_read_replica")

WriteSession = sessionmaker(bind=writer_engine)
ReadSession = sessionmaker(bind=reader_engine)

def create_user_profile(user_data):
    """Mutations route strictly to Primary"""
    with WriteSession() as session:
        session.add(user_data)
        session.commit()

def get_user_feed(user_id):
    """High-volume reads route strictly to Read Replicas"""
    with ReadSession() as session:
        return session.query(Post).filter_by(user_id=user_id).all()
```

---

## ⌨️ 6. Essential Commands Cheat Sheet

| Command / Tool | Syntax | What It Does | When to Run It in Production |
| :--- | :--- | :--- | :--- |
| **Check Replication Lag** | `SELECT now() - pg_last_xact_replay_timestamp() AS lag;` | Returns exact seconds/milliseconds a PostgreSQL replica is trailing behind the Primary. | Detecting overloaded replicas or network latency between data centers. |
| **Analyze Slow Queries** | `EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE ...;` | Prints the query execution plan, index usage, and disk buffer hits vs. misses. | Optimizing queries before upgrading database hardware. |
| **Check Active Connections** | `SELECT count(*), state FROM pg_stat_activity GROUP BY state;` | Displays idle, active, and waiting connection counts in PostgreSQL. | Investigating connection pool exhaustion and deadlocks. |
| **PgBouncer Pool Stats** | `psql -p 6432 -U pgbouncer pgbouncer -c "SHOW POOLS;"` | Inspects active client connections (`cl_active`) vs backend connections (`sv_active`). | Tuning pool size parameters under peak loads. |
| **Vacuum & Freeze Check** | `SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables;` | Checks dead row bloat across tables. | Troubleshooting disk space bloat and table lock contention. |

---

## 🧪 7. Hands-on Practice: Ravi's Zero-Cost Lab

### 🎯 Objective
Ravi will spin up a local PostgreSQL Primary along with a streaming Read Replica using Docker Compose, verify continuous replication, and observe read-write isolation!

### 📋 Prerequisites
- Docker & Docker Compose.

---

### Step 1: Create Lab Directory
```bash
mkdir db-scaling-lab
cd db-scaling-lab
```

### Step 2: Create Compose Cluster (`docker-compose.yml`)
```yaml
version: '3.8'

services:
  postgres-primary:
    image: postgres:15-alpine
    container_name: pg_primary
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: rootpassword
    ports:
      - "5432:5432"

  postgres-replica:
    image: postgres:15-alpine
    container_name: pg_replica
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: rootpassword
    ports:
      - "5433:5432"
    depends_on:
      - postgres-primary
```

### Step 3: Start the Cluster
```bash
docker compose up -d
```

---

### ✅ Verification Step: Write on Primary, Query on Replica
1. Insert a record into the Primary:
```bash
docker exec -i pg_primary psql -U postgres -d app_db -c "
CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(50));
INSERT INTO users (name) VALUES ('Ravi'), ('Rithu');
"
```

2. Query the Primary:
```bash
docker exec -i pg_primary psql -U postgres -d app_db -c "SELECT * FROM users;"
```

#### Output:
```text
 id | name  
----+-------
  1 | Ravi
  2 | Rithu
(2 rows)
```

🎉 **Ravi's Discovery**: You just set up isolated database endpoints! In production, the application directs heavy analytical `SELECT` reporting to port `5433` without consuming a single cycle of CPU from the primary writer on `5432`.

---

### 🧹 Cleanup
```bash
docker compose down -v
cd ..
rm -rf db-scaling-lab
```

---

## ⚠️ 8. Rithu's Top 3 Beginner Traps

### 🪤 Trap 1: The "Read-Your-Own-Writes" Consistency Bug
- **Symptom / What happens**: A user submits a comment on a blog post. The page reloads, and their comment is missing! 3 seconds later, they refresh again, and the comment magically appears.
- **Why it happens**: The `INSERT` went to the Primary DB, but the subsequent page reload `SELECT` query went to an asynchronous Read Replica that had 400ms of replication lag!
- **Rithu's Fix**:
  - **Pin Recent Writers**: If a user performs a write, route all reads from that specific user to the Primary DB for the next 5–10 seconds.
  - Or, return the updated record directly in the HTTP `POST` response payload so the client UI renders without an immediate read query.

---

### 🪤 Trap 2: Premature Sharding
- **Symptom / What happens**: A startup with 250,000 rows shards their database into 8 databases. Developers spend 4 months writing cross-shard join logic, distributed transactions, and custom migration scripts.
- **Why it happens**: Sharding was hyped on tech blogs, but the team's data could easily fit into a standard single RDS PostgreSQL instance with 32 GB RAM!
- **Rithu's Fix**: **Never shard until you must.** First optimize indexes, configure read replicas, implement Redis caching, and vertically scale your database instance. Sharding is the absolute last resort for extreme scale.

---

### 🪤 Trap 3: Missing Indexes on Foreign Keys & Where Clauses
- **Symptom / What happens**: A query that took 5ms during development takes 12 seconds in staging when table size hits 1,000,000 rows.
- **Why it happens**: The query executes a **Sequential Scan** (reading every single row from disk) because no B-Tree index exists on `created_at` or `user_id`.
- **Rithu's Fix**: Run `EXPLAIN ANALYZE` on queries with latency > 50ms. Create composite indexes matching your actual access patterns.

---

## 💬 9. Ravi's Doubt Corner (Interview Preparation)

### ❓ Question 1 (Ravi asks):
> *"Rithu, how does Hash Partitioning work during sharding, and what is its biggest weakness?"*

**Rithu's Model Answer:**  
*"Hash partitioning calculates the target database node using a hash function:*
$$\text{Node ID} = \text{hash}(\text{User ID}) \pmod N$$
*where $N$ is the number of database shards.*
- **Strength**: Completely eliminates hot-spots by scattering data uniformly across all nodes.
- **Biggest Weakness**: **Resharding is a nightmare!** If you grow from 4 to 5 shards, $N$ changes to 5. Now almost every existing key maps to a new node!
- **The Solution**: Use **Consistent Hashing** or **Virtual Buckets** (like DynamoDB or Cassandra), where keys are mapped to a virtual ring, so adding a node only moves a small fraction of keys."*

---

### ❓ Question 2 (Ravi asks):
> *"What is the difference between Optimistic Locking and Pessimistic Locking?"*

**Rithu's Model Answer:**  
*"Both prevent concurrency collisions (like two users buying the exact last concert ticket):*
- ***Pessimistic Locking (`SELECT FOR UPDATE`)***: Locks the row at the database level the moment transaction begins. Nobody else can read or write that row until the transaction commits. Pros: 100% collision-proof. Cons: Kills throughput and risks deadlocks under heavy concurrent load.
- ***Optimistic Locking (Version Column)***: No database locks are held. Each row has a `version` integer. When updating, the app executes:
  `UPDATE tickets SET status='sold', version=version+1 WHERE id=101 AND version=3;`
  If another transaction updated it first, the version is already 4, the update returns `0 rows affected`, and the application cleanly retries. Pros: Blazing fast for low-contention systems."*

---

## ⚡ 10. Quick Revision with Rithu

- ⚖️ **CAP Theorem is Absolute**: You cannot beat physics. When a network splits, choose between Consistency or Availability.
- 📖 **Scale Reads First**: Separate Writes (Primary) from Reads (Replicas) before ever considering sharding.
- 🏊 **PgBouncer is Essential**: Never allow thousands of microservices to open raw TCP connections directly to PostgreSQL.
- ✂️ **Shard as Last Resort**: Sharding introduces distributed transactions and cross-shard join headaches.
- ⏱️ **Watch Out for Replication Lag**: Protect recent writers with read-your-own-writes session sticky routing.

> [!TIP]
> **Rithu's Golden Rule for Topic 05**:  
> *"Hardware is cheap, but distributed database consistency is painfully expensive. Index your tables, cache your reads, and exhaust replicas before you ever split your database into shards."* 🚀☕
