# 03-Configuration Changes and Application Updates: Image Changes, ConfigMap Changes, and Release Validation Approach

## Document Purpose
- Document Purpose: Introduction to the two most common update scenarios in application changes
- Applicable Stage: After completing Deployment rolling update basics, entering the mainline of image changes, configuration changes, and release validation
- Recommended Path: `04-Kubernetes/07-Apply deployment/10-Apply release, update and rollback/03-Cannot initialise Evolution's mail component.ConfigMap Change and release authentication lines`

## Tags
#Kubernetes #Deployment #ConfigMap #MirrorUpdate #ConfigureChanges #Organisation #Rollout #ApplyUpdate #Clouds. #Transport

---

## I. Why Learn "Configuration Changes and Application Updates" Separately

During the application release phase, the most common changes are not "redeploying from scratch", but rather focus on the following two scenarios:

- Image changes
- Configuration changes

Examples include:
- Business releases a new version, image tag upgrades from `v1.0.0` to `v1.0.1`
- Application configuration adjustment, log level changes from `info` to `debug`
- Interface address, database address, switch parameters change
- Resource limits, probe parameters, environment variables adjustment

These changes appear to be "application updates", but their impact paths are not entirely the same.

Therefore, this article's focus is not to repeat rollout basics, but to clearly explain the following three things:

- How image changes trigger application updates
- Why ConfigMap changes cannot be simply understood as "auto-effective after modification"
- How to validate update results after release

---

## II. Why Understand Image Changes and Configuration Changes Separately

Although they both belong to "application updates", their points of action are different.

### What Image Changes Are Like
Image changes are more like:
- Application code version changes
- Binary program changes
- Container content changes

They typically directly correspond to:
- New feature launch
- Bug fixes
- Security patches
- Dependency library upgrades

### What Configuration Changes Are Like
Configuration changes are more like:
- Runtime parameter changes
- Environment variable changes
- Application behavior switch changes
- External dependency address changes
- Log level, thread count, connection pool parameter changes

They typically don't mean the program code has changed, but affect how the application runs.

### Operations Understanding Focus
Although both are called "updates", they are essentially two different change paths:

> **Image changes update the program itself, configuration changes update the program's runtime conditions.**

---

## III. How Image Changes Typically Trigger Application Updates

In Deployment scenarios, image updates are the most typical and intuitive type of release action.

### A Basic Example

Original Deployment:

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

If the image becomes:

    image: my-app:v1.0.1

This means:
- Pod Template has changed
- Deployment will create a new ReplicaSet
- Rolling update begins
- Old and new Pods gradually take over

### Common Update Methods

#### Method 1: Modify YAML and Apply

    kubectl apply -f deployment.yaml

#### Method 2: Directly Set Image

    kubectl set image deployment/demo-web web=my-app:v1.0.1 -n app-demo

### Operations Understanding Focus
In Deployment, the most common result of image changes is:

> **Triggering a new rollout.**

---

## IV. What to Observe When Image Changes Occur

After image updates complete, you shouldn't only check if the command executed successfully, but also observe if the update process meets expectations.

### Common Observation Commands

#### 1. Check Rollout Status

    kubectl rollout status deployment/demo-web -n app-demo

#### 2. Check Deployment Status

    kubectl get deploy -n app-demo
    kubectl describe deploy demo-web -n app-demo

#### 3. Check ReplicaSet Transition Process

    kubectl get rs -n app-demo

#### 4. Check New Pod Status

    kubectl get pod -n app-demo -o wide

#### 5. Check New Version Logs

    kubectl logs -n app-demo <pod-name>

### What Each Layer Is Checking

- Rollout status: Whether the overall update is complete
- Deployment: Whether the desired replicas match the available replicas
- ReplicaSet: Whether new and old version replicas are transitioning
- Pod: Whether the new version has successfully started and is Ready
- Logs: Whether the application has internal errors

### Operations Understanding Focus
Validation of image updates should not stop at the Deployment layer, but should at least check:

> **The update process, object status, and application logs at three layers.**

---

