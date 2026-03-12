# Module 5: VPC and Networking

## What Is a VPC?

VPC (Virtual Private Cloud) = your private network inside AWS.

Analogy: You rent a gated community. Your VPC = the fence around your properties.

```
AWS Region (country)
  ├─ Your VPC (gated community)
  │   ├─ Public Subnet (visible from street)
  │   │   └─ EC2 instance with public IP
  │   ├─ Private Subnet (hidden behind gate)
  │   │   ├─ RDS database (no internet access)
  │   │   └─ ElastiCache (no internet access)
  │   └─ Internet Gateway (gate to internet)
  │
  ├─ Other VPCs (other people's communities)
  └─ AWS services (public internet)
```

By default, AWS creates a default VPC for you. For production, create a custom VPC.

## Subnets: Public vs Private

Subnet = slice of VPC with specific routing rules.

### Public Subnet

Route table:
```
Destination    Target
0.0.0.0/0      Internet Gateway  (sends all external traffic to IGW)
```

Anything in public subnet can reach the internet (and vice versa).

**Put here**: Nginx (ALB), API servers (initially)

### Private Subnet

Route table:
```
Destination    Target
0.0.0.0/0      NAT Gateway  (only outbound, not inbound)
```

Can initiate connections out (e.g., download packages), but internet cannot initiate connections in.

**Put here**: RDS databases, ElastiCache, Celery workers (secrets safe)

## Security Architecture

```
┌─────────────────────────────────────── VPC ──────────────────────────┐
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ Public Subnet (10.0.1.0/24)                             │      │
│  │                                                          │      │
│  │  ┌────────────────┐         ┌──────────────────┐       │      │
│  │  │ ALB (Load      │────────→│ Security Group   │       │      │
│  │  │ Balancer)      │ :80/443 │ Allow: 80, 443   │       │      │
│  │  └────────────────┘         └──────────────────┘       │      │
│  │                                                          │      │
│  │  ┌────────────────┐         ┌──────────────────┐       │      │
│  │  │ API Container  │────────→│ Security Group   │       │      │
│  │  │ (ECS Task)     │ :5000   │ Allow: 5000      │       │      │
│  │  └────────────────┘         │ From: ALB SG     │       │      │
│  │                             └──────────────────┘       │      │
│  └──────────────────────────────────────────────────────────┘      │
│                    ↓ (ENI = Elastic Network Interface)             │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ Private Subnet (10.0.2.0/24)                            │      │
│  │                                                          │      │
│  │  ┌────────────────┐         ┌──────────────────┐       │      │
│  │  │ RDS PostgreSQL │────────→│ Security Group   │       │      │
│  │  │ (Primary)      │ :5432   │ Allow: 5432      │       │      │
│  │  └────────────────┘         │ From: API SG     │       │      │
│  │                             └──────────────────┘       │      │
│  │  ┌────────────────┐         ┌──────────────────┐       │      │
│  │  │ ElastiCache    │────────→│ Security Group   │       │      │
│  │  │ Redis          │ :6379   │ Allow: 6379      │       │      │
│  │  └────────────────┘         │ From: API SG     │       │      │
│  │                             └──────────────────┘       │      │
│  │  ┌────────────────┐         ┌──────────────────┐       │      │
│  │  │ Celery Worker  │────────→│ Security Group   │       │      │
│  │  │ (ECS Task)     │ :5555   │ Allow: 5555      │       │      │
│  │  └────────────────┘         │ From: ALB SG     │       │      │
│  │                             └──────────────────┘       │      │
│  └──────────────────────────────────────────────────────────┘      │
│                    ↓ (NAT Gateway)                                 │
│              (Outbound internet only)                              │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
                    ↓
            Internet Gateway
                    ↓
              External Internet
```

**Security principle**: 
- API in public subnet (receives ALB traffic)
- Database in private subnet (not reachable from internet)
- Database security group allows traffic only from API security group

