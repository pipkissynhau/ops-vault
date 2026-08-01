# 10-Dockerfile Basics: Image Building and Best Practices

#Docker #Dockerfile #MirrorBuild #Containerization #MirrorOptimization #CI-CD #Transport #Clear.

---

## Recommended Path

03-Container Technology/10-Dockerfile Basics: Image Building and Best Practices.md

---

## I. Document Overview

This document organizes the basic syntax of Dockerfile, common instructions, image building process, and production practices, with focus on:

- What is Dockerfile
- Difference between Dockerfile and `docker commit`
- Basic structure of Dockerfile
- Common Dockerfile instructions
- `FROM`
- `FROM ... AS ...`
- `RUN`
- `COPY`
- `ADD`
- `WORKDIR`
- `ENV`
- `ARG`
- `EXPOSE`
- `CMD`
- `ENTRYPOINT`
- `USER`
- `VOLUME`
- `LABEL`
- `HEALTHCHECK`
- `.dockerignore`
- Image building commands
- Image layering and caching
- Multi-stage building
- Common Dockerfile issues
- Production best practices

The goal is:

Can understand Dockerfile

→ Can write basic Dockerfile

→ Can build images

→ Can understand image layering and caching

→ Can understand `FROM ... AS ...` multi-stage building

→ Can reduce image size

→ Can avoid writing sensitive information into images

→ Can lay foundation for subsequent CI/CD image building

---

## II. What is Dockerfile

Dockerfile is a text file that describes how to build an image.

It can be understood as:

```text
Dockerfile = Mirror construction instructions
```

Manual way:

```text
Enter the container.
→ Install software
→ Modify Configuration
→ docker commit Save mirror
```

Dockerfile way:

```text
Write installation steps, configuration copy, start-up commands into files
→ docker build Autobuild mirrors
```

Production environment recommends Dockerfile rather than long-term dependency on `docker commit`.

---

## III. Difference between Dockerfile and docker commit

---

## 1. docker commit

Example:

```bash
docker commit ContainersID my-nginx:v1
```

Features:

- Suitable for temporarily saving state
- Opaque operation process
- Not convenient for review
- Not convenient for version management
- Not suitable for long-term production delivery

---

## 2. Dockerfile

Example:

```dockerfile
FROM nginx:latest

COPY ./html /usr/share/nginx/html

EXPOSE 80
```

Build:

```bash
docker build -t my-nginx:v1 .
```

Features:

- Clear building process
- Can be version managed
- Can be code-reviewed
- Can integrate with CI/CD
- More suitable for team collaboration and production delivery

---

## 3. One-sentence Understanding

```text
docker commit
→ Interim storage container state

Dockerfile
→ Standardised build mirrors
```

Production recommendation:

```text
The temporary barrier is working. docker commit
Official mirror construction priority Dockerfile
```

---

## IV. Basic Structure of Dockerfile

A simple Dockerfile example:

```dockerfile
FROM nginx:latest

COPY ./html /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Basic structure can be understood as:

```text
Select Basic Mirror
→ Installation Dependence
→ Copy File
→ Set Task Directory
→ Set Environmental Variables
→ Exposure Port
→ Specify start-up command
```

---

## V. Common Dockerfile Instructions

---

## Scenario 1: FROM to specify base image

### Example

```dockerfile
FROM nginx:latest
```

### Meaning

`FROM` is used to specify the base image.

Dockerfile usually must start with `FROM`.

Common base images:

```text
nginx
alpine
ubuntu
debian
node
python
openjdk
golang
```

### Example

```dockerfile
FROM ubuntu:22.04
```

```dockerfile
FROM alpine:3.20
```

```dockerfile
FROM openjdk:17-jdk
```

### Production Recommendation

Not recommended for long-term use:

```dockerfile
FROM nginx:latest
```

More recommended to fix version:

```dockerfile
FROM nginx:1.27
```

Reasons:

- `latest` is unstable
- Subsequent build results may vary
- Not conducive to rollback
- Not conducive to problem reproduction

---

## Scenario 2: FROM AS stage naming

### Example

```dockerfile
FROM golang:1.23 AS builder
```

### Meaning

`AS` is used to give a name to the current build stage.

Note:

```text
AS Not alone. Dockerfile Command
AS Yes. FROM Phase aliases in command
```

Full format:

```dockerfile
FROM Basic mirror AS Phase Name
```

Example:

```dockerfile
FROM golang:1.23 AS builder
```

Means:

```text
Use golang:1.23 As basic mirror
and name this build phase as builder
```

---

## Scenario 3: Why need FROM AS

In multi-stage building, there are usually multiple `FROM`.

Example:

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /src

COPY . .

RUN go build -o app main.go


FROM alpine:3.20

WORKDIR /app

COPY --from=builder /src/app /app/app

CMD ["/app/app"]
```

