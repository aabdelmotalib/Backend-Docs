# Module 1: What is Celery

## The Analogy: Task Delegation

You're a manager:
- Employee asks: "Can you convert this PDF?"
- You: "Not right now. Write task on whiteboard."
- Employee: "What do I do while you're busy?"
- You: "Leave. I'll call you when it's done."
- You hand task to assistant
- Later: done, you call employee back

Celery:
- API request: "Convert PDF"
- API: "I'm busy. Added to queue."
- Request: "Return immediately with task_id"
- Worker: picks from queue, does work
- API client: polls /task/{id} for result

This is asynchronous job processing.

## Task Queue Concepts

### What's a Task?

A task is a function that:
- Takes time to execute (seconds, minutes)
- Should not block HTTP response
- Can be retried if it fails
- Might be scheduled for later

Examples:
- Convert PDF to images
- Send email
- Resize images
- Generate summary
- Scan for malware

### What's a Queue?

A queue is a list of tasks waiting to be processed.

```
Queue (Redis list):
┌──────────────────────────────┐
│ Task 1: convert pdf_1        │  (oldest)
│ Task 2: convert pdf_2        │
│ Task 3: convert pdf_3        │  (newest)
└──────────────────────────────┘

Worker picks Task 1, processes it, removes from queue.
Next Task 2 moves up.
```

### What's a Worker?

A worker is a long-running process that:
- Listens to queue
- Takes task from queue
- Executes task
- Reports result
- Waits for next task

```bash
$ celery -A myapp worker --loglevel=info
[2024-01-15 10:00:00,123: INFO/MainProcess] celery@hostname ready.
[2024-01-15 10:00:05,456: INFO/Worker-1] Received task: convert_pdf[abc123]
[2024-01-15 10:00:35,789: INFO/Worker-1] Task convert_pdf[abc123] succeeded
```

## Without Celery: Blocking Request

```python
@app.post("/convert")
async def convert(pdf_url: str):
    # Blocks here for 30 seconds
    result = await convert_pdf_slow(pdf_url)
    return result
```

Timeline:
```
Time    Request              Response
10:00   POST /convert       ⏳ waiting
10:05   ⏳ converting...
10:30   ✓ complete          {"result": "success"}
```

User waits 30 seconds. Slow UX.

## With Celery: Non-Blocking Queue

```python
from celery import Celery

celery = Celery('app', broker='redis://localhost:6379')

@celery.task
def convert_pdf(pdf_url: str):
    # Executed by worker, not request
    result = convert_pdf_slow(pdf_url)
    return result

@app.post("/convert")
async def convert(pdf_url: str):
    # Returns immediately
    task = convert_pdf.delay(pdf_url)
    return {"task_id": task.id, "status": "queued"}

@app.get("/task/{task_id}")
async def get_status(task_id: str):
    task = convert_pdf.AsyncResult(task_id)
    return {"status": task.status, "result": task.result}
```

Timeline:
```
Time    Request              Response              Worker
10:00   POST /convert       {"task_id": "abc"} ✓  pick job
10:01                                             converting...
10:30                                             ✓ done
10:31   GET /task/abc       {"status": "success"}
```

Request returns immediately. Worker does work asynchronously.

## Task Definition

```python
from celery import Celery

app = Celery(
    'pdf_processor',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

@app.task(bind=True, max_retries=3)
def convert_pdf(self, pdf_url: str):
    """
    self = task instance (access retry, request, etc)
    pdf_url = argument passed from .delay()
    """
    try:
        # Download PDF
        pdf_bytes = requests.get(pdf_url).content
        
        # Convert
        images = pdf_to_images(pdf_bytes)
        
        # Return success
        return {"images": len(images), "status": "success"}
    except NetworkError as e:
        # Retry with exponential backoff
        raise self.retry(exc=e, countdown=2 ** self.request.retries)

# Call task asynchronously
task = convert_pdf.delay("s3://bucket/file.pdf")
```

## Calling a Task: task.delay()

### Queued Task

```python
@app.post("/convert")
def start_conversion(pdf_url: str):
    # Queue task immediately
    task = convert_pdf.delay(pdf_url)
    
    # Returns Task object with id
    print(task.id)  # "abc-123-def"
    print(task.status)  # PENDING
    
    return {"task_id": task.id}
```

Multiple arguments:

```python
@celery.task
def add(a, b):
    return a + b

add.delay(5, 3)  # Returns task object

# Or named arguments
add.apply_async(args=(5, 3), countdown=10)  # Run in 10 seconds
```

## Task Status Tracking

