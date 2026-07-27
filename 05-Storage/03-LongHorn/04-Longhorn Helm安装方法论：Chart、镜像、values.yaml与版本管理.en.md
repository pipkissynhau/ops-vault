# Longhorn Helm Installation Methodology: Charts, Images, values.yaml, and Version Management

Recommended Path: 05-Storage/03-LongHorn/04-Longhorn Helm Installation Methodology: Charts, Images, values.yaml, and Version Management.md

Tags: #Longhorn #Helm #Kubernetes #CSI #StorageClass #values.yaml #Image Management #Private Image Repository #containerd #Advanced SRE #Production Operations

---

## I. Document Description

This article is the fourth part of the Longhorn module, focusing on learning the methodology for installing Longhorn using Helm.

Previously completed:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependent Components, and StorageClass

This article officially enters the installation phase, but the focus is not on simply executing an installation command. Instead, it aims to establish a high-level operational perspective on Helm installation methodology:

    First, examine the Chart.
    Next, review the values.yaml file.
    Confirm the version.
    Extract the required images.
    Handle domestic networks and private image repositories.
    Customize your own values.yaml file.
    Execute the helm install command.
    Verify the Pod, CRD, StorageClass, and Volume capabilities.

This article emphasizes one principle:

    Do not arbitrarily disrupt the containerd or Kubernetes underlying runtime just to pull images. Preferably, use Helm’s values.yaml to control image repositories, tags, default configurations, and installation behavior.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand why Helm is recommended for managing Longhorn installations.
2. Add a Longhorn Helm repository.
3. View the version of the Longhorn Chart.
4. Fix the installation version of Longhorn.
5. Export the default values.yaml file.
6. Analyze configurations such as images, StorageClass, number of replicas, and default data paths from the values.yaml file.
7. Extract the list of images required for Longhorn.
8. Synchronize images to a private repository or a domestic image repository.
9. Specify the image repository using the values.yaml file.
10. Configure the default data path through the values.yaml file.
11. Set the default number of replicas using the values.yaml file.
12. Control whether to create a default StorageClass through the values.yaml file.
13. Execute Helm to install Longhorn.
14. Verify the components in the longhorn-system namespace.
15. Check the Longhorn CRD, StorageClass, and CSI components.
16. Troubleshoot installation issues such as ImagePullBackOff, Pod Pending, and CSI exceptions.
17. Establish a repeatable, reversible, and auditable installation process.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: Kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node IP Range: 10.0.0.0/24

Node allocation:

| IP | Host Name | Role | Longhorn Purpose |
|---|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane | Optional for experiments |
| 10.0.0.21 | k8s-worker01 | Worker | Longhorn data node |
| 10.0.0.22 | k8s-worker02 | Worker | Longhorn data node |

---

### 3.2 Prerequisite Dependencies

All nodes where Longhorn Volumes are planned to be deployed must have the following installed:

For Ubuntu 22.04:

    apt update
    apt install -y open-iscsi nfs-common
    systemctl enable --now iscsid

For Rocky Linux 9:

    dnf install -y iscsi-initiator-utils nfs-utils
    systemctl enable --now iscsid

Verification:

    systemctl is-active iscsid
    iscsiadm --version
    mount.nfs -V

---

### 3.3 Data Directory Planning

The data directory for Longhorn in this document is set as:

    /data/longhorn

Verification:

    df -hT /data/longhorn
    lsblk -f
    ls -ld /data/longhorn

Production Note:

    In the experimental environment, you can use the system disk directory initially. In a production environment, it is recommended to mount an independent data disk for /data/longhorn. It is not advisable to place the Longhorn data directory directly on the system disk if it will be used for production operations.

---

## IV. Why Use Helm to Install Longhorn

### 4.1 The Value of Helm

Helm can be considered a Kubernetes application package### 6.3 Principles for Selecting Versions

Principles for selecting versions:

    Prefer stable versions.
    Read the Release Notes.
    Check Kubernetes version compatibility.
    Review known issues.
    Verify the upgrade path.
    Test in a staging environment first.
    Do not pursue the latest version in production.
    Never deploy unverified versions directly into production.

