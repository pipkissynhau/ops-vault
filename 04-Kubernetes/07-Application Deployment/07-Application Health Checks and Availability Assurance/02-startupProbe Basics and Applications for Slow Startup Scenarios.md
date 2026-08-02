# 02-startupProbe Basics and Applications for Slow Startup Scenarios

## Document Description
- Documentation Focus: Completion of the Probe system and introduction to applications with slow startup
- Applicable Phase: After mastering the basics of livenessProbe and readinessProbe, move on to understanding startupProbe and scenarios involving slow startup
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/07-Application Health Check and Availability Assurance/02-startupProbe Basics and Applications for Slow Startup Scenarios`

## Tags
#Kubernetes #Probe #startupProbe #livenessProbe #readinessProbe #Health Check #Slow Startup Application #Application Availability #Application Deployment #Business Containerization #Cloud Native #Operation and Maintenance #Interview Notes

---

## I. Why Learn startupProbe After livenessProbe and readinessProbe
The previous sections have established two crucial concepts:

- `livenessProbe`: Determines whether the application should be restarted.
- `readinessProbe`: Determines whether the application is ready to receive traffic.

These two types of Probes are sufficient for many basic scenarios.  
However, in real production environments, another common issue arises:

> **The application itself takes a very long time to start.**

Examples include:

- Slow startup of Java services
- Long initialization times for Spring Boot applications
- Time-consuming loading of configurations and dependencies for middleware
- Need for cache预热 when starting large-scale applications
- The application requiring waiting for databases, registries, or external dependencies to be ready
- Some services taking several tens of seconds or even minutes to start up

If only livenessProbe and readinessProbe are configured in this case, a troublesome situation may occur:

- The application is still in the process of starting.
- But livenessProbe starts checking too early.
- Since the application is not fully ready, it continuously fails the checks.
- kubelet determines that the container is “unhealthy” and restarts it.
- The application is repeatedly killed before it even has a chance to complete its startup.

This is where startupProbe comes into play:

> **startupProbe** is needed to address this issue.

---

## II. What Problems Does startupProbe Solve
It solves the problem of:

> **Preventing applications that are still starting from being prematurely marked as unhealthy by livenessProbe.**

In other words, startupProbe is not intended to replace livenessProbe or readinessProbe; rather, it provides an additional layer of protection for applications with slow startup processes.

### Simplified Explanation
If we consider:

- `readiness Probe` as determining “whether traffic should be allowed first”
- `livenessProbe` as determining “whether the application needs to be restarted”

then startupProbe serves to ensure that “we don’t rush to conclude that the application is unhealthy while it is still starting.”

---

## III. When Is startupProbe Particularly Needed
The following scenarios are ideal for implementing startup Probe:

### 1. Java / Spring Boot Services
These services typically have a longer startup process, especially when they rely on multiple dependencies.

### 2. Applications That Require Initializing Large Amounts of Resources
For example:

- Cache预热
- Loading large configurations
- Building connection pools
- Initializing thread pools

### 3. Applications That Rely on External Components at Startup
For instance:

- Waiting for databases to become available
- Waiting for Redis or Nacos to start up
- Waiting for configuration centers or registries to be ready

### 4. Middleware-Type Services
Some middleware components, such as:

- Nacos
- Elasticsearch
- Kafka-related components
- Customized complex services

### 5. Businesses That Gradually Reach Stability After Startup
For example, the first cold startup of a service may take significantly longer than subsequent check cycles.

---

## IV. What Are the Most Common Problems Without startupProbe
The most typical issue is:

> **An application takes a long time to start, but it is mistakenly killed by livenessProbe.**

For example, if a Java service normally takes 60 seconds to start up, and your livenessProbe is configured as follows:

- Starts checking after 10 seconds
- Checks every 5 seconds

In the first few dozen seconds, the application has not yet fully started its services.  
As a result, livenessProbe will assume that:

- The probe checks have failed.
- The application is unhealthy.
- It should be restarted.

This creates a vicious cycle:

- The container starts.
- The application is still initializing.
- The probe checks fail.
- The container restarts.
- The process repeats.

The final outcome is:

- The Pod keeps restarting.
- The service never becomes operational.
- It seems like there is an issue with the application, but in reality, it’s due to a mismatch between the Probe configuration and the application’s startup characteristics.

---

## V. What Is the Core Semantics of startupProbe
Here is the most critical point to remember:

> **startupProbe is used to determine whether an application### What Does This Mean?
The total allowable launch detection window is approximately:

- `5 seconds × 12 attempts = 60 seconds`

In other words:

> **The application has a “launch protection period” of about 60 seconds.**

If the startupProbe passes successfully within this time, it indicates that the application has completed its launch; if it fails continuously beyond the threshold, the container will be considered to have failed to start.

---

## XI. Why Is startupProbe Often Considered Together with failureThreshold?
The key reason for using startupProbe is not just “checking once,” but rather:

> **Providing the application with a certain amount of launch tolerance time.**

This duration is usually determined by the following two parameters:

- `periodSeconds`
- `failureThreshold`

### A Simple Formula
Tolerable launch time ≈ `periodSeconds × failureThreshold`

For example:

- `periodSeconds: 5`
- `failureThreshold: 12`

This would result in an approximately 60-second launch window.

### Key Points for Operations Engineers to Understand
This is why startupProbe is particularly suitable for applications with slow launches, as it allows Kubernetes to understand that:

> **Don’t conclude too early; it may take this much time for the application to fully start up.**

---

## XII. What Happens Once startupProbe Succeeds?
Once startupProbe succeeds, it typically indicates that:

> **The application has completed its launch phase.**

After that, Kubernetes will proceed with the normal checks, such as:

- readinessProbe
- livenessProbe

In other words, startupProbe acts more like a “launch gate.”

### Key Points for Operations Engineers to Understand
It is not a primary probe that runs continuously but rather serves as a **pre-launch protection mechanism**.

---

## XIII. What Happens If startupProbe Fails Continuously?
If startupProbe fails repeatedly until it exceeds the allowed threshold, Kubernetes will generally assume that:

- The application has not started successfully.
- The launch phase has failed.
- The container should be restarted.

### Possible Consequences
- The Pod may keep restarting.
- The number of container restarts may increase.
- The service may fail to start up properly.

### This May Look Similar to livenessProbe Failures, but the Meanings Are Different:
- livenessProbe failure: The application is not functioning during runtime.
- startupProbe failure: The application failed to start up in the first place.

---

## XIV. Why Is startupProbe Especially Suitable for Slow-Launching Applications?
It clearly communicates a very practical fact to Kubernetes:

> **Not all applications should be immediately deemed “alive” after starting.**

Slow-launching applications typically have the following characteristics:

- The process starts quickly.
- However, it takes some time before the service becomes fully available.
- Many initialization tasks need to be completed first.

Without startupProbe, Kubernetes might mistakenly classify a slow start as a failure. Therefore, the essential value of startupProbe is to distinguish between:

- **Slow launches**
- and
- **Failed launches**

---

## XV. When Is It Not Necessary to Use startupProbe?
Not all applications require startup Probe.

### Common Scenarios Where It May Not Be Needed:
- Applications start up very quickly.
- The launch logic is simple.
- There are no complex dependencies.
- The service can provide stable functionality within a few seconds.
- readinessProbe and livenessProbe are already sufficient.

For example:

- Simple Nginx pages.
- Very lightweight static services.
- Simple APIs with extremely short startup times.

### Key Points for Operations Engineers to Understand
startupProbe is designed for scenarios where the **launch phase is complex or slow**, not because more probes always mean better performance.

---

## XVI. What Are Some Common Configuration Errors?
### 1. Applications start up slowly, but no startupProbe is configured.
This can lead to livenessProbe failures occurring too early and causing unnecessary termination of the application.

### 2. The startupProbe window is set too short.
If an application actually needs 90 seconds to start up, configuring a 30-second window will result in failures.

### 3. The health check path is incorrectly specified.
This will prevent the startupProbe from ever succeeding.

### 4. StartingProbe is misused as a long-term running probe.
This ignores its primary purpose of providing **launch-phase protection**.

### 5. Confusion between readinessProbe and startupProbe.
It’s important to understand that one focuses on whether the application is ready for traffic, while the other checks if the launch process was successful.

---

## XVII. How Can You Clearly Differentiate Between startupProbe and readinessProbe?
This is a frequently asked question in interviews.

### startupProbe
Focuses on:

> **Whether the application has completed its launch process.**

If failures persist for too long, it indicates that the launch itself failed.

### readinessProbe
Focuses on:

> **Whether the application is currently suitable to receive traffic.**

This is particularly important for applications with slow startup times. In many production environments, the reason why an application fails to start is not that the code is actually broken, but rather that the platform's tolerance for its startup process is insufficiently reasonable.

Therefore, although this article may seem like just a supplement to the discussion on probes, it actually helps you establish:

> **a more comprehensive model for evaluating the entire application lifecycle.**

---

## References
- Kubernetes Probes: https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/
- Configuring Liveness, Readiness, and Startup Probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

---

## Next Steps
It is recommended to organize the following topic in the next article:

[[01- Basics of Requests and Limits: Resource Management and Introduction to OOMKilled]]