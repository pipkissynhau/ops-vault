# 07-StatefulSet Common Field Analysis: serviceName, replicas, selector, volumeClaimTemplates

## Documentation Description
- Document Location: Introduction to the Core Fields of StatefulSet and Their Relationship with Stateful Application Deployment
- Applicable Stage: After understanding the main design concepts of stateful application deployment, further understand how StatefulSet translates these designs into Kubernetes resource layers
- Recommended Path: `04-Kubernetes/07-Application Deployment/03-Stateful Application Deployment/07-StatefulSet Common Field Analysis: serviceName, replicas, selector, volumeClaimTemplates`

## Tags
#Kubernetes #StatefulSet #Stateful Applications #serviceName #replicas #selector #volumeClaimTemplates #PVC #StorageClass #HeadlessService #MySQL #Secret #Application Deployment #Business Containerization #Cloud-Native #Operations and Maintenance

---

## I. Why This Article Returns to the Field Level to Understand StatefulSet

The previous article established several core elements of stateful applications from the perspective of “business deployment design”:

- Identity
- Storage
- Service Discovery
- Launch Order

This layer is very important because it ensures that when you look at StatefulSet, you don’t just see it as an object that “resembles Deployment.”

However, staying only at the design level is not enough.

Because when you actually deploy your business in Kubernetes, it ultimately boils down to YAML files, Helm values, resource fields, and object relationships.

In other words, you will face these questions later on:

- Why does `serviceName` usually need to be paired with a Headless Service?
- Why isn’t `replicas` just about the number of copies in stateful applications?
- Why can’t you write `selector` casually?
- How does `volumeClaimTemplates` ensure that each Pod gets its own volume?
- Why will an image like MySQL fail to start immediately if a key environment variable is missing?
- Why might changing one field affect the entire middleware deployment model?

Therefore, the goal of this article is not to mechanically explain YAML but to reframe the most critical fields of StatefulSet within the context of “business deployment design.”

The most important statement in this article is:

> **StatefulSet uses several key fields to translate ‘stateful application deployment design’ into a resource model that Kubernetes can execute.**

---

## II. Let’s Look at a MySQL StatefulSet Example Closer to Real-World Deployments

This time, instead of starting with the smallest possible example, let’s look at one that is closer to real-world deployments and can actually run MySQL successfully.

First, let’s examine the overall structure without getting into too much detail:

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

In this example, the fields that deserve priority attention are still:

- `serviceName`
- `replicas`
- `selector`
- `volumeClaimTemplates`

However, unlike the previous “minimal structure example,” this one includes additional elements such as:

- `env`
- `Secret`
- The root password required for MySQL to start

This is because:

> **The official MySQL image cannot be started just by providing an image and a mount directory.**
> **If the root password is not passed, the container will likely fail to start immediately.**

This is also one of the most common pitfalls when people first try to deploy MySQL using StatefulSet.

---

## III. Why the Previous Minimal Example Is Incomplete for MySQL

If we only focus on the field structure of StatefulSet, the example that only includes `image`, `ports`, and `volumeMounts` seems fine.

But if you actually try to use it to deploy MySQL, you will likely find that the Pod never starts, and the container logs will show initialization failures.

The root cause is simple:

- The MySQL image requires configuration related to the root account during initialization.
- The most common way> **Ensure that the members within a StatefulSet are part of a network model that can be detected and accessed by member name.**

---

## Six, How Headless Services and `serviceName` Usually Work Together

If you define it as follows:

    spec:
      serviceName: mysql-headless

Then there should typically be a Headless Service with the same name, for example:

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

The key point here is not to “memorize that `clusterIP: None`”, but to understand that:

- A Headless Service does not provide you with a unique virtual IP address.
- It essentially exposes the backend members directly to the DNS system.
- Combined with the stable Pod names from StatefulSet, it creates a more robust access model for the members.

If the current namespace is `test`, then a pod like `mysql-0` would typically be accessible via the following domain name:

- `mysql-0.mysql-headless.test.svc.cluster.local`

This is why `serviceName` and Headless Services are often mentioned together when discussing StatefulSet.

---

