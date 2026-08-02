# Interview Question 47: Practical Notes on Image Building, Tag Generation, and Pushing from GitLab CI to Harbor

## Tags
#CICD #GitLabCI #Harbor #Docker #MirrorManagement #TagManagement #DevOps #ProductGovernance #Interview

## One, What This Note Solves

This article only addresses one minimal main line:

1. Code committed to GitLab
2. GitLab Runner executes the pipeline
3. Generate image Tag in the pipeline
4. Execute docker build
5. Execute docker push to Harbor
6. Understand the relationship between Harbor's Project / Repository / Tag

The goal is not to directly build complex platforms, but to thoroughly explain "how image Tags are generated and how to write `.gitlab-ci.yml`".

---

## Two, Remember the Core Rule First

In GitLab CI, image Tag generation typically does not rely on a mysterious "internal function", but rather:

- GitLab Runner injects a batch of variables when running a job
- Read these variables in `.gitlab-ci.yml`'s `script`
- Use shell to concatenate these variables into a string
- Pass this string as the image Tag to `docker build` and `docker push`

In essence, it's typically:

**Variables + shell concatenation + docker commands**

---

## Three, What Does a Complete Image Address Look Like

A complete image address is usually written as:

    registry/project/repository:tag

Example:

    harbor.example.com/prod/order-service:main-a1b2c3d-1045

Breaking it down:

### 1. registry
Repository address:

    harbor.example.com

### 2. project
Harbor project:

    prod

### 3. repository
Service image name:

    order-service

### 4. tag
Specific version:

    main-a1b2c3d-1045

Therefore, image governance isn't just about the Tag, but also understanding:

1. How Harbor Projects are divided
2. How Repositories are named
3. How Tags are generated

---

## Four, Why Manage Image Tags

The core goals of image Tag management are four:

### 1. Uniqueness
Each build artifact should uniquely identify, avoiding overwrites.

### 2. Traceability
Seeing a Tag should allow tracing back to:

- Which branch it came from
- Which commit it came from
- Which pipeline it came from

### 3. Rollback Capability
When there's an issue in production, quickly revert to a stable previous image.

### 4. Governance
Facilitate Harbor's:

- Retention policies
- Immutability policies
- Permissions and project isolation

---

## Five, Why Not Recommend Using 'latest' in Production

`latest` isn't unusable, but it's unsuitable as the sole basis for production.

Reasons:

- It's not unique
- It drifts
- Difficult to audit
- Not conducive to rollback
- Causes confusion in collaborative environments

Example:

    myapp:latest

The name is the same today and tomorrow, but the actual image content may be completely different.

Therefore, a more secure approach is:

- `latest` can serve as an auxiliary alias
- Actual deployment should record the unique Tag

---

## Six, Common Tag Design Patterns

### 1. Only Use Commit Short SHA

Example:

    a1b2c3d

Advantages:

- Simple
- Directly corresponds to code commits

Disadvantages:

- Can't distinguish repeated builds of the same commit

---

### 2. Commit Short SHA + Pipeline ID

Example:

    a1b2c3d-1045

Advantages:

- Stronger uniqueness
- Can distinguish multiple builds of the same commit
- Can trace to specific pipelines

---

### 3. Branch + Commit Short SHA + Pipeline ID

Example:

    main-a1b2c3d-1045
    develop-a1b2c3d-1045
    feature-login-a1b2c3d-1045

Advantages:

- Can identify which branch it came from
- Can trace code
- Can trace pipelines
- Very suitable for test environments and multi-branch development scenarios

This is a very common production practice.

---

### 4. Semantic Versioning

Example:

    v1.2.3

Advantages:

- Friendly to humans
- Suitable for formal releases

Disadvantages:

- Weak traceability to commits and pipelines when used alone

---

## Seven, Recommended Combined Approach

In production, a "dual-layer Tag" is more recommended:

### First Layer: Unique Tag
Used for deployment, auditing, and rollback.

Example:

    main-a1b2c3d-1045

### Second Layer: Semantic Tag
Used for human identification and release expression.

Example:

    v1.2.3
    prod
    staging

A single image can have both Tags, for example:

    harbor.example.com/prod/order-service:main-a1b2c3d-1045
    harbor.example.com/prod/order-service:v1.2.3

Deployment records are more recommended to use the unique Tag.

---

## Eight, Most Common Variables in GitLab CI

In GitLab CI, these variables are crucial:

### 1. CI_COMMIT_REF_NAME
Current branch name or tag name.

Example:

    main
    develop
    feature/login

### 2. CI_COMMIT_SHORT_SHA
Short SHA of the current commit.

Example:

    a1b2c3d

### 3. CI_PIPELINE_ID
Current pipeline ID.

Example:

    1045

### 4. CI_COMMIT_SHA
Full commit SHA.

