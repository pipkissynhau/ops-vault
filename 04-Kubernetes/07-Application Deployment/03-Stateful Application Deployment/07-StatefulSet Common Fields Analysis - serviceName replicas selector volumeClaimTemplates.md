# 07-StatefulSet Common Field Analysis: serviceName, replicas, selector, volumeClaimTemplates

## Document Notes
- Document Positioning: Core fields of StatefulSet and their relationship with stateful application deployment
- Applicable Stage: After understanding the mainline of stateful application deployment design, further understanding how StatefulSet translates these designs into Kubernetes resource layer
- Recommended Path: `04-Kubernetes/07-Apply deployment/03-Apply deployment in status/07-StatefulSet Common field resolution:serviceNameI don't know.replicasI don't know.selectorI don't know.volumeClaimTemplates`

## Tags
#Kubernetes #StatefulSet #ApplyWithStatus #serviceName #replicas #selector #volumeClaimTemplates #PVC #StorageClass #HeadlessService #MySQL #Secret #ApplyDeployment #OperationalContainerization #Clouds. #Transport

---

## One: Why This Article Returns to the Field Level to Understand StatefulSet

The previous article established the core elements of stateful applications from the perspective of "business deployment design":

- Identity
- Storage
- Service discovery
- Startup order

This layer is very important because it determines that when you look at StatefulSet, you won't just see it as a "object similar to Deployment".

But staying at the design level is still insufficient.

Because when truly deploying business into Kubernetes, you will eventually have to deal with YAML, Helm values, resource fields, and object relationships.

In other words, the problems you will face later include:

- `serviceName` Why is a Headless Service usually configured?
- `replicas` Why is it not just about replica count in stateful applications?
- `selector` Why can't you just write anything?
- `volumeClaimTemplates` Why can one Pod have one volume?
- Why does a MySQL image fail to start directly if it lacks a critical environment variable?
- Why changing a single field might affect the entire middleware deployment model?

The goal of this article is not to mechanically explain YAML, but to re-understand the most core fields in StatefulSet within the context of "business deployment mainline".

The most critical sentence in this article is:

> **StatefulSet translates "stateful application deployment design" into a Kubernetes executable resource model through several key fields.**

---

## Two: Let's Look at a More Realistic MySQL StatefulSet Example

This time, instead of looking at the "minimum example that can be written", let's look at an example closer to real deployment, at least one that can start MySQL normally.

First, look at the overall structure without rushing to dissect details:

    apiVersion: v1
    kind: Secret
    metadata:
      name: mysql-secret
    type: Opaque
    stringData:
      MYSQL_ROOT_PASSWORD: "123456"

    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: mysql
    spec:
      serviceName: mysql-headless
      replicas: 1
      selector:
        matchLabels:
          app: mysql
      template:
        metadata:
          labels:
            app: mysql
        spec:
          containers:
            - name: mysql
              image: mysql:8.0
              ports:
                - containerPort: 3306
              env:
                - name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysql-secret
                      key: MYSQL_ROOT_PASSWORD
              volumeMounts:
                - name: data
                  mountPath: /var/lib/mysql
      volumeClaimTemplates:
        - metadata:
            name: data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 10Gi

In this segment, the fields most worth prioritizing understanding are still:

- `serviceName`
- `replicas`
- `selector`
- `volumeClaimTemplates`

But unlike the previous "minimum structure example", this adds:

- `env`
- `Secret`
- The root password required for MySQL startup

Because:

> **The MySQL official image cannot start just by having an image and a mounted directory.**
> **If the root password is not passed, the container will typically fail to start directly.**

This is also one of the most common pitfalls people encounter when first writing MySQL as a StatefulSet.

---

## Three: Why the Minimal Example Is Incomplete for MySQL Scenarios

If only discussing StatefulSet field structure, the previous "only write image, ports, volumeMounts" example seems fine.

But if you really use it to start MySQL, you might find the Pod can't start, and the container logs will show initialization failure.

The root cause is not complex:

