# 02-KubeVirt Core Architecture: VirtualMachine, VMI, virt-launcher, and Control Components

Recommended Path:

    04-Kubernetes/12-KubeVirt/02-KubeVirt Core Architecture: VirtualMachine, VMI, virt-launcher, and Control Components.md

Tags:

    #Kubernetes
    #KubeVirt
    #VirtualMachine
    #VMI
    #virt-launcher
    #virt-api
    #virt-controller
    #virt-handler
    #virt-operator
    #CloudlandVirtualization
    #PlatformEngineering

---

## I. Document Overview

This document is used to systematically understand KubeVirt's core architecture.

Key points to master:

    1. KubeVirt's overall architecture
    2. Differences between VirtualMachine and VirtualMachineInstance
    3. What is the virt-launcher Pod
    4. Responsibilities of virt-api, virt-controller, virt-handler, and virt-operator
    5. Basic workflow of KubeVirt virtual machine from creation to runtime
    6. How to observe KubeVirt resources using kubectl
    7. How to troubleshoot virtual machine issues using events and Pod relationships

Document positioning:

    Focus on architecture understanding + operational observation.
    Not an installation guide.
    Actual KubeVirt installation will be completed in subsequent installation notes.

Notes:

    If the current cluster has not yet installed KubeVirt, you can first read the conceptual sections.
    After installation, return to execute the experiments commands in this document to deepen understanding.

---

## II. KubeVirt Architecture Overview

KubeVirt is a virtualization extension on Kubernetes.

It integrates virtual machines into the Kubernetes resource model through CRD and Controller.

Overall relationship:

    User / Operations Personnel
          |
          v
    kubectl / virtctl
          |
          v
    Kubernetes API Server
          |
          v
    KubeVirt CRD
          |
          v
    KubeVirt Control Components
          |
          v
    virt-launcher Pod
          |
          v
    KVM / QEMU
          |
          v
    Virtual Machine

From Kubernetes perspective:

    KubeVirt is a set of CRD + Controller + Pod components.

From virtualization perspective:

    KubeVirt is a virtual machine management solution based on KVM / QEMU.

From platform perspective:

    KubeVirt enables Kubernetes to uniformly manage containers and virtual machines.

---

## III. KubeVirt Core Resource Objects

Common KubeVirt resource objects include:

    VirtualMachine
    VirtualMachineInstance
    VirtualMachineInstanceReplicaSet
    VirtualMachineInstanceMigration
    DataVolume
    VirtualMachineSnapshot
    VirtualMachineRestore

The most important ones in the initial stage are:

    VirtualMachine
    VirtualMachineInstance
    DataVolume

This document focuses on:

    VirtualMachine
    VirtualMachineInstance
    virt-launcher

---

## IV. What is VirtualMachine

VirtualMachine is abbreviated as VM.

It represents the desired state and configuration of a virtual machine.

It describes:

    1. Virtual machine name
    2. Whether it is running
    3. CPU configuration
    4. Memory configuration
    5. Disk configuration
    6. Network configuration
    7. Startup strategy
    8. cloud-init configuration
    9. Scheduling rules
    10. Runtime template

It can be understood as:

    VirtualMachine = Virtual machine definition object

Similar to Kubernetes's Deployment:

    Deployment describes the desired state of an application.
    VirtualMachine describes the desired state of a virtual machine.

VirtualMachine is not the virtual machine process itself.

It is more like:

    "I want there to be a virtual machine like this in the cluster."

---

## V. What is VirtualMachineInstance

VirtualMachineInstance is abbreviated as VMI.

It represents a running virtual machine instance.

It can be understood as:

    VMI = Running virtual machine instance

Analogous to native Kubernetes objects:

    Deployment -> Pod
    VirtualMachine -> VirtualMachineInstance

Relationship:

    VirtualMachine describes the desired state.
    VirtualMachineInstance represents the actual running state.

When a VM is started, a VMI is generated.

When a VM is stopped, the VMI typically disappears.

Therefore:

    VM can exist, but VMI may not.
    VMI exists, usually indicating the virtual machine is running.

---

## VI. Differences Between VM and VMI

