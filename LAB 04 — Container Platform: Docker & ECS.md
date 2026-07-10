# LAB 04 — Container Platform: Docker & ECS

**Technologies:** Docker / Amazon ECS / Amazon ECR
**Focus Areas:** Container Image Engineering, Container Security, ECS Orchestration, Blue-Green Deployments

---

# 🎯 Objective

Containerise applications using production-grade Docker images, manage container images through Amazon Elastic Container Registry (ECR), and orchestrate workloads using Amazon Elastic Container Service (ECS) with Blue-Green deployment strategies.

This lab covers:

* Secure Docker image creation
* Multi-stage Docker builds
* Container vulnerability scanning
* ECR image management
* ECS cluster configuration
* Task definitions
* Service scaling
* Zero-downtime deployments

---

# Task 1 — Production Docker Image (Multi-Stage Build)

## 📌 Task Overview

Create secure, optimized production container images using Docker multi-stage builds.

The container image design should:

* Separate build and runtime environments
* Reduce final image size
* Minimise attack surface
* Run applications without root privileges
* Enforce vulnerability scanning before deployment

---

# Step-by-Step Implementation

## Step 1 — Create a Multi-Stage Dockerfile

Create a multi-stage Docker build:

### Builder Stage

Responsibilities:

* Compile application dependencies
* Install required build packages
* Generate production assets

### Runtime Stage

Responsibilities:

* Use a minimal base image
* Run only required application components
* Reduce security exposure

---

## Step 2 — Use a Minimal Runtime Base Image

Use:

* Distroless images
* Alpine Linux images

Purpose:

* Reduce image size
* Minimise available attack vectors
* Improve container security posture

---

## Step 3 — Run Containers as Non-Root User

Containers must not run as root.

Implementation:

* Create an application-specific user
* Assign required permissions
* Use Docker `USER` directive

Benefits:

* Reduces privilege escalation risks
* Improves container isolation

---

## Step 4 — Scan Container Images in CI

Integrate Trivy image scanning into CI pipelines.

Requirement:

⚠️ Fail the build if any **CRITICAL vulnerability** is detected.

Example workflow:

```text id="7x5g1v"
Docker Build
      ↓
Trivy Security Scan
      ↓
Vulnerability Check
      ↓
Push Image to ECR
```

---

## Step 5 — Tag and Push Images to ECR

Tag container images using:

* Git commit SHA
* Semantic version

Example:

```text id="a9k8c3"
magento:a8f92d1
magento:v1.2.3
```

Push images to Amazon ECR.

Configuration requirement:

* Enable immutable image tags

Purpose:

* Prevent accidental image replacement
* Improve deployment reliability

---

# 🔧 Dockerfile Example — Multi-Stage Magento/PHP Build

```dockerfile id="w8r9dz"
FROM composer:2 AS vendor

WORKDIR /app

COPY composer.json composer.lock ./

RUN composer install --no-dev --optimize-autoloader


FROM php:8.3-fpm-alpine AS runtime

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /var/www/html

COPY --from=vendor /app/vendor ./vendor

COPY . .

RUN chown -R appuser:appgroup /var/www/html

USER appuser

EXPOSE 9000

CMD ["php-fpm"]
```

---

# Task 2 — ECS Cluster & Service Configuration

## 📌 Task Overview

Deploy containerized applications using Amazon ECS.

This task covers:

* ECS cluster creation
* Task definition configuration
* Service deployment
* Auto scaling
* Blue-Green deployment automation

---

# Step-by-Step Implementation

## Step 1 — Create ECS Cluster

Create an ECS Cluster using the appropriate launch type:

| Workload Type      | Recommended Launch Type |
| ------------------ | ----------------------- |
| Stateless services | AWS Fargate             |
| Magento workloads  | ECS EC2 Launch Type     |

Purpose:

* Fargate provides managed container execution
* EC2 provides additional operating system-level tuning control

---

