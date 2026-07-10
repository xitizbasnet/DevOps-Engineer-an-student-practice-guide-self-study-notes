# LAB 06 — Magento Platform Stability

## PHP / Nginx / Redis / Varnish 

# 📘 Objective

Take ownership of the Magento e-commerce hosting platform by implementing:

* Performance optimization
* Security hardening
* Zero-downtime deployment practices
* Application stability improvements
* Production-grade caching strategies

This lab focuses on managing and optimizing the Magento application stack:

| Component | Purpose                      |
| --------- | ---------------------------- |
| PHP-FPM   | Application execution layer  |
| Nginx     | Web server and reverse proxy |
| Redis     | Cache and session storage    |
| Varnish   | Full-page HTTP caching layer |

---

# 🚀 Task 1 — Zero-Downtime Deployment Strategy

## Overview

Implement a deployment process that allows Magento releases to be delivered without application downtime.

The deployment strategy should:

* Maintain application availability
* Reduce deployment risk
* Support fast rollback
* Validate application health before switching traffic

---

## Step 1 — Implement Rolling Deployment

Adopt a rolling deployment strategy for Magento.

### Deployment Requirements

* Deploy changes to one server at a time.
* Drain active connections before stopping or replacing instances.
* Maintain service availability throughout deployment.

---

## Step 2 — Configure Magento Maintenance Mode

Magento Maintenance Mode should only be enabled for:

* Database schema changes
* Required maintenance operations

### Maintenance Mode Requirements

Configure:

* Maintenance IP allowlist
* QA verification access before production traffic switch

Example workflow:

```text
Enable Maintenance Mode
        |
        |
Allow QA IP Access
        |
        |
Validate Application
        |
        |
Continue Deployment
```

---

## Step 3 — Run Magento Build Commands Before Traffic Switching

Execute deployment preparation commands before directing traffic to the new release.

Required commands:

```bash
php bin/magento setup:upgrade --keep-generated

php bin/magento setup:di:compile

php bin/magento setup:static-content:deploy -f
```

### Deployment Rule

> Run Magento upgrade and compilation tasks before switching traffic, never after.

---

## Step 4 — Implement Pre-Deployment Smoke Testing

Before switching production traffic:

* Test critical Magento pages.
* Validate HTTP response codes.
* Abort deployment if validation fails.

Example validation targets:

```text
/
 /catalog
 /checkout
```

### Deployment Condition

| Response         | Action              |
| ---------------- | ------------------- |
| HTTP 200         | Continue deployment |
| Non-200 response | Abort deployment    |

---

## Step 5 — Maintain Previous Release for Rollback

Keep previous application releases available for immediate rollback.

Use symbolic links for release management:

```bash
ln -sfn releases/v1.2.3 current
```

### Release Structure Example

```text
/var/www/
|
├── releases/
│   ├── v1.2.2
│   └── v1.2.3
|
└── current -> releases/v1.2.3
```

---

# 🛠️ Zero-Downtime Deployment Script (Bash)

```bash
RELEASE_DIR=/var/www/releases/$(date +%Y%m%d_%H%M%S)

rsync -az --delete build/ $RELEASE_DIR/

cd $RELEASE_DIR && php bin/magento setup:upgrade --keep-generated

php bin/magento setup:di:compile

php bin/magento setup:static-content:deploy -f

# Smoke test

for URL in / /catalog /checkout; do
STATUS=$(curl -s -o /dev/null -w '%{http_code}' http://localhost$URL)

[ "$STATUS" != "200" ] && echo "SMOKE FAIL $URL" && exit 1

done

ln -sfn $RELEASE_DIR /var/www/current

php bin/magento cache:flush
```

---

# ⚡ Task 2 — Performance Tuning (PHP-FPM, Redis, Varnish)

## Overview

Optimize Magento performance by tuning:

* PHP execution resources
* Application caching
* Session storage
* HTTP caching
* Static content delivery

---

# Step 1 — Configure PHP-FPM Pool

Configure PHP-FPM using production-appropriate worker settings.

### Required Configuration

```ini
; php-fpm pool config (/etc/php/8.3/fpm/pool.d/magento.conf)

pm = dynamic

pm.max_children = 50

pm.start_servers = 10

pm.min_spare_servers = 5

pm.max_spare_servers = 20

pm.max_requests = 500
```

### Configuration Purpose

| Setting            | Purpose                                 |
| ------------------ | --------------------------------------- |
| `pm = dynamic`     | Dynamically manages PHP workers         |
| `pm.max_children`  | Limits maximum concurrent PHP processes |
| `pm.start_servers` | Defines initial worker count            |
| `pm.max_requests`  | Restarts workers after request limit    |

