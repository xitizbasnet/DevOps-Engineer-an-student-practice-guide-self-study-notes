# LAB 05 — Kubernetes (EKS / GKE) Operations 

## Kubernetes / Helm / ArgoCD

# 📘 Objective

Provision a production-grade Kubernetes environment using **Amazon Elastic Kubernetes Service (EKS)**, deploy workloads using **Helm**, implement **GitOps workflows with ArgoCD**, and configure automated scaling mechanisms.

This lab covers:

* Kubernetes cluster provisioning
* EKS infrastructure automation using Terraform
* Helm-based application deployment
* GitOps implementation using ArgoCD
* Kubernetes workload autoscaling
* Production Kubernetes best practices

---

# ⚙️ Task 1 — EKS Cluster Provisioning with eksctl / Terraform

## Overview

Provision a production EKS cluster using infrastructure-as-code practices. The cluster will include dedicated node groups for system components and application workloads.

---

## Step 1 — Provision EKS Cluster

Provision an EKS cluster using the:

* `terraform-aws-modules/eks` Terraform module
* Kubernetes version **1.29+**

The cluster should follow production-ready configuration standards.

---

## Step 2 — Create EKS Node Groups

Create two managed node groups:

| Node Group | Instance Type | Capacity Type          | Purpose                     |
| ---------- | ------------- | ---------------------- | --------------------------- |
| `system`   | `t3.medium`   | On-Demand              | Core Kubernetes system pods |
| `app`      | `m5.large`    | Mixed Spot + On-Demand | Application workloads       |

### Node Group Requirements

* The **system** node group should host critical Kubernetes services.
* The **app** node group should run application workloads.
* Use mixed capacity strategies to optimize workload cost and availability.

---

## Step 3 — Enable EKS Managed Add-ons

Enable the following Amazon EKS managed add-ons:

| Add-on         | Purpose                                           |
| -------------- | ------------------------------------------------- |
| CoreDNS        | Kubernetes service discovery                      |
| kube-proxy     | Network communication between Kubernetes services |
| VPC CNI        | AWS-native Kubernetes networking                  |
| EBS CSI Driver | Persistent volume management using Amazon EBS     |

---

## Step 4 — Install AWS Load Balancer Controller

Install the AWS Load Balancer Controller using Helm.

The controller enables Kubernetes workloads to manage AWS Application Load Balancer (ALB) resources through Ingress objects.

### Install AWS Load Balancer Controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=prod-cluster \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller
```

---

## Step 5 — Configure IRSA (IAM Roles for Service Accounts)

Configure **IRSA** to securely assign AWS IAM permissions to Kubernetes workloads.

### Requirements

* Assign IAM roles directly to Kubernetes service accounts.
* Pods must use IAM roles through Service Accounts.
* Do not use node-level EC2 instance profiles for application permissions.

> ⚠️ **Security Recommendation:**
> Never grant AWS permissions at the node level when workloads require specific AWS access. Use IRSA for least-privilege access control.

---

# 🚀 Task 2 — GitOps with ArgoCD

## Overview

Implement GitOps deployment practices using ArgoCD.

The Kubernetes cluster state should be managed declaratively through a dedicated Git repository.

---

## Step 1 — Install ArgoCD

Install ArgoCD into the dedicated namespace:

```text
argocd
```

Configure access through:

* AWS Application Load Balancer (ALB) Ingress
* TLS encryption

---

## Step 2 — Create GitOps Repository

Create a dedicated Git repository for Kubernetes deployment manifests.

### GitOps Repository Pattern

Application source code and Kubernetes deployment configuration should remain separate.

Example:

```text
Application Repository
        |
        |
        +-- Application Code


GitOps Repository
        |
        |
        +-- Kubernetes Manifests
        +-- Helm Values
        +-- Deployment Configuration