| Object | Purpose | Is Long-Term | Analogy |
|---|---|---|---|
| VirtualMachine | Virtual machine definition and desired state | Usually long-term | Deployment |
| VirtualMachineInstance | Running virtual machine instance | Exists during runtime | Pod |

Simple understanding:

    VM is the virtual machine configuration.
    VMI is the running virtual machine.

Example:

    Created a VirtualMachine named ubuntu-vm.
    But it hasn't started yet.

At this time:

    kubectl get vm
        Can see ubuntu-vm

    kubectl get vmi
        May not show ubuntu-vm

After startup:

    kubectl get vm
        Can see ubuntu-vm

    kubectl get vmi
        Can also see ubuntu-vm

After stopping:

kubectl get vm
    Still can see ubuntu-vm

kubectl get vmi
    May not see ubuntu-vm

This is the core difference between VM and VMI.

---

## VII. What is virt-launcher Pod

KubeVirt virtual machines will eventually be hosted by a special Pod.

This Pod is usually called:

    virt-launcher

Each running virtual machine typically corresponds to a virt-launcher Pod.

The relationship is as follows:

    VirtualMachine
          |
          v
    VirtualMachineInstance
          |
          v
    virt-launcher Pod
          |
          v
    QEMU / KVM process
          |
          v
    Guest OS

In other words:

    KubeVirt virtual machines are not running natively outside Kubernetes.
    They run virtual machine processes through virt-launcher Pods on nodes.

From Kubernetes perspective:

    It is a Pod.

From virtualization perspective:

    It runs QEMU/KVM virtual machine processes inside.

---

## VIII. Why virt-launcher is needed

Kubernetes natively only recognizes Pods, not virtual machine processes directly.

KubeVirt needs to integrate the virtual machine runtime into Kubernetes management.

The role of virt-launcher is:

    1. Host virtual machine processes
    2. Allow virtual machines to be scheduled to nodes by Kubernetes
    3. Allow virtual machines to use Kubernetes network
    4. Allow virtual machines to mount PVC disks
    5. Allow virtual machines to be managed by kubelet for lifecycle
    6. Allow virtual machine status to reflect in Kubernetes objects

Therefore, when troubleshooting KubeVirt virtual machines, you cannot only look at VM and VMI.

You also need to check:

    virt-launcher Pod

Common commands:

    kubectl get pods -A | grep virt-launcher

    kubectl describe pod <virt-launcher-pod-name> -n <namespace>

    kubectl logs <virt-launcher-pod-name> -n <namespace>

---

## IX. KubeVirt Core Control Components

Common core components of KubeVirt:

    virt-operator
    virt-api
    virt-controller
    virt-handler
    virt-launcher

These components typically run in:

    kubevirt namespace

Common viewing commands:

    kubectl get pods -n kubevirt -o wide

---

## X. virt-operator

virt-operator is responsible for lifecycle management of KubeVirt components.

Main responsibilities:

    1. Install KubeVirt components
    2. Manage KubeVirt CRD
    3. Manage virt-api
    4. Manage virt-controller
    5. Manage virt-handler
    6. Handle KubeVirt component upgrades
    7. Ensure KubeVirt components are in desired state

It can be understood as:

    virt-operator = Operator for managing KubeVirt itself

If virt-operator fails, it may affect KubeVirt component upgrades, changes, and self-healing.

Check:

    kubectl get pods -n kubevirt | grep virt-operator

    kubectl logs -n kubevirt deploy/virt-operator --tail=100

---

## XI. virt-api

virt-api provides API capabilities for KubeVirt.

It handles requests related to virtual machine operations.

Examples:

    virtctl console
    virtctl start
    virtctl stop
    virtctl restart
    virtctl vnc

These commands usually interact with KubeVirt through virt-api.

It can be understood as:

    virt-api = API entry component for KubeVirt

If virt-api fails, you may see:

    1. virtctl cannot connect to console
    2. virtctl start / stop fails
    3. Virtual machine operation API calls fail

Check:

    kubectl get pods -n kubevirt | grep virt-api

    kubectl logs -n kubevirt deploy/virt-api --tail=100

---

## XII. virt-controller

virt-controller is one of the core controllers of KubeVirt.

It is responsible for controlling based on the desired state of VM / VMI.