Examples in this document use variables:

    export LONGHORN_VERSION=<Replace with the actual selected version>

For example:

    export LONGHORN_VERSION=1.10.0

Notes:

    The specific version should be determined by running `helm search repo longhorn/longhorn --versions`.
    In a production environment, do not simply copy the version number; instead, decide based on official compatibility information, Release Notes, and test results.

---

## VII. Viewing Chart Information

### 7.1 Viewing Chart Metadata

Run:

    helm show chart longhorn/longhorn --version ${LONGHORN_VERSION}

Pay attention to:

    name
    version
    appVersion
    kubeVersion
    description
    home
    sources

Save:

    helm show chart longhorn/longhorn --version ${LONGHORN_VERSION} \
      > reports/longhorn-chart-${LONGHORN_VERSION}.txt

---

### 7.2 Viewing the README

Run:

    helm show readme longhorn/longhorn --version ${LONGHORN_VERSION} | less

Save:

    helm show readme longhorn/longhorn --version ${LONGHORN_VERSION} \
      > reports/longhorn-readme-${LONGHORN_VERSION}.md

---

### 7.3 Pulling the Chart to Local

Run:

    helm pull longhorn/longhorn --version ${LONGHORN_VERSION}

Unzip:

    tar -zxvf longhorn-${LONGHORN_VERSION}.tgz

View the directory structure:

    tree longhorn | head -100

If `tree` is not available, install it:

    apt install -y tree

---

### 7.4 Viewing the Chart Directory Structure

Enter:

    cd longhorn

View:

    ls -lah

Common files include:

    Chart.yaml
    values.yaml
    templates/
    crds/

Notes:

    `Chart.yaml` describes the chart metadata.
    `values.yaml` contains default configurations.
    `templates/` holds Helm templates.
    `crds/` stores Longhorn CRDs.

Return to the working directory:

    cd ~/longhorn-install

---

## VIII. Exporting the Default values.yaml

### 8.1 Exporting Default Values

Run:

    helm show values longhorn/longhorn --version ${LONGHORN_VERSION} \
      > longhorn-values-default.yaml

View:

    less longhorn-values-default.yaml

---

### 8.2 Backing Up the Default Values

Create an unmodifiable copy of the original file:

    cp longhorn-values-default.yaml longhorn-values-default-${LONGHORN_VERSION}.yaml

Advice:

    Use `longhorn-values-default.yaml` only for reference.
    Save custom configurations in `longhorn-values-prod.yaml`.
    Avoid directly modifying the default file to facilitate comparisons later.

---

### 8.3 Finding Key Configurations

To find information related to images:

    grep -nE "image|repository|tag|registry" longhorn-values-default.yaml | head -100

For StorageClass settings:

    grep -nEi "storageclass|defaultClass|reclaimPolicy|replica|numberOfReplicas" longhorn-values-default.yaml | head -100

To locate the default data path:

    grep -nEi "defaultDataPath|dataPath|default.*path" longhorn-values-default.yaml

For UI-related settings:

    grep -nEi "frontend|service|ingress|ui" longhorn-values-default.yaml | head -100

To find backup configurations:

    grep -nEi "backup|s3|nfs" longhorn-values-default.yaml | head -100

---

## IX. Extracting the Longhorn Image List

### 9.1 Why Extract Images

In domestic network environments, issues such as `ImagePullBackOff`, `ErrImagePull`, or failures in pulling docker.io or ghcr.io images may occur. These problems can cause installation delays or Pod startup failures. An advanced Ops approach is not to directly modify the global containerd configuration but instead to identify:

    Which images are required for Longhorn.
    What the repository and tag of each image are.
    Whether these images can be synchronized to a private repository.
    If it is possible to specify the image repositories through `values.yaml`.

---

### 9.2 Extracting Image Fields from values

Run:

    grep -nE "repository:|tag:|registry:" longhorn-values-default.yaml

You can also view the surrounding context:

    grep -nE -A2 -B2 "repository:|tagTo disrupt the entire image pull logic for an application, temporarily add insecure registries in the production cluster without recording any modifications or backing up the containerd configuration. It is also essential to be unaware of which images need to be synchronized and not fix the Longhorn version.

