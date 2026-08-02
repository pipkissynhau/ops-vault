# 04-Nginx Static Page Update Practice: Basics of Deployment Replica Scaling and Rolling Updates

## Document Description
- Document Purpose: Practical guidance on scaling and updating stateless applications
- Applicable Phase: Introduction to scaling and updates after completing basic practices with Deployment, Service, and NodePort
- Recommended Path: `04-Kubernetes/07-Application Deployment/02-Stateless Application Deployment/03-Nginx Static Page Update Practice: Basics of Deployment Replica Scaling and Rolling Updates`

## Tags
#Kubernetes #Deployment #RollingUpdates #ReplicaScaling #Nginx #StaticPages #StatelessApplications #ApplicationDeployment #BusinessContainerization #CloudNative #Ops #InterviewNotes

---

## I. Why Learn About Replica Scaling and Rolling Updates

Previous practices have established the following foundational steps:

- Image building
- Container running
- Creating Pods with Deployment
- Providing access through Service
- Enabling external access via NodePort

However, a truly viable business system will not always remain in a “single replica + unchanged version” state.

In real production environments, two common scenarios are frequently encountered:

### 1. Scaling Up
For example:

- A single replica may not be sufficient to handle increased traffic.
- It is necessary to improve availability.
- Redundant deployments are required.

### 2. Updates
For example:

- The page content needs to be updated.
- The image version changes.
- Bugs in the business code have been fixed.
- A new version must be released.

Therefore, learning about Deployment replica scaling and rolling updates means understanding how Kubernetes enables stateless applications to scale up and switch versions with minimal downtime or interruptions.

---

## II. What This Practice aims to Achieve

After completing this practice, you should be able to:

### 1. Understand the role of `replicas`
### 2. Comprehend why stateless applications are suitable for multiple replicas
### 3. Grasp how Deployment enables rolling updates
### 4. Distinguish between “creating Pods” and “updating Pods”
### 5. Diagnose common issues related to scaling and rolling updates
### 6. Be able to explain this logic effectively in interviews

---

## III. Why Nginx Static Pages Are Ideal for This Practice

Nginx static pages are a typical example of stateless applications, possessing the following characteristics:

- They do not rely on local critical states.
- Any replica can provide the same page content.
- There is no identity difference between replicas.
- If a Pod is deleted, a new one can be easily created to replace it.
- They are very suitable for scaling up and rolling updates.

Therefore, they are ideal for observing how Deployment manages multiple replicas, how Service distributes traffic across multiple Pods, and how the transition between old and new Pods occurs during image updates.

---

## IV. What is Replica Scaling

Replica scaling essentially means:

**Increasing the number of instances of the same application from a small number to a larger one.**

For example:

- Expanding from 1 replica to 2.
- Expanding from 2 replicas to 3.

In Deployment, this is directly reflected by the `replicas` field:

    replicas: 3

This indicates that Kubernetes is expected to maintain 3 Pods that conform to the template definition.

### Key Points for Ops Professionals
Replica scaling does not involve manually running additional containers. Instead:

- You declare the desired number of replicas through declarative configuration.
- Deployment ensures that this number is always maintained.
- If the actual number falls below the target, Kubernetes will add more replicas.
- If it exceeds the target, Kubernetes will remove excess replicas.

---

## V. Why Stateless Applications Are Suitable for Scaling

Stateless applications are naturally suited for multiple replicas because they typically have the following characteristics:

### 1. No identity difference between replicas
Any replica can replace another without affecting service functionality.

### 2. No reliance on local critical data
If a Pod is deleted, the service will not be unavailable due to the loss of local data.

### 3. Easy load balancing
Service can distribute requests evenly across multiple Pods.

### 4. Improved availability
Even if one Pod encounters an issue, other Pods can continue providing services.

### Taking Nginx Static Pages as an Example
If the page content comes from the same image, then all 3 Pods will return identical results.
This represents a standard stateless scaling scenario.

---

## VI. A Simple Example of a Multi-Replica Deployment

Here is an example of a Deployment with 3 replicas:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-web
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx-web
      template:
        metadata:
          labels:
            app: nginx-web
        spec:
          containers:
            - name: nginx-web
              image: harbor.example.com/demo/nginx-web:v1
### 二十、面试里可以怎么回答关于副本扩容的问题

你可以这样回答：

> Deployment 是一种非常适合无状态应用的控制器，它允许我们通过设置 `replicas` 来控制副本的数量。当我们需要进行副本扩容时，实际上就是增加期望的副本数量，Deployment 会自动创建新的 Pod 来满足这个需求。由于无状态应用的副本之间通常没有身份差异，因此使用 Deployment 可以很方便地实现负载分发和多副本容错机制。

---

### 二十一、面试里可以怎么回答关于滚动更新的问题

你可以这样回答：

> Deployment 默认就支持滚动更新功能。对于无状态应用来说，当我们修改了镜像或者 Pod 模板之后，Deployment 会自动创建一个新的 ReplicaSet，然后逐步启动新的 Pod、关闭旧的 Pod，从而完成版本的替换过程。这种更新方式的好处是，在服务不中断的情况下就能完成版本切换，而且客户端也不需要关心后端副本的具体变化过程，因为 Service 会持续把流量转发给那些处于 Ready 状态的 Pod。

---

### 二十二、这个实践中最重要的几个认知

### 1. 副本扩容和滚动更新是 Deployment 的核心能力
这两个功能使得 Deployment 成为处理无状态应用的最佳选择。

### 2. Service 让多副本和更新过程对客户端保持相对透明
客户端只需要关注一个稳定的服务入口，而不用关心底层的 Pod 或镜像变化。

### 3. 扩容关注的是副本数量，更新关注的是版本替换
在进行扩容时，我们关注的是增加多少个副本；而在进行更新时，我们关注的是将旧版本替换为新版本。

### 4. 不是 Deployment 改了就一定成功
要确保更新成功，还需要检查 Pod 的状态、日志信息以及访问结果是否正常。

### 5. 无状态应用特别适合做多副本和滚动更新
因为无状态应用的副本是可以相互替代的，而且它们的状态并不依赖于本地存储，因此非常适合进行多副本部署和滚动更新。