# Module 1: ECS and Containers on AWS

## What Is ECS?

ECS (Elastic Container Service) = Docker orchestration on AWS.

Run your Docker containers without managing EC2 instances.

### Before: docker-compose

```yaml
version: '3'
services:
  api:
    image: pdf-api:latest
    ports:
      - "5000:5000"
    environment:
      DATABASE_URL: postgresql://...
```

You manage: restart crashes, health checks, updates, scaling.

### After: ECS

AWS manages: restarts, health checks, rolling updates, auto-scaling.

You just say: "Run 2 API containers, scale to 10 under load."

## ECS Concepts

### Cluster

Container cluster = set of compute resources (Fargate or EC2).

```bash
# Create cluster (management only, no actual compute)
aws ecs create-cluster --cluster-name pdf-platform
```

### Task Definition

Like `docker-compose` service definition: image, ports, env vars, resources.

```json
{
  "family": "pdf-api",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.eu-west-1.amazonaws.com/pdf-api:v1",
      "memory": 512,
      "cpu": 256,
      "portMappings": [
        {
          "containerPort": 5000,
          "hostPort": 5000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "DATABASE_URL",
          "value": "postgresql://..."
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/pdf-api",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole"
}
```

### Service

"Keep X tasks of task definition Y running."

```bash
aws ecs create-service \
  --cluster pdf-platform \
  --service-name api-service \
  --task-definition pdf-api:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-123],securityGroups=[sg-123],assignPublicIp=DISABLED}"
```

### Task

A single running container (or group of containers).

```
Service: "Run 2 tasks at all times"
  Task 1: Running (cpu=25%, healthy)
  Task 2: Running (cpu=18%, healthy)
```

## Fargate vs EC2 Launch Type

### Fargate (Recommended)

AWS manages EC2 instances for you.

```
You specify: CPU (0.25, 0.5, 1, 2, 4 vCPU), RAM (512MB - 30GB)
AWS provides: Compute capacity, no instance management
Price: $0.05/vCPU-hour + $0.005/GB-hour
```

**Pros**: Simpler, no EC2 mgmt, auto-scaling easier
**Cons**: Slightly more expensive, less control

### EC2 Launch Type

You manage EC2 instances, ECS schedules containers.

```
You have: EC2 instance (t3.medium, 2 vCPU, 4GB RAM)
ECS schedules: Containers on available capacity
```

**Pros**: Cheaper, full control
**Cons**: You manage EC2 (patching, restarts, etc)

**Recommendation**: Use Fargate for simplicity. Cost difference is small.

## ECR: Container Registry

ECR = AWS's Docker Hub.

Store your Docker images:

```bash
# Create ECR repository
aws ecr create-repository --repository-name pdf-api --region eu-west-1

# Get login token
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.eu-west-1.amazonaws.com

# Build image locally
docker build -t pdf-api:v1 .

# Tag for ECR
docker tag pdf-api:v1 123456789012.dkr.ecr.eu-west-1.amazonaws.com/pdf-api:v1

# Push to ECR
docker push 123456789012.dkr.ecr.eu-west-1.amazonaws.com/pdf-api:v1

# When you update task definition, ECS pulls new image from ECR
```

## Deployment Workflow

1. **Build image**
   ```bash
   docker build -t pdf-api:v2 .
   ```

2. **Push to ECR**
   ```bash
   docker tag pdf-api:v2 ACCOUNT.dkr.ecr.REGION.amazonaws.com/pdf-api:v2
   docker push ACCOUNT.dkr.ecr.REGION.amazonaws.com/pdf-api:v2
   ```

3. **Update task definition**
   ```bash
   # Register new version
   aws ecs register-task-definition --cli-input-json file://task-def.json
   ```

4. **Update ECS service**
   ```bash
   aws ecs update-service \
     --cluster pdf-platform \
     --service api-service \
     --task-definition pdf-api:2  # New version
   ```

