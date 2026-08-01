# 05 - Creating the First Virtual Machine: VirtualMachine, VMI, console and Basic Operations

Recommended path:

    04-Kubernetes/12-KubeVirt/05 - Creating the First Virtual Machine: VirtualMachine, VMI, console and Basic Operations.md

Tags:

    #Kubernetes
    #KubeVirt
    #VirtualMachine
    #VMI
    #virtctl
    #virt-launcher
    #containerDisk
    #cloudInit
    #VirtualMachine
    #CloudlandVirtualization

---

## I. Document Description

This document records the complete hands-on process of creating the first virtual machine in KubeVirt.

The focus of this document is not to pursue complex production configurations, but to first establish a minimal working loop:

    1. Create a test namespace
    2. Create the first VirtualMachine
    3. Start the virtual machine
    4. View VM / VMI / virt-launcher Pod
    5. Log in to the virtual machine using virtctl console
    6. Execute basic commands in the virtual machine
    7. Stop, start, and restart the virtual machine
    8. Expose the virtual machine SSH via Service
    9. Clean up test resources
    10. Master the troubleshooting methods for VM startup failures

This document uses:

    KubeVirt
    virtctl
    containerDisk
    cloudInitNoCloud
    CirrOS test image
    masquerade network
    ClusterIP / NodePort Service optional verification

Notes:

    This document uses containerDisk to create a test VM.
    containerDisk is suitable for introductory experience and quick verification.
    It is not a recommended way to create system disk for production virtual machines.
    In production, system disk images are typically imported using CDI / DataVolume / PVC.

---

## II. Experiment Objectives

After completing this document, you should be able to:

    1. How to write a basic VirtualMachine YAML
    2. How to start a VM
    3. How to view the status of VM and VMI
    4. How to find the virt-launcher Pod
    5. How to enter the virtual machine console
    6. How to determine if the VM is actually running
    7. How to stop and restart the VM
    8. How to expose VM ports via Service
    9. How to troubleshoot VM startup failures based on Events

After completing the experiment, you should be able to clearly answer:

    What resources are generated after a KubeVirt virtual machine is created?
    What is the difference between VM and VMI?
    What is the purpose of the virt-launcher Pod?
    Which objects should be checked when the VM fails to start?

---

## III. Experimental Environment

This document assumes the following prerequisites have been completed:

    1. Kubernetes cluster has been deployed
    2. KubeVirt has been installed
    3. virtctl has been installed
    4. Nodes support KVM
    5. /dev/kvm exists
    6. CoreDNS is functioning normally
    7. CNI is functioning normally
    8. containerd is functioning normally
    9. kubectl can access the cluster normally

Execution node:

    k8s-master-01

KubeVirt version:

    v1.4.0

Kubernetes version:

    v1.31.14

Test VM:

    Name: vm-cirros
    Namespace: kubevirt-demo
    Image: quay.io/kubevirt/cirros-container-disk-demo:latest
    Memory: 256Mi
    Network: pod network + masquerade
    Login method: virtctl console
    Test account: cirros
    Test password: kubevirt

---

## IV. Pre-VM Creation Checks

### 4.1 Check KubeVirt Components

Execute:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

Requirements:

    kubevirt Available
    virt-api Running
    virt-controller Running
    virt-handler Running
    virt-operator Running

---

### 4.2 Check virtctl

Execute:

    virtctl version

If virtctl is not available, go back to the installation guide to install virtctl.

---

### 4.3 Check VM / VMI Resources

Execute:

    kubectl get vm -A

    kubectl get vmi -A

If there are no VMs yet, an empty output is normal.

If an error about missing resources occurs, it indicates that the KubeVirt CRD has not been installed successfully.

---

### 4.4 Check Node KVM

On the worker node where the VM will run, execute:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    ls -l /dev/kvm

    lsmod | grep kvm

    sudo kvm-ok

Basic requirements:

    vmx/svm count greater than 0
    /dev/kvm exists
    kvm module is loaded
    kvm-ok passes

---

## V. Create Test Namespace

Create namespace:

    kubectl create namespace kubevirt-demo

View:

    kubectl get namespace kubevirt-demo

Notes:

    All subsequent VM, VMI, virt-launcher Pod, and Service resources will be placed in the kubevirt-demo namespace for easier management and cleanup.

---

## VI. Write the First VM YAML

Create directory:

    mkdir -p /root/k8s-yaml/kubevirt/vm-demo

    cd /root/k8s-yaml/kubevirt/vm-demo

