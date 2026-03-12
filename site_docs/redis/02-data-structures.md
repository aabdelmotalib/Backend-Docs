# Module 2: Data Structures and Operations

## Strings: Simple Key-Value

The simplest Redis data type. Each string stores a value (up to 512MB).

### Commands

```
SET key value              # Store
GET key                    # Retrieve
APPEND key suffix          # Append to value
STRLEN key                 # Length
GETRANGE key 0 5           # Substring
SETRANGE key 0 value       # Replace substring
INCR key                   # Increment (for numbers)
DECR key                   # Decrement
INCRBY key 5               # Add 5
GETSET key newvalue        # Set and return old
```

### Use Cases

```python
# Cache user profile
await redis.set("user:5", json.dumps({
    "id": 5,
    "email": "alice@example.com",
    "name": "Alice"
}))

user = json.loads(await redis.get("user:5"))

# Counter
await redis.incr("conversion:count")  # Increment

# Atomic check-and-set
await redis.getset("current_task", "task_id_123")
```

## Hashes: Object Storage

A hash is a map of field-value pairs. Perfect for objects.

```
HSET key field value       # Set field
HGET key field             # Get field
HMSET key field1 val1 ...  # Multiple fields
HGETALL key                # All fields
HDEL key field             # Delete field
HINCRBY key field 1        # Increment field (numeric)
HEXISTS key field          # Check exists
```

### Example

```python
# Store user object
await redis.hset("user:5", mapping={
    "email": "alice@example.com",
    "name": "Alice Smith",
    "plan": "pro"
})

# Get field
email = await redis.hget("user:5", "email")  # "alice@example.com"

# Get all
user_data = await redis.hgetall("user:5")
# {b'email': b'alice@example.com', b'name': b'Alice Smith', b'plan': b'pro'}

# Update field
await redis.hset("user:5", "plan", "enterprise")

# Increment (tracking usage)
await redis.hincrby("user:5", "conversions", 1)
```

## Lists: Queues and Stacks

Ordered collection. Use for queues (FIFO) or stacks (LIFO).

```
RPUSH key value        # Add to right (end)
LPUSH key value        # Add to left (start)
RPOP key               # Remove from right
LPOP key               # Remove from left
LLEN key               # Length
LRANGE key 0 -1        # All elements
LINDEX key 0           # Get by index
LTRIM key 0 1          # Keep only elements 0-1
LPOP key count         # Pop multiple
```

### Use Cases: Queue

```python
# Celery task queue
await redis.rpush("queue:pdf", json.dumps({
    "job_id": 123,
    "input_url": "s3://bucket/file.pdf"
}))

# Worker polls
job_json = await redis.lpop("queue:pdf")
if job_json:
    job = json.loads(job_json)
    process_pdf(job)
```

### Use Cases: Stack

```python
# Undo stack
await redis.rpush("undo:user:5", "action_1")
await redis.rpush("undo:user:5", "action_2")

# Undo
last_action = await redis.rpop("undo:user:5")  # "action_2"
```

## Sets: Unique Collections

Unordered collection of unique values. No duplicates.

```
SADD key member        # Add member
SREM key member        # Remove member
SMEMBERS key           # All members
SISMEMBER key member   # Check membership
SCARD key              # Count members
SINTER key1 key2       # Intersection
SUNION key1 key2       # Union
SDIFF key1 key2        # Difference
```

### Use Cases: Tags, Permissions

```python
# User tags
await redis.sadd("user:5:tags", "python", "celery", "redis")

# Check if user has tag
has_celery = await redis.sismember("user:5:tags", "celery")  # True

# All tags
tags = await redis.smembers("user:5:tags")

# Intersection: users interested in both python AND redis
python_users = await redis.smembers("tag:python")
redis_users = await redis.smembers("tag:redis")
intersection = await redis.sinter("tag:python", "tag:redis")
```

## Sorted Sets: Rankings and Leaderboards

Set with scores. Members are ordered by score.

```
ZADD key score member           # Add with score
ZREM key member                 # Remove
ZSCORE key member               # Get score
ZRANGE key 0 -1 WITHSCORES      # All members, sorted
ZREVRANGE key 0 -1 WITHSCORES   # Reverse order (highest first)
ZRANK key member                # Position (0-indexed)
ZCARD key                        # Count
ZINCRBY key 1 member            # Increment score
ZCOUNT key min max              # Count in range
ZREMRANGEBYSCORE key min max    # Remove by score range
```

### Use Cases: Leaderboards

