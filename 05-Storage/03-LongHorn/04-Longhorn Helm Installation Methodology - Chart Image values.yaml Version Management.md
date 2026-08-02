# Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management

Recommended path: 05-Storage/03-LongHorn/04-Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management.md

Tags: #Longhorn #Helm #Kubernetes #CSI #StorageClass #values.yaml #MirrorManagement #PrivateMirrorWarehouse #containerd #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the fourth article in the Longhorn module, focusing on the methodology for installing Longhorn using Helm.

Previously completed:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass

This article officially enters the installation phase, but the focus is not on simply executing an installation command. Instead, it establishes an advanced operations perspective on Helm installation methodology:

    First inspect the Chart
    Then examine values.yaml
    Then confirm the version
    Then extract the images
    Then handle domestic network and private registry
    Then create your own values.yaml
    Then execute helm install
    Then verify Pods, CRDs, StorageClass, and Volume capabilities

This article emphasizes a principle:

    Do not arbitrarily disrupt containerd or Kubernetes runtime by pulling images.
    Prioritize controlling image repositories, tags, default configurations, and installation behavior through Helm values.yaml.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand why Longhorn recommends using Helm for management.
2. Add the Longhorn Helm repository.
3. View Longhorn Chart versions.
4. Fix the Longhorn installation version.
5. Export the default values.yaml.
6. Analyze images, StorageClass, replica count, default data path, etc., from values.yaml.
7. Extract the list of required Longhorn images.
8. Synchronize images to a private registry or domestic mirror.
9. Specify the image repository via values.yaml.
10. Configure the default data path via values.yaml.
11. Configure the default replica count via values.yaml.
12. Control whether to create the default StorageClass via values.yaml.
13. Execute Helm installation of Longhorn.
14. Verify components in the longhorn-system namespace.
15. Verify Longhorn CRDs, StorageClass, and CSI components.
16. Troubleshoot installation issues like ImagePullBackOff, Pod Pending, CSI anomalies.
17. Establish a repeatable, reversible, and auditable installation process.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node Network Segment: 10.0.0.0/24

Node planning:

| IP | Hostname | Role | Longhorn Purpose |
|---|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane | Experimental optional |
| 10.0.0.21 | k8s-worker01 | Worker | Longhorn Data Node |
| 10.0.0.22 | k8s-worker02 | Worker | Longhorn Data Node |

---

### 3.2 Prerequisites

All nodes planned to run Longhorn volumes should have completed:

Ubuntu 22.04:

    apt update
    apt install -y open-iscsi nfs-common
    systemctl enable --now iscsid

Rocky Linux 9:

    dnf install -y iscsi-initiator-utils nfs-utils
    systemctl enable --now iscsid

Check:

    systemctl is-active iscsid
    iscsiadm --version
    mount.nfs -V

---

### 3.3 Data Directory Planning

This article plans the Longhorn data directory as:

    /data/longhorn

Check:

    df -hT /data/longhorn
    lsblk -f
    ls -ld /data/longhorn

Production reminder:

    Experimental environments can initially use the system disk directory.
    Production environments recommend mounting /data/longhorn to an independent data disk.
    It is not recommended to place Longhorn data directory directly on the system disk for production workloads.

---

## IV. Why Use Helm to Install Longhorn

### 4.1 Helm's Value

Helm can be understood as a Kubernetes application package management tool.

For a complex system like Longhorn, Helm's value is:

    Unified version management.
    Unified values.yaml management.
    Support for parameterized configuration.
    Support for upgrades and rollbacks.
    Support for viewing historical versions.
    Support for exporting default configurations.
    Convenient for GitOps management.
    More suitable for production change governance than direct kubectl apply with large YAML files.

---

### 4.2 Why Longhorn is Suitable for Helm Management

Longhorn includes multiple components:

    longhorn-manager
    longhorn-driver-deployer
    longhorn-ui
    csi-provisioner
    csi-attacher
    csi-resizer
    csi-snapshotter
    longhorn-csi-plugin
    instance-manager
    engine-image
    CRD
    Service
    StorageClass
    Webhook

If directly using a one-click YAML, installation is fast, but it's not conducive to:

    Controlling versions.
    Controlling image repositories.
    Adjusting default replica count.
    Adjusting default data path.
    Production configuration auditing.
    Subsequent upgrade comparisons.
    Rollbacks and change records.

Helm is more suitable for creating maintainable installation processes.

---

### 4.3 Installation Principles in This Document

Principles adopted in this document:

    Fix the Longhorn version.
    Fix the values.yaml.
    Export default values before installation.
    Analyze images before installation.
    Prioritize synchronizing images to a private repository in domestic network environments.
    Specify the image repository via values.yaml.
    Avoid arbitrary modifications to containerd global configuration.
    Retain values.yaml and helm history after installation.
    Back up values.yaml before subsequent upgrades.

---

## Five. Preparing Helm Environment

### 5.1 Check Helm Installation

Execute:

    helm version

If not installed, Helm needs to be installed first.

Ubuntu example:

    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

If external network access is unavailable, use offline binary installation:

    tar -zxvf helm-v3.x.x-linux-amd64.tar.gz
    mv linux-amd64/helm /usr/local/bin/helm
    chmod +x /usr/local/bin/helm
    helm version

