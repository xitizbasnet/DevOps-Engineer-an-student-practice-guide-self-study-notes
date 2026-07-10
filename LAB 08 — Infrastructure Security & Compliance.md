# LAB 08 — Infrastructure Security & Compliance

## IAM / SAST / CIS Benchmarks 

# 📘 Objective

Implement security best practices across every layer of the infrastructure stack.

This lab focuses on:

* IAM security hardening
* Least privilege access control
* Secrets management
* Vulnerability scanning
* Security compliance monitoring
* CIS benchmark alignment
* Cloud security operational controls

---

# 📖 Learning Outcomes

Upon completing this lab, you will be able to:

* Apply AWS IAM least privilege principles.
* Replace insecure credential management practices.
* Implement centralized secrets management.
* Integrate security scanning into CI/CD pipelines.
* Configure AWS security monitoring services.
* Improve infrastructure compliance posture.

---

# 🔐 Task 1 — IAM Hardening & Least Privilege

## Overview

Implement strong identity and access management controls to reduce security risks caused by excessive permissions, unmanaged credentials, and unauthorized access.

The goal is to follow the principle of:

> **Least Privilege — users and services should receive only the permissions required to perform their intended tasks.**

---

# Step 1 — Audit IAM Users

Perform a complete audit of all AWS IAM users.

## Required Actions

* Delete unused IAM users.
* Enable Multi-Factor Authentication (MFA) for all human users.
* Remove AWS Console access from service accounts.
* Review and remove unnecessary permissions.

### Security Requirements

| Identity Type        | Required Access Model    |
| -------------------- | ------------------------ |
| Human Users          | Federated access + MFA   |
| Service Accounts     | Programmatic access only |
| Kubernetes Workloads | IRSA                     |
| EC2 Workloads        | Instance Profiles        |

---

# Step 2 — Replace Long-Lived IAM Credentials

Replace all permanent IAM access keys.

## Required Migration

Replace:

```text
Long-lived IAM Access Keys
```

With:

```text
OIDC / IRSA (Amazon EKS)

or

EC2 Instance Profiles
```

### Benefits

* Reduced credential exposure risk
* Automatic credential rotation
* Short-lived authentication tokens
* Improved auditability

---

# Step 3 — Implement IAM Permission Boundaries

Configure IAM Permission Boundaries on all developer-created IAM roles.

## Purpose

Prevent:

* Privilege escalation
* Unauthorized permission expansion
* Excessive administrative access

Example:

```text
Developer Creates IAM Role
          |
          v
Permission Boundary Applied
          |
          v
Maximum Allowed Permissions Enforced
```

---

# Step 4 — Enable AWS CloudTrail

Enable AWS CloudTrail with enterprise security settings.

## Required Configuration

Enable:

* Multi-region logging
* Multi-account logging
* Log file validation

Store logs in:

```text
Dedicated Security S3 Bucket
```

### Benefits

* Security auditing
* Incident investigation
* Compliance evidence
* User activity tracking

---

# Step 5 — Enable AWS Config

Enable AWS Config with managed compliance rules.

Required managed rules:

| AWS Config Rule                    | Purpose                                |
| ---------------------------------- | -------------------------------------- |
| `restricted-ssh`                   | Detect unrestricted SSH access         |
| `s3-bucket-public-read-prohibited` | Prevent public S3 read access          |
| `iam-root-access-key-check`        | Ensure root account has no access keys |

---

# 🔑 Task 2 — Secrets Management

## Overview

Centralize application secrets management and eliminate insecure storage methods.

Secrets must never be stored directly in:

* Environment variables
* Application configuration files
* Source repositories

---

# Step 1 — Migrate Secrets to AWS Secrets Manager

Move all application secrets to:

```text
AWS Secrets Manager
```

Examples:

* Database passwords
* API credentials
* Application keys
* Service tokens

---

# Step 2 — Enable RDS Credential Rotation

Configure automatic database credential rotation.

Required components:

* AWS Secrets Manager native rotation
* AWS Lambda rotation function

Benefits:

* Automatic credential lifecycle management
* Reduced exposure window
* Improved compliance posture

---

# Step 3 — Install External Secrets Operator

Install the Kubernetes External Secrets Operator.

Purpose:

Automatically synchronize AWS Secrets Manager values into Kubernetes Secrets.

Architecture:

```text
AWS Secrets Manager
          |
          |
          v
External Secrets Operator
          |
          |
          v
Kubernetes Secret
          |
          |
          v
Application Pod
```

---

# External Secrets Operator — ExternalSecret Manifest

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:
  name: magento-db-creds

spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore

  target:
    name: magento-db-secret

  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/magento/db
        property: password
