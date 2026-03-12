# AWS Advanced: Scaling, Containers, and Security

## Moving from Single-Server to Scalable Cloud

You've learned the basics:
- EC2 (virtual machines)
- S3 (object storage)
- RDS (managed databases)
- VPC (networking)

These are the building blocks. Now: **how to assemble them into a scalable, production-ready system**.

## The Transformation

### Before (Hetzner VPS, Prompt 1-2)

```
Docker Compose on single 4GB VPS:
  ├─ Nginx (reverse proxy) 1 instance
  ├─ FastAPI (API) 1-3 instances
  ├─ PostgreSQL container 1 replica
  ├─ Redis container 1 instance
  ├─ Celery workers 1-2 workers
  └─ Max traffic: 50-100 concurrent users
```

Bottleneck: One machine dies = everything down.

### After (AWS, This Module)

```
Fully scalable AWS infrastructure:
  ├─ ALB (Load Balancer) — AWS manages
  ├─ ECS (Container Orchestration)
  │   ├─ API: 2-10 containers (auto-scale)
  │   └─ Celery Workers: 2-20 containers (auto-scale)
  ├─ RDS (Managed Postgres) Multi-AZ
  ├─ ElastiCache (Managed Redis) Multi-AZ
  ├─ S3 (Object Storage) 99.999999999% uptime
  ├─ IAM (Identity & Access Management)
  ├─ Auto-scaling policies (CPU, memory)
  └─ CloudWatch (Monitoring)
  
  Max traffic: 1000s concurrent users
  Failure: Auto-failover (seconds)
```

## The 5 Advanced Modules

| # | Module | What You Learn | Time |
|---|--------|---|---|
| 1 | **ECS & Containers** | Run Docker on AWS, auto-scale containers | 2 hrs |
| 2 | **ElastiCache** | Managed Redis with automatic failover | 1 hr |
| 3 | **SQS** | AWS managed message queues (Celery alternative) | 1 hr |
| 4 | **IAM Deep Dive** | Least privilege, roles, no hardcoded credentials | 2 hrs |
| 5 | **Auto-Scaling + ALB** | Scale from 1 to 100+ containers, load balancer | 2 hrs |

## The Complete Production Architecture

```
         External Internet
              ↓
         Route53 (DNS)
         mypdf.com → *.*.*.* (ALB public IP)
              ↓
    ┌─────────────────────┐
    │ ALB (Public)        │
    │ - SSL termination   │
    │ - Health checks     │
    │ - Route to ECS      │
    └─────────────────────┘
              ↓
    ┌─────────────────────────────────────┐
    │ ECS Cluster (Private Subnet)         │
    │ ┌─────────────────────────────────┐ │
    │ │ API Service                     │ │
    │ │ ├─ Task 1 (AZ-a)               │ │
    │ │ ├─ Task 2 (AZ-b)               │ │
    │ │ └─ Auto-Scaling (2-10)         │ │
    │ ├─ Celery Workers Service        │ │
    │ │ ├─ Task 1 (AZ-a)               │ │
    │ │ ├─ Task 2 (AZ-b)               │ │
    │ │ └─ Auto-Scaling (2-20)         │ │
    │ └─ Task Definition linked to ECR │ │
    └─────────────────────────────────────┘
              ↓ (Internal Network)
    ┌─────────────────────────────────────┐
    │ Data Layer (Private Subnet)          │
    │ ├─ RDS (Multi-AZ, auto-failover)   │
    │ │  ├─ Primary (AZ-a)                │ │
    │ │  └─ Standby (AZ-b)                │ │
    │ └─ ElastiCache (Multi-AZ)           │
    │    ├─ Primary (AZ-a)                │
    │    └─ Replica (AZ-b)                │
    └─────────────────────────────────────┘
              ↓
          S3 Buckets (global)
```

## Prerequisites

You should understand:
- All AWS Basics (EC2, S3, RDS, VPC)
- Docker (from Module 1)
- SQL and databases (from Module 3)
- Redis (from the cache section)
- Celery task queues (from background jobs section)

## Key Concepts

### 1. Immutability

Docker images are immutable. You can't edit a container; you redeploy.

```
V1: docker build → tag pdf-api:v1 → push to ECR
    Deploy to ECS (all containers now V1)

V2: docker build → tag pdf-api:v2 → push to ECR
    Update ECS service (containers swap V1 → V2)
    Old image still in ECR (can rollback)
```

### 2. Blue-Green Deployments

Minimize downtime with smooth rollouts:

```
Before: 2 containers running V1
Deploy: 2 containers V2 start
        Health checks pass
Rolling: V1 container 1 stops
         V2 container 3 starts
         V1 container 2 stops
         V2 container 4 starts
After: 4 containers running V2 (briefly), then scale down to 2 V2
```

### 3. Auto-Scaling

Scale containers based on metrics:

```
API Service:
  Min tasks: 2 (always on)
  Max tasks: 10 (spike handling)
  Scale up: CPU > 70% for 2 minutes
  Scale down: CPU < 30% for 5 minutes

When 1000 users suddenly arrive:
  → CPU spikes to 80%
  → Scale up → 4 tasks
  → CPU normalizes to 60%
  → Scale up → 6 tasks
  → CPU stabilizes at 70%
  Users served! Cost scales too.
```

### 4. Health Checks

ALB verifies containers are healthy before routing traffic:

```
ALB:
  "GET /health HTTP/1.1"
Container:
  if database_ok and redis_ok:
    return 200 OK
  else:
    return 503 Unhealthy

ALB sees 503:
  → Mark container as unhealthy
  → Stop routing traffic to it
  → Auto-scaling may restart it
```

## Learning Outcomes

By the end of this section, you'll:
- ✅ Deploy Docker containers to ECS Fargate (no server management)
- ✅ Scale API containers 2-10 based on CPU load
- ✅ Use managed Redis (ElastiCache) for caching
- ✅ Understand message queues (SQS)
- ✅ Apply least-privilege IAM roles
- ✅ Set up health checks and auto-healing
- ✅ Handle rolling deployments
- ✅ Monitor with CloudWatch

## Cost Estimate (Advanced Setup)

| Component | Compute | Storage | Network | Total |
|-----------|---------|---------|---------|-------|
| **ECS Fargate** | 4 tasks × 0.25 vCPU + 0.5GB | - | - | $20 |
| **RDS Multi-AZ** | db.t3.small × 2 | 100GB | - | $30 |
| **ElastiCache** | cache.t3.small | - | - | $15 |
| **ALB** | - | - | - | $20 |
| **Data Transfer** | - | - | 1TB OUT | $100 |
| **Total** | | | | **$185** |

(Within first 12 months free tier: might be $50-80/month)

## Next Steps

1. **Module 1** — ECS: Learn to run containers on AWS
2. **Module 2** — ElastiCache: Managed Redis
3. **Module 3** — SQS: Message queues (optional Celery replacement)
4. **Module 4** — IAM: Secure, least-privilege access
5. **Module 5** — Auto-Scaling + ALB: Production-ready system

Each module has hands-on labs. Do them!

---

**Goal**: By end of Prompt 4, your PDF platform runs on AWS with:
- Automatic scaling (1-100 containers)
- Multi-AZ high availability (survives data center failures)
- Managed services (you manage code, AWS manages infrastructure)
- Cost-effective (pay for what you use)

Let's start with **Module 1: ECS and Containers**.
