# 04-KubeVirt Installation Practice: Operator, CRD, virtctl, and Component Verification

Recommended path:

    04-Kubernetes/12-KubeVirt/04-KubeVirt Installation Practice: Operator, CRD, virtctl, and Component Verification.md

Tags:

    #Kubernetes
    #KubeVirt
    #Operator
    #CRD
    #virtctl
    #Virtual
    #KVM
    #QEMU
    #Ubuntu2204
    #CloudlandVirtualization
    #PlatformEngineering

---

## I. Document Description

This document records the hands-on process of installing KubeVirt in a Kubernetes cluster.

Key focus of this document:

    1. Selecting a KubeVirt version compatible with the current Kubernetes version
    2. Installing KubeVirt Operator
    3. Installing KubeVirt CR
    4. Verifying KubeVirt core components
    5. Installing virtctl command-line tool
    6. Verifying VM/VMI CRD availability
    7. Troubleshooting common installation issues

Environment for this document:

    Operating System: Ubuntu 22.04
    Kubernetes: v1.31.14
    Container Runtime: containerd
    Installation Method: Official Operator YAML
    KubeVirt Version: v1.4.0
    Execution Node: k8s-master-01

Notes:

    This document only completes KubeVirt installation and component verification.
    Creating the first virtual machine will be organized separately in the next article.

---

## II. Why Fixed Version

KubeVirt has version compatibility with Kubernetes.

Kubernetes version of this document's cluster is:

    v1.31.14

Therefore, this document uses fixed:

    KubeVirt v1.4.0

Reasons:

    KubeVirt v1.4 aligns with Kubernetes v1.31.
    It is not recommended to directly use the latest KubeVirt version on Kubernetes v1.31 clusters.
    The latest KubeVirt typically aligns with updated Kubernetes versions.

Production recommendations:

    1. Confirm Kubernetes version before installation
    2. Verify KubeVirt compatibility matrix
    3. Avoid using latest blindly
    4. Do not use incompatible versions in production environments
    5. Fix the version before doing image synchronization, installation, and acceptance

Official references:

    https://kubevirt.io/user-guide/cluster_admin/installation/

    https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md

---

## III. Pre-Installation Verification

### 3.1 Check Kubernetes Version

Execute:

    kubectl version

Or:

    kubectl get nodes -o wide

Confirm node version:

    v1.31.14

---

### 3.2 Check Node Status

Execute:

    kubectl get nodes -o wide

Requirements:

    All nodes Ready.

If there are NotReady nodes, do not proceed with KubeVirt installation.

---

### 3.3 Check kube-system Components

Execute:

    kubectl -n kube-system get pods -o wide

Focus on confirming:

    CoreDNS Running
    kube-proxy Running
    CNI Running
    kube-apiserver Normal
    kube-controller-manager Normal
    kube-scheduler Normal

---

### 3.4 Check KVM Capabilities

On the Worker node where virtual machines will run, execute:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    ls -l /dev/kvm

    lsmod | grep kvm

    sudo kvm-ok

Minimum requirements:

    1. vmx/svm count greater than 0
    2. /dev/kvm exists
    3. kvm module is loaded
    4. kvm-ok shows "KVM acceleration can be used"

If these requirements are not met, return to the previous article's installation preparation notes first.

---

### 3.5 Check StorageClass

Execute:

    kubectl get storageclass

Recommend at least one available StorageClass, such as:

    longhorn
    nfs-client

KubeVirt installation itself does not strictly require an immediate business PVC, but subsequent virtual machine disk creation will depend on PVC.

---

## IV. Create Installation Directory

On k8s-master-01, execute:

    mkdir -p /root/k8s-yaml/kubevirt

    cd /root/k8s-yaml/kubevirt

Set version variable:

    export KUBEVIRT_VERSION=v1.4.0

Check:

    echo ${KUBEVIRT_VERSION}

---

## V. Download KubeVirt Installation YAML

### 5.1 Download Operator YAML

Execute:

    curl -LO https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml

Check:

    ls -lh kubevirt-operator.yaml

---

### 5.2 Download KubeVirt CR YAML

Execute:

    curl -LO https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-cr.yaml

Check:

    ls -lh kubevirt-cr.yaml

---

### 5.3 Domestic Environment Notes

If GitHub download is slow, you can:

    1. Download locally via browser
    2. Upload to k8s-master-01
    3. Place in /root/k8s-yaml/kubevirt/
    4. Continue with kubectl apply

Files to download:

    kubevirt-operator.yaml
    kubevirt-cr.yaml

Download address format:

    https://github.com/kubevirt/kubevirt/releases/download/v1.4.0/kubevirt-operator.yaml

    https://github.com/kubevirt/kubevirt/releases/download/v1.4.0/kubevirt-cr.yaml

---

## VI. Install KubeVirt Operator

### 6.1 Apply Operator YAML

Execute:

    kubectl apply -f kubevirt-operator.yaml

Check resources:

    kubectl get namespace kubevirt

    kubectl get crd | grep kubevirt

    kubectl -n kubevirt get pods -o wide

Note:

kubevirt-operator.yaml will install KubeVirt-related CRD, RBAC, ServiceAccount, Operator, and other resources.

---

### 6.2 Wait for virt-operator to start

Execute:

    kubectl -n kubevirt get pods -o wide

Expect to see similar output:

    virt-operator-xxxxx   Running

Check Deployment:

    kubectl -n kubevirt get deploy

Check logs:

    kubectl -n kubevirt logs deploy/virt-operator --tail=100

---

## VII. Install KubeVirt CR

### 7.1 Apply KubeVirt CR

Execute:

    kubectl apply -f kubevirt-cr.yaml

Check KubeVirt CR:

    kubectl -n kubevirt get kubevirt

Or:

    kubectl -n kubevirt get kv

Expect to see:

    NAME       AGE   PHASE
    kubevirt   ...   Deploying

---

### 7.2 Wait for KubeVirt Available

Execute:

    kubectl -n kubevirt wait kv kubevirt \
      --for condition=Available \
      --timeout=10m

If successful, you'll see similar output:

    kubevirt.kubevirt.io/kubevirt condition met

Check:

    kubectl -n kubevirt get kv kubevirt

Check detailed status:

    kubectl -n kubevirt describe kv kubevirt

---

## VIII. Verify KubeVirt Components

### 8.1 Check Pods in kubevirt Namespace

Execute:

    kubectl -n kubevirt get pods -o wide

Common components:

    virt-api
    virt-controller
    virt-handler
    virt-operator

Expect:

    virt-api Running
    virt-controller Running
    virt-handler Running
    virt-operator Running

Note:

    virt-api is typically a Deployment
    virt-controller is typically a Deployment
    virt-handler is typically a DaemonSet
    virt-operator is typically a Deployment

---

### 8.2 Check Deployments

Execute:

    kubectl -n kubevirt get deploy -o wide

Common:

    virt-api
    virt-controller
    virt-operator

Check details:

    kubectl -n kubevirt describe deploy virt-api

    kubectl -n kubevirt describe deploy virt-controller

    kubectl -n kubevirt describe deploy virt-operator

---

### 8.3 Check DaemonSet

Execute:

    kubectl -n kubevirt get ds -o wide

Common:

    virt-handler

Check details:

    kubectl -n kubevirt describe ds virt-handler

Note:

    virt-handler is a node-side component.
    If some nodes lack virt-handler, check node labels, taints, scheduling policies, and KubeVirt configuration.

---

### 8.4 Check Services

Execute:

    kubectl -n kubevirt get svc

Common Service:

    virt-api

---

### 8.5 Check KubeVirt CRD

Execute:

    kubectl get crd | grep kubevirt

Focus on:

    virtualmachines.kubevirt.io
    virtualmachineinstances.kubevirt.io
    virtualmachineinstancemigrations.kubevirt.io
    virtualmachineinstancereplicasets.kubevirt.io
    kubevirts.kubevirt.io

