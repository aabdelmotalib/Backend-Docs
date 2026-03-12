# Module 4: Celery Beat - Periodic Tasks

## The Analogy: Scheduled Maintenance

Your apartment:
- Vacuuming: do it weekly (every Monday)
- Watering plants: do it daily at 9 AM
- Cleaning gutters: do it quarterly
- Take out trash: do it every Sunday at 6 PM

Celery Beat:
- Periodic tasks: scheduled, not triggered by user
- Schedules: run every N minutes, hourly, daily, or at specific time
- Cron patterns: like Linux cron jobs
- Beat scheduler: enables scheduling

## Without Scheduling: Manual Cron

```bash
# Linux crontab
0 9 * * * /usr/bin/python /app/tasks/daily_report.py  # 9 AM daily
0 0 1 * * /usr/bin/python /app/tasks/monthly_cleanup.py  # 1st of month
```

Problems:
- Not integrated with Celery
- Separate system to maintain
- Hard to monitor
- No retry logic

## With Celery Beat: Integrated Scheduling

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    'send-daily-report': {
        'task': 'tasks.send_daily_report',
        'schedule': crontab(hour=9, minute=0),  # Every day at 9 AM
    },
    'cleanup-old-files': {
        'task': 'tasks.cleanup_old_files',
        'schedule': 3600.0,  # Every hour (3600 seconds)
    },
    'generate-monthly-summary': {
        'task': 'tasks.generate_monthly_summary',
        'schedule': crontab(hour=0, minute=0, day_of_month=1),  # 1st of month at midnight
    },
}

@app.task
def send_daily_report():
    # Task runs automatically every day at 9 AM
    ...

@app.task
def cleanup_old_files():
    # Task runs automatically every hour
    ...
```

## Scheduling Options

### Numeric Intervals

```python
from celery.schedules import schedule

app.conf.beat_schedule = {
    'every-10-seconds': {
        'task': 'tasks.check_health',
        'schedule': 10.0,  # Run every 10 seconds
    },
    'every-5-minutes': {
        'task': 'tasks.refresh_cache',
        'schedule': 300.0,  # Every 5 minutes
    },
    'every-hour': {
        'task': 'tasks.hourly_sync',
        'schedule': 3600.0,  # Every hour
    },
}
```

### Crontab Patterns

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    # 9 AM every day
    'daily-at-9am': {
        'task': 'tasks.morning_report',
        'schedule': crontab(hour=9, minute=0),
    },
    
    # Every Monday at 5 PM
    'weekly-status': {
        'task': 'tasks.weekly_report',
        'schedule': crontab(hour=17, minute=0, day_of_week=0),  # 0=Monday
    },
    
    # 1st of month at 12 AM
    'monthly-cleanup': {
        'task': 'tasks.monthly_cleanup',
        'schedule': crontab(hour=0, minute=0, day_of_month=1),
    },
    
    # Every 15 minutes
    'every-15-min': {
        'task': 'tasks.sync_database',
        'schedule': crontab(minute='*/15'),  # Every 15 minutes
    },
    
    # Every 2 hours
    'every-2-hours': {
        'task': 'tasks.process_queue',
        'schedule': crontab(minute=0, hour='*/2'),
    },
    
    # 6 AM to 6 PM every hour
    'business-hours': {
        'task': 'tasks.check_alerts',
        'schedule': crontab(hour='6-18'),
    },
    
    # Every minute (not recommended for heavy tasks)
    'every-minute': {
        'task': 'tasks.monitor_health',
        'schedule': crontab(minute='*'),
    },
}
```

## Real-World Examples: PDF Service

