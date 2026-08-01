# 02-Pod Pending Troubleshooting: Resource Insufficiency, Scheduling Failure, Images and PVC

Recommended Path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/02-Pod Pending Troubleshooting: Resource Insufficiency, Scheduling Failure, Images and PVC.md

Tags:

    #Kubernetes
    #PodPending
    #Scheduler
    #ResourceMovement
    #PVC
    #MirrorPull
    #NodesAndSex
    #SlurpTolerance
    #ClusterInfrastructureBarriers

---

## I. Document Explanation

This document records basic troubleshooting methods for Pods that remain in a Pending state for a long time in a Kubernetes cluster.

Pod Pending indicates:

    The Pod has been received by Kubernetes but has not yet started running.

Common scenarios include:

    1. Pod cannot be scheduled to any node
    2. Node resource insufficiency
    3. Node selector or affinity rules not met
    4. Taints and tolerations mismatch
    5. PVC not bound
    6. StorageClass unavailable
    7. Image pull prerequisites abnormal
    8. Node NotReady
    9. Port, host path, privileged configuration, etc. scheduling constraints not met

This document's goals:

    1. Determine the specific cause of Pod Pending
    2. Understand Pod Events
    3. Understand scheduling failure reasons
    4. Troubleshoot resource insufficiency
    5. Troubleshoot nodeSelector / nodeAffinity
    6. Troubleshoot taints / tolerations
    7. Troubleshoot PVC / StorageClass
    8. Distinguish the boundary between Pending, ContainerCreating, and ImagePullBackOff

Applicable scenarios:

    1. kubeadm self-built cluster
    2. Private Kubernetes cluster
    3. New application deployment after Pod Pending
    4. StatefulSet creation after Pod Pending
    5. PVC causing Pod Pending
    6. Resource insufficiency causing Pod scheduling failure

---

## II. What is Pod Pending

Check Pod:

    kubectl get pods -A

Example:

    NAMESPACE   NAME                         READY   STATUS    RESTARTS   AGE
    default     nginx-demo-6f8c9b7c9-abcde    0/1     Pending   0          5m

Pod Pending indicates the Pod has not yet entered a normal running state.

Pending is generally divided into two categories:

    1. Pre-scheduling Pending
        Scheduler cannot find a suitable node for the Pod.

    2. Post-scheduling Pending / ContainerCreating
        The Pod has been bound to a node, but the container has not yet started successfully.

Strictly speaking, common `ContainerCreating` is no longer in a Pending state, but during troubleshooting, it is often viewed together with Pending because they all belong to "Pod not yet running".

---

## III. Troubleshooting Overview

Do not directly rebuild the Pod when troubleshooting Pod Pending.

Recommended order:

    1. kubectl get pod to check status
    2. kubectl describe pod to check Events
    3. Determine if the Pod has been assigned a Node
    4. If no Node, focus on scheduling issues
    5. If a Node has been assigned, focus on kubelet, image, volume mounting, and runtime
    6. Check resource requests
    7. Check nodeSelector / affinity
    8. Check taints / tolerations
    9. Check PVC / StorageClass / PV
    10. Check node status and events

Troubleshooting branches:

    Pod Pending
        |
        |-- No assigned NODE
        |       |
        |       |-- Scheduler cannot schedule
        |       |-- Resource insufficiency
        |       |-- Node selector mismatch
        |       |-- Taints cannot be tolerated
        |       |-- PVC not bound
        |
        |-- Assigned NODE
                |
                |-- Image pull
                |-- Volume mounting
                |-- CNI network creation
                |-- containerd / kubelet anomalies

---

## IV. Step 1: Confirm Pod Status

Check a specific namespace:

    kubectl get pods -n default -o wide

Check all namespaces:

    kubectl get pods -A -o wide

Only view Pending:

    kubectl get pods -A | grep Pending

Focus on observing:

    1. Pod's namespace
    2. Pod name
    3. STATUS
    4. AGE
    5. Whether NODE is empty
    6. Whether multiple Pods are Pending simultaneously
    7. Whether the same application is Pending
    8. Whether Pods related to the same node are abnormal

Example:

    kubectl get pod nginx-demo-xxx -o wide

