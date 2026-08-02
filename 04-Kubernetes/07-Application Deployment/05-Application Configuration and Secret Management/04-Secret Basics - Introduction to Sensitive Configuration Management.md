# 04-Secret Basics: Introduction to Sensitive Configuration Management

## Document Description
- Document Purpose: Introduction to Kubernetes sensitive configuration management
- Applicable Phase: After completing ConfigMap file mounting and environment variable injection, move on to understanding and using Secrets at a basic level
- Recommended Path: `04-Kubernetes/07-Application Deployment/05-Application Configuration and Key Management/04-Secret Basics: Introduction to Sensitive Configuration Management`

## Tags
#Kubernetes #Secret #Sensitive Configuration #Configuration Management #Key Management #Environment Variables #File Mounting #Application Deployment #Business Containerization #Cloud Native #Ops #Interview Notes

---

## I. Why Learn Secrets After ConfigMap

Previously, we learned about the two main uses of ConfigMap:

- As a file mount
- As an environment variable injection

These are sufficient to handle many **non-sensitive configurations**, such as:

- Page content
- Nginx configuration
- Log levels
- Ordinary port parameters
- Service addresses

However, in real-world applications, configurations include not only **ordinary settings** but also a more critical category:

> **Sensitive configurations.**

Examples include:

- Database usernames and passwords
- Redis passwords
- API Tokens
- Access Keys / Secret Keys
- TLS certificates
- Private keys
- OAuth credentials
- Authentication information for third-party services

If sensitive data is stored in ConfigMap or directly within the image, it poses significant risks:

### 1. Excessive exposure
Ordinary configuration objects are easily viewed and shared.

### 2. Violation of basic security principles
Sensitive and ordinary configurations should be clearly separated.

### 3. Disorganized Ops management
Subsequent permission control, auditing, rotation, and replacement become much more complicated.

Therefore, in Kubernetes, a dedicated object is needed to store sensitive configurations:

> **Secret**

---

## II. What is a Secret

A Secret is an object in Kubernetes used to store **sensitive data**.

It functions similarly to ConfigMap and can be used by Pods in the following ways:

- As an environment variable injection
- As a file mount within the container

However, the key difference between them is:

> **Secrets are designed specifically for storing sensitive configurations, not ordinary settings.**

### Typical Uses
- Passwords
- Tokens
- Keys
- Certificates
- Authentication information

### Simplified Understanding
- **ConfigMap**: Ordinary configurations
- **Secret**: Sensitive configurations

---

## III. Why Not Store Sensitive Information Directly in the Image

This is a crucial security consideration.

If passwords, keys, or tokens are written directly into the image, it usually leads to the following issues:

### 1. Widespread dissemination of sensitive data
Once an image is pulled across multiple nodes and environments, sensitive information is also spread.

### 2. Inflexible password management
Changing a database password requires rebuilding, pushing, and updating the entire image, which is very inefficient.

### 3. Impaired permission isolation
The program image itself and sensitive configurations should be separated to maintain clear access boundaries.

### 4. Difficulty in auditing and rotation
Sensitive information should be managed independently to facilitate future permission management and key rotations.

### Ops Consideration
Images should primarily contain:

- The program code itself
- Runtime environment settings
- Stable dependencies

Rather than:

- Passwords
- Certificates
- Tokens
- Private keys

---

## IV. What Are the Key Differences Between Secrets and ConfigMap?

This is a frequently asked question in interviews and practical scenarios.

### 1. Different Types of Data Stored
#### ConfigMap
Suitable for:

- Ordinary text configurations
- Environment identifiers
- Page content
- Nginx configuration
- Non-sensitive parameters

#### Secret
Suitable for:

- Usernames and passwords
- Tokens
- Certificates
- Private keys
- Access Keys

### 2. Different Security Focuses
Even though their usage is similar, Secrets are explicitly designed for **sensitive data management**.

### 3. Different Permission Frameworks
In real-world environments, ConfigMap and Secrets often require different access control approaches.

### 4. Different Ops Management Practices
Secrets are better suited for:

- Permission control
- Auditing
- Rotation
- Encrypted storage
- Adherence to the principle of least privilege

### Simplified Memory Aid
- **ConfigMap manages ordinary configurations**
- **Secrets manage sensitive configurations**

---

## V. What Are Some Common Types of Secrets?

There’s no need to memorize all types at this stage, but it’s important to have a basic understanding.

### 1. `Opaque`
The most common and versatile type of Secret.
Most custom sensitive key-value pairs can be stored using this type.

### 2. `kubernetes.io/dockerconfigjson`
Often used for image repository authentication, such as pulling credentials from private repositories.

### 3. `kubernetes.io/tls`
🔤image: harbor.example.com/demo/app:v1  
volumeMounts:  
  - name: app-secret-volume  
    mountPath: /etc/secret  
    readOnly: true  
volumes:  
  - name: app-secret-volume  
    secret:  
      secretName: app-secret