## Seven, How to Verify MySQL’s DNS Access Within the Same Namespace

If you have a test pod in the same namespace, such as `dns-test`, you can directly verify the full domain name.

For example, within the `test` namespace, you can enter the test pod and perform the following verification:

    kubectl -n test exec -it dns-test -- sh

Then try resolving the domain name:

    nslookup mysql-0.mysql-headless.test.svc.cluster.local

Or, if the image does not include `nslookup`, you can attempt to connect directly:

    ping mysql-0:mysql-headless.test.svc.cluster.local

If it is a MySQL client image, you can also test the ports or perform login attempts.

It is important to note that:

> **In the same namespace, it is recommended to verify the full domain name first.**

This approach is more intuitive and less likely to lead to misunderstandings due to differences in search domains, missing tools, or command variations.

---

## Eight, `replicas`: Does Not Refer to the Number of Simple Copies, but to the Number of Members

### 1) What Does This Field Look Like?

    spec:
      replicas: 1

Or:

    spec:
      replicas: 3

### 2) What Is Its Most Direct Meaning?

It indicates how many Pod copies the StatefulSet is intended to maintain.

### 3) Why Is It More Than Just a Number in Stateful Applications?

In a Deployment, `replicas: 3` often means:

- Three similar copies
- One can be replaced if lost
- All copies handle traffic similarly

However, in a StatefulSet, `replicas: 3` represents:

- 3 official members
- 3 numbered instances
- 3 nodes with unique identities and potentially different data

For example:

- `mysql-0`
- `mysql-1`
- `mysql-2`

These three are not simply “three additional containers”, but:

> **Define a stateful system consisting of 3 members.**

### 4) Why Is This Important for Business Deployment?

Many middleware systems are not designed to handle just any number of copies; instead, they are affected by factors such as:

- The number of cluster members
- Data distribution across replicas
- Node roles
- Master-slave or primary-backup structures
- Initialization processes
- Scaling complexities

### Key Points for Operations Engineers

In stateful applications, `replicas` should be understood as:

> **The number of official members I want to deploy, not just interchangeable copies.**

---

## Nine, Why Is It Recommended to Start with `replicas: 1` for MySQL Examples?

Although StatefulSet is well-suited for managing multi-replica stateful applications, it is suggested to start with:

    replicas: 1

during the initial learning phase for MySQL. The reason is not that StatefulSet cannot handle multiple replicas, but rather that:

- A single-instance setup makes it easier to verify storage, DNS, mounting, and environment variables.
- It provides a stable foundation before moving on to understanding master-slave, primary-backup, and cluster concepts.
- Starting with `replicas: 3` at the beginning can lead to immediate challenges such as MySQL initialization issues, root password problems, cluster topology concerns, member failures, and data synchronization issues, which can be confusing and difficult to debug.

Therefore, the recommended approach is:

> **First, use StatefulSet to set up a “single-instance MySQL with a stable identity, independent storage, Headless Service, and root password transmission”, before### Key Points for Operations and Maintenance Understanding

The value of `volumeClaimTemplates` lies not just in "automatically creating PVCs", but also in:

> **Establishing a one-to-one correspondence between members and data in stateful systems.**

---

## XIV. The Relationship Between `volumeClaimTemplates` and Container Mount Paths

In a StatefulSet, the common configuration is as follows:

    volumeMounts:
      - name: data
        mountPath: /var/lib/mysql

Combined with:

    volumeClaimTemplates:
      - metadata:
          name: data

The core relationship here is:

- `volumeClaim Templates` defines a volume template named `data`.
- `volumeMounts` mounts this volume to the specified path in the business container.
- Ultimately, each member will see its own data directory within its container path.

### Why This Is Important for Business Deployment

For many middleware components, the mount path is not arbitrary but closely related to the application's default data directory.

For example:

- The common data directory for MySQL is `/var/lib/mysql`.
- PostgreSQL has its own default data directory path.
- Redis, Elasticsearch, Kafka, and others also have specific data path requirements.

### Key Points for Operations and Maintenance Understanding

Deploying a business application is not just about having volumes; it's about ensuring that:

> **The volume declaration, the name of the volume, the container mount path, and the application's data directory are correctly matched.**

---

## XV. Why It’s Better to Store MySQL Passwords in Secrets Rather Than Embed Them Directly in YAML

From a "get-it-running" perspective, you could simply write it like this:

    env:
      - name: MYSQL_ROOT_PASSWORD
        value: "123456"

This would likely allow the container to start. However, from an operations and maintenance perspective and in real-world scenarios, it is more recommended to:

- Store the password in a Secret.
- Have the container reference it through `valueFrom.secretKeyRef`.

This approach has several advantages:

- The password is not directly hard-coded within the StatefulSet configuration.
- It aligns better with Helm values, environment isolation, and template-based management practices.
- It is more standardized than scattering plaintext passwords across multiple YAML files.

It’s also important to note that:

> **Secrets are not absolutely secure; they are just a more reasonable approach than writing plaintext passwords directly in the business YAML.**

For now, focusing on establishing these good practices is sufficient.

---

## XVI. What to Check First If a MySQL Pod Fails to Start

When you first deploy a MySQL StatefulSet and find that the Pod does not start, don’t immediately assume there is something wrong with the StatefulSet itself. It’s recommended to check in the following order:

### 1. Check the Pod Status

    kubectl get pod -n test

Pay special attention to:

- `CrashLoopBackOff`
- `Error`
- `ContainerCreating`

### 2. Check Container Logs

    kubectl logs -n test mysql-0

Many times, the critical information about why MySQL fails to start can be found in the logs.

### 3. Verify Whether Environment Variables Are Being Passed On

Check if the following is correctly defined in the StatefulSet configuration:

    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: MYSQL_ROOT_PASSWORD

And confirm that the Secret exists:

    kubectl get secret -n test

### 4. Check if Mounting Is Successful

Verify whether the PVC and PV have been successfully bound:

    kubectl get pvc -n test

### 5. Ensure Service and Labels Are Matched

If the container has started but cannot be accessed via DNS, check again:

- `serviceName`
- Headless Service
- `selector`
- `template.labels`

### Key Points for Operations and Maintenance Understanding

When a MySQL Pod fails to start, the most logical approach is not to wonder if the StatefulSet is unstable, but rather to consider whether there are issues with the image’s startup requirements, storage mounting, or label/service discovery mechanisms.

---

## XVII. Re-examining These Four Fields in the Context of Real MySQL Deployments

Let’s look at this section again:

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

These elements together- Headless Service: Suitable for member-level discovery  
- Stateful Applications: Applications that rely on membership relationships, data, and identity  
- Business Deployment Capability: The ability to align resource fields with actual system requirements  

---

## Twenty-One: Extended Understanding of Operations and Maintenance  

Many operations and maintenance professionals tend to view Kubernetes objects as “dispersed resource points” when learning about the platform:  

- What is a Deployment?  
- What is a StatefulSet?  
- What is a Service?  
- What is a PVC?  
- What is a Secret?  

However, when you truly begin to focus on “business containerization capabilities,” your approach must change:  

- Is this business stateless or stateful?  
- Does it require a unified service entry point or member-level access?  
- Are its data and members bound together?  
- Is it better suited for a Deployment or a StatefulSet?  
- What system relationships do the fields in a StatefulSet actually represent?  
- What additional conditions are required for this image to start successfully?  

The real value of this section lies not just in explaining these four fields but in helping you develop a way of thinking that is more closely aligned with practical work:  

> **When examining Kubernetes resources, focus not only on their syntax but also on the business deployment models they represent; when looking at middleware images, consider not only their names but also the prerequisites required for their successful startup.**  

Once this mindset is established, you will be able to understand why certain deployment approaches for systems like MySQL, Redis, Nacos, or Kafka are designed in a particular way, rather than simply copying charts without understanding their rationale.  

---

## References  
- Kubernetes StatefulSet: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/  
- Kubernetes Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/  
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/  

---

## Next Day’s Suggestions  
For the next article, it is recommended to organize the following content:  

[[08-Service Discovery Design for Stateful Applications: Headless Service, DNS, and Member Access Methods]]