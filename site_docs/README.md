# Backend Engineering Study Guide

A comprehensive, self-paced curriculum for learning backend engineering by building a real product: a **PDF conversion platform**.

## What's Included

- **78 modules** across 12 sections (200,000+ words)
- **40+ hands-on labs** with runnable code
- **Real examples** from a production PDF conversion platform
- **Capstone project** covering architecture, deployment, scaling, and debugging
- **Cheat sheets** for quick reference

## Sections

1. **Docker & Containers** (7 modules) — Containerization, images, compose
2. **Linux Fundamentals** (5 modules) — Server OS, permissions, networking
3. **FastAPI** (7 modules) — Modern async Python web framework
4. **PostgreSQL** (7 modules) — Relational database design and queries
5. **Redis** (5 modules) — Caching, pub/sub, rate limiting
6. **Celery** (5 modules) — Background job processing
7. **Networking** (6 modules) — TCP/IP, DNS, TLS, load balancing
8. **Security** (5 modules) — Authentication, encryption, API security
9. **Distributed Systems** (7 modules) — Consistency, replication, scaling
10. **AWS Fundamentals** (5 modules) — EC2, S3, RDS, VPC
11. **AWS Advanced** (5 modules) — ECS, auto-scaling, ElastiCache
12. **Capstone** (8 modules) — Full system architecture and deployment

## Quick Start

### Read Online

Visit the documentation at: https://backend-engineering.guide

### Run Locally

```bash
# Install dependencies
pip install mkdocs-material

# Start development server
mkdocs serve

# Open http://localhost:8000 in your browser
```

### Deploy to GitHub Pages

```bash
# Build static site
mkdocs build

# Deploy to GitHub Pages
mkdocs gh-deploy
```

## Directory Structure

```
docs/
├── index.md                 # Homepage
├── mkdocs.yml               # Configuration
├── overrides/               # Custom styling
│   └── stylesheets/
│       └── extra.css
├── docker/                  # Section 1: Docker & Containers
│   ├── index.md
│   ├── 01-what-are-containers.md
│   ├── 02-docker-basics.md
│   └── ... (7 modules total)
├── linux/                   # Section 2: Linux Fundamentals
│   └── ... (5 modules)
├── fastapi/                 # Section 3: FastAPI
│   └── ... (7 modules)
├── postgresql/              # Section 4: PostgreSQL
│   └── ... (7 modules)
├── redis/                   # Section 5: Redis
│   └── ... (5 modules)
├── celery/                  # Section 6: Celery
│   └── ... (5 modules)
├── networking/              # Section 7: Networking
│   └── ... (6 modules)
├── security/                # Section 8: Security
│   └── ... (5 modules)
├── distributed_systems/     # Section 9: Distributed Systems
│   └── ... (7 modules)
├── aws_basic/               # Section 10: AWS Fundamentals
│   └── ... (5 modules)
├── aws_advanced/            # Section 11: AWS Advanced
│   └── ... (5 modules)
└── capstone/                # Section 12: Capstone
    ├── index.md
    ├── 01-architecture-overview.md
    ├── 02-request-lifecycle.md
    ├── 03-upload-and-conversion-pipeline.md
    ├── 04-payment-processing.md
    ├── 05-session-timer-system.md
    ├── 06-production-deployment.md
    ├── 07-scaling.md
    └── 08-what-breaks-and-how-to-fix-it.md
```

## How to Use This Guide

### For Complete Beginners

Follow the modules in order:
1. Start with **Docker & Containers** to understand containerization
2. Learn **Linux Fundamentals** for server concepts
3. Progress through **FastAPI**, **PostgreSQL**, **Redis**, **Celery**
4. Continue with **Networking**, **Security**, **Distributed Systems**
5. Learn **AWS** for scalable cloud infrastructure
6. Finish with the **Capstone** to see everything together

**Estimated time**: 60-80 hours

### For Experienced Engineers

Jump directly to topics you need:
- Need to learn FastAPI? Start with [FastAPI Overview](docs/fastapi/index.md)
- Migrating to AWS? Start with [AWS Fundamentals](docs/aws_basic/index.md)
- Debugging production issues? See [Capstone: Troubleshooting](docs/capstone/08-what-breaks-and-how-to-fix-it.md)