Main responsibilities:

    1. Monitor VirtualMachine
    2. Monitor VirtualMachineInstance
    3. Create or delete VMI
    4. Coordinate virt-launcher Pod
    5. Handle VM start, stop, status changes
    6. Update VM / VMI status
    7. Handle migration, status synchronization, and other control logic

It can be understood as:

    virt-controller = Virtual machine lifecycle controller

Analogous to Kubernetes:

    Deployment Controller creates ReplicaSet / Pod based on Deployment.
    virt-controller creates VMI / virt-launcher related resources based on VM.

Check:

    kubectl get pods -n kubevirt | grep virt-controller

    kubectl logs -n kubevirt deploy/virt-controller --tail=100

---

## XIII. virt-handler

virt-handler is a node-side component.

It typically runs as a DaemonSet on each node.

Main responsibilities:

1. Runs on each node hosting a runnable virtual machine
    2. Manages the lifecycle of virtual machines on the node
    3. Interacts with native virtualization capabilities
    4. Manages virtual machine startup, shutdown, and status synchronization
    5. Monitors VMI on this node
    6. Coordinates with virt-launcher and host virtualization capabilities

    Can be understood as:

    virt-handler = Virtual machine manager on each node

    If a node's virt-handler fails, VMs on that node may not run properly.

    Check:

    kubectl get pods -n kubevirt -o wide | grep virt-handler

    Check a specific node's virt-handler:

    kubectl get pods -n kubevirt -o wide | grep <node-name>

    Check logs:

    kubectl logs -n kubevirt <virt-handler-pod-name> --tail=100

    ---
    
    ## FourteenI don't know.virt-launcher

    virt-launcher is the Pod corresponding to each running virtual machine.

    It does not necessarily reside in the kubevirt namespace.

    Typically shares the same namespace as the VM.

    For example, if a VM is in the default namespace:

    virt-launcher Pod is usually also in the default namespace.

    Check:

    kubectl get pods -n default | grep virt-launcher

    virt-launcher main responsibilities:

    1. Hosts virtual machine processes
    2. Mounts virtual machine disks
    3. Accesses Pod network
    4. Starts QEMU/KVM
    5. Allows VM to be managed by Kubernetes as a Pod

    When troubleshooting VMs, focus on:

    1. VM
    2. VMI
    3. virt-launcher Pod
    4. Pod Events
    5. virt-launcher logs
    6. Pod's node
    7. Node /dev/kvm
    8. PVC / DataVolume

    ---
    
    ## FifteenI don't know.KubeVirt Basic VM Startup Process

    The general workflow for creating and starting a VM:

    1. User creates VirtualMachine
    2. Kubernetes API Server saves VM object
    3. virt-controller detects VM expects to run
    4. virt-controller creates VMI
    5. Kubernetes Scheduler selects node for virt-launcher Pod
    6. kubelet starts virt-launcher Pod on target node
    7. virt-handler takes over VMI management on node
    8. virt-launcher starts QEMU/KVM virtual machine process
    9. VM enters Running state
    10. User accesses VM via virtctl console or SSH

    Flow diagram:

    kubectl apply vm.yaml
            |
            v
    VirtualMachine
            |
            v
    virt-controller
            |
            v
    VirtualMachineInstance
            |
            v
    virt-launcher Pod
            |
            v
    kubelet + virt-handler
            |
            v
    QEMU / KVM
            |
            v
    Guest OS Running

    ---
    
    ## SixteenI don't know.KubeVirt Basic VM Shutdown Process

    General workflow for stopping a VM:

    1. User executes virtctl stop or sets VM running=false
    2. VM expected state becomes stopped
    3. virt-controller processes state change
    4. VMI is deleted or enters termination process
    5. virt-launcher Pod is terminated
    6. Virtual machine process exits
    7. VM object remains
    8. PVC disk remains

    After stopping a VM:

    VM still exists
    VMI is usually not present
    virt-launcher Pod is usually not present
    PVC remains

    This is a critical judgment point.

    ---
    
    ## SeventeenI don't know.KubeVirt Architecture Compared to Kubernetes Native Resources

    | Kubernetes Native | KubeVirt |
    |---|---|
    | Deployment | VirtualMachine |
    | Pod | VirtualMachineInstance / virt-launcher Pod |
    | Controller Manager | virt-controller |
    | Node Agent | virt-handler |
    | kubectl | kubectl + virtctl |
    | PVC | VM disk PVC |
    | Service | VM service exposure |
    | Events | VM / VMI / Pod events |

    Note:

    VMI is not a regular Pod.
    virt-launcher Pod is the actual Pod scheduled by Kubernetes.
    VMI is an abstracted virtual machine runtime instance object from KubeVirt.

    ---
    
    ## EighteenI don't know.Experiment Design Notes

    This experiment observes relationships between KubeVirt architecture objects.

    Experimental prerequisites:

    1. KubeVirt has been installed
    2. virtctl has been installed
    3. At least one node in the cluster supports KVM
    4. Can create test VMs
    5. kubectl can access the cluster normally

    If KubeVirt hasn't been installed yet, you can skip the experiment section.

    Return to execute these notes after completing:

    03-KubeVirt Installation Preparation: KVM, node virtualization capabilities, storage and network checks.md
    04-KubeVirt Installation Practice: Operator, CRD, virtctl and component verification.md
    05-Create First Virtual Machine: VirtualMachine, VMI, console and basic operations.md

    ---
    
    ## NineteenI don't know.Experiment One: Check KubeVirt CRD

    Experiment objective:

    Confirm which Kubernetes resources are extended after KubeVirt installation.

    Execute: /think

