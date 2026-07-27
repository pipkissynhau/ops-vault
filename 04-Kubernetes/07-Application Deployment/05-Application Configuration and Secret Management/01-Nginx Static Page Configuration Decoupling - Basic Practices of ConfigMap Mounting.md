# 01-Nginx Static Page Configuration Decoupling: Basic Practices of ConfigMap Mounting

## Documentation Description
- Document Location: Introduction to Stateless Application Configuration Decoupling Practices
- Applicable Phase: After completing the basics of Deployment, Service, NodePort, and rolling updates, proceed to the first practical application of configuration management
- Recommended Path: `04-Kubernetes/07-Application Deployment/05-Application Configuration and Key Management/01-Nginx Static Page Configuration Decoupling: Basic Practices of ConfigMap Mounting`

## Tags
#Kubernetes #ConfigMap #Nginx #Static Pages #Configuration Management #Configuration Decoupling #Stateless Applications #Application Deployment #Business Containerization #Cloud Native #Operations #Interview Notes

---

## I. Why Learn Configuration Decoupling

In the previous Nginx static page practices, we mainly completed:

- Image building
- Deployment deployment
- Providing access through Service
- Exposing via NodePort
- Replica scaling and rolling updates

However, there was a potential issue that remained unresolved in these practices:

> **The page content and configuration are still embedded in the image.**

This leads to several typical problems:

### 1. Rebuilding the image for any page content changes
For example, changing just a welcome message or an HTML snippet requires rebuilding, pushing, and updating the Deployment again.

### 2. Tight coupling between configuration and image
Images are better suited for storing:

- The program itself
- The runtime environment
- Stable base files

But they are not suitable for frequently changing environmental configurations and display content.

### 3. Difficulty in distinguishing environments
The page content and Nginx configurations in development, testing, and production environments may differ. If they are all hardcoded into the image, maintenance costs will be high.

Therefore, starting from this chapter, we will formally begin:

> **Configuration decoupling from images.**

---

## II. What is Configuration Decoupling

Configuration decoupling can be understood as:

**Separating variable content that was previously directly written into the image or hardcoded in the container, and allowing Kubernetes to inject it at runtime.**

The “variable content” here typically includes:

- Application configuration files
- Site page files
- Runtime parameters
- Non-sensitive environment variables
- Service port configurations
- Nginx configuration segments

In Kubernetes, the core object for storing such **non-sensitive configurations** is:

> **ConfigMap**

---

## III. Why Nginx Static Pages Are Ideal for Learning ConfigMap Basics

Nginx static pages make excellent candidates for learning ConfigMap basics for several reasons:

- The page content is plain text, making it easy to understand.
- Changes are immediately visible.
- It does not involve complex business logic.
- It clearly demonstrates the difference between “content within the image” and “content mounted at runtime.”
- It allows you to practice both mounting page content and Nginx configuration.

Therefore, it is one of the best first objects for understanding ConfigMap.

---

## IV. What is a ConfigMap

A ConfigMap in Kubernetes is an object used to store **non-sensitive configuration data**.

It is typically suitable for storing:

- Configuration file contents
- Application parameters
- Page content
- Text templates
- Ordinary environment variables

You can think of it as:

> **A “configuration container” provided by Kubernetes, specifically designed for storing non-sensitive configurations.**

### An Important Boundary
- **Ordinary configurations**: Store in ConfigMap
- **Sensitive configurations** (such as passwords, tokens, keys): Usually store in Secret

---

## V. How to Inject a ConfigMap into a Pod

There are two common ways to inject a ConfigMap into a Pod:

### 1. As environment variables
Suitable for:

- Applications that read parameters through environment variables
- When the number of parameters is small
- For simple configuration scenarios

### 2. As files mounted into the container
Suitable for:

- Nginx configuration files
- HTML pages
- Application YAML/JSON/properties files
- Script files

This chapter focuses on:

> **Mounting as files into the container.**

Because this method is the most straightforward for Nginx pages and configurations.

---

## VI. What Are the Goals of This Practice

After completing this chapter, you should be able to achieve at least the following:

### 1. Understand why configurations should not always be embedded in images.
### 2. Comprehend the basic function of a ConfigMap.
### 3. Place static page content inside a ConfigMap.
### 4. Mount the ConfigMap into the Nginx container directory.
### 5. Understand how the mounted content overwrites the original files in the image.
### 6. Perform basic troubleshooting for ConfigMap mounting issues.