## The Project: PDF Conversion Platform

Throughout this curriculum, we build a real product:

**Features:**
- User registration and authentication
- PDF upload
- PDF → PNG conversion (5 pages per file)
- Subscription management (free 3/month, paid unlimited/hour)
- Payment processing via Paymob
- Real-time session timer
- File storage in S3/MinIO
- Background job processing with Celery

**Stack:**
- **Backend**: FastAPI (Python)
- **Database**: PostgreSQL
- **Cache**: Redis
- **Jobs**: Celery
- **Storage**: MinIO (S3-compatible)
- **Reverse Proxy**: Nginx
- **OS**: Linux (Ubuntu)
- **Container**: Docker
- **Cloud**: AWS (ECS, RDS, ElastiCache, ALB, Route53, S3)

**Deployment:**
- Phase 1: Hetzner CX21 VPS (€8.50/month)
- Phase 2: AWS ECS + RDS + ElastiCache
- Phase 3: Auto-scaling to 1000+ concurrent users

## Key Features of This Curriculum

✓ **Concept → Implementation** — Understand the "why" before the "how"

✓ **Real Code** — Copy-paste examples from actual code

✓ **Hands-On Labs** — Every module includes executable commands

✓ **Cheat Sheets** — Quick reference for common tasks

✓ **Troubleshooting Guide** — 15 common production problems and solutions

✓ **Architecture Diagrams** — Visual representations of system design

✓ **Best Practices** — Real-world deployment strategies

✓ **Scaling Strategies** — How to grow from 1 to 1000+ users

## Prerequisites

You should be comfortable with:
- Basic programming (Python, JavaScript, or similar)
- How HTTP requests work
- Command line basics (cd, ls, grep)

You do **NOT** need to know:
- Docker, containers, or Kubernetes
- Databases or SQL
- Web frameworks
- Cloud platforms or AWS
- Distributed systems

## Learning Outcomes

After completing this curriculum, you will be able to:

✓ Design a complete backend system architecture

✓ Build a web application with user registration and payments

✓ Set up production infrastructure with containers and VPS

✓ Use PostgreSQL for reliable data storage

✓ Implement caching with Redis

✓ Process long-running jobs asynchronously with Celery

✓ Secure your application with authentication and encryption

✓ Deploy to AWS with auto-scaling

✓ Debug production issues systematically

✓ Monitor system performance and health

## Contributing

Found an error? Want to add content? Contributions welcome!

1. Fork the repository
2. Make changes
3. Submit a pull request

## License

This curriculum is provided as-is for educational purposes.

The code examples are MIT licensed.

## Installation & Setup

### Requirements

- Python 3.7+
- pip
- Git (optional)

### Setup

```bash
# Clone the repository (or download)
git clone https://github.com/yourusername/backend-engineering-guide
cd backend-engineering-guide

# Install mkdocs and dependencies
pip install mkdocs-material mkdocs-offline

# Run locally
mkdocs serve
```

Open http://localhost:8000 in your browser.

## Updates & Maintenance

This curriculum is updated regularly with:
- Latest technology versions
- New case studies
- Reader feedback and corrections
- Additional troubleshooting scenarios

Check back regularly for updates!

## Next Steps

### Start Reading

1. [Home Page](docs/index.md) — Overview of all sections
2. [Docker & Containers](docs/docker/index.md) — Begin the curriculum

### Jump to Specific Topics

- [FastAPI Web Framework](docs/fastapi/index.md)
- [PostgreSQL Database](docs/postgresql/index.md)
- [AWS Cloud Platform](docs/aws_basic/index.md)
- [Production Troubleshooting](docs/capstone/08-what-breaks-and-how-to-fix-it.md)

## Questions?

Have questions or feedback? Open an issue or discussion on GitHub.

---

**Total Content**: 78 modules | 200,000+ words | 40+ hands-on labs

**Time to Complete**: 60-80 hours

**Outcome**: Senior backend engineer (architecture, ops, debugging)

Happy learning! 🚀
