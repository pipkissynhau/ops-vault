# 02-Dockerfile Basics: FROM, AS, WORKDIR, COPY, ADD, RUN, ENV, EXPOSE, CMD, and ENTRYPOINT

## Documentation Description
- Document Focus: Basics of image building
- Applicable Stage: Third part of business containerization learning / Dockerfile Introduction
- Recommended Path: `04-Kubernetes/07-Application Deployment/01-Image and Container Basics/02-Dockerfile Basics: FROM, AS, WORKDIR, COPY, ADD, RUN, ENV, EXPOSE, CMD, and ENTRYPOINT`

## Tags
#Kubernetes #Docker #Dockerfile #Image Building #Container #Business Containerization #Cloud-Native #Operations #Interview Notes

---

## I. Why You Must Master Dockerfile

In the process of business containerization, Dockerfile is one of the most critical entry files.

This is because business applications do not directly become images; instead, a Dockerfile is needed to instruct the build tool on:

- Which base image to use for construction
- Which files should be included in the image
- What dependencies need to be installed
- Which working directory to use
- What environment variables are required
- What commands to execute when the container starts

Without knowledge of Dockerfile, one can only perform basic tasks like pulling images and running containers but cannot address more critical questions such as:

- How a business image is created
- What exactly is contained within the image
- Why commands fail after the container starts
- Why application files are not in the expected directory
- Why changes to code do not take effect in the container
- Why image building is slow, large in size, or disorganized
- Why a final business image should not include unnecessary compilation tools

Therefore, Dockerfile is a fundamental step from "using containers" to "building business images."

---

## II. What is Dockerfile

Dockerfile is a text file that defines:

**How to build an image step by step.**

It essentially consists of a series of build instructions executed in order.  
When the build tool reads the Dockerfile, it follows these instructions to construct the image layer by layer.

You can think of it as:

**An instruction manual for building images.**

For example, the construction process of a simple business image typically includes:

1. Selecting a base image
2. Specifying a working directory
3. Copying business files
4. Installing dependencies
5. Setting environment variables
6. Exposing application ports
7. Designating the container startup command

These steps are usually written in a Dockerfile.

---

## III. Basic Construction Concepts of Dockerfile

The core idea of Dockerfile is not to "pack everything into" the image but rather to:

**Sequentially add only the minimum necessary content for business operation on top of an appropriate base image.**

A common construction workflow can be summarized as follows:

### 1. Select a Base Image
For example:

- Nginx image
- Python image
- OpenJDK image
- Alpine / Debian / Ubuntu base image

### 2. Prepare Business Files
For example:

- Static web pages
- Configuration files
- Python code
- Java JAR packages
- Startup scripts
- Compressed resources

### 3. Define Dependency Installation and Directory Structure
For example:

- Install pip dependencies
- Install system tools
- Create a working directory
- Copy files to the specified path
- Extract business resources

### 4. Define the Startup Method
For example:

- Start Nginx
- Run Python programs
- Execute Java JARs
- Run shell startup scripts

### 5. Minimize the Final Image Size
In production, not all build tools are included in the final image; only necessary content for business operation is retained.  
This is also a key goal of multi-stage building.

---

## IV. FROM: Specifying a Base Image

`FROM` is one of the most fundamental and core instructions in Dockerfile.

Its function is to:

**Specify the starting point for building the current image.**

For example:

    FROM nginx:1.27-alpine

This indicates that the current image will be built based on the `nginx:1.27-alpine` base image.

Another example:

    FROM python:3.11-slim

This means the current image will use Python 3.11 in its slim version as the runtime foundation.

### Key Points for Operations Professionals

#### 1. Without `FROM`, image building usually cannot begin
It is like "deciding from which operating environment to start."

#### 2. The base image determines many aspects:
For example:

- Whether certain commands are included by default
- The type of operating system
- The size of the image
- The level of security risks exposed
- How dependencies will be installed later

#### 3. Avoid choosing overly large base images
Larger images typically mean:

- SlThe advantages of this approach are that it is clearer and more controllable, making it easier to read and maintain.

### Key Points for Operations and Maintenance Understanding

#### 1. When simply copying files, prioritize using `COPY`.
#### 2. If you need the local compressed package to be automatically extracted, consider using `ADD`.
#### 3. For clearer and more controllable behavior, it is recommended to use `COPY + RUN tar`.

---

## Section Ten: COPY: Key Points for Operations and Maintenance Understanding

