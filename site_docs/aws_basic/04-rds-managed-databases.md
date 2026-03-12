# Module 4: RDS Managed Databases

## What Is RDS?

RDS (Relational Database Service) is PostgreSQL running on AWS.

You don't manage the PostgreSQL container. AWS does.

### Your Current Setup

```
docker-compose.yml
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

You own:
- Container management
- Backups (if configured)
- Patching PostgreSQL version
- Replication setup
- Monitoring

### With RDS

AWS owns:
- Container management ✓
- Automated backups ✓
- Patching ✓
- Replication (Multi-AZ) ✓
- Monitoring ✓
- High availability ✓

Cost: ~$10/month (db.t3.micro free tier 12 months)

## RDS Instance Classes

Instance class = compute capacity of the database server.

- **db.t3.micro** (free tier, 12 months): 1 vCPU, 1 GB RAM — good for dev/test
- **db.t3.small**: 1 vCPU, 2 GB RAM
- **db.t3.medium**: 2 vCPU, 4 GB RAM
- **db.m5.large**: 2 vCPU, 8 GB RAM — for production

For your PDF platform:
- **Dev/test**: db.t3.micro (free tier)
- **Production**: db.t3.small or db.m5.large (depending on load)

## Multi-AZ Deployments

AWS creates a standby replica in a different AZ:

```
AZ-1: Primary PostgreSQL (your writes go here)
       ↓ (synchronous replication)
AZ-2: Standby replica (read-only, always synced)

If AZ-1 fails,
  → AWS automatically promotes Standby to Primary
  → Your app keeps running (transparent failover, 1-2 minutes)
```

Costs: Double the instance cost (you pay for 2 instances)

Enables:
- Zero downtime for AZ failures
- Automated backups (synced to S3)

## Read Replicas

Create read-only copies of the database:

```
Primary (writes): PostgreSQL in us-east-1
Read Replica 1: PostgreSQL in us-west-2 (async)
Read Replica 2: PostgreSQL in eu-west-1 (async)

SELECT queries → Read Replica (closer to user, faster)
INSERT/UPDATE/DELETE → Primary (strong consistency)
```

Use case: Multi-region app, serve users from closest region.

## Automated Backups

RDS automatically backs up your database daily:

### Backup Components

1. **Daily snapshot**: Point-in-time backup
2. **Transaction logs**: Incremental changes since snapshot

Together = restore to any second in the past 35 days.

```
Monday 10:00 AM:  Snapshot created
Monday 10:01 AM:  You INSERT row
Monday 10:02 AM:  You DELETE row (by mistake!)
Monday 10:03 AM:  Alert! Need to restore

Restore to Monday 10:01 AM
  → Row is restored!
```

Cost: Snapshot = ~$0.10/GB/month (4 snapshots typical = 5-7 GB cost)

## Parameter Groups

Parameter group = postgresql.conf equivalent.

Settings like:
- `shared_buffers`: Memory for caching (affects performance)
- `max_connections`: Max simultaneous connections
- `log_statement`: What statements to log
- `work_mem`: Memory per operation

### Creating Custom Parameter Group

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name pdf-platform-pg15 \
  --db-parameter-group-family postgres15 \
  --description "PostgreSQL 15 for PDF platform"

# Modify a setting
aws rds modify-db-parameter-group \
  --db-parameter-group-name pdf-platform-pg15 \
  --parameters ParameterName=max_connections,ParameterValue=100,ApplyMethod=immediate
```

## Connecting Your FastAPI App

RDS creates a connection string (endpoint):

```
pdf-platform.c9akciq32.eu-west-1.rds.amazonaws.com:5432
```

Update your `.env`:

```
# Before (MinIO)
DATABASE_URL=postgresql://user:password@localhost:5432/pdf_db

# After (RDS)
DATABASE_URL=postgresql://user:password@pdf-platform.c9akciq32.eu-west-1.rds.amazonaws.com:5432/pdf_db
```

Your FastAPI code doesn't change. Just the connection string.

## Creating RDS Instance

```bash
# Create RDS PostgreSQL instance (AWS console or CLI)
aws rds create-db-instance \
  --db-instance-identifier pdf-platform \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --master-user-password MyPassword123! \
  --allocated-storage 20  # GB \
  --storage-type gp3 \
  --db-name pdf_db \
  --publicly-accessible false \
  --multi-az true  # Automatic failover \
  --backup-retention-period 7  # Keep 7 days backups

# Wait ~5 minutes for creation
# Then get endpoint:
aws rds describe-db-instances --db-instance-identifier pdf-platform \
  --query 'DBInstances[0].Endpoint.Address'
```

