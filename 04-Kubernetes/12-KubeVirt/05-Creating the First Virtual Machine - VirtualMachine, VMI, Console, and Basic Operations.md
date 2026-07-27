# 05-Creating the First Virtual Machine: VirtualMachine, VMI, Console, and Basic Operations

Recommended Path:

    04-Kubernetes/12-KubeVirt/05-Creating the First Virtual Machine: VirtualMachine, VMI, Console, and Basic Operations.md

Tags:

    #Kubernetes
    #KubeVirt
    #VirtualMachine
    #VMI
    #virtctl
    #virt-launcher
    #containerDisk
    #cloudInit
    #virtual machine
    #Cloud-Native Virtualization

---

## I. Document Description

This document records the complete step-by-step process of creating the first virtual machine in KubeVirt.

The focus of this document is not on achieving complex production configurations but rather on setting up a basic working environment:

    1. Creating a test namespace
    2. Creating the first VirtualMachine
    3. Starting the virtual machine
    4. Checking the VM, VMI, and virt-launcher Pod
    5. Logging into the virtual machine using virtctl console
    6. Executing basic commands in the virtual machine
    7. Stopping, starting, and restarting the virtual machine
    8. Exposing the virtual machine's SSH port through a Service
    9. Cleaning up test resources
    10. Learning how to troubleshoot VM startup failures

This document uses:

    KubeVirt
    virtctl
    containerDisk
    cloudInitNoCloud
    CirrOS test image
    masquerade network
    ClusterIP/NodePort Services for verification (optional)

Note:

    This document uses containerDisk to create the test VM. ContainerDisk is suitable for beginner experiences and quick testing purposes. However, it is not recommended as a production method for setting up virtual machine system disks. In production scenarios, CDI, DataVolume, or PVC are typically used to import system disk images.

---

## II. Experimental Objectives

After completing this document, you should be able to:

    1. Write basic VirtualMachine YAML configurations
    2. Start a VM
    3. Check the status of a VM and VMI
    4. Locate the virt-launcher Pod
    5. Access the virtual machine console
    6. Determine whether a VM is actually running
    7. Stop and restart a VM
    8. Expose a VM's ports through a Service
    9. Troubleshoot VM startup failures based on Events

After completing the experiment, you should be able to clearly answer the following questions:

    What resources are generated after creating a KubeVirt virtual machine?
    What is the difference between a VM and a VMI?
    What is the purpose of the virt-launcher Pod?
    Which objects should be checked when a VM fails to start?

---

## III. Experimental Environment

This document assumes that the following prerequisites have been met:

    1. The Kubernetes cluster has been deployed.
    2. KubeVirt has been installed.
    3. virtctl has been installed.
    4. The nodes support KVM.
    5. The /dev/kvm directory exists.
    6. CoreDNS is functioning correctly.
    7. CNI is working properly.
    8. containerd is running smoothly.
    9. kubectl can be used to access the cluster.

Execution Node:

    k8s-master-01

KubeVirt Version:

    v1.4.0

Kubernetes Version:

    v1.31.14

Test VM:

    Name: vm-cirros
    Namespace: kubevirt-demo
    Image: quay.io/kubevirt/cirros-container-disk-demo:latest
    Memory: 256Mi
    Network: pod network + masquerade
    Login Method: virtctl console
    Test Account: cirros
    Test Password: kubevirt

---

## IV. Pre-creation Checks

### 4.1 Checking KubeVirt Components

Execute:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

Requirements:

    kubevirt must be available.
    virt-api, virt-controller, virt-handler, and virt-operator must all be running.

---

### 4.2 Checking virtctl

Execute:

    virtctl version

If virtctl is not installed, return to the installation section to install it first.

---

### 4.3 Checking VM/VMI Resources

Execute:

    kubectl get vm -A

    kubectl get vmi -A

If there are no VMs yet, an empty output is expected.

If an error indicating that resources do not exist is reported, it means that the KubeVirt CRD has not been installed successfully.

---

### 4.Use the NAT mode of the Pod network to allow the VM to access the Kubernetes network.

