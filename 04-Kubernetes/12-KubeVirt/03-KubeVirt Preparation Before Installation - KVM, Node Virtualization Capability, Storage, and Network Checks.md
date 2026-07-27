# 03-KubeVirt Preparation Before Installation: KVM, Node Virtualization Capability, Storage, and Network Checks

Recommended Path:

    04-Kubernetes/12-KubeVirt/03-KubeVirt Preparation Before Installation: KVM, Node Virtualization Capability, Storage, and Network Checks.md

Tags:

    #Kubernetes
    #KubeVirt
    #KVM
    #QEMU
    #Virtualization
    #Node Checks
    #StorageClass
    #PVC
    #CNI
    #Ubuntu2204
    #Cloud-Native Virtualization

---

## I. Document Description

This document records the environmental checks and preparatory steps required before installing KubeVirt.

KubeVirt is different from ordinary Kubernetes Addons.

For regular components, as long as the cluster is Ready, images can be pulled, and Pods can run, they can usually be deployed.

However, since KubeVirt runs virtual machines on Kubernetes nodes, additional checks are necessary:

    1. Verify whether the node's CPU supports hardware virtualization.
    2. Ensure that VT-x / AMD-V are enabled in the BIOS / virtualization platform.
    3. Check if /dev/kvm exists on the node.
    4. Confirm that the KVM kernel modules are loaded.
    5. Determine whether the current environment supports nested virtualization.
    6. Verify that Kubernetes node resources are sufficient.
    7. Ensure that StorageClass and PVC are available.
    8. Check if the CNI network is functioning properly.
    9. Verify that DNS is working correctly.
    10. Confirm that the image repository is accessible.
    11. Ensure that the node on which KubeVirt will be installed allows virtual machine scheduling.
    12. Verify that the node's kernel, containerd, and kubelet are stable.

Objectives of this document:

    1. Complete checks on node virtualization capability before installing KubeVirt.
    2. Identify issues such as the absence of /dev/kvm, CPU not supporting virtualization, or nested virtualization not being enabled in advance.
    3. Verify PVC, StorageClass, network, and DNS settings beforehand.
    4. Lay the foundation for subsequent installation of KubeVirt and creation of the first virtual machine.

Applicable Environment:

    Ubuntu 22.04
    kubeadm-based self-built Kubernetes cluster
    containerd runtime environment
    Common CNI solutions such as Calico or Flannel
    CSI StorageClass solutions like Longhorn or NFS

---

## II. Why Check Before Installation

KubeVirt relies on the virtualization capabilities of the nodes.

If the nodes are not ready, potential issues may occur:

    1. KubeVirt may be installed, but virtual machines will not start.
    2. VMs may be created successfully, but the VMI state remains Pending.
    3. The virt-launcher Pod may fail to create.
    4. Even if the virt-launcher Pod starts, the Guest OS may not boot.
    5. KVM-related errors may appear in logs.
    6. Virtual machine performance may be poor.
    7. DataVolume images may be imported successfully, but virtual machines may not be able to mount the disks.
    8. VM network connectivity may fail.
    9. The console may not be accessible.
    10. Advanced features such as LiveMigration may not be available.

Therefore, it is essential to perform these checks before installing KubeVirt.

Recommended order of troubleshooting:

    Check whether the Kubernetes cluster is healthy.
        |
        v
    Verify if the nodes support KVM.
        |
        v
    Confirm if /dev/kvm exists.
        |
        v
    Ensure that storage is available.
        |
        v
    Check if the network and DNS are functioning correctly.
        |
        v
    Verify that the image repository is accessible.
        |
        v
    Then proceed with installing KubeVirt.

---

## III. Experimental Environment Planning

This document uses the following cluster as an example:

    k8s-master-01     10.0.0.20
    k8s-master-02     10.0.0.21
    k8s-master-03     10.0.0.22
    k8s-worker-01     10.0.0.23
    k8s-worker-02     10.0.0.24
    k8s-worker-03     10.0.0.25

Recommendations:

    KubeVirt virtual machines should preferably run on Worker nodes.
    Master nodes should not be used for regular businessHowever, if /dev/kvm does not exist, it will be very difficult for KubeVirt VMs to function properly.

---

## Section VII: Checking KVM Kernel Modules

### 7.1 Viewing the kvm Module

Execute:

    lsmod | grep kvm

