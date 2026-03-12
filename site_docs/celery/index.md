# Celery: Background Job Processing

## Overview

Celery is a **distributed task queue framework** for Python. Use it to:
- **Offload slow work** from API (PDF conversion, image processing, emails)
- **Schedule jobs** for later (send reminders, generate reports)
- **Retry failed tasks** (network errors, temporary failures)
- **Scale horizontally** (multiple workers processing jobs concurrently)

## Why Celery?

Without Celery:
```python
@app.post("/convert")
async def convert(pdf_url: str):
    result = convert_pdf(pdf_url)  # Blocks for 30 seconds!
    return result
```

Request waits 30 seconds. User sees spinner. Bad UX.

With Celery:
```python
@app.post("/convert")
async def convert(pdf_url: str):
    task = convert_pdf.delay(pdf_url)  # Returns immediately with task_id
    return {"task_id": task.id, "status": "queued"}

@app.get("/task/{task_id}")
async def get_status(task_id: str):
    task = convert_pdf.AsyncResult(task_id)
    return {"status": task.status, "result": task.result}
```

Request returns immediately. Client polls for result. Better UX.

## Architecture

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   FastAPI   │       │   Redis     │       │   Worker    │
│  (Producer) │──────▶│  (Broker)   │───────▶│  (Consumer) │
│             │       │             │       │             │
│ task.delay()│       │ queue:tasks │       │ process()   │
└─────────────┘       └─────────────┘       └─────────────┘
```

1. **Producer** (API): task.delay(args) adds task to Redis queue
2. **Broker** (Redis): stores queue of tasks
3. **Consumer** (Worker): polls queue, processes tasks

## Installation

```bash
pip install celery redis
```

## Modules

1. **01-what-is-celery.md**: Task queue concept, task.delay(), architecture
2. **02-tasks-and-workers.md**: @task decorator, task routing, worker specialization
3. **03-retry-and-error-handling.md**: Failure modes, self.retry(), dead letter queues
4. **04-celery-beat-scheduling.md**: Periodic tasks, Celery Beat scheduler
5. **05-monitoring-with-flower.md**: Flower UI, task monitoring, debugging

## Key Concepts

### Task
A function decorated with @app.task. Can be called with .delay() for async execution.

```python
@app.task
def send_email(user_id: int):
    ...

# Execute task asynchronously
send_email.delay(5)
```

### Queue
Redis list holding pending tasks. Workers pull from queue.

```
redis:0> LRANGE queue:celery 0 -1
1) "{task_id, args, kwargs}"
2) "{task_id, args, kwargs}"
```

### Worker
Long-running process that consumes tasks from queue and executes them.

```bash
celery -A myapp worker --loglevel=info
```

### Result Backend
Storage for task results (completion time, return value). Optional but useful.

```python
# Wait for result
result = task.get(timeout=30)
```

## Configuration

```python
# celery_app.py
from celery import Celery

app = Celery(
    'pdf_service',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_track_started=True,  # Mark task as started
    task_time_limit=60 * 5,   # 5-minute hard timeout
)

@app.task(bind=True, max_retries=3)
def convert_pdf(self, pdf_url: str):
    ...
```

## API Integration

```python
from celery import Celery
from fastapi import FastAPI

app = FastAPI()
celery = Celery('pdf_service', broker='redis://localhost:6379/0')

@app.post("/convert")
async def start_conversion(pdf_url: str):
    # Queue task, return immediately
    task = celery.send_task('convert_pdf', args=(pdf_url,))
    return {"task_id": task.id}

@app.get("/task/{task_id}")
async def get_task_status(task_id: str):
    task = celery.AsyncResult(task_id)
    return {
        "task_id": task_id,
        "status": task.status,
        "result": task.result,
        # Statuses: PENDING, STARTED, PROGRESS, SUCCESS, FAILURE, RETRY
    }
```

## Architecture in Production

```
┌─────────────────────────────────────────────────────────┐
│                    FastAPI Service                      │
│  POST /convert    GET /task/{id}    DELETE /task/{id}   │
└────────────────────────┬────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
    ┌────▼─────────┐          ┌──────────▼──────┐
    │ Redis Broker │          │ Result Backend  │
    │ queue:celery │          │ (Redis/DB)      │
    └────┬─────────┘          └─────────────────┘
         │
    ┌────┴──────────────────────────┐
    │                               │
┌───▼─────────────┐     ┌──────────▼───┐
│ Worker 1: PDF   │     │ Worker 2: PDF │
│ process()       │     │ process()     │
│ convert_pdf.run │     │ convert_pdf.run
└─────────────────┘     └───────────────┘
```

Multiple workers process tasks in parallel, independently.

## Hands-On Lab

### Lab: Basic Setup

```bash
# Terminal 1: Redis
docker run -d -p 6379:6379 redis:latest

# Terminal 2: Worker
celery -A celery_app worker --loglevel=info

# Terminal 3: API / Test
python -m uvicorn main:app --reload
```

```python
# celery_app.py
from celery import Celery

app = Celery(
    'pdf_service',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

@app.task
def convert_pdf(pdf_url: str):
    print(f"Converting {pdf_url}")
    return {"status": "done", "url": pdf_url}

# main.py
from fastapi import FastAPI
from celery_app import app

fastapi = FastAPI()

@fastapi.post("/convert")
def start_conversion(pdf_url: str):
    task = app.send_task('convert_pdf', args=(pdf_url,))
    return {"task_id": task.id}

@fastapi.get("/task/{task_id}")
def get_status(task_id: str):
    task = app.AsyncResult(task_id)
    return {"status": task.status, "result": task.result}
```

## Cheat Sheet: Celery Basics

### Define Task

```python
@app.task
def my_task(arg):
    return result
```

### Queue Task

```python
my_task.delay(arg)
```

### Check Status

```python
task = app.AsyncResult(task_id)
print(task.status)  # PENDING, STARTED, SUCCESS, FAILURE
```

### Start Worker

```bash
celery -A myapp worker --loglevel=info
```

## Key Takeaways

- **Celery = task queue framework** for async, distributed job processing
- **task.delay() = non-blocking, returns immediately**
- **Background workers = process tasks independently**
- **Redis = message broker** (holds queue)
- **Result backend = optional storage for results**
- **Multiple workers = parallel processing**
- **Next modules** teach tasks, retries, scheduling, monitoring

See modules 1-5 for detailed coverage.