## RDS vs Self-Managed PostgreSQL

| Aspect | RDS | Self-Managed |
|--------|-----|--------------|
| **Backups** | Automatic | You manage |
| **Patching** | Automatic | You schedule |
| **Hardware failures** | Auto-failover | Deploy replacement |
| **Scaling storage** | Easy (AWS adds disk) | Potentially risky |
| **Cost** | ~$10-200/month | Lower upfront, higher ops |
| **Control** | Limited (AWS constraints) | Full control |
| **Compliance** | Easier (AWS certifies) | Your responsibility |
| **SSL/TLS** | Built-in | You configure |

For most platforms: **Use RDS**.

## Hands-On Lab

### Lab 4.1: Launch RDS Instance

```bash
# 1. Create RDS PostgreSQL (free tier eligible)
aws rds create-db-instance \
  --db-instance-identifier pdf-platform-dev \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.2 \
  --master-username postgres \
  --master-user-password MyPassword123456! \
  --allocated-storage 20 \
  --storage-type gp3 \
  --db-name pdf_db \
  --publicly-accessible false \
  --region eu-west-1

# 2. Wait ~5 minutes for creation
# Check status:
aws rds describe-db-instances --db-instance-identifier pdf-platform-dev \
  --query 'DBInstances[0].DBInstanceStatus'

# 3. Get endpoint when status = "available"
ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier pdf-platform-dev \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)

echo $ENDPOINT  # pdf-platform-dev.c9akciq32.eu-west-1.rds.amazonaws.com

# 4. Connect with psql
psql -h $ENDPOINT -U postgres -d pdf_db
# Password: MyPassword123456!

# 5. Create table
CREATE TABLE documents (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  title TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

# 6. Insert test data
INSERT INTO documents (id, user_id, title) VALUES
  ('550e8400-e29b-41d4-a716-446655440000'::UUID, '123'::UUID, 'Test Doc');

# 7. Query
SELECT * FROM documents;

# 8. Exit
\q
```

### Lab 4.2: Run Alembic Migrations Against RDS

```bash
# In your project root (with alembic/ directory)

# Update alembic/env.py to use RDS endpoint:
export DATABASE_URL="postgresql://postgres:MyPassword123456!@$ENDPOINT:5432/pdf_db"

# Run migrations
alembic upgrade head

# Verify tables created
psql -h $ENDPOINT -U postgres -d pdf_db -c "\dt"
```

## Cheat Sheet: RDS CLI Commands

```bash
# Create RDS PostgreSQL
aws rds create-db-instance \
  --db-instance-identifier my-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --master-user-password PASSWORD123!

# Get endpoint
aws rds describe-db-instances --db-instance-identifier my-db \
  --query 'DBInstances[0].Endpoint.Address'

# Export for connection
ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier my-db \
  --query 'DBInstances[0].Endpoint.Address' --output text)

# Connect
psql -h $ENDPOINT -U admin -d dbname

# Backup now (manual snapshot)
aws rds create-db-snapshot --db-instance-identifier my-db \
  --db-snapshot-identifier my-db-backup-$(date +%s)

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier my-db-restored \
  --db-snapshot-identifier my-db-backup-123456

# Delete instance
aws rds delete-db-instance --db-instance-identifier my-db \
  --skip-final-snapshot
```

## Connection String Format

```
postgresql://USERNAME:PASSWORD@ENDPOINT:PORT/DATABASE_NAME

Example:
postgresql://postgres:MyPassword123!@pdf-platform.c9akciq32.eu-west-1.rds.amazonaws.com:5432/pdf_db
```

## Key Takeaways

- **RDS** = PostgreSQL managed by AWS (no container management)
- **Instance classes**: db.t3.micro (free tier), db.m5.large (production)
- **Multi-AZ**: Automatic failover to standby replica
- **Backups**: Automatic daily snapshots + transaction logs (point-in-time recovery)
- **Connection**: Change DATABASE_URL, no code changes
- **Parameter groups**: Tune PostgreSQL settings
- **Cost**: Free tier 12 months, then ~$10-50/month depending on instance size
- **Trade-off**: Less control, more reliability; let AWS manage infra

Module 5 teaches VPC and networking — your private AWS network.