Common output for Intel nodes:

    kvm_intel
    kvm

Common output for AMD nodes:

    kvm_amd
    kvm

If no output is displayed, you can attempt to load the module.

---

### 7.2 Loading Modules on Intel Nodes

Execute:

    sudo modprobe kvm

    sudo modprobe kvm_intel

Check:

    lsmod | grep kvm

---

### 7.3 Loading Modules on AMD Nodes

Execute:

    sudo modprobe kvm

    sudo modprobe kvm_amd

Check:

    lsmod | grep kvm

---

### 7.4 Automatically Loading KVM Modules at Boot

For Intel nodes:

    cat <<EOF | sudo tee /etc/modules-load.d/kvm.conf
    kvm
    kvm_intel
    EOF

For AMD nodes:

    cat <<EOF | sudo tee /etc/modules-load.d/kvm.conf
    kvm
    kvm_amd
    EOF

Check:

    cat /etc/modules-load.d/kvm.conf

Note:

    Do not forcibly configure both kvm_intel and kvm_amd on different CPU platforms.
    Configure them according to your actual CPU type.

---

## Section VIII: Installing Virtualization Inspection Tools

### 8.1Installing cpu-checker and libvirt-clients

For Ubuntu 22.04 nodes:

    sudo apt update

    sudo apt install -y cpu-checker libvirt-clients

Note:

    cpu-checker provides the kvm-ok command.
    libvirt-clients provide inspection tools such as virt-host-validate.

---

### 8.2 Using kvm-ok for Inspection

Execute:

    sudo kvm-ok

A normal output would be similar to:

    INFO: /dev/kvm exists
    KVM acceleration can be used

If the output is:

    KVM acceleration can NOT be used

It indicates that the current node cannot use KVM properly.

In this case, you need to revisit the previous checks:

    CPU virtualization support
    /dev/kvm availability
    KVM module installation
    BIOS virtualization settings
    Nested virtualization configuration

---

### 8.3 Using virt-host-validate for Inspection

Execute:

    sudo virt-host-validate qemu

Common outputs include:

    PASS
    WARN
    FAIL

Pay special attention to:

    QEMU: Checking for hardware virtualization
    QEMU: Checking for device /dev/kvm
    QEMU: Checking for device /dev/net/tun
    QEMU: Checking for cgroup controllers
    QEMU: Checking for IOMMU

Note:

    PASS indicates compliance.
    WARN may not prevent basic use but requires attention.
    FAIL indicates a critical issue that needs to be addressed.

For beginners, focus on:

    Whether /dev/kvm returns PASS
    Whether hardware virtualization is supported

---

## Section IX: Checking if it is a Nested Virtualization Environment

Many experimental environments are run within VMware Workstation, vSphere, Proxmox, cloud hosting, or other virtualization platforms.

In such cases, to support KubeVirt, nested virtualization usually needs to be enabled.

### 9.1 Determining if the Current System is a Virtual Machine

Execute:

    systemd-detect-virt

Common outputs include:

    vmware
    kvm
    oracle
    microsoft
    none

If the output is not "none", it indicates that the current node is likely running within a virtualization environment.

---

### 9.2 Requirements for Nested Virtualization

If the Kubernetes node itself is a virtual machine, the underlying virtualization platform must support:

    Nested virtualization
    Exposure of hardware-assisted virtualization to the guest
    Permission for /dev/kvm to be present on the node

Different platforms may use different terms for these requirements:

    VMware:
        Expose hardware assisted virtualization to the guest OS

    Proxmox:
        host CPU / nested virtualization

    KVM:
        nested=1

    Public clouds:
    You need to select an instance specification that supports nested virtualization.

---

### 9.3 Handling the Absence of vmx/svm on the Current Node

If:

    egrep -c '(vmx|svm)' /proc/cpuinfo

Returns 0, it is very likely that KubeVirt will not be able to use KVM acceleration properly.

Possible solutions include:

    1. If it is a physical machine, go to the BIOS and enable VT-x/AMD-V.
    2. If it is a VMware virtual machine, enable nested virtualization options.
    3. If it is vSphere, check the virtual machine's CPU virtualization exposure settings.
    4. If it is1. Dedicated Worker Nodes for VMs
2. Labeling VM Nodes
3. Tainting VM Nodes
4. Configuring Tolerance for VMs
5. Preventing Ordinary Pods from Being Scheduled to VM Nodes

