# 01-PVC and StorageClass Basics: Introduction to Persistent Storage

## Document Description
- Document Purpose: An introductory guide to persistent storage in Kubernetes
- Applicable Phase: After completing the basics of stateless applications, configuration management, health checks, resource management, and traffic exposure, this document serves as a foundation for learning about persistent storage before moving on to stateful applications
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/03-Stateful Application Deployment/01-PVC and StorageClass Basics: Introduction to Persistent Storage.md`

## Tags
#Kubernetes #PVC #PV #StorageClass #PersistentStorage #StatefulApplications #ApplicationDeployment #CloudNative #Ops #InterviewNotes

---

## I. Why Learn PVC and StorageClass Now

The previous topics have mainly focused on stateless applications:

- Image building
- Deployment
- Service / NodePort / Ingress
- ConfigMap / Secret
- Probe
- requests / limits

These aspects are sufficient to run many stateless services. However, when dealing with the following types of components, things become significantly different:

- MySQL
- Redis
- Nacos
- PostgreSQL
- MinIO
- Elasticsearch
- Kafka

These services share one common characteristic:

> **They often cannot rely solely on temporary container layers for their operation.**

In other words, they require more than just “starting processes”; they also need:

- Data to be retained
- Data to be available after the Pod is re-created
- The ability for the application to continue using the same storage resources
- To prevent data loss when containers are terminated

Therefore, starting from this section, we will formally delve into:

> **Persistent storage.**

In Kubernetes, the key components related to persistent storage are:

- PV (Persistent Volume)
- PVC (PersistentVolumeClaim)
- StorageClass

---

## II. What is “Persistent Storage”

Let’s start with the most fundamental concept.

### Issues with Container Default Storage
Containers typically have a writable layer during runtime, but this data has significant limitations:

- Data may be lost when the container is deleted.
- Data may not be restored after the Pod is re-created.
- It is not suitable as a long-term storage solution for databases, caches, or file services.

### What is Persistence
Persistence can be simply understood as:

> **Even if a Pod or container is deleted and recreated, critical data remains intact.**

### Typical Use Cases
For example:

- MySQL data directories
- Redis persistent files
- MinIO object storage data directories
- Nacos persistent configurations or dependency stores
- Application upload directories

---

## III. Why Stateless Applications Usually Do Not Rely Much on PVC

Take Nginx static pages as an example. In previous exercises, the page content came from:

- Images
- Or ConfigMap

Such content typically has the following characteristics:

- It can be replaced easily.
- Data can be restored after re creation.
- It does not depend on runtime local state.

Therefore, these applications tend to be stateless.

### But Databases and Middleware Are Different
For instance, if data is stored in a container’s temporary layer for MySQL:

- If the container is deleted and the Pod is re-created, the data will likely be lost.

This is clearly unacceptable.

### Key Points for Ops Professionals
Therefore:

- **Stateless applications**: Focus more on Deployment, Service, and Probe.
- **Stateful applications**: Also require attention to persistent storage.

---

## IV. What Are PV, PVC, and StorageClass?

Here is a simplified explanation of each component.

### 1. PV (PersistentVolume)
PV is:

> **PersistentVolume, a persistent storage resource abstraction in Kubernetes clusters.**

It represents more of the following:

- The storage itself
- The result of the storage supply process
- A volume resource that already exists or has been prepared

---

### 2. PVC (PersistentVolumeClaim)
PVC is:

> **PersistentVolumeClaim, a request for storage resources submitted by an application.**

It represents more of the following:

- How much storage I need.
- What access mode I want.
- A request for a usable PersistentVolume.

---

### 3. StorageClass
StorageClass is:

> **A set of rules that define how storage should be supplied.**

It can be understood as:

- Rules for storing resources
- Storage templates
- Policies for dynamically creating storage resources

It typically determines:

- Which backend storage to use.
- What parameters to apply when creating volumes.
- How dynamic storage allocation should be managed.

---

## V. Summarizing the Relationship Between PV, PVC, and StorageClass in One Sentence

You can remember this:

> **PVC makes a request, StorageClass defines the rules, and PV provides the actual storage resource.**

Or, in more colloquial terms:

- PVC: I want to request a piece of storage.
- StorageClass: How should this storage be created for me?
-### 🔤 App: demo-app
#### Spec:
```yaml
containers:
  - name: demo-app
    image: harbor.example.com/project/demo-app:v1
    volumeMounts:
      - name: data-volume
        mountPath: /data
volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: demo-pvc
```

---

## **Chapter Sixteen: How to Understand This Mount Relationship**

### 1. **`volumes`**
It defines the source of the volume.

This is neither a `ConfigMap` nor a `Secret`, but rather a `persistentVolumeClaim` with the name `demo-pvc`.

This indicates that:

> The volume comes from a PVC named `demo-pvc`.

### 2. **`volumeMounts`**
It mounts this volume to the container’s `/data` directory.

### **Final Effect**
The `/data` directory inside the container is backed by persistent storage, not just temporary container space.

### **Key Points for Ops Professionals**
This means that:

- After a Pod is recreated,
- as long as the PVC and the underlying volume relationship remain intact,
- critical data in `/data` will likely be preserved.

---

## **Chapter Seventeen: Why PVCs Are a Critical Foundation for Stateful Applications**

Most stateful services rely on “stable data directories”.

### **Examples**
- **MySQL**: Requires persistent storage for:
  - Data files
  - Binlogs
  - Table structures
  - System libraries
- **Redis**: Requires persistent storage for RDB and AOF files.
- **MinIO**: Requires persistent storage for object files.

### **Consequences Without PVCs**
Even if a Pod starts successfully without a PVC, issues may arise such as:

- Data loss after reconstruction
- Inability to maintain stable service states
- Failure to meet production requirements

---

## **Chapter Eighteen: Why This Is an Important Preparatory Step Before Deploying Stateful Applications**

Once you start working with:

- `StatefulSets`
- MySQL
- Redis
- Nacos
- MinIO,

you can’t just focus on:

- Images
- Deployments
- Services.

You must also check:

- Whether storage has been successfully bound
- Whether the PVC is Bound
- Whether the `StorageClass` exists
- Whether the data directories have actually been mounted.

Therefore, PVCs and `StorageClasses` are essential foundations before deploying stateful applications.

---

## **Chapter Nineteen: What Are the Most Common Basic Issues?**

### 1. **PVC Remaining in “Pending” State**
This is one of the most common storage-related issues.

### 2. **Pods Failing to Start or Getting Stuck**
This often happens because the PVC they depend on has not been successfully bound.

### 3. **Volume Is Mounted, but the Application Still Doesn’t Work**
Possible reasons include:

- Incorrect mount path
- The application is not writing data to the volume
- The application is still using temporary container storage

### 4. **Data Not Being Preserved**
Possible causes include:

- Not actually using a PVC
- Using a temporary directory instead
- Incorrect mount path

---

## **Chapter Twenty: What to Check First When Troubleshooting Storage Issues**

Follow this basic order of steps:

### 1. Check the Status of the PVC
Focus on whether it is in “Pending” or “Bound” state.

### 2. Verify Whether the `StorageClass` Exists
If a PVC specifies an non-existent `StorageClass`, it will likely remain in “Pending” status.

### 3. Check if the Access Mode and Capacity Are Reasonable
For example:

- Requesting 100 GiB of storage when the backend actually has less
- Requesting RWX access when the backend does not support it

### 4. Verify Whether the Pod Has Successfully Mounted the Volume
Just because a PVC is Bound doesn’t mean the application is writing data to the correct path.

### 5. Finally, Check whether the Application’s Data Directories Are Actually Located on the Mounted Volume
This step is crucial.

---

## **Chapter Twenty-One: Why Are StorageClasses So Important?**

In many modern Kubernetes clusters, PVCs are often not created manually by provisioning PVs. Instead, it is more common to:

> **Dynamically supply storage through StorageClasses.**

This means that:

- Business teams only need to specify PVCs.
- The platform automatically creates PVs based on the specified `StorageClass`.
- The binding process is automated.
- Access to underlying storage is also automated.

### **Key Points for Ops Professionals**
The value of `StorageClasses` lies in their ability to provide:

- Automation
- Standardization
- Orchestration

These features are crucial for a platform-based Kubernetes environment.

---

## **Chapter Twenty-Two: The Most Important Concepts in This Topic**

### 1. Temporary Container Storage Is