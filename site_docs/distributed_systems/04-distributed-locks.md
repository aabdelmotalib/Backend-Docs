# Module 4: Distributed Locks

## The Problem: Race Conditions at Scale

Single machine:
```python
# Python mutex
lock = threading.Lock()

with lock:
    balance = db.query(Account).get(account_id).balance
    if balance >= 100:
        balance -= 100
        db.commit()
```

Mutex ensures only one thread executes at a time. Safe.

Distributed (multiple API instances):

```
API Instance 1:              API Instance 2:
GET balance = 150           GET balance = 150
Check: 150 >= 100 ✓         Check: 150 >= 100 ✓
Deduct: 150 - 100 = 50      Deduct: 150 - 100 = 50
UPDATE balance = 50         UPDATE balance = 50

Final balance: 50 (should be -50!)
Both withdrew 100 from the same account
```

Mutex only works on one machine. Need lock across machines.

## Locks in Distributed Systems

### Lock Requirements

1. **Mutual exclusion**: Only one process can hold lock at a time
2. **Deadlock protection**: Lock must eventually release
3. **Minimal congestion**: Process shouldn't wait longer than necessary

### Redis-based Lock (Simple)

```python
import redis
import uuid

redis_client = redis.Redis(host='redis')

def acquire_lock(key, timeout=30):
    """Try to acquire lock, return True if success"""
    lock_value = str(uuid.uuid4())  # Unique value
    success = redis_client.set(key, lock_value, nx=True, ex=timeout)
    return success, lock_value

def release_lock(key, lock_value):
    """Release lock only if we hold it"""
    # Read current value
    current_value = redis_client.get(key)
    
    # Only delete if we own it (prevent releasing others' locks)
    if current_value == lock_value:
        redis_client.delete(key)

# Usage
@app.post("/api/accounts/withdraw")
def withdraw(account_id, amount):
    lock_key = f"account_lock:{account_id}"
    
    # Try to acquire lock
    acquired, lock_value = acquire_lock(lock_key, timeout=30)
    if not acquired:
        raise HTTPException(status_code=429, detail="Account locked, try again")
    
    try:
        # Critical section: access shared resource
        account = db.query(Account).filter_by(id=account_id).first()
        if account.balance >= amount:
            account.balance -= amount
            db.commit()
            return {"status": "withdrawn", "new_balance": account.balance}
        else:
            raise HTTPException(status_code=400, detail="Insufficient funds")
    finally:
        # Always release  lock
        release_lock(lock_key, lock_value)
```

Problems:
- If API instance crashes before finally block, lock stays (blocked 30 seconds)
- Lock value comparison is not atomic with delete (race on `current_value == lock_value` vs delete)

### Redlock Algorithm (Robust)

Redlock is Redis' author's (antirez) solution for robust distributed locks.

Requires multiple Redis instances:

```
Redis-1
Redis-2
Redis-3
```

Algorithm:
1. Acquire lock on all 3 instances
2. If majority (2/3) succeed, lock acquired
3. Release requires deleting from all 3

**Benefit**: survives single Redis crash.

Implementation (use existing library):

```python
from redis import Redis
from redlock import Redlock

redis_instances = [
    Redis(host='redis-1'),
    Redis(host='redis-2'),
    Redis(host='redis-3')
]

dlm = Redlock(redis_instances, auto_release_time=30000)  # 30 second TTL

@app.post("/api/accounts/withdraw")
def withdraw(account_id, amount):
    lock_acquired = dlm.lock(f"account:{account_id}", 30000)  # 30s TTL
    
    if not lock_acquired:
        raise HTTPException(status_code=429, detail="Locked, try again")
    
    try:
        account = db.query(Account).filter_by(id=account_id).first()
        account.balance -= amount
        db.commit()
    finally:
        dlm.unlock(f"account:{account_id}")
```

## When Not to Use Locks

**Locks are expensive**. Avoid if possible.

### Example 1: Database-Level Constraint

```python
# Instead of lock, use database transaction

@app.post("/api/subscribe")
def subscribe(user_id):
    # Database handles atomicity
    with db.begin():
        # If user already subscribed, this fails
        subscription = Subscription(user_id=user_id)
        db.add(subscription)
        db.commit()  # ACID: either both inserted, or neither
```

Database ACID transactions are usually sufficient.

### Example 2: Database Pessimistic Lock (SELECT FOR UPDATE)

```python
# Instead of Redis lock, use database row lock

def withdraw_safe(account_id, amount):
    with db.begin():
        # Lock the row, no other transaction can modify
        account = db.query(Account) \
            .filter_by(id=account_id) \
            .with_for_update() \
            .first()
        
        if account.balance >= amount:
            account.balance -= amount
        # Commit releases lock
```

PostgreSQL's FOR UPDATE is simpler, no Redis needed.

### Example 3: Optimistic Locking (Version Numbers)

