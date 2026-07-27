# Interview Question 47: Practical Notes on Image Building, Tag Generation, and Push from GitLab CI to Harbor

## Tags
#CICD #GitLabCI #Harbor #Docker #Image Management #Tag Management #DevOps #Product Governance #Interview Questions

## I. What Problem Does This Note Solve?

This article focuses on one main objective:

1. Code is submitted to GitLab.
2. GitLab Runner executes the pipeline.
3. An image tag is generated within the pipeline.
4. The `docker build` command is executed.
5. The resulting image is pushed to Harbor.
6. Understanding the relationship between Project, Repository, and Tag in Harbor.

The goal is not to create complex platforms but to clearly explain how image tags are generated and how to configure `.gitlab-ci.yml`.

---

## II. Remember the Most Critical Point First

In GitLab CI, the generation of an image tag usually does not depend on some mysterious “internal function.” Instead, it involves:

- Variables are injected when the GitLab Runner runs a job.
- These variables are read within the `script` section of `.gitlab-ci.yml`.
- The variables are concatenated into a string using shell commands.
- This string is then used as the image tag for `docker build` and `docker push`.

In other words, the process essentially consists of:

**Variables + Shell concatenation + Docker commands**

---

## III. What Does a Complete Image Address Look Like?

A complete image address typically follows this format:

    registry/project/repository:tag

For example:

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

Therefore, image governance involves not only understanding tags but also knowing how to organize Harbor projects, name repositories, and generate tags properly.

---

## IV. Why Is Managing Image Tags Important?

There are four key reasons for managing image tags effectively:

### 1. Uniqueness
Each build should be uniquely identified to prevent overwriting of existing images.

### 2. Traceability
A tag allows you to easily trace back to the following information:

- Which branch it came from.
- Which commit produced it.
- Which pipeline executed it.

### 3. Rollback Capability
In case of issues, you can quickly revert to a stable previous version using the image tag.

### 4. Governance
It facilitates better control over Harbor settings, such as retention policies, immutability rules, and access management based on projects.

---

## V. Why Isn’t `latest` Recommended for Production Use?

It’s not that `latest` cannot be used, but it is not suitable as the sole identifier in a production environment. The reasons are:

- It is not unique.
- Its value may change over time.
- It makes auditing difficult.
- It hinders rollback processes.
- It can cause confusion when multiple people work on the same project.

For example:

    myapp:latest

The name might remain the same today and tomorrow, but the actual image content could be completely different. Therefore, it is better to use a unique tag for production deployments.

---

## VI. Common Ways to Design Tags

### 1. Only Use the Commit Short SHA
Example:

    a1b2c3d

Advantages:

- Simple and straightforward.
- Directly corresponds to the code commit.

Disadvantages:

- Cannot distinguish between different builds of the same commit.

---

### 2. Commit Short SHA + Pipeline ID
Example:

    a1b2c3d-1045

Advantages:

- Higher uniqueness.
- Allows distinguishing between multiple builds of the same commit.
- Provides traceability to specific pipelines.

---

### 3. Branch + Commit Short SHA + Pipeline ID
Example:

    main-a1b2c3d-1045
    develop-a1b2c3d-1045
    feature-login-a1b2c3d-1045

Advantages:

- Clearly indicates the branch origin.
- Allows tracking of code and pipeline history.
- Highly suitable for testing environments and multi-branch development scenarios.

This is a common practice in production settings.

---

### 4. Semantic Versioning
Example:

    v1.2.3

Advantages:

- User-friendly and suitable for official releases.

Disadvantages:

- When used alone, it may reduce the ability to trace back to specific commits and pipelines.

---

## VII. Recommended Combined Approach

In production environments, a “dual-tag” approach is recommended:

### First Layer: Unique Tag
Used for deployment, auditing, and rollback purposes.

Example:

IMAGE_TAG = feature-login-a1b2c3d-1045

The final commands would be approximately:

docker build -t myapp:feature-login-a1b2c3d-1045 .
docker push myapp:feature-login-a1b2c3d-1045---  

## Twenty-Five: External Links  

- GitLab CI/CD Predefined Variables  
- GitLab CI/CD YAML Syntax  
- Using Variables in GitLab Job Scripts  
- Harbor Image and Tag Management  
- Harbor Tag Retention Rules  
- Harbor Tag Invariant Rules