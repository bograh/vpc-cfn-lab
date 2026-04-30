# Design Spec: Securely Deploying Resources in a VPC using CloudFormation

**Date:** 2026-04-30
**Author:** Bernard Ograh
**Lab:** Securely deploying resources in a VPC using cfn LAB
**Status:** Approved

---

## 1. Overview

This project provisions a highly available, secure, multi-AZ Virtual Private Cloud (VPC) on AWS using a single CloudFormation template. The architecture demonstrates fault-tolerant network design, controlled internet access, and secure EC2 instance management exclusively through AWS Systems Manager Session Manager (no SSH).

### Goals
- Deploy a production-grade VPC spanning two Availability Zones
- Host a publicly accessible Apache web tier (2 instances)
- Host a private application tier with outbound-only internet access (2 instances)
- Enforce least-privilege security groups (no inbound SSH anywhere)
- Manage all EC2 access via SSM Session Manager
- Deliver clean, well-commented Infrastructure as Code (IaC) via CloudFormation

---

## 2. AWS Region & Availability Zones

| Setting | Value |
|---|---|
| Region | `us-east-1` (N. Virginia) |
| AZ 1 | `us-east-1a` |
| AZ 2 | `us-east-1b` |

---

## 3. Networking Architecture

### 3.1 VPC & Subnet CIDR Layout

| Resource | CIDR Block | AZ | Tier |
|---|---|---|---|
| VPC | `10.0.0.0/16` | — | — |
| Public Subnet A | `10.0.1.0/24` | us-east-1a | Public |
| Public Subnet B | `10.0.2.0/24` | us-east-1b | Public |
| Private Subnet A | `10.0.3.0/24` | us-east-1a | Private |
| Private Subnet B | `10.0.4.0/24` | us-east-1b | Private |

### 3.2 Internet & Egress Components

| Component | Count | Purpose |
|---|---|---|
| Internet Gateway (IGW) | 1 | Inbound/outbound internet for public subnets |
| Elastic IP (EIP-A) | 1 | Static IP for NAT Gateway A |
| Elastic IP (EIP-B) | 1 | Static IP for NAT Gateway B |
| NAT Gateway A | 1 (us-east-1a) | Outbound-only internet for Private Subnet A |
| NAT Gateway B | 1 (us-east-1b) | Outbound-only internet for Private Subnet B |

### 3.3 Route Tables

| Route Table | Associated Subnets | Route: 0.0.0.0/0 Target |
|---|---|---|
| Public Route Table A | Public Subnet A | Internet Gateway |
| Public Route Table B | Public Subnet B | Internet Gateway |
| Private Route Table A | Private Subnet A | NAT Gateway A |
| Private Route Table B | Private Subnet B | NAT Gateway B |

**Design rationale for two NAT Gateways:**
Each private subnet routes egress traffic through its own AZ-local NAT Gateway. This eliminates:
1. Cross-AZ data transfer costs
2. A single point of failure — if one AZ's NAT Gateway fails, the other AZ's private subnet is unaffected

### 3.4 Traffic Flow Diagram

```
Internet
    │
    ▼
Internet Gateway (IGW)
    │
    ├──▶ Public Subnet A (us-east-1a) ──▶ WebServer-A
    │         │
    │         └──▶ NAT Gateway A (EIP-A)
    │                   │
    │                   ▼
    │         Private Subnet A (us-east-1a) ──▶ AppServer-A
    │
    └──▶ Public Subnet B (us-east-1b) ──▶ WebServer-B
              │
              └──▶ NAT Gateway B (EIP-B)
                        │
                        ▼
              Private Subnet B (us-east-1b) ──▶ AppServer-B
```

---

## 4. Compute

### 4.1 EC2 Instance Summary

| Instance Name | Tier | Subnet | AZ | Public IP | Purpose |
|---|---|---|---|---|---|
| `WebServer-A` | Public | Public Subnet A | us-east-1a | Yes (auto-assign) | Apache web server |
| `WebServer-B` | Public | Public Subnet B | us-east-1b | Yes (auto-assign) | Apache web server |
| `AppServer-A` | Private | Private Subnet A | us-east-1a | No | Application tier |
| `AppServer-B` | Private | Private Subnet B | us-east-1b | No | Application tier |

### 4.2 Instance Configuration

| Setting | Value |
|---|---|
| AMI | Dynamic SSM lookup: `{{resolve:ssm:/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2}}` |
| Instance Type | `t3.micro` |
| Key Pair | None (SSM Session Manager only) |
| IAM Instance Profile | `EC2SSMInstanceProfile` |

### 4.3 Apache User Data (Public Instances Only)

Applied to `WebServer-A` and `WebServer-B` via CloudFormation `UserData`:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html>
  <head><title>VPC Lab</title></head>
  <body>
    <h1>Bernard Ograh</h1>
    <h2>Securely deploying resources in a VPC using cfn LAB</h2>
  </body>
