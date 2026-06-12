# 05 — Application Load Balancer — Multi-AZ Setup

> Deploy 3 EC2 web servers across 3 availability zones with automated Apache deployment via User Data, then front them with an Application Load Balancer for high availability.

---

## Architecture

```
                    Internet
                       │
              ┌────────▼────────┐
              │  Application    │
              │  Load Balancer  │
              │  (ALB)          │
              └────┬──────┬─────┘
                   │      │
         ┌─────────┘      └──────────┐
         │                           │
    ┌────▼─────┐  ┌────────┐  ┌─────▼────┐
    │  EC2-1   │  │  EC2-2 │  │  EC2-3   │
    │  AZ-1    │  │  AZ-2  │  │  AZ-3    │
    │ Apache   │  │ Apache │  │  Apache  │
    │ Server 1 │  │Server 2│  │  Server 3│
    └──────────┘  └────────┘  └──────────┘

Target Group → all 3 EC2 instances → Health checks: HTTP /
```

---

## What Was Built

- 3 EC2 instances across 3 different Availability Zones
- Apache web server auto-deployed on each via EC2 User Data
- Each server shows unique content (SERVER1 / SERVER2 / SERVER3)
- Application Load Balancer distributing traffic across all 3
- Target Group with HTTP health checks
- Security Groups allowing HTTP (80) and HTTPS (443)

---

## EC2 User Data Script (Auto-deploys Apache)

```bash
#!/bin/bash
# 1. Update system
yum update -y

# 2. Install Apache Web Server
yum install -y httpd

# 3. Start Apache and enable on boot
systemctl start httpd
systemctl enable httpd

# 4. Create a sample web page
cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>Amazon Linux Web Server</title>
    <style>
        body {
            background-color: #0f172a;
            color: #38bdf8;
            font-family: Arial, sans-serif;
            text-align: center;
            padding-top: 100px;
        }
        h1 { font-size: 48px; }
        p  { font-size: 20px; }
    </style>
</head>
<body>
    <h1>🚀 WELCOME TO THE WORLD OF DEVOPS SERVER1!</h1>
    <p>WE ARE RUNNING APP SERVER</p>
    <p>Deployed automatically using EC2 User Data</p>
</body>
</html>
EOF

# 5. Set correct permissions
chmod 644 /var/www/html/index.html

# 6. Restart Apache
systemctl restart httpd
```
> Paste this in EC2 → Advanced Details → User Data during launch. Change SERVER1 to SERVER2/SERVER3 for each instance.

---

## Step-by-Step Implementation

### Step 1: Security Group Setup
```
Inbound Rules:
  HTTP  (80)  → 0.0.0.0/0
  HTTPS (443) → 0.0.0.0/0
  SSH   (22)  → Your IP
```

### Step 2: Launch 3 EC2 Instances
```
Instance 1: AZ-1 (e.g. ap-southeast-1a) — User Data: SERVER1
Instance 2: AZ-2 (e.g. ap-southeast-1b) — User Data: SERVER2
Instance 3: AZ-3 (e.g. ap-southeast-1c) — User Data: SERVER3

AMI: Amazon Linux 2
Attach Security Group created in Step 1
```

### Step 3: Create Target Group
```
Target type: Instances
Protocol: HTTP
Port: 80
Health check path: /
Healthy threshold: 2
Register all 3 EC2 instances
```

### Step 4: Create Application Load Balancer
```
Type: Application Load Balancer
Scheme: Internet-facing
Listeners: HTTP:80
AZs: Select all 3 AZs where EC2s are deployed
Security Group: Same SG as EC2s
Forward to: Target group created in Step 3
```

### Step 5: Verify
```
Copy ALB DNS name → paste in browser
Refresh multiple times → SERVER1 / SERVER2 / SERVER3 rotate
All instances show healthy in Target Group → ✅
```

---

## Key Concepts

### ALB vs NLB vs CLB
| Feature | ALB | NLB | CLB |
|---|---|---|---|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP) | 4 & 7 |
| Path-based routing | ✅ | ❌ | ❌ |
| Host-based routing | ✅ | ❌ | ❌ |
| WebSockets | ✅ | ✅ | ❌ |
| Use case | Web apps, microservices | High-perf TCP | Legacy |

### Health Check Flow
```
ALB → sends HTTP GET / to each target every 30s
Target responds 200 OK → Healthy ✅
Target fails 2 checks → Unhealthy → removed from rotation
Target recovers → re-added automatically
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Instances in same AZ | Use different AZs for true HA |
| Target group health check fails | Verify Apache is running: `systemctl status httpd` |
| User Data script didn't run | Check /var/log/cloud-init-output.log |
| ALB returns 502 | Security group on EC2 not allowing port 80 from ALB |

---

## Production Relevance
```
This is the baseline for:
→ EKS Service type: LoadBalancer (AWS ALB Ingress Controller)
→ Blue-green deployments: switch target group weights
→ Canary deployments: weighted target groups (90/10 split)
→ Auto Scaling Groups: register ASG with target group
```

---

**Author:** Vinayaka K — DevOps Engineer 
[LinkedIn](https://www.linkedin.com/in/vinayak-k) | [GitHub](https://github.com/vinayak432/aws-infrastructure-projects.git)