The containerd configuration is part of the cluster infrastructure settings. Issues related to Longhorn images should primarily be addressed at the Helm values level.

---

### 10.3 Example of a Private Repository

Assume the private repository is:

    registry.cn-hangzhou.aliyuncs.com/pub-syq

It could also be an internal Harbor:

    10.0.0.10:8090/longhorn

Example synchronization logic:

    docker pull <official-image>
    docker tag <official-image> registry.cn-hangzhou.aliyuncs.com/pub-syq/<image-name>:<tag>
    docker push registry.cn-hangzhou.aliyuncs.com/pub-syq/<image-name>:<tag>

The actual image name and tag must come from:

    images/longhorn-images-${LONGHORN_VERSION}.txt

---

### 10.4 Template for Batch Synchronization Script

Create an image synchronization script:

    cat > images/sync-longhorn-images.sh <<'EOF'
    #!/bin/bash

    set -euo pipefail

    IMAGE_LIST="longhorn-images-${LONGHORN_VERSION}.txt"
    TARGET_REGISTRY="registry.cn-hangzhou.aliyuncs.com/pub-syq"

    if [ ! -f "${IMAGE_LIST}" ]; then
      echo "Image list not found: ${IMAGE_LIST}"
      exit 1
    fi

    while read -r image; do
      [ -z "${image}" ] && continue

      image_name_tag=$(echo "${image}" | awk -F/ '{print $NF}')
      target_image="${TARGET_REGISTRY}/${image_name_tag}"

      echo "==== Pulling ${image} ===>"
      docker pull "${image}"

      echo "==== Tagging ${image} to ${target_image} ===?"
      docker tag "${image}" "${target_image}"

      echo "==== Pushing ${target_image} ===?"
      docker push "${target_image}"

    done < "${IMAGE_LIST}"
    EOF

Note:

    This script is a template. Before actual use, check if there are any name conflicts in the image paths. Some image names may come from different repositories but have the same file name; production scripts should avoid overwriting them. It is more rigorous to retain the upstream path hierarchy, such as longhornio/longhorn-manager. This script is for learning purposes and should not be used directly in production without verification.

Authorization:

    chmod +x images/sync-longhorn-images.sh

Set the version variable before execution:

    export LONGHORN_VERSION=<actual version>

Enter the images directory and execute:

    cd images
    ./sync-longhorn-images.sh

---

### 10.5 More Robust Image Naming Convention

If you want to retain the upstream namespace, use the following logic:

    docker.io/longhornio/longhorn-manager:v1.x.x
      ->
    registry.cn-hangzhou.aliyuncs.com/pub-syq/longhornio-longhorn-manager:v1.x.x

Or:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/longhornio/longhorn-manager:v1.x.x

This depends on whether your image repository supports multi-level paths.

Production recommendations:

    Unify naming rules.
    Keep track of the image source and synchronization date.
    Maintain the Longhorn version.
    Do not mix tags for different versions.

---

## Chapter Eleven: Creating Custom values.yaml Files

### 11.1 Creating a Custom values File

Copy a file:

    cp longhorn-values-default.yaml longhorn-values-prod.yaml

Recommendation:

    Use longhorn-values-prod.yaml even in the experimental environment. All subsequent upgrades, rollbacks, and reviews should refer to this file. The file should be managed in a private Git repository but should not contain sensitive tokens.

---

### 11.2 Configuring the Global Image Repository

If values.yaml supports global.imageRegistry, you can configure it as follows:

    global:
      imageRegistry: "registry.cn-hangzhou.aliyuncs.com/pub-syq"

Note:

    Whether all images will be covered by global.imageRegistry depends on the actual rendering result of the current Chart. After setting this, you must re-run the helm template to check if the final images meet expectations. Do not just modify the values without verifying the rendering results.

---

### 11.3 Using the helm Template to Verify If Images Have Been Successfully Configured

Execute:

    helm template longhorn longhorn/longhorn \
      --namespace longhorn-system \
      --version ${LONGHORN_VERSION} \
      -f longhorn-values-prod.yaml \
      > manifests/rendered/longhornPersistence:  
ReclaimPolicy: Retain  

