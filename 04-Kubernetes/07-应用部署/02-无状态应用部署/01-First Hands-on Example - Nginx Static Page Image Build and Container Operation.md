# 01-First Practical: Building and Running an Nginx Static Page Image in a Container

## Document Notes
- Document Position: First complete practical for business containerization
- Applicable Stage: First hands-on practice after image and container basics
- Recommended Path: `04-Kubernetes/07-Apply deployment/02-No status application deployment/01-First battle:Nginx Static page mirror construction and container running`

## Tags
#Kubernetes #Docker #Nginx #StaticPage #MirrorBuild #ContainerRun #NoStatusApplication #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why Choose Nginx Static Page for the First Practical

In business containerization learning, the first practical object should ideally meet these characteristics:

- Simple enough
- Minimal dependencies
- Easy to start
- Easy to observe results
- Covers basic chain of image building, container runtime, port access, log viewing, etc.

Nginx static page is an ideal choice for the first practice, reasons include:

- Mature and stable base image
- Clear startup command
- Simple and intuitive page content
- Direct browser verification after container startup
- Natural transition to Kubernetes Deployment, Service, Ingress later

Therefore, this practical's focus is not to learn all Nginx configuration details, but to use a simple business object to complete the following chain:

**Static page file → Dockerfile → Image building → Container startup → Port access → Log viewing → Basic troubleshooting**

---

## II. Objectives of This Practical

After completing this practical, it's recommended to at least achieve the following:

### 1. Understand how a simplest business enters an image
### 2. Understand the basic Dockerfile of an Nginx static page image
### 3. Build and start a container locally
### 4. Verify business normality through port access
### 5. Judge container normality through logs and runtime status
### 6. Understand the characteristics of "stateless application" initially

---

## III. Practical Object Explanation

The practical object is a simplest Nginx static page service.

Its structure is usually very simple:

- A `index.html`
- A Dockerfile
- Based on Nginx official image
- Copy page file to Nginx default site directory
- Start Nginx foreground process

---

## IV. Prepare a Simplest Static Page

First prepare a simple `index.html` file, for example:

    <!DOCTYPE html>
    <html lang="zh-CN">
    <head>
      <meta charset="UTF-8">
      <title>nginx demo</title>
    </head>
    <body>
      <h1>Hello Nginx Container</h1>
      <p>This is my first containerized web page.</p>
    </body>
    </html>

### Operations Understanding Focus

This page itself has no complex logic, but it's sufficient to observe the following issues:

- Whether the file is correctly copied into the image
- Whether Nginx successfully starts
- Whether the port is normally open
- Whether the container access chain is normal

---

## V. Prepare Dockerfile

A simplest Dockerfile can be written as:

    FROM nginx:1.27-alpine

    COPY index.html /usr/share/nginx/html/

### Meaning of This Dockerfile

#### 1. Use Nginx official image as base image
The image already includes:

- Nginx program
- Default website directory
- Default startup method

#### 2. Copy local `index.html` to Nginx default site directory
After container startup, accessing Nginx root path will show this page.

### Why No Need to Write CMD Manually
Because `nginx:1.27-alpine` official image already defines default startup command.

This shows an important practical experience:

**Not all Dockerfiles must define CMD manually.**
If the base image provides suitable default startup method, it can be directly reused.

---

## VI. Basic Understanding of Nginx Default Site Directory

The common default page directory of Nginx official image is:

    /usr/share/nginx/html/

When accessing Nginx root path, it usually reads page files from this directory.

### Operations Understanding Focus

In this practical, the essence of `COPY index.html /usr/share/nginx/html/` is:

**Overwrite or replace your business page to Nginx default site directory.**

This is the most basic way in static page image building.

---

## VII. Local Image Building

Assuming current directory has:

- `Dockerfile`
- `index.html`

Then image building can be performed, for example:

    docker build -t nginx-web:v1 .

### Meaning of This Command

#### 1. `docker build`
Execute image building.

#### 2. `-t nginx-web:v1`
Name and tag the image:

- Image name: `nginx-web`
- Tag: `v1`

#### 3. `.`
Indicates current directory is build context.

---

## VIII. What to Pay Attention After Image Building

After successful image building, don't just check "success", but establish several basic inspection awareness.