---

### 5.2 Check kubectl Context

Execute:

    kubectl config current-context
    kubectl get nodes -o wide
    kubectl get pods -A

Confirm:

    kubectl is pointing to the correct cluster.
    Nodes are Ready.
    kube-system components are normal.
    The current user has permissions to install CRD, Namespace, Deployment, DaemonSet, ClusterRole, etc.

---

### 5.3 Create a Working Directory

It is recommended to create a directory on the management node to save installation files:

    mkdir -p ~/longhorn-install
    cd ~/longhorn-install

Directory planning:

    ~/longhorn-install/
      longhorn-values-default.yaml
      longhorn-values-prod.yaml
      images/
      reports/
      manifests/

Create:

    mkdir -p images reports manifests

---

## Six. Adding Longhorn Helm Repository

### 6.1 Add Repository

Execute:

    helm repo add longhorn https://charts.longhorn.io

Update:

    helm repo update

View:

    helm repo list

---

### 6.2 Search Chart Versions

View available versions:

    helm search repo longhorn/longhorn --versions | head -20

Notes:

    Production environments must fix the version.
    It is not recommended to install without specifying a version.
    It is not recommended to use "current latest" as the basis for production installation.

---

### 6.3 Version Selection Principles

Version selection principles:

    Prioritize stable versions.
    Read Release Notes.
    Check Kubernetes version compatibility.
    Check known issues.
    Check upgrade paths.
    Test environment verification first.
    Production does not pursue the latest.
    Do not use unverified versions directly in production.

Example variable used in this document:

    export LONGHORN_VERSION=<replace with actual selected version>

Example:

    export LONGHORN_VERSION=1.10.0

Notes:

    The specific version should be based on the output of helm search repo longhorn/longhorn --versions.
    Do not directly copy version numbers for production environments; decide based on official compatibility, Release Notes, and test results.

---

## Seven. View Chart Information

### 7.1 View Chart Metadata

Execute:

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

### 7.2 View README

Execute:

    helm show readme longhorn/longhorn --version ${LONGHORN_VERSION} | less

Save:

    helm show readme longhorn/longhorn --version ${LONGHORN_VERSION} \
      > reports/longhorn-readme-${LONGHORN_VERSION}.md

---

### 7.3 Pull Chart to Local

Execute:

    helm pull longhorn/longhorn --version ${LONGHORN_VERSION}

Extract:

    tar -zxvf longhorn-${LONGHORN_VERSION}.tgz

View directory:

    tree longhorn | head -100

If tree is not available:

    apt install -y tree

---

### 7.4 View Chart Directory Structure

Enter:

    cd longhorn

View:

    ls -lah

Common files:

    Chart.yaml
    values.yaml
    templates/
    crds/

Notes:

    Chart.yaml describes Chart metadata.
    values.yaml is the default configuration.
    templates/ contains Helm templates.
    crds/ contains Longhorn CRD.

Return to the working directory:

    cd ~/longhorn-install

---

## Eight. Export Default values.yaml

### 8.1 Export Default values

Execute:

helm show values longhorn/longhorn --version ${LONGHORN_VERSION} \
  > longhorn-values-default.yaml

View:

  less longhorn-values-default.yaml

---

### 8.2 Default Values Backup

Save an unmodified original version:

  cp longhorn-values-default.yaml longhorn-values-default-${LONGHORN_VERSION}.yaml

Recommendations:

  longhorn-values-default.yaml is only for reference.
  Custom configurations should be written to longhorn-values-prod.yaml.
  Do not directly modify the default file to facilitate future comparisons.

---

### 8.3 Finding Key Configurations

Search for image-related entries:

  grep -nE "image|repository|tag|registry" longhorn-values-default.yaml | head -100

Search for StorageClass:

  grep -nEi "storageclass|defaultClass|reclaimPolicy|replica|numberOfReplicas" longhorn-values-default.yaml | head -100

Search for default data path:

  grep -nEi "defaultDataPath|dataPath|default.*path" longhorn-values-default.yaml

Search for UI-related entries:

  grep -nEi "frontend|service|ingress|ui" longhorn-values-default.yaml | head -100

Search for backup-related entries:

  grep -nEi "backup|s3|nfs" longhorn-values-default.yaml | head -100

---

## IX. Extracting Longhorn Image List

### 9.1 Why Extract Images

Domestic network environments may encounter:

  ImagePullBackOff
  ErrImagePull
  Failed to pull docker.io images
  Failed to pull ghcr.io images
  Installation gets stuck for a long time
  Pod fails to start

Advanced operations should not directly modify containerd global configurations, but instead first clarify:

  Which images Longhorn needs.
  What repository each image uses.
  What tag each image uses.
  Whether they can be synchronized to a private registry.
  Whether they can be specified via values.yaml.

---

### 9.2 Extracting Image Fields from Values

Execute:

  grep -nE "repository:|tag:|registry:" longhorn-values-default.yaml

You can also output context:

  grep -nE -A2 -B2 "repository:|tag:|registry:" longhorn-values-default.yaml

