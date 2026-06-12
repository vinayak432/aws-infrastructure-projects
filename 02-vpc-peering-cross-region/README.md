# AWS Inter-Account VPC Peering — Cross-Region Architecture

> **Scenario:** Connect two AWS accounts across two separate regions (Tokyo & Sydney) using VPC Peering, enabling private IP communication between EC2 instances without traversing the public internet.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ACCOUNT A (Tokyo)                            │
│                     ap-northeast-1                                  │
│                                                                     │
│   ┌─────────────────────────────────────┐                           │
│   │         VPC-1 (12.0.0.0/16)         │                           │
│   │                                     │                           │
│   │  ┌──────────────┐ ┌──────────────┐  │                           │
│   │  │ Public Subnet│ │Private Subnet│  │                           │
│   │  │              │ │              │  │                           │
│   │  │  EC2 Instance│ │  EC2 Instance│  │                           │
│   │  │  (Public IP) │ │ (Private IP) │  │                           │
│   │  └──────────────┘ └──────────────┘  │                           │
│   │                                     │                           │
│   │  Internet Gateway ✓                 │                           │
│   │  Route Table: 0.0.0.0/0 → IGW      │                           │
│   │  Route Table: 13.0.0.0/16 → PCX    │◄──────────────────────┐   │
│   └─────────────────────────────────────┘                       │   │
└─────────────────────────────────────────────────────────────────┼───┘
                                                                  │
                              VPC Peering Connection              │
                              (Cross-Account + Cross-Region)      │
                              Status: ACTIVE ✓                    │
                                                                  │
┌─────────────────────────────────────────────────────────────────┼───┐
│                        ACCOUNT B (Sydney)                        │   │
│                      ap-southeast-2                             │   │
│                                                                  │   │
│   ┌─────────────────────────────────────┐                        │   │
│   │         VPC-2 (13.0.0.0/16)         │                        │   │
│   │                                     │                        │   │
│   │  ┌──────────────┐ ┌──────────────┐  │                        │   │
│   │  │ Public Subnet│ │Private Subnet│  │                        │   │
│   │  │  (Auto-assign│ │              │  │                        │   │
│   │  │   Public IP) │ │  EC2 Instance│  │                        │   │
│   │  └──────────────┘ └──────────────┘  │                        │   │
│   │                                     │                        │   │
│   │  Internet Gateway ✓                 │                        │   │
│   │  Route Table: 0.0.0.0/0 → IGW      │                        │   │
│   │  Route Table: 12.0.0.0/16 → PCX   │────────────────────────┘   │
│   └─────────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────┘

PCX = VPC Peering Connection
IGW = Internet Gateway
```

---

## Configuration Details

### VPC Setup

| Parameter | VPC-1 (Tokyo) | VPC-2 (Sydney) |
|---|---|---|
| **Account** | Account A | Account B |
| **Region** | ap-northeast-1 | ap-southeast-2 |
| **CIDR Block** | 12.0.0.0/16 | 13.0.0.0/16 |
| **Subnets** | 1 Public + 1 Private | 1 Public + 1 Private |
| **Internet Gateway** | Attached ✓ | Attached ✓ |
| **Auto-assign Public IP** | Enabled on Public Subnet | Enabled on Public Subnet |

---

## Step-by-Step Implementation

### Step 1: Create VPC in Tokyo (Account A)

```
CIDR: 12.0.0.0/16
Region: ap-northeast-1 (Tokyo)
```

- Navigate to VPC Dashboard → Create VPC
- Set CIDR block: `12.0.0.0/16`
- Create 2 subnets: 1 public, 1 private
- Enable auto-assign public IP on the public subnet

### Step 2: Create Internet Gateway — Tokyo

```
Action: Create IGW → Attach to VPC-1
```

- Create Internet Gateway
- Attach to VPC-1 (12.0.0.0/16)

### Step 3: Configure Route Tables — Tokyo

```
Public Route Table:
  Destination: 0.0.0.0/0   → Target: IGW
  Destination: 13.0.0.0/16 → Target: PCX (added after peering)

