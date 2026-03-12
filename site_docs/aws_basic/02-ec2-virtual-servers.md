# Module 2: EC2 Virtual Servers

## What Is EC2?

EC2 (Elastic Compute Cloud) is a virtual machine in the cloud.

It's **exactly like your Hetzner VPS**, except:
- Rent by the hour (not by the year)
- Scale up/down in minutes
- Pay for only what you use

Your Hetzner VPS: $15/month, 1 machine, lasts 12 months
EC2 t3.micro: $0.0104/hour, scale to 1000 machines, pay per second

## Instance Types

Instance type = the compute capacity (CPU, RAM, network).

Naming: `t3.micro` = letter + generation + size

### Letters (Purpose)

- **t** (burstable): Good for variable workloads (development, small apps)
  - Normally 20% CPU, sometimes 90% fine
  - Cost-effective
  - Your PDF platform probably uses this

- **m** (general purpose): Balanced (web servers, app servers)
  - Consistent workload, mix of CPU and memory

- **c** (compute optimized): High CPU (processing-heavy tasks)
  - For Celery workers doing heavy PDF conversion
  - More CPU-per-dollar, less memory

- **r** (memory optimized): High RAM (large databases, caches)
  - Not for your platform (use RDS/ElastiCache instead)

### Generation (Number)

- **t3** = newer, more efficient than t2
- t3.micro > t2.micro in performance, same price

### Size

- **micro** (free tier): 1 vCPU, 1 GB RAM
- **small**: 1 vCPU, 2 GB RAM
- **medium**: 1 vCPU, 4 GB RAM
- **large**: 2 vCPU, 8 GB RAM
- **xlarge**: 4 vCPU, 16 GB RAM

## AMI (Amazon Machine Image)

AMI = snapshot of a machine disk.

When you launch an EC2, you pick an AMI:
- Ubuntu 22.04 LTS (free tier eligible)
- Amazon Linux 2
- Windows Server
- CentOS
- Custom AMIs (you create your own)

**For your platform**: Use **Ubuntu 22.04 LTS** (same as Hetzner).

## Key Pairs: SSH Access

EC2 only allows SSH via key pair (no password login).

### Creating a Key Pair

```bash
# Create key pair (one-time, per region)
aws ec2 create-key-pair --key-name pdf-platform-key --region eu-west-1 \
  > pdf-platform-key.pem

# Set permissions (AWS requires)
chmod 600 pdf-platform-key.pem

# Save this file safely! If lost, you cannot access the instance.
```

### SSH Into EC2

```bash
# Get the instance's public IP
aws ec2 describe-instances --instance-ids i-1234567890abcdef0 \
  --query 'Reservations[0].Instances[0].PublicIpAddress'
# Returns: 203.0.113.42

# SSH in
ssh -i pdf-platform-key.pem ubuntu@203.0.113.42

# You're now inside the machine!
```

## Security Groups: Firewall

Security group = instance-level firewall.

**Inbound rules**: What traffic is allowed in
**Outbound rules**: What traffic is allowed out (usually allow all)

### Example: API with Database

```
API EC2 instance needs:
  - Port 22 (SSH) from your IP only
  - Port 5000 (API) from Load Balancer only

Creator:
aws ec2 create-security-group --group-name api-sg \
  --description "API security group" --vpc-id vpc-123

# Allow SSH from your IP
aws ec2 authorize-security-group-ingress --group-id sg-123 \
  --protocol tcp --port 22 --cidr 203.0.113.0/24  # Your IP range

# Allow port 5000 from ALB
aws ec2 authorize-security-group-ingress --group-id sg-123 \
  --protocol tcp --port 5000 \
  --source-group sg-alb  # ALB's security group
```

## Elastic IP: Static Address

By default, EC2 has a dynamic public IP. Reboot = new IP.

Elastic IP = static public IP that persists across reboots.

```bash
# Allocate elastic IP
aws ec2 allocate-address --domain vpc --region eu-west-1
# Returns: AllocationId, PublicIp

# Associate with instance
aws ec2 associate-address --instance-id i-123456 \
  --allocation-id eipalloc-123456
```

## EBS Volumes: Disk

EBS = Amazon's managed disk.

- Normal case: 1 EBS volume (root volume) attached to EC2
- Can attach additional volumes
- Persists if instance stops (unlike ephemeral storage)

### Volume Types

- **gp3** (General Purpose, default): Good for most workloads, $0.10/GB/month
- **io1** (Provisioned IOPS): High-performance, expensive
- **st1** (Throughput Optimized): Large sequential reads

For your platform: **gp3**

### Size

- Free tier: 30 GB total
- Typical: 20 GB root, 50 GB for databases

## User Data: Setup Script

User data = script that runs when instance boots.

```bash
#!/bin/bash
set -e

# Update packages
apt-get update
apt-get upgrade -y

# Install Docker
apt-get install -y docker.io docker-compose

# Start Docker
systemctl enable docker
systemctl start docker

# Clone your repo
cd /home/ubuntu
git clone https://github.com/yourname/pdf-platform.git
cd pdf-platform

# Start services
docker-compose up -d

# Done! API is running
```

Pass this as user data when launching instance:

```bash
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --user-data file://setup.sh \
  --key-name pdf-platform-key
```

## The Full Workflow

### Step 1: Create Security Group

```bash
aws ec2 create-security-group \
  --group-name pdf-api-sg \
  --description "Security group for PDF API" \
  --vpc-id vpc-12345

# Save the GROUP_ID returned (sg-123...)

# Allow SSH from your IP
aws ec2 authorize-security-group-ingress \
  --group-id sg-123 \
  --protocol tcp --port 22 \
  --cidr 0.0.0.0/0  # (⚠️ only for testing; restrict in production)

# Allow HTTP from ALB
aws ec2 authorize-security-group-ingress \
  --group-id sg-123 \
  --protocol tcp --port 80 \
  --cidr 0.0.0.0/0  # In production, use source-group
```