Example label:

    node-role.kubernetes.io/kubevirt=

Example taint:

    kubectl taint node k8s-worker-01 workload=kubevirt:NoSchedule

Subsequent VMs need to have tolerance configured before they can be scheduled.

During the introductory experiment phase, it is advisable not to perform tainting isolation to avoid complicating things.### 18.1 Checking CNI Components

**Calico Example:**

```bash
kubectl get pods -A -o wide | grep calico
```

**Flannel Example:**

```bash
kubectl get pods -A -o wide | grep flannel
```

**Requirement:** The CNI components must be running.

---

### 18.2 Creating a Network Test Pod

**Creation:**

```bash
kubectl run kubevirt-net-test \
      -n kubevirt-precheck \
      --image=busybox:1.36 \
      --restart=Never \
      --sleep 3600
```

**Viewing:**

```bash
kubectl -n kubevirt-precheck get pod kubevirt-net-test -o wide
```

**Entering the Test Pod:**

```bash
kubectl -n kubevirt-precheck exec -it kubevirt-net-test -- sh
```

**Testing DNS:**

```bash
nslookup kubernetes.default
```

**Testing Service:**

```bash
wget -qO- https://kubernetes.default.svc --no-check-certificate
```

**Exiting:**

```bash
exit
```

If there are issues with the DNS or Service network, repair the underlying cluster network first.

---

### 18.3 Testing Pod Inter-node Communication

**Viewing the Node Where the Test Pod Is Located:**

```bash
kubectl -n kubevirt-precheck get pod kubevirt-net-test -o wide
```

If there are other test pods in the cluster, you can test inter-Pod IP communication.

You can also create a second pod:

```bash
kubectl run kubevirt-net-test-2 \
      -n kubevirt-precheck \
      --image=busybox:1.36 \
      --restart=Never \
      --sleep 3600
```

**Viewing the IPs of Both Pods:**

```bash
kubectl -n kubevirt-precheck get pods -o wide
```

**Entering One Pod to Access the Other Pod's IP:**

```bash
kubectl -n kubevirt-precheck exec -it kubevirt-net-test -- sh

ping <otherPodIP>
```

**Note:** If regular pod networking is not working, KubeVirt VM network is likely to be problematic as well.

---

## Section 19: Checking DNS Capability

KubeVirt-related components and subsequent VM access to services rely on DNS.

**Viewing CoreDNS:**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
```

**Viewing the kube-dns Service:**

```bash
kubectl -n kube-system get svc kube-dns
```

**Executing Commands in the Test Pod:**

```bash
kubectl -n kubevirt-precheck exec -it kubevirt-net-test -- nslookup kubernetes.default

kubectl -n kubevirt-precheck exec -it kubevirt-net-test -- nslookup kubernetes.default.svc.cluster.local
```

If resolution fails, check CoreDNS first.

---

## Section 20: Checking Access to the Image Repository

KubeVirt installation requires pulling multiple images. Subsequent processes like CDI, virt-launcher, and test VMs also involve image retrieval.

### 20.1 Checking If Nodes Can Pull Images

**Execution on a Node:**

```bash
sudo crictl pull busybox:1.36
```

**Viewing:**

```bash
sudo crictl images | grep busybox
```

If public network images cannot be pulled, prepare the following:

1. Internal Harbor
2. Image synchronization
3. Containerd image acceleration or HTTP registry configuration
4. `imagePullSecret`

---

### 20.2 Recommendations for Domestic Environments

In production or restricted networks, it is recommended to:

1. Confirm the list of images required for KubeVirt in advance.
2. Synchronize images to an internal Harbor.
3. Fixate the KubeVirt version and image versions.
4. Avoid relying on temporary public network pulls.
5. Test `crictl pull` on all nodes before installation.

**Note:** The subsequent installation section will address specific issues related to KubeVirt Operators and images. This section only focuses on preliminary connectivity checks.

---

## Section 21: Checking Security Policies

KubeVirt requires running specialized virtualization workloads. If the cluster has strict security policies, verify whether the following are allowed:

1. `privileged` permissions
2. Use of `/dev/kvm`
3. Mounting devices
4. Utilization of specific capabilities
5. Access to node virtualization functions

**Checking if Pod Security Admission Labels Are Enabled:**

```bash
kubectl get ns --show-labels
```

If KubeVirt will be installed in the `kubevirt` namespace, you can set security labels for the namespace as needed```markdown
kubectl label node k8s-worker-03 kubevirt.io/kvm-ready=true --overwrite

