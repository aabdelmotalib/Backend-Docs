# Module 4: Atomic Operations and Locks

## The Analogy: Bank Account Withdrawal

Without locks:
- Account has $100
- Alice checks balance: $100
- Bob checks balance: $100 (at same time)
- Alice withdraws $50, balance = $50
- Bob withdraws $50, balance = $50 (should be $0!)
- Result: $100 was subtracted twice from same account

With locks:
- Alice locks account
- Alice checks balance: $100, withdraws $50, balance = $50, unlocks
- Bob waits for lock
- Bob checks balance: $50, withdraws $50, balance = $0, unlocks
- Result: correct

Redis atomic operations execute as single, indivisible unit. No race conditions.

## INCR: Atomic Counter

```
INCR key
INCRBY key amount
DECR key
DECRBY key amount
```

INCR is guaranteed atomic. Even with 1000 concurrent requests, counter is correct:

```python
# Bad: not atomic
count = await redis.get("conversions")
count = int(count) + 1
await redis.set("conversions", count)
# WRONG: if two requests do this simultaneously, one increment is lost

# Good: atomic
await redis.incr("conversions")  # Always correct
```

### Use Case: Hit Counter

```python
@app.get("/document/{doc_id}")
async def get_document(doc_id: int):
    # Atomic increment
    views = await redis.incr(f"document:{doc_id}:views")
    
    document = await db.get(Document, doc_id)
    return {
        "document": document,
        "views": views
    }
```

Million views? No problem. INCR handles it atomically.

## SETNX: Set If Not Exists

```
SETNX key value    # Returns 1 if set, 0 if key already exists
```

Atomic test-and-set. Use for locks:

```python
# Try to acquire lock
locked = await redis.setnx("lock:email_sending", "1")

if locked:
    try:
        # Send emails (only one process does this)
        send_emails_in_batch()
    finally:
        # Release lock
        await redis.delete("lock:email_sending")
else:
    # Another process holds the lock
    print("Email sending already in progress")
```

## Race Condition: Without Locks

```python
# Thread 1                      # Thread 2
conversions = get("user:5")
                                conversions = get("user:5")
conversions += 1
                                conversions += 1
set("user:5", conversions)     # Race: both increment same value
                                set("user:5", conversions)
# Result: lost update
```

Both see "100", increment to "101", lose updates.

## Solution: INCR (Atomic)

```python
await redis.incr("conversions:user:5")  # Atomic, correct every time
```

## Distributed Locks: RedLock

For real services, use RedLock algorithm (Redisson library):

```python
from redlock import RedLock

lock = RedLock(
    key="lock:email_batch",
    masters=[{"host": "localhost", "port": 6379}],
    auto_renewal=True,
    expire_time=30
)

if lock.acquire():
    try:
        # Critical section: only one process here at a time
        send_emails()
    finally:
        lock.release()
else:
    print("Could not acquire lock; another process is sending emails")
```

## GETSET: Atomic Get-and-Set

```
GETSET key newvalue    # Set key to newvalue, return old value
```

Atomic read-then-write. Useful for state transitions:

```python
# Atomic check and update
old_status = await redis.getset(f"job:{job_id}:status", "in_progress")

if old_status == "queued":
    # We successfully transitioned from queued → in_progress
    process_job()
else:
    # Job was already started or completed by another worker
    print(f"Job status was {old_status}, skipping")
```

## MGET/MSET: Atomic Multiple Operations

```
MSET key1 val1 key2 val2 ...    # Set multiple keys atomically
MGET key1 key2 ...               # Get multiple keys
```

All keys set/get together, atomically:

```python
# Set user profile atomically
await redis.mset({
    "user:5:email": "alice@example.com",
    "user:5:name": "Alice",
    "user:5:plan": "pro"
})

# Get all at once
keys = await redis.mget("user:5:email", "user:5:name", "user:5:plan")
# [b'alice@example.com', b'Alice', b'pro']
```

## Atomic Check-Then-Act

Without atomicity (unsafe):

```python
# Bad: not atomic
balance = await redis.get("account:balance")
if int(balance) >= 100:
    await redis.set("account:balance", int(balance) - 100)
    # Another request could withdraw between check and set!
```

Safe pattern with Lua scripting:

```python
# Atomic: check and decrement together
script = """
if redis.call('GET', KEYS[1]) >= tonumber(ARGV[1]) then
    return redis.call('DECRBY', KEYS[1], ARGV[1])
else
    return nil
end
"""

result = await redis.eval(script, 1, "account:balance", 100)
if result is not None:
    print(f"Withdrawal successful, new balance: {result}")
else:
    print("Insufficient funds")
```

