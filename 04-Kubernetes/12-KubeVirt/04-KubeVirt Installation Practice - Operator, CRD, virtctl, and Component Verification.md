# 04-KubeVirt Installation Practice: Operator, CRD, virtctl, and Component Verification

Recommended Path:

    04-Kubernetes/12-KubeVirt/04-KubeVirt Installation Practice: Operator, CRD, virtctl, and Component Verification.md

Tags:

    #Kubernetes
    #KubeVirt
    #Operator
    #CRD
    #virtctl
    #Virtualization
    #KVM
    #QEMU
    #Ubuntu2204
    #Cloud-Native Virtualization
    #Platform Engineering

---

## I. Document Description

This document records the practical process of installing KubeVirt in a Kubernetes cluster.

Key Points:

    1. Selecting the appropriate version of KubeVirt for the current Kubernetes version.
    2. Installing the KubeVirt Operator.
    3. Installing the KubeVirt CR.
    4. Verifying core KubeVirt components.
    5.Installing the virtctl command-line tool.
    6. Checking if CRDs such as VM and VMI are available.
    7. Troubleshooting common issues during installation.

Environment:

    Operating System: Ubuntu 22.04
    Kubernetes: v1.31.14
    Container Runtime: containerd
    Installation Method: Official Operator YAML
    KubeVirt Version: v1.4.0
    Execution Node: k8s-master-01

Note:

    This document only covers the installation and component verification of KubeVirt. Creating the first virtual machine will be discussed in a separate article.

---

## II. Why Fix a Specific Version?

There is a version compatibility relationship between KubeVirt and Kubernetes.

The Kubernetes version used in this cluster is:

    v1.31.14

Therefore, the following version of KubeVirt is used:

    KubeVirt v1.4.0

Reasons:

    KubeVirt v1.4 is compatible with Kubernetes v1.31.
    It is not recommended to directly apply the latest version of KubeVirt in a Kubernetes v1.31 cluster.
    The latest version of KubeVirt is usually aligned with the updated Kubernetes version.

Production Recommendations:

    1. Confirm the Kubernetes version before installation.
    2. Verify the KubeVirt support matrix.
    3. Do not blindly use the latest version.
    4. Avoid using incompatible versions in a production environment.
    5. Fix the version before performing image synchronization, installation, and acceptance tests.

Official References:

    https://kubevirt.io/user-guide/cluster_admin/installation/

    https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md

---

## III. Pre-Installation Checks

### 3.1 Checking the Kubernetes Version

Execute:

    kubectl version

Or:

    kubectl get nodes -o wide

Confirm the node version:

    v1.31.14

---

### 3.2 Checking Node Status

Execute:

    kubectl get nodes -o wide

All nodes must be in the Ready state.

If there are any NotReady nodes, do not proceed with KubeVirt installation.

---

### 3.3 Checking kube-system Components

Execute:

    kubectl -n kube-system get pods -o wide

Key components to confirm:

    CoreDNS is Running.
    kube-proxy is Running.
    CNI is Running.
    kube-apiserver is functioning normally.
    kube-controller-manager is functioning normally.
    kube-scheduler is functioning normally.

---

### 3.4 Checking KVM Capabilities

On the Worker nodes where virtual machines will be created:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    ls -l /dev/kvm

    lsmod | grep kvm

    sudo kvm-ok

At least the following should be met:

    1. The number of vmx/svm entries is greater than 0.
    2. /dev/kvm exists.
    3. KVM modules are loaded.
    4. kvm-ok indicates that KVM acceleration is available.

If any requirements are not met, return to the pre-installation preparation notes and address the issues.

---

### 3.5 Checking StorageClass

Execute:

    kubectl get storageclass

It is recommended to have at least one available StorageClass, such as:

    longhorn
    nfs-client

Although KubeVirt does not require immediate creation of business PVCs during installation, subsequent virtual machine disk creation will depend on them.

---

## IV. Creating the Installation Directory

On k8s-master-01, execute:

    mkdir -p /root/k8s-yaml/kubevirt

    cd /root/k8s-yaml/kubevirt

