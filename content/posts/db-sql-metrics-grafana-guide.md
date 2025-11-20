---
title: "Monitoring Database Connection Pools in Grafana"
date: 2025-11-20T16:47:37-06:00
tags: ["go", "golang", "sql", "prometheus", "grafana", "database", "monitoring"]
categories: ["database", "monitoring"]
series: ["database monitoring"]
ShowToc: true
TocOpen: true
weight: 3
---

*Practical Guidance for Engineers Using Prometheus + Go’s `database/sql` Metrics*

Most application performance issues in database-backed systems aren’t caused by MySQL itself — they come from **connection pool pressure**, **slow acquisition**, or **misconfigured limits**.  
This post is a practical guide to understanding the key Prometheus **`db_sql_*`** metrics, why they matter, and how to visualize them effectively in Grafana.

---

## 1. What Are `db_sql_*` Metrics?

When using Go’s `database/sql` along with Prometheus instrumentation, your service emits a standardized set of metrics that describe your *client-side* connection pool behavior.  
These cover:

- **Connection churn**
  - `db_sql_connection_closed_max_idle_time_total`
  - `db_sql_connection_closed_max_idle_total`
  - `db_sql_connection_closed_max_lifetime_total`
- **Pool sizing**
  - `db_sql_connection_max_open`
  - `db_sql_connection_open`
- **Wait pressure**
  - `db_sql_connection_wait_total`
  - `db_sql_connection_wait_duration_milliseconds_total`
- **Query latency**
  - `db_sql_latency_milliseconds_bucket`
  - `db_sql_latency_milliseconds_count`
  - `db_sql_latency_milliseconds_sum`

These metrics answer one big question:

> **Is my service struggling to obtain or use database connections?**

They are emitted **per application instance**, not per MySQL user or global cluster.

---

## 2. Why These Metrics Matter

### Connection pools behave differently than most developers expect

A typical Go service might have:
- 10–50 worker goroutines  
- But only 5–20 DB connections available

If the pool is misconfigured or overloaded, you see:
- slow endpoints,  
- timeouts,  
- spikes in p99 latency,  
- cascading failures.

Understanding these metrics helps detect issues before users feel them.

---

## 3. Key Metrics Explained (Human-Friendly)

### 🔹 **1. `db_sql_connection_open`**  
How many connections are currently open (idle + in use).

This is your **actual concurrency**, not your max capacity.

**PromQL example:**
```promql
sum(db_sql_connection_open{app="example-service"}) by (cluster)
```

---

### 🔹 **2. `db_sql_connection_max_open`**  
The configured maximum number of open connections.  
Useful to detect:
- misconfigurations across replicas,
- unexpected restarts that reset config,
- clusters hitting pool exhaustion.

```promql
max(db_sql_connection_max_open{app="example-service"}) by (cluster)
```

---

### 🔹 **3. Connection Pool Saturation (%)**  
How close the pool is to being full:

```promql
sum(db_sql_connection_open{app="example-service"}) 
/
clamp_min(
  max_over_time(db_sql_connection_max_open{app="example-service"}[5m]),
1
)
```

If this exceeds **80–85%**, requests may start blocking.

---

### 🔹 **4. Headroom**  
How many free connections remain *right now*:

```promql
max(db_sql_connection_max_open{app="example-service"})
-
sum(db_sql_connection_open{app="example-service"})
```

This is fantastic for dashboards.

---

### 🔹 **5. Waits & Wait Duration**  
Waits (`db_sql_connection_wait_total`) mean your code requested a connection but none were available.

Latency to obtain a connection:

```promql
sum(rate(db_sql_connection_wait_duration_milliseconds_total[5m]))
/
clamp_min(sum(rate(db_sql_connection_wait_total[5m])), 1)
```

If this number rises → pool limits are too low or queries are too slow.

---

### 🔹 **6. Connection Closes**  
These three counters tell **why connections are closing**:

- lifetime expiration → `closed_max_lifetime_total`
- idle timeout → `closed_max_idle_time_total`
- pool size compression → `closed_max_idle_total`

Unexpected spikes here imply:
- misconfigured GC behavior,
- pool flapping,
- database restarts.

---

### 🔹 **7. Query Latency Histogram**  
From p50 to p99:

```promql
histogram_quantile(0.90, sum(rate(db_sql_latency_milliseconds_bucket[5m])) by (le))
```

This helps diagnose:
- slow queries,
- DB CPU saturation,
- lock contention,
- network latency.

---

## 4. Client-Side vs Server-Side Metrics (Important!)

Your Go service emits **client-side pool metrics**.

MySQL itself emits **server-side user statistics**, such as:

```promql
mysql_info_schema_user_statistics_concurrent_connections
```

These represent **global server-level behavior** per MySQL user.

| Metric Type | Source | Scope | Example |
|-------------|--------|-------|---------|
| `db_sql_*` | Application | One service instance’s pool | App waiting for connection |
| `mysql_info_schema_*` | DB Server | Global across all apps | Total concurrent connections per MySQL user |

You need **both** to get the whole picture.

---

## 5. Building a Practical Grafana Dashboard

A good dashboard contains:

### **1. Open connections (per cluster)**  
Trend line showing how many are actively in use.

### **2. Pool Saturation %**  
Shows when you’re about to run into waits.

### **3. Avg Wait to Acquire**  
Detects bottlenecks caused by small pools or slow queries.

### **4. Connection Closes / Minute**  
Identifies connection churn patterns.

### **5. Query Latency Panels (p50/p90/p99)**  
Lets you correlate latency spikes with pool pressure.

### **6. Headroom Table**  
Shows pool capacity left *right now*.

### **7. (Optional) Per-MySQL User Concurrency**  
Using server-exporter metrics:

```promql
sum(max_over_time(mysql_info_schema_user_statistics_concurrent_connections[5m])) by (user, cluster)
```

---

## 6. Why Some Metrics Appear Flat or Always Zero

This is extremely common.

Possible causes:

### ✔ Metric not labeled the same way as others  
If `app`, `cluster`, or `region` labels don’t exist on a metric, your filters return 0.

### ✔ Metric only increments on specific events  
Closing connections or waits happen only when the pool is under pressure.

### ✔ Low traffic environment  
Idle pools often look “flatline zero” because nothing is being stressed.

### ✔ Misconfigured pool  
If your max open is higher than needed, waits never occur.

### ✔ Incorrect joins or regex filters  
Especially with custom MySQL users or cluster labels.

---

## 7. Why You May Not See Drops After Updating DB Limits

This is one of the biggest surprises for teams.

Reasons include:

- The metric is **server-side** (MySQL) while you changed **client-side** limits.
- Concurrency is driven by **traffic patterns**, not just configuration.
- Long-running connections remain open until idle-lifetime or max-lifetime.
- Other services may still be using the same MySQL user.
- Using sum/max_over_time hides instantaneous dips.

---

## 8. Final Takeaways

- **Pool saturation** is the most important metric for app health.  
- **Wait duration** reveals when requests are blocked on the pool.  
- **Headroom** gives you a simple “capacity remaining” number.  
- **Latency histograms** help correlate MySQL behavior with app stress.  
- **Client metrics ≠ server metrics** — both matter.  
- Grafana variables are global, but you can emulate panel-specific filters.  

A well-built dashboard will help you quickly answer:

- *“Do we have enough connections?”*  
- *“Are we saturating the pool?”*  
- *“Are slow queries causing waits?”*  
- *“Is MySQL under pressure per user?”*  

---

Written by **Ruben Meza**  
Published on **soyunpollo.dev**

