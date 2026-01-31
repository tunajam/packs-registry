# S3 Patterns

Comprehensive guide for AWS S3 operations, security, and performance optimization.

## When to Apply

- Designing bucket policies and access control
- Implementing presigned URLs for secure uploads/downloads
- Optimizing upload performance with multipart
- Configuring lifecycle rules for cost optimization
- Setting up cross-region replication
- Troubleshooting access issues

## Bucket Policies

### Public Read (Static Website)
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-bucket/*"
        }
    ]
}
```

### CloudFront Origin Access Control
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontOAC",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-bucket/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD632BHDS5"
                }
            }
        }
    ]
}
```

### Cross-Account Access
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CrossAccountAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::111122223333:root"
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::my-bucket/shared/*"
        }
    ]
}
```

### Deny Unencrypted Uploads
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUnencryptedUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::my-bucket/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "AES256"
                }
            }
        }
    ]
}
```

### Restrict to VPC Endpoint
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "RestrictToVPCEndpoint",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ],
            "Condition": {
                "StringNotEquals": {
                    "aws:sourceVpce": "vpce-1234567890abcdef0"
                }
            }
        }
    ]
}
```

## Presigned URLs

### Generate Presigned Download URL
```python
import boto3
from botocore.config import Config

s3_client = boto3.client('s3', config=Config(signature_version='s3v4'))

def generate_download_url(bucket, key, expiration=3600):
    """Generate presigned URL for downloading an object"""
    return s3_client.generate_presigned_url(
        'get_object',
        Params={
            'Bucket': bucket,
            'Key': key
        },
        ExpiresIn=expiration
    )

# Usage
url = generate_download_url('my-bucket', 'files/document.pdf')
```

### Generate Presigned Upload URL
```python
def generate_upload_url(bucket, key, content_type, expiration=3600):
    """Generate presigned URL for uploading an object"""
    return s3_client.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': bucket,
            'Key': key,
            'ContentType': content_type
        },
        ExpiresIn=expiration
    )

# Client-side upload
url = generate_upload_url('my-bucket', 'uploads/image.jpg', 'image/jpeg')
# Client uses: PUT <url> with Content-Type: image/jpeg
```

### Presigned POST (Form Upload)
```python
def generate_presigned_post(bucket, key_prefix, max_size_mb=10, expiration=3600):
    """Generate presigned POST for browser form uploads"""
    return s3_client.generate_presigned_post(
        Bucket=bucket,
        Key=f"{key_prefix}/${{filename}}",  # ${filename} replaced by client
        Fields={
            'acl': 'private',
            'Content-Type': ''  # Allow any content type
        },
        Conditions=[
            {'acl': 'private'},
            ['starts-with', '$Content-Type', ''],
            ['starts-with', '$key', key_prefix],
            ['content-length-range', 1, max_size_mb * 1024 * 1024]
        ],
        ExpiresIn=expiration
    )

# Returns: {'url': '...', 'fields': {...}}
# Client builds form with fields and POSTs file
```

## Multipart Uploads

### Basic Multipart Upload
```python
import os
import threading
from concurrent.futures import ThreadPoolExecutor

def multipart_upload(bucket, key, file_path, part_size_mb=50):
    """Upload large file using multipart upload"""
    
    # Initiate upload
    response = s3_client.create_multipart_upload(
        Bucket=bucket,
        Key=key
    )
    upload_id = response['UploadId']
    
    try:
        parts = []
        file_size = os.path.getsize(file_path)
        part_size = part_size_mb * 1024 * 1024
        
        with open(file_path, 'rb') as f:
            part_number = 1
            while True:
                data = f.read(part_size)
                if not data:
                    break
                
                # Upload part
                response = s3_client.upload_part(
                    Bucket=bucket,
                    Key=key,
                    UploadId=upload_id,
                    PartNumber=part_number,
                    Body=data
                )
                
                parts.append({
                    'PartNumber': part_number,
                    'ETag': response['ETag']
                })
                part_number += 1
        
        # Complete upload
        s3_client.complete_multipart_upload(
            Bucket=bucket,
            Key=key,
            UploadId=upload_id,
            MultipartUpload={'Parts': parts}
        )
        
    except Exception as e:
        # Abort on failure
        s3_client.abort_multipart_upload(
            Bucket=bucket,
            Key=key,
            UploadId=upload_id
        )
        raise e
```

### Parallel Multipart Upload
```python
def parallel_multipart_upload(bucket, key, file_path, part_size_mb=50, max_workers=10):
    """Upload parts in parallel for faster uploads"""
    
    response = s3_client.create_multipart_upload(Bucket=bucket, Key=key)
    upload_id = response['UploadId']
    
    file_size = os.path.getsize(file_path)
    part_size = part_size_mb * 1024 * 1024
    num_parts = (file_size + part_size - 1) // part_size
    
    def upload_part(part_info):
        part_number, start_byte = part_info
        with open(file_path, 'rb') as f:
            f.seek(start_byte)
            data = f.read(part_size)
            
        response = s3_client.upload_part(
            Bucket=bucket,
            Key=key,
            UploadId=upload_id,
            PartNumber=part_number,
            Body=data
        )
        return {'PartNumber': part_number, 'ETag': response['ETag']}
    
    part_info = [(i + 1, i * part_size) for i in range(num_parts)]
    
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        parts = list(executor.map(upload_part, part_info))
    
    parts.sort(key=lambda x: x['PartNumber'])
    
    s3_client.complete_multipart_upload(
        Bucket=bucket,
        Key=key,
        UploadId=upload_id,
        MultipartUpload={'Parts': parts}
    )
```

### Presigned Multipart (Client-Side Chunked Upload)
```python
def create_multipart_upload_urls(bucket, key, num_parts, expiration=3600):
    """Generate presigned URLs for client-side multipart upload"""
    
    # Initiate upload
    response = s3_client.create_multipart_upload(Bucket=bucket, Key=key)
    upload_id = response['UploadId']
    
    # Generate presigned URL for each part
    part_urls = []
    for part_number in range(1, num_parts + 1):
        url = s3_client.generate_presigned_url(
            'upload_part',
            Params={
                'Bucket': bucket,
                'Key': key,
                'UploadId': upload_id,
                'PartNumber': part_number
            },
            ExpiresIn=expiration
        )
        part_urls.append({'partNumber': part_number, 'url': url})
    
    return {
        'uploadId': upload_id,
        'key': key,
        'parts': part_urls
    }

def complete_multipart_upload(bucket, key, upload_id, parts):
    """Complete multipart upload after client uploads all parts"""
    s3_client.complete_multipart_upload(
        Bucket=bucket,
        Key=key,
        UploadId=upload_id,
        MultipartUpload={'Parts': parts}  # [{PartNumber, ETag}, ...]
    )
```

## Lifecycle Rules

### Transition and Expiration
```json
{
    "Rules": [
        {
            "ID": "ArchiveOldData",
            "Status": "Enabled",
            "Filter": {
                "Prefix": "logs/"
            },
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "STANDARD_IA"
                },
                {
                    "Days": 90,
                    "StorageClass": "GLACIER"
                },
                {
                    "Days": 365,
                    "StorageClass": "DEEP_ARCHIVE"
                }
            ],
            "Expiration": {
                "Days": 2555
            }
        }
    ]
}
```

### Clean Up Incomplete Multipart Uploads
```json
{
    "Rules": [
        {
            "ID": "AbortIncompleteMultipartUploads",
            "Status": "Enabled",
            "Filter": {},
            "AbortIncompleteMultipartUpload": {
                "DaysAfterInitiation": 7
            }
        }
    ]
}
```

### Expire Old Versions
```json
{
    "Rules": [
        {
            "ID": "ExpireOldVersions",
            "Status": "Enabled",
            "Filter": {},
            "NoncurrentVersionTransitions": [
                {
                    "NoncurrentDays": 30,
                    "StorageClass": "GLACIER"
                }
            ],
            "NoncurrentVersionExpiration": {
                "NoncurrentDays": 90
            }
        }
    ]
}
```

## Performance Optimization

### Prefix Partitioning
```python
# Bad: Sequential prefixes cause hot partitions
# uploads/2024-01-01/file1.jpg
# uploads/2024-01-01/file2.jpg

# Good: Add random prefix for distribution
import hashlib

def get_distributed_key(original_key):
    """Add hash prefix for better S3 partitioning"""
    hash_prefix = hashlib.md5(original_key.encode()).hexdigest()[:4]
    return f"{hash_prefix}/{original_key}"

# Result: a1b2/uploads/2024-01-01/file1.jpg
```

### Transfer Acceleration
```python
# Enable on bucket (console or CLI)
# aws s3api put-bucket-accelerate-configuration \
#     --bucket my-bucket \
#     --accelerate-configuration Status=Enabled

# Use accelerate endpoint
s3_client = boto3.client(
    's3',
    config=Config(s3={'use_accelerate_endpoint': True})
)
```

### Byte-Range Fetches
```python
def download_range(bucket, key, start_byte, end_byte):
    """Download specific byte range of an object"""
    response = s3_client.get_object(
        Bucket=bucket,
        Key=key,
        Range=f'bytes={start_byte}-{end_byte}'
    )
    return response['Body'].read()

# Parallel download of large file
def parallel_download(bucket, key, file_path, chunk_size_mb=50, max_workers=10):
    head = s3_client.head_object(Bucket=bucket, Key=key)
    file_size = head['ContentLength']
    chunk_size = chunk_size_mb * 1024 * 1024
    
    ranges = []
    for start in range(0, file_size, chunk_size):
        end = min(start + chunk_size - 1, file_size - 1)
        ranges.append((start, end))
    
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        chunks = list(executor.map(
            lambda r: download_range(bucket, key, r[0], r[1]),
            ranges
        ))
    
    with open(file_path, 'wb') as f:
        for chunk in chunks:
            f.write(chunk)
```

## Event Notifications

### S3 to Lambda
```json
{
    "LambdaFunctionConfigurations": [
        {
            "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:ProcessUpload",
            "Events": ["s3:ObjectCreated:*"],
            "Filter": {
                "Key": {
                    "FilterRules": [
                        {
                            "Name": "prefix",
                            "Value": "uploads/"
                        },
                        {
                            "Name": "suffix",
                            "Value": ".jpg"
                        }
                    ]
                }
            }
        }
    ]
}
```

### Lambda Handler
```python
def lambda_handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        event_name = record['eventName']
        
        print(f"Event: {event_name}, Bucket: {bucket}, Key: {key}")
        
        # Process the object
        response = s3_client.get_object(Bucket=bucket, Key=key)
        content = response['Body'].read()
        
        # Do something with content...
```

## CORS Configuration

```json
{
    "CORSRules": [
        {
            "AllowedOrigins": ["https://example.com"],
            "AllowedMethods": ["GET", "PUT", "POST"],
            "AllowedHeaders": ["*"],
            "ExposeHeaders": ["ETag"],
            "MaxAgeSeconds": 3600
        }
    ]
}
```

## Common Gotchas

### Eventual Consistency
- S3 provides strong read-after-write consistency (since Dec 2020)
- List operations may show stale results briefly after bulk operations

### 403 Errors
1. Check bucket policy AND IAM policy (both must allow)
2. Check for explicit denies
3. Verify region in client configuration
4. Check for bucket owner enforcement (ACLs disabled)

### Costs
- PUT, COPY, POST, LIST requests cost more than GET
- Data transfer out costs money (use CloudFront)
- Lifecycle transitions have minimum storage duration charges

### Large Objects
- Objects > 5GB MUST use multipart upload
- Single PUT limit: 5GB
- Multipart maximum: 5TB
- Recommended: Use multipart for objects > 100MB

## CLI Commands

```bash
# Sync directory
aws s3 sync ./local-dir s3://bucket/prefix/ --delete

# Copy with metadata
aws s3 cp file.txt s3://bucket/file.txt \
    --metadata '{"key":"value"}' \
    --content-type "text/plain"

# Presigned URL
aws s3 presign s3://bucket/key --expires-in 3600

# List with human-readable sizes
aws s3 ls s3://bucket/ --recursive --human-readable --summarize

# Remove old multipart uploads
aws s3api list-multipart-uploads --bucket my-bucket
aws s3api abort-multipart-upload --bucket my-bucket --key key --upload-id id
```

## References

- [S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/)
- [S3 Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
- [Bucket Policy Examples](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-bucket-policies.html)
- [S3 Pricing](https://aws.amazon.com/s3/pricing/)
