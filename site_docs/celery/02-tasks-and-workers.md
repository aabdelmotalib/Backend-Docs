# Module 2: Tasks, Workers, and Routing

## Task Definition Deep Dive

### Basic Task

```python
@app.task
def simple_task(x):
    return x * 2
```

### Task with Options

```python
@app.task(
    bind=True,              # Receive task instance as first arg (self)
    name='custom_name',     # Custom task name
    queue='default',        # Default queue
    max_retries=3,          # Retry max 3 times
    time_limit=300,         # Hard timeout 5 minutes
    soft_time_limit=240,    # Soft timeout 4 minutes (gives cleanup time)
    autoretry_for=(IOError,),  # Auto-retry on these exceptions
    rate_limit='100/h'      # Max 100 tasks per hour
)
def convert_pdf(self, pdf_url: str) -> dict:
    """
    self = task context
    Provides: self.request, self.retry(), self.update_state()
    """
    try:
        # Do work
        result = process_pdf(pdf_url)
        return {"status": "success", "result": result}
    except TimeoutError as e:
        # Retry
        raise self.retry(exc=e, countdown=5)
```

## Task Parameters

### Positional and Named Arguments

```python
@app.task
def merge_pdfs(pdf_urls: list, output_path: str, quality: str = 'high'):
    # Called with positional args
    task = merge_pdfs.delay(
        ['s3://file1.pdf', 's3://file2.pdf'],  # positional
        '/output/merged.pdf'
    )
    
    # Called with named args
    task = merge_pdfs.apply_async(
        args=['file1.pdf', 'file2.pdf'],
        kwargs={'output_path': '/output', 'quality': 'medium'},
        countdown=10  # Run in 10 seconds
    )
```

### Task Properties

```python
task = my_task.delay(arg)

# Task ID (unique identifier)
print(task.id)

# Task name (function name or custom)
print(task.name)

# Task state
print(task.status)

# Get result (blocks until ready)
result = task.get(timeout=30)

# Check if task is ready
if task.ready():
    print(task.result)
```

## Queues and Routing

### Single Queue

```python
# Default: all tasks go to "celery" queue
@app.task
def send_email(user_id: int):
    ...

send_email.delay(5)  # Goes to "celery" queue
```

### Multiple Queues: Task Routing

Route different tasks to different queues:

```python
# celery_app.py
app = Celery('app', broker='redis://localhost:6379')

# Configure routes
app.conf.task_routes = {
    'tasks.convert_pdf': {'queue': 'pdf'},      # CPU-intensive
    'tasks.send_email': {'queue': 'email'},     # I/O-intensive
    'tasks.generate_thumbnail': {'queue': 'images'},
    'tasks.scan_malware': {'queue': 'security'},
}

# Or define on task
@app.task(queue='pdf')
def convert_pdf(pdf_url: str):
    ...

@app.task(queue='email')
def send_email(to_address: str):
    ...

@app.task(queue='security')
def scan_file(file_path: str):
    ...
```

### Dedicated Workers

Each queue can have dedicated worker(s):

```bash
# Terminal 1: PDF worker (CPU-bound, 2 workers)
celery -A app worker -Q pdf --concurrency=2

# Terminal 2: Email worker (I/O-bound, 10 workers)
celery -A app worker -Q email --concurrency=10

# Terminal 3: Security worker (slow, 1 worker)
celery -A app worker -Q security --concurrency=1

# Terminal 4: Default queue
celery -A app worker -Q celery --concurrency=4
```

## Worker Options

### Concurrency

Number of parallel tasks per worker:

```bash
# 4 parallel tasks
celery -A app worker --concurrency=4

# Auto-detect CPU count
celery -A app worker --concurrency=auto

# CPU-bound: use 1-2 per core
# I/O-bound: can use 10+ per core (waiting for I/O)
```

### Pool

Execution model:

```bash
# prefork: forking (default, better isolation)
celery -A app worker --pool=prefork

# threads: threading (lighter weight, but GIL)
celery -A app worker --pool=threads

# solo: single process (for debugging)
celery -A app worker --pool=solo
```

### Logging

```bash
# Info level
celery -A app worker --loglevel=info

# Debug level (very verbose)
celery -A app worker --loglevel=debug

# Critical (only errors)
celery -A app worker --loglevel=critical
```

## Task States and Lifecycle

```python
@app.task(bind=True)
def process_task(self, data):
    # PENDING: queued, not yet started
    
    self.update_state(
        state='PROGRESS',
        meta={'current': 10, 'total': 100}
    )
    # PROGRESS: task running, call from task to report progress
    
    # ... task does work ...
    
    return {"status": "complete"}
    # SUCCESS: task returned, result stored
    
    # If exception:
    # FAILURE: task raised exception
    
    # If retry:
    # RETRY: task failed, queued for retry

# Client side
task = process_task.delay(data)

for _ in range(30):
    if task.ready():
        print(f"Result: {task.result}")
        break
    
    if task.state == 'PROGRESS':
        print(f"Progress: {task.info['current']}/{task.info['total']}")
    
    time.sleep(1)
```