ports:
    Declare port 22 inside the virtual machine; it can later be exposed via a Service for SSH access.```bash
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

Application:

    kubectl apply -f svc-vm-cirros-ssh.yaml

View:

    kubectl get svc -n kubevirt-demo

View Endpoints:

    kubectl get endpoints vm-cirros-ssh -n kubevirt-demo

If Endpoints are not empty, it means the Service has been matched with the corresponding backend VM.

---

### 16.2 Testing the SSH Port Within the Cluster

Create a temporary test Pod:

    kubectl run ssh-test \
      -n kubevirt-demo \
      --image=busybox:1.36 \
      --restart=Never \
      -it --rm -- sh

Execute inside the test Pod:

    nc -vz vm-cirros-ssh.kubevirt-demo.svc.cluster.local 22

If busybox does not have nc, you can try:

    telnet vm-cirros-ssh.kubevirt-demo.svc.cluster.local 22

Note:

    This step only verifies connectivity on port 22.
    Actual SSH login should be performed from a node with an ssh client or within a debugging container.

---

### 16.3 Optional: Creating a NodePort Service

If you need to test SSH from outside the cluster, you can create a NodePort Service.

Creation:

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

Application:

    kubectl apply -f svc-vm-cirros-ssh-nodeport.yaml

View:

    kubectl get svc -n kubevirt-demo

Access from outside:

    ssh cirros@10.0.0.23 -p 30022

Password:

    kubevirt

Note:

    10.0.0.23 is an arbitrary Worker node IP address.
    If the node's firewall does not allow port 30022, you need to open it first.
    Alternatively, you can use a LoadBalancer Service for exposure.

---

## Chapter Seventeen: Observing Services and Endpoints

View the Service:

    kubectl describe svc vm-cirros-ssh -n kubevirt-demo

View the Endpoints:

    kubectl get endpoints vm-cirros-ssh -n kubevirt-demo

View Pod labels:

    kubectl get pod -n kubevirt-demo --show-labels

Note:

    The Service matches virt-launcher Pods through its selector.
    The virtual machine's ports are ultimately forwarded internally to the VM through the virt-launcher Pod's network configuration.

If the Endpoints are empty, check:

    1. Whether the VM is Running.
    2. Whether the VMI exists.
    3. Whether the virt-launcher Pod is Running.
    4. Whether the Pod labels contain "kubevirt.io/domain=vm-cirros".
    5. Whether there are any errors in the Service selector configuration.

---

## Chapter Eighteen: Viewing Key Fields in Virtual Machine YAML Files

View the VM YAML file:

    kubectl get vm vm-cirros -n kubevirt-demo -o yaml

Key fields to check:

    spec.runStrategy
    spec.template.spec.domainresources
    spec.template/spec.domain_devices.disks
    spec.template/spec.domaindevices.interfaces
    spec.template.spec.networks
    spec.template.spec.volumes

View the VMI YAML file:

    kubectl get vmi vm-cirros -n kubevirt-demo -o yaml

Key fields to check:

    status.phase
    status.nodeName
    status/interfaces
    status_conditions
    status.volumeStatus

Note:

    Check the VM YAML for expected configuration settings.
    Verify the VMI YAML to understand its current operational status.

---

## Chapter Nineteen: Experiment: Modifying Virtual Machine Memory Configuration

Note:

    Modifying resource configurations while a VM is running may not take effect immediately.
    It is recommended to stop the VM first, make the changes, and then restart it during the beginner stage.

Stop the VM:

    virtctl stop vmkubectl describe vmi vm-cirros -n kubevirt-demo

kubectl get events -n kubevirt-demo --sort-by=.lastTimestamp

kubectl get pods -n kubevirt-demo -o wide

Common causes:

1. Insufficient node resources
2. The node does not support KVM
3. virt-handler exception
4. Mismatch between taints and tolerations
5. Mismatch between nodeSelector and affinity
6. Image pull failure

---

### 21.4 virt-launcher Pod ImagePullBackOff

View:

kubectl get pods -n kubevirt-demo

kubectl describe pod <virt-launcher-pod-name> -n kubevirt-demo

Common causes:

1. Unable to pull quay.io/kubevirt/cirros-container-disk-demo:latest
2. The node cannot access the public image repository
3. Abnormal containerd image configuration
4. Company network restrictions on external access

Solution options:

1. Synchronize the image to an internal Harbor
2. Modify the containerDisk.image in the VM YAML file
3. Verify that the node can pull images using crictl

Example:

sudo crictl pull quay.io/kubevirt/cirros-container-disk-demo:latest

---

### 21.5 virt-launcher is Running but the console cannot be logged into

Check:

kubectl get vmi vm-cirros -n kubevirt-demo

kubectl describe vmi vm-cirros -n kubevirt-demo

kubectl logs <virt-launcher-pod-name> -n kubevirt-demo --tail=100

Attempt:

virtctl console vm-cirros -n kubevirt-demo

Press Enter several times after entering.

Common causes:

1. The guest OS has not finished booting
2. Abnormal image startup
3. Inactive console output
4. cloud-init has not completed
5. Issues with virt-api or connection links

---

### 21.6 Incorrect login password

Configuration in this document:

Username: cirros
Password: kubevirt

If it does not work, possible reasons:

1. cloud-init did not execute successfully
2. The default image user is not cirros
3. Issues with userData indentation
4. The VM was not recreated or restarted
5. A different containerDisk image was used

Troubleshooting:

kubectl get vm vm-cirros -n kubevirt-demo -o yaml | grep -A20 cloudInitNoCloud

kubectl logs <virt-launcher-pod-name> -n kubevirt-demo --tail=100

---

### 21.7 Service Endpoints are empty

View:

kubectl get endpoints vm-cirros-ssh -n kubevirt-demo

kubectl get pod -n kubevirt-demo --show-labels

kubectl get svc vm-cirros-ssh -n kubevirt-demo -o yaml

Common causes:

1. The VM is not running
2. The virt-launcher Pod does not exist
3. Incorrect Service selector configuration
4. Mismatch between Pod labels and Service selector
5. Incorrect namespace configuration

---

### 21.8 Unable to establish an SSH connection

Check:

kubectl get svc -n kubevirt-demo

kubectl get endpoints -n kubevirt-demo

kubectl get vmi vm-cirros -n kubevirt-demo

virtctl console vm-cirros -n kubevirt-demo

Inside the VM, check:

ip addr

ps aux | grep ssh

netstat -lntp

Common causes:

1. The sshd service is not running inside the VM
2. The image does not support SSH
3. cloud-init did not enable password-based login
4. Mismatch between Service selector and actual configuration
5. NodePort is blocked by the firewall
6. Incorrect access port settings

---

## Section 22: Standard troubleshooting steps for the first VM

When a VM fails to start, follow these steps in order:

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
11. Check the /dev/kvm directory on the node where the VM is located
12. Review the kubelet and containerd logs of the node

---

## Section 23: List```bash
kubectl delete -f vm-cirros.yaml

