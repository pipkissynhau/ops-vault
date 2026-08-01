# 04-Secret Basics: Introduction to Sensitive Configuration Management

## Document Notes
- Document Focus: Introduction to Kubernetes sensitive configuration management
- Applicable Stage: After completing ConfigMap file mounting and environment variable injection, proceed to learn Secret basics and usage boundaries
- Recommended Path: `04-Kubernetes/07-Apply deployment/05-Apply Configuration and Key Management/04-Secret Foundation: Introduction to Sensitive Configuration Management`

## Tags
#Kubernetes #Secret #SensitiveConfiguration #ConfigurationManagement #KeyManagement #EnvironmentalVariables #FileMount #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## Why Learn Secrets After ConfigMap

Previously, we've learned two core use cases of ConfigMap:

- As file mounting
- As environment variable injection

This is sufficient to solve many **non-sensitive configuration** issues, such as:

- Page content
- Nginx configuration
- Log level
- Ordinary port parameters
- Service address

But in real business scenarios, configurations are not only "ordinary configurations", there's also a more important category:

> **Sensitive configuration.**

Examples include:

- Database usernames and passwords
- Redis password
- API Token
- Access Key / Secret Key
- TLS certificate
- Private key
- OAuth credentials
- Third-party service authentication information

If such content is still placed in ConfigMap or directly written into the image like ordinary configurations, it would bring obvious risks:

### 1. Large exposure surface
Ordinary configuration objects are easier to be directly viewed and spread.

### 2. Doesn't meet the basic security layering
Sensitive configuration and ordinary configuration should have clear boundaries.

### 3. Inconsistent operation management
Subsequent permission control, auditing, rotation, and replacement will become more chaotic.

Therefore, Kubernetes needs a dedicated object to carry sensitive configurations:

> **Secret**

---

## What is a Secret

Secret is an object in Kubernetes used to store **sensitive data**.

It is similar to ConfigMap in that both can be used by Pods through the following methods:

- As environment variable injection
- As file mounting to containers

But the biggest difference between Secret and ConfigMap is:

> **Secret's design goal is to carry sensitive configurations, not ordinary configurations.**

### Typical Use Cases
- Password
- Token
- Key
- Certificate
- Authentication information

### Simplified Understanding
- **ConfigMap**: Ordinary configuration
- **Secret**: Sensitive configuration

---

## Why Can't Sensitive Information Be Written Directly in the Image

This is a critical security boundary.

If passwords, keys, and tokens are directly written into the image, it usually brings the following issues:

### 1. Sensitive information spreads with the image once pulled
After the image is pulled by multiple nodes and environments, sensitive data will spread along with it.

### 2. Changing passwords requires rebuilding the image
For example, after the database password changes, you need to rebuild, push, and update the image, which is very inflexible.

### 3. Not conducive to permission isolation
The program image itself and sensitive configurations should be separated, otherwise it will blur the access boundary.

### 4. Not conducive to auditing and rotation
Sensitive information management should be as independent as possible to facilitate future permission convergence and key rotation.

### Operation Understanding Focus
Images should carry:

- Program itself
- Runtime environment
- Stable dependencies

And should not long-term carry:

- Passwords
- Certificates
- Tokens
- Private keys

---

## What's the Core Difference Between Secret and ConfigMap

This is a high-frequency question in interviews and practical work.

### 1. Different Carried Content
#### ConfigMap
Suitable for:

- Ordinary text configuration
- Environment identifiers
- Page content
- Nginx configuration
- Non-sensitive parameters

#### Secret
Suitable for:

- Username and password
- Token
- Certificate
- Private key
- Access Key

### 2. Different Security Semantics
Even though both have similar usage methods, Secret's core semantics clearly lean more towards:

> **Sensitive data management.**

### 3. Different Permission Boundaries
In real environments, ConfigMap and Secret often should have different access control approaches.

### 4. Different Operation Governance Requirements
Secret is more suitable for inclusion in:

- Permission control
- Auditing
- Rotation
- Encrypted storage
- Principle of least privilege

### Simplified Memory
- **ConfigMap manages ordinary configuration**
- **Secret manages sensitive configuration**

---

## What Are the Common Types of Secret

You don't need to memorize all types at this stage, but you should have a basic understanding.

### 1. `Opaque`
The most common and general Secret type.  
Most custom sensitive key-value pairs can first use this type.

### 2. `kubernetes.io/dockerconfigjson`
Commonly used for image repository authentication, such as private repository pull credentials.

### 3. `kubernetes.io/tls`
Commonly used to store TLS certificates and private keys.

### Current Stage Recommendation
Focus first on the most common:

> **Opaque**

Because it's the most suitable as an introductory type.

---

## A Simple Secret Example

Here's a basic Secret example:

    apiVersion: v1
    kind: Secret
    metadata:
      name: app-secret
    type: Opaque
    stringData:
      DB_USER: admin
      DB_PASSWORD: strongpassword123

---

## How to Understand This Secret

### 1. `kind: Secret`
Indicates this is a Secret object.

### 2. `metadata.name`
This Secret's name is:

    app-secret

### 3. `type: Opaque`
Indicates this is the most common general Secret type.

### 4. `stringData`
Here the raw plaintext form is written, Kubernetes will convert it to the corresponding internal format during processing.

### Operation Understanding Focus
When learning and writing YAML, `stringData` is often more intuitive than directly writing encoded content.

---

## Why Use `stringData` Instead of Directly Writing `data` Here

Because `stringData` is more suitable for learning and daily maintenance.

### Characteristics of `stringData`
- Intuitive writing
- No need for manual encoding conversion
- More suitable for handwritten YAML and understanding configuration content

### Characteristics of `data`
- Usually requires writing processed values
- Slightly worse readability
- Less suitable as the first contact point for beginners

### Operation Understanding Focus
At this stage, remember:

> **When learning and writing Secret YAML, prioritize understanding `stringData` first.**

Understand `data` form gradually later.

---

## How to Inject Secret into Pod

Similar to ConfigMap, Secret commonly has two injection methods:

### 1. Inject as environment variables
Suitable for: /think

- Username
- Password
- Token
- Small sensitive parameters

### 2. As File Mount
Suitable for:

- Certificates
- Private keys
- kubeconfig
- Sensitive configurations that need to be read as files

---

## 10. Example of Injecting Secret as Environment Variables

Here is the simplest example of environment variable injection:

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
                - name: DB_USER
                  valueFrom:
                    secretKeyRef:
                      name: app-secret
                      key: DB_USER
                - name: DB_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: app-secret
                      key: DB_PASSWORD

---

## 11. Understanding This Environment Variable Injection

### 1. `env`
Indicates the definition of container environment variables.

### 2. `valueFrom.secretKeyRef`
Indicates the variable value comes from Secret, not from ConfigMap or hardcoded values.

### 3. `name`
Indicates the actual environment variable name visible in the container.

### Final Effect
The container will receive:

- `DB_USER=admin`
- `DB_PASSWORD=strongpassword123`

### Operations Understanding Focus
This is very similar to the usage of ConfigMap's `env`, but with different semantics:

- ConfigMap: Ordinary configuration
- Secret: Sensitive configuration

---

## 12. Example of Mounting Secret as a File

Here is a basic mounting example:

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
              volumeMounts:
                - name: app-secret-volume
                  mountPath: /etc/secret
                  readOnly: true
          volumes:
            - name: app-secret-volume
              secret:
                secretName: app-secret

---

## 13. Understanding This Mounting Syntax

### 1. `volumes.secret.secretName`
Indicates which Secret the volume data comes from.

### 2. `volumeMounts.mountPath`
Indicates where the Secret content is mounted in the container.

### 3. `readOnly: true`
Indicates the mount is read-only.  
This is a reasonable default approach for sensitive configurations.