```python
# Track conversion score per user (most conversions = highest score)
await redis.zadd("leaderboard:2024", {"user:1": 42, "user:5": 85, "user:3": 60})

# Top 10
top_10 = await redis.zrevrange("leaderboard:2024", 0, 9, withscores=True)
# [(b'user:5', 85.0), (b'user:1', 42.0), (b'user:3', 60.0)]

# User's rank
rank = await redis.zrevrank("leaderboard:2024", "user:5")  # Position 0 (first place)

# Increment on completion
await redis.zincrby("leaderboard:2024", 1, "user:5")
```

## Pub/Sub: Message Broadcasting

Publish messages to channels. Subscribers receive them.

```
SUBSCRIBE channel              # Subscribe
PUBLISH channel message         # Publish
UNSUBSCRIBE channel            # Unsubscribe
PSUBSCRIBE pattern             # Pattern subscription
```

### Use Case: Notifications

```python
# Publish
await redis.publish("notifications:user:5", json.dumps({
    "event": "job_completed",
    "job_id": 123,
    "status": "completed"
}))

# Subscribe (async loop)
pubsub = redis.pubsub()
await pubsub.subscribe("notifications:user:5")

async for message in pubsub.listen():
    if message['type'] == 'message':
        event = json.loads(message['data'])
        print(f"Job {event['job_id']} completed")
```

## Transactions (MULTI/EXEC)

Group commands, execute atomically.

```
MULTI              # Start transaction
SET key1 value1    # Queued
INCR key2          # Queued
EXEC               # Execute all-or-nothing
DISCARD            # Cancel
```

### Example

```python
pipe = redis.pipeline(transaction=True)
await pipe.set("user:5:plan", "pro").incr("conversions:pro").execute()
```

## Hands-On Lab

### Lab 2.1: Hash for User Profile

```python
import redis.asyncio as redis
import json

async def test_hash():
    client = await redis.from_url("redis://localhost:6379/0")
    
    # Store user
    user_id = 5
    await client.hset(f"user:{user_id}", mapping={
        "email": "alice@example.com",
        "name": "Alice Smith",
        "plan": "pro",
        "conversions": "0"
    })
    
    # Get specific field
    email = await client.hget(f"user:{user_id}", "email")
    print(f"Email: {email}")
    
    # Update
    await client.hset(f"user:{user_id}", "conversions", "10")
    
    # Increment
    await client.hincrby(f"user:{user_id}", "conversions", 5)
    
    # All fields
    user = await client.hgetall(f"user:{user_id}")
    print(f"User: {user}")
    
    await client.close()

asyncio.run(test_hash())
```

### Lab 2.2: List as Queue

```python
async def test_queue():
    client = await redis.from_url("redis://localhost:6379/0")
    
    # Enqueue jobs
    for i in range(3):
        await client.rpush("queue:pdf", json.dumps({"job_id": i}))
    
    # Dequeue
    while True:
        job_json = await client.lpop("queue:pdf")
        if not job_json:
            break
        job = json.loads(job_json)
        print(f"Processing job {job['job_id']}")
    
    await client.close()

asyncio.run(test_queue())
```

### Lab 2.3: Sorted Set as Leaderboard

```python
async def test_leaderboard():
    client = await redis.from_url("redis://localhost:6379/0")
    
    # Add scores
    await client.zadd("conversions:2024", {
        "user:1": 42,
        "user:5": 100,
        "user:3": 75
    })
    
    # Top 3
    top = await client.zrevrange("conversions:2024", 0, 2, withscores=True)
    print("Top 3:")
    for user, score in top:
        print(f"  {user}: {score}")
    
    # Increment
    await client.zincrby("conversions:2024", 10, "user:1")
    
    await client.close()

asyncio.run(test_leaderboard())
```

## Cheat Sheet: Data Structures

### String

```
SET key value
GET key
INCR counter
APPEND key suffix
```

### Hash

```
HSET user:1 field value
HGET user:1 field
HGETALL user:1
HINCRBY user:1 field 1
```

### List

```
RPUSH queue item
LPOP queue
LRANGE queue 0 -1
LLEN queue
```

### Set

```
SADD tags member
SMEMBERS tags
SISMEMBER tags member
SCARD tags
```

### Sorted Set

```
ZADD leaderboard 100 user1
ZRANGE leaderboard 0 -1 WITHSCORES
ZREVRANGE leaderboard 0 -1 WITHSCORES
ZINCRBY leaderboard 1 user1
```

## Key Takeaways

- **Strings = simple values and counters**
- **Hashes = objects (better than serialized strings)**
- **Lists = queues and stacks**
- **Sets = unique collections and tags**
- **Sorted sets = leaderboards and rankings**
- **Pub/Sub = broadcasting**
- **Choose data structure for your use case**

Module 3 teaches TTL and expiry.
