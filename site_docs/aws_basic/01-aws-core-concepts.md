# Module 1: AWS Core Concepts

## What Is AWS?

AWS (Amazon Web Services) is the world's largest cloud provider.

Instead of buying servers:
```
❌ Buy Dell server ($5,000 upfront)
   ├─ Rent server space ($100/month)
   ├─ Wait for delivery
   ├─ Rack it yourself
   └─ Maintain it for 5 years
```

You rent services per-hour:
```
✅ Launch EC2 instance (1 minute)
   ├─ Pay $0.02 per hour while running
   ├─ Stop it when done (pay $0)
   └─ Delete it when no longer needed
```

AWS offers:
- **Compute**: EC2 (virtual machines), ECS (containers), Lambda (small functions)
- **Storage**: S3 (files), EBS (disks), Glacier (archive)
- **Database**: RDS (PostgreSQL, MySQL), DynamoDB (NoSQL)
- **Cache**: ElastiCache (Redis, Memcached)
- **Networking**: VPC (private network), ALB (load balancer), CloudFront (CDN)
- **Analytics**: SQS (queues), Kinesis (streaming), Athena (SQL on S3)
- (200+ services total)

## Regions and Availability Zones

AWS is globally distributed.

### Regions
A geographic area (US East, EU West, Asia Pacific):
- us-east-1 (N. Virginia)
- eu-west-1 (Ireland)
- ap-southeast-1 (Singapore)

### Availability Zones (AZs)
Each region has multiple isolated data centers:

```
Region: us-east-1
  ├─ AZ: us-east-1a
  │   └─ Data center (isolated power, cooling, network)
  ├─ AZ: us-east-1b
  │   └─ Data center (independent)
  └─ AZ: us-east-1c
      └─ Data center (independent)
```

**Why it matters**:
- **Latency**: Closer region = lower latency. EU users should use eu-west-1.
- **Redundancy**: If AZ-1 has a power outage, AZ-2 and AZ-3 are unaffected.
  - You can run 2 API instances in different AZs
  - If AZ-1 fails, AZ-2 handles traffic
  - Automatic failover (if using RDS Multi-AZ)

## AWS Free Tier

You get free AWS credits:

### Always Free (no expiration)
- EC2: 750 hours/month of t2.micro (1 instance continuously)
- RDS: 750 hours/month of db.t2.micro
- ElastiCache: 750 hours/month of cache.t2.micro
- S3: 5GB storage + 20k GET/PUT requests
- Lambda: 1 million requests/month

### Free for 12 Months (from account creation)
- EC2 t3.micro (better than t2.micro)
- RDS db.t3.micro
- ElastiCache cache.t3.micro
- 100GB data transfer OUT per month

### How to Avoid Getting Charged
- **Set a billing alarm**: Alert if bill > $50/month
- **Check Cost Explorer**: See what services are costing money
- **Stop instances when not using**: Stopped instance costs $0 (no data charge)
- **Delete volumes and snapshots**: They cost money even if not attached
- **Watch data transfer**: Downloading 100GB costs $15

## IAM: Identity and Access Management

**Never use the AWS root account** (the one created with your email).

Root account = full access = security disaster.

Instead: Create IAM users with limited permissions.

### IAM Concepts

**Users**: Individual people (you, your teams)
**Groups**: Collection of users (Backend Team, DevOps Team)
**Roles**: Permissions that services assume (EC2 role, ECS task role)
**Policies**: Rules defining what can be done (allow S3 access, deny DynamoDB)

### Setup

1. **Create IAM user** (e.g., `dev-user`)
   - Attach policy: `AdministratorAccess` (for development)
   - Generate access key + secret access key (for CLI)

2. **Enable MFA on root account** (2FA with phone)

3. **Use IAM user for everything**

```bash
# Configure AWS CLI with IAM user credentials
aws configure
# Access key ID: AKIA...
# Secret access key: ****...
# Region: eu-west-1 (or your region)
# Output format: json
```

## The AWS Console, CLI, and SDK

### 1. AWS Console (Web UI)

Visit `https://console.aws.amazon.com`.

Good for:
- Visual exploration
- Debugging
- One-time setups

Bad for:
- Repeated tasks (slow, error-prone)
- Automation
- Version control

### 2. AWS CLI

Command-line tool for AWS:

```bash
# List S3 buckets
aws s3 ls

# Launch an EC2 instance
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 --instance-type t3.micro

# Create RDS instance
aws rds create-db-instance --db-instance-class db.t3.micro --db-instance-identifier mydb

# Easier to script, version-control, automate
```

### 3. AWS SDK (Python: boto3)

Call AWS from your code:

```python
import boto3

s3 = boto3.client('s3')

# Upload file to S3
s3.put_object(
    Bucket='my-bucket',
    Key='document.pdf',
    Body=open('document.pdf', 'rb')
)

# List files
response = s3.list_objects_v2(Bucket='my-bucket')
for obj in response.get('Contents', []):
    print(obj['Key'])
```

## AWS Pricing Model

AWS doesn't bill monthly. It bills for **what you use**.

### Example: EC2 t3.micro

- **Price**: $0.0104 per hour (us-east-1)
- **50 hours used**: 50 * $0.0104 = $0.52
- **730 hours/month full utilization**: 730 * $0.0104 = $7.59/month

### RDS db.t3.micro

- **Price**: $0.014 per hour
- **730 hours/month**: ~$10/month
- **Storage**: $0.10 per GB/month (typical: 20GB = $2/month)

### S3 Standard