```python
# Instead of locks, use version field

class Account(Base):
    __tablename__ = "accounts"
    id: UUID
    balance: int
    version: int  # Increment on each change

def withdraw_optimistic(account_id, amount):
    account = db.query(Account).filter_by(id=account_id).first()
    original_version = account.version
    
    if account.balance >= amount:
        account.balance -= amount
        
        # Update only if version unchanged
        # If another process updated, version differs, update fails
        result = db.execute(
            text("""
                UPDATE accounts
                SET balance = :new_balance, version = version + 1
                WHERE id = :id AND version = :version
            """),
            {
                "new_balance": account.balance,
                "id": account_id,
                "version": original_version
            }
        )
        
        if result.rowcount == 0:
            raise HTTPException(status_code=409, detail="Conflict, retry")
```

No lock acquired, but if collision happens, client retries.

## Lock Expiry: The Danger

Redis lock with 30s expiry:

```
08:00:00 - Process A acquires lock
08:00:15 - Process A doing critical work (still within 30s)
08:00:20 - Process A does 10-second I/O operation (filesystem, network)
08:00:30 - Lock expires automatically!
08:00:31 - Process B acquires same lock
08:00:35 - Both A and B modifying shared resource (race condition!)
08:00:40 - Process A finishes, releases lock
```

**Problem**: Lock expiry assumes processing time is predictable. It's not.

### Fencing Tokens

Solution: fencing tokens, a monotonically increasing number.

```python
import time

class FencedLock:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.counter = 0  # Increases every lock
    
    def acquire_lock(self, resource, timeout=30):
        self.counter += 1
        token = self.counter
        
        # Store counter value in Redis
        lock_acquired = self.redis.set(
            f"{resource}:token",
            token,
            nx=True,
            ex=timeout
        )
        
        return lock_acquired, token

def modify_resource(resource, token):
    # Before modifying, verify we hold the lock
    current_token = redis.get(f"{resource}:token")
    
    if int(current_token) != token:
        # Lock expired and was acquired by another process
        # Do NOT modify (would violate consistency)
        raise Exception("Lock lost, aborting")
    
    # Modify resource
    db.query(Resource).filter_by(id=resource).update(...)
```

This prevents automatic (silent) race conditions.

## Hands-On Lab

### Lab 4.1: Simple Redis Lock

```python
import redis
import threading
import time

redis_client = redis.Redis(host='localhost')

def withdraw_without_lock(account_id):
    """Demonstrates race condition"""
    balance = redis_client.get(f"account:{account_id}") or 0
    balance = int(balance)
    
    time.sleep(0.1)  # Simulate processing
    
    balance -= 10
    redis_client.set(f"account:{account_id}", balance)

# Start with 100
redis_client.set("account:1", 100)

# Two threads withdraw simultaneously
t1 = threading.Thread(target=withdraw_without_lock, args=(1,))
t2 = threading.Thread(target=withdraw_without_lock, args=(1,))

t1.start()
t2.start()
t1.join()
t2.join()

# Final balance
print(redis_client.get("account:1"))  # Should be 80, but might be 90!
```

### Lab 4.2: With Redis Lock

```python
import redis
import uuid
import threading

redis_client = redis.Redis(host='localhost')

def withdraw_with_lock(account_id):
    """Demonstrates lock prevention"""
    lock_key = f"lock:{account_id}"
    lock_value = str(uuid.uuid4())
    
    # Acquire lock
    acquired = redis_client.set(lock_key, lock_value, nx=True, ex=30)
    
    if not acquired:
        print("Could not acquire lock, skipping")
        return
    
    try:
        balance = int(redis_client.get(f"account:{account_id}") or 0)
        
        time.sleep(0.1)  # Simulate processing
        
        balance -= 10
        redis_client.set(f"account:{account_id}", balance)
    finally:
        # Release lock
        current = redis_client.get(lock_key)
        if current == lock_value:
            redis_client.delete(lock_key)

# Start with 100
redis_client.set("account:1", 100)

# Two threads with lock
t1 = threading.Thread(target=withdraw_with_lock, args=(1,))
t2 = threading.Thread(target=withdraw_with_lock, args=(1,))

t1.start()
t2.start()
t1.join()
t2.join()

# Final balance
print(redis_client.get("account:1"))  # Now guaranteed 80
```

## Cheat Sheet: Distributed Locks

```python
# Simple Redis lock
lock_acquired = redis.set(key, value, nx=True, ex=30)
if lock_acquired:
    try:
        critical_section()
    finally:
        redis.delete(key)

# Database lock (better for single database)
with db.begin():
    row = db.query(Table).filter_by(id=id).with_for_update().first()
    modify_row(row)
    # Lock released on commit

# Optimistic locking (no lock acquired)
version = row.version
if row.balance > 0:
    row.balance -= amount
    db.execute(
        text("UPDATE table SET balance=:b, version=version+1 WHERE id=:id AND version=:v"),
        {"b": row.balance, "id": row.id, "v": version}
    )
```

## Key Takeaways

- **Race conditions** happen when multiple processes modify shared data
- **Redis locks** use atomic SET with nx=True (not exists)
- **Database locks** use FOR UPDATE (simpler, no Redis needed)
- **Pessimistic** (acquire lock) vs **optimistic** (version + retry)
- **Lock expiry** must account for processing time (use fencing tokens)
- **Redlock** survives single Redis crash (requires 3 instances)
- **Prefer database constraints** (UNIQUE, ACID) over locks

Module 5 teaches idempotency, the key to safe retries.