**Explanation:**  
- **Delete**: When a PVC is deleted, the underlying volume may also be removed.  
- **Retain**: After a PVC is deleted, both the PV and its data are retained, requiring manual intervention.  

**Production Recommendations:**  
- In temporary environments, **Delete** can be used.  
- For critical data, use **Retain** with caution.  
- Important operations must follow backup procedures and require approval before deletion.Public exposure over the internet is not allowed.

---

## Section Sixteen: Creating a PVC to Verify Longhorn Availability

### 16.1 Creating a Test Namespace

Execute:

    kubectl create namespace longhorn-install-demo

---

### 16.2 Creating a PVC

Create a file:

    cat > longhorn-install-pvc.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: install-demo-pvc
      namespace: longhorn-install-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn
      resources:
        requests:
          storage: 1Gi
    EOF

Apply:

    kubectl apply -f longhorn-install-pvc.yaml

Check:

    kubectl get pvc -n longhorn-install-demo
    kubectl get pv

If the PVC is still Pending:

    kubectl describe pvc install-demo-pvc -n longhorn-install-demo
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

---

### 16.3 Creating a Pod to Mount the PVC

Create a file:

    cat > longhorn-install-pod.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: install-demo-pod
      namespace: longhorn-install-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - "while true; do sleep 3600; done"
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: install-demo-pvc
    EOF

Apply:

    kubectl apply -f longhorn-install-pod.yaml

Check:

    kubectl get pod -n longhorn-install-demo -o wide

---

### 16.4 Writing Data

Execute:

    kubectl exec -n longhorn-install-demo install-demo-pod -- sh -c "echo 'hello longhorn helm install' > /data/hello.txt"

Check:

    kubectl exec -n longhorn-install-demo install-demo-pod -- cat /data/hello.txt

Expected output:

    hello longhorn helm install

---

### 16.5 Viewing Longhorn Internal Objects

View the Volume:

    kubectl -n longhorn-system get volumes.longhorn.io

View the Replica:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

View the Engine:

    kubectl -n longhorn-system get engines.longhorn.io

---

## Section Seventeen: Troubleshooting Installation Issues

### 17.1 ImagePullBackOff

Phenomenon:

    kubectl -n longhorn-system get pods
    The Pod status is ImagePullBackOff or ErrImagePull

Troubleshooting:

    kubectl -n longhorn-system describe pod <pod-name>

Check for the following issues:

    Failed to pull image
    Image not found
    I/O timeout
    Pull access denied
    X509 certificate issue
    No basic auth credentials

Possible solutions:

    Verify if the mirror repository is correct in values.yaml.
    Ensure that the image has been synchronized to the private repository.
    Check if the tag matches.
    Confirm that the node can access the private repository.
    Verify the imagePullSecrets settings.
    Avoid changing containerd global configurations first.

View the final rendered image:

    grep -oE 'image: [^ ]+' manifests/rendered/longhorn-final.yaml | sort -u

---

### 17.2 Pod Pending

Troubleshooting:

    kubectl -n longhorn-system describe pod <pod-name>
    kubectl get events -A --sort-by=.lastTimestamp | tail -100
    kubectl get nodes -o wide
    kubectl describe node <node-name>

Common causes:

    Insufficient node resources.
    Node has too many stains.
    nodeSelector does not match.
    Image pull is incomplete.
    CNI issues.
    kubelet failures.

---

### 17.3 CSI Component Errors

Troubleshooting:

    kubectl -n longhorn-system get pods | grep csi
    kubectl -n longhorn-system describe pod <csi-pod>
    kubectl -n longhorn-system logs <csi-pod> --tail=100

Consequences:

    PVC remains Pending.
    PV cannot be created.
    Pod cannot mount the PVC.
    Volume attach fails.

---

### 17.4 PVC Cannot Be Bound

Troubleshooting:

    kubectl get sc
    kubectl describe sc longhorn
    kubectl get pvc -n longhorn-install-demo
    kubectl describe pvc install-demo-pvc -n longDo not directly move directories with data in the production environment.

---

## Section Eighteen: Basics of Helm Upgrade, Rollback, and Uninstallation

### 18.1 Viewing Release History

Execute:

    helm history longhorn -n longhorn-system