### Step 2: Launch Instance

```bash
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0  # Ubuntu 22.04 LTS \
  --instance-type t3.micro \
  --key-name pdf-platform-key \
  --security-group-ids sg-123 \
  --user-data file://setup.sh \
  --region eu-west-1
```

Returns: InstanceId

### Step 3: Wait for Boot

```bash
# Wait ~2 minutes for instance to boot
sleep 120

# Check status
aws ec2 describe-instances --instance-ids i-123456 \
  --query 'Reservations[0].Instances[0].State.Name'
# Returns: "running"

# Get public IP
aws ec2 describe-instances --instance-ids i-123456 \
  --query 'Reservations[0].Instances[0].PublicIpAddress'
```

### Step 4: SSH In

```bash
ssh -i pdf-platform-key.pem ubuntu@IP_ADDRESS

# Once inside:
docker ps  # See running containers
docker logs -f pdf_api_1  # Watch logs
```

### Step 5: Stop (Don't Delete)

```bash
# Stop instance (costs $0 while stopped, can restart)
aws ec2 stop-instances --instance-ids i-123456

# Or terminate (delete, no way to recover)
aws ec2 terminate-instances --instance-ids i-123456
```

## EC2 in Practice: When to Use?

**Use EC2** for:
- Development/testing (launch, delete when done)
- Long-running services that need custom setup
- Stepping stone to ECS/Fargate (which is easier)

**Avoid EC2** for:
- Production API servers (use ECS instead, auto-scaling)
- Databases (use RDS)
- Caches (use ElastiCache)
- Job queues (use SQS or ECS)

Your platform:
- **Development**: EC2 t3.micro for testing ($0/month with free tier)
- **Production**: ECS Fargate (Module 1 of Advanced)

## Hands-On Lab

### Lab 2.1: Launch an EC2 Instance with Docker

```bash
# 1. Create VPC (optional, use default for now)
# 2. Create security group
aws ec2 create-security-group \
  --group-name pdf-lab-sg \
  --description "Lab security group" \
  --region eu-west-1

# Save GROUP_ID returned

# 3. Allow SSH
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp --port 22 \
  --cidr 0.0.0.0/0 \
  --region eu-west-1

# 4. Create setup.sh
cat > setup.sh << 'EOF'
#!/bin/bash
set -e
apt-get update
apt-get install -y docker.io docker-compose
systemctl start docker
EOF

# 5. Create key pair
aws ec2 create-key-pair --key-name lab-key --region eu-west-1 > lab-key.pem
chmod 600 lab-key.pem

# 6. Launch instance
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --key-name lab-key \
  --security-group-ids sg-xxx \
  --user-data file://setup.sh \
  --region eu-west-1

# Note INSTANCE_ID returned

# 7. Wait ~2 minutes
sleep 120

# 8. Get public IP
IP=$(aws ec2 describe-instances \
  --instance-ids <INSTANCE_ID> \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

# 9. SSH in
ssh -i lab-key.pem ubuntu@$IP

# 10. Once inside, test:
docker --version
docker ps

# 11. Launch a simple container
docker run -d -p 5000:5000 flask:latest

# 12. From your computer (not SSH):
curl http://$IP:5000

# 13. Stop instance (or terminate)
aws ec2 stop-instances --instance-ids <INSTANCE_ID>
```

## Cheat Sheet: EC2 CLI Commands

```bash
# Create key pair
aws ec2 create-key-pair --key-name MY_KEY > my-key.pem
chmod 600 my-key.pem

# Create security group
aws ec2 create-security-group --group-name MY_SG --description "..." --vpc-id VPC_ID

# Allow traffic
aws ec2 authorize-security-group-ingress --group-id SG_ID \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

# Launch instance
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro --key-name MY_KEY --security-group-ids SG_ID

# List instances
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"

# Get public IP
aws ec2 describe-instances --instance-ids INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress'

# SSH in
ssh -i my-key.pem ubuntu@IP_ADDRESS

# Stop instance (keep data)
aws ec2 stop-instances --instance-ids INSTANCE_ID

# Terminate instance (delete)
aws ec2 terminate-instances --instance-ids INSTANCE_ID
```

## Instance Type Reference

| Type | vCPU | RAM | $/hour | Typical Use |
|------|------|-----|--------|-----------|
| t3.micro | 1 | 1GB | $0.0104 | Dev, testing (free tier) |
| t3.small | 2 | 2GB | $0.0208 | Small app |
| t3.medium | 2 | 4GB | $0.0416 | Medium app |
| m5.large | 2 | 8GB | $0.096 | Web server |
| c5.large | 2 | 4GB | $0.085 | Compute-heavy |

## Key Takeaways

- **EC2** = virtual machine on AWS, same as Hetzner VPS
- **Instance types**: t3 (burstable), m5 (general), c5 (compute), r5 (memory)
- **AMI**: Snapshot of OS (Ubuntu 22.04 LTS recommended)
- **Key pair**: SSH authentication (single access point)
- **Security group**: Firewall rules (inbound/outbound)
- **Elastic IP**: Static public IP
- **EBS**: Managed disks
- **User data**: Setup script on first boot
- **Cheap for dev**: t3.micro is $0 (free tier 12 months)
- **Use Fargate in production**: ECS Fargate auto-scales better

Module 3 teaches S3 — your replacement for MinIO.
