# Longhorn Backup and Recovery: Backup Targets, Snapshots, and Volume Restoration

Recommended Path: 05-Storage/03-LongHorn/07-Longhorn Backup and Recovery: Backup Targets, Snapshots, and Volume Restoration.md

Tags: #Longhorn #BackupTarget #Snapshot #Backup #Restore #S3 #MinIO #NFS #Kubernetes #PVC #Data Recovery #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the seventh in the Longhorn series, focusing on learning about Longhorn’s snapshotting, backup, recovery, and Backup Target design.

Previously covered topics include:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- Longhorn Helm Installation Methodology: Charts, Images, values.yaml, and Version Management
- Longhorn Dynamic Volume Practices: PVCs, PVs, Pod Mounting, and Data Persistence
- Longhorn Replication Mechanisms: Number of Replicas, Node Distribution, and Data High Availability

This article delves into the most critical aspects of Longhorn data protection:

    What is a Snapshot?
    What is a Backup?
    What are the differences between a Snapshot and a Backup?
    Where should a Backup Target be located?
    Why isn’t a Replica considered a backup?
    Why can’t a Snapshot completely replace a backup?
    How to use NFS as a Backup Target?
    How to use MinIO/S3 as Backup Targets?
    How to back up Longhorn volumes?
    How to restore volumes from backups?
    How to verify the integrity of restored data?
    How to design a backup strategy for production environments?
    How to prevent irreversible damage to PVCs?

This article emphasizes practicality and will demonstrate Longhorn’s data recovery capabilities through real-world examples involving PVCs, Pods, snapshots, backups, and restorations.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the purpose of Longhorn snapshots.
2. Understand the purpose of Longhorn backups.
3. Distinguish between snapshots and backups.
4. Comprehend the role of Backup Targets.
5. Clarify the boundaries between replicas, snapshots, and backups.
6. Configure NFS as a backup target.
7. Set up S3/MinIO as backup targets.
8. Create test PVCs and Pods.
9. Write test data.
10. Generate Longhorn snapshots.
11. Roll back data using snapshots.
12. Perform Longhorn backups.
13. Simulate data loss scenarios.
14. Restore volumes from backups.
15. Rebind restored volumes to Pods for use.
16. Verify the integrity of restored data.
17. Design regular backup routines.
18. Recognize the importance of disaster recovery drills in production environments.

---

## III. Key Conclusions

### 3.1 Replicas Are Not Backups

Longhorn replicas are designed to address:

    Node failures
    Disk failures
    Single-replica anomalies
    Volume availability issues
    Replica reconfiguration processes

However, they cannot prevent:

    User-induced file deletions
    Application-generated data corruption
    Accidental deletion of PVCs
    Administrator-made volume deletions
    Total destruction of the Kubernetes cluster
    Simultaneous loss of all replicas

Conclusion:

    Replicas enhance high availability but are not considered backups.

---

### 3.2 Snapshots Are Not Complete Backups

Snapshots serve to:

    Preserve the state of a volume at a specific point in time
    Enable quick rollbacks
    Facilitate rapid recovery from accidental operations
    Provide short-term data protection

However, snapshots typically rely on the same storage system as the original volume. In scenarios such as:

    Total collapse of the Longhorn cluster
    Severe damage to node data disks
    Deletion of the volume or its snapshot
    Unavailability of the entire Kubernetes cluster

snapshots may not be sufficient for independent recovery.

Conclusion:

    Snapshots provide local protection at a specific moment but are not suitable for off-site backup purposes. They cannot replace Backup Targets.

---

### 3.3 Backups Are the Key to Cross-Fault-Domain Recovery

Backups involve copying volume data to external targets. These targets can include:

    NFS
    S3-compatible object storage
    MinIO
    SMB/CIFS
    Cloud object storage services

This article focuses on two commonly used backup methods:

    NFS Backup Targets
    MinIO/S3 Backup Targets

In production environments, it is recommended to:

    Keep the backup target separate from the Kubernetes cluster and Longhorn data disks.
    Place the backup target in a different fault domain.
    Conduct regular recovery drills.

---

## IV. Experimental Environment

### 4.1 Kubernetes Cluster

Default experimental setup:

    Kubernetes:        Backup Target
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

## VI. Pre-Operation Checks

### 6.1 Checking Longhorn Components

Execute:

    kubectl -n longhorn-system get pods -o wide

Check for any exceptions:

    kubectl -n longhorn-system get pods | grep -Ev "Running|Completed"