```python
app.conf.beat_schedule = {
    # Clean up old conversion logs (midnight daily)
    'cleanup-logs': {
        'task': 'tasks.cleanup_old_logs',
        'schedule': crontab(hour=0, minute=0),
    },
    
    # Generate conversion report (9 AM weekdays)
    'daily-report': {
        'task': 'tasks.daily_conversion_report',
        'schedule': crontab(hour=9, minute=0, day_of_week='0-4'),
    },
    
    # Retry failed conversions (every hour)
    'retry-failures': {
        'task': 'tasks.retry_failed_conversions',
        'schedule': 3600.0,
    },
    
    # Check service health (every 5 minutes)
    'health-check': {
        'task': 'tasks.check_service_health',
        'schedule': 300.0,
    },
    
    # Sync cache with database (every 30 minutes)
    'refresh-cache': {
        'task': 'tasks.refresh_conversion_cache',
        'schedule': 1800.0,
    },
    
    # Generate monthly summary (1st of month at 1 AM)
    'monthly-summary': {
        'task': 'tasks.generate_monthly_summary',
        'schedule': crontab(hour=1, minute=0, day_of_month=1),
    },
}

@app.task
def cleanup_old_logs():
    """Delete logs older than 30 days"""
    from datetime import datetime, timedelta
    cutoff = datetime.now() - timedelta(days=30)
    db.query(ConversionLog).filter(ConversionLog.created_at < cutoff).delete()
    db.commit()

@app.task
def daily_conversion_report():
    """Email summary of conversions"""
    from datetime import datetime, timedelta
    today = datetime.now().date()
    
    stats = db.query(func.count(ConversionJob)).filter(
        ConversionJob.created_at >= today
    ).scalar()
    
    send_email(
        to='admin@example.com',
        subject=f'Conversion Report - {today}',
        body=f'Total conversions: {stats}'
    )

@app.task
def retry_failed_conversions():
    """Retry any failed conversion jobs"""
    failed = db.query(ConversionJob).filter_by(status='failed').all()
    for job in failed:
        # Re-queue
        convert_pdf.delay(job.pdf_url, job.user_id)
        # Mark as retried
        job.status = 'retrying'
        db.commit()

@app.task
def check_service_health():
    """Verify all services are healthy"""
    health_status = {
        'redis': check_redis(),
        'database': check_database(),
        'worker': check_workers(),
    }
    
    if not all(health_status.values()):
        send_alert(f"Service health check failed: {health_status}")

@app.task
def refresh_conversion_cache():
    """Sync Redis cache with database"""
    # Get recent conversions
    recent = db.query(ConversionJob).filter(
        ConversionJob.created_at >= datetime.now() - timedelta(hours=1)
    ).all()
    
    # Update cache
    for job in recent:
        redis.setex(
            f"job:{job.id}",
            3600,
            json.dumps({
                "status": job.status,
                "created": str(job.created_at)
            })
        )
```

## Running Celery Beat

**Single Process** (development):

```bash
# Celery Beat + Worker in single process (not for production)
celery -A app worker --beat --loglevel=info
```

**Separate Processes** (production):

```bash
# Terminal 1: Celery Beat (scheduler)
celery -A app beat --loglevel=info

# Terminal 2: Worker (executes scheduled tasks)
celery -A app worker --loglevel=info
```

Beat should run on only ONE machine. Multiple Beat instances causes duplicate scheduling.

## Monitoring Scheduled Tasks

```python
# Check next scheduled task
from celery.management import inspect

app_inspect = inspect.Inspect()
active = app_inspect.active()
scheduled = app_inspect.scheduled()

print(f"Scheduled tasks: {scheduled}")
```

Or use Flower (Module 5).

## Hands-On Lab

### Lab 4.1: Hourly Task

**celery_app.py**
```python
from celery import Celery
from celery.schedules import crontab
from datetime import datetime

app = Celery('app', broker='redis://localhost:6379')

app.conf.beat_schedule = {
    'every-10-seconds': {
        'task': 'tasks.heartbeat',
        'schedule': 10.0,
    },
}

@app.task
def heartbeat():
    print(f"Heartbeat at {datetime.now()}")
    return {"timestamp": str(datetime.now())}
```

**Run**
```bash
# Terminal 1: Redis
docker run -d -p 6379:6379 redis:latest

# Terminal 2: Beat scheduler
celery -A celery_app beat --loglevel=info

# Terminal 3: Worker
celery -A celery_app worker --loglevel=info

# Watch heartbeat task run every 10 seconds
```

### Lab 4.2: Daily Report

```python
app.conf.beat_schedule = {
    'daily-report': {
        'task': 'tasks.daily_report',
        'schedule': crontab(hour=14, minute=0),  # 2 PM
    },
}

@app.task
def daily_report():
    report = {
        "timestamp": str(datetime.now()),
        "tasks_completed": 100,
        "errors": 5
    }
    print(f"Daily report: {report}")
    return report
```

## Cheat Sheet: Celery Beat

### Simple Schedule

```python
app.conf.beat_schedule = {
    'task-name': {
        'task': 'tasks.my_task',
        'schedule': 300.0,  # Every 5 minutes
    },
}
```

### Crontab Schedule

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    'task-name': {
        'task': 'tasks.my_task',
        'schedule': crontab(hour=9, minute=0),  # 9 AM daily
    },
}
```

### Run

```bash
celery -A app beat
celery -A app worker
```

## Key Takeaways

- **Celery Beat = integrated task scheduler**
- **Crontab patterns = familiar from Linux cron**
- **Schedules = intervals or crontab expressions**
- **Single Beat instance** — runs on only one machine
- **Worker executes** — Beat queues, worker processes
- **Monitoring** — check Flower for scheduled task status
- **Next module** teaches Flower UI for monitoring

Module 5 covers monitoring with Flower.