## Internet Gateway

Gateway connecting your VPC to the public internet.

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region eu-west-1
# Returns VPC_ID

# Create Internet Gateway
aws ec2 create-internet-gateway --region eu-west-1
# Returns IGW_ID

# Attach IGW to VPC
aws ec2 attach-internet-gateway --vpc-id VPC_ID --internet-gateway-id IGW_ID

# Create route in public subnet
aws ec2 create-route --route-table-id RTB_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id IGW_ID
```

Now public subnet can reach the internet.

## NAT Gateway

NAT = Network Address Translation. Lets private subnet reach the internet (outbound only).

```
Private EC2 (10.0.2.5):
  → Initiates connection to github.com via NAT Gateway
  → NAT translates source IP (10.0.2.5 → NAT public IP)
  → github.com receives request
  → github.com responds
  → NAT translates back (NAT public IP → 10.0.2.5)
  → Private EC2 receives response

However:
  Internet cannot initiate → Private EC2
  (NAT blocks unsolicited inbound)
```

Cost: $32/month fixed + $0.06/GB processed (in this region)

```bash
# Create Elastic IP for NAT Gateway
aws ec2 allocate-address --domain vpc --region eu-west-1
# Returns AllocationId

# Create NAT Gateway in public subnet
aws ec2 create-nat-gateway \
  --subnet-id PUBLIC_SUBNET_ID \
  --allocation-id ALLOCATION_ID
# Returns NatGatewayId

# Create route in private subnet
aws ec2 create-route --route-table-id PRIVATE_RTB_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id NAT_GATEWAY_ID
```

## Security Groups vs NACLs

### Security Groups (Instance-level)

- **Stateful**: If you allow outbound, inbound response is auto-allowed
- **Simpler**: Usually sufficient
- **Applied to**: EC2, RDS, ALB, each instance

Example:
```bash
# Allow API traffic in
aws ec2 authorize-security-group-ingress \
  --group-id sg-api \
  --protocol tcp --port 5000 \
  --source-group sg-alb  # From ALB security group

# Automatic rule 2: Response traffic from API outbound is allowed
```

### NACLs (Subnet-level)

- **Stateless**: Must allow both directions explicitly
- **More complex**: Rarely needed
- **Applied to**: Entire subnet

Usually skip NACLs. Security groups are sufficient.

## Real Architecture for Your Platform

```
Region: eu-west-1

VPC: 10.0.0.0/16
  │
  ├─ Public Subnet 1 (10.0.1.0/24, AZ: a)
  │   └─ ALB (no EC2, managed by AWS)
  │       Security Group: allow 80, 443 from 0.0.0.0/0
  │
  ├─ Public Subnet 2 (10.0.2.0/24, AZ: b)
  │   └─ (spare for ALB redundancy)
  │
  ├─ Private Subnet 1 (10.0.10.0/24, AZ: a)
  │   ├─ ECS Task 1 (API)
  │   │   Security Group: allow 5000 from ALB SG
  │   ├─ ECS Task N (API)
  │   │   Security Group: allow 5000 from ALB SG
  │   └─ NAT Gateway (for outbound to download packages)
  │
  ├─ Private Subnet 2 (10.0.11.0/24, AZ: b)
  │   ├─ ECS Task 1 (Celery Worker)
  │   │   Security Group: allow SSH (optional)
  │   └─ Celery heartbeat to Redis
  │
  ├─ RDS Multi-AZ (spans AZ-a and AZ-b)
  │   Primary: 10.0.10.10 (Private Subnet 1)
  │   Standby: 10.0.11.10 (Private Subnet 2)
  │   Security Group: allow 5432 from API SG, Celery SG
  │
  ├─ ElastiCache Redis Multi-AZ
  │   Primary: 10.0.10.20 (Private Subnet 1)
  │   Replica: 10.0.11.20 (Private Subnet 2)
  │   Security Group: allow 6379 from API SG, Celery SG
  │
  └─ Internet Gateway (for ALB to internet)
      NAT Gateway in Public Subnet 1 (for private outbound)
