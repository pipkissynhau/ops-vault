# 03-StatefulSet Basics: Stable Identity and Stable Storage

## Documentation Notes
- Documentation Focus: Core mechanisms and basic operations of StatefulSet
- Applicable Stage: After completing the overview of stateful application deployment, proceed to learn about StatefulSet, Headless Service, PVC, and basic verification
- Recommended Path: `04-Kubernetes/07-Apply deployment/03-Apply deployment in status/03-StatefulSet Foundation: Stabilization of identity and stable storage.md`

## Tags
#Kubernetes #StatefulSet #ApplyWithStatus #Stabilization #StableStorage #PVC #HeadlessService #MySQL #DNS #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## One: Why StatefulSet Can't Just Stay at the Concept Level

Learning StatefulSet shouldn't just remember the phrase:

- StatefulSet is suitable for stateful applications

This statement is too vague.

In real deployment and interviews, it's more important to understand the following questions:

- What's the difference between StatefulSet and Deployment
- What does stable identity mean
- What does stable storage mean
- Why components like MySQL, Redis, and Nacos often use StatefulSet
- Why StatefulSet is usually studied together with Headless Service and PVC
- Why MySQL using StatefulSet means Pod startup doesn't guarantee configuration correctness
- How to verify Pod's stable DNS name within the cluster

This article focuses not on filling concepts, but on explaining the core mechanism of StatefulSet and a key pitfall in a MySQL basic operation.

---

## Two: StatefulSet's Core Solves "Stateful Replicas", Not Just "Multiple Replicas"

Many beginners see StatefulSet and can write:

- `replicas: 1`
- `replicas: 3`

and mistakenly think:

StatefulSet is just another controller that can start multiple Pods.

This understanding is inaccurate.

### What Deployment Focuses On
- How many replicas I need
- These replicas just need to run
- Replicas are interchangeable
- If a Pod disappears, replacing it with a new one is fine

### What StatefulSet Focuses On
- Who are these replicas
- Are their numbering fixed
- Are they bound to their own volumes
- Can they access each other through fixed names
- Do they have predictable order during deletion and creation

### Operations Understanding Focus
StatefulSet manages a group of:

Instances with numbering, identity, storage relationships, and network identifiers.

---

## Three: What Does Stable Identity Mean

Stable identity means:

Pod names and numbering are fixed and predictable.

For example, a MySQL StatefulSet:

- `mysql-0`
- `mysql-1`
- `mysql-2`

These names aren't the random suffix names like Deployment, but are generated in a stable sequence.

### Why This Matters
Because many stateful components aren't "interchangeable", but rather:

- Who is the 0th instance
- Who is the 1st instance
- Which instance assumes what role
- Which instance corresponds to which data
- Which instance is called what name in the cluster membership list

### Difference from Deployment
Deployment Pods are often:

- `nginx-web-6d8f9c4fd8-x2abc`
- `nginx-web-6d8f9c4fd8-r8k9m`

These names suit stateless applications because Pods can be replaced anytime without emphasizing "who you are".

StatefulSet emphasizes:

Pods aren't just replicas, but also retain their logical identity.

---

## Four: What Does Stable Storage Mean

Stable storage can be understood as:

Each Pod binds to its own fixed volume, and after Pod recreation, it still returns to the original data.

For example:

- `mysql-0` corresponds to its own PVC and data directory
- `mysql-1` corresponds to its own PVC and data directory
- `mysql-2` corresponds to its own PVC and data directory

When a Pod is deleted and recreated:

- `mysql-0` should still mount back to `mysql-0`'s volume
- Not randomly mount to someone else's data

### Why This Matters
Database and middleware data isn't "whoever can take over".

If instance identity and volume relationships are chaotic, it will lead to:

- Data corruption
- Mismatch between node identity and data
- Membership relationship anomalies
- Initialization logic errors

### Operations Understanding Focus
One value of StatefulSet is maintaining:

Stable binding between instance identity and data relationships.

---

