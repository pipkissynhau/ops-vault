# Longhorn Backup and Restore: Backup Target, Snapshot, and Volume Restore

Recommended path: 05-Storage/03-LongHorn/07-Longhorn Backup and Restore: Backup Target, Snapshot, and Volume Restore.md

Tags: #Longhorn #BackupTarget #Snapshot #Backup #Restore #S3 #MinIO #NFS #Kubernetes #PVC #DataRestore #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the seventh article of the Longhorn module, focusing on learning Longhorn's snapshot, backup, restore, and Backup Target design.

Previously completed:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management
- Longhorn Dynamic Volume Practice: PVC, PV, Pod Mounting, and Data Persistence
- Longhorn Replica Mechanism: Replica Count, Node Distribution, and Data High Availability

This article enters the most critical part of Longhorn data protection:

    What is a Snapshot
    What is a Backup
    What is the difference between Snapshot and Backup
    Where should Backup Target be placed
    Why Replica is not backup
    Why Snapshot cannot fully equate to backup
    How to use NFS as Backup Target
    How to use MinIO/S3 as Backup Target
    How to back up Longhorn Volume
    How to restore Volume from Backup
    How to verify restored data
    How to design production environment backup strategy
    How to avoid data loss after accidental PVC deletion

This article emphasizes practical operations, verifying Longhorn's data recovery capabilities through real PVC, Pod, Snapshot, Backup, and Restore workflows.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the role of Longhorn Snapshot.
2. Understand the role of Longhorn Backup.
3. Understand the difference between Snapshot and Backup.
4. Understand the role of Backup Target.
5. Understand the boundaries between Replica, Snapshot, and Backup.
6. Configure NFS Backup Target.
7. Configure S3/MinIO Backup Target.
8. Create test PVC and Pod.
9. Write test data.
10. Create Longhorn Snapshot.
11. Roll back data from Snapshot.
12. Create Longhorn Backup.
13. Simulate data loss.
14. Restore Volume from Backup.
15. Rebind restored Volume to Pod.
16. Verify data integrity after restoration.
17. Design regular backup strategy.
18. Understand the importance of recovery drills in production environments.

---

## III. Core Conclusions First

### 3.1 Replica is Not Backup

Longhorn Replica addresses:

    Node failure
    Disk failure
    Single replica anomalies
    Volume availability degradation
    Replica reconstruction

It cannot address:

    User accidental file deletion
    Application data corruption
    User accidental PVC deletion
    Administrator accidental Volume deletion
    Kubernetes cluster-wide damage
    All replicas being destroyed simultaneously

Conclusion:

    Replica provides high availability.
    Replica is not backup.

---

### 3.2 Snapshot is Not Full Backup

Longhorn Snapshot addresses:

    Preservation of volume state at a specific time
    Rapid rollback
    Quick recovery after accidental operations
    Short-term data protection

However, Snapshot typically still relies on the storage system where the volume resides.

If:

    Longhorn cluster-wide damage
    Severe node data disk damage
    Volume deletion
    Snapshot's volume deletion
    Entire Kubernetes cluster unavailable

Snapshot may not serve as an independent recovery source.

Conclusion:

    Snapshot provides local time-point protection.
    Snapshot is not cross-region backup.
    Snapshot cannot replace Backup Target.

---

### 3.3 Backup Provides Cross-Fault Domain Recovery Capability

Longhorn Backup backs up volume data to external Backup Target.

Backup Target can be:

    NFS
    S3-compatible object storage
    MinIO
    SMB/CIFS
    Cloud object storage

This article focuses on two methods:

    NFS Backup Target
    MinIO/S3 Backup Target

Production recommendations:

    Backup Target separated from Kubernetes cluster
    Backup Target separated from Longhorn data disks
    Backup Target not in the same fault domain
    Regular recovery drills

---

## IV. Experimental Environment

### 4.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node Network Segment: 10.0.0.0/24

Node planning:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane |
| 10.0.0.21 | k8s-worker01 | Worker |
| 10.0.0.22 | k8s-worker02 | Worker |

---

### 4.2 Longhorn Environment

Longhorn Namespace:

    longhorn-system

Longhorn StorageClass:

    longhorn

Longhorn Data Directory:

    /data/longhorn

Experimental Namespace:

    longhorn-backup-demo

---

### 4.3 Backup Target Planning

# Two Backup Target Experiment Methods

## Method One: NFS Backup Target

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.10 | ops-server | NFS Backup Target |

NFS Export Directory:

    /data/longhorn-backup

Backup Target URL:

    nfs://10.0.0.10:/data/longhorn-backup

---

## Method Two: MinIO / S3 Backup Target

You can reuse the object storage environment from the previous MinIO module.

MinIO S3 Endpoint Example:

    http://10.0.0.41:9000

Or via a unified entry point:

    https://s3.minio.local

Longhorn Backup Bucket:

    longhorn-backup

Backup Target URL Example:

    s3://longhorn-backup@us-east-1/

S3 Authentication Secret:

    longhorn-s3-backup-secret

Notes:

    This article uses MinIO as an S3-compatible Backup Target example.
    In production, Backup Targets should not be placed in the same fault domain as Longhorn data nodes.
    If the Kubernetes cluster is completely damaged, backup data should still be accessible.

---

## Five. Snapshot, Backup, Restore Relationship

### 5.1 Snapshot

A Snapshot is a local snapshot of a Volume.

Features:

    Fast creation.
    Fast recovery.
    Suitable for short-term rollback.
    Suitable for protection before changes.
    Generally still depends on the current Longhorn cluster.

Suitable Scenarios:

    Take a snapshot before application upgrades.
    Take a snapshot before configuration changes.
    Take a snapshot before minor database changes.
    Quickly rollback after user accidental file deletion.

Not Suitable:

    As the sole backup.
    AsAlien. disaster recovery.
    As a cluster-level disaster recovery method.

---

### 5.2 Backup

Backup is backing up Volume data to an external Backup Target.

