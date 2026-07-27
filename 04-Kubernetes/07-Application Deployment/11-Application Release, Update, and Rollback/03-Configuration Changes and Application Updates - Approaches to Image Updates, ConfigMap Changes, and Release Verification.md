# 03-Configuration Changes and Application Updates: Approaches to Image Updates, ConfigMap Changes, and Release Verification

## Document Description
- Document Purpose: An introduction to the two most common types of updates in application changes.
- Applicable Phase: After mastering the basics of Deployment rolling updates, this section delves into image changes, configuration modifications, and release verification processes.
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/10-Application Release, Updates, and Rollbacks/03-Configuration Changes and Application Updates: Approaches to Image Updates, ConfigMap Changes, and Release Verification`

## Tags
#Kubernetes #Deployment #ConfigMap #Image Update #Configuration Change #Release Verification #Rollout #Application Update #Cloud-Native #Operations and Maintenance

---

## I. Why Study “Configuration Changes and Application Updates” Separately

During the application release phase, the most common types of changes are not about “re-doing the entire deployment process,” but rather involve two main scenarios:

- Image updates
- Configuration modifications

For example:

- When a new version of the business software is released, the image tag may change from `v1.0.0` to `v1.0.1`.
- When the application configuration is adjusted, such as changing the log level from `info` to `debug`.
- When there are changes in API addresses, database locations, or switch parameters.
- When resource limits, probe settings, or environment variables need to be modified.

Although these changes all fall under the category of “application updates,” their actual impacts on the system differ.

Therefore, the focus of this section is not to repeat the basics of rollout processes but to clarify the following three key points:

- How image updates trigger application updates.
- Why ConfigMap changes cannot be simply understood as “automatically taking effect after modification.”
- How to verify the results of updates after they have been released.

---

## II. Why Should Image Updates and Configuration Modifications Be Considered Separately?

Although both are types of “application updates,” they affect different aspects of the system.

### What Image Updates Are More Like
Image updates are more akin to:
- Changes in the application code version.
- Changes in binary programs.
- Changes in the contents of containers.

They usually correspond to:
- The launch of new features.
- Bug fixes.
- Security patches.
- Upgrades of dependency libraries.

### What Configuration Modifications Are More Like
Configuration modifications are more like:
- Changes in runtime parameters.
- Changes in environment variables.
- Changes in application behavior settings.
- Changes in external dependency addresses.
- Adjustments to log levels, number of threads, or connection pool parameters.

While these changes may not necessarily involve altering the program code itself, they can significantly affect how the application operates.

### Key Points for Operations and Maintenance Professionals
Although both are called “updates,” they represent two different types of changes:

> **Image updates modify the core of the application, while configuration modifications alter the conditions under which the application runs.**

---

## III. How Image Updates Typically Trigger Application Updates

In the context of Deployment scenarios, image updates are one of the most typical and direct ways to make updates to an application.

### A Basic Example

Consider the original Deployment:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: demo-web
      namespace: app-demo
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: demo-web
      template:
        metadata:
          labels:
            app: demo-web
        spec:
          containers:
            - name: web
              image: my-app:v1.0.0
              ports:
                - containerPort: 8080

If the image is changed to:

    image: my-app:v1.0.1

This means that:
- The Pod Template will be updated.
- A new ReplicaSet will be created as part of the Deployment.
- A rollout process will begin.
- The old and new Pods will gradually take over from each other.

### Common Ways to Make Image Updates
#### Method 1: Modify the YAML File and Apply It Again
    kubectl apply -f deployment.yaml

#### Method 2: Directly Change the Image Specification
    kubectl set image deployment/demo-web web=my-app:v1.0.1 -n app-demo

### Key Points for Operations and Maintenance Professionals
In the case of a Deployment, the most common outcome of an image update is:

> **Triggering a new rollout process.**

---

## IV. What Should Be Observed When an Image Update Is Completed

After an image update is completed, it is not enough to simply check whether the command was successful; it is also important to verify that the entire update process proceeded as expected.

### Common Commands for Observation
#### 1. Check the Status of the Rollout
    kubectl rollout status deployment/demo-web -n app-demo

#### 2.config-change-time: "2026-04-02-1000"

Whenever the Pod Template changes, the Deployment will trigger a new round of updates.

### Common Method Two: Reapply YAML with Template Changes
For example, updating certain annotations, labels, environment variable placeholders, etc.

### Common Method Three: Use rollout restart

    kubectl rollout restart deployment/demo-web -n app-demo

### The Meaning of This Command
It does not modify the business logic; rather, it:

> **Forces the Pods under the Deployment to restart and be re Rolled out.**

### Key Points for Ops Understanders
After modifying a ConfigMap, if the goal is to “make the new configuration actually take effect in running applications,” one must also address:

> **How to get the Pods to restart with the new configuration.**

---

## Ten: What is `kubectl rollout restart`
This is a very useful command in configuration change scenarios.

### Command Example

    kubectl rollout restart deployment/demo-web -n app-demo

### Function
It initiates a new rollout for the Deployment, causing the Pods to be gradually rebuilt.

### Suitable Scenarios
It is especially suitable for the following situations:

- The ConfigMap has been modified.
- The Secret has been modified.
- The business application does not support hot loading.
- You want to restart only the Pods without changing the image to apply the new configuration.

### What It Does Not Indicate
It does not indicate that:

- The application code version has changed.
- The image version has changed.

Rather, it is more like:

> **Restarting the Pods based on the current configuration.**

### Key Points for Ops Understanders
In configuration change scenarios, `rollout restart` can be seen as:

> **Turning static configuration changes into a controlled process of Pod rebuilding.**

---

## Eleven: How Do Verification Approaches Differ for Image Changes and ConfigMap Changes?
Although both may ultimately result in a Deployment update, the focus of verification is not entirely the same.

### What Should Be Focused on More for Image Changes
- Whether the new image was successfully pulled.
- Whether the new Pods started successfully.
- Whether the logs of the new version are normal.
- Whether the new features work as expected.
- Whether the old version has been smoothly shut down.

### What Should Be Focused On More for ConfigMap Changes
- Whether the content of the ConfigMap has actually been updated.
- Whether the Pods have been rebuilt.
- Whether the new Pods have read the new configuration.
- Whether the application behavior has changed according to the configuration.
- Whether there are any issues where the old configuration is still in effect.

### A More Intuitive Comparison

| Change Type | What Is More Important to Verify |
|---|---|
| Image Change | Whether the new program works correctly. |
| Configuration Change | Whether the new operating conditions have actually taken effect. |

### Key Points for Ops Understanders
The update actions may be similar, but the focus of verification must align with the type of change.

---

## Twelve: Why Cannot Release Verification Be Limited to Checking Whether Pods Are Running?
This is the part that is most easily simplified during the release phase.

### What Does “Pod Running” Indicate?
It only indicates that:

- The main container process is still alive.
- Kubernetes considers the Pod to be in the running state.

### But What Else Should Release Verification Look At?
At minimum, it should also check:

#### 1. At the Deployment Level
- Whether the rollout is complete.
- Whether the “available” status meets requirements.

#### 2. At the Pod Level
- Whether the Pod is Ready.
- Whether there have been any restarts.
- Whether any probes have failed.

#### 3. At the Configuration Level
- Whether the new image is being used.
- Whether the new ConfigMap has been read.
- Whether the expected environment variables have been injected.

#### 4. At the Application Level
- Whether the pages or interfaces are functioning correctly.
- Whether there are any error logs.
- Whether key functions are available.

#### 5. At the Business Level
- Whether core requests are successful.
- Whether the error rate is abnormal.
- Whether there is a significant increase in latency.

### Key Points for Ops Understanders
True release verification is not about “whether the Pod is running” but about:

> **Whether the new version or configuration has successfully taken over business operations.**

---

## Thirteen: A Minimized Example Process for Image Change
### 1. View the Current Deployment

    kubectl get deploy -n app-demo
    kubectl get pod -n app-demo -o wide

### 2. Update the Image

    kubectl set image deployment/demo-web web=my-app:v1.0.1 -n app-demo

### 3. Check the Rollout Status

    kubectl rollout status deployment/demo-web -n app-demo

### 4. View the New and Old