## Real-World Example: Multi-Step PDF Processing

```python
# tasks.py
@app.task(queue='pdf')
def convert_pdf(pdf_url: str) -> dict:
    """Convert PDF to images"""
    pdf_bytes = download_file(pdf_url)
    images = pdf_to_images(pdf_bytes, dpi=300)
    s3_urls = [upload_to_s3(img) for img in images]
    return {"images": s3_urls}

@app.task(queue='images')
def generate_thumbnails(image_urls: list) -> dict:
    """Create thumbnails from images"""
    thumbnails = []
    for url in image_urls:
        img = download_file(url)
        thumb = create_thumbnail(img, size=(150, 150))
        s3_url = upload_to_s3(thumb)
        thumbnails.append(s3_url)
    return {"thumbnails": thumbnails}

@app.task(queue='security')
def scan_for_malware(image_urls: list) -> dict:
    """Scan images using ClamAV"""
    results = []
    for url in image_urls:
        file_data = download_file(url)
        is_clean = clam_scan(file_data)
        results.append({"url": url, "clean": is_clean})
    return {"results": results}

# API
@app.post("/convert")
def start_conversion(pdf_url: str):
    # Chain: convert → thumbnails → scan
    task1 = convert_pdf.delay(pdf_url)
    return {"job_id": task1.id}

# Worker processes conversion result
# When conversion done:
task1_result = task1.get()
task2 = generate_thumbnails.delay(task1_result['images'])

# When thumbnails done:
task2_result = task2.get()
task3 = scan_for_malware.delay(task2_result['thumbnails'])
```

## Hands-On Lab

### Lab 2.1: Task Routing

**celery_app.py**
```python
from celery import Celery
import time

app = Celery('app', broker='redis://localhost:6379')

app.conf.task_routes = {
    'tasks.cpu_task': {'queue': 'cpu'},
    'tasks.io_task': {'queue': 'io'},
}

@app.task(queue='cpu')
def cpu_task(n):
    # Simulate CPU work
    total = 0
    for i in range(n):
        total += i ** 2
    return total

@app.task(queue='io')
def io_task(duration):
    # Simulate I/O work
    time.sleep(duration)
    return f"Waited {duration}s"
```

**main.py**
```python
from fastapi import FastAPI
from celery_app import cpu_task, io_task

app = FastAPI()

@app.post("/cpu")
def queue_cpu_task(n: int = 1000000):
    task = cpu_task.delay(n)
    return {"task_id": task.id, "queue": "cpu"}

@app.post("/io")
def queue_io_task(duration: int = 5):
    task = io_task.delay(duration)
    return {"task_id": task.id, "queue": "io"}

@app.get("/result/{task_id}")
def get_result(task_id: str):
    from celery_app import app as celery_app
    task = celery_app.AsyncResult(task_id)
    return {"status": task.status, "result": task.result}
```

**Run**
```bash
# Terminal 1: Redis
docker run -d -p 6379:6379 redis:latest

# Terminal 2: CPU worker
celery -A celery_app worker -Q cpu --concurrency=1

# Terminal 3: I/O worker
celery -A celery_app worker -Q io --concurrency=5

# Terminal 4: API
python -m uvicorn main:app --reload

# Terminal 5: Test
curl -X POST http://localhost:8000/cpu?n=10000000
curl -X POST http://localhost:8000/io?duration=10
```

### Lab 2.2: Task Progress

```python
@app.task(bind=True)
def process_items(self, items):
    num_items = len(items)
    for i, item in enumerate(items):
        # Process item
        result = process(item)
        
        # Update progress
        self.update_state(
            state='PROGRESS',
            meta={
                'current': i + 1,
                'total': num_items,
                'result': result
            }
        )
    
    return {'status': 'complete'}

# Client
task = process_items.delay(list(range(100)))

while not task.ready():
    info = task.info
    print(f"Progress: {info['current']}/{info['total']}")
    time.sleep(1)

print(f"Done: {task.result}")
```

## Cheat Sheet: Tasks and Routing

### Define Task

```python
@app.task(
    queue='pdf',
    max_retries=3,
    time_limit=300
)
def my_task(arg):
    return result
```

### Route Tasks

```python
app.conf.task_routes = {
    'tasks.pdf_task': {'queue': 'pdf'},
    'tasks.email_task': {'queue': 'email'},
}
```

### Start Workers

```bash
celery -A app worker -Q pdf --concurrency=2
celery -A app worker -Q email --concurrency=10
```

### Task Properties

```python
task = my_task.delay(arg)
print(task.id)
print(task.status)
result = task.get(timeout=30)
```

## Key Takeaways

- **Queue routing = send different tasks to different workers**
- **Dedicated workers = CPU tasks on CPU worker, I/O on I/O worker**
- **Concurrency = parallel tasks per worker**
- **Task.update_state = report progress**
- **Task properties = id, status, result, etc.**
- **Task options = max_retries, timeout, rate limiting**
- **Next module** teaches retries and error handling

Module 3 covers retry logic and handling failures.