- The MySQL image requires root account configuration during initialization
- The most common way is to pass `MYSQL_ROOT_PASSWORD`
- Without this variable, MySQL initialization might fail directly

This highlights an important issue:

> **Learning StatefulSet cannot just focus on the resource fields themselves; it also needs to combine specific business image startup requirements.**

In other words:

- StatefulSet handles "identity, order, storage, member management"
- But whether the container image can actually start depends on the application's own startup conditions

For MySQL, the root password is a very typical startup condition.

| Field | Primary Function | Corresponding Business Deployment Requirements |
|---|---|---|
| `serviceName` | Connect Service / Headless Service | Service discovery, member access |
| `replicas` | Define replica count | Member count, cluster scale |
| `selector` | Define which members StatefulSet manages | Member ownership, label identification |
| `volumeClaimTemplates` | Generate independent PVC for each Pod | Independent storage, one Pod per volume |

If combined with real MySQL deployment, adding an understanding of "container startup conditions" would make this more complete:

| Configuration Item | Primary Function | Corresponding Business Deployment Requirements |
|---|---|---|
| `env` / `Secret` | Pass application initialization parameters | Image startup conditions, password management |

This table is important because it helps avoid a common misunderstanding:

> Don't interpret StatefulSet fields as "YAML syntax points," but rather as "business deployment requirements landing points."

---

## V. `serviceName`: Integrating Member Identity into Service Discovery

### 1) What does the field look like

Typically written as:

    spec:
      serviceName: mysql-headless

### 2) What is its most direct function

It is typically used to specify the name of a Service that pairs with a StatefulSet.

In stateful scenarios, this Service is usually a Headless Service.

### 3) Why is this field important

As previously explained, stateful applications need not only "member identity" but also "member discovery capability."

For example:
- `mysql-0`
- `mysql-1`
- `mysql-2`

These names are just the first step.  
More importantly, other members and clients need to be able to find them through stable means.

And `serviceName` is the key field that connects StatefulSet member identity to Kubernetes' service discovery system.

### 4) How to understand it from a business deployment perspective

If you are deploying:
- MySQL master-slave
- Redis cluster
- Nacos cluster
- ZooKeeper
- Kafka
- Elasticsearch

These components, the system often isn't just needing "a service that can be accessed," but rather:

- Specific members needing to communicate with each other
- Specific members having fixed address expressions
- Being able to write member lists in configuration

At this point, what `serviceName` represents isn't just "associating with a Service," but rather:

> **Establishing a service discovery entry at the member level.**

### 5) Why it's usually paired with Headless Service

Because regular Service emphasizes unified entry and load balancing, which is suitable for stateless services.

Headless Service instead emphasizes:

- Not hiding backend member differences
- More convenient member-level DNS resolution
- More suitable for access models with a fixed set of members

### 6) Key operational understanding

`serviceName` is not a "decorative field," nor is it a field that must be written for syntactic completeness.  
Its true meaning is:

> **Enabling StatefulSet members to enter a network model that is discoverable and accessible by member name.**

---

## VI. How Headless Service and `serviceName` Typically Work Together

If you write:

    spec:
      serviceName: mysql-headless

You usually also need a similarly named Headless Service, such as:

    apiVersion: v1
    kind: Service
    metadata:
      name: mysql-headless
    spec:
      clusterIP: None
      selector:
        app: mysql
      ports:
        - port: 3306
          targetPort: 3306

The key point isn't "remembering to set clusterIP: None," but understanding:

- Headless Service no longer provides a unified virtual IP
- It's more like exposing backend members directly to the DNS system
- Combined with StatefulSet's stable Pod name, it forms a more stable member access model

If the current namespace is `test`, then a Pod like `mysql-0` would commonly be resolvable in this way:

- `mysql-0.mysql-headless.test.svc.cluster.local`

This is why, when discussing StatefulSet earlier, `serviceName` and Headless Service usually appear together.

---

## VII. Verifying MySQL DNS Access Within the Same Namespace

