# Interview Question 48: GitLab CI/CD Quality Gates, Trigger Control, and Execution by Change Scope Notes

## Tags
#CICD #GitLabCI #QualityGateBan #TriggerControl #GitLabRules #GitLabWorkflow #Harbor #MirrorManagement #DevOps #Interview

## I. Topic Overview

This article focuses on three critical issues in production environments:

1. Not everyone can submit code and trigger production release automatically
2. Not every commit deserves to build an image and execute CD
3. CI/CD isn't just automation - it also carries quality gate and release permission control responsibilities

This article emphasizes:

- The role and hierarchy of quality gates
- Who can trigger, which branches can trigger, and which environments can deploy
- How to decide whether to build an image or deploy based on change scope
- Typical usage of GitLab's `workflow:rules`, `rules`, `rules:changes`, manual jobs, protected branches, and protected environments

GitLab officially supports using `workflow:rules` to control whether to create a pipeline, `rules` and `rules:changes` to control job execution, protected branches to restrict who can push/merge main branches, and protected environments/manual jobs to control who can deploy to environments. GitLab also supports using `[ci skip]` / `[skip ci]` to skip pipelines.  
References:
- GitLab `workflow` documentation  
- GitLab job `rules` documentation  
- GitLab protected branches documentation  
- GitLab protected environments documentation  
- GitLab job control / manual jobs documentation

---

## II. Why Quality Gates and Trigger Control Are Essential in Production

Without gates, common issues include:

- Any person can submit code and trigger heavy pipelines, wasting Runner resources
- Minor changes, comment changes, and documentation changes also trigger image building and deployment, low efficiency
- Main branch lacks protection, code risks directly enter release pipeline
- Test environments are frequently updated unnecessarily, affecting debugging and verification
- Production deployment permissions are out of control, anyone can click deploy
- Without quality gates, low-quality code can enter build and release stages

Therefore, CI/CD in production is generally not "automatically execute everything", but:

**Only run pipelines/jobs that are worth executing, and only allow authorized people to trigger critical environment deployments.**

---

## III. Production Gates Can Be Understood as 4 Layers

## 1. Code Gates

Purpose:

- Intercept obviously erroneous code
- Detect quality issues early
- Reduce invalid build/deploy consumption

Common content:

- Lint
- Unit tests
- Basic static checks
- Configuration file validity checks
- Dockerfile validation
- Helm/YAML template checks
- Security scans

The core idea of this layer is:

**Code that doesn't pass gate should not enter subsequent build/deploy stages.**

---

## 2. Merge Gates

Purpose:

- Control who can modify main branch
- Control who can merge changes into main branch
- Prevent unreviewed code from entering release pipeline

Common practices:

- Set main branch as protected branch
- Prohibit arbitrary direct push to main/master
- Merge via Merge Request
- Set approval rules
- Require specified roles to approve before merge

This layer solves:

**Who can send code into main release pipeline.**

---

## 3. Pipeline Gates

Purpose:

- Control which branches run which jobs
- Control which changes trigger builds
- Control which commits only do light checks and no heavy tasks

Common practices:

- Feature branches only run check/test
- Develop branch allows build test image
- Main branch only allows enter deployment pipeline
- Only change docs/README doesn't build image
- Only when `src/`, `Dockerfile`, `helm/`, `deploy/` change do build

This layer solves:

**Not all commits deserve to run full pipeline.**

---

## 4. Deployment Gates

Purpose:

- Control who can deploy to environments
- Control when to deploy
- Control whether production deployment needs manual confirmation

Common practices:

- Set deploy job as manual
- Test environments can be manually triggered
- Production environments bind protected environment
- Only authorized personnel can execute production deployment
- Production environments combine approvals, window periods, and release rules for control

This layer solves:

**Even if pipeline can run, it doesn't mean anyone can deploy environment.**

---

## IV. Why Test Environments Shouldn't Build/Deploy on Every Commit

This is a crucial point in production thinking.

If designed as "build image and deploy test environment on every commit", there will be many issues:

### 1. Waste Resources
- Runner resources are unnecessarily occupied
- Docker build consumes CPU, disk, network
- Harbor image Tag explodes in growth

### 2. Interfere with Debugging and Testing
- Test environment is frequently refreshed
- QA just validates half, environment is overwritten by new commit
- Unstable environment during multi-person debugging

### 3. Minor Changes Don't Warrant Heavy Tasks
For example:

- Only add comments
- Change README
- Change docs
- Change unrelated documentation files