Features:

    Data leaves the Longhorn Volume local storage.
    Can be used for cross-cluster recovery.
    Can be used for recovery after severe node failures.
    Can be used for recovery after Volume deletion.
    Closer to true backup than Snapshot.

Suitable Scenarios:

    Production data protection.
    Cross-cluster recovery.
    Disaster recovery.
    Long-term retention.
    Recovery drills.
    Volume migration.

---

### 5.3 Restore

Restore creates a new Longhorn Volume from a Backup.

After recovery, you typically need:

    Create a new PVC.
    Bind the restored Volume to the PVC.
    Mount the restored PVC with a Pod.
    Verify data integrity.
    Decide whether to switch business operations.

---

### 5.4 Relationship Diagram

    Pod
     |
     v
    PVC
     |
     v
    Longhorn Volume
     |
     +--> Replica 1
     +--> Replica 2
     |
     +--> Snapshot
     |
     +--> Backup
             |
             v
        Backup Target
        NFS / S3 / MinIO
             |
             v
          Restore
             |
             v
        New Longhorn Volume
             |
             v
        New PVC / Pod

---

## Six. Pre-Operation Checks

### 6.1 Check Longhorn Components

Run:

    kubectl -n longhorn-system get pods -o wide

Check for anomalies:

    kubectl -n longhorn-system get pods | grep -Ev "Running|Completed"

If anomalies exist:

    kubectl -n longhorn-system describe pod <pod-name>
    kubectl -n longhorn-system logs <pod-name> --tail=100
    kubectl get events -n longhorn-system --sort-by=.lastTimestamp | tail -100

---

### 6.2 Check StorageClass

Run:

    kubectl get sc
    kubectl describe sc longhorn

Confirm:

    longhorn StorageClass exists.
    provisioner is driver.longhorn.io.
    PVC can dynamically create normally.

---

### 6.3 Check Volume Status

Run:

    kubectl -n longhorn-system get volumes.longhorn.io
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Requirements:

    No long-term Degraded Volumes.
    No Faulted Volumes.
    No large number of Replica anomalies.
    Do not perform backup recovery experiments during storage anomalies.

---

### 6.4 Check Node Status

Run:

    kubectl get nodes -o wide
    kubectl describe nodes | grep -E "Name:|DiskPressure|MemoryPressure|PIDPressure|Ready"

Requirements:

    Ready=True
    DiskPressure=False
    MemoryPressure=False
    PIDPressure=False

---

## Seven. Operation One: Create Test PVC and Pod

### 7.1 Create Namespace

Run:

    kubectl create namespace longhorn-backup-demo

Check:

    kubectl get ns longhorn-backup-demo

---

### 7.2 Create Working Directory

Run: /think

mkdir -p ~/longhorn-backup-demo
cd ~/longhorn-backup-demo

---

### 7.3 Creating PVC

Create file:

    cat > 01-pvc.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: backup-demo-pvc
      namespace: longhorn-backup-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn
      resources:
        requests:
          storage: 2Gi
    EOF

Apply:

    kubectl apply -f 01-pvc.yaml

Check:

    kubectl get pvc -n longhorn-backup-demo
    kubectl get pv

---

### 7.4 Creating Pod with PVC Mount

Create file:

    cat > 02-pod.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: backup-demo-pod
      namespace: longhorn-backup-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          imagePullPolicy: IfNotPresent
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
            claimName: backup-demo-pvc
    EOF

Apply:

    kubectl apply -f 02-pod.yaml

Check:

    kubectl get pod -n longhorn-backup-demo -o wide

If image pull fails, you can first synchronize busybox to your own image registry and then replace the image.

---

### 7.5 Writing Test Data

Write base file:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "echo 'longhorn backup restore demo' > /data/hello.txt"

Write directory structure:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "mkdir -p /data/app/config /data/app/logs /data/db"

Write configuration:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "echo 'app_name=longhorn-backup-demo' > /data/app/config/app.conf"

Write log:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "echo 'log line before snapshot' > /data/app/logs/app.log"

Write simulated database file:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "echo 'user_id,user_name' > /data/db/users.csv"
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "echo '1,alice' >> /data/db/users.csv"
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "echo '2,bob' >> /data/db/users.csv"

Create large file:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "dd if=/dev/zero of=/data/file-100m.bin bs=1M count=100"

Check:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- find /data -maxdepth 4 -type f -print
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- cat /data/hello.txt
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- cat /data/db/users.csv
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- ls -lh /data/file-100m.bin

---

### 7.6 Recording Checksum Values

Execute:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sha256sum /data/hello.txt
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sha256sum /data/db/users.csv
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sha256sum /data/file-100m.bin

It is recommended to save the output to your notes for comparison after recovery.

---

### 7.7 Viewing Longhorn Volume

Execute:

kubectl -n longhorn-system get volumes.longhorn.io -o wide

View Details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Record:

    Volume Name
    PVC Name
    PV Name
    Number of Replicas
    Current Status
    Current Mount Node

---

## VIII. Hands-on Exercise 2: Create Snapshot and Verify Rollback

### 8.1 Create Snapshot via Longhorn UI

Temporarily access Longhorn UI:

    kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80

Browser access:

    http://127.0.0.1:8080

Operation path:

    Volume
      -> Find backup-demo-pvc corresponding Volume
      -> Snapshot
      -> Create Snapshot

Recommended naming:

    before-delete-demo

Notes:

    UI operations are suitable for learning observation.
    In production, it is recommended to combine with Recurring Job or automation strategy.
    This article first uses UI to verify intuitively.

---

### 8.2 View Snapshot Object

View Snapshot list via UI.

You can also view related objects via Longhorn CRD.

Execute:

    kubectl -n longhorn-system get snapshots.longhorn.io

If the current version does not have snapshots.longhorn.io resource, you can view via UI.

Notes:

    Different Longhorn versions may expose different CRD details.
    Use the actual api-resources of the current version as reference.

View Longhorn resources:

    kubectl api-resources | grep -i longhorn

---

### 8.3 Simulate Accidental File Deletion

