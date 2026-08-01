# 01-PVC and StorageClass Basics: Introduction to Persistent Storage

## Document Notes
- Document Focus: Kubernetes Persistent Storage Fundamentals
- Applicable Stage: After completing stateless application, configuration management, health checks, resource management, and traffic exposure basics, before entering stateful application storage fundamentals
- Recommended Path: `04-Kubernetes/07-Apply deployment/03-Apply deployment in status/01-PVC and StorageClass Foundation: Sustainable storage entry.md`

## Tags
#Kubernetes #PVC #PV #StorageClass #EnduringStorage #ApplyWithStatus #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Why Learn PVC and StorageClass Now

The previous mainline has mostly focused on stateless applications:

- Image building
- Deployment
- Service / NodePort / Ingress
- ConfigMap / Secret
- Probe
- requests / limits

These contents are already sufficient to run many stateless services.  
But once you start dealing with these objects, the problems become completely different:

- MySQL
- Redis
- Nacos
- PostgreSQL
- MinIO
- Elasticsearch
- Kafka

Because these services share a common characteristic:

> **They often cannot survive solely on the container's temporary layer.**

In other words, they are not just about "processes running", but also require:

- Data persistence
- Data remaining after Pod reconstruction
- Applications using the same storage continuously
- Data not being lost when containers are destroyed

From this point onward, we officially enter:

> **Persistent storage.**

The most core objects in Kubernetes are:

- PV
- PVC
- StorageClass

---

## II. What is "Persistent Storage"

Start with the most fundamental concept.

### Issues with Container Default Storage
Containers typically have a writable layer, but this layer has obvious limitations:

- Data may be lost after container deletion
- Data may not return after Pod reconstruction
- Not suitable as long-term data carriers for databases, caches, or file services

### What is Persistence
Persistence can be simply understood as:

> **Critical data remains even if Pods or containers are deleted or rebuilt.**

### Typical Scenarios
For example:

- MySQL data directory
- Redis persistence files
- MinIO object storage data directory
- Nacos persistent configuration or dependency storage
- Application upload file directory

---

## III. Why Stateless Applications Typically Don't Rely on PVC

Take Nginx static pages as an example. The content in previous exercises comes from:

- Image
- Or ConfigMap

These types of content typically have characteristics:

- Replaceable
- Recoverable after reconstruction
- Not dependent on runtime local state

So it leans more toward stateless.

### But Databases and Middleware Are Different
For example MySQL, if data is written to the container's temporary layer:

- Container deleted
- Pod rebuilt
- Data may be lost

This is clearly unacceptable.

### Operations Understanding Focus
Therefore:

- **Stateless applications**: Focus more on Deployment, Service, Probe
- **Stateful applications**: Must also pay extra attention to persistent storage

---

## IV. What Are PV, PVC, and StorageClass

Start with the simplest understanding.

### 1. PV
PV is:

> **PersistentVolume, persistent volume.**

It can be understood as the "actual storage resource abstraction" in the Kubernetes cluster.

It leans more toward:

- Storage itself
- Storage supply result
- A storage volume resource that already exists or is prepared

---

### 2. PVC
PVC is:

> **PersistentVolumeClaim, persistent volume claim.**

It can be understood as the "storage application form" requested by the application side.

It leans more toward:

- How much storage I need
- What access mode I want
- I want a usable persistent volume

---

### 3. StorageClass
StorageClass is:

> **Storage class.**

It can be understood as:

- Storage supply rules
- Storage template
- Policy definition for dynamic storage creation

It usually determines:

- Which backend storage to use
- What parameters to use for creating volumes
- How dynamic provisioning is completed

---

## V. Link the Three Together in One Sentence

You can first remember this:

> **PVC makes the request, StorageClass defines the rules, PV delivers the result.**

Or understand it more casually:

- PVC: I want to apply for a storage
- StorageClass: How to create this storage
- PV: The volume resource finally allocated to me

---