These commits typically don't need to build image, let alone trigger CD.

### 4. Affect Change Control
CI/CD should be designed around "worth-releasing changes", not around "every git push".

Therefore, a more reasonable approach is:

**Lightweight changes only run light checks; build only when core code, Dockerfile, or deployment files change; deploy combined with branch and manual trigger control.**

---

## V. Core Capabilities in GitLab Related to This Topic

## 1. `workflow:rules`

Purpose:

- Decide whether to create pipeline

Suitable scenarios:

- Certain commits don't need to start pipeline
- Skip specific type commits
- Combine commit message to control pipeline generation

Example:

    workflow:
      rules:
        - if: '$CI_COMMIT_MESSAGE =~ /(\[ci skip\]|\[skip ci\])/i'
          when: never
        - when: always

This indicates:

- If commit message contains `[ci skip]` or `[skip ci]`, don't create pipeline
- Otherwise, create pipeline normally

---

## 2. `rules`

Purpose:

- Decide job execution conditions

Suitable scenarios: /think

- Only run build on the main branch  
- Only run tests on merge request pipelines  
- Only run formal release on tag/release pipelines  

Example:  

    build_image:  
      rules:  
        - if: '$CI_COMMIT_BRANCH == "main"'  

---  

## 3. `rules:changes`  

Purpose:  

- Determine whether a job runs based on file change scope  

Suitable scenarios:  

- Run build only if `src/` or `Dockerfile` changes  
- Do not run build if only documentation is modified  
- Run deploy-related checks only if deployment files change  

Example:  

    build_image:  
      rules:  
        - changes:  
            - src/**/*  
            - Dockerfile  
            - helm/**/*  
            - deploy/**/*  
        - when: never  

This indicates:  

- Run build only if these paths change  
- Otherwise, do not execute the build job  

---  

## 4. manual job  

Purpose:  

- Not auto-executed, must be triggered manually  

Suitable scenarios:  

- Deployment to test environment  
- Pre-release environment release  
- Production environment release  
- High-risk operation confirmation  

Example:  

    deploy_test:  
      when: manual  

---  

## 5. protected branches  

Purpose:  

- Restrict who can push or merge to the main branch  

Suitable scenarios:  

- Protection for main/master branches  
- Protection for release branches  
- Prevent arbitrary changes to the main release pipeline  

---  

## 6. protected environments  

Purpose:  

- Restrict who can deploy to a specific environment  

Suitable scenarios:  

- Only authorized personnel can deploy to production environment  
- Specific roles execute deployments for pre-release environment  
- Environment deployment permissions are separated from code permissions  

---  

## Six. Common layered approach in production  

You can simplify the pipeline into two categories:  

## 1. Lightweight pipeline  

Suitable for:  

- Comment modifications  
- README modifications  
- Docs modifications  
- Non-core path changes  

Only run:  

- Basic checks  
- YAML/JSON syntax checks  
- Very lightweight validation  

Goal:  

**As fast as possible, as resource-efficient as possible.**  

---  

## 2. Full pipeline  

Suitable for:  

- Code logic changes  
- Dockerfile changes  
- Dependency changes  
- Helm/deployment file changes  

Can run:  

- Lint  
- Unit tests  
- Build image  
- Push Harbor  
- Deploy test (can be manual)  

Goal:  

**Only allow meaningful changes to enter the heavy pipeline.**  

---  

## Seven. A recommended production model  

### Feature branch  
- Allow frequent commits from developers  
- Mainly run lightweight checks  
- Do not auto-build images  
- Do not auto-deploy  

### Develop branch  
- Can enter test pipeline  
- Build test image when change conditions are met  
- Deploy test can be manual or triggered by rules  

### Main / release branch  
- Protected  
- Merge after MR approval  
- Can enter formal build and release pipeline  
- Production deploy is usually manual + protected environment  

This model is more aligned with production realities than "build + deploy for all branches."  

---  

## Eight. Example 1: Minimal quality gate + build control by change scope  

This example demonstrates:  

- Always run basic checks  
- Build image only if core directories change  
- Documentation and comment changes do not trigger build  

    stages:  
      - check  
      - build  

    workflow:  
      rules:  
        - if: '$CI_COMMIT_MESSAGE =~ /(\[ci skip\]|\[skip ci\])/i'  
          when: never  
        - when: always  

    basic_check:  
      stage: check  
      script:  
        - echo "run basic checks"  
      rules:  
        - when: always  

    build_image:  
      stage: build  
      script:  
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')  
        - IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"  
        - echo "build image with tag: ${IMAGE_TAG}"  
        - docker build -t myapp:${IMAGE_TAG} .  
        - docker push myapp:${IMAGE_TAG}  
      rules:  
        - changes:  
            - src/**/*  
            - Dockerfile  
            - docker/**/*  
            - helm/**/*  
            - deploy/**/*  
        - when: never  

