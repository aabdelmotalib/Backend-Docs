# Module 7: Scaling This Platform on AWS

## The Migration Path: When and Why

**Hetzner CX21 handles**:
- ~100 concurrent users
- ~50 PDF conversions/hour
- Cost: €8.50/month

**Limits you'll hit**:
- 4 vCPU maxes out under heavy load
- 8GB RAM → memory pressure when all services run
- Single point of failure (server network goes down? Offline)
- Can't scale Celery workers independently (share same machine)

**When to migrate to AWS**:
- >200 concurrent users
- >100 PDF conversions/hour
- Need high availability (99.9% uptime SLA)
- Want to auto-scale during peak hours

---

## Phase 1: Same Scale, AWS Infrastructure

Replicate Hetzner setup on AWS, but managed services (easier ops).

### Cost Comparison: Hetzner vs AWS Phase 1

```
Hetzner CX21:
  - Single VPS: €8.50/month = $10/month
  Total: $10/month

AWS Phase 1 (equivalent):
  - EC2 t3.large: $0.083/hour = $60/month
  - RDS db.t3.large: $0.165/hour = $120/month
  - ElastiCache cache.t3.small: $0.017/hour = $12/month
  - ELB (load balancer): $16/month
  - Data transfer: $20/month
  Total: ~$230/month (23x more expensive!)
  
BUT: You get
  ✓ Multi-AZ RDS (automatic failover < 1min)
  ✓ Managed backups (point-in-time recovery)
  ✓ Auto-scaling RDS
  ✓ CloudWatch monitoring
  ✓ No ops (AWS patches everything)
```

**Phase 1 Decision**: Worth it if uptime SLA required, or staff cost saved.

### Phase 1 Architecture

```
┌─────────────────────────────────────────────┐
│          AWS EC2 (Hetzner equivalent)        │
│          - FastAPI + Nginx in container      │
│          - Celery worker                     │
│          - SingletonGPU for LibreOffice      │
└─────────────────────────────────────────────┘
                      ↓
          ┌──────────┼──────────┐
          ↓          ↓          ↓
      ┌────────┐ ┌────────┐ ┌─────────┐
      │  RDS   │ │ElastiC │ │   S3    │
      │Postgres│ │ache    │ │(MinIO→) │
      │        │ │Redis   │ │         │
      └────────┘ └────────┘ └─────────┘
          ↓
      ┌────────┐
      │   ALB  │
      │(public)│
      └────────┘
          ↓
      DNS → pdf-platform.example.com
      (Route53 or Namecheap)
```

**Key changes**:
1. **Nginx + FastAPI**: Still on EC2 (just like Hetzner)
2. **PostgreSQL**: RDS managed (not Docker container)
3. **Redis**: ElastiCache managed (not Docker container)
4. **MinIO**: S3 (actually use AWS S3, not MinIO)
5. **Load Balancer**: ALB (for future multi-AZ)

---

## Phase 2: Scale Out (Multiple Instances)

Users increasing. Single EC2 maxes out. Need multiple.

### ECS Fargate: Container Orchestration

Move from "single EC2 running containers" to "ECS managing fleet of containers".

```
Desired state: 2-10 Fargate tasks (containers)

Load increases → ALB distributes
  ↓
ECS auto-scaling detects CPU > 70%
  ↓
Add 2 more tasks (Fargate spins up in 30 seconds)
  ↓
Load decreased → CPU < 30%
  ↓
Scale down to 3 tasks (cost savings)
```

**ECR**: Docker registry (push images there).

**Task definition**: Like docker-compose.yml, but on AWS.

```json
{
  "family": "pdf-api",
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [{
    "name": "api",
    "image": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/pdf-api:latest",
    "portMappings": [{"containerPort": 8000}],
    "environment": [
      {"name": "DATABASE_URL", "value": "postgres://rds-endpoint:5432..."},
      {"name": "REDIS_URL", "value": "redis://elasticache-endpoint:6379"},
      {"name": "CELERY_BROKER_URL", "value": "sqs://"}
    ]
  }]
}
```

**ECS Service**: "Run 2-10 tasks, restart failed ones, balance load".

```bash
aws ecs create-service \
  --cluster pdf-platform \
  --service-name api \
  --task-definition pdf-api:1 \
  --desired-count 2 \
  --launch-type FARGATE
```

**Auto-Scaling**: Scale based on metrics.