## Typical Lock Pattern

```python
import uuid
import time

async def acquire_lock(key: str, timeout: int = 10) -> str:
    """Acquire lock, return lock_id or None if failed"""
    lock_id = str(uuid.uuid4())
    acquired = await redis.set(
        f"lock:{key}",
        lock_id,
        nx=True,  # Only set if doesn't exist
        ex=timeout  # Auto-expire after timeout
    )
    return lock_id if acquired else None

async def release_lock(key: str, lock_id: str) -> bool:
    """Release lock if we still own it"""
    current = await redis.get(f"lock:{key}")
    if current == lock_id:
        await redis.delete(f"lock:{key}")
        return True
    return False

# Usage
lock_id = await acquire_lock("pdf_processing", timeout=30)
if lock_id:
    try:
        # Critical section
        process_pdf()
    finally:
        await release_lock("pdf_processing", lock_id)
else:
    print("Could not acquire lock")
```

## Hands-On Lab

### Lab 4.1: Atomic Counter

```python
import redis.asyncio as redis
import asyncio

async def increment_task(client, task_id):
    """Simulate concurrent increments"""
    for _ in range(100):
        await client.incr("counter")

async def test_atomic_counter():
    client = await redis.from_url("redis://localhost:6379/0")
    await client.set("counter", 0)
    
    # Run 10 tasks concurrently, each incrementing 100 times
    tasks = [increment_task(client, i) for i in range(10)]
    await asyncio.gather(*tasks)
    
    # Check result
    final = await client.get("counter")
    print(f"Final counter: {final}")  # Should be 1000, not less
    print(f"Test passed: {final == b'1000'}")
    
    await client.close()

asyncio.run(test_atomic_counter())
```

### Lab 4.2: Distributed Lock

```python
async def test_lock():
    client = await redis.from_url("redis://localhost:6379/0")
    
    resource = "shared_resource"
    
    async def critical_section(process_id):
        # Try to acquire lock
        lock_id = str(uuid.uuid4())
        acquired = await client.set(
            f"lock:{resource}",
            lock_id,
            nx=True,
            ex=5
        )
        
        if acquired:
            print(f"Process {process_id}: lock acquired")
            try:
                await asyncio.sleep(1)  # Simulate work
                print(f"Process {process_id}: critical section")
            finally:
                await client.delete(f"lock:{resource}")
                print(f"Process {process_id}: lock released")
        else:
            print(f"Process {process_id}: could not acquire lock")
    
    # Multiple processes try to access same resource
    await asyncio.gather(
        critical_section(1),
        critical_section(2),
        critical_section(3)
    )
    
    await client.close()

asyncio.run(test_lock())
```

### Lab 4.3: GETSET State Transition

```python
async def test_getset():
    client = await redis.from_url("redis://localhost:6379/0")
    
    job_id = "job_123"
    
    # Set initial state
    await client.set(f"job:{job_id}:status", "queued")
    
    # Worker tries to transition to in_progress
    old_status = await client.getset(f"job:{job_id}:status", "in_progress")
    
    if old_status == b"queued":
        print(f"Successfully transitioned from {old_status} to in_progress")
    else:
        print(f"Job already in {old_status} state, skipping")
    
    current = await client.get(f"job:{job_id}:status")
    print(f"Current status: {current}")
    
    await client.close()

asyncio.run(test_getset())
```

## Cheat Sheet: Atomic Operations

### Counters

```
INCR key
INCRBY key 5
DECR key
```

### Test-and-Set

```
SETNX key value         # Returns 1 if set, 0 if exists
GETSET key newvalue     # Set and return old value
```

### Multiple Keys

```
MSET key1 val1 key2 val2
MGET key1 key2
```

### Locks

```python
lock_id = str(uuid.uuid4())
acquired = await redis.set("lock:key", lock_id, nx=True, ex=30)
if acquired:
    try:
        # Critical section
    finally:
        await redis.delete("lock:key")
```

## Key Takeaways

- **Race conditions** = lost updates when multiple requesters access same data
- **Atomic operations** = execute as one indivisible unit, no race condition
- **INCR** = always correct counter, even with 1000 concurrent requests
- **SETNX** = atomic test-and-set for locks
- **GETSET** = atomic state transitions
- **Locks with expiry** = prevent deadlocks if process crashes
- **Red/Distributed locks** = for multi-server synchronization

Module 5 teaches Redis as message broker for Celery task queues.