If the output shows NODE is empty:

    Indicates the Pod has not been scheduled to a node.

If NODE has a value:

    Indicates the Pod has been scheduled, and the focus should be on kubelet, image, volume, and runtime afterward.

---

## V. Step 2: View Pod Events

This is the most critical step in troubleshooting Pending.

Execute:

    kubectl describe pod <pod-name> -n <namespace>

Example:

    kubectl describe pod nginx-demo-6f8c9b7c9-abcde -n default

Focus on the last Events:

Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  1m    default-scheduler  0/3 nodes are available: insufficient cpu.

Common Reasons:

    FailedScheduling
    FailedMount
    FailedAttachVolume
    FailedCreatePodSandBox
    ErrImagePull
    ImagePullBackOff
    NetworkPluginNotReady

Note:

    When troubleshooting Pending status, do not only check STATUS.
    Must check Events in describe pod.
    Events usually directly tell you the scheduling failure reason.

---

## Six. Determine if Scheduling is Successful

### 6.1 No Node Assigned

Execute:

    kubectl get pod <pod-name> -n <namespace> -o wide

If NODE is empty:

    NAME        READY   STATUS    NODE
    demo-pod    0/1     Pending   <none>

Explanation:

    Scheduler did not find a suitable node.

Key Troubleshooting:

    1. Resource insufficiency
    2. nodeSelector mismatch
    3. nodeAffinity mismatch
    4. taints / tolerations mismatch
    5. PVC not bound
    6. Node NotReady
    7. Port conflict
    8. topology spread constraints not met

---

### 6.2 Node Assigned

If NODE has a value:

    NAME        READY   STATUS              NODE
    demo-pod    0/1     ContainerCreating   k8s-worker-01

Explanation:

    Scheduler successfully scheduled the pod.
    Subsequent operations like pulling images, mounting volumes, creating networks, and starting containers are handled by the target node's kubelet.

Key Troubleshooting:

    1. Image pulling
    2. PVC mounting
    3. CNI creating Pod network
    4. containerd anomalies
    5. kubelet anomalies
    6. Secret / ConfigMap references missing

---

## Seven. Scheduling Failure: Resource Insufficiency

### 7.1 Common Events

Common events in Pod Events:

    0/3 nodes are available: insufficient cpu.
    0/3 nodes are available: insufficient memory.
    Insufficient cpu
    Insufficient memory
    Too many pods

Explanation:

    Pod's resources.requests exceeds the cluster's schedulable resources.

---

### 7.2 Check Pod requests

Check Pod configuration:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 resources

Or check Deployment:

    kubectl get deploy <deploy-name> -n <namespace> -o yaml | grep -A20 resources

Focus on:

    requests:
      cpu:
      memory:

Explanation:

    Scheduler primarily considers requests during scheduling, not limits.
    Limits affect runtime resource upper limits.
    Requests determine if scheduling can succeed.

---

### 7.3 Check Node Resources

Check node resources:

    kubectl describe node <node-name>

Focus on:

    Capacity
    Allocatable
    Allocated resources

Check all nodes:

    kubectl describe nodes | grep -A8 "Allocated resources"

If metrics-server is installed, view real-time usage:

    kubectl top nodes

    kubectl top pods -A

Note:

    kubectl top shows current usage.
    Scheduler primarily considers allocated requests.
    Combine with kubectl describe node's Allocated resources for analysis.

---

### 7.4 Handling Methods

Common handling methods:

    1. Reduce Pod requests
    2. Scale up nodes
    3. Delete unnecessary Pods
    4. Adjust replica count
    5. Add resources to nodes
    6. Optimize resource quotas
    7. Check namespace ResourceQuota

Check namespace quotas:

    kubectl get resourcequota -n <namespace>

    kubectl describe resourcequota -n <namespace>

---

## Eight. Scheduling Failure: nodeSelector Mismatch

### 8.1 Common Events

Events may show:

    0/3 nodes are available: node(s) didn't match Pod's node affinity/selector.

Explanation:

    Pod requires scheduling to nodes with specific labels, but no nodes meet the criteria.

---

### 8.2 Check Pod nodeSelector

Execute:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A10 nodeSelector

Example:

    nodeSelector:
      disk: ssd