## VI. Why Can't Pods Just "Mount Any Directory" and Be Done

In local experimental environments, many people develop a habit:

- Mount any host directory
- Or use temporary volume
- Feel it can work

But in the Kubernetes formal system, this approach has many problems:

### 1. Lifecycle Confusion
After Pod deletion, volume relationships may not be standardized.

### 2. Lack of Declarative Management
Cannot clearly express "what persistent storage resources this application needs".

### 3. Not Suitable for Scheduling and Automation
The platform finds it hard to uniformly manage volume application, binding, and recycling.

### 4. Not Suitable for Dynamic Provisioning
Cannot automatically create volumes through standardized methods.

### Operations Understanding Focus
Kubernetes designed PV/PVC/StorageClass to bring storage into:

> **A declarative, orchestratable, and governable system.**

---

## VII. What Does PVC Core Express

PVC's core expression is:

> **What kind of storage the application wants.**

Commonly declared content includes:

- How much space is needed
- What access mode is needed
- Whether to specify a StorageClass
- What type of persistent volume to bind

### Operations Understanding Focus
PVC is not the "real storage disk" itself, but more like:

> **An application's application form for storage.**

---

## VIII. What Does PV Core Express

PV's core expression is:

> **The actual persistent volume resources available for binding in the cluster.**

It generally describes:

- Capacity
- Access mode
- Storage type
- Recycling policy
- Backend information

### Operations Understanding Focus
From a perspective:

- PVC leans more toward the application side
- PV leans more toward the storage supply side

---

## IX. What Does StorageClass Core Express

StorageClass's core expression is:

> **When a PVC initiates a request, the system should follow what rules to dynamically create storage.**

In other words, StorageClass is more like "production line rules".

### Typical Problems It Solves
For example:

- This volume should be created from which storage backend
- What parameters to use
- What type of disk
- What recycling policy
- Whether dynamic expansion is allowed

### Operations Understanding Focus
Without StorageClass, you can still have static PV/PVC.  
But with StorageClass, the automation and standardization capabilities of storage will significantly improve.

---

## X. A Simplest PVC Example

Below is a basic PVC: /think

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard

---

## 11. Understanding This PVC

### 1. `kind: PersistentVolumeClaim`
Indicates this is a PVC object.

### 2. `metadata.name`
This PVC name is:

    demo-pvc

### 3. `accessModes`
Indicates the access mode this volume expects.

Here:

    ReadWriteOnce

Broadly understood as:

> Typically used for single-node read/write

### 4. `resources.requests.storage`
Indicates the requested storage size is:

    1Gi

### 5. `storageClassName`
Indicates which StorageClass rule to use for provisioning:

    standard

---

## 12. Understanding Common accessModes

We'll not dive too deep at this stage, just establish the basic understanding.

### 1. ReadWriteOnce (RWO)
Broad understanding:

> Typically suitable for single-node read/write mounting

This is very common in many database and middleware scenarios.

---

### 2. ReadOnlyMany (ROX)
Broad understanding:

> Readable by multiple nodes, but read-only

---

### 3. ReadWriteMany (RWX)
Broad understanding:

> Readable and writable by multiple nodes

This is more common in shared storage scenarios, such as certain NFS or distributed file system backends.

### Current Focus
Remember:

- `RWO` is the most common
- Frequently encountered in database scenarios
- `RWX` often requires higher backend storage capabilities

---

## 13. What Happens After PVC Creation

This depends on whether there's a matching PV or StorageClass in the environment.

### Case 1: Matching PV Exists
The PVC will bind to the appropriate PV.

### Case 2: StorageClass Exists and Supports Dynamic Provisioning
The system will automatically create a PV based on the StorageClass, then complete the binding.

### Case 3: Neither Condition Met
The PVC may remain in:

> **Pending**

This is also a frequent troubleshooting scenario later on.

---

## 14. What Is Bound vs. Pending

This is the most critical state understanding in PVC learning.

