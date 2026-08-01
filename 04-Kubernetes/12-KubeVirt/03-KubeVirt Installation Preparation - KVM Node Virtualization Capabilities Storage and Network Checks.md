# 03-KubeVirt Pre-Installation Preparation: KVM, Node Virtualization Capabilities, Storage and Network Check

Recommended path:

    04-Kubernetes/12-KubeVirt/03-KubeVirt Pre-Installation Preparation: KVM, Node Virtualization Capabilities, Storage and Network Check.md

Tags:

    #Kubernetes
    #KubeVirt
    #KVM
    #QEMU
    #Virtual
    #NodeCheck
    #StorageClass
    #PVC
    #CNI
    #Ubuntu2204
    #CloudlandVirtualization

---

## I. Document Description

This document records the environment checks and preparations required before installing KubeVirt.

KubeVirt differs from regular Kubernetes Addons.

Regular components can typically be deployed as long as the cluster is Ready, images can be pulled, and Pods can run.

However, KubeVirt runs virtual machines on Kubernetes nodes, so additional checks are required:

    1. Whether the node CPU supports hardware virtualization
    2. Whether BIOS / virtualization platform has enabled VT-x / AMD-V
    3. Whether /dev/kvm exists on the node
    4. Whether the KVM kernel module is loaded
    5. Whether the current environment is nested virtualization
    6. Whether Kubernetes node resources are sufficient
    7. Whether StorageClass / PVC is available
    8. Whether CNI network is normal
    9. Whether DNS is normal
    10. Whether the image repository is accessible
    11. Whether the node installing KubeVirt allows scheduling virtual machines
    12. Whether the node kernel, containerd, and kubelet are stable

Document objectives:

    1. Complete node virtualization capability checks before installing KubeVirt
    2. Identify issues such as /dev/kvm not existing, CPU not supporting virtualization, and nested virtualization not enabled in advance
    3. Pre-validate PVC, StorageClass, network, and DNS
    4. Lay the foundation for subsequent KubeVirt installation and creating the first virtual machine

Applicable environments:

    Ubuntu 22.04
    kubeadm self-built Kubernetes cluster
    containerd runtime
    Calico / Flannel and other common CNI
    Longhorn / NFS / other CSI StorageClass

---

## II. Why Pre-Installation Checks Are Necessary

KubeVirt relies on node virtualization capabilities.

If nodes are not properly prepared, the following issues may occur:

    1. KubeVirt can be installed, but virtual machines cannot start
    2. VM creation succeeds, but VMI remains Pending
    3. virt-launcher Pod creation fails
    4. virt-launcher Pod is Running, but Guest OS cannot start
    5. KVM-related errors appear in logs
    6. Poor virtual machine performance
    7. DataVolume image import succeeds, but virtual machine cannot mount disks
    8. VM network connectivity issues
    9. Console connection failure
    10. LiveMigration or other advanced capabilities cannot be used

Therefore, it is essential to perform checks before installing KubeVirt.

Recommended troubleshooting order:

    Kubernetes cluster health
        |
        v
    Node KVM support
        |
        v
    /dev/kvm existence
        |
        v
    Storage availability
        |
        v
    Network / DNS normality
        |
        v
    Image repository accessibility
        |
        v
    Then install KubeVirt

---

## III. Experiment Environment Planning

This document uses the following cluster as an example:

    k8s-master-01     10.0.0.20
    k8s-master-02     10.0.0.21
    k8s-master-03     10.0.0.22
    k8s-worker-01     10.0.0.23
    k8s-worker-02     10.0.0.24
    k8s-worker-03     10.0.0.25

Recommendations:

    KubeVirt virtual machines should prioritize running on Worker nodes.
    Master nodes typically do not run regular workloads or virtual machines.
    If the experimental environment has few nodes, Master nodes can temporarily allow scheduling, but this is not recommended for production.

This document assumes the following nodes will host virtual machines:

    k8s-worker-01
    k8s-worker-02
    k8s-worker-03

---

## IV. Check Kubernetes Cluster Status

### 4.1 Check Node Status

Execute:

    kubectl get nodes -o wide

Requirements:

    All nodes Ready.

Example:

    NAME            STATUS   ROLES           INTERNAL-IP
    k8s-master-01   Ready    control-plane   10.0.0.20
    k8s-master-02   Ready    control-plane   10.0.0.21
    k8s-master-03   Ready    control-plane   10.0.0.22
    k8s-worker-01   Ready    <none>          10.0.0.23
    k8s-worker-02   Ready    <none>          10.0.0.24
    k8s-worker-03   Ready    <none>          10.0.0.25

