# Module 1: What Is a Distributed System?

## The Analogy: A Restaurant Chain

Single-machine system = one restaurant location.
- All ordering, cooking, serving in one room
- If kitchen burns, entire restaurant goes down
- Easy to manage ("ask the chef")

Distributed system = restaurant chain with multiple locations.
- Locations are independent
- If Manhattan location fails, Brooklyn still serves customers
- Harder to manage ("update all locations about special menu items")

Your PDF platform is a chain. The API instances are locations. Data (PostgreSQL) is the central warehouse.

## Definition

**Distributed system**: Multiple computers working together, appearing to users as one logical system.

Key characteristics:
- **Multiple processes** — running on different machines (API instances, workers)
- **Network communication** — between processes (HTTP, message queues)
- **Shared goal** — serve user requests, process tasks
- **Partial failures** — one process can fail without stopping others

## Single-Machine vs Distributed

### Single Machine (Monolith)

```
┌─────────────────┐
│   FastAPI API   │
│   PostgreSQL    │
│   Redis         │
│   Celery        │
└─────────────────┘
(one server)
```

Advantages:
- Simple
- Low latency (same machine)
- Transactions are atomic (ACID)

Disadvantages:
- Cannot scale horizontally
- Single point of failure
- One bad query blocks everything

### Distributed (Your Platform)

```
                  Load Balancer
                    ↙    ↙    ↘
        ┌───────────────────────┐
        ↓           ↓           ↓
     API-1      API-2      API-3
        ↓           ↓           ↓
        └───────────────────────┘
                    ↓
            ┌───────────────┐
            │  PostgreSQL   │
            │  Redis        │
            │  MinIO        │
            └───────────────┘
                    ↓
            ┌───────────────┐
            │ Celery        │
            │ Workers 1-10  │
            └───────────────┘
```

Advantages:
- Scale horizontally (add API instances)
- Fault tolerance (one API dies, others continue)
- Load balancing (spread requests)

Disadvantages:
- Complex debugging
- Network delays
- Potential inconsistency
- Version mismatches (old API, new DB schema)

## Partial Failures

The core challenge: part of the system fails, not all.

Your API instances are fine, but PostgreSQL crashes.
- API can respond (if it caches data)
- But new writes are stuck
- Is the database down or just slow?
- After 30 seconds, timeout. Return 503 to user. Is the user retrying? Does the operation partially complete?

Single machine: if kernel panics, everything stops. Clean failure.
Distributed: ambiguous failures. Is the service down? Did my request go through?

## The Fallacies of Distributed Computing

Eight assumptions programmers wrongly make:

### Fallacy 1: The Network Is Reliable

Reality: Packets drop. Cables cut. ISPs make mistakes.

Your assumption:
```python
response = requests.get("http://postgres:5432")  # Always works
```

Reality:
```python
# Network partition for 5 seconds
# Timeout? Did query execute?
# Client: failed to connect
# Server: executed query, tried to respond, client gone
```

### Fallacy 2: Latency Is Zero

Reality: Network round trips take time.

Your assumption:
```python
# 1ms + 1ms for get from cache + 2ms for put = 4ms per request
result = redis.get("key")
reddit.set("key_new", process(result))
```

Reality at 10k requests/second:
- Some requests take 100ms (network jitter)
- Some cache hosts are geographically distant
- Some requests are slow (disk I/O, Full GC pause)

### Fallacy 3: Bandwidth Is Unlimited

Reality: Your network has limits.

Assumption:
```python
# Download 10GB file and process
response = requests.get("https://s3.example.com/huge_file.zip")
```

Reality:
- 1Gbps connection = 125 MB/s throughput
- 10GB = 80 seconds minimum
- What if connection drops halfway?

### Fallacy 4: The Network Is Secure

Reality: Unencrypted traffic is visible.

Assumption:
```python
# JWT in header, no HTTPS
Authorization: Bearer eyJ...  # Readable in plaintext
```

Reality: Man-in-the-middle attacker reads token, impersonates user.

**Always use HTTPS for all internal + external traffic.**

### Fallacy 5: Topology Doesn't Change

Reality: Servers go down, get replaced, IP changes.

Assumption:
```
api.example.com always resolves to 10.0.0.5
```

Reality:
- Server replacement: IP changes to 10.0.0.6
- DNS cache stale: old clients connect to .5 (replaced)
- Load balancer replaces servers: client still talks to .5

Solution: Service discovery (Kubernetes, Consul). Discover services dynamically.

Your platform uses docker-compose DNS: `postgres:5432` resolves dynamically within the network.

### Fallacy 6: There Is One Administrator

Reality: Multiple teams, domains, companies.

Assumption:
```
"I'll update the API and database schema together"
```

Reality:
```
Team A (API): "We'll deploy Thursday"
Team B (DB): "We're frozen due to migration Friday"
Thursday: API tries new schema, database doesn't have it → errors
```