5. **Monitoring**
   ```bash
   # Watch rolling update
   aws ecs describe-services --cluster pdf-platform \
     --services api-service \
     --query 'services[0].runningCount'
   
   # Old tasks = gradual stop
   # New tasks = gradual start
   # No downtime!
   ```

## ECS Service Auto-Scaling

Tell ECS to scale based on metrics:

```bash
# Register auto-scaling target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/pdf-platform/api-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

# Create scaling policy (scale up at 70% CPU)
aws application-autoscaling put-scaling-policy \
  --policy-name api-scale-up \
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
```

Now:
- **Normal**: 2 tasks running
- **Load spike**: CPU rises to 75%
- **Auto-scale**: 4 tasks
- **CPU drops**: Scale back to 2 tasks

## Hands-On Lab

### Lab 1.1: Push Image to ECR

```bash
# 1. Create ECR repository
REPO=$(aws ecr create-repository \
  --repository-name pdf-api-lab \
  --region eu-west-1 \
  --query 'repository.repositoryUri' \
  --output text)

echo $REPO  # 123456789012.dkr.ecr.eu-west-1.amazonaws.com/pdf-api-lab

# 2. Build your Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5000"]
EOF

# 3. Build image
docker build -t pdf-api-lab:v1 .

# 4. Login to ECR
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS --password-stdin $REPO

# 5. Tag for ECR
docker tag pdf-api-lab:v1 $REPO:v1

# 6. Push to ECR
docker push $REPO:v1

# 7. Verify
aws ecr list-images --repository-name pdf-api-lab
```

### Lab 1.2: Run Task on Fargate

```bash
# 1. Create task definition (JSON)
cat > task-def.json << 'EOF'
{
  "family": "pdf-api-lab",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "REPO:v1",
      "memory": 512,
      "portMappings": [
        {
          "containerPort": 5000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "DATABASE_URL",
          "value": "postgresql://user:pass@rds-endpoint:5432/db"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/pdf-api-lab",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole"
}
EOF

# Replace REPO with your ECR URI
sed -i "s|REPO|$REPO|g" task-def.json

# 2. Create log group
aws logs create-log-group --log-group-name /ecs/pdf-api-lab

# 3. Register task definition
aws ecs register-task-definition --cli-input-json file://task-def.json

# 4. Run task (one-off)
aws ecs run-task \
  --cluster pdf-platform \
  --task-definition pdf-api-lab:1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-123],securityGroups=[sg-123]}"

# 5. Get task info
aws ecs list-tasks --cluster pdf-platform
aws ecs describe-tasks --cluster pdf-platform --tasks <TASK_ARN>

# Check logs
aws logs tail /ecs/pdf-api-lab --follow
```

## Cheat Sheet: ECS CLI Commands

```bash
# Create cluster
aws ecs create-cluster --cluster-name my-cluster

# Register task definition
aws ecs register-task-definition --cli-input-json file://task-def.json

# Create service
aws ecs create-service --cluster my-cluster \
  --service-name my-service --task-definition my-task:1 \
  --desired-count 2 --launch-type FARGATE \
  --network-configuration "..."

# Update service
aws ecs update-service --cluster my-cluster --service my-service \
  --task-definition my-task:2  # New version

# Describe service (see running tasks)
aws ecs describe-services --cluster my-cluster --services my-service

# List tasks
aws ecs list-tasks --cluster my-cluster

# View task logs
aws logs tail /ecs/my-service --follow

# Push to ECR
docker push 123456789012.dkr.ecr.REGION.amazonaws.com/image:tag
```

## Key Takeaways

- **ECS** = managed container orchestration (like Kubernetes, simpler)
- **Fargate** = AWS manages compute, you manage containers
- **Task definition** = like docker-compose service ( image, ports, env, memory)
- **Service** = keep X tasks running, auto-restart failures
- **ECR** = Docker registry on AWS
- **Auto-scaling** = scale containers based on CPU, memory, custom metrics
- **Rolling deployments** = graceful task replacement, zero downtime
- **Cost**: Fargate $0.05/vCPU-hour reasonable for small apps

Module 2 teaches ElastiCache — managed Redis.
