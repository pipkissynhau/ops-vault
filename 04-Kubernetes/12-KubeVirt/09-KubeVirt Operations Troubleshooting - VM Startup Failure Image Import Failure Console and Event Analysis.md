# 09-KubeVirt Operations Troubleshooting: VM Startup Failure, Image Import Failure, Console and Event Analysis

Recommended Path:

    04-Kubernetes/12-KubeVirt/09-KubeVirt Operations Troubleshooting: VM Startup Failure, Image Import Failure, Console and Event Analysis.md

Tags:

    #Kubernetes
    #KubeVirt
    #VmFragmentationBarrier
    #VMI
    #virt-launcher
    #CDI
    #DataVolume
    #PVC
    #Console
    #IncidentAnalysis
    #CloudlandVirtualization
    #PlatformEngineering

---

## I. Document Description

This document records common KubeVirt operations troubleshooting methods, focusing on:

    1. VM Creation Failure
    2. VM Startup Failure
    3. VMI Always Pending
    4. virt-launcher Pod Abnormalities
    5. Image Import Failure
    6. DataVolume Abnormalities
    7. PVC Mount Failure
    8. Console Connection Failure
    9. SSH Access Failure
    10. VM Network Unreachable
    11. VM Stop, Delete, Rebuild Related Issues
    12. How to Quickly Locate Issues via Events

Document Objectives:

    1. Establish a Standard Troubleshooting Path for KubeVirt
    2. Ability to Judge Fault Levels Based on VM, VMI, virt-launcher, PVC, DataVolume Status
    3. Master Common Error Phenomena and Corresponding Check Commands
    4. Ability to Handle Basic KubeVirt Operations Issues
    5. Ability to Explain KubeVirt Troubleshooting Logic in Interviews

Applicable Scenarios:

    1. kubeadm Self-Hosted Kubernetes Cluster
    2. KubeVirt + CDI + Longhorn Environment
    3. Private Cloud Platform
    4. Container and Virtual Machine Hybrid Platform
    5. KubeVirt from Introduction to Basic Operations Stage

---

## II. KubeVirt Troubleshooting Core Logic

KubeVirt troubleshooting cannot focus on a single object.

Standard Pod troubleshooting typically checks:

    Pod
    Events
    Logs
    Service
    PVC

KubeVirt troubleshooting requires additional checks on:

    VirtualMachine
    VirtualMachineInstance
    virt-launcher Pod
    virt-handler
    virt-controller
    DataVolume
    CDI importer Pod
    PVC / PV
    VolumeAttachment
    Node /dev/kvm
    Guest OS Internal Status

Standard Chain:

    VirtualMachine
        |
        v
    VirtualMachineInstance
        |
        v
    virt-launcher Pod
        |
        v
    PVC / DataVolume / Network
        |
        v
    Node / KVM / kubelet / containerd
        |
        v
    Guest OS

Troubleshooting Principles:

    1. Check VM Status First
    2. Then Check VMI Status
    3. Then Check virt-launcher Pod
    4. Then Check PVC / DataVolume
    5. Then Check KubeVirt Control Components
    6. Then Check Node KVM, kubelet, containerd
    7. Finally Troubleshoot Inside Guest OS

---

## III. Common KubeVirt Object Relationships

| Object | Function | Common Issues |
|---|---|---|
| VirtualMachine | Virtual Machine Definition and Desired State | Not Started, Configuration Errors, Status Abnormalities |
| VirtualMachineInstance | Running Virtual Machine Instance | Pending, Scheduling, Running, Failed |
| virt-launcher Pod | Hosts Virtual Machine Process | Pending, ImagePullBackOff, CrashLoopBackOff |
| DataVolume | Image Import and Disk Preparation | Import Failure, Pending, Failed |
| PVC | Virtual Machine Disk | Pending, Mount Failure, Insufficient Capacity |
| PV | Actual Storage Volume | Bound, Released, Failed |
| virt-controller | VM/VMI Controller | VM Status Control Abnormalities |
| virt-handler | Node-side VM Management Component | Abnormalities on Specific Node |
| virt-api | KubeVirt API Entry | virtctl console/start/stop Abnormalities |
| CDI importer | Image Import Pod | Image Download Failure, PVC Write Failure |

---

## IV. Pre-Troubleshooting Preparation Commands

### 4.1 View KubeVirt Components

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

    kubectl -n kubevirt get deploy

    kubectl -n kubevirt get ds

---

### 4.2 View CDI Components

    kubectl -n cdi get pods -o wide

    kubectl -n cdi get cdi

    kubectl get crd | grep cdi

---

### 4.3 View VM / VMI

    kubectl get vm -A

    kubectl get vmi -A

---

### 4.4 View Events

    kubectl get events -A --sort-by=.lastTimestamp

Specify namespace:

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

Filter by VM name:

    kubectl get events -n <namespace> --sort-by=.lastTimestamp | grep <vm-name>

---

## V. KubeVirt Standard Troubleshooting Path

When a VM is abnormal, execute the following steps in order.