</html>
EOF
```

The private instances (`AppServer-A`, `AppServer-B`) have no User Data — they run a vanilla Amazon Linux 2 OS.

---

## 5. IAM — SSM Access

### 5.1 Resources

| Resource | Type | Purpose |
|---|---|---|
| `EC2SSMRole` | IAM Role | Assumed by EC2 instances |
| `EC2SSMInstanceProfile` | Instance Profile | Wraps the role for EC2 attachment |

### 5.2 Attached Policies

| Policy | Source |
|---|---|
| `AmazonSSMManagedInstanceCore` | AWS Managed Policy |

This policy grants EC2 instances the minimum permissions needed for SSM Session Manager (register with SSM, open sessions, write session logs).

---

## 6. Security Groups

### 6.1 WebServerSG (attached to public EC2s)

| Direction | Protocol | Port | Source / Destination | Justification |
|---|---|---|---|---|
| Inbound | TCP | 80 | `0.0.0.0/0` | HTTP access from the internet |
| Inbound | ICMP | All | `10.0.0.0/16` | Ping from within VPC for validation |
| Outbound | All | All | `0.0.0.0/0` | SSM agent, OS updates, general egress |

### 6.2 AppServerSG (attached to private EC2s)

| Direction | Protocol | Port | Source / Destination | Justification |
|---|---|---|---|---|
| Inbound | ICMP | All | `10.0.0.0/16` | Ping from within VPC for validation |
| Outbound | All | All | `0.0.0.0/0` | NAT → internet for SSM agent and OS updates |

**Security principles enforced:**
- No inbound SSH (port 22) on any security group
- Private instances have no inbound TCP access from the internet
- ICMP is restricted to VPC CIDR only (not internet-facing)
- All management performed exclusively via SSM Session Manager

---

## 7. CloudFormation Template Structure

**File:** `template.yaml` — single file, organized into clearly delineated commented sections.

```
template.yaml
├── AWSTemplateFormatVersion + Description
├── Metadata
├── Parameters
│
├── ── SECTION 1: NETWORKING ──────────────────────────
│   ├── VPC
│   ├── Internet Gateway + IGW Attachment
│   ├── Public Subnet A & B
│   ├── Private Subnet A & B
│   ├── Elastic IP A & B
│   ├── NAT Gateway A & B
│   ├── Public Route Table A & B (routes + associations)
│   └── Private Route Table A & B (routes + associations)
│
├── ── SECTION 2: IAM ─────────────────────────────────
│   ├── EC2SSMRole
│   └── EC2SSMInstanceProfile
│
├── ── SECTION 3: SECURITY GROUPS ─────────────────────
│   ├── WebServerSG
│   └── AppServerSG
│
├── ── SECTION 4: COMPUTE ─────────────────────────────
│   ├── WebServer-A (Public, us-east-1a, Apache UserData)
│   ├── WebServer-B (Public, us-east-1b, Apache UserData)
│   ├── AppServer-A (Private, us-east-1a)
│   └── AppServer-B (Private, us-east-1b)
│
└── ── SECTION 5: OUTPUTS ─────────────────────────────
    ├── VPCId
    ├── PublicSubnetAId / PublicSubnetBId
    ├── PrivateSubnetAId / PrivateSubnetBId
    ├── WebServerAPublicIP + WebServerBPublicIP
    └── AppServerAInstanceId + AppServerBInstanceId
```

---

## 8. Tagging Convention

Applied consistently to every resource:

| Tag Key | Tag Value |
|---|---|
| `Name` | Resource-specific (e.g., `vpc-lab-WebServer-A`) |
| `Project` | `VPC-CFN-Lab` |
| `Environment` | `Lab` |
| `Owner` | `Bernard Ograh` |

---

## 9. Repository Structure

```
/
├── template.yaml                          # CloudFormation template (single file)
├── README.md                              # Deployment instructions + architecture notes
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-04-30-vpc-cfn-lab-design.md   # This document
```

---

## 10. Validation Steps (Live Demo Checklist)

1. **Browser access** — open `http://<WebServerA-PublicIP>` and `http://<WebServerB-PublicIP>` in browser; confirm name and lab name display
2. **Public ↔ Private ping** — from WebServer-A Session Manager session: `ping <AppServer-A private IP>`
3. **Private ↔ Public ping** — from AppServer-A Session Manager session: `ping <WebServer-A private IP>`
4. **Outbound internet from private instance** — from AppServer-A Session Manager: `ping 8.8.8.8` or `curl -I https://google.com` or `sudo yum install -y telnet`
5. **Traceroute via NAT** — from AppServer-A: `traceroute 8.8.8.8` (first hop should be NAT Gateway private IP)

---

## 11. Architecture Knowledge Notes (Rubric Q&A)

### What is the AWS Regional NAT Gateway?
AWS announced a **Regional NAT Gateway** mode where a single NAT Gateway can serve traffic from multiple AZs within the same region, rather than being scoped to a single AZ. It is billed as a single resource but provides connectivity across the region.

### How does it differ from AZ-scoped NAT Gateways?
| Aspect | AZ-Scoped NAT Gateway (this lab) | Regional NAT Gateway |
|---|---|---|
| Placement | Deployed in a specific AZ | Region-level, not tied to one AZ |
| Count needed for HA | One per AZ (2 in this lab) | One per region |
| Cross-AZ traffic | Incurs cross-AZ data transfer fees if used cross-AZ | Designed for multi-AZ use, avoids cross-AZ charges |
| Cost | Per NAT Gateway + data processing per AZ | Single resource, potentially lower overall cost |
| Failure domain | AZ-level | Regional (higher resilience boundary) |

### How could it replace the two NAT Gateways in this architecture?
If a Regional NAT Gateway were used, both Private Subnet A and Private Subnet B could point their `0.0.0.0/0` routes to the single Regional NAT Gateway. This would reduce the number of NAT Gateways from 2 to 1, simplifying the template and reducing cost, while maintaining outbound internet access for both private subnets without cross-AZ traffic penalties.
