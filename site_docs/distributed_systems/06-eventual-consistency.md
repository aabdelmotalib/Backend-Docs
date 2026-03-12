# Module 6: Eventual Consistency

## Strong vs Eventual Consistency

### Strong Consistency

All replicas see the same data, immediately.

```
API writes to PostgreSQL primary
Response: "Written"
Next read from any replica: Latest data
```

Trade-off: Adds latency (must replicate, then respond).

Your platform: PostgreSQL strong consistency for user data.

### Eventual Consistency

Replicas may lag momentarily. Eventually sync.

```
Write to Redis primary
Response: "Written" (before replicas sync)
Read from replica 1ms later: Might be stale
Read from replica 100ms later: Probably fresh
Read from replica 5 seconds later: Definitely fresh
```

Trade-off: Fast response, but stale reads briefly.

Your platform: Redis cache is eventually consistent.

## The Cache Gap

Your platform has both:

```
API writes to PostgreSQL (strong consistency)
API updates Redis cache (should, but often lazy)
     ↓
     Gap: Redis stale, DB fresh
```

Scenario:

```
User edits document
  ↓
API writes to PostgreSQL (immediately durable)
API should update Redis cache
But if update fails or is slow: cache is stale
  ↓
Next read: Returns stale data from Redis
```

### Preventing Stale Reads

#### Strategy 1: Cache Invalidation on Write

```python
@app.put("/api/documents/{id}")
def edit_document(id: str, content: str):
    # Write to DB
    doc = db.query(Document).filter_by(id=id).first()
    doc.content = content
    db.commit()
    
    # Invalidate cache immediately
    redis.delete(f"document:{id}")
    
    return {"status": "saved"}

# Next read:
@app.get("/api/documents/{id}")
def get_document(id: str):
    # Try cache
    cached = redis.get(f"document:{id}")
    if cached:
        return json.loads(cached)  # Fresh
    
    # Cache miss: fetch from DB
    doc = db.query(Document).filter_by(id=id).first()
    
    # Repopulate cache
    redis.set(f"document:{id}", json.dumps(doc), ex=300)  # 5 min TTL
    
    return doc
```

#### Strategy 2: TTL with Graceful Stale

```python
# Accept stale data within 5 minutes
redis.set(f"document:{id}", json.dumps(doc), ex=300)

# On read: return if exists, even if stale
cached = redis.get(f"document:{id}")
if cached:
    return json.loads(cached)  # Might be 5 minutes old
```

Acceptable for non-critical data (document listing, user profile image).

#### Strategy 3: Version Tracking

```python
# Store version in cache
doc_version = 5
redis.set(f"document:{id}:version", doc_version)

# Client checks version
@app.get("/api/documents/{id}")
def get_document(id: str):
    cached_doc = redis.get(f"document:{id}")
    cached_version = int(redis.get(f"document:{id}:version") or 0)
    
    db_doc = db.query(Document).filter_by(id=id).first()
    
    if cached_version == db_doc.version:
        return json.loads(cached_doc)  # Cache fresh
    else:
        return db_doc  # Cache stale, return fresh
```

## Distributed Cache Invalidation

Harder with multiple APIs:

```
API-1: Edits document, invalidates cache
API-2: Still has document cached (different process)
```

Solution: Event-driven invalidation.

```
API-1 writes to DB, publishes event: "document:123 updated"
Redis subscriber listens to event
All subscribers (API-1, API-2) invalidate document:123 cache
```

Implementation (Redis pub/sub):

```python
# Publisher (API that edited)
@app.put("/api/documents/{id}")
def edit_document(id: str, content: str):
    doc = db.query(Document).filter_by(id=id).first()
    doc.content = content
    db.commit()
    
    # Publish event
    redis.publish(f"cache:invalidate", json.dumps({
        "key": f"document:{id}"
    }))
    
    return {"status": "saved"}

# Subscriber (all API instances)
import threading

def cache_invalidation_subscriber():
    pubsub = redis.pubsub()
    pubsub.subscribe("cache:invalidate")
    
    for message in pubsub.listen():
        if message['type'] == 'message':
            event = json.loads(message['data'])
            key_to_delete = event['key']
            redis.delete(key_to_delete)

# Start subscriber thread on API startup
threading.Thread(
    target=cache_invalidation_subscriber,
    daemon=True
).start()
```

## Read-Your-Writes Consistency

User edits document, refreshes page. Should see their own edit immediately.