## Step 2 — Define ECS Task Definition

Create an ECS Task Definition containing:

Required configuration:

* CPU limits
* Memory limits
* CloudWatch logging
* Secrets Manager references

Sensitive credentials must be retrieved from:

* AWS Secrets Manager
* Parameter Store

---

## Step 3 — Create ECS Service

Create an ECS Service with:

| Configuration | Value                       |
| ------------- | --------------------------- |
| Desired Count | 2                           |
| Availability  | Multiple Availability Zones |
| Load Balancer | ALB Target Group            |

The service must distribute workloads across Availability Zones.

---

## Step 4 — Configure ECS Service Auto Scaling

Configure scaling based on:

```text id="7n0f5y"
ALB RequestCountPerTarget Metric
```

Scaling rule:

```text id="e0q3t2"
Scale Out:
500 requests per target
```

Purpose:

* Automatically handle increased traffic
* Maintain application performance

---

## Step 5 — Implement Blue-Green Deployment

Use:

* AWS CodeDeploy
* ECS deployment controller

Deployment configuration:

```text id="k5qv8w"
Traffic Shift Bake Time:
10 minutes
```

Deployment flow:

```text id="4r5f0p"
Current Environment (Blue)
          |
          ↓
New Environment (Green)
          |
          ↓
Traffic Validation
          |
          ↓
Traffic Shift
          |
          ↓
Production Release
```

Benefits:

* Zero-downtime deployments
* Safer production releases
* Automated rollback capability

---

# 🔧 ECS Task Definition Example — JSON Excerpt

```json id="m0g8k6"
"containerDefinitions": [
  {
    "name": "magento",

    "image": "123456.dkr.ecr.ap-south-1.amazonaws.com/magento:v1.2.3",

    "cpu": 1024,

    "memory": 2048,

    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:...:db_password"
      }
    ],

    "logConfiguration": {
      "logDriver": "awslogs",

      "options": {
        "awslogs-group": "/ecs/magento"
      }
    }
  }
]
```

---

# ✅ Best Practices — Docker & ECS

## 🏷️ Use Immutable Image Versions

Always use specific image tags in production ECS Task Definitions.

Example:

✅ Recommended:

```text id="5z3x0k"
magento:v1.2.3
```

❌ Avoid:

```text id="p9k2qm"
magento:latest
```

---

## ⚙️ Define CPU and Memory Limits

Set:

* CPU limits
* Memory limits

for every container.

Purpose:

* Prevent resource exhaustion
* Avoid noisy neighbour problems
* Improve workload stability

---

## 🚨 Limit ECS Exec Usage

Use ECS Exec only for:

* Break-glass troubleshooting scenarios
* Emergency investigation

⚠️ Do not use ECS Exec as a routine operational workflow.

---

## 🔐 Secure Application Secrets

Store all secrets in:

* AWS Secrets Manager
* AWS Systems Manager Parameter Store

Never store credentials as:

* Plain environment variables
* Hardcoded values
* Source-controlled configuration files

---

## 🔍 Enable ECR Image Scanning

Enable:

* ECR image scanning on push

Configure lifecycle policies:

* Remove untagged images
* Delete images older than 14 days

Benefits:

* Reduce storage usage
* Maintain image hygiene
* Improve security posture

---

## ☁️ Select the Correct ECS Launch Type

Use:

### AWS Fargate

Recommended for:

* Stateless microservices
* Serverless container workloads

---

### ECS EC2 Launch Type

Recommended for:

* Magento workloads
* Applications requiring OS-level tuning
* Advanced infrastructure control

---

# 🏁 Lab Completion Summary

After completing this lab, you will have implemented:

✅ Production-grade multi-stage Docker builds

✅ Secure non-root container execution

✅ Automated container vulnerability scanning

✅ ECR image management with immutable tags

✅ ECS cluster and service configuration

✅ Container auto scaling

✅ Blue-Green ECS deployments

✅ Enterprise container security practices