```

## Hands-On Lab

### Lab 5.1: Create VPC with Subnets

```bash
# 1. Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region eu-west-1 \
  --query 'Vpc.VpcId' --output text)

# 2. Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway --region eu-west-1 \
  --query 'InternetGateway.InternetGatewayId' --output text)

# 3. Attach IGW
aws ec2 attach-internet-gateway --vpc-id $VPC_ID \
  --internet-gateway-id $IGW_ID

# 4. Create public subnet (AZ-a)
PUBLIC_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone eu-west-1a \
  --query 'Subnet.SubnetId' --output text)

# 5. Create private subnet (AZ-a)
PRIVATE_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.10.0/24 \
  --availability-zone eu-west-1a \
  --query 'Subnet.SubnetId' --output text)

# 6. Create route table for public subnet
PUBLIC_RTB=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)

# 7. Route public traffic to IGW
aws ec2 create-route --route-table-id $PUBLIC_RTB \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# 8. Associate public subnet with route table
aws ec2 associate-route-table \
  --subnet-id $PUBLIC_SUBNET \
  --route-table-id $PUBLIC_RTB

echo "VPC: $VPC_ID"
echo "Public Subnet: $PUBLIC_SUBNET"
echo "Private Subnet: $PRIVATE_SUBNET"
```

### Lab 5.2: Test Private Subnet Isolation

```bash
# 1. Launch EC2 in public subnet
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --subnet-id $PUBLIC_SUBNET \
  --key-name lab-key

# Note: INSTANCE_ID_PUBLIC

# 2. Launch RDS in private subnet
aws rds create-db-instance \
  --db-instance-identifier private-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --master-user-password Password123! \
  --db-subnet-group-name default  # Uses VPC's private subnets
  --publicly-accessible false

# 3. SSH into public EC2
ssh -i lab-key.pem ubuntu@PUBLIC_IP

# 4. From inside public EC2, try to reach RDS
# (This might fail because RDS security group doesn't allow it)
# Modify RDS security group to allow from EC2's security group

# 5. Verify: Internet cannot reach RDS directly
# Try from your computer:
psql -h RDS_ENDPOINT -U admin -d postgres
# Times out (good! Security Group blocks it)

# 6. From public EC2, RDS is reachable (if SG allows)
# ssh into public EC2, then:
psql -h RDS_ENDPOINT -U admin -d postgres
# Works!
```

## Cheat Sheet: VPC Setup

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' --output text)

# Create subnets
PUBLIC_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --availability-zone REGION-a \
  --query 'Subnet.SubnetId' --output text)

PRIVATE_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.10.0/24 \
  --availability-zone REGION-a \
  --query 'Subnet.SubnetId' --output text)

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Create NAT Gateway
ALLOC=$(aws ec2 allocate-address --domain vpc \
  --query 'AllocationId' --output text)
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_SUBNET \
  --allocation-id $ALLOC \
  --query 'NatGateway.NatGatewayId' --output text)
```

## Key Takeaways

- **VPC** = private network inside AWS (your "gated community")
- **Public subnet** = internet-facing (ALB, NAT Gateway)
- **Private subnet** = hidden (RDS, ElastiCache, Celery workers)
- **Internet Gateway** = connects VPC to public internet
- **NAT Gateway** = private subnet outbound internet (costs $32+/month)
- **Security Groups** = instance-level firewall (stateful)
- **NACLs** = subnet-level firewall (usually skip)
- **Architecture**: Public ALB → Private API → Private RDS/ElastiCache
- **Multi-AZ**: Spread subnets across AZs for high availability
- **Cost**: VPC free, but NAT Gateway is $32+/month

This completes AWS Basics. Next: AWS Advanced (ECS, ElastiCache, SQS, IAM, Auto-Scaling).
