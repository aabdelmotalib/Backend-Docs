# Backend Engineering — PDF Platform Study Guide

Welcome to the most comprehensive backend engineering curriculum available.

This site documents how to build, deploy, and scale a production web application from first principles.

---

## What Is This?

This is a **complete study guide** for backend software engineers.

You'll learn by building a real product: a **PDF conversion platform** where users upload PDFs, convert them to images, and download the results.

- Free plan: 3 files/month
- Paid plan: unlimited conversions in a 1-hour window
- Payment processing via Paymob
- Real-time session timers
- Asynchronous task processing
- Production deployment on Hetzner VPS or AWS

---

## What You'll Learn

### Core Technologies

- **Docker** — Package apps with all dependencies
- **Linux** — Server fundamentals (networking, permissions, pkg management)
- **FastAPI** — Modern async Python web framework
- **PostgreSQL** — Relational database (the source of truth)
- **Redis** — In-memory cache and message broker
- **Celery** — Background task processor for long-running jobs
- **Nginx** — Reverse proxy and load balancer
- **MinIO/S3** — Object storage for files

### Advanced Topics

- **Networking** — TCP/IP, DNS, TLS/HTTPS, rate limiting
- **Security** — Authentication, JWT, password hashing, API keys
- **Distributed Systems** — Consistency, availability, fault tolerance, scaling
- **AWS** — Cloud infrastructure (EC2, RDS, ElastiCache, ECS, ALB, auto-scaling)
- **Production Deployment** — On Hetzner VPS, then migrate to AWS

### The Capstone

Everything comes together:
- Architecture overview (all 8 services, their roles)
- Request lifecycle (one HTTP request, traced end-to-end)
- Full upload/conversion pipeline (file enters, PNG exits)
- Payment processing (user pays, subscription activates)
- Session timer system (Redis + PostgreSQL sync)
- Production deployment (step-by-step walkthrough)
- Scaling strategies (Hetzner → AWS Phase 2 → Phase 3)
- Troubleshooting (the 15 most common production problems)

---

## How to Use This Site

### For Beginners: Read in Order

Start with **Docker & Containers** (Module 1), then **Linux Fundamentals**, then the frameworks (FastAPI, PostgreSQL, Redis, Celery). This builds a solid foundation.

After Modules 1-3: You'll understand containerized applications.

After Modules 1-5: You can build a basic web app locally.

After Modules 1-10: You understand production systems.

After the **Capstone**: You can architect, deploy, and troubleshoot complex platforms.

**Estimated time**: 60-80 hours of reading + hands-on labs.

### For Experienced Engineers: Jump to What You Need

If you know Docker but want to learn FastAPI: Jump to **FastAPI** section.

If you know web frameworks but want to understand PostgreSQL: Jump to **PostgreSQL** section.

If you're migrating to AWS: Jump to **AWS** section.

If your system is broken: Jump to **Capstone: Module 8 (Troubleshooting)**.

### How to Get the Most Out of This

Each module follows this structure:

1. **Concept & Analogy** — Understand the "why"
2. **Technical Depth** — How it works under the hood
3. **Real Code** — Copy-paste examples from the PDF platform
4. **Hands-On Lab** — Execute commands, see results
5. **Cheat Sheet** — Quick reference for common tasks

**Best practice**: Read + Run the lab. Don't just read.

---

## The Stack at a Glance

```
Frontend:
  React SPA → localStorage for JWT

Internet:
  HTTPS (TLS 1.3)

Server (Hetzner CX21):
  Nginx (reverse proxy, port 80/443)
    ↓
  FastAPI (web framework, port 8000)
    ↓
  PostgreSQL (database, port 5432, source of truth)
  Redis (cache + queue, port 6379)
  Celery (background tasks)
  MinIO (file storage, S3-compatible)
  LibreOffice (PDF→PNG conversion)
  ClamAV (antivirus scan)

Cost: €8.50/month

Scales to: 100 concurrent users, 50 conversions/hour
```

For higher scale, migrate to AWS:
- ECS Fargate (auto-scaling containers)
- RDS (managed PostgreSQL)
- ElastiCache (managed Redis)
- ALB (load balancer)
- Route53 (DNS)
- S3 (file storage)

Cost: $400-1000/month
Scale: 1000+ concurrent users, auto-scaling to demand

---

## What You'll Be Able to Build

After this curriculum:

✓ A complete web application (registration, payments, file uploads)

