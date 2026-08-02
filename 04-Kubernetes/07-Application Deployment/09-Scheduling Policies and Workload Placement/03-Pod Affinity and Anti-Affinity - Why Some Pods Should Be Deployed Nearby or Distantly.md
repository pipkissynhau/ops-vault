# 03-Pod Affinity and Anti-Affinity: Why Some Pods Should Be Deployed Nearby or Distantly

## Document Description
- Document Category: Advanced Kubernetes Scheduling Strategies
- Applicable Phases: 07-Application Deployment / 09-Scheduling Policies and Workload Placement
- Learning Objectives:
  - Understand what Pod affinity and anti-affinity aim to solve.
  - Grasp the basic concepts of `podAffinity` and `podAntiAffinity`.
  - Distinguish between "deploying Pods nearby" and "deploying them distantly".
  - Be able to interpret basic YAML configurations for Pod affinity and anti-affinity.
  - Comprehend common use cases such as multi-replica high availability, same-node deployment, and cross-node distribution.

## Establish an Initial Understanding

The previous two sections discussed:

- `nodeSelector`
- `nodeAffinity`

Essentially, they both address one question:

**On which node should this Pod be placed?**

However, sometimes scheduling decisions depend not only on nodes but also on the locations of other Pods.

For example:

- A business-related Pod might want to be deployed near a cache Pod to reduce network latency.
- Multiple replicas of a service should not all reside on the same node to avoid single-point failures.
- Some middleware components need to be distributed across different hosts.
- A logging component may prefer to be placed close to specific business Pods.

In these cases, the scheduler needs to consider:

**The locations of other Pods.**

This is where `podAffinity` and `podAntiAffinity` come into play.

## What is Pod Affinity

Pod affinity (`podAffinity`) means that a Pod:

**Wants or must be scheduled near certain Pods.**

The term "near" here is not arbitrary but depends on specific topological criteria, such as:

- Being on the same node
- Belonging to the same availability zone
- Sharing the same rack
- Being within the same topology domain

For beginners, focus on two of the most common scenarios:

- Being on the same node
- Being in the same availability zone

## What is Pod Anti-Affinity

Pod anti-affinity (`podAntiAffinity`) means that a Pod:

**Wants or must be scheduled away from certain Pods.**

A typical example is when you want to ensure that multiple replicas of a Deployment are not all on the same node, thereby preventing single-node failures.

In summary, you can remember:

- `podAffinity`: Deploy Pods nearby.
- `podAntiAffinity`: Deploy Pods away from specific Pods.

## Why Does Kubernetes Need Pod Affinity and Anti-Affinity

In real production environments, workloads often have relationships with each other.

### Scenarios Where “Nearby” Deployment Is Needed

For example:

- Applications that frequently communicate with local caches should be scheduled near them to reduce network latency.
- Collaborative services should be deployed within the same topology domain for better coordination.
- Logging or proxy components should be placed close to the business systems they serve.

### Scenarios Where “Distant” Deployment Is Needed

For example:

- Multiple replicas of a web service should not all reside on one node to prevent system crashes.
- Database and message queue replicas should be distributed across different nodes for high availability.
- If a single node fails, it should not affect all related services.

In essence, Pod affinity and anti-affinity help determine whether Pods should be grouped together or spread out.

## Differentiation from Node Affinity

It’s important to distinguish between these concepts:

### Node Affinity

Node affinity focuses on the **labels of nodes**. For example:

- `disktype=ssd`
- `env=prod`

### Pod Affinity/Anti-Affinity

Pod affinity/anti-affinity considers the **locations of other Pods and their labels**. In other words:

- Node affinity: Pods select nodes based on their own labels.
- Pod affinity/anti-affinity: Pods consider the locations and labels of other Pods when selecting a node.

## The Core Determinants of Pod Affinity and Anti-Affinity

These mechanisms do not rely solely on Pod names but typically use a combination of:

**Pod labels + topologyKey**

The scheduler will check:

- Whether there are Pods with specific labels.
- Where these Pods are currently located.
- Whether the new Pod should be placed near or away from them.

## What Is a TopologyKey

This is one of the most critical fields in this context. The `topologyKey` defines the **topological criterion used to determine proximity or distance**.

Common examples include:

- `kubernetes.io/hostname`: Determines proximity based on the node name.
- `topology.kubernetes.io/zone`: Determines proximity based on the availability zone.

For beginners, `kubernetes.io/hostname` is usually the most straightforward option. It essentially means:

**Determining whether Pods are located on the same host.**

##requiredDuringSchedulingIgnoredDuringExecution

This means that:

**Pods of the same type must not be scheduled on the same node.**

When the number of nodes is insufficient, this can directly cause some Pods to fail to be scheduled.

For example:

- You have 2 nodes.
- The number of Deployment replicas is 3.
- You require each replica to be on a different node.

In this case, the third Pod will definitely not start and will remain in a Pending state.

Therefore, in general business scenarios:

- If you want to distribute Pods but don't want scheduling to become blocked, preferred affinity is commonly used.
- Required affinity is only considered when extremely high availability is required.

## Example of required anti-affinity: Pods must be distributed across different nodes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hard-anti-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-hard
  template:
    metadata:
      labels:
        app: nginx-hard
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - nginx-hard
              topologyKey: kubernetes.io/hostname
          containers:
            - name: nginx
              image: nginx:1.25