---

### 9.3 Using helm template to Render Images

Create a temporary rendering directory:

  mkdir -p manifests/rendered

Execute rendering:

  helm template longhorn longhorn/longhorn \
    --namespace longhorn-system \
    --version ${LONGHORN_VERSION} \
    -f longhorn-values-default.yaml \
    > manifests/rendered/longhorn-rendered-default.yaml

Extract image:

  grep -oE 'image: [^ ]+' manifests/rendered/longhorn-rendered-default.yaml | sort -u

Further processing:

  grep -oE 'image: [^ ]+' manifests/rendered/longhorn-rendered-default.yaml \
    | awk '{print $2}' \
    | sort -u \
    > images/longhorn-images-${LONGHORN_VERSION}.txt

View:

  cat images/longhorn-images-${LONGHORN_VERSION}.txt

---

### 9.4 Typical Image Types

Common image types in Longhorn include:

  longhorn-manager
  longhorn-engine
  longhorn-instance-manager
  longhorn-share-manager
  longhorn-backing-image-manager
  longhorn-ui
  longhorn-support-bundle-kit
  csi-attacher
  csi-provisioner
  csi-resizer
  csi-snapshotter
  node-driver-registrar
  livenessprobe

Notes:

  The specific image names and tags are determined by the current Chart rendering results.
  The list of images may vary across Longhorn versions.
  Do not write image lists based on memory.
  The image list should be extracted through helm template or helm show values.

Directly modify containerd global registry configuration.
To break all image pull logic for an application.
Temporarily adding insecure registry in production clusters.
Not recording modification content.
Not backing up containerd configuration.
Not knowing which images need synchronization.
Not fixing Longhorn version.

Containerd configuration is cluster infrastructure configuration.

Longhorn image issues should be prioritized at the Helm values layer.

---

### 10.3 Private Registry Example

Assume the private registry is:

    registry.cn-hangzhou.aliyuncs.com/pub-syq

It can also be an internal Harbor:

    10.0.0.10:8090/longhorn

Example synchronization logic:

    docker pull <official-image>
    docker tag <official-image> registry.cn-hangzhou.aliyuncs.com/pub-syq/<image-name>:<tag>
    docker push registry.cn-hangzhou.aliyuncs.com/pub-syq/<image-name>:<tag>

The actual image name and tag must come from:

    images/longhorn-images-${LONGHORN_VERSION}.txt

---

### 10.4 Batch Synchronization Script Template

Create an image synchronization script:

    cat > images/sync-longhorn-images.sh <<'EOF'
    #!/bin/bash

    set -euo pipefail

    IMAGE_LIST="longhorn-images-${LONGHORN_VERSION}.txt"
    TARGET_REGISTRY="registry.cn-hangzhou.aliyuncs.com/pub-syq"

    if [ ! -f "${IMAGE_LIST}" ]; then
      echo "image list not found: ${IMAGE_LIST}"
      exit 1
    fi

    while read -r image; do
      [ -z "${image}" ] && continue

      image_name_tag=$(echo "${image}" | awk -F/ '{print $NF}')
      target_image="${TARGET_REGISTRY}/${image_name_tag}"

      echo "==== Pull ${image} ===="
      docker pull "${image}"

      echo "==== Tag ${image} -> ${target_image} ===="
      docker tag "${image}" "${target_image}"

      echo "==== Push ${target_image} ===="
      docker push "${target_image}"

    done < "${IMAGE_LIST}"
    EOF

Notes:

    This script is a template.
    Before actual use, confirm if there are name conflicts in the image path.
    Some image names may come from different registries but have the same filename; production scripts should avoid overwriting.
    A more rigorous approach is to retain the upstream path hierarchy, such as longhornio/longhorn-manager.
    This script is for learning methodology and is not recommended for direct production execution without review.

Permissions:

    chmod +x images/sync-longhorn-images.sh

Set the version variable before execution:

    export LONGHORN_VERSION=<actual version>

Enter the images directory and execute:

    cd images
    ./sync-longhorn-images.sh

---

### 10.5 More Secure Image Naming Convention

If you want to retain the upstream namespace, use the following logic:

    docker.io/longhornio/longhorn-manager:v1.x.x
      ->
    registry.cn-hangzhou.aliyuncs.com/pub-syq/longhornio-longhorn-manager:v1.x.x

Or:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/longhornio/longhorn-manager:v1.x.x

This depends on whether your image registry supports multi-level paths.

Production recommendations:

    Use a unified naming convention.
    Retain image source records.
    Retain synchronization dates.
    Retain Longhorn version.
    Do not mix tags across different versions.

---

## Eleven. Writing Custom values.yaml

### 11.1 Creating a Custom Values File

Copy a file:

    cp longhorn-values-default.yaml longhorn-values-prod.yaml

Recommendations:

    Also use longhorn-values-prod.yaml in experimental environments.
    All upgrades, rollbacks, and reviews should be based on this file.
    The file should be managed in a private Git repository but should not include sensitive tokens.

---

### 11.2 Configuring Global Image Registry

If the values.yaml supports global.imageRegistry, configure it:

    global:
      imageRegistry: "registry.cn-hangzhou.aliyuncs.com/pub-syq"

