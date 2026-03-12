# Module 5: Redis as Message Broker

## The Analogy: Mailbox and Postal Service

Your home:
- You want to send packages to Alice
- You put package in mailbox
- Postal service picks up mailbox
- Delivers to Alice's mailbox
- Alice opens mailbox and processes

Redis as message broker:
- Producer (API endpoint) puts job in Redis list
- Message broker (Redis) stores in queue
- Consumer (worker) picks from queue
- Processes job, removes from queue

This is how Celery uses Redis.

## Queue Pattern: LPUSH/RPOP

```
RPUSH queue job              # Producer: add to end of queue
LPOP queue                   # Consumer: remove from start of queue
```

### Producer: Add Job

```python
@app.post("/convert")
async def start_conversion(pdf_url: str, user_id: int):
    job_id = str(uuid.uuid4())
    
    # Add job to queue (JSON payload)
    job_data = {
        "job_id": job_id,
        "pdf_url": pdf_url,
        "user_id": user_id,
        "created_at": time.time()
    }
    
    await redis.rpush("queue:pdf", json.dumps(job_data))
    
    return {"job_id": job_id, "status": "queued"}
```

### Consumer: Process Job

```python
async def pdf_worker():
    """Long-running worker that processes jobs"""
    while True:
        # Block until job available (0 = wait forever)
        job_json = await redis.blpop("queue:pdf", timeout=0)
        
        if not job_json:
            continue
        
        job = json.loads(job_json[1])  # [0]=key, [1]=value
        
        try:
            print(f"Processing {job['job_id']}")
            
            # Update status
            await redis.hset(f"job:{job['job_id']}", mapping={
                "status": "processing",
                "started_at": time.time()
            })
            
            # Do work
            result = convert_pdf(job['pdf_url'])
            
            # Update status
            await redis.hset(f"job:{job['job_id']}", mapping={
                "status": "completed",
                "result": result,
                "completed_at": time.time()
            })
        except Exception as e:
            # Update status with error
            await redis.hset(f"job:{job['job_id']}", mapping={
                "status": "failed",
                "error": str(e)
            })

# Run worker
asyncio.run(pdf_worker())
```

### Client: Check Job Status

```python
@app.get("/job/{job_id}")
async def get_job_status(job_id: str):
    status_data = await redis.hgetall(f"job:{job_id}")
    
    if not status_data:
        raise HTTPException(status_code=404)
    
    return {
        "job_id": job_id,
        "status": status_data[b"status"].decode(),
        "result": status_data.get(b"result", b"").decode(),
        "error": status_data.get(b"error", b"").decode()
    }
```

## Multiple Queues: Specialization

Different worker types, different queues:

```python
# Router jobs (lightweight)
await redis.rpush("queue:router", json.dumps({
    "path": "/convert",
    "handler": "handle_pdf"
}))

# Heavy computation (PDF processing)
await redis.rpush("queue:heavy", json.dumps({
    "pdf_url": "s3://bucket/file.pdf",
    "params": {"dpi": 300}
}))

# Email sending (I/O bound)
await redis.rpush("queue:email", json.dumps({
    "user_id": 5,
    "template": "conversion_complete"
}))
```

Workers specialize:

```python
# Fast worker for routing
async def router_worker():
    while True:
        job_json = await redis.blpop("queue:router", timeout=0)
        # Light work
        
# Slow worker for PDF processing (maybe fewer instances)
async def heavy_worker():
    while True:
        job_json = await redis.blpop("queue:heavy", timeout=0)
        # Heavy work
        
# Email worker
async def email_worker():
    while True:
        job_json = await redis.blpop("queue:email", timeout=0)
        # Send email
```

## Queue Depth Monitoring

Track how many jobs are waiting:

```python
async def monitor_queues():
    """Monitor queue health"""
    while True:
        depth_pdf = await redis.llen("queue:pdf")
        depth_email = await redis.llen("queue:email")
        
        print(f"PDF queue: {depth_pdf} jobs pending")
        print(f"Email queue: {depth_email} jobs pending")
        
        # Alert if queue is growing
        if depth_pdf > 1000:
            send_alert("PDF queue backlog too high")
        
        await asyncio.sleep(10)
```

## Pub/Sub: Broadcasting

For real-time notifications (not jobs):

```python
# Producer publishes event
await redis.publish("notifications:user:5", json.dumps({
    "event": "job_completed",
    "job_id": "abc123"
}))

# Client subscribes
pubsub = redis.pubsub()
await pubsub.subscribe("notifications:user:5")

async for message in pubsub.listen():
    if message['type'] == 'message':
        event = json.loads(message['data'])
        # User receives notification in real-time
```

## Success Pattern: Celery

Celery uses Redis for queuing:

```python
from celery import Celery

app = Celery('pdf_service', broker='redis://localhost:6379/0')

@app.task(bind=True, max_retries=3)
def convert_pdf(self, pdf_url: str, user_id: int):
    try:
        # Do work
        result = convert(pdf_url)
        return {"status": "completed", "result": result}
    except Exception as exc:
        # Retry with exponential backoff
        self.retry(exc=exc, countdown=2 ** self.request.retries)

# API: queue task
@app.post("/convert")
def start_conversion(pdf_url: str, user_id: int):
    task = convert_pdf.delay(pdf_url, user_id)
    return {"task_id": task.id}

# API: check status
@app.get("/task/{task_id}")
def get_task_status(task_id: str):
    task = convert_pdf.AsyncResult(task_id)
    return {"status": task.status, "result": task.result}
```

Celery handles:
- Queuing (RPUSH)
- Worker loop (BLPOP)
- Status tracking (Redis hash)
- Retries (put back in queue)
- Timeouts (kill job)
- Task results storage

## Hands-On Lab

### Lab 5.1: Basic Queue

```python
import redis.asyncio as redis
import asyncio
import json

async def producer():
    """Add jobs to queue"""
    client = await redis.from_url("redis://localhost:6379/0")
    
    for i in range(5):
        job = {"job_id": i, "work": f"process_{i}"}
        await client.rpush("queue:work", json.dumps(job))
        print(f"Queued job {i}")
        await asyncio.sleep(1)
    
    await client.close()

async def consumer():
    """Process jobs from queue"""
    client = await redis.from_url("redis://localhost:6379/0")
    
    for _ in range(5):
        # Block for up to 5 seconds
        result = await client.blpop("queue:work", timeout=5)
        
        if result:
            key, job_json = result
            job = json.loads(job_json)
            print(f"Processing {job['job_id']}")
            await asyncio.sleep(2)  # Simulate work
        else:
            print("Queue empty, timeout")
    
    await client.close()

# Run producer and consumer
asyncio.run(asyncio.gather(
    producer(),
    consumer()
))
```

### Lab 5.2: Queue Depth

```python
async def test_queue_depth():
    client = await redis.from_url("redis://localhost:6379/0")
    
    # Add 100 jobs
    for i in range(100):
        await client.rpush("queue:tasks", json.dumps({"id": i}))
    
    # Check depth
    depth = await client.llen("queue:tasks")
    print(f"Queue depth: {depth} jobs")
    
    # Process and monitor
    processed = 0
    while processed < 100:
        await client.blpop("queue:tasks", timeout=1)
        processed += 1
        depth = await client.llen("queue:tasks")
        
        if processed % 10 == 0:
            print(f"Processed: {processed}, Remaining: {depth}")
    
    await client.close()

asyncio.run(test_queue_depth())
```

### Lab 5.3: Job Status Tracking

```python
async def test_job_status():
    client = await redis.from_url("redis://localhost:6379/0")
    
    job_id = "job_123"
    
    # Producer: queue job
    await client.rpush("queue:jobs", json.dumps({
        "job_id": job_id,
        "data": "process_me"
    }))
    
    # Worker simulation
    job_json = await client.lpop("queue:jobs")
    job = json.loads(job_json)
    
    # Update status: queued → in_progress
    await client.hset(f"job:{job_id}", mapping={
        "status": "in_progress",
        "started": time.time()
    })
    
    # Simulate work
    await asyncio.sleep(1)
    
    # Update status: in_progress → completed
    await client.hset(f"job:{job_id}", mapping={
        "status": "completed",
        "result": "success",
        "completed": time.time()
    })
    
    # Client: check status
    status = await client.hgetall(f"job:{job_id}")
    print(f"Status: {status[b'status']}")
    
    await client.close()

asyncio.run(test_job_status())
```

## Cheat Sheet: Message Broker Pattern

### Queue

```python
# Producer: enqueue
await redis.rpush("queue:name", json.dumps(job))

# Consumer: dequeue (blocking)
result = await redis.blpop("queue:name", timeout=0)
job = json.loads(result[1])

# Check depth
depth = await redis.llen("queue:name")
```

### Job Status

```python
# Set status
await redis.hset(f"job:{job_id}", mapping={"status": "in_progress"})

# Get status
status = await redis.hget(f"job:{job_id}", "status")
```

### Pub/Sub

```python
# Publish
await redis.publish("notifications", json.dumps({"event": "done"}))

# Subscribe
pubsub = redis.pubsub()
await pubsub.subscribe("notifications")
```

## Key Takeaways

- **RPUSH/LPOP = queue pattern** — producers enqueue, consumers dequeue
- **BLPOP = blocking pop** — worker waits for jobs, not busy-waiting
- **Hash for status tracking** — job metadata stored separately
- **Multiple queues = specialization** — different worker types
- **Queue depth = health metric** — monitor backlog
- **Pub/Sub = broadcast** — for real-time, not persistent queuing
- **Celery = production queue** — abstracts away queue handling

Next: Celery section uses these same patterns.
