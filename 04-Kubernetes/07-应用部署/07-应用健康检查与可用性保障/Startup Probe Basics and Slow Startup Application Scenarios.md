# 02-startupProbe Basics and Slow Startup Scenarios

## Document Notes
- Document Focus: Completing the Probe system and understanding slow startup scenarios
- Applicable Stage: After completing livenessProbe and readinessProbe basics, move to understanding startupProbe and slow startup scenarios
- Recommended Path: `04-Kubernetes/07-Apply deployment/07-Application of health screening and availability safeguards/02-startupProbe Base and start slow application scene`

## Tags
#Kubernetes #Probe #startupProbe #livenessProbe #readinessProbe #HealthScreening #SlowStartApplication #ApplyUsability #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## One: Why Learn startupProbe After livenessProbe and readinessProbe

The previous document established two critical understandings:

- `livenessProbe`: Mainly determines whether to restart
- `readinessProbe`: Mainly determines whether to accept traffic

These two Probes are sufficient for many basic scenarios.  
However, in real production environments, another very typical issue may occur:

> **The application itself takes a long time to start.**

Examples include:

- Java service startup is slow
- Spring Boot initialization takes a long time
- Middleware takes time to load configuration and dependencies
- Large applications need cache preheating during startup
- Applications need to wait for database, registry center, or external dependencies to be ready
- Some services take tens of seconds or even minutes for cold start

If only configured with:

- livenessProbe
- readinessProbe

A very problematic situation may occur:

- The application is still normally starting
- But livenessProbe starts detecting early
- The application hasn't been ready yet, leading to repeated failures
- kubelet determines the container "unhealthy"
- Directly restarts the container
- The application hasn't finished starting yet, gets killed repeatedly

At this point, you need:

> **startupProbe**

---

## Two: What Problem Does startupProbe Solve

It solves:

> **How to avoid being prematurely judged as dead by livenessProbe while the application is still in the normal startup process.**

In other words, startupProbe isn't meant to replace livenessProbe or readinessProbe, but to provide a startup protection phase for "slow-starting applications."

### Simplified Understanding
If we say:

- `readinessProbe` solves "Don't accept traffic yet"
- `livenessProbe` solves "Should we restart?"

Then:

- `startupProbe` solves "Don't rush to judge as dead, I'm still starting"

---

## Three: When Is startupProbe Especially Needed

The following scenarios typically benefit from startupProbe:

### 1. Java / Spring Boot Services
Long startup process, especially noticeable when dependencies are numerous.

### 2. Applications Need to Initialize Large Resources
Examples include:

- Cache preheating
- Loading large configurations
- Building connection pools
- Initializing thread pools

### 3. Startup Depends on External Components
Examples include:

- Waiting for database
- Waiting for Redis
- Waiting for Nacos
- Waiting for configuration center
- Waiting for registry center

### 4. Middleware-Type Services
Examples include certain:

- Nacos
- Elasticsearch
- Kafka-related components
- Custom complex services

### 5. Business That Enters Stable State Gradually After Startup
Examples where cold start takes significantly longer than subsequent check intervals.

---

## Four: What Problems Occur Without startupProbe

The most typical issue is:

> **Slow startup, but gets killed by livenessProbe.**

For example, a Java service normally takes 60 seconds to start.  
If your livenessProbe is configured as:

- Starts checking 10 seconds later
- Checks every 5 seconds

The result in the first few dozen seconds is that the application hasn't fully initialized its interfaces.  
So livenessProbe will think:

- Probe failure
- Application unhealthy
- Should restart

This creates a loop:

- Container starts
- Application hasn't completed initialization
- Probe fails
- Container restarts
- Restart again
- Fail again

The final manifestation is:

- Pod keeps restarting
- Business never starts
- It looks like the application has issues, but actually it's Probe configuration and startup characteristics mismatch

---

## Five: What Is the Core Semantics of startupProbe

Remember this key sentence first:

> **startupProbe is used to determine whether the application has completed startup.**

Before startupProbe succeeds, Kubernetes considers the application still in "startup phase."

