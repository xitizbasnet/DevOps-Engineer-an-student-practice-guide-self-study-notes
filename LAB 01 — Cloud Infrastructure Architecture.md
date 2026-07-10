# LAB 01 — Cloud Infrastructure Architecture

**Technologies:** AWS / GCP / Azure
**Focus Areas:** Cloud Networking, Compute, Database, Storage, Terraform Infrastructure as Code (IaC)

---

# 🎯 Objective

Design, provision, and manage a **scalable, secure, and cost-effective multi-tier cloud infrastructure** using **Terraform as Infrastructure as Code (IaC)**.

This lab focuses on building production-grade cloud infrastructure patterns including:

* Network architecture
* Compute provisioning
* Database deployment
* Storage configuration
* Load balancing
* Auto scaling
* Security controls
* Infrastructure best practices

---

# Task 1 — Design the Network (VPC / Subnets / Security Groups)

## 📌 Task Overview

A well-designed network is the backbone of any production environment.

In this task, you will create a Virtual Private Cloud (VPC) with:

* Public and private subnets distributed across two Availability Zones
* Internet Gateway configuration
* NAT Gateway configuration
* Route table management
* Strict Security Group controls

The final architecture provides isolated application and database tiers while preventing direct Internet exposure to private resources.

---

## Step-by-Step Implementation

### Step 1 — Create a VPC

Create a VPC with the following configuration:

| Parameter  | Value                                                   |
| ---------- | ------------------------------------------------------- |
| CIDR Block | `10.0.0.0/16`                                           |
| Region     | Selected cloud region (`ap-south-1` for Mumbai example) |

---

### Step 2 — Create Public Subnets

Create two public subnets across separate Availability Zones:

| Subnet          | CIDR          | Availability Zone |
| --------------- | ------------- | ----------------- |
| Public Subnet A | `10.0.1.0/24` | AZ-a              |
| Public Subnet B | `10.0.2.0/24` | AZ-b              |

---

### Step 3 — Create Private Subnets

Create two private subnets for application and database workloads:

| Subnet           | CIDR           | Purpose          |
| ---------------- | -------------- | ---------------- |
| Private Subnet A | `10.0.10.0/24` | Application Tier |
| Private Subnet B | `10.0.11.0/24` | Database Tier    |

---

### Step 4 — Configure Internet Gateway

Attach an Internet Gateway (IGW) to the VPC.

Update the public route table:

```text
0.0.0.0/0 → Internet Gateway (IGW)
```

This enables Internet access for resources deployed in public subnets.

---

### Step 5 — Configure NAT Gateway

Create a NAT Gateway inside a public subnet.

Configure private subnet routing:

```text
Private Subnet Traffic → NAT Gateway → Internet Gateway
```

This allows private resources to access external services without exposing them directly to the Internet.

---

### Step 6 — Define Security Groups

Configure security boundaries between infrastructure layers.

| Security Group             | Allowed Traffic                                      |
| -------------------------- | ---------------------------------------------------- |
| ALB Security Group         | HTTP/HTTPS (80/443) open                             |
| Application Security Group | Port 8080 from ALB Security Group only               |
| Database Security Group    | PostgreSQL 5432 from Application Security Group only |

---

# 🔧 Terraform Configuration Example — VPC

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "prod-vpc"
    Environment = "production"
  }
}