View:

kubectl get nodes -l kubevirt.io/kvm-ready=true -o wide

Explanation:

This is a custom label. In subsequent steps, you can use the `nodeSelector` to schedule tasks onto these nodes. It may not be necessary for beginner experiments but is very useful in production planning.
---

## Experiment 27: Creating a Node Check Script

Experiment Objective:

To quickly check whether the current node meets the basic requirements for running KubeVirt VMs.

Create a script on each Worker node:

cat <<'EOF' > /tmp/kubevirt-node-precheck.sh
#!/usr/bin/env bash
set -e

echo "===== Hostname ====="
hostname

echo
echo "===== OS ====="
cat /etc/os-release | grep -E "PRETTY_NAME|VERSION_ID"

echo
echo "===== Kernel ====="
uname -r

echo
echo "===== CPU Virtualization Flag ====="
VIRT_COUNT=$(egrep -c '(vmx|svm)' /proc/cpuinfo || true)
echo "vmx/svm count: ${VIRT_COUNT}"

echo
echo "===== /dev/kvm ====="
if [ -e /dev/kvm ]; then
  ls -l /dev/kvm
else
  echo "/dev/kvm not found"
fi

echo
echo "===== KVM Modules ====="
lsmod | grep kvm || echo "kvm module not loaded"

echo
echo "===== systemd-detect-virt ====="
systemd-detect-virt || true

echo
echo "===== Disk ====="
df -h | grep -E '^Filesystem|/$|/data|/var'

echo
echo "===== Memory ====="
free -h

echo
echo "===== containerd ====="
systemctl is-active containerd || true

echo
echo "===== kubelet ====="
systemctl is-active kubelet || true

echo
echo "===== kvm-ok ====="
if command -v kvm-ok >/dev/null 2>&1; then
  sudo kvm-ok || true
else
  echo "kvm-ok not installed"
fi

echo
echo "===== virt-host-validate ====="
if command -v virt-hostvalidate >/dev/null 2>&1; then
  sudo virt-host-validation qemu || true
else
  echo "virt-host-validate not installed"
fi
EOF

Grant execute permissions:

chmod +x /tmp/kubevirt-node-precheck.sh

Execute the script:

/tmp/kubevirt-node-precheck.sh

Key points to check:

- Whether the vmx/svm count is greater than 0.
- Whether /dev/kvm exists.
- Whether the KVM module is loaded.
- Whether kvm-ok passes the test.
- Whether virt-host-validate shows any errors.
- Whether containerd and kubelet are active.
---

## Experiment 28: Creating a Cluster-Level Check List

Experiment Objective:

To quickly check the basic Kubernetes status before installing KubeVirt on the Master node.

Execute this on k8s-master-01:

cat <<'EOF' > /tmp/kubevirt-cluster-precheck.sh
#!/usr/bin/env bash
set -e

echo "===== Nodes ====="
kubectl get nodes -o wide

echo
echo "===== kube-system Pods ====="
kubectl -n kube-system get pods -o wide

echo
echo "===== CNI Pods ====="
kubectl get pods -A -o wide | grep -E "calico|flannel|cilium" || true

echo
echo "===== StorageClass ====="
kubectl get storageclass

echo
echo "===== PVC All Namespace ====="
kubectl get pvc -A || true

echo
echo "===== CoreDNS ====="
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system get svc kube-dns

echo
echo "===== Metrics ====="
kubectl top nodes || true

echo
echo "===== KVM Ready Nodes Label ====="
kubectl get nodes -l kubevirt.io/kvm-ready=true -o wide || true
EOF

Grant execute permissions:

chmod +x /tmp/kubevirt-cluster-precheck.sh

Execute the script:

/tmp/kubevirt-cluster-precheck.sh
---

## Chapter 29: Common Issues and Solutions

### 29.1 vmx/svm Output is 0

Symptom:

`egrep -c '(vmx|svm)' /proc/cpuinfo` returns 0.

Cause:

1. Virtualization is not enabled in the BIOS.
2. The current node is a virtual machine but nested virtualization is disabled.
3. The cloud hosting specifications do not support nested virtualization.
4. The CPU does not support virtualization.

