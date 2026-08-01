# 03-ConfigMap as Environment Variables Injection: Basic Syntax and Application Scenarios

## Document Notes
- Document Position: Advanced basics of ConfigMap injection methods
- Applicable Stage: After completing ConfigMap file mounting practices, enter understanding and practical entry of environment variable injection
- Recommended Path: `04-Kubernetes/07-Apply deployment/05-Apply Configuration and Key Management/03-ConfigMap Injection as an environmental variable: basic syntax and application scenario`

## Tags
#Kubernetes #ConfigMap #EnvironmentalVariables #env #envFrom #ConfigurationManagement #ConfigureSolver #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why Learn "Environment Variables Injection"

The previous two articles have completed two typical ConfigMap practices:

- Page content mounted as files via ConfigMap
- Nginx configuration file `default.conf` mounted as files via ConfigMap

These two methods both belong to:

> **File-type configuration injection.**

However, in real business scenarios, not all configurations are suitable to be stored as files.  
Many applications commonly read configurations in the following ways:

- Read environment variables at startup
- Retrieve parameters from environment variables during runtime
- Use environment variables to determine ports, environments, database addresses, log levels, switch items, etc.

Therefore, starting from this article, we need to supplement the second very common usage of ConfigMap:

> **ConfigMap injected as environment variables into the container.**

---

## II. When to Use Environment Variables Instead of File Mounting

Both methods are common, but their applicable scenarios differ slightly.

### More Suitable for File Mounting Scenarios
Usually include:

- Nginx configuration files
- HTML pages
- YAML/JSON/properties configuration files
- Script files
- Scenarios where the program explicitly requires reading file paths

### More Suitable for Environment Variables Injection Scenarios
Usually include:

- Application runtime mode
- Service listening port
- Database host address
- Redis address
- Log level
- Functional switches
- Non-sensitive small configuration items

### Simplified Understanding
- **File-type configuration**: More suitable for mounting
- **Key-value parameters**: More suitable for environment variables injection

---

## III. What Goals to Achieve in This Article

After completing this article, it is recommended to at least achieve the following:

### 1. Understand why many applications prefer to read configurations through environment variables
### 2. Understand the basic method of ConfigMap injection as environment variables
### 3. Distinguish `env` and `envFrom`
### 4. Understand the differences between ConfigMap file mounting and environment variables injection
### 5. Judge what configurations are suitable for environment variables injection
### 6. Perform basic troubleshooting for environment variables injection issues

---

## IV. What is Environment Variables Injection

Environment variables injection, essentially:

> **Passes Kubernetes configuration data to the container process in key-value pair form.**

For applications, it typically perceives:

- `APP_ENV=prod`
- `PORT=8080`
- `LOG_LEVEL=info`
- `DB_HOST=mysql-service`

In other words, the program doesn't know whether these values come from ConfigMap or manual definitions; it only knows:

> **The environment variables already exist when the container starts.**

---

## V. Why Many Applications Prefer to Read Environment Variables

This is a very common practice in modern application deployment, with reasons including:

### 1. Simple and Direct
The program can read them at startup without needing to parse complex files.

### 2. More Suitable for 12-Factor App Style
Many cloud-native applications naturally encourage external configuration and prioritize reading through environment variables.

### 3. More Suitable for Environment Differentiation
For example:

- Development environment: `APP_ENV=dev`
- Testing environment: `APP_ENV=test`
- Production environment: `APP_ENV=prod`

### 4. More Suitable for Parameter-Type Configurations
For example:

- Port
- Log level
- Switch items
- Service address

### Operations Understanding Focus
Environment variables are not a replacement for all configuration files, but:

> **Especially suitable for carrying "small and clear key-value configuration".**

---

## VI. Two Common Ways of ConfigMap as Environment Variables Injection

In Kubernetes, ConfigMap as environment variables injection has two most common methods:

### 1. `env`
Precisely specify that a single environment variable comes from a specific key in ConfigMap.

### 2. `envFrom`
Batch inject all keys in ConfigMap as environment variables at once.