### 1. Is the image name correct
For example, is it really:

    nginx-web:v1

### 2. Are there any abnormal prompts during building
For example:

- File not found
- Dockerfile path error
- COPY failed
- Base image pull failed

### 3. Is the building speed normal
If a simplest static page image builds unusually slowly, start thinking:

- Network issues
- Base image pull issues
- Local environment issues

---

## IX. Start Container to Run Image

After successful image building, start container, for example:

    docker run -d --name nginx-web -p 8080:80 nginx-web:v1

### Meaning of This Command

#### 1. `-d`
Run container in background.

#### 2. `--name nginx-web`
Give container a name for easier subsequent viewing and management.

#### 3. `-p 8080:80`
Do port mapping:

- Host 8080
- Mapped to container 80

#### 4. `nginx-web:v1`
Specify the image to run.

---

## X. Why Use `8080:80` Here

Because Nginx container usually listens on port 80 internally.

If host also uses port 80 directly, it may conflict with existing services on the host.  
So during local practice, it's common to map host port to 8080 for easier observation.

### Understanding This Port Pair

- `80`: Application listening port inside container
- `8080`: Host access port

When accessing, generally access host's 8080, not directly access container's 80.

## Eleven. How to Verify the Page is Working

After the container starts, you can access it via the browser:

    http://HostIP:8080

If accessing locally, common addresses are:

    http://127.0.0.1:8080
    http://localhost:8080

If the page displays the custom `Hello Nginx Container` normally, it likely means the following chain has been successfully established:

- Dockerfile is correct
- Page files were copied successfully
- Image build succeeded
- Container started successfully
- Nginx is running normally
- Port mapping is correct

---

## Twelve. How to Check Container Status

After local success, it's important to develop the habit of checking container status.  
Common commands like:

    docker ps

are mainly used to view currently running containers.

### Operations Understanding Focus

The focus here isn't memorizing commands, but forming awareness:

- Page accessibility doesn't mean troubleshooting is complete
- Need to confirm the container is actually running
- Need to establish the habit of combining "running status + logs + access results" for judgment

---

## Thirteen. How to Check Container Logs

Common commands for viewing logs include:

    docker logs nginx-web

### Why Check Logs
Because logs often provide the first clues when pages are inaccessible, containers are abnormal, or services are unstable.

For this Nginx practice, logs commonly show:

- Nginx startup information
- Request access records
- Error messages (if configuration or page path is abnormal)

### Operations Understanding Focus

Even if the page is accessible, it's recommended to check the logs.  
This helps establish a basic understanding:

**What the logs look like when the container runs successfully.**

This makes it easier to identify anomalies when problems occur later.

---

## Fourteen. What Type of Application Is This

This Nginx static page service is a typical example of:

**A stateless application.**

### Why It's Stateless

Because it usually has these characteristics:

- No identity differences between multiple instances
- Any instance can provide the same content
- Doesn't store critical business data locally
- Page content still comes from the image itself after container deletion and restart
- Suitable for Deployment multi-replica deployment

### This Is Important
Because after entering Kubernetes, you need to start training yourself to judge:

- Whether the business is stateless or stateful
- Whether to use Deployment or StatefulSet

Nginx static pages are the most typical practice for stateless applications.

---

## Fifteen. Several Key Awareness to Establish in This Practice

### 1. Business Files Must Enter the Image
If `index.html` hasn't been included in the image via `COPY`, even if the container starts, it won't display its own page.

### 2. Internal Container Port and External Access Port Are Different
Container listening on 80 doesn't mean external access is necessarily on 80.

### 3. Container Can Start, Doesn't Mean Business Is Correct
Also check:

- Whether the page content is correct
- Whether the logs are normal
- Whether the port chain is correct

### 4. Start Separating Understanding of Image Layer, Container Layer, and Access Layer
This practice already includes three layers:

- Image layer: Dockerfile, COPY, build
- Container layer: run, main process, logs
- Access layer: port mapping, browser access

---

## Sixteen. Common Issues and Troubleshooting Approaches

### 1. Build Error: `COPY failed`
Common causes:

- `index.html` not in current build directory
- File name written incorrectly
- Build context is incorrect

Troubleshooting direction:

- Check if the file actually exists in the current directory
- Check if the Dockerfile path is correct
- Check if the execution directory for `docker build` is correct

