# 06-CDI and DataVolume: Importing Virtual Machine Images, Managing PVCs and Boot Disks

Recommended Path:

    04-Kubernetes/12-KubeVirt/06-CDI and DataVolume: Importing Virtual Machine Images, Managing PVCs and Boot Disks.md

Tags:

    #Kubernetes
    #KubeVirt
    #CDI
    #DataVolume
    #PVC
    #StorageClass
    #Virtual machine image
    #Image import
    #Cloud-native virtualization
    #Platform engineering

---

## I. Document Description

This document records the basic usage of CDI and DataVolume in KubeVirt.

In the previous section, the first test virtual machine was created using containerDisk.

containerDisk is suitable for:

    1. Quick trials
    2. Demo tests
    3. Temporary verification of whether KubeVirt can run VMs

However, it is generally not recommended to directly use containerDisk as the system disk for production environment virtual machines.

A more common approach is:

    1. Prepare a virtual machine image
    2. Use CDI to import the image
    3. Use CDI to create a DataVolume
    4. Use the DataVolume to generate a PVC
    5. Use this PVC as the boot disk for the VM

The objectives of this document are:

    1. Understand what CDI is
    2. Understand what DataVolume is
    3. Install CDI
    4. Import a CirrOS image using HTTP
    5. Understand the relationship between DataVolume, PVC, and importer Pod
    6. Use DataVolume as the boot disk for a virtual machine
    7. Start the VM and access its console
    8. Learn how to troubleshoot common issues with CDI/DataVolume

---

## II. What is CDI?

CDI stands for:

    Containerized Data Importer

It is a component in the KubeVirt ecosystem used to manage the import of virtual machine disk data.

CDI primarily addresses the issue of how virtual machine images can be integrated into Kubernetes PVCs.

Without CDI, users would need to handle the following steps manually:

    1. Download qcow2/raw images
    2. Create a PVC
    3. Write the image to the PVC
    4. Verify the format and permissions
    5. Allow the VM to use the PVC

With CDI, these tasks can be automated through declarative DataVolume resources.

In simple terms:

    CDI = A component that helps KubeVirt prepare virtual machine disk data

---

## III. What is DataVolume?

DataVolume is a CRD resource provided by CDI.

It represents:

    A volume of disk data for use in a virtual machine

DataVolume automatically performs the following actions:

    1. Creates a PVC
    2. Generates an importer Pod
    3. Downloads the image from a specified source
    4. Writes the image to the PVC
    5. Updates the import status
    6. Makes the PVC available for use as a disk by the VM

Common sources for DataVolume include:

    HTTP
    Registry
    PVC Clone
    Upload
    Blank

This document will focus on the simplest method of importing images via HTTP.

---

## IV. The Relationship between CDI, DataVolume, PVC, and VM

The relationship is as follows:

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

In simple terms:

    DataVolume is responsible for preparing the disk data.
    PVC is the Kubernetes storage object that actually stores the disk data.
    The VM uses the PVC as its boot disk.

---

## V. Differences between CDI and containerDisk

| Comparison Item | containerDisk | DataVolume / PVC |
|---|---|---|
| Purpose | Quick trials | More closely resembles a real VM disk |
| Data Persistence | Not suitable for long-term persistence | PVCs are persistent |
| Image Source | Container images | HTTP/Registry/Upload/Clone |
    Suitability for Production | Not recommended as a production system disk | More suitable for basic production scenarios |
    Boot Disk Changes | Follow the image changes | The changes are reflected in the PVC |
    Data Retention | Data is not retained after the VM is deleted | PVCs retain data |

Conclusion:

    containerDisk is suitable for beginners.
    DataVolume + PVC provide a more realistic approach to disk management.

---

## VI. Experimental Objectives

After completing this experiment, you should beexport CDI_VERSION=v1.65.0

Note:

The version number should be confirmed based on actual verification.
Do not directly use an unverified version in a production environment.

---

### 9.3 Download CDI Operator YAML

Execute:

    curl -LO https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-operator.yaml

Check:

    ls -lh cdi-operator.yaml

---

### 9.4 Download CDI CR YAML

Execute:

    curl -LO https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-cr.yaml

