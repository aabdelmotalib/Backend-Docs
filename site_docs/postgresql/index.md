# PostgreSQL Mastery: Data Persistence for the PDF SaaS

Welcome to PostgreSQL. PostgreSQL is a powerful open-source relational database. It stores, organizes, and retrieves data reliably.

## Why PostgreSQL for This Project

The PDF processing platform needs to:

1. **Store structured data** — Users, subscriptions, jobs, payments
2. **ACID guarantees** — If a payment is recorded, it stays recorded
3. **Relationships** — A user has many subscriptions; a subscription has many jobs
4. **Complex queries** — "Show me all users who have completed 10+ jobs"
5. **Scale confidently** — Handle millions of records without losing data

PostgreSQL is the best-in-class relational database for these needs.

## The Data Model

The PDF SaaS stores 6 core entities:

```
users (id, email, created_at)
├── subscriptions (id, user_id, plan, active)
│   ├── jobs (id, subscription_id, status, pages)
│   └── payments (id, subscription_id, amount, date)
└── sessions (id, user_id, token, expires_at)
```

Relationships:
- One user has many subscriptions
- One subscription has many jobs
- One subscription has many payments
- One user has many sessions

## What You'll Learn

7 modules covering PostgreSQL from basics to production:

1. **Relational Fundamentals** — Tables, keys, constraints, SQL
2. **Setup and Basics** — Installation, connection, psql CLI
3. **SQLAlchemy ORM** — Python objects instead of SQL
4. **Alembic Migrations** — Version control for schema
5. **Indexes and Performance** — Make queries fast
6. **Transactions and Concurrency** — Handle simultaneous changes
7. **Production Setup** — Backups, monitoring, replication

## Module Dependencies

```
01 → 02 → 03 → 04 ↘
              05 → 06 → 07
```

- **Module 1** teaches relational concepts
- **Module 2** gets PostgreSQL running
- **Module 3** uses ORM instead of raw SQL
- **Module 4** manages schema changes
- **Module 5** optimizes with indexes
- **Module 6** handles concurrent access safely
- **Module 7** prepares for production

## Architecture Overview

```
┌─────────────────────────────────┐
│   FastAPI App (Module 1-7)      │
├─────────────────────────────────┤
│   SQLAlchemy ORM (Module 3)     │
├─────────────────────────────────┤
│  PostgreSQL Database (Module 2) │
│  - 6 tables                     │
│  - Indexes (Module 5)           │
│  - Transactions (Module 6)      │
│  - Point-in-time recovery       │
│    (Module 7)                   │
└─────────────────────────────────┘
```

## Prerequisites

- **Python 3.12+** with pip
- **Docker** — to run PostgreSQL in a container
- **Basic SQL** — SELECT, INSERT, UPDATE (taught in Module 1)
- **Understanding async** — from FastAPI Module 2

## Structure of Each Module

Every module follows this pattern:

1. **Concept** — Simple analogy first
2. **Technical Explanation** — Deep dive with real details
3. **Real SQL Examples** — From the actual project
4. **Hands-On Lab** — You build working tables
5. **Cheat Sheet** — Quick reference for commands and patterns

Let's start with the fundamentals of relational databases.
