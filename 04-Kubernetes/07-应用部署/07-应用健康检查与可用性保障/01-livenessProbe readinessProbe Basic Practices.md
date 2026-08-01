# 01-livenessProbe and readinessProbe Fundamental Practice

## Document Overview
- Document Positioning: Application health check introductory practice
- Applicable Stage: After completing stateless application basic deployment, Service exposure, ConfigMap/Secret and private registry authentication, enter Probe health check fundamentals
- Recommended Path: `04-Kubernetes/07-Apply deployment/07-Application of health screening and availability safeguards/01-livenessProbe and readinessProbe Basic practice`

## Tags
#Kubernetes #Probe #livenessProbe #readinessProbe #HealthScreening #ApplyUsability #Nginx #NoStatusApplication #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## One, Why Start Learning Probe Now

The previous mainline has completed these foundational capabilities:

- Image building and pushing
- Deployment creates Pod
- Service provides access entry
- NodePort external exposure
- ConfigMap / Secret configuration management
- imagePullSecrets private registry authentication

At this point, the application has "started running".  
But in real production, achieving "being able to run" is far from sufficient, and we must continue to answer these questions:

- Container process being alive doesn't mean the application is really available, what to do
- Application just started but not ready, why can't it immediately receive traffic
- Application appears dead, stuck, or interface unresponsive, how does Kubernetes determine
- Why some Pods are Running but Service still has no traffic
- Why some containers are constantly restarted

The core of these questions will fall to:

> **Probe (Health Check Mechanism)**

The two most essential to master are:

- `livenessProbe`
- `readinessProbe`

---

## Two, What Problems Does Probe Solve

Kubernetes won't be satisfied with "container process started".  
In production, common scenarios include:

### 1. Process is still running, but the application is actually stuck
For example:
- Process hasn't exited
- Port is still occupied
- But interface is no longer working

### 2. Application just started but not ready
For example:
- Process is up
- But dependent services haven't connected
- Configuration hasn't loaded
- Cache hasn't preheated
- Shouldn't receive traffic temporarily

### 3. Application appears running but actually unavailable
For example:
- Service thread is hung
- Internal dependency anomalies
- Application is just an empty shell process

So Probe solves the core problem of:

> **Let Kubernetes not only check if "process is alive" but further determine if "application is alive" and "application is ready".**

---

## Three, What Are livenessProbe and readinessProbe

This is the most fundamental core understanding.

### 1. livenessProbe
In Chinese, it can be understood as:

> **Liveness Probe**

It mainly answers the question:

> **Is the application in this container still alive?**

If livenessProbe fails continuously, Kubernetes typically considers:

- Application is broken
- Although process is still running, it's abnormal
- This container should be restarted

So its core function is:

> **Detecting containers that need to be restarted.**

---

### 2. readinessProbe
In Chinese, it can be understood as:

> **Readiness Probe**

It mainly answers the question:

> **Is this Pod ready to receive traffic now?**

If readinessProbe fails, Kubernetes typically won't restart the container immediately, but considers:

- Application isn't ready yet
- Can't forward traffic to it temporarily

So its core function is:

> **Controlling whether "traffic should be received".**

---

## Four, What's the Most Essential Difference Between Them

This is the most commonly asked point in interviews and practical scenarios.

### livenessProbe
Focuses on judging:

> **Whether to restart**

After failure, common consequences are:

- Container is restarted by kubelet

---

### readinessProbe
Focuses on judging:

> **Whether to receive traffic**

After failure, common consequences are:

- Pod won't receive traffic from Service backend
- Container typically won't be restarted directly for this

---

### Simplified Memory
You can first remember as:

- `livenessProbe`: Determine whether it should still be alive
- `readinessProbe`: Determine whether it can currently receive traffic

---

## Five, Why Learn These Two First Instead of startupProbe

Because these are the most fundamental, most commonly used, and easiest to encounter in practical scenarios.

### livenessProbe
Solves "whether to restart when application is broken"

### readinessProbe
Solves "don't receive traffic when application isn't ready"

While `startupProbe` is more introduced when facing:

- Slow-starting Java applications
- Slow-starting middleware
- Avoiding livenessProbe prematurely killing the application

So now focus first on:

- Survival
- Readiness

These two concepts are most important.

---

## Six, What Are the Common Detection Methods for Probe

Kubernetes supports three common detection methods for Probe:

### 1. HTTP Detection
For example, accessing:

- `/health`
- `/ready`
- `/live`

As long as it returns a HTTP status code that meets expectations, it's typically considered successful.

Suitable for:
- Web services
- HTTP API services
- Nginx
- Java / Python / Go web applications