Notes:

    Whether all images can be covered by global.imageRegistry depends on the current Chart rendering result.
    After setting, you must re-run helm template to check if the final image matches expectations.
    Do not only modify values without verifying the rendering result.

---

### 11.3 Using helm template to Verify Image Effectiveness

Execute:

    helm template longhorn longhorn/longhorn \
      --namespace longhorn-system \
      --version ${LONGHORN_VERSION} \
      -f longhorn-values-prod.yaml \
      > manifests/rendered/longhorn-rendered-prod.yaml

Check image: /think

grep -oE 'image: [^ ]+' manifests/rendered/longhorn-rendered-prod.yaml | sort -u

Verify:

    Ensure all images point to private repositories.
    Are there still references to docker.io?
    Are there still references to registry.k8s.io?
    Are there still references to ghcr.io?
    Are there any unsynchronized images?

If external images remain, further adjustments to values.yaml are needed.

---

### 11.4 Configure Default Data Path

Find default path configuration:

    grep -nEi "defaultDataPath|dataPath|default.*path" longhorn-values-prod.yaml

Depending on the Chart version, you may need to configure similar fields:

    defaultSettings:
      defaultDataPath: "/data/longhorn"

Notes:

    The actual field name depends on the current values.yaml.
    Different versions may have variations.
    After modification, validate with helm template.
    After installation, confirm the node data directory in Longhorn UI or CRD.

---

### 11.5 Configure Default Replica Count

For current experimental environments, if only two Worker nodes are used, plan:

    defaultSettings:
      defaultReplicaCount: 2

Or set in StorageClass parameters:

    persistence:
      defaultClassReplicaCount: 2

Notes:

    The actual field name depends on the current Chart values.yaml.
    Different versions may have variations.
    Must confirm with grep and helm template.

Search:

    grep -nEi "replica|Replica" longhorn-values-prod.yaml | head -50

Production recommendations:

    2 data nodes: start with 2 replicas.
    3 or more data nodes: can use 3 replicas.
    Replica count must match the actual number of schedulable data nodes.
    Do not blindly set 3 replicas and ignore replica scheduling failures.

---

### 11.6 Configure Whether to Create Default StorageClass

Search:

    grep -nEi "storageclass|defaultClass|default.*class" longhorn-values-prod.yaml | head -100

Depending on the Chart version, you may have similar configurations:

    persistence:
      defaultClass: true
      defaultClassReplicaCount: 2
      reclaimPolicy: Delete

Production recommendations:

    In learning environments, set defaultClass: true.
    In production environments with multiple storage systems, cautiously set default StorageClass.
    It's better to explicitly specify storageClassName: longhorn for business PVCs.

---

### 11.7 Configure ReclaimPolicy

If values support:

    persistence:
      reclaimPolicy: Delete

Or:

    persistence:
      reclaimPolicy: Retain

Understanding:

    Delete: After PVC deletion, the backend Volume may be deleted.
    Retain: After PVC deletion, PV and backend data are retained, requiring manual handling.

Production recommendations:

    Temporary environments can use Delete.
    Critical data should cautiously use Delete.
    Important business must have backup and deletion approval processes.

---

### 11.8 Configure Longhorn UI Service

Search for frontend:

    grep -nEi "frontend|service|ui|ingress" longhorn-values-prod.yaml | head -100

Experimental environment:

    Can use ClusterIP + port-forward.

Production environment:

    Avoid exposing via NodePort to public internet.
    If using Ingress, must add HTTPS, authentication, and source restrictions.
    Longhorn UI is the management entry point with high-risk capabilities like deleting Volumes.

---

## Twelve, Pre-Installation Rendering and Review

### 12.1 Render Final Installation Manifest

Execute:

    helm template longhorn longhorn/longhorn \
      --namespace longhorn-system \
      --version ${LONGHORN_VERSION} \
      -f longhorn-values-prod.yaml \
      > manifests/rendered/longhorn-final.yaml

---

### 12.2 Check Final Images

Execute:

    grep -oE 'image: [^ ]+' manifests/rendered/longhorn-final.yaml | sort -u

Verify:

    Whether the image registry meets expectations.
    Whether the tag meets expectations.
    Whether external images still exist.
    Whether all images have been synchronized.

---

### 12.3 Check Namespace

Execute:

    grep -n "namespace:" manifests/rendered/longhorn-final.yaml | head -50

Longhorn typically installs to:

    longhorn-system

---

### 12.4 Check StorageClass

Execute:

    grep -n "kind: StorageClass" -A30 manifests/rendered/longhorn-final.yaml

Focus on:

    name
    provisioner
    reclaimPolicy
    allowVolumeExpansion
    volumeBindingMode
    parameters
    numberOfReplicas

---

### 12.5 Check CRD

Execute:

    grep -n "kind: CustomResourceDefinition" manifests/rendered/longhorn-final.yaml | head

Notes:

    Longhorn will install many CRDs.
    CRDs are the foundation for Longhorn's internal object management.
    Must handle CRDs cautiously during upgrades and uninstallation.

---

## Thirteen, Execute Helm Installation /think