### 5. CI_COMMIT_BRANCH
Current branch name, often used in branch pipelines.

---

## Nine, Look at the Minimal Model

You only need to understand the following four steps:

### Step 1: Get Variables

    $CI_COMMIT_REF_NAME
    $CI_COMMIT_SHORT_SHA
    $CI_PIPELINE_ID

### Step 2: Concatenate Tag

    IMAGE_TAG="${CI_COMMIT_REF_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

### Step 3: Build

    docker build -t myapp:${IMAGE_TAG} .

### Step 4: Push

    docker push myapp:${IMAGE_TAG}

This is the minimal core workflow.

## 10. Why Branch Names Should Be Standardized

Many branch names look like this:

    feature/login
    bugfix/order/api
    release/2026.04

If these branch names are directly used as Tags, they are often not standardized.  
So the common practice is to replace `/` with `-`:

    feature-login
    bugfix-order-api
    release-2026.04

Common shell syntax:

    BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')

---

## 11. `.gitlab-ci.yml` Minimal Example: Generating a Unique Tag Only

The following example does the bare minimum:

- Read commit short sha
- Read pipeline id
- Generate a unique Tag
- Build
- Push

    stages:
      - build

    variables:
      IMAGE_NAME: "myapp"

    build_image:
      stage: build
      script:
        - IMAGE_TAG="${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
        - echo "IMAGE_TAG=${IMAGE_TAG}"
        - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        - docker push ${IMAGE_NAME}:${IMAGE_TAG}

### How to Understand This

Assume the current variables are:

- CI_COMMIT_SHORT_SHA = a1b2c3d
- CI_PIPELINE_ID = 1045

Then:

    IMAGE_TAG = a1b2c3d-1045

The final executed commands are approximately:

    docker build -t myapp:a1b2c3d-1045 .
    docker push myapp:a1b2c3d-1045

---

## 12. `.gitlab-ci.yml` Example: Tag with Branch Name

Here is a more common approach:

    stages:
      - build

    variables:
      IMAGE_NAME: "myapp"

    build_image:
      stage: build
      script:
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
        - IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
        - echo "IMAGE_TAG=${IMAGE_TAG}"
        - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        - docker push ${IMAGE_NAME}:${IMAGE_TAG}

### Assuming These Pipeline Variables

- CI_COMMIT_REF_NAME = feature/login
- CI_COMMIT_SHORT_SHA = a1b2c3d
- CI_PIPELINE_ID = 1045

Then the final result is:

    BRANCH_NAME = feature-login
    IMAGE_TAG = feature-login-a1b2c3d-1045

The final executed commands are approximately:

    docker build -t myapp:feature-login-a1b2c3d-1045 .
    docker push myapp:feature-login-a1b2c3d-1045

---

## 13. `.gitlab-ci.yml` Example: Pushing to Harbor

If the image registry is not GitLab's built-in Registry but Harbor, it's usually written like this:

    stages:
      - build

    variables:
      HARBOR_HOST: "harbor.example.com"
      HARBOR_PROJECT: "test"
      IMAGE_NAME: "order-service"

    build_image:
      stage: build
      script:
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
        - IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
        - FULL_IMAGE_NAME="${HARBOR_HOST}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
        - echo "FULL_IMAGE_NAME=${FULL_IMAGE_NAME}"
        - docker login -u "$HARBOR_USER" -p "$HARBOR_PASSWORD" "$HARBOR_HOST"
        - docker build -t "${FULL_IMAGE_NAME}" .
        - docker push "${FULL_IMAGE_NAME}"

### What Does This Do

Assume:

- HARBOR_HOST = harbor.example.com
- HARBOR_PROJECT = test
- IMAGE_NAME = order-service
- CI_COMMIT_REF_NAME = feature/login
- CI_COMMIT_SHORT_SHA = a1b2c3d
- CI_PIPELINE_ID = 1045

Then:

    BRANCH_NAME = feature-login
    IMAGE_TAG = feature-login-a1b2c3d-1045
    FULL_IMAGE_NAME = harbor.example.com/test/order-service:feature-login-a1b2c3d-1045

The final executed commands are approximately:

docker login -u xxx -p yyy harbor.example.com
docker build -t harbor.example.com/test/order-service:feature-login-a1b2c3d-1045 .
docker push harbor.example.com/test/order-service:feature-login-a1b2c3d-1045

---

## Fourteen. The Most Important Point: Do Not Hardcode Harbor Username/Password in Repository

Do not do this:

    docker login -u admin -p 123456 harbor.example.com

A more reasonable approach is to store sensitive information in GitLab project CI/CD Variables, such as:

- HARBOR_USER
- HARBOR_PASSWORD

Then use them in `.gitlab-ci.yml`:

    docker login -u "$HARBOR_USER" -p "$HARBOR_PASSWORD" "$HARBOR_HOST"

