# Module 5: Auto-Scaling and Load Balancing

## The Complete Picture

From single VPS to scalable cloud:

```
Before (Hetzner VPS):
  ┌──────────────────┐
  │ Single machine   │
  │ Nginx + API      │
  │ PostgreSQL       │
  │ Redis container  │
  └──────────────────┘
  → 50 concurrent users max
  → Single point of failure
  → Manual scaling (resize machine, restart)

After (AWS):
  ┌────────────────────────────────────────┐
  │ Route53 (DNS)                          │
  ├────────────────────────────────────────┤
  │ ALB (Load Balancer)                    │
  │ Port 80 (HTTP), 443 (HTTPS)            │
  ├────────────────────────────────────────┤
  │ ECS Service (Auto-scaling 2-10 tasks)  │
  │ Task 1 (ECS Fargate, 256 CPU)          │
  │ Task 2 (ECS Fargate, 256 CPU)          │
  │ Task 3 (ECS Fargate, 256 CPU) ← Scales │
  │ ...                                    │
  ├────────────────────────────────────────┤
  │ RDS (Multi-AZ PostgreSQL)              │
  │ ElastiCache (Multi-AZ Redis)           │
  │ S3 (API documents)                     │
  │ SQS (Task queue)                       │
  └────────────────────────────────────────┘
  → 1000s concurrent users
  → Multi-AZ redundancy
  → Automatic scaling (no intervention)
  → Distributed, resilient
```

This module brings it all together.

## ALB: Application Load Balancer

ALB distributes traffic across multiple targets (ECS tasks).

```
User requests come in:
  https://pdf-platform.example.com/upload
        ↓
  Route53 (DNS)
        ↓
  ALB (public IP, port 443)
        ↓
  Health check: GET /health → 200 OK?
        ↓
  Route to healthy ECS task
        ↓
  Task returns response
        ↓
  User gets response
```

### ALB Components

**Listener**: Port ALB listens on
```
Listener rule:
  Port: 443 (HTTPS)
  Protocol: HTTPS
  Certificate: ACM (AWS Certificate Manager)
  Default action: Forward to target group
```

**Target Group**: Where to send traffic
```
Target group: ECS Service
  Health check path: /health
  Health check port: 5000
  Healthy threshold: 2 (2 consecutive checks pass)
  Unhealthy threshold: 3 (3 consecutive checks fail)
```

**Targets**: Individual ECS tasks registered here
```
Target 1: ECS task 1, port 5000, health: Healthy
Target 2: ECS task 2, port 5000, health: Healthy
Target 3: ECS task 3, port 5000, health: Healthy
Target 4: ECS task 4, port 5000, health: Unhealthy (remove from rotation)
```

## Health Checks

ALB periodically sends GET request to each task:

```
ALB → GET http://ECS_TASK_IP:5000/health
ECS Task → Response:
  {
    "status": "healthy",
    "database": "connected",
    "redis": "connected"
  }
  HTTP 200 OK

ALB: Task healthy ✓ (keep routing)
```

If task doesn't respond:
```
ALB → GET http://ECS_TASK_IP:5000/health
ECS Task → No response (crashed, timeout)

ALB: Wait (unhealthy_threshold = 3 checks)
After 3 failures: Task marked unhealthy
ALB: Stop routing traffic to task
ECS Service: "1 task unhealthy, 1 less than desired count"
ECS Auto-scaling: Start new task
```

### Health Check Endpoint

Your API must implement `/health`:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    """
    ALB health check endpoint.
    Must be lightweight, fast (<1 second).
    """
    return {
        "status": "healthy",
        "version": "1.0.0"
    }
```

**Critical**: Health check must include database, cache checks (if needed):

```python
@app.get("/health")
def health():
    try:
        # Check database
        db.execute("SELECT 1")
        
        # Check Redis
        redis_client.ping()
        
        return {"status": "healthy"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}, 503
```

## Auto-Scaling

Automatically adjust number of tasks based on metrics.

### Target Tracking Scaling (Simplest)

"Keep CPU at 70%"

```
Desired tasks: 2
  Task 1: 60% CPU
  Task 2: 65% CPU
  Average: 62.5% CPU

Target: 70%
  62.5% < 70%
  → Scale in (reduce tasks)
  
Desired tasks: 1
  Task 1: 95% CPU
  
Target: 70%
  95% > 70%
  → Scale out (add tasks)
