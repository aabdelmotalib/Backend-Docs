# Module 2: Async Python and Why It Matters

## The Analogy: The Event Loop Chef

Imagine a single chef (the event loop) in a kitchen:

**Sync approach**: Chef picks order 1, cooks the whole dish start to finish without stopping. Takes 10 minutes. Only then picks order 2.
- 4 orders = 40 minutes
- Customers wait forever

**Async approach**: Chef picks order 1, starts cooking, then while it's simmering (waiting), picks order 2, starts it, then picks order 3 while order 2 is waiting. Chef never sits idle.
- 4 orders = 12 minutes (mostly parallel waiting)
- Customers happy

The event loop is the chef, and I/O operations (database queries, HTTP calls) are the "waiting" time.

## What Async/Await Actually Means

### Coroutines

A **coroutine** is a function that can pause and resume:

```python
async def fetch_data():
    print("Starting fetch")
    result = await some_operation()  # Pause here
    print(f"Got {result}")
    return result
```

When we hit `await`, the function pauses and the event loop takes over other work. When the operation completes, it resumes.

### The Event Loop

The event loop is a queue manager:

```
┌─────────────────────────────────────┐
│  Event Loop (running in one thread) │
├─────────────────────────────────────┤
│                                     │
│  coroutine 1: waiting on DB         │
│  coroutine 2: processing request    │
│  coroutine 3: waiting on Redis      │
│  coroutine 4: ready to run          │
│                                     │
│  (rotating between them)            │
└─────────────────────────────────────┘
```

The loop switches between coroutines. If coroutine 1 is waiting on the database, it runs coroutine 4. When the database responds, it switches back to 1.

## Sync vs Async: The Performance Difference

### Sync Version (Blocks)

```python
import time
from fastapi import FastAPI

app = FastAPI()

def slow_operation():
    time.sleep(2)  # Simulate database query
    return "result"

@app.get("/sync")
def sync_endpoint():
    return {"data": slow_operation()}
```

When you call this endpoint:

```
Request arrives
├─ Call slow_operation()
├─ time.sleep(2) blocks the entire worker
├─ Worker is stuck, cannot handle other requests
├─ 2 seconds pass
└─ Response sent

Meanwhile: 5 users waiting for their requests to be processed
```

With 4 Gunicorn workers, you can handle 4 concurrent requests. The 5th waits.

### Async Version (Non-Blocking)

```python
import asyncio
from fastapi import FastAPI

app = FastAPI()

async def slow_operation():
    await asyncio.sleep(2)  # Non-blocking pause
    return "result"

@app.get("/async")
async def async_endpoint():
    return {"data": await slow_operation()}
```

When you call this endpoint:

```
Request 1 arrives
├─ Call slow_operation()
├─ await asyncio.sleep(2) pauses, event loop switches out
│
Request 2 arrives (while Request 1 is paused!)
├─ Call slow_operation()
├─ await asyncio.sleep(2) pauses, event loop switches out
│
... both are waiting in parallel ...
│
2 seconds pass
├─ Both operations complete
└─ Both responses sent

1 worker handles requests simultaneously!
```

With 1 Uvicorn worker, you can handle 100+ concurrent requests.

### The Key Difference

| Sync | Async |
|------|-------|
| `time.sleep(2)` blocks worker | `await asyncio.sleep(2)` releases to event loop |
| Can't handle other requests | Other requests progress meanwhile |
| 4 requests need 8 seconds | 4 requests need 2 seconds (parallelized) |

## asyncio Basics: Coroutines, Tasks, gather

### Creating and Running Coroutines

```python
import asyncio

async def hello():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

# Run a coroutine
asyncio.run(hello())

# Or in FastAPI (event loop already running)
@app.get("/hello")
async def route():
    await hello()
    return {}
```

### Concurrent Execution with gather

To run multiple coroutines concurrently:

