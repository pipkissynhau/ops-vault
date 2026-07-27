---
tags:
  - Kubernetes
  - K8s
  - ingress-nginx
  - Grayscale Release
  - Blue/Green Deployment
  - Release Strategy
  - Operations
  - Cloud-Native
---

# Interview Question 40: Notes on Using ingress-nginx for Grayscale and Blue/Green Deployments in K8s

## 1. Conclusion First

In K8s, if you use **ingress-nginx**:

- **Grayscale Release (Canary Release)**
  - Ingress-nginx **natively supports** this.
  - It mainly controls the traffic ratio, Header matching, and Cookie matching through **a second canary Ingress + annotations**.
  - This is suitable for small-scale trials to verify if a new version is stable.

- **Blue/Green Deployment**
  - Ingress-nginx **does not have a dedicated "blue-green deployment switch"**.
  - The common approach is:
    - Deploy two complete versions: blue and green.
    - Finally, switch the backend Service of the Ingress or change the Service selector to achieve **instant switching of all traffic**.
  - This is suitable for scenarios where rapid rollback and clear version switching are required.

---

## 2. Differences Between Grayscale Release and Blue/Green Deployment

| Comparison Item | Grayscale Release | Blue/Green Deployment |
|---|---|---|
| Core Idea | New version handles only a small portion of traffic first | Two environments run in parallel with immediate full-switching |
| Traffic Control | Gradual increase in traffic: 10%, 20%, 50%, 100% | Immediate switch from blue to green |
| Risk | Lower, as observations can be made step by step | Larger impact at the moment of switching |
| Rollback | Adjust weights back to 0 or remove the canary Ingress | Directly switch back to the old version |
| Suitable Scenarios | Cautious online verification | Quick switching and rapid rollback are needed |

---

## 3. Core Principle of Using ingress-nginx for Grayscale Release

The essence of using ingress-nginx for grayscale release is:

1. First, there is a **main Ingress**.
   - All normal traffic goes to the old version service.

2. Then, create a **Canary Ingress**.
   - Its host and path are **the same as those of the main Ingress**.
   - But its backend points to the **new version service**.
   - Relevant annotations are added to control this behavior.

In this way, ingress-nginx will direct a portion of requests to the canary version based on the annotation rules.

---

## 4. Common Control Methods for Grayscale Release

### 4.1 Grayscale by Weight
Most common.

For example:
- 10% of traffic goes to the new version.
- 90% of traffic still uses the old version.

Suitable for:
- Initial small-scale verification.
- Gradual increase in traffic: 10% → 30% → 50% → 100%.

---

### 4.2 Grayscale by Request Header
You can specify a certain Header to direct requests to the new version.

For example:
- Requests from testers should include `Canary: always`.
- Only test traffic will go to the new version.

Suitable for:
- Initial verification by testing or development teams.
- No impact on regular users.

---

### 4.3 Grayscale by Cookie
Requests with a specific Cookie will be directed to the new version.

Suitable for:
- Targeted user groups for grayscale release.
- Only certain authenticated users can access the new version after logging in.

---

## 5. Practical Examples of Grayscale Release

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