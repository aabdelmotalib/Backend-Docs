# Module 3: Retry and Error Handling

## The Problem: Network Errors

Task fails:
- Network timeout: temporary, try again
- Database locked: temporary, try again
- Rate limited: temporary, try again
- Invalid input: permanent, don't retry

Celery retry logic retries transient errors, gives up on permanent failures.

## Task Failure Modes

Without retry:
```python
@app.task
def download_file(url: str):
    # Network flaky
    response = requests.get(url, timeout=5)
    return response.content
    # TimeoutError: task fails, job lost
```

With retry:
```python
@app.task(bind=True, max_retries=3)
def download_file(self, url: str):
    try:
        response = requests.get(url, timeout=5)
        return response.content
    except (ConnectionError, TimeoutError) as e:
        # Retry with backoff
        raise self.retry(exc=e, countdown=2 ** self.request.retries)
        # Retry count: 0, 1, 2, 3 (backoff: 1s, 2s, 4s)
```

## Retry Configuration

### Simple Retry

```python
@app.task(bind=True, max_retries=3)
def my_task(self):
    try:
        risky_operation()
    except TemporaryError as e:
        self.retry(exc=e)  # Retry immediately
```

### Exponential Backoff

Wait increasingly longer between retries:

```python
@app.task(bind=True, max_retries=5)
def download_api(self, url: str):
    try:
        return requests.get(url).json()
    except requests.RequestException as e:
        # Countdown: 1s, 2s, 4s, 8s, 16s
        countdown = min(2 ** self.request.retries, 3600)  # cap at 1 hour
        raise self.retry(exc=e, countdown=countdown)
```

### Auto-Retry Configuration

```python
@app.task(
    autoretry_for=(ConnectionError, TimeoutError),
    retry_kwargs={'max_retries': 3},
    retry_backoff=True,  # exponential
    retry_backoff_max=600,  # cap at 10 minutes
    retry_jitter=True,  # random jitter to avoid thundering herd
)
def auto_retry_task(url: str):
    # No explicit retry code needed
    response = requests.get(url)
    return response.json()
```

## Distinguishing Errors

Retry on transient, fail fast on permanent:

```python
@app.task(bind=True, max_retries=3)
def process_data(self, data: dict) -> dict:
    try:
        # Download from external API
        response = requests.get(data['api_url'], timeout=5)
        response.raise_for_status()
        
        # Parse response
        parsed = response.json()
        
        # Validate
        if 'required_field' not in parsed:
            raise ValueError("Missing required field")
        
        return parsed
        
    except (requests.Timeout, requests.ConnectionError) as e:
        # Transient: retry
        raise self.retry(exc=e, countdown=5)
        
    except ValueError as e:
        # Permanent: fail immediately
        raise Exception(f"Invalid data: {e}")
        
    except Exception as e:
        # Unknown: try a few times
        if self.request.retries < 2:
            raise self.retry(exc=e, countdown=10)
        else:
            raise
```

## Handling Specific Status Codes

```python
@app.task(bind=True, max_retries=3)
def call_api(self, endpoint: str):
    try:
        response = requests.get(endpoint)
        
        if response.status_code == 429:  # Rate limited
            # Retry with longer backoff
            raise self.retry(countdown=60)
            
        elif response.status_code == 500:  # Server error
            # Transient, retry
            raise self.retry(countdown=10)
            
        elif response.status_code == 404:  # Not found
            # Permanent, fail
            raise Exception(f"Not found: {endpoint}")
            
        response.raise_for_status()
        return response.json()
        
    except requests.RequestException as e:
        raise self.retry(exc=e, countdown=5)
```

## Dead Letter Queue (DLQ)

Tasks that fail after all retries go to DLQ for inspection:

```python
@app.task(bind=True, max_retries=3)
def process_job(self, job_data: dict):
    try:
        # Do work
        result = process(job_data)
        return result
        
    except Exception as e:
        if self.request.retries < self.max_retries:
            # Still have retries left
            raise self.retry(exc=e, countdown=10)
        else:
            # All retries exhausted
            # Send to DLQ
            send_to_dlq("process_job", job_data, str(e))
            raise

async def send_to_dlq(task_name: str, data: dict, error: str):
    """Store failed task for manual inspection"""
    db.add(DeadLetterTask(
        task_name=task_name,
        data=json.dumps(data),
        error=error,
        created_at=datetime.now()
    ))
    await db.commit()

# Later: inspect and retry manually
@app.get("/dlq")
async def get_dlq_tasks():
    tasks = await db.query(DeadLetterTask).filter_by(resolved=False).all()
    return tasks

@app.post("/dlq/{task_id}/retry")
async def retry_dlq_task(task_id: int):
    dlq_task = await db.get(DeadLetterTask, task_id)
    data = json.loads(dlq_task.data)
    
    # Retry task
    celery_task = process_job.delay(**data)
    
    # Mark DLQ task as retried
    dlq_task.resolved = True
    dlq_task.retry_task_id = celery_task.id
    await db.commit()
    
    return {"task_id": celery_task.id}
```

## Monitoring Failed Tasks