## V. Why ConfigMap Changes Cannot Simply Be Understood as "Auto-Effective After Modification"

This is the easiest point to misunderstand in configuration changes.

Many people when first using ConfigMap tend to form an intuitive understanding:

- ConfigMap is a configuration object
- Changing ConfigMap equals changing application configuration
- Therefore, the application will automatically run with the new configuration

This understanding doesn't always hold.

### A More Accurate Understanding
After ConfigMap updates, whether the application will actually use the new configuration depends on the following questions:

- How the application reads the configuration
- Whether the configuration is injected via environment variables or mounted as a file
- Whether the application supports hot reloading
- Whether the Pod will automatically restart
- Whether the release process explicitly triggers an update

### A Basic Conclusion
ConfigMap changes the:

> **Content of the configuration object in the cluster**

But this does not automatically mean:

> **The running Pod is definitely using the new configuration.**

---

## VI. Characteristics of ConfigMap Updates When Used as Environment Variables

First look at a common usage: referencing ConfigMap via environment variables.

### Example

env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: demo-config
        key: LOG_LEVEL

### What Does This Mean
This indicates that the container will read the value from the ConfigMap when it starts and inject it into the environment variable.

### Key Points
Environment variables are injected **when the container starts**.  
This means:

- The ConfigMap is modified later
- Environment variables in already running Pods will not automatically change

### What is Usually Done to Make New Configurations Take Effect
Typically, you need to:
- Rebuild the Pod
- Or trigger a new Deployment rollout

### Operations Understanding Focus
If ConfigMap is injected via environment variables, remember this key sentence:

> **Modifying the ConfigMap does not automatically change environment variables in already running containers.**

---

## VII. Characteristics of Updates When ConfigMap is Mounted as a File

Another common approach is to mount the ConfigMap as a file.

### Example

    volumes:
      - name: app-config
        configMap:
          name: demo-config

    volumeMounts:
      - name: app-config
        mountPath: /etc/app-config

### What Does This Mean
This indicates that the contents of the ConfigMap will be mounted as files into the container.

### How Is This Different From Environment Variables
In this approach, when the ConfigMap content is updated, the mounted file may reflect the new content under certain circumstances.

But there are issues:

- File content changes do not automatically trigger application re-reads
- Many applications only read configuration files at startup
- If the application does not support hot reloading, even if the file changes, the running logic may not change

### Operations Understanding Focus
When mounted as a file, ConfigMap updates are more like:

> **New configuration files have arrived in the container**

But this does not necessarily mean:

> **The application is already running with the new configuration.**

---

## VIII. Why Configuration Changes Often Still Require Triggering Pod Rebuilds

At this stage, establish a pragmatic judgment first:

> **Many configuration changes ultimately still require a new Pod startup process.**

### Reasons Include

#### 1. Applications Only Read Configuration at Startup
This is the most common scenario.

#### 2. Applications Do Not Support Hot Reloading
Many business applications and middleware do not actively monitor configuration changes.

#### 3. Environment Variables Do Not Support In-Place Refresh
Environment variables typically do not dynamically change after the container starts.

### Common Practices
Therefore, many teams commonly take the following approaches when facing ConfigMap changes:

- Modify the ConfigMap
- Then trigger a Deployment update
- Let the Pod rebuild
- Let the new Pod start with the new configuration

### Operations Understanding Focus
Configuration changes do not automatically form a "release action,"  
Often, you still need:

> **Explicitly trigger a new release process.**

---

## IX. How to Trigger Deployment Updates When ConfigMap Changes

The most straightforward approach is:

- After the ConfigMap is modified
- Then make the Deployment's Pod Template change
- Thus triggering a rollout

### Common Method One: Manually Modify Deployment Annotations

For example, add a change in the Pod Template:

    spec:
      template:
        metadata:
          annotations:
            config-change-time: "2026-04-02-1000"

As long as the Pod Template changes, the Deployment will trigger a new update.

### Common Method Two: Reapply YAML with Template Changes
For example, update an annotation, label, or environment variable placeholder.