---

# Step 2 — Enable PHP OPcache

Enable PHP OPcache to improve application execution performance.

## PHP.ini OPcache Settings

```ini
; php.ini opcache settings

opcache.enable = 1

opcache.memory_consumption = 512

opcache.max_accelerated_files = 100000

opcache.validate_timestamps = 0 ; disable in production
```

### Production Recommendation

Disable timestamp validation in production environments to avoid unnecessary filesystem checks.

---

# Step 3 — Configure Redis for Magento

Enable Redis for:

* Magento cache backend
* Magento session storage

### Redis Requirements

Use separate Redis instances:

| Redis Instance | Purpose                   |
| -------------- | ------------------------- |
| Redis Cache    | Magento application cache |
| Redis Session  | Customer session storage  |

> ⚠️ Avoid using a single Redis instance for both cache and sessions in production environments.

---

# Step 4 — Configure Varnish Full Page Cache

Deploy:

* Varnish 7.x
* Nginx
* Magento Full Page Cache integration

Architecture:

```text
User Request
      |
      v
 Varnish 7.x
      |
      v
   Nginx
      |
      v
 PHP-FPM
      |
      v
 Magento
```

---

## Varnish Requirements

Configure Varnish to:

* Cache full-page HTML responses
* Serve anonymous user traffic
* Improve response times

Target:

```text
90%+ Cache Hit Rate
```

---

# Step 5 — Enable Nginx FastCGI Cache

Configure Nginx FastCGI caching as a fallback mechanism.

Purpose:

* Maintain performance when Varnish is unavailable.
* Reduce PHP-FPM workload.

---

# Step 6 — Configure Magento Full Page Cache

Enable Magento Full Page Cache using:

* Varnish as the backend cache service
* Cache warming after deployments

Deployment workflow:

```text
Deploy Release
      |
      v
Clear Cache
      |
      v
Warm Magento Pages
      |
      v
Serve Cached Content
```

---

# 📝 Best Practices — Magento Platform

## 🚫 Avoid Static Content Deployment During Peak Hours

Never run:

```bash
setup:static-content:deploy
```

during peak traffic periods.

### Recommended Approach

* Pre-generate static assets.
* Deploy prepared assets during release windows.

---

## 📊 Monitor Varnish Cache Performance

Use:

```bash
varnishstat
```

Monitor:

* Cache hits
* Cache misses
* Cache invalidation events

### Performance Target

```text
Cache Hit Rate: 85%+
```

If the ratio falls below this level:

* Investigate cache invalidation.
* Review Magento cache configuration.
* Validate Varnish rules.

---

## 🔴 Use Redis High Availability

Production Redis deployments should use:

* Redis Sentinel
* Redis Cluster

Reason:

> A single Redis instance represents a Single Point of Failure (SPOF).

---

## ⏱️ Separate Magento Cron Workloads

Do not run Magento cron jobs on PHP-FPM application servers.

Recommended architecture:

```text
Magento Application Servers
          |
          |
Dedicated Cron Worker Instance
```

Benefits:

* Prevents background jobs from affecting web traffic.
* Improves application stability.

---

## 🔍 Monitor Magento Indexers

Regularly run:

```bash
bin/magento indexer:status
```

Purpose:

* Detect stuck indexers.
* Prevent stale catalog data.
* Maintain search and product accuracy.

---

## 📈 Enable Application Performance Monitoring

Enable Magento monitoring integrations:

Recommended tools:

* New Relic Magento integration
* Datadog APM for PHP

Purpose:

* Trace slow transactions.
* Identify application bottlenecks.
* Monitor PHP performance.

---

# ✅ Lab Completion Checklist

| Task                                          | Status |
| --------------------------------------------- | ------ |
| Rolling deployment strategy implemented       | ☐      |
| Magento maintenance workflow configured       | ☐      |
| Pre-deployment smoke tests implemented        | ☐      |
| Release rollback strategy configured          | ☐      |
| PHP-FPM optimized                             | ☐      |
| OPcache enabled                               | ☐      |
| Redis cache and session separation configured | ☐      |
| Varnish full-page caching enabled             | ☐      |
| Nginx FastCGI fallback configured             | ☐      |
| Magento cache warming configured              | ☐      |
| Redis HA strategy implemented                 | ☐      |
| Dedicated Magento cron worker configured      | ☐      |
| APM monitoring enabled                        | ☐      |

---

