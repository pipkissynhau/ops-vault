# 09-KubeVirt Operational Troubleshooting: Failed VM Startup, Failed Image Import, Console and Event Analysis

Recommended Location:

    04-Kubernetes/12-KubeVirt/09-KubeVirt Operational Troubleshooting: Failed VM Startup, Failed Image Import, Console and Event Analysis.md

Tags:

    #Kubernetes
    #KubeVirt
    #VM Troubleshooting
    #VMI
    #virt-launcher
    #CDI
    #DataVolume
    #PVC
    #Console
    #Event Analysis
    #Cloud-Native Virtualization
    #Platform Engineering

---

## I. Document Overview

This document outlines common operational troubleshooting methods for KubeVirt, focusing on:

    1. Failed VM creation
    2. Failed VM startup
    3. VMI remaining in Pending state
    4. Issues with the virt-launcher Pod
    5. Failed image import
    6. DataVolume errors
    7. Failed PVC mounting
    8. Unreachable Console
    9. SSH access issues
    10. VM network connectivity problems
    11. Problems related to stopping, deleting, and rebuilding VMs
    12. How to quickly identify issues using Events

Objectives of this document:

    1. Establish a standard troubleshooting process for KubeVirt
    2. Determine the level of the issue based on the status of VMs, VMI, virt-launcher, PVCs, and DataVolumes
    3. Understand common error scenarios and corresponding inspection commands
    4. Be able to handle basic KubeVirt operational issues
    5. Clearly explain KubeVirt troubleshooting approaches during interviews

Applicable Scenarios:

    1. Self-built Kubernetes clusters using kubeadm
    2. KubeVirt + CDI + Longhorn environments
    3. Private cloud platforms
    4. Hybrid container and virtual machine platforms
    5. KubeVirt from beginner to basic operational levels

---

## II. Core Troubleshooting Approach for KubeVirt

Troubleshooting KubeVirt cannot be limited to a single object.

For general Pod troubleshooting, check:

    Pod
    Events
    Logs
    Service
    PVC

For KubeVirt troubleshooting, additional objects to check include:

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
    Internal status of the Guest OS

Standard Process:

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

    1. Check the status of the VM first.
    2. Then check the status of the VMI.
    3. Next, check the virt-launcher Pod.
    4. Then check the PVC/DataVolume.
    5. After that, check the KubeVirt control components.
    6. Follow up by checking the node's KVM, kubelet, and containerd.
    7. Finally, investigate any internal issues within the Guest OS.

---

## III. Common Object Relationships in KubeVirt

| Object | Function | Common Issues |
|---|---|---|
| VirtualMachine | Defines and manages the virtual machine state | Failed to start, configuration errors, abnormal status |
| VirtualMachineInstance | Represents a running virtual machine instance | Pending, Scheduling, Running, Failed |
| virt-launcher Pod | Houses the processes that manage the virtual machine | Pending, ImagePullBackOff, CrashLoopBackOff |
| DataVolume | Handles image import and disk preparation | Import failure, Pending, Failed |
| PVC | Links the virtual machine to physical storage | Pending, mounting failure, insufficient capacity |
| PV | Represents actual storage volumes | Bound, Released, Failed |
| virt-controller | Manages VM/VMI operations | Abnormal VM status control |
| virt-handler | Handles node-side VM management | Issues with a specific node's VMs |
| virt-api | Provides access to the KubeVirt API | Errors when using virtctl console/start/stop |
| CDI importer | Imports images into DataVolumes | Image download failures, writing issues to PVC |

---

## IV. Pre-Troubleshooting Commands