If you've already prepared a test Pod in the same namespace, such as `dns-test`, you can directly verify the fully qualified domain name.

For example, in the `test` namespace, you can enter the test Pod:

    kubectl -n test exec -it dns-test -- sh

Then test resolution:

    nslookup mysql-0.mysql-headless.test.svc.cluster.local

Or if the image doesn't include `nslookup`, you can directly try connecting:

    ping mysql-0.mysql-headless.test.svc.cluster.local

If it's a MySQL client image, you can further test port or login.

A special note to pay attention to:

> **Within the same namespace, it's recommended to prioritize verifying the fully qualified domain name first.**

Because this is the most intuitive way, and it's also least likely to be misjudged due to search domain issues, missing image tools, or command differences.

---

## VIII. `replicas`: Defining Not Just Simple Replica Count, But Member Count

### 1) What does the field look like

    spec:
      replicas: 1

Or:

    spec:
      replicas: 3

### 2) What is its most direct meaning

It indicates how many Pod replicas the StatefulSet expects to maintain.

### 3) Why in stateful applications, it's not just a number

In Deployment, `replicas: 3` often represents:

- Three similar replicas
- Replace missing ones
- Any replica can handle traffic

But in StatefulSet, `replicas: 3` is more like:

- Three formal members
- Three numbered instances
- Three nodes with individual identities and possibly data

For example:

- `mysql-0`
- `mysql-1`
- `mysql-2`

These three are usually not simply "running three more containers," but rather:

> **Defining a stateful system composed of three members.**

### 4) Why this is important for business deployment

Because many middleware systems aren't as simple as "more replicas are better," but are influenced by these factors:

- Number of Cluster Members
- Data Replica Distribution
- Node Roles
- Master-Slave or Master-Slave Architecture
- Initialization Logic
- Scaling Complexity

### Operations Understanding Focus

In stateful applications, `replicas` should be translated as:

> **I need to deploy how many formal members, not how many replaceable replicas.**

---

## IX. Why the Example Suggests Starting with MySQL replicas: 1

Although StatefulSet is very suitable for hosting multi-replica stateful applications, during the MySQL learning phase, it is recommended to start with:

    replicas: 1

The reason is not that StatefulSet cannot support multiple replicas, but:

- Single instance makes it easier to first verify storage, DNS, mounting, and environment variables
- After running a single instance successfully, understanding master-slave, master-slave, and cluster configurations will be more stable
- If you start with `replicas: 3`, you will simultaneously face:
  - MySQL initialization issues
  - Root password problems
  - Cluster topology issues
  - Member discovery issues
  - Data synchronization issues

This mix of issues at this stage makes troubleshooting more difficult.

Therefore, the recommendation here is:

> **First use StatefulSet to run "single-instance MySQL + stable identity + independent storage + Headless Service + root password transmission", then proceed to the master-slave or cluster phase.**

---

## X. `selector`: Define Which Members StatefulSet Considers as Its Own

### 1) What does the field look like

    selector:
      matchLabels:
        app: mysql

### 2) What is its most direct purpose

Tell StatefulSet:

> **Which Pods with specific labels belong to my management.**

### 3) Why this field is critical

Because Kubernetes controller management objects are not based on "guessing names", but on label matching relationships.

In other words:

- StatefulSet identifies ownership through selector
- Pods are identified through labels
- Services also find Pods through selector

This entire relationship system relies on labels as the key language connecting resources.

### 4) Why it must be prioritized in business deployments

Because when deploying real business later, you often have more than one StatefulSet.

You will also have:

- Service
- Headless Service
- ConfigMap
- Secret
- Ingress
- Helm templates
- Monitoring or ServiceMonitor objects

Once the label system becomes chaotic, you may encounter:

- Service cannot select Pods
- Headless Service fails to work
- Controller ownership anomalies
- Application communication paths not matching expectations

### Operations Understanding Focus

The essence of `selector` is not just "writing an app label", but:

> **Clearly define member ownership relationships across the entire business deployment.**

---

## XI. `selector` Must Align with `template.labels`

This is very critical.

### Common Writing Style

    selector:
      matchLabels:
        app: mysql

    template:
      metadata:
        labels:
          app: mysql

This indicates:

- StatefulSet recognizes `app=mysql` Pods
- The Pods it creates will also carry `app=mysql`

This forms a closed loop.

### What happens if they are misaligned

For example:

    selector:
      matchLabels:
        app: mysql

But:

    template:
      metadata:
        labels:
          app: mydb

Then the Pods created by StatefulSet will not match its selector.

This leads to:

- Controllers cannot correctly identify resource ownership
- Services may also fail to select target Pods
- The entire deployment chain becomes abnormal

### Business Deployment Perspective

You can think of it as:

- `selector` is the "ownership rule"
- `template.labels` is the "factory identity label"

If the rule and identity don't match, the business deployment objects cannot be connected.

### Operations Understanding Focus

Many "business deployment succeeded but service connectivity failed" issues have simple root causes, often being:

> **Label system misalignment.**

---

## XII. `volumeClaimTemplates`: Template Independent Storage Capabilities

### 1) What does the field look like

    volumeClaimTemplates:
      - metadata:
          name: data
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi

### 2) What is its most direct purpose

Tell StatefulSet:

> **Generate an independent PVC for each Pod based on this template.**

### 3) Why this is particularly important for stateful applications

Because many stateful applications require:

- Each member has its own data directory
- Each member's data cannot be shared
- After reconstruction, they should be able to reconnect to the original data volume

For example:

- `mysql-0` uses its own volume
- `mysql-1` uses its own volume
- `mysql-2` uses its own volume

### 4) What fundamental problem does it solve

As previously explained, the key for stateful applications is not "whether data can be written", but:

> **Whether a member's data identity can remain stable.**

And `volumeClaimTemplates` is standardizing and automating this process.

---

## XIII. Why `volumeClaimTemplates` is More Suitable for Multi-Replica Stateful Deployments Than Manually Writing Individual PVCs

### Ordinary PVC Approach

Is more like:

- Manually request a volume
- Manually mount it to a Pod
- Suitable for single-instance or special control scenarios

### `volumeClaimTemplates` Approach

Is more like:

- Tell StatefulSet that each replica should have its own volume
- Volume count automatically expands with the number of members
- Each member maintains a relatively stable relationship with its volume

### Business Deployment Perspective

If you deploy:

- Single-instance testing MySQL
- Or some simple components

Manual PVC might still work.

But if you deploy:

- 3-node MySQL master-slave or cluster
- 3-node Nacos
- 3-node Kafka
- 3-node ZooKeeper

These systems with "multi-member + each member has independent data", the `volumeClaimTemplates` approach is more natural.

### Operations Understanding Focus

The value of `volumeClaimTemplates` is not just "automatically creating PVCs", but:

> **Fixing the relationship between members and data in stateful systems.**

## Fourteen. The Relationship Between `volumeClaimTemplates` and Container Mount Paths

In a StatefulSet, a common approach is to pair it with:

    volumeMounts:
      - name: data
        mountPath: /var/lib/mysql

Combined with:

    volumeClaimTemplates:
      - metadata:
          name: data

The core relationship is:

- `volumeClaimTemplates` defines a volume template named `data`
- `volumeMounts` mounts this volume to the specified path in the business container
- Ultimately, each member sees its own data directory in its container path

### Why This Matters for Business Deployments

Because for many middleware systems, the mount path isn't arbitrary - it's strongly related to the application's default data directory.

For example:

- MySQL's common data directory is `/var/lib/mysql`
- PostgreSQL has its own default path for data directories
- Redis, Elasticsearch, Kafka, etc., also have specific data path requirements

### Operations Understanding Focus

Deploying business applications isn't "having a volume is enough" - it requires ensuring:

> **The volume declaration, volume name, container mount path, and application data directory must correctly align with each other.**