Set### Expectations:

    virt-api should be Running.
    virt-controller should be Running.
    virt-handler should be Running.
    virt-operator should be Running.

### Notes:

    virt-api is usually a Deployment.
    virt-controller is usually a Deployment.
    virt-handler is usually a DaemonSet.
    virt-operator is usually a Deployment.

---

### 8.2 Viewing Deployments

Execute the following command:

    kubectl -n kubevirt get deploy -o wide

Common entries include:

    virt-api
    virt-controller
    virt-operator

To view detailed information:

    kubectl -n kubevirt describe deploy virt-api
    kubectl -n kubevirt describe deploy virt-controller
    kubectl -n kubevirt describe deploy virt-operator

---

### 8.3 Viewing DaemonSets

Execute the following command:

    kubectl -n kubevirt get ds -o wide

Common entry is:

    virt-handler

To view detailed information:

    kubectl -n kubevirt describe ds virt-handler

### Notes:

    virt-handler is a component that runs at the node level.
    If some nodes lack virt-handler, check the node labels, taints, scheduling policies, and KubeVirt configurations.

---

### 8.4 Viewing Services

Execute the following command:

    kubectl -n kubevirt get svc

A common Service entry is:

    virt-api

---

### 8.5 Viewing KubeVirt CRDs

Execute the following command:

    kubectl get crd | grep kubevirt

Pay special attention to the following CRDs:

    virtualmachines.kubevirt.io
    virtualmachineinstances.kubevirt.io
    virtualmachineinstancemigrations.kubevirt.io
    virtualmachineinstancereplicasets.kubevirt.io
    kubevirts.kubevirt.io

To view API resources:

    kubectl api-resources | grep kubevirt

Alternative commands to check resources:

    kubectl get vm -A
    kubectl get vmi -A

If no virtual machines have been created yet, the output will be empty, which is normal.

---

## Section 9: Installing virtctl

virtctl is a command-line tool for KubeVirt. It is used to perform the following tasks:

    1. Start virtual machines.
    2. Stop virtual machines.
    3. Restart virtual machines.
    4. Access the console of a virtual machine.
    5. Open a VNC session for a virtual machine.
    6. Trigger virtual machine migrations.
    7. Upload images.

While kubectl can manage VM and VMI resources, virtctl is more suitable for performing specific virtual machine operations.

---

### 9.1 Downloading virtctl

On the node k8s-master-01, execute the following commands:

    cd /root/k8s-yaml/kubevirt
    curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/virtctl-${KUBEVIRT_VERSION}-linux-amd64

Check the downloaded file:

    ls -lh virtctl

---

### 9.2 Installing virtctl

Grant execute permissions to the file:

    chmod +x virtctl

Move the file to the system's bin directory:

    sudo mv virtctl /usr/local/bin/virtctl

Verify the installation by running:

    virtctl version

If you encounter issues such as connection failures or incomplete version information, first ensure that all KubeVirt components are installed and functioning correctly.

---

### 9.3 Download Instructions for Domestic Environments

If the download from GitHub is slow, you can manually download the file:

    virtctl-v1.4.0-linux-amd64

Download address:

    https://github.com/kubevirt/kubevirt/releases/download/v1.4.0/virtctl-v1.4.0-linux-amd64

After downloading, grant execute permissions and move the file to the system's bin directory:

    chmod +x virtctl-v1.4.0-linux-amd64
    sudo mv virtctl-v1.4.0-linux-amd64 /usr/local/bin/virtctl

Then verify the installation by running:

    virtctl version

---

## Section 10: Basic Verification After Installation

### 10.1 Checking KubeVirt Status

Execute the following command:

    kubectl -n kubevirt get kv kubevirt

To view detailed information:

    kubectl -n kubevirt describe kv kubevirt

Key points to check for normal operation:

    Phase should be Normal.
    Available should be True.
    Progressing should be False or indicate a stable state.
    Degraded should be False.

---

### 10.2 Checking All KubeVirt Pods

Execute the following command:

    kubectl -n kubevirt get pods -o wide