Check:

    ls -lh cdi-cr.yaml

---

### 9.5 Instructions for Domestic Environments

If downloading from GitHub is slow, you can:

    1. Download cdi-operator.yaml in your browser.
    2. Download cdi-cr.yaml in your browser.
    3. Upload them to /root/k8s-yaml/kubevirt/cdi.
    4. Then execute kubectl apply.

The download addresses are in the following format:

    https://github.com/kubevirt/containerized-data-importer/releases/download/<CDI_VERSION>/cdi-operator.yaml

    https://github.com/kubevirt/containerized-data-importer/releases/download/<CDI_VERSION>/cdi-cr.yaml

---

## Section Ten: Deploying the CDI Operator

Execute:

    kubectl apply -f cdi-operator.yaml

Check the namespace:

    kubectl get ns cdi

View the Pods:

    kubectl -n cdi get pods -o wide

You should see:

    cdi-operator Running

View the logs:

    kubectl -n cdi logs deploy/cdi-operator --tail=100

---

## Section Eleven: Creating the CDI CR

Execute:

    kubectl apply -f cdi-cr.yaml

Check the CDI resources:

    kubectl -n cdi get cdi

Wait for the CDI deployment to complete:

    kubectl -n cdi wait cdi cdi --for condition=Available --timeout=10m

If the `wait` command is not supported or fails due to version differences, you can continue monitoring:

    kubectl -n cdi get cdi

    kubectl -n cdi get pods -o wide

Common components include:

    cdi-apiserver
    cdi-deployment
    cdi-operator
    cdi-uploadproxy
    cdi-uploadserver
    cdi-controller

Component names may vary slightly depending on the CDI version; refer to the actual output.

---

## Section Twelve: Verifying CDI Installation

### 12.1 Checking the CDI Pod

Execute:

    kubectl -n cdi get pods -o wide

The key Pod should be Running.

---

### 12.2 Checking the CDI CRD

Execute:

    kubectl get crd | grep cdi

Pay special attention to:

    datavolumes.cdi.kubevirt.io
    cdiconfigs.cdi.kubevirt.io
    cdis.cdi.kubevirt.io
    storageprofiles.cdi.kubevirt.io

---

### 12.3 Checking API Resources

Execute:

    kubectl api-resources | grep cdi

Verify the DataVolume:

    kubectl get dv -A

If there are no DataVolumes, it is normal for the output to be empty.

If the resources are not displayed, it means the CDI CRD was not installed successfully.

---

### 12.4 Checking StorageProfile

CDI will generate a StorageProfile based on the StorageClass.

Execute:

    kubectl get storageprofile

View the longhorn StorageProfile:

    kubectl describe storageprofile longhorn

Explanation:

    The StorageProfile records the CDI-related capabilities of a particular StorageClass.
    If there are issues with DataVolume import, it may be necessary to check the StorageProfile.

---

## Section Thirteen: Creating a Test Namespace

If the `kubevirt-demo` namespace already exists, you can skip this step.

Execute:

    kubectl create namespace kubevirt-demo

Or:

    kubectl create namespace kubevirt-demo --dry-run=client -o yaml | kubectl apply -f -

Check:

    kubectl get ns kubevirt-demo

---

## Section Fourteen: Experiment One: Using HTTP to Import a CirrOS Image into a DataVolume

### 14.1 Creating a DataVolume

Create a directory:

    mkdir -p /root/k8s-yaml/kubevirt/cdi-demo

    cd /root/k8s-yaml/kubevirt/cdi-demo

Create the DataVolume YAML:

    cat <<EOF > dv-cirros-http.yaml
    apiVersion: cdi.kubevirt.io/v1beta1
    kind: DataVolume
    metadata:
      name: cirros-dv
      namespace: kube### 15.2 Creating a VM

Execute:

    `kubectl apply -f vm-cirros-dv.yaml`

View:

    `kubectl -n kubevirt-demo get vm`

At this point, the VM should be in the Stopped state because the `runStrategy` is set to Manual.

View the VMI:

    `kubectl -n kubevirt-demo get vmi`

It is usually empty.

---

### 15.3 Starting the VM

