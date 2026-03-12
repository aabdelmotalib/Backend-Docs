# Module 2: Message Queues and Async Processing

## The Analogy: A Restaurant Order Queue

Single machine:
- Customer orders → Chef cooks → Customer eats (synchronous)
- If customer takes 1 hour to eat, chef cannot serve next customer

Distributed:
- Customer orders → Put in queue → Chef sees queue → Cooks eventually → Customer gets food
- Chef can serve many customers by queueing orders
- Customer doesn't wait for cooking; finds seat, eats when food is ready

Celery is your order queue. API adds tasks. Workers process them.

## Why Message Queues?

### Problem 1: Slow Operations

PDF conversion takes 30 seconds.

Without queue:
```python
# Bad: synchronous
@app.post("/api/documents/upload")
def upload_document(file: UploadFile):
    contents = file.read()
    
    pdf = convert_to_pdf(contents)  # 30 seconds!
    save_to_storage(pdf)
    
    return {"status": "converted"}  # Response takes 30 seconds
```

User waits 30 seconds for response (browser times out).

With queue:
```python
# Good: async
@app.post("/api/documents/upload")
def upload_document(file: UploadFile):
    contents = file.read()
    
    # Put task in queue (instant)
    task = convert_pdf_task.delay(contents)
    
    return {"task_id": task.id, "status": "queued"}  # Response instant

# Separate worker processes
@app_celery.task
def convert_pdf_task(contents):
    pdf = convert_to_pdf(contents)  # Takes 30s, but doesn't block API
    save_to_storage(pdf)
```

User gets instant response, conversion happens later.

### Problem 2: Load Spikes

Normally: 100 requests/second → Fast
Spike: 10,000 requests/second for 5 seconds

Without queue:
- API tries to handle all 10,000 at once
- Memory explodes, CPU maxes out
- Most requests timeout
- System crashes

With queue:
- API adds tasks to queue (instant, memory-bounded queue)
- Queue holds 100,000 tasks
- Workers process at their capacity (e.g., 100/second)
- Spike takes 100 seconds to process (queue backlog)
- No crashes, eventual success

### Use Cases from Your Platform

1. **PDF Conversion** — takes 30s, user waits → async task
2. **Malware Scanning** — takes 5s per file → async with ClamAV
3. **Sending Emails** — takes 2s per email → async batch
4. **Report Generation** — takes 2 minutes → async, send notification when done
5. **Database Cleanup** — 1x daily cleanup → scheduled task

## Delivery Guarantees

Message queues offer different delivery guarantees:

### At-Most-Once

Message delivered 0 or 1 times.

```
Producer: Task added to queue
Consumer: Worker reads task, starts processing
         System crashes mid-task
Result: Task never completes, never retried
```

**Issue**: Data loss (task disappears).
**Advantage**: No duplicates.
**Use**: Non-critical tasks (cache warming, analytics).

### At-Least-Once

Message delivered 1+ times.

```
Producer: Task added to queue
Consumer: Worker reads task, processes, crashes before ack
         Message returned to queue
         Another worker picks it up, processes again
Result: Task may execute 2+ times
```

**Issue**: Duplicates (task runs twice).
**Advantage**: No data loss.
**Use**: Most tasks (requires idempotency).

### Exactly-Once

Message delivered exactly once.

Requires coordination:
- Consumer process task
- Atomically: mark "done" in queue, write results to DB
- If either fails, both rollback

Much harder, typically only for critical systems.

**Use**: Financial transactions (payment processing).

## Celery: At-Least-Once Delivery

Your platform uses Celery with Redis broker.

```python
from celery import Celery

app_celery = Celery('tasks', broker='redis://redis:6379')

@app_celery.task(bind=True, max_retries=3)
def process_pdf(self, pdf_id, user_id):
    try:
        pdf = db.query(PDF).get(pdf_id)
        result = convert_pdf(pdf)
        save_result(result)
        return {"status": "completed", "result_id": result.id}
    except Exception as exc:
        # Retry up to 3 times with exponential backoff
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
```

Flow:
1. Task added to Redis queue
2. Worker picks up task
3. Worker executes function
4. Worker acks task (marks task done in queue)
5. If worker crashes mid-execution, task is retried

**Risk**: If worker crashes between step 3 and 4, task runs twice.

