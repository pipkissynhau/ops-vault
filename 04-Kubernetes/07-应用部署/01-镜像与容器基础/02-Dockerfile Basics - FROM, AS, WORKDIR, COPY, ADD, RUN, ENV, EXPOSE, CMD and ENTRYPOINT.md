# 02-Dockerfile Basics: FROM, AS, WORKDIR, COPY, ADD, RUN, ENV, EXPOSE, CMD, and ENTRYPOINT

## Document Notes
- Document Focus: Image building fundamentals
- Applicable Stage: Business containerization learning third article / Dockerfile introduction
- Recommended Path: `04-Kubernetes/07-Apply deployment/01-Mirror and container base/02-Dockerfile Basis:FROMI don't know.ASI don't know.WORKDIRI don't know.COPYI don't know.ADDI don't know.RUNI don't know.ENVI don't know.EXPOSEI don't know.CMD and ENTRYPOINT`

## Tags
#Kubernetes #Docker #Dockerfile #MirrorBuild #Containers #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why You Must Master Dockerfile

In the process of business containerization, Dockerfile is one of the most core entry files.

Because business programs won't directly become images, but need to tell the build tool through Dockerfile:

- Based on which base image to build
- Which files to include in the image
- Which dependencies to install
- Which working directory to use
- Which environment variables are needed
- What commands to execute when the container starts

Without knowing Dockerfile, it's easy to remain at the level of "knowing how to pull images and run containers", but unable to answer these more critical questions:

- How is a business image created
- What exactly is included in the image
- Why the container reports "command not found" after startup
- Why application files aren't in the expected directory
- Why changes to code don't take effect in the container
- Why image building is slow, large in size, and has messy layers
- Why the final image for the same business shouldn't include compiler toolchains

Thus, Dockerfile is the key foundation for moving from "using containers" to "building business images".

---

## II. What is Dockerfile

Dockerfile is a text file that defines:

**How to build an image step by step.**

It essentially consists of a set of build instructions executed in sequence.  
After reading the Dockerfile, the build tool will construct the image layer by layer according to these instructions.

You can think of it as:

**A manual for building images.**

For example, a simple business image construction process typically includes:

1. Selecting a base image
2. Specifying the working directory
3. Copying business files
4. Installing dependencies
5. Setting environment variables
6. Exposing application ports
7. Specifying the container startup command

These steps are usually written in the Dockerfile.

---

## III. Basic Dockerfile Construction Logic

The core idea of Dockerfile is not "putting everything in", but:

**Adding the minimal content needed for business operation on an appropriate base image in sequence.**

A common construction flow can be abstracted as:

### 1. Select Base Image
Examples:
- Nginx image
- Python image
- OpenJDK image
- Alpine / Debian / Ubuntu base images

### 2. Prepare Business Files
Examples:
- Static pages
- Configuration files
- Python code
- Java JAR packages
- Startup scripts
- Compressed resource files

### 3. Define Dependency Installation and Directory Structure
Examples:
- Install pip dependencies
- Install system tools
- Create working directories
- Copy files to specified paths
- Decompress business resources

### 4. Define Startup Method
Examples:
- Start Nginx
- Run Python program
- Execute Java JAR
- Execute shell startup script

### 5. Minimize the Final Image
In production, not all build tools are retained in the final image, but only the content needed for business operation.  
This is also the problem to be solved by multi-stage builds later.

---

## IV. FROM: Specify Base Image

`FROM` is one of the most fundamental and core instructions in Dockerfile.

Its purpose is:

**Specify the starting point for current image construction.**

Example:

    FROM nginx:1.27-alpine

Indicates that the current image is built based on `nginx:1.27-alpine`.

Another example:

    FROM python:3.11-slim

Indicates that the current image is built based on the Python 3.11 slim version.

### Operations Understanding Focus

#### 1. Without `FROM`, image construction usually cannot start
It's equivalent to "from which runtime environment to start building".

#### 2. The base image determines many things
Examples:
- Whether certain commands are pre-installed
- Operating system type
- Image size
- Security exposure surface
- Subsequent dependency installation methods

#### 3. Avoid blindly selecting large base images
Larger images typically mean:
- Slower pulls
- Slower builds
- Larger attack surface
- Heavier runtime environment

