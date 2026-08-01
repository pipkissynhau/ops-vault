# GitLab CI/CD and .gitlab-ci.yml Practical Understanding (Interview Edition)

#cicd #gitlab #devops #pipeline #Interviews

---

## One. Core Summary (Interview One-Sentence Version)

GitLab CI/CD defines pipeline through the `.gitlab-ci.yml` file in the repository, which is automatically triggered when code is pushed, achieving automation of build, test, and deployment processes.

---

## Two. What is .gitlab-ci.yml (Key Understanding)

> ❗`.gitlab-ci.yml It is. pipeline Profile`

---

### ❗Not Related:

- ❌ Conflicts with pipeline
    
- ❌ Independent of pipeline
    

---

### ✅ Correct Relationship:

```text
.gitlab-ci.yml → Definitions pipeline
pipeline        → By this document
runner          → Carry out specific tasks
```

---

## Three. Where is this file located (Your Focus Point)

👉 Must be placed:

```text
Project repository root directory
```

Example:

```text
project/
├── src/
├── Dockerfile
├── README.md
└── .gitlab-ci.yml
```

---

👉 Conclusion:

> ✅ Pushed together with code to the repository

---

## Four. How is pipeline triggered (Your Just Asked Focus Point)

> ❗Not triggered by ".gitlab-ci.yml being modified"

---

### Correct Process:

```text
git push
    ↓
GitLab Code changes detected
    ↓
Read .gitlab-ci.yml
    ↓
Create pipeline
    ↓
runner Implementation job
```

---

👉 Key Conclusion:

> ❗Pipeline triggered by push, .gitlab-ci.yml determines how pipeline executes

---

## Five. CI/CD Core Structure (Must Know)

```yaml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "build"
```

---

### Core Concepts:

|Concept|Explanation|
|---|---|
|pipeline|Complete execution process|
|stage|Stage (sequential execution)|
|job|Task|
|script|Execution commands|
|runner|Executor|

---

## Six. Variables (Your Focus Point)

### Definition

> ❗Variables are environment variables in CI/CD used to pass parameters during pipeline execution

---

### Usage

- Store passwords / Tokens / private keys
    
- Store configurations (environment, addresses)
    

---

### Example

```yaml
script:
  - docker login -u $DOCKER_USER -p $DOCKER_PASS
```

---

👉 Variable Sources:

```text
GitLab → Settings → CI/CD → Variables
```

---

### Essential Understanding

> ❗Variables = "Secure environment variables" in CI stages

---

### Relationship with Kubernetes Secret (Interview Bonus)

|Project|Variables|Secret|
|---|---|---|
|Usage Stage|CI stage|Runtime stage|
|Purpose|Build/deployment|Application runtime|

---

👉 One-Sentence Summary:

> CI uses Variables, runtime uses Secret

---

## Seven. Actual CI/CD Process (Combined with K8s)

```text
Code push
    ↓
GitLab Trigger pipeline
    ↓
Build Docker Mirror
    ↓
Send mirror warehouse
    ↓
kubectl / helm Deployment
```

---

## Eight. Correct Expression of "Handwritten Pipeline" (Focus Point)

👉 Do not say in interview:

❌ I don't know GitLab CI

---

👉 Correct Statement:

> I previously wrote pipeline configurations manually, mainly based on actual release processes to define execution steps, such as building and deployment.
> 
> Previously I focused more on usage and implementation, now I'm strengthening my understanding of GitLab CI/CD's overall structure, such as pipeline, stage, job, runner, variables, etc.

---

## Nine. Common Misconceptions (Your Just Fallen Pit)

---

### ❌ Misconception 1: .gitlab-ci.yml triggers pipeline

👉 Incorrect

✔ Correct:

> Triggered by push, CI file is just the rules

---

### ❌ Misconception 2: CI file conflicts with pipeline

👉 Incorrect

✔ Correct:

> CI file is the pipeline definition

---

### ❌ Misconception 3: Variables are ordinary variables

👉 Incorrect

✔ Correct:

> Are secure environment variables (for sensitive information)

---

## Ten. Standard Interview Answer (Memorize Directly)

---

> GitLab CI/CD defines pipeline through the .gitlab-ci.yml file in the repository.
> 
> When code is pushed, GitLab reads this file to create and execute pipeline based on defined stages, jobs, and script, with runner handling task execution.
> 
> .gitlab-ci.yml is typically placed in the project root directory and committed with code.
> 
> Sensitivity information can also be managed through Variables, used as environment variables in pipeline.

---

## Eleven. Keyword Quick Recall

- .gitlab-ci.yml
    
- pipeline
    
- stage / job / script
    
- runner
    
- push trigger
    
- variables
    
- automatic deployment

---