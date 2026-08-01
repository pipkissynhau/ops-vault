# Interview Question 46: CI/CD Image Tag Management and GitLab CI Practice Notes

## Tags
#CICD #GitLabCI #Docker #Harbor #MirrorManagement #TagManagement #DevOps #ProductGovernance #Interview

## I. Topic Overview

This article focuses on a common issue encountered in both interviews and actual work:

- How should image tags be designed in CI/CD build processes?
- How does GitLab CI generate image tags?
- What roles do Project, Repository, and Tag play in Harbor?
- Why do many companies use `branch + commit short sha + pipeline id` to generate tags?
- How to avoid version drift and rollback difficulties caused by `latest`?

---

## II. Establishing Overall Understanding

Image management cannot be viewed solely through the lens of tags.  
A more comprehensive understanding should be:

    registry / project / repository : tag

For example:

    harbor.example.com/prod/order-service:main-a1b2c3d-1045

This path can be divided into four layers:

### 1. registry
Image repository address, for example:

    harbor.example.com

### 2. project
Harbor project, used for environment, team, and business line isolation, for example:

    prod
    test
    middleware
    payment

### 3. repository
Specific service or image name, for example:

    order-service
    user-service
    nginx-base

### 4. tag
The specific version identifier of the image, for example:

    main-a1b2c3d-1045
    v1.2.3
    release-20260401

Therefore, image governance essentially isn't just defining tags, but also governing:

1. Project planning
2. Repository naming
3. Tag rules

---

## III. Core Goals of Image Tag Management

The core goals of image tag management are four:

### 1. Uniqueness
Each build artifact should be uniquely identifiable to avoid overwriting.

### 2. Traceability
Through image tags, you can trace back to:

- Which branch it came from
- Which commit it corresponds to
- Which pipeline it came from
- Which build log it corresponds to

### 3. Rollback Capability
When issues occur in production releases, you should be able to quickly find the previous stable image version.

### 4. Governance
Facilitate in Harbor:

- Permission isolation
- Lifecycle management
- Tag retention policy
- Immutable tag management

---

## IV. Why Not Recommend Using 'latest' in Production

The biggest problem with `latest` isn't "not being usable", but:

- Not unique
- Drifts over time
- Difficult to audit accurately
- Not conducive to quick rollback
- Prone to confusion in parallel development

For example:

Today's:

    myapp:latest

And tomorrow's:

    myapp:latest

The names are the same, but the image content may be completely different.

Therefore:

**Developers can retain 'latest' as a convenience alias, but it's not recommended to rely solely on 'latest' for production releases.**

---

## V. Common Tag Design Patterns

### 1. Only Use Commit SHA

Example:

    a1b2c3d

Advantages:

- Directly corresponds to code commits
- Convenient for automation
- Simple

Disadvantages:

- Poor readability
- Cannot distinguish repeated builds of the same commit

---

### 2. Commit SHA + Pipeline ID

Example:

    a1b2c3d-1045

Advantages:

- Strong uniqueness
- Can trace back to code commits
- Can trace back to specific pipelines
- Can distinguish repeated builds of the same commit

This is a very common and mature practice for production environments.

---

### 3. Branch + Commit SHA + Pipeline ID

Example:

    main-a1b2c3d-1045
    develop-a1b2c3d-1045
    feature-login-f6e8a1b-1099

Advantages:

- Can distinguish which branch the image comes from
- Can distinguish different builds
- Suitable for test environments and multi-branch development scenarios

Disadvantages:

- Branch names need standardized processing
- Readability decreases when branch names are too long

---

### 4. Semantic Versioning

Example:

    v1.2.3

Advantages:

- Friendly to humans
- Suitable for formal releases

Disadvantages:

- Weak ability to trace back to commits/pipelines when used alone

---

### 5. Timestamp

Example:

    20260401-231500

Advantages:

- Clearly reflects build time

Disadvantages:

- Cannot directly correspond to Git commits
- Poorer audit capabilities compared to SHA

---

## VI. Recommended Design Principles

In production, a recommended approach is:

### First Layer: Unique Tag
Used for deployment, audit, and rollback.

Example:

    main-a1b2c3d-1045

### Second Layer: Semantic Tag
Used for human readability.

Example:

    v1.2.3
    staging
    prod

A single image can have multiple tags.

Example:

    harbor.example.com/prod/order-service:main-a1b2c3d-1045
    harbor.example.com/prod/order-service:v1.2.3

Where:

- The actual deployment record is recommended to use unique tags
- Semantic tags are more suitable for release management and human reading

---

## VII. Roles of Project, Repository, and Tag in Harbor

## 1. Project: Isolation Layer
Harbor projects typically handle:

- Environment isolation
- Team isolation
- Business line isolation
- Permission control
- Quota control
- Policy configuration