If there are any issues:

    kubectl -n longhorn-system describe pod <pod-name>
    kubectl -n longhorn-system logs <pod-name> --tail=100
    kubectl get events -n longhorn-system --sort-by=.lastTimestamp | tail -100

---

### 6.2 Checking StorageClass

Execute:

    kubectl get sc
    kubectl describe sc longhorn

Confirm:

    The longhorn StorageClass exists.
    The provisioner is driver.longhorn.io.
    PVCs can be created dynamically without issues.

---

### 6.3 Checking Volume Status

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Requirements:

    There should be no Volume in a state of Degraded.
    No Faulty Volumes are allowed.
    A large number of Replica anomalies are not acceptable.
    It is not recommended to perform backup and recovery experiments when storage is abnormal.

---

### 6.4 Checking Node Status

Execute:

    kubectl get nodes -o wide
    kubectl describe nodes | grep -E "Name:|DiskPressure|MemoryPressure|PIDPressure|Ready"

Requirements:

    The status should be Ready=True.
    There should be no DiskPressure, MemoryPressure, or PIDPressure issues.

---

## VII. Practical Operation 1: Creating a Test PVC and Pod

### 7.1 Creating a Namespace

Execute:

    kubectl create namespace longhorn-backup-demo

Verify:

    kubectl get ns longhorn-backup-demo

---

### 7.2 Creating a Working Directory

Execute:

    mkdir -p ~/longhorn-backup-demo
    cd ~/longhorn-backup-demo

---

### 7.3 Creating a PVC

Create the file:

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

Apply the configuration:

    kubectl apply -f 01-pvc.yaml

Verify the creation:

    kubectl get pvc -n longhorn-backup-demo
    kubectl get pv

---

### 7.4 Creating a Pod with the PVC Mounted

Create the file:

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

Apply the configuration:

    kubectl apply -f 02-pod.yaml

Verify the creation:

    kubectl get pod -n longhorn-backup-demo -o wide

If the image pull fails, you can first synchronize the busybox image to your own repository and then replace it in the Pod definition.

---

### 7.5 Writing Test Data

Write a basic file:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "echo 'longhorn backup restore demo' > /data/hello.txt"

Create a directory structure:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "mkdir -p /data/app/config /data/app/logs /data/db"

Write configuration settings:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "echo 'app_name=longhorn-backup-demo' > /data/app/config/app.conf"

Record logs:

    kubectl exec -n longhorn-backup-demo backup-demo-pod -- sh -c "echo 'log line before snapshot' > /data/app/logs/app.log"

Write a simulated database file:

    kubectl exec -n longhorn### 10.1 Why Recommend S3 Backup TargetsAdvantages of Using S3 / MinIO as Backup Targets:

- More suitable for object-based data backup.
- No need to mount NFS directories on each node.
- Better for cross-cluster recovery.
- Easier to access across nodes, data centers, and platforms.
- MinIO can be used as a private object storage backup target.
- Cloud providers' S3 / OSS can serve as off-site backup targets.

Production Recommendations:

- Back up data within the Longhorn cluster to external S3 / MinIO locations.
- Avoid placing both MinIO and critical Longhorn data on the same set of faulty nodes.
- Ensure that backup targets have independent availability.

---

### 10.2 Creating a MinIO Bucket

After configuring the MinIO alias in the mc client, execute the following command:

    mc mb minio-admin/longhorn-backup

If using the Docker version of mc:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mb minio-admin/longhorn-backup

To check the configuration:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-admin

---

### 10.3 Creating a MinIO Access User and Policy

It is recommended to create a dedicated MinIO user for Longhorn, rather than using the root account.

User:

    longhorn-backup-user

Password:

    LonghornBackup@123456

To create the user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user add minio-admin longhorn-backup-user 'LonghornBackup@123456'

To create the Policy file:

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

To apply the Policy:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v $(pwd):/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy create minio-admin longhorn-backup-policy /demo/longhorn-backup-policy.json

To bind the Policy:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy attach minio-admin longhorn-backup-policy --user longhorn-backup-user

To verify the configuration:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user info minio-admin longhorn-backup-user

---

### 10.4 Creating a Longhorn S3 Secret

The Longhorn S3 backup target requires a Kubernetes Secret to store access credentials.