Solution:

For physical machines:
Enter the BIOS and enable Intel VT-x or AMD-V.

For VMware/vSphere:
```markdown
kubectl -n kube-system get endpoints kube-dns

kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

DNS issues can affect component access, business functionality, and subsequent VM network tests.

---

### 29.6 Insufficient Node Resources

Symptoms:

    Pod Pending
    Insufficient CPU
    Insufficient memory

Checks:

    kubectl describe node <node-name>

    kubectl top nodes

Actions:

    1. Reduce the specifications of the experimental VMs.
    2. Remove unnecessary Pods.
    3. Expand the node capacity.
    4. Increase node resources.
    5. Adjust resource requests accordingly.

---

## Thirty, Pre-Installation Checklist

Before installing KubeVirt, ensure the following:

    1. All Kubernetes Nodes are Ready.
    2. Core components of kube-system are functioning correctly.
    3. CNI networking is operational.
    4. CoreDNS is working properly.
    5. containerd is running smoothly.
    6. kubelet is functioning normally.
    7. Nodes where VMs will be deployed have vmx or svm support.
    8. /dev/kvm exists on the designated nodes.
    9. KVM kernel modules are loaded.
    10. The kvm-ok test passes.
    11. virt-host-validate shows no critical failures.
    12. StorageClasses are available.
    13. PVCs can be bound successfully.
    14. Pods can mount and read from/write to PVCs.
    15. Nodes have sufficient disk space.
    16. Sufficient memory and CPU resources are allocated.
    17. Access to the image repository is functioning correctly.
    18. Time synchronization is in place.
    19. In virtualized environments, nested virtualization is enabled.
    20. Nodes supporting KVM have appropriate tags for scheduling purposes.

---

## Thirty-One, Recommendations for Production Environments

Before installing KubeVirt in a production environment, consider the following additional factors:

    1. Planning for dedicated VM node pools.
    2. Determining the StorageClass to be used for VM disks.
    3. Assessing the need for Longhorn/Ceph/commercial storage solutions.
    4. Checking support for snapshots and backups.
    5. Evaluating the requirement for LiveMigration.
    6. Deciding whether Multus multi-NICs are necessary.
    7. Determining if integration with traditional Layer 2 networks is required.
    8. Ensuring availability of VM image repositories and import processes.
    9. Setting up monitoring and alert systems.
    10. Establishing backup and recovery procedures for VMs.
    11. Implementing resource quotas and permission controls.
    12. Having a plan in place for handling VM failures.

The setup process can be simplified for introductory experiments, but production environments require additional considerations.

---

## Thirty-Two, Clearing Up Experimental Resources

Clean up network test Pods:

    kubectl -n kubevirt-precheck delete pod kubevirt-net-test --ignore-not-found

    kubectl -n kubevirt-precheck delete pod kubevirt-net-test-2 --ignore-not-found

Clean up PVC test Pods:

    kubectl -n kubevirt-precheck delete pod kubevirt-precheck-pvc-test --ignore-not-found

Clean up PVCs:

    kubectl -n kubevirt-precheck delete pvc kubevirt-precheck-pvc --ignore-not-found

Clean up the namespace:

    kubectl delete namespace kubevirt-precheck

If you wish to retain the testing namespace for future use, you can skip this step.

---

## Thirty-Three, Summary of This Article

The key focus before installing KubeVirt is to ensure that not only is the Kubernetes cluster Ready, but also that each node has the necessary capabilities to run virtual machines. Important checks include:

    1. Presence of vmx/svm on CPUs.
    2. Availability of /dev/kvm.
    3. Loading of KVM kernel modules.
    4. Successful kvm-ok test.
    5. Absence of critical failures in virt-host-validate.
    6. Functional StorageClasses.
    7. Ability to bind PVCs.
    8. Proper mounting and read/write capabilities for Pods using PVCs.
    9. Normal operation of CNI and DNS.
    10. Sufficient node resources.

Key commands for verification include:

    egrep -c '(vmx|svm)' /proc/cpuinfo
    ls -l /dev/kvm
    lsmod | grep kvm
    sudo kvm-ok
    sudo virt-host-validate qemu
    kubectl get storageclass
    kubectl get pvc -A
    kubectl get nodes -o wide
   