---

### 18.2 Viewing Current Values

Execute:

    helm get values longhorn -n longhorn-system

To view all values:

    helm get values longhorn -n longhorn-system --all

---

### 18.3 Upgrade Principles

Upgrading Longhorn involves significant changes to storage components, which carry high risks.

Before upgrading, you must:

    Read the official upgrade documentation.
    Review the Release Notes.
    Ensure that the upgrade path is supported.
    Back up the current values.yaml file.
    Verify that the Backup Target is available.
    Make sure that critical volumes have backups.
    Upgrade in a test environment first.
    Perform the upgrade during off-peak business hours.
    Monitor the status of volumes, replicas, and engines.

The command format for upgrading is:

    helm upgrade longhorn longhorn/longhorn \
      --namespace longhorn-system \
      --version <new-version> \
      -f longhorn-values-prod.yaml

Notes:

    Do not attempt to upgrade across major version gaps.
    Never deviate from the official upgrade path.
    Do not upgrade when volumes are abnormal or replicas are being rebuilt.

---

### 18.4 Rollback Principles

To view history:

    helm history longhorn -n longhorn-system

The command format for rolling back is:

    helm rollback longhorn <revision> -n longhorn-system

Production warnings:

    Rolling back a storage system is different from rolling back a typical stateless application.
    A Chart rollback may not fully restore CRDs, data structures, or Longhorn's internal status.
    Always refer to the official upgrade and rollback guidelines.
    Ensure that the current environment is preserved before rolling back and assess potential data risks.

---

### 18.5 Uninstallation Principles

Uninstalling Longhorn is extremely risky.

Before uninstalling, confirm:

    That all PVCs have been migrated.
    That all volumes have been backed up.
    That no business pods are still using Longhorn PVCs.
    That you understand the consequences of deletion.
    That you have a recovery plan in place.
    That you have obtained necessary business approval.

To view PVCs:

    kubectl get pvc -A

To view volumes:

    kubectl -n longhorn-system get volumes.longhorn.io

The command format for uninstalling is:

    helm uninstall longhorn -n longhorn-system

High-risk warnings:

    Do not uninstall Longhorn in a production environment.
    Do not directly delete the longhorn-system namespace.
    Do not directly remove CRDs or data directories.
    Always refer to the official uninstallation documentation before proceeding.

---

## Section Nineteen: Production Installation Baselines

### 19.1 File Baselines

Production installations should retain at least:

    longhorn-values-default.yaml
    longhorn-values-prod.yaml
    Information about Longhorn Charts
    Results of helm template rendering
    List of images
    Records of image synchronization
    Outputs of helm status and history commands
    Pre-installation check reports
    Post-installation validation reports

---

### 19.2 Configuration Baselines

The production values.yaml file should specify:

    The Longhorn version.
    The image repository.
    The default data path.
    The default number of replicas.
    Whether to use the default StorageClass.
    The ReclaimPolicy settings.
    Volume expansion options.
    The way the UI will be exposed.
    Tolerations and nodeSelector settings.
    Any backup-related configurations.

---

### 19.3 Security Baselines

Production requirements include:

    The Longhorn UI must not be exposed directly to the public internet.
    Ingresses must use HTTPS.
    The UI must require authentication.
    Access should be limited to only the operations network segment.
    Regular business users should not have the permission to delete volumes.
    High-risk operations must be approved.

---

### 19.4 Operations Baselines

Production environments must have:

    Monitoring for longhorn-system Pods.
    Monitoring of Volume health, degradation, and failure status.
    Monitoring of replica reconstruction processes.
    Monitoring of node disk capacity.
    Monitoring of Backup Targets.
    Monitoring of PVC/PV status.
    Access to installation and upgrade documentation.
    Regular backup and recovery drills.

---

## Section Twenty: Experimental Cleanup

### 20.1 Deleting Test Pods

Execute:

    kubectl delete -f longhorn-install-pod.yaml

---

### 20.2 Deleting Test PVCs

High-risk warning:

    Deleting a PVC may result in the deletion of the corresponding Longhorn volume.
    These test PVCs can be deleted, but in a production environment, make sure to handle data ownership and backups properly.