Execute:

    `virtctl start vm-cirros-dv -n kubevirt-demo`

View:

    `kubectl -n kubevirt-demo get vm`
    `kubectl -n kubevirt-demo get vmi`
    `kubectl -n kubevirt-demo get pods -o wide`

Expected result:

    The VM should be Running.
    The VMI should be Running.
    The `virt-launcher Pod` should be Running.

---

### 15.4 Entering the Console

Execute:

    `virtctl console vm-cirros-dv -n kubevirt-demo`

Login:

    Username: cirros
    Password: kubevirt

After logging in, execute the following commands:

    `hostname`
    `ip addr`
    `df -h`
    `free -m`

To exit:

    `Ctrl + ]`

---

## Experiment 16: Stopping a VM to Verify PVC Persistence

Stop the VM:

    `virtctl stop vm-cirros-dv -n kubevirt-demo`

View:

    `kubectl -n kubevirt-demo get vm`
    `kubectl -n kubevirt-demo get vmi`
    `kubectl -n kubevirt-demo get pods`

Check the PVC:

    `kubectl -n kubevirt-demo get pvc`

Expected result:

    The VM should still exist.
    The VMI should disappear.
    The `virt-launcher Pod` should disappear.
    The `cirros-dv PVC` should still exist.

Explanation:

    This demonstrates the difference between using a DataVolume/PVC as the system disk for a VM and using it as a containerDisk. When the VM is stopped, the data on the PVC is retained.

---

## Experiment 17: Restarting the VM

Execute:

    `virtctl start vm-cirros-dv -n kubevirt-demo`

View:

    `kubectl -n kubevirt-demo get vm`
    `kubectl -n kubevirt-demo get vmi`

Enter the console again:

    `virtctl console vm-cirros-dv -n kubevirt-demo`

If files were written to the system disk earlier, they should still be there after restarting.

You can test this inside the VM by executing:

    `echo "hello from datavolume disk" > /home/cirros/dv-test.txt`

Then restart the VM:

    `sudo reboot`

Alternatively, you can exit the console and then restart it using:

    `virtctl restart vm-cirros-dv -n kubevirt-demo`

After logging in again, check the file:

    `cat /home/cirros/dv-test.txt`

Explanation:

    If the data is preserved, it confirms that the PVC system disk is persisting properly.

---

## Experiment 18: Using dataVolumeTemplates to Automatically Create DataVolumes

The previous method involved creating a DataVolume first and then a VM that referenced that DataVolume. KubeVirt also allows you to use `dataVolumeTemplates` directly within a VM configuration.

This method is suitable for:

    Automatically creating the corresponding DataVolume and PVC when a VM is created.

---

### 18.1 Creating a VM + DataVolume Template

Create the template:

    `cat <<EOF > vm-cirros-dv-template.yaml`
    apiVersion: kubevirt.io/v1
    kind: VirtualMachine
    metadata:
      name: vm-cirros-dv-template
      namespace: kubevirt-demo
      labels:
        app: vm-cirros-dv-template
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
## Networks:
- name: default
  pod: {}
## Volumes:
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

Application:

kubectl apply -f vm-cirros-dv-template.yaml

View:

kubectl -n kubevirt-demo get vm

kubectl -n kubevirt-demo get dv

kubectl -n kubevirt-demo get pvc

Start:

virtctl start vm-cirros-dv-template -n kubevirt-demo

Observation:

kubectl -n kubevirt-demo get dv,pvc,vm,vmi,pod

---

### 18.2 Comparison of Two Methods

| Method | Characteristics |
|---|---|
| Creating DataVolume Individually | Clear process, suitable for learning and troubleshooting |
| dataVolumeTemplates | Disks are automatically created when creating VMs, suitable for templating |

Beginner Recommendation:

Master creating DataVolume individually first.
Then use dataVolumeTemplates.

---

## Section Nineteen: Optional Experiment: Exposing SSH via Service

Create a Service:

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

Application:

kubectl apply -f svc-vm-cirros-dv-ssh.yaml

View:

kubectl -n kubevirt-demo get svc

kubectl -n kubevirt-demo get endpoints vm-cirros-dv-ssh

