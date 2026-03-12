# Module 3: The CAP Theorem

## The Theorem

You can have **at most 2 of 3** properties:

- **Consistency** (C): All replicas see the same data
- **Availability** (A): System responds to requests despite failures
- **Partition Tolerance** (P): System works despite network splits

```
    Consistency
         / \
        /   \
       /     \
      /       \
     /_________\
    Availability Partition
```

You must choose 2. Every system makes this choice, explicitly or implicitly.

## The Three Choices

### CP: Consistency + Partition Tolerance (Sacrifices Availability)

When network splits, one partition goes offline to maintain consistency.

**Example**: PostgreSQL with 2 replicas, split-brain prevention.

```
Network splits into Group A (2 nodes) and Group B (1 node)

Group A: Has quorum (2/3), continues accepting writes
Group B: No quorum (1/3), goes offline for writes

Result: Consistency maintained (one source of truth)
        But Group B unavailable
```

**Use case**: Financial systems (money correctness > occasional downtime).

Your platform: **PostgreSQL is CP-leaning** (strong consistency, occasional replica lag).

### AP: Availability + Partition Tolerance (Sacrifices Consistency)

When network splits, both sides continue. Might diverge. Eventually sync.

**Example**: Redis cache, DynamoDB, Cassandra.

```
Network splits into Group A and Group B

User in Group A:  SET x=1 (in Group A's cache)
User in Group B:  SET x=2 (in Group B's cache)

Network heals
Sync conflict: Group A has x=1, Group B has x=2
Last-write-wins: The one written later wins
```

**Use case**: Social media (availability > perfect consistency). If comment likes diverge momentarily, not a disaster.

Your platform: **Redis is AP** (available, eventually consistent).

### CA: Consistency + Availability (Sacrifices Partition Tolerance)

No network splits (single machine or tightly-coupled cluster).

**Example**: Single PostgreSQL server (no replicas).

```
One machine: FastAPI + PostgreSQL
No network split possible (one machine can't split from itself)
Consistency: one source of truth
Availability: always up
```

**Issue**: Not realistic in production. Network splits happen.

## Your Platform's Choices

| Component | Choice | Why |
|-----------|--------|-----|
| **PostgreSQL** | CP | ACID transactions, strong consistency. Replica lag acceptable. |
| **Redis** | AP | Fast, available, eventual consistency. Cache misses okay. |
| **Celery Queue** | AP | Exactly-once is expensive; at-least-once + idempotency is practical. |
| **MinIO** | AP | Distributed storage, availability over instant consistency. |

### PostgreSQL: Consistency Over Availability

```python
# Strong consistency
user = db.query(User).filter_by(id=user_id).first()
user.balance -= 100
db.commit()  # ACID transaction

# If network splits, PostgreSQL chooses consistency
# Continues: "Sorry, replicas are unreachable, but I have the data"
# Stops: "Would lose data if I continued, so I stop"
```

At most, PostgreSQL unavailable during network split that would cause data loss.

### Redis: Availability Over Consistency

```python
# Eventual consistency
redis.set("user_balance", 900, ex=300)  # Cache for 5 minutes

# If Redis replicates and network splits:
# Both replicas continue serving cached data
# On heal, merge (last-write-wins, or combine)
```

Might return stale data briefly, but always available.

## PACELC Extension

CAP theorem is binary (P is present). Real systems have another choice:

**PACELC**: Given **P**artition, choose **E**lysium and **L**atency vs **C**onsistency. Else, **L**atency vs **C**.

When no partition:
- **EC**: Emphasize **E**ventual consistency to reduce **L**atency
- **LA**: Emphasize **L**atency (respond fast) over consistency

Your platform:
- **PostgreSQL**: When operational, chooses LA (responds immediately, trade consistency if necessary)
- **Redis**: Chooses LA (always fast, eventual consistency)
- **Celery**: Chooses LA (fast queue, eventual processing)

## Trade-offs in Practice

### Scenario 1: User Balance Transfer

User transfers $100 from account A to account B.

```
Atomicity needed: Either both succeed or both fail
Consistency needed: Balance must be correct
```

**Choice**: PostgreSQL (CP).

