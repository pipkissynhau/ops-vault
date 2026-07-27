# 09-HPA and VPA: Horizontal and Vertical Scaling, and Applicable Scenarios

## Document Description
- Document Purpose: To clarify the relationship between HPA and VPA and provide an introduction to scenario selection during Kubernetes resource management and auto-scaling.
- Target Audience: Those who have already understood concepts such as requests/limits, Pending, QoS, Eviction, basic principles of HPA, and VPA, and wish to build a comprehensive understanding of horizontal and vertical scaling.
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Auto Scaling/09-HPA and VPA: Horizontal and Vertical Scaling, and Applicable Scenarios.md`

## Tags
#Kubernetes #HPA #VPA #HorizontalPodAutoscaler #VerticalPodAutoscaler #horizontal scaling #vertical scaling #resource management #auto-scaling #application deployment #cloud-native #operation and maintenance #interview notes

---

## I. Why Separate the Relationship Between HPA and VPA After Learning About Them

Previously, we have learned about:

- What HPA is
- How HPA automatically adjusts the number of replicas based on metric changes
- Why increasing the number of Pods with HPA may not always resolve issues
- What VPA is
- Why metrics-server and VPA components are required for VPA
- Why `VerticalPodAutoscaler` is a custom resource
- Why it is better to start using VPA with `updateMode: Off`

However, if we stop here, although we have individually understood HPA and VPA, we will still face several practical problems in actual work:

- Whether the current issue is more related to HPA or VPA
- When we should increase the number of replicas first
- When we should adjust the resource specifications of a single Pod
- Whether one replaces the other
- Whether one is suitable for stateless applications and the other for stateful ones
- Whether they can be used together
- Which approach—adding more Pods or providing more resources to a single Pod—is better for solving resource issues

Therefore, the core focus of this section is not to explain each component separately but to integrate these two auto-scaling approaches:

> **What HPA solves, what VPA solves, what scenarios they are suitable for, and why they cannot be confused as one concept.**

---

## II. Key Conclusions to Remember

If you only need to remember a few things from this section, focus on the following conclusions.

### 1. HPA is for horizontal scaling
Its main function is:

> **To increase or decrease the number of Pod replicas based on metric changes.**

### 2. VPA is for vertical scaling
Its main function is:

> **To provide more reasonable CPU/memory request recommendations for a single Pod or to promote resource specification updates in appropriate modes.**

### 3. HPA and VPA do not replace each other
They address different issues:

- HPA focuses on the number of instances
- VPA focuses on resource specifications

### 4. HPA is more suitable for horizontally scalable scenarios
Typical examples include many stateless Web/API services.

### 5. VPA is more suitable for optimizing single-instance resources
Typical cases include:

- When it is difficult to predict the resource requirements of a single Pod over time
- When simply increasing the number of replicas is not effective
- In stateful components

### 6. The statement “HPA is suitable for stateless applications, and VPA is suitable for stateful applications” is not entirely incorrect but too absolute
A more accurate description would be:

> **HPA is more suitable for horizontally scalable scenarios; VPA is more suitable for optimizing single-instance resources. It is commonly used in stateful components but is not limited to them.**

---

## III. The Fundamental Difference Between HPA and VPA

The fundamental difference can be understood from what they adjust.

### What HPA Adjusts
HPA adjusts:

- `replicas`
- That is, the number of workload replicas

For example:

- Initially, there are 2 Pods.
- When the load increases, it becomes 4 Pods.
- When the load decreases, it reduces back to 2 Pods.

Therefore, HPA's core action is:

> **To distribute the load among more instances or reduce the number of instances during off-peak times.**

---

### What VPA Adjusts
VPA adjusts:

- The resource recommendations for a single Pod's container
- For example, CPU request and memory request
- And promotes the actual implementation of these resource specifications in certain modes

For example:

- Initially, the CPU request is `200m`.
- VPA recommends increasing it to `500m`.

Or:

- Initially, the memory request is `256Mi`.
- VPA recommends increasing it to `1Gi`.

Therefore, VPA's core action### Conclusion of This Chapter

The purpose of this chapter was not merely to distinguish between HPA and VPA in terms of their names, but to establish a more mature approach to resource management and auto-scaling. When facing operational challenges, it is crucial not to rush into solutions such as adding more Pods or allocating additional resources. Instead, one should first determine whether what is needed is:

- **More instances** to handle increased load, in which case HPA would be a suitable choice;
- **More appropriate resource specifications** for individual Pods, in which case VPA would be more effective.

By focusing on these key distinctions and understanding their respective roles, one can develop a more targeted and efficient approach to managing resource elasticity.