Access from Outside:

ssh cirros@10.0.0.23 -p 30023

Password:

kubevirt

If SSH connection fails, first check the sshd status inside the VM via console:

virtctl console vm-cirros-dv -n kubevirt-demo

Inside the VM, check:

ps aux | grep ssh

netstat -lntp

Note:

The CirrOS image is small and has limited functionality.
When using Ubuntu / CentOS / Rocky images later on, SSH behavior may differ.

---

## Section Twenty: Explanation of CDI and DataVolume Status

View DataVolume:

kubectl -n kubevirt-demo get dv

Common statuses:

ImportScheduled
ImportInProgress
CloneScheduled
UploadScheduled
Succeeded
Failed
Pending

View details:

kubectl -n kubevirt-demo describe dv cirros-dv

Pay special attention to:

Phase
Conditions
Events

After the DataVolume is successful:

The PVC should be Bound.

View PVC:

kubectl -n kubevirt-demo get pvc

kubectl -n kubevirt-demo describe pvc cirros-dv

---

## Section Twenty-One: Key Objects During CDI Import Process

After creating a DataVolume, you will typically see:

DataVolume
PVC
importer Pod

View:

kubectl -n kubevirt-demo get dv

kubectl -n kubevirt-demo get pvc

kubectl -n kubevirt-demo get pods

After the import is complete:

DataVolume becomes Succeeded
PVC gets Bound
The importer Pod completes or disappears.

If the importer Pod fails:

kubectl -n kubevirt-demo describe pod <importer-pod-name>

kubectl -n kubevirt-demo logs <importer-pod-name> --tail=100

---

## Section Twenty-Two: Common Problem Troubleshooting

### 22.1 CDI Pod Not Running

View:

kubectl -n cdi get pods -o wide

kubectl -n cdi get events --sort-by=.lastTimestamp

Check logs:

kubectl -n cdi logs deploy/cdi-operator --tail=100

Common causes:

1. Image retrieval failed
2. Insufficient node resources
3. Security policy restrictions
4. CRD not installed completely
5. cdi CR not created properly

---

### 22.2 Error When Using kubectl get dv

Message:

"the server doesn't have a resource type 'dv'

Explanation:

The DataVolume CRD was not installed successfully.

Check:

kubectl get crd | grep datavolumes

kubectl -n cdi get pods

Solution:

Recheck the CDI installation steps.

---

### 22.3 DataVolume Remains Pending

View:

kubectl -n kubevirt-demo describe dv cirros-dv

kubectl -n kubevirt-demo get pvc

kubectl -n kubevirt-demo describe pvc cirros-dv

Common causes:

1. StorageClass does not exist
2. PVC cannoturl: "http://10.0.0.10/images/cirros-0.6.2-x86_64-disk.img"

Production Recommendations:

    Virtual machine images should be stored in the company's internal image repository, object storage, or a controlled HTTP service.
    It is not recommended to rely on public network downloads for production environments.

---

### 22.6 PVC Bound but VM Fails to Start

Check the VM:

    kubectl -n kubevirt-demo describe vm vm-cirros-dv

Check the VMI:

    kubectl -n kubevirt-demo describe vmi vm-cirros-dv

Check virt-launcher:

    kubectl -n kubevirt-demo get pods | grep virt-launcher

    kubectl -n kubevirt-demo describe pod <virt-launcher-pod-name>

    kubectl -n kubevirt-demo logs <virt-launcher-pod-name> --tail=100

Common Causes:

    1. The image is not a bootable disk.
    2. Abnormal disk format.
    3. Issues with cloud-init configuration.
    4. The node does not support KVM.
    5. Insufficient memory.
    6. PVC mounting failure.
    7. Virt-handler errors.

---

### 22.7 Unable to Log in to the Console

Check:

    kubectl -n kubevirt-demo get vmi

    kubectl -n kubevirt-demo describe vmi vm-cirros-dv

    kubectl -n kubevirt-demo logs <virt-launcher-pod-name> --tail=100

Try:

    virtctl console vm-cirros-dv -n kubevirt-demo

Press Enter several times after entering.

