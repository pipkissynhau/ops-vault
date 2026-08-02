# 10-Dockerfile Basics: Image Building and Best Practices

# Docker # Dockerfile # Image Building # Containerization # Image Optimization # CI-CD # Operations # Security

---

## Recommended Path

03-Container Technology/10-Dockerfile Basics: Image Building and Best Practices.md

---

## I. Document Description

This document outlines the basic syntax, common commands, image building process, and best practices for Dockerfiles, focusing on:

- What a Dockerfile is
- The difference between Dockerfiles and `docker commit`
- The basic structure of a Dockerfile
- Common Dockerfile commands
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
- Best practices for production environments

The goal is:

To understand Dockerfiles

→ Be able to write basic Dockerfiles

→ Build images

→ Comprehend image layering and caching

→ Understand multi-stage building with `FROM ... AS ...`

→ Reduce image size

→ Avoid including sensitive information in images

→ Lay a foundation for subsequent CI/CD image builds

---

## II. What is a Dockerfile

A Dockerfile is a text file that describes how to build an image.

It can be thought of as:

```text
Dockerfile = An instruction manual for building images
```

Manual approach:

```text
Enter the container
→ Install software
→ Modify configurations
→ Use docker commit to save the image
```

Dockerfile approach:

```text
Write installation steps, configuration copying, and startup commands in a file
→ Use docker build to automatically build the image
```

For production environments, Dockerfiles are more recommended over long-term reliance on `docker commit`.

---

## III. The Difference between Dockerfiles and docker commit

---

## 1. docker commit

Example:

```bash
docker commit containerID my-nginx:v1
```

Characteristics:

- Suitable for temporary saving of work in progress
- Opacity during the process
- Inconvenient for review
- Difficult to manage versions
- Not suitable for long-term production use

---

## 2. Dockerfile

Example:

```dockerfile
FROM nginx:latest

COPY ./html /usr/share/nginx/html

EXPOSE 80
```

Building:

```bash
docker build -t my-nginx:v1 .
```

Characteristics:

- Clear building process
- Support for version management
- Facilitates code review
- Compatible with CI/CD pipelines
- More suitable for teamwork and production use

---

## 3. In one sentence

```text
docker commit
→ Temporarily saves container state

Dockerfile
→ Standardizes image building processes
```

Production recommendation:

```text
Use docker commit for temporary troubleshooting
Opt for Dockerfiles for official image builds
```

---

## IV. The Basic Structure of a Dockerfile

A simple Dockerfile example:

```dockerfile
FROM nginx:latest

COPY ./html /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

The basic structure can be understood as:

```text
Select a base image
→ Install dependencies
→ Copy files
→ Set the working directory
→ Configure environment variables
→ Expose ports
→ Specify the startup command
```

---

## V. Common Dockerfile Commands

---

## Scenario 1: Using FROM to specify a base image

### Example

```dockerfile
FROM nginx:latest
```

### Meaning

`FROM` is used to designate the base image for the build.

Dockerfiles always start with `FROM`.

Common base images include:

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

It is not recommended to use `latest` indefinitely:

```dockerfile
FROM nginx:latest
```

Instead, it is better to specify a fixed version:

```dockerfile
FROM nginx:1.27
```

Reasons:

- `latest` versions are unstable
- The resulting image may change over time
- It makes rolling back difficult
- It can complicate issue reproduction

---

## Scenario 2: Using FROM AS for stage naming

### Example

```dockerfile
FROM golang:1.23 AS builder
```

###### Copy Files Using `COPY`

`COPY` is used to copy files from the build context into the image.

**Common Uses:**
- Copying application code
- Copying configuration files
- Copying static resources
- Copying startup scripts

**Examples:**
```dockerfile
COPY nginx.conf /etc/nginx/nginx.conf
COPY app.jar /app/app.jar
```

---

## Scenario 10: Using `ADD` to Copy Files or Auto-Unzip Them

**Example:**
```dockerfile
ADD app.tar.gz /app/
```

**Meaning:**
`ADD` can also be used to copy files, but it offers additional features:
- It can automatically unzip local tar packages.
- It can add files from a URL.

**Production Recommendation:**
Generally, `COPY` should be preferred. Use `ADD` only when you need to automatically unzip tar packages.

**Reason:**
- `COPY` has clearer semantics.
- `ADD` performs more actions and may lead to misunderstandings.

---

## Scenario 11: Using `WORKDIR` to Set the Working Directory

**Example:**
```dockerfile
WORKDIR /app
```

**Meaning:**
`WORKDIR` is used to set the working directory for subsequent commands.

**Example Usage:**
```dockerfile
FROM ubuntu:22.04

WORKDIR /app

COPY app.sh .

RUN chmod +x app.sh

CMD ["./app.sh"]
```

**Equivalence:**
Subsequent `COPY`, `RUN`, `CMD`, and other operations will use `/app` as the current working directory by default.

---

## Scenario 12: Using `ENV` to Set Environment Variables

**Example:**
```dockerfile
ENV APP_ENV=prod
ENV TZ=Asia/Shanghai
```

**Meaning:**
`ENV` is used to set environment variables inside the image. These variables can also be read by containers after they start.

**Example Usage:**
```dockerfile
FROM nginx:1.27

