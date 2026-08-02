## Why CPU and Memory Exceeding Limits Behave Differently

The mechanisms Kubernetes uses to handle CPU and Memory exceeding their limits are different, mainly because their roles in the system and their impact scope differ.

### CPU Exceeding Limits
When CPU usage exceeds the value set in `limits.cpu`, the container is typically throttled. This throttling is usually gradual, starting with lowering the container's priority so that other resources are allocated first. If the condition persists, the container may eventually be "throttled" significantly, meaning its execution speed is greatly reduced, and it may even be unable to complete certain tasks.

### Memory Exceeding Limits
Unlike CPU, Memory exceeding limits usually leads to more direct and severe consequences. When memory usage exceeds the value set in `limits.memory`, Kubernetes takes immediate action to protect other containers and the stability of the node. The most common measure is to "OOM Kill" the container, which forcibly terminates its process. This approach is taken to prevent memory leaks or other serious memory-related issues from causing the entire system to crash.

Therefore, the different system behaviors after CPU and Memory limits are exceeded reflect their distinct roles and importance in resource management. CPU exceeding limits mainly affects container execution efficiency and performance, while Memory exceeding limits directly threatens the overall stability and reliability of the system. This is extremely crucial and is a common issue encountered during troubleshooting.

#### CPU Exceeds Limits
This usually manifests as:

> **Restricted usage speed**

In other words, the application is still running, but:

- It runs slower.
- Response times are longer.
- Throughput decreases.

#### Memory Exceeds Limits
This usually results in:

> **The container is directly terminated.**

This is because memory is a more rigid resource. Once the limit is exceeded, the system terminates the container to maintain overall stability.

### Key Points for Operations and Maintenance Professionals
Therefore, during troubleshooting:

- CPU issues often appear as "slowness."
- Memory issues often result in "container termination."

---

## Section Sixteen: What is OOMKilled?
`OOMKilled` is one of the most frequently occurring abnormal states in Kubernetes / containers.

It typically indicates that:

> **The container is terminated by the system due to excessive memory usage or insufficient memory.**

#### The Term "OOM" Here
Refers to:

> Out Of Memory

In other words, there is not enough memory available.

### Common Scenarios
- The actual memory usage of the container exceeds `limits.memory`.
- There is a memory leak in the program.
- The configured memory limit is too low.
- The memory peak during startup is higher than expected.

---

## Section Seventeen: What Phenomena are Commonly Associated with OOMKilled?
In practical scenarios, you will often encounter these phenomena:

### 1. Pods Keep Restarting
Especially under the management of a Deployment, old containers are terminated and then re-created automatically.

### 2. The OOMKilled Status Appears in the Container Information
This is a very direct indication of the issue.

### 3. Intermittent Unavailability of Services
Since containers are frequently terminated, the services become unstable.

### 4. Problems Are More Likely to Occur During Startup
Some applications consume more memory during startup than when they are running stably. If the limit is too low, the application may be terminated during the startup phase.

---

## Section Eighteen: Why is OOMKilled Important?
This is not just a simple "program exit," but rather:

> **A forced action by the platform to control resource usage when it gets out of hand.**

In other words, when investigating OOMKilled issues, you should not only focus on the code but also consider the following factors:

- Whether the limit is too low.
- Whether the requested resources are reasonable.
- Whether the application experiences memory peaks.
- Whether there are any memory leaks.
- Whether there is a temporary high memory consumption during startup.

### Key Points for Operations and Maintenance Professionals
这类 issues typically illustrate that:

> **Resource configuration is also an essential part of application deployment.**

---

## Section Nineteen: What Problems Can Occur if requests Are Set Too Low?
### 1. It’s Easy to Underestimate Scheduling Requirements
The scheduler may think there are enough resources on the node, but in reality, the application will face significant operational pressure.

### 2. Nodes May Be Overloaded
Too many Pods will be scheduled onto a node, leading to intense resource competition.

### 3. Service Stability Declines
Especially when multiple services set too low requests, the overall node resources will become increasingly unstable.

### Key Points for Operations and Maintenance Professionals
Setting requests too low is essentially "underreporting the actual resource requirements."

---

## Section Twenty: What Problems Can Occur if limits Are Set Too Low?
### 1. If the CPU limit is too low, the application will frequently experience speed restrictions, manifested as:

- Slowness.
- Lagging.
- High latency.

### 2. If the memory limit is too low, the application is more likely to be:

- `OOMKilled`.

### 3. Issues Are More Likely to Occur During Startup
The memory peak during application startup may be temporarily higher. If the limit is too tight, the application may not even be able to start successfully.

### Key Points for Operations and Maintenance Professionals
Setting limits too low may seem like a way to save resources, but in reality, it can easily cause the application to fail.

---

## Section Twenty-One: Do Higher Values for requests and Limits Always Mean Better?
Not necessarily.

### Setting Them Too High Can Also Cause Problems

#### 1. If requests are set too high, it will make scheduling more difficult.
The scheduler may feel that there are not enough resources on the node to accommodate the requested amount.

#### 2. If limits are set too high, the resource boundaries will become too loose, allowing some problematic applications to consume excessive resources and affect other services.

### Key Points for Operations and Maintenance Professionals
The goal of resource configuration is not to set them as "as large as possible" or "as small as possible," but rather to:

> **Closely match the actual business needs while reserving a reasonable amount of redundancy.**