Check API resources:

    kubectl api-resources | grep kubevirt

Verify shorthand:

    kubectl get vm -A

    kubectl get vmi -A

It's normal to see empty output if no virtual machines have been created yet.

---

## IX. Install virtctl

virtctl is the command-line tool for KubeVirt.

It is used for:

    1. Starting virtual machines
    2. Stopping virtual machines
    3. Restarting virtual machines
    4. Entering console
    5. Opening VNC
    6. Triggering virtual machine migration
    7. Uploading images

kubectl can manage VM / VMI resources.

virtctl is more suitable for virtual machine operations.

---

### 9.1 Download virtctl

Execute on k8s-master-01:

    cd /root/k8s-yaml/kubevirt

    curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/virtctl-${KUBEVIRT_VERSION}-linux-amd64

Check:

    ls -lh virtctl

---

### 9.2 Install virtctl

Add execute permissions:

    chmod +x virtctl

Move to system path:

    sudo mv virtctl /usr/local/bin/virtctl

Verify:

    virtctl version

If you get connection errors or incomplete version information, first confirm that KubeVirt components are functioning properly.

---

### 9.3 Download Instructions for Domestic Environments

If GitHub downloads are slow, you can manually download:

    virtctl-v1.4.0-linux-amd64

Download address:

    https://github.com/kubevirt/kubevirt/releases/download/v1.4.0/virtctl-v1.4.0-linux-amd64

After uploading, execute:

chmod +x virtctl-v1.4.0-linux-amd64

sudo mv virtctl-v1.4.0-linux-amd64 /usr/local/bin/virtctl

virtctl version

---

## 10. Post-Installation Basic Verification

### 10.1 Check KubeVirt Status

Execute:

    kubectl -n kubevirt get kv kubevirt

Check details:

    kubectl -n kubevirt describe kv kubevirt

Normal focus points:

    Phase: Normal
    Available=True
    Progressing=False or stable state
    Degraded=False

---

### 10.2 Check All KubeVirt Pods

Execute:

    kubectl -n kubevirt get pods -o wide

Requirements:

    All core components Running.

---

### 10.3 Check Events

Execute:

    kubectl -n kubevirt get events --sort-by=.lastTimestamp

If there are anomalies, focus on:

    FailedScheduling
    FailedMount
    FailedCreate
    ImagePullBackOff
    CrashLoopBackOff
    Error

---

### 10.4 Check VM / VMI Resources

Execute:

    kubectl get vm -A

    kubectl get vmi -A

There are currently no VMs created, so being empty is normal.

If it prompts that resources do not exist, it indicates that CRD is not properly installed.

---

### 10.5 Check virtctl

Execute:

    virtctl version

Normally, it should output client/server version information.

---

## 11. Experiment One: Confirm KubeVirt Control Component Relationships

Experiment goal:

    Understand which components are Deployments and which are DaemonSets after KubeVirt installation.

Execute:

    kubectl -n kubevirt get deploy

    kubectl -n kubevirt get ds

Record:

    What resource is virt-api?
    What resource is virt-controller?
    What resource is virt-operator?
    What resource is virt-handler?

Expected understanding:

    virt-api, virt-controller, virt-operator are Deployments.
    virt-handler is a DaemonSet, responsible for node-side virtual machine management.

---

## 12. Experiment Two: Confirm KubeVirt API Resources

Experiment goal:

    Confirm that Kubernetes API Server has recognized KubeVirt resources.

Execute:

    kubectl api-resources | grep kubevirt

    kubectl get crd | grep kubevirt

    kubectl get vm -A

    kubectl get vmi -A

Expected results:

    Can recognize vm and vmi.
    When there are no VMs, the list is empty, but no error is reported.

---

## 13. Experiment Three: Confirm virt-handler Distribution

Experiment goal:

    Confirm which nodes are running virt-handler.

Execute:

    kubectl -n kubevirt get pods -o wide | grep virt-handler