---

### 8.3 Check Node Labels

Execute:

    kubectl get nodes --show-labels

Or check a specific node:

kubectl get node k8s-worker-01 --show-labels

If a Pod requires:

    disk=ssd

But the node lacks this label, the Pod cannot be scheduled.

---

### 8.4 Resolution

Add a label to the node:

    kubectl label node k8s-worker-01 disk=ssd

Check:

    kubectl get node k8s-worker-01 --show-labels

Or modify the Pod/Deployment to remove or adjust nodeSelector.

Note:

    Do not arbitrarily add labels to nodes in production environments.
    Labels typically represent node capabilities, such as disk type, data center, availability zone, business group, GPU, etc.

---

## Nine. Scheduling Failure: Node Affinity Mismatch

### 9.1 View Affinity

Execute:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A40 affinity

Common configuration:

    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: disk
              operator: In
              values:
              - ssd

Explanation:

    requiredDuringSchedulingIgnoredDuringExecution is a hard requirement.
    If no node satisfies this, the Pod will definitely be in Pending state.

---

### 9.2 Troubleshooting

Check Pod requirements:

    kubectl get pod <pod-name> -n <namespace> -o yaml

Check node labels:

    kubectl get nodes --show-labels

Confirm:

    1. Whether the key exists
    2. Whether the value matches
    3. Whether the operator is correct
    4. Whether all conditions are overly strict

---

### 9.3 Resolution

Common resolution methods:

    1. Modify node labels
    2. Adjust nodeAffinity rules
    3. Change required to preferred
    4. Add nodes that meet the criteria

Note:

    required is a hard constraint.
    preferred is a soft preference.
    Do not arbitrarily write overly strict required conditions in production.

---

## Ten. Scheduling Failure: Taint and Tolerance Mismatch

### 10.1 Common Events

Events may show:

    node(s) had untolerated taint
    node-role.kubernetes.io/control-plane:NoSchedule
    dedicated=xxx:NoSchedule

Explanation:

    The node has taints, and the Pod lacks corresponding tolerations, so it cannot be scheduled.

---

### 10.2 View Node Taints

Check all nodes:

    kubectl describe nodes | grep -i taints -A2

Check a specific node:

    kubectl describe node k8s-master-01 | grep -i taints -A2

Common Master taint:

    node-role.kubernetes.io/control-plane:NoSchedule

This indicates ordinary business Pods cannot be scheduled to Master nodes.

---

### 10.3 View Pod Tolerations

Execute:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 tolerations

If the Pod lacks corresponding tolerations, it cannot be scheduled to nodes with taints.

---

### 10.4 Resolution

Method 1: Add tolerations to the Pod.

Example:

    tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "ops"
      effect: "NoSchedule"

Method 2: Remove the node taint.

Example:

    kubectl taint node k8s-worker-01 dedicated=ops:NoSchedule-

Note:

    Do not arbitrarily remove taints from Master nodes.
    Unless it's an experimental environment or explicitly allowed for business workloads to run on Master nodes.

---

## Eleven. Scheduling Failure: Node NotReady

### 11.1 Common Events

Events may show:

    node(s) were not ready
    node(s) had condition DiskPressure
    node(s) had condition MemoryPressure
    node(s) had condition NetworkUnavailable

Check nodes:

    kubectl get nodes -o wide

If a node is NotReady, the Pod cannot be scheduled to that node.

---

### 11.2 Resolution

First troubleshoot the Node:

    kubectl describe node <node-name>

    systemctl status kubelet --no-pager

    systemctl status containerd --no-pager

    journalctl -u kubelet -n 100 --no-pager

Reference:

    01-Node NotReady Troubleshooting: kubelet, containerd, CNI and Node Events.md

---

## Twelve. Scheduling Failure: PVC Not Bound

PVC is a very common reason for Pod Pending, especially for StatefulSet, stateful services, databases, and middleware.

### 12.1 Common Events

Pod Events may show:

pod has unbound immediate PersistentVolumeClaims
persistentvolumeclaim "data-mysql-0" not found
0/3 nodes are available: pod has unbound immediate PersistentVolumeClaims

Explanation:

    The PVC that the Pod depends on is not properly Bound.

---

