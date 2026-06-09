# 01 — AWS VPC, Subnets & Internet Gateway

> Build a custom VPC from scratch with public and private subnets, route tables, and internet connectivity — the foundation of every AWS architecture.

---

## What Was Built

```
VPC: 172.20.48.0/22
│
├── Public Subnets (x3) ──► Internet Gateway ──► Internet
│       └── EC2 Instance (public-facing)
│
└── Private Subnets (x3)
        └── EC2 Instance (no direct internet access)
```

---

## Configuration Details

| Parameter | Value |
|---|---|
| **VPC CIDR** | 172.20.48.0/22 |
| **Total Subnets** | 6 (3 public + 3 private) |
| **Internet Gateway** | 1 — attached to VPC |
| **Route Tables** | Public RT + Private RT |

---

## Step-by-Step Implementation

### Step 1: Create Custom VPC
```
CIDR Block: 172.20.48.0/22
```
- VPC Dashboard → Create VPC
- Enter CIDR: `172.20.48.0/22`
- Tenancy: Default

### Step 2: Create 6 Subnets
```
Public Subnets:
  172.20.48.0/24  — AZ-1
  172.20.49.0/24  — AZ-2
  172.20.50.0/24  — AZ-3

Private Subnets:
  172.20.51.0/26  — AZ-1
  172.20.51.64/26 — AZ-2
  172.20.51.128/26— AZ-3
```
- Enable **Auto-assign Public IPv4** on public subnets only

### Step 3: Create & Attach Internet Gateway
```
Create IGW → Attach to VPC (172.20.48.0/22)
```

### Step 4: Configure Route Tables

**Public Route Table:**
```
Destination     Target
0.0.0.0/0   →  IGW (internet gateway)
172.20.48.0/22 → local
```
Associate all 3 public subnets.

**Private Route Table:**
```
Destination       Target
172.20.48.0/22 → local
```
Associate all 3 private subnets. No internet route.

### Step 5: Launch EC2 Instances
- **Public EC2:** Place in public subnet → accessible via public IP
- **Private EC2:** Place in private subnet → no public IP, no direct internet

---

## Key Concepts

### Public vs Private Subnet
| Feature | Public Subnet | Private Subnet |
|---|---|---|
| Route to IGW | ✅ Yes | ❌ No |
| Public IP | Auto-assigned | Not assigned |
| Use case | Web servers, Bastion | Databases, App servers |

### CIDR Sizing Reference
```
/22 = 1,024 IPs  (VPC)
/24 = 256 IPs    (large subnet)
/26 = 64 IPs     (small subnet)
/28 = 16 IPs     (minimal subnet)
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Auto-assign public IP not enabled | Enable in subnet settings, not instance settings |
| IGW created but not attached | Attach IGW to the VPC explicitly |
| Route table not associated to subnet | Associate subnet to correct route table |
| Private instances unreachable | Expected — use Bastion host or SSM |

---

## Production Relevance
```
This pattern is the baseline for every AWS architecture:
→ EKS worker nodes: private subnets
→ RDS databases: private subnets
→ Load balancers: public subnets
→ NAT Gateway: enables private subnet outbound internet
```

---
 
**Author:** Vinayaka K — DevOps Engineer 
[LinkedIn](https://www.linkedin.com/in/vinayak-k) | [GitHub](https://github.com/vinayak432)