✓ Authentication system (JWT tokens, logout, password reset)

✓ Background job processor (convert files asynchronously)

✓ Payment integration (Paymob, webhooks, idempotency)

✓ Real-time monitoring (session timers, job status polling)

✓ Production deployment (SSL, DNS, monitoring, backups)

✓ Infrastructure scaling (from single server to auto-scaling fleet)

✓ Debugging production issues (15 common problems covered)

✓ Security best practices (HTTPS, rate limiting, SQL injection prevention)

✓ Database design (schema, migrations, transactions, replication)

---

## All 11 Sections


| # | Section | Description | Modules |
|---|---------|-------------|---------|
| **1** | **Docker & Containers** | Package everything in containers | [7 modules](docker/index.md) |
| **2** | **Linux Fundamentals** | Server, OS, permissions, networking | [5 modules](linux/index.md) |
| **3** | **FastAPI** | Modern async Python web framework | [7 modules](fastapi/index.md) |
| **4** | **PostgreSQL** | Relational database, queries, migrations | [7 modules](postgresql/index.md) |
| **5** | **Redis** | In-memory cache, queues, pub/sub | [5 modules](redis/index.md) |
| **6** | **Celery** | Background tasks, async jobs, retries | [5 modules](celery/index.md) |
| **7** | **Networking** | DNS, TLS, HTTP, load balancing, rate limiting | [6 modules](networking/index.md) |
| **8** | **Security** | Authentication, encryption, HTTPS, API keys | [5 modules](security/index.md) |
| **9** | **Distributed Systems** | Consistency, CAP theorem, replication, scaling | [7 modules](distributed_systems/index.md) |
| **10** | **AWS — Fundamentals** | Cloud basics, EC2, S3, RDS, VPC | [5 modules](aws_basic/index.md) |
| **11** | **AWS — Advanced** | ECS, ElastiCache, SQS, IAM, Auto-Scaling | [5 modules](aws_advanced/index.md) |
| **12** | **Capstone** | Everything together (architecture, deployment, scaling, debugging) | [8 modules](capstone/index.md) |

**Total: 78 modules, ~200,000 words, 40+ hands-on labs.**

---

## Prerequisites

You should know:
- Basic programming (Python, JavaScript, SQL)
- How HTTP requests work (browsers, APIs)
- Command line basics (cd, ls, grep)

You do **NOT** need to know:
- Docker, Kubernetes, or containers
- PostgreSQL, Redis, or any databases
- FastAPI, Django, or web frameworks
- AWS, cloud platforms, or distributed systems

---

## Key Philosophy

**You learn by building.**

Every concept is illustrated with code from the actual PDF platform.

Every lab lets you execute commands and see results.

No placeholder code. No "imagine this works." Everything is concrete.

---

## Getting Started

### Option 1: Read Online

You're already here! Each section has links to every module.

Start with **Docker & Containers**, Module 1.

### Option 2: Run Locally

```bash
# Install mkdocs-material
pip install mkdocs-material

# Run locally
mkdocs serve

# Open browser
http://localhost:8000
```

### Option 3: Deploy to GitHub Pages

```bash
# Build static site
mkdocs build

# Deploy
mkdocs gh-deploy
```

---

## About This Curriculum

This was created for engineers who want to **understand systems deeply**.

Not just "use Docker" but "understand containerization."

Not just "use FastAPI" but "understand async web frameworks."

Not just "use AWS" but "understand cloud scaling."

By the end, you won't just know technologies. You'll understand **why they exist** and **when to use them**.

---

## Support

Each module has a **Cheat Sheet** section for quick reference.

The **Capstone: Module 8 (Troubleshooting)** has diagnostic steps for 15 common problems.

---

## Let's Begin

Ready?

### [Start with Docker & Containers →](docker/index.md)

If you're jumping to a specific topic:
- [FastAPI (web framework)](fastapi/index.md)
- [PostgreSQL (database)](postgresql/index.md)
- [AWS (cloud)](aws_basic/index.md)
- [Production Troubleshooting](capstone/08-what-breaks-and-how-to-fix-it.md)

---

**Total Content**: 78 modules | 200,000+ words | 40+ hands-on labs

**Time to Complete**: 60-80 hours

**Skill Level After**: Senior backend engineer (architecture, ops, debugging)

Good luck. You're about to become a backend expert.

---

*Last updated: March 12, 2026*

*All code examples tested and verified for current AWS, Docker, and Python versions.*
