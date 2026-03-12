# Module 5: Monitoring with Flower

## What is Flower?

Flower is a real-time web UI for monitoring Celery:
- Task status (active, completed, failed)
- Task details (arguments, result, traceback)
- Worker status (load, uptime, active tasks)
- Queue depth
- Statistics and graphs

```
         ┌──────────────────────────────┐
         │   Flower Web UI (port 5555)  │
         │  - Active tasks              │
         │  - Task history              │
         │  - Worker stats              │
         └──────────────────────────────┘
                  ▲
                  │
         ┌────────┴──────────┐
         │                   │
    ┌────▼─────┐      ┌─────▼────┐
    │  Worker  │      │  Worker  │
    │ (running │      │ (running │
    │  tasks)  │      │  tasks)  │
    └────┬─────┘      └─────┬────┘
         │                   │
         └────────┬──────────┘
                  │
            ┌─────▼────┐
            │   Redis  │
            │  (queue) │
            └──────────┘
```

## Installation

```bash
pip install flower
```

## Starting Flower

```bash
# Connect to Celery app
flower -A celery_app --port=5555

# Or with Celery broker
flower --broker=redis://localhost:6379/0
```

Open browser: `http://localhost:5555`

## Dashboard: Real-Time Monitoring

### Active Tasks Tab

