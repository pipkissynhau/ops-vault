# 12-BuildKit, BuildX, and Advanced Image Building

#Docker #BuildKit #buildx #Image Building #Multi-Architectural Images #Build Caching #CI-CD #Image Optimization #Operations and Maintenance

---

## Recommended Path

03-Container Technology/12-BuildKit, BuildX, and Advanced Image Building.md

---

## I. Document Overview

This document compiles information related to Docker BuildKit, BuildX, and advanced image building, focusing on the following topics:

- What is BuildKit?
- Differences between BuildKit and traditional docker build
- What is BuildX?
- Common BuildX commands
- Builder instance management
- Multi-architecture image building
- `linux/amd64` versus `linux/arm64`
- Build caching optimization
- `.dockerignore` and build context
- `RUN --mount=type=cache`
- `--cache-from`
- `--cache-to`
- Registry caching
- Local caching
- Inline caching
- Using secrets during building
- `RUN --mount=type=secret`
- `--secret`
- Using SSH during building
- `--ssh`
- How to use BuildKit/BuildX in CI/CD
- Common issues and troubleshooting

The goal is:

- To understand the role of BuildKit
- To be able to use BuildX for image building
- To know how to build multi-architecture images
- To optimize build caching
- To avoid including secrets in Dockerfiles
- To improve image building efficiency in CI/CD processes
- To establish more standardized procedures for production image creation

---

## II. What is BuildKit?

BuildKit is Docker's new generation of image building backend.

It can be understood as:

```text
BuildKit = A more powerful Docker image building engine
```

Traditional building methods:

```text
Build step by step according to the Dockerfile sequence
Relatively basic caching capabilities
Limited advanced building features
```

BuildKit building methods:

```text
Better cache management
Support for parallel building
Ability to skip unused stages
Support for cache mounts
Support for secret mounts
Support for SSH mounts
More flexible build output options
More suitable for CI/CD and complex projects
```

In one sentence:

```text
Regular docker build
→ Can still build images

BuildKit / BuildX
→ Faster, more flexible, and better suited for production pipeline building
```

---

## III. What Problems Does BuildKit Solve?

---

## Scenario 1: Speeding Up Image Building

BuildKit can make better use of caching and handle some independent building steps in parallel.

Common benefits:

```text
Faster repeated builds
Cachable dependency installations
More efficient multi-stage builds
Optimized build context transmission
```

---

## Scenario 2: Reducing Unnecessary Builds

During multi-stage building, if certain stages are ultimately not used, BuildKit can skip them.

For example:

```dockerfile
FROM alpine:3.20 AS base

RUN echo "base"

FROM alpine:3.20 AS test

RUN echo "test"

FROM alpine:3.20 AS prod

RUN echo "prod"
```

If only the `prod` stages need to be built, the unused stages will not be executed in full.

---

## Scenario 3: Using Build Secrets More Safely

It is not recommended to include secrets directly in Dockerfiles:

```dockerfile
ARG TOKEN
RUN echo "$TOKEN" > /app/token.txt
```

Reasons:

```text
Secrets may end up in the image layer
They may be included in build history
They could potentially be pushed to a container registry
```

BuildKit supports secret mounts:

```dockerfile
RUN --mount=type=secret,id=npm_token \
    cat /run/secrets/npm_token
```

During building, pass the secret as follows:

```bash
docker buildx build \
  --secret id=npm_token,src=.npm_token \
  -t myapp:v1 .
```

Features:

```text
Secrets are only temporarily available during the building process
They are not written into the final image by default
They should not be included in the image layer
This approach is better for accessing private repositories or dependencies
```

---

## IV. Enabling and Checking BuildKit

---

## Scenario 4: Checking Docker Version

```bash
docker version
```

To view detailed Docker information:

```bash
docker info
```

---

## Scenario 5: Temporarily Enabling BuildKit for Building

```bash
DOCKER_BUILDKIT=1 docker build -t myapp:v1 .
```

Explanation:

```text
DOCKER_BUILDKIT=1
→ Temporarily enables BuildKit for building
```

---

## Scenario 6: Using BuildX for Building

```bash
docker buildx build -t myapp:v1 .
```

Explanation:

```text
docker```bash
docker buildx build \
  --no-cache \
  -t myapp:v1 .
```

---

## Scenario 20: Build and Load to Local Docker

```bash
docker buildx build \
  --load \
  -t myapp:v1 .
```

Explanation:

```text
--load
→ Loads the build results into the local Docker image registry
```

View images:

```bash
docker images
```

---

## Scenario 21: Build and Push to Harbor

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Explanation:

```text
--push
→ Pushes the built image directly to the registry after completion
```

---

## Scenario 22: Apply Multiple Tags Simultaneously

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 \
  -t 10.0.0.10:8090/project/myapp:latest \
  .
```

Production Recommendation:

```text
The `latest` tag can be used as a secondary option,
but it is not recommended for production deployments.
```

---

## Chapter 9: Building Multicore Architecture Images

---

## Scenario 23: What are Multicore Architecture Images?

Common CPU architectures include:

```text
amd64
arm64
```

Common platform notations are:

```text
linux/amd64
linux/arm64
```

Multicore architecture images can be understood as:

```text
The same image name and tag
containing versions for different platforms such as amd64 and arm64
```

For example:

```text
myapp:v1
→ linux/amd64
→ linux/arm64
```

When machines with different architectures pull the same image tag, they will automatically select the version suitable for their platform.

---

## Scenario 24: Checking Supported Platforms by the Builder

```bash
docker buildx inspect --bootstrap
```

Check the `Platforms` section in the output:

```text
Platforms
```

---

## Scenario 25: Building an amd64 Image

```bash
docker buildx build \
  --platform linux/amd64 \
  --load \
  -t myapp:amd64 .
```

---

## Scenario 26: Building an arm64 Image

```bash
docker buildx build \
  --platform linux/arm64 \
  --load \
  -t myapp:arm64 .
```

---

## Scenario 27: Building and Pushing Multicore Architecture Images

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Explanation:

```text
--platform linux/amd64,linux/arm64
→ Builds images for both amd64 and arm64 architectures

--push
→ Pushes the resulting images to a registry or repository
```

---

## Scenario 28: Why is it Usually Necessary to Push Multicore Architecture Images?

Multicore architecture images essentially contain multiple versions tailored for different platforms.

In production, these images are typically pushed directly to a registry:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Subsequently, machines with different architectures can pull the appropriate images:

```bash
docker pull 10.0.0.10:8090/project/myapp:v1
```

Kubernetes nodes can also download the correct image based on their own architecture.

---

## Scenario 29: Checking Image Manifest

```bash
docker buildx imagetools inspect 10.0.0.10:8090/project/myapp:v1
```

Key aspects to check include:

```text
Whether it includes versions for linux/amd64 and linux/arm64
Whether the image digest is correct
Whether the image tag is accurate
```

---

## Scenario 30: Precautions for Multicore Architecture Building

Not all projects can successfully implement multicore architecture building without issues.

Key considerations include:

```text
Whether the base image supports the target architectures
Whether application dependencies are compatible with the target architectures
Whether the compilation toolchain is suitable for the target architectures
The presence of any native dependencies
The existence of x86-specific binaries
Whether cross-compilation is required
```

Example:

```dockerfile
FROM alpineCOPY requirements.txt .

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]## fourteen, Using SSH During Building

---

## Scenario 52: Applicable Scenarios for SSH Mounting

When building images, it may be necessary to access private Git repositories:

```text
git clone private repository
go mod download private modules
npm / pip / maven access private dependencies
```

It is not recommended to copy the private key into the image:

```dockerfile
COPY id_rsa /root/.ssh/id_rsa
```

This is extremely dangerous.

SSH mounting is a much better option.

---

## Scenario 53: Using SSH Mounting in Dockerfiles

```dockerfile
# syntax=docker/dockerfile:1

FROM alpine:3.20

RUN apk add --no-cache git openssh-client

RUN --mount=type=ssh \
    git clone git@git.example.com:project/private-repo.git /src
```

To build the image:

```bash
docker buildx build \
  --ssh default \
  -t ssh-demo:v1 .