This prevents credentials from being exposed in the repository code.

---

## Fifteen. `.gitlab-ci.yml` Example: Tagging with Both Unique and Version Tags Simultaneously

In actual work, it's often necessary to tag two versions:

- Unique Tag for real deployment
- Version Tag for human identification

Example:

    stages:
      - build

    variables:
      HARBOR_HOST: "harbor.example.com"
      HARBOR_PROJECT: "prod"
      IMAGE_NAME: "order-service"
      RELEASE_TAG: "v1.2.3"

    build_image:
      stage: build
      script:
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
        - UNIQUE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
        - IMAGE_UNIQUE="${HARBOR_HOST}/${HARBOR_PROJECT}/${IMAGE_NAME}:${UNIQUE_TAG}"
        - IMAGE_RELEASE="${HARBOR_HOST}/${HARBOR_PROJECT}/${IMAGE_NAME}:${RELEASE_TAG}"
        - docker login -u "$HARBOR_USER" -p "$HARBOR_PASSWORD" "$HARBOR_HOST"
        - docker build -t "${IMAGE_UNIQUE}" .
        - docker tag "${IMAGE_UNIQUE}" "${IMAGE_RELEASE}"
        - docker push "${IMAGE_UNIQUE}"
        - docker push "${IMAGE_RELEASE}"

### What This Means

First build a unique image:

    harbor.example.com/prod/order-service:main-a1b2c3d-1045

Then create a semantic version Tag based on it:

    harbor.example.com/prod/order-service:v1.2.3

Then push both.

---

## Sixteen. Example of Switching Harbor Project by Branch

Many companies do not push all branches to the same Harbor Project.  
Instead, they map branches to environments, for example:

- main -> prod
- develop -> test
- Other branches -> dev

Example:

    stages:
      - build

    variables:
      HARBOR_HOST: "harbor.example.com"
      IMAGE_NAME: "order-service"

    build_image:
      stage: build
      script:
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
        - |
          if [ "$CI_COMMIT_REF_NAME" = "main" ]; then
            HARBOR_PROJECT="prod"
          elif [ "$CI_COMMIT_REF_NAME" = "develop" ]; then
            HARBOR_PROJECT="test"
          else
            HARBOR_PROJECT="dev"
          fi
        - IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
        - FULL_IMAGE_NAME="${HARBOR_HOST}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
        - echo "HARBOR_PROJECT=${HARBOR_PROJECT}"
        - echo "FULL_IMAGE_NAME=${FULL_IMAGE_NAME}"
        - docker login -u "$HARBOR_USER" -p "$HARBOR_PASSWORD" "$HARBOR_HOST"
        - docker build -t "${FULL_IMAGE_NAME}" .
        - docker push "${FULL_IMAGE_NAME}"

### What This Example Demonstrates

This demonstrates:

- Branch strategy
- Environment isolation
- Harbor Project planning
- Pipeline-to-environment mapping relationship

---

## Seventeen. What Does It Usually Mean When Someone Says "Call an Internal Function to Generate Tag"

In reality, there are typically three scenarios:

### Scenario 1: It's Actually Not a Function, Just String Concatenation
For example:

    IMAGE_TAG="${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

This is simply shell string concatenation.

---

### Scenario 2: The Company Has Written the Logic into a Shell Script

For example `ci/build.sh`: /think

