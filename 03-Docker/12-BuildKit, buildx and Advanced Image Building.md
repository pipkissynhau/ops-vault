# 12-BuildKit, buildx, and Advanced Image Building

#Docker #BuildKit #buildx #MirrorBuild #MultistructureMirror #BuildCache #CI-CD #MirrorOptimization #Transport

---

## Recommended Path

03-Container Technology/12-BuildKit, buildx, and Advanced Image Building.md

---

## I. Document Overview

This document organizes content about Docker BuildKit, buildx, and advanced image building, focusing on:

- What is BuildKit
- Differences between BuildKit and traditional docker build
- What is buildx
- Common buildx commands
- Builder instance management
- Multi-architecture image building
- `linux/amd64` and `linux/arm64`
- Build cache optimization
- `.dockerignore` and build context
- `RUN --mount=type=cache`
- `--cache-from`
- `--cache-to`
- registry cache
- local cache
- inline cache
- Using secrets during build
- `RUN --mount=type=secret`
- `--secret`
- Using SSH during build
- `--ssh`
- Using BuildKit/buildx in CI/CD
- Common issues and troubleshooting

Goals:

- Understand BuildKit's role
→ Be able to use buildx to build images
→ Be able to build multi-architecture images
→ Be able to optimize build cache
→ Be able to avoid writing secrets into Dockerfile
→ Be able to improve image building efficiency in CI/CD
→ Establish more standardized processes for production image building

---

## II. What is BuildKit

BuildKit is Docker's new generation image building backend.

You can understand it as:

```text
BuildKit = Stronger Docker Mirror Build Engine
```

Traditional building method:

```text
Press Dockerfile Build sequentially step by step
Relative basis for cache capacity
Low advanced build capacity
```

BuildKit building method:

```text
Better cache management
Support parallel construction
Support skipping unused stages
Support cache mount
Support secret mount
Support ssh mount
Support more flexible build output
It's more appropriate. CI/CD and complex project construction
```

One-sentence understanding:

```text
Normal docker build
→ Can build mirrors

BuildKit / buildx
→ Faster, more flexible and more suitable for the production of current line construction
```

---

## III. What Problems Does BuildKit Solve

---

## Scenario 1: Accelerate Image Building Speed

BuildKit can better utilize caching and process some independent build steps in parallel.

Common benefits:

```text
Repeat construction faster
Cache with installation
More efficient multi-stage construction
Build context transfer optimized
```

---

## Scenario 2: Reduce Useless Builds

During multi-stage builds, if some stages are not used in the end, BuildKit can skip unused stages.

Example:

```dockerfile
FROM alpine:3.20 AS base

RUN echo "base"


FROM alpine:3.20 AS test

RUN echo "test"


FROM alpine:3.20 AS prod

RUN echo "prod"
```

If only building `prod` related stages, unused stages may not be fully executed.

---

## Scenario 3: Use Build Secrets More Safely

Not recommended:

```dockerfile
ARG TOKEN
RUN echo "$TOKEN" > /app/token.txt
```

Reason:

```text
Keys may enter mirror layers
Keys may enter build history
The key could be pushed. Harbor
```

BuildKit supports secret mount:

```dockerfile
RUN --mount=type=secret,id=npm_token \
    cat /run/secrets/npm_token
```

Passing secrets during build:

```bash
docker buildx build \
  --secret id=npm_token,src=.npm_token \
  -t myapp:v1 .
```

Features:

```text
secret Only temporary in build steps
Do not write the final mirror by default
Shouldn't go into the mirror.
Better suited to visit private warehouses or private dependence
```

---

## IV. Enabling and Viewing BuildKit

---

## Scenario 4: Check Docker Version

```bash
docker version
```

Check Docker information:

```bash
docker info
```

---

## Scenario 5: Temporarily Enable BuildKit Building

```bash
DOCKER_BUILDKIT=1 docker build -t myapp:v1 .
```

Note:

```text
DOCKER_BUILDKIT=1
→ Temporary Enable BuildKit Build
```

---

## Scenario 6: Use buildx to Build

```bash
docker buildx build -t myapp:v1 .
```

Note:

```text
docker buildx build
→ Use buildx Build mirrors
```

---

## Scenario 7: Check buildx Version

```bash
docker buildx version
```

---

## V. What is buildx

buildx is a Docker CLI build extension tool.

You can understand it as:

```text
buildx = Docker Advanced Build Command Entry
```

Normal build:

```bash
docker build -t myapp:v1 .
```

buildx build:

```bash
docker buildx build -t myapp:v1 .
```

buildx is commonly used for:

```text
Multi-structure mirror construction
Remote builder
Build Cache Import Export
Build Results Direct push
Advanced BuildKit Functions
CI/CD Build Optimization
```

---

## VI. buildx Builder Management

---

## Scenario 8: View Builder List

```bash
docker buildx ls
```

Note:

```text
See what's currently available. builder
Which is currently being used builder
builder Use what? driver
Which platforms are supported
```

---

## Scenario 9: View Current Builder Info

```bash
docker buildx inspect
```

Initialize and view:

```bash
docker buildx inspect --bootstrap
```

Note:

```text
--bootstrap
→ Initialize builder, and display information such as available platforms
```

---

## Scenario 10: Create New Builder

```bash
docker buildx create --name mybuilder --use
```

Note:

```text
--name mybuilder
→ builder Name

--use
→ Switch to use this directly after creation builder
```

---

## Scenario 11: Create docker-container Driver Builder

```bash
docker buildx create \
  --name mybuilder \
  --driver docker-container \
  --use
```

Note:

```text
docker-container driver
→ Run using packagings BuildKit builder
→ Better suited to advanced construction, multiplatform construction,CI/CD scene
```

View:

```bash
docker buildx ls
```

Initialize:

```bash
docker buildx inspect --bootstrap
```

---

## Scenario 12: Switch Builder

```bash
docker buildx use mybuilder
```

---

## Scenario 13: Delete Builder

```bash
docker buildx rm mybuilder
```

---

## VII. Build Context and .dockerignore

---

## Scenario 14: What is Build Context

Execute:

```bash
docker build -t myapp:v1 .
```

Final dot:

```text
.
```

Indicates current directory as build context.

You can understand build context as:

```text
docker build File range visible
```

In Dockerfile:

```dockerfile
COPY . /app
```

Only files within build context can be copied.

---

## Scenario 15: Large Build Context Causes Slower Builds

Common useless files:

```text
.git
node_modules
dist
target
logs
*.log
.tmp
.cache
.env
Large compression package
Test Data
```

If these files enter build context, it will cause:

```text
Slows construction context transfer
Cache is easier to fail
Mirror construction slows down
Sensitivity files wrongly entered the mirror.
```

---

## Scenario 16: Optimize Build Context with .dockerignore

`.dockerignore` Example:

```text
.git
.gitignore
node_modules
dist
target
*.log
*.tmp
.env
.env.*
id_rsa
*.pem
.cache
```

Check current directory:

```bash
ls -lah
```

Check `.dockerignore`:

```bash
cat .dockerignore
```

Build:

```bash
docker buildx build -t myapp:v1 .
```

---

## VIII. Basic buildx Build Commands

---

## Scenario 17: Build Image

```bash
docker buildx build -t myapp:v1 .
```

---

## Scenario 18: Specify Dockerfile

```bash
docker buildx build \
  -f Dockerfile.prod \
  -t myapp:prod .
```

---

## Scenario 19: Build Without Cache

```bash
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

Note:

```text
--load
→ Load build results locally Docker Mirror List
```

Check image:

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

Note:

```text
--push
→ Upon completion of construction, push to mirror warehouse
```

---

## Scenario 22: Tag Multiple Images at Once

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 \
  -t 10.0.0.10:8090/project/myapp:latest \
  .
```

Production recommendation:

```text
latest Could be used as a supporting label
But production deployment is not recommended to rely on latest
```

---

## IX. Multi-Architecture Image Building

---

## Scenario 23: What is Multi-Architecture Image

Common CPU architectures:

```text
amd64
arm64
```

Common platform notations:

```text
linux/amd64
linux/arm64
```

Multi-architecture image can be understood as:

```text
Same mirror name and tag
Organisation amd64 and arm64 Waiting for mirror versions of different platforms
```

Example:

```text
myapp:v1
→ linux/amd64
→ linux/arm64
```

When different architecture machines pull the same image tag, they will automatically select the appropriate image for their platform.

---

## Scenario 24: Check Supported Platforms by Builder

```bash
docker buildx inspect --bootstrap
```

View the output in:

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

## Scenario 27: Building and Pushing Multi-Architecture Images

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Note:

```text
--platform linux/amd64,linux/arm64
→ Build simultaneously amd64 and arm64

--push
→ Push it. Harbor or other registry
```

---

## Scenario 28: Why Multi-Architecture Builds Usually Need to Push

Multi-architecture images essentially contain multiple platform image manifests.

In production, it's typically directly pushed to the image registry:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Then different architecture nodes pull:

```bash
docker pull 10.0.0.10:8090/project/myapp:v1
```

Kubernetes nodes can also pull the corresponding image based on their architecture.

---

## Scenario 29: Viewing Image Manifest

```bash
docker buildx imagetools inspect 10.0.0.10:8090/project/myapp:v1
```

Common focus:

```text
Include linux/amd64
Include linux/arm64
Mirror digest Correct?
Mirror tag Correct?
```

---

## Scenario 30: Notes on Multi-Architecture Builds

Multi-architecture builds aren't always successful for all projects.

Need to pay attention to:

```text
Whether basic mirrors support target structures
Application of dependency to support target architecture
Whether the compilation tool chain supports the target structure
Existence native Dependency
Existence x86 Exclusive Binary
Need for cross-compilation
```

Example:

```dockerfile
FROM alpine:3.20
```

If the base image supports multi-architecture, the build will be smoother.

If using a private base image, also confirm it has corresponding architecture versions.

---

## Ten: Build Cache Foundation

---

## Scenario 31: Basic Principles of Docker Build Caching

Each step in Dockerfile may form a cache.

For example:

```dockerfile
FROM node:20

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci

COPY . .

RUN npm run build
```

If:

```text
package.json It hasn't changed.
package-lock.json It hasn't changed.
```

Then:

```dockerfile
RUN npm ci
```

This step may reuse the cache.

If written as:

```dockerfile
COPY . .
RUN npm ci
```

Then any change in source code files may cause the dependency installation cache to become invalid.

---

## Scenario 32: Cache Optimization Principles

```text
Less variable steps ahead
We've got a lot of different steps behind us.
File Dependence First COPY
After Source File COPY
Reliance on a separate layer of installation
Build product and run mirror separation
Use .dockerignore Reduce Context
```

---

## Scenario 33: Node.js Cache Optimization Example

Not recommended:

```dockerfile
FROM node:20

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build

CMD ["node", "dist/main.js"]
```

More recommended:

```dockerfile
FROM node:20

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci

COPY . .

RUN npm run build

CMD ["node", "dist/main.js"]
```

---

## Scenario 34: Python Cache Optimization Example

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

Note:

```text
requirements.txt Copy first.
Reliance first installation
Copy after Source
```

This way, dependencies may not be reinstalled every time the source code changes.

---

## Scenario 35: Go Cache Optimization Example

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /src

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN go build -o app main.go


FROM alpine:3.20

WORKDIR /app

COPY --from=builder /src/app /app/app

CMD ["/app/app"]
```

Note:

```text
go.mod / go.sum Copy first.
go mod download Cache first
Copy after Source
```

---

## Eleven: RUN --mount=type=cache

---

## Scenario 36: What is cache mount

BuildKit supports mounting cache directories during the build process.

Format:

```dockerfile
RUN --mount=type=cache,target=Cache Directory Command
```

Can be understood as:

```text
Some download-dependent directories can be reused across construction
But it won't be written directly into the final mirror layer.
```

Suitable for:

```text
apt Cache
npm Cache
pip Cache
go build Cache
go module Cache
maven Cache
```

---

## Scenario 37: apt cache mount example

```dockerfile
# syntax=docker/dockerfile:1

FROM ubuntu:22.04

RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y curl
```

Note:

```text
/var/cache/apt
→ apt Package Cache

/var/lib/apt
→ apt Metadata Cache
```

---

## Scenario 38: npm cache mount example

```dockerfile
# syntax=docker/dockerfile:1

FROM node:20

WORKDIR /app

COPY package.json package-lock.json ./

RUN --mount=type=cache,target=/root/.npm \
    npm ci

COPY . .