Both methods are commonly used but suitable for different scenarios.

---

## VII. Let's Look at a Simple ConfigMap Example

The following is a basic ConfigMap:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: app-config
    data:
      APP_ENV: prod
      LOG_LEVEL: info
      APP_PORT: "8080"

### Understanding This ConfigMap

#### 1. `kind: ConfigMap`
Indicates this is a ConfigMap object.

#### 2. `metadata.name`
This ConfigMap's name is:

    app-config

#### 3. `data`
Contains three key-value pairs:

- `APP_ENV=prod`
- `LOG_LEVEL=info`
- `APP_PORT=8080`

### Operations Understanding Focus
The key names here, when used for environment variables injection, typically become environment variable names directly.

---

## VIII. Method 1: Use `env` to Precisely Inject a Single Variable

The following is an example: /think

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
          image: harbor.example.com/demo/app:v1
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: LOG_LEVEL

---

## IX. How to Understand the `env` Writing Method

### 1. `env`
Indicates defining environment variables for the container.

### 2. `name`
Indicates the name of the environment variable injected into the container.

Example:

    name: APP_ENV

Indicates the container will have an environment variable called `APP_ENV`.

### 3. `valueFrom.configMapKeyRef`
Indicates the value is not hard-coded, but comes from ConfigMap.

Example:

- ConfigMap name: `app-config`
- key: `APP_ENV`

### Final Effect
The container will receive:

- `APP_ENV=prod`
- `LOG_LEVEL=info`

### Operations Understanding Focus
The characteristic of this method is:

> **Precise, controllable, only injecting variables you explicitly specify.**

---

## X. Method Two: Using `envFrom` to Batch Inject All Variables

Here is an example:

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
              image: harbor.example.com/demo/app:v1
              envFrom:
                - configMapRef:
                    name: app-config

---

## XI. How to Understand the `envFrom` Writing Method

### 1. `envFrom`
Indicates importing environment variables from a source.

### 2. `configMapRef.name`
Indicates where these variables come from, which is a ConfigMap.

Here it is:

    app-config

### Final Effect
If `app-config` contains:

- `APP_ENV=prod`
- `LOG_LEVEL=info`
- `APP_PORT=8080`

All these keys will become environment variables in the container.

### Operations Understanding Focus
The characteristic of this method is:

> **Simple, batch, suitable for importing entire configuration items.**

---

## XII. Difference Between `env` and `envFrom`

This is a highly frequent practical point and also a common interview question.

### `env`
Features:

- Precise specification
- Only injects desired variables
- Can control variable names in the container
- More granular control

Suitable scenarios:

- Only needs a few variables
- Wants explicit control over mapping relationships
- Wants to avoid exposing all ConfigMap keys into the container

### `envFrom`
Features:

- One-time batch import
- More concise syntax
- Suitable for scenarios where the entire ConfigMap needs to be injected

Suitable scenarios:

- Many configuration items
- Naming is already standardized
- Wants overall injection

### Simplified Understanding
- `env`: Pick-and-choose injection
- `envFrom`: Full-package injection

---

## XIII. Difference Between Environment Variable Injection and File Mounting

This is one of the key boundaries in understanding ConfigMap.

### File Mounting
Features:

- Appears as a file in the container
- Suitable for configuration files, pages, scripts
- Programs typically read via file paths

### Environment Variable Injection
Features:

- Exists as environment variables in the process environment
- Suitable for small key-value parameters
- Programs typically read via environment variables

### Selection Recommendation
If the application code already reads:

- `APP_ENV`
- `DB_HOST`
- `LOG_LEVEL`

It's more suitable for environment variable injection.

If the application requires reading:

- `/etc/app/config.yaml`
- `/usr/share/nginx/html/index.html`
- `/etc/nginx/conf.d/default.conf`

It's more suitable for file mounting.

---

## XIV. What Configurations Are Suitable for Environment Variables

Typically, the following are more suitable for environment variables:

### 1. Runtime Environment Identifier
Examples:

- `APP_ENV`
- `SPRING_PROFILES_ACTIVE`

