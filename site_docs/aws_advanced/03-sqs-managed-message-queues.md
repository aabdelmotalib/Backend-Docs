# Module 3: SQS - AWS Managed Message Queues

## What Is SQS?

SQS (Simple Queue Service) = managed message queue on AWS.

It's an alternative to Redis as your Celery broker.

**Current setup** (Redis broker):
```python
# Celery uses Redis to queue tasks
CELERY_BROKER_URL = "redis://localhost:6379"

@app.task
def process_pdf(file_id):
    # Task goes to Redis queue
    # Celery worker picks it up
```

**With SQS** (AWS managed):
```python
# Celery uses AWS SQS to queue tasks
CELERY_BROKER_URL = "sqs://your-queue"

@app.task
def process_pdf(file_id):
    # Task goes to SQS queue
    # Celery worker picks it up
    # AWS manages queue (no Redis needed)
```

## Why SQS Instead of Redis?

You pick one of:
1. **ElastiCache Redis** ($14-28/month, you manage connection string)
2. **SQS Queue** ($0.40 per million messages, scaled pay-per-use)

**Choose SQS when**:
- Async tasks are occasional (cost-efficient)
- Decoupling critical (producer/consumer separate)
- AWS ecosystem (SNS, Lambda integration)
- Durability critical (all messages persisted)

**Choose Redis when**:
- High throughput (>100k msgs/min)
- Cache + queue together (same tool)
- Existing Redis expertise

**For PDF platform**: Both valid. SQS simpler if low-volume, Redis if high-volume.

## SQS Queue Types

### Standard Queue

**Default**: At-least-once delivery, any order.

```
Upload PDF → Queue:
  {"file_id": 123, ...}
  {"file_id": 124, ...}
  {"file_id": 125, ...}

Celery worker polls → Gets message
Process → Deletes message from queue
If worker crashes → Message visible again (retried)
```

Works if: Duplicate processing OK (idempotent operations).

### FIFO Queue

Exactly-once, ordered delivery. Slower.

```
Upload PDF (timestamp) → Queue (FIFO, immutable order):
  {"file_id": 123, ...}  ← Process first
  {"file_id": 124, ...}  ← Process second
  {"file_id": 125, ...}  ← Process third

Celery worker processes in order.
```

Works if: Order matters, exactly-once required.