### Common Base Image Approaches

#### For Static Pages
Commonly used:
    FROM nginx:alpine

#### For Python Applications
Commonly used:
    FROM python:3.11-slim

#### For Java Applications
Commonly used:
    FROM eclipse-temurin:17-jre
    FROM openjdk:17-jdk

#### For Go Binary Applications
Common practice is to use Golang image for building and a smaller image for runtime.

---

## V. AS: Name Build Stage

`AS` is not an independent instruction, usually used together with `FROM`.

Example:

    FROM golang:1.22 AS builder

Its purpose is:

**Give a name to the current build stage.**

This allows you to reference the files generated in the previous stage through `COPY --from=builder` in multi-stage builds.

### Why It Matters

In production environments, many business images don't include source code, compilers, build caches, and dependency tools in the final image, but instead use:
- Previous stage for compilation
- Subsequent stage only retains runtime content

Benefits include:
- Smaller final image
- Lower security risks
- Cleaner runtime environment
- Faster pulls
- More compliant with production standards

### A Simple Example

    FROM golang:1.22 AS builder
    WORKDIR /app
    COPY . .
    RUN go build -o myapp .

    FROM alpine:3.20
    WORKDIR /app
    COPY --from=builder /app/myapp /app/myapp
    CMD ["/app/myapp"]

This Dockerfile can be understood as:
- First stage named `builder`
- First stage handles compilation
- Second stage doesn't need Go compiler environment
- Second stage only copies the compiled binary file
- Final image is smaller and more suitable for runtime

### Operations Understanding Focus

#### 1. `AS` is mainly used for multi-stage builds
#### 2. It makes different build stages clearer
#### 3. Subsequent stages reference previous stage's outputs through `COPY --from=Phase Name`
#### 4. Multi-stage builds are very common in Go, frontend, Java, etc. business images
#### 5. This is an important foundation for optimizing production images

---

## VI. WORKDIR: Set Working Directory

`WORKDIR`'s purpose is:

**Specify the default directory for subsequent instruction execution.**

Example:

WORKDIR /app

This indicates that subsequent operations like `ADD`, `COPY`, `RUN`, `CMD`, etc., will default to use `/app` as the current working directory.

### Why It Matters

Without setting a working directory, many file copies, dependency installations, and startup commands can easily become confusing.

For example:

- Not knowing where the code is actually copied to
- Not finding program files at startup
- Incorrect paths during dependency installation
- Disorganized directory structure after unpacking files

### Operations Understanding Focus

#### 1. `WORKDIR` makes the image directory structure clearer
For example, uniformly placing business content in:

- `/app`
- `/opt/app`
- `/usr/share/nginx/html`

#### 2. It affects the meaning of subsequent relative paths
Many commands appear fine but actually have incorrect paths, often related to `WORKDIR` settings.

#### 3. Unified working directory helps with troubleshooting
For example, after entering the container, you can quickly determine the location of code, configuration, and startup scripts.

---

## VII. ADD: Enhanced Copy Instruction

The purpose of `ADD` is:

**To copy files to the image while providing additional capabilities.**

### Common Capabilities

#### 1. Copy local files or directories to the image
This is similar to `COPY`.

#### 2. Automatically decompress local tar archives
This is a key difference from `COPY`.

#### 3. Support for fetching content from URLs
However, this approach is generally not recommended in production due to poor controllability and repeatability.

---

## VIII. COPY: Copy Files to the Image

The purpose of `COPY` is:

**To copy files from the build context into the image.**

For example:

    COPY . /app

Indicates copying the contents of the current directory to the image's `/app` directory.

Another example:

    COPY index.html /usr/share/nginx/html/

Indicates copying the local `index.html` to the Nginx default site directory.

### What It's Mainly Used For

- Copy application code
- Copy configuration files
- Copy static resources
- Copy startup scripts
- Copy compiled outputs

---

## IX. Difference Between ADD and COPY

This is a common point encountered in practical operations and may also be asked in interviews.

### 1. `COPY`
More pure, only responsible for copying files or directories.

For example:

    COPY app.tar.gz /app/

This typically just copies the `app.tar.gz` to the `/app/` directory,
**without automatic decompression**.