---

### 2. TCP Detection
Judging whether a certain port can be connected.

Suitable for:
- Services without a dedicated health interface
- But port listening status has reference value

---

### 3. Exec Detection
Executing a command inside the container, judging success based on the command's return code.

Suitable for:
- Some special scenarios
- Services without HTTP interface
- Not suitable for simple TCP port checks

---

## Seven, Why Nginx is Suitable for the First Probe Practice

Because Nginx has several excellent characteristics:

- Simple deployment
- Intuitive HTTP interface
- Fast container startup
- Can directly use HTTP detection `/`
- Success and failure phenomena are easy to observe

Moreover, as a stateless application, Nginx is very suitable for helping you first establish:

- `livenessProbe`
- `readinessProbe`

Fundamental understanding.

Therefore, this article doesn't pursue complexity, focusing only on:

> **Let you truly understand the position and role of Probe in Deployment.**

---

## Eight, A Simplest livenessProbe + readinessProbe Example

Below is a simplest Nginx Deployment example: /think

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-web
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
        - name: nginx-web
          image: harbor.example.com/project/nginx-web:v1
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
            failureThreshold: 3

---

## Nine. Overview of What This YAML Does

This Deployment does the following:

### 1. Creates an Nginx Pod
The image runs your Nginx web service.

### 2. Container Listens on Port 80
Nginx provides HTTP services externally.

### 3. Configures livenessProbe
Kubernetes periodically accesses:

- Path: `/`
- Port: `80`

To determine if the container is still alive.

### 4. Configures readinessProbe
Kubernetes also periodically accesses:

- Path: `/`
- Port: `80`

To determine if the Pod is ready to receive traffic.

### 5. Explicitly specifies `failureThreshold: 3`
This indicates that the probe isn't considered failed after just 1 failure, but rather:

- **Fails 3 times consecutively**
- Before Kubernetes considers this Probe truly failed

---

## Ten. Understanding the livenessProbe Section

First, look at this part:

    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3

### 1. `httpGet`
Indicates HTTP request is used for health checks.

### 2. `path: /`
Indicates accessing the root path `/`.

### 3. `port: 80`
Indicates accessing port 80 inside the container.

### 4. `initialDelaySeconds: 5`
Indicates waiting 5 seconds after container startup before starting probes.  
This gives the application some startup buffer time. Kubernetes official documentation states the default value for this field is **0 seconds**.

### 5. `periodSeconds: 10`
Indicates checking every 10 seconds. Kubernetes official documentation states the default value for this field is **10 seconds**.

### 6. `failureThreshold: 3`
Indicates that Kubernetes only considers this Probe truly failed after **3 consecutive failures**. Kubernetes official documentation states that `failureThreshold` default value is **3**; for startupProbe or livenessProbe, if this failure count is reached, Kubernetes will mark the container as unhealthy and trigger a restart.

### Operations Understanding Focus
livenessProbe isn't "restart on first failure", but rather:

- Checks periodically
- Fails **consecutively `failureThreshold` times**
- Triggers "container unhealthy" judgment
- Then kubelet restarts the container :contentReference[oaicite:2]{index=2}

---

## Eleven. Understanding the readinessProbe Section

Now look at this part:

    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
      failureThreshold: 3

### 1. Detection method is still HTTP
Also accesses port 80 at `/`.

### 2. `initialDelaySeconds: 3`
Indicates waiting 3 seconds after container startup before starting readiness checks.

### 3. `periodSeconds: 5`
Indicates checking every 5 seconds.

### 4. `failureThreshold: 3`
Indicates that if readinessProbe **fails 3 times consecutively**, Kubernetes will mark this Pod as NotReady.

### Operations Understanding Focus
If readinessProbe fails, common results are:

- Pod won't join Service backend available list
- External traffic typically won't reach it
- Container usually won't be restarted just because of readiness failure
- kubelet will continue running this container and keep doing readinessProbe checks until it recovers successfully :contentReference[oaicite:3]{index=3}

---

## Twelve. Why Both Can Access `/`, But Their Meanings Are Completely Different

This is a common point of confusion for beginners learning Probes.

You might see:

- livenessProbe accesses `/`
- readinessProbe also accesses `/`

And mistakenly think they're just redundant checks.

Actually, it's not.

### livenessProbe Checks:
> "Should this container be restarted?"

### readinessProbe Checks:
> "Are you ready to receive traffic?" /think

Even if the detection method and path are the same, the consequences they trigger can be completely different.

### Operations Understanding Focus
Don't just focus on "which interface is being accessed", but rather pay more attention to:

> **What Kubernetes does after a probe fails.**

---

## Thirteen, How to Understand failureThreshold Correctly

This is a very critical parameter in Probe settings, but it's often overlooked by beginners.

### Definition
`failureThreshold` indicates:

> **How many consecutive failures are needed for Kubernetes to consider this Probe as a whole failure.**

The Kubernetes official documentation explains this field as:

- Kubernetes considers the Probe as a whole failure after `failureThreshold` consecutive failures
- The default value is **3**
- The minimum value is **1** :contentReference[oaicite:4]{index=4}

### Significance for livenessProbe
If it's a livenessProbe:

- After reaching `failureThreshold` consecutive failures
- The container is considered unhealthy
- kubelet will restart this container

### Significance for readinessProbe
If it's a readinessProbe:

- After reaching `failureThreshold` consecutive failures
- The Pod is marked as NotReady
- It will be removed from the Service's available backend
- But the container is generally not restarted directly due to this

### Simple Memory Tip
You can first remember it as:

- `failureThreshold`: How many consecutive failures count as "true failure"

---

## Fourteen, A Timeline Example of "3 Consecutive Failures"

Assume the following configuration:

    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3

### Timeline Understanding

#### 0 Seconds
Container starts.

#### 5 Seconds
First livenessProbe check begins.  
If it fails, the current failure count is 1.

#### 15 Seconds
Second check.  
If it fails again, the current failure count is 2.

#### 25 Seconds
Third check.  
If it fails again, the current failure count is 3.

#### After 25 Seconds
Because the consecutive failures have reached `failureThreshold: 3`, Kubernetes will mark this container as unhealthy and trigger a restart.

### Operations Understanding Focus
This is why many people mistakenly believe:

- `periodSeconds: 10` means "restart after 10 seconds"

Actually, this is incorrect.

The actual time of failure needs to be judged comprehensively based on:

- `initialDelaySeconds`
- `periodSeconds`
- `failureThreshold`
- `timeoutSeconds`

---

## Fifteen, Why a Pod in Running State May Still Not Accept Traffic

This is one of the most important practical values of readinessProbe.

Many people new to Kubernetes often think:

- Pod is Running
- It should be accessible

Actually, it's not necessarily true.

If readinessProbe fails, the following may occur:

- Pod status appears to be running
- Container process is active
- But the Service backend will not consider it as a "ready replica"

Common phenomena at this time include:

- Pod is Running
- But the business cannot access it
- Or it's not in Endpoints
- Or the Service backend count is incorrect

### Operations Understanding Focus
This is also a highly frequent source of issues when entering `09-Apply deployment barriers` later on.

---

## Sixteen, Why livenessProbe Should Not Be Configured Arbitrarily

If livenessProbe is configured improperly, it may lead to a very problematic situation:

> **Restarting applications that could still recover or are just temporarily slow.**

For example:

- The application itself starts slowly
- The probe starts too early
- The check frequency is too high
- Timeout is too short
- Path is selected incorrectly
- `failureThreshold` is too small

The result may be:

- Container just starts
- Not yet ready
- livenessProbe considers it "dead"
- Then restarts continuously

### Operations Understanding Focus
livenessProbe configuration needs to be more cautious, as it's a probe with restart actions.  
The Kubernetes official documentation also clearly warns that poorly implemented livenessProbe can lead to cascading failures, increased container restarts, and increased pressure on remaining Pods.:contentReference[oaicite:5]{index=5}

---

## Seventeen, Why readinessProbe Should Not Be Configured Arbitrarily Either

Although readinessProbe doesn't directly cause container restarts, improper configuration can still cause serious issues.

For example:

- Probe path is written incorrectly
- The application is actually normal, but it's always judged as NotReady
- `failureThreshold` is too small, causing brief fluctuations to be quickly removed from traffic
- Eventually, the Pod can't enter the Service backend
- The business manifestation is "Pod is present, but service is unreachable"

### Operations Understanding Focus
Therefore, the common issue with readinessProbe is not "killing the container", but:

> **Blocking Pods that could serve from the traffic entry point.**

---

## Eighteen, Which Parameters Are Most Critical to Remember Now

At this stage, it's recommended to focus on the most core parameters first:

### 1. `httpGet`
Indicates using HTTP request for probing.

### 2. `path`
Indicates the path to be probed.

### 3. `port`
Indicates the container port to be probed.

### 4. `initialDelaySeconds`
Indicates how long to wait after container startup before starting probing.

### 5. `periodSeconds`
Indicates how often to probe.