resource "aws_subnet" "public_a" {
  vpc_id = aws_vpc.main.id

  cidr_block = "10.0.1.0/24"

  availability_zone = "ap-south-1a"

  map_public_ip_on_launch = true
}
```

---

> ✅ **Result**
> You have a production-grade VPC with isolated tiers and no direct Internet exposure to private resources.

---

# Task 2 — Provision Compute, Database & Storage

## 📌 Task Overview

Deploy the core application infrastructure components:

* Application Load Balancer
* Auto Scaling compute layer
* Managed PostgreSQL database
* Secure object storage
* Global content delivery

The architecture provides:

* High availability
* Horizontal scalability
* Managed database operations
* CDN-based content acceleration

---

## Step-by-Step Implementation

### Step 1 — Deploy Application Load Balancer (ALB)

Create an Application Load Balancer:

Configuration requirements:

* Deploy across both public subnets
* Configure HTTPS listener
* Attach ACM SSL/TLS certificate

---

### Step 2 — Create EC2 Launch Template

Create an EC2 Launch Template with:

| Parameter        | Configuration                 |
| ---------------- | ----------------------------- |
| Operating System | Latest Amazon Linux 2023 AMI  |
| Instance Type    | `t3.medium`                   |
| Initialization   | User-data installation script |

The user-data script should install required application dependencies automatically.

---

### Step 3 — Configure Auto Scaling Group

Attach an Auto Scaling Group to the ALB Target Group.

Configuration:

| Setting           | Value |
| ----------------- | ----- |
| Minimum Instances | 2     |
| Maximum Instances | 6     |
| Desired Capacity  | 2     |
| Health Check Type | ELB   |

---

# 🔧 Terraform Configuration Example — Auto Scaling Group

```hcl
resource "aws_autoscaling_group" "app" {

  desired_capacity = 2
  max_size         = 6
  min_size         = 2

  target_group_arns = [
    aws_lb_target_group.app.arn
  ]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  health_check_type = "ELB"
}
```

---

### Step 4 — Provision RDS PostgreSQL

Create a managed PostgreSQL database:

| Parameter     | Configuration           |
| ------------- | ----------------------- |
| Engine        | PostgreSQL              |
| Instance Type | `db.t3.medium`          |
| Availability  | Multi-AZ                |
| Network       | Private DB subnet group |

---

### Step 5 — Create S3 Storage

Create an S3 bucket for static assets.

Requirements:

* Enable versioning
* Block all public access
* Secure object storage configuration

---

### Step 6 — Configure CloudFront Distribution

Create a CloudFront Distribution in front of:

* S3 static assets
* Application Load Balancer

Purpose:

* Content caching
* Global delivery
* Reduced latency

---

> ✅ **Result**
> Scalable compute behind a load balancer, with a managed database and CDN-accelerated static content.

---

# ✅ Best Practices — Cloud Infrastructure

## 🏷️ Resource Tagging

Always tag every resource with:

* `Environment`
* `Owner`
* `CostCenter`

Purpose:

* Cost allocation
* Ownership tracking
* Resource management

---

## 🌍 High Availability

Use Multi-AZ deployment for all stateful services:

* RDS
* ElastiCache

⚠️ **Production Requirement:**
Single-AZ deployment is never acceptable for production workloads.

---

## 🔐 EC2 Instance Metadata Security

Enforce **IMDSv2** on all EC2 instances.

Purpose:

* Prevent SSRF-based metadata attacks
* Improve instance security posture

---

## 🗄️ Terraform State Management

Store Terraform state remotely:

Recommended configuration:

* Amazon S3 for state storage
* DynamoDB for state locking

⚠️ **Important:**
Never commit Terraform state files to Git repositories.

---

## 📊 Network Monitoring

Enable:

* VPC Flow Logs
* Centralized log shipping to:

  * Amazon S3
  * CloudWatch

Purpose:

* Network visibility
* Security investigations
* Network forensics

---

## 🔑 IAM Security

Implement least-privilege IAM practices:

Recommended approach:

* Use Instance Profiles
* Assign only required permissions
* Never hardcode credentials

---

# 🏁 Lab Completion Summary

After completing this lab, you will have implemented:

✅ Production-ready VPC architecture

✅ Public and private network segmentation

✅ Secure application and database tiers

✅ Load-balanced scalable compute layer

✅ Multi-AZ managed database deployment

✅ Secure object storage and CDN delivery

✅ Terraform-managed cloud infrastructure

✅ Enterprise cloud security best practices