```

Setup:

```bash
# Register auto-scaling
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/pdf-platform/api-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

# Create scaling policy (target tracking CPU 70%)
aws application-autoscaling put-scaling-policy \
  --policy-name api-cpu-scaling \
  --service-namespace ecs \
  --resource-id service/pdf-platform/api-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    }
  }'
```

### Step Scaling (More Control)

Define thresholds:

```
CPU < 30%:        Scale in (reduce 1 task)
CPU 30-50%:       No action
CPU 50-70%:       No action
CPU 70-90%:       Scale out (add 1 task)
CPU > 90%:        Scale out (add 2 tasks)
```

## Rolling Deployments

Deploy new version without downtime:

```
Current state: 4 tasks, old version

1. Start new task (v2)
   3 old (v1) + 1 new (v2)

2. Health check passes on v2
   ALB: Route traffic to v2
   
3. Stop 1 old task
   2 old (v1) + 1 new (v2)

4. Start new task (v2)
   2 old (v1) + 2 new (v2)

5. Stop another old task
   1 old (v1) + 2 new (v2)

6. Start new task (v2)
   0 old (v1) + 3 new (v2)

No downtime! Gradual switchover.
```

ECS configuration:

```json
{
  "cluster": "pdf-platform",
  "service": "api-service",
  "deploymentConfiguration": {
    "maximumPercent": 200,  # Allow 200% capacity during deployment
    "minimumHealthyPercent": 100  # Keep 100% of tasks healthy
  }
}
```

## Complete Architecture Example

### Setup

```bash
# 1. Create VPC and subnets (from AWS Basic)
# ... (reviewed in Module 5, AWS Basic)