---

## Fifteen. Why MySQL Passwords Should Be Placed in Secrets Rather Than Written in YAML Directly

From the perspective of "getting it to run", you could also directly write:

    env:
      - name: MYSQL_ROOT_PASSWORD
        value: "123456"

The container will likely start anyway.

But from operational habits and real-world scenarios, it's recommended to:

- Place passwords in Secrets
- Have containers reference them via `valueFrom.secretKeyRef`

This approach has several advantages:

- Passwords aren't hardcoded in the StatefulSet main body
- It better aligns with Helm values, environment isolation, and templating management concepts
- It's more standardized than scattering plaintext passwords across multiple YAML files

It's also important to clarify:

> **Secrets aren't absolutely secure, but they're more reasonable than writing plaintext passwords directly in the business YAML body.**

Establishing correct habits at this stage is sufficient.

---

## Sixteen. What to Check First If MySQL Pod Can't Start

When you first deploy a MySQL StatefulSet and find the Pod can't start, don't immediately suspect the StatefulSet itself is problematic.

Recommend checking in this order:

### 1. Check Pod Status

    kubectl get pod -n test

Focus on:

- `CrashLoopBackOff`
- `Error`
- `ContainerCreating`

### 2. Check Container Logs

    kubectl logs -n test mysql-0

Often, the most critical information about MySQL startup failure is here.

### 3. Verify Environment Variables Are Passed

Check if the StatefulSet actually contains:

    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: MYSQL_ROOT_PASSWORD

And confirm the Secret exists:

    kubectl get secret -n test

### 4. Check Mounts Are Normal

Verify PVC and PV binding is successful:

    kubectl get pvc -n test

### 5. Check Service and Labels Alignment

If the container is up but DNS or access is failing, then check:

- `serviceName`
- Headless Service
- `selector`
- `template.labels`

### Operations Understanding Focus

When MySQL can't start, the most common approach shouldn't be:

> "Is the StatefulSet unstable?"

It should be:

> "Is the container startup condition unmet, or is storage not properly mounted, or is there a problem with labels/service discovery chain?"

---

## Seventeen. Re-examining the Four Fields in the MySQL Real Deployment Mainline

Now re-examine this section:

    spec:
      serviceName: mysql-headless
      replicas: 1
      selector:
        matchLabels:
          app: mysql
      template:
        metadata:
          labels:
            app: mysql
        spec:
          containers:
            - name: mysql
              image: mysql:8.0
              env:
                - name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysql-secret
                      key: MYSQL_ROOT_PASSWORD
              volumeMounts:
                - name: data
                  mountPath: /var/lib/mysql
      volumeClaimTemplates:
        - metadata:
            name: data

These elements together aren't "basic StatefulSet syntax", but rather express a complete deployment design:

### 1. `serviceName`
These members need to enter the member-level service discovery system.

### 2. `replicas`
Deploy 1 formal member first to get the single instance running.

### 3. `selector`
These members are identified and managed through a unified label system.

### 4. `template.labels`
Created members must be correctly claimed by StatefulSet and Service.

### 5. `env + Secret`
The root password required for MySQL initialization must be passed in.

### 6. `volumeClaimTemplates`
This member must automatically get its own independent storage.

### Operations Understanding Focus

Therefore, a truly functional MySQL StatefulSet must simultaneously express:

- Member count
- Member identity
- Member discovery method
- Member ownership rules
- Startup initialization conditions
- Member storage model

This is why real business deployments always add several layers of understanding beyond the "minimum example".

## Eighteen. Commonly Overlooked Points from a Real-World Deployment Perspective

### 1. Only Looking at StatefulSet Fields, Not Image Startup Requirements

This is one of the most common issues.  
The YAML structure appears fine, but the business container simply can't start.

### 2. Assuming Mounted Storage Guarantees Database Startup

Whether a database can start depends not only on storage mounting, but also on initialization parameters, permissions, directories, and environment variables.

### 3. Only Writing `serviceName` Without Knowing Why

