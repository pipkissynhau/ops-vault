# 01 - The First Practical Exercise: Building Nginx Static Page Images and Running Containers

## Document Description
- Document Position: The first complete practical exercise in business containerization
- Applicable Stage: The first hands-on practice after mastering image and container basics
- Recommended Path: `04-Kubernetes/07-Application Deployment/02-Stateless Application Deployment/01-The First Practical Exercise: Building Nginx Static Page Images and Running Containers`

## Tags
#Kubernetes #Docker #Nginx #Static Pages #Image Building #Container Running #Stateless Applications #Business Containerization #Cloud-Native #Ops #Interview Notes

---

## I. Why Choose Nginx Static Pages for the First Practical Exercise

In learning business containerization, the first practical object should ideally meet the following criteria:

- Simple enough
- Requires few dependencies
- Easy to start up
- Easy to observe results
- Covers basic processes such as image building, container running, port access, and log checking

Nginx static pages are an excellent choice for the first practice because they:

- Have mature and stable base images
- Have clear startup commands
- Have simple and intuitive page content
- Can be directly verified through a browser after the container starts
- Can smoothly transition into Kubernetes Deployment, Service, and Ingress later on

Therefore, the focus of this exercise is not to learn all the configuration details of Nginx itself, but to use a simple business object to establish the following links:

**Static page files → Dockerfile → Image building → Container startup → Port access → Log checking → Basic troubleshooting**

---

## II. Goals of This Practical Exercise

After completing this exercise, you should be able to do at least the following:

### 1. Understand how a simple business can be packaged into an image
### 2. Comprehend the basic Dockerfile for Nginx static page images
### 3. Build and start a container locally
### 4. Verify if the business is functioning properly through port access
### 5. Determine if the container is running correctly by checking logs and status
### 6. Gain a preliminary understanding of the characteristics of "stateless applications"

---

## III. Description of the Practical Exercise Object

The object of this exercise is a very simple Nginx static page service.

Its structure is usually quite straightforward:

- An `index.html` file
- A Dockerfile
- Based on the official Nginx image
- The page files are copied to Nginx's default website directory
- The Nginx frontend process is started

---

## IV. Prepare a Simple Static Page

You can start by preparing a simple `index.html` file, for example:

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

### Key Points for Ops Professionals

This page itself doesn't contain any complex logic, but it's perfect for observing the following issues:

- Whether the files have been correctly copied into the image
- Whether Nginx has started successfully
- Whether the port is open properly
- Whether the container access link is functioning correctly

---

## V. Prepare a Dockerfile

A very simple Dockerfile can be written as follows:

    FROM nginx:1.27-alpine

    COPY index.html /usr/share/nginx/html/

### Meaning of This Dockerfile

#### 1. Use the official Nginx image as the base image
This means the image already includes:

- The Nginx program
- The default website directory
- The default startup settings

#### 2. Copy the local `index.html` file to Nginx's default website directory
This way, when the container starts, accessing the root path of Nginx will display this page.

### Why No Need to Write a CMD Here
Since the `nginx:1.27-alpine` official image already defines a default startup command, there's no need to write one separately.

This highlights an important practical tip:

**Not all Dockerfiles require a custom CMD.**
If the base image provides a suitable default setup, you can simply use it.

---

## VI. Basic Understanding of Nginx's Default Website Directory

The common default page directory for official Nginx images is:

    /usr/share/nginx/html/

When accessing the root path of Nginx, it typically retrieves page files from this directory.

### Key Points for Ops Professionals

In this exercise, `COPY index.html /usr/share/nginx/html/` essentially does the following:

**Overwrites or replaces the default website content with your own business pages.**

This is also the most basic method for building static page images.

- Check if the local port is being used

### 3. Inability to access the page
Common reasons:

- The container is not running properly
- Incorrect host machine port mapping
- Local firewall or network restrictions
- Wrong access address

Inspection directions:

- Check the container status
- Verify port mapping
- Review logs
- Ensure that you are accessing the host machine port, not an internal container port

### 4. The page displays non-custom content
Common reasons:

- `index.html` file was not copied over
- Incorrect file path
- Image was not rebuilt
- A different image was started

Inspection directions:

- Check the Dockerfile COPY command path
- Rebuild the image
- Ensure that the correct tag of the image is being used