If any nodes are NotReady, do not install KubeVirt yet.

First troubleshoot:

    kubelet
    containerd
    CNI
    Node network
    Disk pressure
    Memory pressure

---

### 4.2 Check System Components

Execute:

    kubectl get pods -A -o wide

Focus on confirming:

    kube-system
    calico-system or CNI-related namespace
    ingress-nginx
    cert-manager
    longhorn-system or storage-system

Basic requirements:

    CoreDNS Running
    kube-proxy Running
    CNI Running
    Storage components Running

---

### 4.3 Check Node Resources

Execute: /think

kubectl describe nodes | grep -A8 "Allocated resources"

If metrics-server is installed:

    kubectl top nodes

Focus on:

    CPU sufficiency
    Memory sufficiency
    Whether Worker nodes are fully utilized by workloads

KubeVirt virtual machines require higher resource allocation than regular Pods.

For example, a small VM may need:

    CPU: 1 Core
    Memory: 1Gi or 2Gi

If node resources are tight, virt-launcher Pods may become Pending.

---

## Five. Check Node CPU Virtualization Capabilities

The following operations are executed on each node planned to run KubeVirt VMs.

Example:

    k8s-worker-01
    k8s-worker-02
    k8s-worker-03

---

### 5.1 Check if CPU Supports Virtualization Extensions

For Intel CPUs, check vmx:

    egrep -c '(vmx)' /proc/cpuinfo

For AMD CPUs, check svm:

    egrep -c '(svm)' /proc/cpuinfo

General check:

    egrep -c '(vmx|svm)' /proc/cpuinfo

If output is greater than 0, it indicates the CPU exposes virtualization extensions.

Example:

    16

Means the system can see virtualization instruction sets.

If output is:

    0

It means the node cannot detect virtualization extensions.

Common causes:

    1. BIOS not enabled virtualization
    2. Current node is a VM without nested virtualization
    3. Public cloud instance specs do not support nested virtualization
    4. Virtualization platform does not expose CPU virtualization capabilities to Guest
    5. Host CPU does not support virtualization

---

### 5.2 Intel and AMD Field Explanations

Intel:

    vmx

Represents Intel VT-x virtualization extensions.

AMD:

    svm

Represents AMD-V virtualization extensions.

Judgment rule:

    Only nodes with vmx or svm have the basic conditions for KVM virtualization.

Note:

    Presence of vmx/svm does not guarantee KubeVirt availability.
    Also check /dev/kvm and KVM kernel modules.

---

## Six. Check /dev/kvm

### 6.1 View /dev/kvm

Execute on the node:

    ls -l /dev/kvm

Normal example:

    crw-rw---- 1 root kvm 10, 232 Apr 26 10:00 /dev/kvm

If /dev/kvm exists, it means KVM device is exposed.

If it does not exist:

    ls: cannot access '/dev/kvm': No such file or directory

Means the node cannot directly use KVM.

Common causes:

    1. CPU virtualization not enabled
    2. Current environment does not support nested virtualization
    3. kvm kernel module not loaded
    4. Node is a restricted VM
    5. Cloud provider instance does not support KVM

---

### 6.2 Check kvm Device Permissions

View:

    ls -l /dev/kvm

Common permissions:

    root:kvm

Check kvm group:

    getent group kvm

Manual modification of /dev/kvm permissions is generally unnecessary.

KubeVirt components will use it via privileged or device mounting.

However, if /dev/kvm does not exist, KubeVirt VMs will have difficulty running normally.

---

## Seven. Check KVM Kernel Modules

### 7.1 View kvm Modules

Execute:

    lsmod | grep kvm

Intel node common output:

    kvm_intel
    kvm

AMD node common output:

    kvm_amd
    kvm

If no output, try loading it.

---

### 7.2 Load Modules for Intel Nodes

Execute:

    sudo modprobe kvm
    sudo modprobe kvm_intel

Check:

    lsmod | grep kvm

---

### 7.3 Load Modules for AMD Nodes

Execute:

    sudo modprobe kvm
    sudo modprobe kvm_amd

Check:

    lsmod | grep kvm

---

### 7.4 Enable KVM Modules on Boot

Intel node:

    cat <<EOF | sudo tee /etc/modules-load.d/kvm.conf
    kvm
    kvm_intel
    EOF