### 4.1 Check KubeVirt Components

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

    kubectl -n kubevirt get deploy```markdown
kubectl -n kubevirt-demo get vmi vm-cirros-dv -o wide

or:

kubectl -n kubevirt-demo get pods -o wide | grep virt-launcher

Log in to the node and check the following:

ls -l /dev/kvm

egrep -c '(vmx|svm)' /proc/cpuinfo

systemctl status kubelet --no-pager

systemctl status containerd --no-pager

journalctl -u kubelet -n 100 --no-pager

---

## Section 6: Issue 1: VM Remains Stopped After Creation

### 6.1 Observation

Run the following command:

kubectl get vm -n kubevirt-demo

The output might look like this:

NAME           AGE   STATUS    READY
vm-cirros      1m    Stopped   False

### 6.2 Common Causes

If the VM YAML configuration includes:

runStrategy: Manual

then the VM will not start automatically after creation.

This is a normal behavior and not considered a fault.

### 6.3 Solutions

Start the VM manually:

virtctl start vm-cirros -n kubevirt-demo

Then check again:

kubectl get vm -n kubevirt-demo

kubectl get vmi -n kubevirt-demo

kubectl get pods -n kubevirt-demo

### 6.4 Verification

If the VM changes from Stopped to Running, it indicates that the issue has been resolved.

If the manual start fails, further investigate issues with the VMI, virt-launcher, and Events.
---

## Section 7: Issue 2: virtctl Start Fails

### 7.1 Observation

When attempting to start the VM using virtctl:

virtctl start vm-cirros -n kubevirt-demo

You may encounter errors such as:

error starting VirtualMachine
server could not find the requested resource
connection refused
unauthorized
Internal error occurred

### 7.2 Troubleshooting virtctl

Check the version of virtctl by running:

which virtctl

virtctl version

### 7.3 Checking the KubeVirt API

Run the following commands:

kubectl -n kubevirt get pods | grep virt-api

kubectl -n kubevirt get svc

kubectl -n kubevirt logs deploy/virt-api --tail=100

### 7.4 Checking KubeVirt Status

Run:

kubectl -n kubevirt get kv kubevirt

kubectl -n kubevirt describe kv kubevirt

### 7.5 Common Causes

1. The virt-api service is not running.
2. The required KubeVirt CRs are not available.
3. The version of virtctl does not match the KubeVirt requirements.
4. The current kubeconfig file is incorrect.
5. The current user lacks the necessary permissions.
6. The KubeVirt CRDs are not fully installed.
---
## Section 8: Issue 3: VMI Remains in a Pending State

### 8.1 Observation

Run:

kubectl get vmi -n kubevirt-demo

The output might show:

NAME           AGE   PHASE
vm-cirros      5m    Pending

### 8.2 Checking VMI Details

Use:

kubectl describe vmi vm-cirros -n kubevirt-demo

Also, check the Events by running:

kubectl get events -n kubevirt-demo --sort-by=.lastTimestamp

### 8.3 Common Causes

1. The node lacks sufficient resources.
2. The node does not have the /dev/kvm directory.
3. The node does not support virtualization.
4. The PVC has not been bound to the VMI.
5. The DataVolume creation is incomplete.
6. The node selector criteria are not matched.
7. There are conflicts between taints and tolerations.
8. There are issues with the virt-handler process.
9. The virt-launcher Pod cannot be scheduled.

### 8.4 Checking the virt-launcher Pod

Run:

kubectl get pods -n kubevirt-demo -o wide | grep virt-launcher

If the pod is also in a Pending state, check its Events carefully:

Common error messages include:

insufficient cpu resources
insufficient memory
pod has unbound PersistentVolumeClaims
node(s) have untolerated taints
node(s) do not match the node selector
0/3 nodes are available

### 8.5 Solutions

If resources are insufficient:

Reduce the VM's memory and CPU requirements.
Expand the node’s capacity.
Remove unnecessary loads from the node.

If the PVC has not been bound:

Check the configuration of PVC, StorageClass, and related services.

If the node does not support KVM virtualization:

Verify the presence of /dev/kvm and check vmx/svm capabilities.
Consider using other virtualization technologies if necessary.

---
## Section 9: Issue 4: virt-launcher Pod Experiences ImagePullBackOffPlace the image on Nginx at 10.0.0.10 or in object storage.

PVC Pending:

    kubectl get storageclass
    kubectl describe pvc
    Check Longhorn / NFS / CSI

Insufficient capacity:

    Increase pvc.resources.requests.storage in DataVolume

---

## Section Eleven: Issue Six: DataVolume Failed

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

### 11.3 Common Error Sources

    1. http status 404
    2. connection timeout
    3. no route to host
    4. certificate signed by unknown authority
    5. no space left on device
    6. PVC mount failed
    7. image format not recognized
    8. importer container OOMKilled

### 11.4 Solutions

    1. Verify that the URL is accessible.
    2. Use an internal image source.
    3. Adjust the PVC capacity.
    4. Check the StorageClass.
    5. Inspect the CDI Pod.
    6. Review the importer Pod logs.
    7. Recreate the DataVolume.

---

## Section Twelve: Issue Seven: PVC Bound but VM Failed to Start

### 12.1 Symptoms

The PVC is bound:

    kubectl get pvc -n kubevirt-demo

But the VM cannot start.

### 12.2 Check the VM / VMI

    kubectl describe vm <vm-name> -n kubevirt-demo

    kubectl describe vmi <vm-name> -n kubevirt-demo

### 12.3 Verify virt-launcher

    kubectl get pods -n kubevirt-demo -o wide | grep virt-launcher

    kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo

    kubectl logs <virt-launcher-pod-name> -n kubevirt-demo --tail=100

### 12.4 Common Causes

    1. The disk image cannot be started.
    2. The image format is incorrect.
    3. The system disk bus configuration is inappropriate.
    4. There are errors in the cloud-init configuration.
    5. The node does not support KVM.
    6. The node lacks sufficient resources.
    7. The PVC mounting failed.
    8. An exception occurred during Longhorn Volume attachment.
    9. There is an issue with virt-handler.

### 12.5 Steps to Resolve

First, confirm:

    Whether the DataVolume was successfully created.
    Whether the PVC is bound.
    Whether the virt-launcher Pod is running.
    Whether /dev/kvm exists on the node.
    Whether virt-handler is functioning properly.
    Whether the Guest OS image can be started.

---

## Section Thirteen: Issue Eight: FailedMount / FailedAttachVolume

### 13.1 Symptoms

Events in the virt-launcher Pod show:

    FailedMount
    FailedAttachVolume
    Unable to attach or mount volumes
    MountVolume.SetUp failed
    Timed out waiting for the condition

### 13.2 Check the Pod

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

    1. The Longhorn Volume cannot be attached.
    2. iscsid is not running.
    3. open-iscsi is not installed```markdown