RUN npm run build
```

Note:

```text
/root/.npm
→ npm Cache Directory
```

---

## Scenario 39: pip cache mount example

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

Note:

```text
/root/.cache/pip
→ pip Download Cache
```

If pursuing a cleaner final image, you can also continue using:

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

Choose between the two approaches based on specific build environments.

---

## Scenario 40: Go cache mount example

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.23 AS builder

WORKDIR /src

COPY go.mod go.sum ./

RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

RUN --mount=type=cache,target=/root/.cache/go-build \
    go build -o app main.go


FROM alpine:3.20

WORKDIR /app

COPY --from=builder /src/app /app/app

CMD ["/app/app"]
```

Note:

```text
/go/pkg/mod
→ Go module Cache

/root/.cache/go-build
→ Go Compile Cache
```

---

## Scenario 41: Maven cache mount example

```dockerfile
# syntax=docker/dockerfile:1

FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /src

COPY pom.xml .

RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:go-offline

COPY . .

RUN --mount=type=cache,target=/root/.m2 \
    mvn package -DskipTests


FROM eclipse-temurin:17-jre

WORKDIR /app

COPY --from=builder /src/target/app.jar /app/app.jar

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Note:

```text
/root/.m2
→ Maven Local repository cache
```

---

## Twelve: External Build Cache

---

## Scenario 42: Why External Cache is Needed

When building locally, BuildKit has internal caching.

However, in CI/CD, it's often encountered:

```text
Every water line is a new machine.
Local cache does not exist
Redownload every time.
Long build time
```

In such cases, external caching can be used:

```text
inline cache
registry cache
local cache
```

---

## Scenario 43: inline cache

inline cache can embed cache information into the image.

Build and push:

```bash
docker buildx build \
  --push \
  --cache-to type=inline \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Next build use this image as the cache source:

```bash
docker buildx build \
  --cache-from type=registry,ref=10.0.0.10:8090/project/myapp:v1 \
  --push \
  -t 10.0.0.10:8090/project/myapp:v2 .
```

Suitable for:

```text
Simple CI/CD Cache
The mirror itself can be used as a cache source.
```

---

## Scenario 44: registry cache

registry cache pushes the cache to a separate cache reference in the image registry.

Build and export cache:

```bash
docker buildx build \
  --push \
  --cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Next build import cache:

```bash
docker buildx build \
  --push \
  --cache-from type=registry,ref=10.0.0.10:8090/project/myapp:buildcache \
  --cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max \
  -t 10.0.0.10:8090/project/myapp:v2 .
```

Note:

```text
myapp:v1
→ Official mirror

myapp:buildcache
→ Build Cache
```

Suitable for:

```text
CI/CD
Multibuild Nodes
The current is accelerating.
Don't want to depend on local caches.
```

---

## Scenario 45: local cache

Export cache to local directory:

```bash
docker buildx build \
  --cache-to type=local,dest=/tmp/buildkit-cache \
  -t myapp:v1 .
```

Import cache from local directory:

```bash
docker buildx build \
  --cache-from type=local,src=/tmp/buildkit-cache \
  -t myapp:v2 .
```

Suitable for:

```text
Local Debug
Self-build CI Nodes
Build node directory sustainable
```

---

## Scenario 46: Cache mode mode=min vs mode=max

Common writing:

```bash
--cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max
```

Understanding:

```text
mode=min
→ Export less cache, smaller size

mode=max
→ More caches are exported, more usable, but more caches are available
```

Common in CI/CD:

```text
mode=max
```

But note Harbor storage space.

---

## Thirteen: Build Secrets

---

## Scenario 47: Why Not Use ARG to Pass Secrets

Wrong example:

```dockerfile
ARG TOKEN

RUN echo "$TOKEN" > /app/token.txt
```

Build:

```bash
docker build \
  --build-arg TOKEN=xxxxx \
  -t myapp:v1 .
```

Risk:

```text
Keys may enter mirror layers
Keys may enter build history
Keys may be log records
The key could be pushed. Harbor
```

Not recommended to pass sensitive information via `ARG`.

---

## Scenario 48: Using secret mount

Dockerfile example:

```dockerfile
# syntax=docker/dockerfile:1

FROM alpine:3.20

RUN --mount=type=secret,id=mytoken \
    cat /run/secrets/mytoken
```

Pass secret file during build:

```bash
docker buildx build \
  --secret id=mytoken,src=./mytoken.txt \
  -t secret-demo:v1 .
```

Note:

```text
secret Default mount path:
/run/secrets/<id>
```

Example:

```text
/run/secrets/mytoken
```

---

## Scenario 49: Passing secret via environment variables

Set environment variables:

```bash
export API_TOKEN=xxxxx
```

Build:

```bash
docker buildx build \
  --secret id=API_TOKEN \
  -t myapp:v1 .