Here are two stages:

```text
Phase I:builder
→ Responsible for source code compilation

Phase II:alpine
→ Save final running file only
```

Key line is this one:

```dockerfile
COPY --from=builder /src/app /app/app
```

Means:

```text
From builder Phase Copy /src/app
to the current stage /app/app
```

---

## Scenario 4: Common AS naming

Common stage names:

```text
builder
build
deps
runtime
base
test
prod
```

Example:

```dockerfile
FROM node:20 AS deps
```

```dockerfile
FROM node:20 AS builder
```

```dockerfile
FROM nginx:1.27 AS runtime
```

### Naming Recommendation

Stage names should reflect their purpose.

Example:

```text
deps
→ Reliance on installation phase

builder
→ Compile build phase

runtime
→ Final operational phase

test
→ Test phase

prod
→ Production mirror phase
```

---

## Scenario 5: Can we skip using AS?

Yes, but not recommended.

Without stage names, we can reference by stage number.

Example:

```dockerfile
FROM golang:1.23

WORKDIR /src

COPY . .

RUN go build -o app main.go


FROM alpine:3.20

WORKDIR /app

COPY --from=0 /src/app /app/app

CMD ["/app/app"]
```

Here:

```dockerfile
COPY --from=0 /src/app /app/app
```

Means copy files from the 0th build stage.

But this writing style has poor readability.

More recommended:

```dockerfile
FROM golang:1.23 AS builder
```

Then:

```dockerfile
COPY --from=builder /src/app /app/app
```

---

## Scenario 6: Core role of AS

```text
AS Role of:
Name for build phase

COPY --from=Phase name:
Copy files from specified stage
```

Common combinations:

```dockerfile
FROM golang:1.23 AS builder
```

```dockerfile
COPY --from=builder /src/app /app/app
```

### One-sentence Understanding AS

```text
AS builder
→ Name the current construction phase builder

COPY --from=builder
→ From builder Get the files.
```

Core value of multi-stage building is:

```text
Retain compilation tools and source code for build phase
Only final products are retained during the operation phase

→ The mirror is smaller.
→ The attack is smaller.
→ The production environment is cleaner.
```

---

## Scenario 7: RUN to execute build commands

### Example

```dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y curl
```

### Meaning

`RUN` is used to execute commands during image building.

Common uses:

- Install software packages
- Create directories
- Modify configurations
- Download dependencies
- Clean cache

---

## Scenario 8: Merge multiple RUN commands

