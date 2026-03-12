# Module 2: ElastiCache - Managed Redis

## What Is ElastiCache?

ElastiCache is Redis running on AWS — AWS manages it.

Just like RDS for PostgreSQL, but for caching.

**Your current setup**:
```
docker-compose:
  redis:
    image: redis:7
    ports:
      - "6379:6379"
```

You manage: restarts, replication, failover, monitoring.

**With ElastiCache**:
- AWS manages restarts
- Multi-AZ replication (automatic)
- Automatic failover to replica
- CloudWatch monitoring
- Automated backups (optional)

Cost: ~$14/month (cache.t3.micro, 12 months free)

## ElastiCache vs Redis Container

| Aspect | Redis Container | ElastiCache |
|--------|---|---|
| **Failover** | Manual (container dies, down until restart) | Automatic (replica takes over) |
| **Replication** | You configure | AWS handles |
| **Scaling** | Stop container, bigger machine | Change instance type (downtime) |
| **Backups** | Manual snapshots | Automated optional |
| **Monitoring** | You set up | CloudWatch included |
| **Cost** | Free (container) | $14/month (cache.t3.micro) |

**When to migrate**: Production systems need high availability.

## Cluster vs Non-Cluster Mode

### Non-Cluster Mode (Default, Simplified)

```
┌──────────────────────┐
│ Primary Node         │
│ (reads/writes) 6GB   │
├──────────────────────┤
│ Replica Node         │
│ (reads only) 6GB     │
└──────────────────────┘
(async replication)

If Primary fails → Replica promoted to Primary
All data in one place (not truly distributed)
```

Use when: Data fits on one node (<100GB) — typical for caching.

### Cluster Mode (Advanced)

```
Shard 1:
  Primary: redis-1.example.com
  Replica: redis-1-replica.example.com

Shard 2:
  Primary: redis-2.example.com
  Replica: redis-2-replica.example.com

Keys distributed across shards (hash slot)
```

Use when: Data > 100GB, need partitioning.

**For your platform**: Non-cluster mode sufficient (cache, not primary data).

## ElastiCache in Private Subnet

AWS best practice: ElastiCache in private subnet (not reachable from internet).

```
Public Subnet:
  └─ API EC2

Private Subnet:
  ├─ RDS PostgreSQL
  ├─ ElastiCache Redis
  └─ NAT Gateway for outbound

API connects to ElastiCache via internal network:
  redis://elasticache-endpoint.c9akciq32.ng.0001aps.euw1.cache.amazonaws.com:6379
```

Connection doesn't leave VPC (secure, fast).

## Connection String

ElastiCache gives you an endpoint (hostname):

```
pdf-cache.c9akciq32.ng.0001aps.euw1.cache.amazonaws.com:6379
```

Use in your .env:

```bash
# Before (Redis container)
REDIS_URL=redis://localhost:6379

# After (ElastiCache)
REDIS_URL=redis://pdf-cache.c9akciq32.ng.0001aps.euw1.cache.amazonaws.com:6379
```

Your Python code doesn't change:

```python
import redis

redis_client = redis.from_url(os.getenv("REDIS_URL"))
redis_client.set("key", "value")
```

## Clustering and Eviction

### Memory Management

Specify cache size (e.g., 500MB). What happens when full?

**Eviction policies** (what to remove):
- **noeviction**: Return error when full (safest)
- **allkeys-lru**: Remove least recently used key (typical)
- **volatile-lru**: Remove least recently used expiring key
- **allkeys-random**: Remove random key

Default: **allkeys-lru** (makes sense for cache)

```bash
aws elasticache create-cache-cluster \
  --cache-cluster-id pdf-cache \
  --cache-node-type cache.t3.micro \
  --engine redis \
  --num-cache-nodes 1 \
  --parameter-group-name default.redis7 \
  --engine-version "7.0"

# Modify eviction policy
aws elasticache modify-parameter-group \
  --parameter-group-name pdf-custom-redis \
  --parameter-name-values Name=maxmemory-policy,Value=allkeys-lru
```

### Multi-AZ

Enable automatic failover:

```bash
aws elasticache create-replication-group \
  --replication-group-description "PDF platform cache" \
  --engine redis \
  --engine-version "7.0" \
  --cache-node-type cache.t3.micro \
  --num-cache-clusters 2 \
  --automatic-failover-enabled \
  --multi-az
```

Creates:
- Primary in AZ-1
- Replica in AZ-2
- If AZ-1 fails → Replica auto-promotes (transparent)

Cost: 2x instance cost (~$28/month for cache.t3.micro)

## CloudWatch Metrics

ElastiCache publishes metrics to CloudWatch:

```
CPU Utilization (%): 5%, 10%, 50%, ...
(Alert if > 80% for 5 min)

Memory Utilization (%): 20%, 50%, 90%, ...
(Alert if > 90%, means evicting keys)

Evictions (count/min): 0, 1, 100, ...
(High evictions = cache thrashing, upgrade size)

Network Bytes In/Out (bytes/sec): 1MB, 10MB, 100MB, ...
(Understand traffic)
```