AMD node:

    cat <<EOF | sudo tee /etc/modules-load.d/kvm.conf
    kvm
    kvm_amd
    EOF

Check:

    cat /etc/modules-load.d/kvm.conf

Note:

    Do not force write both kvm_intel and kvm_amd for different CPU platforms.
    Configure according to actual CPU type.

---

## Eight. Install Virtualization Check Tools

### 8.1 Install cpu-checker and libvirt-clients

Execute on Ubuntu 22.04 nodes:

    sudo apt update
    sudo apt install -y cpu-checker libvirt-clients

Note:

    cpu-checker provides the kvm-ok command.
    libvirt-clients provides tools like virt-host-validate for checks.

---

### 8.2 Use kvm-ok to Check

Execute:

    sudo kvm-ok

Normal output is similar to:

    INFO: /dev/kvm exists
    KVM acceleration can be used

If output is:

    KVM acceleration can NOT be used

It means the node cannot use KVM normally.

Return to previous checks:

    CPU virtualization extensions
    /dev/kvm
    kvm modules
    BIOS virtualization
    Nested virtualization

---

### 8.3 Use virt-host-validate to Check

Execute:

    sudo virt-host-validate qemu

Common outputs include:

    PASS
    WARN
    FAIL

Focus on: /think

QEMU: Checking for hardware virtualization  
QEMU: Checking for device /dev/kvm  
QEMU: Checking for device /dev/net/tun  
QEMU: Checking for cgroup controllers  
QEMU: Checking for IOMMU  

**Note:**  
- **PASS** indicates the requirement is met.  
- **WARN** may not block basic usage but requires attention.  
- **FAIL** requires immediate resolution.  

**Focus during initial setup:**  
- Whether /dev/kvm is **PASS**  
- Whether hardware virtualization is **PASS**  

---

## Nine. Checking for nested virtualization environment  

Many experimental environments run on VMware Workstation, vSphere, Proxmox, cloud hosts, or other virtualization platforms.  

In such cases, enabling nested virtualization is typically required to support KubeVirt.  

### 9.1 Determine if the current system is a virtual machine  

Execute:  
```bash  
systemd-detect-virt  
```  

Common outputs:  
```text  
vmware  
kvm  
oracle  
microsoft  
none  
```  

If the output is not **none**, it likely indicates the node is running in a virtualization environment.  

---

### 9.2 Requirements for nested virtualization  

If the Kubernetes node itself is a virtual machine, the host virtualization platform must support:  
- Nested virtualization  
- Exposing hardware-assisted virtualization to the guest  
- Allowing /dev/kvm to appear inside the node  

Different platforms have different terminology:  
- **VMware**:  
  Expose hardware assisted virtualization to the guest OS  
- **Proxmox**:  
  host CPU / nested virtualization  
- **KVM**:  
  nested=1  
- **Public cloud**:  
  Choose an instance specification that supports nested virtualization  

---

### 9.3 Handling when the current node lacks vmx / svm  

If:  
```bash  
egrep -c '(vmx|svm)' /proc/cpuinfo  
```  

Outputs 0, KubeVirt may not function properly with KVM acceleration.  

**Solutions:**  
1. If it's a physical machine, enable VT-x / AMD-V in BIOS  
2. If it's a VMware virtual machine, enable nested virtualization option  
3. If it's vSphere, check the virtual machine's CPU virtualization exposure settings  
4. If it's Proxmox/KVM, enable nested virtualization  
5. If it's a cloud host, confirm the instance specification supports nested virtualization  
6. If not supported, switch nodes or environments  

---

## Ten. Checking node kernel and system version  

### 10.1 Check operating system  

Execute:  
```bash  
cat /etc/os-release  
```  

Expected output:  
```text  
Ubuntu 22.04  
```  

---

### 10.2 Check kernel version  

Execute:  
```bash  
uname -r  
```  

Example output:  
```text  
5.15.0-xxx-generic  
```  

**Note:**  
- KubeVirt depends on Linux KVM capabilities.  
- Low kernel versions or custom kernels may cause compatibility issues.  
- Ubuntu 22.04's default kernel usually meets basic experimental requirements.  

---

### 10.3 Check system architecture  

Execute:  
```bash  
uname -m  
```  

Common outputs:  
```text  
x86_64  
```  

This document assumes:  
```text  
x86_64 / amd64  
```  

If it's ARM architecture, images, virtualization support, and installation methods require special confirmation.  

---

## Eleven. Checking containerd and kubelet  