Record:

    virt-handler Pod name
    Node it is on
    Status

If the plan is to only run VMs on Worker nodes, focus on confirming that Worker nodes have virt-handler.

If some nodes do not have virt-handler, check the DaemonSet:

    kubectl -n kubevirt describe ds virt-handler

Focus on:

    Node-Selector
    Tolerations
    Events

---

## 14. Experiment Four: Check KubeVirt Image Sources

Experiment goal:

    Confirm which images KubeVirt components use, to facilitate synchronization to internal Harbor in domestic environments.

Check Deployment images:

    kubectl -n kubevirt get deploy -o yaml | grep "image:"

Check DaemonSet images:

    kubectl -n kubevirt get ds -o yaml | grep "image:"

Check all Pod images:

    kubectl -n kubevirt get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'

Record images:

    virt-api image
    virt-controller image
    virt-handler image
    virt-operator image

Notes:

    If the production environment cannot access public image repositories, these images need to be synchronized to internal Harbor in advance.
    It is not recommended to rely on temporary public image pulls in production.

---

## 15. Experiment Five: View Node KVM Labels and Device Plugins, Just for Understanding

After KubeVirt installation, it may maintain information related to virtualization capabilities on nodes.

Check node labels:

    kubectl get nodes --show-labels | grep kubevirt

Check node allocatable resources:

    kubectl describe node k8s-worker-01 | grep -i kubevirt -A5

Notes:

    Output varies slightly by KubeVirt version.
    This section is for understanding whether nodes are recognized as VM-capable by KubeVirt.

If VMs cannot be scheduled later, focus on:

    1. Whether /dev/kvm exists on the node
    2. Whether virt-handler is running on the node
    3. Whether the node is Ready
    4. Whether resources are sufficient
    5. Whether taint/tolerations match

---

## 16. Handling Mirror Image Issues in Domestic Environments

### 16.1 Common Phenomena

After installation, Pod status anomalies: /think

ImagePullBackOff
ErrImagePull
ContainerCreating is stuck for a long time

Check:

    kubectl -n kubevirt get pods -o wide

Check details:

    kubectl -n kubevirt describe pod <pod-name>

Common causes:

    1. Nodes cannot access quay.io
    2. Nodes cannot access registry.k8s.io
    3. Nodes cannot access GitHub-related addresses
    4. Corporate network restrictions on pulling public images
    5. containerd is not configured with a proxy or internal image registry

---

### 16.2 Viewing Failed Images

Execute:

    kubectl -n kubevirt describe pod <pod-name>

Find:

    Failed to pull image

Record the failed image address.

You can also view all images:

    kubectl -n kubevirt get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'

---

### 16.3 Handling Methods

Common handling methods:

    1. Pull the image on a machine with internet access
    2. Retag to internal Harbor
    3. Push to internal Harbor
    4. Modify KubeVirt image registry configuration
    5. Reinstall or update KubeVirt CR

Production recommendations:

    1. Prepare an image list before installation
    2. Synchronize all images to internal Harbor in advance
    3. Nodes should only pull from internal Harbor
    4. Do not temporarily resolve public image issues during production installation

---

## SeventeenI don't know.Namespace Security Policy Issues

If the cluster has enabled Pod Security Admission, KubeVirt components may require higher privileges.

You can add the privileged label to the kubevirt namespace.

Check namespace:

    kubectl get namespace kubevirt --show-labels

Set:

    kubectl label namespace kubevirt \
      pod-security.kubernetes.io/enforce=privileged \
      pod-security.kubernetes.io/audit=privileged \
      pod-security.kubernetes.io/warn=privileged \
      --overwrite

Check again:

    kubectl get namespace kubevirt --show-labels

Note:

    If the cluster has not enabled strict Pod Security Admission, this operation may not be needed.
    Production environments must comply with company security baselines.
    Do not arbitrarily set privileged for business namespaces.

---

## EighteenI don't know.KubeVirt Installation Failure Troubleshooting

### 18.1 Operator Pod is Not Normal

