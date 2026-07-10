# LAB 10 — Developer Experience (DevEx)

## Dev Containers / LocalStack

---

# 📘 Overview

This lab focuses on improving **Developer Experience (DevEx)** by creating standardised, automated, and efficient development environments.

The objective is to become a **force multiplier for the engineering team** by reducing setup friction, accelerating onboarding, and implementing fast feedback loops through inner-loop development practices.

---

# 🎯 Objective

**Be a force multiplier for the engineering team — build fast local development environments, streamline onboarding, and implement inner-loop CI so developers get feedback in minutes, not hours.**

---

# 🛠️ Task 1 — Standardised Local Development with Dev Containers

## Purpose

Use **Dev Containers** to provide a consistent development environment where the complete toolchain is defined as code.

A single configuration file should define:

* Development environment
* Required tools
* IDE extensions
* Workspace configuration
* Supporting services

---

## Step 1 — Create Dev Container Configuration

Create a `.devcontainer/devcontainer.json` file for each service:

* Next.js
* Magento
* API

Each file must define the complete development environment.

### Expected Outcome

Developers should be able to open the project in a container and immediately start development without manual configuration.

---

## Step 2 — Include Required Development Tools

The Dev Container image must include all required development tools.

Required tools include:

* Node.js
* PHP
* Composer
* AWS CLI
* Terraform

### Goal

Provide:

* ✅ Zero manual setup
* ✅ Consistent developer environments
* ✅ Reduced onboarding effort

---

## Step 3 — Configure Dependent Services

Add `docker-compose.yml` support to the Dev Container environment.

Required dependent services:

* 🗄️ MySQL
* ⚡ Redis
* 🔎 Elasticsearch
* 📧 Mailhog

These services should be automatically available during local development.

---

## Step 4 — Standardise VS Code Experience

Configure VS Code extensions inside `devcontainer.json`.

Every developer should automatically receive the same IDE experience.

---

# 📄 Example Dev Container Configuration

```json
// .devcontainer/devcontainer.json

{
  "name": "Magento Dev",
  "dockerComposeFile": [
    "docker-compose.yml",
    "docker-compose.devcontainer.yml"
  ],
  "service": "app",
  "workspaceFolder": "/var/www/html",
  "extensions": [
    "bmewburn.vscode-intelephense-client",
    "ms-azuretools.vscode-docker",
    "hashicorp.terraform"
  ],
  "postCreateCommand": "composer install && php bin/magento setup:install"
}
```

---

# ☁️ Task 2 — LocalStack for AWS Service Emulation

## Purpose

Use LocalStack to emulate AWS services locally, allowing developers to test cloud integrations without using real AWS resources.

---

## Step 1 — Add LocalStack to Docker Compose

Add LocalStack to the `docker-compose.yml` configuration.

LocalStack provides local emulation for:

* 🪣 Amazon S3
* 📬 Amazon SQS
* 📢 Amazon SNS
* 🔐 AWS Secrets Manager
* ⚡ AWS Lambda

---

## Step 2 — Create LocalStack Initialisation Script

Create an initialisation script that automatically creates required resources when LocalStack starts.

Resources include:

* Queues
* Buckets
* Secrets

### Expected Behaviour

When developers start the local environment, required AWS resources should already exist.

---

## Step 3 — Configure Local AWS CLI Profile

Create a local AWS CLI profile pointing to:

```text
http://localhost:4566
```

This profile should be used for local AWS service testing.

---

## Step 4 — Configure Applications for LocalStack

Update application configuration to use LocalStack endpoints when:

```text
APP_ENV=development
```

### Benefits

* No real AWS costs during development
* Faster local testing
* Safer experimentation

---

# 🚀 Task 3 — Onboarding Automation

## Purpose

Automate developer onboarding so engineers can move from a fresh machine to a running development environment quickly.

---

## Step 1 — Create a Single Setup Command

Create a single Makefile target:

```bash
make setup
```

The command must automate:

* Repository cloning
* Dependency installation
* Local environment configuration
* Database seeding
* Service startup

---

## Step 2 — Define Onboarding Target

Target onboarding time:

> A new engineer should have a running local environment in **under 15 minutes**.

---

## Step 3 — Implement Pre-Commit Hooks

Use the **pre-commit framework** to run checks locally before code reaches CI.

Required checks:

* ✅ Linting
* ✅ Formatting
* 🔐 Secrets scanning

### Performance Requirement

Pre-commit hooks should complete in:

```text
< 30 seconds
```

---

## Step 4 — Configure GitHub Codespaces

Set up GitHub Codespaces as a cloud-based fallback environment.

Use cases:

* Low-powered developer machines
* Network limitations
* Remote development requirements

---

## Step 5 — Create a Developer Portal

Create a Developer Portal using:

* Backstage
* Similar internal developer platform solutions

The portal should list:

* Service catalogue
* On-call owners
* Runbooks
* Deployment status

---

# 📄 Example Makefile

```makefile
.PHONY: setup

setup:
	@echo '--- Cloning repositories ---'
	git clone git@github.com:company/magento.git

	@echo '--- Installing dependencies ---'
	cd magento && composer install

	@echo '--- Starting services ---'
	docker compose up -d

	@echo '--- Seeding database ---'
	docker compose exec app php bin/magento setup:install --use-sample-data

	@echo '✓ Setup complete! Visit http://localhost:8080'
```

---

# 📚 Best Practices — Developer Experience

## ✅ DevEx Principles

---

## 1. Measure Onboarding Time

> If onboarding takes more than **1 day**, it is a DevEx crisis requiring immediate attention.

Track onboarding friction continuously and remove unnecessary manual steps.

---

## 2. Keep the Makefile as the Single Entry Point

Use standard commands:

```bash
make setup
make test
make build
```

Avoid relying on:

* Tribal knowledge
* Unwritten procedures
* Individual developer experience

---

## 3. Keep Pre-Commit Hooks Fast

Pre-commit hooks should run in:

```text
< 30 seconds
```

Slow hooks lead to:

* Developers disabling them
* Reduced adoption
* Poor code quality enforcement

---

## 4. Provide Realistic Test Data

Give developers access to:

* Realistic
* Anonymised
* Production-like

data dumps for local testing.

---

## 5. Run Monthly DevEx Office Hours

Hold monthly **DevEx office hours**.

Purpose:

* Collect developer feedback
* Identify workflow friction
* Resolve recurring problems before they become cultural issues

---

## 6. Track Inner-Loop Metrics

Measure and optimise:

* 🏗️ Local build time
* 🧪 Test execution time
* 🔥 Hot-reload time

### Goal

Continuously improve developer productivity through measurable optimisation.

---

# 📌 Lab Summary

| Area                    | Technology                   | Outcome                              |
| ----------------------- | ---------------------------- | ------------------------------------ |
| Development Environment | Dev Containers               | Standardised local development setup |
| Service Dependencies    | Docker Compose               | Automated supporting services        |
| AWS Development         | LocalStack                   | Local AWS service emulation          |
| Onboarding Automation   | Makefile                     | One-command environment setup        |
| Code Quality            | Pre-commit Framework         | Early validation before CI           |
| Cloud Development       | GitHub Codespaces            | Remote development fallback          |
| Developer Platform      | Backstage / Developer Portal | Centralised engineering information  |

---