```bash
generate_image_tag() {
  local branch_name="$1"
  local commit_sha="$2"
  local pipeline_id="$3"
  branch_name=$(echo "$branch_name" | tr '/' '-')
  echo "${branch_name}-${commit_sha}-${pipeline_id}"
}

IMAGE_TAG=$(generate_image_tag "$CI_COMMIT_REF_NAME" "$CI_COMMIT_SHORT_SHA" "$CI_PIPELINE_ID")

Then `.gitlab-ci.yml` calls it like this:

    build_image:
      stage: build
      script:
        - bash ci/build.sh

This scenario appears as if "the pipeline is calling an internal function," but fundamentally it's:

- `.gitlab-ci.yml` calls the script
- The script defines the function
- The function concatenates GitLab-injected variables

---

### Scenario 3: Company encapsulates public CI templates

For example:

    include:
      - project: devops/gitlab-ci-templates
        file: docker-build.yml

Then the business project only writes:

    variables:
      APP_NAME: "order-service"
      ENV_NAME: "test"

In this scenario, it appears as if "calling platform functions," but fundamentally it's:

- Reusing company-unified templates
- The templates already encapsulate build / tag / push logic internally

---

## Eighteen, Understanding Project / Repository / Tag in Harbor

### 1. Project
Project mainly handles:

- Environment isolation
- Team isolation
- Business isolation
- Permission control
- Quota and policy configuration

Example:

    dev
    test
    prod
    middleware
    payment

### 2. Repository
Repository corresponds to specific applications or services, for example:

    order-service
    user-service
    java-base
    nginx-ingress

### 3. Tag
Tag corresponds to different versions of the same service image, for example:

    main-a1b2c3d-1045
    v1.2.3
    develop-f6e8a1b-1088

---

## Nineteen, Why Harbor isn't just "storing images"

Harbor can also participate in governance beyond storing images:

### 1. Tag retention rules
Frequent builds generateMass Tags, requiring regular cleanup.

Examples:

- Only retain test image tags from the last 30 days
- Only retain up to 100 development Tags
- Permanently retain release or prod versions

### 2. Tag immutability rules
Some formal version Tags cannot be overwritten after release.

Examples:

- `v1.2.3`
- `prod-*`

This prevents formal version Tags from being overwritten by repeated pushes.

### 3. Project-level permissions and configuration
Different Harbor Projects can control:

- Member permissions
- Public/private access
- Quotas
- Security policies

---

## Twenty, Common interview pitfalls

### Pitfall 1: Immediately saying "we all use latest"
This shows weak artifact governance awareness.

### Pitfall 2: Only mentioning commit sha, not pipeline id
Lacks an additional layer of "build uniqueness" expression.

### Pitfall 3: Only mentioning Tag, not Harbor Project
Shows insufficient understanding of actual repository governance.

### Pitfall 4: Hardcoding Harbor credentials in `.gitlab-ci.yml`
This is clearly an unprofessional practice.

### Pitfall 5: Directly using branch names in Tag without cleaning
For example, directly using `feature/login` in Tag, which is less secure.

---

## Twenty-one, Most stable interview response templates

### Template 1: Basic answer

Image Tag management core is uniqueness, traceability, and rollability. Production environments shouldn't rely solely on latest; typically, Tags are generated based on branch names, commit short SHA, and pipeline IDs. This allows reverse lookup of the corresponding code version and pipeline record. Images are ultimately pushed to Harbor, managed through three layers: project, repository, and tag. Projects handle environment/business isolation, repositories represent specific services, and tags represent specific build versions.

### Template 2: If asked about "specific implementation"

Implementation typically isn't complex internal function calls, but rather GitLab Runner injects predefined variables during job execution, such as `CI_COMMIT_REF_NAME`, `CI_COMMIT_SHORT_SHA`, `CI_PIPELINE_ID`. These variables are then concatenated into image Tags in `.gitlab-ci.yml`'s `script` or public shell scripts, then passed to `docker build` and `docker push`. If the company has platformization, it might be encapsulated through public templates.

### Template 3: If you haven't implemented it

I haven't fully led production implementation, so operational depth isn't strong yet, but I understand the implementation logic. Typically, GitLab provides branch, commit, and pipeline ID variables during pipeline execution. These variables are concatenated into unique Tags in `.gitlab-ci.yml` or build scripts, then pushed to Harbor; repository governance is handled through project, repository, tag, and retention/immutability policies.

---

## Twenty-two, The minimum chain you should master first

Don't think about complex platforms yet; just memorize these 6 steps:

1. GitLab triggers pipeline
2. Runner executes `.gitlab-ci.yml`
3. Reads `CI_COMMIT_REF_NAME`, `CI_COMMIT_SHORT_SHA`, `CI_PIPELINE_ID`
4. Concatenates into `IMAGE_TAG`
5. Executes `docker build -t Mirror Path:IMAGE_TAG .`
6. Executes `docker push Mirror Path:IMAGE_TAG`

This is the most important understandingMain at your current stage.

---

## Twenty-three, Recommended learning order

### Step 1
First understand the minimum `.gitlab-ci.yml` example, no need to memorize all syntax.

### Step 2
Memorize this sentence:

    IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

### Step 3
Memorize the full Harbor path:

    harbor.example.com/test/order-service:feature-login-a1b2c3d-1045

### Step 4
Understand dual-layer Tags: /think

- Unique Tag: For deployment
- Version Tag: For reading

### Step Five
Go back to understand Harbor's retention rules and immutability rules

---

## 24. Summary

Don't overthink this knowledge.

Just remember three things first:

### 1. How Tags are generated
GitLab variables + shell concatenation

### 2. Where the image is finally pushed to
Harbor's:

    registry/project/repository:tag

### 3. Why this design
To achieve:

- Unique
- Traceable
- Rollback
- Governable

One-sentence summary:

**The management of image Tags in GitLab CI essentially leverages pipeline variables to generate traceable version identifiers, then pushes the image to Harbor's project-based repository for governance.**

---

## 25. External Links

- GitLab CI/CD Predefined Variables
- GitLab CI/CD YAML Syntax
- Using Variables in GitLab Job Script
- Harbor Image and Tag Management
- Harbor Tag Retention Rules
- Harbor Tag Immutability Rules