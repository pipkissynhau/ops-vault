# Interview Question 46: CI/CD Image Tag Management and GitLab CI Practice Notes

## Tags
#CICD #GitLabCI #Docker #Harbor #Image Management #Tag Management #DevOps #Product Governance #Interview Questions

## I. Topic Explanation

This article focuses on a question that is commonly encountered in both interviews and practical work:

- How should image tags be designed during the CI/CD build process?
- How does GitLab CI generate image tags?
- What roles do Project, Repository, and Tag play in Harbor?
- Why do many companies use `branch + commit short sha + pipeline id` to generate tags?
- How can we avoid version drift and difficulties in rolling back caused by the `latest` tag?

---

## II. Establishing an Overall Understanding

Image management cannot be understood solely in terms of tags. A more comprehensive understanding involves:

    registry / project / repository : tag

For example:

    harbor.example.com/prod/order-service:main-a1b2c3d-1045

This path can be broken down into four layers:

### 1. Registry
The address of the image repository, for example:

    harbor.example.com

### 2. Project
In Harbor, a project is used to isolate environments, teams, or business lines, for example:

    prod
    test
    middleware
    payment

### 3. Repository
The name of a specific service or image, for example:

    order-service
    user-service
    nginx-base

### 4. Tag
The specific version identifier of the image, for example:

    main-a1b2c3d-1045
    v1.2.3
    release-20260401

Therefore, image governance is not just about defining tags; it also involves managing:

1. Project planning
2. Repository naming
3. Tag rules

---

## III. Core Objectives of Image Tag Management

The core objectives of image tag management are fourfold:

### 1. Uniqueness
Each build artifact should be uniquely identified to prevent overwriting.

### 2. Traceability
Image tags allow for tracing back to:

- The branch from which it came
- The corresponding commit
- The pipeline that created it
- The build log associated with it

### 3. Rollback Capability
In the event of issues during online deployment, it is essential to quickly locate the last stable image version.

### 4. Governance
It facilitates tasks such as:

- Permission control in Harbor
- Lifecycle management
- Tag retention policies
- Management of immutable tags

---

## IV. Why Producing Environments Should Not Rely Solely on `latest`

The biggest problems with using `latest` are not that it cannot be used, but rather that it:

- Is not unique
- Can lead to version drift
- Makes auditing difficult
- Hinders quick rollbacks
- Causes confusion in multi-person development environments

For example:

Today's:

    myapp:latest

and tomorrow's:

    myapp:latest

may have the same name, but their image contents could be completely different.

Therefore:

**`latest` can be used as a convenient alias in development environments, but it should not be relied on for production deployments.**

---

## V. Common Tag Design Approaches

### 1. Using Only Commit SHA
Example:

    a1b2c3d

Advantages:

- Directly corresponds to the code commit
- Convenient for automation
- Simple to understand

Disadvantages:

- Readability is average
- It is difficult to distinguish between multiple builds of the same commit

---

### 2. Commit SHA + Pipeline ID
Example:

    a1b2c3d-1045

Advantages:

- High uniqueness
- Allows tracing back to the code commit and specific pipeline
- Helps differentiate between different builds of the same commit

This is a very common and mature practice in production environments.

---

### 3. Branch + Commit SHA + Pipeline ID
Example:

    main-a1b2c3d-1045
    develop-a1b2c3d-1045
    feature-login-f6e8a1b-1099

Advantages:

- Allows distinguishing between images from different branches
- Helps differentiate between different builds
- Suitable for testing environments and multi-branch development scenarios

Disadvantages:

- Branch names need to be standardized
- Long branch names can reduce readability

---

### 4. Semantic Versioning
Example:

    v1.2.3

Advantages:

- User-friendly
- Suitable for formal releases

Disadvantages:

- When used alone, it may make it difficult to trace back to specific commits or pipelines

---

### 5. Timestamp
Example:

    20260401-231500        - echo "IMAGE_TAG=$IMAGE_TAG"
        - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        - docker push ${IMAGE_NAME}:${IMAGE_TAG}

### What this means

- Get the short SHA of the current commit.
- Obtain the ID of the current pipeline.
- Combine them to form a unique Tag.
- Build the image.
- Push the image.