Assumption:

    namespace: kubevirt-demo
    VM: vm-cirros-dv

### 5.1 View VM

    kubectl -n kubevirt-demo get vm vm-cirros-dv

    kubectl -n kubevirt-demo describe vm vm-cirros-dv

Focus On:

    Printable Status
    Ready
    Conditions
    Created
    Desired Generation
    Observed Generation
    Events

---

### 5.2 View VMI

    kubectl -n kubevirt-demo get vmi vm-cirros-dv

    kubectl -n kubevirt-demo describe vmi vm-cirros-dv

Focus On:

    Phase
    Node Name
    Interfaces
    Volumes
    Conditions
    Events

---

### 5.3 View virt-launcher Pod

    kubectl -n kubevirt-demo get pods -o wide | grep virt-launcher

    kubectl -n kubevirt-demo describe pod <virt-launcher-pod-name>

    kubectl -n kubevirt-demo logs <virt-launcher-pod-name> --tail=100

Focus On:

    Status
    Node
    Events
    Containers
    Volumes
    ImagePull
    FailedMount
    FailedScheduling

---

### 5.4 View DataVolume

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo describe dv <datavolume-name>

Focus On:

    Phase
    Progress
    Conditions
    Events

---

### 5.5 View PVC

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo describe pvc <pvc-name>

Focus On:

    Status
    Volume
    StorageClass
    Events

---

### 5.6 View KubeVirt Control Component Logs

    kubectl -n kubevirt logs deploy/virt-controller --tail=100

    kubectl -n kubevirt logs deploy/virt-api --tail=100

Check virt-handler:

    kubectl -n kubevirt get pods -o wide | grep virt-handler

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=100

---

### 5.7 View Node Status

Find the node where VM resides:

    kubectl -n kubevirt-demo get vmi vm-cirros-dv -o wide

Or:

    kubectl -n kubevirt-demo get pods -o wide | grep virt-launcher

Login to the node and check:

    ls -l /dev/kvm

    egrep -c '(vmx|svm)' /proc/cpuinfo

    systemctl status kubelet --no-pager

    systemctl status containerd --no-pager

    journalctl -u kubelet -n 100 --no-pager

---

## SixI don't know.Problem One: VM Stays Stopped After Creation

### 6.1 Phenomenon

Check:

    kubectl get vm -n kubevirt-demo

Output similar to:

    NAME           AGE   STATUS    READY
    vm-cirros      1m    Stopped   False

### 6.2 Common Causes

If VM YAML configures:

    runStrategy: Manual

The VM will not start automatically after creation.

This is normal behavior, not a fault.

### 6.3 Resolution

Start manually:

    virtctl start vm-cirros -n kubevirt-demo

Check:

    kubectl get vm -n kubevirt-demo

    kubectl get vmi -n kubevirt-demo

    kubectl get pods -n kubevirt-demo

### 6.4 Judgment

If VM transitions from Stopped to Running, it's normal.

If start fails, continue checking VMI, virt-launcher, and Events.

---

## SevenI don't know.Problem Two: virtctl start Fails

### 7.1 Phenomenon

Execute:

    virtctl start vm-cirros -n kubevirt-demo

May report errors:

    error starting VirtualMachine
    server could not find the requested resource
    connection refused
    unauthorized
    Internal error occurred

### 7.2 Troubleshoot virtctl

Check virtctl:

    which virtctl

    virtctl version

### 7.3 Check KubeVirt API

    kubectl -n kubevirt get pods | grep virt-api

    kubectl -n kubevirt get svc

    kubectl -n kubevirt logs deploy/virt-api --tail=100

### 7.4 Check KubeVirt Status

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt describe kv kubevirt

### 7.5 Common Causes /think

1. virt-api is not running  
2. KubeVirt CR is not Available  
3. virtctl version does not match KubeVirt  
4. Current kubeconfig is incorrect  
5. Current user lacks permissions  
6. KubeVirt CRD is not installed completely  

---

## 8. Problem Three: VMI Always Pending  

### 8.1 Phenomenon  

Check:  

    kubectl get vmi -n kubevirt-demo  

Output:  

    NAME           AGE   PHASE  
    vm-cirros      5m    Pending  

### 8.2 Check VMI Details  

    kubectl describe vmi vm-cirros -n kubevirt-demo  

Check events:  

    kubectl get events -n kubevirt-demo --sort-by=.lastTimestamp  

### 8.3 Common Causes  

    1. Node resource insufficiency  
    2. Node lacks /dev/kvm  
    3. Node does not support virtualization  
    4. PVC is not Bound  
    5. DataVolume is not completed  
    6. nodeSelector does not match  
    7. taints / tolerations do not match  
    8. virt-handler is abnormal  
    9. virt-launcher Pod cannot be scheduled  

### 8.4 Check virt-launcher Pod  

    kubectl get pods -n kubevirt-demo -o wide | grep virt-launcher  

If the Pod is also Pending:  

    kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo  

Focus on Events.  