```python
async def fetch_user(user_id):
    await asyncio.sleep(1)
    return f"User {user_id}"

async def get_dashboard():
    # Run 3 fetches in parallel
    user, posts, comments = await asyncio.gather(
        fetch_user(1),
        fetch_posts(1),
        fetch_comments(1)
    )
    return {"user": user, "posts": posts, "comments": comments}
```

Without `gather()`, you'd do:

```python
async def get_dashboard():
    user = await fetch_user(1)          # 1 second
    posts = await fetch_posts(1)        # 1 second
    comments = await fetch_comments(1)  # 1 second
    # Total: 3 seconds
```

With `gather()`:

```python
async def get_dashboard():
    user, posts, comments = await asyncio.gather(
        fetch_user(1),
        fetch_posts(1),
        fetch_comments(1)
    )
    # All run in parallel: 1 second total
```

## Async Database Drivers: asyncpg, Motor, etc.

### The Critical Rule

In FastAPI, you **must use async database drivers**:

- **PostgreSQL**: `asyncpg` (not psycopg2)
- **MongoDB**: `motor` (not pymongo)
- **MySQL**: `aiomysql` (not mysql-connector)

### Why

`psycopg2` is synchronous:

```python
import psycopg2

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    conn = psycopg2.connect("dbname=mydb user=postgres")  # Blocks!
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    result = cursor.fetchone()
    return result
```

This blocks the entire event loop. Other requests wait.

`asyncpg` is asynchronous:

```python
import asyncpg

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    conn = await asyncpg.connect("postgresql://user:pass@localhost/mydb")
    result = await conn.fetchrow("SELECT * FROM users WHERE id = $1", user_id)
    await conn.close()
    return result
```

With `await`, the event loop continues working while the database query runs.

## SQLAlchemy AsyncSession

In real projects, you use SQLAlchemy with async:

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

# Create async engine
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/mydb",
    echo=False
)

# Create async session factory
async_session = sessionmaker(
    engine, 
    class_=AsyncSession, 
    expire_on_commit=False
)

# Use in a route
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    async with async_session() as session:
        result = await session.execute(
            select(User).where(User.id == user_id)
        )
        user = result.scalar_one_or_none()
        return user
```

Every interaction with the database is `await`ed. Non-blocking.

## Common Mistake: Calling Sync Code in Async Routes

### The Problem

```python
import time

@app.get("/process")
async def process():
    time.sleep(5)  # WRONG! Blocks the entire event loop
    return {"status": "done"}
```

All requests to your API are now blocked for 5 seconds. This defeats the purpose of async.

### The Fix

```python
import asyncio

@app.get("/process")
async def process():
    await asyncio.sleep(5)  # Correct - non-blocking
    return {"status": "done"}
```

Or:

```python
import asyncio

def slow_sync_operation():
    time.sleep(5)
    return "result"

@app.get("/process")
async def process():
    result = await asyncio.to_thread(slow_sync_operation)
    return {"result": result}
```

`asyncio.to_thread()` runs sync code in a thread pool, preventing blocking.

!!! danger
    If you use sync code in an async route, FastAPI will hang. The event loop gets stuck and cannot process other requests. Always use async drivers or explicitly use `asyncio.to_thread()`.

## Hands-On Lab

### Lab 2.1: Measure Sync vs Async

Create `benchmark.py`:

```python
import asyncio
import time
from fastapi import FastAPI
from concurrent.futures import ThreadPoolExecutor

app = FastAPI()

# Simulate database delay
def slow_db_operation():
    time.sleep(1)
    return "data"

async def slow_async_operation():
    await asyncio.sleep(1)
    return "data"

# Sync endpoint
@app.get("/sync")
def sync_endpoint():
    result = slow_db_operation()
    return {"data": result, "type": "sync"}