---

## Example 13: GitLab CI with Branch-Named Tags

    stages:
      - build

    variables:
      IMAGE_NAME: "myapp"

    build_image:
      stage: build
      script:
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
        - IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_pipeline_ID}"
        - echo "IMAGE_tag=$IMAGE_TAG"
        - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        - docker push ${IMAGE_NAME}:${IMAGE_TAG}

### Assuming the pipeline variables are:

- Branch: feature/login
- Commit short SHA: a1b2c3d
- Pipeline ID: 1045

Then the final Tag will be:

    feature-login-a1b2c3d-1045

---

## Example 14: GitLab CI: Pushing to Harbor

The following example represents a common practice in enterprise settings.

    stages:
      - build

    variables:
      HARBOR_HOST: "harbor.example.com"
      HARBOR_Project: "test"
      IMAGE_NAME: "order-service"

    build_image:
      stage: build
      script:
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
        - IMAGE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
        - FULL_IMAGE_NAME="${HARBOR_HOST}/${HARBORPROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
        - echo "Preparing to build image: ${FULL_IMAGE_NAME}"
        - docker login -u "$HARBOR_USER" -p "$HARBOR_PASSWORD" "$HARBOR_HOST"
        - docker build -t "${FULL_IMAGE_NAME}" .
        - docker push "${FULL IMAGE_NAME}"

### Explanation

Here, it is assumed that:

- The Harbor account credentials are configured through GitLab CI/CD Variables.
- `HARBOR_USER` and `HARBOR_PASSWORD` are not hard-coded in the repository.
- The Harbor project is named `test`.
- The service image is called `order-service`.

If the variables are set as:

- Branch = feature/login
- Commit short SHA = a1b2c3d
- Pipeline ID = 1045

The final image URL might be:

    harbor.example.com/test/order-service:feature-login-a1b2c3d-1045

---

## Example 15: GitLab CI: Using Both Unique Tags and Semantic Tags

In practical production, it is common to include both a unique Tag and a semantic Tag.

    stages:
      - build

    variables:
      HARBOR_HOST: "harbor.example.com"
      HARBORPROJECT: "prod"
      IMAGE_NAME: "order-service"
      RELEASE_TAG: "v1.2.3"

    build_image:
      stage: build
      script:
        - BRANCH_NAME=$(echo "$CI_COMMIT_REF_NAME" | tr '/' '-')
        - UNIQUE_TAG="${BRANCH_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_pipeline_ID}"
        - IMAGE_UNIQUE="${HARBOR_HOST}/${HARBORPROJECT}/${IMAGE_NAME}:${UNIQUE_TAG}"
        - IMAGE_RELEASE="${HARBOR_HOST}/${HARBOR PROJECT}/${IMAGE_NAME}:${RELEASE_TAG}"
        - docker login -u "$HARBOR_USER" -p "$HARBOR_PASSWORD" "$HARBOR_HOST"
        - docker build -t "${IMAGE_UNIQUE}" .
        - docker tag "${IMAGEUniqueId}" "${IMAGE_RELEASE}"
        - docker push "${IMAGE.unique}"
        - docker push "${IMAGERELEASE}"

### Suitable for What Scenarios

- When a truly unique and traceable Tag is required.
- When it is necessary to provide both a clear version number and semantic information for different stakeholders.

---

## Example 16: Common Mapping Strategies for Branches and Environments

Many companies do not push all branches to the same Harbor project.

Common strategies include:

### 1. Develop branch pushed to test project

    develop -> harbor/test/...