## Five: What Does Stable Network Identifier Mean

Besides stable names and volume relationships, StatefulSet usually pairs with Headless Service to provide predictable DNS names for each Pod.

For example:

- `mysql-0.mysql-headless.test.svc.cluster.local`
- `mysql-1.mysql-headless.test.svc.cluster.local`
- `mysql-2.mysql-headless.test.svc.cluster.local`

### Why It Matters
Because many stateful applications discover cluster members not by "temporary Pod IP", but by:

- Fixed hostnames
- Fixed member names
- Predictable DNS names

### Operations Understanding Focus
IP can change, but names should remain as stable as possible.
This is the network value of StatefulSet + Headless Service.

---

## Six: Why StatefulSet Often Appears with Headless Service

StatefulSet handles:

- Stable Pod names
- Stable replica numbering
- Stable volume relationships
- Ordered creation and deletion

Headless Service handles:

- Providing DNS methods suitable for stateful member discovery
- Letting Pods be accessed through fixed names

You can simplify it as:

- StatefulSet: Define "who is who"
- Headless Service: Define "how to find who"

---

## Seven: Why StatefulSet and PVC/volumeClaimTemplates Are Closely Related

Stateful applications don't just need:

- Processes to start
- Ports to listen
- Service accessibility

They also need:

- Each instance has its own data directory
- Rebuilding should continue mounting to their own data
- Different replicas shouldn't share the same write disk

So StatefulSet often includes:

- `volumeMounts`
- `volumeClaimTemplates`

### What volumeClaimTemplates Does
You can understand it as:

A template to automatically generate independent PVCs for each StatefulSet replica.

This way, you don't need to manually create:

- `mysql-0` PVC
- `mysql-1` PVC
- `mysql-2` PVC

Instead, StatefulSet automatically derives them.

### Operations Understanding Focus
This makes StatefulSet not just "ordered Pod startup", but possesses:

Ordered instances + independent storage complete capabilities.

---

## Eight: A Key Operational Point for MySQL Using StatefulSet

Many people when writing their first MySQL StatefulSet only focus on the following:

- Whether the image is correct
- Whether PVC has been successfully provisioned
- Whether Pod is Running
- Whether Service has been created successfully

But the official MySQL image has another critical startup requirement:

When MySQL is initialized for the first time, it must be passed root password-related environment variables.

If not provided, the container will typically fail to start and restart repeatedly.

### Typical Symptoms
For example, the Pod may show:

- `CrashLoopBackOff`
- `Exit Code: 1`

And in `describe pod` you might also see:

- `Environment: <none>`

This often indicates that although you've written the StatefulSet, the parameters required for the MySQL image startup haven't been fully provided.

---

## 9. Why MySQL Must Pass the Password on Startup

For official images like `mysql:8.0`, when initializing an empty data directory for the first time, it typically requires at least one of the following:

- `MYSQL_ROOT_PASSWORD`
- `MYSQL_ALLOW_EMPTY_PASSWORD`
- `MYSQL_RANDOM_ROOT_PASSWORD`

In basic learning and testing environments, the most common and intuitive way is:

- Setting `MYSQL_ROOT_PASSWORD`

### Operations Understanding Focus
It's important to distinguish between two levels:

#### 1) StatefulSet Layer
Responsible for:

- Stable identity
- Stable storage
- Network identity
- Replica order

#### 2) Image Startup Layer
Responsible for:

- Whether MySQL can normally initialize
- Whether root password is available
- Whether the database can complete first-time startup

In other words:

Writing a correct StatefulSet does not guarantee that the application image will definitely start normally.

---

## 10. Basic Example of MySQL StatefulSet

The following is a basic example to understand the relationship between StatefulSet, Headless Service, PVC, and MySQL startup parameters.

### 1) Headless Service

    apiVersion: v1
    kind: Service
    metadata:
      namespace: test
      name: mysql-headless
    spec:
      clusterIP: None
      selector:
        app: mysql
      ports:
        - port: 3306
          targetPort: 3306
          name: mysql