### Final Effect
Each key in the Secret will typically appear as a file in:

    /etc/secret/

directory.

For example:

- `/etc/secret/DB_USER`
- `/etc/secret/DB_PASSWORD`

---

## 14. When to Use Environment Variables vs File Mounts

### Scenarios Better for Environment Variables
Usually include:

- Database username
- Database password
- API Token
- Simple authentication parameters

### Scenarios Better for File Mounts
Usually include:

- TLS certificates
- Private key files
- Sensitive configurations needing file path access
- kubeconfig-like files

### Simplified Understanding
- **Small sensitive key-value pairs**: Better for environment variables
- **File-type sensitive content**: Better for file mounts

---

## 15. Why Secret Cannot Be "Deified"

This needs to be clearly explained.

Many beginners mistakenly believe:

> "Putting something in Secret makes it absolutely safe."

This understanding is inaccurate.

### The Meaning of Secret
- Clearer semantic layering
- Separation of configuration from image
- More suitable for permission control and governance

### But It Does Not Mean
- No need for permission control
- No need for least privilege principle
- No need for auditing
- No need for node security and access control

### Operations Understanding Focus
Secret is a **more suitable carrier location**, not "safe just by putting it in."

---

## 16. Common Issues When Using Secret

### 1. Putting All Ordinary Configurations into Secret
This blurs the boundary between ordinary and sensitive configurations, making management harder.

### 2. Putting Sensitive Configurations Hardcoded in Image
This is a high-risk practice already discussed earlier.

### 3. Typing Variable Names or Key Names Wrong
Causes the application to not read the Secret.

### 4. Secret Changed, but Application Not Updated
Like ConfigMap, many running processes won't automatically get new values.

### 5. Directly Making Complex Configurations into Environment Variables
Makes readability and maintainability worse.

---

## 17. Will the Application Immediately Reflect Secret Changes?

This understanding should align with ConfigMap's.

### If Injected via Environment Variables
Usually need special attention:

> **Environment variables generally do not auto-refresh after process startup.**

So common practice remains:

- Rebuild Pod
- Or use rolling update to let new Pod use new Secret

### Mounting as a File
In some scenarios, files may be updated, but whether the application is aware of these changes depends on whether the program supports re-reading.

### Operations Understanding Focus
Never consider the following as the same thing:

- "The Secret object has been changed"
- "The application has already used the new value"

---

## Eighteen. Why Is the Image Repository Pull Credential Also a Secret

This is a practical point you will encounter soon.

When Kubernetes needs to pull an image from a private repository, it typically uses authentication information such as:

- Repository username
- Repository password
- Token

This type of content clearly belongs to sensitive data, so Kubernetes also stores them using Secret.

This is why Secret is not only used for business programs but also commonly used for platform capabilities, such as:

- Private image pull authentication
- TLS certificate
- Access credentials for third-party systems

---

## Nineteen. How to Remember the Relationship Between Secret and ConfigMap

It is recommended to remember it as the following system:

### ConfigMap
Responsible for:

- Ordinary configuration
- Non-sensitive parameters
- Textual ordinary content
- Page and ordinary configuration files

### Secret
Responsible for:

- Sensitive configuration
- Password
- Token
- Key
- Certificate
- Authentication information

### Common Points
- Both can be injected into Pod
- Both can be used as environment variables
- Both can be mounted as files

### Essential Differences
- Different sensitivity levels of configuration content
- Different operation governance requirements

---

## Twenty. What Can This Topic Naturally Extend To

After completing this article, you can naturally expand to these directions:

### 1. Secret as Image Pull Credential
This will be very close to your subsequent Kubernetes practical work.

### 2. Secret Certificate Mounting
For example, Nginx TLS configuration.

### 3. Mixed Injection of ConfigMap and Secret
Managing ordinary configuration and sensitive configuration together.

### 4. Configuration Update and Rolling Release
Including the effectiveness strategy after ConfigMap and Secret changes.

