# LAB 02 — Terraform IaC Deep Dive

**Technologies:** Terraform / CloudFormation
**Focus Areas:** Terraform Modules, Remote State Management, Workspaces, CI/CD Enforcement, Infrastructure Security

---

# 🎯 Objective

Master advanced Terraform Infrastructure as Code (IaC) practices, including:

* Terraform module design
* Remote state management
* Workspace management
* CI-enforced plan and apply workflows
* Infrastructure security validation

The goal is to build production-ready IaC workflows that enable teams to safely develop, review, and deploy cloud infrastructure at scale.

---

# Task 1 — Modularise Your Terraform Code

## 📌 Task Overview

Structure your Terraform repository into reusable modules so teams can compose infrastructure components like **Lego blocks**.

A modular Terraform architecture improves:

* Code reuse
* Maintainability
* Environment consistency
* Team collaboration
* Infrastructure scalability

---

## Step-by-Step Implementation

### Step 1 — Create Terraform Module Structure

Create the following reusable module directories:

```text
modules/
├── vpc/
├── ec2/
├── rds/
└── alb/
```

Each module should represent an independent infrastructure component.

---

### Step 2 — Define Module Inputs and Outputs

Each Terraform module must contain:

| File           | Purpose                         |
| -------------- | ------------------------------- |
| `variables.tf` | Defines module input variables  |
| `outputs.tf`   | Exposes reusable module outputs |

Example:

```text
modules/vpc/
├── main.tf
├── variables.tf
└── outputs.tf
```

---

### Step 3 — Create Environment-Specific Root Modules

Create separate Terraform root configurations for each environment:

```text
environments/
├── dev/
├── staging/
└── prod/
```

Each environment should maintain independent configuration and state management.

---

### Step 4 — Call Modules from Environment Roots

Each environment root should consume reusable modules with environment-specific variable files:

```text
environments/prod/
├── main.tf
├── variables.tf
├── outputs.tf
└── prod.tfvars
```

---

# 🔧 Terraform Configuration Example — Production Environment

**File:** `environments/prod/main.tf`

```hcl
module "vpc" {

  source = "../../modules/vpc"

  cidr_block = var.vpc_cidr

  environment = "production"
}


module "rds" {

  source = "../../modules/rds"

  vpc_id = module.vpc.vpc_id

  subnet_ids = module.vpc.private_subnet_ids

  instance_class = "db.r6g.large"

  multi_az = true
}
```

---

# Task 2 — Remote State & Workspace Management

## 📌 Task Overview

Terraform state management is critical for team-based infrastructure deployment.

In this task, you will configure:

* Remote Terraform state storage
* State locking
* Environment isolation
* Workspace-based environment management

---

## Step-by-Step Implementation

### Step 1 — Create Remote State Storage

Create an S3 bucket:

```text
company-terraform-state
```

Required configuration:

* Enable versioning
* Enable server-side encryption
* Restrict access using IAM policies

---

### Step 2 — Create Terraform State Locking

Create a DynamoDB table:

```text
terraform-locks
```

Configuration:

| Setting       | Value           |
| ------------- | --------------- |
| Table Name    | terraform-locks |
| Partition Key | LockID          |

Purpose:

* Prevent concurrent Terraform operations
* Protect state consistency

---

### Step 3 — Configure S3 Backend

Configure the S3 backend in each environment root.

Each environment must use a unique state path.

Example:

```text
prod/networking/terraform.tfstate
staging/networking/terraform.tfstate
dev/networking/terraform.tfstate
```

---

### Step 4 — Manage Feature Environments with Terraform Workspace

Use Terraform workspaces for temporary environments.

Example:

```bash
terraform workspace new feature-x
```

Workspaces allow teams to isolate infrastructure states for feature development.

---

# 🔧 Terraform Backend Configuration Example