```python
@app.post("/api/accounts/transfer")
def transfer(from_id, to_id, amount):
    # ACID transaction
    with db.begin():
        from_account = db.query(Account).filter_by(id=from_id).with_for_update().first()
        to_account = db.query(Account).filter_by(id=to_id).with_for_update().first()
        
        from_account.balance -= amount
        to_account.balance += amount
        
    # Committed atomically
    return {"status": "success"}
```

If network splits, PostgreSQL stops (prevents inconsistency).

### Scenario 2: Cache Most Recent Documents

Show user 10 most recently viewed documents.

```
Exact order doesn't matter much
Fast response matters
```

**Choice**: Redis (AP).

```python
# Cache for 5 minutes, eventual consistency
recent_docs = redis.lrange(f"user:{user_id}:recent", 0, 10)
if recently_docs:
    return recent_docs  # May be 5 minutes stale

# Cache miss, fetch fresh
fresh_docs = db.query(Document).filter_by(user_id=user_id).order_by(desc(created_at)).limit(10).all()
redis.delete(f"user:{user_id}:recent")
redis.rpush(f"user:{user_id}:recent", [doc.id for doc in fresh_docs])

return fresh_docs
```

Acceptable to show slightly old list if cache is fresh.

### Scenario 3: Async Task Processing

Process uploaded PDF: convert, scan, extract metadata.

```
Takes 30 seconds
User can't wait
Exact-once processing is hard
```

**Choice**: Celery with at-least-once (requires idempotency, see Module 5).

```python
@app_celery.task
def process_pdf(pdf_id):
    pdf = db.query(PDF).get(pdf_id)
    
    # Idempotent: safe to call multiple times
    # Marked as processing (skip if already done)
    if pdf.status == "processed":
        return
    
    # Extract metadata
    metadata = extract_metadata(pdf)
    pdf.metadata = metadata
    
    # Mark done
    pdf.status = "processed"
    db.commit()
```

If worker crashes and retries, idempotent operation handles it.

## Hands-On Lab

### Lab 3.1: Test CAP with Docker Chaos

```bash
# Drop all network traffic to Redis
docker exec bridge_redis_1 ip link set eth0 down

# API requests to Redis now timeout
curl http://localhost:8000/api/cache_test
# Returns error (AP system: unavailable)

# Restore Redis
docker exec bridge_redis_1 ip link set eth0 up
```

See how Redis becomes unavailable during partition.

### Lab 3.2: PostgreSQL Replica Split

```bash
# Start PostgreSQL with replicas
docker-compose up postgres postgres-replica-1

# Monitor replication
docker exec postgres_postgres_1 psql -c "SELECT * FROM pg_stat_replication;"

# Kill network to replica
docker network disconnect bridge postgres_replica_1

# Try write to primary
curl -X POST http://localhost:8000/api/pay --data '{"amount": 100}'
# Success (primary continues, replica isolated)

# Restore network
docker network connect bridge postgres_replica_1

# Replication catches up
```

See how PostgreSQL (CP) continues with primary, isolates replica to maintain consistency.

## Cheat Sheet: CAP Choices

| Property | When to Choose | Cost |
|----------|---|---|
| **CP** (consistency + partition) | Money, correctness critical | Availability, latency |
| **AP** (availability + partition) | Availability, speed critical | Eventual consistency |
| **CA** (consistency + availability) | Not practical (no partition handling) | Single point of failure |

## Decision Tree

```
Is data loss unacceptable?
├─ Yes (money, critical state)
│  └─ Use CP (PostgreSQL, etcd, Zookeeper)
└─ No (cache, analytics, logs)
   └─ Use AP (Redis, DynamoDB, Cassandra)

Is response latency critical (< 10ms)?
├─ Yes
│  └─ Use AP (cache, local storage)
└─ No
   └─ Can use CP if data criticality requires
```

## Key Takeaways

- **CAP theorem**: Choose 2 of consistency, availability, partition tolerance
- **Consistency** = all replicas identical, correctness guaranteed
- **Availability** = always responds, possibly stale
- **Partition tolerance** = continues despite network split
- **PostgreSQL**: CP (stops to protect data)
- **Redis**: AP (available, eventual consistency)
- **Celery**: AP (fast, requires idempotency)
- **Choose CP** for financial data, user data, correctness-critical
- **Choose AP** for caches, analytics, UI data

Module 4 teaches distributed locks, critical for preventing concurrent modifications.
