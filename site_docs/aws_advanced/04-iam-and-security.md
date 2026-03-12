# Module 4: IAM and Security

## What Is IAM?

IAM (Identity and Access Management) = AWS credential system.

Control who can access what.

### The Critical Rule

**NEVER use root AWS account credentials in code.**

```
ROOT ACCOUNT:
- Email: abdelmoteleb@example.com
- Password: ...
- MFA: ✓ (required!)
- AWS_ACCESS_KEY_ID: (NEVER share)
- AWS_SECRET_ACCESS_KEY: (NEVER share)
Purpose: Account recovery only, keep in vault

DEV ACCOUNT:
- Username: dev-user
- Password: (for login)
- AWS_ACCESS_KEY_ID: (for API)
- AWS_SECRET_ACCESS_KEY: (for API)
Purpose: Daily development
```

If root credentials leak: Attacker has your entire AWS account.

If dev credentials leak: Attacker has only what dev-user can access.

## IAM Basics

### Users

Real people accessing AWS.

```bash
# Create user
aws iam create-user --user-name dev-user

# Generate access keys (for API/CLI/SDK)
aws iam create-access-key --user-name dev-user

# Output:
# {
#   "AccessKeyId": "AKIA...",
#   "SecretAccessKey": "wJatXx..."
# }

# Save to ~/.aws/credentials (on laptop)
[dev-profile]
aws_access_key_id = AKIA...
aws_secret_access_key = wJatXx...

# Use in CLI
aws s3 ls --profile dev-profile
```

### Groups

Collection of users with same permissions.

```bash
# Create group
aws iam create-group --group-name developers

# Add user to group
aws iam add-user-to-group --group-name developers --user-name dev-user

# All developers get same permissions
```

### Roles

Services assume roles (no password, automatic credentials).

```
EC2 instance needs to read S3 bucket.
  ↓
Create role: "EC2-S3-Reader"
  ↓
Attach policy: "Allow S3 read on bucket pdf-uploads"
  ↓
EC2 instance gets role
  ↓
Instance automatically gets temporary credentials
  ↓
Code reads from S3 (credentials from IAM, not .env)
```

**Key insight**: Roles = credentials automatically injected, no hardcoding.

## Policies (JSON)

Policies define permissions.

### Example: Allow S3 Read-Only

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::pdf-uploads",
        "arn:aws:s3:::pdf-uploads/*"
      ]
    }
  ]
}
```

Breakdown:
- **Effect**: Allow or Deny
- **Action**: What operations (s3:GetObject = read file)
- **Resource**: On which resources (bucket + all objects)

### Least Privilege: The Golden Rule

Give only permissions needed, no more.

**Bad** ❌:
```json
{
  "Effect": "Allow",
  "Action": "s3:*",  # Everything
  "Resource": "*"    # Everything
}
```

**Good** ✅:
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::pdf-uploads/*"
}
```

## EC2 Instance Role (No .env Credentials)

### Old Way (Bad)

```bash
# EC2 instance ~/.env
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=wJatXx...
```

Problems:
- Credentials visible in files
- If instance compromised → credentials leaked
- Hard to rotate (change .env all instances)

### New Way (Good)

EC2 instance gets role automatically:

```bash
# 1. Create role
ROLE=$(aws iam create-role \
  --role-name EC2-S3-Uploader \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }]
  }' \
  --query 'Role.Arn' \
  --output text)

# 2. Create policy (allow S3 specific bucket)
POLICY=$(cat << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::pdf-uploads/*"
  }]
}
EOF
)

# 3. Attach policy to role
aws iam put-role-policy \
  --role-name EC2-S3-Uploader \
  --policy-name S3-policy \
  --policy-document "$POLICY"

# 4. Create instance profile
PROFILE=$(aws iam create-instance-profile \
  --instance-profile-name EC2-S3-Uploader \
  --query 'InstanceProfile.Arn' \
  --output text)

# 5. Add role to profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-Uploader \
  --role-name EC2-S3-Uploader

# 6. Launch EC2 with profile
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --iam-instance-profile Name=EC2-S3-Uploader

# 7. Inside EC2, no credentials in .env!
# Python code automatically gets credentials
import boto3
s3 = boto3.client('s3')
s3.put_object(Bucket='pdf-uploads', Key='file.pdf', Body=b'...')
```