Create VM YAML: /think

```yaml
cat <<EOF > vm-cirros.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-cirros
  namespace: kubevirt-demo
  labels:
    app: vm-cirros
spec:
  runStrategy: Manual
  template:
    metadata:
      labels:
        app: vm-cirros
        kubevirt.io/domain: vm-cirros
    spec:
      terminationGracePeriodSeconds: 0
      domain:
        resources:
          requests:
            memory: 256Mi
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
            ports:
            - name: ssh
              port: 22
      networks:
      - name: default
        pod: {}
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/cirros-container-disk-demo:latest
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            password: kubevirt
            chpasswd:
              expire: false
            ssh_pwauth: true
EOF
```

**Explanation:**

- `kind: VirtualMachine`  
  Indicates creating a KubeVirt virtual machine object.

- `runStrategy: Manual`  
  Indicates manual control over VM startup and shutdown.

- `resources.requests.memory: 256Mi`  
  Requests 256Mi of memory for the VM.

- `containerDisk`  
  Uses a container image's virtual machine disk to quickly boot VM.

- `cloudInitNoCloud`  
  Injects initialization configuration into the VM, such as passwords and SSH login settings.

- `masquerade`  
  Uses NAT mode of Pod network to allow VM access to Kubernetes network.

- `ports:`  
  Declares the 22 port inside the VM, which can be exposed via Service later.

---

## Seven. Creating VirtualMachine

Apply YAML:

```bash
kubectl apply -f vm-cirros.yaml
```

Check VM:

```bash
kubectl get vm -n kubevirt-demo
```

Example output:

```
NAME        AGE   STATUS    READY
vm-cirros   10s   Stopped   False
```

**Explanation:**

- The VM has been created but hasn't started yet.
- Because `runStrategy` is Manual.
- There is typically no VMI at this point.
- There is typically no virt-launcher Pod at this point.

Check VMI:

```bash
kubectl get vmi -n kubevirt-demo
```

Expected:

```
No resources found
```

Check Pod:

```bash
kubectl get pods -n kubevirt-demo
```

Expected:

```
No resources found
```

This step indicates:

- The VM object can exist, but the VM doesn't necessarily have to be running.

---

## Eight. Starting the Virtual Machine

Start using virtctl:

```bash
virtctl start vm-cirros -n kubevirt-demo
```

Check VM:

```bash
kubectl get vm -n kubevirt-demo
```

Check VMI:

```bash
kubectl get vmi -n kubevirt-demo
```

Check Pod:

```bash
kubectl get pods -n kubevirt-demo -o wide
```

Expected to see:

1. VM status changes to Running  
2. VMI appears  
3. virt-launcher Pod appears  
4. virt-launcher Pod Running

Example:

```bash
kubectl get vm -n kubevirt-demo
```

```
NAME        AGE   STATUS    READY
vm-cirros   1m    Running   True
```

```bash
kubectl get vmi -n kubevirt-demo
```

```
NAME        AGE   PHASE     IP
vm-cirros   30s   Running   10.244.x.x
```

```bash
kubectl get pods -n kubevirt-demo -o wide
```

```
NAME                              READY   STATUS    NODE
virt-launcher-vm-cirros-xxxxx     1/1     Running   k8s-worker-01
```

## IX. Observing VM, VMI, virt-launcher Relationships

### 9.1 Viewing VM

Execute:

    kubectl describe vm vm-cirros -n kubevirt-demo

Focus on:

    Run Strategy
    Printable Status
    Created
    Ready
    Conditions
    Events

---

### 9.2 Viewing VMI

Execute:

    kubectl describe vmi vm-cirros -n kubevirt-demo

Focus on:

    Phase
    Node Name
    Interfaces
    Volumes
    Conditions
    Events

---

### 9.3 Viewing virt-launcher Pod

Find virt-launcher:

    kubectl get pods -n kubevirt-demo -o wide | grep virt-launcher

Check details:

    kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo

Check logs:

    kubectl logs <virt-launcher-pod-name> -n kubevirt-demo --tail=100

Explanation:

    VM is the virtual machine definition object.
    VMI is the running virtual machine instance.
    virt-launcher Pod is the Pod that actually hosts the virtual machine process.

Relationship chain:

    VirtualMachine
        |
        v
    VirtualMachineInstance
        |
        v
    virt-launcher Pod
        |
        v
    QEMU / KVM
        |
        v
    Guest OS

---

## X. Entering Virtual Machine Console