Problem: Reads might hit cache, sees stale data.

Solution: After write, invalidate user's cache + redirect to DB.

```python
@app.put("/api/documents/{id}")
def edit_document(id: str, content: str, current_user):
    doc = db.query(Document).filter_by(id=id).first()
    doc.content = content
    db.commit()
    
    # Invalidate user's view cache
    redis.delete(f"user:{current_user.id}:documents")
    
    return {"status": "saved", "document": doc}
```

Client immediately gets fresh doc in response (not from cache).

## CQRS: Command Query Responsibility Segregation

Separate write model from read model.

```
Writes: PostgreSQL (strong, slow, durable)
Reads: Redis/Elasticsearch (eventual, fast)
```

Architecture:

```
User request → API
             ↓
         Writes → PostgreSQL writes
         Reads ← Redis reads
         ↓
         Async sync: PostgreSQL → Redis (eventual)
```

Example:

```python
# Write path
@app.post("/api/documents")
def create_document(content: str, current_user):
    doc = Document(
        id=uuid.uuid4(),
        content=content,
        user_id=current_user.id
    )
    db.add(doc)
    db.commit()
    
    # Publish event for async sync
    message = {
        "event": "document_created",
        "document_id": doc.id,
        "ts": datetime.utcnow().isoformat()
    }
    celery_app.send_task("sync_to_cache", args=[message])
    
    return {"id": doc.id}

# Read path (from cache/search)
@app.get("/api/documents")
def list_documents(current_user):
    # Read from cache (eventually consistent)
    cached = redis.get(f"user:{current_user.id}:documents")
    if cached:
        return json.loads(cached)
    
    # Cache miss: read from DB
    docs = db.query(Document).filter_by(user_id=current_user.id).all()
    
    # Update cache
    redis.set(
        f"user:{current_user.id}:documents",
        json.dumps([{"id": d.id, "content": d.content} for d in docs]),
        ex=300
    )
    
    return docs

# Async sync task
@app_celery.task
def sync_to_cache(message):
    """Sync PostgreSQL writes to Redis cache"""
    if message["event"] == "document_created":
        # Get fresh data from DB
        doc = db.query(Document).get(message["document_id"])
        
        # Update cache
        redis.set(f"document:{doc.id}", json.dumps(doc))
        
        # Invalidate user's document list
        redis.delete(f"user:{doc.user_id}:documents")
```

## Hands-On Lab

### Lab 6.1: Observe Cache Staleness

```bash
# Start API with Redis
docker-compose up api redis postgres

# Terminal 1: Poll for document
watch -n 0.5 'curl http://localhost:8000/api/documents/1'

# Terminal 2: Edit document directly in DB
docker-compose exec postgres psql -c \
  "UPDATE documents SET content='NEW' WHERE id='1'"

# Terminal 1: See old value returned (cache stale) for until TTL expires
```

### Lab 6.2: Test Cache Invalidation

```python
# Set a document in cache with -1 (no TTL)
redis.set("document:1", json.dumps({"content": "old"}))

# Edit document (invalidates cache)
# curl -X PUT http://localhost:8000/api/documents/1 -d '{"content":"new"}'

# Check cache is deleted
redis.get("document:1")  # Returns None (invalidated)

# Next read fetches fresh from DB
# curl http://localhost:8000/api/documents/1
# Returns "new"
```

## Cheat Sheet: Eventual Consistency

```python
# Cache with TTL (stale acceptable)
redis.set(f"doc:{id}", json.dumps(doc), ex=300)

# Invalidate on write
redis.delete(f"doc:{id}")

# Event-driven invalidation (multi-instance)
redis.publish("cache:invalidate", key)

# CQRS: separate reads/writes
# Writes: PostgreSQL (strong)
# Reads: Redis (eventual)
# Sync: Celery async task

# Read-your-writes
# After write, invalidate user's cache
redis.delete(f"user:{user_id}:cache")
```

## Key Takeaways

- **Strong consistency** (PostgreSQL) = immediate but slow
- **Eventual consistency** (Redis) = fast but stale briefly
- **Cache invalidation** = delete on write or TTL
- **Event-driven** sync invalidates multiple instances
- **CQRS** separates write (DB) and read (cache) paths
- **Read-your-writes** = bypass cache after own writes
- **Accept staleness** for non-critical data (document lists, counts)
- **Prefer TTL** (simpler) over explicit invalidation (harder)

Module 7 teaches observability — how to see what's happening in production.
