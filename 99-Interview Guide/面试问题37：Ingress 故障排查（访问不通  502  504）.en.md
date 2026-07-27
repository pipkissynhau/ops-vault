# Interview Question 37: Ingress Troubleshooting (Unable to Access / 502 / 504)

#kubernetes #ingress #troubleshooting #nginx #interview

---

## I. Core Summary (One Sentence for the Interview)

The essence of Ingress troubleshooting is:

> ❗Troubleshoot layer by layer along the chain: Client → LB → Ingress → Service → Pod

---

## II. Complete Troubleshooting Chain (Must Know)

```text
Client
  ↓
DNS / Domain Name Resolution
  ↓
Load Balancer (LB)
  ↓
Ingress Controller
  ↓
Service
  ↓
Pod
```

---

## III. Step 1: Domain Name / DNS

---

### Check

```bash
nslookup your-domain.com
```

---

### Notes

- Whether it resolves to the correct IP address.
- Whether it resolves to the LB address.

---

---

## IV. Step 2: Load Balancer (LB)

---

### Check

```bash
curl -v http://your-domain.com
```

---

### Notes

- Whether it is reachable.
- Whether there is a response.

---

---

## V. Step 3: Ingress

---

### View Ingress

```bash
kubectl get ingress
kubectl describe ingress xxx
```

---

### Notes

- Whether the host is correct.
- Whether the path matches.
- Whether the backend is pointing to the correct Service.

---

---

## VI. Step 4: Ingress Controller

---

### View Pods

```bash
kubectl get pods -n ingress-nginx
```

---

### Check Logs (Key)

```bash
kubectl logs -n ingress-nginx <controller-pod>
```

---

### Common Issues

- Configuration errors.
- Upstream unavailable.
- Timeout.

---

---

## VII. Step 5: Service

---

### Check

```bash
kubectl get svc
kubectl describe svc xxx
```

---

### Notes

- Whether the port is correct.
- Whether the selector matches the Pod.

---

---

## VIII. Step 6: Endpoints (Key Point)

---

```bash
kubectl get endpoints xxx
```

---

### If Empty:

> ❗The Service has not selected any Pods.

---

---

## IX. Step 7: Pod

---

### Check

```bash
kubectl get pods
kubectl logs <pod>
```

---

### Notes

- Whether it is Running.
- Whether there are any errors.
- Whether the port is being listened on.

---

---

## X. Common Error Categories (Key Points for the Interview)

---

### 1️⃣ Unable to Access

👉 Possible Reasons:

- DNS error.
- LB unavailable.
- Ingress configuration error.

---

---

### 2️⃣ 502 Bad Gateway

👉 Key Understanding:

> ❗The Ingress cannot find the backend service.

---

Reasons:

- The Pod is unavailable.
- The Service selector is incorrect.
- The port does not match.

---

---

### 3️⃣ 504 Gateway Timeout

👉 Key Understanding:

> ❗The backend response is too slow.

---

Reasons:

- The application is slow.
- The timeout configuration is too low.

---

---

## XI. Additional Points with ingress-nginx Parameters (Bonus)

---

### Solving 504

```yaml
proxy-read-timeout: "120"
proxy-send-timeout: "120"
```

---

---

### Handling Upload Failures (413)

```yaml
proxy-body-size: "50m"
```

---

---

## XII. Standard Interview Answers (Memorize These)

---

> Ingress troubleshooting generally follows the chain from the client to the Pod, layer by layer.
> 
> First, check DNS and domain name resolution, then confirm if the load balancer is reachable;
> 
> Next, verify whether the Ingress configuration, including host and backend, is correct;
> 
> Then examine the logs of the ingress-nginx controller;
> 
> Confirm that the Service and Endpoints are functioning properly;
> 
> Finally, check the Pod status and application logs.
> 
> If it’s a 502 error, it usually indicates a problem with the backend service; if it’s a 504 error, it often means the backend response is timed out.

---

## XIII. Quick Reference Keywords

- DNS
- LB
- Ingress
- Service
- Endpoints
- Pod
- 502 / 504
```