Delete some data in Pod:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- rm -f /data/db/users.csv
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- rm -f /data/app/config/app.conf

Confirm deletion:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- ls -lah /data/db
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- ls -lah /data/app/config

---

### 8.4 Rollback from Snapshot

High-risk warning:

    Snapshot rollback will affect current Volume data state.
    Before rollback in production environment, confirm business stop writing, backup and impact scope.
    This experiment only runs in test namespace.

Recommend to stop business Pod first to avoid writes during rollback.

Delete Pod:

    kubectl delete pod backup-demo-pod -n longhorn-backup-demo

Execute in Longhorn UI:

    Volume
      -> backup-demo-pvc corresponding Volume
      -> Snapshot
      -> Select before-delete-demo
      -> Revert

Wait for operation completion.

---

### 8.5 Recreate Pod to Verify Data

Recreate Pod:

    kubectl apply -f 02-pod.yaml

Wait for Running:

    kubectl get pod -n longhorn-backup-demo -o wide

Verify file recovery:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- cat /data/db/users.csv
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- cat /data/app/config/app.conf
    kubectl exec -n longhorn-backup-demo backup-demo-pod -- cat /data/hello.txt

Expected:

    users.csv is recovered.
    app.conf is recovered.
    hello.txt still exists.

---

### 8.6 Understanding Snapshot Rollback

Snapshot rollback is suitable for:

    Quick rollback after failed changes
    Rollback after failed application upgrades
    Recovery from small-scale accidental deletions
    Local quick recovery

Not suitable for:

    Cluster-level disaster recovery
    Recovery after Volume deletion
    Cross-cluster recovery when Backup Target does not exist
    Long-term backup retention

---

## IX. Hands-on Exercise 3: Configure NFS Backup Target

### 9.1 NFS Backup Target Planning

NFS Server:

    10.0.0.10

Export directory:

    /data/longhorn-backup

Backup Target:

    nfs://10.0.0.10:/data/longhorn-backup

Notes:

    NFS method is suitable for learning and small-to-medium environments.
    In production, ensure NFS has high availability and reliability.
    NFS Backup Target should not be on the same node or fault domain as Longhorn data disks.

---

### 9.2 Install NFS Service on ops-server

Execute on 10.0.0.10:

    apt update
    apt install -y nfs-kernel-server

Create directory:

    mkdir -p /data/longhorn-backup

Set permissions:

    chown -R nobody:nogroup /data/longhorn-backup
    chmod 0777 /data/longhorn-backup

High-risk warning:

    chmod 0777 is only suitable for experiments.
    In production, configure with minimal permissions, fixed client subnet, and security policies.
    Backup Target stores backup data, should not be exposed without control.

---

### 9.3 Configure NFS Export

Edit:

    vi /etc/exports

Add: /think

/data/longhorn-backup 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)

Apply:

    exportfs -ra

Check:

    exportfs -v

Start and set to boot:

    systemctl enable --now nfs-server
    systemctl status nfs-server

---

### 9.4 Installing NFS Client on Kubernetes Nodes

Execute on all Kubernetes nodes:

    apt update
    apt install -y nfs-common

Check:

    which mount.nfs
    mount.nfs -V

Test mount:

    mkdir -p /mnt/test-longhorn-backup
    mount -t nfs4 10.0.0.10:/data/longhorn-backup /mnt/test-longhorn-backup
    df -hT /mnt/test-longhorn-backup
    touch /mnt/test-longhorn-backup/test-from-$(hostname).txt
    umount /mnt/test-longhorn-backup

Notes:

    If a node cannot mount NFS, Longhorn using NFS Backup Target may fail.
    First resolve NFS network, permissions, and client dependencies before configuring Longhorn.

---

### 9.5 Configuring NFS Backup Target in Longhorn UI

Open Longhorn UI:

    kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80

Browser access:

    http://127.0.0.1:8080

Operation path:

    Settings
      -> Backup Target
      -> Backup Target

Fill:

    nfs://10.0.0.10:/data/longhorn-backup

Save.

You can also check:

    Backup
      -> Should be able to see Backup Target available

---

### 9.6 Viewing Configuration with kubectl

Check settings:

    kubectl -n longhorn-system get settings.longhorn.io | grep -i backup

Check backup-target:

    kubectl -n longhorn-system get settings.longhorn.io backup-target -o yaml

Notes:

    Longhorn's Backup Target configuration may vary across versions.
    Newer versions may have default backup target resources.
    Refer to the current version's actual UI and CRD.

---

## Ten. Practical Step Four: Configuring MinIO / S3 Backup Target

### 10.1 Why Recommend S3 Backup Target

Advantages of S3 / MinIO as Backup Target:

    More suitable for object-based backup data.
    No need to mount NFS directories on each node.
    More suitable for cross-cluster recovery.
    Easier for cross-node, cross-datacenter, cross-platform access.
    MinIO can serve as a private object storage backup target.
    Cloud provider S3 / OSS can serve asAlien. backup target.

Production recommendations:

    Backup Longhorn cluster data to S3 / MinIO outside the cluster.
    Do not place MinIO and Longhorn critical data on the same batch of failure nodes.
    Backup Target should have independent availability.

---

### 10.2 Creating MinIO Bucket

After configuring MinIO alias with mc client, execute:

    mc mb minio-admin/longhorn-backup

If using Docker version of mc:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mb minio-admin/longhorn-backup

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-admin

---

### 10.3 Creating MinIO Access User and Policy

Recommend creating a dedicated MinIO user for Longhorn, not using root.

User:

    longhorn-backup-user

Password:

    LonghornBackup@123456

Create user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user add minio-admin longhorn-backup-user 'LonghornBackup@123456'

Create Policy file: /think

```bash
cat > longhorn-backup-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::longhorn-backup"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::longhorn-backup/*"
      ]
    }
  ]
}
EOF
```

**Create Policy:**

```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v $(pwd):/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin policy create minio-admin longhorn-backup-policy /demo/longhorn-backup-policy.json
```