### Explanation  

#### `workflow:rules`  
Controls whether the pipeline is created:  

- If commit message contains `[ci skip]` / `[skip ci]`, the pipeline is not created  
- Otherwise, the pipeline is created  

#### `basic_check`  
Runs in any case, serving as a lightweight gate.  

#### `build_image`  
Builds only if these paths change:  

- `src/**/*`  
- `Dockerfile`  
- `docker/**/*`  
- `helm/**/*`  
- `deploy/**/*`  

If only the following are modified:  

- `README.md`  
- `docs/**/*`  
- Irrelevant comment files  

The build job will not execute.  

---  

## Nine. Example 2: Add test environment deploy, but make it manual  

This example demonstrates:  

- Build only for meaningful changes  
- Only the develop branch enters the test deployment pipeline  
- Deployment to the test environment is also manual  

    stages:  
      - check  
      - build  
      - deploy /think

```markdown
basic_check:
  stage: check
  script:
    - echo "run lint"
    - echo "run unit test"
  rules:
    - when: always

build_image:
  stage: build
  script:
    - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
    - IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
    - echo "IMAGE_TAG=${IMAGE_TAG}"
    - docker build -t myapp:${IMAGE_TAG} .
    - docker push myapp:${IMAGE_TAG}
  rules:
    - changes:
        - src/**/*
        - Dockerfile
        - helm/**/*
        - deploy/**/*
    - when: never

deploy_test:
  stage: deploy
  script:
    - echo "deploy to test"
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      changes:
        - src/**/*
        - Dockerfile
        - helm/**/*
        - deploy/**/*

### Explanation

#### Why `deploy_test` is manual
Because the test environment should not be automatically refreshed with every suitable change.  
A more reasonable approach is:

- First build the image
- Deploy to test manually after confirmation by developers or testers

This is more suitable for collaborative work and debugging.

---

## Ten. Example 3: Adding Merge Gatekeeping and Production Deployment Strategy

The following is not a complete enterprise template, but it can help understand "who can modify the main branch, who can deploy to production":

    stages:
      - check
      - build
      - deploy

    lint_and_test:
      stage: check
      script:
        - echo "run lint"
        - echo "run unit test"
      rules:
        - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

    build_prod_image:
      stage: build
      script:
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
        - IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
        - echo "build prod image: ${IMAGE_TAG}"
      rules:
        - if: '$CI_COMMIT_BRANCH == "main"'
          changes:
            - src/**/*
            - Dockerfile
            - helm/**/*
            - deploy/**/*

    deploy_prod:
      stage: deploy
      script:
        - echo "deploy to production"
      environment:
        name: production
      when: manual
      rules:
        - if: '$CI_COMMIT_BRANCH == "main"'

### The Production Philosophy Demonstrated in This Section

#### `lint_and_test`
Only runs in the Merge Request pipeline, used for quality gatekeeping before merging.

#### `build_prod_image`
Only builds when the main branch has core changes.

#### `deploy_prod`
Not automatic deployment, but:

- `when: manual`
- And should configure `production` as a protected environment

This way, even if the main pipeline can run, not everyone can trigger production deployment.

---

## Eleven. Typical Differences Between Test and Production Environments

## 1. Test Environment
More focused on:

- Validation efficiency
- Stability of debugging
- Not refreshing with every commit

Common strategies:

- Build selectively executed
- Deploy commonly manual
- Develop branch enters the test pipeline
- Feature branches typically don't trigger full deployment

---

## 2. Production Environment
More focused on:

- Permission control
- Approval
- Stable release
- Traceability and rollback capability

Common strategies:

- Protected branch
- MR approval
- Manual deployment
- Protected environment
- Unique image Tag
- Formal release can add immutable Tag

---

## Twelve. What Changes Usually Don't Warrant Building an Image

Common "changes that don't warrant triggering heavy tasks":

- README
- docs directory
- Pure comment modifications
- Irrelevant explanatory files
- Non-service build pipeline-related content

These changes are more suitable for:

- Running lightweight checks
- Or directly skipping CI
- At least don't trigger build and deploy

---

## Thirteen. What Changes Usually Warrant Building an Image

Common "changes worth entering the heavy pipeline":

- Business code directory changes
- Dockerfile changes
- Build script changes
- Dependency file changes
- Helm chart changes
- K8s deployment file changes
- Release configuration changes

Because these changes affect:

- Image content
- Runtime behavior
- Deployment results

---

## Fourteen. Quality Gatekeeping Isn't Just Test, It Also Includes What Else

You can understand quality gatekeeping more comprehensively in interviews.

### 1. Code Quality Gatekeeping
- Lint
- Unit testing
- Code formatting checks

### 2. Build Gatekeeping
- Dockerfile legality
- Whether base dependencies can be installed
- Whether the image can be successfully built
```