### 2) StatefulSet

    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      namespace: test
      name: mysql
    spec:
      serviceName: mysql-headless
      replicas: 3
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

## 11. The Most Critical Points in This StatefulSet.yml

### 1) `serviceName: mysql-headless`
This is not written arbitrarily.
It corresponds to the Headless Service above and participates in the stable DNS generation for StatefulSet Pods.

### 2) `replicas: 3`
Indicates that there will be:

- `mysql-0`
- `mysql-1`
- `mysql-2`

### 3) `volumeClaimTemplates`
Indicates that each replica will automatically generate its own PVC.

### 4) `env`
This is a very critical section in this MySQL practice:

    env:
      - name: MYSQL_ROOT_PASSWORD
        value: "123456"

If this is not written, MySQL may fail to complete the initial setup, and the Pod will enter repeated restarts.

### 5) `volumeMounts`
Indicates that each Pod will mount its data volume to:

- `/var/lib/mysql`

This is typically MySQL's data directory.

---

## 12. Why This YAML, Although Having 3 Replicas, Does Not Automatically Form a MySQL Cluster

This is a common misunderstanding.

Many people see:

    replicas: 3

And think:

I have deployed a 3-node MySQL cluster.

This is actually inaccurate.

### Actually, this is:
- 3 MySQL Pods with fixed identities
- Each Pod has its own PVC
- Each Pod has its own data directory
- They obtain stable naming through StatefulSet

### But this does not equal:
- Automatic master-slave replication
- Automatic high availability
- Automatic group replication
- Automatic failover

### Operations Understanding Focus
StatefulSet is responsible for:

Instance management model

Rather than:

Database cluster replication logic

---

## 13. How to Determine if StatefulSet + Headless Service Are Working Normally After Pod Startup

Assume you've already seen:

    kubectl -n test get pods

Output similar to:

- `mysql-0 Running`
- `mysql-1 Running`
- `mysql-2 Running`

This can only indicate:

- The Pod has started

But it cannot confirm:

- Does the Pod have a stable DNS
- Is the Headless Service working properly
- Can other Pods in the same namespace correctly resolve these names

Further verification is still needed.

---

## Fourteen. Why Not Prioritize Verifying Pod DNS on the Host Machine

Many people directly execute:

    nslookup mysql-0.mysql-headless

This may not reflect the actual DNS results within the cluster.

Reasons:

- The host uses its own DNS configuration
- Kubernetes Service / Pod DNS is mainly used internally by Pods
- The results obtained on the host may go through local DNS, intermediate forwarding, or other upstream DNS

### Operations Understanding Focus
The most reliable way to verify StatefulSet Pod DNS is not on the host machine, but:

Launch a temporary test Pod in the same namespace to perform DNS resolution verification within the cluster.

---

## Fifteen. How to Validate DNS in the Same Namespace Using dns-test

This hands-on practice is very important and a new focus of this article.

### Step 1: Create a Temporary Test Pod in the `test` Namespace

    kubectl -n test run dns-test --image=busybox:1.36 -it --rm -- sh

Notes:

- `-n test`: Place in the same namespace as MySQL
- `busybox:1.36`: Lightweight, suitable for DNS testing
- `-it --rm`: Interactive entry, automatically deleted after exit
- If you don't want automatic deletion, remove `--rm`.

---

## Sixteen. Why Must Validation Be Done in the Same Namespace

Because in the same namespace:

- DNS search domain is closer to the actual business scenario
- It's easier to verify short names and full domains
- It better matches the actual pattern of business Pods accessing each other

Example:

- `dns-test` and `mysql-0/mysql-1/mysql-2` are both in `test` namespace
- Their DNS resolution context is consistent

### Operations Understanding Focus
This is closer to the actual business access path than verifying on the host machine.

---

## Seventeen. What Should Be Verified in dns-test

After entering `dns-test`, the core is to verify the full domain name.

