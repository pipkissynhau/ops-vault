---
tags:
  - Kubernetes
  - K8s
  - ingress-nginx
  - Greyscale Release
  - Blue Green Launch
  - Publishing Policy
  - Transport
  - Clouds.
---

# Interview Question 40: Notes on Controlling Canary and Blue/Green Deployments with K8s ingress-nginx

## 1. Let's Summarize First

In K8s, if you're using **ingress-nginx**:

- **Canary Release**
  - **Natively supported**
  - Mainly controlled via **second canary Ingress + annotations** to manage traffic ratio, Header matching, Cookie matching
  - Suitable for small-scale trial runs, verifying new version stability

- **Blue/Green Deployment**
  - **No dedicated blue/green deployment switch** in ingress-nginx
  - General approach:
    - Deploy two complete versions: blue / green
    - Finally switch **Ingress backend Service** or **switch Service selector**
    - Achieve "instant full traffic switch"
  - Suitable for scenarios requiring quick rollback, clean version switching

---

## 2. Differences Between Canary and Blue/Green Deployments

| Comparison Item | Canary Release | Blue/Green Deployment |
|---|---|---|
| Core Idea | New version first receives a small portion of traffic | Two environments run in parallel, full traffic switch |
| Traffic Control | Gradually increase to 10%, 20%, 50%, 100% | One-time switch from blue to green |
| Risk | Lower, can gradually observe | Significant impact at switch moment |
| Rollback | Adjust weight back to 0 or delete canary Ingress | Directly switch back to old version |
| Suitable Scenarios | Online cautious verification | Need for quick switching, quick rollback |

---

## 3. Core Principle of Canary Deployment with ingress-nginx

The essence of canary deployment with ingress-nginx is:

1. First have a **main Ingress**
   - All normal traffic goes to old version service

2. Then create a **canary Ingress**
   - host, path same as main Ingress
   - But backend points to **new version service**
   - Plus canary-specific annotations

Ingress-nginx will then route part of the requests to the canary version based on annotations.

---

## 4. Common Canary Deployment Control Methods

### 4.1 Weight-based Canary
Most common.

Example:
- 10% traffic goes to new version
- 90% traffic still goes to old version

Suitable for:
- Initial small-scale verification
- Gradual scaling: 10% → 30% → 50% → 100%

---

### 4.2 Header-based Canary
Specify a Header to route to new version.

Example:
- Test requests with `Canary: always`
- Only test traffic enters new version

Suitable for:
- Test team, R&D team verification
- No impact on regular users

---

### 4.3 Cookie-based Canary
Requests with a specific Cookie go to new version.

Suitable for:
- Targeted user group canary
- Only allow part of logged-in users to access new version

---

## 5. Canary Deployment Practical Example

---

### 5.1 Old Version Deployment and Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
        - name: app
          image: myapp:v1
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-stable
spec:
  selector:
    app: myapp
    version: v1
  ports:
    - port: 80
      targetPort: 80
```