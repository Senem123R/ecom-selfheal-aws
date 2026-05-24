# 🛒 AWS AI-native — Self-Healing E-Commerce DevOps Platform

> An AI-native autonomous DevOps platform that detects, diagnoses, and fixes production incidents **without human involvement** — built on AWS serverless infrastructure using the OODA loop pattern.

---

## 🧠 What is this?

This platform watches your e-commerce microservices 24/7. When something breaks, it:

1. **OBSERVES** — scans CloudWatch logs every 5 minutes for errors
2. **ANALYZES** — sends the incident to an AI (LLaMA 3.2) for root cause analysis
3. **DECIDES** — AI recommends a fix action (restart / scale_up / alert_only)
4. **ACTS** — system executes the fix automatically

No human wakes up. No Slack alert. It just fixes itself.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│           E-Commerce Microservices (AWS Lambda)      │
│  Auth  │  Products  │  Cart  │  Orders  │  Payments  │
└────────────────────────┬────────────────────────────┘
                         │ logs
                         ▼
              ┌─────────────────────┐
              │   OBSERVE Pillar    │
              │  (CloudWatch Logs)  │
              │  runs every 5 mins  │
              └──────────┬──────────┘
                         │ incident found → DynamoDB
                         ▼
              ┌─────────────────────┐
              │   ANALYZE Pillar    │
              │  (OpenRouter AI)    │
              │  root cause analysis│
              └──────────┬──────────┘
                         │ diagnosis
                         ▼
              ┌─────────────────────┐
              │   DECIDE + ACT      │
              │  auto-remediation   │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │  React Dashboard    │
              │  live monitoring    │
              └─────────────────────┘
```

---

## 🗂️ Project Structure

```
ecom-selfheal-aws/
├── services/
│   ├── auth/           ← register + login (DynamoDB: ecom-users)
│   ├── payments/       ← payment processing with fake failures
│   ├── products/       ← product catalog
│   ├── cart/           ← shopping cart
│   ├── orders/         ← order management
│   └── tracking/       ← shipment tracking
├── observe/            ← CloudWatch log scanner (runs every 5 mins)
├── analyze/            ← OpenRouter AI root cause analysis
├── decide/             ← fix decision engine
├── act/                ← auto-remediation executor
├── frontend/           ← React live monitoring dashboard
└── template.yaml       ← AWS SAM Infrastructure as Code
```

---

## ⚙️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Compute | AWS Lambda (Python 3.11) |
| Database | AWS DynamoDB |
| Messaging | AWS SNS |
| Monitoring | AWS CloudWatch Logs |
| API | AWS API Gateway |
| Infrastructure | AWS SAM (template.yaml) |
| AI | OpenRouter — LLaMA 3.2 3B |
| Frontend | React + Vite |
| Hosting | AWS S3 Static Website |

---

## 🚀 How to Deploy

### Prerequisites

```bash
# Install AWS CLI
https://aws.amazon.com/cli/

# Install AWS SAM
https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html

# Configure AWS credentials
aws configure
```

### Deploy everything with one command

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/ecom-selfheal-aws.git
cd ecom-selfheal-aws

# Deploy all Lambda functions, DynamoDB tables, SNS topic
sam build
sam deploy --guided
```

### Deploy React dashboard

```bash
cd frontend
npm install
npm run build

# Upload to S3
aws s3 mb s3://ecom-selfheal-dashboard-YOUR_NAME --region us-east-1
aws s3 website s3://ecom-selfheal-dashboard-YOUR_NAME \
  --index-document index.html \
  --error-document index.html
aws s3 sync dist/ s3://ecom-selfheal-dashboard-YOUR_NAME --acl public-read
```

---

## 🔁 How the Self-Healing Loop Works

### Step 1 — Payments service injects failures (intentional)
```python
# 5% of requests simulate gateway timeout
if random.random() < 0.05:
    logger.error("CRITICAL: Payment gateway timeout — connection refused")

# 10% simulate declined payment
if random.random() < 0.10:
    logger.warning("Payment declined by issuing bank")
```

These errors go to **CloudWatch Logs** automatically.

### Step 2 — OBSERVE scans CloudWatch every 5 minutes
```python
resp = logs_client.filter_log_events(
    logGroupName='/aws/lambda/ecom-payments-service',
    filterPattern='ERROR'
)
```