KubeVirt's virt-launcher is ultimately a Pod.  

Therefore, containerd and kubelet must be healthy.  

### 11.1 Check containerd  

Execute:  
```bash  
systemctl status containerd --no-pager  
```  

Check configuration:  
```bash  
containerd config dump | grep -E "SystemdCgroup|sandbox_image"  
```  

Confirm:  
```text  
SystemdCgroup = true  
```  

Check crictl:  
```bash  
sudo crictl info  
sudo crictl ps  
```  

If containerd is abnormal, do not install KubeVirt.  

---

### 11.2 Check kubelet  

Execute:  
```bash  
systemctl status kubelet --no-pager  
```  

Check logs:  
```bash  
sudo journalctl -u kubelet -n 100 --no-pager  
```  

Confirm no obvious errors:  
```text  
container runtime is down  
cni config uninitialized  
network plugin is not ready  
failed to contact API server  
certificate expired  
```  

---

## Twelve. Checking if node allows scheduling VM  

### 12.1 Check node taint  

Execute:  
```bash  
kubectl describe node k8s-worker-01 | grep -i taints -A2  
```  

Check all node taints:  
```bash  
kubectl describe nodes | grep -i taints -A2  
```  

Common master taint:  
```text  
node-role.kubernetes.io/control-plane:NoSchedule  
```  

This indicates ordinary Pods and VMs cannot be scheduled to master by default.  

Worker nodes should typically not have scheduling-blocking taints.  

---

### 12.2 Check node label  

Execute:  
```bash  
kubectl get nodes --show-labels  
```  

Later, you can label nodes supporting KVM.  

Example:  
```bash  
kubectl label node k8s-worker-01 kubevirt.io/kvm-ready=true  
kubectl label node k8s-worker-02 kubevirt.io/kvm-ready=true  
kubectl label node k8s-worker-03 kubevirt.io/kvm-ready=true  
```  

Check:  
```bash  
kubectl get nodes -l kubevirt.io/kvm-ready=true  
```  

**Note:**

This is a custom label to facilitate scheduling VMs to nodes supporting KVM.
Whether this label is used depends on whether the subsequent VM YAML configures nodeSelector / affinity.

---

### 12.3 Planning KubeVirt Dedicated Nodes (Optional)

If you want VMs to run only on specific nodes in production, you can design:

    1. Dedicated Worker Nodes for VMs
    2. Label VM nodes
    3. Add taints to VM nodes
    4. Configure toleration for VMs
    5. Prevent regular Pods from scheduling to VM nodes

Example label:

    node-role.kubernetes.io/kubevirt=

Example taint:

    kubectl taint node k8s-worker-01 workload=kubevirt:NoSchedule

Subsequent VMs need to configure toleration to schedule.

During initial experimentation, you can skip taint isolation to avoid complexity.

---

## Thirteen. Checking Storage Capabilities

KubeVirt virtual machines typically require PVCs as disks.

Therefore, you must confirm StorageClass availability before installation.

---

### 13.1 Viewing StorageClass

Execute:

    kubectl get storageclass

Example:

    NAME                   PROVISIONER
    nfs-client (default)   cluster.local/nfs-subdir-external-provisioner
    longhorn               driver.longhorn.io

Recommendations:

    KubeVirt experiments can prioritize using longhorn.
    NFS can also be used for some experiments, but block storage scenarios are more recommended to use Longhorn / Ceph / Cloud Disk CSI.

---

### 13.2 Checking Default StorageClass

Execute:

    kubectl get storageclass

If you see:

    (default)

It indicates there is a default StorageClass.

If there is no default StorageClass, subsequent PVCs need to explicitly write:

    storageClassName: longhorn

Or manually set the default:

    kubectl patch storageclass longhorn \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

Notes:

    Confirm there are no other default classes in the cluster before setting the default StorageClass.
    Do not arbitrarily modify the default StorageClass in production environments.

---

## Fourteen. Experiment One: Verifying PVC Dynamic Provisioning

Experiment Objective:

    Confirm the cluster can normally create PVCs and PVs.

Create a test namespace:

    kubectl create namespace kubevirt-precheck

Create PVC:

    cat <<EOF > pvc-kubevirt-precheck.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: kubevirt-precheck-pvc
      namespace: kubevirt-precheck
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: longhorn
    EOF

Notes:

    If your StorageClass is not longhorn, you need to replace storageClassName.
    If you want to use the default StorageClass, you can remove the storageClassName field.

Apply:

    kubectl apply -f pvc-kubevirt-precheck.yaml

