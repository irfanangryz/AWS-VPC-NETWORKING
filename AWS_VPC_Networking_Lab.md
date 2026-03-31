# AWS VPC Networking — Infrastructure Setup Guide

> A practical walkthrough for provisioning a custom VPC with public/private subnet architecture, internet routing, EC2 deployment, and NAT Gateway configuration on AWS.

---

## Part 1 — VPC, Subnets, Internet Gateway & EC2

### Step 1 — Create Custom VPC

| Field | Value |
|-------|-------|
| Name | `week1-vpc` |
| CIDR Block | `10.10.0.0/16` |

---

### Step 2 — Create Subnets

Both subnets in the **same Availability Zone** inside the VPC:

| Subnet | CIDR |
|--------|------|
| Public Subnet | `10.10.1.0/24` |
| Private Subnet | `10.10.2.0/24` |

---

### Step 3 — Create & Attach Internet Gateway

- **Name:** `week1-igw`
- **Attach to:** `week1-vpc`

---

### Step 4 — Configure Route Table

- **Name:** `week1-public-rt`
- **Route:** `0.0.0.0/0` → IGW
- **Associate:** Public subnet **only**
- ⚠️ Private subnet must **NOT** have an IGW route

---

### Step 5 — Launch EC2 Instance

| Setting | Value |
|---------|-------|
| AMI | Amazon Linux |
| Instance Type | `t2.micro` |
| Subnet | `public-subnet` |
| Auto-assign Public IP | **ENABLE** |
| Security Group | SSH (port 22) → My IP |

---

### Step 6 — Deploy Nginx Web Server

Connect via SSH and run:

```bash
sudo yum update -y
sudo yum install nginx -y
sudo systemctl start nginx
```

Verify in browser:

```
http://<public-ip>
```

> ✅ Nginx default page loading confirms successful deployment.

---

## Part 2 — Private Subnet & NAT Gateway

### Concept Overview

The private subnet is intentionally isolated:
- ❌ No route to Internet Gateway
- ❌ No public IP assigned

Outbound internet access is handled through a **NAT Gateway**.

> **NAT Gateway** provides outbound-only internet connectivity — inbound connections from the internet are not possible.

### Traffic Flow

```
Private EC2 → Route Table → NAT Gateway → IGW → Internet
```

---

### Step 1 — Launch EC2 in Private Subnet

| Setting | Value |
|---------|-------|
| VPC | `week1-vpc` |
| Subnet | `private-subnet` |
| Auto-assign Public IP | **DISABLE** |
| Security Group | SSH from `10.10.1.0/24` (public subnet CIDR) |

> Direct SSH from a local machine is not possible — this is by design.

---

### Step 2 — Allocate Elastic IP

```
EC2 → Elastic IPs → Allocate Elastic IP
```

Reserve the Elastic IP for attachment to the NAT Gateway.

---

### Step 3 — Create NAT Gateway

```
VPC → NAT Gateways → Create
```

| Setting | Value |
|---------|-------|
| Name | `week1-nat` |
| Subnet | `public-subnet` ⚠️ NAT Gateway must reside in the public subnet |
| Elastic IP | Attach allocated Elastic IP |

> ⏳ Wait until status shows **Available** before proceeding.

---

### Step 4 — Configure Private Route Table

1. Create a new route table:
   - **Name:** `week1-private-rt`
   - **VPC:** `week1-vpc`

2. Add route:
   - `0.0.0.0/0` → **NAT Gateway**

3. Associate with `private-subnet`

---

### Step 5 — Validate NAT Gateway Connectivity

Direct SSH to the private EC2 is not possible. Use the public EC2 as a **bastion host**:

**1.** SSH into the public EC2:
```bash
ssh -i <key.pem> ec2-user@<public-ec2-public-ip>
```

**2.** From the public EC2, SSH into the private EC2 using its private IP:
```bash
ssh ec2-user@<private-ec2-private-ip>
```

**3.** Test outbound internet access:
```bash
sudo yum update -y
```

> ✅ A successful update confirms the NAT Gateway is routing traffic correctly.

---

## Architecture Overview

```
Internet
    │
   IGW (week1-igw)
    │
   VPC (10.10.0.0/16)
    ├── Public Subnet (10.10.1.0/24)
    │       ├── EC2 Instance  (public IP — internet accessible)
    │       └── NAT Gateway   (week1-nat)
    │
    └── Private Subnet (10.10.2.0/24)
            └── EC2 Instance  (no public IP — outbound via NAT only)
```