### Recommended Verification Commands

    nslookup mysql-0.mysql-headless.test.svc.cluster.local
    nslookup mysql-1.mysql-headless.test.svc.cluster.local
    nslookup mysql-2.mysql-headless.test.svc.cluster.local

### What Should Be Seen Normally
These names should resolve to the corresponding Pod IPs, for example:

- `mysql-0.mysql-headless.test.svc.cluster.local` → `mysql-0` Pod IP
- `mysql-1.mysql-headless.test.svc.cluster.local` → `mysql-1` Pod IP
- `mysql-2.mysql-headless.test.svc.cluster.local` → `mysql-2` Pod IP

### What Does This Indicate
It indicates the following links are established:

- StatefulSet creates Pods normally
- Headless Service exists normally
- Endpoints are correctly associated with backend Pods
- CoreDNS can properly generate and return records for these Pods

---

## Eighteen. Why Is It Recommended to Prioritize Full Domain Name Lookup

In actual testing, many people directly query short names, such as:

    nslookup mysql-0
    nslookup mysql-0.mysql-headless

Such queries may sometimes result in unstable outcomes or strange addresses due to:

- Search domain completion behavior
- Differences in tool-specific resolution
- Upstream DNS forwarding
- Typing errors

### Therefore, during basic verification, it is recommended to prioritize:

    nslookup mysql-0.mysql-headless.test.svc.cluster.local

This is the most direct and least prone to misjudgment.

### After Full Domain Name Verification Passes
You can then understand:

- Why short names are sometimes usable
- Why DNS lookup on the host may be inaccurate
- Why different tools behave differently

This will be clearer.

---

## Nineteen. What Does It Mean If Full Domain Name Resolution Succeeds

If you see in `dns-test`:

- `mysql-0.mysql-headless.test.svc.cluster.local` can resolve
- `mysql-1.mysql-headless.test.svc.cluster.local` can resolve
- `mysql-2.mysql-headless.test.svc.cluster.local` can resolve

Then you can basically confirm:

### 1) StatefulSet is Normal
Pod identity and numbering are stable.

### 2) Headless Service is Normal
The service exists as the network foundation for the StatefulSet.

### 3) Endpoints are Normal
The Headless Service has correctly attached to the MySQL Pods.

### 4) CoreDNS is Normal
DNS record generation and return are normal within the Kubernetes cluster.

---

## Twenty. What Ability Is This DNS Verification Actually Testing for StatefulSet

On the surface, you're running `nslookup`, but essentially you're verifying:

Whether the StatefulSet truly provides each stateful instance with a stable, discoverable, and predictable network identity.

This is critical for later learning the following components:

- MySQL
- Redis
- Nacos
- Elasticsearch
- MinIO
- Kafka
- ZooKeeper

Because many of these components rely on:

- Member discovery
- Node names
- Mutual access within the cluster

---

## Twenty-One. Basic Troubleshooting Approach for MySQL StatefulSet

If you've written a StatefulSet but the MySQL Pod fails to start, follow this troubleshooting order.

### 1) Check Pod Status First

    kubectl -n test get pods

### 2) Check Detailed Information

    kubectl -n test describe pod mysql-0

Focus on:

- `Events`
- `State`
- `Last State`
- `Exit Code`
- `Environment`

### 3) Check Logs

    kubectl -n test logs mysql-0
    kubectl -n test logs mysql-0 --previous

### 4) Prioritize Determining if Initialization Parameters Are Missing
If you see:

- `CrashLoopBackOff`
- `Exit Code: 1`
- `Environment: <none>`

Prioritize suspecting:

- Whether `MYSQL_ROOT_PASSWORD` was not passed

### 5) Then Check PVC Status

    kubectl -n test get pvc

### 6) Then Check Headless Service and Endpoints /think

kubectl -n test get svc
kubectl -n test get endpoints mysql-headless

### Operational Understanding Focus
Troubleshooting should be layered:

- Storage layer
- Scheduling layer
- Image pull layer
- Application startup parameter layer
- DNS / Service layer

