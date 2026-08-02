# Practical Understanding of GitLab CI/CD and .gitlab-ci.yml (For Interviews)

#cicd #gitlab #devops #pipeline #interview

---

## 1. Core Summary (One-Sentence Version for Interviews)

GitLab CI/CD defines a pipeline through the `.gitlab-ci.yml` file in the repository. It is automatically triggered after code is pushed, enabling an automated process of building, testing, and deploying.

---

## 2. What is .gitlab-ci.yml (Key Understanding)

> ❗`.gitlab-ci.yml itself is the configuration file for the pipeline`.

---

### ❗Not Related to:

- ❌ Conflicts with the pipeline

- ❌ Is independent of the pipeline

---

### ✅ Correct Relationship:

```text
.gitlab-ci.yml → Defines the pipeline
pipeline        → Executes based on this file
runner          → Performs specific tasks
```

---

## 3. Where to Place This File (The Key Question You Were Asked)

👉 It must be placed in:

```text
The root directory of the project repository
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

> ✅ Push it to the repository together with the code

---

## 4. How is the Pipeline Triggered (The Key Question You Just Asked)

> ❗It is not triggered only when `.gitlab-ci.yml` is modified

---

### Correct Process:

```text
git push
    ↓
GitLab detects code changes
    ↓
reads .gitlab-ci.yml
    ↓
creates a pipeline
    ↓
runner executes the job
```

---

👉 Key Conclusion:

> ❗A push triggers the pipeline, and `.gitlab-ci.yml` determines how the pipeline is executed

---

## 5. Core Structure of CI/CD (Must Know)

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
|pipeline|A complete execution process in one go|
|stage|A phase that is executed sequentially|
|job|A specific task within a pipeline|
|script|Commands to be executed|
|runner|The entity that performs the tasks|

---

## 6. Variables (The Key Question You Asked)

### Definition

> ❗Variables are environment variables in CI/CD used to pass parameters during pipeline execution

---

### Uses

- Storing passwords, tokens, or private keys

- Holding configuration details (such as environments and addresses)

---

### Example

```yaml
script:
  - docker login -u $DOCKER_USER -p $DOCKER_PASS
```

---

👉 Source of Variables:

```text
GitLab → Settings → CI/CD → Variables
```

---

### Essential Understanding

> ❗Variables serve as “secure environment variables” during the CI phase

---

### Relationship with Kubernetes Secret (Bonus for Interviews)

|Project|Variables|Secret|
|---|---|---|
|Usage Phase|CI Phase|Run Phase|
|Purpose|Building/Deploying|Application Running|

---

👉 In One Sentence:

> Variables are used in CI, while Secrets are used during application runtime

---

## 7. Actual CI/CD Process (Combined with K8s)

```text
Code is pushed
    ↓
GitLab triggers the pipeline
    ↓
Docker image is built
    ↓
Image is pushed to the repository
    ↓
kubectl /helm is used for deployment
```

---

## 8. The Correct Way to Describe “Writing a Pipeline Manually” (Key)

👉 Don't say during an interview:

❌ I don't know how GitLab CI works

---

👉 Correct Statements:

> I have previously written pipeline configurations manually, mainly by defining the steps required for building and deploying.
> 
> In the past, my focus was on practical implementation. Now, I am learning about the overall structure of GitLab CI/CD, such as pipelines, stages, jobs, runners, and variables.

---

## 9. Common Misunderstandings (Pitfalls You Just Encountered)

---

### ❌ Misunderstanding 1: .gitlab-ci.yml Triggers the Pipeline

👉 Wrong

✔ Correct:

> Only a push triggers it; the CI file serves as the rule set

---

### ❌ Misunderstanding 2: The CI File Conflicts with the Pipeline

👉 Wrong

✔ Correct:

> The CI file is the definition of the pipeline itself

---

### ❌ Misunderstanding 3: Variables Are Ordinary Variables

👉 Wrong

✔ Correct:

> They are secure environment variables used for sensitive information

---

## 10. Standard Interview Answers (Memorize These)

---

> GitLab CI/CD defines a