Solution: Idempotency (Module 5).

## Backpressure: Preventing Queue Overflow

If workers are slow, queue grows infinitely. Eventually memory exhausts.

### Without Backpressure

```python
@app.post("/api/convert")
def queue_conversion(file_id):
    for i in range(1000):
        process_pdf.delay(file_id)  # Add 1000x to queue
    # Queue now has 1M tasks in memory
```

Queue explodes.

### With Backpressure

```python
@app.post("/api/convert")
def queue_conversion(file_id):
    queue_size = redis.llen("celery_queue")
    if queue_size > 100_000:
        return {"status": "queue_full", "status_code": 429}
    
    process_pdf.delay(file_id)
    return {"status": "queued"}
```

If queue is full, return 429 "Too Many Requests" to client.

Client implements retry logic (exponential backoff).

## Celery in Production

Real configuration for your platform:

```python
# config.py
app_celery = Celery('pdf_platform')

app_celery.conf.update(
    broker_url='redis://redis:6379/0',
    result_backend='redis://redis:6379/1',
    
    # Task configuration
    task_track_started=True,  # Track task start
    task_max_retries=3,
    task_default_retry_delay=60,  # Retry after 60s
    
    # Queue configuration
    task_queue_max_priority=10,
    task_acks_late=True,  # Ack after task completes (safer)
    
    # Worker pool
    worker_prefetch_multiplier=1,  # Worker fetches 1 task at a time
    worker_max_tasks_per_child=1000,  # Restart worker after 1000 tasks (memory safety)
    
    # Timeout
    task_soft_time_limit=600,  # Soft timeout after 10 minutes
    task_time_limit=720,  # Hard timeout after 12 minutes
)
```

### Key Settings

- `task_acks_late=True`: Worker acks task only after completion (safer, but slower)
- `worker_prefetch_multiplier=1`: Fetch one task at a time (prevents one slow task blocking others)
- `worker_max_tasks_per_child=1000`: Reload worker process periodically (prevent memory leaks)
- `task_time_limit`: Hard timeout (worker killed if exceeded)

## Hands-On Lab

### Lab 2.1: Queue Tasks

```python
from celery import Celery
import time

app_celery = Celery('tasks', broker='redis://localhost:6379')

@app_celery.task
def slow_task(x):
    time.sleep(10)  # Simulate 10s work
    return x * 2

# Queue 100 tasks
for i in range(100):
    task = slow_task.delay(i)
    print(f"Task {i}: {task.id}")

# Check queue size
# redis-cli LLEN celery

# Start worker in separate terminal
# celery -A tasks worker --loglevel=info

# Watch tasks process
# As workers process, queue shortens
```

### Lab 2.2: Monitor Celery Flower Dashboard

```bash
# Start Flower (web dashboard for Celery)
docker-compose up flower

# Visit http://localhost:5555

# See:
# - Active tasks
# - Completed tasks
# - Failed tasks
# - Worker status
# - Queue depth
```

## Backpressure in Action

```bash
# Start producer (add many tasks)
python producer.py

# Monitor queue depth
watch -n 1 'redis-cli LLEN celery'

# You'll see queue grow while workers catch up

# Add backpressure
# Modify producer to check queue before adding
# Queue stays bounded
```

## Cheat Sheet: Message Queues

```python
# Define celery task
@app_celery.task(max_retries=3)
def process_pdf(pdf_id):
    try:
        pdf = db.query(PDF).get(pdf_id)
        convert(pdf)
    except Exception as exc:
        raise self.retry(exc=exc, countdown=60)

# Queue task (non-blocking)
task = process_pdf.delay(pdf_id)

# Check task status
result = task.get(timeout=30)  # Wait max 30s

# Check queue size (Redis)
queue_size = redis.llen("celery")

# Backpressure
if queue_size > THRESHOLD:
    raise Exception("Queue full")
```

## Key Takeaways

- **Message queues** decouple producer and consumer
- **At-least-once** delivery (Celery default) requires idempotency
- **Backpressure** prevents queue overflow
- **Timeouts** prevent stuck tasks
- **Monitoring** shows what's happening (Flower)
- **Retries** with exponential backoff handle transient failures

Module 3 explains the CAP theorem, which constrains all choices here.