### 1. Pending
Indicates:

> **The PVC has not successfully bound to an available storage.**

Possible reasons include:

- No available PV
- StorageClass does not exist
- Dynamic provisioning failed
- Capacity or access mode mismatch

---

### 2. Bound
Indicates:

> **The PVC has successfully bound to a PV.**

This typically means:

- The request and provisioning have matched successfully
- The Pod can theoretically continue using this volume

### Operations Understanding Focus
If a PVC remains Pending, the subsequent Pod may be affected.

---

## 15. Simple Pod/Deployment PVC Mounting Example

Here's a minimal mounting snippet:

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
              image: harbor.example.com/project/demo-app:v1
              volumeMounts:
                - name: data-volume
                  mountPath: /data
          volumes:
            - name: data-volume
              persistentVolumeClaim:
                claimName: demo-pvc

---

## 16. Understanding This Mounting Relationship

### 1. `volumes`
Defines a volume source.

Here it's not a ConfigMap or Secret, but:

    persistentVolumeClaim:
      claimName: demo-pvc

Indicates:

> This volume comes from the PVC named `demo-pvc`.

### 2. `volumeMounts`
Mounts this volume to the container's:

    /data

### Final Effect
The directory `/data` inside the container corresponds to a persistent storage, not just the container's temporary layer.

### Operations Understanding Focus
This means:

- After Pod recreation
- As long as the PVC and backend volume relationship remains
- The critical data in `/data` can potentially be preserved

---

## 17. Why PVC Is a Critical Prerequisite for Stateful Applications

Because most stateful services rely on "stable data directories."

### Example: MySQL
Needs to persist:

- Data files
- Binlog
- Table structure
- System databases

### Example: Redis
Needs to persist:

- RDB
- AOF

### Example: MinIO
Needs to persist:

- Object file data

### Without PVC
These applications may:

- Lose data after Pod recreation
- Fail to form stable service states
- Simply not meet production requirements

---

## 18. Why This Is a Critical Prerequisite Before Entering "Stateful Application Deployment"

Because once you enter:

- StatefulSet
- MySQL
- Redis
- Nacos
- MinIO

You can't just look at:

- Image
- Deployment
- Service

You must also check:

- Whether storage binding succeeded
- Whether PVC is Bound
- Whether StorageClass exists
- Whether the data directory is actually mounted

Thus, PVC/StorageClass is a foundational layer that must be solidified before entering stateful application deployment.

---

## 19. Common Basic Issues

### 1. PVC Remains Pending
This is one of the most classic storage issues.

### 2. Pod Fails to Start or Gets Stuck
Because the PVC it depends on hasn't bound successfully.

### 3. The volume is mounted, but the application still doesn't work  
Possible reasons:  

- Mount path is wrong  
- The program isn't writing data to this volume  
- The application is still writing to the container's temporary layer  

### 4. Data isn't preserved  
Possible reasons:  

- Not really using PVC  
- Using a temporary directory  
- Mount path is incorrect  

---

## 20. What to Check First When Troubleshooting Storage Issues  

Establish a basic order first.  

### 1. Check PVC status first  
Focus on:  

- Is it Pending or Bound  

### 2. Check if StorageClass exists  
If PVC specifies a non-existent StorageClass, it will easily be Pending.  

### 3. Check if access mode and capacity are reasonable  
For example:  

- Requested 100Gi, but the backend doesn't have that much  
- Requested RWX, but the backend doesn't support it  

### 4. Check if the Pod successfully mounted  
Even if PVC is Bound, it doesn't mean the application is writing to the correct path.  

### 5. Finally, confirm whether the application's data directory is actually on the mounted volume  
This step is crucial.  

---

## 21. Why is StorageClass So Important  

Because in many modern Kubernetes clusters, PVCs don't necessarily rely on manually prepared PVs, but are more commonly:  

> **Provisioning storage dynamically through StorageClass.**  

This means:  

