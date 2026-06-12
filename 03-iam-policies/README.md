# 03 — AWS IAM: Custom Policies, Roles & Least-Privilege Access

> Build custom IAM policies with granular permissions, attach them to users and roles, assign roles to EC2 instances, and verify that only permitted actions are allowed.

---

## What Was Built

```
Scenario 1:
  Custom Policy → EC2 read-only + VPC read-only + EBS delete
  Attached to IAM Users → verified per-user permissions

Scenario 2:
  Custom Policy → EC2 start/stop + CloudWatch read-only
  Attached to IAM Role → Role assigned to EC2 instance
  Verified via AWS CLI from inside the instance:
    ✅ Start → allowed
    ✅ Stop  → allowed
    ❌ Terminate → blocked
    ✅ CloudWatch metrics → allowed
```

---

## Scenario 1: Custom Policy — EC2 + VPC Read + EBS Delete

### Policy JSON
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2ReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "VPCReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeRouteTables",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EBSDeleteOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

### Steps
```
IAM → Policies → Create Policy → JSON tab → paste above
Policy name: EC2VPCRead-EBSDelete

Attach to User 1 (virat):
  IAM → Users → virat → Add permissions → Attach policy

Attach to User 2 (Atharva) — read-only only:
  Create a separate policy without EBS delete
  Attach to Atharva
```

### Verification
```
User virat:
  ✅ Can describe EC2 instances
  ✅ Can describe VPCs
  ✅ Can delete EBS volumes
  ❌ Cannot launch/terminate instances

User Atharva:
  ✅ Can describe EC2 instances
  ✅ Can describe VPCs
  ❌ Cannot delete EBS volumes
```

---

## Scenario 2: IAM Role on EC2 — Start/Stop + CloudWatch

### Policy JSON
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2StartStop",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchReadOnly",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "cloudwatch:DescribeAlarms"
      ],
      "Resource": "*"
    }
  ]
}
```

### Steps

#### Create IAM Role
```
IAM → Roles → Create Role
Trusted entity: AWS Service → EC2
Attach policy: EC2StartStop-CloudWatchRead (created above)
Role name: ec2-start-stop-role
```

#### Attach Role to EC2 Instance
```
EC2 → Instances → Select instance
Actions → Security → Modify IAM Role
Select: ec2-start-stop-role → Update
```

#### Test from Inside EC2 (AWS CLI)
```bash
# Install AWS CLI if not present
sudo apt install -y awscli

# Start an instance
aws ec2 start-instances --instance-ids i-0xxxxxxxxxxxxxxxxx
# ✅ Response: StateChange: stopped → pending

# Stop an instance
aws ec2 stop-instances --instance-ids i-0xxxxxxxxxxxxxxxxx
# ✅ Response: StateChange: running → stopping

# Attempt to terminate (should be blocked)
aws ec2 terminate-instances --instance-ids i-0xxxxxxxxxxxxxxxxx
# ❌ Response: UnauthorizedOperation ← policy working correctly

# List CloudWatch metrics
aws cloudwatch list-metrics --namespace AWS/EC2
# ✅ Returns metric list
```

---

## Key Concepts

### IAM Policy Evaluation Logic
```
Default: DENY everything
Explicit Allow: grants permission
Explicit Deny: overrides any Allow

Evaluation order:
1. Explicit Deny → DENY (final)
2. Explicit Allow → ALLOW
3. No match → DENY (implicit)
```

### IAM Role vs IAM User for EC2
```
❌ IAM User + Access Keys on EC2:
   Keys stored in ~/.aws/credentials
   Risk: keys leaked if instance compromised
   Manual rotation required

✅ IAM Role on EC2:
   Temporary credentials via Instance Metadata Service
   Auto-rotated by AWS
   No keys stored on instance
   AWS CLI uses them automatically
```

### Least-Privilege Principle
```
Grant only the permissions required to perform the task.
Never use AdministratorAccess or * on Action + Resource together.

❌ Too broad:
   "Action": "*", "Resource": "*"

✅ Least-privilege:
   "Action": ["ec2:StartInstances", "ec2:StopInstances"]
   "Resource": "arn:aws:ec2:region:account:instance/i-xxx"
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Policy attached but actions still denied | Wait 30–60s for IAM propagation |
| Role not appearing in Modify IAM Role | Ensure role trust policy allows `ec2.amazonaws.com` |
| CLI returns credentials error | Verify role is attached: `aws sts get-caller-identity` |
| User can do more than expected | Check for other policies attached to user or group |

---

## Production Relevance
```
→ IRSA (IAM Roles for Service Accounts) in EKS:
  Kubernetes pods assume IAM roles using OIDC
  Same principle as EC2 role — no keys in pods

→ Terraform execution role:
  Terraform needs IAM permissions to create resources
  Use role on EC2/runner, not user access keys in CI

→ CodePipeline/GitHub Actions OIDC:
  CI/CD assumes IAM role via OIDC federation
  No long-lived credentials in CI environment

→ SCPs (Service Control Policies):
  Organisation-level guardrails — even admins can't bypass
```

---

**Author:** Vinayaka K — DevOps Engineer | [LinkedIn](https://www.linkedin.com/in/vinayak-k)