```

Read in Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1

FROM alpine:3.20

RUN --mount=type=secret,id=API_TOKEN \
    cat /run/secrets/API_TOKEN
```

---

## Scenario 50: Specifying secret mount path

```dockerfile
# syntax=docker/dockerfile:1

FROM alpine:3.20

RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm config list
```

Build:

```bash
docker buildx build \
  --secret id=npmrc,src=.npmrc \
  -t node-app:v1 .
```

Suitable for:

```text
Visit private npm Warehouse
Visit private pip Source
Visit private Maven Warehouse
Visit private Git Warehouse
```

---

## Scenario 51: Secret Usage Principles

```text
Don't. secret Write Dockerfile
Don't use it. ARG Send sensitive information
Don't. echo secret To file and keep
Don't let secret Enter final mirror
Build Log Do Not Output secret
CI/CD Use the platform Secret Variables
```

---

## Fourteen: Using SSH During Build

---

## Scenario 52: SSH mount Applicable Scenarios

When building images, you may need to access private Git repositories:

```text
git clone Private warehouse
go mod Download Private Modules
npm / pip / maven Access to private dependency
```

Not recommended to copy private keys into the image:

```dockerfile
COPY id_rsa /root/.ssh/id_rsa
```

This is very dangerous.

More recommended to use SSH mount.

---

## Scenario 53: Dockerfile Using SSH mount

```dockerfile
# syntax=docker/dockerfile:1

FROM alpine:3.20

RUN apk add --no-cache git openssh-client

RUN --mount=type=ssh \
    git clone git@git.example.com:project/private-repo.git /src
```

Build:

```bash
docker buildx build \
  --ssh default \
  -t ssh-demo:v1 .
```

Note:

```text
--ssh default
→ Use this machine SSH agent
```

---

## Scenario 54: Confirming Local SSH Agent

Check SSH agent:

```bash
ssh-add -l
```

If private key not loaded:

```bash
ssh-add ~/.ssh/id_rsa
```

Build:

```bash
docker buildx build \
  --ssh default \
  -t myapp:v1 .
```

---

## Fifteen: BuildKit / buildx Approach in CI/CD

---

## Scenario 55: Recommended Build Pipeline in CI/CD

```text
Pull Replace Code
→ Login Harbor
→ Create or Initialize buildx builder
→ buildx Build mirrors
→ Use registry cache Accelerate construction
→ Trivy / Scout Scan Mirror
→ Send Harbor
→ Deployed for testing / Advance publication / Production
```

---

## Scenario 56: CI/CD Build Command Examples

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
→ Build amd64 Mirror

--push
→ Build completed and sent Harbor

--cache-from
→ From Harbor Read Build Cache

--cache-to
→ Write back this build cache Harbor

${CI_COMMIT_SHORT_SHA}
→ Use commit Short ID As a mirror. tag
```

---

## Scenario 57: Multi-tag Build Example

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

Recommended tag design:

```text
Branch Name
commitID
pipelineID
Version Number
Build Time
```

Not recommended for production use:

```text
latest
```

---

## Scenario 58: Using Secret in CI/CD

Example:

```bash
docker buildx build \
  --secret id=npmrc,src=.npmrc \
  --push \
  -t 10.0.0.10:8090/project/myapp:${CI_COMMIT_SHORT_SHA} \
  .
```

Dockerfile:

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
.npmrc Access to private npm Warehouse
secret Only npm ci Interim use of steps
Should not enter the final mirror
```

---

## Sixteen. BuildKit Cache Cleanup

---

## Scenario 59: View Docker Builder Disk Usage

```bash
docker buildx du
```

---

## Scenario 60: Clean buildx Cache

```bash
docker buildx prune
```

Force cleanup:

```bash
docker buildx prune -f
```

Clean all unused cache:

```bash
docker buildx prune -a
```

---

## Scenario 61: Clean Traditional Builder Cache

```bash
docker builder prune
```

Force cleanup:

```bash
docker builder prune -f
```

---

## Scenario 62: Production Cleanup Notes

```text
Do not clear cache during construction peak period
Cleaning the cache will slow down subsequent construction
CI/CD Node to plan cache directories and capacity
registry cache Pay attention. Harbor Storage space
```

