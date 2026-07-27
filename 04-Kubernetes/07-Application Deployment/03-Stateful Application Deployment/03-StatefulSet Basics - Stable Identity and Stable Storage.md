# 03-StatefulSet Basics: Stable Identity and Stable Storage

## Document Description
- Document Focus: Core mechanisms and practical operations of StatefulSet
- Applicable Phase: After completing the overview of stateful application deployment, move on to learning about StatefulSet, Headless Service, PVC, and basic verification
- Recommended Path: `04-Kubernetes/07-Application Deployment/03-StatefulSet Basics: Stable Identity and Stable Storage.md`

## Tags
#Kubernetes #StatefulSet #Stateful Applications #Stable Identity #Stable Storage #PVC #HeadlessService #MySQL #DNS #Application Deployment #Cloud-Native #Ops #Interview Notes

---

## I. Why StatefulSet Shouldn’t Remain Just a Concept

When learning about StatefulSet, it’s not enough to remember just one thing:

- StatefulSet is suitable for stateful applications.

This statement is too general.

In real-world deployments and interviews, it’s more important to understand the following questions:

- What exactly are the differences between StatefulSet and Deployment?
- What does “stable identity” mean?
- What does “stable storage” mean?
- Why are components like MySQL, Redis, and Nacos often deployed using StatefulSet?
- Why is StatefulSet often considered in conjunction with Headless Service and PVC?
- Why doesn’t the successful startup of a Pod using StatefulSet guarantee that its configuration is correct?
- How can you verify the stable DNS name of a Pod within a cluster?

The focus of this article is not to overwhelm you with concepts but to clarify the core mechanisms of StatefulSet along with key issues encountered in practical MySQL deployments.

---

## II. What StatefulSet Really Solves Is Not “Multiple Replicas,” But “Identified Replicas”

Many beginners, upon seeing StatefulSet, might simply write:

- `replicas: 1`
- `replicas: 3`

and mistakenly think that StatefulSet is just another controller that can create multiple Pods.

This understanding is not accurate enough.

### What Deployment Focuses On
- How many replicas I need.
- As long as the replicas can run, that’s fine.
- The replicas are interchangeable.
- If one Pod goes down, it doesn’t matter if a new one is created.

### What StatefulSet Focuses On
- Which specific replicas these are.
- Whether their numbers are fixed.
- Whether they are bound to their own volumes.
- Whether they can access each other using fixed names.
- Whether their creation and deletion follow a predictable order.

### Key Points for Ops Professionals
StatefulSet manages not just a set of “randomly interchangeable replicas” but a group of instances that:

- Have fixed numbers and identities.
- Have established storage relationships.
- Have unique network identifiers.

---

## III. What Does “Stable Identity” Mean?

The most straightforward way to understand stable identity is that:

The Pod name and number are fixed and predictable.

For example, in a MySQL StatefulSet:

- `mysql-0`
- `mysql-1`
- `mysql-2`

These names are not randomly generated suffixes like those in Deployment but are sequentially assigned and remain stable.

### Why Is This Important?

Many stateful components are not identical; instead, it’s crucial to know:

- Which instance is the first one.
- Which instance performs which role.
- Which instance should store which data.
- What name this instance has within the cluster.

### The Difference from Deployment

Pods in Deployment often have names like:

- `nginx-web-6d8f9c4fd8-x2abc`
- `nginx-web-6d8f9c4fd8-r8k9m`

Such names are suitable for stateless services because Pods can be replaced at any time, and their identity doesn’t matter.

However, StatefulSet ensures that:

A Pod is more than just a replica; it also retains its logical identity.

---

## IV. What Does “Stable Storage” Mean?

Stable storage means that:

Each Pod is bound to its own fixed volume, so that when the Pod is recreated, it will access the same data as before.

For example:

- `mysql-0` is bound to its own PVC and data directory.
- `mysql-1` is bound to its own PVC and data directory.
- `mysql-2` is bound to its own PVC and data directory.

This way, when a Pod is deleted and recreated:

- `mysql-0` should still be mounted on its original volume.
- It shouldn’t be assigned to someone else’s data.

### Why Is This Important?

Data in databases and middleware cannot be easily transferred between different instances. If the identity of an instance and its volume relationship are confused, it can lead to:

- Data corruption.
- Inconsistent node identities and data.
- Abnormal member relationships.
- Initialization errors.

### Key Points for Ops Professionals

One of the key benefits of Statefulmetadata:
          labels:
            app: mysql
        spec:
          containers:
            - name: mysql
              image: mysql:8.0
              ports:
                - containerPort: 3306
                  name: mysql
              env:
                - name: MYSQL_ROOT_PASSWORD
                  value: "123456"
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
                storage: 1Gi

---

## Section Eleven: Key Points in This StatefulSet.yml File

### 1) `serviceName: mysql-headless`
This is not chosen arbitrarily.
It corresponds to the Headless Service mentioned earlier and participates in generating a stable DNS for the StatefulSet Pods.

### 2) `replicas: 3`
This means that there will be:

- `mysql-0`
- `mysql-1`
- `mysql-2`

### 3) `volumeClaimTemplates`
This indicates that each replica will automatically generate its own PVC.

### 4) `env`
This section is crucial in this MySQL implementation:

    env:
      - name: MYSQL_ROOT_PASSWORD
        value: "123456"

If this value is not set, MySQL may fail to initialize properly, causing the Pod to restart repeatedly.

### 5) `volumeMounts`
Each Pod will mount its data volume to:

- `/var/lib/mysql`

This is typically where MySQL stores its data.

---

## Section Twelve: Why Does Having 3 Replicas in This YAML Not Mean an Automatic MySQL Cluster?

This is a common misunderstanding.

Many people think that setting `replicas: 3` means they have deployed a 3-node MySQL cluster.

However, this is not accurate.

### In reality, it means:
- 3 MySQL Pods with unique identities
- Each Pod has its own PVC
- Each Pod has its own data directory
- They receive stable names through the StatefulSet

### But this does not equate to:
- Automatic master-slave replication
- Automatic high availability
- Automatic group replication
- Automatic failover

### Key Points for Operations and Maintenance
StatefulSet is responsible for:

- Managing instance identities

but not for:

- Implementing database cluster replication logic

---

## Section Thirteen: How to Verify Whether StatefulSet + Headless Service Are Working Properly After the Pods Are Started

Assuming you have checked:

    kubectl -n test get pods

and seen output like:

- `mysql-0 Running`
- `mysql-1 Running`
- `mysql-2 Running`

This only indicates that the Pods have started.

However, it does not confirm whether:

- The Pods have stable DNS
- The Headless Service is functioning correctly
- Other Pods in the same namespace can resolve these names properly

Further verification is needed.

---

## Section Fourteen: Why Not Verify Pod DNS Directly on the Host Machine?

Many people try to verify the Pod DNS directly on the host machine by running:

    nslookup mysql-0.mysql-headless

However, this may not reflect the actual DNS results within the cluster.

The reasons are:

- The host machine uses its own DNS configuration
- Kubernetes' Service/Pod DNS is mainly designed for use within the Pods themselves
- Results obtained on the host machine may go through local DNS systems, intermediate proxies, or other upstream DNS servers

### Key Points for Operations and Maintenance
The most reliable way to verify StatefulSet Pod DNS is not on the host machine but by:

- Creating a temporary test Pod in the same namespace
- Performing resolution checks within the cluster itself

---

## Section Fifteen: How to Use dns-test to Verify DNS in the Same Namespace

This practical step is very important and is a new focus of this article.

### Step 1: Create a Temporary Test Pod in the `test` Namespace

    kubectl -n test run dns-test --image=busybox:1.36 -it --rm -- sh

Explanation:

- `-n test`: Place the pod in the same namespace as the MySQL Pods.
- `busybox:1.36`: A lightweight image suitable for DNS testing.
- `-it --rm`: Enter interactively and the pod will be automatically deleted after exit.

If you do not want the pod to be deleted, you can remove `--rm`.

---

## Section Sixteen: Why Is It Important to Verify in the Same Namespace?

In the same namespace:

- The DNS search domain is more consistent with real business scenarios.
- It is easier to verify both short and full domain names.
- It better reflects the actual pattern of "business Pods communicating with each other."

For example, if `dns-test` and `mysql-0/mysql-1/mysql-2` are all in the `test` namespace,- `Environment`