Create the file:

    cat > 03-longhorn-s3-secret.yaml <<'EOF'
    apiVersion: v1
    kind: Secret
    metadata:
      name: longhorn-s3-backup-secret
      namespace: longhorn-system
    type: Opaque
    stringData:
     ```bash
kubectl -n longhorn-system get settings.longhorn.io backup-target-credential-secret -o yaml
```

Note:

The specific setting name may vary depending on the current version. New versions might use the BackupTarget CRD for management. You can check this by executing the following command:

```bash
kubectl api-resources | grep -i backup
```          imagePullPolicy: IfNotPresent
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

Application:

    kubectl apply -f 04-restored-pod.yaml

View:

    kubectl get pod -n longhorn-backup-demo -o wide

---

### 13.6 Verify Restored Data

Execute:

    kubectl exec -n longhorn-backup-demo restored-pod -- find /restore -maxdepth 4 -type f -print

View key files:

    kubectl exec -n longhorn-backup-demo restored-pod -- cat /restore/hello.txt
    kubectl exec -n longhorn-backup-demo restored-pod -- cat /restore/db/users.csv
    kubectl exec -n longhorn-backup-demo restored-pod -- cat /restore/app/config/app.conf
    kubectl exec -n longhorn-backup-demo restored-pod -- ls -lh /restore/file-100m.bin

Calculate verification:

    kubectl exec -n longhorn-backup-demo restored-pod -- sha256sum /restore/hello.txt
    kubectl exec -n longhorn-backup-demo restored-pod -- sha256sum /restore/db/users.csv
    kubectl exec -n longhorn-backup-demo restored-pod -- sha256sum /restore/file-100m.bin

Compare with records before backup.

---

### 13.7 Recovery Conclusion

If the data is intact after recovery:

    The Backup Target is usable.
    The Backup process is functional.
    The Restore procedure works correctly.
    The restored Volume can be re-mounted.
    The data can be accessed by business pods.

Only then has the entire backup cycle been successfully completed.

Creating a Backup without verifying the restoration process does not constitute a fully operational backup capability.

---

## Chapter Fourteen: Practical Exercise Eight: Switching Business Operations After Data Recovery from a Backup

### 14.1 Principles for Production Switching

In production environments, it is not recommended to directly overwrite existing data during recovery.

The recommended procedure is as follows:

    1. Restore the data to a new Volume.
    2. Create a new Persistent Volume Claim (PVC).
    3. Set up a verification Pod.
    4. Verify the files and business data.
    5. Stop any writing operations on the original data.
    6. Configure the business workload to use the new PVC.
    7. Restart the service.
    8. Conduct tests to ensure the service is functioning correctly.
    9. Retain the original PVC for a certain period of time.
    10. Conduct a post-recovery review.

---

### 14.2 Example of Switching PVC in a Deployment

Assume that the original Deployment uses:

    backup-demo-pvc

After recovery, the new PVC will be:

    restored-pvc

To update the Deployment's volume settings, modify the `persistentVolumeClaim` section as follows:

    persistentVolumeClaim:
      claimName: restored-pvc

Then execute the following commands:

    kubectl apply -f deployment.yaml
    kubectl rollout status deploy/<deploy-name> -n <namespace>

Production notes:

    Ensure that all writing operations are stopped before making the switch.
    For database applications, perform consistency checks in conjunction with the database system.
    Do not attempt to switch volumes while data is still being written to the old volume.
    Keep the old volume available in case of a rollback.

---

### 14.3 Considerations for Switching StatefulSets

When using `volumeClaimTemplates` with StatefulSets, switching PVCs becomes more complex.

Key factors to consider include:

    Pod names.
    PVC names.
    Rules generated by `volumeClaimTemplates`.
    PV binding relationships.
    Data consistency.
    Whether it is necessary to rebuild the StatefulSet.
    Whether it is possible to restore using a PVC with the same name.

In production, any changes to StatefulSets should be carefully planned and tested before implementation. Manual adjustments should be avoided whenever possible.

---

## Chapter Fifteen: Recurring Jobs: Scheduled Snapshots and Backups

### 15.1 Why Repeating Jobs Are Needed

Manual backups are unreliable.

In a production environment, it is essential to have:

    Automated scheduled snapshots.
    Automated scheduled backups.
    Automatic retention of a specified number of backups.
    Automatic deletion of old backups.
    Alerts in case of backup failures.
    Regular recovery drills.

Longhorn allows users to configure Repeating Jobs through its user interface or configuration files.

---

### 15.2 Common Backup Strategies

Example strategies include:

| Data Type | Snapshot Frequency### 17.1 NFS Single-Point Issues

