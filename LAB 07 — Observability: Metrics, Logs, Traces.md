# LAB 07 — Observability: Metrics, Logs, Traces

## Datadog / Prometheus / Grafana 

# 📘 Objective

Implement a comprehensive observability strategy for Kubernetes-hosted applications using **Datadog**, **Prometheus**, and **Grafana**.

This lab focuses on building a production-grade observability platform by implementing:

* Service Level Indicators (SLIs)
* Service Level Objectives (SLOs)
* Error budgets
* Metrics collection
* Dashboards
* Alerting
* Distributed tracing
* Operational runbooks

---

# 📖 Learning Outcomes

Upon completing this lab, you will be able to:

* Define meaningful SLIs and SLOs for production services.
* Deploy and configure the Prometheus monitoring stack.
* Build Grafana dashboards using the USE methodology.
* Monitor SLO error budget consumption.
* Configure Alertmanager routing and alert escalation.
* Implement actionable alerts with linked operational runbooks.
* Apply observability best practices for production environments.

---

# 🎯 Task 1 — Define SLOs and SLIs

## Overview

Before instrumenting any application or infrastructure, define what **"good"** means for every service.

Service Level Indicators (SLIs) measure service performance, while Service Level Objectives (SLOs) define acceptable reliability targets.

---

## Service Reliability Targets

| Service          | SLI Metric                           | SLO Target | Error Budget (30 Days) |
| ---------------- | ------------------------------------ | ---------: | ---------------------: |
| Magento Checkout | Success rate of `/checkout` POST     |  **99.9%** |       **43.2 minutes** |
| Magento Catalog  | p99 latency < **800 ms**             |  **99.5%** |          **3.6 hours** |
| API Gateway      | HTTP 5xx rate < **0.1%**             |  **99.9%** |       **43.2 minutes** |
| Background Jobs  | Job completion within **10 minutes** |  **99.0%** |          **7.2 hours** |

> 💡 **Note:** Error budgets define the amount of acceptable service degradation within a rolling 30-day period before reliability work should take priority over feature delivery.

---

# 📊 Task 2 — Prometheus + Grafana Stack Setup

## Overview

Deploy a complete Kubernetes monitoring stack capable of collecting infrastructure and application metrics, visualizing performance, and monitoring SLO compliance.

---

## Step 1 — Deploy kube-prometheus-stack

Deploy the **kube-prometheus-stack** using Helm.

The stack includes:

| Component     | Purpose                        |
| ------------- | ------------------------------ |
| Prometheus    | Metrics collection and storage |
| Grafana       | Dashboards and visualization   |
| Alertmanager  | Alert routing and notification |
| node-exporter | Node-level metrics collection  |

---

## Step 2 — Configure Prometheus Service Discovery

Configure Prometheus scrape configurations for all Kubernetes services using **ServiceMonitor** Custom Resource Definitions (CRDs).

### Requirements

* Discover Kubernetes services automatically.
* Scrape application metrics.
* Standardize monitoring across workloads.

---

## Step 3 — Monitor Magento PHP

Install the Magento PHP exporter or use the Datadog APM Agent.

Monitor:

* PHP-FPM metrics
* PHP OPcache metrics
* PHP application performance

Supported options:

* Magento PHP Exporter
* Datadog APM Agent

---

## Step 4 — Build Grafana Dashboards

Create Grafana dashboards covering the **USE methodology** (Utilization, Saturation, Errors).

### Required Dashboard Metrics

| Metric       | Description                       |
| ------------ | --------------------------------- |
| Request Rate | Incoming application requests     |
| Error Rate   | Failed requests                   |
| p50 Latency  | Median response time              |
| p95 Latency  | High-percentile response time     |
| p99 Latency  | Worst-case user experience        |
| Saturation   | Resource utilization and capacity |

Recommended dashboard layout:

```text
┌──────────────────────────────────────────────┐
│ Request Rate                                 │
├──────────────────────────────────────────────┤
│ Error Rate                                   │
├──────────────────────────────────────────────┤
│ p50 │ p95 │ p99 Latency                      │
├──────────────────────────────────────────────┤
│ CPU │ Memory │ Saturation │ Queue Length     │
└──────────────────────────────────────────────┘
```

---

## Step 5 — Import the SLO Dashboard

Create an SLO dashboard displaying:

* Current service availability
* Remaining error budget
* Error budget burn rate

Configure burn-rate monitoring for:

* **1-hour burn rate**
* **6-hour burn rate**

These views provide both rapid and long-term visibility into service reliability.

