# AWS Basics: From Single Server to Cloud Scale

## The Journey So Far

Your PDF platform started on a single Hetzner VPS:
- One machine running Docker
- One PostgreSQL container
- One Redis container
- One Celery worker (or however many fit)
- Nginx reverse proxy

Limited by:
- One machine can only hold so many containers
- Single point of failure (machine dies = everything dies)
- Scaling means renting a bigger machine (expensive, bounded)

## The Shift: AWS and Managed Services

AWS lets you rent services, not machines.

Instead of:
```
One Hetzner VPS running:
  - Docker (API containers)
  - PostgreSQL container
  - Redis container
  - Celery workers in containers
```

You use:
```
AWS services:
  - ECS (run API containers, auto-scale)
  - RDS (managed PostgreSQL, automated backups, multi-AZ failover)
  - ElastiCache (managed Redis, multi-AZ)
  - SQS or ECS for Celery workers
  - S3 (replace MinIO)
  - ALB (replace Nginx)
```

## The Mental Model: Service Mapping

| Your Platform | Hetzner VPS | AWS Service | Benefit |
|---------------|-------------|-------------|---------|
| API (FastAPI) | Docker container | ECS (Fargate) | Auto-scale, no server mgmt |
| PostgreSQL | Docker container | RDS | Automated backups, multi-AZ failover |
| Redis | Docker container | ElastiCache | Auto-failover, managed |
| Celery + Redis | Docker container | ECS or SQS | Durable, scalable |
| MinIO (storage) | Docker container | S3 | Globally distributed, 99.999% reliable |
| Nginx (load balancer) | Nginx config | ALB | AWS manages it, auto-scaling |
| Server SSH access | Direct | EC2 (rare needed) | Only for troubleshooting |

## What This Section Covers

### AWS Basics (5 modules)

1. **AWS Core Concepts** — Regions, IAM, how to not get bill-shocked
2. **EC2 Virtual Servers** — When you need a raw machine (usually just for initial setup)
3. **S3 Object Storage** — Replace MinIO, host static files
4. **RDS Databases** — Managed PostgreSQL with backups and failover
5. **VPC and Networking** — Your private "fence" around everything, security architecture

### AWS Advanced (5 modules)

1. **ECS Containers** — Run Docker on AWS without managing EC2
2. **ElastiCache** — Managed Redis with automatic failover
3. **SQS Queues** — AWS's managed message queue (optional Celery alternative)
4. **IAM Deep Dive** — Least privilege access, no hardcoded credentials
5. **Auto-Scaling + Load Balancing** — Scale from 1 to 100 containers automatically

## Key Principles

### 1. Use Managed Services When Possible

**Don't do**:
```
EC2 instance
  ├─ Install PostgreSQL yourself
  ├─ Setup backups
  ├─ Configure replication
  └─ Monitor it 24/7
```

**Do**:
```
Use RDS
  ├─ Automated backups ✓
  ├─ Multi-AZ automatic failover ✓
  ├─ Monitoring ✓
  └─ Patching ✓
```

### 2. Think in Availability Zones

AWS regions are split into Availability Zones (AZs). Each AZ is isolated:
- If AZ-1 has a power outage, AZ-2 and AZ-3 are fine
- Place replicas in different AZs for redundancy

Your platform:
- Primary database in AZ-1
- Standby replica in AZ-2 (automatic failover if AZ-1 fails)

### 3. Security Through Layers

- **VPC**: Create a private network
- **Security Groups**: Firewall at instance level (allow API port 5000, block RDS port 5432)
- **IAM roles**: Services authenticate without hardcoded passwords
- **Secrets Manager**: Store PAYMOB_KEY, JWT_SECRET securely

### 4. Pay Attention to Costs

AWS's "pay per use" is powerful but dangerous.