Check PVC:

    kubectl -n kubevirt-precheck get pvc

Expected:

    STATUS should be Bound

Check PV:

    kubectl get pv | grep kubevirt-precheck

If PVC is Pending, first troubleshoot:

    Whether StorageClass exists
    Whether Provisioner is normal
    Whether Longhorn / NFS / CSI is normal
    Whether node storage is available

---

## Fifteen. Experiment Two: Verifying PVC Mount Read/Write

Experiment Objective:

    Confirm Pods can normally mount PVCs and read/write.

Create a test Pod:

    cat <<EOF > pod-kubevirt-precheck-pvc.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: kubevirt-precheck-pvc-test
      namespace: kubevirt-precheck
    spec:
      containers:
      - name: busybox
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "kubevirt pvc precheck $(date)" > /data/precheck.txt
          sleep 3600
        volumeMounts:
        - name: test-data
          mountPath: /data
      volumes:
      - name: test-data
        persistentVolumeClaim:
          claimName: kubevirt-precheck-pvc
    EOF

Apply:

    kubectl apply -f pod-kubevirt-precheck-pvc.yaml

Check Pod:

    kubectl -n kubevirt-precheck get pod -o wide

Verify read/write: /data/precheck.txt

kubectl -n kubevirt-precheck exec -it kubevirt-precheck-pvc-test -- cat /data/precheck.txt

Expected output:

    kubevirt pvc precheck ...

Notes:

    If regular Pods fail to mount PVCs, subsequent KubeVirt VM disk mounts may also fail.

Cleanup test Pod:

    kubectl -n kubevirt-precheck delete pod kubevirt-precheck-pvc-test

Keep PVC or delete it:

    kubectl -n kubevirt-precheck delete pvc kubevirt-precheck-pvc

---

## SixteenI don't know.Check Longhorn, if used

If planning to use Longhorn for KubeVirt VM disks, confirm Longhorn health.

Check Pods:

    kubectl -n longhorn-system get pods -o wide

Check StorageClass:

    kubectl get storageclass longhorn

Check Longhorn nodes:

    kubectl -n longhorn-system get nodes.longhorn.io

Check Longhorn Volumes:

    kubectl -n longhorn-system get volumes.longhorn.io

Check node dependencies:

    systemctl status iscsid --no-pager

    dpkg -l | grep open-iscsi

If open-iscsi is not installed:

    sudo apt update

    sudo apt install -y open-iscsi

    sudo systemctl enable --now iscsid

    sudo systemctl enable --now open-iscsi

Notes:

    Longhorn depends on node iSCSI capabilities.
    If Longhorn PVC mounting fails, KubeVirt VM disk mounting may also fail.

---

## SeventeenI don't know.Check NFS, if used

If planning to use NFS StorageClass, confirm NFS availability.

Check NFS provisioner:

    kubectl -n storage-system get pods -o wide

Check StorageClass:

    kubectl get storageclass nfs-client

Check NFS Server exports:

    showmount -e 10.0.0.10

Manually mount test on node:

    sudo mkdir -p /mnt/nfs-kubevirt-test

    sudo mount -t nfs 10.0.0.10:/data/nfs/k8s /mnt/nfs-kubevirt-test

    echo "kubevirt nfs precheck $(hostname)" | sudo tee /mnt/nfs-kubevirt-test/precheck-$(hostname).txt

    sudo umount /mnt/nfs-kubevirt-test

Notes:

    NFS is more suitable for shared file scenarios.
    VM system disks are recommended to use block storage backends.
    If the experiment environment only has NFS, basic experience can be done first, but production should evaluate performance and consistency risks.

---

## EighteenI don't know.Check network capabilities

KubeVirt VMs default to using Kubernetes Pod network.

Confirm cluster CNI is normal before installation.

---

### 18.1 Check CNI components

Calico example:

    kubectl get pods -A -o wide | grep calico

Flannel example:

    kubectl get pods -A -o wide | grep flannel

Requirements:

    CNI components must be Running.

---

### 18.2 Create network test Pod

Create:

    kubectl run kubevirt-net-test \
      -n kubevirt-precheck \
      --image=busybox:1.36 \
      --restart=Never \
      -- sleep 3600

Check:

    kubectl -n kubevirt-precheck get pod kubevirt-net-test -o wide

Enter test Pod:

    kubectl -n kubevirt-precheck exec -it kubevirt-net-test -- sh

Test DNS:

    nslookup kubernetes.default