kubectl get pods -n <namespace> --show-labels

The Service selector should match:

kubevirt.io/domain=<vm-name>

### 15.4 Endpoints are present but SSH connection fails

To access the VM:

virtctl console <vm-name> -n <namespace>

Check inside the VM:

ip addr

ip route

ps aux | grep ssh

netstat -lntp

ss -lntp

Common causes:

1. sshd is not running
2. cloud-init does not enable password login
3. The VM's firewall is blocking connections
4. Incorrect username or password
5. Incorrect Service targetPort
6. NodePort is blocked by the firewall
7. kube-proxy is malfunctioning

### 15.5 Checking NodePort

sudo ipvsadm -Ln | grep <nodePort>

sudo ufw status

kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
---
## Section Sixteen: Issue Eleven: VM Network Issues

### 16.1 Symptoms

- Unable to access the outside world from within the VM
- DNS resolution fails inside the VM
- External access is denied to the VM
- The second network interface of the VM is not functioning

### 16.2 Checking VMI Network

kubectl get vmi <vm-name> -n <namespace> -o wide

kubectl describe vmi <vm-name> -n <namespace>

kubectl get vmi <vm-name> -n <namespace> -o yaml | grep -A50 interfaces

### 16.3 Checking the virt-launcher Pod Network

kubectl get pod -n <namespace> -o wide | grep virt-launcher

### 16.4 Accessing the VM

virtctl console <vm-name> -n <namespace>

Inside the VM, perform the following checks:

ip addr

ip route

cat /etc/resolv.conf

ping -c 3 <gateway-ip>

ping -c 3 <kubernetes-service-ip>

### 16.5 Checking CoreDNS

kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

kubectl -n kube-system get svc kube-dns

kubectl -n kube-system get endpoints kube-dns

### 16.6 Checking CNI

kubectl get pods -A -o wide | grep -E "calico|flannel|cilium"

### 16.7 Checking Services

kubectl get svc -n <namespace>

kubectl get endpoints -n <namespace>

### 16.8 Additional Checks for Multus Scenarios

kubectl get pods -A | grep -i multus

kubectl get net-attach-def -A

kubectl describe net-attach-def <nad-name> -n <namespace>

kubectl describe vmi <vm-name> -n <namespace>

On the node:

ls -l /opt/cni/bin/

ip addr

ip route
---
## Section Seventeen: Issue Twelve: PVC Remains After VM Deletion

### 17.1 Symptoms

After deleting a VM:

kubectl delete vm <vm-name> -n <namespace>

The PVC still exists when checked:

kubectl get pvc -n <namespace>

### 17.2 Explanation

This is a common and expected behavior.

A virtual machine definition and its associated disks are not the same thing.

Deleting a VM does not necessarily result in the deletion of its PVC.

### 17.3 Risks

In production environments, PVCs should not be deleted carelessly.

PVCs may contain:

1. The VM's system disk
2. Data disks
3. Business data
4. Database files
5. User-uploaded files

### 17.4 Pre-deletion Checks

Before deleting a PVC, ensure:

kubectl get vm -n <namespace>

kubectl get vmi -n <namespace>

kubectl get pvc -n <namespace>

kubectl describe pvc <pvc-name> -n <namespace>

kubectl get pv

kubectl describe pv <pv-name>

Confirm that:

1. No VMs are using the PVC
2. The data does not need to be retained
3. Backups have been made
4. The StorageClass's reclaimPolicy meets requirements
---
## Section Eighteen: Issue Thirteen: VMs Cannot Be Scheduled to a Specific Node

### 18.1 Symptoms

The VMI shows "Pending", and the following errors appear in Events:

- The node(s) do not match the node selector.
- The node(s) have untolerated taints.
- Insufficient CPU resources.
- Insufficient memory.

### 18.2 Checking the VM YAML File

kubectl get vm <vm-name> -n <namespace> -o yaml | grep -A40 nodeSelector

kubectl    kubectl -n kubevirt logs deploy/virt-operator --tail=200

### 20.4 Viewing Events

    kubectl -n kubevirt get events --sort-by=.lastTimestamp

### 20.5 Common Causes

    1. virt-api Exception
    2. virt-controller Exception
    3. KubeVirt CR Degradation
    4. webhook Exception
    5. APIService Exception
    6. Certificate Issues
    7. Failure to Pull KubeVirt Component Images
    8. Namespace Security Policy Restrictions

---

## Chapter Twenty-One: Event Analysis Methods

Kubernetes Events are one of the key troubleshooting tools.

Common methods used for troubleshooting with KubeVirt include:

    kubectl describe vm
    kubectl describe vmi
    kubectl describe pod
    kubectl describe dv
    kubectl describe pvc
    kubectl get events

### 21.1 Sorting by Time

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

### 21.2 Filtering by VM Name

    kubectl get events -n <namespace> --sort-by=.lastTimestamp | grep <vm-name>

### 21.3 Filtering for Exceptions

    kubectl get events -n <namespace> --sort-by=.lastTimestamp | grep -E "Warning|Failed|Error"

### 21.4 Common Event Meanings

| Event          | Common Meaning                                      |
|-----------------|-------------------------------------------------------------|
| FailedScheduling | Scheduling failed                                    |
| FailedMount      | Volume mounting failed                                   |
| FailedAttachVolume | Volume attachment failed                                  |
| ErrImagePull     | Image pull failed                                        |
| ImagePullBackOff | Continuous image pull failure                          |
| ImportInProgress | DataVolume import in progress                         |
| ImportFailed       | DataVolume import failed                                    |
| Started         | VM started                                                    |
| Stopped          | VM stopped                                                     |
| SuccessfulCreate | Controller successfully created resources                |

---

## Chapter Twenty-Two: Log Viewing Methods

### 22.1 virt-launcher Logs

    kubectl logs <virt-launcher-pod-name> -n <namespace> --tail=200

Suitable for troubleshooting:

    VM startup failures
    QEMU issues
    Disk problems
    Guest OS startup issues

---

### 22.2 virt-controller Logs

    kubectl -n kubevirt logs deploy/virt-controller --tail=200

Suitable for troubleshooting:

    VM/VMI control exceptions
    VMI creation failures
    Status synchronization issues

---

### 22.3 virt-api Logs

    kubectl -n kubevirt logs deploy/virt-api --tail=200

Suitable for troubleshooting:

    virtctl console errors
    start/stop API failures
    Subresource access issues

---

### 22.4 virt-handler Logs

First, find the corresponding virt-handler node:

    kubectl -n kubevirt get pods -o wide | grep <node-name>

View logs:

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=200

Suitable for troubleshooting:

    Node-side VM management issues
    KVM-related problems
    Issues with specific nodes' VMs

---

### 22.5 CDI importer Logs

    kubectl logs <importer-pod-name> -n <namespace> --tail=200

Suitable for troubleshooting:

    Image import failures
    HTTP download issues
    PVC write failures

---

## Chapter Twenty-Three: Experiment One: Simulating a VM Not Starting

### 23.1 Creating a Manual VM

If the VM is configured with:

    runStrategy: Manual

It will not start automatically after creation.

To check:

    kubectl get vm -n kubevirt-demo

### 23.2 Observing the Status

    kubectl describe vm <vm-name> -n kubevirt-demo

    kubectl get vmi -n kubevirt-demo

    kubectl get pods -n kubevirt-demo

Expected results:

    The VM exists.
    The VMI does not exist.
    The virt-launcher Pod does not exist.

### 23.3 Starting the VM

    virtctl start <vm-name> -n kubevirt-demo

Observations after starting:

    kubectl get vm,vmi,pod -n kubevirt-demo

Key points to learn:

    The existence of a VM does not mean it is running.
    The presence of a VMI usually indicates that the virtual machine is running.

---

## Chapter Twenty-Four: Experiment Two: Simulating a Service Selector Error

