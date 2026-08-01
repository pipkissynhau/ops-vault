# 01-Nginx Static Page Configuration Decoupling: ConfigMap Mounting Basics Practice

## Document Notes
- Document Positioning: Stateless Application Configuration Decoupling Introduction Practice
- Applicable Stage: After completing Deployment, Service, NodePort, and Rolling Update basics, entering the first practical session of configuration management
- Recommended Path: `04-Kubernetes/07-Application Deployment/05-Application Configuration and Secret Management/01-Nginx Static Page Configuration Decoupling: ConfigMap Mounting Basics Practice

## Tags
#Kubernetes #ConfigMap #Nginx #StaticPage #ConfigurationManagement #ConfigureSolver #NoStatusApplication #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why Learn Configuration Decoupling

The previous Nginx static page practice mainly completed:

- Image building
- Deployment deployment
- Service access entry provision
- NodePort external exposure
- Replica scaling and rolling updates

But these practices had an unresolved issue:

> **Page content and configuration content are still embedded in the image.**

This leads to several typical problems:

### 1. Page content modification requires image rebuild
For example, changing a welcome message or HTML text requires re-building, pushing, and updating Deployment.

### 2. Configuration and image coupling is too tight
Images are better suited for carrying:

- Program itself
- Runtime environment
- Stable base files

Rather than frequent changes in environment configuration and display content.

### 3. Not conducive to environment differentiation
Development, testing, and production environments may have different page content and Nginx configurations. If they are hard-coded in the image, maintenance costs will be high.

Therefore, starting from this article, we will officially enter:

> **Configuration and image decoupling.**

---

## II. What is Configuration Decoupling

Configuration decoupling can be understood as:

**Separating variable content originally written into images or hard-coded in containers, to be injected by Kubernetes at runtime.**

The "variable content" typically includes:

- Application configuration files
- Website page files
- Runtime parameters
- Non-sensitive environment variables
- Service port configuration
- Nginx configuration snippets

In Kubernetes, the core object for carrying such **non-sensitive configurations** is:

> **ConfigMap**

---

## III. Why Nginx Static Pages Are Ideal for ConfigMap Practice

Nginx static pages are very suitable as the first object for ConfigMap practice, reasons include:

- Page content is plain text, intuitive
- Modification effects are easy to observe
- No complex business logic involved
- Clearly see the difference between "content in the image" and "runtime-mounted content"
- Can practice both page content mounting and Nginx configuration mounting

So it is one of the best first objects for understanding ConfigMap.

---

## IV. What is ConfigMap

ConfigMap is an object in Kubernetes used to store **non-sensitive configuration data**.

It is typically suitable for storing:

- Configuration file content
- Application parameters
- Page content
- Text templates
- Ordinary environment variables

You can understand it as:

> **A "configuration container" provided by Kubernetes specifically for storing non-sensitive configurations.**

### A Very Important Boundary
- **Ordinary configuration**: Place in ConfigMap
- **Sensitive configuration** (e.g., passwords, tokens, keys): Usually place in Secret

---

## V. How to Inject ConfigMap into Pod

ConfigMap has two common injection methods:

### 1. Inject as environment variables
Suitable for:

- Applications reading parameters via environment variables
- Not too many parameters
- Simpler configuration scenarios

### 2. Mount as files into the container
Suitable for:

- Nginx configuration files
- HTML pages
- Application YAML/JSON/properties files
- Script files

This article focuses on:

> **Mounting as files into the container.**

Because for Nginx pages and configurations, this method is the most intuitive.

---

## VI. What Goals Should This Practice Achieve

After completing this article, it is recommended to at least achieve the following:

### 1. Understand why configurations should not always be embedded in images
### 2. Understand the basic role of ConfigMap
### 3. Be able to place static page content into a ConfigMap
### 4. Be able to mount a ConfigMap to the Nginx container directory
### 5. Understand how mounted content can override existing files in the image
### 6. Be able to perform basic troubleshooting for ConfigMap mounting issues

---

## VII. What Is the Minimal Practice Approach

The core idea of this practice is very simple:

### Originally
Page files `index.html` were placed in the image via Dockerfile during image building.

### Now
Place page file content into a ConfigMap, then mount it to the Nginx page directory when the Pod runs.

This way:

- The image can remain unchanged
- Page content can be modified independently
- Configuration and image start to decouple

---

## VIII. A Simplest Example of Page Content ConfigMap

Here is a minimal ConfigMap example:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: nginx-web-content
    data:
      index.html: |
        <!DOCTYPE html>
        <html lang="zh-CN">
        <head>
          <meta charset="UTF-8">
          <title>nginx configmap demo</title>
        </head>
        <body>
          <h1>Hello ConfigMap</h1>
          <p>This page is mounted from ConfigMap.</p>
        </body>
        </html>

---

## IX. How to Understand This ConfigMap

### 1. `kind: ConfigMap`
Indicates this is a ConfigMap object.

### 2. `metadata.name`
Indicates the name of this ConfigMap is:

    nginx-web-content

### 3. `data`
Indicates the data content it stores.

### 4. `index.html`
The key name here is very important because when mounted as a file, it usually becomes the filename directly.

That is:

- key: `index.html`
- Mounted filename is usually: `index.html`

### Operations Understanding Focus
This point is very critical:

> **The keys in ConfigMap often correspond to the mounted filenames.**

---

## X. Mounting ConfigMap to Deployment

# A Minimal Deployment Example: Mounting a ConfigMap to Nginx Page Directory

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
              image: harbor.example.com/demo/nginx-web:v1
              ports:
                - containerPort: 80
              volumeMounts:
                - name: web-content
                  mountPath: /usr/share/nginx/html/index.html
                  subPath: index.html
          volumes:
            - name: web-content
              configMap:
                name: nginx-web-content

---

## 11. Understanding the Deployment Mounting Logic

### 1. `volumes`
This defines a volume:

    web-content

The data source for this volume is not a disk, nor a PVC, but:

    configMap:
      name: nginx-web-content

This means the volume's content comes from a ConfigMap.

### 2. `volumeMounts`
Here, this volume is mounted into the container.

The most critical part is:

    mountPath: /usr/share/nginx/html/index.html
    subPath: index.html

This indicates:

- Extracting the key `index.html` from the ConfigMap
- Mounting it to the container's `/usr/share/nginx/html/index.html`

### 3. Final Effect
When Nginx accesses the default homepage, it will read this mounted `index.html`.

This means:

> **Runtime-mounted page files will overwrite the same-path files in the original image.**

---

## 12. Why Use `subPath` Here

This is a critical best practice.

### Without Using `subPath`
Mounting the entire ConfigMap directly to:

    /usr/share/nginx/html/

May replace the entire directory. This is acceptable in some scenarios but may introduce side effects, such as:

- Other files in the original directory becoming invisible
- Lack of precision
- Overkill when only replacing a single file

### Benefits of Using `subPath`
It allows:

- Mounting only a single file
- Precisely replacing a target file
- More suitable for partial overrides

### Simplified Understanding
- Without `subPath`: More like mounting an entire directory
- With `subPath`: More like precisely mounting a single file

---

## 13. What Should the Access Result Be After This Practice

If:

- The Pod is running normally
- The ConfigMap is created successfully
- The mounting path is correct
- Nginx reads the file normally

Then accessing the page should show:

- `Hello ConfigMap`
- `This page is mounted from ConfigMap.`

This indicates:

- The page content is no longer purely from the image
- But from the ConfigMap file mounted at runtime

---

## 14. Why This Is Called "Decoupling Configuration from the Image"

Because the page content is no longer fixed during image build.

### Previously
To change page content, you needed:

- Modify local files
- Rebuild the image
- Push the image
- Update the Deployment

### Now
You only need to modify the ConfigMap content and let the Pod use the new configuration.

### Operational Focus
This is the core value of configuration decoupling:

> **The image handles stable content, while the ConfigMap handles volatile content.**

---

## 15. What to Observe Most After Mounting a ConfigMap

When performing this practice, focus on observing the following points.

### 1. Did the ConfigMap Create Successfully
If the ConfigMap wasn't created successfully, the Pod mounting will fail naturally.

### 2. Is the Pod Running Normally
Mounting errors may cause the Pod to fail to start or run abnormally.

### 3. Has the Page Content Become the New Content
This is the most direct business verification.

### 4. Is the Mounting Path Accurate
Nginx defaults to reading:

    /usr/share/nginx/html/index.html

If the path is incorrect, even if the ConfigMap is mounted, the expected page won't be returned.

---

## 16. Will the Page Change Immediately After a ConfigMap Update

This is a critical question.

### Scenario 1: Mounting an Entire Directory
In some scenarios, after a ConfigMap update, mounted files may refresh automatically, but with potential delays.

### Scenario 2: Using `subPath`
If using `subPath`, note:

> **Files mounted via subPath won't typically auto-refresh when the ConfigMap updates.**

This means:

- The ConfigMap changes
- The Pod's single file mounted via subPath may not change immediately
- Common practice is to rebuild the Pod or perform a rolling update on the Deployment

### Operational Focus
This is why, in production, when discussing ConfigMap updates, it's essential to distinguish between:

- Mounting an entire directory
- Mounting a single file via subPath

---

## 17. Common Issues in This Practice

### 1. Pod Fails to Start
Common causes:

- Incorrect ConfigMap name
- YAML indentation errors
- Incorrect mounting path
- Non-existent subPath key

### 2. Page Still Shows Old Content
Common causes:

- ConfigMap update failed
- Pod not recreated
- subPath mount not hot-updated
- Browser cache interference
- Accessing the wrong Pod/Service

### 3. Page Access Error
Common causes:

- Mounting path override errors
- Incorrect Nginx page directory
- File content format anomalies

### 4. Mismatch Between ConfigMap Key and subPath
For example:

- ConfigMap key is `homepage.html`
- subPath is written as `index.html`

In this scenario, mounting typically fails or does not meet expectations.

---

## 18. What to Check First When Troubleshooting ConfigMap Mounting Issues

Recommend checking in this order.

### 1. First, Check if the ConfigMap Exists
Confirm that the object has been successfully created.

### 2. Then Check if the Deployment and Pod Are Normal
Confirm that the Pod has successfully started.

### 3. Then Check if the Mount Path and Key Match
This is one of the most common places to make a mistake.

### 4. Then Check the Final Page Content
Confirm whether the business result meets expectations.

### 5. Finally Consider Whether subPath Updates Are Not Taking Effect
This is especially common if you modify the ConfigMap but the page does not change.

---

## 19. Why ConfigMap Is Well Suited for Nginx Pages and Configuration Files

Because Nginx itself heavily relies on files:

- Page files
- Site configurations
- Location configurations
- Upstream configurations
- Reverse proxy rules

All of these are naturally suitable for management through "file mounting."

Therefore, Nginx is an excellent subject for practicing ConfigMap usage.

---

## 20. What Can This Practice Naturally Extend To

After completing this page mounting practice, you can naturally proceed to the following topics.

### 1. Mount Nginx Configuration Files
For example, placing:

- `default.conf`
- Custom site configurations

into a ConfigMap and then mounting it to the Nginx configuration directory.

### 2. Mount an Entire Directory Using ConfigMap
Observe the difference compared to `subPath`.

### 3. Inject ConfigMap Using Environment Variables
Suitable for practicing application configuration rather than file configuration.

### 4. Compare with Secret
Understand which configurations are suitable for ConfigMap and which must be placed in Secret.

### 5. Trigger Rolling Deployment with Configuration Updates
Learn the update strategy after configuration changes.

---

## 21. The Most Important Cognitive Points in This Practice

### 1. ConfigMap Is Used to Store Non-Sensitive Configurations
It is not used for storing passwords and keys.

### 2. ConfigMap Can Be Mounted as Files into Containers
This is one of the core methods for managing file-based application configurations.

### 3. The Keys in ConfigMap Often Become the Mounted File Names
Therefore, key naming is very important.

### 4. `subPath` Can Precisely Mount a Single File
But note its hot update behavior.

### 5. The Essence of Configuration Decoupling Is to Make the Image More Stable and Manage the Mutable Content Separately
This is crucial for actual operations.

---

## 22. What Level Should You Reach to Understand This Article

At this stage, it is recommended to reach the following level:

### 1. Understand Why Configuration Decoupling Is Done
### 2. Understand the Basic Purposes of ConfigMap
### 3. Be Able to Place Page Content into ConfigMap
### 4. Be Able to Mount ConfigMap to the Nginx Page Directory
### 5. Understand the Role of `subPath`
### 6. Be Able to Make Basic Judgments for "ConfigMap Changed But Page Not Updated"

---

## 23. Common Follow-Up Questions in Interviews

Common questions in this area include:

- What is ConfigMap used for?
- What is the difference between ConfigMap and Secret?
- How to inject ConfigMap into a Pod?
- What is the difference between file mounting and environment variable injection?
- What is `subPath` and what is its role?
- Why does the Pod's file not change immediately after ConfigMap changes?
- Why is it not recommended to put all application configurations into the image?
- Why are Nginx configuration files suitable for management with ConfigMap?

---

## 24. Stage Summary

Decoupling Nginx static page configurations is the first step from "image carrying everything" to "separating image and configuration management."

The most important part of this article is not remembering YAML, but establishing the following core understandings:

- ConfigMap is used to store non-sensitive configurations
- ConfigMap can mount text content as files into containers
- Images and configurations should be decoupled as much as possible
- `subPath` is suitable for precise single-file override
- Whether configuration updates take effect depends on the mounting method

As long as these relationships are clearly understood, subsequent learning about Secret, full directory mounting, environment variable injection, Nginx configuration mounting, and configuration change deployment will be much smoother.

---

## 25. Keyword Quick Notes

- ConfigMap: A Kubernetes object for storing non-sensitive configurations
- Configuration Decoupling: Separating mutable configurations from the image
- volume: Definition of data volume in a Pod
- volumeMount: Definition of mounting in the container
- subPath: Precise mounting of a single file or directory within a volume
- key: The data key name in ConfigMap
- Mount Override: Overriding the image's file at runtime

---

## 26. Operational Extension Understanding

From an operations perspective, ConfigMap is not an "additional feature," but rather an important starting point for standardized application deployment.

When business scale grows, if all pages, configurations, and parameters are still packed into the image, it will bring many problems:

- Small changes require rebuilding the image
- Managing environment differences becomes difficult
- Release frequency is dictated by configuration changes
- High operational adjustment costs

Therefore, the true value of configuration decoupling is not just "technically being able to mount a single file," but:

> **Let the image return to carrying the program itself, and let configurations return to runtime management.**

This will directly affect subsequent release efficiency, troubleshooting efficiency, and environment management capabilities.

---

## References
- Kubernetes ConfigMap: https://kubernetes.io/docs/concepts/configuration/configmap/
- Add ConfigMap data to a specific path in the Volume: https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/
- Kubernetes Volumes: https://kubernetes.io/docs/concepts/storage/volumes/

---

## Next Day Suggestions
Next article suggestion to organize:

[[02-Nginx Configuration File Decoupling - ConfigMap Mounting default.conf Basic Practice]]