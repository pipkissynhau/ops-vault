# 02-Pod Pending Troubleshooting: Insufficient Resources, Scheduling Failures, Images and PVCs

Recommended Path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/02-Pod Pending Troubleshooting: Insufficient Resources, Scheduling Failures, Images and PVCs.md

Tags:

    #Kubernetes
    #PodPending
    #Scheduler
    #Resource Scheduling
    #PVC
    #Image Pulling
    #Node Affinity
    #Taint Tolerance
    #Cluster Basic Troubleshooting

---

## I. Document Description

This document records the basic troubleshooting methods for Pods that remain in a Pending state for an extended period in a Kubernetes cluster.

Pod Pending indicates that:

    The Pod has been accepted by Kubernetes but has not yet started running.

Common scenarios include:

    1. The Pod cannot be scheduled to any node.
    2. Insufficient node resources.
    3. Node selector or affinity rules are not met.
    4. Taints and tolerances do not match.
    5. The PVC is not bound.
    6. The StorageClass is unavailable.
    7. Preconditions for image pulling are abnormal.
    8. The node is NotReady.
    9. Scheduling constraints such as ports, host paths, and privilege settings are not met.

Objectives of this document:

    1. Determine the specific cause of Pod Pending.
    2. Analyze Pod Events.
    3. Examine the reasons for scheduling failures.
    4. Investigate insufficient resources.
    5. Check nodeSelector / nodeAffinity settings.
    6. Verify taints / tolerations.
    7. Assess PVC / StorageClass issues.
    8. Distinguish between Pending, ContainerCreating, and ImagePullBackOff states.

Applicable Scenarios:

    1. Self-built Kubernetes clusters using kubeadm.
    2. Private Kubernetes clusters.
    3. Pods remaining in a Pending state after new applications are deployed.
    4. Pods remaining in a Pending state after StatefulSets are created.
    5. Pods remaining in a Pending state due to PVC issues.
    6. Cases where insufficient resources prevent Pod scheduling.

---

## II. What is Pod Pending

View Pods:

    kubectl get pods -A

Example:

    NAMESPACE   NAME                         READY   STATUS    RESTARTS   AGE
    default     nginx-demo-6f8c9b7c9-abcde    0/1     Pending   0          5m

Pod Pending indicates that the Pod has not yet entered the normal running state.

Pending can generally be divided into two categories:

    1. Pending before scheduling
        The Scheduler is unable to find a suitable node for the Pod.

    2. Pending after scheduling / ContainerCreating
        The Pod has been bound to a node, but the container has not yet started successfully.

Strictly speaking, `ContainerCreating` is no longer considered a Pending state, but it is often grouped with Pending during troubleshooting because both indicate that the Pod has not started running yet.

---

## III. General Troubleshooting Approach

When troubleshooting Pod Pending issues, do not attempt to recreate the Pod directly.

Recommended order:

    1. Use `kubectl get pod` to check the status.
    2. Use `kubectl describe pod` to view Events.
    3. Determine whether a node has been assigned to the Pod.
    4. If no node is assigned, focus on scheduling issues.
    5. If a node is assigned, focus on kubelet, images, volume mounting, and runtime components.
    6. Check resource requests.
    7. Verify nodeSelector / affinity settings.
    8. Check taints / tolerations.
    9. Assess PVC / StorageClass / PV issues.
    10. Examine the node status and Events.

Branches of troubleshooting:

    Pod Pending
        |
        |-- No node assigned
        |       |
        |       |-- The Scheduler cannot schedule it.
        |       |-- Insufficient resources.
        |       |-- Node selection does not match.
        |       |-- Taints are not tolerated.
        |       |-- PVC is not bound.
        |
        |-- Node assigned
                |
                |-- Image pulling.
                |-- Volume mounting.
                |-- CNI network creation.
                |-- containerd / kubelet exceptions.

---

## IV. Step 1: Confirm Pod Status

View a specified namespace:

    kubectl get pods -n default -o wide

View all namespaces:

    kubectl get pods -A -o wide

View only Pods in the Pending state:

    kubectl get pods -A | grep Pending

Key points to observe:

    1. The namespace where the Pod is located.
    2. The### 7.3 Viewing Node Resources

To view node resources:

    kubectl describe node <node-name>

Key sections to check:

    Capacity
    Allocatable
    Allocated resources

To view all nodes:

    kubectl describe nodes | grep -A8 "Allocated resources"

If the metrics-server is installed, you can also see real-time usage:

    kubectl top nodes

    kubectl top pods -A

Note:

    `kubectl top` shows current usage.
    The Scheduler primarily considers the allocated `requests` when scheduling.
    Therefore, it is important to also refer to the `Allocated resources` from `kubectl describe node`.The PVC relied on by the Pod has not been properly bound.The configmap "xxx" was not found.

To check:

    Use `kubectl get secret -n <namespace>` and `kubectl get configmap -n <namespace>`.

Possible solutions:

    1. Create the missing Secret.
    2. Create the missing ConfigMap.
    3. Correct the Pod's reference name.
    4. Verify if the namespace is correct.If the NODE is empty:

    Key areas to check:
        1. Insufficient resources
        2. Node labels
        3. Affinity settings
        4. Pollution tolerance
        5. PVC configuration
        6. Node status
        7. Quota restrictions

If the NODE has a value:

    Key areas to check:
        1. Image retrieval
        2. Volume mounting
        3. Secret/ConfigMap settings
        4. CNI configuration
        5. kubelet behavior
        6. containerd operations

The most important command:

    kubectl describe pod <pod-name> -n <namespace>

In the vast majority of cases where a Pod is in the Pending state, the first clue can be found in the Events section.