### 2. Container Start Failure
Common causes:

- Image name written incorrectly
- Local image doesn't exist
- Port conflict
- Docker environment anomaly

Troubleshooting direction:

- Check if the image was built successfully
- Check container logs
- Check if the host port is occupied

### 3. Page Not Accessible
Common causes:

- Container not running normally
- Host port mapping error
- Local firewall or network restrictions
- Access address written incorrectly

Troubleshooting direction:

- Check container status
- Check port mapping
- Check logs
- Confirm accessing the host port, not the container internal port

### 4. Page Displays Non-Custom Content
Common causes:

- `index.html` not copied into the image
- File path incorrect
- Image not rebuilt
- Running the wrong image tag

Troubleshooting direction:

- Check Dockerfile COPY path
- Rebuild
- Confirm running the correct tag image

---

## Seventeen. What This Practice Can Naturally Transition To

After completing this exercise, you can naturally move to Kubernetes-related topics.

### 1. Deployment
Assign this Nginx image to be managed by Deployment.

### 2. Service
Convert container access to a stable internal access entry in Kubernetes.

### 3. Ingress
Expose the page via domain or path.

### 4. ConfigMap
Decouple Nginx configuration files or page content from the image.

### 5. Probe
Configure health checks for the Nginx container.

This is why using Nginx static pages as the first practice is suitable:  
It naturally extends to Kubernetes orchestration layers.

---

## Eighteen. What Level Should You Reach to Learn This Practice

At this stage, it's recommended to reach the following levels:

### 1. Be able to independently prepare a simple `index.html`
### 2. Be able to understand and write the most basic Nginx Dockerfile
### 3. Be able to build an image and start a container
### 4. Be able to access the page via port mapping
### 5. Be able to check container logs and running status
### 6. Be able to explain why it belongs to a stateless application

---

## Nineteen. Possible Interview Extensions

Common questions in this area include:

- How to turn a static page into an image
- Why Nginx static pages are suitable as stateless applications
- Why a Dockerfile with only FROM and COPY can start
- What does `-p 8080:80` mean
- What to check first when the page is inaccessible
- Possible reasons when the container starts but the page isn't as expected
- Why Nginx static sites are suitable for Deployment
- What differences exist between Docker local runtime and Kubernetes deployment in access chains

---

## Twenty. Stage Summary

Building an Nginx static page image and running the container is the most suitable first complete practice for business containerization.

Its value doesn't lie in how complex the page itself is, but in helping quickly establish the following core foundational links:

- How business files enter the image
- How the image is built
- How the container runs the image
- How ports are mapped and accessed
- How to check logs
- Why a business can be classified as a stateless application

Once this chain is established, subsequent learning about Kubernetes's Deployment, Service, Ingress, Probe, and ConfigMap will be much smoother.

---

## Twenty-One. Keyword Quick Notes

- Static Page: A page directly returned by the Web server as file content
- Nginx: Common Web server and reverse proxy service
- Site Directory: Nginx default directory for storing web file content
- Port Mapping: Host port mapped to container port
- Stateless Application: Instances can be replaced, typically without relying on local critical state
- Container Logs: Key troubleshooting entry point during runtime

---

## 22. Operational Extension Understanding

From an operations perspective, although this practical exercise is simple, it already covers the most basic problem layers in application deployment:

- Build Layer: Whether files have entered the image
- Runtime Layer: Whether the container has actually started
- Access Layer: Whether port mapping is correct
- Content Layer: Whether business pages meet expectations

Many subsequent issues encountered in Kubernetes essentially just move these layered relationships to a more complex platform. 
Therefore, clarifying this simplest practical exercise first is more effective than directly using complex middleware from the start.

---

## References
- NGINX Docker Hub: https://hub.docker.com/_/nginx
- Docker Docs: https://docs.docker.com/
- Docker build reference: https://docs.docker.com/reference/cli/docker/buildx/build/
- Docker run reference: https://docs.docker.com/reference/cli/docker/container/run/
- Docker logs reference: https://docs.docker.com/reference/cli/docker/container/logs/

---

## Next Day Suggestions
Next post suggestion to organize:

[[Static Page Integration with Kubernetes - Deployment and Service Fundamentals]]