---

## PromQL Examples

### Checkout Success Rate

```promql
sum(rate(http_requests_total{
job="magento", handler="/checkout", status=~"2.."}[1h]))
/
sum(rate(http_requests_total{job="magento", handler="/checkout"}[1h]))
```

---

### p99 Latency

```promql
histogram_quantile(
  0.99,
  sum(
    rate(http_request_duration_seconds_bucket{job="magento"}[5m])
  ) by (le)
)
```

---

# 🚨 Task 3 — Alerting Strategy

## Overview

Implement a production-ready alerting strategy that prioritizes incidents based on severity while minimizing unnecessary alert noise.

---

## Step 1 — Configure Alertmanager Routing

Configure Alertmanager to route alerts based on priority.

| Priority | Action                                      |
| -------- | ------------------------------------------- |
| **P1**   | Page the on-call engineer using PagerDuty   |
| **P2**   | Send notification to Slack **#alerts-prod** |
| **P3**   | Create a Jira ticket                        |

---

## Step 2 — Create Alert Rules

Implement alert rules for the following production conditions.

| Alert                                                     | Severity |
| --------------------------------------------------------- | -------- |
| Error budget burn rate > **2×**                           | P1       |
| Pod crash-looping more than **3 times within 15 minutes** | P1       |
| Disk utilization greater than **85%**                     | P2       |

These alerts should provide actionable information and include sufficient context for responders.

---

## Step 3 — Configure Alert Inhibition

Implement alert inhibition to reduce duplicate notifications.

### Objective

Suppress downstream alerts when an upstream service is already in a failed state.

Benefits include:

* Preventing alert storms
* Reducing alert fatigue
* Improving incident focus

Example:

```text
Database Down
      │
      ▼
Suppress:
    • API Errors
    • Checkout Errors
    • Worker Errors
```

---

## Step 4 — Create Operational Runbooks

Create a runbook for every **P1** and **P2** alert.

Each alert annotation should include:

* Runbook URL
* Incident response steps
* Validation procedures
* Recovery guidance

> 💡 Every operational alert should guide engineers directly to the documented remediation process.

---

# 📝 Best Practices — Observability

## 🎯 Alert on Symptoms, Not Causes

Focus alerts on user impact rather than infrastructure conditions.

**Preferred**

* Checkout error rate is high

**Avoid**

* CPU utilization is high

---

## 📈 Monitor the Four Golden Signals

Begin every monitoring strategy with the **Google SRE Four Golden Signals**:

* Latency
* Traffic
* Errors
* Saturation

These metrics provide a comprehensive view of service health and user experience.

---

## 📚 Every Alert Must Have a Runbook

Every operational alert should include an associated runbook.

> An alert without a runbook trains teams to ignore it.

---

## 🔥 Configure SLO Burn Rate Alerts

Configure both long-term and short-term burn-rate alerts.

| Burn Rate | Time Window | Purpose                             |
| --------- | ----------- | ----------------------------------- |
| **2×**    | 1 hour      | Detect slow reliability degradation |
| **5×**    | 5 minutes   | Detect rapid service degradation    |

Using multiple burn-rate windows improves incident detection without increasing false positives.

---

## 🪵 Use Structured Logging

Adopt structured JSON logging across all applications and services.

Benefits include:

* Faster searching
* Easier filtering
* Improved log aggregation
* Better analytics at scale

Avoid relying on unstructured free-text logs.

---

## 🔍 Implement Distributed Tracing

Enable distributed tracing using one of the following:

* Datadog APM
* Jaeger

Distributed tracing enables correlation between:

* Requests
* Logs
* Metrics
* Microservices

This significantly improves root cause analysis during production incidents.

---

# ✅ Lab Completion Checklist

| Task                                   | Status |
| -------------------------------------- | ------ |
| SLOs and SLIs defined for all services | ☐      |
| Error budgets documented               | ☐      |
| kube-prometheus-stack deployed         | ☐      |
| ServiceMonitor CRDs configured         | ☐      |
| Magento PHP metrics exported           | ☐      |
| Grafana dashboards created             | ☐      |
| SLO dashboard implemented              | ☐      |
| PromQL queries validated               | ☐      |
| Alertmanager routing configured        | ☐      |
| Production alert rules implemented     | ☐      |
| Alert inhibition configured            | ☐      |
| Runbooks created for all P1/P2 alerts  | ☐      |
| Structured logging implemented         | ☐      |
| Distributed tracing enabled            | ☐      |

---