---

## VII. The Core Idea of This Practice

The core idea of this practice is simple:

### Original Approach
The `index.html## Thirteen, What Should the Access Results of This Practice Be

If:

- The Pod is running normally.
- The ConfigMap is created successfully.
- The mount path is correct.
- Nginx is reading the file correctly.

Then, when accessing the page, you should see:

- `Hello ConfigMap`
- `This page is mounted from ConfigMap.`

This indicates that:

- The page content no longer comes solely from the image.
- Instead, it comes from the ConfigMap file that was mounted at runtime.

---

## Fourteen, Why Is This Called “Configuration and Image Decoupling”

Because now, the page content is no longer fixed during the image build phase.

### In the Past
To change the page content, you needed to:

- Modify local files.
- Rebuild the image.
- Push the updated image.
- Update the Deployment.

### Now
You only need to modify the contents of the ConfigMap and then let the Pod use the new configuration.

### Key Points for Ops Professionals
This is the core value of configuration decoupling:

> **The image is responsible for stable content, while the ConfigMap handles variable content.**

---

## Fifteen, What Should Be Observed Most After Mounting a ConfigMap

When performing this kind of practice, it is recommended to focus on the following points.

### 1. Whether the ConfigMap was created successfully
If the ConfigMap was not created at all, the Pod’s mount will naturally fail.

### 2. Whether the Pod is running normally
In case of mounting errors, the Pod may not start or may run abnormally.

### 3. Whether the page content has indeed changed to the new one
This is the most direct way to verify the business outcome.

### 4. Whether the mount path is correct
Nginx reads from the default path:

    /usr/share/nginx/html/index.html

If the path is incorrect, even if the ConfigMap is mounted correctly, the expected page will not be displayed.

---

## Sixteen, Will the Page Change Immediately After Updating a ConfigMap?

This question is very important.

### Case 1: Mounting an Entire Directory
In some scenarios, after updating a ConfigMap, the files mounted into the container may be automatically refreshed, but there might be a delay.

### Case 2: Using `subPath`
If you use `subPath`, it is particularly important to note that:

> **Files mounted using subPath are often not automatically updated in the container when the ConfigMap is changed.**

This means that:

- Even if the ConfigMap has been modified, the single file mounted through `subPath` may not change immediately.
- A common approach is to rebuild the Pod or perform a rolling update of the Deployment.

### Key Points for Ops Professionals
This is also why, in production environments, it is essential to distinguish between:

- Mounting an entire directory
- And mounting a single file using `subPath` when discussing ConfigMap updates.

---

## Seventeen, Common Issues in This Practice

### 1. The Pod Does Not Start
Common reasons include:

- Incorrect naming of the ConfigMap.
- YAML indentation errors.
- Incorrect mount path.
- Non-existent `subPath` key.

### 2. The Page Still Shows Old Content
Common causes are:

- The ConfigMap was not updated successfully.
- The Pod was not recreated.
- The subPath mount did not trigger an automatic update.
- Browser cache interference.
- The actual page being accessed does not belong to the current Pod/Service.

### 3. Page Access Errors
Common reasons include:

- Incorrect mount path coverage.
- Nginx’s page directory is set incorrectly.
- Abnormal file content format.

### 4. Inconsistent ConfigMap keys and subPaths
For example:

- The key in the ConfigMap is `homepage.html`, but the `subPath` is set to `index.html`.

In this case, the mount will usually fail or result in unexpected behavior.

---

## Eighteen, What Should Be Checked First When Troubleshooting ConfigMap Mount Issues?

It is recommended to follow this order for troubleshooting:

### 1. First, check whether the ConfigMap exists.
Ensure that the object itself has been successfully created.

### 2. Then, check whether the Deployment and Pod are running normally.
Confirm that the Pod has started successfully.

### 3. Next, verify whether the mount path and key match.
This is one of the most common areas where errors can occur.

### 4. After that, check the final page content.
Ensure that the business outcome matches expectations.

### 5. Finally, consider whether the `subPath` update has not taken effect.
If you have changed the ConfigMap but the page remains unchanged, this is a particularly common issue.

---

## Nineteen, Why Is ConfigMap Very Suitable for Nginx Pages and Configuration Files

Because Nginx itself relies heavily on files:

- Page files.
- Site configuration