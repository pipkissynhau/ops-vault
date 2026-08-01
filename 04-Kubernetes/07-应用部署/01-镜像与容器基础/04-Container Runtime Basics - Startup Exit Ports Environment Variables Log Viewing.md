# 04-Container Runtime Basics: Starting, Exiting, Ports, Environment Variables, and Log Viewing

## Document Notes
- Document Position: Container Runtime Layer Basics
- Applicable Stage: Business Containerization Learning Fifth Chapter / Container Runtime and Basic Troubleshooting Introduction
- Recommended Path: `04-Kubernetes/07-Application Deployment/01-Image and Container Basics/04-Container Runtime Basics: Starting, Exiting, Ports, Environment Variables and Log Viewing

## Tags
#Kubernetes #Docker #Containers #ContainerRun #Log #EnvironmentalVariables #PortMap #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why Learn Container Runtime Basics

The previous chapters mainly addressed:

- What is an image
- How to build an image with Dockerfile
- How to name, tag, push, and pull images

But merely understanding "how images are created" is insufficient. You must also understand:

**What happens when an image is started and how the container runs.**

Because in actual deployment and troubleshooting, you often encounter issues like:

- The image can be built successfully, but the container exits immediately after startup
- The container is running, but the port is unreachable
- Environment variables are not effective
- Logs are not output normally
- Processes appear to be running in the container, but the business is actually unavailable
- Kubernetes Pod anomalies, with the root cause being container runtime layer issues

Therefore, container runtime basics is the key step from "knowing how to build images" to "judging why the application fails to start."

---

## II. What is Container Runtime

Container runtime refers to:

**The process of activating an image into an actual running isolated process environment.**

From a process perspective, it typically involves:

1. Finding the target image
2. Creating a container instance based on the image
3. Starting the container's main process
4. Allocating network, file system, environment variables, and other runtime conditions to the container
5. Running the application process within the container

You can think of it as:

**An image is a static template, while container runtime is the process of activating the template.**

---

## III. Core Focus Points of Container Runtime

When learning container runtime, it's recommended to first focus on these five aspects:

### 1. What is executed when the container starts
That is:

- What is the main process
- What is the start command
- Whether the command can be executed normally

### 2. Why does the container exit
Container exit does not necessarily mean an error or anomaly.  
The key is to determine:

- Whether the main process has ended
- Whether it exited normally or abnormally
- What is the exit code

### 3. Which ports is the container listening on
You need to distinguish:

- Which port the application is listening on inside the container
- Which port is mapped on the host
- Which port the Kubernetes Service points to

### 4. Which environment variables does the container have
Many application startup behaviors and configuration loading logic are directly related to environment variables.

### 5. Where are the container logs output
If logs are not output correctly, troubleshooting will be very passive.

---

## IV. The Essence of Container Startup

When a container starts, the most critical thing is:

**Starting the main process in the container.**

This main process usually comes from:

- The `CMD` in the Dockerfile
- The `ENTRYPOINT` in the Dockerfile
- Or the explicitly specified command when running the container

### Operations Understanding Focus

#### 1. Container lifecycle is typically bound to the main process
If the main process exits, the container usually exits as well.

#### 2. A container is not a "complete operating system after boot"
Do not consider a container as a complete virtual machine.  
A container is essentially an environment to run one or more processes with clear main responsibilities.

#### 3. The main process must be suitable for foreground operation
If the main process defaults to running in the background, the container may experience:

- The business appears to have started
- But the main process has actually exited
- The container terminates immediately

This is why many images explicitly use foreground operation parameters, such as:

    nginx -g 'daemon off;'

---

## V. Why Does a Container Exit

This is a highly frequent foundational issue.

### 1. The main process exits normally
If the main process completes its task and exits naturally, the container will also exit.

For example:

    docker run busybox echo hello

The container naturally exits after the command is executed.  
This is not a fault, but an expected behavior.

### 2. The main process exits abnormally
For example:

- The start command is written incorrectly
- The program file does not exist
- The configuration is incorrect
- The dependent service is unreachable
- The program throws an error during startup

These situations fall under abnormal exits.

### 3. The main process is killed
For example:

- The system kills it due to insufficient memory
- The container's resource limits trigger OOM
- The container is manually stopped externally

### Operations Understanding Focus
"Container exit" itself is not an issue; the key is to determine:

- Why did it exit
- Whether the exit was expected
- What do the exit code and logs indicate

---

## VI. Basic Understanding of Exit Codes

Exit codes (Exit Code) are important signals for determining the result of container process execution.

### 1. `0`
Usually indicates:

**Normal exit.**

For example, when executing a one-time task and exiting afterward, this is often a normal phenomenon.

### 2. Non-`0`
Usually indicates:

**Abnormal exit.**

For example:

- Command execution fails
- Configuration error
- File not found
- Startup error

### 3. Common Operations Understanding
In container or Kubernetes troubleshooting, if you see a container exit, you need to combine:

- Exit code
- Logs
- Start command
- Application scenario

to make a judgment.

For example:

- Job container exit code 0: may indicate normal completion
- Web service container exit code 0: may indicate the main process did not start correctly
- Container exit code non-0: likely requires log inspection to locate the error

---

## VII. How to View Ports Correctly

Container port issues are also frequent points of confusion.

### 1. Port the application listens on inside the container
This is the port the application actually listens on inside the container.  
For example, Nginx listens on 80 inside the container, Flask listens on 5000.

### 2. Docker port mapping
If using Docker locally, port mapping is commonly done, such as:

    -p 8080:80

The meaning is usually:

- Host port 8080
- Mapped to container port 80

### 3. Port relationships in Kubernetes
In Kubernetes, you also need to distinguish:

- `containerPort`
- `targetPort`
- `port`
- `nodePort`

### Operations Understanding Focus
When a port is unreachable, you cannot just look at a single number. You need to clarify:

- Which port the application actually listens on
- Which port is used inside the container
- Which port the host or Service exposes
- Which port the Ingress ultimately forwards to

---

## VIII. Why "Port is Written" Does Not Equal "Port is Accessible"

This is a very common misconception.

### 1. `EXPOSE` is written in Dockerfile
This only declares the expected port for the image, not necessarily open to the outside.

### 2. `containerPort` is written in Kubernetes
This is mainly a declaration and auxiliary configuration, not automatically opening access paths.

### 3. The application must actually listen on the port
If the application fails to start or the listening port does not match the configuration, external access is impossible.

### 4. External access requires additional exposure links
For example:

- Docker needs `-p`
- Kubernetes needs a Service
- External HTTP/HTTPS access may also require Ingress

---

## IX. What Are Environment Variables

Environment variables can be understood as:

**A set of key-value parameters passed to an application at runtime.**

For example:

- `APP_ENV=prod`
- `PORT=8080`
- `DB_HOST=mysql`
- `LOG_LEVEL=info`

Applications typically read these variables at startup to determine:

- Current environment
- Service listening port
- Database address
- Log level
- Feature switches

---

## X. Why Are Environment Variables Important

### 1. Make Images More Universal
The same image can adapt to different environments through environment variables:

- Development environment
- Testing environment
- Production environment

### 2. Avoid Hardcoding All Configurations
If all parameters are hardcoded in the image, it leads to:

- Rebuilding the image when switching environments
- Inflexible configuration changes
- Difficult operations management

### 3. Facilitate Kubernetes Configuration Injection
In Kubernetes, common methods include:

- `env`
- `envFrom`
- ConfigMap
- Secret

To inject environment variables into containers.

---

## XI. Common Scenarios for Environment Variables

### 1. Specify the Running Environment
For example:

- `APP_ENV=dev`
- `APP_ENV=test`
- `APP_ENV=prod`

### 2. Specify Service Port
For example:

- `PORT=8080`

### 3. Specify Database Connection Information
For example:

- `DB_HOST`
- `DB_PORT`
- `DB_NAME`

### 4. Specify Log Level
For example:

- `LOG_LEVEL=debug`
- `LOG_LEVEL=info`

### 5. Control Feature Switches
For example:

- `FEATURE_X_ENABLED=true`

---

## XII. Why Container Logs Are Important

In containerized environments, logs are one of the most direct troubleshooting entry points.

Often, issues like containers failing to start, frequent exits, or probe failures fundamentally require checking logs first.

### Common Information That Can Be Seen from Logs

- Whether the program actually starts
- Whether configurations are successfully read
- Whether dependent services are reachable
- Whether ports are successfully bound
- Whether syntax or parameters are incorrect
- What the program's exception stack trace is

---

## XIII. Why Logs Are Usually Required to Output to Standard Output in Production

In containerized and Kubernetes scenarios, the common best practice is:

**Application logs should be output to stdout / stderr as much as possible.**

### Reasons Include

#### 1. Easier Unified Collection
Container platforms can directly collect standard output logs from containers.

#### 2. Not Dependent on Local Files Inside the Container
If logs are only written to container internal files:

- Logs may be lost after the container is destroyed
- The log collection chain becomes more complex
- Troubleshooting operations are inconvenient

#### 3. More in Line with Cloud-Native Platform Design
Kubernetes, Docker, and log collection components typically find it easier to integrate around standard output.

---

## XIV. What Happens If Logs Are Not Output to Standard Output

Potential issues may include:

- `docker logs` Unable to see critical business logs
- `kubectl logs` Unable to see application outputs
- Logs only exist in container internal files, requiring additional entry into the container for troubleshooting
- Logs are difficult to retain after container destruction
- Log collection system integration becomes complex

Therefore, when containerizing, it's advisable to avoid writing critical business logs only to container-local files.

---

## XV. Common Issues with Container Run Layers

### 1. Container Starts and Exits Immediately
Common causes:

- Main process finishes execution and exits
- Startup command error
- Program file does not exist
- Program startup error

### 2. Container is Running, but Business Is Unreachable
Common causes:

- Application is not actually listening on the target port
- Service only listens on loopback address
- Port mapping relationship is incorrect
- Program starts but has not fully initialized

### 3. Environment Variables Are Not Effective
Common causes:

- Variable name is written incorrectly
- Application reads variables incorrectly
- Variables are only set in the image but overridden at runtime
- Kubernetes injection logic is problematic

### 4. Logs Are Not Visible
Common causes:

- Logs are not output to stdout/stderr
- Application writes to container internal files
- Main process does not output logs normally
- Container exits immediately after startup, resulting in few logs

### 5. Container Repeatedly Restarts
Common causes:

- Startup anomalies
- Dependencies are unreachable
- Insufficient resources
- Probe failure (especially common in Kubernetes)

---

## XVI. How to Judge from an Operations Perspective Whether "Container Has Actually Run Normally"

Recommend judging from the following layers.

### 1. Does the Main Process Exist
If the main process does not exist, the container is typically already exited or about to exit.

### 2. Is the Container Continuously Running
Need to check the container status, not just whether it briefly started.

### 3. Is the Application Port Actually Listening
Sometimes the container process exists, but the application has not actually started listening on the port.

### 4. Do Logs Show the Application Is Ready
Many services log:

- startup complete
- listening on port ...
- connected to database
- ready to serve requests

### 5. Is the External Access Path Complete
For truly service-oriented applications, also need to verify:

- Docker port mapping is correct
- Kubernetes Service is correct
- Probe passes
- Upstream routing is normal

---

## XVII. What Level of Understanding Should Be Reached When Learning Container Runtime Basics

Currently, reaching the following level is sufficient:

### 1. Be able to explain that container startup essentially starts the main process
### 2. Understand why the main process exiting causes the container to exit
### 3. Distinguish between container internal ports and external access ports
### 4. Understand the role of environment variables in container runtime
### 5. Understand why logs should be output to stdout / stderr as much as possible
### 6. Be able to preliminarily determine which layer a container runtime issue belongs to

For example:

- Starts and exits immediately: likely a startup command or application layer issue
- Port is unreachable: likely a listening, mapping, or exposure chain issue
- No logs: likely a log output method issue
- Configuration not effective: likely an environment variable or configuration injection issue

---

## XVIII. Common Follow-Up Questions in Interviews

Common questions in this area include:

- Why does a container exit
- What is the relationship between the main process and the container lifecycle
- What does Docker's port mapping mean
- `EXPOSE` Is equal to the port being open
- What are the general uses of environment variables in containers
- Why are container logs usually recommended to output to stdout / stderr
- How to troubleshoot when a container starts successfully but business is unreachable
- Why modifying files in a container may not persist
- What is the relationship between a container and a process

---

## XIX. Stage Summary

The core of container runtime basics is not about memorizing many commands first, but establishing a runtime layer cognitive framework:

- An image is just a static template
- A container is the running instance of an image
- A container's continuous existence depends on the main process running continuously
- Whether a port is accessible requires distinguishing between listening port, mapped port, and exposure chain
- Environment variables are an important entry point for runtime configuration
- Logs are the primary troubleshooting material, and they should be output to standard output as much as possible

Only by clearly understanding this layer will subsequent learning about Kubernetes's Pod, Probe, Service, ConfigMap, log collection, and application troubleshooting not mistakenly attribute all issues to "Kubernetes configuration errors."

---

## XX. Keyword Quick Notes /think

- Main Process: The core running process after the container starts
- Container Exit: Typically directly related to the termination of the main process
- Exit Code: Represents the result of process execution
- Port Mapping: Maps external ports to container ports
- Environment Variables: Key-value parameters passed to the application at runtime
- Standard Output: stdout
- Standard Error: stderr
- Log Inspection: A critical entry point for container troubleshooting

---

## 21. Operational Deep Dive

From an operations perspective, many issues exposed in Kubernetes ultimately originate from the container runtime layer:

- Main process fails to start
- Application exits immediately after startup
- Application doesn't listen on ports
- Environment variables not passed in
- No log output
- Application hasn't fully initialized when upper layers start routing traffic

Therefore, mastering container runtime fundamentals isn't just about running images locally, but also about verifying whether the application is truly alive in the container before proceeding to Kubernetes orchestration, Service exposure, Probe health checks, and ConfigMap injection.

---

## References
- Docker Docs: https://docs.docker.com/
- Docker run reference: https://docs.docker.com/reference/cli/docker/container/run/
- Docker logs reference: https://docs.docker.com/reference/cli/docker/container/logs/
- Kubernetes Containers: https://kubernetes.io/docs/concepts/containers/
- Kubernetes Logging Architecture: https://kubernetes.io/docs/concepts/cluster-administration/logging/

---

## Next Day Suggestions
Next post suggestion to organize:

[[01-First Hands-on Example - Nginx Static Page Image Build and Container Operation]]