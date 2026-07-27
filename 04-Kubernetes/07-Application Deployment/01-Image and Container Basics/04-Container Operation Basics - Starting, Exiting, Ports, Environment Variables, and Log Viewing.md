# 04-Container Operation Basics: Starting, Exiting, Ports, Environment Variables, and Log Viewing

## Document Description
- Document Location: Basics of Container Operation Layer
- Applicable Stage: Fifth Chapter of Business Containerization Learning / Introduction to Container Operation and Basic Troubleshooting
- Recommended Path: `04-Kubernetes/07-Application Deployment/01-Image and Container Basics/04-Container Operation Basics: Starting, Exiting, Ports, Environment Variables, and Log Viewing`

## Tags
#Kubernetes #Docker #Container #Container Operation #Log #Environment Variable #Port Mapping #Business Containerization #Cloud-Native #Ops #Interview Notes

---

## I. Why Learn Container Operation Basics

The previous chapters mainly addressed:

- What an image is
- How to build an image using Dockerfile
- How to name, tag, push, and pull images

However, understanding "how images are created" is not enough; it's also essential to understand:

**How containers actually run after the image is started.**

In practical deployment and troubleshooting, common issues include:

- The image is successfully built, but the container exits immediately upon startup
- The container has started, but the ports are not accessible
- Environment variables are not taking effect
- Logs are not being output correctly
- Processes seem to be running inside the container, but the service is unavailable
- In Kubernetes, Pod failures often stem from issues at the container operation level

Therefore, mastering container operation basics is a critical step in moving from "being able to build images" to "understanding why applications fail to start."

---

## II. What Is Container Operation

Container operation refers to:

**The process of creating an isolated process environment that actually runs after the image is started.**

The typical steps include:

1. Finding the target image
2. Creating a container instance based on the image
3. Starting the main container process
4. Assigning network, file system, environment variables, and other running conditions to the container
5. Running the application process inside the container

You can think of it this way:

**An image is a static template, while container operation is the process of activating that template.**

---

## III. Key Focus Areas in Container Operation

When learning about container operation, it's recommended to focus on the following five aspects:

### 1. What Happens When the Container Starts
This includes:

- What the main process is
- What the startup command is
- Whether the command executes successfully

### 2. Why Does the Container Exit
A container's exit does not necessarily indicate an error or anomaly. The key points to consider are:

- Whether the main process has ended
- Whether it exited normally or abnormally
- What the exit code is

### 3. Which Ports Does the Container Listen On
It's important to distinguish between:

- Which port the application listens on inside the container
- Which port is mapped on the host machine
- Which port the Kubernetes Service points to

### 4. What Environment Variables Does the Container Obtain
Many application startup behaviors and configuration loading processes are directly related to environment variables.

### 5. Where the Container Logs Are Outputted
If logs are not correctly outputted, subsequent troubleshooting will be much more difficult.

---

## IV. The Essence of Container Startup

The most critical step in container startup is:

**Starting the main process inside the container.**

This main process usually comes from:

- The `CMD` instruction in the Dockerfile
- The `ENTRYPOINT` instruction in the Dockerfile
- Or a command explicitly specified when running the container

### Key Points for Ops Professionals
#### 1. The container's lifecycle is typically tied to its main process
If the main process exits, the container usually exits as well.

#### 2. A container is not a "complete operating system after boot"
Do not think of a container as a complete virtual machine.  
A container is essentially designed to run one or a group of processes with clear primary functions.

#### 3. The main process must be suitable for running in the foreground
If the main process runs in the background by default, the container may:

- Seem to start successfully
- But actually exit immediately
- And then terminate

This is why many images explicitly use parameters to ensure they run in the foreground, such as:

    nginx -g 'daemon off;'

---

## V. Why Do Containers Exit

This is a very common basic question.

### 1. The main process exits normally
If the main process completes its tasks and ends naturally, the container will also exit.

For example:

    docker run busybox echo hello

After the command finishes executing, the container will end.  
This is not an error but expected behavior.

### 2. The main process exits abnormally
Examples include:

- Incorrect startup command
- Missing program files
- Configuration errors
- Unreachable dependent services
- Errors during program startup