Execute:

    virtctl console vm-cirros -n kubevirt-demo

If prompted successfully connected, press Enter.

Login information:

    Username: cirros
    Password: kubevirt

After entering VM, execute:

    hostname

    ip addr

    uname -a

    df -h

    free -m

Exit console:

    Ctrl + ]

Explanation:

    virtctl console is the most direct way to access the virtual machine during the initial phase.
    If the console is unresponsive, try pressing Enter multiple times.
    If still unable to enter, check the status of VMI and virt-launcher Pod.

---

## XI. Basic Validation Inside the Virtual Machine

After logging into VM, execute:

    hostname

    ip addr

    ip route

    cat /etc/resolv.conf

    ping -c 3 10.96.0.1

Explanation:

    10.96.0.1 is typically the ClusterIP of the kubernetes.default Service.
    If your cluster's Service CIDR is different, use the actual output from kubectl get svc kubernetes.

Check Kubernetes Service on k8s-master-01:

    kubectl get svc kubernetes -n default

Test DNS inside VM:

    nslookup kubernetes.default.svc.cluster.local

If nslookup is missing, the test image may be minimal, you can skip this step.

---

## XII. Checking Virtual Machine IP

Check VMI from Kubernetes side:

    kubectl get vmi vm-cirros -n kubevirt-demo -o wide

Check YAML:

    kubectl get vmi vm-cirros -n kubevirt-demo -o yaml | grep -A20 interfaces

Check virt-launcher Pod:

    kubectl get pod -n kubevirt-demo -o wide | grep virt-launcher

Explanation:

    When using masquerade networking, the network behavior of VMI/Pod differs from regular Pods.
    During initial phase, focus on mastering:
        VM can start
        console can be accessed
        VM network interface exists
        VM can expose ports via Service

---

## XIII. Stopping the Virtual Machine

Stop VM:

    virtctl stop vm-cirros -n kubevirt-demo

Check VM:

    kubectl get vm -n kubevirt-demo

Check VMI:

    kubectl get vmi -n kubevirt-demo

Check Pods:

    kubectl get pods -n kubevirt-demo

Expected:

    VM still exists.
    VMI will disappear.
    virt-launcher Pod will disappear or enter Terminating state and disappear.

Explanation:

    Stopping VM does not equal deleting VM.
    Stopping VM does not equal deleting disk.
    It only stops the running VMI and virt-launcher Pod.

---

## XIV. Restarting the Virtual Machine

Start:

    virtctl start vm-cirros -n kubevirt-demo

Check:

    kubectl get vm -n kubevirt-demo

    kubectl get vmi -n kubevirt-demo

    kubectl get pods -n kubevirt-demo -o wide

Expected:

    VMI is recreated.
    virt-launcher Pod is recreated.
    VM is back to Running.

---

## XV. Rebooting the Virtual Machine

Execute:

    virtctl restart vm-cirros -n kubevirt-demo

Observe:

    kubectl get vm -n kubevirt-demo

    kubectl get vmi -n kubevirt-demo

    kubectl get pods -n kubevirt-demo -o wide

Explanation:

restart will restart the virtual machine.
When troubleshooting, observe whether VMI and virt-launcher Pod change.

---

## SixteenI don't know.Expose Virtual Machine SSH via Service

### 16.1 Create ClusterIP Service

Create Service:

    cat <<EOF > svc-vm-cirros-ssh.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: vm-cirros-ssh
      namespace: kubevirt-demo
    spec:
      type: ClusterIP
      selector:
        kubevirt.io/domain: vm-cirros
      ports:
      - name: ssh
        protocol: TCP
        port: 22
        targetPort: 22
    EOF

Apply:

    kubectl apply -f svc-vm-cirros-ssh.yaml

Check:

    kubectl get svc -n kubevirt-demo

Check Endpoints:

    kubectl get endpoints vm-cirros-ssh -n kubevirt-demo

If Endpoints is not empty, it indicates the Service has matched the backend of the VM.

---

### 16.2 Test SSH Port Within Cluster

Create temporary test Pod:

    kubectl run ssh-test \
      -n kubevirt-demo \
      --image=busybox:1.36 \
      --restart=Never \
      -it --rm -- sh

Execute in test Pod:

    nc -vz vm-cirros-ssh.kubevirt-demo.svc.cluster.local 22

If busybox does not have nc, try:

    telnet vm-cirros-ssh.kubevirt-demo.svc.cluster.local 22