### 2. Service Listening Ports
Examples:

- `PORT`
- `APP_PORT`

### 3. Log Level
Examples:

- `LOG_LEVEL`

### 4. Service Addresses
Examples:

- `DB_HOST`
- `REDIS_HOST`
- `NACOS_SERVER_ADDR`

### 5. Feature Switches
Examples:

- `FEATURE_X_ENABLED=true`

### Operations Understanding Focus
Configurations suitable for environment variables typically share common characteristics:

> **Short, small, clear, and key-value typed.**

### 1. Large Configuration Text
For example, complete Nginx configurations, long YAML, large JSON.

### 2. Multi-level Complex Configuration
For example, complex application configuration tree structures.

### 3. Configuration with File Path Semantics
For example, programs that explicitly require reading a specific configuration file.

### 4. Sensitive Configuration
For example:

- Passwords
- Tokens
- Private keys

These are typically better placed in Secret rather than ordinary ConfigMap.

---

## Sixteen. How Applications Use Environment Variables After Injection

This section should be understood from the "application perspective."

If the container already has:

- `APP_ENV=prod`
- `LOG_LEVEL=info`

The application typically retrieves these values during startup or runtime using the language's native environment variable reading methods.

For example, the application may decide based on environment variables:

- Current runtime environment
- Enabled log level
- Which service address to connect to
- Which listening port to bind

### Operations Understanding Focus
Kubernetes only handles:

> **Injecting the variables.**

Whether the application actually reads, how it reads, or if the reading is successful still depends on the program's own logic.

---

## Seventeen. Common Issues with Environment Variable Injection

### 1. ConfigMap Created Successfully, but Application Didn't Read Variables
Common reasons:

- Deployment didn't correctly write `env` or `envFrom`
- The program didn't read environment variables properly
- Variable names don't match what the program expects

### 2. Variable Names Written Incorrectly
For example, the program expects:

    APP_ENV

But you injected:

    APP-ENV

Or another name, and the program might not read it.

### 3. ConfigMap Key Written Incorrectly
For example:

- ConfigMap key is `APP_PORT`
- Deployment references `PORT`

If not correctly mapped, inconsistencies will occur.

### 4. ConfigMap Changed, but Application Didn't Take Effect
Often it's not that the ConfigMap wasn't updated successfully, but:

- Pod didn't refresh
- Application only reads environment variables at startup
- Environment variables aren't hot-updatable configurations

### Operations Understanding Focus
Environment variable injection differs from file mounting, so pay special attention to this:

> **After the application process starts, environment variables are typically not dynamically updated to the process.**

So often, even if the ConfigMap changes, the running process may not immediately perceive it.

---

## Eighteen. Why Pod Rebuilds Are Often Needed After Environment Variable Injection Updates

Because environment variables are usually injected into the process at container startup.

That means:

- Pod starts by reading ConfigMap
- The process gets a copy of environment variables
- During process runtime, these values typically don't auto-update

So if you modify ConfigMap values, the common approach is usually:

- Rebuild Pod
- Or use rolling update to let new Pod use new variables

### Operations Understanding Focus
This is similar to what was discussed earlier in `subPath`, and you should form a unified awareness:

> **"Configuration object changed" and "running application has adopted new configuration" are not the same thing.**

---

## Nineteen. What's the Relationship Between This Topic and Secret

ConfigMap and Secret have very similar injection methods, both can:

- Through `env`
- Through `envFrom`
- Through file mounting

But the core difference lies in the content they carry:

### ConfigMap
Suitable for:

- Non-sensitive configurations
- Ordinary text parameters
- Environment identifiers
- Service addresses

### Secret
Suitable for:

- Passwords
- Tokens
- Keys
- Certificate-related sensitive information

### Operations Understanding Focus
Injection methods can be similar, but security boundaries must not be mixed.

---

## Twenty. What Can This Topic Naturally Extend To

After completing this section, you can naturally expand to the following directions:

### 1. Secret as Environment Variable Injection
Incorporate sensitive configurations into the complete system.