Common events:  

    insufficient cpu  
    insufficient memory  
    pod has unbound immediate PersistentVolumeClaims  
    node(s) had untolerated taint  
    node(s) didn't match node selector  
    0/3 nodes are available  

### 8.5 Handling Directions  

Resource insufficiency:  

    Reduce VM memory / cpu request  
    Expand nodes  
    Clean up unused workloads  

PVC not bound:  

    Check PVC / StorageClass / Longhorn  

Node does not support KVM:  

    Check /dev/kvm  
    Check vmx / svm  
    Check nested virtualization  

---

## 9. Problem Four: virt-launcher Pod ImagePullBackOff  

### 9.1 Phenomenon  

Check:  

    kubectl get pods -n kubevirt-demo  

Output:  

    virt-launcher-vm-xxx   0/1   ImagePullBackOff  

### 9.2 Check Details  

    kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo  

Focus on:  

    Failed to pull image  
    ErrImagePull  
    ImagePullBackOff  

### 9.3 Common Causes  

    1. containerDisk image cannot be pulled  
    2. KubeVirt component image cannot be pulled  
    3. quay.io cannot be accessed  
    4. registry.k8s.io cannot be accessed  
    5. private Harbor authentication failure  
    6. imagePullSecret is missing  
    7. containerd has no HTTP registry trust configuration  

### 9.4 Verify Pull on Node  

Execute on the node where the Pod is located:  

    sudo crictl pull <image>  

Example:  

    sudo crictl pull quay.io/kubevirt/cirros-container-disk-demo:latest  

### 9.5 Handling Methods  

For domestic environments:  

    1. Synchronize the image to internal Harbor in advance  
    2. Modify VM YAML's containerDisk.image  
    3. Configure imagePullSecret  
    4. Configure containerd to trust internal Harbor  
    5. Avoid relying on real-time public network pulls  

---

## 10. Problem Five: DataVolume Always ImportInProgress or Pending  

### 10.1 Phenomenon  

Check:  

    kubectl get dv -n kubevirt-demo  

Output may be:  

    ImportInProgress  
    Pending  
    ImportScheduled  

### 10.2 Check DataVolume  

    kubectl describe dv <dv-name> -n kubevirt-demo  

### 10.3 Check PVC  

    kubectl get pvc -n kubevirt-demo  

    kubectl describe pvc <pvc-name> -n kubevirt-demo  

### 10.4 Check importer Pod  

    kubectl get pods -n kubevirt-demo | grep importer  

    kubectl describe pod <importer-pod-name> -n kubevirt-demo  

    kubectl logs <importer-pod-name> -n kubevirt-demo --tail=100  

### 10.5 Common Causes  

    1. Image URL cannot be accessed  
    2. Image download is too slow  
    3. Company network prohibits external network access  
    4. HTTP certificate anomaly  
    5. PVC is Pending  
    6. StorageClass does not exist  
    7. Longhorn / NFS / CSI anomaly  
    8. PVC capacity insufficiency  
    9. importer Pod cannot be scheduled  
    10. importer Pod image cannot be pulled  

### 10.6 Handling Methods  

Image URL unreachable:  

    Curl the image address on the node  
    Switch to internal HTTP file server  
    Place the image on 10.0.0.10 Nginx or object storage  

PVC Pending:

kubectl get storageclass  
kubectl describe pvc  
Check Longhorn / NFS / CSI  

Insufficient capacity:  

    Increase the storage request in the DataVolume's pvc.resources.requests.storage  

---  

## 11. Problem Six: DataVolume Failed  

### 11.1 Symptoms  

    kubectl get dv -n kubevirt-demo  

Output:  

    Failed  

### 11.2 Troubleshooting Commands  

    kubectl describe dv <dv-name> -n kubevirt-demo  

    kubectl get pods -n kubevirt-demo | grep importer  

    kubectl logs <importer-pod-name> -n kubevirt-demo --tail=200  

    kubectl get pvc -n kubevirt-demo  

    kubectl describe pvc <pvc-name> -n kubevirt-demo  

### 11.3 Common Error Directions  

    1. http status 404  
    2. connection timeout  
    3. no route to host  
    4. certificate signed by unknown authority  
    5. no space left on device  
    6. PVC mount failed  
    7. image format not recognized  
    8. importer container OOMKilled  

### 11.4 Recommended Actions  

    1. Confirm the URL is accessible  
    2. Use an internal mirror source  
    3. Adjust PVC capacity  
    4. Check StorageClass  
    5. Check CDI Pod  
    6. Check importer Pod logs  
    7. Recreate DataVolume  

---  

## 12. Problem Seven: PVC Bound but VM Startup Failed  

### 12.1 Symptoms  

PVC is already Bound:  

    kubectl get pvc -n kubevirt-demo  

But VM cannot start as Running.  

### 12.2 Troubleshoot VM / VMI  

    kubectl describe vm <vm-name> -n kubevirt-demo  

    kubectl describe vmi <vm-name> -n kubevirt-demo  