Associate public subnet with this route table.
```

### Step 4: Create VPC in Sydney (Account B)

```
CIDR: 13.0.0.0/16
Region: ap-southeast-2 (Sydney)
```

- Repeat same steps as Tokyo
- Different CIDR block — must NOT overlap

### Step 5: Create VPC Peering Connection

```
Requester:  Account A | VPC-1 | ap-northeast-1 (Tokyo)
Accepter:   Account B | VPC-2 | ap-southeast-2 (Sydney)
Type:       Cross-Account + Cross-Region
```

**From Account A (Tokyo):**
- VPC Dashboard → Peering Connections → Create Peering Connection
- Select: Different account
- Enter Account B's Account ID
- Enter VPC-2's VPC ID
- Select region: ap-southeast-2

**From Account B (Sydney):**
- VPC Dashboard → Peering Connections
- Find the pending request
- Action: Accept Request ✓

> ⚠️ **Common mistake:** Forgetting to accept from the destination account. The peering connection stays in "Pending Acceptance" until explicitly accepted.

### Step 6: Update Route Tables on BOTH Sides

> ⚠️ **Critical:** Route tables must be updated on BOTH VPCs. One-sided routes = no traffic flow.

**Tokyo Route Table — add:**
```
Destination: 13.0.0.0/16
Target:      pcx-xxxxxxxxxx (peering connection ID)
```

**Sydney Route Table — add:**
```
Destination: 12.0.0.0/16
Target:      pcx-xxxxxxxxxx (peering connection ID)
```

### Step 7: Configure Security Groups

> Apply on BOTH EC2 instances in both VPCs.

**Inbound Rules:**
```
Type        Protocol    Port    Source
ICMP        ICMP        All     12.0.0.0/16  (Tokyo CIDR)
ICMP        ICMP        All     13.0.0.0/16  (Sydney CIDR)
SSH         TCP         22      12.0.0.0/16
SSH         TCP         22      13.0.0.0/16
```

### Step 8: Configure Network ACLs

> ⚠️ **NACLs are stateless** — unlike Security Groups, you must explicitly allow both inbound AND outbound traffic.

**Allow on BOTH sides (Inbound + Outbound):**
```
Rule    Type    Protocol    Port Range    Source/Dest
100     ICMP    ICMP        All           Peer VPC CIDR
110     SSH     TCP         22            Peer VPC CIDR
*       ALL     ALL         ALL           DENY (default)
```

### Step 9: Launch EC2 Instances

```
Tokyo EC2:  Place in peered subnet | Use private IP for testing
Sydney EC2: Place in peered subnet | Use private IP for testing
```

- Use same key pair OR set up passwordless SSH between instances
- For private subnet instances: no EC2 Instance Connect — use bastion or SSM

### Step 10: Verify Connectivity

**Test 1 — Ping (ICMP):**
```bash
# From Tokyo EC2, ping Sydney EC2 private IP
ping 13.x.x.x

# Expected output:
# PING 13.x.x.x: 56 data bytes
# 64 bytes from 13.x.x.x: icmp_seq=0 ttl=255 time=170ms ✓
```

**Test 2 — SSH:**
```bash
# From Tokyo EC2, SSH into Sydney EC2 using private IP
ssh -i your-key.pem ec2-user@13.x.x.x

# Successful passwordless SSH = peering working correctly ✓
```

---

## Key Concepts

### VPC Peering is Non-Transitive

```
VPC-A ↔ VPC-B  ✓ (direct peering)
VPC-B ↔ VPC-C  ✓ (direct peering)
VPC-A ↔ VPC-C  ✗ (NO — non-transitive)

For A↔C communication, a separate peering connection is required.
At scale, use AWS Transit Gateway instead.
```

### Security Groups vs NACLs

| Feature | Security Groups | NACLs |
|---|---|---|
| **State** | Stateful | Stateless |
| **Level** | Instance level | Subnet level |
| **Rules** | Allow only | Allow + Deny |
| **Return traffic** | Auto-allowed | Must explicitly allow |
| **For peering** | Allow peer CIDR | Allow inbound + outbound |

---

## Common Mistakes & Fixes

| Mistake | Symptom | Fix |
|---|---|---|
| Did not accept peering request | Status: Pending Acceptance | Accept from destination account |
| Route table updated on one side only | One-way traffic or no traffic | Add routes on BOTH VPCs |
| NACL not updated | Ping/SSH fails even with SG rules correct | Allow ICMP + SSH both inbound & outbound on both NACLs |
| Overlapping CIDR blocks | Peering creation fails | Use non-overlapping CIDRs |
| Wrong VPC ID in peering request | Peering fails | Double-check VPC ID from correct account/region |

---

## Production Use Cases

```
1. Multi-account strategy
   Dev Account ↔ Shared Services Account (logging, monitoring, secrets)

2. Cross-environment private database access
   App VPC → peering → RDS VPC (no public internet exposure)

3. Centralised security tooling
   All accounts peer to Security Account for GuardDuty, CloudTrail aggregation

4. Microservices across isolated VPCs
   Service-A VPC ↔ Service-B VPC for private API communication
```

> **At scale:** Consider **AWS Transit Gateway** over VPC Peering.
> Transit Gateway supports transitive routing, simplifies management,
> and works better for 5+ VPCs.

---

## Tech Stack

![AWS](https://img.shields.io/badge/AWS-VPC-FF9900?style=flat&logo=amazonaws)
![Region](https://img.shields.io/badge/Regions-Tokyo%20%7C%20Sydney-232F3E?style=flat&logo=amazonaws)
![Type](https://img.shields.io/badge/Type-Cross--Account%20%7C%20Cross--Region-4A9EE8?style=flat)

---

## Author

**Author:** Vinayaka K — DevOps Engineer 
[LinkedIn](https://www.linkedin.com/in/vinayak-k) | [GitHub](https://github.com/vinayak432/aws-infrastructure-projects.git)

