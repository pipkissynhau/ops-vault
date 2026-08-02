# 01-livenessProbe and readinessProbe Basic Practices

## Document Description
- Document Location: Introduction to Application Health Check Practices
- Applicable Phase: After completing the basic deployment of stateless applications, exposing Services, authenticating ConfigMap/Secrets with private image repositories, proceed to learn about Probe health checks.
- Recommended Path: `04-Kubernetes/07-Application Deployment/07-Application Health Checks and Availability Assurance/01-livenessProbe and readinessProbe Basic Practices`

## Tags
#Kubernetes #Probe #livenessProbe #readinessProbe #Health Check #Application Availability #Nginx #Stateless Application #Application Deployment #Business Containerization #Cloud Native #Ops #Interview Notes

---

## I. Why Start Learning about Probes Now

The previous steps have already established these foundational capabilities:

- Image building and deployment
- Creating Pods with Deployments
- Providing access through Services
- Exposing services via NodePorts
- Managing configurations with ConfigMap/Secrets
- Authenticating private image repositories using imagePullSecrets

At this point, the application is “ready to run.”  
However, in real production scenarios, merely ensuring that an application can start up is far from enough. We also need to address the following questions:

- Just because a container process is running doesn’t mean the application is actually usable. What should we do?
- Why can’t traffic be directed to an application immediately after it starts up if it isn’t fully ready?
- How can Kubernetes determine if an application is “dead,” stuck, or unresponsive?
- Why might some Pods remain in the Running state but still not receive any traffic through their Services?
- Why might certain containers keep restarting?

The core of these issues lies in:

> **Probes (health check mechanisms)**

Among them, the first two types to master are:

- `livenessProbe`
- `readinessProbe`

---

## II. What Problems Do Probes Solve?

Kubernetes is not satisfied with just confirming that “container processes have started.”  
In production environments, common scenarios include:

### 1. The process is still running, but the application has actually become stuck
Examples:
- The process hasn’t terminated.
- The port is still occupied.
- But the application’s interfaces are no longer functioning.

### 2. The application has just started up and isn’t ready yet
Examples:
- The process has started.
- But dependent services aren’t connected yet.
- Configuration files haven’t been loaded.
- Caches haven’t been initialized.
- It’s not appropriate to receive traffic at this time.

### 3. The application appears to be running, but it’s actually unavailable
Examples:
- Service threads have hung.
- Internal dependencies are malfunctioning.
- Only an empty shell process remains in the application.

Therefore, the primary purpose of Probes is to:

> **Enable Kubernetes to go beyond merely checking if processes exist and determine whether the application is truly alive and ready for use.**

---

## III. What Are livenessProbe and readinessProbe?

These are the most fundamental concepts to understand.

### 1. livenessProbe
In Chinese, it can be understood as:

> **Survival Probe**

Its main purpose is to determine:

> **Is the application inside this container still running?**

If the livenessProbe fails repeatedly, Kubernetes will likely assume that:

- The application has failed.
- Although the process is still running, it’s not functioning properly.
- This container should be restarted.

Therefore, its primary function is to:

> **Identify containers that need to be restarted.**

---

### 2. readinessProbe
In Chinese, it can be understood as:

> **Readiness Probe**

Its main purpose is to determine:

> **Is this Pod ready to receive traffic now?**

If the readinessProbe fails, Kubernetes will not immediately restart the container. Instead, it will assume that:

- The application is not yet ready.
- Traffic should not be directed to it for the time being.

Therefore, its primary function is to:

> **Control whether traffic can be forwarded to an application.**

---

## IV. What Is the Most Fundamental Difference Between the Two?

This is a common question in interviews and practical applications.

### livenessProbe
Its main focus is to determine:

> **Whether a container needs to be restarted**

If it fails, the typical consequence is that the kubelet will restart the container.

---

### readinessProbe
Its main focus is to determine:

> **Whether an application is ready to receive traffic**

If it fails, the typical consequence is that the Pod will not receive any traffic from the Service, but the container itself is usually not restarted directly as a result.

---

### Simplified Memory Aid
You can remember it this way:

- `livenessProbe`: Checks if an application is still alive.
- `readinessProbe`: Checks if an application is ready to receive traffic.

---

## V. Why Learn These Two First Instead of Starting with startupProbe?

### 5. `periodSeconds: 10`
This indicates that checks are performed every 10 seconds. According to Kubernetes official documentation, the default value for this field is **10 seconds**.

### 6. `failureThreshold: 3`
This means that if a Probe fails **three times in a row**, Kubernetes will consider the livenessProbe to have failed genuinely. The Kubernetes official documentation states that the default value for `failureThreshold` is **3**; for startupProbes or livenessProbes, if this number of failures is reached, Kubernetes will mark the container as unhealthy and trigger a restart.

### Key Points for Operations and Maintenance
A livenessProbe does not cause a restart after just **one failure**, but rather:

- Checks are conducted at regular intervals.
- Only after `failureThreshold` consecutive failures are thereafter will it trigger a determination that the container is unhealthy.
- Subsequently, kubelet will restart the container :contentReference[oaicite:2]{index=2}

---

## Section Eleven: How to Understand readinessProbe

Let's look at this section:

    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
      failureThreshold: 3

### 1. The Detection Method Is Still HTTP
It also accesses port 80 at `/`.

### 2. `initialDelaySeconds: 3`
This means that the container waits for **3 seconds** after starting before beginning readiness checks.

### 3. `periodSeconds: 5`
This indicates that checks are performed every 5 seconds.

### 4. `failureThreshold: 3`
If the readinessProbe fails **three times in a row**, Kubernetes will consider the Pod to be NotReady.

### Key Points for Operations and Maintenance
If a readinessProbe fails, common consequences include:

- The Pod will not be added to the Service's available backend list.
- External traffic usually will not be directed to it.
- The container is generally not restarted just because the readiness check failed.
- kubelet will continue running the container and perform subsequent readiness checks until it succeeds again :contentReference[oaicite:3]{index=3}

---

## Section Twelve: Why Do Both ReadinessProbe and livenessProbe Access `/`, But Their Meanings Are Completely Different?

This is a point that many people easily confuse when first learning about Probes.

You might see:

- The livenessProbe accesses `/`.
- The readinessProbe also accesses `/`.

And then mistakenly assume they are just performing duplicate checks.

But this is not the case.

### What the livenessProbe Checks:
> “Whether it’s time to restart the container.”

### What the readinessProbe Checks:
> “Whether the container is ready to receive traffic.”

Even though their detection methods and paths are the same, the consequences of their failures are entirely different.

### Key Points for Operations and Maintenance
Don’t just focus on “which interface is being accessed”; instead, consider:

> **What Kubernetes will do after a probe fails.**

---

## Section Thirteen: How to Understand failureThreshold

This is a very crucial parameter in Probes, but it’s also the one that beginners often overlook.

### Definition
`failureThreshold` indicates:

> **How many consecutive failures are required before a Probe is considered to have failed overall.**

Kubernetes official documentation explains this field as follows:

- Only after a Probe has failed `failureThreshold` times in a row will Kubernetes consider the entire check to have failed.
- The default value is **3**.
- The minimum value is **1**:contentReference[oaicite:4]{index=4}

### Implications for livenessProbe
In the case of a livenessProbe:

- If `failureThreshold` consecutive failures are reached, the container will be deemed unhealthy.
- kubelet will then restart the container.

### Implications for readinessProbe
For a readinessProbe:

- If `failureThreshold` consecutive failures occur, the Pod will be marked as NotReady.
- It will be removed from the Service’s available backend list.
- However, the container itself is generally not restarted directly due to this.

### Simple Memory Tip
You can remember it as follows:

- `failureThreshold`: The number of consecutive failures required before a Probe is considered “truly failed”.

---

## Section Fourteen: An Example of the Timeline for “Three Consecutive Failures”

Suppose the following configuration is in place:

    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3

### The timeline can be understood as follows:

#### At Second 0
The container starts up.

#### At Second 5
The first livenessProbe check is performed.  
If it fails, the current cumulative number of failures is 1.

