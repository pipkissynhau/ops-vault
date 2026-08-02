# 06-CDI and DataVolume: Virtual Machine Image Import, PVC, and Boot Disk Management

Recommended path:

    04-Kubernetes/12-KubeVirt/06-CDI and DataVolume: Virtual Machine Image Import, PVC, and Boot Disk Management.md

Tags:

    #Kubernetes
    #KubeVirt
    #CDI
    #DataVolume
    #PVC
    #StorageClass
    #VirtualMirror
    #MirrorImport
    #CloudlandVirtualization
    #PlatformEngineering

---

## I. Document Overview

This document records the basic usage of CDI and DataVolume in KubeVirt.

The previous article already created the first test virtual machine using containerDisk.

containerDisk is suitable for:

    1. Quick experience
    2. Demo testing
    3. Temporary verification of KubeVirt's ability to run VM

However, in production environments, the virtual machine system disk is typically not recommended to directly rely on containerDisk.

A more common approach is:

    1. Prepare the virtual machine image
    2. Import the image using CDI
    3. CDI creates DataVolume
    4. DataVolume generates PVC
    5. VM uses this PVC as the boot disk

This article's objectives:

    1. Understand what CDI is
    2. Understand what DataVolume is
    3. Install CDI
    4. Import CirrOS image using HTTP
    5. Observe the relationship between DataVolume, PVC, and Importer Pod
    6. Use DataVolume as the VM boot disk
    7. Start VM and enter console
    8. Master common troubleshooting methods for CDI/DataVolume

---

## II. What is CDI

CDI full name:

    Containerized Data Importer

It is a component in the KubeVirt ecosystem for managing virtual machine disk data import.

CDI mainly solves:

    How to get virtual machine images into Kubernetes PVC

Without CDI, users need to handle:

    1. Download qcow2/raw image
    2. Create PVC
    3. Find a way to write the image into PVC
    4. Confirm format and permissions
    5. Then let VM use this PVC

With CDI, these actions can be completed through declarative DataVolume resources.

Simple understanding:

    CDI = Component that prepares virtual machine disk data for KubeVirt

---

## III. What is DataVolume

DataVolume is a CRD resource provided by CDI.

It represents:

    A disk data volume for a virtual machine

DataVolume automatically completes:

    1. Create PVC
    2. Create importer Pod
    3. Download image from specified source
    4. Write image into PVC
    5. Update import status
    6. Allow VM to use this PVC as a disk

Common sources:

    HTTP
    Registry
    PVC Clone
    Upload
    Blank

This article first uses the easiest-to-understand HTTP import method.

---

## IV. Relationship between CDI, DataVolume, PVC, and VM

Relationship:

    Image file
      |
      v
    DataVolume
      |
      v
    CDI Importer Pod
      |
      v
    PVC
      |
      v
    VirtualMachine
      |
      v
    VMI
      |
      v
    virt-launcher Pod
      |
      v
    Guest OS

Simple understanding:

    DataVolume is responsible for preparing the disk.
    PVC is the actual Kubernetes storage object that saves disk data.
    VM uses PVC as the boot disk.

---

## V. Difference from containerDisk

| Comparison item | containerDisk | DataVolume / PVC |
|---|---|---|
| Purpose | Quick experience | Closer to real VM disk |
| Data persistence | Not suitable for long-term persistence | PVC persistence |
| Image source | Container image | HTTP / Registry / Upload / Clone |
| Production suitability | Not recommended as production system disk | More suitable for production base mode |
| Boot disk change | Follows the image | Falls to PVC |
| Data retention | Not suitable for retaining state after VM deletion | PVC can retain |

Conclusion:

    containerDisk is suitable for beginners.
    DataVolume + PVC is closer to real usage.

---

## VI. Experiment Objectives