**Attach Policy:**

```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin policy attach minio-admin longhorn-backup-policy --user longhorn-backup-user
```

**Verify:**

```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin user info minio-admin longhorn-backup-user
```

---

### 10.4 Create Longhorn S3 Secret

Longhorn S3 Backup Target requires Kubernetes Secret to store access credentials.

**Create File:**

```bash
cat > 03-longhorn-s3-secret.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: longhorn-s3-backup-secret
  namespace: longhorn-system
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "longhorn-backup-user"
  AWS_SECRET_ACCESS_KEY: "LonghornBackup@123456"
  AWS_ENDPOINTS: "http://10.0.0.41:9000"
  AWS_REGION: "us-east-1"
EOF
```

**Apply:**

```bash
kubectl apply -f 03-longhorn-s3-secret.yaml
```

**Check:**

```bash
kubectl -n longhorn-system get secret longhorn-s3-backup-secret
```

**Security Reminder:**

- Secret contains plaintext credentials.
- YAML should not be committed to public Git.
- Production should use more secure Secret management.
- AccessKey should be configured with minimal permissions, only allowing access to longhorn-backup Bucket.

---

### 10.5 S3 Backup Target Format

Backup Target Example:

```
s3://longhorn-backup@us-east-1/
```

**Meaning:**

- Bucket: longhorn-backup
- Region: us-east-1

**Endpoint** is specified through:

```
AWS_ENDPOINTS
```

in the Secret.

If using HTTPS MinIO endpoint:

```
AWS_ENDPOINTS: "https://s3.minio.local"
```

If using self-signed certificate, additional certificate trust handling is required.

For experimental environments, internal HTTP endpoint can be used for simplicity:

```
http://10.0.0.41:9000
```

**Production Recommendation:**

- External or cross-network access should use HTTPS.
- Internal trusted network can evaluate HTTP, but must have network isolation.
- Backup Target credentials must have minimal permissions.

---

### 10.6 Configure S3 Backup Target in Longhorn UI

**Open UI:**

```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
```

**Access via Browser:**

```
http://127.0.0.1:8080
```

**Operation Path:**

```
Settings
  -> Backup Target
```

**Fill Backup Target:**

```
s3://longhorn-backup@us-east-1/
```

**Fill Credential Secret:**

```
longhorn-s3-backup-secret
```

**Save.**

Check if Backup page can load normally.

---

### 10.7 View Backup Target Configuration with kubectl

**Check:**

```bash
kubectl -n longhorn-system get settings.longhorn.io | grep -i backup
```

**Check backup target:** /think

kubectl -n longhorn-system get settings.longhorn.io backup-target -o yaml

View credential secret:

kubectl -n longhorn-system get settings.longhorn.io backup-target-credential-secret -o yaml

Notes:

The specific setting name is based on the current version.
Newer versions may use BackupTarget CRD for management.
You can execute the following command to check:

kubectl api-resources | grep -i backup

---

## 11. Practical Operation Five: Create Longhorn Backup

### 11.1 Pre-check Before Creating Backup

Confirm Pod data:

kubectl exec -n longhorn-backup-demo backup-demo-pod -- find /data -maxdepth 4 -type f -print
kubectl exec -n longhorn-backup-demo backup-demo-pod -- cat /data/db/users.csv

Confirm Volume:

kubectl -n longhorn-system get volumes.longhorn.io -o wide

Confirm Backup Target:

Longhorn UI -> Backup page has no errors

---

### 11.2 Create Backup via UI

Operation path:

Volume
  -> Find the backup-demo-pvc corresponding Volume
  -> Snapshot
  -> Create Snapshot

After creating Snapshot:

Select the Snapshot
  -> Create Backup

Recommended naming or note:

backup-before-delete-demo

Wait for Backup to complete.

---

### 11.3 Check Backup Status

Longhorn UI:

Backup
  -> View Backup list
  -> View Volume Backup
  -> Confirm status Completed

If using NFS Backup Target, check on ops-server:

find /data/longhorn-backup -maxdepth 5 -type f | head -50
du -sh /data/longhorn-backup

If using MinIO Backup Target:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-admin/longhorn-backup

Check capacity:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure du minio-admin/longhorn-backup

---

### 11.4 Troubleshoot Failed Backup

If Backup fails, first check:

Longhorn UI error message
longhorn-manager logs
Backup Target accessibility
Secret correctness
MinIO Bucket existence
MinIO Policy has PutObject permission
NFS writability
DNS/Endpoint reachability

Commands:

kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

Check Secret:

kubectl -n longhorn-system get secret longhorn-s3-backup-secret -o yaml

Test MinIO user permissions:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure alias set longhorn-bak-test http://10.0.0.41:9000 longhorn-backup-user 'LonghornBackup@123456'

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure ls longhorn-bak-test/longhorn-backup

---

## 12. Practical Operation Six: Simulate Volume Data Loss

### 12.1 Simulate Application Layer Data Deletion

Delete data in Pod:

kubectl exec -n longhorn-backup-demo backup-demo-pod -- rm -rf /data/db
kubectl exec -n longhorn-backup-demo backup-demo-pod -- rm -rf /data/app/config
kubectl exec -n longhorn-backup-demo backup-demo-pod -- rm -f /data/hello.txt

Confirm:

kubectl exec -n longhorn-backup-demo backup-demo-pod -- find /data -maxdepth 4 -type f -print

At this point, the data has been deleted by the application layer.

Note:

Longhorn Replica will faithfully replicate deletion operations.
Multiple replicas do not prevent accidental deletion.
This is why Replica is not a backup.

### 12.2 Misunderstanding When Trying to Recover via Replica

Misunderstanding:

    With 2 replicas or 3 replicas, data can be recovered from other replicas after accidental deletion.

Correct Understanding:

    Deletion operations are synchronized to all healthy replicas.
    Replicas only guarantee block device consistency.
    Replicas do not understand whether application-layer files were accidentally deleted.
    Accidental deletion recovery relies on Snapshot or Backup.

