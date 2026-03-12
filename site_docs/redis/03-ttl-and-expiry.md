# Module 3: TTL and Key Expiry

## The Analogy: Milk and Expiration Dates

Milk in fridge:
- Store: bought today
- Use: drink before expiration
- Auto-expire: after 2 weeks, automatically discard (bad milk)

Redis keys:
- Store: SET session:token "user_id=5"
- Use: GET session:token
- Auto-expire: after 24 hours, automatically DELETE (stale session)

Without expiry, old data accumulates forever. Redis TTL (Time-To-Live) automatically deletes expired keys.

## Setting Expiry

### Approach 1: SETEX (Set with Expiry)

```
SETEX key seconds value
```

Set key with TTL in one command:

```python
# Session expires in 24 hours (86400 seconds)
await redis.setex("session:token_abc123", 86400, "user_id=5")

# After 24 hours, key is automatically deleted
```

### Approach 2: SET + EXPIRE

```
SET key value
EXPIRE key seconds
```

Set, then add expiry:

```python
await redis.set("cache:user:5", json.dumps(user_data))
await redis.expire("cache:user:5", 3600)  # Expire in 1 hour
```

### Approach 3: PSETEX (Milliseconds)

```
PSETEX key milliseconds value
```

For millisecond precision:

```python
# Expire in 500ms (for rate limiters needing high precision)
await redis.psetex("rate_limit:5", 500, "1")
```

## Checking Expiry

```
TTL key            # Seconds remaining (-1 if no expiry, -2 if doesn't exist)
PTTL key           # Milliseconds remaining
EXPIREAT key timestamp   # Expire at Unix timestamp
PERSIST key        # Remove expiry
```

### Example

```python
# Check remaining TTL
ttl = await redis.ttl("session:abc123")
if ttl == -1:
    print("Key exists but has no expiry")
elif ttl == -2:
    print("Key doesn't exist")
else:
    print(f"Expires in {ttl} seconds")

# Renew expiry
await redis.expire("session:abc123", 86400)

# Remove expiry
await redis.persist("session:abc123")
```

## Session Storage Pattern

Typical session workflow:

```python
@app.post("/login")
async def login(email: str, password: str):
    user = verify_credentials(email, password)
    
    # Generate token
    token = secrets.token_urlsafe(32)
    
    # Store in Redis with 24-hour expiry
    await redis.setex(
        f"session:{token}",
        86400,  # 24 hours
        str(user.id)
    )
    
    return {"token": token}

@app.get("/profile")
async def get_profile(token: str = Depends(get_token)):
    # Get user_id from session
    user_id = await redis.get(f"session:{token}")
    
    if not user_id:
        raise HTTPException(status_code=401, detail="Session expired")
    
    user = await get_user_from_db(int(user_id))
    return user
```

When session expires (24 hours), key is automatically deleted. Next request gets 401.

## Rate Limiting with TTL

```python
@app.get("/api/convert")
async def convert(user_id: int, image_url: str):
    # Key for this hour
    hour = datetime.now().hour
    key = f"rate_limit:{user_id}:{hour}"
    
    # Increment counter
    count = await redis.incr(key)
    
    # First request of the hour: set expiry
    if count == 1:
        await redis.expire(key, 3600)  # Expire in 1 hour
    
    # Check limit
    if count > 100:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    
    return convert_pdf(image_url)
```

Traffic pattern:
- 1:00 PM: requests count from 0
- 1:59 PM: requests count up to ~100
- 2:00 PM: counter expires, resets to 0

Next hour, new counter starts.

## Sliding Window Rate Limiting

More accurate: expires individual requests:

```python
async def rate_limit_sliding(user_id: int, max_requests: int, window_seconds: int):
    key = f"rate_limit:{user_id}"
    now = int(time.time())
    
    # Remove old requests outside window
    await redis.zremrangebyscore(key, 0, now - window_seconds)
    
    # Count requests in window
    count = await redis.zcard(key)
    
    if count >= max_requests:
        raise HTTPException(status_code=429)
    
    # Add current request
    await redis.zadd(key, {str(now): now})
    
    # Ensure key expires after window
    await redis.expire(key, window_seconds)
```

## Cache with Expiry

```python
async def get_user_cached(user_id: int):
    cache_key = f"user:{user_id}"
    
    # Try cache
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Cache miss: query database
    user = await db.get(User, user_id)
    
    # Cache for 1 hour
    await redis.setex(
        cache_key,
        3600,
        json.dumps({
            "id": user.id,
            "email": user.email,
            "name": user.name
        })
    )
    
    return user
```