### Common Method Three: Use rollout restart

    kubectl rollout restart deployment/demo-web -n app-demo

### Meaning of This Command
It does not modify the business logic, but rather:

> **Forces a new rollout of the Deployment, making the Pods restart gradually.**

### Operations Understanding Focus
After the ConfigMap is modified, if the goal is "to make the new configuration truly enter the running application," you usually still need to solve:

> **How to make the Pod restart with the new configuration.**

---

## X. What is `kubectl rollout restart`

This is a very practical command in configuration change scenarios.

### Command Example

    kubectl rollout restart deployment/demo-web -n app-demo

### Purpose
It initiates a new rollout for the Deployment, making the Pods restart gradually.

### When Is This Command Suitable
It is especially suitable for the following scenarios:

- The ConfigMap has been modified
- The Secret has been modified
- The business application does not support hot reloading
- You want to restart the Pod without changing the image to make the new configuration take effect

### What This Command Does Not Mean
It does not mean:
- The application code version has changed
- The image version has changed

It is more like:

> **Restarting a round of Pods based on the current configuration.**

### Operations Understanding Focus
In configuration change scenarios, `rollout restart` can be seen as:

> **Converting static configuration changes into a controlled Pod rebuild process.**

---

## XI. Differences in Verification Approaches Between Image Changes and ConfigMap Changes

Although both may ultimately manifest as Deployment updates, the verification focus is not entirely the same.

### What Should Be Focused on for Image Changes
- Whether the new image is pulled successfully
- Whether the new Pod starts successfully
- Whether the new version logs are normal
- Whether the new features work as expected
- Whether the old version has smoothly exited

### What Should Be Focused on for ConfigMap Changes
- Whether the ConfigMap content has truly been updated
- Whether the Pod has been rebuilt
- Whether the new Pod has read the new configuration
- Whether the application behavior has changed according to the configuration
- Whether there are issues with the old configuration not taking effect

### A More Intuitive Comparison

| Change Type | What to Focus On |
|---|---|
| Image Change | Whether the new program works normally |
| Configuration Change | Whether the new runtime conditions have truly taken effect |

### Operations Understanding Focus
The update action may be similar, but the verification focus must follow the change type.

---

## XII. Why Can't You Only Check if Pod is Running for Release Validation

This is the easiest part to simplify during the release phase.

### What Does Pod Running Indicate
It only indicates:
- The container's main process is still alive
- Kubernetes considers the Pod to be in a running state

### What Else Should Be Checked for Release Validation
At least, you should also check:

#### 1. Deployment Level
- Whether the rollout is complete
- Whether the available count meets the requirements

#### 2. Pod Level
- Whether the Pod is Ready
- Whether there are restarts
- Whether there are probe failures

#### 3. Configuration Layer
- Is the new image being used
- Has the new ConfigMap been read
- Are the expected environment variables injected

#### 4. Application Layer
- Are pages or interfaces functioning normally
- Are there error logs
- Are key features available

#### 5. Business Layer
- Are core requests successful
- Is the error rate abnormal
- Is there a noticeable delay increase

### Operations Understanding Focus
True release validation is not about "whether the Pod exists", but:

> [!note] Whether the new version or configuration has stably taken over the business.

---

## Thirteen. Minimal Image Change Example Process

### 1. Check Current Deployment

    kubectl get deploy -n app-demo
    kubectl get pod -n app-demo -o wide

### 2. Update Image

    kubectl set image deployment/demo-web web=my-app:v1.0.1 -n app-demo

### 3. Check Rollout Status

    kubectl rollout status deployment/demo-web -n app-demo

### 4. Check Old and New ReplicaSet

    kubectl get rs -n app-demo

### 5. Check New Pod

    kubectl get pod -n app-demo -o wide

### 6. Check Logs

    kubectl logs -n app-demo <new-pod-name>

### 7. Perform Business Access Validation
For example:
- Page access
- API calls
- Health check interface validation

---

## Fourteen. Minimal ConfigMap Change Example Process

Assume the current application provides log level configuration via ConfigMap.