```hcl
terraform {

  backend "s3" {

    bucket = "company-terraform-state"

    key = "prod/networking/terraform.tfstate"

    region = "ap-south-1"

    dynamodb_table = "terraform-locks"

    encrypt = true
  }
}
```

---

# Task 3 — CI/CD Enforcement of Terraform Plans

## 📌 Task Overview

Implement automated Terraform validation and security controls through CI/CD pipelines.

The workflow ensures:

* Code formatting compliance
* Configuration validation
* Automated plan reviews
* Controlled production deployments
* Infrastructure security scanning

---

## Step-by-Step Implementation

### Step 1 — Validate Terraform Code on Pull Requests

Add a GitHub Actions workflow that runs:

```bash
terraform fmt -check
terraform validate
```

on every pull request.

Purpose:

* Enforce Terraform formatting standards
* Detect configuration errors early

---

### Step 2 — Generate Terraform Plan in CI

Run:

```bash
terraform plan
```

and publish the output as a pull request comment.

Use:

* `hashicorp/setup-terraform` GitHub Action

Purpose:

* Allow reviewers to verify infrastructure changes before approval

---

### Step 3 — Protect Terraform Apply

Terraform apply must require manual approval.

Configure:

* GitHub Environment protection rules
* Required reviewers

Production deployment flow:

```text
Pull Request
      ↓
Terraform Validate
      ↓
Terraform Plan
      ↓
Security Scan
      ↓
Manual Approval
      ↓
Terraform Apply
```

---

### Step 4 — Add Terraform Security Scanning

Integrate security scanning tools:

* tfsec
* Checkov

Purpose:

* Detect insecure Terraform configurations
* Enforce cloud security standards

---

# 🔧 GitHub Actions Terraform Workflow Example

**File:** `.github/workflows/terraform.yml`

```yaml
- name: Terraform Plan

  run: |
    terraform init
    terraform plan -out=tfplan -var-file=prod.tfvars


- name: Security Scan

  run: checkov -d . --framework terraform


- name: Apply (prod only, manual gate)

  if: github.ref == 'refs/heads/main'

  environment: production

  run: terraform apply tfplan
```

---

# ✅ Best Practices — Terraform IaC

## 📌 Provider and Module Version Pinning

Always pin provider and module versions.

⚠️ **Production Requirement:**

Never use:

```text
latest
>= any
```

for production deployments.

---

## 📚 Generate Module Documentation

Use:

```text
terraform-docs
```

to automatically generate module documentation.

Benefits:

* Keeps documentation synchronized with code
* Improves module usability
* Supports team adoption

---

## 🔍 Always Run Terraform Plan

Run:

```bash
terraform plan
```

before every apply operation.

⚠️ **Important:**
Never skip Terraform plan validation, even in automated workflows.

---

## 🔗 Use Data Sources Instead of Hardcoded IDs

Avoid hardcoded resource identifiers.

Example:

```hcl
data "aws_ami" "latest" {}
```

Benefits:

* Portable configurations
* Easier environment migration
* Reduced maintenance effort

---

## 🔄 Implement Drift Detection

Schedule automated Terraform plans:

```text
Nightly CI Job
        ↓
terraform plan
        ↓
Detect Configuration Drift
        ↓
Alert Engineering Team
```

Purpose:

* Identify manual infrastructure changes
* Maintain desired state consistency

---

## 🗂️ Separate Terraform State Files

Maintain separate state files:

* Per environment
* Per logical infrastructure layer

Recommended structure:

```text
terraform-state/
├── dev/
│   ├── network/
│   ├── compute/
│   └── data/
│
├── staging/
│
└── prod/
    ├── network/
    ├── compute/
    └── data/
```

---

# 🏁 Lab Completion Summary

After completing this lab, you will have implemented:

✅ Reusable Terraform module architecture

✅ Environment-based infrastructure management

✅ Secure remote Terraform state storage

✅ Workspace-based development workflows

✅ CI/CD Terraform validation pipelines

✅ Automated security scanning

✅ Production-grade Terraform governance practices