- The business side only requests PVC  
- The platform automatically creates PVs based on StorageClass  
- Automatic binding  
- Automatic integration with backend storage  

### Operations Understanding Focus  
The value of StorageClass lies in enabling storage to have:  

- Automation  
- Standardization  
- Orchestration capabilities  

This is crucial for platformization.  

---

## 22. The Most Important Understandings in This Topic  

### 1. The container's temporary layer is unsuitable for critical business data  
This is the starting point for learning persistence.  

### 2. PVC is an application's request for persistent storage  
It represents demand, not the backend storage itself.  

### 3. PV is the abstraction of actual volume resources  
It represents the supply result.  

### 4. StorageClass is the rule for dynamic provisioning  
It determines how to automatically create volumes.  

### 5. PVC Pending / Bound is a very critical status signal  
You will frequently see this during troubleshooting.  

---

## 23. What Level Should You Reach When Learning This Topic  

At this stage, it's recommended to reach the following level:  

### 1. Be able to explain what PV, PVC, and StorageClass are respectively  
### 2. Understand that PVC is a request, PV is a resource, and StorageClass is a rule  
### 3. Be able to understand a simple PVC YAML  
### 4. Understand how Deployment / Pod mounts PVC  
### 5. Be able to make basic judgments about PVC Pending  

---

## 24. Common Follow-up Questions in Interviews  

Common questions in this area include:  

- What are PV, PVC, and StorageClass respectively  
- What's the difference between PVC and PV  
- What does StorageClass do  
- Why is PVC always Pending  
- What's the basic way for Pod to use PVC  
- Why stateful applications rely more on PVC  
- What's the difference between ReadWriteOnce and ReadWriteMany roughly  
- Why data is lost after Pod restart  

---

## 25. Stage Summary  

PVC and StorageClass are the most fundamental and critical objects in Kubernetes' persistent storage system.  

The most important thing about this article isn't to learn all storage backends, but to establish the following core understandings first:  

- Critical business data cannot rely on the container's temporary layer  
- PVC is responsible for requesting storage  
- PV is responsible for carrying actual volume resources  
- StorageClass is responsible for defining dynamic provisioning rules  
- The Pending / Bound status of PVC directly affects whether the application can normally use the persistent volume  

As long as these understandings are established, the following will be much smoother:  

- Overview of stateful application deployment  
- StatefulSet  
- MySQL / Redis / Nacos  
- Storage class troubleshooting  

---

## 26. Keyword Mnemonics  

- PV: PersistentVolume, persistent volume  
- PVC: PersistentVolumeClaim, persistent volume claim  
- StorageClass: Storage class, dynamic provisioning rules  
- Persistent storage: Data remains after Pod restart  
- Pending: PVC hasn't been successfully bound yet  
- Bound: PVC has been successfully bound to PV  
- RWO: ReadWriteOnce  
- RWX: ReadWriteMany  

---

## 27. Operational Extended Understanding  

From an operational perspective, the significance of PVC / StorageClass isn't just "learning more objects," but it marks the entry of Kubernetes storage into a declarative management system.  

Without this system, many application deployments still remain in:  

- Temporary directories  
- Manual mounting  
- Host machine paths  
- Non-reusable storage methods  

Once entering the PVC / StorageClass mindset, the platform truly begins to have:  

- Standardized storage requests  
- Automated volume provisioning  
- Decoupling of applications and storage  
- Foundation for stateful service deployment  

This is why, if you want to truly understand:  

- StatefulSet  
- Containerized databases  
- Deployment of persistent middleware  
- Storage class troubleshooting  

You must first solidify this article.  

---

## References  
- Kubernetes Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/  
- Dynamic Volume Provisioning: https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/  
- Storage Classes: https://kubernetes.io/docs/concepts/storage/storage-classes/  

---

## Next Day Recommendation  
Next article recommendation:  

[[02-Stateful Application Deployment Overview - StatefulSet, Headless Service, and Persistence Strategy]]  
 /think