### 13.1 Creating Namespace

Run:

    kubectl create namespace longhorn-system

If it already exists:

    kubectl get ns longhorn-system

---

### 13.2 Executing Installation

Run:

    helm install longhorn longhorn/longhorn \
      --namespace longhorn-system \
      --version ${LONGHORN_VERSION} \
      -f longhorn-values-prod.yaml

---

### 13.3 Viewing Helm Release

Run:

    helm list -n longhorn-system

Check status:

    helm status longhorn -n longhorn-system

Check history:

    helm history longhorn -n longhorn-system

---

### 13.4 Saving Installation Records

Save Helm status:

    helm status longhorn -n longhorn-system > reports/helm-status-longhorn-${LONGHORN_VERSION}.txt

Save values:

    helm get values longhorn -n longhorn-system > reports/helm-values-longhorn-${LONGHORN_VERSION}.yaml

Save all values:

    helm get values longhorn -n longhorn-system --all > reports/helm-values-longhorn-${LONGHORN_VERSION}-all.yaml

---

## Fourteen, Post-Installation Basic Verification

### 14.1 Viewing Namespace Resources

Run:

    kubectl -n longhorn-system get all

---

### 14.2 Viewing Pod Status

Run:

    kubectl -n longhorn-system get pods -o wide

Continuous observation:

    kubectl -n longhorn-system get pods -w

Expected:

    longhorn-manager Running
    longhorn-driver-deployer Running or Completed
    longhorn-csi-plugin Running
    csi-attacher Running
    csi-provisioner Running
    csi-resizer Running
    csi-snapshotter Running
    longhorn-ui Running
    instance-manager Running

Note:

    Component names may vary with versions.
    Use actual Pod names as reference.
    Do not only check Running; also check READY and RESTARTS.

---

### 14.3 Viewing Abnormal Pods

Run:

    kubectl -n longhorn-system get pods | grep -Ev "Running|Completed"

If there are issues, continue troubleshooting:

    kubectl -n longhorn-system describe pod <pod-name>
    kubectl -n longhorn-system logs <pod-name> --tail=100

---

### 14.4 Viewing Services

Run:

    kubectl -n longhorn-system get svc

Focus on:

    longhorn-frontend

---

### 14.5 Viewing CRD

Run:

    kubectl get crd | grep longhorn

View API resources:

    kubectl api-resources | grep -i longhorn

---

### 14.6 Viewing StorageClass

Run:

    kubectl get sc

Expected to see:

    longhorn

View details:

    kubectl describe sc longhorn

Key confirmations:

    Provisioner must be Longhorn.
    ReclaimPolicy must match expectations.
    AllowVolumeExpansion must match expectations.
    numberOfReplicas must match expectations.
    Must be set as default.

---

### 14.7 Viewing Longhorn Node Objects

Run:

    kubectl -n longhorn-system get nodes.longhorn.io

View details:

    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>

Key confirmations:

    Nodes must be schedulable.
    Data path must be /data/longhorn.
    Disk capacity must be recognized.
    Disk status must be normal.

---

## Fifteen, Accessing Longhorn UI

### 15.1 Temporarily Access via port-forward

Run:

    kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80

Browser access:

    http://127.0.0.1:8080

Note:

    port-forward is only suitable for temporary experiments.
    Not recommended for long-term use in production.

---

### 15.2 Access via NodePort

In experimental environments, you can temporarily modify Service type, but it's not recommended for production exposure.

Check:

    kubectl -n longhorn-system get svc longhorn-frontend

Not recommended to directly expose Longhorn UI on production public network.

---

### 15.3 Access via Ingress

In production, if exposing Longhorn UI via Ingress, it should have:

    HTTPS
    Authentication
    Source IP restriction
    VPN or bastion host access
    RBAC control
    Audit logs

Reminder:

    Longhorn UI can delete Volumes.
    Longhorn UI is a management interface, not a business access interface.
    Public exposure is not allowed.

---

## Sixteen, Create PVC to Verify Longhorn Availability

### 16.1 Creating Test Namespace

Run:

kubectl create namespace longhorn-install-demo

---

### 16.2 Creating PVC

Create file:

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

If PVC is Pending:

    kubectl describe pvc install-demo-pvc -n longhorn-install-demo
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

---

### 16.3 Creating Pod with PVC Mount

Create file:

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

Check Volume:

    kubectl -n longhorn-system get volumes.longhorn.io

Check Replica:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Check Engine:

    kubectl -n longhorn-system get engines.longhorn.io

---

## SeventeenI don't know.Installation Troubleshooting

### 17.1 ImagePullBackOff

Phenomenon:

    kubectl -n longhorn-system get pods
    Pod status is ImagePullBackOff or ErrImagePull

Troubleshoot:

    kubectl -n longhorn-system describe pod <pod-name>

Focus on:

    Failed to pull image
    image not found
    i/o timeout
    pull access denied
    x509 certificate
    no basic auth credentials

Resolution direction:

    Check if the image repository in values.yaml is correct.
    Check if the image has been synchronized to the private registry.
    Check if the tag is consistent.
    Check if nodes can access the private registry.
    Check imagePullSecrets.
    Do not prioritize modifying containerd global configuration.