Common Causes:

    1. The guest OS has not finished booting.
    2. Image startup failure.
    3. cloud-init failed to execute.
    4. Incorrect username or password.
    5. Console output is inactive.

---

### 22.8 Is the PVC Still Present After Deleting a DataVolume?

DataVolumes and PVCs have a lifecycle relationship.

Before deletion, check:

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo get pvc

Delete the DataVolume:

    kubectl -n kubevirt-demo delete dv cirros-dv

Then check the PVC:

    kubectl -n kubevirt-demo get pvc

Depending on configuration and ownerReference settings, the PVC may be deleted along with the DataVolume.

Do not arbitrarily delete DataVolumes or PVCs in a production environment.

Before deletion, confirm:

    1. Whether the VM is still in use.
    2. Whether the PVC needs to be retained.
    3. Whether backup copies of the backend data exist.
    4. Whether the reclaimPolicy is set to Delete or Retain.

---

## Twenty-Three: Standard Troubleshooting Steps

### 23.1 DataVolume Import Failure

Execution sequence:

    kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo describe dv cirros-dv

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo get pods | grep importer

    kubectl -n kubevirt-demo describe pod <importer-pod-name>

    kubectl -n kubevirt-demo logs <importer-pod-name> --tail=100

    kubectl get storageclass

    Check the status of backend storage components.

---

### 23.2 PVC Pending

Execution sequence:

    kubectl -n kubevirt-demo get pvc

    kubectl -n kubevirt-demo describe pvc cirros-dv

    kubectl get storageclass

    kubectl describe storageclass longhorn

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get volumes.longhorn.io

---

### 23.3 VM Fails to Start After Using a DataVolume

Execution sequence:

    kubectl -n kubevirt-demo get vm

    kubectl -n kubevirt-demo describe vm vm-cirros-dv

    kubectl -n kubevirt-demo get vmi

    kubectl -n kubevirt-demo describe vmi vm-cirros-dv

    kubectl -n kubevirt-demo get pod -o wide | grep virt-launcher

    kubectl -n kubevirt-demo describe pod <virt-launcher-pod-name>

    kubectl -n kubevirt-demo logs <virt-launcher-pod-name> --tail=100

---

## Twenty-Four: List of Common Commands

View CDI:

    kubectl -n cdi get cdi

    kubectl -n cdi get pods -o wide

    kubectl get crd | grep cdi

View DataVolume:

    kubectl get dv -A

       kubectl -n kubevirt-demo get dv

    kubectl -n kubevirt-demo get pvc

Delete the DataVolume:

    kubectl delete -f dv-cirros-http.yaml --ignore-not-found

Note:

Deleting the DataVolume may affect the PVC.
In a testing environment, it is okay to perform this operation.
However, in a production environment, deleting DataVolumes should be done with caution.

If you are sure that no data remains in the namespace, you can delete it:

    kubectl delete namespace kubevirt-demo

---

## Summary of Article 27

This article has provided practical guidance on using CDI and DataVolume.

Key steps include:

1. Installing the CDI Operator
2. Creating a CDI CR
3. Verifying the CDI Pod and CRD
4. Creating a DataVolume
5. Importing a CirrOS image via HTTP
6. Monitoring the importer Pod
7. Confirming that the PVC is bound
8. Creating a VM that uses the DataVolume
9. Starting the VM
10. Logging in to the VM via the console
11. Stopping the VM and verifying that the PVC remains intact
12. Learning how dataVolumeTemplates can automatically create disks

Key object relationships:

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

Key considerations:

    1. containerDisk is suitable for experimentation.
    2. Using DataVolume with PVC mimics the behavior of real virtual machine disks.
    3. If a DataVolume operation fails, check the importer Pod's logs first.
    4. When a PVC shows "Pending" status, check the StorageClass and backend storage.
    5. If a VM fails to start, examine the VM, VMI, virt-launcher, and PVC simultaneously.
    6. In production environments, avoid relying on public internet image URLs.
    7. Always confirm whether you need to keep data before deleting DataVolumes or PVCs.

Suggested next topic for further study:

    07-KubeVirt Storage Basics: PVC, DataVolume, Longhorn, and Virtual Machine Disks.md