```python
@app.get("/task/{task_id}")
def get_task_status(task_id: str):
    # Get task result
    task = convert_pdf.AsyncResult(task_id)
    
    # Status can be:
    # - PENDING: not yet started (queued or not found)
    # - STARTED: worker is processing
    # - SUCCESS: task completed successfully
    # - FAILURE: task failed
    # - RETRY: task failed, will be retried
    # - REVOKED: task was cancelled
    
    return {
        "task_id": task_id,
        "status": task.status,
        "result": task.result,  # Return value if SUCCESS
        # When SUCCESS:
        #   result = {"images": 200, "status": "success"}
        # When FAILURE:
        #   result = <exception traceback>
    }
```

## Real-World PDF Conversion Flow

```python
# API request
@app.post("/pdf-to-images")
def convert_endpoint(pdf_url: str, user_id: int):
    task = convert_pdf.delay(pdf_url, user_id)
    
    # Store job record in database
    db.add(ConversionJob(
        job_id=task.id,
        user_id=user_id,
        pdf_url=pdf_url,
        status="queued",
        created_at=datetime.now()
    ))
    db.commit()
    
    return {"job_id": task.id}

# Status check
@app.get("/pdf-to-images/{job_id}")
def get_conversion_status(job_id: str):
    # Check Celery status
    task = convert_pdf.AsyncResult(job_id)
    
    return {
        "job_id": job_id,
        "status": task.status,
        "celery_status": task.status,
        "result": task.result
    }

# Actual task (runs in worker)
@app.task(bind=True, max_retries=3)
def convert_pdf(self, pdf_url: str, user_id: int):
    try:
        # Download
        pdf_bytes = download_from_s3(pdf_url)
        
        # Convert (slow)
        images = pdf_to_images(pdf_bytes)
        
        # Upload results
        image_urls = [upload_to_s3(img) for img in images]
        
        # Return results
        return {
            "images": image_urls,
            "count": len(image_urls)
        }
    except Exception as e:
        # Retry
        raise self.retry(exc=e, countdown=min(2 ** self.request.retries, 600))
```

## Hands-On Lab

### Lab 1.1: Basic Task

Create three files:

**celery_app.py**
```python
from celery import Celery

app = Celery('demo', broker='redis://localhost:6379/0')

@app.task
def add(x, y):
    return x + y

@app.task
def slow_task(duration):
    import time
    time.sleep(duration)
    return f"Slept for {duration}s"
```

**main.py**
```python
from fastapi import FastAPI
from celery_app import add, slow_task

app = FastAPI()

@app.post("/add")
def queue_add(x: int, y: int):
    task = add.delay(x, y)
    return {"task_id": task.id}

@app.post("/slow")
def queue_slow(duration: int):
    task = slow_task.delay(duration)
    return {"task_id": task.id}

@app.get("/result/{task_id}")
def get_result(task_id: str):
    from celery_app import app as celery_app
    task = celery_app.AsyncResult(task_id)
    return {
        "task_id": task_id,
        "status": task.status,
        "result": task.result
    }
```

**Run**
```bash
# Terminal 1: Redis
docker run -d -p 6379:6379 redis:latest

# Terminal 2: Worker
celery -A celery_app worker --loglevel=info

# Terminal 3: API
python -m uvicorn main:app --reload

# Terminal 4: Test
curl -X POST http://localhost:8000/add?x=5&y=3
# {"task_id": "abc-123"}
curl http://localhost:8000/result/abc-123
# {"task_id": "abc-123", "status": "SUCCESS", "result": 8}
```

### Lab 1.2: Monitor Queue

```python
# Show pending tasks
from celery_app import app

pending = app.control.inspect().active()
print(pending)

# Or Redis directly
import redis
r = redis.Redis()
print(f"Queue depth: {r.llen('celery')}")
```

## Cheat Sheet: Task Queue

### Define

```python
@app.task
def my_task(arg):
    return result
```

### Queue

```python
task = my_task.delay(arg)
print(task.id)
```

### Check Status

```python
task = app.AsyncResult(task_id)
print(task.status)  # PENDING, STARTED, SUCCESS, FAILURE
print(task.result)
```

### Start Worker

```bash
celery -A myapp worker --loglevel=info
```

## Key Takeaways

- **Celery = task queue for async work**
- **task.delay() = queue, don't wait**
- **Worker = processes tasks from queue**
- **Non-blocking API = return immediately, client polls**
- **Status tracking = PENDING → STARTED → SUCCESS/FAILURE**
- **Redis = message broker holding queue**
- **Next module** teaches task routing and specialization

Module 2 covers task definition, routing, and worker specialization.