Note:

    This only verifies the connectivity of port 22.
    Actual SSH login can be performed on a node with an ssh client or a debug container.

---

### 16.3 Optional: Create NodePort Service

If you need to test SSH from outside the cluster, you can create NodePort.

Create:

    cat <<EOF > svc-vm-cirros-ssh-nodeport.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: vm-cirros-ssh-nodeport
      namespace: kubevirt-demo
    spec:
      type: NodePort
      selector:
        kubevirt.io/domain: vm-cirros
      ports:
      - name: ssh
        protocol: TCP
        port: 22
        targetPort: 22
        nodePort: 30022
    EOF

Apply:

    kubectl apply -f svc-vm-cirros-ssh-nodeport.yaml

Check:

    kubectl get svc -n kubevirt-demo

Access from outside:

    ssh cirros@10.0.0.23 -p 30022

Password:

    kubevirt

Note:

    10.0.0.23 is any Worker node IP.
    If the node firewall does not allow 30022, you need to open it first.
    If using MetalLB, you can also expose via LoadBalancer Service.

---

## SeventeenI don't know.Observe Service and Endpoints

Check Service:

    kubectl describe svc vm-cirros-ssh -n kubevirt-demo

Check Endpoints:

    kubectl get endpoints vm-cirros-ssh -n kubevirt-demo

Check Pod label:

    kubectl get pod -n kubevirt-demo --show-labels

Note:

    Service matches virt-launcher Pod via selector.
    The virtual machine port is finally forwarded to the VM internally via virt-launcher Pod network.

If Endpoints is empty, check:

    1. Is VM Running?
    2. Does VMI exist?
    3. Is virt-launcher Pod Running?
    4. Does Pod label include kubevirt.io/domain=vm-cirros?
    5. Is Service selector written incorrectly?

---

## EighteenI don't know.View Key Fields in Virtual Machine YAML

Check VM YAML:

    kubectl get vm vm-cirros -n kubevirt-demo -o yaml

Key fields:

    spec.runStrategy
    spec.template.spec.domain.resources
    spec.template.spec.domain.devices.disks
    spec.template.spec.domain.devices.interfaces
    spec.template.spec.networks
    spec.template.spec.volumes

Check VMI YAML:

    kubectl get vmi vm-cirros -n kubevirt-demo -o yaml

Key fields:

    status.phase
    status.nodeName
    status.interfaces
    status.conditions
    status.volumeStatus

Note:

    VM YAML shows expected configuration.
    VMI YAML shows runtime status.

---

## NineteenI don't know.Experiment: Modify VM Memory Configuration

Note:

    Modifying resource configuration for a running VM may not take effect immediately.
    In the initial stage, it's recommended to stop the VM first, then modify, then start it.

Stop VM: /think

virtctl stop vm-cirros -n kubevirt-demo

Wait for the VMI to disappear:

    kubectl get vmi -n kubevirt-demo

Modify memory:

    kubectl patch vm vm-cirros -n kubevirt-demo --type='merge' \
      -p '{"spec":{"template":{"spec":{"domain":{"resources":{"requests":{"memory":"512Mi"}}}}}}}}'

Check:

    kubectl get vm vm-cirros -n kubevirt-demo -o yaml | grep -A5 resources

Start VM:

    virtctl start vm-cirros -n kubevirt-demo

Check:

    kubectl get vmi vm-cirros -n kubevirt-demo -o yaml | grep -A5 resources

Note:

    In production environments, modifying VM CPU/memory should be combined with the KubeVirt version, virtual machine status, and business window.
    Do not arbitrarily modify production VM configurations online.

---

## Twenty: Experiment: Delete VM but Keep the Mindset

This document uses containerDisk, without an independent PVC system disk.

Delete VM:

    kubectl delete vm vm-cirros -n kubevirt-demo

Check:

    kubectl get vm -n kubevirt-demo

    kubectl get vmi -n kubevirt-demo

    kubectl get pods -n kubevirt-demo

Note:

    If the VM uses PVC as the system disk, deleting the VM does not necessarily mean deleting the PVC.
    Before deleting a VM in a production environment, you must confirm whether the disk PVC needs to be retained.
    This document's containerDisk VM does not have data retention issues after deletion.

If you need to recreate:

    kubectl apply -f vm-cirros.yaml

    virtctl start vm-cirros -n kubevirt-demo

---

## Twenty-one: Common Issue Troubleshooting

### 21.1 VM does not start after creation

