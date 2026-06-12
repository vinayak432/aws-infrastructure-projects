# 04 — CloudWatch Monitoring, CPU Alarms & SNS Email Alerts

> Configure automated CPU monitoring on EC2 with CloudWatch alarms and SNS email notifications — plus EC2-to-S3 integration using AWS CLI and IAM roles.

---

## What Was Built

```
┌─────────────────────────────────────────────────┐
│                  EC2 Instance                   │
│                                                 │
│  stress --cpu 2 --timeout 600  (load generator) │
│            ↓ CPU spikes > threshold             │
│                                                 │
│         CloudWatch Alarm (CPU > 40%)            │
│                    ↓                            │
│              SNS Topic                          │
│                    ↓                            │
│         📧 Email Notification Sent              │
└─────────────────────────────────────────────────┘

Plus:
EC2 Instance (IAM Role attached)
     ↓ AWS CLI
S3 Bucket — list, upload, download objects
```

---

## Assignment 1: EC2 CPU Monitoring & Email Alerts

### Step 1: Launch EC2 & Install Stress Tool
```bash
sudo apt update
sudo apt install stress -y
stress --version
```

### Step 2: Create IAM Role with CloudWatch Access
```
Policy: CloudWatchFullAccess (or custom read/write)
Role type: EC2
Attach to: EC2 instance via Actions → Security → Modify IAM Role
```

### Step 3: Create SNS Topic & Subscription
```
SNS → Create Topic → Standard → Name: cpu-alert-topic
Create Subscription:
  Protocol: Email
  Endpoint: your-email@example.com
```
> ⚠️ Confirm the subscription from your email inbox before proceeding.

### Step 4: Create CloudWatch Alarm
```
Metric:    EC2 → Per-Instance Metrics → CPUUtilization
Condition: Greater than 40% for 1 consecutive period (1 min)
Action:    Send notification to SNS topic (cpu-alert-topic)
```

### Step 5: Trigger & Test
```bash
# Generate artificial CPU load
stress --cpu 2 --timeout 600

# This spikes CPU to ~100% for 10 minutes
# CloudWatch detects threshold breach
# SNS sends email notification within ~3 minutes
```

**Result:** Email alert received confirming alarm triggered ✅

---

## Assignment 2: EC2 to S3 Integration via AWS CLI

### Step 1: Create S3 Bucket
```
Region: ap-northeast-1 (Tokyo)
Bucket name: ec2-s3-integrate
```

### Step 2: Create & Attach IAM Role
```
Policy: AmazonS3FullAccess
Attach to EC2 instance via Modify IAM Role
```

### Step 3: Install AWS CLI on EC2
```bash
sudo apt update
sudo apt install awscli -y
```

### Step 4: S3 Operations via CLI
```bash
# List all buckets
aws s3 ls

# List objects in specific bucket
aws s3 ls s3://ec2-s3-integrate/
aws s3 ls s3://bucket-name/folder-name/

# Upload file to S3
aws s3 cp vk.txt s3://ec2-s3-integrate/

# Download file from S3
aws s3 cp s3://ec2-s3-integrate/vk.txt ./downloaded-vk.txt
```

---

## Key Concepts

### CloudWatch Alarm States
```
OK        → Metric within threshold
ALARM     → Metric breached threshold → action triggered
INSUFFICIENT_DATA → Not enough data points yet
```

### IAM Role vs Access Keys for EC2
```
❌ Don't do this:
   aws configure (hardcoded Access Key + Secret)

✅ Do this:
   Attach IAM Role to EC2
   AWS CLI automatically uses instance metadata credentials
   No keys stored on the instance
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| SNS subscription not confirmed | Check email and click confirm link |
| Alarm stays in INSUFFICIENT_DATA | Wait 2–3 minutes after stress test starts |
| AWS CLI shows "Access Denied" | Check IAM role is attached and has correct policy |
| CPU alarm never triggers | Lower threshold or increase stress duration |

---

## Production Relevance
```
This maps directly to production observability:
→ CloudWatch → Prometheus equivalent on AWS
→ SNS → Alertmanager / PagerDuty equivalent
→ IAM Role on EC2 → IRSA on EKS pods
→ CPU alarms → Foundation for Auto Scaling policies
```

---

**Author:** Vinayaka K — DevOps Engineer 
[LinkedIn](https://www.linkedin.com/in/vinayak-k) | [GitHub](https://github.com/vinayak432/aws-infrastructure-projects.git)
