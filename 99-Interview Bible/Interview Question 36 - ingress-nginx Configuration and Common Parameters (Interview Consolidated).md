# Interview Question 36: ingress-nginx Configuration System and Common Parameter Details (Interview Integration Version)

#kubernetes #ingress #nginx #configmap #annotation #Interviews

---

## One. Core Summary (Interview One-Sentence)

In ingress-nginx:

- **Global configuration → ConfigMap**
    
- **Single business configuration → Annotation**
    
- **Production environment → Recommended to manage via Helm values**
    

---

## Two. Core Essence (What is the interviewer testing?)

> ❗How to modify ingress-nginx behavior while avoiding impact on other businesses

---

## Three. Three Ways to Modify Configuration (Must Master)

---

### 1️⃣ ConfigMap (Global Configuration)

👉 Modify ingress-nginx controller behavior

```yaml
data:
  proxy-body-size: "20m"
  proxy-read-timeout: "60"
```

---

#### Features

|Feature|Description|
|---|---|
|Scope|Global|
|Risk|❗Affects all Ingress|
|Use Case|Unified configuration|

---

### 2️⃣ Annotation (Recommended)

👉 Modify individual Ingress

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
```

---

#### Features

|Feature|Description|
|---|---|
|Scope|Single business|
|Risk|Low|
|Use Case|Customization needs|

---

### 3️⃣ Helm values (Production Recommended)

```yaml
controller:
  config:
    proxy-body-size: "20m"
```

---

👉 Then:

```bash
helm upgrade
```

---

#### Features

|Feature|Description|
|---|---|
|Maintainability|⭐⭐⭐⭐⭐|
|Recommendation|Production best practice|

---

## Four. Relationship Between the Three (Interview Focus)

```text
Helm values → Generate ConfigMap → controller Entry into force
Annotation → Overwrite Single Ingress
```

---

## Five. Priority (Critical)

> ❗Annotation has higher priority than ConfigMap

---

## Six. Common Configuration Parameter Details (Must Know)

---

### 1️⃣ proxy-body-size (Request Body Size)

```yaml
proxy-body-size: "20m"
```

👉 Control upload size

---

📌 Scenario:

- Upload error: 413

---

🧠 One-Sentence Summary:

> Limit client request body size

---

---

### 2️⃣ proxy-read-timeout (Backend Response Timeout)

```yaml
proxy-read-timeout: "60"
```

👉 Waiting time for backend response

---

📌 Scenario:

- Request gets stuck
    
- Export/slow API endpoints

---

🧠 One-Sentence Summary:

> Backend response timeout control

---

### 3️⃣ proxy-send-timeout (Request Send Timeout)

```yaml
proxy-send-timeout: "60"
```

👉 Time limit for sending requests to backend

---

🧠 One-Sentence Summary:

> Request send timeout control

---

### 4️⃣ worker-processes (Concurrency Capability)

```yaml
worker-processes: "auto"
```

👉 Control worker count

---

🧠 One-Sentence Summary:

> Control Nginx concurrency capability

---

### 5️⃣ use-forwarded-headers (Real IP)

```yaml
use-forwarded-headers: "true"
```

👉 Obtain real client IP

---

📌 Scenario:

- LB / CDN scenarios

---

🧠 One-Sentence Summary:

> Obtain real client IP

---

### 6️⃣ server-tokens (Security)

```yaml
server-tokens: "false"
```

👉 Whether to expose Nginx version

---

🧠 One-Sentence Summary:

> Hide version information to enhance security

---

## Seven. Troubleshooting Approach (Practical)

---

### 1️⃣ Find ConfigMap

```bash
kubectl get configmap -n ingress-nginx
```

---

### 2️⃣ Check which ConfigMap the controller uses

```bash
kubectl get deploy -n ingress-nginx ingress-nginx-controller -o yaml
```

---

### 3️⃣ Check configuration content

```bash
kubectl get configmap -n ingress-n
```