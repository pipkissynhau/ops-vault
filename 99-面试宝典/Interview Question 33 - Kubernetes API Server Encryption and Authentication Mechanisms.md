# Interview Question 33: Kubernetes API Server Encryption and Authentication Mechanisms

#kubernetes #apiserver #Clear. #Interviews

---

## I. Core Summary (One-Sentence Interview Version)

Kubernetes API Server's security mechanisms can be divided into two categories:

1. **Transport Encryption (TLS)**: Ensures data security during communication
    
2. **Data Encryption (Encryption at Rest)**: Ensures security of data stored in etcd
    

Combined with:

- Authentication
    
- Authorization
    
Forms a complete security system.

---

## II. Transport Encryption (TLS/HTTPS)

### 1. Basic Principles

All communication with API Server defaults to **HTTPS (TLS)** encryption:

- kubectl → API Server
    
- kubelet → API Server
    
- scheduler / controller → API Server
    
👉 Essence:

> Establishes an encrypted channel using TLS certificates, ensuring data confidentiality and integrity

---

### 2. Certificate System (PKI)

Kubernetes uses **CA (Certificate Authority)** to manage certificates:

- CA is responsible for issuing certificates
    
- Each component holds its own certificate
    
Common certificates:

- apiserver.crt / key
    
- kubelet client cert
    
- admin (kubectl) certificate
    
---

### 3. Mutual TLS

API Server typically uses **mutual TLS**:

- Clients verify API Server (prevents man-in-the-middle attacks)
    
- API Server verifies client identity
    
👉 Result:

> Encrypts data and confirms "who is accessing"

---

## III. Authentication

After receiving a request, API Server first performs authentication:

### Common Authentication Methods:

- Certificate authentication (most common)
    
- Token (ServiceAccount)
    
- OIDC (integrates with external identity systems)
    
- Basic Auth (no longer recommended)
    

---

👉 Purpose:

> Confirms "who you are"

---

## IV. Authorization

After authentication, the system enters the authorization phase:

### Common Methods:

- RBAC (most commonly used)
    
- ABAC (less commonly used)
    
- Webhook
    
---

👉 Purpose:

> Confirms "what you can do"

---

## V. Data Encryption (etcd Static Encryption)

### 1. Background

API Server data is ultimately stored in etcd.

By default:

> ❗etcd data may be plaintext (especially Secrets)

---

### 2. Solution: Encryption at Rest

Configure encryption policies via API Server:

👉 **EncryptionConfiguration**

---

### 3. Common Encryption Methods

|Provider|Description|
|---|---|
|aescbc|Common, symmetric encryption|
|secretbox|Based on NaCl|
|kms|Integrates with external| /think