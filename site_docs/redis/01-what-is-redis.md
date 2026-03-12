# Module 1: What is Redis

## The Analogy: RAM vs Disk

**Hard Disk** (like PostgreSQL):
- Reads: 5-10ms per query
- Reliable: survives power loss
- Large: terabytes of data

**RAM** (Redis):
- Reads: 0.1ms (100x faster!)
- Temporary: lost if power cut
- Limited: gigabytes

Redis trades durability for speed. Perfect for caches, sessions, queues.

## In-Memory Storage

Redis keeps everything in **RAM**:

```
USUAL DATABASE:
┌──────────────────────┐
│  Hard Disk (slow)    │
│  [all data on disk]  │
└──────────────────────┘
         ↑ read/write
       Application

REDIS:
┌──────────────────────┐
│  RAM (fast)          │
│  [all data in RAM]   │
└──────────────────────┘
         ↑ read/write
       Application
```

Trade-off: RAM is expensive. PostgreSQL stores everything. Redis stores hot data.

## Use Cases

### 1. Cache

```python
@app.get("/user-profile/{user_id}")
async def get_profile(user_id: int, redis_client):
    # Check cache
    cached = await redis_client.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)  # 0.1ms
    
    # Query database
    user = await get_user_from_db(user_id)  # 5ms
    
    # Store in cache for 1 hour
    await redis_client.setex(f"user:{user_id}", 3600, json.dumps(user))
    
    return user
```

### 2. Session Storage

```python
@app.post("/login")
async def login(email: str, password: str, redis_client):
    user = verify_credentials(email, password)
    
    # Create session token
    token = generate_jwt()
    
    # Store in Redis (expires after 24 hours)
    await redis_client.setex(f"session:{token}", 86400, user.id)
    
    return {"token": token}
```

### 3. Rate Limiting

```python
@app.get("/api/convert")
async def convert(user_id: int, redis_client):
    # Count requests per hour
    key = f"rate_limit:{user_id}:{current_hour}"
    count = await redis_client.incr(key)
    
    if count > 100:  # Max 100 requests/hour
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    
    # Expire key after 1 hour
    await redis_client.expire(key, 3600)
    
    return convert_pdf()
```

### 4. Task Queue (Celery)

```python
# FastAPI
@app.post("/submit-job")
async def submit_job(job_data):
    # Queue job in Redis
    task = celery_app.send_task('tasks.convert_pdf', args=(job_data,))
    return {"task_id": task.id}

# Celery worker
@celery_app.task
def convert_pdf(job_data):
    result = libreoffice_convert(job_data)
    return result
```

## Redis Installation

### Docker

```bash
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine
```

Test:

```bash
docker exec redis redis-cli ping
# PONG
```

### Connection

```python
import redis.asyncio as redis

client = await redis.from_url("redis://localhost:6379/0")

# Test
await client.ping()  # True

# Close
await client.close()
```

## Persistence Options

Redis data lives in RAM. But you can save to disk:

### Option 1: RDB (Snapshots)

```ini
# redis.conf
save 900 1   # Save if 1 key changes in 900 seconds
save 300 10  # Or if 10 keys change in 300 seconds
```

Saves entire dataset to `dump.rdb` on disk. On restart, reload from disk.

**Pros**: Fast, compact
**Cons**: Data between snapshots is lost

### Option 2: AOF (Append-Only File)

```ini
# redis.conf
appendonly yes
appendfsync everysec  # Fsync every second
```

Every write command is logged. On restart, replay all commands.

**Pros**: More durable
**Cons**: Slower, bigger file

### Option 3: No Persistence

```python
# Treat as pure cache
# Data is lost on restart (acceptable for sessions/cache)
```

For our PDF SaaS:
- **Sessions**: AOF (fsync every second) — important but can lose 1 second
- **Cache**: No persistence — if lost, re-query database
- **Task queue**: AOF — jobs must not be lost

## Replication

Redis can replicate to a replica:

```
Master Redis (read/write)
        ↓
Replica Redis (read-only)
        ↓
Replica Redis (read-only)
```

Reads can be distributed. Writes still go to master.

## Data Types

Redis supports multiple data types:

| Type | Example | Use |
|------|---------|-----|
| String | `"Alice"` | Simple values, cache |
| List | `[1, 2, 3]` | Queue, leaderboard |
| Hash | `{name: "Alice", age: 30}` | Objects, session data |
| Set | `{a, b, c}` | Unique items, tags |
| Sorted Set | `{a:1, b:2, c:3}` | Scores, leaderboards |

## Real PDF SaaS Data

```
STRING: user:5:token = "eyJ..."
HASH:   user:5 = {id: 5, email: "alice@example.com", created_at: "2024-01-15"}
LIST:   queue:pdf = [job1, job2, job3, ...]  (Celery uses this)
SET:    user:5:permissions = {read, write, admin}
ZSET:   leaderboard:2024 = {user5: 42, user3: 38, user1: 35}
```

## Lua Scripting

For atomic multi-step operations:

```python
# Increment counter, expire key (atomic)
script = """
redis.call('INCR', KEYS[1])
redis.call('EXPIRE', KEYS[1], ARGV[1])
"""

await client.eval(script, 1, 'counter:5', 3600)
```

All-or-nothing. Prevents race conditions.

## Hands-On Lab

### Lab 1.1: Basic Commands

```bash
docker run -it redis:7-alpine redis-cli

# Strings
SET mykey "Hello"
GET mykey
DELETE mykey

# Hashes
HSET user:1 name Alice age 30
HGET user:1 name
HGETALL user:1

# Lists
RPUSH queue job1 job2 job3
LPOP queue
LLEN queue

# Sets
SADD tags python celery redis
SMEMBERS tags
SISMEMBER tags python

# Sorted Sets
ZADD scores 100 alice 85 bob
ZRANGE scores 0 -1 WITHSCORES

# TTL
SETEX session:token 3600 "user_id=5"
TTL session:token
EXPIRE session:token 7200
```

### Lab 1.2: Python Client

```python
import redis.asyncio as redis
import asyncio

async def test():
    client = await redis.from_url("redis://localhost:6379/0")
    
    # String
    await client.set("name", "Alice")
    print(await client.get("name"))
    
    # Hash
    await client.hset("user:1", mapping={"name": "Alice", "age": 30})
    print(await client.hgetall("user:1"))
    
    # Expire
    await client.setex("temp", 10, "data")
    await asyncio.sleep(11)
    print(await client.get("temp"))  # None
    
    await client.close()

asyncio.run(test())
```

## Cheat Sheet: Redis Basics

### Installation

```bash
docker run -d -p 6379:6379 redis:7-alpine
```

### Connection (Python)

```python
import redis.asyncio as redis

client = await redis.from_url("redis://localhost:6379/0")
await client.close()
```

### String Commands

```
SET key value
GET key
SETEX key seconds value
DEL key
INCR key
```

### Hash Commands

```
HSET key field value
HGET key field
HGETALL key
HDEL key field
```

### List Commands

```
RPUSH key value (right)
LPUSH key value (left)
LPOP key
RPOP key
LLEN key
```

### Set Commands

```
SADD key member
SMEMBERS key
SISMEMBER key member
SCARD key
```

## Key Takeaways

- **Redis is in-memory** — 100x faster than disk
- **Perfect for cache** — session tokens, user profiles
- **Perfect for rate limiting** — counter per IP
- **Perfect for task queue** — Celery message broker
- **Persistence optional** — RDB snapshots or AOF logs
- **TTL = expiration** — keys automatically deleted
- **Multiple data types** — strings, lists, hashes, sets, sorted sets

Module 2 teaches data structures and when to use each.
