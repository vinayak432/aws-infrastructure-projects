# 07 — AWS EBS & EFS Storage

> Explore the difference between block storage (EBS) and shared file storage (EFS) through hands-on implementation — detaching and re-attaching EBS volumes and mounting EFS across multiple EC2 instances simultaneously.

---

## EBS vs EFS — Quick Reference

| Feature | EBS | EFS |
|---|---|---|
| Type | Block storage | Managed NFS (file storage) |
| Attachment | One EC2 at a time (GP3) | Multiple EC2s simultaneously |
| Availability | Single AZ | Multi-AZ |
| Protocol | Block (like a hard drive) | NFS4 |
| Use case | OS volumes, databases | Shared config, logs, media |
| Persistence | Manual management | Fully managed |

---

## Part 1: EBS — Elastic Block Store

### What Was Built
```
EC2 Instance-1 (AZ-1)     EC2 Instance-2 (AZ-1)
      │                           │
      │ ← EBS Volume (GP3) ──────►│
      │   vol-0f93c1499cfaee352   │
      │                           │
  Files created              Files visible
  on Instance-1         after volume re-attach ✅
```

### Step-by-Step

#### Step 1: Create 2 EC2 Instances
```
Both in same AZ (EBS is AZ-scoped)
Create test files on Instance-1:
  mkdir /data
  echo "EBS test file" > /data/test.txt
```

#### Step 2: Create EBS Volume
```
Volume type: GP3
Size: 8 GiB (or as needed)
AZ: same as EC2 instances
Volume ID: vol-0f93c1499cfaee352
```

#### Step 3: Detach from Instance-1
```
EC2 → Volumes → Select volume
Actions → Detach Volume
Wait for state: Available
Stop Instance-1 before detaching (recommended for root volumes)
```

#### Step 4: Attach to Instance-2
```
EC2 → Volumes → Select volume
Actions → Attach Volume
Select Instance-2
Device: /dev/xvdf (or auto-assigned)
```

#### Step 5: Mount & Verify on Instance-2
```bash
# List block devices
lsblk

# Mount the volume
sudo mount /dev/xvdf /mnt/ebs

# Verify files from Instance-1 are visible
ls /mnt/ebs/data/
cat /mnt/ebs/data/test.txt  # ✅ File visible
```

> ⚠️ io2 Multi-Attach (attach to multiple EC2s simultaneously) is NOT available on free tier. GP3 = one EC2 at a time only.

---

## Part 2: EFS — Elastic File System

### What Was Built
```
EC2 Instance-1 (AZ-1)     EC2 Instance-2 (AZ-2)
        │                          │
        └──────────┬───────────────┘
                   │
           EFS File System
           fs-0cea63c2a109200a4
           (NFS4 — shared mount)
                   │
        /mnt/efs on both instances
        Files written by one = visible on both ✅
```

### Step-by-Step

#### Step 1: Create Security Groups
```
EC2 Security Group:
  Outbound: NFS (2049) → EFS Security Group

EFS Security Group:
  Inbound: NFS (2049) → EC2 Security Group
```

#### Step 2: Create EC2 Instances
```
Use the EC2 Security Group created above
Can be in different AZs — EFS is multi-AZ
```

#### Step 3: Create EFS File System
```
EFS → Create file system
VPC: same as EC2 instances
Mount targets: one per AZ (auto-created)
EFS ID: fs-0cea63c2a109200a4
ARN: arn:aws:elasticfilesystem:ap-northeast-1:936045463151:file-system/fs-0cea63c2a109200a4
```

#### Step 4: Install EFS Utils on Both Instances
```bash
sudo apt-get update
sudo apt-get install -y amazon-efs-utils

# Alternative (NFS client)
sudo apt update
sudo apt install -y nfs-common
```

#### Step 5: Create Mount Points & Mount EFS
```bash
# Create mount directory
sudo mkdir /mnt/efs

# Mount using NFS4
sudo mount -t nfs4 -o nfsvers=4.1 \
  fs-0cea63c2a109200a4.efs.ap-northeast-1.amazonaws.com:/ \
  /mnt/efs
```

#### Step 6: Persist Mount (fstab)
```bash
sudo nano /etc/fstab

# Add this line:
fs-0cea63c2a109200a4.efs.ap-northeast-1.amazonaws.com:/ /mnt/efs nfs4 _netdev,nfsvers=4.1 0 0

# Reload and remount
sudo systemctl daemon-reload
sudo mount -a
```

#### Step 7: Test Shared Storage
```bash
# On Instance-1: create a file
echo "written from instance-1" > /mnt/efs/shared-test.txt

# On Instance-2: verify immediately visible
cat /mnt/efs/shared-test.txt
# Output: written from instance-1 ✅
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| EBS attach fails (different AZ) | EBS is AZ-scoped — instance and volume must be in same AZ |
| EFS mount fails | Check security group NFS port 2049 is open between EC2 and EFS |
| fstab entry breaks boot | Add `_netdev` option — ensures network is up before mount |
| io2 multi-attach unavailable | Expected on free tier — use EFS for shared storage instead |

---

## Production Relevance
```
EBS:
→ Kubernetes PersistentVolumes (gp3 StorageClass on EKS)
→ RDS database storage
→ EC2 root volumes

EFS:
→ Kubernetes ReadWriteMany PersistentVolumes
→ Shared config files across multiple pods/instances
→ Jenkins build workspaces shared across agents
→ Content management systems with shared media storage
```

---

**Author:** Vinayaka K — DevOps Engineer | [LinkedIn](https://www.linkedin.com/in/vinayak-k)
