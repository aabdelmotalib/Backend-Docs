# Module 7: Observability - Logs, Metrics, Traces

## The Challenge: Production is Dark

Your API is running in production. A user's request is slow. Where's the bottleneck?

```
User: "Upload PDF is slow"
You: "Let's add logging"
You: "Let's capture metrics"
You: "Let's trace requests"
```

Without observability, you're blind.

## Three Pillars of Observability

### Logs: What Happened?

Detailed record of events.

```
2024-01-15 10:23:45 INFO User logged in id=user_123
2024-01-15 10:23:46 INFO Started PDF conversion pdf_id=pdf_456
2024-01-15 10:23:50 ERROR PDF conversion failed timeout after 5s
2024-01-15 10:23:51 INFO Retrying PDF conversion
```

Stores events in files or log aggregation (ELK, Datadog).

### Metrics: How Often? How Long?

Aggregated numbers.

```
request_count: 1000 (total requests processed)
request_duration_p50: 100ms (50th percentile)
request_duration_p95: 500ms (95th percentile)
request_duration_p99: 2000ms (99th percentile)
queue_depth: 50 (tasks waiting)
db_connection_pool_available: 5/10
```

Stores in time-series database (Prometheus, Datadog).

### Traces: How Did We Get Here?

Request spanning multiple services.

```
Request ID: req_789

[API-1] GET /api/documents/123 (10ms)
  ├─ [PostgreSQL] SELECT * FROM documents (5ms)
  ├─ [Redis] GET document:123 (1ms, miss)
  ├─ [PostgreSQL] Fallback query (4ms)
  ├─ [Redis] SET document:123 (0.1ms)
  └─ Response: 200 OK (10ms total)
```

Stores in trace database (Jaeger, Datadog).

## Structured Logging

### Plain Text Logs (Bad)

```
2024-01-15 10:23:45 User user_123 logged in from 203.0.113.50
2024-01-15 10:23:46 Started PDF conversion
2024-01-15 10:23:50 Error: timeout
```

Hard to parse. Can't aggregate. Can't alert.

### Structured JSON Logs (Good)

```json
{
  "timestamp": "2024-01-15T10:23:45Z",
  "level": "INFO",
  "user_id": "user_123",
  "event": "login",
  "ip_address": "203.0.113.50",
  "request_id": "req_123"
}

{
  "timestamp": "2024-01-15T10:23:46Z",
  "level": "INFO",
  "event": "pdf_conversion_started",
  "pdf_id": "pdf_456",
  "user_id": "user_123",
  "request_id": "req_123"
}

{
  "timestamp": "2024-01-15T10:23:50Z",
  "level": "ERROR",
  "event": "pdf_conversion_failed",
  "pdf_id": "pdf_456",
  "duration_ms": 4000,
  "error": "timeout after 5s",
  "request_id": "req_123"
}
```

Parseable. Queryable. Alertable.

### JSON Logging in Python

```python
from pythonjsonlogger import jsonlogger
import logging

# Configure logger
logger = logging.getLogger()
handler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter()
handler.setFormatter(formatter)
logger.addHandler(handler)

# Log with structured data
logger.info("user_login", extra={
    "user_id": "user_123",
    "ip_address": "203.0.113.50",
    "request_id": request_id
})
```

## Application Metrics

Track operational health.

```python
from prometheus_client import Counter, Histogram, Gauge

# Count
request_count = Counter('http_requests_total', 'Total requests')

# Histogram (distribution)
request_duration = Histogram(
    'http_request_duration_seconds',
    'Request duration in seconds',
    buckets=[0.01, 0.1, 0.5, 1.0, 5.0]
)

# Gauge (current value)
queue_depth = Gauge('celery_queue_depth', 'Tasks in queue')

# Usage
@app.post("/api/documents/upload")
async def upload(file: UploadFile):
    start = time.time()
    
    try:
        # Process
        result = process_pdf(file)
        request_count.labels(method="POST", endpoint="/upload", status=200).inc()
    except Exception as e:
        request_count.labels(method="POST", endpoint="/upload", status=500).inc()
    finally:
        duration = time.time() - start
        request_duration.observe(duration)
    
    # Update gauge
    queue_size = celery_app.control.inspect().active()
    queue_depth.set(queue_size)
    
    return result
```

### Key Metrics for Distributed Systems

```
API:
  - request_count (by endpoint, status code, method)
  - request_duration (p50, p95, p99)
  - error_rate (5xx errors / total)

Database:
  - query_duration (p50, p95, p99)
  - active_connections
  - slow_queries (> 1s)

Cache:
  - hit_rate (hits / total requests)
  - miss_rate
  - eviction_rate

Queue:
  - queue_depth (tasks waiting)
  - processing_rate (tasks/second)
  - average_delay (time from queue to start)

System:
  - cpu_usage
  - memory_usage
  - disk_usage
```

## Distributed Tracing

Request spans multiple services. Hard to debug without tracing.

### X-Request-ID

Every request gets unique ID. Propagate through logs and services.

```python
import uuid

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    # Get or generate request ID
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    
    # Attach to request context
    request.state.request_id = request_id
    
    # Call next handler
    response = await call_next(request)
    
    # Add to response headers
    response.headers["X-Request-ID"] = request_id
    
    return response

# Usage in logs
@app.post("/api/documents")
def create_document(current_user, request: Request):
    request_id = request.state.request_id
    
    logger.info("document_created", extra={
        "request_id": request_id,
        "user_id": current_user.id
    })
```

### Full Distributed Trace