Set alarms:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name redis-memory-high \
  --alarm-description "Alert if Redis memory > 90%" \
  --metric-name DatabaseMemoryUsagePercentage \
  --namespace AWS/ElastiCache \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold
```

## Migrating from Redis Container to ElastiCache

1. **Create ElastiCache cluster**
   ```bash
   # Create in AWS
   ```

2. **Update connection string in .env**
   ```bash
   REDIS_URL=redis://elasticache-endpoint:6379
   ```

3. **Test from your API**
   ```bash
   # API connects to ElastiCache
   # Old Redis container still running
   ```

4. **Migrate data (optional)**
   ```bash
   # If you need old cache values
   redis-cli --pipe < dump.rdb
   ```

5. **Delete old Redis container**
   ```bash
   docker-compose down redis
   ```

Done! API now uses managed ElastiCache.

## Hands-On Lab

### Lab 2.1: Create ElastiCache Cluster

```bash
# 1. Create security group (allow from API)
SG=$(aws ec2 create-security-group \
  --group-name redis-sg \
  --description "ElastiCache Redis" \
  --vpc-id VPC_ID \
  --query 'GroupId' --output text)

# Allow port 6379 from API security group
aws ec2 authorize-security-group-ingress \
  --group-id $SG \
  --protocol tcp --port 6379 \
  --source-group sg-api

# 2. Create subnet group
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name pdf-cache-subnet \
  --cache-subnet-group-description "PDF platform" \
  --subnet-ids subnet-123 subnet-456

# 3. Create ElastiCache cluster
aws elasticache create-replication-group \
  --replication-group-description "PDF platform cache" \
  --replication-group-id pdf-cache \
  --engine redis \
  --engine-version "7.0" \
  --cache-node-type cache.t3.micro \
  --num-cache-clusters 2 \
  --automatic-failover-enabled \
  --multi-az \
  --cache-subnet-group-name pdf-cache-subnet \
  --security-group-ids $SG

# 4. Wait ~5 minutes

# 5. Get endpoint
aws elasticache describe-replication-groups \
  --replication-group-id pdf-cache \
  --query 'ReplicationGroups[0].PrimaryEndpoint'

# Returns: pdf-cache.c9akciq32.ng.0001aps.euw1.cache.amazonaws.com:6379
```

### Lab 2.2: Test Connection

```bash
# Option 1: redis-cli (if installed locally)
redis-cli -h pdf-cache.c9akciq32.ng.0001aps.euw1.cache.amazonaws.com -p 6379

> PING
PONG  # Success!

> SET mykey "Hello"
OK

> GET mykey
"Hello"

# Option 2: Python
cat > test_cache.py << 'EOF'
import redis
import os

r = redis.Redis(
    host=os.getenv("REDIS_ENDPOINT"),
    port=6379,
    decode_responses=True
)

print(r.ping())  # Should return True
print(r.set("test", "Hello"))  # True
print(r.get("test"))  # "Hello"
EOF

export REDIS_ENDPOINT="pdf-cache.c9akciq32...."
python test_cache.py
```

### Lab 2.3: Monitor with CloudWatch

```bash
# View metrics in AWS Console:
# https://console.aws.amazon.com/elasticache

# Or use CLI:
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name DatabaseMemoryUsagePercentage \
  --dimensions Name=ReplicationGroupId,Value=pdf-cache \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T01:00:00Z \
  --period 300 \
  --statistics Average

# Check if any evictions happening
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name Evictions \
  --dimensions Name=ReplicationGroupId,Value=pdf-cache \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T01:00:00Z \
  --period 300 \
  --statistics Sum

# High evictions = upgrade cache size
```

## Cheat Sheet: ElastiCache Commands

```bash
# Create replication group (Multi-AZ)
aws elasticache create-replication-group \
  --replication-group-id my-cache \
  --engine redis --engine-version 7.0 \
  --cache-node-type cache.t3.micro \
  --num-cache-clusters 2 \
  --automatic-failover-enabled \
  --multi-az

# Get endpoint
aws elasticache describe-replication-groups \
  --replication-group-id my-cache \
  --query 'ReplicationGroups[0].PrimaryEndpoint'

# Modify node type (causes brief downtime)
aws elasticache modify-replication-group \
  --replication-group-id my-cache \
  --cache-node-type cache.t3.small \
  --apply-immediately

# Delete cluster
aws elasticache delete-replication-group \
  --replication-group-id my-cache \
  --retain-primary-cluster

# List clusters
aws elasticache describe-replication-groups

# Connection string
redis://ENDPOINT:6379
```

## Key Takeaways

- **ElastiCache** = managed Redis on AWS (automatic failover, scaling)
- **Multi-AZ** = Primary + Replica in different AZs (auto-failover)
- **Non-cluster mode** = sufficient for most caches (<100GB)
- **Cluster mode** = needed for large caches with sharding
- **Eviction policy** = allkeys-lru (remove least used)
- **CloudWatch** = monitor CPU, memory, evictions
- **Connection** = REDIS_URL points to ElastiCache endpoint
- **Cost**: Multi-AZ cache.t3.micro ~$28/month

Module 3 teaches SQS — AWS managed message queues.