### 24.1 Creating an Incorrect Service

Example of an intentionally incorrect selector:

    selector:
      kubevirt.io/domain: wrong-vm-name

After applying it, check:

    kubectl get endpoints <svc| SSH Connection Failed | svc / endpoints / sshd within the VM | Selectors, sshd, firewall settings |
| VM Network Issue | VMI / IP address configuration within the VM | CNI, DNS, Service, Multus configurations |
| Abnormalities in Specific VMs | virt-handler on those nodes | KVM, kubelet, containerd components |
| General Operational Issues with All VMs | kubevirt namespace settings | virt-api, virt-controller, KubeVirt CR configurations |

---

## Section 28: Precautions for Production Troubleshooting

When troubleshooting KubeVirt, the following actions should be taken with caution:

    1. Do not delete PVCs arbitrarily.
    2. Do not delete PVs indiscriminately.
    3. Avoid forced deletion of VolumeAttachments.
    4. Do not remove Longhorn Volumes without proper consideration.
    5. Do not restart production nodes at will.
    6. Do not stop containerd services haphazardly.
    7. Do not terminate virt-launcher Pods without caution.
    8. Avoid deleting DataVolumes unnecessarily.
    9. Do not rebuild the system disk without a backup in place.
    10. Do not directly modify the disk and network configurations of production VMs.

Before proceeding with any troubleshooting, it is essential to verify the following:

    1. Whether the VM is currently handling any business traffic.
    2. The presence of data disks on the VM.
    3. The availability of backups for critical data.
    4. The existence of a maintenance window for the system.
    5. The possibility of safely stopping the VM for troubleshooting purposes.
    6. The availability of a rollback plan in case of errors.

---

## Section 29: Comprehensive List of KubeVirt Troubleshooting Commands

### 29.1 VMs / VMIs

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

### 29.4 CDI / DataVolumes

    kubectl -n cdi get pods -o wide
    kubectl get dv -A
    kubectl describe dv <dv-name> -n <namespace>
    kubectl get pods -n <namespace> | grep importer
    kubectl logs <importer-pod-name> -n <namespace> --tail=200

---

### 29.5 PVCs / PVs / Storage

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

### 29.7 Networking

    kubectl get svc -n <namespace>
    kubectl get endpoints -n <namespace>
    kubectl get pods -n <namespace> --show-labels
    kubectl get net-attach-def -A
    kubectl get pods -A | grep -i multus

---

### 29.8 Nodes

    kubectl get nodes -o wide
    kubectl describe node <node-name>

For local node checks:

    ls -l /dev/kvm
    egrep -c '(vmx|svm)' /proc/cpuinfo
    systemctl status kubelet --no-pager
    systemctl status containerd --3. virt-launcher ImagePullBackOff:
    Check the image repository and the process of pulling images using containerd.

4. DataVolume Failed:
    Examine the logs, URL, PVC, and StorageClass of the importer Pod.

5. PVC Bound but VM Failed to Start:
    Review the logs of virt-launcher, as well as the status of KVM and the image used for startup.

6. Console Unreachable:
    Check the configurations related to VM/VMI, virt-api, virt-launcher, and the Guest OS.

7. SSH Connection Issues:
    Verify the Service settings, Endpoints, sshd within the VM, NodePort, and firewall configurations.

8. Abnormalities in All VMs on a Node:
    Investigate the operations of virt-handler, /dev/kvm, kubelet, and containerd on that node.

Most Important Commands:

    kubectl describe vm <vm-name> -n <namespace>

    kubectl describe vmi <vm-name> -n <namespace>

    kubectl describe pod <virt-launcher-pod-name> -n <namespace>

    kubectl logs <virt-launcher-pod-name> -n <namespace> --tail=200

    kubectl describe dv <dv-name> -n <namespace>

    kubectl logs <importer-pod-name> -n <namespace> --tail=200

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

Guidelines for Troubleshooting:

    1. Check the Events log first, then examine the detailed logs.
    2. Determine which layer—VM, VMI, or Pod—is experiencing the issue.
    3. For DataVolume problems, focus on the importer Pod.
    4. For storage-related issues, inspect PVCs, PVs, VolumeAttachment objects, and Longhorn configurations.
    5. For node-related issues, investigate virt-handler and /dev/kvm processes.
    6. For network problems, check Service settings, Endpoints, and the IP addresses within the VM.
    7. During troubleshooting in a production environment, ensure that PVCs and data are protected at all times.

Next Step: Recommended Further Study:

    10-KubeVirt Interview Preparation: Differences Compared to vSphere, OpenStack, and Ordinary Pods.md