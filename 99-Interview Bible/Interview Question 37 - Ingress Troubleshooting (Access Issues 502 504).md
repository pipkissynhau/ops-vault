# Interview Question 37: Ingress Troubleshooting (Access Failure / 502 / 504)

#kubernetes #ingress #TheBarrier. #nginx #Interviews

---

## I. Core Summary (One-Sentence Interview Answer)

Ingress troubleshooting fundamentally involves:

> ❗Step-by-step troubleshooting along the chain: Client → LB → Ingress → Service → Pod

---

## II. Complete Troubleshooting Chain (Must Know)

```text
Client
  ↓
DNS / Domain Name Resolution
  ↓
Load BalanceLBI'm not sure.
  ↓
Ingress Controller
  ↓
Service
  ↓
Pod
```

---

## III. Step 1: Domain / DNS

---

### Check

```bash
nslookup your-domain.com
```

---

### Focus

- Whether it resolves to the correct IP
    
- Whether it resolves to the LB address
    

---

---

## IV. Step 2: Load Balancer (LB)

---

### Check

```bash
curl -v http://your-domain.com
```

---

### Focus

- Whether it is reachable
    
- Whether it responds
    

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

### Focus

- Whether the host is correct
    
- Whether the path matches
    
- Whether the backend points to the correct service
    

---

---

## VI. Step 4: Ingress Controller

---

### View Pod

```bash
kubectl get pods -n ingress-nginx
```

---

### View Logs (Critical)

```bash
kubectl logs -n ingress-nginx <controller-pod>
```

---

### Common Issues

- Configuration errors
    
- Upstream unreachable
    
- Timeout
    

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

### Focus

- Whether the port is correct
    
- Whether the selector matches the Pod
    

---

---

## VIII. Step 6: Endpoints (Critical Point)

---

```bash
kubectl get endpoints xxx
```

---

### If Empty:

> ❗Service has not selected any Pod

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

### Focus

- Whether it is Running
    
- Whether it has errors
    
- Whether the ports are listening
    

---

---

## X. Common Error Classification (Interview Focus)

---

### 1️⃣ Access Failure

👉 Possible Causes:

- DNS error
    
- LB unreachable
    
- Ingress configuration error
    

---

---

### 2️⃣ 502 Bad Gateway

👉 Key Understanding:

> ❗Ingress cannot find the backend service

---

Causes:

- Pod is unavailable
    
- Service selector is wrong
    
- Port mismatch
    

---

---

### 3️⃣ 504 Gateway Timeout

👉 Key Understanding:

> ❗Backend response is too slow

---

Causes:

- Application is slow
    
- Timeout configuration is too small
    

---

---

## XI. Combining ingress-nginx Parameters (Bonus)

---

### 504 Resolution

```yaml
proxy-read-timeout: "120"
proxy-send-timeout: "120"
```

---

---

### Upload Failure (413)

```yaml
proxy-body-size: "50m"
```

---

---

## XII. Standard Interview Answer (Memorize Directly)

---

> Ingress troubleshooting generally follows the chain from client to Pod, step-by-step.
> 
> First check DNS and domain resolution, then confirm LB reachability;
> 
> Next verify Ingress resource configuration, including host and backend;
> 
> Then check ingress-nginx controller logs;
> 
> Confirm Service and Endpoints are normal;
> 
> Finally troubleshoot Pod status and application logs.
> 
> If it's 502, it's usually a backend service issue; if it's 504, it's typically a backend response timeout.

---

## XIII. Keyword Mnemonics

- DNS
    
- LB
    
- Ingress
    
- Service
    
- Endpoints
    
- Pod
    
- 502 / 504
    

---