kubectl get crd | grep kubevirt

Common resources include:

    virtualmachines.kubevirt.io
    virtualmachineinstances.kubevirt.io
    virtualmachineinstancereplicasets.kubevirt.io
    virtualmachineinstancemigrations.kubevirt.io
    kubevirts.kubevirt.io

Check API resources:

    kubectl api-resources | grep kubevirt

Focus on:

    virtualmachines
    virtualmachineinstances

Check abbreviations:

    kubectl api-resources | grep -E "virtualmachines|virtualmachineinstances"

Common abbreviations may include:

    vm
    vmi

Verification:

    kubectl get vm -A

    kubectl get vmi -A

Expected results:

    Able to recognize vm and vmi resources normally.

Notes:

    If kubectl get vm reports resource not found, it indicates KubeVirt CRD installation failed or the current cluster lacks KubeVirt.

---

## Twenty, Experiment Two: Check KubeVirt Control Plane Components

Experiment objective:

    Observe whether KubeVirt control plane components are running normally.

Execution:

    kubectl get pods -n kubevirt -o wide

Common components:

    virt-api
    virt-controller
    virt-handler
    virt-operator

Check Deployments:

    kubectl get deploy -n kubevirt

Check DaemonSets:

    kubectl get ds -n kubevirt

Key understanding:

    virt-api is typically a Deployment
    virt-controller is typically a Deployment
    virt-operator is typically a Deployment
    virt-handler is typically a DaemonSet

Check component logs:

    kubectl logs -n kubevirt deploy/virt-api --tail=50

    kubectl logs -n kubevirt deploy/virt-controller --tail=50

    kubectl logs -n kubevirt deploy/virt-operator --tail=50

Check virt-handler:

    kubectl get pods -n kubevirt -o wide | grep virt-handler

Select a specific virt-handler Pod:

    kubectl logs -n kubevirt <virt-handler-pod-name> --tail=50

Expected results:

    KubeVirt key components are in Running state.
    virt-handler is distributed on nodes capable of running virtual machines.

---

## Twenty-one, Experiment Three: Observe the Relationship Between VM, VMI, and virt-launcher

Experiment objective:

    Observe which Kubernetes objects correspond to a running virtual machine.

Prerequisites:

    A test VM has been created and started.

Assumed VM name:

    testvm

Namespace:

    default

Check VM:

    kubectl get vm -n default

Check VMI:

    kubectl get vmi -n default

Check Pod:

    kubectl get pods -n default -o wide | grep virt-launcher

Check detailed information:

    kubectl describe vm testvm -n default

    kubectl describe vmi testvm -n default

Find virt-launcher Pod:

    kubectl get pods -n default -o wide | grep testvm

Check virt-launcher Pod:

    kubectl describe pod <virt-launcher-pod-name> -n default