Free tier mistakes:
```
❌ Leave an EC2 instance on 24/7 by accident
   Cost: $30-40/month (should be $0 free tier)

❌ Transfer 100GB out to the internet
   Cost: $10-15 charge (should be free within limit)

✅ Use CloudWatch to track costs
✅ Set billing alarms (alert if bill > $50/month)
✅ Use AWS Cost Explorer to see what's expensive
```

## Prerequisites

You should understand:
- Docker basics (from Module 1)
- Linux and VPS concepts (from Module 2)
- FastAPI (from the main project section)
- PostgreSQL and Redis basics (from relevant sections)

## Learning Path

Start with **AWS Basics**:
1. Set up an AWS account, IAM user, CLI
2. Launch an EC2 instance, SSH in, run Docker
3. Understanding S3, RDS, VPC
4. Gain comfort with AWS Console

Then move to **AWS Advanced**:
1. Run your actual API on ECS Fargate (auto-scaling)
2. Use RDS and ElastiCache for data
3. Configure IAM roles for each service
4. Auto-scaling policies + load balancing
5. Profit

## Architecture: Before and After

### Before (Hetzner)

```
┌─────────────────────────────┐
│   One Hetzner VPS           │
│                             │
│  ┌───────────────────────┐  │
│  │  Nginx (port 80/443)  │  │
│  └───────────────────────┘  │
│           │                 │
│  ┌────────┼────────┐        │
│  │        │        │        │
│  ▼        ▼        ▼        │
│ API-1   API-2   API-3       │ (up to ~4)
│  │        │        │        │
│  └────────┼────────┘        │
│           │                 │
│    ┌──────┼──────┐          │
│    │      │      │          │
│    ▼      ▼      ▼          │
│  PostgreSQL Redis MinIO      │
│                             │
│ Scaled by: Bigger machine   │
│ Failure: Entire VPS down    │
└─────────────────────────────┘
```

### After (AWS)

```
         Route53 (DNS)
              │
              ▼
    ┌─────────────────┐
    │  ALB (Public)   │
    └─────────────────┘
              │
    ┌─────────┴────────────┐
    │                      │
    ▼        AZ-1          ▼        AZ-2
  ECS Task              ECS Task
  (API, CPU 2, 4GB)    (API, CPU 2, 4GB)
    │                      │
    └──────────┬───────────┘
               │
    ┌──────────┴──────────┐
    ▼                     ▼
 PrimaryDB          StandbyDB  (auto-failover)
 (Multi-AZ RDS, PostgreSQL, automated backups)
    │                     │
    │ ← Replication →     │
    ▼
 ElastiCache Redis (Multi-AZ, auto-failover)
    │
 S3 (99.999% uptime, global)

Scaled by: Change ASG max from 2 to 100 tasks ✓
Failure: Auto-failover in seconds ✓
Cost: Measured, predictable ✓
```

## Estimated AWS Costs (Monthly)

For your PDF platform at 100k API requests/day:

| Service | What | Cost |
|---------|------|------|
| **ECS Fargate** | 2-5 tasks, 0.5 vCPU + 1GB RAM | $10-20 |
| **RDS PostgreSQL** | db.t3.micro (free tier 12mo) | $0 (first year) |
| **ElastiCache Redis** | cache.t3.micro (free tier 12mo) | $0 (first year) |
| **ALB** | Load balancer | $15-20 |
| **S3** | ~100GB storage + transfers | $5-15 |
| **NAT Gateway** | (needed for private subnet) | $30-50 |
| **CloudWatch** | Logs, metrics | $5-10 |
| **Total (first year free tier)** | | **$70-100** |
| **Total (after free tier)** | | **$150-250** |

## Next Steps

Start with **AWS Core Concepts** to set up your AWS account and understand cost controls. Then move module by module, practicing with hands-on labs.

By the end, you'll have:
- Production-ready app on AWS
- Auto-scaling to handle spikes
- Multi-AZ failover for reliability
- Cost-effective architecture
- Deep understanding of AWS services
