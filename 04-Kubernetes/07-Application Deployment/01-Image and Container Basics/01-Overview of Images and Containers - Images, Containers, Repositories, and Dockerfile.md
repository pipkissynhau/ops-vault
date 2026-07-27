# 01-Overview of Images and Containers: Images, Containers, Repositories, and Dockerfile

## Document Description
- Document Purpose: Introduction to the basics of business containerization
- Applicable Phase: Chapter 2 of Kubernetes Application Deployment / Overview of Images and Containers
- Recommended Path: `04-Kubernetes/07-Application Deployment/01-Images and Containers/01-Overview of Images and Containers: Images, Containers, Repositories, and Dockerfile`

## Tags
#Kubernetes #Containers #Images #Docker #Dockerfile #Image Repository #Business Containerization #Application Deployment #Cloud-Native #Ops #Interview Notes

---

## I. Why Learn the Basics of Images and Containers First

Before learning about Kubernetes application deployment, it is essential to understand one thing clearly:

**What exactly does Kubernetes manage?**

From a runtime perspective, Kubernetes does not directly execute “source code,” nor “compressed packages” or “configuration directories.” The core units it schedules and manages are ultimately:

- Images
- Containers
- Container instances within Pods

Therefore, if you do not first understand images, containers, image repositories, and Dockerfile, you may encounter the following issues:

- You can write Deployment scripts but do not know what is actually inside the images.
- You can modify image tags but do not understand the logic of image version management.
- You can monitor Pod status but do not know the root causes of container startup failures.
- You can deploy applications but cannot clearly explain how the business programs enter Kubernetes.

Thus, “the basics of images and containers” form the first foundation in learning about business containerization.

---

## II. What is an Image

An image can be understood as:

**A static encapsulation of an application and its runtime environment.**

It typically includes:

- The basic operating system environment
- The application runtime
- The application itself
- Dependencies
- Configuration files (sometimes included, but it is not recommended to hardcode all environmental settings in production)
- Definition of startup commands

From an operations perspective, an image can be seen as:

**A standard software package that can be replicated, distributed, and started.**

As long as the images are identical, containers launched from these images on different machines or nodes should theoretically have the same runtime environment.

### Key Characteristics of Images

#### 1. Images are read-only
The image itself is static and read-only.
Changes made during container runtime will not directly modify the image itself.

#### 2. Images are layered
Images consist of multiple layers, such as:

- The base image layer
- The runtime environment installation layer
- The application file copy layer
- The startup configuration layer

This structure makes image building, cache reuse, and distribution more efficient.

#### 3. Images are the core medium for application delivery
In a cloud-native environment, business programs are usually not directly “uploaded with code.” Instead, images are first built, and then the platform pulls these images to run them.

---

## III. What is a Container

A container can be understood as:

**The running instance after an image is started.**

If an image is considered a “template,” then a container is the “actual process environment created from that template.”

For example:

- An image is like an installation package or a template.
- A container is like the program instance after the installation package is run.

The same image can be used to start multiple container instances.  
These instances share the image content but have their own independent runtime states.

### Key Characteristics of Containers

#### 1. Containers are in a running state
The image itself does not run; it is the container that actually executes operations.

#### 2. Containers are essentially isolated processes
From a Linux perspective, containers are not virtual machines but rather groups of process environments isolated through mechanisms such as namespaces and cgroups.

#### 3. Containers have their own temporary writable layer
Although images are read-only, containers have a writable layer after they start.  
If no persistence is set up, these runtime data will be lost when the container is deleted.

#### 4. Containers are more lightweight
Compared to virtual machines, containers do not need to fully simulate an entire operating system. They start faster and require fewer resources, making them more suitable for cloud-native scenarios.

---

## IV. The Relationship between Images and Containers

This is a fundamental and frequently discussed topic in learning.

### 1. An image is a static template
It defines the environment and content required for the application to run.

### 2. A container is an instance of an image
An image becomes a container only after it is started.

### 3. One image can start multiple containers
For example, the same Nginx image can run on multiple nodes and Pods simultaneously.

### 4. Changes during container runtime will not automatically update the image
For instance, if you manually create a file inside a container, this change will only exist in that container’s writable layer and will not be reflected## Thirteen: To What Extent Should One Master the Basics of Learning Images and Containers

At this stage, there is no need to delve into all underlying mechanisms from the outset. Reaching the following level is sufficient:

### 1. Be able to clearly explain what images, containers, repositories, and Dockerfiles are.
### 2. Understand the relationships between them.
### 3. Comprehend what a simple Dockerfile does.
### 4. Recognize why businesses should first transform their applications into images.
### 5. Grasp how images are integrated into Kubernetes.
### 6. Identify which layer basic containerization issues pertain to.

For example:

- Construction failures: Often stem from Dockerfiles or construction layers.
- Pulling failures: Usually due to image repositories, network issues, or authentication problems.
- Startup failures: Frequently caused by container entry commands, dependencies, or configuration errors.

---

## Fourteen: Recommended Further Learning Path

After mastering this overview, it is recommended to proceed with the following sequence:

### 1. Basics of Images, Containers, and Image Repositories
Focus on understanding:

- Image tags.
- Image naming conventions.
- Image pulling and pushing processes.
- Repository authentication methods.

### 2. Dockerfile Basics
Focus on understanding:

- `FROM`.
- `COPY`.
- `RUN`.
- `WORKDIR`.
- `ENV`.
- `CMD`.
- `ENTRYPOINT`.

### 3. Basics of Container Operation
Focus on understanding:

- Container startup processes.
- Port mapping techniques.
- Environment variable transmission methods.
- Log viewing approaches.
- Reasons for container exit.

### 4. The Process of Integrating Images into Kubernetes
Focus on understanding:

- How Pods pull images.
- The role of `imagePullPolicy`.
- Pulling from private repositories.
- The relationship between Pod startup and container operation.

---

## Fifteen: Common Extended Questions in Interviews

Common questions in this area include:

- What is the difference between an image and a container?
- What is the purpose of a Dockerfile?
- How does a container differ from a virtual machine?
- What is the role of an image repository?
- How can a business application be transformed into an image?
- Why is containerization more suitable for Kubernetes?
- Why might changes made to files inside a container be lost after a restart?
- What is the difference between `CMD` and `ENTRYPOINT`?
- Why is it not recommended to use `latest` images in production environments?

---

## Sixteen: Phase Summary

The basics of images and containers mark the true beginning of learning about business containerization. The most important thing here is not to memorize a large number of commands but to establish a comprehensive understanding of these concepts:

- Dockerfiles define how images are built.
- Images are static encapsulations of applications and their runtime environments.
- Image repositories store and distribute images.
- Containers are running instances created from images.
- Kubernetes manages containerized applications through Pods.

Only by clearly understanding this framework can subsequent learning of stateless and stateful deployments, configuration management, probes, Services, Ingresses, and troubleshooting be more effective and less fragmented.

---

## Seventeen: Key Terms for Quick Reference

- Image: A static encapsulation of an application and its runtime environment.
- Container: A running instance created from an image.
- Image Repository: A service that stores and distributes images.
- Dockerfile: A script that defines how an image is built.
- Multi-layer Image: An image structure composed of multiple layers.
- Container Writeable Layer: A temporary layer where data can be written during container operation.
- Standard Delivery Unit: In cloud-native environments, often refers to an image.

---

## Eighteen: Extended Understanding from an Operations Perspective

From an operations standpoint, the basics of images and containers are not just development knowledge but also essential prerequisites for subsequent application deployment, release management, and troubleshooting. Many online issues that appear in Kubernetes may actually have their roots earlier in the process:

- Incomplete image construction.
- Missing dependencies.
- Incorrect startup commands.
- Hard-coded configurations.
- Confused image versions.
- Failed repository authentication.

Therefore, understanding images, containers, repositories, and Dockerfiles is not just about being able to perform containerization tasks but also about being able to quickly identify the root cause of issues during deployment, release, rollback, and troubleshooting processes.

---

## References
- Docker Docs: https://docs.docker.com/
- Dockerfile reference: https://docs.docker.com/reference/dockerfile/
- Kubernetes Images: https://kubernetes.io/docs/concepts/containers/images/
- Kubernetes Containers: https://kubernetes.io/docs/concepts/containers/
- OCI Image Spec: https://github.com/opencontainers/image-spec

---

## Next Day's Suggestions
It is recommended to organize the following content for the next article:

[[02-Dockerfile Basics: FROM, AS, WORKDIR, COPY, ADD, RUN, ENV, EXPOSE, CMD, and ENTRYPOINT]]