Expected relationships:

    VM exists
    VMI exists
    virt-launcher Pod exists
    virt-launcher Pod is in Running state
    VMI and virt-launcher Pod are on the same node

Record:

    VM name
    VMI name
    virt-launcher Pod name
    Node
    Pod IP
    PVC name

---

## Twenty-two, Experiment Four: Stop VM and Observe Changes in VMI and virt-launcher

Experiment objective:

    Understand changes in VM, VMI, and virt-launcher when VM is stopped.

Stop VM:

    virtctl stop testvm -n default

Observe VM:

    kubectl get vm testvm -n default

Observe VMI:

    kubectl get vmi -n default

Observe virt-launcher Pod:

    kubectl get pods -n default | grep virt-launcher

Expected results:

    VM object still exists.
    VMI object disappears.
    virt-launcher Pod disappears or enters Terminating state and then disappears.
    PVC still exists.

Check PVC:

    kubectl get pvc -n default

Understanding:

    Stopping VM does not equal deleting VM.
    Stopping VM does not equal deleting the disk.
    VM definition and PVC data are still preserved.

---

## Twenty-three, Experiment Five: Start VM and Observe Object Re-creation

Experiment objective:

    Understand how VMI and virt-launcher are re-created when VM starts.

Start VM:

    virtctl start testvm -n default

Observation:

    kubectl get vm testvm -n default

    kubectl get vmi testvm -n default

kubectl get pods -n default -o wide | grep virt-launcher

Check events:

    kubectl describe vm testvm -n default

    kubectl describe vmi testvm -n default

Expected results:

    VMI reappears.
    virt-launcher Pod reappears.
    VM enters Running state.

If startup fails, continue checking:

    kubectl describe vmi testvm -n default

    kubectl get events -n default --sort-by=.lastTimestamp

    kubectl describe pod <virt-launcher-pod-name> -n default

---

## 24. Experiment Six: Reverse Lookup VM from virt-launcher Pod

Experiment goal:

    Become familiar with reverse lookup VM/VMI from Pod during troubleshooting.

Check virt-launcher Pod:

    kubectl get pods -n default | grep virt-launcher

Check Pod labels:

    kubectl get pod <virt-launcher-pod-name> -n default --show-labels

Check Pod ownerReferences:

    kubectl get pod <virt-launcher-pod-name> -n default -o yaml | grep -A20 ownerReferences

Check related resources:

    kubectl get vmi -n default

    kubectl get vm -n default

Understanding:

    When you only see one virt-launcher Pod anomaly, you can reverse lookup which VM it belongs to via labels, ownerReferences, and Pod name.

Troubleshooting scenarios:

    1. virt-launcher Pod Pending
    2. virt-launcher Pod CrashLoopBackOff
    3. VM fails to start
    4. VMI status anomaly
    5. VM anomaly on certain node

---

## 25. Experiment Seven: Observing VM Event Chain

Experiment goal:

    Learn to troubleshoot KubeVirt VM lifecycle issues through Events.

Check VM details:

    kubectl describe vm testvm -n default

Check VMI details:

    kubectl describe vmi testvm -n default

Check namespace events:

    kubectl get events -n default --sort-by=.lastTimestamp

Filter by VM name:

    kubectl get events -n default --sort-by=.lastTimestamp | grep testvm

Common event types:

    SuccessfulCreate
    Started
    Stopped
    FailedCreate
    FailedScheduling
    FailedMount
    FailedAttachVolume

Troubleshooting approach:

    If VMI not created:
        Check VM Events and virt-controller logs.

    If virt-launcher Pod Pending:
        Check Pod Events, scheduling, resources, PVC.

    If Pod created but VM not started:
        Check virt-launcher logs, virt-handler logs, node KVM capabilities.

---

## 26. Experiment Eight: Observing VM's Node

Experiment goal:

    Understand VM will eventually run on Kubernetes Node.

Check VMI:

    kubectl get vmi testvm -n default -o wide

Check virt-launcher Pod:

    kubectl get pod -n default -o wide | grep virt-launcher

Record NODE:

    <node-name>

Check node:

    kubectl describe node <node-name>

Check KVM on node:

    ls -l /dev/kvm

    egrep -c '(vmx|svm)' /proc/cpuinfo

