# Module 3: S3 Object Storage

## What Is S3?

S3 (Simple Storage Service) is MinIO in the cloud.

Like MinIO:
- Upload files with a key
- Download files with a key
- Supports presigned URLs (temporary access)
- Bucket policy for access control

Unlike MinIO:
- AWS manages it (you don't run containers)
- 99.999999999% durability (11 nines!)
- Global replication possible
- Cheaper at scale

Your current stack: MinIO container running 24/7 ($0 free tier cost, but costs server resources)
With S3: AWS manages it, costs $0.023/GB/month for first 1TB

## Buckets and Objects

**Bucket** = storage container (like a volume in MinIO)
- Must have globally unique name: `pdf-platform-2024`
- Contain "objects" (files)

**Object** = file with a key
- Key: path in bucket (e.g., `documents/user-123/report.pdf`)
- Contains metadata, ACLs

```
Bucket: pdf-platform-2024
  ├─ Key: documents/user-123/report.pdf (5 MB file)
  ├─ Key: images/logo.png (200 KB)
  └─ Key: backups/2024-01-15/db.sql.gz (1 GB)
```

## Storage Classes: Cost vs Speed

Different tiers based on access patterns:

### S3 Standard
- **Price**: $0.023/GB/month
- **Retrieval**: Instant
- **Use**: Frequently accessed (your documents, reports)

### S3 Standard-IA (Infrequent Access)
- **Price**: $0.0125/GB/month (saves 45%)
- **Retrieval**: Instant, but $0.01 per GET
- **Minimum**: Must keep for 30 days (or pay penalty)
- **Use**: Backups, archives you might access

### S3 Glacier (Archive)
- **Price**: $0.004/GB/month (saves 82%)
- **Retrieval**: 1-5 minutes (slow)
- **Use**: Year-end backups, regulatory archives

### Lifecycle Policies

Automatically move to cheaper tiers over time:

```
Day 0: Upload document
  ↓ Stored in S3 Standard ($0.023/GB)
Day 30: Auto-transition
  ↓ Move to S3 IA ($0.0125/GB)
Day 90: Auto-transition
  ↓ Move to Glacier ($0.004/GB)
Day 365: Auto-delete (optional)
```

**Configuration**:

```json
{
  "Rules": [
    {
      "Id": "Archive old documents",
      "Filter": {"Prefix": "documents/"},
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    }
  ]
}
```

## Bucket Policies: Access Control

Control who can access what.

### Public Read (Static Website)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::pdf-platform-2024/*"
    }
  ]
}
```

Everyone can download (read), but not upload.

### Admin Only

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/admin-user"
      },
      "Action": ["s3:*"],
      "Resource": "arn:aws:s3:::pdf-platform-2024/*"
    }
  ]
}
```

Only admin-user can access bucket.

### API with IAM Role

APIs use presigned URLs or IAM roles (safer than hardcoding credentials):

```python
# EC2 instance has IAM role with S3 access
import boto3

s3 = boto3.client('s3')

# Credentials come from EC2 role, not .env file
s3.put_object(
    Bucket='pdf-platform-2024',
    Key=f'documents/{doc_id}.pdf',
    Body=file_content
)
```

No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY needed!

## Presigned URLs

Like MinIO: generate temporary, signed URLs for downloads:

```python
import boto3
from datetime import timedelta

s3 = boto3.client('s3')

# Generate presigned URL (valid for 1 hour)
url = s3.generate_presigned_url(
    'get_object',
    Params={
        'Bucket': 'pdf-platform-2024',
        'Key': 'documents/user-123/report.pdf'
    },
    ExpiresIn=3600  # 1 hour in seconds
)

# Returns: https://pdf-platform-2024.s3.amazonaws.com/...?X-Amz-Algorithm=... (very long)

# Send to user
return {"download_url": url}
```

User clicks link, downloads file. Link expires in 1 hour.

## Static Website Hosting

Host React build directly from S3:

### Step 1: Upload Files

```bash
# Build React
npm run build

# Upload to S3
aws s3 sync ./build s3://pdf-platform-2024/website --delete

# Deletes old versions of changed files
```

### Step 2: Enable Static Website

```bash
# Create index document policy
aws s3 website s3://pdf-platform-2024 \
  --index-document index.html \
  --error-document error.html
```

### Step 3: Make Public

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::pdf-platform-2024/website/*"
    }
  ]
}
```

Website is now live at: `https://pdf-platform-2024.s3.amazonaws.com/web/index.html`

Or use CloudFront CDN for faster global distribution.

## Versioning: Protection Against Accidents

Enable versioning so you can recover deleted files:

```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket pdf-platform-2024 \
  --versioning-configuration Status=Enabled
```

Now S3 keeps all versions:

```
Delete document.pdf   →   S3 keeps old versions
                          You can restore from version-id
```

Cost: Additional storage for old versions (unless lifecycle cleanup).

## Migrating from MinIO to S3

Your MinIO code uses the S3 API. Switching is simple:

**Before (MinIO)**:
```python
from minio import Minio

client = Minio(
    "minio:9000",
    access_key="minioadmin",
    secret_key="minioadmin",
    secure=False
)

client.put_object("uploads", "file.pdf", file_stream)
```

**After (S3)**:
```python
import boto3

# Credentials come from EC2 role (no hardcoding!)
s3 = boto3.client('s3', region_name='eu-west-1')

s3.put_object(Bucket='pdf-platform-2024', Key='uploads/file.pdf', Body=file_stream)
```

That's it! Both use S3 API, mostly compatible.

## S3 Event Notifications (Trigger Lambdas)

When a file is uploaded, trigger a Lambda:

```
Upload PDF to S3
  ↓
S3 publishes event to SNS/SQS
  ↓
Lambda wakes up, processes PDF
  ↓
Convert, scan, extract text
```

(Advanced, not covered here; Module 3 of AWS Advanced covers this)

## Hands-On Lab

### Lab 3.1: Create Bucket and Upload File

```bash
# 1. Create bucket (must be globally unique)
BUCKET_NAME="pdf-platform-$(date +%s)"

aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1

# 2. Upload a file
echo "Test file content" > test.txt
aws s3 cp test.txt s3://$BUCKET_NAME/test.txt

# 3. List files
aws s3 ls s3://$BUCKET_NAME/

# 4. Download file
aws s3 cp s3://$BUCKET_NAME/test.txt downloaded.txt
cat downloaded.txt

# 5. Generate presigned URL (1 hour)
aws s3 presign s3://$BUCKET_NAME/test.txt --expires-in 3600

# Copy URL and open in browser (works without credentials!)
```

### Lab 3.2: Bucket Policy Setup

```bash
# Enable public read for a prefix
cat > policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::BUCKET_NAME/public/*"
    }
  ]
}
EOF

# Replace BUCKET_NAME
sed -i "s/BUCKET_NAME/$BUCKET_NAME/g" policy.json

# Apply policy
aws s3api put-bucket-policy \
  --bucket $BUCKET_NAME \
  --policy file://policy.json

# Upload to public/ prefix
echo "Public content" > public.txt
aws s3 cp public.txt s3://$BUCKET_NAME/public/public.txt

# Access via browser
# https://$BUCKET_NAME.s3.amazonaws.com/public/public.txt
```

### Lab 3.3: Versioning

```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

# Upload v1
echo "Version 1" > file.txt
aws s3 cp file.txt s3://$BUCKET_NAME/file.txt

# Upload v2 (overwrites)
echo "Version 2" > file.txt
aws s3 cp file.txt s3://$BUCKET_NAME/file.txt

# List all versions
aws s3api list-object-versions --bucket $BUCKET_NAME

# Restore to v1
VERSION_ID="abcd1234"  # from list-object-versions
aws s3api get-object --bucket $BUCKET_NAME --key file.txt --version-id $VERSION_ID restored.txt

cat restored.txt  # Says "Version 1"
```

## Cheat Sheet: S3 CLI Commands

```bash
# Create bucket
aws s3api create-bucket --bucket BUCKET_NAME --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1

# Upload file
aws s3 cp file.txt s3://BUCKET_NAME/file.txt

# Upload directory
aws s3 sync ./directory s3://BUCKET_NAME/directory --delete

# List files
aws s3 ls s3://BUCKET_NAME/

# Download file
aws s3 cp s3://BUCKET_NAME/file.txt file.txt

# Generate presigned URL (1 hour)
aws s3 presign s3://BUCKET_NAME/file.txt --expires-in 3600

# Enable versioning
aws s3api put-bucket-versioning --bucket BUCKET_NAME \
  --versioning-configuration Status=Enabled

# Enable public read
aws s3api put-bucket-policy --bucket BUCKET_NAME --policy file://policy.json
```

## S3 Bucket Policy Syntax

```json
{
  "Effect": "Allow|Deny",
  "Principal": "*|user:arn",
  "Action": "s3:GetObject|s3:PutObject|s3:*",
  "Resource": "arn:aws:s3:::bucket-name/*",
  "Condition": {
    "StringEquals": {"aws:username": "admin"}
  }
}
```

## Key Takeaways

- **S3** = MinIO in the cloud, industry standard for object storage
- **Buckets**: Globally unique, contain objects (files)
- **Storage classes**: Standard (instant, $0.023/GB), IA (slower, $0.0125/GB), Glacier (archive, $0.004/GB)
- **Presigned URLs**: Temporary, signed download links (no credentials needed)
- **Bucket policies**: Control who can access what
- **Static website**: Host React build directly from S3
- **Versioning**: Keep all file versions, recover from deletion
- **Lifecycle**: Auto-transition to cheaper tiers over time
- **Migration**: MinIO → S3 is mostly config change
- **Cost**: $0.023/GB for Standard, cheaper for less frequent access

Module 4 teaches RDS — your managed PostgreSQL database.
