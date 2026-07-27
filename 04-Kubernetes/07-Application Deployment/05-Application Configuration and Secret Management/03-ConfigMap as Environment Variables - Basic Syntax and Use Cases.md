# 03-ConfigMap as Environment Variables: Basic Syntax and Use Cases

## Document Description
- Document Focus: Advanced basics of using ConfigMap for environment variable injection
- Applicable Stage: After practicing mounting ConfigMap files, move on to understanding and applying environment variable injection
- Recommended Path: `04-Kubernetes/07-Application Deployment/05-Application Configuration and Key Management/03-ConfigMap as Environment Variables: Basic Syntax and Use Cases`

## Tags
#Kubernetes #ConfigMap #EnvironmentVariables #env #envFrom #ConfigurationManagement #ConfigurationDecoupling #ApplicationDeployment #BusinessContainerization #CloudNative #Ops #InterviewNotes

---

## I. Why Learn About “Environment Variable Injection”

In the previous two articles, we explored two typical uses of ConfigMap:

- Mounting page content as files through ConfigMap
- Mounting Nginx configuration files `default.conf` as files through ConfigMap

Both of these methods fall under:

> **File-based configuration injection.**

However, in real-world applications, not all configurations are suitable for being stored in files.  
Many applications commonly read configurations in the following ways:

- Reading environment variables at startup
- Obtaining parameters from environment variables during program execution
- Determining settings such as ports, environments, database addresses, log levels, and switches based on environment variables

Therefore, starting from this article, we will cover the second very common use of ConfigMap:

> **Using ConfigMap to inject values into containers as environment variables.**

---

## II. When to Use Environment Variables Instead of File Mounting

Both methods are widely used, but their applicable scenarios differ slightly.

### Scenarios More Suitable for File Mounting
Common examples include:

- Nginx configuration files
- HTML pages
- YAML/JSON-properties configuration files
- Script files
- Situations where the program explicitly requires reading a file path

### Scenarios More Suitable for Environment Variable Injection
Common examples include:

- Application runtime modes
- Service listening ports
- Database host addresses
- Redis addresses
- Log levels
- Function switches
- Less sensitive, small configuration items

### Simplified Understanding
- **File-based configurations**: Better suited for file mounting
- **Key-value parameters**: Better suited for environment variable injection

---

## III. What This Article Aims to Achieve

After completing this article, you should be able to:

### 1. Understand why many applications prefer reading configurations through environment variables
### 2. Comprehend the basic methods of using ConfigMap for environment variable injection
### 3. Distinguish between `env` and `envFrom`
### 4. Recognize the differences between mounting ConfigMap files and using them as environment variables
### 5. Determine which configurations are suitable for environment variable injection
### 6. Perform basic troubleshooting for environment variable injection issues

---

## IV. What Is Environment Variable Injection

Essentially, environment variable injection means:

> **Passing configuration data from Kubernetes in the form of key-value pairs to container processes.**

For applications, they typically perceive these values as:

- `APP_ENV=prod`
- `PORT=8080`
- `LOG_LEVEL=info`
- `DB_HOST=mysql-service`

In other words, the program doesn’t know whether these values come from a ConfigMap or were manually defined; it only knows that:

> **The environment variables exist when the container starts up.**

---

## V. Why Many Applications Prefer Reading Environment Variables

This is a common practice in modern application deployment for several reasons:

### 1. Simplicity and Directness
Values can be read immediately upon program startup, eliminating the need to parse complex files.

### 2. Better Compliance with the 12-Factor App Style
Many cloud-native applications inherently encourage externalizing configurations and prefer reading them through environment variables.

### 3. Easier Environment Differentiation
For example:

- Development environment: `APP_ENV=dev`
- Testing environment: `APP_ENV=test`
- Production environment: `APP_ENV=prod`

### 4. Suitable for Parameter-Based Configurations
Examples include:

- Ports
- Log levels
- Switches
- Service addresses

### Key Points for Ops Professionals
Environment variables are not meant to replace all configuration files; instead, they are particularly suitable for:

> **Carrying “small, clear key-value configurations.”**

---

## VI. Two Common Ways to Use ConfigMap as Environment Variables in Kubernetes

In Kubernetes, there are two common ways to use ConfigMap as environment variables:

### 1. `env`
Exactly specify which environment variable should come from a specific key in the ConfigMap.

### 2. `envFrom`
Import all keys from the ConfigMap into environment variables at once.

Both methods are widely used, but their applicable scenarios differ.

---

## VII. A Simple ConfigMap Example

