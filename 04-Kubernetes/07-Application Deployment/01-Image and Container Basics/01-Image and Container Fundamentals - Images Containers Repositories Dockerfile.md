# 01-Overview of Images and Containers: Images, Containers, Repositories, and Dockerfile

## Document Notes
- Document Purpose: Foundation of containerization for business applications
- Applicable Stage: Kubernetes Application Deployment Mainline Part 2 / Overview of Images and Containers
- Recommended Path: `04-Kubernetes/07-Apply deployment/01-Mirror and container base/01-Mirrors and packaging basic overview: mirrors, containers, warehouses and Dockerfile`

## Tags
#Kubernetes #Containers #Mirror #Docker #Dockerfile #MirrorRepository #OperationalContainerization #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Why Learn Image and Container Basics First

Before studying Kubernetes application deployment, you must first understand:

**What exactly Kubernetes manages.**

From an operational perspective, Kubernetes does not directly run "source code", "compressed packages", or "configuration directories". The core runtime units it schedules and manages ultimately remain:

- Images
- Containers
- Container instances in a Pod

Therefore, without first understanding images, containers, image repositories, and Dockerfile, you may encounter issues like:

- Can write Deployment but don't know what's inside the image
- Can modify image tags but don't understand version management logic
- Can check Pod status but don't know the root cause of container startup failures
- Can deploy applications but can't explain how the application enters Kubernetes

Thus, "Image and Container Basics" is the first foundational block for business containerization learning.

---

## II. What is an Image

An image (Image) can be understood as:

**A static encapsulation result of an application and its runtime environment.**

It typically contains:

- Base operating system environment
- Application runtime
- The application itself
- Dependency libraries
- Configuration files (sometimes included, but not recommended to hardcode environment configurations in production)
- Startup command definitions

From an operations perspective, an image can be viewed as:

**A replicable, distributable, and launchable standard software package.**

As long as the image remains consistent, containers launched on different machines or nodes should theoretically have consistent runtime environments.

### Key Characteristics of Images

#### 1. Images are read-only
The image itself is a static read-only content.  
Changes in container runtime do not directly modify the image itself.

#### 2. Images are layered
Images are composed of multiple layers, such as:

- Base image layer
- Runtime environment installation layer
- Application file copy layer
- Startup configuration layer

This makes image building, cache reuse, and distribution more efficient.

#### 3. Images are the core carrier for application delivery
In cloud-native environments, business programs are typically not "uploaded as code", but first built into images, then pulled by the platform to run.

---

## III. What is a Container

A container (Container) can be understood as:

**The running instance of an image.**

If an image is a "template", then a container is the "actual process environment launched from the template".

Example understanding:

- An image is like an installation package or template
- A container is like the program instance running after the installation package

Multiple container instances can be started from the same image.  
These instances share the image content but have independent runtime states.

### Key Characteristics of Containers

#### 1. Containers are in runtime
The image itself does not run; it's the container that runs.

#### 2. Containers are essentially isolated processes
From a Linux perspective, containers are not virtual machines but a group of process environments isolated using mechanisms like namespace and cgroup.

#### 3. Containers have their own temporary writable layer
Images are read-only, but when a container starts, it has a writable layer.  
If no persistence is done, the runtime data in containers usually disappears after deletion.

#### 4. Containers are lighter
Compared to virtual machines, containers don't need to fully simulate an operating system, so they start faster and have lower resource overhead, making them more suitable for cloud-native scenarios.

---

## IV. Relationship Between Images and Containers

This is a fundamental and frequently encountered point in learning.

### 1. Images are static templates
Used to define the environment and content required for application runtime.

### 2. Containers are the running instances of images
Only after an image is started does it become a container.

### 3. One image can start multiple containers
For example, the same Nginx image can run simultaneously in multiple nodes and Pods.

### 4. Changes in container runtime do not automatically rewrite the image
For example, manually creating a file in a container typically only exists in the current container's writable layer and does not become a new image.

### 5. Kubernetes ultimately schedules Pods, but the actual runtime inside Pods is containers
Understanding images and containers helps explain many root causes of Pod startup failures.

---

## V. What is an Image Repository

An image repository (Registry) can be understood as:

**A centralized place for storing and distributing images.**

Its role is similar to a code hosting platform, but it stores images instead of source code.

Common image repositories include:

- Docker Hub
- Harbor
- Alibaba Cloud Container Image Service
- Huawei Cloud SWR
- Tencent Cloud TCR
- Enterprise internal private image repository

### Functions of an Image Repository

#### 1. Unified storage of images
Development, testing, and production environments can pull specific version images from the same image repository.

#### 2. Facilitates version management
Managing different versions through tags, for example:

- `v1.0.0`
- `v1.0.1`
- `prod-20260331`
- `dev-latest`