Check final rendered image:

    grep -oE 'image: [^ ]+' manifests/rendered/longhorn-final.yaml | sort -u

---

### 17.2 Pod Pending

Troubleshoot:

    kubectl -n longhorn-system describe pod <pod-name>
    kubectl get events -A --sort-by=.lastTimestamp | tail -100
    kubectl get nodes -o wide
    kubectl describe node <node-name>

Common causes:

    Insufficient node resources.
    Node taints are not tolerable.
    nodeSelector does not match.
    Image not pulled yet.
    CNI issues.
    kubelet issues.

---

### 17.3 CSI Component Abnormalities

Troubleshoot:

    kubectl -n longhorn-system get pods | grep csi
    kubectl -n longhorn-system describe pod <csi-pod>
    kubectl -n longhorn-system logs <csi-pod> --tail=100

Impact: /think

PVC Pending.
PV cannot be created.
Pod cannot mount PVC.
Volume attach failed.

---

### 17.4 PVC cannot be Bound

Troubleshooting:

    kubectl get sc
    kubectl describe sc longhorn
    kubectl get pvc -n longhorn-install-demo
    kubectl describe pvc install-demo-pvc -n longhorn-install-demo
    kubectl -n longhorn-system get pods
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

Common causes:

    longhorn StorageClass does not exist.
    CSI Provisioner is abnormal.
    Longhorn Manager is abnormal.
    Node disk is not schedulable.
    Replica scheduling failed.
    Default data path is abnormal.

---

### 17.5 Pod mounting PVC failed

Troubleshooting:

    kubectl describe pod install-demo-pod -n longhorn-install-demo
    kubectl get events -A --sort-by=.lastTimestamp | tail -100
    systemctl status iscsid
    iscsiadm --version
    kubectl -n longhorn-system get pods | grep csi
    kubectl -n longhorn-system get volumes.longhorn.io

Common causes:

    open-iscsi is not installed.
    iscsid is not running.
    Longhorn CSI Plugin is abnormal.
    Volume attach failed.
    Engine is abnormal.
    Instance Manager is abnormal.

Handling:

    apt install -y open-iscsi
    systemctl enable --now iscsid

---

### 17.6 Data directory does not match expectation

Troubleshooting:

    kubectl -n longhorn-system get nodes.longhorn.io
    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>

Node-side checks:

    df -hT /data/longhorn
    lsblk -f
    du -sh /data/longhorn

If data is found written to the system disk:

    Pause continuing to create production PVC.
    Confirm values.yaml default data path.
    Confirm Longhorn node disk configuration.
    Evaluate whether reconfiguring node disk path is needed.
    Do not directly move data directory in production environment.

---

## Eighteen, Helm Upgrade, Rollback and Uninstallation Basics

### 18.1 View Release History

Execute:

    helm history longhorn -n longhorn-system

---

### 18.2 View Current values

Execute:

    helm get values longhorn -n longhorn-system

View all values:

    helm get values longhorn -n longhorn-system --all

---

### 18.3 Upgrade Principles

Longhorn upgrade is a high-risk storage component change.

Before upgrading, must:

    Read official upgrade documentation.
    Read Release Notes.
    Confirm upgrade path is supported.
    Backup current values.yaml.
    Confirm Backup Target is available.
    Confirm critical Volume has backup.
    Upgrade in test environment first.
    Execute during business low-peak hours.
    Monitor Volume, Replica, Engine status.

Upgrade command format:

    helm upgrade longhorn longhorn/longhorn \
      --namespace longhorn-system \
      --version <new-version> \
      -f longhorn-values-prod.yaml

Notes:

    Do not arbitrarily upgrade across major versions.
    Do not skip official upgrade path.
    Do not upgrade during Volume anomalies or Replica rebuild.

---

### 18.4 Rollback Principles

View history:

    helm history longhorn -n longhorn-system

Rollback command format:

    helm rollback longhorn <revision> -n longhorn-system

Production reminder:

    Storage system rollback is not equivalent to rolling back stateless applications.
    Chart rollback may not fully revert CRD, data structures, and Longhorn internal state.
    Must refer to official upgrade and rollback documentation.
    Preserve the current state and assess data risks before rollback.

---

### 18.5 Uninstallation Principles

Uninstalling Longhorn is extremely dangerous.

Before uninstalling, confirm:

    All PVCs have been migrated.
    All Volumes have been backed up.
    No business Pod is using Longhorn PVC.
    Consequences of deletion have been confirmed.
    Recovery plan is in place.
    Business approval is obtained.

View PVCs:

    kubectl get pvc -A

View Volumes:

    kubectl -n longhorn-system get volumes.longhorn.io

Uninstallation command format:

    helm uninstall longhorn -n longhorn-system

High-risk reminder:

    Do not arbitrarily uninstall Longhorn in production environment.
    Do not directly delete longhorn-system namespace.
    Do not directly delete CRD and data directory.
    Must refer to official uninstallation documentation before uninstalling.

---

## Nineteen, Production Installation Baseline

### 19.1 File Baseline

Production installation should at least save: /think