### 5. Permission Control and Principle of Least Privilege
Gradually entering a more standardized security governance mindset.

---

## Twenty-One. The Most Important Understandings in This Topic

### 1. Secret is the standard object in Kubernetes for carrying sensitive configurations
It is not a duplicate of ConfigMap, but a layered object on the security boundary.

### 2. Sensitive information should not be directly written into the image
This is a fundamental security principle in containerization and cloud-native deployment.

### 3. Secret can be injected into Pod like ConfigMap
Including:

- Environment variable method
- File mounting method

### 4. Secret is suitable for "sensitive data," but does not mean absolute security automatically
It still requires permission control, access boundaries, and audit awareness.

### 5. Whether the application is effective after Secret changes depends on the injection method and application behavior
This point must be connected with the previous understanding of ConfigMap.

---

## Twenty-Two. What Level Should You Master to Learn This Article

At this stage, it is recommended to reach the following level:

### 1. Understand why Secret is needed instead of using ConfigMap entirely
### 2. Distinguish the usage boundaries between Secret and ConfigMap
### 3. Understand a simple Secret YAML
### 4. Understand the two ways of injecting Secret into Pod: environment variables and file mounting
### 5. Judge which content should be placed in Secret
### 6. Make a basic judgment on "Secret changed but the application did not take effect"

---

## Twenty-Three. Common Interview Extensions

Common questions in this area include:

- What is the difference between Secret and ConfigMap
- Why is it not recommended to put passwords in ConfigMap
- How can Secret be injected into Pod
- What types of configuration are suitable for Secret
- Why the application did not take effect immediately after Secret changes
- Why image pull credentials are stored in Secret
- Why certificates are usually also managed by Secret
- Does Secret mean absolute security

---

## Twenty-Four. Stage Summary

Secret is the core object in Kubernetes' configuration management system for handling sensitive configurations.

The most important thing in this article is not memorizing syntax, but establishing the following core understandings:

- Ordinary configuration and sensitive configuration must be managed in layers
- Secret is responsible for carrying sensitive data
- Secret can be injected into Pod through environment variables or file mounting
- Passwords, Tokens, keys should not be hard-coded in images long-term
- Whether the application takes effect after Secret changes still needs to be judged based on injection method and process behavior

As long as these boundaries are clearly understood, it will be smoother to continue learning image pull credentials, TLS certificate management, and mixed design of ConfigMap and Secret later.

---

## Twenty-Five. Keyword Quick Notes

- Secret: The object in Kubernetes for storing sensitive configurations
- Opaque: The most common general Secret type
- stringData: The original text field of Secret for convenient writing
- secretKeyRef: Taking values from Secret to inject environment variables
- secret volume: Mounting Secret as a file to the container
- Sensitive configuration: Passwords, Tokens, certificates, keys, etc.
- Configuration layering: Managing ordinary configuration and sensitive configuration separately

---

## Twenty-Six. Operational Extended Understanding

From an operations perspective, the value of Secret is not just "being able to store passwords," but pushing configuration management into more standardized layered governance.

Many teams often go through this stage when starting containerization:

- All configurations are written into the image
- Then they find it difficult to maintain
- Then they move all things into ConfigMap
- Finally, they find the sensitive information boundary is messy

A truly mature approach is usually to gradually establish this awareness:

> **The image carries the program, ConfigMap carries ordinary configuration, and Secret carries sensitive configuration.**

After clearly distinguishing these three layers, subsequent environment governance, release processes, security controls, and credential rotation can more easily move toward standardization.

---

## References
- Kubernetes Secret: https://kubernetes.io/docs/concepts/configuration/secret/
- Distribute Credentials Securely Using Secrets: https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/
- Kubernetes Volumes: https://kubernetes.io/docs/concepts/storage/volumes/

---

## Next Day Recommendation
Next article suggestion to organize:

[[05 ConfigMap and Secret Difference Boundary and Selection Principles]]