# 2. Create ALB
ALB=$(aws elbv2 create-load-balancer \
  --name pdf-platform-alb \
  --subnets subnet-public-1 subnet-public-2 \
  --security-groups sg-alb \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

# 3. Create target group (for ECS tasks)
TG=$(aws elbv2 create-target-group \
  --name pdf-api-tasks \
  --protocol HTTP \
  --port 5000 \
  --vpc-id vpc-123 \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

# 4. Create ALB listener (HTTP → HTTPS redirect)
aws elbv2 create-listener \
  --load-balancer-arn $ALB \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

# 5. Create ALB listener (HTTPS)
# (Requires SSL certificate from AWS Certificate Manager)
aws elbv2 create-listener \
  --load-balancer-arn $ALB \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:... \
  --default-actions Type=forward,TargetGroupArn=$TG

# 6. Create ECS cluster
aws ecs create-cluster --cluster-name pdf-platform

# 7. Register task definition
# (See Module 1: ECS)

# 8. Create ECS service (with ALB)
aws ecs create-service \
  --cluster pdf-platform \
  --service-name api-service \
  --task-definition pdf-api:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={subnets=[subnet-private-1,subnet-private-2],securityGroups=[sg-api]}' \
  --load-balancers targetGroupArn=$TG,containerName=api,containerPort=5000

# 9. Register auto-scaling
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/pdf-platform/api-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

# 10. Create scaling policy
aws application-autoscaling put-scaling-policy \
  --policy-name api-scaling \
  --service-namespace ecs \
  --resource-id service/pdf-platform/api-service \
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

# 11. Create DNS record
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "pdf-platform.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z32O12XQLNTSW2",
          "DNSName": "pdf-platform-alb-123.eu-west-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

## Hands-On Lab

### Lab 5.1: Deploy with ALB + Auto-Scaling

```bash
# 1. Create ALB (if not exist)
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
ALB=$(aws elbv2 create-load-balancer \
  --name lab-alb \
  --subnets subnet-public-1 subnet-public-2 \
  --security-groups sg-alb \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

echo "ALB ARN: $ALB"

# 2. Create target group
TG=$(aws elbv2 create-target-group \
  --name lab-tasks \
  --protocol HTTP \
  --port 5000 \
  --vpc-id VPC_ID \
  --health-check-path /health \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

# 3. Create listener
aws elbv2 create-listener \
  --load-balancer-arn $ALB \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG

# 4. Create task definition (with /health endpoint)
TASK_DEF=$(cat << 'EOF'
{
  "family": "lab-api",
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "requiresCompatibilities": ["FARGATE"],
  "containerDefinitions": [{
    "name": "api",
    "image": "ACCOUNT.dkr.ecr.eu-west-1.amazonaws.com/lab-api:v1",
    "portMappings": [{
      "containerPort": 5000,
      "protocol": "tcp"
    }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/lab-api",
        "awslogs-region": "eu-west-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }],
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole"
}
EOF
)

aws ecs register-task-definition --cli-input-json "$TASK_DEF"

# 5. Create ECS service
aws ecs create-service \
  --cluster pdf-platform \
  --service-name lab-service \
  --task-definition lab-api:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-private-1,subnet-private-2],securityGroups=[sg-api]}" \
  --load-balancers targetGroupArn=$TG,containerName=api,containerPort=5000

# 6. Configure auto-scaling
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/pdf-platform/lab-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

aws application-autoscaling put-scaling-policy \
  --policy-name lab-scaling \
  --service-namespace ecs \
  --resource-id service/pdf-platform/lab-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    }
  }'

echo "Deployment complete!"
echo "ALB DNS: $(aws elbv2 describe-load-balancers --load-balancer-arns $ALB --query LoadBalancers[0].DNSName --output text)"
```

### Lab 5.2: Load Testing and Auto-Scaling

```bash
# 1. Install Apache Bench
apt-get install apache2-utils

# 2. Get ALB DNS
ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB --query 'LoadBalancers[0].DNSName' --output text)

# 3. Warm-up request
curl http://$ALB_DNS/health

# 4. Simulate load (1000 requests, 10 concurrent)
ab -n 1000 -c 10 http://$ALB_DNS/

# 5. Watch auto-scaling
watch -n 5 'aws ecs describe-services --cluster pdf-platform --services lab-service --query "Services[0].runningCount"'

# Output:
# Every 5.0s: running tasks
# Running Count: 2
# → (load increases)
# Running Count: 4
# → (load increases more)
# Running Count: 6
# → (load decreases)
# Running Count: 5
# → (load stabilizes)
# Running Count: 3

# 6. Check metrics in CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=lab-service Name=ClusterName,Value=pdf-platform \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T01:00:00Z \
  --period 60 \
  --statistics Average,Maximum
```

## Cheat Sheet: ALB + Auto-Scaling

```bash
# Create ALB
aws elbv2 create-load-balancer --name my-alb --subnets subnet-1 subnet-2

# Create target group (ECS tasks)
aws elbv2 create-target-group --name my-targets \
  --protocol HTTP --port 5000 --vpc-id VPC_ID \
  --health-check-path /health

# Create listener
aws elbv2 create-listener --load-balancer-arn ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=TG_ARN

# Create ECS service with ALB
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-service \
  --task-definition my-task:1 \
  --load-balancers targetGroupArn=TG_ARN,containerName=web,containerPort=5000

# Register auto-scaling target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/CLUSTER/SERVICE \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 10

# Create target tracking policy (CPU)
aws application-autoscaling put-scaling-policy \
  --policy-name scale-policy \
  --service-namespace ecs \
  --resource-id service/CLUSTER/SERVICE \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    }
  }'

# Update service (rolling deployment)
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --task-definition my-task:2
```

## Key Takeaways

- **ALB** = Application Load Balancer (distributes traffic across targets)
- **Target group** = collection of ECS tasks ALB routes to
- **Health checks** = ALB verifies task is healthy (/health endpoint)
- **Rolling deployment** = replace tasks gradually (zero downtime)
- **Auto-scaling** = adjust task count based on CPU, memory, custom metrics
- **Target tracking** = simplest scaling (keep metric at target value)
- **Minimum healthy** = keep % of old tasks running during deployment
- **Costs**: ALB $20/month + tasks scale from 2-10 = $50-150/month

## Complete AWS Platform Summary

You now understand:

1. **AWS Basics** (Module 1-4): Regions, EC2, S3, RDS, VPC
2. **ECS** (Module 1): Container orchestration, Fargate, ECR
3. **ElastiCache** (Module 2): Managed Redis, Multi-AZ
4. **SQS** (Module 3): Message queues, Celery integration
5. **IAM** (Module 4): Roles, policies, secrets (no hardcoded credentials)
6. **ALB + Auto-Scaling** (Module 5): Load balancing, rolling deployments

From single VPS with docker-compose → fully managed AWS architecture with auto-scaling and high availability.

**Next steps**: Deploy your PDF platform to AWS, experiment with auto-scaling under load, monitor CloudWatch metrics, optimize costs.

Prompt 4 complete!
