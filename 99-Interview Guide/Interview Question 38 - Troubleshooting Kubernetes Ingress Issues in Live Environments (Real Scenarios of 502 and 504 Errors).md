# Interview Question 38: Troubleshooting Kubernetes Ingress Issues in Live Environments (Real Scenarios of 502 and 504 Errors)

#kubernetes #ingress #troubleshooting #incident handling #interview

---

## I. Scenario Description (Interview Question)

The core online service is exposed through Ingress, but suddenly a large number of users experience access failures:

- Errors code 502 or 504 are returned.

- The cluster is functioning normally.

- All Pods are running.

- The Service is also operating correctly.

- A new version was just released recently.

---

## II. Key Approach (One Sentence for the Interview)

> ❗First, stop the bleeding, then trace the issue step by step: Ingress → Service → Pod → Application

---

## III. Step 1: Stopping the Bleeding (Most Important)

👉 Prioritize:

- Rolling back the recently released version.

- Restoring the Service.

---

🧠 Interview Response:

> If it's a core service issue, I would first roll back the latest release to quickly restore service availability before conducting a detailed investigation.

---

---

## IV. Step 2: Identifying the Error Type

---

### 1️⃣ 502 Bad Gateway

> ❗The backend service is unavailable.

---

### 2️⃣ 504 Gateway Timeout

> ❗The backend response has timed out.

---

---

## V. Step 3: Tracing the Issue Step by Step

---

### 1️⃣ Ingress Layer

```bash
kubectl logs -n ingress-nginx <controller-pod>
```

---

Pay attention to:

- upstream connect error

- no endpoints available

- timeout

---

👉 Key log message:

```text
no endpoints available
```

---

Explanation:

> ❗The Service has not selected any Pods.

---

---

### 2️⃣ Service / Endpoints (Frequently Occurring Issues)

```bash
kubectl get svc
kubectl get endpoints <svc-name>
```

---

👉 Check:

- If Endpoints are empty → The selector is incorrect.

- If there are IP addresses → Proceed to the next step.

---

---

### 3️⃣ Pod Layer (Critical)

```bash
kubectl get pods
kubectl logs <pod>
```

---

Pay attention to:

- Whether the Pod is running.

- Any error messages.

- Whether it has been restarted.

---

👉 Key consideration:

> ❗If a new version was just released, suspect it first.

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

👉 Possible issues:

- The application is not listening on the specified port.

- There are configuration errors with the port settings.

- The health checks are inaccurate.

---

---

### 5️⃣ Timeout Issues (504)

---

👉 Check:

- The application's response time.

- The ingress-nginx configuration settings.

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

## VII. Common Root Causes Summary (Key Points for the Interview)

|Error|Cause|
|---|---|
|502|The Pod is unavailable / The selector is incorrect / The port setting is wrong|
|504|The application responds slowly / The timeout configuration is inappropriate|
|Inaccessibility|DNS configuration errors / Ingress configuration issues|

---

## VIII. Standard Interview Answer (Memorized)

---

> When encountering such a situation, I would first assess the impact range. If it involves a core service, I would prioritize rolling back the latest release to stabilize the system.
> 
> Then, I would trace the issue step by step, starting with the Ingress controller logs to check for any upstream connection errors or missing endpoints;
> 
> Next, I would verify if the Service and Endpoints are functioning correctly;
> 
> Since a new version was recently released, I would focus on checking the status and logs of the new Pods, as well as whether the application is listening on the correct ports;
> 
> In cases of 504 errors, I would also examine the backend response time and the ingress configuration settings for timeouts;
> 
> Finally, after restoring service availability, I would conduct a post-troubleshooting analysis.

---

## IX. Key Terms to Remember

- Stopping the bleeding (Rollback)

- Ingress logs

- Endpoints

- Release-related issues

- 502 / 504 errors

- Timeout settings