# Module 6: Transactions and Concurrency

## The Analogy: Bank Account

Alice and Bob each have $100. The bank balance is $200.

**Without transactions:**

Alice withdraws $50:
1. Read balance: 200
2. Calculate new balance: 200 - 50 = 150
3. **Bank system fails** — process freezes
4. Bob withdraws $50:
   1. Read balance: 200 (system didn't update!)
   2. Calculate: 200 - 50 = 150
5. System recovers. Final balance: 150 (should be 100!)

Alice's $50 disappeared.

**With transactions:**

A transaction is atomic: all-or-nothing.

Alice's withdrawal:
1. START TRANSACTION
2. Read balance: 200
3. Deduct 50: 150
4. **COMMIT** (officially recorded)

Now Bob withdraws:
1. START TRANSACTION
2. Read balance: 150 (sees Alice's change)
3. Deduct 50: 100
4. COMMIT

Final balance: 100. Correct!

## ACID Properties

Transactions ensure **ACID**:

### Atomicity

All-or-nothing. Either:
- All operations complete
- None of them complete

```python
async with session.begin():
    user = await session.get(User, 1)
    user.balance -= 100
    
    other = await session.get(User, 2)
    other.balance += 100
    
    await session.flush()  # If error here, both rollback
    # Both complete or none do
```

### Consistency

Database rules are always satisfied. Constraints are never violated.

```sql
-- If FOREIGN KEY constraint exists:
-- Cannot INSERT job with non-existent subscription_id
-- Cannot DELETE subscription with existing jobs (if ON DELETE RESTRICTED)
```

### Isolation

Concurrent transactions don't interfere.

```
Transaction 1: Read Account A (balance: 100)
  Transaction 2: Read Account B (balance: 100)
  Transaction 2: Update Account B (+50, now 150)
  Transaction 2: Commit
Transaction 1: Read Account B (sees 150, not original 100)
```

### Durability

Once committed, data is permanent even if system crashes.

```python
await session.commit()
# Even if power cuts off now, data is saved
```

## SQLAlchemy Transactions

### Implicit Transactions

```python
async with async_session() as session:
    user = await session.get(User, 1)
    user.balance -= 100
    await session.commit()  # Automatic transaction
```

### Explicit Transactions

```python
async with async_session() as session:
    async with session.begin():
        # START TRANSACTION

        user = await session.get(User, 1)
        user.balance -= 100
        
        other = await session.get(User, 2)
        other.balance += 100
        
        # Implicit COMMIT on exit
        # If exception, automatic ROLLBACK
```

### Rollback on Error

```python
async with async_session() as session:
    async with session.begin():
        user = await session.get(User, 1)
        user.balance -= 100
        
        if user.balance < 0:
            raise ValueError("Insufficient funds")
        
        # If exception raised, entire transaction rolls back
        # user.balance -= 100 is undone
```

## Isolation Levels

Different isolation levels balance consistency vs performance:

### Read Uncommitted (Dangerous)

See uncommitted data from other transactions. Not recommended.

```
Transaction 1: Balance = 100
  Transaction 2: Balance = 100 - 50 = 50 (not committed)
Transaction 1: Reads balance as 50 (sees uncommitted change!)
  Transaction 2: Fails and rolls back
Transaction 1: Balance actually 100, but thought it was 50 (inconsistent!)
```

### Read Committed (Default, Good)

Only see committed data.

```
Transaction 1: Balance = 100
  Transaction 2: Balance = 100 - 50 = 50 (not committed)
Transaction 1: Reads balance as 100 (doesn't see uncommitted)
  Transaction 2: Commit → balance is 50
Transaction 1: Reads balance again as 50 (sees committed)
```

Safe. Allows some concurrency.

### Repeatable Read

Snapshot isolation. See consistent view throughout transaction.

```
Transaction 1: First read balance = 100
  Transaction 2: Update balance to 50, commit
Transaction 1: Second read balance = 100 (still sees original)
```

Very safe. Slower.

### Serializable

Transactions run as if serial (one after another). Safest but slowest.

PostgreSQL default is **Read Committed**. Usually sufficient.

## Preventing Race Conditions

### The Lost Update Problem

```
User 1:                           User 2:
1. Read balance: 100
                                  1. Read balance: 100
2. Add 50: 150
                                  2. Add 50: 150
3. Commit: balance = 150
                                  3. Commit: balance = 150

Expected: 200. Got: 150. User 1's +50 lost!
```

### Solution 1: Explicit Locking

```python
async with session.begin():
    # Lock the row so others wait
    user = await session.execute(
        select(User).where(User.id == 1).with_for_update()
    )
    user = user.scalar_one()
    user.balance += 50
```

Now:

```
User 1: Lock row, read balance (100), add 50, commit. Release lock.
User 2: Wait for lock... lock acquired. Read balance (150), add 50, commit.

Result: 200. Correct!
```

### Solution 2: Atomic Operations (No Locking)

```python
async with session.begin():
    # Atomic increment
    await session.execute(
        update(User).where(User.id == 1).values(balance=User.balance + 50)
    )
```

PostgreSQL handles atomicity without locking due to MVCC (Multi-Version Concurrency Control).

## Deadlocks

A **deadlock** happens when transactions wait for each other:

```
Transaction 1: Lock User 1
  Transaction 2: Lock User 2
Transaction 1: Wait for User 2 (held by Txn 2)
  Transaction 2: Wait for User 1 (held by Txn 1)

Deadlock! Both wait forever.
```

PostgreSQL detects deadlocks and aborts one:

```
psycopg2.errors.InFailedSqlTransaction: current transaction is aborted due to conflict
```

Fix: Retry the transaction.

```python
import asyncio

async def transfer(user1_id, user2_id, amount):
    max_retries = 3
    for attempt in range(max_retries):
        try:
            async with async_session().begin() as session:
                # Deadlock might occur here
                user1 = await session.execute(
                    select(User).where(User.id == user1_id).with_for_update()
                )
                user2 = await session.execute(
                    select(User).where(User.id == user2_id).with_for_update()
                )
                # ... do transfer
                return True
        except Exception as e:
            if "deadlock" in str(e) and attempt < max_retries - 1:
                await asyncio.sleep(0.1 * (2 ** attempt))  # Exponential backoff
                continue
            raise
```

## Savepoints

Partial rollback within transaction:

```python
async with session.begin():
    user = await session.get(User, 1)
    user.balance -= 100
    
    savepoint = await session.begin_nested()
    try:
        # Risky operation
        user.balance -= 50
        await savepoint.commit()
    except Exception:
        await savepoint.rollback()  # Only undo the -50
        # The -100 still applies
```

## Real PDF SaaS Transactions

```python
@app.post("/process-job")
async def process_job(
    job_id: int,
    current_user: int = Depends(get_current_user),
    session: AsyncSession = Depends(get_session)
):
    """Atomically: update job status, deduct quota, log activity"""
    
    async with session.begin():
        # Get job and subscription
        job = await session.execute(
            select(Job)
            .where(Job.id == job_id)
            .with_for_update()  # Lock to prevent race
        )
        job = job.scalar_one()
        
        sub = await session.execute(
            select(Subscription)
            .where(Subscription.id == job.subscription_id)
            .with_for_update()
        )
        sub = sub.scalar_one()
        
        # Check quota
        if sub.pages_used + job.page_count > sub.pages_per_month:
            raise HTTPException(status_code=429, detail="Quota exceeded")
        
        # Update job
        job.status = "processing"
        job.started_at = datetime.utcnow()
        
        # Deduct quota
        sub.pages_used += job.page_count
        
        # Log activity
        log = ActivityLog(subscription_id=sub.id, action="job_started")
        session.add(log)
        
        # All-or-nothing: if any operation fails, all rollback
        # If power cuts off after COMMIT, all are saved
        
        await session.flush()
    
    return {"status": "processing", "job_id": job_id}
```

## Hands-On Lab

### Lab 6.1: Observe Race Condition

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy import Column, Integer, String, select, update
from sqlalchemy.orm import declarative_base
import asyncio

Base = declarative_base()

class Account(Base):
    __tablename__ = "accounts"
    id = Column(Integer, primary_key=True)
    name = Column(String(50))
    balance = Column(Integer, default=0)

engine = create_async_engine("sqlite+aiosqlite:///:memory:")
AsyncSession = sessionmaker(engine, class_=AsyncSession)

async def init():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    async with AsyncSession() as session:
        session.add(Account(name="Alice", balance=100))
        await session.commit()

async def read_modify_write(name, amount):
    """UNSAFE: 3 separate operations"""
    async with AsyncSession() as session:
        acc = await session.get(Account, 1)
        balance = acc.balance
        await asyncio.sleep(0.1)  # Simulate processing
        acc.balance = balance + amount
        await session.commit()

async def atomic_add(amount):
    """SAFE: atomic operation"""
    async with AsyncSession() as session:
        await session.execute(
            update(Account).values(balance=Account.balance + amount)
        )
        await session.commit()

async def test():
    await init()
    
    # Race condition
    await asyncio.gather(
        read_modify_write("Thread1", 50),
        read_modify_write("Thread2", 50)
    )
    
    async with AsyncSession() as session:
        acc = await session.get(Account, 1)
        print(f"After unsafe adds: {acc.balance}")  # Expected 200, got 150!
```

### Lab 6.2: Explicit Locking

```python
async def transfer_with_lock(from_id, to_id, amount):
    """With explicit locking"""
    async with AsyncSession() as session:
        async with session.begin():
            # Lock both accounts
            from_acc = await session.execute(
                select(Account)
                .where(Account.id == from_id)
                .with_for_update()
            )
            from_acc = from_acc.scalar_one()
            
            to_acc = await session.execute(
                select(Account)
                .where(Account.id == to_id)
                .with_for_update()
            )
            to_acc = to_acc.scalar_one()
            
            # Transfer
            from_acc.balance -= amount
            to_acc.balance += amount
```

## Cheat Sheet: Transactions

### Basic Transaction

```python
async with session.begin():
    user = await session.get(User, 1)
    user.balance -= 100
    # Auto-commit on context exit
    # Auto-rollback on exception
```

### Explicit Lock

```python
async with session.begin():
    user = await session.execute(
        select(User).where(User.id == 1).with_for_update()
    )
    user = user.scalar_one()
    user.balance += 50
```

### Savepoint (Partial Rollback)

```python
async with session.begin():
    user.balance -= 100
    
    sp = await session.begin_nested()
    try:
        risky_operation()
        await sp.commit()
    except:
        await sp.rollback()
```

## Key Takeaways

- **Transactions group operations** — all-or-nothing
- **ACID guarantees data safety** — atomicity, consistency, isolation, durability
- **Race conditions without isolation** — multiple users modify same row
- **Locking prevents race conditions** — `with_for_update()`
- **Atomic operations** — update/increment without explicit locks
- **Deadlocks happen** — retry with exponential backoff
- **Savepoints allow partial rollback** — recover from failures within transaction

Module 7 teaches production setup for PostgreSQL.