#### 3. Facilitates distribution
Cluster nodes don't need to store source code; they just need to pull images to run applications.

#### 4. Facilitates permission control and auditing
Enterprise internal image repositories often combine with projects, users, roles, and pull permissions management.

---

## VI. What is a Dockerfile

A Dockerfile can be understood as:

**A script file that defines "how to build an image".**

Its purpose is not to start a container, but to tell the image building tool:

- Which base image to start with
- Which files to copy
- Which dependencies to install
- What the default working directory is
- What command to execute at startup

A Dockerfile is the key entry point for business containerization.  
Because business programs must first be correctly built into images before they can be stably deployed by Kubernetes.

---

## VII. Basic Understanding of Common Dockerfile Instructions

Here's an overview without going into too much detail.

### 1. `FROM`
Specifies the base image.

Examples:

- `nginx:alpine`
- `python:3.11-slim`
- `openjdk:17-jdk`

It indicates which existing image the current image is built upon.

### 2. `WORKDIR`
Specifies the working directory.  
Many subsequent commands will be executed from this directory.

### 3. `COPY`
Copies local files into the image.

Examples:

- Copy application code
- Copy static pages
- Copy configuration files

### 4. `RUN`
Executes commands during image building.  
Commonly used for:

- Installing dependencies
- Installing software packages
- Processing files
- Compiling programs

### 5. `ENV`
Defines environment variables.

### 6. `EXPOSE`
Declares the ports the container will use.  
It's more of a descriptive nature and doesn't equate to exposing services externally.

### 7. `CMD`
Defines the default command to execute when the container starts.

### 8. `ENTRYPOINT`
Defines the main entry program for the container.  
Usually used in conjunction with `CMD`.

---

## VIII. Why a Business Application Needs to Be Converted into an Image First

In traditional methods, business programs may be deployed through the following approaches:

- Deploy code to the server
- Manually install dependencies
- Manually configure the runtime environment
- Manually start the service

The issues with this approach are:

- Inconsistent environments
- Poor repeatability
- High manual operation requirements
- High risk of configuration omissions
- Difficult to standardize delivery

Creating a container image significantly enhances these advantages.

### 1. Uniform runtime environment
Regardless of which node the container is launched on, as long as the image is the same, the base environment will tend to be consistent.

### 2. More standardized application delivery
Shift from "delivering code" to "delivering images".

### 3. More suitable for automated deployment
CI/CD, Kubernetes, Helm, and other tools all revolve around standardized image delivery.

### 4. Easier version management and rollback
Image tags can serve as version identifiers, making rollback more direct.

---

## IX. Relationship between images, containers, repositories, and Dockerfile

These four concepts are often confused and need to be understood in context.

### 1. Dockerfile
Defines "how to build an image".

### 2. Image
The result of the build process, a static package of the application and its runtime environment.

### 3. Image repository
Used to store and distribute images.

### 4. Container
The running instance of an image after it is launched.

This can be understood as a chain:

**Business program → Dockerfile → Build image → Push to image repository → Node pulls image → Launch container**

In Kubernetes scenarios, it extends further to:

**Container → Pod → Service / Ingress → Provide service externally**

---

## X. Core differences between containers and virtual machines

This is a classic question in operations and interviews.

### 1. Virtual machine
Virtual machines typically include:

- Complete guest OS
- Virtual hardware
- Hypervisor layer isolation

Characteristics:

- Strong isolation
- Closer to a full host
- Slower startup
- Higher resource overhead

### 2. Container
Containers typically:

- Share the host's kernel
- Rely on Linux isolation mechanisms
- Only package the application and its runtime environment

Characteristics:

- Lighter
- Faster startup
- More flexible deployment
- Better suited for microservices and cloud-native platforms

### 3. Operational focus
Do not simply equate containers with "small virtual machines".  
Containers are essentially more akin to:

**An isolated, restricted, and encapsulated environment for an application process.**

---

## XI. Common misconceptions in business containerization

### 1. Mistaking an image for a container
An image is a template, a container is a running instance; they are not the same thing.

### 2. Mistaking file changes in a container as permanent
Without rebuilding the image or implementing persistence, these changes typically only exist during the current container's lifecycle.

### 3. Mistaking `EXPOSE` for the service being publicly accessible
`EXPOSE` merely indicates the image will use a certain port, not that traffic is actually open.

### 4. Mistaking Dockerfile as a runtime script
Dockerfile is primarily used for building images, not directly for launching business processes.

### 5. Mistaking having an image as guaranteeing stable Kubernetes operation
An image is just the first step; later steps involve:

- Configuration injection
- Service exposure
- Health checks
- Resource limits
- Storage mounting
- Deployment and troubleshooting

---