Solution: Backwards-compatible changes. Old API must work with new schema for some time, and vice versa.

### Fallacy 7: Transport Cost Is Zero

Reality: Network bandwidth costs money.

Assumption:
```python
# For every page view, fetch 100 related items
for item_id in related_ids:
    item = requests.get(f"http://api/items/{item_id}")
    # 100 round trips per page view
```

Reality: At 1M page views/day = 100M API calls =expensive.

Solution: Batch requests. Return 100 items in one request.

### Fallacy 8: The Network Is Homogeneous

Reality: Different protocols, vendors, versions.

Assumption:
```
"All services use gRPC with protobuf"
```

Reality:
```
API: Python gRPC
Worker: Node.js gRPC
Legacy service: JSON over HTTP
Payment service (Paymob): REST API
```

Solution: Standardize on HTTP+JSON. It's slow, but compatible. Or gRPC within your infrastructure.

## Challenges

Three fundamental challenges in distributed systems:

### 1. Unreliable Communication

```
You send: "Process payment"
Network drops
Server received it, processed, tried to respond
You never get response
Do you retry? Was it processed twice?
```

Solution: Idempotent operations (Module 5).

### 2. Partial Failures

```
API → Database (connection succeeds)
API sends query
Database crashes before responding
API timeout after 30s
Did the query execute? Unknown state.
```

Solution: Timeouts + retries (but retries need idempotency).

### 3. Inconsistency

```
API writes to PostgreSQL (immediate consistency)
Redis cache is stale for 5 seconds
User gets old data from cache
```

Solution: Eventual consistency models (Module 6).

## Your Platform's Consistency Model

### Strong Consistency (PostgreSQL)

```python
# User edits document
@app.put("/api/documents/{id}")
def edit_document(doc_id: str, content: str):
    db.query(Document).filter_by(id=doc_id).update({"content": content})
    db.commit()  # Immediately durable
    return {"status": "saved"}
```

Next read always sees latest data.

### Eventual Consistency (Redis)

```python
# Cache hit
cache_hit = redis.get(f"document:{id}")
if cache_hit:
    return json.loads(cache_hit)  # May be stale

# Cache miss, fetch from DB and update cache
doc = db.query(Document).filter_by(id=id).first()
redis.set(f"document:{id}", json.dumps(doc), ex=300)  # 5-minute TTL
return doc
```

After 5 minutes, cache refreshes.

## Hands-On Lab

### Lab 1.1: Trigger a Partial Failure

Stop Redis, observe graceful degradation:

```bash
# Terminal 1: Start API
cd ~/docs_project
docker-compose up api

# Terminal 2: Test normal request (Redis running)
curl http://localhost:8000/api/documents/1
# Instant response (from cache)

# Terminal 3: Stop Redis
docker-compose down redis
docker-compose up api postgres  # No Redis

# Test with Redis down
curl http://localhost:8000/api/documents/1
# Takes longer (no cache, hit database)
# Still works (graceful degradation)

# Check API logs
docker-compose logs api
# Logs show Redis connection timeout, fell back to DB
```

Knowledge gained:
- Services can partially fail
- Application logic needs fallbacks (DB if cache misses)
- Monitoring shows what's failing

### Lab 1.2: Network Latency

Simulate network delay with `tc` (traffic control):

```bash
# On Linux, add 100ms latency to Redis traffic
sudo tc qdisc add dev docker0 root netem delay 100ms

# API requests will be 100ms slower (round trip to Redis)
curl http://localhost:8000/api/documents/1
# Now takes ~100ms longer

# Remove latency
sudo tc qdisc del dev docker0 root
```

Knowledge gained:
- Network latency is variable
- Timeouts must account for variance
- Caching reduces latency

## Cheat Sheet: Distributed System Concepts

| Concept | Definition | Your Platform |
|---------|-----------|---|
| **Partial Failure** | One component fails, others don't | If Redis down, API still works (slower) |
| **Consistency** | All replicas see same data | PostgreSQL: strong; Redis: eventual |
| **Availability** | System returns data despite failures | API handles missing cache |
| **Partition Tolerance** | System works despite network split | Services continue working independently |
| **Trade-off** | Must choose 2 of CAP | PostgreSQL (CA), Redis (AP) |

## Key Takeaways

- **Distributed systems** = multiple processes coordinating via network
- **Challenges**: Unreliable networks, partial failures, inconsistency
- **Fallacies**: Don't assume reliability, latency, bandwidth, security are free
- **Your platform**: Already distributed (multiple API instances, cache, queue)
- **Design for failure**: Expect partial failures and handle gracefully
- **Test failure scenarios**: Chaos engineering (kill services, add latency)

Module 2 teaches message queues, the backbone of async systems.