### 12.2 Check PVCs Referenced by Pod

Execute:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 volumes

Check PVC:

    kubectl get pvc -n <namespace>

Example:

    NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
    data-mysql-0    Pending                                      longhorn

If PVC is Pending, the Pod is typically also Pending.

---

### 12.3 Check PVC Details

Execute:

    kubectl describe pvc <pvc-name> -n <namespace>

Focus on Events:

    no persistent volumes available for this claim
    storageclass.storage.k8s.io "xxx" not found
    waiting for first consumer to be created before binding
    failed to provision volume with StorageClass

---

### 12.4 Check StorageClass

Execute:

    kubectl get storageclass

Check specified StorageClass:

    kubectl describe storageclass <storageclass-name>

Common issues:

    1. storageClassName is written incorrectly
    2. StorageClass does not exist
    3. provisioner is not running
    4. NFS / Longhorn / CSI anomalies
    5. dynamic provisioning failure
    6. unsupported access modes
    7. insufficient storage capacity

---

### 12.5 Troubleshoot NFS PVC

If using NFS StorageClass:

    kubectl -n storage-system get pods -o wide

    kubectl -n storage-system logs deploy/nfs-subdir-external-provisioner

    showmount -e 10.0.0.10

Common causes:

    1. nfs-subdir-external-provisioner is not running
    2. NFS Server is unreachable
    3. nfs-common is not installed
    4. NFS directory permissions are insufficient
    5. NFS path configuration is incorrect

---

### 12.6 Troubleshoot Longhorn PVC

If using Longhorn:

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get volumes.longhorn.io

    kubectl -n longhorn-system get nodes.longhorn.io

Common causes:

    1. Longhorn CSI is not running
    2. open-iscsi is not installed
    3. iscsid is not running
    4. replica count exceeds available nodes
    5. /data/longhorn space is insufficient
    6. Longhorn nodes are not schedulable

---

## ThirteenI don't know.Scheduling Failure: ResourceQuota Limit

Namespace may have configured ResourceQuota.

### 13.1 Check ResourceQuota

Execute:

    kubectl get resourcequota -n <namespace>

Check details:

    kubectl describe resourcequota -n <namespace>

Common limits:

    1. requests.cpu
    2. requests.memory
    3. limits.cpu
    4. limits.memory
    5. number of pods
    6. number of pvc
    7. total storage

---

### 13.2 Common Symptoms

Creating Pod or Deployment may fail directly, or cause resources to fail creation.

Check events:

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

Common information:

    exceeded quota
    must specify requests.cpu
    must specify requests.memory

---

### 13.3 Handling Methods

Common handling methods:

    1. Add requests / limits to Pod
    2. Reduce resource requests
    3. Adjust namespace quota
    4. Clean up unused resources
    5. Apply for higher quota

---

## FourteenI don't know.Scheduling Failure: LimitRange Default Resource Limit

Namespace may have configured LimitRange, which imposes restrictions on Pod default requests / limits.

Check:

    kubectl get limitrange -n <namespace>

Details:

    kubectl describe limitrange -n <namespace>

Impact:

    1. Automatically inject default requests to containers
    2. Automatically inject default limits to containers
    3. Restrict minimum and maximum resources
    4. Cause Pod resource requests to be higher than expected

When troubleshooting, note that:

    If requests are not specified in user YAML, it does not mean the final Pod has no requests.
    LimitRange may automatically inject values.

Check final Pod: /think

kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 resources

---

## 15. Scheduling Failure: hostPort Port Conflict

If a Pod uses hostPort, the same port on the same node can only be occupied by one Pod.

### 15.1 Common Events

You may see in Events:

    didn't have free ports for the requested pod ports

---

### 15.2 Check if Pod Uses hostPort

Run:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A10 hostPort

Or check the Deployment:

    kubectl get deploy <deploy-name> -n <namespace> -o yaml | grep -A10 hostPort

---

### 15.3 Handling Methods

Common handling methods:

    1. Reduce replica count
    2. Avoid using hostPort
    3. Use Service for exposure
    4. Ensure each node schedules only one replica
    5. DaemonSet scenarios are more suitable for hostPort

---

## 16. Scheduling Failure: Pod Topology Distribution Constraints