Phenomenon:

    kubectl get vm -n kubevirt-demo

Shows:

    Stopped

Cause:

    runStrategy: Manual

Resolution:

    virtctl start vm-cirros -n kubevirt-demo

Note:

    This is a normal phenomenon, not a fault.

---

### 21.2 virtctl start fails

Check:

    virtctl version

    kubectl -n kubevirt get pods -o wide

    kubectl -n kubevirt get svc

    kubectl -n kubevirt logs deploy/virt-api --tail=100

Common causes:

    1. virt-api is abnormal
    2. virtctl version mismatch
    3. kubeconfig permissions are insufficient
    4. KubeVirt CR is not Available

---

### 21.3 VMI remains in Pending state

Check:

    kubectl describe vmi vm-cirros -n kubevirt-demo

    kubectl get events -n kubevirt-demo --sort-by=.lastTimestamp

    kubectl get pods -n kubevirt-demo -o wide

Common causes:

    1. Node resource insufficiency
    2. Node does not support KVM
    3. virt-handler is abnormal
    4. taints/tolerations mismatch
    5. nodeSelector/affinity mismatch
    6. Image pull failure

---

### 21.4 virt-launcher Pod ImagePullBackOff

Check:

    kubectl get pods -n kubevirt-demo

    kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo

Common causes:

    1. Unable to pull quay.io/kubevirt/cirros-container-disk-demo:latest
    2. Node cannot access public image registry
    3. containerd image configuration anomaly
    4. Corporate network restrictions on external internet

Resolution direction:

    1. Synchronize the image to internal Harbor
    2. Modify the containerDisk.image in the VM YAML
    3. Confirm the node can crictl pull the image

Example:

    sudo crictl pull quay.io/kubevirt/cirros-container-disk-demo:latest

---

### 21.5 virt-launcher Running but console cannot log in

Check:

    kubectl get vmi vm-cirros -n kubevirt-demo

    kubectl describe vmi vm-cirros -n kubevirt-demo

    kubectl logs <virt-launcher-pod-name> -n kubevirt-demo --tail=100

Try:

    virtctl console vm-cirros -n kubevirt-demo

Press Enter multiple times after entering.

Common causes:

    1. Guest OS has not completed startup
    2. Image startup anomaly
    3. Console output is inactive
    4. cloud-init has not completed
    5. virt-api or connection link anomaly

---

### 21.6 Incorrect login password

Configuration in this document:

    Username: cirros
    Password: kubevirt

If this does not work, possible causes:

    1. cloud-init did not execute successfully
    2. The default user in the image is not cirros
    3. userData indentation error
    4. VM has not been recreated or restarted
    5. A different containerDisk image was used

Troubleshoot: /think

kubectl get vm vm-cirros -n kubevirt-demo -o yaml | grep -A20 cloudInitNoCloud

kubectl logs <virt-launcher-pod-name> -n kubevirt-demo --tail=100

---

### 21.7 Service Endpoints are Empty

Check:

    kubectl get endpoints vm-cirros-ssh -n kubevirt-demo

    kubectl get pod -n kubevirt-demo --show-labels

    kubectl get svc vm-cirros-ssh -n kubevirt-demo -o yaml

Common Causes:

    1. VM is not Running
    2. virt-launcher Pod does not exist
    3. Service selector is incorrect
    4. Pod label does not match
    5. namespace is incorrect

---

### 21.8 SSH Connection Failure

Check:

    kubectl get svc -n kubevirt-demo

    kubectl get endpoints -n kubevirt-demo

    kubectl get vmi vm-cirros -n kubevirt-demo

    virtctl console vm-cirros -n kubevirt-demo

Check inside the VM:

    ip addr

    ps aux | grep ssh

    netstat -lntp

Common Causes:

    1. sshd is not running inside the VM
    2. The image does not support SSH
    3. cloud-init has not enabled password login
    4. Service selector does not match
    5. NodePort firewall is blocked
    6. Access port is incorrect

---

## Twenty-two, Standard Troubleshooting Path for the First VM

When VM fails to start, troubleshoot in the following order:

    1. kubectl get vm -n kubevirt-demo
    2. kubectl describe vm vm-cirros -n kubevirt-demo
    3. kubectl get vmi -n kubevirt-demo
    4. kubectl describe vmi vm-cirros -n kubevirt-demo
    5. kubectl get pods -n kubevirt-demo -o wide
    6. kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo
    7. kubectl logs <virt-launcher-pod-name> -n kubevirt-demo --tail=100
    8. kubectl get events -n kubevirt-demo --sort-by=.lastTimestamp
    9. kubectl -n kubevirt get pods -o wide
    10. kubectl -n kubevirt logs deploy/virt-controller --tail=100
    11. Check /dev/kvm on the node where VM resides
    12. Check kubelet/containerd logs on the node