### 12.3 Troubleshoot virt-launcher  

    kubectl get pods -n kubevirt-demo -o wide | grep virt-launcher  

    kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo  

    kubectl logs <virt-launcher-pod-name> -n kubevirt-demo --tail=100  

### 12.4 Common Causes  

    1. Bootable disk image is unavailable  
    2. Image format is abnormal  
    3. System disk bus configuration is inappropriate  
    4. cloud-init configuration error  
    5. Node does not support KVM  
    6. Node resource insufficiency  
    7. PVC mount failed  
    8. Longhorn Volume attach anomaly  
    9. virt-handler anomaly  

### 12.5 Troubleshooting Directions  

First confirm:  

    Whether DataVolume is Succeeded  
    Whether PVC is Bound  
    Whether virt-launcher Pod is Running  
    Whether /dev/kvm exists on the node  
    Whether virt-handler is normal  
    Whether Guest OS image is bootable  

---  

## 13. Problem Eight: FailedMount / FailedAttachVolume  

### 13.1 Symptoms  

Events in virt-launcher Pod show:  

    FailedMount  
    FailedAttachVolume  
    Unable to attach or mount volumes  
    MountVolume.SetUp failed  
    Timed out waiting for the condition  

### 13.2 Check Pod  

    kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo  

### 13.3 Check PVC / PV  

    kubectl get pvc -n kubevirt-demo  

    kubectl describe pvc <pvc-name> -n kubevirt-demo  

    kubectl get pv  

    kubectl describe pv <pv-name>  

### 13.4 Check VolumeAttachment  

    kubectl get volumeattachment  

    kubectl describe volumeattachment <name>  

### 13.5 Longhorn Troubleshooting  

    kubectl -n longhorn-system get pods -o wide  

    kubectl -n longhorn-system get volumes.longhorn.io  

    kubectl -n longhorn-system get nodes.longhorn.io  

    kubectl -n longhorn-system get replicas.longhorn.io  

Node checks:  

    systemctl status iscsid --no-pager  

    systemctl status open-iscsi --no-pager  

    dpkg -l | grep open-iscsi  

### 13.6 Common Causes  

    1. Longhorn Volume cannot be attached  
    2. iscsid is not running  
    3. open-iscsi is not installed  
    4. Volume is already mounted on another node  
    5. RWO volume is used by multiple VMs simultaneously  
    6. Node is NotReady  
    7. Storage backend anomaly  
    8. CSI Node Plugin anomaly  

---  

## 14. Problem Nine: Console Connection Failed  

### 14.1 Symptoms  

Execute: /think

```
virtctl console <vm-name> -n <namespace>

Possible manifestations:

    1. Stuck indefinitely
    2. No output
    3. Connection error
    4. No response after entering username and password
    5. Login failure

### 14.2 Confirm that the VM is Running

    kubectl get vm -n <namespace>

    kubectl get vmi -n <namespace>

If the VMI does not exist, the console will definitely not work properly.

### 14.3 Check virt-api

    kubectl -n kubevirt get pods | grep virt-api

    kubectl -n kubevirt logs deploy/virt-api --tail=100

### 14.4 Check VMI

    kubectl describe vmi <vm-name> -n <namespace>

### 14.5 Check virt-launcher

    kubectl get pods -n <namespace> | grep virt-launcher

    kubectl logs <virt-launcher-pod-name> -n <namespace> --tail=100

### 14.6 Common Causes

    1. VM is not Running
    2. VMI does not exist
    3. virt-api anomaly
    4. Guest OS has not completed boot
    5. Image does not output console
    6. cloud-init has not completed
    7. Username or password error
    8. virtctl version mismatch

### 14.7 Recommendations

    1. Press Enter multiple times after entering the console
    2. Wait for Guest OS to complete boot
    3. Check virt-launcher logs
    4. Confirm that the image supports serial console
    5. Confirm that cloud-init configuration is correct
    6. Try virtctl restart VM
    7. Check virt-api

Exit console:

    Ctrl + ]

---

## FifteenI don't know.Problem Ten: VM is running but SSH is unreachable

### 15.1 Manifestations

VM is Running, Service has been created, but SSH connection fails.

### 15.2 Check Service

    kubectl get svc -n <namespace>

    kubectl describe svc <service-name> -n <namespace>

    kubectl get endpoints <service-name> -n <namespace>

### 15.3 Endpoints is empty

Common causes:

    1. Selector is written incorrectly
    2. virt-launcher Pod label does not match
    3. VM is not Running
    4. namespace is written incorrectly

Check:

    kubectl get pods -n <namespace> --show-labels

Service selector should match:

    kubevirt.io/domain=<vm-name>

### 15.4 Endpoints is not empty but SSH is unreachable

Enter VM:

    virtctl console <vm-name> -n <namespace>

Check inside VM:

    ip addr

    ip route

    ps aux | grep ssh

    netstat -lntp

    ss -lntp

Common causes:

    1. sshd is not started
    2. cloud-init has not enabled password login
    3. Firewall blocks inside VM
    4. Username or password error
    5. Service targetPort is incorrect
    6. NodePort firewall blocks
    7. kube-proxy anomaly

### 15.5 NodePort Troubleshooting

    sudo ipvsadm -Ln | grep <nodePort>

    sudo ufw status

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

---

## SixteenI don't know.Problem Eleven: VM network is unreachable

### 16.1 Manifestations

    VM cannot access external network
    VM cannot resolve DNS
    External cannot access VM
    Second network interface of VM is unreachable

### 16.2 Check VMI network

    kubectl get vmi <vm-name> -n <namespace> -o wide

    kubectl describe vmi <vm-name> -n <namespace>

    kubectl get vmi <vm-name> -n <namespace> -o yaml | grep -A50 interfaces

### 16.3 Check virt-launcher Pod network

    kubectl get pod -n <namespace> -o wide | grep virt-launcher

### 16.4 Enter VM

    virtctl console <vm-name> -n <namespace>

Execute inside VM:

    ip addr

    ip route

    cat /etc/resolv.conf

    ping -c 3 <gateway-ip>

    ping -c 3 <kubernetes-service-ip>

### 16.5 Check CoreDNS

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system get svc kube-dns

    kubectl -n kube-system get endpoints kube-dns

### 16.6 Check CNI

    kubectl get pods -A -o wide | grep -E "calico|flannel|cilium"

### 16.7 Check Service

    kubectl get svc -n <namespace>

    kubectl get endpoints -n <namespace>

### 16.8 Additional checks for Multus scenarios

    kubectl get pods -A | grep -i multus
```

