# Interview Question 39: GitLab CI/CD Pipeline Mechanism and Practical Experience with Kubernetes Automated Deployment (Commented Version)

#cicd #gitlab #devops #pipeline #kubernetes #interview

---

## I. Core Summary (One Sentence for the Interview)

> `.gitlab-ci.yml` itself is the pipeline configuration file for GitLab CI/CD.  
> When code is pushed to GitLab, it reads this file, creates a pipeline, and the runner executes the respective jobs.  
> In a Kubernetes context, the typical process is: code submission → image building → image pushing → Deployment update.

---

## II. Overall Execution Flow (Must Be Understood)

```text
git push
  ↓
GitLab detects code changes
  ↓
reads .gitlab-ci.yml
  ↓
creates pipeline
  ↓
runner executes jobs
```

---

## III. Core Concepts

### 1️⃣ .gitlab-ci.yml

- Located in the repository root directory
- Submitted along with the code
- Defines the pipeline configuration

---

### 2️⃣ Pipeline

- A complete execution sequence
- Comprises multiple stages

---

### 3️⃣ Stage

- A sequential phase of the pipeline
- Example: build → push → deploy

---

### 4️⃣ Job

- A task within a stage
- Example: build-job / deploy-job

---

### 5️⃣ Runner

- The component that executes jobs
- Can be shell, docker, or k8s

---

### 6️⃣ Variables

- CI/CD environment variables
- Used to store sensitive information

Examples:

- DOCKER_USER
- DOCKER_PASS
- KUBE_CONFIG

---

## IV. Key Concepts You Must Be Able to Explain

### ❓ How is a pipeline triggered?

> ❗ It's not triggered by `.gitlab-ci.yml` itself

👉 Instead, it is triggered by:

- git push
- merge request
- tag creation
- Manual initiation

---

### ❓ What is the relationship between `.gitlab-ci.yml` and a pipeline?

> ✔ `.gitlab-ci.yml` is the file that defines the pipeline configuration

---

### ❓ What are Variables?

> ✔ They are environment variables used during CI phases (can store sensitive data)

---

### ❓ What is the difference between Variables and Secrets?

| Item         | Variables    | Secrets       |
|-----------------|--------------|---------------------------|
| Purpose      | CI/CD pipeline | Application runtime configuration |
| Usage Stage  | During pipeline execution | Within Pods at runtime     |
| Security    | Store sensitive info   | Store highly confidential data |

---

## V. Typical Kubernetes Automated Deployment Process

```text
Code is pushed
  ↓
GitLab triggers the pipeline
  ↓
Docker image is built
  ↓
Image is pushed to the repository
  ↓
kubectl updates the Deployment
```

---

## VI. Example of `.gitlab-ci.yml` (with Comments)

```yaml
# Define the sequence of pipeline stages
stages:
  - build
  - push
  - deploy

# Global variables
variables:
  IMAGE_NAME: registry.example.com/myapp   # Image repository address
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA          # Use commit ID as version tag

# =========================
# Build stage
# =================--------
build:
  stage: build
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_tag .   # Build the Docker image
  only:
    - main   # This stage is triggered only on the main branch

# ===========================
# Push the image
# =================--------
push:
  stage: push
  script:
    - echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin registry.example.com
    - docker push $IMAGE_NAME:$IMAGE_TAG
  only:
    - main

# =========================
# Deploy to Kubernetes
# =================--------
deploy:
  stage: deploy
  script:
    - echo "$KUBE_CONFIG" > kubeconfig
    - export KUBECONFIG=kubeconfig
    - kubectl set image deployment/myapp myapp=$IMAGE_NAME:$IMAGE_TAG
    - kubectl rollout status deployment/myapp
  only:
    - main
```

---

## VII. Explanation of Key Commands

### 1️⃣ docker build

Builds a Docker image:

```bash
docker build -t image:tag .
```

---

### 2️⃣ docker push

Pushes a Docker image to the repository:

```bash
docker push image:tag
```

---

### 3️⃣ kubectl set image

Updates the Deployment image:

```bash
kubectl set image deployment/myapp myapp=image:tag
```

---

### 4️⃣ rollout status

 Checks the deployment status:

```bash
kubectl rollout status deployment/myapp
```

---

## VIII. Variables to Be Configured

```text