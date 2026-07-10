# LAB 03 — CI/CD Pipeline Engineering 

**Technologies:** GitHub Actions / GitLab CI / Jenkins
**Focus Areas:** Continuous Integration, Continuous Delivery, Automated Testing, Deployment Automation, Zero-Downtime Releases

---

# 🎯 Objective

Build robust, automated CI/CD pipelines covering:

* Next.js frontend applications
* Capacitor mobile application builds
* Magento PHP backend services

The lab focuses on implementing production-grade delivery workflows with:

* Automated testing
* Build automation
* Artifact management
* Cloud deployment
* Environment promotion
* Blue-Green deployments
* Automated rollback mechanisms

---

# Task 1 — Next.js Frontend CI/CD Pipeline

## 📌 Task Overview

Create a complete CI/CD workflow for a Next.js frontend application.

The pipeline should provide:

* Automated code quality validation
* Application testing
* Production-ready builds
* Artifact management
* Automated deployment
* Controlled environment promotion

---

# Step-by-Step Implementation

## Step 1 — Configure Pipeline Triggers

Create a GitHub Actions workflow triggered by:

* Push events to the `main` branch
* All `pull_request` events

Example workflow location:

```text
.github/workflows/frontend.yml
```

---

# Pipeline Stage 1 — Lint + Test

## Purpose

Validate application quality before building or deploying.

Run:

* ESLint
* TypeScript validation
* Jest unit tests

The pipeline must **fail fast** when any validation step fails.

Example commands:

```bash
npm run lint
npm test -- --coverage
```

---

# Pipeline Stage 2 — Build

## Purpose

Generate production-ready frontend assets.

Run:

```bash
next build
```

Upload the generated build artifact:

```text
.next
```

The artifact will be consumed by the deployment stage.

---

# Pipeline Stage 3 — Deploy

## Purpose

Deploy static assets and refresh CDN content.

Deployment tasks:

1. Sync built assets to Amazon S3.
2. Trigger CloudFront cache invalidation.

Example:

```bash
aws s3 sync .next/static s3://$BUCKET/_next/static --delete

aws cloudfront create-invalidation \
--distribution-id $CF_ID \
--paths '/*'
```

---

# Environment Promotion Strategy

Implement deployment promotion rules:

| Environment | Deployment Rule                  |
| ----------- | -------------------------------- |
| Development | Automatic deployment after merge |
| Production  | Manual approval required         |

Production deployments must use environment protection controls.

---

# 🔧 GitHub Actions Example — Next.js Pipeline

**File:** `.github/workflows/frontend.yml`

```yaml
jobs:

  test:

    runs-on: ubuntu-latest

    steps:

    - uses: actions/checkout@v4

    - uses: actions/setup-node@v4

      with:
        node-version: '20'

    - run: npm ci

    - run: npm run lint && npm test -- --coverage


  build:

    needs: test

    steps:

    - run: npm run build

    - uses: actions/upload-artifact@v4

      with:
        name: nextjs-build

        path: .next/


  deploy-prod:

    needs: build

    environment: production

    steps:

    - run: aws s3 sync .next/static s3://$BUCKET/_next/static --delete

    - run: aws cloudfront create-invalidation --distribution-id $CF_ID --paths '/*'
```

---

# Task 2 — Capacitor Mobile Build Pipeline

## 📌 Task Overview

Create a dedicated mobile CI/CD pipeline for Capacitor applications.

The pipeline should automate:

* Android builds
* iOS builds
* Dependency caching
* Application signing
* Release publishing

---

# Step-by-Step Implementation

## Step 1 — Configure Mobile Release Workflow

Create a separate workflow triggered by version tags:

```text
v*.*.*
```

Example:

```text
v1.0.0
v2.5.1
v3.0.0
```

---

## Step 2 — Configure Build Runners

Use platform-specific GitHub Actions runners:

| Platform | Runner          |
| -------- | --------------- |
| iOS      | `macos-latest`  |
| Android  | `ubuntu-latest` |

---

## Step 3 — Enable Dependency Caching

Cache:

* Gradle dependencies
* CocoaPods dependencies

Purpose:

* Reduce build execution time
* Improve pipeline efficiency

Expected improvement:

> Dependency caching can reduce build time by approximately 60%.

---

## Step 4 — Configure Application Signing

### Android

Sign:

* APK
* AAB