Understanding:

    VM runs on specific Kubernetes Node.
    Node must have virtualization capabilities.
    If node lacks KVM support, VM may fail to start or suffer severe performance issues.

---

## 27. Common KubeVirt Object Inspection Commands

Check VM:

    kubectl get vm -A

Check VMI:

    kubectl get vmi -A

Check virt-launcher:

    kubectl get pods -A | grep virt-launcher

Check KubeVirt components:

    kubectl get pods -n kubevirt -o wide

Check KubeVirt CR:

    kubectl get kubevirt -n kubevirt

Check events:

    kubectl get events -A --sort-by=.lastTimestamp

Check specific VM:

    kubectl describe vm <vm-name> -n <namespace>

Check specific VMI:

    kubectl describe vmi <vm-name> -n <namespace>

Check virt-launcher Pod:

    kubectl describe pod <virt-launcher-pod-name> -n <namespace>

Check virt-launcher logs:

    kubectl logs <virt-launcher-pod-name> -n <namespace>

---

## 28. KubeVirt Troubleshooting Object Order

When troubleshooting VM issues, don't focus on a single object only.

Recommended order:

1. kubectl get vm  
2. kubectl describe vm  
3. kubectl get vmi  
4. kubectl describe vmi  
5. kubectl get pod | grep virt-launcher  
6. kubectl describe pod virt-launcher  
7. kubectl logs virt-launcher  
8. kubectl get pvc  
9. kubectl describe pvc  
10. kubectl get events  
11. §§code_0§§  
12. §§code_1§§  
13. §§code_2§§  

Troubleshooting Flow:  

    Is VM definition normal?  
        |  
        v  
    Is VMI created?  
        |  
        v  
    Is virt-launcher Pod created?  
        |  
        v  
    Is Pod scheduled to node?  
        |  
        v  
    Is PVC mounted successfully?  
        |  
        v  
    Does node support KVM?  
        |  
        v  
    Is QEMU / Guest OS started successfully?  

---

## Twenty-Nine, Common Architecture Issues  

### 29.1 VM Exists, but VMI Does Not Exist  

Possible Causes:  

    1. VM is not started  
    2. VM running=false  
    3. VMI is deleted after virtctl stop  
    4. VM failed to start, VMI not created successfully  

Checks:  

    kubectl describe vm <vm-name> -n <namespace>  

    kubectl get events -n <namespace> --sort-by=.lastTimestamp  

---

### 29.2 VMI Exists, but virt-launcher Pod is Pending  

Possible Causes:  

    1. Node resource insufficient  
    2. PVC not Bound  
    3. nodeSelector mismatch  
    4. taints / tolerations mismatch  
    5. Node does not support virtualization  
    6. Storage topology not satisfied  

Checks:  

    kubectl describe vmi <vm-name> -n <namespace>  

    kubectl get pod -n <namespace> | grep virt-launcher  

    kubectl describe pod <virt-launcher-pod-name> -n <namespace>  

---

### 29.3 virt-launcher Running, but VM Not Started Normally  

Possible Causes:  

    1. Image cannot start  
    2. Disk image corrupted  
    3. cloud-init configuration error  
    4. Guest OS startup failure  
    5. KVM / QEMU related issues  
    6. Resource insufficient  

Checks:  

    kubectl logs <virt-launcher-pod-name> -n <namespace>  

    virtctl console <vm-name> -n <namespace>  

    kubectl describe vmi <vm-name> -n <namespace>  

---

### 29.4 VMs on a Node Are All Abnormal  

Possible Causes:  

    1. virt-handler on the node is abnormal  
    2. /dev/kvm on the node is abnormal  
    3. containerd / kubelet on the node is abnormal  
    4. Storage mount on the node is abnormal  
    5. Network on the node is abnormal  
    6. Node resource insufficient  

Checks:  

    kubectl get pods -n kubevirt -o wide | grep <node-name>  

    kubectl logs -n kubevirt <virt-handler-pod-name> --tail=100  

    systemctl status kubelet --no-pager  

    systemctl status containerd --no-pager  

    ls -l /dev/kvm  

---

## Thirty, Relationship Between KubeVirt and Kubernetes Scheduling  