### 2. Release branch pushed to pre-release project

    release/* -> harbor/uat/...

### 3. Main or master branch pushed to production project

    main -> harbor/prod/...

The advantages of this approach are:

- Clear environmental boundaries.
- Easier control over permissions.
- Reduced risk of accidental releases.

---

## Example 17: GitLab CI: Switching Harbor Projects Based on Branch

    stages:
      - build

    variables:
      HARBOR_HOST: "harbor.example.com"
     ```markdown
branch_name=$(echo "$branch_name" | tr '/' '-')
echo "${branch_name}-${commit_sha}-${pipeline_id}"
}

IMAGE_TAG=$(generate_image_tag "$CI_COMMIT_REF_NAME" "$CI_COMMIT_SHORT_SHA" "$CI_PIPELINE_ID")

Then call it in `.gitlab-ci.yml`:

build_image:
  stage: build
  script:
    - bash ci/build.sh
```

---

### Scenario 3: Companies Use Encapsulated Common Templates
Some companies maintain standardized CI templates, for example:

```yaml
include:
  - project: devops/gitlab-ci-templates
    file: docker-build.yml
```

In this case, business projects only need to pass in some variables:

```yaml
variables:
  APP_NAME: "order-service"
  ENV_NAME: "test"
```

This might seem like “calling an internal function,” but in reality, it means:

- Using a common CI template
- The template already includes logic for login, building, tagging, and pushing

---

## Chapter 19: The Relationship Between Harbor’s Governance Capabilities and Tag Management

Harbor is not just used for storing images; it can also be utilized for various governance tasks.

### 1. Tag Retention
Harbor allows you to configure rules for retaining tags.
This is crucial because frequent builds can quickly result in a large number of tags, leading to storage issues.

Example scenarios:

- Retain only the most recent N tags.
- Keep tags from the last N days.
- Permanently retain release/prod-related tags.

---

### 2. Tag Immutability
Harbor supports configuring rules to ensure tags remain unchanged.
This is essential for production tags.

For example:

- Once `v1.2.3` is released, it should not be overwritten.
- `prod-*` tags must not be repushed and overwritten.

This prevents:

- Duplicate tags with the same name.
- Distorted release records.
- Loss of rollback references.

---

### 3. Project Configuration
Harbor projects can also be configured for:

- Public or private access.
- Vulnerability scanning.
- Content trust settings.
- Tag management options.
- Permission models.

This means that image governance is not just about CI; it also involves repository management.

---

## Chapter 20: Recommended Practices in Real Production Environments

## 1. Do Not Store Sensitive Information in the Repository
Examples include:

- Harbor usernames and passwords.
- Tokens.
- Registry credentials.

These should be stored in GitLab’s CI/CD Variables.

---

## 2. Use Unique Tags for Actual Deployments
For example:

`main-a1b2c3d-1045`

Instead of using generic tags like `latest`, `staging`, or `prod`, as these can become ambiguous over time.

---

## 3. Use Semantic Tags as Supplementary Information
Tags like `v1.2.3`, `rc`, or `prod` can be retained, but they should not replace unique tags.

---

## 4. Apply Immutability Rules to Official Release Tags
Once an official release tag is generated, it should not be allowed to be overwritten.

---

## 5. Configure Retention Rules to Clean Up Ineffective Images
For example, only retain:

- Test images from the last 30 days.
- The latest 100 development tags.
- Permanent release versions.

---

## 6. Maintain a Mapping Relationship in the Release System
It is recommended to record at least:

- Branch name.
- Commit SHA.
- Pipeline ID.
- Full image path.
- Release time.
- Release environment.
- Release owner.

This ensures complete traceability and auditability.

---

## Chapter 21: Common Misconceptions

### Misconception 1: Image Management Is Simply About Giving Images Names
Wrong. Image management is essentially about product governance, which includes:

- Path planning.
- Tag design.
- Permission control.
- Retention strategies.
- Rollback auditing.

---

### Misconception 2: Having `latest` Is Enough
Wrong. `latest` is not suitable as the sole basis for production environments.

---

### Misconception 3: Tags with Only Commit SHA Are Sufficient
Not necessarily. If the same commit results in multiple builds, it’s better to include the pipeline ID as well.

---

### Misconception 4: Branch Names Can Be Directly Used in Tags
It is not recommended. Standardization should be applied first.

---

### Misconception 5: Harbor Projects Can Be Arranged Anyway
Wrong. Project segmentation directly affects:

- Permission models.
- Environmental isolation.
- Operational complexity.
- Risks associated with error-prone repositories.

---

### Misconception 6: A Successful Build Means the Tag Scheme Is Correct
Not necessarily. The effectiveness of a tag scheme depends on whether it is:

- Traceable.
- Rollbackable.
- Suitable for governance.
- In line with branch and