```bash
aws application-autoscaling put-scaling-policy \
  --policy-name api-scale-policy \
  --service-namespace ecs \
  --resource-id service/pdf-platform/api \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

### Worker Scaling: SQS Depth

Instead of Redis queue on single machine, use SQS (AWS managed).

```
Celery config:
CELERY_BROKER_URL = "sqs://..."

When task enqueued:
  api → SQS queue → Workers consume
  
When queue depth > 10:
  ECS auto-scaling adds workers
  
When queue depth < 2:
  Scale down workers
```

---

## Phase 2 Cost

```
Phase 2: 2-10 Fargate tasks + SQS + RDS + read replicas

Base:
- RDS db.t3.xlarge (xDBinstance): $0.33/hour = $240/month
- ElastiCache: $12/month
- ALB: $16/month
- NAT Gateway: $32/month
- Data transfer: $50/month

API tasks (2-10, average 4):
- Fargate: 1vCPU × $0.05/hour = $0.05/hour × 730 hours × 4 tasks = $146/month

Workers (1-3, average 2):
- Fargate: 0.5vCPU × $0.025/hour = ~$37/month × 2 = $74/month

SQS queue:
- $0.40 per million messages
- 100,000 messages/day = ~$12/month

Total: ~$440/month
(5x Hetzner, 2x Phase 1)

BUT: You're at 500+ concurrent users, 99.99% uptime, auto-scaling → wins at volume.
```

---

## Phase 3: Global Scale (Read Replicas + Multi-Region)

Traffic from Europe and USA. Latency issues.

### RDS Read Replicas

Create read-only copies for `/jobs/{id}/status` polling (heavy read).

```
Primary (writes):
  - User sessions (/session/status writes)
  - Subscription creation (writes)
  
Read replica:
  - Poll job status (read-only, heavy)
  - Aggregate reports (read-only)

PostgreSQL replication:
  Primary → async write → Replica
  Latency: 100-500ms (tolerable)
```

**Cost**: Read replica = half price of primary.
- Primary: $240/month
- 1 Read replica: +$120/month
- Total RDS: $360/month

### CloudFront CDN

Cache static files (React build, CSS, JS, images).

```
React build output → S3 → CloudFront
  ↓
User in USA requests /index.html
  ↓
CloudFront edge location (nearest to user)
  returns cached file (< 100ms)
  ↓
No backend roundtrip needed
```

**Cost**: CloudFront = $0.085/GB (expensive for large files, cheap for static).

For 10GB/day static serve:
- $0.085 × 10GB × 30 days = $25.50/month

---

## Phase 3+ Architecture

```
┌──────────────────────────────────────────┐
│         CloudFront CDN                   │
│    (static files cached globally)        │
└──────────────────────────────────────────┘
          ↓
  ┌───────────────────────────────────────┐
  │           Route53 DNS                 │
  │    (geolocation routing: USA→ALB-US, │
  │     EU→ALB-EU)                       │
  └───────────────────────────────────────┘
      ↓ (USA)            ↓ (EU)
┌────────────────┐  ┌────────────────┐
│ ALB US-EAST    │  │ ALB EU-WEST    │
│ (load balance) │  │ (load balance) │
└────────────────┘  └────────────────┘
      ↓                   ↓
┌────────────────┐  ┌────────────────┐
│ ECS US tasks   │  │ ECS EU tasks   │
│ (2-10)         │  │ (2-10)         │
└────────────────┘  └────────────────┘
      ↓                   ↓
  ┌─────────────────────────────┐
  │   RDS Primary (EU)          │
  │   (writes, +read replica)   │
  ├─────────────────────────────┤
  │   DynamoDB (multi-region)   │
  │   (sessions, cache)         │
  └─────────────────────────────┘
      ↓
  ┌─────────────────────────────┐
  │   S3 + CloudFront           │
  │   (static + converted PDFs) │
  └─────────────────────────────┘
```

---

## Auto-Scaling Policies at Scale

### CPU-Based (Simple)

Keep CPU at 70%:
```
Users browse: CPU = 40%
  → don't scale
  
Heavy load: CPU = 75%
  → add 2 tasks
  
Super heavy: CPU = 95%
  → add 4 tasks (keep scaling aggressively)
```

### Queue-Depth Based (Workers)

SQS queue depth target = 2 messages per worker:

```
2 workers running
  Queue depth: 15 messages
  15 / 2 = 7.5 workers needed
    → Scale to 8 workers
    
Users done uploading
  Queue depth: 1 message
  1 / 2 = 0.5 workers needed
    → Scale to 1 worker (min)