### 2. `ADD`
In addition to copying files, it can also automatically decompress local tar archives.

For example:

    ADD app.tar.gz /app/

If `app.tar.gz` is a local compressed archive in the build context, it may be directly decompressed to `/app/` during construction under certain conditions.

### 3. Simplified Understanding
- `COPY`: Only copy, no automatic decompression
- `ADD`: More features, can be used for automatic decompression of local archives

### 4. Why COPY is Often Recommended
Although `ADD` has more capabilities, COPY is usually preferred in production for reasons including:

- Clearer semantics
- More predictable behavior
- Better readability
- More in line with the principle of minimal capabilities

### 5. What to Do If You Want to Use ADD for Manual Decompression
You can do this:

    COPY app.tar.gz /tmp/
    RUN tar -xzf /tmp/app.tar.gz -C /app

This approach has the advantages of being more explicit, more controllable, and easier to read and maintain.

### Operations Understanding Focus

#### 1. Prefer COPY for ordinary file copying
#### 2. Consider ADD when you clearly need automatic decompression of local archives
#### 3. Recommend COPY when you want clearer and more controllable behavior

---

## X. COPY: Operations Understanding Focus

### 1. `COPY` copies "files visible during construction"
It's not about reading local files after container runtime, but packaging files into the image during image construction.

### 2. Paths Must Be Clear
Many build issues essentially boil down to:

- Wrong local file path
- Wrong target path in the image
- Code being copied to the wrong directory

### 3. Whether to Use COPY for Configuration Depends on the Scenario
If the configuration is environment-agnostic base configuration, it can be included with the image.  
If the configuration is strongly environment-dependent, it's usually better to inject it at runtime via ConfigMap or Secret rather than hardcoding it into the image.

### 4. In multi-stage builds, COPY has another common usage
For example:

    COPY --from=builder /app/dist /usr/share/nginx/html

This indicates:

- Not copying from the local directory
- But copying files from a previous build stage

This is also a common pattern in multi-stage builds.

---

## XI. RUN: Execute Commands During Image Construction

The purpose of `RUN` is:

**To execute commands during the image construction phase.**

For example:

    RUN apt-get update && apt-get install -y curl

Indicates updating the package index and installing curl during image construction.

Another example:

    RUN pip install -r requirements.txt

Indicates installing Python dependencies during image construction.

Another example:

    RUN tar -xzf /tmp/app.tar.gz -C /app

Indicates manually decompressing the archive during the construction phase.

### Operations Understanding Focus

#### 1. `RUN` is executed during image construction, not at container startup
This is a highly common point of confusion.

- `RUN`: Executed during image construction
- `CMD` / `ENTRYPOINT`: Executed at container startup

#### 2. `RUN` is commonly used for installing dependencies
For example:

- Installing system packages
- Installing Python packages
- Unpacking files
- Modifying permissions
- Compiling programs

#### 3. Each `RUN` often creates a new image layer
So when writing Dockerfile, you need to consider the number of layers, caching, and image size.

### Common Misunderstandings

#### Misunderstanding 1: Putting business startup commands in `RUN`
For example:

    RUN python app.py

This approach is usually incorrect because it only runs once during image construction and doesn't become the container's startup command.

#### Misunderstanding 2: Too many `RUN` and fragmented layers
This leads to too many image layers, affecting construction and maintenance.

---

## XII. ENV: Set Environment Variables

The purpose of `ENV` is:

**To define environment variables in the image.**

For example:

ENV APP_ENV=prod

Sets an environment variable `APP_ENV=prod` in the image.

### Common Uses

- Define default environment variables
- Define program runtime parameters
- Define default paths
- Define language environments

### Operations Understanding Focus

#### 1. `ENV` is suitable for default values
However, in production environments, many variables that change with the environment are better injected at runtime by Kubernetes rather than hardcoded in the image.

#### 2. Do not put sensitive information directly into Dockerfile
For example, database passwords, keys, etc., should typically not be written in Dockerfile's `ENV`.

#### 3. Distinguish between "default values" and "production configuration"
Default values can be written in the image, while environment-specific values are better injected at the deployment layer.

---

## ThirteenI don't know.EXPOSE: Declare container used ports

`EXPOSE`'s purpose is:

**Declare the port the application inside the container expects to use.**

