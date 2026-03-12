# Distributed Systems: Resilience at Scale

## What This Section Covers

Your PDF platform is already distributed:
- **API** runs on multiple instances (load balanced)
- **Database** is a single PostgreSQL (but replicated in production)
- **Cache** is Redis (distributed state)
- **Task queue** is Celery (distributed async work)
- **Storage** is MinIO (distributed object storage)

This section teaches the principles that make systems like this reliable.

## The Challenges

Distributed systems face problems single-machine systems don't:

1. **Partial failures** — one service fails, others don't; clients don't know who failed
2. **Network unreliability** — TCP guarantees delivery, but only within a connection; network partitions happen
3. **Latency** — calls between services take time; timeout decisions are hard
4. **Consistency** — data replicas may disagree; when to sync?
5. **Messaging** — delivery guarantees matter; "at least once" vs "exactly once"

## The 8 Modules

| # | Module | Why It Matters |
|---|--------|---|
| 1 | What is a Distributed System? | Understand fundamental challenges & fallacies |
| 2 | Message Queues & Async | Decouple services; handle spikes |
| 3 | The CAP Theorem | Choose your trade-off (consistency vs availability) |
| 4 | Distributed Locks | Prevent concurrent modifications |
| 5 | Idempotency | Make retries safe |
| 6 | Eventual Consistency | Sync data across replicas |
| 7 | Observability | Debug production issues |

## Learning Path

1. **Module 1** — understand the [definition](#) + [fallacies](#)
2. **Module 2** — message queues are everywhere (Celery, RabbitMQ, Kafka); delivery matters
3. **Module 3** — CAP theorem constrains what's possible
4. **Module 4** — locks prevent races; distributed locks are hard
5. **Module 5** — idempotency keys + UNIQUE constraints = reliability
6. **Module 6** — data doesn't instantly sync; understand the gap
7. **Module 7** — you can't fix what you can't see

## PDF Platform Architecture

Your platform shows all these concepts:

```
User (1)
  ↓
Load Balancer → Nginx
  ↓
API Instance 1, 2, 3 (stateless)
  ↓
PostgreSQL (strong consistency, single write)
  ↓
Redis (eventual consistency cache)
  ↓
Celery Workers (message queue: at-least-once)
  ↓
MinIO (distributed storage)
  ↓
ClamAV (external service)
```

- **PostgreSQL** = strong consistency (ACID)
- **Redis** = weak consistency (eventual)
- **Celery** = at-least-once delivery (retries create duplicates)
- **API instances** = stateless (no affinity required)
- **MinIO** = replicated storage (availability vs consistency trade-off)

Each component makes specific CAP choices.

## Prerequisites

You should understand:
- Module 3 from Networking (how services communicate)
- Module 1 from Security (authentication, authorization)
- FastAPI basics (from the Platform section)
- PostgreSQL transactions (from the Database section)
- Redis (from the Cache section)
- Celery (from the Background Jobs section)

## Key Assumptions

1. **Services fail** — always design for partial failures
2. **Networks partition** — treats the scenario where group A talks to B, but not C
3. **Latency is variable** — don't assume timeouts are constant
4. **Clocks drift** — timestamps may be out of sync; use logical clocks
5. **Humans make mistakes** — monitoring catches errors logs don't
6. **Data is valuable** — losing even one byte is expensive

This section teaches patterns to handle failure.

---

**Next**: Module 1 defines what "distributed" means. Start there.