All core components should be Running.

---

### ### virt-operator Images

**Note:**  
If the production environment cannot access public image repositories, these images must be synchronized to an internal Harbor in advance. It is not recommended to rely on temporary public network access for production environments.

---

## Section 15: Experiment 5: Checking Node KVM Tags and Device Plugins (For Information Only)

After installing KubeVirt, relevant virtualization-related information may be stored on nodes.

**Checking node tags:**  
```bash
kubectl get nodes --show-labels | grep kubevirt
```

**Checking available resources on a node:**  
```bash
kubectl describe node k8s-worker-01 | grep -i kubevirt -A5
```

**Note:**  
Output may vary slightly depending on the KubeVirt version. This section helps determine whether a node is recognized by KubeVirt as suitable for running VMs.

If VM scheduling fails later, focus on checking the following:  
1. Whether the node has `/dev/kvm`.  
2. Whether `virt-handler` is running on that node.  
3. Whether the node is in the “Ready” state.  
4. Whether there are sufficient resources.  
5. Whether the taints and tolerations match correctly.

---

## Section 16: Approaches to Handling Image Issues in Domestic Environments

### 16.1 Common Problems

- Abnormal Pod status after installation: `ImagePullBackOff`, `ErrImagePull`, `ContainerCreating` remaining inactive for a long time.
  - Check: `kubectl -n kubevirt get pods -o wide`
  - For details: `kubectl -n kubevirt describe pod <pod-name>`

**Common causes:**  
1. The node cannot access quay.io.  
2. The node cannot access registry.k8s.io.  
3. The node cannot access GitHub-related addresses.  
4. Corporate network restrictions prevent accessing public image repositories.  
5. `containerd` is not configured with a proxy or an internal image repository.

---

### 16.2 Checking Failed Images

- Execute: `kubectl -n kubevirt describe pod <pod-name>`
- Look for the line “Failed to pull image” and record the failed image address.
- You can also view all images using:  
  ```bash
  kubectl -n kubevirt get pods -o jsonpath='{"range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'
  ```

---

### 16.3 Solutions

- Pull images from a machine with public network access.  
- Re-tag the images and store them in an internal Harbor.  
- Push the images back to the internal Harbor.  
- Modify KubeVirt’s image repository configuration.  
- Reinstall or update the KubeVirt CRD.

**Production recommendations:**  
1. Prepare an image list before installation.  
2. Synchronize all images to the internal Harbor in advance.  
3. Ensure that nodes only pull images from the internal Harbor.  
4. Avoid attempting to resolve public network image issues during production installations.

---

## Section 17: Issues with Namespace Security Policies

If Pod Security Admission is enabled in the cluster, the KubeVirt components may require higher permissions. You can add the `privileged` label to the `kubevirt` namespace:

- **Viewing the namespace:** `kubectl get namespace kubevirt --show-labels`
- **Setting the label:** `kubectl label namespace kubevirt \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite`

**Note:**  
If strict Pod Security Admission is not enabled in the cluster, this step may not be necessary. Production environments must comply with corporate security guidelines. Do not arbitrarily assign the `privileged` label to business namespaces.

---

## Section 18: Troubleshooting KubeVirt Installation Failures

### 18.1 Abnormalities with the Operator Pod

- Check: `kubectl -n kubevirt get pods -o wide`
- `kubectl -n kubevirt describe pod <virt-operator-pod-name>`
- `kubectl -n kubevirt logs deploy/virt-operator --tail=100`

**Common causes:**  
1. Failed image pull.  
2. Abnormal RBAC creation.  
3. Incomplete installation of CRDs.  
4. Restrictions due to namespace security policies.  
5. Insufficient node resources.

---

### 18.2 The KubeVirt CR Being “Not Available” Constantly