kubectl get net-attach-def -A

kubectl describe net-attach-def <nad-name> -n <namespace>

kubectl describe vmi <vm-name> -n <namespace>

On the node:

    ls -l /opt/cni/bin/

    ip addr

    ip route

---

## 17. VM Deleted but PVC Still Exists

### 17.1 Phenomenon

After deleting VM:

    kubectl delete vm <vm-name> -n <namespace>

Check PVC:

    kubectl get pvc -n <namespace>

PVC still exists.

### 17.2 Explanation

This is a common and reasonable behavior.

Virtual machine definition and virtual machine disk are not the same thing.

Deleting VM does not necessarily delete PVC.

### 17.3 Risks

Do not delete PVC casually in production.

PVC may contain:

    1. VM system disk
    2. VM data disk
    3. Business data
    4. Database files
    5. User uploaded files

### 17.4 Pre-deletion Checks

    kubectl get vm -n <namespace>

    kubectl get vmi -n <namespace>

    kubectl get pvc -n <namespace>

    kubectl describe pvc <pvc-name> -n <namespace>

    kubectl get pv

    kubectl describe pv <pv-name>

Confirm:

    1. No VMs in use
    2. Data does not need to be retained
    3. Backups already exist
    4. StorageClass reclaimPolicy matches expectations

---

## 18. VM Cannot Schedule to Specified Node

### 18.1 Phenomenon

VMI Pending, Events show:

    node(s) didn't match node selector
    node(s) had untolerated taint
    insufficient cpu
    insufficient memory

### 18.2 Check VM YAML

    kubectl get vm <vm-name> -n <namespace> -o yaml | grep -A40 nodeSelector

    kubectl get vm <vm-name> -n <namespace> -o yaml | grep -A60 affinity

    kubectl get vm <vm-name> -n <namespace> -o yaml | grep -A40 tolerations

### 18.3 Check Nodes

    kubectl get nodes --show-labels

    kubectl describe node <node-name> | grep -i taints -A2

    kubectl describe node <node-name> | grep -A8 "Allocated resources"

### 18.4 Resolution

    1. Correct nodeSelector
    2. Correct nodeAffinity
    3. Add tolerations
    4. Reduce VM resource request
    5. Expand nodes
    6. Confirm node supports KVM

---

## 19. VMs on Node Are All Abnormal

### 19.1 Phenomenon

Multiple VMs on a single node are abnormal.

### 19.2 Check VM/Pod on the Node

    kubectl get vmi -A -o wide | grep <node-name>

    kubectl get pods -A -o wide | grep virt-launcher | grep <node-name>

### 19.3 Check virt-handler

    kubectl -n kubevirt get pods -o wide | grep virt-handler | grep <node-name>

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=200

### 19.4 Native Node Checks

    ls -l /dev/kvm

    egrep -c '(vmx|svm)' /proc/cpuinfo

    systemctl status kubelet --no-pager

    systemctl status containerd --no-pager

    journalctl -u kubelet -n 200 --no-pager

    df -h

    free -h

### 19.5 Common Causes

    1. virt-handler abnormal
    2. Node KVM abnormal
    3. kubelet abnormal
    4. containerd abnormal
    5. Node disk pressure
    6. Node network abnormal
    7. Longhorn / CSI Node Plugin abnormal
    8. Node resource insufficiency

---

## 20. All VM Operations Are Abnormal

### 20.1 Phenomenon

All VMs cannot start, stop, or access console.