Here is a basic ConfigMap example:

    apiVersion: v1
    kind: ConfigMap### 🔤 Summary of Key Points on Environment Variable Injection with ConfigMap

- **Suitable Scenarios**: Ideal for scenarios with many configuration items, well-named settings, and a need for overall injection.
- **Understanding Simplified**: `env` for selective injection; `envFrom` for bulk injection.

---

## § Thirteen: Differences Between Environment Variable Injection and File Mounting

This is one of the critical distinctions when understanding ConfigMap.

### File Mounting
- **Characteristics**:
  - Appears as a file in the container.
  - Suitable for configuration files, web pages, and scripts.
  - Programs usually read through file paths.

### Environment Variable Injection
- **Characteristics**:
  - Exists as environment variables within the process environment.
  - Ideal for small key-value parameters.
  - Programs typically read these through environment variables.

### **Selection Recommendations**
- If your application code already reads `APP_ENV`, `DB_HOST`, etc., then environment variable injection is more appropriate.
- If your application needs to access `/etc/app/config.yaml` or similar files, file mounting is a better choice.

---

## § Fourteen: What Configurations Are Better Suited for Environment Variables

Common examples include:

### 1. Running Environment Identifiers
- `APP_ENV`
- `SPRING_PROFILES_ACTIVE`

### 2. Service Listening Ports
- `PORT`
- `APP_PORT`

### 3. Log Levels
- `LOG_LEVEL`

### 4. Service Addresses
- `DB_HOST`
- `REDIS_HOST`
- `NACOS_SERVER_ADDR`

### 5. Feature Switches
- `FEATURE_X_ENABLED=true`

### **Ops Perspective**
Configurations suitable for environment variables usually have these common traits:

> **Short, specific values in key-value format.**

---

## § Fifteen: What Configurations Are Not Suitable for Direct Use with Environment Variables

Although environment variables are convenient, they are not suitable for all situations:

- **Large Configuration Texts**: Such as complete Nginx configurations or large YAML/YJSON files.
- **Multi-Level Complex Configurations**: Like complex application configuration trees.
- **Configurations That Require File Path Semantics**: For example, when a program explicitly needs to read a specific file.
- **Sensitive Configurations**: Such as passwords, tokens, or private keys.

These types of configurations are better stored in Secrets rather than ordinary ConfigMaps.

---

## § Sixteen: How Applications Use Environment Variables After Injection

This requires understanding from the “program’s perspective”.

If a container already has `APP_ENV=prod` and `LOG_LEVEL=info`, the application typically retrieves these values at startup or during runtime using the language-specific methods for reading environment variables.

For example, the application might use these values to determine:

- The current running environment.
- Which log level to use.
- Which service address to connect to.
- Which listening port to bind to.

### **Ops Perspective**
Kubernetes is only responsible for injecting the variables. Whether the application actually reads them, how it reads them, or whether the reading is successful depends entirely on the program’s logic.

---

## § Seventeen: Common Issues with Environment Variable Injection

### 1. ConfigMap Created Successfully, but Application Doesn’t Read Variables
- **Common Reasons**:
  - The Deployment doesn’t correctly specify `env` or `envFrom`.
  - The program doesn’t read the variables in the expected way.
  - The variable name doesn’t match what the program expects.

### 2. Incorrect Variable Names
- For example, if the program expects `APP_ENV`, but you inject `APP-ENV`, it won’t be recognized.

### 3. Incorrect ConfigMap Key Names
- For example, if the key in the ConfigMap is `APP_PORT` but the Deployment references it as `PORT`, there will be a mismatch.

### 4. Changes to ConfigMap Don’t Take Effect Immediately
- Often, it’s not that the ConfigMap wasn’t changed successfully, but rather:
  - The Pod hasn’t been refreshed.
  - The application only reads environment variables at startup.
  - The environment variables are not designed for real-time updates.

### **Ops Perspective**
Environment variable injection is different from file mounting. It’s important to remember that:

> **After an application process starts, environment variables are usually not updated in real-time.**

Therefore, changes to the ConfigMap may not be immediately reflected in the running process.

---

## § Eighteen: Why Rebuilding Pods Is Often Needed After Updating Environment Variables

This is because environment variables are typically injected into processes when the container starts.

In other words:

- The Pod reads the ConfigMap at startup.
- The process obtains the set of environment variables.
- These values usually don’t get automatically updated during the process’s runtime.

Therefore, if you modify the values in the ConfigMap, a common approach is to rebuild the Pod or use rolling updates so that new Pods use the updated variables.

### **Ops Perspective**