Use:

* Keystore file
* Credentials stored securely in GitHub Secrets

---

### iOS

Sign:

* IPA package

Use:

* Provisioning profile
* Secure signing credentials

---

## Step 5 — Publish Mobile Releases

Automatically upload build artifacts to:

* GitHub Release

Triggered by:

* Version tag creation

---

# Task 3 — Magento PHP Backend Pipeline

## 📌 Task Overview

Build a production CI/CD pipeline for Magento PHP applications.

The pipeline includes:

* Code quality validation
* Integration testing
* Docker image creation
* Container registry publishing
* ECS deployment
* Automated rollback

---

# Step-by-Step Implementation

## Step 1 — PHP Code Quality Checks

Run the following tools on every pull request:

### PHP_CodeSniffer

Configuration:

```text
PSR-12 Standard
```

Purpose:

* Enforce PHP coding standards

---

### PHPStan

Configuration:

```text
Level 6+
```

Purpose:

* Static code analysis
* Early defect detection

---

# Step 2 — Run PHPUnit Integration Tests

Execute PHPUnit integration tests using:

```text
MySQL service container
```

Purpose:

* Validate application behavior against a real database service

---

# Step 3 — Build and Publish Magento Docker Image

Create Docker image:

Base image:

```text
php:8.3-fpm
```

Push image to:

```text
Amazon Elastic Container Registry (ECR)
```

Image tagging strategy:

```text
<image>:SHA
<image>:latest
```

---

# Step 4 — Deploy Magento to ECS

Deploy using:

```text
Blue-Green Deployment Strategy
```

Traffic migration:

```text
ALB Listener Rule
        ↓
Blue Environment
        ↓
Green Environment
        ↓
Traffic Shift
```

Benefits:

* Reduced deployment risk
* Zero-downtime releases
* Safer production changes

---

# Step 5 — Automatic Rollback

Monitor:

```text
ALB HTTP 5xx Error Rate
```

Rollback condition:

```text
Error rate exceeds 1% within 5 minutes
```

---

# 🔧 Automatic Rollback Check — CI Bash Example

```bash
ERROR_RATE=$(aws cloudwatch get-metric-statistics \
--metric-name HTTPCode_Target_5XX_Count \
--start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
--end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
--period 300 \
--statistics Sum | jq '.Datapoints[0].Sum')


if [ "$ERROR_RATE" -gt 50 ]; then

echo 'High error rate — triggering rollback'

aws ecs update-service \
--cluster prod \
--service magento \
--task-definition $PREVIOUS_TASK_DEF

fi
```

---

# ✅ Best Practices — CI/CD Pipelines

## ⚡ Fail Fast

Place:

* Linting
* Static analysis
* Validation

at the earliest pipeline stage.

Benefits:

* Low execution cost
* Faster feedback
* Early defect detection

---

## 🚀 Cache Dependencies Aggressively

Cache:

* `node_modules`
* Python packages
* Gradle dependencies
* Maven dependencies

Expected improvement:

> Pipeline execution time can be reduced by 50–70%.

---

## 🔐 Use OIDC Federation for Cloud Credentials

Use:

* OpenID Connect (OIDC)
* Short-lived credentials

Avoid:

* Long-lived cloud access keys stored in secrets

---

## 📁 Store Pipeline Definitions with Application Code

Keep CI/CD definitions:

```text
.github/workflows/
.gitlab-ci.yml
Jenkinsfile
```

inside the same repository as application code.

Benefits:

* Version control
* Code review through pull requests
* Improved governance

---

## 🛡️ Enforce Deployment Security Controls

Never deploy directly from feature branches to production.

Implement:

* Branch protection rules
* Required approvals
* Environment controls

---

## 📊 Track DORA Metrics

Measure pipeline and delivery performance using:

* Deployment frequency
* Lead time for changes
* Change failure rate
* Mean Time To Recovery (MTTR)

Purpose:

* Measure engineering effectiveness
* Improve delivery processes

---

# 🏁 Lab Completion Summary

After completing this lab, you will have implemented:

✅ Automated Next.js CI/CD workflow

✅ Mobile application build automation

✅ Secure application signing processes

✅ Magento PHP quality and testing pipeline

✅ Docker-based deployment workflow

✅ ECS Blue-Green deployments

✅ Automated production rollback

✅ Enterprise CI/CD governance practices