### 2. `env` and `envFrom` Mixed Use
Some variables precisely controlled, others batch imported.

### 3. ConfigMap + Rolling Update
Address configuration change effectiveness issues.

### 4. Applications Reading Ports and Service Addresses from Environment Variables
Closer to real business containerization scenarios.

### 5. Mixed Design of File Mounting and Environment Variable Injection
For example:

- Nginx uses file mounting
- Java/Python applications use environment variables

---

## Twenty-one. The Most Important Cognitive Points in This Topic

### 1. ConfigMap Isn't Only for File Mounting
It's also commonly used for environment variable injection.

### 2. `env` and `envFrom` Can Both Inject Variables from ConfigMap
But with different granularities.

### 3. File Mounting and Environment Variable Injection Have No Absolute Advantages or Disadvantages; It Depends on How the Application Reads
Choose based on the program's behavior.

### 4. Environment Variables Are More Suitable for Small Key-Value Configurations
Not suitable for carrying complex large text configurations.

### 5. After ConfigMap Updates, Running Processes Typically Don't Automatically Get New Environment Variables
This is very important.

---

## Twenty-two. What Level Should You Master to Learn This Topic

At this stage, it's recommended to reach the following levels:

### 1. Understand the basic purpose of ConfigMap as environment variable injection
### 2. Distinguish between `env` and `envFrom`
### 3. Judge which configurations are more suitable for environment variable injection
### 4. Distinguish the application scenarios of environment variable injection and file mounting
### 5. Make basic judgments about "ConfigMap changed but the program didn't take effect"

---

## Twenty-three. Common Interview Extensions for This Topic

Common questions in this area include:

- What are the injection methods for ConfigMap?
- What's the difference between `env` and `envFrom`?
- What configurations are suitable for environment variable injection?
- How to choose between file mounting and environment variable injection?
- Why didn't the program immediately take effect after ConfigMap changes?
- What's the difference between ConfigMap and Secret?
- Why do many cloud-native applications prefer reading configurations through environment variables?

---

## Twenty-four. Stage Summary

ConfigMap as environment variable injection is another core line in Kubernetes configuration management.

The most important part of this article isn't memorizing syntax, but establishing the following core understandings:

- ConfigMap can be mounted as a file or injected as environment variables
- Environment variable injection is particularly suitable for small key-value configurations
- `env` is suitable for precise injection, `envFrom` is suitable for batch injection
- Whether to choose environment variable injection depends on how the program reads configurations
- After ConfigMap updates, running applications typically don't automatically get new environment variables

As long as these boundaries are clearly understood, it will be much smoother to continue learning Secret, configuration update strategies, and application startup parameter design later on.

---

## Twenty-five. Keyword Quick Recall

- ConfigMap: Kubernetes object for storing non-sensitive configurations
- Environment Variable Injection: Passing configurations to processes in key-value form
- env: Precisely inject single or few variables
- envFrom: Batch import entire ConfigMap
- Configuration Decoupling: Separating variable configurations from images
- File Mounting: Mounting configurations as files into containers
- Key-Value Configuration: Small configuration items suitable for environment variable injection

## 26. Operational Extension Understanding

From an operations perspective, environment variable injection is closer to the actual configuration methods of many modern applications compared to file mounting.

Especially in microservices, Web APIs, and backend services, common programs often natively support reading from environment variables:

- Environment identifier
- Port
- Upstream service address
- Switch items
- Log level

Therefore, mastering ConfigMap's environment variable injection isn't just learning a YAML writing method, but understanding:

> **How applications receive runtime parameters in cloud-native environments in a lighter, more standardized way.**

This will directly affect your depth of understanding for containerizing business services like Java, Python, Go, and Node.js.

---

## References
- Kubernetes ConfigMap: https://kubernetes.io/docs/concepts/configuration/configmap/
- Define Environment Variables for a Container: https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/
- Configure a Pod to Use a ConfigMap: https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/

---

## Next Day's Recommendation
Next post recommendation: 

[[04-Secret Introduction to Sensitive Configuration Management]]