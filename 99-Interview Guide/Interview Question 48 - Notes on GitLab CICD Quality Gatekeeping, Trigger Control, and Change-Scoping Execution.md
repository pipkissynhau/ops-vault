# Interview Question 48: Notes on GitLab CICD Quality Gatekeeping, Trigger Control, and Change-Scoping Execution

## Tags
#CICD #GitLabCI #QualityGatekeeping #TriggerControl #GitLabRules #GitLabWorkflow #Harbor #ImageManagement #DevOps #InterviewQuestion

## I. Topic Explanation

This article focuses on three crucial aspects in a production environment:

1. Not every code submission should trigger the entire production release process.
2. Not every submission is worth building an image and executing the continuous delivery process.
3. CI/CD is not just about automated execution; it also plays a role in quality gatekeeping and controlling deployment permissions.

This article emphasizes the following key points:

- The role and layers of quality gatekeeping.
- Who can trigger certain actions, which branches allow it, and which environments can be deployed to.
- How to decide whether to build an image or deploy based on the scope of changes.
- Typical uses of `workflow:rules`, `rules`, `rules:changes`, manual jobs, protected branches, and protected environments in GitLab.

GitLab officially supports using `workflow:rules` to control whether a pipeline is created, `rules` and `rules:changes` to control whether a job runs, protected branches to restrict who can push/merge into the main branch, and protected environments and manual jobs to control who can deploy to specific environments. GitLab also allows you to use `[ci skip]` / `[skip ci]` to skip a pipeline.
For more information:
- GitLab `workflow` documentation
- GitLab job `rules` documentation
- GitLab protected branches documentation
- GitLab protected environments documentation
- GitLab job control / manual jobs documentation

---

## II. Why Quality Gatekeeping and Trigger Control Are Essential in Production Environments

Without proper gatekeeping, common issues include:

- Anybody can submit code, potentially wasting Runner resources.
- Minor changes, such as comments or document edits, also trigger image building and deployment, resulting in inefficiency.
- The main branch lacks protection, allowing code risks to directly enter the release process.
- The testing environment is frequently updated with meaningless changes, affecting integration and verification.
- Production deployment permissions are uncontrolled; anyone can initiate a deploy.
- In the absence of quality gatekeeping, low-quality code may also make it through the build and release stages.

Therefore, CI/CD in production environments is not about “automating everything.” Instead, it’s about:

**Only running pipelines and jobs that are necessary and allowing only authorized personnel to trigger critical environment deployments.**

---

## III. Production Gatekeeping Can Be Understood as Four Layers

## 1. Code Gatekeeping

Role:

- Intercept obviously erroneous code.
- Identify quality issues early on.
- Reduce unnecessary waste in subsequent build and deployment steps.

Common practices include:

- Linting.
- Unit testing.
- Basic static checks.
- Validation of configuration files.
- Verification of Dockerfiles.
- Inspection of Helm/YAML templates.
- Security scans.

The core idea here is:

**If the code does not meet the standards, it should not proceed to the build and deployment stages.**

---

## 2. Merge Gatekeeping

Role:

- Control who can modify the main branch.
- Prevent unreviewed code from directly entering the release process.

Common methods include:

- Designating the main branch as a protected branch.
- Prohibiting direct pushes to main/master.
- Using Merge Requests for merging changes.
- Setting up approval processes.
- Ensuring that only designated roles can approve before merging.

This layer ensures that only appropriate code is considered for deployment.

---

## 3. Pipeline Gatekeeping

Role:

- Determine which branches should trigger which jobs.
- Decide which changes require building images.
- Specify which submissions only need light checks and no heavy processing.

Common practices include:

- Running check/test jobs only on feature branches.
- Allowing build and test image creation on the develop branch.
- Only allowing the main branch to proceed to the deployment stage.
- Not building images for changes in files like docs/README.
- Building images only when paths like `src/`, `Dockerfile`, `helm/`, or `deploy/` have changed.

This layer ensures that not all changes warrant a full pipeline execution.

---

## 4. Deployment Gatekeeping

Role:

- Control who can deploy code to environments.
- Determine when deployments should occur.
- Ensure that production deployments require human confirmation.

Common practices include:

- Setting the deploy job as manual.
- Allowing manual triggering of tests in the testing environment.
- Binding production deployments to protected environments.
- Ensuring that only authorized personnel can perform production deployments.
- Using approval processes, windows, and release rules to control production deployments.

This layer ensures that even if a pipeline can execute, not everyone is allowed to deploy code.

---

## IV. Why Tests Should Not Always Require Image Building and Deployment for Every Submission

This is a crucial concept in production environments```markdown
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
        - docker build -t myapp:${IMAGE_tag} .
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
Controls whether to create a pipeline:

- If the commit message contains `[ci skip]` or `[skip ci]`, do not create a pipeline.
- In all other cases, create a pipeline.

#### `basic_check`
Runs under any circumstances as a lightweight check.

#### `build_image`
Build is only executed when these files change:

- `src/**/*`
- `Dockerfile`
- `docker/**/*
- `helm/**/*
- `deploy/**/*`

Changes in files such as `README.md` or `docs/**/*`, which are unrelated to the build process, will not trigger the build job.
```### 3. Not all submissions need to go through the entire pipeline
It’s important to distinguish between different types of changes based on their severity.

### 4. The testing environment also needs to be managed properly
A testing environment should not be used as an unlimited public playground.

### 5. Build and deploy processes should be decoupled
In many cases, it’s suitable to have:

- Automatic build
- Manual deployment

### 6. Quality checks should be performed at an early stage
Fundamental issues should not be discovered only during the deployment process.

---

## Sixteen: How to Answer Interview Questions More Confidently

### Answer Template 1: Quality Checks
In CI/CD, quality checks are more than just running a test; they involve controlling code quality, configuration validity, build feasibility, and release permissions at an early stage. Typically, linting, unit testing, and basic verification are performed first within the MR or branch pipeline. If these steps fail, the process moves no further to build or deploy.

### Answer Template 2: Why Not Build and Deploy Every Time a Submission Is Made
Testing environments are not designed to automatically build images and trigger deployments for every submission. It’s more practical to distinguish between minor and major changes based on their scope. For example, changes to README files, documentation, or comments only require lightweight checks without building images. Only when there are changes to the code directory, Dockerfile, Helm configuration, or deployment files should the build process be initiated. Deployments can also be set up manually to prevent frequent updates in the testing environment.

### Answer Template 3: Who Can Trigger Production Deployments
In a production environment, code permissions, pipeline access rights, and environmental controls are typically separated. The main branch is usually set as a protected branch that requires MR approval before merging. Pipelines use rules to determine which branches can proceed to the deployment phase. Actual deployments are often carried out through manual jobs in a protected environment, ensuring that only authorized personnel can execute them.

---

## Seventeen: Simplified Phrases You Can Memorize

In production scenarios, not every GitLab submission leads to a full build and deploy process. Controls are implemented at multiple levels: First, quality checks ensure code suitability; second, merge approval restricts access to the main branch; third, pipeline rules dictate which tasks should be executed based on branch and change type; fourth, manual deployment processes in protected environments ensure only authorized personnel can release software to testing or production environments. This approach saves resources and aligns with best practices for production governance.

---

## Eighteen: Key Terms You Should Remember

1. Quality checks
2. Merge approval
3. Pipeline rules
4. Manual deployment
5. Protected environment
6. Lightweight pipeline
7. Full pipeline
8. Change-based execution

---

## Nineteen: Summary

The essence of this topic is not about mastering a few GitLab syntaxes but about developing the right mindset for production use:

### 1. Not all code changes warrant going through the full build and deploy process
Minor modifications should not waste resources.

### 2. Only authorized personnel should have access to critical pipeline steps
Code, pipeline, and environmental permissions must be carefully managed.

### 3. CI/CD is not just about automation; it’s a system for governance
It serves multiple purposes:

- Quality control
- Resource management
- Permission management
- Release coordination

In summary:

**In a production environment, the core of CI/CD is “conditional, bounded, and permission-controlled automation.”**