Example:

    EXPOSE 80

Indicates the application inside the container typically listens on port 80.

### Operations Understanding Focus

#### 1. `EXPOSE` is more explanatory in nature
It does not automatically expose the port.

#### 2. Actual traffic exposure requires additional mechanisms
For example:

- Docker's `-p`
- Kubernetes' Service
- Kubernetes' Ingress

#### 3. `EXPOSE` mainly helps with reading and understanding the image's purpose
It allows others to quickly know which port the service in this image is likely listening on.

#### 4. Not writing `EXPOSE` does not mean the service cannot listen on a port
It's more about readability and convention declaration.

---

## FourteenI don't know.CMD: Define container default startup command

`CMD`'s purpose is:

**Specify the command to be executed by default when the container starts.**

Example:

    CMD ["nginx", "-g", "daemon off;"]

Indicates the container will default to executing the Nginx foreground command after startup.

Another example:

    CMD ["python", "app.py"]

Indicates the container will default to running the Python program after startup.

### Operations Understanding Focus

#### 1. `CMD` is runtime behavior, not build-time behavior
It only takes effect when the container starts.

#### 2. `CMD` can be overridden
In some runtime scenarios, external startup commands can override the `CMD` defined in Dockerfile.

#### 3. Whether a container can start correctly largely depends on whether `CMD` is correct
If `CMD` is written incorrectly, common phenomena include:

- Container starts and exits immediately
- Command not found
- Application not starting properly

---

## FifteenI don't know.ENTRYPOINT: Define container main entry

`ENTRYPOINT`'s purpose is:

**Define the main entry program for the container when it starts.**

Example:

    ENTRYPOINT ["python"]

If combined with:

    CMD ["app.py"]

Then the container's startup effect is similar to:

    python app.py

### Relationship with CMD

It can be simply understood as:

- `ENTRYPOINT`: Main program
- `CMD`: Default parameters or supplementary commands

### Operations Understanding Focus

#### 1. `ENTRYPOINT` emphasizes "the container is to run this program"
Suitable for images with very clear container responsibilities.

#### 2. `CMD` is more suitable as default command or default parameters
When combined, the expression capability is stronger.

---

## SixteenI don't know.Difference between CMD and ENTRYPOINT

This is a very classic interview question.

### 1. `CMD`
More inclined to:

**Default execution command / Default parameters**

Features:

- Can be used as the container's default startup command
- Easier to be overridden externally

### 2. `ENTRYPOINT`
More inclined to:

**Fixed entry program**

Features:

- Emphasizes the main responsibility of the container
- Usually not intended to be replaced easily

### 3. Common combination method
For example:

    ENTRYPOINT ["python"]
    CMD ["app.py"]

Indicates the default execution of `python app.py`.

### 4. During learning phase, you can first remember a simplified understanding
- `CMD`: Default how to start
- `ENTRYPOINT`: Main entry is who

---

## SeventeenI don't know.A simple Dockerfile reading example

Below is a simple Python Web service example.

    FROM python:3.11-slim

    WORKDIR /app

    COPY requirements.txt /app/
    RUN pip install --no-cache-dir -r requirements.txt

    COPY . /app

    EXPOSE 5000

    CMD ["python", "app.py"]

### This Dockerfile can be understood as follows

#### 1. Based on the Python 3.11 slim image
Indicates the runtime environment already has Python.

#### 2. Set the working directory to `/app`
Subsequent operations are all around `/app`.

#### 3. First copy the dependency file, then install dependencies
Helps utilize build caching.

#### 4. Then copy all the code
This way, when business code updates, it doesn't necessarily need to reinstall all dependencies.

#### 5. Declare the container will use port 5000
Helps understand the service's purpose.

#### 6. The container executes `python app.py` after startup
This is the default startup method.

---

## EighteenI don't know.A reading example with ADD

Below is a simple example of automatically unpacking a resource package.

    FROM nginx:1.27-alpine

    WORKDIR /usr/share/nginx/html

    ADD site.tar.gz /usr/share/nginx/html/

### This Dockerfile can be understood as follows

#### 1. Based on the Nginx official image
Indicates the image already has Nginx and the default site directory.

#### 2. Set the working directory to the Nginx default page directory
Helps manage site content.