---

## Thirteen, Practical Seven: Recovering Volume from Backup

### 13.1 Recommendations Before Restore

Production recovery recommendations:

    Do not directly overwrite the original business.
    Prioritize restoring to a new Volume.
    Use new PVC and new Pod to verify data.
    Decide whether to switch business after verification.
    Preserve the original Volume scene.

In experiments, also adopt:

    Restore from Backup to a new Volume
    Create new PVC to mount the restored Volume
    Use new Pod to verify data

---

### 13.2 Restoring from Backup via UI

Longhorn UI operation path:

    Backup
      -> Find the backup-demo-pvc corresponding backup
      -> Restore Latest Backup
      -> Input new Volume name

Recommended restored Volume name:

    restore-backup-demo-volume

Wait for Restore to complete.

---

### 13.3 Viewing the Restored Volume

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Should see:

    restore-backup-demo-volume

Check details:

    kubectl -n longhorn-system describe volumes.longhorn.io restore-backup-demo-volume

Focus on:

    State
    Robustness
    Size
    Number Of Replicas

---

### 13.4 Creating PVC for the Restored Volume

Longhorn UI typically supports creating PVC from Volume.

You can also handle it by specifying Longhorn Volume via Kubernetes PVC.

Recommended for learning phase using UI:

    Volume
      -> restore-backup-demo-volume
      -> Create PV/PVC

Fill in:

    Namespace: longhorn-backup-demo
    PVC Name: restored-pvc

After creation, check:

    kubectl get pvc -n longhorn-backup-demo
    kubectl get pv

---

### 13.5 Creating a Restore Verification Pod

Create file:

    cat > 04-restored-pod.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: restored-pod
      namespace: longhorn-backup-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          imagePullPolicy: IfNotPresent
          command:
            - sh
            - -c
            - "while true; do sleep 3600; done"
          volumeMounts:
            - name: data
              mountPath: /restore
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: restored-pvc
    EOF

Apply:

    kubectl apply -f 04-restored-pod.yaml

Check:

    kubectl get pod -n longhorn-backup-demo -o wide

---

### 13.6 Verifying Restored Data

Execute:

    kubectl exec -n longhorn-backup-demo restored-pod -- find /restore -maxdepth 4 -type f -print

Check key files:

    kubectl exec -n longhorn-backup-demo restored-pod -- cat /restore/hello.txt
    kubectl exec -n longhorn-backup-demo restored-pod -- cat /restore/db/users.csv
    kubectl exec -n longhorn-backup-demo restored-pod -- cat /restore/app/config/app.conf
    kubectl exec -n longhorn-backup-demo restored-pod -- ls -lh /restore/file-100m.bin

Calculate checksums:

    kubectl exec -n longhorn-backup-demo restored-pod -- sha256sum /restore/hello.txt
    kubectl exec -n longhorn-backup-demo restored-pod -- sha256sum /restore/db/users.csv
    kubectl exec -n longhorn-backup-demo restored-pod -- sha256sum /restore/file-100m.bin

Compare with pre-backup records.

---

### 13.7 Conclusion of Restoration

If data is intact after restoration:

    Backup Target is available.
    Backup is available.
    Restore process is available.
    The restored Volume can be remounted.
    Data can be read by business Pod.

Only then is the backup cycle complete.

Creating Backup without Restore verification cannot be considered as complete production backup capability.

---

## Fourteen, Practical Eight: Switching Business After Restoration from Backup

### 14.1 Production Switching Principles

In production, it's not recommended to blindly overwrite the original business.

Recommended process: /think

1. Recover from Backup to a New Volume.
2. Create a New PVC.
3. Create a Validation Pod.
4. Validate Files and Business Data.
5. Stop Original Business Writes.
6. Modify Business Workload to Use the Restored PVC.
7. Start Business Operations.
8. Validate Business Operations.
9. Retain the Original PVC for a Period of Time.
10. Complete Post-Mortem Analysis.

---

### 14.2 Deployment PVC Switching Example

Assume the original Deployment uses:

    backup-demo-pvc

After recovery, the new PVC is:

    restored-pvc

Modify the Deployment's volumes:

    persistentVolumeClaim:
      claimName: restored-pvc

Then:

    kubectl apply -f deployment.yaml
    kubectl rollout status deploy/<deploy-name> -n <namespace>

Production reminders:

    Writing must be stopped before switching.
    Database applications should combine with database consistency checks.
    Do not switch to the new PVC while writing to the old volume.
    Retain the old PVC for rollback after switching.

---

### 14.3 StatefulSet Switching Considerations

Switching PVCs with StatefulSet using volumeClaimTemplates is more complex.

Considerations include:

    Pod Name
    PVC Name
    volumeClaimTemplates Generation Rules
    PV Binding Relationships
    Data Consistency
    Whether to Rebuild StatefulSet
    Whether to Restore to the Same-Named PVC

Production StatefulSet recovery must undergo detailed rehearsals; manual operations are not recommended.

---

## FifteenI don't know.Recurring Job: Regular Snapshots and Backups

### 15.1 Why Need Recurring Job

Manual backups are unreliable.

Production requires:

    Automatic Regular Snapshot
    Automatic Regular Backup
    Automatic Retention of a Certain Number
    Automatic Cleanup of Old Backups
    Backup Failure Alerts
    Regular Recovery Drills

Longhorn supports setting Recurring Jobs via UI or configuration.

---

### 15.2 Common Backup Strategies

Example strategy:

| Data Type | Snapshot | Backup |
|---|---|---|
| Test Data | Daily | Weekly |
| Regular Business | Every 6 Hours | Daily |
| Critical Business | Hourly | Every 4 Hours or Daily |
| Database | Combined with Database Own Backup | Longhorn Backup as Auxiliary |

Notes:

    Database consistency cannot rely solely on volume snapshots.
    MySQL, PostgreSQL, etc., should combine with their own backup mechanisms.
    Longhorn Backup can serve as volume-level recovery capability but cannot replace database logical backups.