Common division methods include three types:

### Divided by Environment
For example:

    dev
    test
    uat
    prod

Advantages:

- Clear environment boundaries
- Lower risk of pushing to the wrong environment

---

### Divided by Business Line
For example:

    payment
    user-center
    platform
    infra

Advantages:

- Aligns with organizational structure
- Natural for permission governance

---

### Divided by System Domain
For example:

    mall
    monitoring
    middleware

Advantages:

- Suitable for long-term maintenance

---

## 2. Repository: Service Layer
Repository represents a specific image repository, typically corresponding to:

- A microservice
- A base image
- A common component

For example:

    order-service
    user-service
    java17-base
    nginx-ingress

---

## 3. Tag: Version Layer
Tag represents different versions or build artifacts of the same repository.

For example:

    main-a1b2c3d-1045
    release-2026.04-a1b2c3d-1045
    v2.0.1

---

## VIII. How Tags Are Actually Generated in GitLab CI

This is where many people tend to get confused.

The key point is just one sentence:

**When GitLab CI runs a job, it automatically injects a batch of predefined variables; you use shell in `script` to concatenate these variables into a Docker image Tag.**

This is typically not a "mysterious internal function"—essentially it's:

1. GitLab provides variables
2. `gitlab-ci.yml` reads the variables
3. shell script concatenates strings
4. passes the constructed Tag to `docker build` and `docker push`

---

## IX. Common Predefined Variables in GitLab CI

GitLab officially provides a large number of predefined variables. The most commonly used ones in this context are:

### 1. CI_COMMIT_SHORT_SHA
The short SHA of the current commit.

Example:

    a1b2c3d

---

### 2. CI_COMMIT_SHA
The full SHA of the current commit.

---

### 3. CI_PIPELINE_ID
The ID of the current pipeline.

Example:

    1045

---

### 4. CI_COMMIT_REF_NAME
The name of the current branch or tag.

Example:

    main
    develop
    feature/login

---

### 5. CI_COMMIT_BRANCH
The name of the current branch. Suitable for branch pipeline scenarios.

---

### 6. CI_REGISTRY_IMAGE
The image path for GitLab's built-in container registry.  
If using an external Harbor, it's common to define a custom registry address variable.

---

## X. Minimal Usable Mental Model

You can understand the image build process as the following four steps:

### Step 1: Get Variables

    $CI_COMMIT_REF_NAME
    $CI_COMMIT_SHORT_SHA
    $CI_PIPELINE_ID

### Step 2: Concatenate Tag

    IMAGE_TAG="${CI_COMMIT_REF_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

### Step 3: Build Image

    docker build -t myapp:${IMAGE_TAG} .

### Step 4: Push Image

    docker push myapp:${IMAGE_TAG}

This is the core workflow.

---

## XI. Why Standardize Branch Names

Many Git branch names look like this:

    feature/login
    release/2026.04
    bugfix/order/api

Directly concatenating these into a Tag would be unstandardized and inconvenient for future use.

The usual approach is to replace `/` with `-`:

    feature-login
    release-2026.04
    bugfix-order-api

Common shell processing in scripts:

    BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')

---

## XII. GitLab CI Minimal Example: Generate Unique Tag Only

Here's the most basic example:

    stages:
      - build

    variables:
      IMAGE_NAME: "myapp"

    build_image:
      stage: build
      script:
        - IMAGE_TAG="${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
        - echo "IMAGE_TAG=$IMAGE_TAG"
        - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        - docker push ${IMAGE_NAME}:${IMAGE_TAG}

### Meaning of This Section

- Get current commit short SHA
- Get current pipeline ID
- Concatenate into a unique Tag
- Build image
- Push image

---

## XIII. GitLab CI Example: Tag with Branch Name

    stages:
      - build

    variables:
      IMAGE_NAME: "myapp"

    build_image:
      stage: build
      script:
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
        - IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
        - echo "IMAGE_TAG=$IMAGE_TAG"
        - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        - docker push ${IMAGE_NAME}:${IMAGE_TAG}

### Assuming These Pipeline Variables

- Branch: feature/login
- Commit short sha: a1b2c3d
- Pipeline id: 1045

The final Tag would be:

    feature-login-a1b2c3d-1045

---

## XIV. GitLab CI Example: Push to Harbor

The following example is closer to common enterprise practices.

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
    - echo "Preparing to build image: ${FULL_IMAGE_NAME}"
    - docker login -u "$HARBOR_USER" -p "$HARBOR_PASSWORD" "$HARBOR_HOST"
    - docker build -t "${FULL_IMAGE_NAME}" .
    - docker push "${FULL_IMAGE_NAME}"

### Notes

Here we assume:

- Harbor credentials are configured through GitLab CI/CD Variables
- `HARBOR_USER` and `HARBOR_PASSWORD` are not hard-coded in the repository
- Harbor project is `test`
- Service image name is `order-service`