```

---

## Step 3 — Define ArgoCD Application Resources

Create ArgoCD Application Custom Resources (CRDs) that reference Helm charts stored in the GitOps repository.

### ArgoCD Application Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: magento
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/company/k8s-gitops
    targetRevision: HEAD
    path: apps/magento

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Step 4 — Enable Auto-Sync and Self-Healing

Enable ArgoCD automated synchronization.

Configuration:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### Expected Behavior

* ArgoCD continuously monitors the Git repository.
* Manual `kubectl` changes are detected.
* The cluster automatically returns to the desired Git-defined state.

---

## Step 5 — Configure ArgoCD Image Updater

Install and configure **ArgoCD Image Updater**.

Purpose:

* Monitor container images stored in Amazon Elastic Container Registry (ECR).
* Detect newly published image versions.
* Automatically update image tags in the GitOps repository.

---

# 📈 Task 3 — Autoscaling (HPA + KEDA + Cluster Autoscaler)

## Overview

Configure Kubernetes autoscaling capabilities at both workload and infrastructure levels.

---

## Step 1 — Configure Horizontal Pod Autoscaler (HPA)

Install Metrics Server and configure HPA for the Magento deployment.

### Configuration Requirement

* CPU utilization target: **60%**

The HPA should automatically increase or decrease pod replicas based on CPU usage.

---

## Step 2 — Configure KEDA Autoscaling

Install KEDA and create a `ScaledObject`.

### Scaling Trigger

Scale background job workers based on:

* Amazon SQS queue depth

Example workload:

```text
Magento Background Job Workers
        |
        |
        +-- SQS Queue Depth
                |
                |
                +-- KEDA Scaling Decision
```

---

## Step 3 — Configure Cluster Autoscaler

Install and configure either:

* Cluster Autoscaler
  **or**
* Karpenter

Purpose:

* Automatically add EC2 nodes when pods cannot be scheduled.
* Remove unused nodes when capacity is no longer required.

---

## Step 4 — Configure PodDisruptionBudgets (PDB)

Configure PodDisruptionBudgets for all production deployments.

### Requirement

During node maintenance or draining operations:

* At least **1 pod must remain available**.

Example:

```yaml
minAvailable: 1
```

---

# 📝 Kubernetes Best Practices

## Recommended Production Practices

### ✅ Define Resource Requests and Limits

Always define:

* CPU requests
* CPU limits
* Memory requests
* Memory limits

> Without resource definitions, pods may consume excessive node resources and impact cluster stability.

---

### 🔒 Use Namespace Isolation and Network Policies

Implement:

* Dedicated namespaces for workloads
* Kubernetes NetworkPolicies
* Default-deny inter-namespace traffic rules

Recommended approach:

```text
Default Policy:
Deny All Traffic

Explicit Rules:
Allow Required Communication Only
```

---

### 🛡️ Do Not Run Containers as Root

Use:

* Kubernetes Pod Security Standards
* Restricted security profile
* Cluster-wide enforcement policies

---

### 📦 Use Helm Values Per Environment

Maintain:

* One Helm chart
* Multiple environment-specific values files

Example:

```text
helm-chart/
|
├── values-dev.yaml
├── values-test.yaml
├── values-staging.yaml
└── values-production.yaml
```

---

### 📊 Enable Kubernetes Audit Logging

Enable audit logging on the EKS control plane.

Recommended destination:

```text
Amazon CloudWatch
```

Purpose:

* Security monitoring
* Compliance auditing
* Troubleshooting

---

### 🔍 Kubernetes Diagnostic Command

Regularly run:

```bash
kubectl get events --sort-by=timestamp
```

Purpose:

* Quickly identify cluster issues
* Review recent Kubernetes events
* Perform first-pass troubleshooting

---

# ✅ Lab Completion Checklist

| Task                                       | Status |
| ------------------------------------------ | ------ |
| Production EKS cluster provisioned         | ☐      |
| Kubernetes 1.29+ configured                | ☐      |
| System and application node groups created | ☐      |
| EKS managed add-ons enabled                | ☐      |
| AWS Load Balancer Controller installed     | ☐      |
| IRSA configured                            | ☐      |
| ArgoCD installed and configured            | ☐      |
| GitOps repository created                  | ☐      |
| Auto-sync and self-healing enabled         | ☐      |
| ArgoCD Image Updater configured            | ☐      |
| HPA configured                             | ☐      |
| KEDA configured                            | ☐      |
| Cluster Autoscaler/Karpenter configured    | ☐      |
| Production PDBs configured                 | ☐      |
| Kubernetes security best practices applied | ☐      |


---