# Async endpoint
@app.get("/async")
async def async_endpoint():
    result = await slow_async_operation()
    return {"data": result, "type": "async"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

#### Step 1: Run the app

```bash
pip install fastapi uvicorn
python benchmark.py
```

#### Step 2: Benchmark sync endpoint

```bash
# Make 5 concurrent requests (this will take 5 seconds with sync!)
time (for i in {1..5}; do curl http://localhost:8000/sync & done; wait)
```

#### Step 3: Benchmark async endpoint

```bash
# Make 5 concurrent requests (this will take ~1 second with async!)
time (for i in {1..5}; do curl http://localhost:8000/async & done; wait)
```

You should see:

- Sync: ~5 seconds (one after another)
- Async: ~1 second (all in parallel)

### Lab 2.2: Use asyncio.gather

Add to benchmark.py:

```python
@app.get("/parallel")
async def parallel_endpoint():
    # All 3 run in parallel
    results = await asyncio.gather(
        slow_async_operation(),
        slow_async_operation(),
        slow_async_operation()
    )
    return {"data": results, "count": len(results)}
```

Test:

```bash
time curl http://localhost:8000/parallel
```

Should take ~1 second (not 3).

### Lab 2.3: Async Database Query (Simulated)

```python
import asyncpg
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

# Simulating async database
async def fetch_user_async(user_id: int):
    await asyncio.sleep(0.5)  # Simulate DB query
    return {"id": user_id, "name": f"User {user_id}"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = await fetch_user_async(user_id)
    return user

@app.get("/dashboard/user/{user_id}")
async def dashboard(user_id: int):
    # Fetch user, posts, comments in parallel
    user, posts, comments = await asyncio.gather(
        fetch_user_async(user_id),
        fetch_posts_async(user_id),
        fetch_comments_async(user_id)
    )
    return {
        "user": user,
        "posts": posts,
        "comments": comments
    }

async def fetch_posts_async(user_id: int):
    await asyncio.sleep(0.5)
    return [{"id": 1, "title": "Post 1"}]

async def fetch_comments_async(user_id: int):
    await asyncio.sleep(0.5)
    return [{"id": 1, "text": "Comment 1"}]
```

Test:

```bash
# Get user (0.5 sec)
time curl http://localhost:8000/users/1

# Get dashboard (also ~0.5 sec, all in parallel!)
time curl http://localhost:8000/dashboard/user/1
```

The dashboard should take ~0.5 seconds, not 1.5 seconds, because all three operations run in parallel.

## Cheat Sheet: Async Patterns

### Basic Async Function

```python
async def my_function():
    result = await some_coroutine()
    return result
```

### Run Multiple in Parallel

```python
result1, result2, result3 = await asyncio.gather(
    operation1(),
    operation2(),
    operation3()
)
```

### Handle Errors in Parallel

```python
try:
    results = await asyncio.gather(
        operation1(),
        operation2(),
        return_exceptions=True  # Errors don't stop others
    )
except Exception as e:
    # Handle error
    pass
```

### Wait for Completion with Timeout

```python
try:
    result = await asyncio.wait_for(operation(), timeout=5.0)
except asyncio.TimeoutError:
    # Took too long
    pass
```

### Run Sync Code in Thread Pool

```python
result = await asyncio.to_thread(sync_function, arg1, arg2)
```

## When to Use Async

| Scenario | Use |
|----------|-----|
| Database queries | Always async in FastAPI |
| HTTP calls to other services | Always async (httpx, aiohttp) |
| File I/O | Usually async (aiofiles) |
| Redis operations | Async (redis[asyncio]) |
| Computing (no I/O) | Async def, but doesn't help |
| Executing tasks in parallel | Async with gather/TaskGroup |

## Key Takeaways

- **Async allows the event loop to switch tasks while waiting** for I/O
- **1 Uvicorn worker + async code = 100+ concurrent requests**
- **Must use async database drivers** — asyncpg not psycopg2
- **Never call sync functions that block** — use `asyncio.to_thread()` if necessary
- **`asyncio.gather()`  runs coroutines in parallel**
- **`await` pauses the function**, letting other work happen
- **Type hint with `async def`** — FastAPI knows it's non-blocking

Now Module 3 teaches you to organize your growing codebase with routers and dependency injection.