ENV APP_ENV=prod
ENV TZ=Asia/Shanghai
```

**Note:**
It's not recommended to include sensitive information in `ENV` files:
- Such information will be included in the image’s history.
- It may be visible when using `docker inspect`.
- Sensitive data can easily be leaked if the image is distributed.

Sensitive information is better passed through other methods, such as:

- `-e` option when running a container
- Docker Compose environment files
- Kubernetes Secrets
- CI/CD Secrets

---

## Scenario 13: Using `ARG` to Set Build Parameters

**Example:**
```dockerfile
ARG APP_VERSION=1.0.0
```

**Usage During Building:**
```bash
docker build --build-arg APP_VERSION=1.0.1 -t myapp:1.0.1 .
```

**Meaning:**
`ARG` is a parameter used during the image building process.

**Difference from `ENV`:**
- `ARG` is only used when building the image.
- `ENV` exists both during and after the image is built.

**Example:**
```dockerfile
FROM alpine:3.20

ARG APP_VERSION=1.0.0

RUN echo "build version: ${APP_VERSION}"
```

**Building Command:**
```bash
docker build --build-arg APP_VERSION=2.0.0 -t test:v2 .
```

---

## Scenario 14: Using `EXPOSE` to Declare Ports

**Example:**
```dockerfile
EXPOSE 80
```

**Meaning:**
`EXPOSE` is used to specify the ports that services inside the container will listen on.

**Note:**
`EXPOSE` only declares the ports; they are not actually published unless explicitly specified when running the container:

```bash
docker run -d -p 8080:80 nginx
```

**Understanding:**
In the Dockerfile:
```dockerfile
EXPOSE 80
```

When running the container:
```bash
docker run -d -p 8080:80 my-nginx:v1
```

Access from the host:
```text
Host IP:8080
```

---

## Scenario 15: Using `CMD` to Specify the Default Start Command

**Example:**
```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

**Meaning:**
`CMD` is used to specify the default command that will be executed when the container starts.

It's recommended to use a JSON array format:

```dockerfile
CMD ["command", "parameter1", "parameter2"]
```

For example:
```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

The shell format is less recommended because it may lead to unclear signal handlingnode_modules
dist
target
*.log
*.tmp
.env
.env.*
id_rsa
*.pem```bash
COPY package.json package-lock.json /app/
RUN npm install
COPY . /app
```- The stage name is incorrect.
- `FROM ... AS builder` has not been defined.
- There are inconsistencies in casing.
- The order of multi-stage builds is incorrect.
- The wrong stage number is being used.

Incorrect example:

```dockerfile
FROM golang:1.23 AS build

RUN go build -o app main.go

FROM alpine:3.20

COPY --from=builder /src/app /app/app
```

Issue:

```text
The defined stage name is "build", but the reference uses "builder".
The stage names are inconsistent.
```

Correct example:

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

## Issue 9: Incorrect path for COPY --from

Incorrect example:

```dockerfile
COPY --from=builder /app/app /app/app
```

However, the actual build output is in:

```text
/src/app
```

Correct way to write it:

```dockerfile
COPY --from=builder /src/app /app/app
```

Troubleshooting steps:

```text
Confirm the WORKDIR of the builder stage.
→ Confirm the output path for the RUN command during construction.
→ Confirm the source path specified in COPY --from.
→ Confirm the target path of the final stage.
```

---

## XII. Summary of Common Commands

---

## Building Images

Build from the current directory:

```bash
docker build -t myapp:v1 .
```

Specify a Dockerfile:

```bash
docker build -f Dockerfile.prod -t myapp:prod .
```

Build without using cache:

```bash
docker build --no-cache -t myapp:v1 .
```

Pass parameters during building:

```bash
docker build --build-arg APP_VERSION=1.0.1 -t myapp:1.0.1 .
```

---

## Viewing Images

View the list of images:

```bash
docker images
```

View the image history:

```bash
docker history myapp:v1
```

View image details:

```bash
docker inspect myapp:v1
```

---

## Running Tests

Run a container:

```bash
docker run -d --name myapp -p 8080:8080 myapp:v1
```

Temporarily enter a container:

```bash
docker run --rm -it myapp:v1 /bin/sh
```

View containers:

```bash
docker ps -a
```

View logs:

```bash
docker logs -f myapp
```

Enter a running container:

```bash
docker exec -it myapp /bin/sh
```

---

## Tagging and Pushing Images

Tag an image:

```bash
docker tag myapp:v1 10.0.0.10:8090/project/myapp:v1
```

Log in to Harbor:

```bash
docker login 10.0.0.10:8090
```

Push an image:

```bash
docker push 10.0.0.10:8090/project/myapp:v1
```

Pull an image:

```bash
docker pull 10.0.0.10:8090/project/myapp:v1
```

---

## Troubleshooting Assistance

View the current directory:

```bash
pwd
```

View files:

```bash
ls -lh
```

View `.dockerignore`:

```bash
cat .dockerignore
```

View the Dockerfile:

```bash
cat Dockerfile
```

View the build context:

```bash
ls -lah
```

---

## XIII. In One Sentence

The core value of a Dockerfile is to:

- Transform the steps involved in manually building an image into a repeatable text file.
- Use `docker build` to automatically generate the image.
- Then push it to Harbor or make it available for use by Kubernetes.

Key process flow:

```text
Write a Dockerfile.
→ Create a `.dockerignore` file.
→ Use `docker build` to construct the image.
→ Run it locally for verification.
→ Add repository tags using `docker tag`.
→ Push the image to Harbor.
→ Use it in Kubernetes or Compose.
```

Understanding common Dockerfile commands:

```text
FROM
→ Selects the base image.

FROM ... AS ...
→ Gives a name to a specific stage in a multi-stage build.

RUN
→ Executes commands during the build phase.

COPY
→ Copies files into the image.

COPY --from=stage_name
→ Copies files from a specified build stage.

ADD
→ Copies files, with additional support for automatic decompression.

WORK