---

## Seventeen. Common Issues and Troubleshooting

---

## Issue 1: docker buildx Command Not Found

Check version:

```bash
docker version
```

Check buildx:

```bash
docker buildx version
```

Possible causes:

```text
Docker older version
buildx Plugin not installed
Current System Docker CLI Incomplete
```

---

## Issue 2: Docker Images Not Visible After Build

Cause:

```text
buildx Default may not load mirrors locally Docker image store
```

Solution:

```bash
docker buildx build \
  --load \
  -t myapp:v1 .
```

Check:

```bash
docker images
```

If pushing directly to registry:

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

---

## Issue 3: Multi-architecture Build Failed

Common causes:

```text
builder Target platform not supported
Basic mirrors do not support target structures
Dependence on unsupported target structure
Existence native Binary Dependence
Incorrect Cross-Compilation Configuration
```

Check builder platform:

```bash
docker buildx inspect --bootstrap
```

Check base image:

```bash
docker buildx imagetools inspect nginx:1.27
```

---

## Issue 4: COPY --from Path Error

Common error in multi-stage builds:

```dockerfile
COPY --from=builder /app/app /app/app
```

But actual output path might be:

```text
/src/app
```

Troubleshooting approach:

```text
Confirm. builder Phase WORKDIR
Confirm. RUN Build Output Path
Confirm. COPY --from Source Path
Confirm final stage target path
```

---

## Issue 5: cache mount Not Effective

Common causes:

```text
Not used BuildKit
Dockerfile Nothing. syntax Command
Cache Path Error
The cache directory was not used by the relying tool
CI/CD Every time it's new. builder No external cache
```

Recommended Dockerfile start:

```dockerfile
# syntax=docker/dockerfile:1
```

Confirm build command:

```bash
docker buildx build -t myapp:v1 .
```

---

## Issue 6: Secret Not Mounted Successfully

Common causes:

```text
Build command not passed --secret
Dockerfile Medium id Wrong
secret File path error
Environmental variables do not exist
RUN Step not used --mount=type=secret
```

Build command:

```bash
docker buildx build \
  --secret id=mytoken,src=./mytoken.txt \
  -t myapp:v1 .
```

Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1

FROM alpine:3.20

RUN --mount=type=secret,id=mytoken \
    ls -lh /run/secrets/mytoken
```

---

## Issue 7: Registry Cache Push Failed

Common causes:

```text
Not Login Harbor
Project does not exist
No account. push Permissions
Harbor Do not support or limit relevance media type
HTTP Harbor Untrusted
Cache tag Path error
```

Login first:

```bash
docker login 10.0.0.10:8090
```

Test push regular image:

```bash
docker push 10.0.0.10:8090/project/myapp:v1
```

Then test cache:

```bash
docker buildx build \
  --push \
  --cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

---

## Eighteen. Production Best Practices

---

## 1. Prioritize buildx for Production Image Builds

Recommended:

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Reason:

```text
More complete.
It's more appropriate. CI/CD
It's easier to access multiple structures.
Easy access to external caches
It's easier to access. secret mount
```

---

## 2. Control Build Context Strictly

Must use:

```text
.dockerignore
```

Avoid:

```text
.git
node_modules
target
dist
.env
Private Key
Certificate
Log
Big document
```

Entering build context.

---

## 3. Secrets Must Use secret mount

Not recommended:

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

## 4. CI/CD Use registry cache

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
You can reset the caches between different current lines.
Build faster
Do not depend on local node cache
Fit for Self-Building GitLab / Jenkins / Harbor Environment
```

---

## 5. Multi-architecture Images Must Specify Platform

Example:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

In production, confirm:

```text
K8s Node Structure
Basic mirror structure support
Application dependency architecture support
Is the private mirror warehouse kept properly? manifest
```

---

## 6. Image tags Must Be Traceable

Not recommended for production:

```text
latest
```

More recommended:

```text
Branch Name-commitID-pipelineID
Version Number
Date-commitID
```

Example:

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:main-a1b2c3d-1024 \
  .
```

---

## Nineteen. Common Commands Summary

---

## BuildKit / buildx Basics

Check Docker version:

```bash
docker version
```

Check Docker info:

```bash
docker info
```

Temporarily enable BuildKit:

```bash
DOCKER_BUILDKIT=1 docker build -t myapp:v1 .
```