This leads to being able to copy YAML but not knowing when to configure a Headless Service.

### 4. Ignoring the Labeling System

If `selector`, Pod labels, and Service selectors are misaligned, the business access chain will break.

### 5. Knowing Only That Persistence Is Needed, Not Why One Pod Needs One Volume

This keeps you at a superficial level of "databases need persistent storage" without understanding the relationship between members and data.

### 6. Starting with a Multi-Replica MySQL

This mixes StatefulSet, DNS, master-slave, initialization, password, and synchronization issues together, making troubleshooting chaotic.

### Operations Understanding Focus

Now that you're learning this article, the most important takeaway should not be the definitions of four fields, but:

> **When I deploy a real middleware, I know what business problems these fields are solving for me, and I also know what startup conditions the image itself requires.**

---

## Nineteen. Stage Summary

The most common fields in a StatefulSet are not isolated YAML syntax points, but specific implementation points for deploying stateful applications in Kubernetes.

You can summarize it like this:

- `serviceName`: Integrates member identity into the service discovery system  
- `replicas`: Defines the number of members in a stateful system  
- `selector`: Defines member affiliation and label recognition rules  
- `volumeClaimTemplates`: Defines the independent storage template for each member  

If combined with real MySQL deployment, add this critical operations understanding:

- `env` / `Secret`: Meets business image initialization and startup requirements  

By this point, the real ability you should establish is not "memorizing StatefulSet," but:

- Seeing a field, knowing which business needs it serves  
- Seeing a middleware, being able to judge why these fields are designed this way  
- Seeing a StatefulSet YAML, being able to read it from a business deployment perspective  
- Seeing a container fail to start, knowing whether to check StatefulSet relationships first or application startup conditions  

Once this foundation is solid, you can proceed to:

- Headless Service and Member Access  
- MySQL Single-Instance Startup Verification  
- MySQL Master-Slave Deployment Approach  
- Differences in Stateful Design for Nacos / Redis and Other Middleware  

This will make the learning process smoother.

---

## Twenty. Keyword Mnemonics

- `serviceName`: Connection field for member-level service discovery entry  
- `replicas`: Definition of member count in a stateful system  
- `selector`: Rules for the controller to identify member affiliation  
- `template.labels`: Member's label identity  
- `volumeClaimTemplates`: Template mechanism for one Pod-one volume  
- `env`: Entry point for container startup parameters  
- `Secret`: Carrier for sensitive configurations like passwords  
- Headless Service: Suitable for member-level discovery  
- Stateful Application: Application dependent on member relationships, data, and identity  
- Business Deployment Capability: Ability to map resource fields to real system requirements  

---

## Twenty-One. Operations Extended Understanding

Many operations personnel tend to view Kubernetes objects as "dispersed resource points" when learning:

- What is a Deployment  
- What is a StatefulSet  
- What is a Service  
- What is a PVC  
- What is a Secret  

But when you truly start building "business containerization capabilities," your thinking must shift to another perspective:

- Is this business stateless or stateful  
- Does it need a unified service entry or member-level access  
- Is its data bound to members  
- Is it suitable for Deployment or StatefulSet  
- What system relationships do these fields in StatefulSet actually express  
- What additional conditions does this image require during startup  

Therefore, the true value of this article isn't just explaining four fields, but helping you establish a reading approach closer to actual work:

> **When looking at Kubernetes resources, don't just look at syntax - see what business deployment model it's expressing; when looking at a middleware image, don't just look at the image name - see what prerequisites it requires for startup.**

Once this foundation is established, when you later look at deployment methods for MySQL, Redis, Nacos, Kafka, etc., you won't just "copy charts" but will gradually understand why charts are designed this way.

---

## References
- Kubernetes StatefulSet: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/  
- Kubernetes Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/  
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/  

---

## Tomorrow's Recommendation
Next article recommendation:  

[[08-Service Discovery Design for Stateful Applications - Headless Service DNS Member Access Methods]]