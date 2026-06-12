# 08 — AWS RDS MySQL — Managed Database Setup

> Provision an Amazon RDS MySQL instance, configure security group access, connect from EC2, and run SQL operations against a production-style employees table.

---

## Architecture

```
EC2 Instance (App/Bastion)
        │
        │ MySQL client (port 3306)
        │
        ▼
RDS MySQL Instance
(Private subnet — not publicly accessible)
        │
        └── Database: db1
            └── Table: employees
```

---

## What Was Built

- RDS MySQL instance provisioned on AWS
- Security group inbound rule: port 3306 from EC2 security group
- EC2 connected to RDS using endpoint, username, and password
- Database and table created
- Records inserted and queried

---

## Step-by-Step Implementation

### Step 1: Create RDS MySQL Instance
```
Engine: MySQL
Template: Free tier
DB instance identifier: your-db-name
Master username: admin
Master password: your-password
Instance class: db.t3.micro
Storage: 20 GiB (GP2)
Public access: No (recommended)
VPC: Same as EC2
```

### Step 2: Configure Security Group
```
RDS Security Group — Inbound Rule:
  Type: MySQL/Aurora
  Protocol: TCP
  Port: 3306
  Source: EC2 Security Group ID (not IP — use SG reference)
```
> ✅ Using SG reference instead of IP allows all EC2s in that SG to access RDS — more flexible and secure.

### Step 3: Get RDS Endpoint
```
RDS → Databases → Your DB → Connectivity & Security
Copy the Endpoint: your-db.xxxxxxxxx.region.rds.amazonaws.com
```

### Step 4: Connect from EC2
```bash
# Install MySQL client on EC2
sudo apt update
sudo apt install -y mysql-client

# Connect to RDS
mysql -h your-db.xxxxxxxxx.region.rds.amazonaws.com \
      -u admin \
      -p
# Enter password when prompted
```

### Step 5: Create Database & Table
```sql
-- Create database
CREATE DATABASE db1;
USE db1;

-- Create employees table
CREATE TABLE employees (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(45) NOT NULL,
    last_name  VARCHAR(45) NOT NULL,
    department VARCHAR(100),
    salary     DECIMAL(10,2),
    hire_date  DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Step 6: Insert & Query Data
```sql
-- Insert a record
INSERT INTO employees (first_name, last_name, department, salary, hire_date)
VALUES ('Jane', 'Doe', 'Engineering', 75000.00, '2026-01-15');

-- View table structure
DESCRIBE employees;

-- Query all records
SELECT * FROM employees;
```

---

## Key Concepts

### RDS vs Self-Managed MySQL on EC2
| Feature | RDS | EC2 MySQL |
|---|---|---|
| Automated backups | ✅ | Manual |
| Multi-AZ failover | ✅ | Manual setup |
| Patching | AWS managed | Manual |
| Read replicas | ✅ | Manual |
| Cost | Higher | Lower |
| Control | Less | Full |

### Security Best Practices
```
✅ No public access — connect only from within VPC
✅ Security Group scoped to EC2 SG (not 0.0.0.0/0)
✅ Strong master password
✅ Store credentials in AWS Secrets Manager (production)
✅ Enable encryption at rest
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Connection timeout | Check SG inbound rule for port 3306 from correct source |
| Public access needed | Use EC2 as bastion or enable temporarily for testing only |
| Wrong endpoint | Use RDS endpoint, not EC2 IP |
| Access denied | Verify username/password and DB user grants |

---

## Production Relevance
```
→ Terraform RDS module: automates this entire setup
→ AWS Secrets Manager: store DB credentials, rotate automatically
→ RDS Proxy: connection pooling for Lambda/EKS workloads
→ Read replicas: scale read-heavy applications
→ Multi-AZ: automatic failover for production databases
→ Parameter groups: tune MySQL config without SSH access
```

---

**Author:** Vinayaka K — DevOps Engineer | [LinkedIn](https://www.linkedin.com/in/vinayak-k)