Cache invalidation: When user updates profile, delete cache:

```python
@app.post("/profile")
async def update_profile(user_id: int, **updates):
    # Update database
    user = await db.get(User, user_id)
    for key, value in updates.items():
        setattr(user, key, value)
    await db.commit()
    
    # Invalidate cache
    await redis.delete(f"user:{user_id}")
    
    return user
```

## Background Job Cleanup

Some jobs benefit from automatic expiry:

```python
@app.post("/convert")
async def convert(pdf_url: str, user_id: int):
    job_id = str(uuid.uuid4())
    
    # Queue job (don't expire; worker processes it)
    await redis.rpush("queue:pdf", json.dumps({
        "job_id": job_id,
        "pdf_url": pdf_url,
        "user_id": user_id
    }))
    
    # Track job status with 7-day expiry
    # Old jobs automatically cleaned up
    await redis.setex(
        f"job:{job_id}:status",
        604800,  # 7 days
        "queued"
    )
    
    return {"job_id": job_id}

@app.get("/job/{job_id}")
async def get_job_status(job_id: str):
    status = await redis.get(f"job:{job_id}:status")
    
    if not status:
        raise HTTPException(status_code=404, detail="Job not found or expired")
    
    return {"job_id": job_id, "status": status.decode()}
```

## Hands-On Lab

### Lab 3.1: Session with TTL

```python
import redis.asyncio as redis
import asyncio
import time

async def test_session():
    client = await redis.from_url("redis://localhost:6379/0")
    
    # Create session
    token = "token_abc123"
    await client.setex(f"session:{token}", 5, "user_id=5")  # 5 second expiry
    
    # Check immediately
    user_id = await client.get(f"session:{token}")
    print(f"1. Immediately: {user_id}")  # user_id=5
    
    # Check TTL
    ttl = await client.ttl(f"session:{token}")
    print(f"2. TTL: {ttl} seconds")  # ~5 seconds
    
    # Wait 6 seconds
    await asyncio.sleep(6)
    
    # Key should be expired
    user_id = await client.get(f"session:{token}")
    print(f"3. After 6 seconds: {user_id}")  # None (expired)
    
    await client.close()

asyncio.run(test_session())
```

### Lab 3.2: Rate Limiting

```python
async def test_rate_limit():
    client = await redis.from_url("redis://localhost:6379/0")
    
    user_id = 5
    max_requests = 3
    
    for i in range(5):
        key = f"rate_limit:{user_id}"
        count = await client.incr(key)
        
        if count == 1:
            await client.expire(key, 10)  # Reset every 10 seconds
        
        ttl = await client.ttl(key)
        print(f"Request {i+1}: count={count}, ttl={ttl}s, allowed={count <= max_requests}")
        
        await asyncio.sleep(1)
    
    await client.close()

asyncio.run(test_rate_limit())
```

### Lab 3.3: Cache Invalidation

```python
async def test_cache():
    client = await redis.from_url("redis://localhost:6379/0")
    
    user_id = 5
    cache_key = f"user:{user_id}"
    
    # Set cache
    user_data = {"id": 5, "name": "Alice", "email": "alice@example.com"}
    await client.setex(cache_key, 3600, str(user_data))
    
    # Retrieve from cache
    cached = await client.get(cache_key)
    print(f"1. From cache: {cached}")
    
    # Invalidate (user profile updated)
    await client.delete(cache_key)
    print(f"2. Cache invalidated")
    
    # Try to get again
    cached = await client.get(cache_key)
    print(f"3. After invalidation: {cached}")  # None
    
    await client.close()

asyncio.run(test_cache())
```

## Cheat Sheet: TTL and Expiry

### Set with Expiry

```
SETEX key seconds value
PSETEX key milliseconds value
```

### Check/Manage Expiry

```
TTL key
PTTL key
EXPIRE key seconds
PERSIST key
EXPIREAT key timestamp
```

### Python

```python
await redis.setex("key", 3600, "value")  # 1 hour
ttl = await redis.ttl("key")
await redis.expire("key", 7200)  # Extend to 2 hours
```

## Key Takeaways

- **TTL = automatic cleanup** — keys expire and are deleted automatically
- **SETEX = set with expiry** — one command, fewer bugs
- **Session storage** — token expires, forces user to login again
- **Rate limiting** — counter resets every hour/minute
- **Cache with expiry** — stale data automatically removed
- **No manual cleanup** — Redis handles deletion
- **Trade-off** — less control, but simpler and automatically correct

Module 4 teaches atomic operations for safe concurrent access.