These situations### 2. Whether the container is running continuously
It is necessary to check the container status, rather than just whether it has been started briefly.

### 3. Whether the application port is actually being monitored
Sometimes the container process exists, but the application does not actually monitor the port.

### 4. Whether the logs indicate that the application is ready
Many services will print the following in their logs:

- startup complete
- listening on port ...
- connected to database
- ready to serve requests

### 5. Whether the external access path is complete
For truly service-oriented applications, it is also necessary to check:

- Whether Docker port mapping is correct
- Whether the Kubernetes Service is set up correctly
- Whether probes can pass through
- Whether the upstream routing is functioning normally

---

## Chapter Seventeen: To What Extent Should One Master the Basics of Container Operation

At this stage, achieving the following levels is sufficient:

### 1. Being able to explain that container startup essentially involves starting the main process
### 2. Understanding why the exit of the main process leads to the termination of the container
### 3. Distinguishing between internal container ports and external access ports
### 4. Comprehending the role of environment variables in container operation
### 5. Grasping why logs should preferably be output to standard output
### 6. Being able to initially determine which layer a container operation issue belongs to

For example:

- Immediate exit after startup: It is very likely due to issues with the startup command or at the application layer.
- Inaccessible ports: It is most likely related to problems with listening, mapping, or exposure of the ports.
- Absence of logs: This is often due to issues with how logs are being output.
- Configuration not taking effect: This is usually caused by problems with environment variables or configuration injection.

---

## Chapter Eighteen: Common Extended Questions in Interviews

Common questions in this area include:

- Why does a container exit?
- What is the relationship between the main process and the lifecycle of a container?
- What does Docker port mapping mean?
- Does `EXPOSE` mean that the port is already open?
- What are the general uses of environment variables inside a container?
- Why is it recommended to output container logs to stdout / stderr?
- If a container starts successfully but the service does not function, how should one troubleshoot it?
- Why might changes made to files inside a container not be persistent?
- What is the relationship between containers and processes?

---

## Chapter Nineteen: Phase Summary

The core of understanding the basics of container operation does not lie in memorizing many commands, but rather in establishing a cognitive framework for container operation:

- An image is merely a static template.
- A container is an instance that runs after the image is started.
- Whether a container continues to exist depends on whether the main process continues to run.
- To determine if a port is accessible, one must distinguish between listening ports, mapped ports, and exposed links.
- Environment variables are an important means of configuring runtime settings.
- Logs are essential for troubleshooting and should ideally be output to standard output.

Only by clearly understanding these concepts can one properly address issues related to Pod management, Probe health checks, Service configuration, ConfigMap injection, and application troubleshooting within Kubernetes.

---

## Chapter Twenty: Key Terms for Quick Reference

- Main process: The most critical running process after a container starts.
- Container exit: Usually directly related to the termination of the main process.
- Exit code: Indicates the outcome of a process execution.
- Port mapping: Links an external port to a container port.
- Environment variable: A key-value pair passed to an application during runtime.
- Standard output: stdout
- Standard error: stderr
- Log viewing: An important means for troubleshooting containers.

---

## Chapter Twenty-One: Extended Understanding from an Operations Perspective

From an operations standpoint, many issues that arise in Kubernetes actually have their roots at the container operation level:

- The main process of the container does not start.
- A program exits immediately after starting.
- The application does not monitor any ports.
- Environment variables are not passed on.
- Logs are not generated.
- The application is not ready to receive traffic from higher layers.

Therefore, mastering the basics of container operation is not just about being able to run images locally; it is also crucial before proceeding with Kubernetes tasks such as service configuration, probe health checks, ConfigMap injection, and troubleshooting applications in containers.

---

## References
- Docker Docs: https://docs.docker.com/
- Docker run reference: https://docs.docker.com/reference/cli/docker/container/run/
- Docker logs reference: https://docs.docker.com/reference/cli/docker/container/logs/
- Kubernetes Containers: https://kubernetes.io/docs/concepts/containers/
- Kubernetes Logging Architecture: https://kubernetes.io/docs/concepts/cluster-administration/logging/

---

## Next Day's Suggestion
For the next article, it is recommended to organize the following content:

[[0