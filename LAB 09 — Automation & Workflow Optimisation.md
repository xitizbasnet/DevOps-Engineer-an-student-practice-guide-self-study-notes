# LAB 09 — Automation & Workflow Optimisation

## Bash / Python / Ansible

## 📘 Overview

This lab focuses on building a culture of automation by developing **self-service tools**, **runbook automation**, and **ChatOps integrations** that reduce manual operational effort across the engineering organisation.

The goal is to eliminate repetitive manual tasks, improve operational consistency, and enable engineers to perform common workflows safely through automated systems.

---

# 🎯 Objective

**Drive a culture of automation — build self-service tools, runbook automation, and ChatOps integrations that eliminate manual toil across the engineering org.**

---

# 🛠️ Task 1 — Runbook Automation with Ansible

## Purpose

Use Ansible to automate recurring operational activities while ensuring tasks are **repeatable, reliable, and idempotent**.

> 💡 **Key Principle:** Running an automation task multiple times must produce the same desired outcome without causing unintended changes.

---

## Step 1 — Identify Manual Operational Tasks

Identify the top five manual operational tasks your team performs monthly.

Examples include:

* 🖥️ Server patching
* 🔐 Certificate renewal
* 🧹 Cache flush operations
* 🔄 Service restarts
* 📦 Application maintenance activities

Document each task before beginning automation.

---

## Step 2 — Create Idempotent Ansible Playbooks

Write an Ansible playbook for each identified operational task.

### Requirements

* Each playbook must follow Ansible best practices.
* Each playbook must be **idempotent**.
* Running the same playbook multiple times must produce the same result.
* Avoid unnecessary changes when the desired state has already been achieved.

---

## Step 3 — Store Playbooks Securely

Store all Ansible playbooks in a dedicated Git repository.

### Repository Requirements

* Maintain version control for all automation code.
* Follow a structured repository layout.
* Use meaningful commit messages.
* Protect sensitive information.

### Security Requirement

Use **Ansible Vault** to encrypt sensitive variables, including:

* Passwords
* API keys
* Credentials
* Environment-specific secrets

---

## Step 4 — Schedule Recurring Automation

Schedule recurring operational tasks, such as operating system patching, using:

* AWS Systems Manager Automation
* Maintenance Windows

### Example Use Cases

* Automated OS patch cycles
* Scheduled compliance checks
* Recurring maintenance operations

---

# 📄 Example Ansible Playbook

## Redis Cache Flush Automation

```yaml
# Ansible playbook — Redis cache flush

---
- name: Flush Magento Redis Cache
  hosts: magento_app

  tasks:
    - name: Flush Redis cache backend
      community.general.redis:
        command: flush
        db: 0
      when: flush_cache | default(false) | bool

    - name: Run Magento cache clean
      command: /var/www/current/bin/magento cache:clean
      become: true
      become_user: www-data
```

---

# 🤖 Task 2 — Self-Service Internal Tools

## Purpose

Create internal automation tools that allow engineers to execute common workflows without requiring manual intervention from operations teams.

---

## Step 1 — Create Slack Deployment Command

Build a Slack slash command:

```text
/deploy service=magento env=staging version=v1.2.3
```

The command should be backed by an AWS Lambda function.

---

## Step 2 — Lambda Validation and Pipeline Triggering

The Lambda function must:

1. Validate command inputs.
2. Confirm that the caller belongs to the **`engineers` Slack group**.
3. Trigger a CodePipeline execution.

---

## Step 3 — Deployment Status Notifications

Post deployment status updates back to the Slack channel as the pipeline progresses.

Deployment states:

* 🔄 In progress
* ✅ Success
* ❌ Failure

---

## Step 4 — Self-Service Database Snapshot Tool

Build a self-service database snapshot workflow.

### Workflow

1. Engineer requests a database snapshot through Slack.
2. Slack request invokes Lambda.
3. Lambda creates the database snapshot.
4. System sends a notification when the snapshot is ready.

---

# 🐍 Example Python Lambda Function

## Slack Deploy Command Handler

```python
# Lambda handler — Slack deploy command (Python)

import boto3
import json

codepipeline = boto3.client('codepipeline')


def handler(event, context):
    params = parse_slack_payload(event['body'])

    if not is_authorised(params['user_id']):
        return {
            'statusCode': 403,
            'body': 'Not authorised'
        }

    codepipeline.start_pipeline_execution(
        name=f"{params['service']}-{params['env']}-pipeline",
        variables=[
            {
                'name': 'VERSION',
                'value': params['version']
            }
        ]
    )

    return slack_respond(
        f"Deploy started: {params['service']} {params['version']}"
    )
```

---

# 📚 Best Practices — Automation

## ✅ Automation Principles

### 1. Automate Before Incidents

> Automate the runbook **before an incident**, not during.

Write and test automation when there is sufficient time to validate functionality safely.

---

### 2. Maintain Complete Audit Logging

Every automated action must record:

* 👤 Who triggered the automation
* 🕒 When it was triggered
* ⚙️ What parameters were used

---

### 3. Include Dry-Run Capability

Build a dry-run mode into every automation script.

Benefits:

* Safely test changes
* Validate execution logic
* Reduce production risks

---

### 4. Measure Operational Toil

Use automation investment based on measurable effort.

> If a manual task takes **more than 1 hour per week**, it is worth approximately **1 week of automation investment**.

---

### 5. Define Processes Before Automation

> Never automate an ill-defined process.

Required approach:

1. Document the process.
2. Validate the workflow.
3. Automate the approved procedure.

---

### 6. Use Idempotency as a Core Principle

Every automation script should be safe to execute repeatedly.

**Expected behaviour:**

* First execution applies the required change.
* Subsequent executions confirm the desired state.
* No unexpected side effects occur.

---

# 📌 Lab Summary

| Area                      | Technology          | Outcome                                               |
| ------------------------- | ------------------- | ----------------------------------------------------- |
| Runbook Automation        | Ansible             | Automated operational tasks with idempotent playbooks |
| Secure Automation Storage | Git + Ansible Vault | Controlled and secure automation management           |
| Scheduled Operations      | AWS Systems Manager | Automated recurring maintenance                       |
| ChatOps                   | Slack + Lambda      | Self-service engineering workflows                    |
| Deployment Automation     | AWS CodePipeline    | Automated application deployments                     |
| Database Operations       | Lambda + Slack      | Self-service database snapshots                       |

---