---

### 15.3 Retention Planning

Retention strategy example:

    Retain 24 hourly Snapshots.
    Retain 7 daily Backups.
    Retain 4 weekly Backups.
    Retain 3 monthly Backups.

Production requires combining:

    Data Importance
    Storage Capacity
    Recovery Time Objective (RTO)
    Recovery Point Objective (RPO)
    Compliance Requirements
    Business Acceptable Data Loss Time

---

### 15.4 UI Create Recurring Job

Longhorn UI path:

    Recurring Jobs
      -> Create Recurring Job

Configurable options:

    Name
    Task: snapshot / backup
    Cron
    Retain
    Concurrency
    Labels

Example:

    Name: daily-backup
    Task: backup
    Cron: 0 2 * * *
    Retain: 7
    Concurrency: 1

Explanation:

    Indicates daily backup at 2:00 AM, retaining 7 copies.

---

### 15.5 Apply Recurring Job to Volume

Longhorn UI:

    Volume
      -> Select Volume
      -> Recurring Job
      -> Add Job

Alternatively, configure via StorageClass or Longhorn-related settings to automatically add recurring jobs to new volumes.

Production recommendations:

    Group by business importance.
    Different businesses use different backup strategies.
    Avoid backing up all volumes at the same time to prevent IO peaks.
    Backup windows should avoid business peak hours.

---

## SixteenI don't know.S3 Backup Target Production Notes

### 16.1 MinIO Minimal Permissions

Longhorn backup users should only access:

    longhorn-backup Bucket

Should not grant:

    s3:*
    Resource: *
    Administrator Permissions
    Other Business Bucket Permissions
    MinIO Root User Permissions

Recommendations:

    Independent AccessKey.
    Independent Policy.
    Regular Rotation.
    Do not commit to Git.
    Secret encrypted management.

---

### 16.2 S3 Endpoint Selection

Options:

    Internal HTTP Endpoint
    Internal HTTPS Endpoint
    External HTTPS Endpoint

Experimental environments can use:

    http://10.0.0.41:9000

Production recommendations:

    Use HTTPS for cross-network access.
    Internal HTTP must be within trusted networks and security boundaries.
    Do not give MinIO root account to Longhorn.
    Do not let Backup Target share the same failure nodes as Longhorn.

---

### 16.3 Bucket Lifecycle Policy

Longhorn typically manages backupstore lifecycle through its own.

Production should not directly use object storage lifecycle policies on Backup Targets to delete Longhorn-managed backup objects.

Risks:

    Longhorn metadata and actual objects become inconsistent.
    Backup list anomalies.
    Restore failure.
    Backup chain damage.

Recommendations:

    Manage backup retention through Longhorn Recurring Job's Retain strategy.
    Do not manually delete internal backupstore objects.
    Delete backups preferentially via Longhorn UI or API.

---

## SeventeenI don't know.NFS Backup Target Production Notes

### 17.1 NFS Single Point Issues

NFS as a single machine:

    NFS Server failure will cause backup failure.
    NFS disk damage will lead to data loss.
    NFS performance may become a bottleneck.
    Improper NFS permission configuration poses security risks.

Production recommendations:

    Use high availability NFS.
    Or prioritize more reliable object storage.
    NFS Backup Target also needs backup.
    NFS capacity requires monitoring.

---

### 17.2 NFS Permission Control

May be used in experiments:

    no_root_squash
    chmod 0777

Production does not recommend crude usage.

Should combine:

    Client subnet restrictions
    Minimum permissions
    Only allow K8s nodes access
    Firewall restrictions
    Audit
    NFS service monitoring

---

### 17.3 NFS Capacity Monitoring

Must monitor:

    /data/longhorn-backup capacity
    inode
    NFS service status
    NFS latency
    Backup failure
    Mount anomalies

Commands:

    df -hT /data/longhorn-backup
    exportfs -v
    systemctl status nfs-server

---

## EighteenI don't know.Database Scenario Backup Reminders

### 18.1 Volume-Level Backup Is Not Equivalent to Database Consistency Backup

Longhorn Backup is volume-level backup.

For databases, must consider:

    Database cache
    Unflushed transactions
    WAL / binlog
    Data file consistency
    Whether writing is happening during backup

Directly backing up a database volume with high write frequency may pose application consistency risks.

---

### 18.2 MySQL Recommendations

MySQL production backup recommendations:

    mysqldump
    xtrabackup
    binlog
    Master-slave replication
    Regular recovery drills

Longhorn Backup can serve as:

    Underlying volume-level supplemental protection
    Full directory recovery solution
    Disaster recovery assistance

But should not replace database-specific backups.

---

### 18.3 PostgreSQL Recommendations

PostgreSQL production backup recommendations:

    pg_dump
    pg_basebackup
    WAL archive
    PITR
    Regular recovery drills

Longhorn Backup serves the same role as a volume-level supplement.

---

## NineteenI don't know.Common Issue Troubleshooting

### 19.1 Backup Target Unavailable

Symptoms:

    Longhorn UI Backup page error.
    Backup creation failure.
    longhorn-manager logs indicate inability to access backupstore.

Troubleshooting:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200
    kubectl -n longhorn-system get settings.longhorn.io | grep -i backup
    kubectl -n longhorn-system get secret
    kubectl get nodes -o wide

NFS direction:

    showmount -e 10.0.0.10
    mount -t nfs4 10.0.0.10:/data/longhorn-backup /mnt/test
    exportfs -v
    systemctl status nfs-server

S3 direction:

    Check if Bucket exists.
    Check AccessKey / SecretKey.
    Check Policy for ListBucket, GetObject, PutObject, DeleteObject.
    Check if Endpoint is reachable.
    Check if HTTP / HTTPS configuration is incorrect.
    Check if self-signed certificate is trusted.

---

### 19.2 Backup Failure

Troubleshooting:

    Longhorn UI error messages
    longhorn-manager logs
    Volume health status
    Backup Target writability
    Target capacity sufficiency
    Network connectivity
    Secret correctness