AWS automatically injects temporary credentials into EC2 metadata service.

### Code (No .env)

```python
# Before (bad: credentials in code)
import boto3
s3 = boto3.client('s3',
    aws_access_key_id='AKIA...',
    aws_secret_access_key='wJatXx...'
)

# After (good: credentials from role)
import boto3
s3 = boto3.client('s3')  # Auto-reads from EC2 role
```

## ECS Task Role

Each ECS task gets its own role.

```
┌────────────────────┐
│ ECS Service        │
├────────────────────┤
│ Task 1 (role-1)    │ → Can read S3 bucket A only
├────────────────────┤
│ Task 2 (role-2)    │ → Can read database only
├────────────────────┤
│ Task 3 (role-3)    │ → Can write to SQS only
└────────────────────┘
```

Each task has different permissions.

Task definition:

```json
{
  "family": "pdf-api",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ECS-PDF-API",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "...",
      "environment": [
        // No AWS_ACCESS_KEY_ID!
        // No AWS_SECRET_ACCESS_KEY!
      ]
    }
  ]
}
```

Container automatically gets credentials (from task role).

## AWS Secrets Manager

Store secrets (API keys, passwords) securely.

```
Bad ❌:
  .env
  PAYMOB_API_KEY=pk_live_abc123...

Good ✅:
  Secrets Manager
  Secret: paymob-api-key
  Value: pk_live_abc123...
  Encrypted at rest
  IAM controlled access
  Automatic rotation
```

### Setup

```bash
# 1. Create secret
aws secretsmanager create-secret \
  --name paymob-api-key \
  --secret-string 'pk_live_abc123...'

# 2. Grant EC2 role permission
POLICY=$(cat << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "secretsmanager:GetSecretValue",
    "Resource": "arn:aws:secretsmanager:eu-west-1:123456789012:secret:paymob-api-key-*"
  }]
}
EOF
)

aws iam put-role-policy \
  --role-name EC2-S3-Uploader \
  --policy-name SecretsManager \
  --policy-document "$POLICY"

# 3. In code, retrieve secret
import boto3
import json

client = boto3.client('secretsmanager', region_name='eu-west-1')
secret = client.get_secret_value(SecretId='paymob-api-key')
paymob_key = secret['SecretString']

# Use paymob_key...
```

No hardcoded secrets, automatic rotation.

## CloudTrail: Audit Logging

Track all API calls (who, when, what).

```bash
# Enable CloudTrail (tracks all API calls)
aws cloudtrail start-logging --trail-name management-events

# Query logs
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=dev-user \
  --max-results 50

# Output:
# {
#   "Events": [
#     {
#       "EventName": "DeleteBucket",
#       "EventTime": "2024-01-15T10:30:00Z",
#       "Username": "dev-user",
#       "Resources": [{"ARN": "arn:aws:s3:::pdf-uploads"}]
#     }
#   ]
# }

# Detect: Who deleted the S3 bucket?
# Answer: dev-user at 10:30 UTC
```

Critical for compliance, debugging, security.

## Hands-On Lab

### Lab 4.1: Create EC2 Role (S3 Read-Only)

```bash
# 1. Create role
ROLE=$(aws iam create-role \
  --role-name EC2-S3-ReadOnly \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }' \
  --query 'Role.Arn' --output text)

# 2. Create policy (S3 read specific bucket)
POLICY='{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::pdf-uploads",
      "arn:aws:s3:::pdf-uploads/*"
    ]
  }]
}'

aws iam put-role-policy \
  --role-name EC2-S3-ReadOnly \
  --policy-name S3-ReadPolicy \
  --policy-document "$POLICY"

# 3. Create instance profile
aws iam create-instance-profile \
  --instance-profile-name EC2-S3-ReadOnly

aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-ReadOnly \
  --role-name EC2-S3-ReadOnly

# 4. Launch EC2 with role
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --iam-instance-profile Name=EC2-S3-ReadOnly \
  --query 'Instances[0].InstanceId' --output text

# 5. SSH into instance
# aws ssm start-session --target <INSTANCE_ID>

# 6. Test (should work)
python3 << 'EOF'
import boto3
s3 = boto3.client('s3')
s3.list_objects(Bucket='pdf-uploads')  # Works!
EOF

# 7. Test read-write (should fail)
python3 << 'EOF'
import boto3
s3 = boto3.client('s3')
s3.put_object(Bucket='pdf-uploads', Key='file.pdf', Body=b'...')
# ERROR: Access Denied! ✓
EOF
```