After completing this article's experiment, you should be able to:

    1. Install CDI
    2. Check CDI component status
    3. Create DataVolume
    4. Observe importer Pod
    5. Check PVC Bound status
    6. Create VM using DataVolume
    7. Start VM
    8. Enter console
    9. After stopping VM, confirm PVC remains
    10. Troubleshoot issues like image import failure, PVC Pending, and VM startup failure

---

## VII. Experimental Environment

This article assumes the following environment:

    Operating system: Ubuntu 22.04
    Kubernetes: v1.31.14
    KubeVirt: v1.4.0
    Container runtime: containerd
    StorageClass: longhorn
    Namespace: kubevirt-demo
    Test image: CirrOS
    Execution node: k8s-master-01

Notes:

    If your StorageClass is not longhorn, you need to replace the storageClassName in the text.
    If using NFS, you can also perform the experiment, but virtual machine system disk is more recommended to use block storage backend first, such as Longhorn, Ceph RBD, cloud disk CSI, etc.

---

## VIII. Pre-Installation Checks

### 8.1 Check KubeVirt

Execute:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

Requirements: /think

KubeVirt Available  
virt-api Running  
virt-controller Running  
virt-handler Running  
virt-operator Running  

---

### 8.2 Check virtctl  

Run:  

    virtctl version  

---

### 8.3 Check StorageClass  

Run:  

    kubectl get storageclass  

Confirm the existence of:  

    longhorn  

or other available StorageClass.  

View details:  

    kubectl describe storageclass longhorn  

---

### 8.4 Check PVC Creation  

If you have already completed the PVC test in the KubeVirt installation preparation, you can skip this step.  

Quick test:  

    kubectl create namespace kubevirt-demo --dry-run=client -o yaml | kubectl apply -f -  

    cat <<EOF > pvc-cdi-precheck.yaml  
    apiVersion: v1  
    kind: PersistentVolumeClaim  
    metadata:  
      name: cdi-precheck-pvc  
      namespace: kubevirt-demo  
    spec:  
      accessModes:  
      - ReadWriteOnce  
      resources:  
        requests:  
          storage: 1Gi  
      storageClassName: longhorn  
    EOF  

Apply:  

    kubectl apply -f pvc-cdi-precheck.yaml  

Check:  

    kubectl -n kubevirt-demo get pvc cdi-precheck-pvc  

Expected:  

    STATUS is Bound  

Cleanup:  

    kubectl -n kubevirt-demo delete pvc cdi-precheck-pvc  

If the PVC cannot be Bound, do not proceed with CDI installation and usage.  

---

## IX. Install CDI  

### 9.1 Create Installation Directory  

Run on k8s-master-01:  

    mkdir -p /root/k8s-yaml/kubevirt/cdi  

    cd /root/k8s-yaml/kubevirt/cdi  

---

### 9.2 Set CDI Version  