- Check: `kubectl -n kubevirt get kv kubevirt`
- `kubectl -n kubevirt describe kv kubevirt`
- Check related components: `kubectl -n kubevirt get pods -o wide`
- Check events: `kubectl### virtctl Version

The following conditions must be met:

1. The `kubevirt` namespace exists.
2. The KubeVirt CR is in the "Available" state.
3. The `virt-operator` is running.
4. The `virt-api` is running.
5. The `virt-controller` is running.
6. The `virt-handler` is running.
7. The VM/VMI CRDs have been installed.
8. Running `kubectl get vm -A` should not produce any errors.
9. Running `kubectl get vmi -A` should not produce any errors.
10. The `virtctl version` command should return a valid result.

---

## Section 21: Quick Troubleshooting

| Issue | Common Causes | Priority Check Items |
|---|---|---|
| `virt-operator ImagePullBackOff` | Failed image pull | `describe pod` |
| KubeVirt CR not available | Not all components started | `describe kv` |
| `virt-api` not running | Issues with images, permissions, or scheduling | `virt-api logs` |
| `virt-controller` not running | Issues with images, permissions, or scheduling | `virt-controller logs` |
| `virt-handler` not running | Failed DaemonSet scheduling | `describe ds` |
| `kubectl get vm` fails | CRDs not installed | `kubectl get crd` |
| `virtctl version` command fails | Issues with `virt-api` or `kubeconfig` | `virt-api/kubeconfig` |
| Pod rejected by security policies | PSA restrictions | `namespace labels` |
| Components in "Pending" state | Node resources or taints | `describe pod` |

---

## Section 22: Standard Debugging Commands

To view the KubeVirt CR:

```bash
kubectl -n kubevirt get kv kubevirt
kubectl -n kubevirt describe kv kubevirt
```

To view Pods:

```bash
kubectl -n kubevirt get pods -o wide
```

To view events:

```bash
kubectl -n kubevirt get events --sort-by=.lastTimestamp
```

To view Deployments:

```bash
kubectl -n kubevirt get deploy
kubectl -n kubevirt describe deploy virt-api
kubectl -n kubevirt describe deploy virt-controller
kubectl -n kubevirt describe deploy virt-operator
```

To view DaemonSets:

```bash
kubectl -n kubevirt get ds
kubectl -n kubevirt describe ds virt-handler
```

To view logs:

```bash
kubectl -n kubevirt logs deploy/virt-api --tail=100
kubectl -n kubevirt logs deploy/virt-controller --tail=100
kubectl -n kubevirt logs deploy/virt-operator --tail=100
kubectl -n kubevirt logs <virt-handler-pod-name> --tail=100
```

To view CRDs:

```bash
kubectl get crd | grep kubevirt
```

To view API resources:

```bash
kubectl api-resources | grep kubevirt
```

To view VMs:

```bash
kubectl get vm -A
kubectl get vmi -A
```

To view `virtctl` version:

```bash
virtctl version
```

---

## Section 23: Recommendations for Production Environments

Before installing KubeVirt in a production environment, it is recommended to do the following:

1. Verify the compatibility between KubeVirt and the Kubernetes version.
2. Fix the KubeVirt version to a specific release, rather than using "latest".
3. Synchronize images to an internal repository in advance.
4. Prepare a dedicated node pool for VMs.
5. Ensure that nodes support KVM.
6. Confirm that the `/dev/kvm` directory exists.
7. Verify that StorageClasses support VM disks.
8. Ensure stable CNI and DNS services.
9. Plan the VM network configuration.
10. Determine how VM images will be imported.
11. Establish procedures for VM backup and recovery.
12. Control the permissions of the `kubevirt` namespace.
13. Record the installed version, image versions, and installation YAML files.
14. Create health check commands for components.
15. Conduct a test environment verification before deploying in production.

---

## Section 24: Summary of This Article

This article describes the basic steps for installing KubeVirt:

1. Fix the KubeVirt version to `v1.4.0`.
2. Download the `kubevirt-operator.yaml` and `kubevirt-cr.yaml` files.
3. Install the KubeVirt Operator.
4. Create the KubeVirt CR.
5. Wait for the KubeVirt status to change to "Available".
6. Verify that `virt-api`, `virt-controller`, `virt-handler`, and `virt-operator` are running correctly.
7. Install `virtctl`.
8. Verify that the