#### 3. Use `ADD` to put the local compressed package into the image and automatically unpack it
If `site.tar.gz` is a local compressed package in the build context, it might be directly expanded to the target directory during the build.

### Operations Understanding Focus
This approach is suitable for:

- Static resources are already pre-packed into tar packages
- Want to expand content directly during the build
- Want to reduce manual unpacking commands

However, if you want more explicit behavior, you can also change it to:

    COPY site.tar.gz /tmp/
    RUN tar -xzf /tmp/site.tar.gz -C /usr/share/nginx/html

---

## NineteenI don't know.A reading example with AS /think

Here's a simple example of a frontend multi-stage build:

    FROM node:20-alpine AS builder
    WORKDIR /app
    COPY package*.json /app/
    RUN npm install
    COPY . /app
    RUN npm run build

    FROM nginx:1.27-alpine
    COPY --from=builder /app/dist /usr/share/nginx/html
    EXPOSE 80
    CMD ["nginx", "-g", "daemon off;"]

### Understanding this Dockerfile

#### 1. First stage uses Node environment to build frontend artifacts
This stage requires Node, npm, source code, and build tools.

#### 2. First stage is named `builder`
This makes it easier to reference later.

#### 3. Second stage no longer needs Node environment
Static page execution only requires Nginx.

#### 4. Build artifacts are copied via `COPY --from=builder`
Only the frontend build results are included in the final image.

#### 5. Final image is smaller and more suitable for production
It doesn't include all frontend build toolchains in the runtime image.

---

## 20. Common Issues in Dockerfile Writing

### 1. Choosing overly large base images
Problem manifestations:

- Large image size
- Slow pulls
- Slow startup
- Expanded attack surface

### 2. Unclear copy paths
Problem manifestations:

- Files not in expected directories
- File not found errors on startup
- Incorrect configuration paths

### 3. Misunderstanding archive behavior
Problem manifestations:

- Assuming `COPY` automatically unpacks, but it doesn't
- Assuming `ADD` and `COPY` are identical, ignoring auto-unpacking capabilities
- Mismatched directory structure after unpacking

### 4. Writing runtime commands in `RUN`
Problem manifestations:

- Commands run during image build
- Business processes not executed when container starts

### 5. Writing `CMD` incorrectly
Problem manifestations:

- Container exits immediately
- "executable file not found" errors
- Startup command parameter errors

### 6. Hardcoding environment-specific configurations
Problem manifestations:

- Difficulty reusing images across environments
- Configuration changes require image rebuilds
- Incompatibility with Kubernetes dynamic configuration injection

### 7. Including sensitive information in Dockerfile
Problem manifestations:

- Risk of key leaks
- Image exposure risks
- Non-compliance with production security requirements

### 8. Not understanding multi-stage builds
Problem manifestations:

- Compiler and source code residues in final image
- Excessively large image size
- Mixed build and runtime environments
- Increased security and maintenance costs

---

## 21. Viewing Dockerfile from an Operations Perspective

From an operations perspective, Dockerfile isn't just a file developers write for build tools - it's also a critical reference for subsequent deployment, troubleshooting, and release processes.

### 1. Analyze base images
Can quickly determine:

- What runtime environment it uses
- Estimated image size
- Which package manager to use for dependencies

### 2. Analyze startup commands
Can quickly determine:

- What is the main application process
- Why the container exits
- Whether foreground execution is needed

### 3. Analyze working directory
Can quickly determine:

- Where the code is located
- Whether configuration file paths are reasonable
- How to design mount directories

### 4. Analyze dependency installation process
Can determine:

- Whether runtime dependencies are installed
- Why the build is slow
- Why the image is large

### 5. Analyze file entry methods
Can determine:

- Whether files enter the image via `COPY`
- Or via `ADD` auto-unpacking
- Whether multi-stage copying is used
- Whether target directories are reasonable

### 6. Analyze port declarations
Can help infer Service, Probe, Ingress configuration logic.

### 7. Analyze environment variables
Can determine:

- What is the default behavior
- Which configurations are hardcoded
- Whether Kubernetes injection replacement is needed

### 8. Analyze multi-stage build usage
Can determine:

- Whether build and runtime environments are separated
- Whether image optimization is possible
- Whether unnecessary tools remain in the final image

