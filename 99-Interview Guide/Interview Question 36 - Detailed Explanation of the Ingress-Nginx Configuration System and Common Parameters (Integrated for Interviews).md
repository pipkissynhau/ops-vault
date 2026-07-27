# Interview Question 36: Detailed Explanation of the Ingress-Nginx Configuration System and Common Parameters (Integrated for Interviews)

#kubernetes #ingress #nginx #configmap #annotation #interview

---

## I. Core Summary (One Sentence for Interviews)

In ingress-nginx:

- **Global configuration → ConfigMap**
    
- **Configuration for individual services → Annotation**
    
- **For production environments → It is recommended to use Helm values for management**
    

---

## II. The Essence of the Question (What Interviewers Want to Test)

> ❗How to modify the behavior of ingress-nginx without affecting other services

---

## III. Three Ways to Modify Configurations (Must Be Mastered)

---

### 1️⃣ ConfigMap (Global Configuration)

👉 Modifies the behavior of the ingress-nginx controller

```yaml
data:
  proxy-body-size: "20m"
  proxy-read-timeout: "60"
```

---

#### Characteristics

|Characteristic|Description|
|---|---|
|Scope of Effect|Global|
|Risk|❗Affects all Ingresses|
|Use Case|Uniform configuration|

---

### 2️⃣ Annotation (Recommended)

👉 Modifies a single Ingress

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
```

---

#### Characteristics

|Characteristic|Description|
|---|---|
|Scope of Effect|Individual service|
|Risk|Low|
|Use Case|Customization requirements|

---

### 3️⃣ Helm values (Recommended for Production)

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

#### Characteristics

|Characteristic|Description|
|---|---|
|Maintainability|⭐⭐⭐⭐⭐|
|Recommendation Level|Best practice for production|

---

## IV. The Relationship Among the Three (Key Point for Interviews)

```text
Helm values → Generate ConfigMap → Take effect on the controller
Annotation → Applies to a single Ingress|
```

---

## V. Priority (Critical)

> ❗Annotations have higher priority than ConfigMaps

---

## VI. Detailed Explanation of Common Configuration Parameters (Must Be Familiar With)

---

### 1️⃣ proxy-body-size (Request Body Size)

```yaml
proxy-body-size: "20m"
```

👉 Controls the size of the uploaded request body

---

📌 Scenarios:

- Errors occur when uploading files: 413

---

🧠 In One Sentence:

> Limits the size of the client's request body

---

---

### 2️⃣ proxy-read-timeout (Backend Response Timeout)

```yaml
proxy-read-timeout: "60"
```

👉 Controls the waiting time for backend responses

---

📌 Scenarios:

- Requests get stuck

- For slow or export-heavy APIs

---

🧠 In One Sentence:

> Controls the timeout for backend responses

---

### 3️⃣ proxy-send-timeout (Request Sending Timeout)

```yaml
proxy-send-timeout: "60"
```

👉 Sets a time limit for sending requests to the backend

---

🧠 In One Sentence:

> Controls the timeout when sending requests to the backend

---

### 4️⃣ worker-processes (Concurrent Capacity)

```yaml
worker-processes: "auto"
```

👉 Controls the number of worker processes

---

🧠 In One Sentence:

> Determines the concurrent processing capability of Nginx

---

### 5️⃣ use-forwarded-headers (True IP)

```yaml
use-forwarded-headers: "true"
```

👉 Allows access to the real client IP address

---

📌 Scenarios:

- In load balancing or CDN scenarios

---

🧠 In One Sentence:

> Provides the real client IP for processing

---

### 6️⃣ server-tokens (Security)

```yaml
server-tokens: "false"
```

👉 Determines whether to expose Nginx version information

---

🧠 In One Sentence:

> Hides the version number to enhance security

---

## VII. Troubleshooting Steps (Practical Application)

---

### 1️⃣ Locate the ConfigMap

```bash
kubectl get configmap -n ingress-nginx
```

---

### 2️⃣ Check which ConfigMap is being used by the controller

```bash
kubectl get deploy -n ingress-nginx ingress-nginx-controller -o yaml
```

---

### 3️⃣ View the configuration content

```bash
kubectl get configmap -n ingress-n
```