Shows currently processing tasks:
- Task name
- Task ID
- Worker
- Runtime (how long it's been running)
- Argv (arguments)

Example:
```
convert_pdf (abc-123-def)
Worker: worker1@hostname
Runtime: 25.3s
Argv: ('s3://file.pdf',)
```

### Task Details

Click on any task to see:
- **Name**: task function name
- **ID**: unique task identifier
- **State**: PENDING, STARTED, SUCCESS, FAILURE, RETRY
- **Argv**: positional arguments
- **Kwargs**: named arguments
- **Result**: returned value (if SUCCESS)
- **Traceback**: exception message (if FAILURE)
- **Expires**: when result is deleted from backend
- **Queue**: which queue task was on
- **Exchange**: message format

### Worker Stats

- **Hostname**: worker process name
- **Status**: online/offline
- **Concurrency**: max parallel tasks
- **Active**: currently processing
- **Processed**: total completed
- **Failed**: total failed
- **Uptime**: how long running
- **CPU**: CPU usage
- **Memory**: memory usage

## Common Tasks in Flower

### Retry a Failed Task

1. Go to **Tasks** tab
2. Find failed task
3. Click "Retry" button
4. Task is re-queued with same arguments
5. Worker picks it up and processes again

Useful for:
- Network errors (API returned 503)
- Temporary resource failures
- Database locks

### Revoke (Cancel) a Task

1. **Active** task running too long?
2. Click **Revoke**
3. Task is cancelled, doesn't finish
4. Worker stops and moves to next

### View Task Result

For SUCCESS tasks:
- Click task
- See **Result** section
- Contains exactly what task returned

Example:
```python
@app.task
def convert_pdf(url):
    return {"images": 200, "status": "completed"}

# In Flower, Result shows:
# {"images": 200, "status": "completed"}
```

### Monitor Queue Depth

**Queues** tab shows:
- Queue name
- Messages (pending tasks)
- Current size
- Max size

If queue depth keeps growing, workers can't keep up. Scale up workers.

## Real-Time Graphs

### Task Graph

- X-axis: time
- Y-axis: tasks/second
- Shows throughput over time

Useful:
- Detect slow periods
- Peak traffic patterns
- Scaling decisions

### Worker Availability

- Which workers online/offline
- When workers started/stopped
- Worker uptime

## API: Programmatic Access

Flower exposes API for monitoring:

```python
import requests

# Get active tasks
response = requests.get('http://localhost:5555/api/tasks/active')
active = response.json()

for task_id, task_info in active.items():
    print(f"{task_info['name']}: {task_info['time_start']}")

# Get worker stats
response = requests.get('http://localhost:5555/api/workers')
workers = response.json()

for worker_name, stats in workers.items():
    print(f"{worker_name}: {stats['pool']['max-concurrency']} workers")

# Get queue depth
response = requests.get('http://localhost:5555/api/queues')
queues = response.json()

for queue_name, queue_stats in queues.items():
    print(f"{queue_name}: {queue_stats['messages']} pending")
```

## Production Setup: Flower + Supervisor

Keep Flower running with Supervisor:

```ini
# /etc/supervisor/conf.d/flower.conf
[program:flower]
command=flower -A celery_app --port=5555 --persistent=True
directory=/home/app
stdout_logfile=/var/log/flower/flower.log
autostart=true
autorestart=true
user=celery
numprocs=1
startsecs=10
stopwaitsecs=600
```

Enable persistence (saves history):
```bash
flower -A celery_app --persistent=True --db=/data/flower.db
```

## Monitoring Tasks: Real-World Example

```python
# celery_app.py
from flower import Flower

app = Celery('pdf_service', broker='redis://localhost:6379')

app.conf.update(
    task_track_started=True,  # Let Flower see STARTED state
    task_send_sent_event=True,  # Send SENT event
)

@app.task(bind=True)
def convert_pdf(self, pdf_url: str):
    # Flower shows this task is STARTED
    self.update_state(state='STARTED')
    
    try:
        # Do work
        result = convert(pdf_url)
        
        # Return SUCCESS
        return result
    except Exception as e:
        # Task shows FAILURE with traceback
        raise

# API: integration with Flower monitoring
@app.get("/admin/tasks/active")
async def admin_active_tasks():
    """Show active tasks from Flower API"""
    response = requests.get('http://localhost:5555/api/tasks/active')
    return response.json()

@app.get("/admin/workers")
async def admin_workers():
    """Show worker status from Flower API"""
    response = requests.get('http://localhost:5555/api/workers')
    return response.json()

@app.get("/admin/queues")
async def admin_queues():
    """Show queue depth"""
    response = requests.get('http://localhost:5555/api/queues')
    return response.json()
```

## Alerting Based on Flower Metrics

```python
import requests

async def check_queue_health():
    """Alert if queue is growing"""
    response = requests.get('http://localhost:5555/api/queues')
    queues = response.json()
    
    for queue_name, stats in queues.items():
        messages = stats.get('messages', 0)
        
        # Alert if queue > 1000
        if messages > 1000:
            send_slack_alert(
                f"⚠️ Queue {queue_name} has {messages} pending tasks"
            )
        
        # Critical alert if queue > 5000
        if messages > 5000:
            send_pagerduty_alert(
                f"🚨 CRITICAL: Queue {queue_name} backlog: {messages}"
            )

async def check_worker_health():
    """Alert if workers offline"""
    response = requests.get('http://localhost:5555/api/workers')
    workers = response.json()
    
    expected_workers = 5
    online_workers = len(workers)
    
    if online_workers < expected_workers:
        send_slack_alert(
            f"⚠️ Only {online_workers}/{expected_workers} workers online"
        )

async def check_task_failures():
    """Alert on task failures"""
    response = requests.get('http://localhost:5555/api/tasks')
    tasks = response.json()
    
    failed_tasks = [t for t in tasks.values() if t['state'] == 'FAILURE']
    
    if len(failed_tasks) > 10:
        send_slack_alert(
            f"⚠️ {len(failed_tasks)} tasks failed in last hour"
        )

# Schedule health checks
app.conf.beat_schedule = {
    'check-queue-health': {
        'task': 'tasks.check_queue_health',
        'schedule': 300.0,  # Every 5 minutes
    },
    'check-worker-health': {
        'task': 'tasks.check_worker_health',
        'schedule': 600.0,  # Every 10 minutes
    },
    'check-failures': {
        'task': 'tasks.check_task_failures',
        'schedule': 300.0,  # Every 5 minutes
    },
}
```

## Hands-On Lab

### Lab 5.1: Flower Setup

```bash
# Terminal 1: Redis
docker run -d -p 6379:6379 redis:latest

# Terminal 2: Celery Worker
celery -A celery_app worker --loglevel=info

# Terminal 3: Flower
flower -A celery_app --port=5555

# Terminal 4: Generate tasks
python
>>> from celery_app import convert_pdf
>>> for i in range(10):
...     convert_pdf.delay(f"s3://file_{i}.pdf")

# Open browser: http://localhost:5555
# Click on Tasks tab to see active tasks
```

### Lab 5.2: Task Monitoring

```python
# celery_app.py
@app.task(bind=True)
def monitored_task(self, duration):
    self.update_state(state='PROGRESS', meta={'duration': duration})
    time.sleep(duration)
    return {"completed_in": duration}

# Generate task
task = monitored_task.delay(5)

# Check status in Flower
# http://localhost:5555 → Tasks → search for task.id
```

### Lab 5.3: API Integration

```python
# main.py
import requests

@app.get("/admin/dashboard")
async def admin_dashboard():
    """Get dashboard data from Flower"""
    tasks_response = requests.get('http://localhost:5555/api/tasks/active')
    workers_response = requests.get('http://localhost:5555/api/workers')
    
    return {
        "active_tasks": tasks_response.json(),
        "workers": workers_response.json(),
        "timestamp": datetime.now()
    }
```

## Cheat Sheet: Flower Monitoring

### Start Flower

```bash
flower -A celery_app --port=5555
```

### Web UI

- **http://localhost:5555/tasks**: active, completed, failed tasks
- **http://localhost:5555/workers**: worker health
- **http://localhost:5555/queues**: queue depth

### API

```
GET /api/tasks/active      # Currently processing
GET /api/tasks             # All tasks
GET /api/workers           # Worker status
GET /api/queues            # Queue depth
GET /api/pool/restart/     # Restart worker pool
```

### Task Actions

- **Retry**: re-queue failed task
- **Revoke**: cancel running task
- **View**: see details, arguments, result

## Key Takeaways

- **Flower = real-time Celery monitoring UI**
- **Web dashboard** at localhost:5555 (customizable)
- **Task history** — view state, arguments, results
- **Worker stats** — uptime, concurrency, CPU/memory
- **Queue depth** — see backlog, detect scaling needs
- **Retry failed tasks** — click button in Flower
- **API access** — programmatic monitoring
- **Alerting** — query Flower API, send alerts if unhealthy

**Congratulations!** You now understand Celery end-to-end: tasks, routing, retries, scheduling, and monitoring.

Module complete. All Celery fundamentals covered.