Not recommended:

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
```

More recommended:

```dockerfile
RUN apt-get update \
    && apt-get install -y curl vim \
    && rm -rf /var/lib/apt/lists/*
```

### Explanation

Benefits of merging `RUN`:

- Reduce image layers
- Reduce intermediate caching
- Reduce image size
- Avoid apt cache residue

---

## Scenario 9: COPY to copy files

### Example

```dockerfile
COPY ./html /usr/share/nginx/html
```

### Meaning

`COPY` is used to copy files from the build context to the image.

Common uses:

- Copy application code
- Copy configuration files
- Copy static resources
- Copy startup scripts

### Example

```dockerfile
COPY nginx.conf /etc/nginx/nginx.conf
```

```dockerfile
COPY app.jar /app/app.jar
```

---

## Scenario 10: ADD to copy files or auto-unpack

### Example

```dockerfile
ADD app.tar.gz /app/
```

### Meaning

`ADD` can also copy files, but has more capabilities than `COPY`:

- Can auto-unpack local tar packages
- Can add files from URLs

### Production Recommendation

Generally prefer using:

```dockerfile
COPY
```

Only consider using:

```dockerfile
ADD
```

when absolutely needed to auto-unpack tar packages.

Reasons:

```text
COPY More semantic.
ADD More behavior, more misunderstanding.
```

---

## Scenario 11: WORKDIR Set Working Directory

### Example

```dockerfile
WORKDIR /app
```

### Meaning

`WORKDIR` is used to set the working directory for subsequent commands.

Example:

```dockerfile
FROM ubuntu:22.04

WORKDIR /app

COPY app.sh .

RUN chmod +x app.sh

CMD ["./app.sh"]
```

Equivalent understanding:

```text
Follow-up COPYI don't know.RUNI don't know.CMD Waiting for the default to /app As Current Directory
```

---

## Scenario 12: ENV Set Environment Variables

### Example

```dockerfile
ENV APP_ENV=prod
ENV TZ=Asia/Shanghai
```

### Meaning

`ENV` is used to set environment variables in the image.

These variables can be read after the container runs.

Example:

```dockerfile
FROM nginx:1.27

ENV APP_ENV=prod
ENV TZ=Asia/Shanghai
```

### Note

Avoid putting sensitive information into `ENV`:

```dockerfile
ENV DB_PASSWORD=123456
```

Reasons:

- Will enter the image history
- May be seen by `docker inspect`
- Easy to leak after image distribution

Sensitive information is better passed through:

```text
docker run -e
Docker Compose env_file
Kubernetes Secret
CI/CD Secret
```

---

## Scenario 13: ARG Set Build Arguments

### Example

```dockerfile
ARG APP_VERSION=1.0.0
```

Pass arguments during build:

```bash
docker build --build-arg APP_VERSION=1.0.1 -t myapp:1.0.1 .
```

### Meaning

`ARG` is a build-time parameter.

Difference from `ENV`:

```text
ARG
→ Use to build mirrors

ENV
→ When the mirror is built, the container also exists when running.
```

### Example

```dockerfile
FROM alpine:3.20

ARG APP_VERSION=1.0.0

RUN echo "build version: ${APP_VERSION}"
```

Build:

```bash
docker build --build-arg APP_VERSION=2.0.0 -t test:v2 .
```

---

## Scenario 14: EXPOSE Declare Ports

### Example

```dockerfile
EXPOSE 80
```

### Meaning

`EXPOSE` is used to declare ports that the service inside the container listens on.

Note:

```text
EXPOSE It's just a statement. It's not the real publisher. mouth
```

To actually publish ports, use:

```bash
docker run -d -p 8080:80 nginx
```

during container runtime.

### Understanding

In Dockerfile:

```dockerfile
EXPOSE 80
```

When the container runs:

```bash
docker run -d -p 8080:80 my-nginx:v1
```

Access:

```text
HostIP:8080
```

---

## Scenario 15: CMD Specify Default Start Command

### Example

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

### Meaning

`CMD` is used to specify the default command to execute when the container starts.

Recommended to use JSON array format:

```dockerfile
CMD ["Command", "Parameters1", "Parameters2"]
```

Example:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

Not recommended shell format:

```dockerfile
CMD nginx -g "daemon off;"
```

Reasons:

- Signal handling is less clear than exec format
- PID 1 behavior may not meet expectations
- Container shutdown may be less graceful

---

## Scenario 16: ENTRYPOINT Specify Entry Command

### Example

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

### Meaning

`ENTRYPOINT` is used to specify the main entry command for the container.

It's more like:

```text
What are the procedures for this container to be fixed?
```

Example:

```dockerfile
FROM openjdk:17-jdk

WORKDIR /app

COPY app.jar /app/app.jar

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

---

## Scenario 17: Difference Between CMD and ENTRYPOINT

### CMD Example

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

Can be overridden at runtime:

```bash
docker run my-nginx:v1 echo hello
```

### ENTRYPOINT Example

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

At runtime, it's more like a fixed entry.

### Common Combination

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
CMD ["--spring.profiles.active=prod"]
```

Understanding:

```text
ENTRYPOINT
→ Fixed master command

CMD
→ Default Parameters
```

---

## Scenario 18: USER Specify Runtime User

### Example

```dockerfile
USER nginx
```

### Meaning

`USER` is used to specify which user the container runs as.

By default, many images run as root.

In production, it's recommended to run as a non-root user.

Example:

```dockerfile
FROM alpine:3.20

RUN adduser -D appuser

USER appuser

CMD ["sh"]
```

### Production Recommendation

Avoid long-term root usage for ordinary business containers.

Better to use:

```dockerfile
RUN adduser -D appuser
USER appuser
```

---

## Scenario 19: VOLUME Declare Data Volumes

### Example

```dockerfile
VOLUME ["/data"]
```

### Meaning

`VOLUME` is used to declare data directories in the container.

Note:

```text
VOLUME Just declare a mount point
```

In actual production, it's more common to explicitly mount volumes when running containers or in Compose/Kubernetes.

Example:

```bash
docker run -d -v my-volume:/data myapp:v1
```

---

## Scenario 20: LABEL Add Image Metadata

### Example

```dockerfile
LABEL maintainer="ops@example.com"
LABEL app="myapp"
LABEL version="1.0.0"
```

### Meaning

`LABEL` is used to add metadata to the image.

Common uses:

- Maintainer information
- Application name
- Version information
- Build source
- Git commit
- CI/CD pipeline number

Example:

```dockerfile
LABEL app="myapp" \
      version="1.0.0" \
      description="demo application image"
```

---

## Scenario 21: HEALTHCHECK Health Check

### Example

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

### Meaning

`HEALTHCHECK` is used to tell Docker how to check if the container is healthy.

Common statuses:

```text
starting
healthy
unhealthy
```

Check container health status:

```bash
docker ps
```

View details:

```bash
docker inspect ContainersID
```

### Note

The health check command itself requires the corresponding tool in the image.

For example, using:

```bash
curl
```

The image must install `curl`.

---

## VI. .dockerignore File

---

## Scenario 22: Why Need .dockerignore

`.dockerignore` is used to exclude files not needed to be sent to Docker build context.

It can be understood as:

```text
.gitignore Here. Git I used it.
.dockerignore Here. docker build I used it.
```

Without `.dockerignore`, many unrelated files may be sent to the build context:

- `.git`
- Log files
- Temporary files
- Local cache
- node_modules
- Test files
- Key files
- Large files

---

## Scenario 23: .dockerignore Example

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
```

### Note

Benefits of `.dockerignore`:

- Reduce build context size
- Speed up image building
- Avoid sensitive files entering the image
- Reduce cache invalidation
- Lower risk of accidental packaging

---

## VII. Dockerfile Build Image

---

## Scenario 24: Basic Build Command

Dockerfile exists in the current directory:

```bash
docker build -t myapp:v1 .
```

Meaning:

```text
-t myapp:v1
→ Specify mirror name and tag

.
→ Current directory as construction context
```

---

## Scenario 25: Specify Dockerfile File

```bash
docker build -f Dockerfile.prod -t myapp:prod .
```

### Note

Suitable for scenarios with multiple Dockerfiles:

```text
Dockerfile
Dockerfile.dev
Dockerfile.prod
```

---

## Scenario 26: Build Without Cache

```bash
docker build --no-cache -t myapp:v1 .
```

### Note

Suitable scenarios:

- Suspect cache causes build issues
- Dependencies have been updated
- Need complete rebuild
- Troubleshoot build process

Not recommended to use `--no-cache` every time, as it would reduce build efficiency.

---

## Scenario 27: Pass ARG During Build

```bash
docker build --build-arg APP_VERSION=1.0.1 -t myapp:1.0.1 .
```

Dockerfile Example:

```dockerfile
FROM alpine:3.20

ARG APP_VERSION=1.0.0

RUN echo "APP_VERSION=${APP_VERSION}"
```

---

## Scenario 28: View Image Build History

```bash
docker history myapp:v1
```

### Note

Used to view image layer history.

Helps analyze:

- Which layer is large
- Whether sensitive information exists
- Whether extra build steps exist
- Whether unreasonable layers exist

---

## Scenario 29: View Image Details

```bash
docker inspect myapp:v1
```

Common uses: /think

- View image environment variables
- View start command
- View working directory
- View exposed ports
- View labels
- View image metadata

---

## Eight, Dockerfile Examples

---

## Example 1: Nginx Static Page Image

Directory structure:

```text
my-nginx/
├── Dockerfile
└── html/
    └── index.html
```

Dockerfile:

```dockerfile
FROM nginx:1.27

COPY ./html /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Build image:

```bash
docker build -t my-nginx:v1 .
```

Run container:

```bash
docker run -d --name my-nginx -p 8080:80 my-nginx:v1
```

Access:

```text
http://HostIP:8080
```

View logs:

```bash
docker logs -f my-nginx
```

---

## Example 2: Simple Tool Image Based on Alpine

Dockerfile:

```dockerfile
FROM alpine:3.20

RUN apk add --no-cache curl bind-tools iproute2

CMD ["/bin/sh"]
```

Build:

```bash
docker build -t ops-tools:v1 .
```

Run:

```bash
docker run --rm -it ops-tools:v1
```

Common uses:

- Temporary curl testing
- DNS testing
- Network route inspection
- Container environment troubleshooting

---

## Example 3: Java JAR Application Image

Directory structure:

```text
java-app/
├── Dockerfile
└── app.jar
```

Dockerfile:

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY app.jar /app/app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Build:

```bash
docker build -t java-app:v1 .
```

Run:

```bash
docker run -d --name java-app -p 8080:8080 java-app:v1
```

View logs:

```bash
docker logs -f java-app
```

---

## Example 4: Image with Non-root User

Dockerfile:

```dockerfile
FROM alpine:3.20

RUN adduser -D appuser

WORKDIR /app

COPY app.sh /app/app.sh

RUN chmod +x /app/app.sh \
    && chown -R appuser:appuser /app

USER appuser

CMD ["/app/app.sh"]
```

Note:

```text
Create appuser
→ Copy Apply File
→ Modify Permissions
→ Switch to appuser Run
```

This is safer than running as root by default.

---

## Example 5: Multi-stage Build Example

Multi-stage builds are commonly used for:

- Compilation stage requires many dependencies
- Runtime stage only needs final product
- Reduce final image size
- Avoid compiler tools in production image

Example:

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /src

COPY . .

RUN go build -o app main.go


FROM alpine:3.20

WORKDIR /app

COPY --from=builder /src/app /app/app

CMD ["/app/app"]
```

Understanding:

```text
Phase I builder
→ Compiled

Phase II alpine
→ Keep only the files required for running
```

The final image will not include Go compiler environment.

Key points:

```dockerfile
FROM golang:1.23 AS builder
```

Indicates the first stage is named `builder`.

```dockerfile
COPY --from=builder /src/app /app/app
```

Indicates copying compiled artifacts from `builder` stage to final runtime stage.

---

## Example 6: Node.js Multi-stage Build Example

Directory structure:

```text
node-app/
├── Dockerfile
├── package.json
├── package-lock.json
└── src/
```

Dockerfile:

```dockerfile
FROM node:20 AS deps

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci


FROM node:20 AS builder

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules

COPY . .

RUN npm run build


FROM node:20-slim AS runtime

WORKDIR /app

ENV NODE_ENV=production

COPY --from=builder /app/dist ./dist

COPY --from=builder /app/package.json ./package.json

CMD ["node", "dist/main.js"]
```

Stage explanation:

```text
deps
→ Installation Dependence

builder
→ Copy Source and Build

runtime
→ Only keep the files needed to run
```

Advantages:

- Reusable cache for dependency installation stage
- Separation of build and runtime stages
- Cleaner final image

---

## Example 7: Using ARG and LABEL to Mark Image Version

Dockerfile:

```dockerfile
FROM alpine:3.20

ARG APP_VERSION=1.0.0
ARG GIT_COMMIT=unknown

LABEL app="demo-app" \
      version="${APP_VERSION}" \
      git_commit="${GIT_COMMIT}"

CMD ["/bin/sh"]
```

Build:

```bash
docker build \
  --build-arg APP_VERSION=1.0.1 \
  --build-arg GIT_COMMIT=abc1234 \
  -t demo-app:1.0.1 .
```

View image details:

```bash
docker inspect demo-app:1.0.1
```

Common uses:

- Mark application version
- Mark Git commit
- Mark build source
- Facilitate CI/CD tracking of images

---

## Nine, Image Layering and Caching

---

## 1. Each Dockerfile instruction may form a layer

Example:

```dockerfile
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y curl
COPY app.sh /app/app.sh
```

Can be understood as:

```text
FROM First floor
RUN First floor
RUN First floor
COPY First floor
```

More image layers don't necessarily mean worse, but unreasonable layers will increase image size and build complexity.

---

## 2. Cache Hit and Miss

Docker will try to reuse cache during build.

If earlier layers change, later layers will typically rebuild.

Example:

```dockerfile
COPY . /app
RUN npm install
```

As long as any file in current directory changes, `COPY . /app` may cause `RUN npm install` to re-execute.

Better practice is usually to copy dependency declaration files first:

```dockerfile
COPY package.json package-lock.json /app/
RUN npm install
COPY . /app
```

This way, when dependencies don't change, cache for `npm install` can be reused.

---

## 3. Cache Optimization Principles

```text
Put the less changed in front.
The change is behind you.
Reliance on installation as separate layers as possible
Copy the source code as far as you can.
```

---

## 4. Multi-stage Build and Caching

Multi-stage builds also utilize caching.

Example:

```dockerfile
FROM node:20 AS deps

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci


FROM node:20 AS builder

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules

COPY . .

RUN npm run build
```

If `package.json` and `package-lock.json` don't change, the `npm ci` stage can reuse cache.

If only business source code changes, typically only the later build stages will re-execute.

---

## Ten, Dockerfile Production Best Practices

---

## 1. Use fixed version base images

Not recommended:

```dockerfile
FROM nginx:latest
```

Recommended:

```dockerfile
FROM nginx:1.27
```

Reasons:

- More stable build results
- Easier to reproduce issues
- Clearer rollback
- More controllable CI/CD

---

## 2. Prefer smaller base images

Common choices:

```text
alpine
slim
distroless
```

Example:

```dockerfile
FROM python:3.12-slim
```

Note:

```text
Small mirrors are not absolutely the best, depending on operational dependency and accessibility.
```

Using Alpine may encounter glibc/musl compatibility issues in some scenarios, need to judge based on actual application.

---

## 3. Don't write sensitive information into Dockerfile

Wrong example:

```dockerfile
ENV DB_PASSWORD=123456
```

Wrong example:

```dockerfile
RUN echo "token=xxxxx" > /app/token.txt
```

Reasons:

- May enter image layers
- May enter image history
- May be pushed to Harbor
- May be seen after someone pulls

Better practice:

```text
Run-time injection
CI/CD Secret
Kubernetes Secret
Docker Compose env_file
```

---

## 4. Prefer non-root user to run

Example:

```dockerfile
RUN adduser -D appuser

USER appuser
```

Benefits:

- Lower container escape risk
- Lower risk of accidental operations
- Comply with least privilege principle
- More suitable for production security baseline

---

## 5. Use .dockerignore

`.dockerignore` example:

```text
.git
node_modules
*.log
.env
*.pem
id_rsa
```

Effects:

- Avoid irrelevant files entering build context
- Avoid sensitive files entering image
- Improve build speed
- Reduce image size

---

## 6. Merge RUN commands reasonably

Not recommended:

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

Recommended:

```dockerfile
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

---

## 7. Use exec format CMD / ENTRYPOINT

Recommended:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

Not recommended:

```dockerfile
CMD nginx -g "daemon off;"
```

Reasons:

- Clearer signal handling with exec format
- More controllable container shutdown
- More aligned with container main process model

---

## 8. Run only one main process per container

Containers are not virtual machines.

Not recommended to run many services in one container:

```text
nginx + mysql + redis + cron + sshd
```

Better practice:

```text
One container, one main service.
Multiple services Compose / Kubernetes Organization
```

---

## 9. Separate build artifacts and runtime environment

For applications requiring compilation, recommend using multi-stage build:

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /src

COPY . .

RUN go build -o app main.go


FROM alpine:3.20

WORKDIR /app

COPY --from=builder /src/app /app/app

CMD ["/app/app"]
```

Benefits:

- Smaller final image
- No compiler tools included
- Reduce attack surface
- More suitable for production release

---

## 10. Stage names should be clear

Not recommended:

```dockerfile
FROM node:20 AS a
```

```dockerfile
FROM node:20 AS b
```

Better practice:

```dockerfile
FROM node:20 AS deps
```

```dockerfile
FROM node:20 AS builder
```

```dockerfile
FROM node:20-slim AS runtime
```

Reasons:

- Better readability
- Easier maintenance later
- Clearer troubleshooting in CI/CD
- Less confusion when copying between stages in multi-stage builds

---

## Eleven, Common Issues and Troubleshooting

---

## Problem 1: docker build can't find file

Symptoms:

```text
COPY failed: file not found
```

Common causes:

- File not in build context
- Wrong path
- `.dockerignore` excludes the file
- Wrong directory for executing `docker build`

Troubleshoot:

```bash
ls -lh
``` /think

View `.dockerignore`:

```bash
cat .dockerignore
```

Confirm build command:

```bash
docker build -t myapp:v1 .
```

---

## Problem 2: Slow Image Build

Common Causes:

- Build context is too large
- No `.dockerignore`
- Always using `--no-cache`
- Dependency installation steps don't leverage caching
- Base image is too large
- Network download dependencies is slow

Troubleshoot:

```bash
docker build -t myapp:v1 .
```

Check build context size and time per step.

Optimize:

```text
Increase .dockerignore
Adjustment COPY Order
Reduce invalid files
Fixed dependent version
Use Build Cache
```

---

## Problem 3: Large Image Size

Check image:

```bash
docker images
```

Check image history:

```bash
docker history myapp:v1
```

Common Causes:

- Base image is too large
- Unnecessary software is installed
- Package manager cache isn't cleaned
- Source code, test files, and log files are copied into the image
- Multi-stage build isn't used

Optimization Direction:

```text
Select smaller base mirror
Use .dockerignore
Clear Cache
Reduction of unnecessary dependency
Use multistage construction
```

---

## Problem 4: Container Exits Immediately After Start

Check container:

```bash
docker ps -a
```

Check logs:

```bash
docker logs ContainersID
```

Check image startup command:

```bash
docker inspect Mirror Name
```

Common Causes:

- `CMD` is written incorrectly
- `ENTRYPOINT` is written incorrectly
- Main process exits after completion
- Application startup fails
- Configuration file error
- Insufficient permissions
- File not found

For example, if the container's main command is:

```dockerfile
CMD ["echo", "hello"]
```

The container will exit after execution, which is normal.

Long-running services need a foreground main process.

---

## Problem 5: Nginx Container Exits Immediately After Start

Incorrect Writing:

```dockerfile
CMD ["nginx"]
```

Recommended:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

Cause:

```text
The container requires a front-stage main process
Nginx Default may be by daemon Organisation
The main process exits and the container exits.
```

---

## Problem 6: apt Installation Fails During Build

Example:

```dockerfile
RUN apt-get update && apt-get install -y curl
```

Common Causes:

- Network issues
- DNS resolution failure
- apt source is unavailable
- Base image is outdated
- Package name is written incorrectly

Troubleshoot can temporarily enter the base image:

```bash
docker run --rm -it ubuntu:22.04 /bin/bash
```

Manual Testing:

```bash
apt-get update
```

```bash
apt-get install -y curl
```

---

## Problem 7: Insufficient File Permissions

Common Phenomenon:

```text
Permission denied
```

Possible Causes:

- `USER` switched to non-root
- File owner is incorrect
- File lacks execute permission
- Directory lacks write permission

Example Fix:

```dockerfile
RUN chmod +x /app/start.sh
```

```dockerfile
RUN chown -R appuser:appuser /app
```

---

## Problem 8: COPY --from Cannot Find Stage

Phenomenon:

```text
COPY --from=builder failed
```

Common Causes:

- Stage name is written incorrectly
- `FROM ... AS builder` is not defined
- Case mismatch
- Multi-stage build order is scrambled
- Wrong stage number is used

Incorrect Example:

```dockerfile
FROM golang:1.23 AS build

RUN go build -o app main.go

FROM alpine:3.20

COPY --from=builder /src/app /app/app
```

Problem:

```text
By definition, build
Quoted builder
The stage name is not consistent
```

Correct Example:

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /src

COPY . .

RUN go build -o app main.go


FROM alpine:3.20

WORKDIR /app

COPY --from=builder /src/app /app/app

CMD ["/app/app"]
```

---

## Problem 9: COPY --from Path is Written Incorrectly

Incorrect Example:

```dockerfile
COPY --from=builder /app/app /app/app
```

But actual build output is in:

```text
/src/app
```

Correct Writing:

```dockerfile
COPY --from=builder /src/app /app/app
```

Troubleshooting Approach:

```text
Confirm. builder Phase WORKDIR
→ Confirm. RUN Build Output Path
→ Confirm. COPY --from Source Path
→ Confirm final stage target path
```

---

## Section Twelve: Common Commands Summary

---

## Build Image

Build in current directory:

```bash
docker build -t myapp:v1 .
```

Specify Dockerfile:

```bash
docker build -f Dockerfile.prod -t myapp:prod .
```

Build without cache:

```bash
docker build --no-cache -t myapp:v1 .
```

Pass parameters during build:

```bash
docker build --build-arg APP_VERSION=1.0.1 -t myapp:1.0.1 .
```

---

## View Image

View image list:

```bash
docker images
```

View image history:

```bash
docker history myapp:v1
```

View image details:

```bash
docker inspect myapp:v1
```

---

## Run Test

Run container:

```bash
docker run -d --name myapp -p 8080:8080 myapp:v1
```

Temporarily enter container:

```bash
docker run --rm -it myapp:v1 /bin/sh
```

View container:

```bash
docker ps -a
```

View logs:

```bash
docker logs -f myapp
```

Enter running container:

```bash
docker exec -it myapp /bin/sh
```

---

## Tag and Push

Tag:

```bash
docker tag myapp:v1 10.0.0.10:8090/project/myapp:v1
```

Login to Harbor:

```bash
docker login 10.0.0.10:8090
```

Push image:

```bash
docker push 10.0.0.10:8090/project/myapp:v1
```

Pull image:

```bash
docker pull 10.0.0.10:8090/project/myapp:v1
```

---

## Troubleshooting Assistance

View current directory:

```bash
pwd
```

View file:

```bash
ls -lh
```

View `.dockerignore`:

```bash
cat .dockerignore
```

View Dockerfile:

```bash
cat Dockerfile
```

View build context:

```bash
ls -lah
```

---

## Section Thirteen: One-Sentence Summary

The core value of Dockerfile is:

Transform manual image building steps

→ Into a repeatable text file

→ Automatically generate image via docker build

→ Then push to Harbor or hand over to Kubernetes

Core workflow:

```text
Prepared Dockerfile
→ Prepared .dockerignore
→ docker build Build mirrors
→ docker run Local Authentication
→ docker tag Play repository tags
→ docker push Send Harbor
→ Kubernetes / Compose Use mirror
```

Understanding common Dockerfile instructions:

```text
FROM
→ Select Basic Mirror

FROM ... AS ...
→ Naming a phase in a multi-stage construction

RUN
→ Command execution for build phase

COPY
→ Copy file to mirror

COPY --from=Phase Name
→ Copy files from the specified build phase

ADD
→ Copy files, extra support for automatic decompression

WORKDIR
→ Set Task Directory

ENV
→ Set run-time environment variable

ARG
→ Set build parameters

EXPOSE
→ Declaration of container port

CMD
→ Default Start Command

ENTRYPOINT
→ Fixed Entry Command

USER
→ Specify running user

VOLUME
→ Declaration Data Catalogue

LABEL
→ Add mirror metadata

HEALTHCHECK
→ Definition of health screening
```

Understanding multi-stage build:

```text
FROM golang:1.23 AS builder
→ Create builder Build Phase

COPY --from=builder /src/app /app/app
→ From builder Phase reproduction product

The final mirror only keeps the files needed to run
→ No compilation tool maintained
→ Do not keep source
→ The mirror is smaller.
→ The attack is smaller.
```

Production Recommendations:

```text
Base mirror fixed version, not permanently used latest
Don't take the password.tokenKey writing Dockerfile
Use non as much as possible root User Run
Use .dockerignore Exclude irrelevant files
Rational use of build caches
Reduction of unnecessary dependency
Clean Cache After Build
Use Priority exec Format CMD / ENTRYPOINT
Compiler application prioritizes multi-stage construction
A clear proposal for multi-stage construction AS Phase Name
Local validation is required after mirror construction is completed
```