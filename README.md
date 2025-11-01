# 🚀 Build a Bedrock-Powered LLM API on AWS Lambda  
### with S3 Logging and Secure Function URL (Step-by-Step Guide)

This guide shows how to create a **serverless AI API** using **AWS Lambda**, **Amazon Bedrock (Nova)**, and **S3**, then expose it securely using a **Lambda Function URL**.  
It’s perfect for AWS beginners — no API Gateway needed.

---

## 🧰 Prerequisites

Before starting, make sure you have:
- ✅ An AWS account (Free Tier works)
- ✅ Amazon Bedrock access (enable `amazon.nova-micro-v1:0` or `anthropic.claude-3-haiku`)
- ✅ Basic IAM permissions to create Lambda, IAM roles, and S3 buckets

**Region used in this tutorial:** `us-east-1`

---

## Step 1: Create an S3 Bucket for Artifacts

We’ll use this bucket to store every LLM request/response.

1. Go to **S3 → Create bucket**
2. Name it like: `llm-demo-yourname-us-east-1`
3. Region: **US East (N. Virginia)**
4. Keep defaults:
    - Block all public access
    - Versioning: Optional
5. Click **Create bucket**

---

## Step 2: Create an IAM Policy for Bedrock + S3 Access

We’ll let Lambda talk to both Bedrock and S3.

1. Navigate to **IAM → Policies → Create policy → JSON**
2. Paste this JSON:

```json
{
"Version": "2012-10-17",
"Statement": [
 {
   "Sid": "BedrockInvoke",
   "Effect": "Allow",
   "Action": [
     "bedrock:InvokeModel",
     "bedrock:InvokeModelWithResponseStream"
   ],
   "Resource": "*"
 },
 {
   "Sid": "S3WriteArtifacts",
   "Effect": "Allow",
   "Action": ["s3:PutObject"],
   "Resource": "arn:aws:s3:::llm-demo-yourname-us-east-1/*"
 }
]
}
```

3. Name it: `llm-lambda-bedrock-s3-policy`

4. Click **Create policy**

## Step 3: Create a Lambda Execution Role

1. Go to IAM → Roles → Create role

2. Trusted entity type: AWS Service

3. Use case: Lambda

4. Click Next → Permissions

5. Attach:
    * AWSLambdaBasicExecutionRole
    * llm-lambda-bedrock-s3-policy

6. Name it: `llm-lambda-role`
   
7. Click ** Create role**

Step 4: Create the Lambda Function

Navigate to Lambda → Create function

Choose:

Name: llm-handler

Runtime: Python 3.12

Architecture: x86_64

Permissions: Choose existing role → llm-lambda-role

Click Create function

Update configuration:

Go to Configuration → General configuration → Edit

Memory: 256 MB

Timeout: 1–3 minutes

Save
   