### 20.2 Prioritize Checking KubeVirt Control Plane

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

    kubectl -n kubevirt get deploy

    kubectl -n kubevirt get ds

### 20.3 Check Logs

    kubectl -n kubevirt logs deploy/virt-api --tail=200

    kubectl -n kubevirt logs deploy/virt-controller --tail=200

    kubectl -n kubevirt logs deploy/virt-operator --tail=200

### 20.4 Check Events

    kubectl -n kubevirt get events --sort-by=.lastTimestamp

### 20.5 Common Causes

1. virt-api Exception  
2. virt-controller Exception  
3. KubeVirt CR Degraded  
4. webhook Exception  
5. APIService Exception  
6. Certificate Exception  
7. KubeVirt Component Image Pull Failure  
8. namespace Security Policy Limitation  

---

## Twenty-oneI don't know.Event Analysis Methods

Kubernetes Events is one of the troubleshooting entry points.

KubeVirt troubleshooting commonly uses:

    kubectl describe vm  
    kubectl describe vmi  
    kubectl describe pod  
    kubectl describe dv  
    kubectl describe pvc  
    kubectl get events  

### 21.1 Sort by Time  

    kubectl get events -n <namespace> --sort-by=.lastTimestamp  

### 21.2 Filter by VM Name  

    kubectl get events -n <namespace> --sort-by=.lastTimestamp | grep <vm-name>  

### 21.3 Filter Abnormal Events  

    kubectl get events -n <namespace> --sort-by=.lastTimestamp | grep -E "Warning|Failed|Error"  

### 21.4 Common Event Meanings  

| Event | Common Meaning |  
|---|---|  
| FailedScheduling | Scheduling Failure |  
| FailedMount | Volume Mount Failure |  
| FailedAttachVolume | Volume Attach Failure |  
| ErrImagePull | Image Pull Failure |  
| ImagePullBackOff | Continuous Image Pull Failure |  
| ImportInProgress | DataVolume is Importing |  
| ImportFailed | DataVolume Import Failure |  
| Started | VM Started |  
| Stopped | VM Stopped |  
| SuccessfulCreate | Controller Successfully Created Resource |  

---

## Twenty-twoI don't know.Log Viewing Methods  

### 22.1 virt-launcher Logs  

    kubectl logs <virt-launcher-pod-name> -n <namespace> --tail=200  

Suitable for troubleshooting:  

    VM Startup Failure  
    QEMU Exception  
    Disk Issues  
    Guest OS Startup Issues  

---

### 22.2 virt-controller Logs  

    kubectl -n kubevirt logs deploy/virt-controller --tail=200  

Suitable for troubleshooting:  

    VM / VMI Control Exception  
    VMI Creation Failure  
    Status Synchronization Exception  

---

### 22.3 virt-api Logs  

    kubectl -n kubevirt logs deploy/virt-api --tail=200  

Suitable for troubleshooting:  

    virtctl console Exception  
    start / stop API Exception  
    subresource Access Exception  

---

### 22.4 virt-handler Logs  

First find the node's corresponding virt-handler:  

    kubectl -n kubevirt get pods -o wide | grep <node-name>  

View logs:  

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=200  

Suitable for troubleshooting:  

    Node-side VM Management Exception  
    KVM Related Exception  
    VM Abnormality on a Specific Node  

---

### 22.5 CDI importer Logs  

    kubectl logs <importer-pod-name> -n <namespace> --tail=200  

Suitable for troubleshooting:  

    Image Import Failure  
    HTTP Download Failure  
    PVC Write Failure  

---

## Twenty-threeI don't know.Experiment One: Simulate VM Not Starting State  

### 23.1 Create Manual VM  

If VM uses:  

    runStrategy: Manual  

The VM will not start automatically after creation.  

Check:  

    kubectl get vm -n kubevirt-demo  

### 23.2 Observe Status  

    kubectl describe vm <vm-name> -n kubevirt-demo  

    kubectl get vmi -n kubevirt-demo  

    kubectl get pods -n kubevirt-demo  

Expected:  

    VM Exists  
    VMI Does Not Exist  
    virt-launcher Does Not Exist  

### 23.3 Start  

    virtctl start <vm-name> -n kubevirt-demo  

Observe:  

    kubectl get vm,vmi,pod -n kubevirt-demo  

Learning Points:  

    VM Existing Does Not Mean the Virtual Machine is Running.  
    VMI Existing Usually Indicates the Virtual Machine is Running.  

---

## Twenty-fourI don't know.Experiment Two: Simulate Service Selector Error  

### 24.1 Create Error Service  

Example selector intentionally written wrong:  

    selector:  
      kubevirt.io/domain: wrong-vm-name  

After applying, check:  

    kubectl get endpoints <svc-name> -n <namespace>  

Expected:  

    Endpoints: <none>  

### 24.2 Correct Selector  

Check virt-launcher Pod label:  

    kubectl get pods -n <namespace> --show-labels  

Correct selector:  

    kubevirt.io/domain: <vm-name>  

After modification:  

    kubectl apply -f <svc-yaml>  

