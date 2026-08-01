# Interview Question 38: Kubernetes Ingress Online Fault Diagnosis (502 504 Real Scenarios)

#kubernetes #ingress #TheBarrier. #AccidentManagement #Interviews

---

## I. Scenario Description (Interview Question)

A core business service exposed via Ingress suddenly experiences widespread user access failures:

- Returns 502 / 504
- Cluster is normal
- Pod is Running
- Service is normal
- Recently released a new version

---

## II. Core Approach (Interview One-Sentence Answer)

> ❗First stop the bleeding, then troubleshoot along the chain: Ingress → Service → Pod → Application

---

## III. First Step: Stop the Bleeding (Most Important)

👉 Prioritize:

- Roll back the latest release version
- Restore the service

---

🧠 Interview Expression:

> If it's a core business failure, I would first roll back the latest release version to quickly restore the service, then proceed with detailed troubleshooting.

---

---

## IV. Second Step: Determine Error Type

---

### 1️⃣ 502 Bad Gateway

> ❗Backend service is unavailable

---

### 2️⃣ 504 Gateway Timeout

> ❗Backend response timeout

---

---

## V. Third Step: Troubleshoot Along the Chain

---

### 1️⃣ Ingress Layer

```bash
kubectl logs -n ingress-nginx <controller-pod>
```

---

Focus on:

- upstream connect error
- no endpoints
- timeout

---

👉 Key Logs:

```text
no endpoints available
```

---

Explanation:

> ❗Service hasn't selected a Pod

---

---

### 2️⃣ Service / Endpoints (High-Frequency Issue)

```bash
kubectl get svc
kubectl get endpoints <svc-name>
```

---

👉 Judgment:

- Empty Endpoints → selector error
- Has IP → Continue to the next step

---

---

### 3️⃣ Pod Layer (Focus)

```bash
kubectl get pods
kubectl logs <pod>
```

---

Focus on:

- Is it Running
- Are there errors
- Has it restarted

---

👉 Key Judgment:

> ❗Recently released a new version → Prioritize suspicion of the new version

---

---

### 4️⃣ Application Layer (Often Overlooked)

```bash
kubectl exec -it <pod> -- netstat -tlnp
```

---

Or:

```bash
curl localhost:<port>
```

---

👉 Possible Issues:

- Application isn't listening on the port
- Port configuration error
- Health check inaccuracies

---

---

### 5️⃣ Timeout Issues (504)

---

👉 Check:

- Application response time
- ingress-nginx configuration

---

```yaml
proxy-read-timeout
proxy-send-timeout
```

---

---

## VI. Complete Troubleshooting Chain (Must Remember)

```text
Client
  ↓
Ingress Controller
  ↓
Service
  ↓
Endpoints
  ↓
Pod
  ↓
Application
```

---

## VII. Common Root Causes Summary (Interview Focus)

|Issue|Cause|
|---|---|
|502|Pod unavailable / selector error / port error|
|504|Application slow / timeout configuration unreasonable|
|Unable to access|DNS / Ingress configuration error|

---

## VIII. Standard Interview Answer (Memorize Directly)

---

> When encountering this situation, I would first assess the impact scope. For core business issues, I would prioritize rolling back the latest release version to stop the bleeding.
> 
> Then troubleshoot along the chain, starting with Ingress controller logs to check for upstream or no endpoints errors;
> 
> Next verify Service and Endpoints status;
> 
> Since a new version was recently released, I would focus on the status and logs of the new Pod and whether the application is properly listening on the port;
> 
> If it's a 504 error, I would also monitor backend response time and ingress timeout configuration;
> 
> Finally, after restoring the service, I would conduct a post-mortem analysis.

---

## IX. Keyword Mnemonics

- Stop the bleeding (rollback)
- Ingress logs
- Endpoints
- Release issues
- 502 / 504
- Timeout