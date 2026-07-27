---
tags: [observability, prometheus, loki, jaeger, opentelemetry, kubernetes, interview questions, architecture]
---

# A Comprehensive Overview of Frontline Observability Technologies (Prometheus / Loki / Jaeger / OpenTelemetry)

## 🧭 I. What is Observability

Observability is the ability to understand a system's status through data it emits, such as metrics, logs, and traces.

Core objectives:

- Detect issues (Monitoring)
- Locate problems (Debugging)
- Analyze problems (Tracing)
![[Overall Observability System Diagram.png]]

---

## 🎯 II. Three Core Components (Must Be Able to Discuss)

Metrics → Prometheus  
Logs → Loki / ELK  
Traces → Jaeger / Zipkin  

---

## 🟢 III. Prometheus (Metric Monitoring)

### 📌 What It Is

Prometheus is an open-source monitoring system used for collecting, storing, and querying time-series metric data.

---

### 🌐 Official Website

https://prometheus.io/

---

### ⚙️ How to Use It (From an Operations Perspective)

#### 1️⃣ Data Collection

- node-exporter (for hosts)
- kube-state-metrics (for Kubernetes)
- Application-exposed /metrics interfaces

---

#### 2️⃣ Querying (with PromQL)

Example:

cpu_usage > 80

---

#### 3️⃣ Alerts

Using Alertmanager:

- Grouping
- Deduplication
- Inhibition

---

#### 4️⃣ Visualization

- Grafana dashboards

---

### 🎯 One Sentence for Interviews

Prometheus is used to collect system metrics, which are analyzed with PromQL and combined with Alertmanager to generate alerts.

---

---

## 🟡 IV. Loki (Log System)

### 📌 What It Is

Loki is a log system within the Grafana ecosystem. Similar to ELK, but it only indexes tags, resulting in lower costs.

---

### 🌐 Official Website

https://grafana.com/oss/loki/

---

### ⚙️ How to Use It (From an Operations Perspective)

#### 1️⃣ Log Collection Pipeline

Pod → stdout → Promtail → Loki → Grafana

---

#### 2️⃣ Querying

Through Grafana:

- Query by tags
- Filter by time range

---

#### 3️⃣ Features

- No full-text indexing
- Lower storage costs
- Deep integration with Kubernetes

---

### 🎯 One Sentence for Interviews

Loki is used for log collection and querying. Its tag-based indexing reduces storage costs, making it suitable for Kubernetes environments.

---

---

## 🔵 V. Jaeger (Trace Tracking)

### 📌 What It Is

Jaeger is a distributed trace tracking system used to analyze the call paths of requests across multiple services.

---

### 🌐 Official Website

https://www.jaegertracing.io/

---

### ⚙️ How to Use It (From an Operations Perspective)

#### 1️⃣ Data Collection

- Application integration with SDKs
- Or via OpenTelemetry

---

#### 2️⃣ Trace Analysis

Example:

User request  
→ Gateway  
→ Service A  
→ Service B  
→ Database  

This allows you to see:

- Time taken for each step
- Which service is slowest
- Call relationships

---

### 🎯 One Sentence for Interviews

Jaeger is used for trace tracking, enabling the analysis of request call paths and performance bottlenecks.

---

---

## 🔴 VI. OpenTelemetry (Unified Data Collection Standard)

### 📌 What It Is

OpenTelemetry is a unified standard for collecting observability data.

---

### 🌐 Official Website

https://opentelemetry.io/

---

### ⚙️ How to Use It (From an Operations Perspective)

#### Purpose

- Unified collection of metrics, logs, and traces
- Decoupling collection from storage

---

#### Data Flow

Application → OpenTelemetry → Prometheus / Loki / Jaeger

---

### 🎯 One Sentence for Interviews

OpenTelemetry is used for the unified collection of observability data, avoiding multiple separate systems.

---

---

## 🧠 VII. The Relationship Between the Three (Key Interview Questions)

---

Metrics (Prometheus):

- Quickly identify issues (e.g., CPU usage, QPS)

Logs (Loki):

- Provide detailed error information

Traces (Jaeger):

- Analyze request paths and performance bottlenecks

---

👉 Form a closed-loop:

Identify issue → Locate issue → Analyze cause

---

---

## 🏗 VIII. Typical Architecture (Must Be Able to Describe)

Prometheus + Loki + Jaeger + Grafana + Alertmanager

---

---

## ⚠️ IX. Common Interview Questions

---

### 1️⃣ Why Are Three Types of Data Needed?

- Metrics: Lightweight but provide less information
- Logs: Detailed but difficult to aggregate
-