Check:  

    kubectl get endpoints <svc-name> -n <namespace>  

Learning Points:  

    Service Existing Normally Does Not Mean Access to VM is Possible.  
    When Endpoints Are Empty, Prioritize Checking Selector and Pod Label.  

---

## Twenty-fiveI don't know.Experiment Three: Observe DataVolume Import Process  

### 25.1 Create DataVolume and Immediately Observe

kubectl get dv -n <namespace> -w

Open a new window:

    kubectl get pods -n <namespace> -w

### 25.2 View importer logs

    kubectl logs <importer-pod-name> -n <namespace> -f

Learning Points:

    DataVolume does not complete instantly.
    It creates an importer Pod.
    The importer Pod downloads the image and writes to PVC.
    DataVolume only becomes Succeeded after import completes.

---

## Twenty-sixth, Experiment Four: Observing Object Changes After VM Shutdown

### 26.1 Start VM

    virtctl start <vm-name> -n <namespace>

Check:

    kubectl get vm,vmi,pod,pvc -n <namespace>

### 26.2 Stop VM

    virtctl stop <vm-name> -n <namespace>

Check again:

    kubectl get vm,vmi,pod,pvc -n <namespace>

Expected:

    VM still exists
    VMI disappears
    virt-launcher Pod disappears
    PVC still exists

Learning Points:

    Separate understanding of VM lifecycle and disk lifecycle.

---

## Twenty-seventh, Common Troubleshooting Checklist

| Phenomenon | Priority Check | Common Causes |
|---|---|---|
| VM Stopped | runStrategy | Manual not started |
| virtctl start failed | virt-api | virt-api anomalies, permissions, version mismatch |
| VMI Pending | describe vmi / pod | Scheduling failure, resource insufficiency, PVC not bound |
| virt-launcher Pending | describe pod | Resources, PVC, taint, nodeSelector |
| ImagePullBackOff | describe pod | Image pull failure |
| DataVolume Pending | describe dv / pvc | StorageClass, PVC, CDI |
| DataVolume Failed | importer logs | URL, network, capacity, format |
| PVC Bound but VM cannot start | vmi / virt-launcher logs | Image cannot start, KVM, mounting |
| Console inaccessible | virt-api / VMI / launcher | VM not Running, virt-api, Guest not started |
| SSH unreachable | svc / endpoints / VM internal sshd | selector, sshd, firewall |
| VM network unreachable | VMI / VM internal ip addr | CNI, DNS, Service, Multus |
| VMs on certain nodes abnormal | virt-handler / node | KVM, kubelet, containerd |
| All VM operations abnormal | kubevirt namespace | virt-api, virt-controller, KubeVirt CR |

---

## Twenty-eighth, Production Handling Precautions

When troubleshooting KubeVirt, be cautious with the following operations:

    1. Do not arbitrarily delete PVC
    2. Do not arbitrarily delete PV
    3. Do not forcibly delete VolumeAttachment
    4. Do not arbitrarily delete Longhorn Volume
    5. Do not arbitrarily restart production nodes
    6. Do not arbitrarily restart containerd
    7. Do not arbitrarily delete virt-launcher Pod
    8. Do not arbitrarily delete DataVolume
    9. Do not rebuild system disk without backup
    10. Do not directly modify production VM's disk and network configuration

Before handling, confirm:

    1. Is there business traffic on VM?
    2. Are there data disks?
    3. Is there a backup?
    4. Is there a maintenance window?
    5. Can VM be stopped?
    6. Is there a rollback plan?

---

## Twenty-ninth, KubeVirt Troubleshooting Command Summary

### 29.1 VM / VMI

    kubectl get vm -A

    kubectl get vmi -A

    kubectl describe vm <vm-name> -n <namespace>

    kubectl describe vmi <vm-name> -n <namespace>

---

### 29.2 virt-launcher

    kubectl get pods -n <namespace> -o wide | grep virt-launcher

    kubectl describe pod <virt-launcher-pod-name> -n <namespace>

    kubectl logs <virt-launcher-pod-name> -n <namespace> --tail=200

---

### 29.3 KubeVirt Components

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

    kubectl -n kubevirt logs deploy/virt-api --tail=200

    kubectl -n kubevirt logs deploy/virt-controller --tail=200

    kubectl -n kubevirt logs deploy/virt-operator --tail=200

    kubectl -n kubevirt get pods -o wide | grep virt-handler

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=200

---

### 29.4 CDI / DataVolume

    kubectl -n cdi get pods -o wide

    kubectl get dv -A

    kubectl describe dv <dv-name> -n <namespace>

    kubectl get pods -n <namespace> | grep importer

    kubectl logs <importer-pod-name> -n <namespace> --tail=200

### 29.5 PVC / PV / Storage

    kubectl get pvc -A

    kubectl describe pvc <pvc-name> -n <namespace>

    kubectl get pv

    kubectl describe pv <pv-name>

    kubectl get storageclass

    kubectl get volumeattachment

    kubectl describe volumeattachment <name>