```python
# tasks.py
@app.on_after_finalize.connect
def setup_periodic_tasks(sender, **kwargs):
    # Every 5 minutes, check for failed tasks
    sender.add_periodic_task(300.0, check_failed_tasks.s())

@app.task
def check_failed_tasks():
    """Find tasks that failed"""
    from celery_app import app
    
    inspect = app.control.inspect()
    active = inspect.active()
    
    failed_count = 0
    for worker, tasks in (active or {}).items():
        for task in tasks:
            if task['state'] == 'FAILURE':
                failed_count += 1
                log_failure(task)
    
    if failed_count > 0:
        send_alert(f"{failed_count} tasks failed; check DLQ")
```

## Real-World: PDF Conversion with Retries

```python
@app.task(bind=True, max_retries=5)
def convert_pdf(self, pdf_url: str, user_id: int):
    """Convert PDF with smart retry logic"""
    try:
        # Download PDF
        pdf_bytes = download_from_s3(pdf_url)
        
        # Convert to images
        images = convert_to_images(pdf_bytes, dpi=300)
        
        # Upload results
        image_urls = []
        for i, img in enumerate(images):
            url = upload_to_s3(img, f"conversions/{user_id}/page_{i}.png")
            image_urls.append(url)
        
        return {
            "status": "completed",
            "images": image_urls,
            "count": len(image_urls)
        }
        
    except (ConnectionError, TimeoutError) as e:
        # Network issue: retry with backoff
        countdown = min(2 ** self.request.retries, 600)
        raise self.retry(exc=e, countdown=countdown)
        
    except convert_error.OutOfMemory as e:
        # Temporary: maybe we need to free memory first
        invalidate_cache()
        raise self.retry(exc=e, countdown=30)
        
    except Exception as e:
        # Check retry count
        if self.request.retries < self.max_retries:
            # Still have retries
            raise self.retry(exc=e, countdown=30)
        else:
            # All retries exhausted
            # Log to DLQ
            await send_to_dlq(
                "convert_pdf",
                {"pdf_url": pdf_url, "user_id": user_id},
                str(e)
            )
            # Notify user
            notify_user(user_id, f"PDF conversion failed: {pdf_url}")
            raise
```

## Hands-On Lab

### Lab 3.1: Exponential Backoff

**celery_app.py**
```python
from celery import Celery
import time

app = Celery('app', broker='redis://localhost:6379')

@app.task(bind=True, max_retries=4)
def flaky_network_task(self, url: str):
    import random
    
    # Simulate network that fails 70% of the time
    if random.random() < 0.7:
        countdown = 2 ** self.request.retries
        print(f"Failed! Retrying in {countdown}s (attempt {self.request.retries + 1})")
        raise self.retry(
            exc=ConnectionError("Network failed"),
            countdown=countdown
        )
    
    print(f"Success on attempt {self.request.retries + 1}")
    return {"status": "success"}
```

**Usage**
```bash
# Terminal 1: Redis
docker run -d -p 6379:6379 redis:latest

# Terminal 2: Worker
celery -A celery_app worker --loglevel=info

# Terminal 3: Python
from celery_app import flaky_network_task
import time

task = flaky_network_task.delay("http://example.com")

# Watch task retries
for _ in range(30):
    print(f"Status: {task.status}, Info: {task.info}")
    if task.ready():
        break
    time.sleep(1)

print(f"Final result: {task.result}")
```

### Lab 3.2: Smart Retry Logic

```python
import requests

@app.task(bind=True, max_retries=3)
def smart_retry_task(self, api_url: str):
    try:
        response = requests.get(api_url, timeout=5)
        
        if response.status_code == 429:  # Rate limit
            raise self.retry(countdown=60)
        elif response.status_code >= 500:  # Server error
            raise self.retry(countdown=10)
        elif response.status_code >= 400:  # Client error
            raise Exception(f"Client error: {response.status_code}")
        
        return response.json()
        
    except requests.Timeout:
        raise self.retry(countdown=5)
    except requests.ConnectionError as e:
        countdown = min(2 ** self.request.retries, 300)
        raise self.retry(exc=e, countdown=countdown)

# Test
task = smart_retry_task.delay("http://httpbin.org/delay/10")
```

## Cheat Sheet: Retry and Error Handling

### Simple Retry

```python
@app.task(bind=True, max_retries=3)
def task(self):
    try:
        risky()
    except TemporaryError as e:
        raise self.retry(exc=e)
```

### Exponential Backoff

```python
countdown = min(2 ** self.request.retries, 3600)
raise self.retry(exc=e, countdown=countdown)
```

### Auto-Retry

```python
@app.task(
    autoretry_for=(ConnectionError,),
    retry_kwargs={'max_retries': 3},
    retry_backoff=True
)
def task(): ...
```

### DLQ

```python
if self.request.retries >= self.max_retries:
    send_to_dlq(task_name, data, error)
    raise
```

## Key Takeaways

- **Transient vs permanent errors** — retry transient, fail permanently
- **Exponential backoff** — wait longer between retries
- **Max retries** — give up after N attempts
- **Dead letter queue** — inspect failures manually
- **Smart retry** — different logic for different status codes
- **Auto-retry config** — let Celery handle simple cases
- **Next module** teaches scheduling periodic tasks with Celery Beat

Module 4 covers scheduled tasks and periodic jobs.