KubeVirt virtual machines are ultimately scheduled to nodes through virt-launcher Pods.  

Therefore, VM scheduling is also affected by Kubernetes scheduling rules.  

Includes:  

    1. CPU requests  
    2. Memory requests  
    3. nodeSelector  
    4. nodeAffinity  
    5. podAffinity  
    6. taints / tolerations  
    7. PVC topology  
    8. Node resources  
    9. Node status  
    10. Node virtualization capability  

Thus, when a VM fails to start, it could also be a regular Kubernetes scheduling issue.  

Examples:  

    insufficient cpu  
    insufficient memory  
    had untolerated taint  
    node(s) didn't match node selector  
    pod has unbound immediate PersistentVolumeClaims  

Troubleshooting these issues is similar to regular Pods.  

---

## Thirty-One, Relationship Between KubeVirt and PVC  

Virtual machine disks typically use PVCs.  

Therefore, KubeVirt heavily relies on Kubernetes storage capabilities.  

Common disk sources:  

    1. PVC  
    2. DataVolume  
    3. containerDisk  
    4. emptyDisk  
    5. cloudInitNoCloud  

Common in production:  

    DataVolume imports image  
        |  
        v  
    PVC as system disk  
        |  
        v  
    VM mounts PVC to start  

If PVC is abnormal, VM may fail to start.  

Common issues:  

    1. PVC Pending  
    2. DataVolume Import failed  
    3. StorageClass does not exist  
    4. CSI abnormal  
    5. Longhorn Volume abnormal  
    6. NFS does not support expected access mode  
    7. Disk image format error  

Troubleshooting: §§code_3§§

kubectl get pvc -n <namespace>

kubectl describe pvc <pvc-name> -n <namespace>

kubectl get dv -n <namespace>

kubectl describe dv <data-volume-name> -n <namespace>

---

## Thirty-two, The Relationship Between KubeVirt and Networking

KubeVirt virtual machines can use Kubernetes Pod network by default.

Common network configurations:

    1. masquerade
    2. bridge
    3. slirp
    4. Multus multi-network interface
    5. SR-IOV

In the initial stage, understand:

    VM can be scheduled to nodes like Pods.
    VM can obtain connectivity through Pod network.
    VM can expose ports through Service.
    Multi-network interface and access to traditional Layer 2 networks typically require Multus.

If VM network is not working, troubleshooting should include:

    1. VMI network configuration
    2. virt-launcher Pod network
    3. CNI
    4. Service
    5. Multus network attachment definition
    6. Guest OS internal network interface configuration
    7. Firewall
    8. Routing

---

## Thirty-three, The Relationship Between KubeVirt and virtctl

kubectl is responsible for operating Kubernetes resources.

virtctl is the dedicated command-line tool provided by KubeVirt.

Common operations:

    virtctl start <vm-name>
    virtctl stop <vm-name>
    virtctl restart <vm-name>
    virtctl console <vm-name>
    virtctl vnc <vm-name>

kubectl is more resource management oriented:

    kubectl get vm
    kubectl get vmi
    kubectl describe vm
    kubectl apply -f vm.yaml

virtctl is more focused on virtual machine operation experience:

    Power on
    Power off
    Restart
    Enter console
    Open VNC
    Migration

Both need to be mastered.

---

## Thirty-four, Impact Scope of KubeVirt Component Failures

| Component | Impact of Failure |
|---|---|
| virt-operator | KubeVirt installation, upgrades, and component lifecycle anomalies |
| virt-api | virtctl operations, console, API subresource anomalies |
| virt-controller | VM / VMI status control anomalies |
| virt-handler | Node-side VM management anomalies |
| virt-launcher | Single VM runtime anomalies |
| CDI | Image import, DataVolume anomalies |
| CSI | VM disk creation, mount anomalies |
| CNI | VM network anomalies |

Troubleshooting approach:

    Single VM anomaly:
        Prioritize checking VM / VMI / virt-launcher / PVC.

    VM anomalies on a specific node:
        Prioritize checking virt-handler / node KVM / kubelet / containerd.

    All VM operations are anomalous:
        Prioritize checking virt-api / virt-controller / kubevirt components.

    All image imports are anomalous:
        Prioritize checking CDI.

    All disk mounts are anomalous:
        Prioritize checking CSI / StorageClass / backend storage.