### 1. Check Current ConfigMap

    kubectl get configmap -n app-demo
    kubectl describe configmap demo-config -n app-demo

### 2. Modify ConfigMap

    kubectl edit configmap demo-config -n app-demo

Change for example:

    LOG_LEVEL=info

To:

    LOG_LEVEL=debug

### 3. Trigger Deployment Restart

    kubectl rollout restart deployment/demo-web -n app-demo

### 4. Check Rollout Status

    kubectl rollout status deployment/demo-web -n app-demo

### 5. Check New Pod

    kubectl get pod -n app-demo -o wide

### 6. Enter Pod or Check Logs to Validate Configuration Effectiveness
For example:
- Check container environment variables
- Check configuration file content
- Check log level changes

### Operations Understanding Focus
The key of this process is not "ConfigMap changed", but:

> [!note] Whether the application has truly restarted with the new configuration.

---

## Fifteen. Common Misconceptions in Configuration Changes

### 1. Assuming changing ConfigMap equals completion of release
Actually, you need to check:
- Whether the Pod is rebuilt
- Whether the application re-reads the configuration
- Whether the configuration truly takes effect

### 2. Only checking Deployment rollout success, not application behavior
Rollout success only indicates object update completion, not business logic correctness.

### 3. Not performing log and interface validation after configuration changes
This easily lets "silent errors" slip through.

### 4. Using environment variables to inject configuration, yet mistakenly believing Pod will automatically refresh
Environment variable injection typically does not support dynamic refresh.

### 5. Configuration changes lack clear release actions
If only the ConfigMap is changed without triggering rebuild, running applications may still use the old configuration.

---

## Sixteen. Most Important Recognitions in This Article

### 1. Image and configuration changes both belong to application updates, but with different impact paths
This is the first recognition.

### 2. Deployment image updates directly trigger new rollout
This is the second recognition.

### 3. ConfigMap changes do not necessarily automatically make the application use the new configuration
This is the third recognition.

### 4. Many configuration changes still require explicit Pod rebuild to take effect
This is the fourth recognition.

### 5. Release validation must focus on whether the change truly takes effect, rather than just checking if objects are Running
This is the fifth recognition.

---

## Seventeen. Stage Summary

The two most common types of application update actions are:

- Image change
- Configuration change

Image change leans toward program version changes, typically directly triggering a new rollout from Deployment.  
Configuration change leans toward runtime condition changes, but whether it truly takes effect depends on:

- Configuration injection method
- Whether the application supports hot reloading
- Whether a new Pod rebuild is explicitly triggered

Through this article, you should at least establish these basic understandings:

- Update actions have different types
- Rollout is just the update process, not equal to business validation
- ConfigMap changes often still require rollout restart
- Release validation should move from object layer to application and business layer

As long as this layer of understanding is clear, subsequent steps into rollback capabilities will be more natural.

---

## Eighteen. Keyword Mnemonics

- Image change: Program version change
- Configuration change: Runtime parameter change
- ConfigMap: Common configuration carrier
- rollout restart: Force trigger new Pod rebuild
- Deployment update: Triggered by Pod Template change
- Release validation: Confirm if the update result is truly usable
- Pod Running: Not equal to configuration and business being correctly effective

---

## Nineteen. Operations Extended Understanding

During application release, the most common error is not "not knowing how to execute update commands", but treating all updates as the same type of problem.

In reality:

- Image change is more like "changing the program"
- Configuration change is more like "changing runtime conditions"

Although they may both eventually manifest as Pod rebuilds, the release validation approach is not entirely the same.

Only by separating these two types of changes will subsequent judgment be more stable when handling:

- Deployment updates
- Helm upgrade
- Configuration release
- Fault rollback

---

## References
- Kubernetes Deployment Official Documentation
- Kubernetes ConfigMap Official Documentation
- Kubernetes Official Documentation

---

## Next Day Suggestions
Next article suggestion to organize:

[[04-Helm Deployment Practice - Installation Viewing Upgrade and Uninstallation of an Application]]