When NFS is used on a single machine:

    A failure of the NFS Server can cause backup failures.
    Damage to the NFS disk can result in lost backups.
    NFS performance may become a bottleneck.
    Improper NFS permission settings pose security risks.

Production recommendations:

    Use highly available NFS.
    Alternatively, prefer more reliable object storage solutions.
    The NFS Backup Target also needs to be backed up regularly.
    Monitor the NFS capacity regularly.

---

### 17.2 NFS Permission Control

In experiments, the following options may be used:

    no_root_squash
    chmod 0777

However, these should not be used recklessly in production environments.

Instead, consider combining them with:

    Client IP range restrictions
    Minimum permissions setting
    Access limited to only K8s nodes
    Firewall controls
    Auditing measures
    NFS service monitoring

---

### 17.3 NFS Capacity Monitoring

It is essential to monitor the following:

    The capacity of /data/longhorn-backup
    The number of inodes
    The status of the NFS service
    NFS latency
    Backup failure occurrences
    Any mount-related exceptions

Commands for monitoring:

    df -hT /data/longhorn-backup
    exportfs -v
    systemctl status nfs-server

---

## Section Eighteen: Database Backup Considerations

### 18.1 Volume-Level Backup Does Not Ensure Database Consistency

Longhorn Backup performs volume-level backups.

For databases, additional considerations are necessary:

    Database caches
    Unflushed transactions
    WAL / binlog files
    Data file consistency
    Whether the database is currently being written to

Directly backing up a database volume that is under high-frequency writing can lead to application consistency issues.

---

### 18.2 MySQL Recommendations

For MySQL in production environments, the following backup methods are recommended:

    mysqldump
    xtrabackup
    Binlog files
    Master-slave replication
    Regular recovery drills

Longhorn Backup can serve as:

    An additional layer of volume-level protection
    A comprehensive directory restoration solution
    An aid in disaster recovery efforts

However, it should not replace the database's own backup mechanisms.

---

### 18.3 PostgreSQL Recommendations

For PostgreSQL in production environments, the following backup methods are recommended:

    pg_dump
    pg_basebackup
    WAL archive files
    PITR (Point-In-Time Recovery)
    Regular recovery drills

Similar to MySQL, Longhorn Backup should only be used as an additional volume-level backup option.

---

## Section Nineteen: Troubleshooting Common Issues

### 19.1 Unavailable Backup Target

Symptoms:

    Errors appear on the Longhorn UI's backup page.
    Backup creation fails.
    The longhorn-manager log indicates that access to the backupstore is denied.

Troubleshooting steps:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200
    kubectl -n longhorn-system get settings.longhorn.io | grep -i backup
    kubectl -n longhorn-system get secret
    kubectl get nodes -o wide

For NFS-based targets:

    showmount -e 10.0.0.10
    mount -t nfs4 10.0.0.10:/data/longhorn-backup /mnt/test
    exportfs -v
    systemctl status nfs-server

For S3-based targets:

    Verify if the Bucket exists.
    Check the AccessKey and SecretKey.
    Ensure that the Policy allows ListBucket, GetObject, PutObject, and DeleteObject operations.
    Confirm that the Endpoint is reachable.
    Check for any configuration errors related to HTTP/HTTPS.
    Verify if self-signed certificates are trusted.

---

### 19.2 Backup Failure

Troubleshooting steps:

    Check the error messages displayed on the Longhorn UI.
    Review the longhorn-manager logs.
    Verify if the Volume is in a healthy state.
    Confirm whether the Backup Target is writable.
    Ensure that there is sufficient target capacity.
    Check if the network connection is stable.
    Verify if the Secret values are correct.

Commands for troubleshooting:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=300
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

---

### 19.3 Restore Failure

Troubleshooting steps:

    Verify if the backup process was completed successfully.
    Check if the Backup Target is accessible.
    Confirm that the target Longhorn cluster is functioning properly.
    Ensure that enough replicas are available for restoration.
    Verify if the node disk capacity is sufficient.
    Review the longhorn-manager logs for any error messages.

Commands| Original Volume | |
| Backup Time | |
| Backup Target | |
| Restore Volume | |
| Restoration Time | |
| Data Verification | Passed / Failed |
| Application Verification | Passed / Failed |
| Issues Recorded | |
| Further Improvements | |

---

## Section 22: High-Risk Operation Warnings