Test Service:

    wget -qO- https://kubernetes.default.svc --no-check-certificate

Exit:

    exit

If DNS or Service network issues occur, fix the base cluster network first.

---

### 18.3 Test Pod cross-node communication

Check test Pod's node:

    kubectl -n kubevirt-precheck get pod kubevirt-net-test -o wide

If there are other test Pods in the cluster, test PodIP connectivity.

Alternatively, create a second Pod:

    kubectl run kubevirt-net-test-2 \
      -n kubevirt-precheck \
      --image=busybox:1.36 \
      --restart=Never \
      -- sleep 3600

Check two PodIPs:

    kubectl -n kubevirt-precheck get pods -o wide

Enter one Pod to access another PodIP:

    kubectl -n kubevirt-precheck exec -it kubevirt-net-test -- sh

    ping <another PodIP>

Notes:

    If regular Pod network is not working, KubeVirt VM network may also fail later.

---

## NineteenI don't know.Check DNS capabilities

KubeVirt components and subsequent VM service access depend on DNS.

Check CoreDNS:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

Check kube-dns Service:

    kubectl -n kube-system get svc kube-dns

Execute in the test Pod:

    kubectl -n kubevirt-precheck exec -it kubevirt-net-test -- nslookup kubernetes.default

    kubectl -n kubevirt-precheck exec -it kubevirt-net-test -- nslookup kubernetes.default.svc.cluster.local

If resolution fails, first troubleshoot CoreDNS.

---

## 20. Check Image Repository Access Capability

KubeVirt installation requires pulling multiple images.

Subsequent CDI, virt-launcher, and test VMs will also involve image pulling.

### 20.1 Check if Nodes Can Pull Images

Execute on the node:

    sudo crictl pull busybox:1.36

Check:

    sudo crictl images | grep busybox

If unable to pull public images, prepare:

    1. Internal Harbor
    2. Image synchronization
    3. containerd image acceleration or HTTP registry configuration
    4. imagePullSecret

---

### 20.2 Recommendations for Domestic Environments

In production or restricted networks, recommend:

    1. Confirm the list of required KubeVirt images in advance
    2. Synchronize images to internal Harbor
    3. Fix KubeVirt version and image versions
    4. Avoid relying on temporary public pulls
    5. Test crictl pull on all nodes before installation

Note:

    The installation guide will specifically handle KubeVirt Operator and image issues later.
    This document only performs pre-installation connectivity checks.

---

## 21. Check Security Policies

KubeVirt needs to run special virtualization workloads.

If the cluster has strict security policies enabled, confirm whether the following are allowed:

    1. privileged permissions
    2. Use of /dev/kvm
    3. Device mounting
    4. Use of specific capabilities
    5. Access to node virtualization capabilities

Check if Pod Security Admission labels are enabled:

    kubectl get ns --show-labels

If the installation is later in the kubevirt namespace, you can set security labels as needed for the namespace.

Example:

    kubectl label namespace kubevirt \
      pod-security.kubernetes.io/enforce=privileged \
      pod-security.kubernetes.io/audit=privileged \
      pod-security.kubernetes.io/warn=privileged \
      --overwrite

Note:

    Whether this setting is needed depends on the KubeVirt installation method, Kubernetes version, and cluster security policies.
    In production environments, follow company security baselines and do not arbitrarily loosen namespace permissions.

---

## 22. Check Node Time Synchronization

Time desynchronization may cause issues with certificates, API communication, image repository TLS, and log timestamps.

Execute on each node:

    timedatectl

    chronyc sources -v

Requirements:

    System clock synchronized: yes

If not synchronized:

    sudo systemctl restart chrony

Check again:

    timedatectl

---

## 23. Check Node Disk Space

KubeVirt VM images and PVCs may occupy significant space.

Execute on the node:

    df -h

    df -ih

Focus on checking:

    /
    /var
    /data
    /data/containerd
    /data/longhorn
    /var/lib/kubelet

If containerd data directory is located at:

    /data/containerd

Check:

    df -h /data/containerd

    sudo du -sh /data/containerd

If Longhorn data directory is located at:

    /data/longhorn

Check:

    df -h /data/longhorn

Recommendations:

    VM images and disks are much larger than regular container images.
    Do not run KubeVirt VMs on nodes with tight disk space.

---

## 24. Check Memory and CPU Resources

KubeVirt VM resources are generally heavier than regular Pods.

Check node resources:

    free -h

    lscpu

    nproc