Execute:

    kubectl## 22. High-Risk Operation Warnings

The following operations must be carried out with caution in a production environment:

- Directly using `helm install` without specifying a version.
- Installing without exporting the `values.yaml` file.
- Installing without checking the image.
- Starting installation without performing a pre-check.
- Exposing the Longhorn UI to the public network.
- Setting Longhorn as the default `StorageClass` without notifying the relevant departments.
- Setting the `ReclaimPolicy` to `Delete` without having a backup in place.
- Changing the default number of replicas without evaluating the capacity requirements.
- Upgrading Longhorn when there are issues with volumes.
- Directly deleting the `longhorn-system` component.
- Directly deleting the Longhorn CRD.
- Directly deleting the `/data/longhorn` directory.

Before proceeding, it is essential to confirm the following:

- Whether these operations will affect existing PVCs.
- Whether there are any backups in place.
- Whether a rollback plan has been prepared.
- Whether there is a suitable maintenance window.
- Whether the relevant departments have given their approval.
- Whether installation and change records will be properly maintained.

---

## 23. Completion Criteria for This Guide

After completing this guide, you should have achieved at least the following:

| Item | Standard |
|---|---|
| Helm | Installed and ready to use. |
| Chart Repository | The `longhorn repo` has been added successfully. |
| Version | The specific `LONGHORN_VERSION` has been selected and fixed. |
- Default Values | The default `values.yaml` file has been exported. |
- Custom Values | The `longhorn-values-prod.yaml` file containing custom configurations has been prepared. |
- Image List | All required images have been extracted using the `helm template`. |
- Image Repository | It has been determined whether to use a private repository for images. |
- Default Data Path | The `/data/longhorn` directory has been designated as the default data storage location. |
- Default Number of Replicas | The appropriate number of replicas has been configured based on the number of nodes. |
- StorageClass | It has been clarified whether Longhorn should use the default `StorageClass`. |
- Helm Installation | The `longhorn release` has been successfully deployed. |
- Pod Status | The `longhorn-system` component is running in a healthy state. |
- CRD Existence | The corresponding CRD is present and functional. |
- StorageClass Availability | The `Longhorn StorageClass` is properly configured. |
- PVC Verification | It has been confirmed that PVCs can be bound to the relevant volumes. |
- Pod Mounting | Pods can be mounted successfully, and data can be written to them. |
- UI Access | The Longhorn UI can be temporarily accessed via port-forwarding. |

---

## 24. Interview Response Guidelines

If you are asked during an interview:

"How would you install Longhorn? What precautions should be taken in a production environment?"

You could answer as follows:

"I recommend using Helm to install Longhorn instead of directly applying the configuration through `kubectl apply`. This approach allows for better version control, enables the export of `values.yaml` files, and provides more flexibility when managing image repositories, default data paths, and replica settings. It also makes it easier to perform subsequent upgrades and rollbacks."

Before starting the installation process, I would first conduct a pre-check to ensure that all Kubernetes nodes are in a Ready state, that the `kube-system` components are functioning correctly, and that both `containerd` and `kubelet` are running smoothly. Additionally, I would confirm that all Longhorn nodes have the `open-iscsi` driver installed and that the `iscsid` service is started. In cases where an RWX storage configuration is required, I would also ensure that an NFS client is available.

For image retrieval, I would first add the Longhorn Helm repository using `helm repo add longhorn`. Then, I would list available versions by executing `helm search repo longhorn/longhorn --versions` and select a verified version. Next, I would export the default configuration by running `helm show values`, and use `helm template` to generate the final YAML file that contains all the necessary image details.

If accessing external image repositories via the domestic network proves unreliable, I would consider synchronizing the images to a private repository such as Harbor or Alibaba Cloud. This would allow me to specify the image URL directly in the `values.yaml` file, rather than modifying the global `containerd` configuration.

After installation, I would check all components related to Longhorn, including Pods, Services, CRDs, and the `StorageClass` settings. To verify the functionality of dynamic volumes, I would create a test PVC and Pod to ensure that volumes can be successfully created, mounted, and used for data storage. In a production environment, it is crucial to secure the Longhorn UI by enabling HTTPS, implementing