If topologySpreadConstraints are configured, it may cause Pending due to overly strict distribution constraints.

Check:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A40 topologySpreadConstraints

Common issues:

    1. Missing node topology labels
    2. maxSkew is too strict
    3. whenUnsatisfiable is DoNotSchedule
    4. Insufficient available nodes

Handling methods:

    1. Add topology labels to nodes
    2. Relax maxSkew
    3. Change DoNotSchedule to ScheduleAnyway
    4. Increase node count

---

## 17. Already Scheduled but Stuck in ContainerCreating

If the Pod NODE has a value, but the STATUS is:

    ContainerCreating

Focus on checking:

    kubectl describe pod <pod-name> -n <namespace>

Common causes:

    1. Image is being pulled
    2. PVC is mounting
    3. Secret/ConfigMap mount failed
    4. CNI failed to create Pod network
    5. containerd failed to create container

---

### 17.1 Image Issues

If you see events like:

    Failed to pull image
    ErrImagePull
    ImagePullBackOff

Troubleshoot:

    kubectl describe pod <pod-name> -n <namespace>

    sudo crictl pull <image>

    sudo crictl images

Common causes:

    1. Incorrect image address
    2. Non-existent image tag
    3. Private registry lacks imagePullSecret
    4. Node cannot access registry
    5. containerd lacks HTTP Harbor trust configuration
    6. Image repository certificate anomaly

Note:

    Strictly speaking, ImagePullBackOff is no longer Pending.
    However, it is often handled together with Pod startup failure during troubleshooting.

---

### 17.2 Secret/ConfigMap Not Found

Events may show:

    secret "xxx" not found
    configmap "xxx" not found

Check:

    kubectl get secret -n <namespace>

    kubectl get configmap -n <namespace>

Handling:

    1. Create missing Secret
    2. Create missing ConfigMap
    3. Correct Pod reference name
    4. Confirm namespace is correct

---

### 17.3 CNI Network Creation Failure

Events may show:

    FailedCreatePodSandBox
    failed to setup network for sandbox
    cni config uninitialized

Troubleshoot:

    kubectl get pods -A -o wide | grep -E "calico|flannel"

    ls -l /etc/cni/net.d/

    journalctl -u kubelet -n 100 --no-pager

Handling:

    1. Fix CNI plugin
    2. Check node network
    3. Check CNI configuration file
    4. Check containerd

---

## 18. Special Focus Points for StatefulSet Pending

StatefulSet Pending is often closely related to PVC.

Check StatefulSet:

    kubectl get sts -n <namespace>

Check Pod:

    kubectl get pod -n <namespace> -o wide

Check PVC:

    kubectl get pvc -n <namespace>

Common issues:

    1. PVC created by volumeClaimTemplates is Pending
    2. StorageClass does not exist
    3. Dynamic provisioner anomaly
    4. Longhorn replicas insufficient
    5. NFS provisioner anomaly
    6. accessModes mismatch

Example:

    mysql-0 Pending
    data-mysql-0 Pending

Troubleshooting order:

    1. describe pod mysql-0
    2. describe pvc data-mysql-0
    3. get storageclass
    4. Check provisioner/CSI
    5. Check backend storage

---

## 19. Special Focus Points for Deployment Pending

Deployment Pending is common in:

1. Too many replicas  
2. High requests  
3. nodeSelector mismatch  
4. Affinity too strict  
5. Missing imagePullSecret  
6. Namespace quota insufficient  

Check Deployment:  

    kubectl describe deploy <deploy-name> -n <namespace>  

Check ReplicaSet:  

    kubectl get rs -n <namespace>  

Check Pod:  

    kubectl get pod -n <namespace> -o wide  

Events remain based on Pod describe.  

---

## TwentyI don't know.DaemonSet Pending Special Attention Points  

DaemonSet expects each qualifying node to run a Pod.  

Common Pending reasons:  

    1. Node taints not tolerated  
    2. nodeSelector mismatch  
    3. hostPort conflict  
    4. Node resource insufficient  
    5. Image pull failure  

Check:  

    kubectl describe ds <daemonset-name> -n <namespace>  

    kubectl get pod -n <namespace> -o wide  

    kubectl describe pod <pod-name> -n <namespace>  