#### At Second These default values are all derived from the current official Kubernetes documentation. :contentReference[oaicite:6]{index=6}

### Key Points for Operations and Maintenance Understanding
If you do not explicitly specify these parameters in your YAML file, it does not mean that the corresponding behavior will not occur; rather:

> **Kubernetes will use the default values.**

For example, the following configuration:

    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10

can be roughly interpreted as:

    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds: 1
      successThreshold: 1
      failureThreshold: 3

---

## Question 20: If the Application Is Already Accessible, Why Do We Still Need a ReadinessProbe?
This is a frequently asked question.

Without a readinessProbe, Kubernetes would struggle to determine:

- Whether the application is truly ready for use
- Whether it is still in the initialization phase
- Whether it is suitable to receive traffic at this moment

### Potential Issues Without a ReadinessProbe
As long as a Pod's processes start up, it may be assumed that it is ready to receive traffic. However, in reality, many applications require additional steps after their processes begin:

- Reading configuration files
- Connecting to databases
- Preheating caches
- Registering with services
- Initializing thread pools

If traffic is allowed before these steps are completed, the application may encounter errors.

### The Purpose of a ReadinessProbe
It serves to clearly distinguish between "the processes have started" and "the application is ready to receive traffic."

---

## Question 21: If the Application Is Already Accessible, Why Do We Still Need a LivenessProbe?
This question is also crucial.

After running for some time, applications may experience situations where:

- The processes are still running
- The ports are still occupied
- But the application logic has become stuck
- HTTP responses are no longer normal
- Or the thread pool has been exhausted
- Or dependent services are unavailable due to errors

Without a livenessProbe, Kubernetes might not be able to automatically detect such "false alive" states.

### The Purpose of a LivenessProbe
It allows Kubernetes to identify cases where:

> **Although the processes have not exited, they are no longer worth maintaining.**

Kubernetes can then attempt to restore the service by restarting it.

---

## Question 22: What Are the Most Common Issues in Implementing These Probes?
### 1. Incorrect Probe Path
For example, if Nginx is accessible at `/`, specifying a path like `/healthz` will result in probe failures.

### 2. Incorrect Probe Port
Even if a container is listening on port 80, specifying a different port in the probe configuration will cause issues.

### 3. `initialDelaySeconds` Too Short
If the probe starts too early before the application has fully started up, it may lead to incorrect judgments.

### 4. `timeoutSeconds` Too Short
If the application occasionally responds slowly, it may be incorrectly identified as a probe failure.

### 5. `failureThreshold` Too Low
Short-term fluctuations in performance may be quickly interpreted as failures, leading to frequent traffic interruptions or restarts.

### 6. Focusing Only on Pod Running Status and Ignoring Readiness Status
This is particularly common when using readinessProbes.

### 7. Confusion Between livenessProbe and readinessProbe
It is important to understand which one is responsible for restarting the service and which one determines if traffic can be allowed.

---

## Question 23: What Should Be Checked First When Troubleshooting Probe Issues?
It is recommended to follow this simple troubleshooting sequence:

### 1. Check the Pod Status
Focus on:

- Whether it is Running
- Whether it is Ready
- Whether there have been any increases in restart attempts

### 2. Review Event Logs
Many times, event logs will directly indicate probe failures.

### 3. Verify the Probe Configuration
Check the following settings:

- path
- port
- initialDelaySeconds
- periodSeconds
- timeoutSeconds
- failureThreshold

### 4. Ensure the Application Is Actually Accessible
For example:

- Whether the root page returns correctly
- Whether Nginx is indeed listening on port 80

### 5. Finally, Examine Logs
Review Nginx or application logs to determine if there are issues with the probe path itself.

---

## Question 24: Why Is This Article a Critical Preparatory Step Before Entering "Application Deployment Troubleshooting"?
Because after you formally start working on `12-Application Deployment Troubleshooting`, you will frequently encounter these problems:

- The Pod is Running,- Configure Liveness, Readiness, and Startup Probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

---

## Suggestions for the Next Day
Organize the following suggestions into an article:

[[02-startupProbe Basics and Use Cases for Slow Starting Applications]]