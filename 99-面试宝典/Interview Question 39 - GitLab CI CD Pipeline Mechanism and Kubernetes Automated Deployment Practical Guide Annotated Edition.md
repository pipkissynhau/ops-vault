# Interview Question 39: GitLab CI/CD Pipeline Mechanism and Kubernetes Automated Deployment (Annotated Version)

#cicd #gitlab #devops #pipeline #kubernetes #Interviews

---

## I. Core Summary (Interview One-Liner)

> `.gitlab-ci.yml` itself is the pipeline configuration file for GitLab CI/CD.  
> When code is pushed to GitLab, GitLab reads this file to create a pipeline, which is executed by the runner.  
> In Kubernetes scenarios, the typical workflow is: code commit → build image → push image → update Deployment.

---

## II. Overall Execution Flow (Must Understand)

```text
git push
  ↓
GitLab Test code changes
  ↓
Read .gitlab-ci.yml
  ↓
Create pipeline
  ↓
runner Implementation job
```

---

## III. Core Concepts

### 1️⃣ .gitlab-ci.yml

- Placed in the repository root directory  
- Submitted together with code  
- Used to define the pipeline

---

### 2️⃣ pipeline

- A complete execution process  
- Contains multiple stages

---

### 3️⃣ stage

- Stage (executed in sequence)  
- Example: build → push → deploy

---

### 4️⃣ job

- Task under each stage  
- Example: build-job / deploy-job

---

### 5️⃣ runner

- Component that executes jobs  
- Can be shell / docker / k8s

---

### 6️⃣ Variables

- CI/CD environment variables  
- Used to store sensitive information

Example:

- DOCKER_USER  
- DOCKER_PASS  
- KUBE_CONFIG

---

## IV. Key Understanding (You Must Be Able to Explain)

### ❓ How is a pipeline triggered?

> ❗ Not `.gitlab-ci.yml` triggers

👉 Instead:

- git push  
- merge request  
- tag  
- manual trigger

---

### ❓ What's the relationship between .gitlab-ci.yml and pipeline?

> ✔ `.gitlab-ci.yml` = pipeline definition file

---

### ❓ What are Variables?

> ✔ CI stage environment variables (can store sensitive information)

---

### ❓ What's the difference between Variables and Secret?

| Item | Variables | Secret |
|------|----------|--------|
| Usage Stage | CI/CD | Pod execution |
| Purpose | Build/deployment | Application configuration |

---

## V. Typical K8s Automated Deployment Flow

```text
Code push
  ↓
GitLab CI Trigger pipeline
  ↓
Build Docker Mirror
  ↓
Send mirror warehouse
  ↓
kubectl Update Deployment
```

---

## VI. .gitlab-ci.yml Example (With Comments)

```yaml
# Definitions pipeline Order of the implementation phase
stages:
  - build
  - push
  - deploy

# Global Variables
variables:
  IMAGE_NAME: registry.example.com/myapp   # Mirror repository address
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA          # Use commit id As Version Number

# =========================
# Build Phase
# =========================
build:
  stage: build
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .   # Build mirrors
  only:
    - main   # Only main Branch Trigger

# =========================
# Send mirrors
# =========================
push:
  stage: push
  script:
    # Login mirror repository (variable from GitLab VariablesI'm not sure.
    - echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin registry.example.com

    # Send mirrors
    - docker push $IMAGE_NAME:$IMAGE_TAG
  only:
    - main

# =========================
# Deployed Kubernetes
# =========================
deploy:
  stage: deploy
  script:
    # Writing kubeconfig
    - echo "$KUBE_CONFIG" > kubeconfig

    # Assign kubectl Use this configuration
    - export KUBECONFIG=kubeconfig

    # Update Deployment Mirror
    - kubectl set image deployment/myapp myapp=$IMAGE_NAME:$IMAGE_TAG

    # Waiting for rolling release to complete
    - kubectl rollout status deployment/myapp
  only:
    - main
```

---

## VII. Key Command Explanations

### 1️⃣ docker build

Build image:

```bash
docker build -t image:tag .
```

---

### 2️⃣ docker push

Push image:

```bash
docker push image:tag
```

---

### 3️⃣ kubectl set image

Update Deployment:

```bash
kubectl set image deployment/myapp myapp=image:tag
```

---

### 4️⃣ rollout status

Check release status:

```bash
kubectl rollout status deployment/myapp
```

---

## VIII. Required Variables

```text
DOCKER_USER
DOCKER_PASS
KUBE_CONFIG
```

Path:

```text
GitLab → Settings → CI/CD → Variables
```

---

## IX. Interview Standard Answer (Memorize Directly)

> GitLab CI/CD defines pipelines through `.gitlab-ci.yml`, which are automatically triggered upon code push.  
> Pipelines typically consist of three stages: build, push, and deploy. First, Docker images are built and tagged, then pushed to the image registry, and finally updated via kubectl to Kubernetes Deployment for rolling release.  
> Sensitive information is injected via Variables, not written directly in configuration files.

---

## X. Bonus Points (Must Mention)

### ✔ Why use commit as tag?

- Unique version  
- Easy rollback

---

### ✔ Why use rollout?

- Ensure release success  
- Not just "command execution completed"

---

### ✔ Why not use plaintext passwords?

- Security risks  
- Use Variables instead

---

## XI. Rollback (Interview Bonus)

```bash
kubectl rollout undo deployment/myapp
```

---

## XII. Common Misconceptions

❌ Think `.gitlab-ci.yml` triggers pipeline  
✔ Actually triggered by push

❌ Think Variables are regular variables  
✔ Actually secure environment variables

❌ Confuse CI with runtime  
✔ CI uses Variables, runtime uses Secret

---

## XIII. One-Sentence Summary

> ❗ CI handles building and deployment, Kubernetes handles runtime and rolling updates

---

## XIV. Keyword Mnemonics

- .gitlab-ci.yml  
- pipeline  
- stage / job / script  
- runner  
- variables  
- docker  
- kubectl  
- rollout