Check Kubernetes allocatable resources:

    kubectl describe node k8s-worker-01 | grep -A8 Allocatable

Check allocated resources:

    kubectl describe node k8s-worker-01 | grep -A8 "Allocated resources"

Recommended experimental VM preparation:

    CPU: 1
    Memory: 1Gi

If planning to run multiple VMs, estimate in advance:

    Total CPU requests
    Total Memory requests
    Storage capacity
    Image usage
    Node schedulable resources

---

## 25. Check if Default RuntimeClass Exists in Cluster (Just for Awareness)

KubeVirt generally does not require users to manually configure RuntimeClass to run VMs.

However, if the cluster has special runtime policies, you can check:

    kubectl get runtimeclass

This is just for awareness.

Focus remains on:

    KVM
    /dev/kvm
    virt-handler
    virt-launcher
    PVC
    CNI

---

## 26. Experiment 3: Label Nodes Supporting KVM

Experiment Objective:

    Mark nodes supporting KVM to facilitate VM scheduling later.

Prerequisites:

    Confirmed that the following nodes have /dev/kvm functioning properly:

        k8s-worker-01
        k8s-worker-02
        k8s-worker-03

Label the nodes:

    kubectl label node k8s-worker-01 kubevirt.io/kvm-ready=true --overwrite

kubectl label node k8s-worker-02 kubevirt.io/kvm-ready=true --overwrite

kubectl label node k8s-worker-03 kubevirt.io/kvm-ready=true --overwrite

Check:

kubectl get nodes -l kubevirt.io/kvm-ready=true -o wide

Explanation:

This is a custom label.
When creating VMs later, you can specify scheduling to these nodes via nodeSelector.
Not mandatory for the introductory experiment, but very useful for production planning.

---

## 27. Experiment Four: Create Node Check Script

Experiment Objective:

Quickly check if the current node meets the basic conditions for running KubeVirt VM.

Create the script on each Worker node:

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
if command -

Add permissions:

chmod +x /tmp/kubevirt-node-precheck.sh

Execute:

/tmp/kubevirt-node-precheck.sh

Focus on:

vmx/svm count whether greater than 0
whether /dev/kvm exists
whether kvm module is loaded
whether kvm-ok passes
whether virt-host-validate has FAIL
whether containerd is active
whether kubelet is active

---

## 28. Experiment Five: Create Cluster-Level Check Checklist

Experiment Objective:

Quickly check the Kubernetes base status before KubeVirt installation on the Master node.

Execute on k8s-master-01:

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

