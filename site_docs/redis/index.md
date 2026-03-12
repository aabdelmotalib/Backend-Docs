# Redis Mastery: Caching, Sessions, and Task Queues

Welcome to Redis. Redis is an in-memory data store. It's **fast** because it keeps data in RAM instead of disk.

## Why Redis for This Project

The PDF processing platform needs:

1. **Fast caching** — Store recent conversions to avoid re-processing
2. **Session storage** — Track which user is logged in (fast TTL expiry)
3. **Task queue** — Queue PDF jobs for Celery workers
4. **Distributed locks** — Prevent duplicate job processing
5. **Rate limiting** — Block users who exceed quota
6. **Atomic counters** — Track pages processed per month

Redis does all six better than PostgreSQL because it's in-memory.

## When to Use Redis vs PostgreSQL

| Need | PostgreSQL | Redis |
|------|-----------|-------|
| Persistent data (users, jobs) | ✅ | ❌ |
| Transactions/ACID | ✅ | ❌ |
| Complex queries | ✅ | ❌ |
| Cache (fast read/write) | ❌ | ✅ |
| Session storage (with TTL) | ❌ | ✅ |
| Task queue | ❌ | ✅ |
| Rate limiting (counters) | ❌ | ✅ |

**Truth**: Use both. PostgreSQL for forever data, Redis for temporary data.

## Redis in the Architecture

```
┌─────────────────────────────────┐
│   FastAPI App                   │
├─────────────────────────────────┤
│  ┌──────────────┐               │
│  │   Redis      │ ← Session tokens, rate limits, cache
│  └──────────────┘ ← Task queue for Celery
│                                 │
│  ┌──────────────┐               │
│  │ PostgreSQL   │ ← Users, subscriptions, jobs
│  └──────────────┘                │
└─────────────────────────────────┘
     ↓
┌─────────────────────────────────┐
│   Celery Worker                 │
│   (processes PDF jobs)          │
│   Reads from Redis queue        │
│   Writes back to PostgreSQL     │
└─────────────────────────────────┘
```

## What You'll Learn

5 modules covering Redis fundamentals:

1. **What is Redis** — In-memory store, data types, persistence
2. **Data Structures** — Strings, lists, hashes, sets, sorted sets
3. **TTL and Expiry** — Key expiration, session timers
4. **Atomic Operations** — Race condition prevention, distributed locks
5. **Redis as Message Broker** — Celery queue mechanism

## Module Dependencies

```
01 → 02 → 03 → 04 ↘
               05
```

- **Module 1** explains why Redis is fast
- **Module 2** teaches data types
- **Module 3** teaches expiration (critical for sessions)
- **Module 4** ensures concurrent access is safe
- **Module 5** uses it as Celery broker

## Redis vs Memcached

Both cache, but:

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data types | Strings, lists, hashes, sets | Strings only |
| Persistence | Yes (optional) | No |
| Transactions | Yes | No |
| Pub/Sub | Yes | No |
| Task queue | Yes (Celery broker) | No |

Redis is more powerful. Use Redis.

## Architecture Patterns

### Pattern 1: Cache-Aside

```
GET user_profile(id=5)
  ├─ Check Redis
  │  └─ HIT? Return from Redis
  ├─ MISS? Query PostgreSQL
  │  └─ Store in Redis with TTL
  └─ Return to user
```

Fast if cached. Falls back to database.

### Pattern 2: Celery Task Queue

```
FastAPI          Redis            Celery Worker
  │                 │                    │
  ├─ task.delay() → Enqueue job         │
  │                 ├─ Worker polls →   │
  │                 │                  Process
  │                 │                    │
  │                 ← Result ← ────────┘
  ← Return task_id
```

Background processing without blocking API.

### Pattern 3: Session Storage

```
/login endpoint
  ├─ Verify password
  ├─ Create token
  ├─ Store in Redis: token → user_id (TTL: 24h)
  └─ Return token

/profile endpoint
  ├─ Get token from request
  ├─ Lookup in Redis
  │  └─ Token → user_id
  └─ Fetch user details from PostgreSQL
```

Fast session lookup. Automatic expiry.

## What You'll Build

In labs, you'll:

1. Connect to Redis with redis-py and asyncpg
2. Cache PDF conversion results
3. Implement session storage with TTL
4. Prevent duplicate job processing with locks
5. Build task queue for Celery (tested in Celery module)

## Prerequisites

- **Python 3.12+** with pip
- **Docker** — to run Redis container
- **Redis client library** — redis[asyncio]
- **Understanding of async** — FastAPI Module 2
- **Understanding of Celery basics** — covered in Celery module

## Structure

Each module:

1. **Concept** — Analogy first
2. **Technical Depth** — Redis commands and data structures
3. **Real Code** — PDF SaaS examples
4. **Hands-On Lab** — Try it yourself
5. **Cheat Sheet** — Quick reference

Let's start with what makes Redis fast.