---

### 29.6 Longhorn

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get volumes.longhorn.io

    kubectl -n longhorn-system get nodes.longhorn.io

    kubectl -n longhorn-system get replicas.longhorn.io

---

### 29.7 Network

    kubectl get svc -n <namespace>

    kubectl get endpoints -n <namespace>

    kubectl get pods -n <namespace> --show-labels

    kubectl get net-attach-def -A

    kubectl get pods -A | grep -i multus

---

### 29.8 Nodes

    kubectl get nodes -o wide

    kubectl describe node <node-name>

Node Native:

    ls -l /dev/kvm

    egrep -c '(vmx|svm)' /proc/cpuinfo

    systemctl status kubelet --no-pager

    systemctl status containerd --no-pager

    journalctl -u kubelet -n 200 --no-pager

    df -h

    free -h

---

## 30. Interview Answer: How to Troubleshoot KubeVirt VM Startup Failure

You can answer:

    I would first check the status of VM and VMI to confirm whether VM is not started, VMI is not created, or VMI is created but in Pending state.
    Then I would check the Events in describe vm and describe vmi.
    If virt-launcher Pod has been created, I would check its status, Events, and logs.
    If it's a scheduling issue, I would focus on node resources, taint, nodeSelector, and whether PVC is Bound.
    If it's an image or disk issue, I would check DataVolume, PVC, PV, CDI importer logs, and backend storage.
    If Pod is Running but the virtual machine hasn't started, I would check virt-launcher logs, virt-handler logs, and node /dev/kvm.
    Therefore, KubeVirt troubleshooting should follow the chain of VM -> VMI -> virt-launcher Pod -> PVC/DataVolume -> Node/KVM.

---

## 31. Interview Answer: How to Troubleshoot DataVolume Import Failure

You can answer:

    When DataVolume import fails, I would first run kubectl describe dv to check status and Events.
    Then I would check whether the corresponding PVC is Bound.
    Next, I would find the importer Pod and check describe pod and logs.
    Common causes include inaccessible image URL, network timeout, PVC capacity insufficient, StorageClass not existing, or CSI or Longhorn anomalies.
    In production environments, I would recommend using internal image registry or internal HTTP mirror source, avoiding reliance on public addresses for real-time import.

---

## 32. Interview Answer: How to Troubleshoot Console Access Issues

You can answer:

    When console access fails, I would first confirm whether VM is Running and VMI exists.
    If VM is not running, console access will definitely fail.
    Then I would check virt-api status, as virtctl console depends on KubeVirt API.
    Next, I would check VMI and virt-launcher Pod status and logs.
    If virt-launcher is Running but console has no output, it might be due to Guest OS not completing boot, unsupported serial console for image, cloud-init anomalies, or incorrect user password configuration.
    I could also try pressing Enter multiple times after entering console, or validate VM status via SSH side-channel.

---

## 33. Summary of This Article

The core of KubeVirt operations and troubleshooting is layered localization.

Don't only look at VM, nor only look at Pod.

Standard chain:

    VM
      |
      v
    VMI
      |
      v
    virt-launcher Pod
      |
      v
    DataVolume / PVC / Network
      |
      v
    virt-handler / Node / KVM
      |
      v
    Guest OS

Different phenomena correspond to different focuses:

    1. VM Stopped:
        Check runStrategy, whether virtctl start is needed.

    2. VMI Pending:
        Check scheduling, resources, PVC, node, taint.

    3. virt-launcher ImagePullBackOff:
        Check image repository and containerd pull.

    4. DataVolume Failed:
        Check importer Pod logs, URL, PVC, StorageClass.

    5. PVC Bound but VM startup failure:
        Check virt-launcher logs, KVM, whether image is bootable.

    6. Console not accessible:
        Check VM/VMI, virt-api, virt-launcher, Guest OS.

    7. SSH not accessible:
        Check Service, Endpoints, sshd in VM, NodePort, firewall.

8. All VMs on a node are abnormal:
    Check virt-handler, /dev/kvm, kubelet, containerd.

Important commands:

    kubectl describe vm <vm-name> -n <namespace>

    kubectl describe vmi <vm-name> -n <namespace>

    kubectl describe pod <virt-launcher-pod-name> -n <namespace>

    kubectl logs <virt-launcher-pod-name> -n <namespace> --tail=200

    kubectl describe dv <dv-name> -n <namespace>

    kubectl logs <importer-pod-name> -n <namespace> --tail=200

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

Experience-based judgment:

    1. Check Events first, then check logs
    2. First confirm which layer (VM/VMI/Pod) is abnormal
    3. DataVolume issues check importer
    4. Storage issues check PVC, PV, VolumeAttachment, Longhorn
    5. Node issues check virt-handler and /dev/kvm
    6. Network issues check Service, Endpoints, VM internal ip addr
    7. In production environments, troubleshooting must protect PVC and data

Next suggested learning material:

    10-KubeVirt Interview Summary: Differences with vSphere, OpenStack, and Regular Pods.md