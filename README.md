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

## Step 4: Create the Lambda Function

1. Navigate to Lambda → Create function

2. Choose:

   * Name: `llm-handler`

   * Runtime: Python 3.12

   * Architecture: x86_64

   * Permissions: Choose existing role → llm-lambda-role

3. Click Create function

## Update configuration:

* Go to Configuration → General configuration → Edit

    * Memory: 256 MB

    * Timeout: 1–3 minutes

    * Save

## Step 5: Add Environment Variables

Under **Configuration → Environment variables**, click Edit and add:

| Key      | Value                       |
| -------- | --------------------------- |
| BUCKET   | llm-demo-yourname-us-east-1 |
| MODEL_ID | amazon.nova-micro-v1:0      |

   
## Step 6: Paste the Lambda Code

Replace the entire code in the Code tab with this:

```python
import json, os
from datetime import datetime
import boto3

region = os.environ.get("AWS_REGION", "us-east-1")
bedrock = boto3.client("bedrock-runtime", region_name=region)
s3 = boto3.client("s3", region_name=region)

BUCKET = os.environ.get("BUCKET", "llm-demo-yourname-us-east-1")
MODEL_ID = os.environ.get("MODEL_ID", "amazon.nova-micro-v1:0")

def lambda_handler(event, context):
    body = event.get("body") if isinstance(event, dict) else None
    if isinstance(body, str):
        try:
            body = json.loads(body)
        except Exception:
            body = {}
    elif not isinstance(body, dict):
        body = event if isinstance(event, dict) else {}

    prompt = body.get("prompt") or "Explain what AWS Lambda does in one sentence."

    req = {
        "messages": [{"role": "user", "content": [{"text": prompt}]}],
        "inferenceConfig": {"maxTokens": 150, "temperature": 0.2}
    }

    resp = bedrock.invoke_model(modelId=MODEL_ID, body=json.dumps(req))
    payload = json.loads(resp["body"].read())

    content = payload.get("output", {}).get("message", {}).get("content", [])
    output_text = "".join([c.get("text", "") for c in content])

    key = f"artifacts/http-run-{datetime.now().strftime('%Y%m%d-%H%M%S')}.json"
    s3.put_object(
        Bucket=BUCKET,
        Key=key,
        Body=json.dumps({"model": MODEL_ID, "prompt": prompt, "response": output_text}),
        ContentType="application/json"
    )

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json", "Access-Control-Allow-Origin": "*"},
        "body": json.dumps({"model": MODEL_ID, "s3_key": key, "output": output_text})
    }
```
Click **Deploy**.


## Step 7: Test in the Console

1. Click **Test → Configure test event**

2. Event JSON:

   ```json
   {"prompt": "How are you?"}
    ```

3. Run the test → You should see `statusCode: 200`

4. Check your **S3 bucket** → you’ll find a JSON file under `/artifacts/`


## Step 8: Create a Function URL (Public Test)

1. Go to **Configuration → Function URL → Create**

2. Auth type: `NONE`

3. Additional settings → Enable **CORS**

4. Allow origin: `*`

5. Click **Save**

## Test with cURL:

```bash
curl -X POST "https://<your-function-id>.lambda-url.us-east-1.on.aws/" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Write a haiku about cloud computing"}'
```
✅ You should get a nice JSON response with your model output.


## Step 9: Secure It with IAM

Now let’s lock down the endpoint using AWS_IAM authentication.

1. Edit Function URL → **Change Auth type to AWS_IAM → Save**

2. Add permission via CLI:

```bash
   aws lambda add-permission \
  --function-name llm-handler \
  --statement-id AllowFunctionUrlInvokeFromMyUser \
  --action lambda:InvokeFunctionUrl \
  --principal arn:aws:iam::<ACCOUNT_ID>:user/<YOUR_USERNAME> \
  --function-url-auth-type AWS_IAM
```

## 🔒 Step 9 — Secure It with AWS IAM

Now let’s lock down the Lambda Function URL so that only **IAM-signed requests** can call it.

### 1️⃣ Enable IAM Authentication
1. In your Lambda function console, open  
   **Configuration → Function URL**  
2. Click **Edit**, change **Auth type** to `AWS_IAM`, then **Save** ✅

---

### 2️⃣ Add a Resource-Based Permission (CLI)

Open **AWS CloudShell** or your terminal and run:

```bash
aws lambda add-permission \
  --function-name llm-handler \
  --statement-id AllowFunctionUrlInvokeFromMyUser \
  --action lambda:InvokeFunctionUrl \
  --principal arn:aws:iam::<ACCOUNT_ID>:user/<YOUR_USERNAME> \
  --function-url-auth-type AWS_IAM
```   
  
### 🔍 3️⃣ Verify the Policy

After adding the permission, confirm that your Lambda function now accepts only **IAM-signed requests**.

Run the following command in **CloudShell** or your terminal:

```bash
aws lambda get-policy --function-name llm-handler
```
✅ Expected Output

You should see a policy statement similar to this:

```json
{
  "Action": "lambda:InvokeFunctionUrl",
  "Principal": "arn:aws:iam::<ACCOUNT_ID>:user/<YOUR_USERNAME>",
  "Condition": {
    "StringEquals": {
      "lambda:FunctionUrlAuthType": "AWS_IAM"
    }
  }
}
```
✅ What This Means

The Action lambda:InvokeFunctionUrl allows invoking the Function URL.

The Principal is your IAM user, meaning only that identity can call the URL.

The Condition enforces that the call must use AWS_IAM authentication.

💡 Result: your Lambda Function URL is now securely locked down to accept only signed IAM requests.