---

## Thirty-five, Interview Answer: KubeVirt Core Architecture

You can answer like this:

    KubeVirt is a solution that integrates virtual machines into Kubernetes management through CRD and Controller.
    After users create a VirtualMachine object, KubeVirt will create a VirtualMachineInstance according to the desired state.
    A running virtual machine will eventually correspond to a virt-launcher Pod.
    The virt-launcher Pod runs on a Kubernetes node and hosts the QEMU/KVM virtual machine process.
    KubeVirt's core control components include virt-operator, virt-api, virt-controller, and virt-handler.
    virt-controller handles VM/VMI control logic, virt-handler manages virtual machines on the node side, virt-api provides KubeVirt API capabilities, and virt-operator manages the lifecycle of KubeVirt components.

---

## Thirty-six, Interview Answer: Difference Between VM and VMI

You can answer like this:

    VirtualMachine is the desired state and configuration object of a virtual machine, similar to Deployment.
    VirtualMachineInstance is the running virtual machine instance, similar to a running Pod.
    A VM will create a VMI when it starts.
    When a VM stops, the VMI usually disappears, but the VM object and disk PVC remain.
    Therefore, when troubleshooting KubeVirt, you need to check VM, VMI, and virt-launcher Pod simultaneously.

---

## Thirty-seven, Interview Answer: What is virt-launcher

You can answer like this:

    virt-launcher is the Pod that hosts the virtual machine process in KubeVirt.
    Each running virtual machine typically corresponds to a virt-launcher Pod.
    Kubernetes schedules this virt-launcher Pod, and the virtual machine process runs inside it.
    The underlying process runs the Guest OS through QEMU/KVM.
    Therefore, from a Kubernetes perspective, it is a Pod, but from a business perspective, it is a virtual machine.

---

## Thirty-eight, Common Misconceptions

### 38.1 Misconception One: VM is Equivalent to Pod

Inaccurate.

VM is the virtual machine definition object in KubeVirt.

virt-launcher is the Pod that hosts the virtual machine process.

---

### 38.2 Misconception Two: VMI and VM Are the Same Thing

Inaccurate.

VM is the definition and desired state.
VMI is the runtime instance.

VM can exist without VMI.

---

### 38.3 Misconception Three: Only Checking VM Status is Enough

Insufficient.

When troubleshooting, at least check:

    VM
    VMI
    virt-launcher Pod
    PVC / DataVolume
    Events
    virt-handler
    virt-controller

---

### 38.4 Mistake Four: KubeVirt Can Run VMs Once Installed

Not necessarily.

Also required:

    Node supports KVM
    /dev/kvm available
    Storage available
    Network available
    virt-handler normal
    virt-launcher can schedule
    Image can be imported
    PVC can be mounted

---

## Thirty-Nine, Summary of This Article

KubeVirt's core architecture can be summarized as:

    CRD defines virtual machine resources
    Controller manages virtual machine state
    virt-launcher Pod hosts virtual machine processes
    KVM/QEMU providesBottom virtualization capabilities
    PVC provides virtual machine disk
    CNI provides virtual machine network

Core object relationships:

    VirtualMachine
        Describes the desired state of a virtual machine

    VirtualMachineInstance
        Represents a running virtual machine instance

    virt-launcher Pod
        Actually hosts virtual machine processes

    virt-handler
        Node-side virtual machine management component

    virt-controller
        Manages VM / VMI state

    virt-api
        Provides KubeVirt API capabilities

    virt-operator
        Manages KubeVirt component lifecycle

Most important architecture chain:

    VM -> VMI -> virt-launcher Pod -> QEMU/KVM -> Guest OS

Most important troubleshooting chain:

    kubectl get vm
    kubectl describe vm
    kubectl get vmi
    kubectl describe vmi
    kubectl get pod | grep virt-launcher
    kubectl describe pod virt-launcher
    kubectl logs virt-launcher
    kubectl get pvc
    kubectl get events

Next suggested reading:

    03-KubeVirt Installation Preparation: KVM, Node Virtualization Capabilities, Storage and Network Check.md