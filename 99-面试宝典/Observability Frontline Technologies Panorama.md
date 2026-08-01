---
tags: "[observability, prometheus, loki, jaeger, opentelemetry, kubernetes, Interview High Frequency, Architecture]"
---

# Observability Frontline Technical Panorama (Prometheus / Loki / Jaeger / OpenTelemetry)

## 🧭 1. What is Observability

Observability is the ability to understand the state of a system through its output data (metrics, logs, traces).

Core objectives:

- Detect issues (Monitoring)
- Locate issues (Debugging)
- Analyze issues (Tracing)
![[General view of the observing system.png]]

---

## 🎯 2. Three Core Components (Must Know)

Metrics → Prometheus  
Logs → Loki / ELK  
Traces → Jaeger / Zipkin  

---

## 🟢 3. Prometheus (Metrics Monitoring)

### 📌 What is it

Prometheus is an open-source monitoring system used for collecting, storing, and querying time-series metric data.

---

### 🌐 Official Website

https://prometheus.io/

---

### ⚙️ How to Use (Operations Perspective)

#### 1️⃣ Data Collection

- node-exporter (Host)
- kube-state-metrics (K8s)
- Application exposes /metrics interface

---

#### 2️⃣ Querying (PromQL)

Example:

cpu_usage > 80

---

#### 3️⃣ Alerting

Through Alertmanager:

- Grouping (grouping)
- Deduplication (deduplication)
- Inhibition (inhibition)

---

#### 4️⃣ Display

- Grafana dashboard

---

### 🎯 Interview One-Liner

Prometheus is used to collect system metrics, analyze them via PromQL, and implement alerts through Alertmanager.

---

---

## 🟡 4. Loki (Log System)

### 📌 What is it

Loki is a log system in the Grafana ecosystem, similar to ELK but indexes only labels, with lower cost.

---

### 🌐 Official Website

https://grafana.com/oss/loki/

---

### ⚙️ How to Use (Operations Perspective)

#### 1️⃣ Log Collection Pipeline

Pod → stdout → Promtail → Loki → Grafana

---

#### 2️⃣ Querying

Through Grafana:

- Query by label
- Filter by time range

---

#### 3️⃣ Features

- No full-text indexing
- Low storage cost
- Deep integration with Kubernetes

---

### 🎯 Interview One-Liner

Loki is used for log collection and querying, reducing storage costs via label indexing, suitable for Kubernetes environments.

---

---

## 🔵 5. Jaeger (Tracing System)

### 📌 What is it

Jaeger is a distributed tracing system used to analyze request paths across multiple services.

---

### 🌐 Official Website

https://www.jaegertracing.io/

---

### ⚙️ How to Use (Operations Perspective)

#### 1️⃣ Data Collection

- Application integrates SDK
- Or via OpenTelemetry

---

#### 2️⃣ Tracing Analysis

Example:

User request  
→ Gateway  
→ Service A  
→ Service B  
→ Database  

Can see:

- Time spent at each step
- Which service is slow
- Call relationships

---

### 🎯 Interview One-Liner

Jaeger is used for tracing, analyzing request paths and performance bottlenecks.

---

---

## 🔴 6. OpenTelemetry (Unified Collection Standard)

### 📌 What is it

OpenTelemetry is a unified standard for observability data collection.

---

### 🌐 Official Website

https://opentelemetry.io/

---

### ⚙️ How to Use (Operations Perspective)

#### Function

- Unified collection of metrics, logs, traces
- Decouple collection from storage

---

#### Data Flow

Application → OpenTelemetry → Prometheus / Loki / Jaeger

---

### 🎯 Interview One-Liner

OpenTelemetry is used to unify observability data collection, avoiding multiple collection systems.

---

---

## 🧠 7. Relationship Between the Three (Core Interview Question)

---

Metrics (Prometheus):

- Quickly detect issues (CPU, QPS)

Logs (Loki):

- View detailed error information

Traces (Jaeger):

- Analyze request paths and performance bottlenecks

---

👉 Form a closed loop:

Detect issues → Locate issues → Analyze root causes

---

---

## 🏗 8. Typical Architecture (Must Know)

Prometheus + Loki + Jaeger + Grafana + Alertmanager

---

---

## ⚠️ 9. High-Frequency Interview Questions

---

### 1️⃣ Why are three types of data needed?

- Metrics: Lightweight but limited information  
- Logs: Detailed but hard to aggregate  
- Traces: Locate call issues  

---

### 2️⃣ Loki vs ELK

- Loki: Lightweight, low cost, Kubernetes-friendly  
- ELK: Powerful features but high resource consumption  

---

### 3️⃣ What is the role of OpenTelemetry

- Unified collection  
- Decouple systems  
- Support multiple backends  

---

---

## 🎯 10. Final Summary (Recommended to Memorize)

---

The observability system consists of three parts: metrics, logs, and traces:

- Prometheus handles metrics monitoring  
- Loki handles log collection  
- Jaeger handles trace tracking  

Through OpenTelemetry for unified collection, it achieves a complete closed loop from detecting issues to locating them.