Commands:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=300
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

---

### 19.3 Restore Failure

Troubleshooting:

    Backup completion status.
    Backup Target accessibility.
    Target cluster Longhorn health.
    Replica scheduling capability.
    Node disk capacity sufficiency.
    longhorn-manager logs for errors.

Commands:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=300

---

### 19.4 PVC Cannot Be Mounted After Restore

Troubleshooting:

    kubectl get pvc -n longhorn-backup-demo
    kubectl describe pvc restored-pvc -n longhorn-backup-demo
    kubectl describe pod restored-pod -n longhorn-backup-demo
    kubectl get events -A --sort-by=.lastTimestamp | tail -100
    systemctl status iscsid
    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Common causes:

    PVC not bound to correct PV.
    Volume not attached.
    iscsid anomaly.
    RWO volume occupied by other Pod.
    CSI component anomaly.

---

### 19.5 Backup Files In MinIO Are Not Readable

This is normal behavior.

Longhorn maintains its own backup objects and metadata in the backupstore.

Do not:

    Manually rename
    Manually move
    Manually delete partial objects
    Clean internal objects directly with mc rm
    Clean internal objects arbitrarily with object storage lifecycle

Should use:

    Longhorn UI
    Longhorn API
    Recurring Job Retain

To manage backup lifecycle.

---

## 20. Production Backup Strategy Design

### 20.1 By Business Tier

| Business Type | Backup Recommendation |
|---|---|
| Temporary Testing | Can only do Snapshot |
| Ordinary Application | Daily Backup |
| Middleware | Snapshot + Daily Backup |
| Database | Database own backup + Longhorn Backup |
| Critical Business | High-frequency backup +Alien. Backup Target + Recovery Drill |

---

### 20.2 RPO and RTO

RPO:

    Maximum data loss that can be accepted.

RTO:

    Maximum time to restore business.

Example:

    RPO 24 hours: daily backup once.
    RPO 1 hour: hourly backup once.
    RTO 2 hours: recovery process must complete within 2 hours.

Backup strategy must be designed around RPO/RTO, not just set daily backup arbitrarily.

---

### 20.3 Recommended Base Strategy

Ordinary Business:

    Daily Backup
    Retain 7 days
    Weekly recovery drillCheck.

More Important Business:

    Backup every 6 hours
    Retain 7 to 14 days
    Monthly recovery drill

Database Business:

    Database logical backup daily
    binlog/WAL continuous archiving
    Longhorn Backup daily or before change
    Regularly restore to temporary environment for verification

---

### 20.4 Backup Window Off-peak

Do not backup all Volumes at the same time.

Wrong Practice:

    All Volumes backup at 02:00 daily

May cause:

    Disk I/O peak
    Network peak
    Backup Target pressure peak
    Application latency increase

Recommend:

    Batch by business
    Off-peak execution
    Control concurrency
    Avoid business peak
    Monitor backup duration and failure rate

---

## 21. Recovery Drill Process

### 21.1 Why Must Recovery Drill

Backup without recovery drill has high risk.

May occur:

    Backup always fails but no one notices
    Backup Target writable but unreadable
    Secret expired
    No one knows recovery steps
    Backup object damage
    RTO far exceeds expectation
    Application cannot start after recovery

---

### 21.2 Standard Recovery Drill Process

Recommended process:

    1. Choose a non-production time window
    2. Choose a representative Volume
    3. Create Backup
    4. Restore from Backup to new Volume
    5. Create new PVC
    6. Create verification Pod
    7. Verify critical files
    8. Verify application can start
    9. Record recovery duration
    10. Clean up drill resources
    11. Output recovery drill report

---

### 21.3 Recovery Drill Report Template

| Item | Content |
|---|---|
| Drill Time |  |
| Business |  |
| Original PVC |  |
| Original Volume |  |
| Backup Time |  |
| Backup Target |  |
| Restore Volume |  |
| Recovery Duration |  |
| Data Verification | Pass / Fail |
| Application Verification | Pass / Fail |
| Issue Record |  |
| Improvement Plan |  |

---

## 22. High-risk Operation Reminder

The following operations must be cautious in production:

    Delete PVC
    Delete PV
    Delete Longhorn Volume
    Delete Snapshot
    Delete Backup
    Clear Backup Target
    Directly delete MinIO longhorn-backup Bucket
    Directly delete NFS /data/longhorn-backup directory
    Modify Backup Target
    Modify S3 Secret
    Directly rollback Snapshot for running database
    Do recovery test without backup
    Use root MinIO account as Longhorn Backup credential

Before execution, must confirm:

    Whether there is latest Backup
    Whether Backup is recoverable
    Whether current business is write-stop
    Whether there is maintenance window
    Whether there is business confirmation
    Whether there is rollback plan
    Whether there is recovery verification checklist
    Whether original Volume scene is preserved

---

## 23. Experimental Cleanup

### 23.1 Delete Recovery Verification Pod

Execute:

    kubectl delete -f 04-restored-pod.yaml --ignore-not-found=true

---

### 23.2 Delete Original Test Pod

Execute:

    kubectl delete -f 02-pod.yaml --ignore-not-found=true

---

### 23.3 Delete Recovery PVC

High-risk reminder:

    Deleting PVC may delete corresponding restored Volume
    Here only clean experimental resources

Execute:

    kubectl delete pvc restored-pvc -n longhorn-backup-demo --ignore-not-found=true

---

### 23.4 Delete Original PVC

Execute:

    kubectl delete -f 01-pvc.yaml --ignore-not-found=true

Check:

    kubectl get pvc -n longhorn-backup-demo
    kubectl get pv
    kubectl -n longhorn-system get volumes.longhorn.io

---

### 23.5 Delete Namespace

After confirming no resources:

    kubectl delete namespace longhorn-backup-demo

---

### 23.6 Whether to Delete Backup

During learning phase, can keep Backup for later recovery drill.

If need to delete:

    Longhorn UI
      -> Backup
      -> Delete corresponding Backup