### 1. `COPY` copies "files that are visible during the build process"
This means that instead of reading local files after the container starts running, the files are packaged into the image during the build process.

### 2. The path must be clear
Many build issues essentially stem from:
- Incorrect local file paths
- Incorrect target path in the image
- Code being copied to the wrong directory

### 3. Whether to use `COPY` for configuration depends on the context
If the configuration is a basic setting that does not depend on the environment, it can be included with the image.
However, if the configuration is closely related to the environment, it is usually better to inject it at runtime using ConfigMap or Secret, rather than including it directly in the image.

### 4. In multi-stage builds, `COPY` has another common usage
For example:

    COPY --from=builder /app/dist /usr/share/nginx/html

This indicates that:
- The files are not copied from the local directory.
- Instead, they are copied from a previous build stage.

This is also a frequently used pattern in multi-stage builds.

---

## Section Eleven: RUN: Executing Commands During Image Building

The role of `RUN` is to:

**Execute commands during the image building phase.**

For example:

    RUN apt-get update && apt-get install -y curl

This command updates the package index and installs curl during image building.

Another example:

    RUN pip install -r requirements.txt

This command installs Python dependencies during image building.

Still another example:

    RUN tar -xzf /tmp/app.tar.gz -C /app

This command manually extracts a compressed file during the build phase.

### Key Points for Operations and Maintenance Understanding

#### 1. `RUN` is executed during image building, not when the container starts
This is a very common point of confusion.

- `RUN`: Executed during image building.
- `CMD` / `ENTRYPOINT`: Executed when the container starts.

#### 2. `RUN` is often used to install dependencies
For example:
- Installing system packages
-Installing Python packages
- Extracting files
- Changing permissions
- Compiling programs

#### 3. Each `RUN` command typically results in a new image layer
Therefore, when writing a Dockerfile, you need to consider the number of layers, caching, and size of the resulting image.

### Common Misunderstandings

#### Misunderstanding 1: Including business startup commands in `RUN`
For example:

    RUN python app.py

This approach is generally incorrect because the command will only be executed once during image building and will not become the container's startup command.

#### Misunderstanding 2: Having too many `RUN` commands or having them spread across multiple layers
This can lead to an excessive number of image layers, which may affect build efficiency and maintenance.

---

## Section Twelve: ENV: Setting Environment Variables

The role of `ENV` is to:

**Define environment variables within the image.**

For example:

    ENV APP_ENV=prod

This sets the environment variable `APP_ENV=prod` within the image.

### Common Uses

- Defining default environment variables
- Specifying program runtime parameters
- Setting default paths
- Determining the language environment

### Key Points for Operations and Maintenance Understanding

#### 1. `ENV` is suitable for setting default values
However, in a production environment, many variables that change depending on the environment are better injected at runtime using Kubernetes rather than being hardcoded into the image.

#### 2. Do not include sensitive information directly in the Dockerfile
For example, database passwords or keys should not be stored in the `ENV` of a Dockerfile.

#### 3. Differentiate between "default values" and "production configuration"
Default values can be included in the image, but values that are strongly dependent on the environment are better injected at deployment time.

---

## Section Thirteen: EXPOSE: Declaring Ports Used by the Container

The role of `EXPOSE` is to:

**Specify the ports that the applications inside the container intend to use.**

For example:

    EXPOSE 80

This indicates that the application inside the container typically listens on port 80.

### Key Points for Operations and Maintenance Understanding

#### 1. `EXPOSE` is more of a descriptive tool
It does not automatically expose the ports.

#### 2. Actual traffic exposure requires additional mechanisms
### Summary of the Chapter

Dockerfile serves as the fundamental entry point for building images. The most important aspect here is not to memorize all the details immediately, but rather to establish a few core understandings:

- `FROM` determines the base environment from which construction begins.
- `AS` is used to name the build stage and is essential for multi-stage builds.
- `WORKDIR` sets the default working directory.
- Both `ADD` and `COPY` can add files to the image, but their behaviors differ; `COPY` is more straightforward for copying files.
- `RUN` specifies commands to execute during the build phase.
- `ENV` defines default environment variables.
- `EXPOSE` declares the ports the container will use.
- `CMD` determines how the container starts by default.
- `ENTRYPOINT` identifies the main entry program of the container.

By clearly understanding the role of these basic instructions, you can avoid confusing the build phase with the runtime phase when learning about image optimization, container operation, Kubernetes deployment, and troubleshooting.