- **Storage**: $0.023 per GB/month (1TB = $23/month)
- **Requests**: $0.0004 per PUT, $0.00008 per GET

### Data Transfer OUT

- **First 1GB**: Free
- **1GB - 10TB**: $0.12 per GB
- **10TB - 50TB**: $0.10 per GB
- **Downloading 100GB**: ~$12 charge (this surprises people)

## Estimating Costs

Use AWS Pricing Calculator: https://calculator.aws/

Example setup for your PDF platform:

```
┌─ EC2 t3.micro x 3
│    (API instances, 730 hours/month)
│    = 3 × $7.59 = $22.77

├─ RDS db.t3.micro
│    (PostgreSQL, 730 hours/month + 100GB storage)
│    = $10 + $10 = $20

├─ ElastiCache cache.t3.micro
│    (Redis, 730 hours/month)
│    = $14

├─ ALB (Load Balancer)
│    (LCU charges, typical small app)
│    = $16

├─ S3
│    (100GB storage + transfers)
│    = $2.30 + $5 = $7.30

├─ Data Transfer OUT
│    (1TB/month to internet)
│    = $100

├─ CloudWatch logs + metrics
│    = $5

└─ NAT Gateway (for private subnet outbound)
   = $32 (fixed) + $0.06 per GB processed

TOTAL: ~$200/month (after free tier)
```

**With free tier (first 12 months)**: ~$30/month
**After free tier**: ~$150-200/month (depending on traffic)

## Avoiding Bill Shock

1. **Set billing alerts**
   - https://console.aws.amazon.com/billing/home#/account
   - Click on "Billing Preferences"
   - Set "Estimated Bill Alert": $50/month

2. **Use Cost Explorer**
   - See daily costs by service
   - Identify expensive services
   - Set budget limits

3. **Tag resources**
   ```bash
   # Tag EC2 instance with cost center
   aws ec2 create-tags --resources i-1234567890abcdef0 \
     --tags Key=CostCenter,Value=Development Key=Project,Value=PDFPlatform
   ```

4. **Delete what you don't use**
   - Stop EC2 instances when not needed
   - Delete unattached volumes and snapshots
   - Empty old S3 buckets or move to Glacier

## Hands-On Lab

### Lab 1.1: Create AWS Account and Set Up Credentials

```bash
# 1. Create account at https://aws.amazon.com
#    (uses your email, credit card, phone)

# 2. Enable MFA on root account
#    - AWS Console → Account dropdown → Security Credentials
#    - "Activate MFA Device"
#    - Use Google Authenticator or Authy

# 3. Create IAM user
aws iam create-user --user-name dev-user

# 4. Attach admin policy
aws iam attach-user-policy --user-name dev-user \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 5. Create access key
aws iam create-access-key --user-name dev-user
# Returns: AccessKeyId and SecretAccessKey
# (save these safely!)

# 6. Configure AWS CLI
aws configure
# AWS Access Key ID: AKIA...
# AWS Secret Access Key: ***...
# Default region: eu-west-1 (pick your region)
# Default output format: json

# 7. Verify it works
aws sts get-caller-identity
# Returns your account ID and IAM user ARN
```

### Lab 1.2: Check Estimated Costs

```bash
# Use AWS Pricing Calculator
# https://calculator.aws/

# Estimate this setup:
# - 1 x EC2 t3.micro (50 hours/month for testing)
# - 1 x RDS db.t3.micro
# - 1 x ElastiCache cache.t3.micro
# - 1 x ALB
# - 100GB S3 storage
# - 1TB outbound data transfer

# Expected: ~$50-100/month (more if you run continuously)
```

## Cheat Sheet: AWS Core Concepts

### AWS CLI Basics

```bash
# Configure credentials
aws configure

# Check identity
aws sts get-caller-identity

# List available regions
aws ec2 describe-regions --all-regions

# Check current region
echo $AWS_DEFAULT_REGION
```

### IAM Setup Checklist

```
☐ Create AWS account
☐ Enable MFA on root account
☐ Create dev-user IAM user
☐ Attach AdministratorAccess policy
☐ Generate access key for dev-user
☐ Run aws configure with dev-user credentials
☐ Set billing alert ($50/month)
☐ Never use root account again
```

### Regions Reference

| Region | Code | When to Use |
|--------|------|-----------|
| US East (N. Virginia) | us-east-1 | US users, cheapest, most services |
| US West (Oregon) | us-west-2 | West Coast US |
| EU (Ireland) | eu-west-1 | European users |
| Asia Pacific (Singapore) | ap-southeast-1 | Asian users |
| Asia Pacific (Tokyo) | ap-northeast-1 | Japan/Korea users |

### Pricing Estimate Command

```bash
# AWS Pricing Calculator (web)
# https://calculator.aws/#/

# Or rough calculation:
# t3.micro: $0.0104/hour → $7.60/month
# db.t3.micro: $0.014/hour → $10.20/month
# cache.t3.micro: $0.019/hour → $13.87/month
```

## Key Takeaways

- **AWS** = rent servers and services by the hour, not by the machine
- **Regions** = geographic areas (US, EU, Asia); pick one close to your users
- **AZs** = isolated data centers within a region; use for redundancy
- **Free Tier** = 750 hours/month of micro instances, 12 months free or always free
- **Root account** = never use it; create IAM user with limited permissions
- **AWS CLI** = automate tasks, version control infrastructure
- **Billing** = set alerts, use Cost Explorer, delete unused resources
- **Pricing** = typically $50-200/month for small-medium apps (after free tier)

Module 2 teaches EC2 — your first virtual machine on AWS.