If variables are:

- branch = feature/login
- commit short sha = a1b2c3d
- pipeline id = 1045

The final image address might be:

    harbor.example.com/test/order-service:feature-login-a1b2c3d-1045

---

## FifteenI don't know.GitLab CI Example: Tagging with Unique Tag and Semantic Tag

In actual production scenarios, it's common to create both a unique tag and a semantic tag.

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

### Suitable Scenarios

- Need a unique tag for traceability
- Want product, operations, and R&D teams to see an easily recognizable version number

---

## SixteenI don't know.Common Branch-to-Environment Mapping Strategies

Many companies don't push all branches to the same Harbor project.

Common strategies are as follows:

### 1. Develop branch pushes to test project

    develop  -> harbor/test/...

### 2. Release branch pushes to pre-production project

    release/* -> harbor/uat/...

### 3. Main or master branch pushes to production project

    main -> harbor/prod/...

This approach has the following advantages:

- Clear environment boundaries
- Easier permission control
- Lower risk of accidental deployment

---

## SeventeenI don't know.GitLab CI Example: Switching Harbor Project by Branch

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
        - echo "HARBOR_PROJECT=$HARBOR_PROJECT"
        - echo "FULL_IMAGE_NAME=$FULL_IMAGE_NAME"
        - docker login -u "$HARBOR_USER" -p "$HARBOR_PASSWORD" "$HARBOR_HOST"
        - docker build -t "${FULL_IMAGE_NAME}" .
        - docker push "${FULL_IMAGE_NAME}"

### Notes

This approach demonstrates:

- Branch strategy
- Environment isolation
- Release path planning

When explaining this in an interview, your maturity level will be significantly higher.

---

## 18. What does "calling internal functions" usually mean when a company says it

Many people get nervous hearing "internal functions," but the common scenarios are actually just three:

### Scenario 1: Just variable concatenation in a script
For example:

    IMAGE_TAG="${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

This is not a function at all—it's just shell string concatenation.

---

### Scenario 2: Encapsulated in a shell script
For example `ci/build.sh`:

    generate_image_tag() {
      local branch_name="$1"
      local commit_sha="$2"
      local pipeline_id="$3"
      branch_name=$(echo "$branch_name" | tr '/' '-')
      echo "${branch_name}-${commit_sha}-${pipeline_id}"
    }

    IMAGE_TAG=$(generate_image_tag "$CI_COMMIT_REF_NAME" "$CI_COMMIT_SHORT_SHA" "$CI_PIPELINE_ID")

Then called in `.gitlab-ci.yml`:

    build_image:
      stage: build
      script:
        - bash ci/build.sh

---

### Scenario 3: Company encapsulates public templates
Some companies maintain unified CI templates, for example:

    include:
      - project: devops/gitlab-ci-templates
        file: docker-build.yml

Then business projects just pass some variables:

    variables:
      APP_NAME: "order-service"
      ENV_NAME: "test"

This looks like "calling internal functions," but essentially it's:

- Calling public CI templates
- The templates already implement login, build, tag, and push logic internally

---

## 19. The relationship between Harbor's governance capabilities and Tag management

Harbor isn't just for storing images—it can also perform many governance actions when integrated.

### 1. Tag Retention
Harbor supports configuring Tag retention rules.  
This is important because frequent builds quickly accumulateMass Tags, leading to storage bloat.

Example approach:

- Retain only the latest N Tags
- Retain only Tags from the last N days
- Permanently retain release/prod-related Tags

---

### 2. Tag Immutability
Harbor supports configuring Tag immutability rules.  
This is critical for production Tags.

For example:

- `v1.2.3` Tags are immutable once released
- `prod-*` Tags cannot be overwritten by new pushes

This prevents:

- Same-named Tags being overwritten repeatedly
- Distorted release records
- Loss of rollback basis

---

### 3. Project Configuration
Harbor projects can also be configured:

- Public/private access
- Vulnerability scanning
- Content trust
- Tag management
- Permission models

This means image governance is not just a CI issue—it's also a repository governance issue.

---

## 20. Recommended practices in real production environments

## 1. Do not write sensitive information into repositories
For example:

- Harbor username/password
- Tokens
- Registry credentials

These should be placed in GitLab's CI/CD Variables.

---

## 2. Use unique Tags as the real deployment basis
For example:

    main-a1b2c3d-1045

Instead of just recording:

    latest
    staging
    prod

Because these semantic Tags will drift.

---

## 3. Semantic Tags are only auxiliary
For example:

    v1.2.3
    rc
    prod

These can be retained, but should not replace unique Tags.

---

## 4. Apply immutability rules to formal release Tags
Once formal release Tags are generated, they should not be allowed to be overwritten.

---

## 5. Configure retention rules to clean up invalid images
For example retain only:

- Test images from the last 30 days
- Last 100 development Tags
- Permanently retain release versions

---

## 6. Maintain mapping relationships in the release system
It's recommended to at least record:

- Branch name
- Commit sha
- Pipeline id
- Full image path
- Release time
- Release environment
- Release person

Only then will tracing and auditing capabilities be complete.

---

## 21. Common misconceptions

### Misconception 1: Image management is just giving images names
Wrong.  
Image management essentially is artifact governance, including:

- Path planning
- Tag design
- Access control
- Retention policies
- Rollback auditing

---

### Misconception 2: Having latest is enough
Wrong.  
latest is not suitable as the sole basis for production environments.

---

### Misconception 3: Tags with commit sha are always sufficient
Not necessarily.  
If the same commit is built repeatedly, adding pipeline id is recommended.

---

### Misconception 4: Branch names can be directly used in Tags
Not recommended.  
Standardization should be done first.

---

### Misconception 5: Harbor project organization doesn't matter
Wrong.  
Project organization directly affects:

- Permission models
- Environment isolation
- Operation complexity
- Risk of pushing to wrong repositories

---

### Misconception 6: A successful build means the Tag scheme is fine
Not necessarily.  
Whether the Tag scheme is reasonable depends on:

- Whether it's traceable
- Whether it's rollable
- Whether it's suitable for governance
- Whether it aligns with branch and environment strategies

---

## 22. How to answer more confidently in interviews

### Basic Answer

The core of image Tag management is uniqueness, traceability, and rollability. In production environments, using latest alone is not recommended—typically, unique Tags are generated based on branch names, commit short sha, and pipeline id. This allows tracing the corresponding code version and pipeline records. For Harbor, you also need to consider project, repository, and tag together. Projects are generally used for environment or business isolation, repositories represent specific services, and tags indicate specific build versions.

---

### Advanced Answer /think

GitLab CI typically generates image Tags not through a complex internal function, but by leveraging predefined variables injected by GitLab Runner during job execution, such as `CI_COMMIT_REF_NAME`, `CI_COMMIT_SHORT_SHA`, and `CI_PIPELINE_ID`. These variables are then concatenated into a Tag in `.gitlab-ci.yml`'s `script` or public shell scripts and passed to `docker build` and `docker push`. If the company has standardized packaging, this logic might be reused via shared templates or scripts. Repository-side governance further refines this through Harbor's project planning, Tag retention rules, and immutability rules.

---

## 23. How to Answer if You Haven't Done It Yourself

You could say:

This part I haven't fully led in actual work, so my hands-on experience isn't deep yet, but I understand the implementation logic. Typically, CI platforms provide variables like branch, commit, pipeline ID, etc., during pipeline execution. These are concatenated into a unique Tag in `.gitlab-ci.yml` or build scripts, then pushed to Harbor. Harbor further governs via project, repository, and Tag classification. If I were to strengthen this, I'd start with the minimal GitLab CI build + Tag + push pipeline.

This phrasing is more stable than "completely unfamiliar" and better reflects reality.

---

## 24. Recommended Minimal Practice Path

If you want to strengthen your hands-on experience, follow this order:

### Step 1
Prepare a simple Dockerfile project locally.

### Step 2
Manually execute:

    docker build -t myapp:test .
    docker tag myapp:test harbor.example.com/dev/myapp:test
    docker push harbor.example.com/dev/myapp:test

First, thoroughly understand the image path.

### Step 3
Write a minimal `.gitlab-ci.yml` that only does:

- Print variables
- Generate Tag
- Print full image name

Don't rush to push yet.

### Step 4
Integrate Harbor login and push.

### Step 5
Finally add:

- Branch-to-environment mapping
- Multiple Tags
- Release Tags
- Harbor retention and immutability rules

---

## 25. Summary

Image Tag management isn't simply about "how to concatenate strings" — it's fundamentally a problem of artifact governance.

The complete model can be remembered as:

### 1. Path Structure
    registry / project / repository : tag

### 2. Governance Goals
- Unique
- Traceable
- Rollback-capable
- Governable

### 3. GitLab CI Implementation Essentials
- Runner-injected variables
- Tag generation in `script`
- `docker build`
- `docker push`

### 4. Harbor Governance Mechanisms
- Project isolation
- Tag retention
- Tag immutability
- Permissions and quotas

One-sentence summary:

**Image Tags should ideally both locate code sources and build processes, while Harbor governs these images by project, repository, and policy.**

---

## 26. References

### GitLab Official
- Predefined CI/CD variables
- CI/CD YAML syntax reference
- Use CI/CD variables in job scripts
- Scripts and job logs

### Harbor Official
- Working with images and tags
- Repositories
- Tag retention rules
- Tag immutability rules
- Project configuration