```

Explanation:

```text
--ssh default
→ Uses the local SSH agent
```

---

## Scenario 54: Verifying the Local SSH Agent

To check the SSH agent:

```bash
ssh-add -l
```

If the private key is not loaded:

```bash
ssh-add ~/.ssh/id_rsa
```

To build the image:

```bash
docker buildx build \
  --ssh default \
  -t myapp:v1 .
```

---

## fifteen, BuildKit / buildx in CI/CD

---

## Scenario 55: Recommended CI/CD Construction Process

```text
Pull code
→ Log in to Harbor
→ Create or initialize a buildx builder
→ Use buildx to build the image
→ Accelerate building using registry cache
→ Scan the image with Trivy / Scout
→ Push the image to Harbor
→ Deploy it to testing / pre-release / production
```

---

## Scenario 56: Example CI/CD Build Command

```bash
docker buildx build \
  --platform linux/amd64 \
  --push \
  --cache-from type=registry,ref=10.0.0.10:8090/project/myapp:buildcache \
  --cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max \
  -t 10.0.0.10:8090/project/myapp:${CI_COMMIT_SHORT_SHA} \
  .
```

Explanation:

```text
--platform linux/amd64
→ Builds an amd64 image

--push
→ Pushes the built image to Harbor after completion

--cache-from
→ Loads the build cache from Harbor

--cache-to
→ Saves this build cache back to Harbor

${CI_COMMIT_SHORT_SHA}
→ Uses the commit short ID as the image tag
```

---

## Scenario 57: Example of Multi-tag Building

```bash
docker buildx build \
  --platform linux/amd64 \
  --push \
  --cache-from type=registry,ref=10.0.0.10:8090/project/myapp:buildcache \
  --cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max \
  -t 10.0.0.10:8090/project/myapp:${CI_COMMIT_SHORT_SHA} \
  -t 10.0.0.10:8090/project/myapp:${CI_pipeline_ID} \
  .
```

Recommended tag structure:

```text
Branch name
commitID
pipelineID
Version number
Build time
```

It is not recommended to use only `latest` for production environments.

---

## Scenario 58: Using Secrets in CI/CD

Example:

```bash
docker buildx build \
  --secret id=npmrc,src=.npmrc \
  --push \
  -t 10.0.0.10:8090/project/myapp:${CI_COMMIT_SHORT_SHA} \
  .
```

Dockerfile example:

```dockerfile
# syntax=docker/dockerfile:1

FROM node:20 AS deps

WORKDIR /app

COPY package.json package-lock.json ./

RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

Explanation:

```text
.npmrc is used to access a private npm repository
The secret is only temporarily used in the npm ci step
It should not be included in the final image
```

---

## sixteen, Clearing BuildKit Caches

---

## Scenario 59: Checking Disk Usage of Docker Builder

```bash
docker buildx du
```

---

## Scenario The account does not have push permissions.
Harbor does not support or restrict the relevant media type.
HTTP Harbor is not configured with trust.
The cache tag path was entered incorrectly.

First, log in:

```bash
docker login 10.0.0.10:8090
```

Test pushing a regular image:

```bash
docker push 10.0.0.10:8090/project/myapp:v1
```

Then test caching:

```bash
docker buildx build \
  --push \
  --cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

---

## Chapter 18: Best Practices for Production

---

## 1. Prefer using buildx to create production images

Recommended:

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Reasons:

```text
It offers more comprehensive capabilities.
It is better suited for CI/CD pipelines.
It makes it easier to integrate with multiple architectures.
It facilitates access to external caches.
It simplifies the use of secret mounts.
```

---

## 2. Build context must be carefully controlled

Always use:

```text
.dockerignore
```

Avoid including:

```text
.git
node_modules
target
dist
.env
Private keys
Certificates
Logs
Large files
```

When entering the build context.

---

## 3. Secrets must be managed using secret mounts

Do not recommend:

```dockerfile
ARG TOKEN
ENV TOKEN=xxxxx
COPY id_rsa /root/.ssh/id_rsa
```

Recommended:

```bash
docker buildx build \
  --secret id=mytoken,src=./mytoken.txt \
  -t myapp:v1 .
```

Or:

```bash
docker buildx build \
  --ssh default \
  -t myapp:v1 .