```

---

# Step 4 — Prevent Secret Leakage

Implement secret detection tools:

* `git-secrets`
* `gitleaks`

Run them:

* As a Git pre-commit hook
* During CI pipeline execution

## Objective

Prevent credentials and sensitive information from being committed into source repositories.

---

# 🛡️ Task 3 — Vulnerability Scanning & Compliance

## Overview

Implement continuous security scanning and compliance validation across cloud infrastructure, containers, and applications.

---

# Step 1 — Enable Amazon Inspector

Enable Amazon Inspector for:

* EC2 vulnerability scanning
* Container image vulnerability scanning

Purpose:

* Continuous vulnerability detection
* Risk prioritization
* Security visibility

---

# Step 2 — Integrate Trivy into CI/CD

Run Trivy vulnerability scanning during CI builds.

## Build Requirements

| Severity | Action                                    |
| -------- | ----------------------------------------- |
| CRITICAL | Fail build                                |
| HIGH     | Create remediation ticket for next sprint |
| MEDIUM   | Track and remediate according to priority |

Example workflow:

```text
Code Commit
      |
      v
CI Pipeline
      |
      v
Trivy Scan
      |
      +---- Critical Finding → Build Failed
      |
      +---- Medium/High → Ticket Created
```

---

# Step 3 — Enable AWS Security Hub

Enable AWS Security Hub with:

```text
CIS AWS Foundations Benchmark
```

## Compliance Target

Target compliance score:

```text
90%+
```

Security Hub provides:

* Centralized security findings
* Compliance reporting
* Continuous assessment

---

# Step 4 — Implement AWS WAF

Enable AWS WAF on all Application Load Balancers (ALBs).

Configure AWS Managed Rules:

| Rule Set                 | Purpose                          |
| ------------------------ | -------------------------------- |
| Core Rule Set            | Common web exploits              |
| SQL Injection Protection | Prevent SQL injection attacks    |
| Known Bad Inputs         | Block malicious request patterns |

---

# Step 5 — Enable GuardDuty

Enable Amazon GuardDuty across all AWS accounts.

Configure:

* Threat detection
* Continuous monitoring
* Security finding analysis

## Notification Requirement

Send HIGH severity findings to:

```text
Security Slack Channel
```

using SNS notifications.

---

# 📝 Best Practices — Security

## 🐞 Treat Security Findings Like Production Bugs

Every security finding should have:

* Severity classification
* Assigned owner
* Remediation SLA

Security issues must follow the same operational discipline as production incidents.

---

## 🚫 Never Allow Public SSH Access

Never configure:

```text
0.0.0.0/0 on TCP Port 22
```

in any Security Group.

Recommended alternative:

```text
AWS Systems Manager Session Manager
```

Benefits:

* No inbound SSH exposure
* Centralized access auditing
* Improved security posture

---

## 🔒 Encrypt Everything

Enable encryption for all supported AWS services.

Required encryption targets:

| Service | Encryption Requirement                     |
| ------- | ------------------------------------------ |
| EBS     | Encryption enabled                         |
| RDS     | Encryption enabled                         |
| S3      | SSE-S3 minimum, SSE-KMS for sensitive data |
| SQS     | Encryption enabled                         |
| SNS     | Encryption enabled                         |

---

## 🔄 Rotate Credentials Regularly

Rotate credentials on a defined schedule.

Security principle:

> Assume breach and design systems accordingly.

Credential rotation reduces the impact of potential compromise.

---

## 🌐 Implement VPC Endpoints

Configure VPC endpoints for:

* Amazon S3
* Amazon DynamoDB

Benefits:

* Keep traffic on the AWS private backbone
* Reduce public internet exposure
* Improve network security controls

---

## 📚 Maintain Security Documentation

Document all security controls in a shared security runbook.

Purpose:

* Operational reference
* Audit readiness
* Cyber insurance evidence
* Compliance documentation

---

# ✅ Lab Completion Checklist

| Task                                    | Status |
| --------------------------------------- | ------ |
| IAM users audited                       | ☐      |
| MFA enabled for human users             | ☐      |
| Service account console access removed  | ☐      |
| Long-lived IAM keys replaced            | ☐      |
| IRSA/Instance Profiles implemented      | ☐      |
| Permission boundaries configured        | ☐      |
| CloudTrail enabled                      | ☐      |
| AWS Config rules enabled                | ☐      |
| Secrets migrated to AWS Secrets Manager | ☐      |
| RDS credential rotation configured      | ☐      |
| External Secrets Operator deployed      | ☐      |
| Secret scanning integrated into CI/CD   | ☐      |
| Amazon Inspector enabled                | ☐      |
| Trivy CI scanning implemented           | ☐      |
| Security Hub CIS benchmark enabled      | ☐      |
| AWS WAF configured                      | ☐      |
| GuardDuty enabled                       | ☐      |
| Security runbook documented             | ☐      |

---