In experimental environments, you can use GitHub releases latest to automatically get the version:  

    export CDI_VERSION=$(curl -Ls https://github.com/kubevirt/containerized-data-importer/releases/latest | grep -m 1 -o "v[0-9]\\.[0-9]*\\.[0-9]*")  

    echo ${CDI_VERSION}  

In production environments, it is recommended to:  

    1. Do not use latest  
    2. Fix CDI version  
    3. Validate KubeVirt + CDI + Kubernetes compatibility in test environments  
    4. Record the installed version  
    5. Synchronize images to internal Harbor in advance  

If you cannot access GitHub, you can manually specify the version, for example:  

    export CDI_VERSION=v1.65.0  

Note:  

    The version number should be based on actual verification.  
    Do not directly copy unverified versions to production environments.  

---

### 9.3 Download CDI Operator YAML  

Run:  

    curl -LO https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-operator.yaml  

Check:  

    ls -lh cdi-operator.yaml  

---

### 9.4 Download CDI CR YAML  

Run:  

    curl -LO https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-cr.yaml  

Check:  

    ls -lh cdi-cr.yaml  

---

### 9.5 Domestic Environment Notes  

If GitHub downloads are slow, you can:  

    1. Download cdi-operator.yaml via browser  
    2. Download cdi-cr.yaml via browser  
    3. Upload to /root/k8s-yaml/kubevirt/cdi  
    4. Then execute kubectl apply  

Download address format:  

    https://github.com/kubevirt/containerized-data-importer/releases/download/<CDI_VERSION>/cdi-operator.yaml  

    https://github.com/kubevirt/containerized-data-importer/releases/download/<CDI_VERSION>/cdi-cr.yaml  

---

## X. Deploy CDI Operator  

Run:  

    kubectl apply -f cdi-operator.yaml  

Check namespace:  

    kubectl get ns cdi  

Check Pod:  

    kubectl -n cdi get pods -o wide  

Expected to see:  

    cdi-operator Running  

Check logs:  

    kubectl -n cdi logs deploy/cdi-operator --tail=100  

---

## XI. Create CDI CR  

Run:  

    kubectl apply -f cdi-cr.yaml  

Check CDI resources:  

    kubectl -n cdi get cdi  

Wait for CDI deployment completion:  

    kubectl -n cdi wait cdi cdi --for condition=Available --timeout=10m  

If wait is not supported or version differences cause failure, you can continuously observe:  

    kubectl -n cdi get cdi  

    kubectl -n cdi get pods -o wide  

Common components:  

    cdi-apiserver  
    cdi-deployment  
    cdi-operator  
    cdi-uploadproxy  
    cdi-uploadserver  
    cdi-controller  

Component names may vary slightly by CDI version; refer to actual output.  

---

## XII. Verify CDI Installation  

### 12.1 Check CDI Pod  

Run:  

    kubectl -n cdi get pods -o wide  

Requirement:  

    Key Pod Running  

---

### 12.2 Check CDI CRD  

Run:  

    kubectl get crd | grep cdi  

Focus on:  

    datavolumes.cdi.kubevirt.io  
    cdiconfigs.cdi.kubevirt.io  
    cdis.cdi.kubevirt.io  
    storageprofiles.cdi.kubevirt.io  

---

### 12.3 Check API Resources  

Run:

kubectl api-resources | grep cdi

Verify DataVolume:

    kubectl get dv -A

If no DataVolume exists, an empty output is normal.

If you receive a "resource not found" error, it indicates that the CDI CRD was not installed successfully.

---

### 12.4 View StorageProfile

CDI generates StorageProfile based on StorageClass.

Execute:

    kubectl get storageprofile

Check longhorn:

    kubectl describe storageprofile longhorn

Notes:

    StorageProfile records CDI-related capabilities for a specific StorageClass.
    If DataVolume import fails, sometimes you need to check StorageProfile.

---

## Thirteen. Create Test Namespace

If kubevirt-demo already exists, you can skip this step.

Execute:

    kubectl create namespace kubevirt-demo

Or:

    kubectl create namespace kubevirt-demo --dry-run=client -o yaml | kubectl apply -f -

Check:

    kubectl get ns kubevirt-demo

---

## Fourteen. Experiment One: Import CirrOS Image to DataVolume via HTTP

### 14.1 Create DataVolume

Create directory:

    mkdir -p /root/k8s-yaml/kubevirt/cdi-demo

    cd /root/k8s-yaml/kubevirt/cdi-demo

Create DataVolume YAML:

    cat <<EOF > dv-cirros-http.yaml
    apiVersion: cdi.kubevirt.io/v1beta1
    kind: DataVolume
    metadata:
      name: cirros-dv
      namespace: kubevirt-demo
    spec:
      source:
        http:
          url: "https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
        storageClassName: longhorn
    EOF

Notes:

    source.http.url
        Indicates downloading the VM image from an HTTP address.

    pvc.storageClassName
        Specifies which StorageClass to use for creating PVC.

    accessModes: ReadWriteOnce
        Common access mode for VM system disks.

    storage: 1Gi
        CirrOS image is small, 1Gi is sufficient for experimentation.

If it's not longhorn, replace:

    storageClassName: <your StorageClass>

---

### 14.2 Apply DataVolume

Execute:

    kubectl apply -f dv-cirros-http.yaml

Check DataVolume:

    kubectl -n kubevirt-demo get dv

Check PVC:

    kubectl -n kubevirt-demo get pvc

Check Pod:

    kubectl -n kubevirt-demo get pods -o wide

You'll typically see an importer Pod.

Example:

    importer-cirros-dv-xxxxx

---

### 14.3 Observe Import Process

Check DataVolume:

    kubectl -n kubevirt-demo get dv cirros-dv

Check details:

    kubectl -n kubevirt-demo describe dv cirros-dv

Check PVC:

    kubectl -n kubevirt-demo get pvc cirros-dv

Check importer Pod:

    kubectl -n kubevirt-demo get pods | grep importer

Check importer logs:

    kubectl -n kubevirt-demo logs <importer-pod-name> --tail=100

Wait until import completes:

    kubectl -n kubevirt-demo get dv

Expected status:

    Succeeded

PVC status:

    Bound

Notes:

    After successful DataVolume import, PVC remains bound.
    The importer Pod may exit or be cleaned up depending on version and configuration.

---

## Fifteen. Experiment Two: Create VM Using DataVolume

### 15.1 Create VM YAML

Create: /think

```yaml
cat <<EOF > vm-cirros-dv.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-cirros-dv
  namespace: kubevirt-demo
  labels:
    app: vm-cirros-dv
spec:
  runStrategy: Manual
  template:
    metadata:
      labels:
        app: vm-cirros-dv
        kubevirt.io/domain: vm-cirros-dv
    spec:
      terminationGracePeriodSeconds: 0
      domain:
        resources:
          requests:
            memory: 512Mi
        devices:
          disks:
          - name: rootdisk
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
      - name: rootdisk
        dataVolume:
          name: cirros-dv
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

`volumes.rootdisk.dataVolume.name: cirros-dv`  
Indicates the VM uses the previously imported DataVolume as the system disk.

`cloudInitNoCloud`  
Used to set the login password and enable SSH password login.

`memory: 512Mi`  
Allocates 512Mi of memory to the VM.

---

### 15.2 Creating VM

Execute:

```bash
kubectl apply -f vm-cirros-dv.yaml
```

Check:

```bash
kubectl -n kubevirt-demo get vm
```

The VM should be *Stopped*, since runStrategy is Manual.

Check VMI:

```bash
kubectl -n kubevirt-demo get vmi
```

Typically, it will be empty.

---

### 15.3 Starting VM

Execute:

```bash
virtctl start vm-cirros-dv -n kubevirt-demo
```

Check:

```bash
kubectl -n kubevirt-demo get vm
kubectl -n kubevirt-demo get vmi
kubectl -n kubevirt-demo get pods -o wide
```

Expected results:

- VM Running  
- VMI Running  
- virt-launcher Pod Running

---

### 15.4 Entering Console

Execute:

```bash
virtctl console vm-cirros-dv -n kubevirt-demo
```

Login:

- Username: cirros  
- Password: kubevirt

After logging in, execute:

```bash
hostname
ip addr
df -h
free -m
```

Exit:

`Ctrl + ]`

---

## SixteenI don't know.Experiment Three: Verifying PVC Retention After Stopping VM

Stop VM:

```bash
virtctl stop vm-cirros-dv -n kubevirt-demo
```

Check:

```bash
kubectl -n kubevirt-demo get vm
kubectl -n kubevirt-demo get vmi
kubectl -n kubevirt-demo get pods
```

Check PVC:

```bash
kubectl -n kubevirt-demo get pvc
```

Expected results:

- VM still exists  
- VMI disappears  
- virt-launcher Pod disappears  
- cirros-dv PVC still exists

**Explanation:**  
This demonstrates the difference between DataVolume/PVC as a virtual machine system disk and containerDisk.  
After stopping the VM, the PVC still retains disk data.

---

## SeventeenI don't know.Experiment Four: Restarting VM

Execute:

```bash
virtctl start vm-cirros-dv -n kubevirt-demo
```

Check:

```bash
kubectl -n kubevirt-demo get vm
kubectl -n kubevirt-demo get vmi
```

Enter console:

```bash
virtctl console vm-cirros-dv -n kubevirt-demo
```

If you previously wrote files to the system disk, they should still exist after restarting.

You can test this inside the VM:

```bash
echo "hello from datavolume disk" > /home/cirros/dv-test.txt
```

Restart VM:

```bash
sudo reboot
```

Or exit and execute: `/think`

virtctl restart vm-cirros-dv -n kubevirt-demo

After logging in again, check:

    cat /home/cirros/dv-test.txt

Explanation:

    If the data is preserved, it indicates that the PVC system disk persistence is working correctly.

---

## Eighteen. Experiment Five: Using dataVolumeTemplates to Automatically Create DataVolume

Previous method:

    First create DataVolume
    Then create VM referencing DataVolume

KubeVirt can also use dataVolumeTemplates within VM.

This approach is suitable for:

    Automatically creating corresponding DataVolume and PVC when creating VM

---

### 18.1 Creating VM + DataVolume Template

Create:

    cat <<EOF > vm-cirros-dv-template.yaml
    apiVersion: kubevirt.io/v1
    kind: VirtualMachine
    metadata:
      name: vm-cirros-dv-template
      namespace: kubevirt-demo
      labels:
        app: vm-cir
    spec:
      runStrategy: Manual
      dataVolumeTemplates:
      - metadata:
          name: cirros-template-dv
        spec:
          source:
            http:
              url: "https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
          pvc:
            accessModes:
            - ReadWriteOnce
            resources:
              requests:
                storage: 1Gi
            storageClassName: longhorn
      template:
        metadata:
          labels:
            app: vm-cirros-dv-template
            kubevirt.io/domain: vm-cirros-dv-template
        spec:
          terminationGracePeriodSeconds: 0
          domain:
            resources:
              requests:
                memory: 512Mi
            devices:
              disks:
              - name: rootdisk
                disk:
                  bus: virtio
              - name: cloudinitdisk
                disk:
                  bus: virtio
              interfaces:
              - name: default
                masquerade: {}
          networks:
          - name: default
            pod: {}
          volumes:
          - name: rootdisk
            dataVolume:
              name: cirros-template-dv
          - name: cloudinitdisk
            cloudInitNoCloud:
              userData: |
                #cloud-config
                password: kubevirt
                chpasswd:
                  expire: false
                ssh_pwauth: true
    EOF

Apply:

    kubectl apply -f vm-cirros-dv-template.yaml

Check:

    kubectl -n kubevirt-demo get vm

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo get pvc

Start:

    virtctl start vm-cirros-dv-template -n kubevirt-demo

Observe:

    kubectl -n kubevirt-demo get dv,pvc,vm,vmi,pod

---

### 18.2 Comparison of Two Methods

| Method | Characteristics |
|---|---|
| Separate DataVolume creation | Clear process, suitable for learning and troubleshooting |
| dataVolumeTemplates | Automatically creates disk when VM is created, suitable for templating |

Getting Started Recommendation:

    First master separate DataVolume.
    Then use dataVolumeTemplates.

---

## Nineteen. Optional Experiment: Exposing SSH via Service

Create Service:

    cat <<EOF > svc-vm-cirros-dv-ssh.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: vm-cirros-dv-ssh
      namespace: kubevirt-demo
    spec:
      type: NodePort
      selector:
        kubevirt.io/domain: vm-cirros-dv
      ports:
      - name: ssh
        protocol: TCP
        port: 22
        targetPort: 22
        nodePort: 30023
    EOF

Apply:

    kubectl apply -f svc-vm-cirros-dv-ssh.yaml

Check:

    kubectl -n kubevirt-demo get svc

kubectl -n kubevirt-demo get endpoints vm-cirros-dv-ssh

Access from outside:

    ssh cirros@10.0.0.23 -p 30023

Password:

    kubevirt

If SSH fails, first check the VM console for sshd status:

    virtctl console vm-cirros-dv -n kubevirt-demo

Check inside the VM:

    ps aux | grep ssh

    netstat -lntp

Note:

    The CirrOS image is small and has limited functionality.
    SSH behavior may differ when using Ubuntu / CentOS / Rocky images later.

---

## Twenty, CDI and DataVolume Status Explanation

Check DataVolume:

    kubectl -n kubevirt-demo get dv

Common statuses:

    ImportScheduled
    ImportInProgress
    CloneScheduled
    UploadScheduled
    Succeeded
    Failed
    Pending

Check details:

    kubectl -n kubevirt-demo describe dv cirros-dv

Focus on:

    Phase
    Conditions
    Events

After DataVolume succeeds:

    PVC should be Bound

Check PVC:

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo describe pvc cirros-dv

---

## Twenty-one, Key Objects During CDI Import

After creating DataVolume, you'll typically see:

    DataVolume
    PVC
    importer Pod

Check:

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo get pods

After import completes:

    DataVolume Succeeded
    PVC Bound
    importer Pod completed or disappeared

If importer Pod fails:

    kubectl -n kubevirt-demo describe pod <importer-pod-name>

    kubectl -n kubevirt-demo logs <importer-pod-name> --tail=100

---

## Twenty-two, Common Troubleshooting

### 22.1 CDI Pod Not Running

Check:

    kubectl -n cdi get pods -o wide

    kubectl -n cdi get events --sort-by=.lastTimestamp

Check logs:

    kubectl -n cdi logs deploy/cdi-operator --tail=100

Common causes:

    1. Image pull failure
    2. Node resource insufficiency
    3. Security policy restrictions
    4. Incomplete CRD installation
    5. cdi CR not properly created

---

### 22.2 kubectl get dv Fails

Symptom:

    the server doesn't have a resource type "dv"

Note:

    DataVolume CRD not installed successfully.

Check:

    kubectl get crd | grep datavolumes

    kubectl -n cdi get pods

Resolution:

    Recheck CDI installation steps.

---

### 22.3 DataVolume Stuck in Pending

Check:

    kubectl -n kubevirt-demo describe dv cirros-dv

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo describe pvc cirros-dv

Common causes:

    1. StorageClass does not exist
    2. PVC cannot be Bound
    3. StorageClass provisioner anomaly
    4. Longhorn / NFS / CSI anomaly
    5. Storage capacity insufficient

Resolution:

    kubectl get storageclass

    kubectl -n longhorn-system get pods -o wide

    kubectl -n storage-system get pods -o wide

---

### 22.4 DataVolume Import Failed

Check importer Pod:

    kubectl -n kubevirt-demo get pods | grep importer

Check details:

    kubectl -n kubevirt-demo describe pod <importer-pod-name>

Check logs:

    kubectl -n kubevirt-demo logs <importer-pod-name> --tail=100

Common causes:

    1. Image URL inaccessible
    2. Company network cannot access the internet
    3. HTTPS certificate issues
    4. Unsupported image format
    5. PVC capacity too small
    6. importer Pod cannot write to PVC
    7. StorageClass mounting anomaly

Resolution approach:

    1. Curl image URL on the node
    2. Use internal HTTP file server
    3. Place image in internal object storage or Nginx
    4. Increase PVC capacity
    5. Check StorageClass backend status

---

### 22.5 Unable to Download CirrOS Image in Domestic Environment

If accessing:

    https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

is slow or fails, switch to internal address.

For example, deploy Nginx on ops-server:

    /data/www/images/cirros-0.6.2-x86_64-disk.img

Provide HTTP:

    http://10.0.0.10/images/cirros-0.6.2-x86_64-disk.img

Modify DataVolume to:

    source:
      http:
        url: "http://10.0.0.10/images/cirros-0.6.2-x86_64-disk.img"

Production recommendation: /think

Virtual machine images should be stored in the company's internal image repository, object storage, or controlled HTTP service.
It is not recommended to rely on public internet downloads in production environments.

---

### 22.6 PVC Bound But VM Fails to Start

Check VM:

    kubectl -n kubevirt-demo describe vm vm-cirros-dv

Check VMI:

    kubectl -n kubevirt-demo describe vmi vm-cirros-dv

Check virt-launcher:

    kubectl -n kubevirt-demo get pods | grep virt-launcher

    kubectl -n kubevirt-demo describe pod <virt-launcher-pod-name>

    kubectl -n kubevirt-demo logs <virt-launcher-pod-name> --tail=100

Common causes:

    1. The image is not a bootable disk
    2. Disk format anomaly
    3. cloud-init configuration anomaly
    4. Node does not support KVM
    5. Insufficient memory
    6. PVC mounting failure
    7. virt-handler anomaly

---

### 22.7 Console Cannot Log In

Check:

    kubectl -n kubevirt-demo get vmi

    kubectl -n kubevirt-demo describe vmi vm-cirros-dv

    kubectl -n kubevirt-demo logs <virt-launcher-pod-name> --tail=100

Try:

    virtctl console vm-cirros-dv -n kubevirt-demo

Press Enter multiple times after entering.

Common causes:

    1. Guest OS has not completed boot
    2. Image boot failure
    3. cloud-init not executed
    4. Incorrect username/password
    5. Console output is inactive

---

### 22.8 Does PVC Remain After DataVolume Deletion

DataVolume and PVC have a lifecycle relationship.

Before deletion, check:

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo get pvc

Delete DataVolume:

    kubectl -n kubevirt-demo delete dv cirros-dv

Check PVC again:

    kubectl -n kubevirt-demo get pvc

Depending on configuration and ownerReference behavior, PVC may be deleted together.

Do not arbitrarily delete DataVolume or PVC in production environments.

Before deletion, must confirm:

    1. Whether VM is still in use
    2. Whether PVC needs to be retained
    3. Whether backend data has backups
    4. Whether reclaimPolicy is Delete or Retain

---

## Twenty-Three, Standard Troubleshooting Path

### 23.1 DataVolume Import Failure

Execution order:

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo describe dv cirros-dv

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo get pods | grep importer

    kubectl -n kubevirt-demo describe pod <importer-pod-name>

    kubectl -n kubevirt-demo logs <importer-pod-name> --tail=100

    kubectl get storageclass

    Check backend storage component status

---

### 23.2 PVC Pending

Execution order:

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo describe pvc cirros-dv

    kubectl get storageclass

    kubectl describe storageclass longhorn

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get volumes.longhorn.io

---

### 23.3 VM Fails to Start After Using DataVolume

Execution order:

    kubectl -n kubevirt-demo get vm

    kubectl -n kubevirt-demo describe vm vm-cirros-dv

    kubectl -n kubevirt-demo get vmi

    kubectl -n kubevirt-demo describe vmi vm-cirros-dv

    kubectl -n kubevirt-demo get pod -o wide | grep virt-launcher

    kubectl -n kubevirt-demo describe pod <virt-launcher-pod-name>

    kubectl -n kubevirt-demo logs <virt-launcher-pod-name> --tail=100

---

## Twenty-Four, Common Command List

Check CDI:

    kubectl -n cdi get cdi

    kubectl -n cdi get pods -o wide

    kubectl get crd | grep cdi

Check DataVolume:

    kubectl get dv -A

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo describe dv cirros-dv

Check PVC:

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo describe pvc cirros-dv

Check importer:

    kubectl -n kubevirt-demo get pods | grep importer

kubectl -n kubevirt-demo logs <importer-pod-name> --tail=100

View VM:

    kubectl -n kubevirt-demo get vm

    kubectl -n kubevirt-demo describe vm vm-cirros-dv

View VMI:

    kubectl -n kubevirt-demo get vmi

    kubectl -n kubevirt-demo describe vmi vm-cirros-dv

Operate VM:

    virtctl start vm-cirros-dv -n kubevirt-demo

    virtctl stop vm-cirros-dv -n kubevirt-demo

    virtctl restart vm-cirros-dv -n kubevirt-demo

    virtctl console vm-cirros-dv -n kubevirt-demo

View events:

    kubectl -n kubevirt-demo get events --sort-by=.lastTimestamp

---

## Twenty-Five, Production Environment Recommendations

When using CDI/DataVolume in production environments, it is recommended to:

    1. Do not use public image URLs as production dependencies
    2. Place virtual machine images uniformly in internal image repositories or object storage
    3. Fix CDI version
    4. Fix image version
    5. Use formal OS images, such as Ubuntu, Rocky Linux, Windows, etc.
    6. Establish image verification mechanisms
    7. Set appropriate capacity for VM system disk PVC
    8. Use appropriate StorageClass
    9. Perform PVC backups
    10. Confirm business impact before deleting DataVolume/PVC
    11. Use SSH Key for production VMs, do not recommend simple passwords
    12. Plan image import, cloning, snapshot, and backup processes

---

## Twenty-Six, Clean Up Experimental Resources

Stop VM:

    virtctl stop vm-cirros-dv -n kubevirt-demo

    virtctl stop vm-cirros-dv-template -n kubevirt-demo

Delete Service:

    kubectl delete -f svc-vm-cirros-dv-ssh.yaml --ignore-not-found

Delete VM:

    kubectl delete -f vm-cirros-dv.yaml --ignore-not-found

    kubectl delete -f vm-cirros-dv-template.yaml --ignore-not-found

View VM/VMI:

    kubectl -n kubevirt-demo get vm

    kubectl -n kubevirt-demo get vmi

View DataVolume/PVC:

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo get pvc

Delete DataVolume:

    kubectl delete -f dv-cirros-http.yaml --ignore-not-found

Note:

    Deleting DataVolume may affect PVC.
    Experimental environments can be cleaned.
    Production environments cannot delete arbitrarily.

Delete namespace, if confirmed no data to retain:

    kubectl delete namespace kubevirt-demo

---

## Twenty-Seven, Summary of This Article

This article completed the basic hands-on practice of CDI and DataVolume.

Core workflow:

    1. Install CDI Operator
    2. Create CDI CR
    3. Verify CDI Pod and CRD
    4. Create DataVolume
    5. Import CirrOS image via HTTP
    6. Observe importer Pod
    7. Confirm PVC Bound
    8. Create VM referencing DataVolume
    9. Start VM
    10. Login VM via console
    11. Confirm PVC remains after stopping VM
    12. Learn dataVolumeTemplates for automatic disk creation

Core object relationships:

    DataVolume
      |
      v
    importer Pod
      |
      v
    PVC
      |
      v
    VM rootdisk
      |
      v
    VMI / virt-launcher
      |
      v
    Guest OS

Most important commands:

    kubectl -n cdi get pods

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo describe dv cirros-dv

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo logs <importer-pod-name>

    virtctl start vm-cirros-dv -n kubevirt-demo

    virtctl console vm-cirros-dv -n kubevirt-demo

Experience judgment:

    1. containerDisk is suitable for experience
    2. DataVolume + PVC is closer to real virtual machine disks
    3. Check importer Pod logs first when DataVolume fails
    4. Check StorageClass and backend storage first when PVC is Pending
    5. Check VM, VMI, virt-launcher, and PVC when VM fails to start
    6. Do not rely on public image URLs in production environments
    7. Must confirm data retention before deleting DataVolume/PVC

Next suggested learning material:

    07-KubeVirt Storage Basics: PVC, DataVolume, Longhorn and Virtual Machine Disks.md