High-risk reminder:

    Do not manually delete internal objects directly in MinIO or NFS backend
    Prefer to delete Backup through Longhorn UI/API

--- /think

### 23.7 Whether to Delete Backup Target

If it's an experimental NFS:

    You can retain /data/longhorn-backup for future experiments.

If cleanup is mandatory:

    systemctl stop nfs-server
    rm -rf /data/longhorn-backup/*

High-risk warning:

    This directory may contain all Longhorn backups.
    Direct rm -rf is prohibited in production environments.

If it's MinIO:

    Direct deletion of the Bucket is not recommended.
    If confirmed as experimental:

    mc rm --recursive --force minio-admin/longhorn-backup

---

### 23.8 Deleting Local YAML Files

Execute:

    rm -f 01-pvc.yaml
    rm -f 02-pod.yaml
    rm -f 03-longhorn-s3-secret.yaml
    rm -f 04-restored-pod.yaml
    rm -f longhorn-backup-policy.json

---

## Twenty-Four, Completion Criteria for This Article

After completing this article, the following should be at least achieved:

| Item | Standard |
|---|---|
| Test PVC | backup-demo-pvc successfully Bound |
| Test Pod | backup-demo-pod successfully mounts PVC |
| Test Data | Multiple types of files written under /data |
| Snapshot | Ability to create Snapshot |
| Snapshot Rollback | Ability to recover deleted files |
| Backup Target | At least one of NFS or S3 configured |
| Backup | Ability to successfully create Volume Backup |
| Backup Storage | Ability to see backup data in NFS or MinIO |
| Restore | Ability to restore new Volume from Backup |
| Restored PVC | restored-pvc mountable |
| Restored Pod | restored-pod readable of restored data |
| Data Verification | SHA256 checksums of key files consistent |
| Understanding of Policies | Clear boundaries of Replica, Snapshot, and Backup |
| Production Awareness | Clear backup strategy, recovery drills, and high-risk operations |

---

## Twenty-Five, Interview Answering Approach

If asked in an interview:

    How does Longhorn perform backup and recovery? What's the difference between Snapshot and Backup?

You can answer:

    Longhorn's data protection should distinguish between Replica, Snapshot, and Backup. Replica is a replication mechanism, mainly addressing high availability for node or disk failures; Snapshot is a local point-in-time snapshot of the Volume, suitable for quick rollback before changes; Backup is backing up Volume data to an external Backup Target, such as NFS or S3-compatible object storage, which is closer to true cross-failure-domain recovery capabilities.
    I would not treat Replica as backup, because when application deletes files, deletes PVC, or writes corrupted data, deletion or errors will synchronize to all replicas. Snapshot cannot fully replace Backup, because Snapshot typically still relies on the Longhorn cluster itself. If the entire cluster or Volume fails, we need to rely on external Backup Target for recovery.
    When configuring Backup Target, you can use NFS or S3-compatible storage like MinIO. NFS is more intuitive, but consider NFS's own high availability and capacity; S3/MinIO is more suitable as object-based backup targets. In production, I would create a dedicated MinIO Bucket and minimal-permission user for Longhorn, not using MinIO root account, and store AccessKey, SecretKey, and Endpoint via Kubernetes Secret.
    In the backup process, I would first create PVC and Pod, write test data, then create Snapshot, and create Backup from Snapshot. After Backup succeeds, I would restore a new Longhorn Volume from Backup, then create new PVC and verify Pod mounting it, checking if key files and checksums are consistent. Only when Backup successfully recovers, the backup loop is considered usable.
    For database applications, I would emphasize that Longhorn Backup is volume-level backup, which may not guarantee database application consistency. Production MySQL/PostgreSQL still requires combining mysqldump, xtrabackup, pg_dump, WAL/binlog, etc., database-specific backup mechanisms. Longhorn Backup can serve as a supplementary volume-level protection rather than replacing database backups.
    In production, recurring jobs should be set for regular Snapshot and Backup, and regular recovery drills should be conducted, monitoring for Backup failures to avoid backups that cannot be recovered.

---

## Twenty-Six, Summary of This Article

This article completes the learning of Longhorn backup and recovery:

1. Replica is high availability capability, not backup.
2. Snapshot is a local point-in-time snapshot, suitable for quick rollback.
3. Snapshot cannot fully replace external backup.
4. Backup is backing up to external Backup Target.
5. Backup Target can use NFS.
6. Backup Target can use S3-compatible object storage.
7. MinIO can serve as Longhorn S3 Backup Target.
8. Longhorn Backup users should use minimal permissions.
9. MinIO root user should not be used as backup credentials.
10. AccessKey / SecretKey in Secret should not be submitted to public Git.
11. Deleted files will synchronize to all Replica.
12. Data recovery should rely on Snapshot or Backup.
13. When restoring from Backup, it's recommended to restore as a new Volume.
14. After restoration, new PVC and new Pod should be created to verify data.
15. Before production switch, stop writes, verify, then switch.
16. Database scenarios cannot rely solely on volume-level backup.
17. MySQL / PostgreSQL still require database-specific backup mechanisms.
18. Recurring Job can be used for regular Snapshot and Backup.
19. Backup must undergo recovery drills; otherwise, it's not a complete loop.
20. The next article will learn Longhorn troubleshooting: Volume Degraded, Replica rebuilding, and node anomalies.

---

## Twenty-Seven, Reference Documents

Longhorn official documentation:

    https://longhorn.io/docs/latest/

Longhorn Snapshot and Backup:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn Backup and Restore:

    https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/

Longhorn Setting a Backup Target: /think

https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/set-backup-target/

Longhorn Recurring Snapshot and Backup:

    https://longhorn.io/docs/latest/snapshots-and-backups/scheduling-backups-and-snapshots/

Longhorn Disaster Recovery Volume:

    https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/create-disaster-recovery-volume/

Longhorn StorageClass Parameters:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Longhorn Troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Secrets:

    https://kubernetes.io/docs/concepts/configuration/secret/

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Policy Management: