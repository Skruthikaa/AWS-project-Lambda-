## Serverless Image Resizer & Logger with S3 + Lambda

Built on Amazon Web Services using Amazon S3, AWS Lambda, and Amazon CloudWatch

---

# 1️ Objective

Build a fully serverless image processing system that:

* Accepts image uploads (.jpg/.jpeg/.png)
* Automatically creates a 150x150 thumbnail
* Stores thumbnails in a separate bucket
* Logs processing details
* Handles invalid file types safely

This lab demonstrates **event-driven architecture** and **serverless automation**.

---

# 2️ Architecture Overview

```
User Upload →
S3 Upload Bucket →
S3 Event Trigger →
Lambda Function →
Thumbnail saved to Thumbnails Bucket →
Logs stored in CloudWatch
```

---

# 3️ Step-by-Step Implementation

---

##  Step 1: Create S3 Buckets

Create two buckets:

### 1. marketing-uploads-bucket

### 2. marketing-thumbnails-bucket

### Configuration for BOTH buckets:

*  Enable Versioning
*  Block all public access
*  No static website hosting

---

## 2.1 Create Role
* Go to IAM → Roles
* Click Create Role
* Select:
    * Trusted entity: AWS service
    * Use case: Lambda

## 2.2 Attach Policies
 * Attach:
    * AmazonS3FullAccess (lab purpose)
    * CloudWatchLogsFullAccess
    * AWSLambdaBasicExecution
    * AmazonDynamoDBFullAccess
 
---

##  Step 3: Create Lambda Function

* Runtime: Python 3.x
* Architecture: x86_64
* Attach IAM role created above
* Timeout: 30 seconds
* Memory: 256 MB (recommended)

---

##  Step 4: Add Environment Variables (Best Practice)

Under Configuration → Environment Variables:

| Key              | Value                       |
| ---------------- | --------------------------- |
| UPLOAD_BUCKET    | marketing-uploads-bucket    |
| THUMBNAIL_BUCKET | marketing-thumbnails-bucket |

---

# 5 Lambda Function Code
     * Paste this code inside lambda_function.py
```python
import json
import boto3
import os
from PIL import Image
import io
import urllib.parse

s3 = boto3.client('s3')

UPLOAD_BUCKET = os.environ['UPLOAD_BUCKET']
THUMBNAIL_BUCKET = os.environ['THUMBNAIL_BUCKET']

def lambda_handler(event, context):
    
    print("Event received:", event)
    
    for record in event['Records']:
        
        bucket = record['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])
        
        if not key.lower().endswith(('.jpg', '.jpeg', '.png')):
            print(f"WARNING: Invalid file type uploaded: {key}")
            return
        
        response = s3.get_object(Bucket=bucket, Key=key)
        file_content = response['Body'].read()
        file_size = response['ContentLength']
        
        print(f"Original File: {key}")
        print(f"Original File Size: {file_size} bytes")
        
        image = Image.open(io.BytesIO(file_content))
        image.thumbnail((150, 150))
        
        buffer = io.BytesIO()
        image.save(buffer, image.format)
        buffer.seek(0)
        
        thumbnail_key = key.split('.')[0] + "-thumbnail." + key.split('.')[-1]
        
        s3.put_object(
            Bucket=THUMBNAIL_BUCKET,
            Key=thumbnail_key,
            Body=buffer,
            ContentType=response['ContentType']
        )
        
        print(f"Thumbnail created successfully: {thumbnail_key}")
        
    return {
        'statusCode': 200,
        'body': json.dumps('Thumbnail creation completed')
    }
```
##  Step 6: Add Pillow Library (Lambda Layer Recommended)

* Create Lambda Layer (Pillow)
    Lambda does NOT include Pillow by default.
    We must package it manually.

* 6.1 Open AWS CloudShell
    Go to top navigation → Click CloudShell

* 6.2 Create Layer Directory
   * mkdir pillow-layer
   * cd pillow-layer
   * mkdir python

* 6.3 Install Pillow Inside python Folder
* Run:
docker run --rm \
-v "$PWD":/var/task \
--entrypoint "" \
public.ecr.aws/lambda/python:3.11 \
pip install pillow -t python/

* 6.4 Zip the Layer
    * zip -r pillow-layer.zip python

* 6.5 Download Zip (Optional)
    * You can download it locally if needed.
  
* 6.6 Create Layer in Lambda
    * Go to Lambda → Layers
    * Click Create layer
    * Name:pillow-layer
    * Upload pillow-layer.zip
    * Compatible runtime:Python 3.11

* 6.7 Attach Layer to Lambda
    * Open Lambda → prod-image-resizer
    * Scroll to Layers
    * Click Add layer
    * Select:Custom layers-pillow-layer-Add
---

##  Step 7: Configure S3 Trigger

Go to:

* Upload bucket → Properties → Event Notifications
* Event Type: ObjectCreated (PUT)
* Destination: Lambda Function
* Select your Lambda

---

# 8 Testing

###  Test 1: Upload image1.jpg

Expected:

* Thumbnail created in thumbnails bucket
* CloudWatch logs show filename + size + success

###  Test 2: Upload image2.png

Expected:

* Thumbnail created
* Logs show success

###  Test 3: Upload file.txt

Expected:

* No thumbnail created
* CloudWatch log shows WARNING

---

# 9 Verification

### Check:

* Thumbnails bucket contains:

  * image1-thumbnail.jpg
  * image2-thumbnail.png

### Check CloudWatch Logs:

* Correct filename
* File size
* Success message
* Warning for invalid file

---

# 10 Short Explanation (Submission Section)

### Why use Lambda for image processing?

Lambda enables automatic, scalable, and cost-effective image processing without managing servers. It runs only when triggered by an S3 upload event, making it ideal for event-driven workloads. This reduces operational overhead and ensures high availability.

### What happens if multiple images are uploaded simultaneously?

Lambda automatically scales by creating multiple concurrent execution environments. Each image upload triggers a separate Lambda invocation, allowing parallel thumbnail processing without performance impact.

### How does S3 event trigger Lambda?

When a new object is uploaded, S3 generates an ObjectCreated event. This event notification is configured to invoke the Lambda function, passing metadata (bucket name, object key) to the function for processing.

---