The following operations must be performed with caution in a production environment:

    Deleting PVCs
    Deleting PVs
    Deleting Longhorn Volumes
    Deleting Snapshots
    Deleting Backups
    Clearing the Backup Target
    Directly deleting the MinIO longhorn-backup Bucket
    Directly deleting the NFS /data/longhorn-backup directory
    Modifying the Backup Target
    Modifying the S3 Secret
    Directly rolling back a Snapshot for a running database
    Conducting recovery tests without a backup
    Using the root MinIO account as Longhorn Backup credentials

Before proceeding, it is essential to confirm:

    Whether there is the latest backup.
    Whether the backup can be restored.
    Whether current operations have been paused.
    Whether there is a maintenance window available.
    Whether business stakeholders have given approval.
    Whether a rollback plan is in place.
    Whether a recovery verification checklist exists.
    Whether the original Volume remains intact.

---

## Section 23: Experimental Cleanup

### 23.1 Deleting the Recovery Verification Pod

Execute:

    kubectl delete -f 04-restored-pod.yaml --ignore-not-found=true

---

### 23.2 Deleting the Original Test Pod

Execute:

    kubectl delete -f 02-pod.yaml --ignore-not-found=true

---

### 23.3 Deleting the Recovery PVC

High-Risk Warning:

    Deleting a PVC may result in the deletion of the corresponding recovery Volume.
    This step is only for clearing up experimental resources.

Execute:

    kubectl delete pvc restored-pvc -n longhorn-backup-demo --ignore-not-found=true

---

### 23.4 Deleting the Original PVC

Execute:

    kubectl delete -f 01-pvc.yaml --ignore-not-found=true

Verify:

    kubectl get pvc -n longhorn-backup-demo
    kubectl get pv
    kubectl -n longhorn-system get volumes.longhorn.io

---

### 23.5 Deleting the Namespace

After confirming there are no remaining resources:

    kubectl delete namespace longhorn-backup-demo

---

### 23.6 Whether to Delete the Backup

During the learning phase, it is acceptable to retain the backup for subsequent recovery exercises.

If you decide to delete it:

    Through Longhorn UI:
      -> Backup
      -> Delete the corresponding backup

High-Risk Warning:

    Do not manually delete internal objects directly on the MinIO or NFS backend.
    Preferably use the Longhorn UI/API to delete backups.

---

### 23.7 Whether to Delete the Backup Target

If it is an experimental NFS environment:

    You can keep the /data/longhorn-backup directory for future experiments.

If you must clean it up:

    systemctl stop nfs-server
    rm -rf /data/longhorn-backup/*

High-Risk Warning:

    This directory may contain all Longhorn backups.
    Direct deletion using rm -rf is prohibited in a production environment.

If it is MinIO:

    It is not recommended to directly delete the Bucket.
    If it is for experimental purposes only:

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

## Section 24: Completion Standards for This Guide

After completing this guide, you should have achieved at least the following:

| Item | Standard |
|---|---|
| Test PVC | The backup-demo-pvc is successfully Bound. |
| Test Pod | The backup-demo-pod is successfully mounted with the PVC. |
| Test Data | Multiple types of files are written to the /data directory. |
    Snapshot Creation | Snapshots can be created. |
    Snapshot Rollback | Deleted files can be recovered from snapshots. |
    Backup Target | At least one of NFS or S3-compatible object storage is configured. |
    Backup Creation | Volume backups can be successfully created. |
    Backup Storage Verification | Backup data is visible in NFS or MinIO. |
    Recovery | A new Longhorn Volume can be restored from the backup. |
    PVC Recovery | The restored-pvc can be mounted successfully. |
    Pod Recovery | The restored-pod can read the recovered data. |
    Data Verification | The sha256sums of key files match. |
    Policyhttps://longhorn.io/docs/latest/

Longhorn Snapshots and Backups:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn Backup and Restore:

    https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/

Setting a Backup Target for Longhorn:

    https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/set-backup-target/

Recurring Snapshots and Backups with Longhorn:

    https://longhorn.io/docs/latest/snapshots-and-backups/scheduling-backups-and-snapshots/

Disaster Recovery Volume for Longhorn:

    https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/create-disaster-recovery-volume/

StorageClass Parameters for Longhorn:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Troubleshooting with Longhorn:

    https://longhorn.io/kb/troubleshooting/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Secrets:

    https://kubernetes.io/docs/concepts/configuration/secret/

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

Managing MinIO Policies:

    https://min.io/docs/minio/linux/administration/identity-access-management/policy-based-access-control.html