Verify the deletion:

kubectl get vm -n kubevirt-demo

kubectl get vmi -n kubevirt-demo

kubectl get pods -n kubevirt-demo

Delete the namespace:

kubectl delete namespace kubevirt-demo

Note:

In this document, the VM uses containerDisk and does not involve persistent system disk PVCs. When using DataVolume/PVCs later on, it is necessary to confirm whether these PVCs need to be retained during cleanup.
---

## Summary of Article 26

This article has completed the creation and basic operations of KubeVirt's first virtual machine.

Key steps:

1. Created the kubevirt-demo namespace.
2. Compiled the VirtualMachine YAML file.
3. Launched the CirrOS VM using containerDisk.
4. Set the login password using cloudInitNoCloud.
5. Started the VM using virtctl start.
6. Checked the VM, VMI, and virt-launcher Pod status.
7. Logged into the VM using virtctl console.
8. Managed the VM using virtctl stop/start/restart commands.
9. Exposed the SSH port via Service.
10. Learned basic troubleshooting methods for failed VM startups.

Most important object relationships:

- **VM**: Represents the virtual machine definition and desired state.
- **VMI**: Represents the currently running virtual machine instance.
- **virt-launcher Pod**: Actualizes the virtual machine process.

Key commands:

- `kubectl get vm -n kubevirt-demo`
- `kubectl get vmi -n kubevirt-demo`
- `kubectl get pods -n kubevirt-demo -o wide`
- `virtctl start vm-cirros -n kubevirt-demo`
- `virtctl console vm-cirros -n kubevirt-demo`

Practical tips:

- The existence of a VM does not necessarily mean it is running.
- The presence of a VMI usually indicates that the VM is running.
- A running virt-launcher Pod is an important sign of a functional VM.
- If you cannot access the console, check the VMI and virt-launcher logs first.
- ContainerDisk is suitable for beginners but not for persistent system disk use in production environments.
- For production VMs, use CDI/DataVolume/PVCs to manage system disks.

Next step: Continue learning about 06-CDI and DataVolume: Importing Virtual Machine Images, Managing PVCs and Boot Disks.md.