DaemonSet is commonly used for:  

    1. calico-node  
    2. kube-proxy  
    3. node-exporter  
    4. log-agent  
    5. storage csi node plugin  

If DaemonSet hasMass Pending, it may affect the entire cluster's basic capabilities.  

---

## Twenty-oneI don't know.Standard Troubleshooting Command Checklist  

### 21.1 Check Pod  

    kubectl get pod -n <namespace> -o wide  

    kubectl describe pod <pod-name> -n <namespace>  

    kubectl get events -n <namespace> --sort-by=.lastTimestamp  

---

### 21.2 Check Nodes  

    kubectl get nodes -o wide  

    kubectl describe node <node-name>  

    kubectl top nodes  

    kubectl top pods -A  

---

### 21.3 Check Scheduling Constraints  

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 nodeSelector  

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A40 affinity  

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A30 tolerations  

    kubectl describe nodes | grep -i taints -A2  

    kubectl get nodes --show-labels  

---

### 21.4 Check PVC / StorageClass  

    kubectl get pvc -n <namespace>  

    kubectl describe pvc <pvc-name> -n <namespace>  

    kubectl get pv  

    kubectl get storageclass  

    kubectl describe storageclass <storageclass-name>  

---

### 21.5 Check Quotas  

    kubectl get resourcequota -n <namespace>  

    kubectl describe resourcequota -n <namespace>  

    kubectl get limitrange -n <namespace>  

    kubectl describe limitrange -n <namespace>  

---

### 21.6 Node Native Check  

Execute on target node:  

    systemctl status kubelet --no-pager  

    systemctl status containerd --no-pager  

    journalctl -u kubelet -n 100 --no-pager  

    crictl ps -a  

    crictl images  

    ls -l /etc/cni/net.d/  

    df -h  

    free -h  

---

## Twenty-twoI don't know.Common Pending Reasons Quick Check  

| Phenomenon | Common Causes | Priority Check |  
|---|---|---|  
| NODE is empty | Scheduling failure | describe pod Events |  
| insufficient cpu | High CPU requests | describe node |  
| insufficient memory | High Memory requests | describe node |  
| unbound PVC | PVC not bound | describe pvc |  
| node selector mismatch | Node label mismatch | nodeSelector / labels |  
| untolerated taint | No tolerance for taints | taints / tolerations |  
| too many pods | Pod count reached node limit | describe node |  
| didn't have free ports | hostPort conflict | hostPort |  
| ImagePullBackOff | Image pull failure | describe pod |  
| FailedMount | Volume mount failure | PVC / CSI / NFS |  
| FailedCreatePodSandBox | CNI anomaly | CNI / kubelet |  

---

## Twenty-threeI don't know.Handling Recommendations  

When handling Pod Pending, it is recommended to follow:

1. Check Events first, don't guess  
2. First determine if a NODE has been assigned  
3. If there's no NODE, check scheduling  
4. If there's a NODE, check kubelet, image, volume, CNI  
5. For PVC Pending, check PVC first, don't just focus on Pod  
6. For resource insufficiency, check requests first, don't just look at top  
7. For scheduling rules, check nodeSelector, affinity, taints  
8. Don't delete business PVCs randomly  
9. Don't remove taints from production nodes randomly  
10. Don't blindly restart the entire node  

---

## Twenty-Four, Summary  

The core of Pod Pending troubleshooting is to first determine:  

    Is the Pod not scheduled successfully?  
    Or is it already scheduled but failed to start?  

Key judgment points:  

    Whether the NODE is empty in `kubectl get pod -o wide`.  

If NODE is empty:  

    Focus on checking:  
        1. Resource insufficiency  
        2. Node labels  
        3. Affinity  
        4. Taint tolerance  
        5. PVC  
        6. Node status  
        7. Quota limits  

If NODE has a value:  

    Focus on checking:  
        1. Image pull  
        2. Volume mounting  
        3. Secret / ConfigMap  
        4. CNI  
        5. kubelet  
        6. containerd  

The most important command:  

    `kubectl describe pod <pod-name> -n <namespace>`  

Most of the reasons for Pod Pending can be found in Events as the first clue.