### Lab 4.2: Secrets Manager + Application

```bash
# 1. Create secret
aws secretsmanager create-secret \
  --name jwt-secret \
  --secret-string 'super-secret-key-change-me-in-production'

# 2. Grant role permission
POLICY='{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "secretsmanager:GetSecretValue",
    "Resource": "arn:aws:secretsmanager:*:*:secret:jwt-secret-*"
  }]
}'

aws iam put-role-policy \
  --role-name EC2-S3-ReadOnly \
  --policy-name SecretsPolicy \
  --policy-document "$POLICY"

# 3. Update application code
cat > app.py << 'EOF'
import boto3
import os
from fastapi import FastAPI, Depends
from jose import JWTError, jwt

client = boto3.client('secretsmanager')

def get_jwt_secret():
    """Retrieve JWT secret from Secrets Manager."""
    secret = client.get_secret_value(SecretId='jwt-secret')
    return secret['SecretString']

app = FastAPI()

@app.post("/login")
def login(username: str, password: str):
    jwt_secret = get_jwt_secret()
    token = jwt.encode(
        {"username": username},
        jwt_secret,
        algorithm="HS256"
    )
    return {"access_token": token}

@app.get("/profile")
def profile(token: str):
    jwt_secret = get_jwt_secret()
    try:
        payload = jwt.decode(token, jwt_secret, algorithms=["HS256"])
        return {"username": payload["username"]}
    except JWTError:
        return {"error": "Invalid token"}
EOF

# 4. Run (secrets retrieved automatically)
python app.py
# POST /login → generates token (secret from Secrets Manager)
# GET /profile?token=xyz → validates token (secret from Secrets Manager)
```

## Cheat Sheet: IAM Commands

```bash
# Create user
aws iam create-user --user-name USERNAME
aws iam create-access-key --user-name USERNAME

# Create role
aws iam create-role --role-name ROLE_NAME \
  --assume-role-policy-document file://trust-policy.json

# Attach policy to role
aws iam put-role-policy --role-name ROLE_NAME \
  --policy-name POLICY_NAME \
  --policy-document file://policy.json

# Create instance profile
aws iam create-instance-profile --instance-profile-name PROFILE_NAME
aws iam add-role-to-instance-profile \
  --instance-profile-name PROFILE_NAME \
  --role-name ROLE_NAME

# List all access keys (check who uses what)
aws iam list-access-keys --user-name USERNAME

# Rotate access key (security best practice)
aws iam delete-access-key --user-name USERNAME --access-key-id AKIA...

# CloudTrail
aws cloudtrail start-logging --trail-name NAME
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=USERNAME

# Secrets Manager
aws secretsmanager create-secret --name SECRET_NAME --secret-string VALUE
aws secretsmanager get-secret-value --secret-id SECRET_NAME
aws secretsmanager rotate-secret --secret-id SECRET_NAME
```

## Key Takeaways

- **Never** use root AWS account credentials in code
- **Always** use IAM roles for EC2/ECS (credentials auto-injected)
- **Least privilege** = grant only needed permissions
- **Secrets Manager** = store API keys, passwords (encrypted, rotatable)
- **CloudTrail** = audit all API calls (who deleted that bucket?)
- **Instance profile** = role attached to EC2 instance
- **Task role** = each ECS task gets own permissions
- **EC2 metadata service** = instance retrieves credentials automatically

Module 5 teaches Auto-Scaling and Load Balancing — making it all work at scale.