```

This rule means that:

- Multiple Pods with `app=nginx-hard` cannot exist on the same node.
- If there are not enough nodes, some Pods will definitely remain in a Pending state.

## When is Pod affinity suitable?

### 1. Services with strong dependencies and frequent communications should be deployed close together.

For example:

- A certain application communicates frequently with caches, proxies, and auxiliary services.
- It is desirable to reduce cross-node network access.

### 2. If you want certain types of Pods to work together within the same topology domain.

For example:

- To achieve low-latency communication within the same availability zone.
- To share some local resources on the same node.

However, it should be noted that:

**Pod affinity can lead to more concentrated workloads.**
Concentration may bring advantages in terms of latency, but it can also pose risks, such as resource competition and increased risk of failures.

## When is Pod anti-affinity suitable?

### 1. To achieve high availability for multi-replica applications by distributing them across different nodes.

This is the most common use case.

### 2. To prevent similar services from being concentrated on a single node.

This can prevent a single point of failure from affecting all replicas.

### 3. To distribute replicas of middleware applications.

For example, multiple replicas of stateful applications should be distributed across different nodes.

### 4. To reduce resource competition.

If multiple resource-intensive Pods are placed together, they may compete for CPU, memory, and disk I/O resources.

## How to understand labelSelector

Pod affinity/anti-affinity is not about directly specifying "I want to be close to Pod A". Instead, it selects target Pods based on labels.

For example:

```yaml
labelSelector:
  matchExpressions:
    - key: app
      operator: In
      values:
        - redis
```

This means:

**Find all Pods with the label `app=redis`.**

Then, use the `topologyKey` to determine in which topology domains these Pods are located.

## A more practical way of understanding it

You can think of Pod affinity and anti-affinity as "social rules".

### Pod affinity

A Pod says:

**I want to be neighbors with certain types of Pods.**

### Pod anti-affinity

A Pod says:

**I don't want to be grouped with certain types of Pods.**

The criterion for determining "neighbors" or "separation" is the `topologyKey`.

## The relationship between Pod affinity/anti-affinity and the number of replicas

This is an important concept in production scenarios.

Suppose you have 3 replicas:

- If no anti-affinity rules are applied, all 3 replicas may be scheduled on the same node.
- If preferred anti-affinity is used, the scheduler will try to distribute them across different nodes.
- If required anti-affinity is used, they must be distributed; otherwise, they will not start.

Therefore, when considering high availability for replicas, it's not enough to just look at `replicas=3`. You also need to ensure that:

**These replicas are actually distributed across different fault domains.**

## The difference between podAntiAffinity and topologySpreadConstraints

You will learn about `topologySpreadConstraints` later on. For now, here is a basic understanding:

- `podAntiAffinity` is more about "avoiding being grouped with certain types of Pods".
- `topologySpreadConstraints` is more about "distributing Pods evenly across the topologyApp: nginx-spread-test  
Template:  
metadata:  
  labels:  
    app: nginx-spread-test  
spec:  
  affinity:  
    podAntiAffinity:  
      preferredDuringSchedulingIgnoredDuringExecution:  
        - weight: 100  
        podAffinityTerm:  
          labelSelector:  
            matchLabels:  
              app: nginx-spread-test  
            topologyKey: kubernetes.io/hostname  
containers:  
  - name: nginx  
    image: nginx:1.25  

Application:  
`kubectl apply -f nginx-spread-test.yaml`  

To check the distribution:  
`kubectl get pods -o wide`  

You can observe whether multiple replicas are distributed across different nodes. If your cluster has enough nodes, the effect will be more noticeable.  

## Key Points to Remember  
1. `podAffinity` helps Pods stay near similar Pods.  
2. `podAntiAffinity` prevents Pods from being placed on the same nodes as specific Pods.  
3. These rules consider “labels and locations of other Pods,” not just node labels.  
4. The `topologyKey` determines how proximity or dispersion is determined.  
5. `kubernetes.io/hostname` is a common topology key.  
6. For applications with multiple replicas, `podAntiAffinity` is often used for distributed deployment.  
7. `preferred` is more suitable for general use cases, while `required` is stricter but may cause Pending status.  

## Common Commands  
`kubectl get pods`  
`kubectl get pods -o wide`  
`kubectl get pods --show-labels`  
`kubectl get pods -l app=redis -o wide`  
`kubectl apply -f pod-affinity.yaml`  
`kubectl apply -f pod-anti-affinity.yaml`  
`kubectl describe pod <pod_name>`  

## Summary  
Pod affinity and anti-affinity determine whether Pods should be grouped together or spread out. Affinity promotes coordination, while anti-affinity ensures high availability by avoiding node conflicts.  

## Tags  
#Kubernetes #Application Deployment #Scheduling Strategies #Pod Affinity #Pod Anti-Affinity #High Availability #Replica Distribution  

## Further Reading  
- Kubernetes Official Documentation: Assigning Pods to Nodes  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/  
- Kubernetes Official Documentation: Inter-pod Affinity and Anti-affinity  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity  
- Kubernetes Official Documentation: Well-Known Labels, Annotations, and Taints  
  https://kubernetes.io/docs/reference/labels-annotations-taints/  

## Next Steps  
- Learn about [[04-Taints and Tolerations Basics]].  
- Understand the difference between “actively selecting nodes” and “preventing Pods from landing on certain nodes.”  
- Lay the foundation for advanced scheduling controls.