Check:

    kubectl -n kubevirt get pods -o wide

    kubectl -n kubevirt describe pod <virt-operator-pod-name>

    kubectl -n kubevirt logs deploy/virt-operator --tail=100

Common causes:

    1. Image pull failure
    2. RBAC creation anomalies
    3. CRD not installed completely
    4. Namespace security policy restrictions
    5. Node resource insufficiency

---

### 18.2 KubeVirt CR is Never Available

Check:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt describe kv kubevirt

Check components:

    kubectl -n kubevirt get pods -o wide

Check events:

    kubectl -n kubevirt get events --sort-by=.lastTimestamp

Common causes:

    1. virt-api not started
    2. virt-controller not started
    3. virt-handler not started
    4. Image pull failure
    5. Node scheduling failure
    6. Security policy restrictions
    7. Webhook or APIService anomalies

---

### 18.3 virt-handler is Not Running Normally

Check DaemonSet:

    kubectl -n kubevirt get ds virt-handler

    kubectl -n kubevirt describe ds virt-handler

Check Pod:

    kubectl -n kubevirt get pods -o wide | grep virt-handler

Check logs:

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=100

Common causes:

    1. Node does not meet scheduling conditions
    2. Node taints are not tolerated
    3. Image pull failure
    4. Node lacks /dev/kvm
    5. Security policy restrictions
    6. kubelet or containerd anomalies

---

### 18.4 virtctl version Fails

Check:

    which virtctl

    virtctl version

    kubectl -n kubevirt get svc

    kubectl -n kubevirt get pods | grep virt-api

    kubectl -n kubevirt logs deploy/virt-api --tail=100

Common causes:

    1. virtctl not installed in PATH
    2. virtctl version mismatch with KubeVirt
    3. virt-api not running
    4. kubeconfig is incorrect
    5. Current user lacks access permissions to KubeVirt API

---

## NineteenI don't know.Uninstalling KubeVirt

Note: /think

Uninstall KubeVirt before confirming there are no virtual machine resources.  
Do not directly execute uninstallation in production environments.  
Before uninstalling, stop and delete test VM, VMI, DataVolume, PVC, and other resources first.

Check for existing VMs:

    kubectl get vm -A

    kubectl get vmi -A

If it's just an experimental environment, you can proceed with the following uninstallation steps in order.

---

### 19.1 Delete KubeVirt CR

Execute:

    kubectl delete -f kubevirt-cr.yaml

Wait for resource cleanup:

    kubectl -n kubevirt get kv

---

### 19.2 Delete KubeVirt Operator

Execute:

    kubectl delete -f kubevirt-operator.yaml

Check:

    kubectl get namespace kubevirt

    kubectl get crd | grep kubevirt

Note:

    Whether to delete CRD depends on the operator YAML content and actual residual state.  
    If there are still business VMs, do not delete CRD.

---

### 19.3 Delete virtctl (Optional)

Execute:

    sudo rm -f /usr/local/bin/virtctl

Check:

    which virtctl

---

## Twenty, Installation Completion Checklist

After installation, execute:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

    kubectl -n kubevirt get deploy

    kubectl -n kubevirt get ds

    kubectl get crd | grep kubevirt

    kubectl api-resources | grep kubevirt

    kubectl get vm -A

    kubectl get vmi -A

    virtctl version

Requirements:

    1. kubevirt namespace exists  
    2. KubeVirt CR is Available  
    3. virt-operator Running  
    4. virt-api Running  
    5. virt-controller Running  
    6. virt-handler Running  
    7. VM / VMI CRD is installed  
    8. kubectl get vm -A does not report errors  
    9. kubectl get vmi -A does not report errors  
    10. virtctl version outputs normally

---

## Twenty-one, Common Issues Quick Reference