### 3. Configuration Access
- YAML/JSON Validity
- Helm Template Rendering
- K8s Manifest Validation

### 4. Security Access
- Static Scanning
- Secret Leak Scanning
- Dependency Vulnerability Scanning
- Image Vulnerability Scanning

### 5. Release Access
- Only Specific Branches Can Be Released
- Only After Approval Can Be Released
- Only Authorized Personnel Can Release

---

## Fifteen. Points Often Overlooked but Important in Production

### 1. Not Everyone Can Trigger Production Release
Code permissions and environment permissions should be separated.

### 2. Not Everyone Can Modify the Main Branch
Protecting the main branch is critical.

### 3. Not All Commits Need Full Pipeline
Distinguish between light and heavy tasks based on change type.

### 4. Test Environment Also Needs Governance
Test environment is not an unrestricted public playground.

### 5. Build and Deploy Should Be Decoupled
Many scenarios are suitable for:

- Automatic Build
- Manual Deploy

### 6. Quality Gates Should Be Frontloaded
Basic issues should not be discovered until deployment.

---

## Sixteen. How to Answer Steadily in Interviews

### Answer Template 1: Quality Gates

Quality gates in CI/CD are not just running a test, but controlling code quality, configuration legality, build availability, and release permissions upfront. Typically, linting, unit tests, and basic checks are run in MR or branch pipelines first. If they fail, the process doesn't proceed to subsequent build and deploy stages.

### Answer Template 2: Why Not Build and Deploy Every Commit

Test environments are generally not designed to automatically build images and trigger deployments for every commit. A more reasonable approach is to differentiate between light and heavy tasks based on change scope. For example, changes to README, docs, or comments only run light checks and don't build images. Only when code directories, Dockerfile, Helm, or deployment files change do we enter the build pipeline. Deployments can also be made manual to avoid frequent refreshes in test environments.

### Answer Template 3: Who Can Trigger Production Release

Production environments usually separate code permissions, pipeline permissions, and environment permissions. Main branches are typically set as protected branches, requiring MR approval before merging. Pipelines then use rules to control which branches can enter the release pipeline. When deploying to production, manual jobs with protected environments are usually used, allowing only authorized personnel to execute.

---

## Seventeen. Simplified Expressions to Memorize

In production CI/CD, not every commit can go through the entire build-to-deploy pipeline, nor is every commit worth building an image. Typically, it's layered control: the first layer is quality gates, deciding whether code continues through linting, testing, and basic checks. The second layer is merge gates, controlling who can enter the main branch through protected branches and MR approvals. The third layer is pipeline gates, using `rules` and `rules:changes` to decide which jobs run based on branch and change scope. The fourth layer is deployment gates, controlling who can actually deploy to test or production environments through manual jobs and protected environments. This saves resources and better aligns with production governance.

---

## Eighteen. Core Keywords to Remember

1. Quality Gates
2. Merge Gates
3. Pipeline Gates
4. Deployment Gates
5. Protected Branch
6. Rules
7. rules:changes
8. Manual Deploy
9. Protected Environment
10. Lightweight Pipeline
11. Full Pipeline
12. Execute by Change Scope

---

## Nineteen. Summary

The essence of this section isn't learning a few GitLab syntaxes, but establishing the right mindset for production:

### 1. Not All Code Deserves Heavy Pipeline
Minor changes shouldn't waste build and deployment resources.

### 2. Not Everyone Can Operate Critical Pipelines
Code permissions, pipeline permissions, and environment permissions should be managed separately.

### 3. CI/CD Isn't Just Automation, It's a Governance System
It simultaneously handles:

- Quality Control
- Resource Control
- Permission Control
- Deployment Control

One-sentence summary:

**CI/CD in production environments, the core isn't "automation," but "conditional automation, bounded automation, and permission-based automation."**