**For PDF platform**: Standard queue sufficient (PDF processing order doesn't matter).

## How SQS Works

```
1. Producer (API) sends message:
   API → SQS Queue (stored, persisted, encrypted)

2. Consumer (Celery worker) polls queue:
   Worker → SQS: "Give me a message"
   SQS → Worker: {"file_id": 123, "user_id": 456, ...}

3. Visibility Timeout (30 seconds default):
   Message hidden for 30s while worker processes
   
4a. Worker finishes:
    Worker → SQS: "Delete this message"
    SQS → Deleted
   
4b. Worker crashes:
    Worker dies (no delete)
    After 30s timeout → Message visible again
    Another worker picks it up
```

## Dead Letter Queue (DLQ)

For messages that fail repeatedly:

```
Worker tries to process → Fails
Retry 1 → Fails
Retry 2 → Fails
Retry 3 → Fails
Max retries reached → Send to DLQ

DLQ = separate queue for failed messages
Manual inspection: Why did this fail?
Fix issue → Manually re-process from DLQ
```

Setup:

```bash
# Create main queue
aws sqs create-queue --queue-name pdf-process-queue

# Create DLQ
aws sqs create-queue --queue-name pdf-process-queue-dlq

# Set DLQ on main queue (maxReceiveCount=3)
# After 3 failures, message goes to DLQ
```

## Celery + SQS Configuration

### Installation

```bash
pip install celery[sqs] boto3
```

### Django Settings

```python
# settings.py

CELERY_BROKER_URL = "sqs://"  # Uses boto3 credentials
CELERY_BROKER_TRANSPORT_OPTIONS = {
    "region": "eu-west-1",
    "queue_name_prefix": "pdf-",  # All queues prefixed
}

CELERY_TASK_DEFAULT_QUEUE = "pdf-process"
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_BACKEND = "redis://..."  # Store results in Redis still
```

### Task Definition

```python
from celery import shared_task

@shared_task
def process_pdf(file_id: int):
    """
    SQS message format:
    {
        "file_id": 123,
        "user_id": 456,
        ...
    }
    """
    file = File.objects.get(id=file_id)
    file.extract_text()
    file.save()
```

### Celery Worker

```bash
# Worker reads from SQS queue
celery -A myapp worker --loglevel=info
```

Behind the scenes:
- Worker polls SQS every 1 second
- Gets message → executes task
- On success → deletes message
- On failure → retries (configurable)

## SQS + Lambda (Advanced)

SQS can trigger Lambda:

```
S3 upload PDF
  ↓
SQS message queued
  ↓
Lambda triggered (event source mapping)
  ↓
Lambda runs Python function (serverless)
  ↓
Extract text → RDS
```

Versus Celery: Lambda for occasional work (cheaper), Celery for continuous workers.

## Hands-On Lab

### Lab 3.1: Create SQS Queue

```bash
# 1. Create main queue
QUEUE=$(aws sqs create-queue \
  --queue-name pdf-process-queue \
  --attributes MessageRetentionPeriod=86400,VisibilityTimeout=300 \
  --region eu-west-1 \
  --query 'QueueUrl' \
  --output text)

echo $QUEUE
# https://sqs.eu-west-1.amazonaws.com/123456789012/pdf-process-queue

# 2. Create DLQ
DLQ=$(aws sqs create-queue \
  --queue-name pdf-process-queue-dlq \
  --region eu-west-1 \
  --query 'QueueUrl' \
  --output text)

# 3. Get queue ARN
DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url $DLQ \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

# 4. Set DLQ policy on main queue
aws sqs set-queue-attributes \
  --queue-url $QUEUE \
  --attributes RedrivePolicy="{\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"3\"}" \
  --region eu-west-1

# Now: after 3 failures, message goes to DLQ
```

### Lab 3.2: Send/Receive Messages

```bash
# 1. Send message to queue
aws sqs send-message \
  --queue-url $QUEUE \
  --message-body '{"file_id": 123, "user_id": 456}' \
  --region eu-west-1

# Response: MessageId, MD5OfMessageBody

# 2. Receive message
aws sqs receive-message \
  --queue-url $QUEUE \
  --max-number-of-messages 1 \
  --visibility-timeout 30 \
  --region eu-west-1

# Response:
# {
#   "Messages": [
#     {
#       "MessageId": "12345",
#       "ReceiptHandle": "abc123...",
#       "Body": "{\"file_id\": 123, ...}"
#     }
#   ]
# }

# 3. Process message (in your code)
def process_message(body):
    data = json.loads(body)
    process_pdf(data["file_id"])

# 4. Delete message after processing
aws sqs delete-message \
  --queue-url $QUEUE \
  --receipt-handle "abc123..." \
  --region eu-west-1

# 5. Queue statistics
aws sqs get-queue-attributes \
  --queue-url $QUEUE \
  --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible,ApproximateNumberOfMessagesDelayed
```

### Lab 3.3: Celery + SQS Integration

```bash
# 1. Install dependencies
pip install celery[sqs] boto3

# 2. Create celery.py in your Django project
cat > myapp/celery.py << 'EOF'
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myapp.settings')

app = Celery('myapp')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
EOF

# 3. Update settings.py
cat >> myapp/settings.py << 'EOF'

# Celery + SQS (boto3 uses IAM role for credentials)
CELERY_BROKER_URL = "sqs://"
CELERY_BROKER_TRANSPORT_OPTIONS = {
    "region": "eu-west-1",
    "queue_name_prefix": "pdf-",
    "visibility_timeout": 300,  # 5min for task processing
}
CELERY_RESULT_BACKEND = "redis://elasticache-endpoint:6379"
CELERY_TASK_SERIALIZER = "json"
CELERY_ACCEPT_CONTENT = ["json"]
EOF

# 4. Create a task
cat > myapp/tasks.py << 'EOF'
from celery import shared_task

@shared_task
def process_pdf(file_id):
    """Process PDF asynchronously via SQS."""
    from myapp.models import File
    file = File.objects.get(id=file_id)
    file.extract_text()
    file.save()
    return f"Processed file {file_id}"
EOF

# 5. Call task from API (sends to SQS)
cat > myapp/views.py << 'EOF'
from django.http import JsonResponse
from myapp.tasks import process_pdf

def upload_pdf(request):
    file = request.FILES['file']
    # Save file
    file_obj = File.objects.create(file=file)
    
    # Send to SQS queue (task queued, not executed yet)
    process_pdf.delay(file_obj.id)
    
    return JsonResponse({"status": "processing", "file_id": file_obj.id})
EOF

# 6. Start Celery worker (reads from SQS)
celery -A myapp worker --loglevel=info

# Behind scenes:
# - Worker polls SQS queue
# - Gets message: {"file_id": 123}
# - Executes process_pdf(123)
# - Deletes message from SQS
# - Repeats
```

## Cheat Sheet: SQS Commands

```bash
# Create standard queue
aws sqs create-queue --queue-name my-queue

# Create FIFO queue (slower, exactly-once)
aws sqs create-queue --queue-name my-queue.fifo \
  --attributes FifoQueue=true

# Send message
aws sqs send-message --queue-url <QUEUE_URL> \
  --message-body '{"key": "value"}'

# Receive message
aws sqs receive-message --queue-url <QUEUE_URL> \
  --max-number-of-messages 10

# Delete message (after processing)
aws sqs delete-message --queue-url <QUEUE_URL> \
  --receipt-handle <RECEIPT_HANDLE>

# Check queue depth
aws sqs get-queue-attributes --queue-url <QUEUE_URL> \
  --attribute-names ApproximateNumberOfMessages

# Purge queue (delete all messages)
aws sqs purge-queue --queue-url <QUEUE_URL>

# Delete queue
aws sqs delete-queue --queue-url <QUEUE_URL>

# Celery broker URL
CELERY_BROKER_URL = "sqs://"  # Auto-uses IAM credentials
```

## Key Takeaways

- **SQS** = managed message queue (alternative to Redis as Celery broker)
- **Standard queue** = at-least-once, any order (typical)
- **FIFO queue** = exactly-once, ordered (slower, specific use cases)
- **Visibility timeout** = message hidden while processing (30-300 seconds typical)
- **Dead Letter Queue** = failed messages after max retries
- **Celery + SQS** = no Redis broker needed, AWS manages queue persistence
- **Cost**: $0.40 per million messages (very cheap for low-volume)
- **Advantage over Redis**: Durability (persisted), scaling (no size limits), AWS integration

Module 4 teaches IAM — secure credential management.