---

## 22. What Level of Understanding Should You Master for Dockerfile

Currently, you don't need to master complex advanced optimizations right away.  
Achieving the following level is sufficient:

### 1. Understand what a simple Dockerfile does
### 2. Distinguish between build-time instructions and runtime instructions
### 3. Understand the purpose of `FROM`I don't know.`AS`I don't know.`WORKDIR`I don't know.`ADD`I don't know.`COPY`I don't know.`RUN`I don't know.`ENV`I don't know.`EXPOSE`I don't know.`CMD`I don't know.`ENTRYPOINT`
### 4. Understand the difference between `ADD` and `COPY`
### 5. Understand the difference between `AS` and the basic concept of multi-stage builds
### 6. Be able to trace the build process of a simple business image
### 7. Be able to preliminarily locate common Dockerfile issues

---

## 23. Common Follow-up Questions in Interviews

Common questions in this area include:

- What is Dockerfile used for?
- What is the purpose of `FROM`?
- What does `AS` do?
- What is multi-stage build?
- Why are multi-stage builds commonly used in production images?
- What is the difference between `ADD` and `COPY`?
- What is the difference between `RUN` and `CMD`?
- What is the difference between `CMD` and `ENTRYPOINT`?
- Why shouldn't the container startup command be written in `RUN`?
- Why is `COPY` often preferred?
- In what scenarios would you consider using `ADD`?
- Why is it not recommended to include sensitive information in Dockerfile?
- Why should production images be as small as possible?
- Why do many business images need foreground execution commands?

---

## 24. Stage Summary

Dockerfile is the foundation for image building.

The most important part of this section isn't memorizing all details immediately, but first establishing several core understandings:

- `FROM` Determines the base environment to start the build from
- `AS` Used to name the build stage, which is the foundation of multi-stage builds
- `WORKDIR` Determines the default working directory
- `ADD` and `COPY` can both copy files into the image, but their behaviors are not entirely identical
- `COPY` is purer and only handles copying
- `ADD` has more functionality and can be used for automatic decompression of local archives
- `RUN` Determines which installation and processing actions the build stage executes
- `ENV` Used to define default environment variables
- `EXPOSE` Used to declare ports the container expects to use
- `CMD` Determines how the container starts by default
- `ENTRYPOINT` Determines who the main entry program of the container is

Only by clearly understanding the roles of these base instructions can you avoid confusing build layers with runtime layers when learning about image optimization, container execution, Kubernetes deployment, and troubleshooting later.

---

## 25. Keyword Mnemonics

- Dockerfile: Image build instruction manual
- FROM: Specify the base image
- AS: Name the build stage
- WORKDIR: Specify the default working directory
- ADD: Enhanced copy, can be used for automatic decompression of local archives
- COPY: Copy files into the image
- RUN: Commands executed during the build stage
- ENV: Set environment variables
- EXPOSE: Declare ports the container uses
- CMD: Define default startup command
- ENTRYPOINT: Define the main entry program
- Multi-stage build: Separate the build environment from the runtime environment

---

## 26. Operational Extension Understanding

In production environments, many "container startup failures" originate not from Kubernetes itself, but are already planted during the image build phase:

- Dockerfile path written incorrectly
- Dependencies not fully installed
- Startup command errors
- Program files not copied into the image
- Archives not decompressed as expected
- Environment variables set improperly
- Large image size leading to low build and pull efficiency
- Build toolchain incorrectly included in the final runtime image

Therefore, understanding Dockerfile isn't just about writing images, but also about quickly determining where the problem occurs when deployment fails:

- Build layer
- Image layer
- Container startup layer
- Kubernetes orchestration layer

The introduction of `AS` and multi-stage builds marks your transition from "being able to write simple Dockerfiles" to "understanding production image optimization."

---

## References
- Dockerfile reference: https://docs.docker.com/reference/dockerfile/
- Docker Docs: https://docs.docker.com/
- Kubernetes Images: https://kubernetes.io/docs/concepts/containers/images/
- OCI Image Spec: https://github.com/opencontainers/image-spec

---

## Next Day Suggestions
Next article suggestion to organize:

[[03-Image Naming, Tag, Build, Push, and Pull Basics]]