```

---

## 4. Use registry caches in CI/CD pipelines

Recommended:

```bash
docker buildx build \
  --push \
  --cache-from type=registry,ref=10.0.0.10:8090/project/myapp:buildcache \
  --cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max \
  -t 10.0.0.10:8090/project/myapp:${CI_COMMIT_SHORT_SHA} \
  .
```

Benefits:

```text
Caches can be reused across different pipelines.
Construction speed is improved.
It does not rely on local cache storage.
It is suitable for self-hosted GitLab, Jenkins, or Harbor environments.
```

---

## 5. Clearly specify the platform for multi-architecture images

Example:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

In production, ensure to verify:

```text
The architecture of K8s nodes.
Whether the base image supports the required architecture.
Whether the application dependencies are compatible with the architecture.
Whether the private image repository is functioning correctly for saving manifest files.
```

---

## 6. Image tags should be traceable

It is not recommended to use only `latest` in production:

```text
latest
```

A more advisable practice is to use:

```text
branch-name-commit-ID-pipeline-id
Version number
Date-commit-ID
```

Example:

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:main-a1b2c3d-1024 \
  .
```

---

## Chapter 19: Summary of Common Commands Used in This Chapter

---

## Basic BuildKit/buildx Commands

Check the Docker version:

```bash
docker version
```

View Docker information:

```bash
docker info
```

Temporarily enable BuildKit:

```bash
DOCKER_BUILDKIT=1 docker build -t myapp:v1 .
```

View the buildx version:

```bash
docker buildx version
```

List available builders:

```bash
docker buildx ls
```

Inspect the current builder:

```bash
docker buildx inspect
```

Initialize a builder:

```bash
docker buildx inspect --bootstrap
```

Create a new builder:

```bash
docker buildx create --name mybuilder --use
```

Create a docker-container builder:

```bash
docker buildx-t myapp:v1.

Using SSH:

```bash
docker buildx build \
  --ssh default \
  -t myapp:v1.
```

Viewing the SSH agent:

```bash
ssh-add -l
```

Loading the private key:

```bash
ssh-add ~/.ssh/id_rsa
```

---

## CI/CD Construction Example

```bash
docker buildx build \
  --platform linux/amd64 \
  --push \
  --cache-from type=registry,ref=10.0.0.10:8090/project/myapp:buildcache \
  --cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max \
  -t 10.0.0.10:8090/project/myapp:${CI_COMMIT_SHORT_SHA} \
  -t 10.0.0.10:8090/project/myapp:${CI_PIPELINE_ID} \
  .
```

---

## Summary in One Sentence

The core value of BuildKit / buildx is:

```text
To make Docker image building faster, safer, and more suitable for CI/CD processes.
```

Ordinary construction process:

```text
docker build
→ Builds a single-architecture image
→ Uses it locally or pushes it manually
```

Advanced construction process:

```text
docker buildx build
→ Utilizes BuildKit
→ Applies caching strategies
→ Uses secret mounts
→ Supports multiple architectures
→ Pushes the built images directly to Harbor
→ Allows CI/CD to reuse cache
```

Understanding multi-architecture construction:

```text
linux/amd64
→ A common architecture for x86_64 servers

linux/arm64
→ Common for ARM servers or Apple Silicon devices

docker buildx build --platform linux/amd64,linux/arm64
→ Builds multiple architecture images in one process
```

Caching optimization:

```text
.dockerignore
→ Reduces the build context size

Proper COPY order
→ Increases cache hit rates

RUN --mount=type=cache
→ Caches dependent files and directories

--cache-from / --cache-to
→ Allows reuse of built caches in CI/CD or Harbor
```

Security considerations:

```text
Do not pass private keys via ARG/ENV variables
Never include private keys in the image itself
Use --secret to transfer build secrets
Employ --ssh for accessing private Git repositories
Secrets and SSH connections are only temporary during the build process
```

Production recommendations:

```text
For production images, prefer using buildx for construction
Utilize registry caches within CI/CD pipelines
Specify the platform clearly when building multi-architecture images
Use .dockerignore to control the build context effectively
Ensure that build secrets are transferred securely via secret mounts
Use ssh mounts for private Git access
Make sure image tags are traceable
After construction, perform scanning, pushing, and verification steps
```