longhorn-values-default.yaml  
longhorn-values-prod.yaml  
longhorn-chart Information  
helm template Rendering Result  
Image List  
Image Synchronization Record  
helm status Output  
helm history Output  
Pre-installation precheck Report  
Post-installation Verification Report  

---

### 19.2 Configuration Baseline  

Production values.yaml should explicitly define:  

    Longhorn version.  
    Image repository.  
    Default data path.  
    Default replica count.  
    Whether StorageClass is default.  
    ReclaimPolicy.  
    Volume expansion.  
    UI exposure method.  
    tolerations / nodeSelector.  
    Backup-related reserved configuration.  

---

### 19.3 Security Baseline  

Production requirements:  

    Longhorn UI must not be exposed to the public internet.  
    Ingress must use HTTPS.  
    UI must have authentication.  
    Only allow access from operations network segments.  
    Ordinary business personnel should not have delete Volume permissions.  
    High-risk operations require approval.  

---

### 19.4 Operations Baseline  

Production must have:  

    longhorn-system Pod monitoring.  
    Volume Healthy / Degraded / Faulted monitoring.  
    Replica reconstruction monitoring.  
    Node disk capacity monitoring.  
    Backup Target monitoring.  
    PVC / PV status monitoring.  
    Installation and upgrade documentation.  
    Backup and recovery drills.  

---

## Twenty, Experiment Cleanup  

### 20.1 Delete Test Pod  

Execute:  

    kubectl delete -f longhorn-install-pod.yaml  

---

### 20.2 Delete Test PVC  

High-risk warning:  

    Deleting PVC may result in deletion of backend Longhorn Volume.  
    This experiment's PVC can be deleted.  
    In production environments, confirm data ownership and backups before deletion.  

Execute:  

    kubectl delete -f longhorn-install-pvc.yaml  

Check:  

    kubectl get pvc -n longhorn-install-demo  
    kubectl get pv  
    kubectl -n longhorn-system get volumes.longhorn.io  

---

### 20.3 Delete Experimental Namespace  

Execute:  

    kubectl delete namespace longhorn-install-demo  

---

### 20.4 Delete Local Test YAML  

Execute:  

    rm -f longhorn-install-pvc.yaml  
    rm -f longhorn-install-pod.yaml  

---

### 20.5 Whether to Uninstall Longhorn  

If the experiment is only for installation, it is recommended to retain Longhorn for subsequent use in 05.  

If uninstallation is required:  

    helm uninstall longhorn -n longhorn-system  

High-risk warning:  

    Uninstalling Longhorn will affect all business relying on Longhorn PVC.  
    Confirm there are no critical PVCs before uninstallation.  
    Do not directly delete the longhorn-system namespace.  

---

## Twenty-one, Common Issues Summary  

### 21.1 helm repo add Fails  

Possible causes:  

    Unable to access charts.longhorn.io.  
    DNS resolution failure.  
    Proxy issues.  
    Certificate problems.  

Troubleshoot:  

    curl -I https://charts.longhorn.io  
    nslookup charts.longhorn.io  
    helm repo list  

Resolution:  

    Configure proxy.  
    Download Chart using accessible network.  
    Transfer Chart offline to internal network.  
    Use helm pull and install locally.  

---

### 21.2 helm install Fails  

Troubleshoot:  

    helm status longhorn -n longhorn-system  
    helm history longhorn -n longhorn-system  
    kubectl get events -n longhorn-system --sort-by=.lastTimestamp  
    kubectl -n longhorn-system get pods  

Common causes:  

    CRD installation failure.  
    Insufficient permissions.  
    values.yaml field errors.  
    Version incompatibility.  
    Image pull failure.  

---

### 21.3 Pod Long ImagePullBackOff  

Resolution process:  

    kubectl describe pod <pod-name> -n longhorn-system  
    Check failed image.  
    Compare with images/longhorn-images-${LONGHORN_VERSION}.txt.  
    Confirm image synchronization.  
    Confirm values.yaml effectiveness.  
    Re-validate with helm template.  
    Apply corrected values.yaml with helm upgrade.  

---

### 21.4 longhorn StorageClass Does Not Exist  

Troubleshoot:  

    kubectl get sc  
    helm status longhorn -n longhorn-system  
    kubectl -n longhorn-system get pods  
    kubectl get crd | grep longhorn  

Possible causes:  

    Installation not completed.  
    StorageClass disabled in values.  
    Longhorn CSI not started normally.  
    Helm Release anomaly.  

---

### 21.5 Longhorn UI Unreachable  

Troubleshoot:  

    kubectl -n longhorn-system get svc  
    kubectl -n longhorn-system get pods | grep frontend  
    kubectl -n longhorn-system logs <longhorn-ui-pod> --tail=100  

Temporary access:  

    kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80  

Browser:  

    http://127.0.0.1:8080  

---

## Twenty-two, High-risk Operation Reminder /think

# Strict Rules

## Operations that must be cautious in production environments:

- Install Helm without specifying a version.
- Install without exporting values.yaml.
- Install without checking images.
- Install without precheck.
- Expose Longhorn UI to the public internet.
- Set Longhorn as the default StorageClass without notifying business.
- Set ReclaimPolicy to Delete without backup.
- Modify default replica count without capacity assessment.
- Upgrade Longhorn during Volume anomalies.
- Directly delete longhorn-system.
- Directly delete Longhorn CRD.
- Directly delete /data/longhorn.