Classifies severity:
- 10+ errors → `CRITICAL`
- 3–9 errors → `HIGH`
- 1–2 errors → `MEDIUM`

### Step 3 — Incident saved to DynamoDB + published to SNS
```python
table.put_item(Item=incident)          # saved for dashboard
sns_client.publish(Message=incident)   # triggers AI analysis
```

### Step 4 — ANALYZE Lambda wakes up via SNS
```python
# AI receives the incident and returns:
{
  "root_cause": "Payment gateway connection pool exhausted",
  "fix_action": "restart",
  "confidence": 0.85,
  "explanation": "High error rate suggests gateway overload"
}
```

### Step 5 — React dashboard shows everything live
Polls every 60 seconds. Color-coded by severity.

---

## 📊 Incident Severity Levels

| Level | Error Count | Color |
|-------|------------|-------|
| CRITICAL | 10+ | 🔴 Red |
| HIGH | 3–9 | 🟠 Orange |
| MEDIUM | 1–2 | 🟡 Yellow |
| LOW | 0 | 🟢 Green |

---

## 🛡️ Key Design Decisions

**Why SHA-256 for passwords?**
Passwords are never stored in plain text. SHA-256 hashes them before saving to DynamoDB. Even if the database is breached, passwords cannot be recovered.

**Why UUID for IDs?**
Sequential IDs (1, 2, 3) are predictable — attackers can guess them. UUID4 generates random IDs like `f47ac10b-58cc-4372-a567-0e02b2c3d479` that are impossible to guess.

**Why DecimalEncoder?**
DynamoDB returns numbers as Python `Decimal` type. Standard `json.dumps()` crashes on Decimal. The custom encoder converts `Decimal('5')` → `int(5)` and `Decimal('9.99')` → `float(9.99)` before sending to the React dashboard.

**Why environment variables for config?**
Table names, SNS topic ARNs, and API keys are stored in AWS environment variables — never hardcoded in source code. This keeps secrets off GitHub.

**Why connections outside the handler?**
```python
# Created once — reused across warm Lambda invocations
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(TABLE_NAME)

def lambda_handler(event, context):
    # uses existing connection — faster!
```

---

## 🧪 Test the Self-Healing Loop

```bash
# Hit payments 20 times — some will fail by design
for i in {1..20}; do
  curl -s -X POST https://YOUR_API/Prod/payments/pay \
    -H "Content-Type: application/json" \
    -d '{"amount":49.99}' &
done

# Manually trigger OBSERVE right now
curl -X POST https://YOUR_API/Prod/observe

# Check DynamoDB for incidents
aws dynamodb scan --table-name ecom-incidents \
  --query 'Items[*].[service_name.S,severity.S,title.S]' \
  --output table

# Watch AI analysis in real time
aws logs tail /aws/lambda/ecom-analyze --follow
```

---

## 🗺️ AWS Services Used (All Free Tier)

| Service | What it does in this project |
|---------|------------------------------|
| Lambda | Runs all microservices and OODA pillars |
| DynamoDB | Stores users, payments, incidents |
| CloudWatch | Stores all logs — OBSERVE scans these |
| SNS | Event bus between OBSERVE and ANALYZE |
| API Gateway | Exposes Lambda functions as HTTP APIs |
| EventBridge | Triggers OBSERVE every 5 minutes |
| S3 | Hosts the React dashboard |
| SAM | Deploys everything with one command |

---

## 📝 Environment Variables

| Variable | Used in | Purpose |
|----------|---------|---------|
| `TABLE_NAME` | observe | DynamoDB incidents table name |
| `SNS_TOPIC_ARN` | observe | SNS topic address for publishing |
| `OPENROUTER_KEY` | analyze | API key for LLaMA 3.2 AI |
| `REGION` | all | AWS region (us-east-1) |

---

## 👤 Author

Built as an AI-Native DevOps project demonstrating agentic AI patterns on AWS serverless infrastructure.

**Tech Stack:** Python · AWS Lambda · DynamoDB · CloudWatch · SNS · API Gateway · AWS SAM · React · OpenRouter AI (LLaMA 3.2) · Vite