---

## Twenty-three, Common Command List

Check VM:

    kubectl get vm -A

    kubectl get vm -n kubevirt-demo

    kubectl describe vm vm-cirros -n kubevirt-demo

Start VM:

    virtctl start vm-cirros -n kubevirt-demo

Stop VM:

    virtctl stop vm-cirros -n kubevirt-demo

Restart VM:

    virtctl restart vm-cirros -n kubevirt-demo

Check VMI:

    kubectl get vmi -n kubevirt-demo

    kubectl describe vmi vm-cirros -n kubevirt-demo

Enter console:

    virtctl console vm-cirros -n kubevirt-demo

Check virt-launcher:

    kubectl get pods -n kubevirt-demo -o wide | grep virt-launcher

    kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo

    kubectl logs <virt-launcher-pod-name> -n kubevirt-demo --tail=100

Check events:

    kubectl get events -n kubevirt-demo --sort-by=.lastTimestamp

Check Service:

    kubectl get svc -n kubevirt-demo

    kubectl get endpoints -n kubevirt-demo

---

## Twenty-four, Differences Between Production and Experiment

This document is for experimental VM.

Used in experiments:

    containerDisk
    CirrOS
    Simple password
    Optional NodePort exposure
    Small memory specification

It is not recommended to use this directly in production.

In production, consider:

    1. Use PVC/DataVolume as the system disk
    2. Use formal OS image
    3. Use SSH Key instead of simple password
    4. Configure resource requests/limits
    5. Plan dedicated nodes for VM
    6. Plan storage backup
    7. Plan network and security policies
    8. Limit VM management permissions
    9. Integrate monitoring and logs
    10. Avoid exposing NodePort arbitrarily
    11. Confirm PVC and data retention policies before deleting VM

---

## Twenty-five, Clean Up Experimental Resources

Stop VM:

    virtctl stop vm-cirros -n kubevirt-demo

Delete Service:

    kubectl delete -f svc-vm-cirros-ssh.yaml --ignore-not-found

kubectl delete -f svc-vm-cirros-ssh-nodeport.yaml --ignore-not-found

Delete VM:

    kubectl delete -f vm-cirros.yaml

Verify:

    kubectl get vm -n kubevirt-demo

    kubectl get vmi -n kubevirt-demo

    kubectl get pods -n kubevirt-demo

Delete namespace:

    kubectl delete namespace kubevirt-demo

Notes:

    This article uses containerDisk for VM, which does not involve persistent system disk PVC.
    After using DataVolume / PVC, cleanup must additionally confirm whether PVC needs to be retained.

---

## 26. Summary of This Article

This article completes the creation and basic operations of the first virtual machine with KubeVirt.

Core workflow:

    1. Create kubevirt-demo namespace
    2. Write VirtualMachine YAML
    3. Start CirrOS VM using containerDisk
    4. Set login password using cloudInitNoCloud
    5. Start VM using virtctl start
    6. View VM / VMI / virt-launcher Pod
    7. Login to VM using virtctl console
    8. Manage VM using virtctl stop / start / restart
    9. Expose SSH port using Service
    10. Master the basic troubleshooting path for VM startup failure

Most important object relationships:

    VM
      Represents virtual machine definition and desired state

    VMI
      Represents running virtual machine instance

    virt-launcher Pod
      Actually hosts the virtual machine process

Most important commands:

    kubectl get vm -n kubevirt-demo

    kubectl get vmi -n kubevirt-demo

    kubectl get pods -n kubevirt-demo -o wide

    virtctl start vm-cirros -n kubevirt-demo

    virtctl console vm-cirros -n kubevirt-demo

Experience judgment:

    1. VM existence does not mean the virtual machine is running
    2. VMI existence usually indicates the virtual machine is running
    3. virt-launcher Pod Running is an important sign of VM operation
    4. When console access fails, first check VMI and virt-launcher logs
    5. containerDisk is suitable for beginners, not for production persistent system disks
    6. Production VM should use CDI / DataVolume / PVC for system disk management

Next article recommendation:

    06-CDI and DataVolume: Virtual Machine Image Import, PVC and Boot Disk Management.md