## Confirmations before execution:

- Does it affect existing PVCs?
- Is there backup?
- Is there a rollback plan?
- Is there a maintenance window?
- Has it been confirmed by business?
- Are installation and change records preserved?

---

## 23. Completion standards for this article's practical operations

After completing this article, the following standards should be met at least:

| Item | Standard |
|---|---|
| Helm | Installed and operational |
| Chart Repository | longhorn repo has been added |
| Version | LONGHORN_VERSION has been fixed |
| Default values | Exported |
| Custom values | longhorn-values-prod.yaml has been written |
| Image list | Extracted via helm template |
| Image repository | Whether private repository is used is clear |
| Default data path | /data/longhorn has been planned |
| Default replica count | Planned according to node count |
| StorageClass | Whether default is clear |
| Helm installation | longhorn release has been deployed |
| Pod | longhorn-system components are Running |
| CRD | longhorn CRD exists |
| StorageClass | longhorn exists |
| PVC verification | Test PVC can be Bound |
| Pod mounting | Test Pod can mount and write data |
| UI access | Can be temporarily accessed via port-forward |

---

## 24. Interview answerThinking.

If asked in an interview:

> "How would you install Longhorn? What precautions are needed in production?"

You can answer:

> "I prefer using Helm to install Longhorn rather than directly applying kubectl. Helm allows version fixing, values.yaml export, image repository control, default data path configuration, replica count, and StorageClass behavior, making it easier for subsequent upgrades and rollbacks.
> Before installation, I will perform precheck to confirm Kubernetes nodes are Ready, kube-system components are normal, containerd and kubelet are functioning, all Longhorn nodes have open-iscsi installed and iscsid is running, and for RWX scenarios, NFS client is installed. For disks, I will confirm /data/longhorn is mounted to an independent data disk, and production environments should not write Longhorn data to the system disk.
> When installing via Helm, I will first run helm repo add longhorn, then helm search repo longhorn/longhorn --versions to check versions, fix a verified version. Then helm show values to export default values.yaml, use helm template to render actual YAML, and extract all image lists. If domestic network pulls are unstable, I will synchronize images to Harbor or Alibaba Cloud registry, then specify imageRegistry via values.yaml instead of arbitrarily modifying containerd global configuration.
> After installation, I will check Pods, Services, CRD, and StorageClass in longhorn-system, confirm longhorn StorageClass exists, and create a test PVC and Pod to verify dynamic volume creation, mounting, and data writing. In production, Longhorn UI should not be exposed to the public internet, and HTTPS, authentication, and source restrictions must be added.
> Additionally, Longhorn replicas are not backups. Besides replica count and StorageClass, backup targets, monitoring alerts, and recovery drills must be planned. When upgrading Longhorn, official upgrade paths and Release Notes must be checked first, and values.yaml and helm history must be preserved, not treated like regular stateless applications for arbitrary upgrades and rollbacks."

---

## 25. Summary of this article

This article completes the methodology for Helm installation of Longhorn:

1. Helm is more suitable for production management of Longhorn.
2. Longhorn installation must fix the version.
3. It is not recommended to install without specifying a version.
4. helm show chart can view Chart information.
5. helm show values can export default values.yaml.
6. Custom configurations should be written to longhorn-values-prod.yaml.
7. helm template can render final YAML before installation.
8. Image list should be extracted from rendered results.
9. In domestic network environments, it is recommended to synchronize images to a private registry.
10. Image repository should prioritize management via values.yaml.
11. It is not recommended to arbitrarily disrupt containerd global configuration for Longhorn image pulls.
12. Pay attention to imageRegistry, default data path, replica count, StorageClass, and UI exposure method in values.yaml.
13. Check longhorn-system components after installation.
14. Check CRD, StorageClass, and CSI components after installation.
15. Create PVC and Pod after installation to verify real usability.
16. Longhorn UI is the management entry point and should not be exposed to the public internet.
17. Save Helm history, helm values, and rendered results.
18. Production upgrades must first assess version compatibility, upgrade paths, and data risks.
19. Longhorn replicas are not backups; subsequent configuration of Backup Target is required.
20. The next article will enter Longhorn dynamic volume practice: PVC, PV, Pod mounting, and data persistence.

---

## 26. Reference Documents

Longhorn official documentation:

    https://longhorn.io/docs/latest/

Longhorn installation documentation:

    https://longhorn.io/docs/latest/deploy/install/

Longhorn Helm Chart:

    https://github.com/longhorn/longhorn/tree/master/chart

Longhorn values.yaml:

    https://github.com/longhorn/longhorn/blob/master/chart/values.yaml

Longhorn StorageClass parameters:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Longhorn nodes and volumes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn backup and recovery:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn Troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Helm Official Documentation:

    https://helm.sh/docs/

Helm Chart Documentation:

    https://helm.sh/docs/topics/charts/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes CSI Documentation:

    https://kubernetes-csi.github.io/docs/ /think