```
Request: POST /api/documents/upload
Request ID: req_789

Client logs:
  [req_789] POST /api/documents/upload started

API-1 logs:
  [req_789] Received upload, validating file
  [req_789] File validated, queueing task
  [req_789] Task queued, returning 202

Worker logs (async):
  [req_789] Processing pdf_456
  [req_789] Calling PostgreSQL
  [req_789] PostgreSQL responded in 5ms
  [req_789] Scanning with ClamAV
  [req_789] ClamAV responded in 2000ms
  [req_789] Extraction complete, storing result
  [req_789] Processing complete

All logs are correlated by req_789
```

### Jaeger Tracing (Advanced)

Visualize request across services:

```python
from jaeger_client import Config

# Initialize tracer
jaeger_config = Config(
    config={'sampler': {'type': 'const', 'param': 1}},
    service_name='pdf-api',
)
tracer = jaeger_config.initialize_tracer()

# Trace a request
with tracer.start_active_span('upload_document') as scope:
    with tracer.start_active_span('validate_file'):
        validate(file)
    
    with tracer.start_active_span('queue_task'):
        queue_conversion_task(pdf_id)
    
    # Span includes duration, parent/child relationships
```

## Hands-On Lab

### Lab 7.1: Structured JSON Logging

```python
import json
import logging
import sys
from datetime import datetime

# Simple JSON logger
class JSONLogger:
    def log(self, level, event, **kwargs):
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": level,
            "event": event,
            **kwargs
        }
        print(json.dumps(log_entry), file=sys.stdout)

logger = JSONLogger()

# Log events
logger.log("INFO", "user_login", user_id="user_123", ip="203.0.113.50")
logger.log("INFO", "pdf_conversion_started", pdf_id="pdf_456", user_id="user_123")
logger.log("ERROR", "pdf_conversion_failed", pdf_id="pdf_456", duration_ms=5000)

# Output
# {"timestamp": "2024-01-15T10:23:45.123456Z", "level": "INFO", "event": "user_login", "user_id": "user_123", "ip": "203.0.113.50"}
# {"timestamp": "2024-01-15T10:23:46.234567Z", "level": "INFO", "event": "pdf_conversion_started", "pdf_id": "pdf_456", "user_id": "user_123"}
# {"timestamp": "2024-01-15T10:23:50.345678Z", "level": "ERROR", "event": "pdf_conversion_failed", "pdf_id": "pdf_456", "duration_ms": 5000}
```

### Lab 7.2: Request Correlation with X-Request-ID

```bash
# Make request with generated ID
curl -H "X-Request-ID: req_scenario_1" http://localhost:8000/api/documents

# Check logs for req_scenario_1
docker-compose logs -f api | grep req_scenario_1

# All logs for this request contain req_scenario_1
# Can correlate API logs, database logs, cache logs
```

### Lab 7.3: Metrics with Prometheus

```bash
# Install Prometheus
docker-compose up prometheus

# Visit http://localhost:9090

# Query metrics in Prometheus dashboard
# http_requests_total
# http_request_duration_seconds
# redis_connection_pool_available

# See request latency over time:
# histogram_quantile(0.95, http_request_duration_seconds)
```

## Cheat Sheet: Observability

```python
# Structured JSON logging
logger.info("event", extra={
    "user_id": user_id,
    "request_id": request_id,
    "duration_ms": duration
})

# Metrics
from prometheus_client import Counter, Histogram, Gauge

requests = Counter('http_requests_total', 'Total requests')
duration = Histogram('request_duration_seconds', 'Request duration')
queue = Gauge('queue_depth', 'Tasks in queue')

# X-Request-ID propagation
request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
logger.info("event", extra={"request_id": request_id})

# Pass to downstream services
headers = {"X-Request-ID": request_id}
requests.get("http://api:8000", headers=headers)
```

## Key Takeaways

- **Logs** = what happened (events, errors)
- **Metrics** = how much, how long, how often
- **Traces** = request path across services
- **JSON logs** = parseable, queryable, aggregatable
- **X-Request-ID** = correlate logs across services
- **Prometheus** = metrics storage and dashboards
- **Jaeger** = distributed tracing (advanced)
- **Percentiles** (p50, p95, p99) better than averages
- **Alerting** on metrics (queue > 1000, error_rate > 5%)

## Complete Production Observability Stack

```
Application Logs
  ├─ JSON format
  ├─ X-Request-ID correlation
  ├─ Aggregator (ELK, Datadog, Loki)

Metrics
  ├─ Prometheus client (app)
  ├─ Prometheus server (scraper)
  ├─ Grafana (dashboard)

Traces
  ├─ Jaeger SDK (instrumentation)
  ├─ Jaeger backend (storage)
  ├─ Jaeger UI (visualization)

Alerting
  ├─ Prometheus AlertManager
  ├─ Datadog alerts
  ├─ PagerDuty (on-call)
```

---

**Congratulations!** You've learned distributed systems fundamentals:
1. What a distributed system is (failures, latency, caps)
2. Message queues (at-least-once, idempotency)
3. CAP theorem (consistency vs availability vs partition)
4. Distributed locks (prevent races)
5. Idempotency (safe retries)
6. Eventual consistency (cache sync)
7. Observability (logs, metrics, traces)

Apply these patterns to your PDF platform. Start with strong consistency (PostgreSQL) for user data, eventual consistency (Redis) for cache, at-least-once Celery tasks with idempotency keys. Monitor with JSON logs and Prometheus. Your system will be reliable at scale.