### 6. `failureThreshold`
Indicates how many consecutive failures are needed for this Probe to be considered truly failed.

### 7. `timeoutSeconds`
Indicates the maximum time to wait for each probe, and timeout means the probe fails.

### 8. `successThreshold`
Indicates how many consecutive successful probes are needed after probe failure to restore to a successful state.

---

## Nineteen, Default Values for Common Probe Parameters

The Kubernetes official documentation provides the following default values for common Probe fields:

### 1. `initialDelaySeconds`
- Default value: `0`
- Meaning: After container startup, it will not wait additionally by default, and probing can start immediately when conditions are met

### 2. `periodSeconds`
- Default value: `10`
- Meaning: Probe is executed every 10 seconds by default

### 3. `timeoutSeconds`
- Default value: `1`
- Meaning: Single Probe defaults to waiting 1 second, and timeout is considered as a failure /think

### 4. `successThreshold`
- Default value: `1`
- Meaning: After a probe failure, it defaults to requiring 1 consecutive success to restore a successful state
- Special note: For **livenessProbe** and **startupProbe**, this value must be `1`

### 5. `failureThreshold`
- Default value: `3`
- Meaning: Defaults to 3 consecutive failures before the probe is considered truly failed

These default values come from the current Kubernetes official documentation. :contentReference[oaicite:6]{index=6}

### Operations Understanding Key Points
If you don't explicitly write these parameters in your YAML, it doesn't mean there's no behavior - instead:

> **Kubernetes will work according to the default values.**

So the following snippet:

    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10

Can be approximately understood as:

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

## Twenty, Why is readinessProbe needed even if the application is accessible

This question is often asked.

Without readinessProbe, Kubernetes has difficulty knowing:

- Whether the application is truly ready
- Whether it's still in the initialization phase
- Whether it's suitable to accept traffic at this moment

### Potential issues without readinessProbe
A Pod is considered ready to accept traffic as long as the process is running.  
But in reality, many applications need to do more after the process starts:

- Read configuration
- Connect to the database
- Preheat cache
- Register services
- Initialize thread pools

If traffic is accepted before these stages are complete, business errors may occur.

### The significance of readinessProbe
It clearly separates the two events of "process started" and "ready to accept traffic".

---

## Twenty-one, Why is livenessProbe needed even if the application is accessible

This is also a critical question.

Many applications may encounter the following situations after running for some time:

- The process is still running
- The port is still occupied
- But the application logic has stalled
- HTTP interfaces no longer respond normally
- Or the thread pool is exhausted
- Or dependencies cause service unavailability

Without livenessProbe, Kubernetes may not be able to automatically detect this "pseudo-alive" state.

### The significance of livenessProbe
It allows Kubernetes to have the opportunity to discover:

> **Although the process hasn't exited, it's no longer worth keeping.**

Then through restarts to attempt to recover the service.

---

## Twenty-two, What are the most common issues encountered in practice

### 1. Probe path is written incorrectly
For example, Nginx can access `/`, but you wrote `/healthz`, resulting in probe failure.

### 2. Probe port is written incorrectly
The container is clearly listening on 80, but the Probe is written to a different port.

### 3. `initialDelaySeconds` is too short
The probe starts before the application has fully started, leading to false judgments.

### 4. `timeoutSeconds` is too short
The application itself is fine, but occasional slow responses are considered probe failures.

### 5. `failureThreshold` is too small
Brief fluctuations are quickly judged as failures, leading to frequent traffic removal or restarts.

### 6. Ignore Ready status just because the Pod is Running
This is especially common in readinessProbe scenarios.

### 7. Confusion between livenessProbe and readinessProbe semantics
Not knowing which is responsible for restarts and which is responsible for traffic acceptance.

---

## Twenty-three, What to check first when troubleshooting Probe issues

It's recommended to form a simple troubleshooting sequence now.

### 1. Check Pod status first
Focus on:

- Whether it's Running
- Whether it's Ready
- Whether restart counts have increased

### 2. Check event information next
Events often directly indicate probe failures.

### 3. Check Probe configuration itself
Verify:

- path
- port
- initialDelaySeconds
- periodSeconds
- timeoutSeconds
- failureThreshold

### 4. Check if the application itself is truly accessible
For example:

- Whether the root path returns normally
- Whether Nginx is actually listening on port 80

### 5. Finally check logs
Combine with Nginx or application logs to determine if the probe path itself has issues.

---

## Twenty-four, Why this is a key prerequisite before entering "application deployment troubleshooting"

Because after you officially enter `12-Apply deployment barriers`, you will inevitably encounter these issues frequently:

- Pod Running but Service has no traffic
- Pod is always NotReady
- Container keeps restarting
- The image is fine, but the business just won't start
- Deployment update is stuck

Among these issues, Probes will be one of the high-frequency core factors.

Therefore, this article isn't just about "learning a new field", but laying the foundation for the troubleshooting mainline later.

---

## Twenty-five, The most important cognitions in this topic

### 1. Probe is not as simple as checking if the process is running
It's the mechanism Kubernetes uses to judge application health and availability.

### 2. livenessProbe mainly determines "whether to restart"
This is its most important semantic.

### 3. readinessProbe mainly determines "whether to accept traffic"
This is its most important semantic.

### 4. Pod Running doesn't equal Ready
This is a highly frequent troubleshooting prerequisite cognition.

### 5. Unreasonable Probe configuration may lead to both false restarts and traffic blockage
So you can't just write it, you also need to judge the consequences.

### 6. `failureThreshold` determines "how many consecutive failures count as true failure"
It's not a single failure that triggers the final action.

### 7. Not writing parameters doesn't mean there's no behavior
Many Probe fields have official default values, which must be known.

---

## Twenty-six, What level should you master when learning this article

At this stage, it's recommended to reach the following level:

### 1. Be able to explain the difference between livenessProbe and readinessProbe
### 2. Be able to understand a basic HTTP Probe YAML
### 3. Be able to understand why a Pod Running may still not accept traffic
### 4. Be able to understand what problems unreasonable Probe configuration may cause
### 5. Be able to understand the actual role of `failureThreshold`
### 6. Be able to name several common Probe default values
### 7. Be able to make preliminary judgments on basic Probe issues

--- /think

## 27. Common Follow-up Questions in Interviews

This section includes common questions:

- What's the difference between livenessProbe and readinessProbe
- Why might a Pod be in Running state but the application not be available
- Why does readinessProbe affect Service traffic
- What happens when livenessProbe fails
- What happens when readinessProbe fails
- What are the common types of Probe
- What phenomenon occurs if the probe path is incorrect
- What does `failureThreshold` mean
- How should `periodSeconds` and `failureThreshold` be understood together
- Why do some containers restart repeatedly
- Why is Probe needed even if the application is already accessible
- What is the default value when not specifying `failureThreshold`

---

## 28. Stage Summary

livenessProbe and readinessProbe are the most fundamental and important two probes in Kubernetes application health check system.

The most important thing in this section is not memorizing all parameters, but first establishing these core understandings:

- Just because the process is running doesn't mean the application is truly available
- Even if the application is available, it doesn't mean it's ready to receive traffic
- livenessProbe determines whether to restart
- readinessProbe determines whether to receive traffic
- Pod Running and Pod Ready are two different things
- `failureThreshold` determines how many consecutive failures count as overall failure
- Probe configuration directly affects service availability and subsequent troubleshooting judgment
- Official default values affect actual behavior and should not be ignored

As long as these understandings are established, the thinking process will be much clearer when continuing to learn `startupProbe`, resource control, Service/Endpoints troubleshooting, and formally entering `12-Apply deployment barriers`.

---

## 29. Keyword Mnemonics

- Probe: Kubernetes health check mechanism
- livenessProbe: Liveness probe, determines whether to restart the container
- readinessProbe: Readiness probe, determines whether to receive traffic
- HTTP Probe: Checks application status through HTTP requests
- initialDelaySeconds: When to start probing after startup
- periodSeconds: How often to probe
- timeoutSeconds: Maximum waiting time for a single probe
- failureThreshold: How many consecutive failures count as overall failure
- successThreshold: How many consecutive successes restore success
- Running: Container process is running
- Ready: Pod is ready to receive traffic

---

## 30. Operations Perspective Understanding

From an operations perspective, Probe marks the turning point where Kubernetes transitions from "being able to start Pods" to "being able to manage application status."

Without Probe, the platform can only roughly determine:

- Whether the container is running
- Whether the process is still alive

With Probe, the platform gains these capabilities:

- Detecting if the application is pseudo-dead
- Controlling when the application receives traffic
- Avoiding premature exposure of unready replicas
- Attempting automatic recovery in abnormal states

This is why many so-called "business anomalies" eventually trace back to Probe, Service, Endpoints, and the application's own status.

Therefore, although this section is about basic health checks, it's actually a key step from "being able to deploy" to "being able to judge application availability."

---

## References
- Kubernetes Probes: https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/
- Configure Liveness, Readiness and Startup Probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

---

## Next Day Suggestions
Next post suggestion: organize

[[Startup Probe Basics and Slow Startup Application Scenarios]]