### Its Most Core Meaning Is
During the startup phase:

- Temporarily don't use livenessProbe to kill it
- Give it a reasonable startup buffer time

### Simplified Understanding
- `livenessProbe`: Runtime health check
- `readinessProbe`: Runtime traffic access check
- `startupProbe`: Startup phase protection check

---

## Six: What Is the Relationship Between startupProbe and livenessProbe, readinessProbe

This is one of the most confusing yet important points.

### startupProbe Is Not a Replacement for livenessProbe
It's not that you don't need livenessProbe, but:

> **startupProbe takes charge before the application completes startup.**

### startupProbe Is Also Not a Replacement for readinessProbe
Even if the application completes startup, it doesn't necessarily mean it should immediately accept traffic.  
So readinessProbe still has its own role.

### A More Reasonable Overall Understanding Is
- startupProbe: First determine "Are you still in normal startup?"
- readinessProbe: Then determine "Can you accept business traffic?"
- livenessProbe: Finally continuously determine "Should you be restarted?"

---

## Seven: You Can Chain the Three Probes into a Complete Timeline

To better understand, you can remember it this way:

### First Stage: Startup Phase
Handled by `startupProbe`.

Question:

> Have you successfully completed startup?

---

### Second Stage: Readiness Phase
Handled by `readinessProbe`.

Question:

> Can you now accept business traffic?

---

### Third Stage: Runtime Phase
Handled by `livenessProbe`.

Question:

> Are you now unhealthy enough to need restart?

---

## Eight: A Simple startupProbe Example

Below is a basic Deployment example: /think

apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: demo-app
          image: harbor.example.com/project/demo-app:v1
          ports:
            - containerPort: 8080
          startupProbe:
            httpGet:
              path: /health
              port: 8080
            periodSeconds: 5
            failureThreshold: 12
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10

---

## IX. Understanding the YAML as a Whole

This Deployment configures three probe types:

### 1. startupProbe
During application startup, it continuously checks:

- `/health`
- `8080`

As long as this probe hasn't succeeded, it indicates the application hasn't completed startup.

### 2. readinessProbe
After startup, it determines whether this Pod is ready to accept traffic.

### 3. livenessProbe
After running for some time, it continuously checks whether the application has failed to the point of needing restart.

### Operations Understanding Focus
This indicates that Probes are not mutually exclusive, but rather:

> **Perform different responsibilities in different stages.**

---

## X. How to Understand the startupProbe Section

First look at this section:

    startupProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 5
      failureThreshold: 12

### 1. `httpGet`
Indicates using HTTP to check application startup status.

### 2. `path: /health`
Indicates accessing the health check interface `/health`.

### 3. `port: 8080`
Indicates accessing port 8080 inside the container.

### 4. `periodSeconds: 5`
Indicates checking every 5 seconds.

### 5. `failureThreshold: 12`
Indicates allowing 12 consecutive failures.

### What Does This Mean
The total allowable startup probe window is approximately:

- `5 sec × 12 Minor = 60 sec`

This means:

> **The application has approximately a 60-second "startup protection period".**

If it successfully passes the startupProbe within this time, it indicates the application has completed startup; if it fails continuously beyond the threshold, the container will be considered failed to start.

---

## XI. Why startupProbe is Often Viewed Together with failureThreshold

Because the most critical aspect of startupProbe is not just "checking once," but:

> **How long to tolerate startup time.**

This duration is typically determined by these two parameters together:

- `periodSeconds`
- `failureThreshold`

### A Simple Formula
Tolerable startup time ≈ `periodSeconds × failureThreshold`

For example:

- `periodSeconds: 5`
- `failureThreshold: 12`

Allows approximately a 60-second startup window.

### Operations Understanding Focus
This is why startupProbe is particularly suitable for slow-starting applications, as it explicitly tells Kubernetes:

> **Don't jump to conclusions too early; I might need this long to truly come online.**

---

## XII. What Happens After startupProbe Succeeds

Once startupProbe succeeds, it typically indicates:

> **The application has completed the startup phase.**

After that, Kubernetes will proceed with normal logic to enter:

- readinessProbe
- livenessProbe

In other words, startupProbe's role is more like a "startup gate."

### Operations Understanding Focus
It is not a long-term main probe, but rather more of:

> **A pre-protection mechanism for the startup phase.**

---

## XIII. What Happens If startupProbe Fails Continuously

If startupProbe fails continuously until exceeding the allowed threshold, Kubernetes typically considers:

- The application failed to start
- The startup phase has failed
- This container should be restarted

### Possible Outcomes
- Pod keeps restarting
- Container restart count increases
- Business fails to start

### This may appear similar to livenessProbe failure
But the semantics are different:

- livenessProbe failure: Application failed during runtime
- startupProbe failure: Application never started during startup

---

## XIV. Why startupProbe is Particularly Suitable for Slow-Starting Applications

Because it explicitly tells Kubernetes a very realistic fact:

> **Not all applications should be judged as "alive" within a very short time.**

Slow-starting applications typically have these characteristics:

- Process starts quickly
- But service becomes available slowly
- Requires extensive initialization in between

Without startupProbe, Kubernetes easily misjudges "slow" as "dead."

Thus, the core value of startupProbe is to distinguish between:

- **Slow startup**
- and
- **Startup failure**

---

## XV. When startupProbe Is Not Always Necessary

Not all applications must be configured with startupProbe.

### Scenarios Where It's Usually Not Needed
- Application starts up extremely quickly
- Startup logic is simple
- No complex dependencies
- Stable service provision within seconds
- livenessProbe and readinessProbe are already sufficiently reasonable

Example:

- Simple Nginx page
- Very lightweight static service
- Simple API with extremely short startup time

### Operational Understanding Focus
startupProbe is for scenarios where "startup phase is complex or slow," not necessarily the more configurations you set.

---

## Sixteen, Common Configuration Errors to Avoid

### 1. Slow startup without startupProbe
Causes livenessProbe to kill prematurely.

### 2. Too short startupProbe window
Application needs 90 seconds to start, but you only give 30 seconds window.

### 3. Health check path written incorrectly
Resulting in startup probe never succeeding.

### 4. Treating startupProbe as long-running probe
Ignoring its stronger "startup phase protection" semantics.

### 5. Confusing semantics between readinessProbe and startupProbe
Not knowing which is "startup protection" and which is "traffic admission."

---

## Seventeen, How to Clearly Explain the Difference Between startupProbe and readinessProbe

This is a common interview question.

### startupProbe
Focuses on:

> **Whether the application has completed the startup process.**

If it fails for too long, it indicates the startup itself failed.

### readinessProbe
Focuses on:

> **Whether the application is currently suitable to receive traffic.**

Even if the application has completed startup, it might temporarily not be ready to receive traffic.

### Simplified Understanding
- startupProbe: Can the application complete startup
- readinessProbe: Can the application receive traffic after startup

---

## Eighteen, How to Clearly Explain the Difference Between startupProbe and livenessProbe

### startupProbe
Mainly solves:

> **Protecting against premature termination during startup.**

### livenessProbe
Mainly solves:

> **Restarting when the application fails during runtime.**

### Simplified Understanding
- startupProbe: Startup protection
- livenessProbe: Runtime survival judgment

---

## Nineteen, What to Check First When Troubleshooting startupProbe Issues

Recommend forming a simple troubleshooting sequence.

### 1. First check if the Pod restarts repeatedly
If it restarts many times, consider whether it failed during startup.

### 2. Then check events
Events often contain Probe failure information.

### 3. Then check startupProbe configuration
Focus on verifying:

- `path`
- `port`
- `periodSeconds`
- `failureThreshold`

### 4. Then check application logs
Confirm whether the application is:

- Truly failed to start
- Or just slow to start

### 5. Finally judge if the window is sufficient
If logs show the application completes startup at the 70th second, but you only gave a 60-second window, the issue is clear.

---

## Twenty, Why This is an Important Step Before Entering Troubleshooting

When entering `12-Apply deployment barriers` later, you will frequently encounter these phenomena:

- Container keeps restarting
- Application seems fine but can't start
- Java service starts very slowly
- Pod keeps failing during initialization
- livenessProbe looks normal, but the application can't survive

Without understanding startupProbe, these issues are easily misjudged as:

- Image problem
- Application code problem
- Service problem

In reality, it's often:

> **Health check strategy and application startup characteristics mismatch.**

---

## Twenty-one, The Most Important Understandings in This Topic

### 1. startupProbe mainly serves slow-starting applications
Not all applications need it, but slow-starting applications really need it.

### 2. Its core value is to prevent livenessProbe from killing prematurely
This is the most important semantic.

### 3. startupProbe, readinessProbe, and livenessProbe act in different phases
This is the key to understanding the Probe system.

### 4. `periodSeconds × failureThreshold` determines the approximate startup tolerance window
This is a very practical judgment method.

### 5. Application can't start, it's not necessarily the application itself failed, it could be the startup window configuration is unreasonable
This is very important for subsequent troubleshooting.

---

## Twenty-two, What Level Should You Reach When Learning This

At this stage, it's recommended to reach the following level:

### 1. Be able to explain what startupProbe does
### 2. Understand the differences between startupProbe, livenessProbe, and readinessProbe
### 3. Be able to read a basic startupProbe YAML
### 4. Understand why slow-starting applications are prone to premature termination
### 5. Be able to make an initial judgment on startupProbe-related issues

---

## Twenty-three, Common Follow-up Questions in Interviews

Common questions in this area include:

- What is startupProbe for
- Why do we still need startupProbe if we have livenessProbe and readinessProbe
- What's the difference between startupProbe and livenessProbe
- What's the difference between startupProbe and readinessProbe
- What scenarios are particularly suitable for startupProbe
- Why does an application restart repeatedly when it starts slowly
- How to roughly estimate the startup window for startupProbe
- Why is startupProbe often considered for Java services that start slowly

---

## Twenty-four, Stage Summary

startupProbe is a probe in the Kubernetes Probe system specifically for "startup phase protection."

The most important part of this article isn't memorizing several fields, but first establishing these core understandings:

- Slow startup doesn't mean the application is broken
- startupProbe's core role is to provide a protection window for slow-starting applications
- It can prevent livenessProbe from killing the application prematurely
- readinessProbe manages traffic admission, livenessProbe manages runtime survival, startupProbe manages startup phase protection
- An unreasonable startup window configuration itself can become a common deployment fault

As long as you clearly understand these relationships, many phenomena will be clearer when continuing to learn resource management, Service/Endpoints, PVC, and formally entering `12-Apply deployment barriers` later.

---

## Twenty-five, Keyword Quick Notes

- startupProbe: Startup probe, determines if the application has completed startup
- Startup protection: Prevents the application from being prematurely killed during startup
- Slow-starting application: Service with long startup time
- failureThreshold: Allowed number of consecutive failures
- periodSeconds: Probe interval
- Startup window: Approximate time range the application is allowed to complete startup
- livenessProbe: Runtime survival judgment
- readinessProbe: Traffic admission judgment

---

## Twenty-six, Operational Extended Understanding

From an operations perspective, the value of startupProbe isn't just "adding another Probe," but making Kubernetes' understanding of application lifecycle more aligned with reality.

Without startupProbe, platforms often make overly coarse judgments about applications:

- Slow startup ≈ Unable to start
- No timely response ≈ Already dead

With startupProbe, the platform can finally distinguish between:

- Still starting up
- Already running
- Ready to accept traffic
- Needs restart

This is particularly important for applications with slow startup times.  
Many production environment "applications failing to start" aren't due to broken code, but rather unreasonable startup tolerance settings from the platform.

So while this article appears to be just a supplement to Probe, it's actually helping you build:

> **A more comprehensive application lifecycle judgment model.**

---

## References
- Kubernetes Probes:https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/
- Configure Liveness, Readiness and Startup Probes:https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

---

## Next Day Suggestions
Next article suggestion to organize:

[[01-Requests and Limits Basics - Resource Management and OOMKilled Introduction]]