## XII. Key containerization points to focus on from an operations perspective

### 1. Is the application startup command clear?
Must know exactly what the container executes upon startup.

### 2. Where are application logs output?
In production, it's typically recommended to output to standard output / standard error for easier centralized collection.

### 3. Is configuration decoupled from the image?
Avoid hardcoding all configurations in the image for different environments.

### 4. Is data persistence required?
If application data is written to the container's temporary layer, it may be lost after the container is deleted.

### 5. Is the image sufficiently concise?
A large base image and excessive dependencies can affect build speed, pull efficiency, and security.

### 6. Is the image version traceable?
Image tags should be clear and manageable; long-term reliance on `latest` is not recommended.

---

## XIII. What level of understanding is needed for image and container basics

At this stage, it's not necessary to dive deep into allBottom mechanisms upfront.  
Reaching the following level is sufficient:

### 1. Be able to clearly explain what images, containers, repositories, and Dockerfile are
### 2. Be able to explain their relationships
### 3. Be able to understand what a simple Dockerfile does
### 4. Be able to understand why applications should first be made into images
### 5. Be able to understand how images enter Kubernetes
### 6. Be able to identify which layer a basic containerization issue belongs to

For example:

- Build failure: mostly Dockerfile / build layer issues
- Pull failure: mostly image repository / network / credential issues
- Start failure: mostly container entry command / dependency / configuration issues

---

## XIV. Recommended learning path after mastering this overview

After understanding this overview, it's recommended to continue learning in the following order.

### 1. Basics of images, containers, and image repositories
Focus on understanding:

- Image tags
- Image naming
- Image pulling and pushing
- Repository authentication

### 2. Dockerfile basics
Focus on understanding:

- `FROM`
- `COPY`
- `RUN`
- `WORKDIR`
- `ENV`
- `CMD`
- `ENTRYPOINT`

### 3. Container operation basics
Focus on understanding:

- Container startup
- Port mapping
- Environment variable passing
- Log viewing
- Container exit reasons

### 4. The process of images entering Kubernetes
Focus on understanding:

- Pod pulling images
- imagePullPolicy
- Pulling from private repositories
- Pod startup and container operation relationship

---

## XV. Common follow-up questions in interviews

Common questions in this area include:

- What's the difference between images and containers?
- What does Dockerfile do?
- What's the difference between containers and virtual machines?
- What's the role of an image repository?
- How to turn a business program into an image?
- Why is containerization more suitable for Kubernetes?
- Why might changes in a container be lost after a restart?
- What's the difference between `CMD` and `ENTRYPOINT`?
- Why is it not recommended to long-term use `latest` in production?

---

## XVI. Stage summary

The basics of images and containers are the true starting point for learning business containerization.

The most important part of this section is not memorizing a lot of commands upfront, but establishing a complete understanding chain:

- Dockerfile defines how to build an image
- An image is the static result of an application and its runtime environment
- Image repositories are used to store and distribute images
- A container is the running instance of an image
- Kubernetes ultimately manages containerized applications through Pods

Only by clearly understanding this chain will subsequent learning about stateless deployments, stateful deployments, configuration management, probes, Services, Ingress, and troubleshooting not become fragmented.

---

## XVII. Keyword quick notes

- Image: Static encapsulation result of an application and its runtime environment
- Container: Running instance of an image after it's launched
- Image repository: Centralized service for storing and distributing images
- Dockerfile: Script file defining how to build an image
- Layered image: Image structure composed of multiple layers
- Container writable layer: Temporary write layer during container runtime
- Standard delivery artifact: In cloud-native environments, typically refers to an image

---

## XVIII. Operational extended understanding /think

From an operations perspective, image and container fundamentals are not merely development knowledge—they are prerequisite capabilities for subsequent application deployment, release management, and troubleshooting.

Many online issues appear to occur in Kubernetes, but their root causes may actually lie earlier:

- Incomplete image build
- Missing dependencies
- Incorrect startup command
- Hardcoded configurations
- Confusing image versions
- Repository authentication failure

Therefore, understanding images, containers, repositories, and Dockerfile is not just about "being able to containerize" but enables rapid determination of which layer the issue belongs to during deployment, release, rollback, and troubleshooting.

---

## References
- Docker Docs:https://docs.docker.com/
- Dockerfile reference:https://docs.docker.com/reference/dockerfile/
- Kubernetes Images:§surl_2§§
- Kubernetes Containers:https://kubernetes.io/docs/concepts/containers/
- OCI Image Spec:https://github.com/opencontainers/image-spec

---

## Tomorrow's Suggestions
Next post suggestion to organize:

[[02-Dockerfile Basics - FROM, AS, WORKDIR, COPY, ADD, RUN, ENV, EXPOSE, CMD and ENTRYPOINT]]