```bash
echo
echo "===== KVM Ready Nodes Label ====="
kubectl get nodes -l kubevirt.io/kvm-ready=true -o wide || true
EOF

Add permissions:

    chmod +x /tmp/kubevirt-cluster-precheck.sh

Execute:

    /tmp/kubevirt-cluster-precheck.sh

---

## 29. Common Issues and Troubleshooting

### 29.1 vmx / svm Output is 0

Phenomenon:

    egrep -c '(vmx|svm)' /proc/cpuinfo
    0

Cause:

    1. BIOS not enabled virtualization
    2. Current node is a virtual machine but nested virtualization is not enabled
    3. Cloud instance specifications do not support nested virtualization
    4. CPU does not support virtualization

Resolution:

    Physical machine:
        Enter BIOS to enable Intel VT-x or AMD-V.

    VMware / vSphere:
        Enable exposing hardware-assisted virtualization to Guest.

    KVM / Proxmox:
        Enable nested virtualization.

    Public cloud:
        Replace with an instance specification that supports nested virtualization.

---

### 29.2 /dev/kvm Does Not Exist

Check:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    lsmod | grep kvm

    sudo modprobe kvm

    sudo modprobe kvm_intel

Or AMD:

    sudo modprobe kvm_amd

If it still does not exist, it basically indicates the current environment lacks available KVM capabilities.

---

### 29.3 kvm-ok Fails

Execute:

    sudo kvm-ok

If it shows KVM cannot be used, continue checking:

    CPU virtualization flag
    /dev/kvm
    KVM module
    BIOS
    Nested virtualization

Do not forcibly deploy KubeVirt VM on nodes where kvm-ok fails.

---

### 29.4 PVC Cannot Be Bound

Check:

    kubectl get storageclass

    kubectl describe pvc <pvc-name> -n <namespace>

If using Longhorn:

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get volumes.longhorn.io

If using NFS:

    kubectl -n storage-system get pods -o wide

    showmount -e 10.0.0.10

Do not proceed with installation or VM creation if PVC is abnormal.

---

### 29.5 Network Test Pod Cannot Resolve DNS

Check:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system get svc kube-dns

    kubectl -n kube-system get endpoints kube-dns

    kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

DNS anomalies will affect component access, business access, and subsequent VM network testing.

---

### 29.6 Insufficient Node Resources

Phenomenon:

    Pod Pending
    insufficient cpu
    insufficient memory

Check:

    kubectl describe node <node-name>

    kubectl top nodes

Resolution:

    1. Reduce experimental VM specifications
    2. Clean up unused Pods
    3. Expand nodes
    4. Increase node resources
    5. Adjust requests

---

## 30. Pre-Installation Checklist

Before installing KubeVirt, at least confirm the following:

    1. All Kubernetes Nodes are Ready
    2. kube-system core components are normal
    3. CNI network is normal
    4. CoreDNS is normal
    5. containerd is normal
    6. kubelet is normal
    7. Nodes planned to run VMs have vmx or svm
    8. Nodes planned to run VMs have /dev/kvm
    9. KVM kernel modules are loaded
    10. kvm-ok check passes
    11. virt-host-validate has no critical FAIL
    12. StorageClass exists
    13. PVC can be Bound
    14. Pod can mount PVC and read/write
    15. Node disk space is sufficient
    16. Node memory and CPU resources are sufficient
    17. Image repository access is normal
    18. Time synchronization is normal
    19. If it's a virtualized environment, nested virtualization is enabled
    20. KVM-enabled nodes are labeled for easier scheduling

---

## 31. Production Environment Recommendations

Before installing KubeVirt in production, it's recommended to additionally confirm:

    1. VM dedicated node pool planning
    2. StorageClass for virtual machine disks
    3. Whether to use Longhorn / Ceph / commercial storage
    4. Whether snapshot and backup are supported
    5. Whether LiveMigration is needed
    6. Whether Multus multi-network interface is needed
    7. Whether to connect to traditional Layer 2 network
    8. Whether there is a VM image repository or image import process
    9. Whether there is monitoring and alerting
    10. Whether there is a VM backup and recovery plan
    11. Whether there are resource quotas and permission isolation
    12. Whether there is a VM failureExercise process

Entry-level experiments can be simplified.

Production cannot only pursue "installation".

---

## 32. Clean Up Experimental Resources

Clean up network test Pods:

    kubectl -n kubevirt-precheck delete pod kubevirt-net-test --ignore-not-found

    kubectl -n kubevirt-precheck delete pod kubevirt-net-test-2 --ignore-not-found

Clean up PVC test Pods:

kubectl -n kubevirt-precheck delete pod kubevirt-precheck-pvc-test --ignore-not-found

Clean up PVC:

    kubectl -n kubevirt-precheck delete pvc kubevirt-precheck-pvc --ignore-not-found

Clean up namespace:

    kubectl delete namespace kubevirt-precheck

If you prefer not to delete the test namespace, you can retain it for continued use.

---

## Thirty-Three: Summary of This Article

The core preparation for KubeVirt installation is:

    Do not only check if the Kubernetes cluster is Ready.
    Also check if nodes have the capability to run virtual machines.

Most important checks:

    1. Does the CPU have vmx / svm?
    2. Does /dev/kvm exist?
    3. Is the KVM module loaded?
    4. Does kvm-ok pass?
    5. Does virt-host-validate have critical FAILs?
    6. Is StorageClass available?
    7. Can PVC be Bound?
    8. Can Pod mount PVC for read/write?
    9. Is CNI and DNS functioning normally?
    10. Are node resources sufficient?

Most important commands:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    ls -l /dev/kvm

    lsmod | grep kvm

    sudo kvm-ok

    sudo virt-host-validate qemu

    kubectl get storageclass

    kubectl get pvc -A

    kubectl get nodes -o wide

    kubectl get pods -A -o wide

Experience-based judgment:

    1. If /dev/kvm does not exist, do not rush to install KubeVirt
    2. If PVC is abnormal, do not rush to create VM
    3. If CNI / DNS is abnormal, VM network may also have issues later
    4. VM consumes more resources than regular Pod, node resources need to be planned in advance
    5. In virtualized environments, nested virtualization must be confirmed
    6. The more detailed the pre-installation checks, the fewer troubleshooting issues will occur later

Recommended next article to study:

    04-KubeVirt Installation Practice: Operator, CRD, virtctl and Component Verification.md