```

### Scheduled Scaling

You know traffic patterns:

```
9 AM: Offices open, traffic spike
  → Pre-scale to 6 tasks at 8:55 AM
  
5 PM: Work ends, traffic drops
  → Scale down to 2 tasks at 5:15 PM

Saves cost, responds faster than reactive scaling.
```

---

## Migration Path: Step by Step

### 1. Setup AWS Infrastructure

```bash
# Create VPC, subnets, security groups
# Create RDS PostgreSQL (multi-AZ)
# Create ElastiCache Redis
# Create S3 bucket for files
# Create ECR repository for images
# Create ECS cluster
```

### 2. Replicate Database

```bash
# From Hetzner Postgres container:
docker exec postgresql pg_dump -U pdf_user pdf_db | gzip > dump.sql.gz

# Restore to RDS:
gunzip < dump.sql.gz | aws rds-proxy ...
```

### 3. Migrate Files

```bash
# From MinIO to S3:
aws s3 sync s3://minio-local/ s3://production-bucket/
```

### 4. Deploy First Task

```bash
# Build Docker image
docker build -t pdf-api:aws .

# Tag for ECR
docker tag pdf-api:aws ACCOUNT.dkr.ecr.REGION.amazonaws.com/pdf-api:aws

# Push to ECR
docker push ACCOUNT.dkr.ecr.REGION.amazonaws.com/pdf-api:aws

# Create ECS task definition
aws ecs register-task-definition --cli-input-json file://task-def.json

# Create ECS service
aws ecs create-service --cluster pdf-platform ...

# Test
curl https://pdf-platform-alb-123.eu-west-1.elb.amazonaws.com/health
```

### 5. DNS Cutover

```bash
# Update DNS to point to AWS ALB (not old Hetzner)
# In Route53 or DNS provider:

@ → pdf-platform-alb-123.eu-west-1.elb.amazonaws.com

# Wait for propagation (30 min)
# Traffic now flows to AWS
```

### 6. Monitor Old Hetzner

Keep Hetzner running in background for 1 week.

```bash
# If AWS fails, quickly revert DNS back to Hetzner
# After 1 week: Delete Hetzner (backup data first)
```

---

## Key Decisions at Each Phase

```
Phase 1 (Hetzner):
- Single machine
- All services on one Docker Compose
- Cost: $10/month
- Max scale: 100 users
- Good for: MVP, testing, learning

Phase 2 (AWS - Scale Out):
- ECS Fargate (auto-scaling)
- Managed RDS + ElastiCache
- ALB (load balance)
- SQS (instead of Redis queue)
- Cost: $400/month
- Scale: 500-1000 users
- Good for: Growing product, steady users

Phase 3+ (AWS - Global):
- Multiple regions (USA, EU, Asia)
- CloudFront CDN
- Read replicas
- DynamoDB (global sessions)
- Cost: $1000+/month
- Scale: 10,000+ users worldwide
- Good for: Global product, consistent high traffic
```

---

## Cost Optimization Strategies

**1. Reserved Instances** (23% discount)
```
Payment: Commit to 1-year RDS + EC2 instance
Savings: $60/month → $46/month
```

**2. Spot Instances** (70% discount, but interruptible)
```
For workers (can tolerate interruption):
On-demand Fargate: $0.05/hour
Spot Fargate: $0.015/hour (66% cheaper)
Downside: AWS stops task on 2min notice
```

**3. Auto-Scaling Down Aggressively**
```
Night hours (0-6 AM): Scale to 1 task
Saves: $140/month (5 fewer tasks × 8 hours × $0.05)
```

**4. Use Cheaper Regions**
```
AWS Pricing by region:
- us-east-1 (Virginia): Cheapest
- eu-west-1 (Ireland): 10% more
- ap-southeast-1: 20% more

Move compute to us-east-1 if possible.
```

---

## Key Takeaways

- **Phase 1**: Hetzner (€8.50/month, 100 users)
- **Phase 2**: AWS ECS (€400/month, 1000 users, auto-scaling)
- **Phase 3+**: Multi-region (€1000+/month, global)
- **Trigger Phase 2**: When Hetzner CPU > 70% constantly
- **Trigger Phase 3**: When traffic spans multiple continents
- **Cost/user**: Scales differently at each phase
  - Hetzner: $0.10/user (100 users)
  - AWS P2: $0.40/user (1000 users)
  - AWS P3: $0.10/user (10,000 users)
- **RTO/RPO**: Hetzner (4 hours recovery), AWS (< 1 min auto-failover)

Next module: Troubleshooting the 15 most common production problems.