Check buildx version:

```bash
docker buildx version
```

Check builder:

```bash
docker buildx ls
```

Check current builder:

§

## Build Image

Basic build:

```bash
docker buildx build -t myapp:v1 .
```

Specify Dockerfile:

```bash
docker buildx build \
  -f Dockerfile.prod \
  -t myapp:prod .
```

No cache:

```bash
docker buildx build \
  --no-cache \
  -t myapp:v1 .
```

Build and load locally:

```bash
docker buildx build \
  --load \
  -t myapp:v1 .
```

Build and push:

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

---

## Multi-architecture Build

Check image manifest:

```bash
docker buildx imagetools inspect nginx:1.27
```

Build amd64:

```bash
docker buildx build \
  --platform linux/amd64 \
  --load \
  -t myapp:amd64 .
```

Build arm64:

```bash
docker buildx build \
  --platform linux/arm64 \
  --load \
  -t myapp:arm64 .
```

Build and push multi-architecture image:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Check Harbor image manifest:

```bash
docker buildx imagetools inspect 10.0.0.10:8090/project/myapp:v1
```

---

## Build Cache

Inline cache:

```bash
docker buildx build \
  --push \
  --cache-to type=inline \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Registry cache:

```bash
docker buildx build \
  --push \
  --cache-from type=registry,ref=10.0.0.10:8090/project/myapp:buildcache \
  --cache-to type=registry,ref=10.0.0.10:8090/project/myapp:buildcache,mode=max \
  -t 10.0.0.10:8090/project/myapp:v1 .
```

Local cache export:

```bash
docker buildx build \
  --cache-to type=local,dest=/tmp/buildkit-cache \
  -t myapp:v1 .
```

Local cache import:

```bash
docker buildx build \
  --cache-from type=local,src=/tmp/buildkit-cache \
  -t myapp:v2 .
```

Check buildx disk usage:

```bash
docker buildx du
```

Clean buildx cache:

```bash
docker buildx prune
```

Force clean:

```bash
docker buildx prune -f
```

Clean all unused cache:

```bash
docker buildx prune -a
```

---

## Secret / SSH

Use secret file:

```bash
docker buildx build \
  --secret id=mytoken,src=./mytoken.txt \
  -t myapp:v1 .
```

Use environment variable secret:

```bash
export API_TOKEN=xxxxx
```

```bash
docker buildx build \
  --secret id=API_TOKEN \
  -t myapp:v1 .
```

Use SSH:

```bash
docker buildx build \
  --ssh default \
  -t myapp:v1 .
```

Check SSH agent:

```bash
ssh-add -l
```

Load private key:

```bash
ssh-add ~/.ssh/id_rsa
```

---

## CI/CD Build Example

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

## Twenty. One-sentence Summary

The core value of BuildKit / buildx is:

```text
Jean. Docker Mirrors are faster, safer, more suitable. CI/CD
```

Ordinary build pipeline:

```text
docker build
→ Build a single frame mirror
→ Locally used or manual push
```

Advanced build pipeline:

```text
docker buildx build
→ Use BuildKit
→ Use Cache
→ Use secret mount
→ Support for multi-structure
→ Direct after Build push Harbor
→ CI/CD Reusable Cache
```

Understanding multi-architecture builds:

```text
linux/amd64
→ x86_64 Common server architecture

linux/arm64
→ ARM Server or Apple Silicon Common architecture

docker buildx build --platform linux/amd64,linux/arm64
→ Construct multiple frames at once
```

Cache Optimization Understanding:

```text
.dockerignore
→ Reduce construction context

Reasonable. COPY Order
→ Increase Cache Rate

RUN --mount=type=cache
→ Cache dependent download directory

--cache-from / --cache-to
→ Yes. CI/CD or Harbor Reuse Build Cache
```

Secure Build Understanding:

```text
Don't use it. ARG / ENV Passkey
Don't. COPY Private key entry image
Use --secret Pass Build Key
Use --ssh Visit private Git
secret and ssh Only temporary for build steps
```

Production Recommendations:

```text
Production mirror priority buildx Build
CI/CD Use registry cache
Multi-structure mirrors need to be clear. platform
Build context must be used .dockerignore Control
The building key must be used secret mount
Private Git Access Usage ssh mount
Mirror tag Must be traceable.
Scan, push, verify when construction is completed
```