| Phenomenon | Common Causes | Priority Checks |
|---|---|---|
| virt-operator ImagePullBackOff | Image pull failure | describe pod |
| KubeVirt CR not Available | Components not fully started | describe kv |
| virt-api not Running | Image, permissions, scheduling issues | virt-api logs |
| virt-controller not Running | Image, permissions, scheduling issues | virt-controller logs |
| virt-handler not Running | DaemonSet scheduling failure | describe ds |
| kubectl get vm reports errors | CRD not installed | kubectl get crd |
| virtctl version fails | virt-api or kubeconfig issues | virt-api / kubeconfig |
| Pod rejected by security policy | PSA restrictions | namespace labels |
| Components Pending | Node resources or taint | describe pod |

---

## Twenty-two, Standard Troubleshooting Commands

Check KubeVirt CR:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt describe kv kubevirt

Check Pods:

    kubectl -n kubevirt get pods -o wide

Check Events:

    kubectl -n kubevirt get events --sort-by=.lastTimestamp

Check Deployments:

    kubectl -n kubevirt get deploy

    kubectl -n kubevirt describe deploy virt-api

    kubectl -n kubevirt describe deploy virt-controller

    kubectl -n kubevirt describe deploy virt-operator

Check DaemonSet:

    kubectl -n kubevirt get ds

    kubectl -n kubevirt describe ds virt-handler

Check Logs:

    kubectl -n kubevirt logs deploy/virt-api --tail=100

    kubectl -n kubevirt logs deploy/virt-controller --tail=100

    kubectl -n kubevirt logs deploy/virt-operator --tail=100

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=100

Check CRD:

    kubectl get crd | grep kubevirt

Check API Resources:

    kubectl api-resources | grep kubevirt

Check VM:

    kubectl get vm -A

    kubectl get vmi -A

Check virtctl:

    virtctl version

---

## Twenty-three, Production Environment Recommendations

Before and after installing KubeVirt in production environments, it is recommended to do the following:

1. Clarify the compatibility between KubeVirt and Kubernetes versions  
2. Fix KubeVirt version, do not use latest  
3. Synchronize images to internal Harbor in advance  
4. Prepare dedicated node pool for VMs  
5. Confirm nodes support KVM  
6. Confirm /dev/kvm exists  
7. Confirm StorageClass supports VM disks  
8. Confirm CNI and DNS stability  
9. Plan VM network  
10. Plan VM image import method  
11. Plan VM backup and recovery  
12. Control kubevirt namespace permissions  
13. Record installation version, image version, and installation YAML  
14. Establish component health check commands  
15. Perform test environment verification before production  

---

## 24. Summary of This Article  

This article completes the basic installation of KubeVirt.  

Core steps:  

    1. Fix KubeVirt version v1.4.0  
    2. Download kubevirt-operator.yaml  
    3. Download kubevirt-cr.yaml  
    4. Install KubeVirt Operator  
    5. Create KubeVirt CR  
    6. Wait for KubeVirt Available  
    7. Verify virt-api, virt-controller, virt-handler, virt-operator  
    8. Install virtctl  
    9. Verify VM / VMI CRD  
    10. Check after installation completion  

Core commands:  

    export KUBEVIRT_VERSION=v1.4.0  

    kubectl apply -f kubevirt-operator.yaml  

    kubectl apply -f kubevirt-cr.yaml  

    kubectl -n kubevirt wait kv kubevirt --for condition=Available --timeout=10m  

    kubectl -n kubevirt get pods -o wide  

    virtctl version  

Experience judgment:  

    1. A successful KubeVirt installation does not guarantee that the virtual machine will start  
    2. Next, create a VM to verify KVM, PVC, network, and console  
    3. If KubeVirt components are abnormal, first check Pods and Events in the kubevirt namespace  
    4. If virt-handler is abnormal, focus on nodes, taint, KVM, and security policies  
    5. If image pull fails in domestic environment, prioritize synchronizing images to internal Harbor  
    6. Do not recommend directly using new KubeVirt versions on Kubernetes v1.31 clusters  

Next suggested topic to learn:  

    05-Create the First Virtual Machine: VirtualMachine, VMI, console, and Basic Operations.md