In this MySQL hands-on practice, a typical pitfall is:

Pods are scheduled successfully, images are pulled down, but due to not passing MySQL root password, the container still fails.

---

## 22. Several Common Pitfalls in This Section

### 1. Only wrote StatefulSet, didn't write MySQL startup parameters
Result: Pod `CrashLoopBackOff`.

### 2. Thought `replicas: 3` equals MySQL cluster
Actually it's just 3 independent instances with stable identities, not automatically forming a replication cluster.

### 3. Only focused on Pod being Running, didn't verify DNS
This can't confirm if StatefulSet's stable network identity is truly effective.

### 4. Directly tested Pod DNS on the host
Easy to be misled by host DNS and upstream forwarding.

### 5. Wrote wrong or incomplete domain name
Causing misjudgment of CoreDNS or StatefulSet issues.

---

## 23. What Level Should You Master After Learning This Section

Current stage recommendations:

### 1) Can explain core differences between StatefulSet and Deployment
### 2) Understand what stable identity, stable storage, stable network identity mean
### 3) Understand the purpose of `serviceName`, `volumeClaimTemplates`, `volumeMounts`
### 4) Know that when deploying MySQL with StatefulSet, must pay attention to initialization password variables
### 5) Know how to verify full domain resolution with `dns-test` in same namespace

---

## 24. Common Interview Follow-up Questions

Common questions include:

- What's the difference between StatefulSet and Deployment
- What scenarios are suitable for StatefulSet
- What's stable identity
- What's stable storage
- Why is StatefulSet more suitable for MySQL, Redis, Nacos
- Why Headless Service often appears with StatefulSet
- What's `volumeClaimTemplates` for
- Why MySQL container failing doesn't necessarily mean PVC issues
- How to verify DNS for StatefulSet Pod

---

## 25. Stage Summary

StatefulSet's core isn't "starting multiple replicas", but:

- Stable identity
- Stable storage
- Stable network identity

It often appears with:

- Headless Service
- PVC
- `volumeClaimTemplates`

to express a more realistic stateful application model.

In MySQL basic practice, need to pay extra attention:

- `StatefulSet.yml` isn't just about writing image, volume, and port
- `mysql:8.0` must pass root password related environment variables during first initialization
- Otherwise Pod may still fail to start even if scheduled successfully

Additionally, when verifying StatefulSet DNS, recommend:

- Create `dns-test` in same namespace
- Use full domain name query inside Pod
- Don't rely only on host's `nslookup`

As long as these points are solidified, proceeding to:

- Headless Service basics
- StatefulSet common field parsing
- MySQL / Redis / Nacos middleware deployment

Overall understanding will be much smoother.

---

## 26. Keyword Mnemonics

- StatefulSet: Controller for stateful applications
- Stable identity: Fixed, predictable Pod name and number
- Stable storage: Pod maintains binding relationship with independent PVC
- Stable network identity: Pod has predictable DNS name
- Headless Service: Provides DNS foundation for stateful member discovery
- `volumeClaimTemplates`: Automatically generates independent PVC for each replica
- `MYSQL_ROOT_PASSWORD`: Common required environment variables for MySQL first initialization
- `dns-test`: Temporary test Pod to verify StatefulSet DNS in same namespace

---

## Operational Extended Understanding

From an operations perspective, StatefulSet's true value lies in:

It allows Kubernetes to not just batch start Pods, but begin to have the ability to maintain "specific instance persistence".

Stateless world is more like:

- Replica count is sufficient

Stateful world is more like:

- Which instances are these
- What data each carries
- How to discover each other through fixed names
- Whether it's still "the original instance" after deletion and recreation

This is the most critical mindset shift in StatefulSet learning.

---

## References
- Kubernetes StatefulSet: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- Kubernetes DNS for Services and Pods: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/

---

## Next Day Recommendation
Next article suggestion: organize

[[Headless Service Basics - Stateful Service Discovery Getting Started]]