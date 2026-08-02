# Practical Exercises with Longhorn Dynamic Volumes: PVC, PV, Pod Mounting, and Data Persistence

Recommended Path: 05-Storage/03-LongHorn/05-Practical Exercises with Longhorn Dynamic Volumes: PVC, PV, Pod Mounting, and Data Persistence.md

Tags: #Longhorn #Kubernetes #PVC #PV #StorageClass #CSI #Pod Mounting #Data Persistence #StatefulSet #Block Storage #Advanced SRE #Production Operations

---

## I. Document Introduction

This article is the fifth part of the Longhorn module, focusing on verifying the capabilities of Longhorn dynamic volumes through actual Kubernetes resources.

What has been covered previously includes:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- Longhorn Helm Installation Methodology: Charts, Images, values.yaml, and Version Management

This article now delves into the core practical operations of Longhorn:

    - Creating a PVC
    - Observing the automatic creation of PVs
    - Creating a Pod to mount the PVC
    - Writing data to the mounted directory
    - Deleting the Pod
    - Rebuilding the Pod
    - Verifying that the data still exists
    - Checking the status of Longhorn Volume, Engine, and Replica objects
    - Understanding the differences between Deployments and StatefulSets using Longhorn
    - Mastering basic troubleshooting methods for PVC Pending, Pod mounting failures, and Volume Degradation

The focus of this article is not to "write a YAML file" but to understand the entire process:

    PVC -> StorageClass -> CSI -> Longhorn Volume -> PV -> Pod Mounting -> Data Persistence

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Create a PVC using the Longhorn StorageClass.
2. Understand how a PVC triggers the creation of a dynamic PV.
3. Comprehend the relationship between PVCs, PVs, and Longhorn Volumes.
4. Create a Pod to mount a Longhorn PVC.
5. Write data to the mounted directory within the Pod.
6. Verify that the PVC and Volume still exist after deleting the Pod.
7. Confirm that the data remains intact after rebuilding the Pod.
8. Check the status of Longhorn Volume, Engine, and Replica objects.
9. Understand the mounting restrictions for RWO volumes.
10. Differentiate between how Deployments and StatefulSets use PVCs.
11. Be able to troubleshoot issues with PVC Pending.
12. Identify and resolve problems related to long-duration ContainerCreating in Pods.
13. Diagnose and fix errors such as FailedMount and FailedAttachVolume.
14. Understand the potential risks associated with deleting PVCs from a production environment.
15. Lay a foundation for understanding replication mechanisms, backup and recovery processes, and fault troubleshooting.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

    - Kubernetes: Kubeadm cluster
    - Operating system: Ubuntu Server 22.04.5 LTS
    - Container runtime: containerd
    - CNI: Calico
    - Node IP range: 10.0.0.0/24

Node configuration:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane |
| 10.0.0.21 | k8s-worker01 | Worker |
| 10.0.0.22 | k8s-worker02 | Worker |

---

### 3.2 Longhorn Environment

Longhorn namespace:

    longhorn-system

Longhorn StorageClass:

    longhorn

Data directory configuration for Longhorn:

    /data/longhorn

Viewing commands:

    kubectl -n longhorn-system get pods -o wide
    kubectl get sc
    kubectl -n longhorn-system get volumes.longhorn.io

---

### 3.3 Experimental Namespace

The experimental namespace used in this article is:

    longhorn-volume-demo

All test PVCs, Pods, Deployments, and StatefulSets will be placed within this namespace.

---

## IV. The Complete Process of Dynamic Volumes

### 4.1 Longhorn Dynamic Volume Process

Complete sequence:

    - The user creates a PVC.
        |
        v
    - The PVC specifies the storageClassName: longhorn.
        |
        v
    - Kubernetes locates the Longhorn StorageClass.
        |
        v
    - The CSI Provisioner calls Longhorn.
        |
        v
    - Longhorn creates a Volume.
        |
        v
    - Kubernetes automatically generates a PV.
        |
        v
    - The PVC is bound```markdown
kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker01
kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker02

Key Points to Note:

    Allow Scheduling
    Conditions
    Disk Path
    Storage Available
    Storage Scheduled
    Storage Maximum

Node-side Checks:

    df -hT /data/longhorn
    lsblk -f

---

## Section 6: Practical Operation 1: Creating an Experimental Namespace

### 6.1 Creating a Namespace

Execution:

    kubectl create namespace longhorn-volume-demo

Verification:

    kubectl get ns longhorn-volume-demo

---

### 6.2 Creating an Experimental Directory

Create a YAML directory on the management node:

    mkdir -p ~/longhorn-volume-demo
    cd ~/longhorn-volume-demo

---

## Section 7: Practical Operation 2: Creating a PVC and Observing Automatic PV Creation

### 7.1 Creating PVC YAML

Create a file:

    cat > 01-pvc-rwo.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: data-pvc
      namespace: longhorn-volume-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn
      resources:
        requests:
          storage: 1Gi
    EOF

Explanation:

    accessModes: ReadWriteOnce indicates that the volume can usually only be mounted by one node for read and write operations.
    storageClassName: longhorn specifies that Longhorn should dynamically create the volume.
    storage: 1Gi means requesting a capacity of 1 GiB.

---

### 7.2 Applying the PVC

Execution:

    kubectl apply -f 01-pvc-rwo.yaml

Verification of the PVC:

    kubectl get pvc -n longhorn-volume-demo

Expected Output:

    NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
    data-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO            longhorn

---

### 7.3 Viewing PVC Details

Execution:

    kubectl describe pvc data-pvc -n longhorn-volume-demo

Key Points to Note:

    Status
    Volume
    StorageClass
    Capacity
    Access Modes
    Events

If the status is Bound, it means the PVC has been bound to a PV.

If the status is Pending, check the Errors in the Events section.

---

### 7.4 Viewing the PV

Execution:

    kubectl get pv

Find the corresponding PV:

    kubectl get pv | grep data-pvc

View details:

    kubectl describe pv <pv-name>

Key Points to Note:

    StorageClass
    Claim
    Reclaim Policy
    Status
    CSI Driver
    VolumeHandle
    VolumeAttributes

Explanation:

    PV is a persistent volume object at the Kubernetes level.
    Longhorn Volume is the actual data volume managed internally by Longhorn.
    The two are associated through the CSI field of the PV.

---

### 7.5 Viewing the Longhorn Volume

Execution:

    kubectl -n longhorn-system get volumes.longhorn.io

Alternatively:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

View details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Key Points to Note:

    State
    Robustness
    Size
    Number Of Replicas
    Current Node
    Kubernetes Status
    Conditions

Expected Output:

    The state may be detached or attached.
    If no Pod is using the PVC, the Volume is usually in a detached state.
    When a Pod mounts it, the Volume will attach to the node where the Pod is located.

---

## Section 8: Practical Operation 3: Creating a Pod with a Mounted PVC

### 8.1 Creating Pod YAML

Create a file:

    cat > 02-pod-with-pvc.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: pvc-test-pod
      namespace: longhorn-volume-demo
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
            claimName: data-pvc
    EOF

Explanation:

    The Pod mounts the data-pvc volume to the /data directory inside the container.
    Files can be written to /data later on.
    If the busybox image fails```markdown
kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "date > /data/create-time.txt"

Write node information:

kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "hostname > /data/pod-hostname.txt"

View contents:

kubectl exec -n longhorn-volume-demo pvc-test-pod -- ls -lah /data
kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/hello.txt
kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/create-time.txt
kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/pod-hostname.txt

---

### 9.3 Create multi-level directories and files

Execute the following command:

kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "mkdir -p /data/app/logs /data/app/config"

Write the configuration file:

kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "echo 'app_name=longhorn-demo' > /data/app/config/app.conf"

Write logs:

kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "echo 'log line 1' > /data/app/logs/app.log"

View files:

kubectl exec -n longhorn-volume-demo pvc-test-pod -- find /data -maxdepth 4 -type f -print

---

### 9.4 Check the status of Longhorn Volume

Execute the following command:

kubectl -n longhorn-system get volumes.longhorn.io -o wide

View details:

kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Pay attention to:

Current Node
State
Robustness
Number Of Replicas
Kubernetes Status

Expected results:

The Volume is attached to the node where the Pod is located.
The Robustness should be healthy or in a similar healthy state.

---

### 9.5 Check the Replica distribution

Execute the following command:

kubectl -n longhorn-system get replicas.longhorn.io -o wide

View details:

kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

Pay attention to:

Node ID
Disk ID
Data Path
State
Healthy At

Explanation:

After data is written to /data, Longhorn will store it in the Replica through the Engine.
The actual data of the Replica is located in the data directory of the Longhorn node. It is not recommended to modify it manually.

---

## Section 10: Practical Exercise 5: Verify Data Remains After Deleting a Pod

### 10.1 Delete the Pod

Execute the following command:

kubectl delete pod pvc-test-pod -n longhorn-volume-demo

View results:

kubectl get pod -n longhorn-volume-demo
kubectl get pvc -n longhorn-volume-demo
kubectl get pv

Expected results:

The Pod has been deleted.
The PVC still exists.
The PV still exists.
The Longhorn Volume still exists.

---

### 10.2 Check the Volume status

Execute the following command:

kubectl -n longhorn-system get volumes.longhorn.io -o wide

Explanation:

After the Pod is deleted, the Volume may change from attached to detached.
However, the data remains in the Longhorn Volume.
Deleting a Pod does not mean deleting the PVC or the Longhorn Volume.

---

### 10.3 Re-create the Pod

Reapply the Pod YAML:

kubectl apply -f 02-pod-with-pvc.yaml

View results:

kubectl get pod -n longhorn-volume-demo -o wide

Wait for the Pod to start running.

---

### 10.4 Verify that old data still exists

Execute the following commands:

kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/hello.txt
kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/create-time.txt
kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/pod-hostname.txt
kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/app/config/app.conf
kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/app/logs/app.log

Expected results:

All files still exist.

Conclusion:

After the Pod is deleted and re-created, the data is not lost.
The data is stored in the Longhorn Volume associated with the PVC.
A Pod is merely a running instance that uses the Volume; it does not represent the actual data itself.

---

## Section 11: Practical Exercise 6: Delete a Pod and Reschedule It to Another Node

### 11.1 Why Verify Cross-Node Operations

One of the key benefits of Longhorn is that:

Even if a Pod is rescheduled to another node, it can still mount the original Longhorn Volume.

However, it is important to note```markdown
kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/app/config/app.conf

Expected result:

The data should still be readable.

Check the current status of the Longhorn Volume node:

kubectl -n longhorn-system get volumes.longhorn.io -o wide
kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Explanation:

The volume should be attached to the new Pod on that node. If the attachment fails, carefully check the iscsid, CSI, Longhorn Engine status, node status, and any related events.
---

## Section Twelve: Practical Exercise Seven: Using a Deployment to Mount the Same RWO PVC

### 12.1 Risks of Using a PVC with a Deployment

Deployments are generally suitable for stateless applications.

If a Deployment uses an RWO PVC, the following precautions should be taken:

- The number of replicas should be set to 1 only.
- Sharing the same RWO PVC among multiple replica Deployments may cause scheduling and mounting conflicts.
- An RWO volume cannot be mounted simultaneously for reading and writing by multiple nodes.
- For stateful applications such as databases or middleware, it is recommended to use StatefulSets instead.

---

### 12.2 Delete the Current Test Pod

Execute the following command:

kubectl delete pod pvc-test-pod -n longhorn-volume-demo --ignore-not-found=true

---

### 12.3 Create a Single-Replica Deployment

Create a YAML file with the following content:

cat > 04-deployment-single-replica.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-demo-deploy
  namespace: longhorn-volume-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pvc-demo-deploy
  template:
    metadata:
      labels:
        app: pvc-demo-deploy
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
          claimName: data-pvc
    EOF

Apply the configuration:

kubectl apply -f 04-deployment-single-replica.yaml

Check the deployment status:

kubectl get deploy -n longhorn-volume-demo
kubectl get pod -n longhorn-volume-demo -o wide

---

### 12.4 Verify That the Deployment Pod Can Read Old Data

Get the Pod name:

POD_NAME=$(kubectl get pod -n longhorn-volume-demo -l app=pvc-demo-deploy -o jsonpath '{.items[0].metadata.name}')

Verify the data:

kubectl exec -n longhorn-volume-demo $POD_NAME -- cat /data/hello.txt

Write new data:

kubectl exec -n longhorn-volume-demo $POD_NAME -- sh -c "echo 'write from deployment' > /data/deployment.txt"

Check the updated data:

kubectl exec -n longhorn-volume-demo $POD_NAME -- cat /data/deployment.txt

---

### 12.5 Try to Scale the Deployment to 2 Replicas

High-risk warning:

This experiment is designed to observe potential conflicts between an RWO PVC and a multi-replica Deployment. It should only be performed in the designated experimental namespace.

Execute the following command:

kubectl scale deploy pvc-demo-deploy -n longhorn-volume-demo --replicas=2

Check the Pod status and any related events:

kubectl get pod -n longhorn-volume-demo -o wide
kubectl get events -n longhorn-volume-demo --sort-by=.lastTimestamp | tail -50

Possible outcomes:

- If both Pods are scheduled to the same node, they may still be able to mount the volume successfully.
- If they are scheduled to different nodes, issues such as Multi-Attach, FailedAttachVolume, or mounting failures may occur.
- The actual behavior depends on Kubernetes scheduling algorithms, volume access mechanisms, and node locations.

Conclusion:

It is not recommended to share the same RWO PVC among multiple Deployment replicas. For stateful applications with multiple replicas, it is better to use StatefulSets along with volumeClaimTemplates to ensure that each Pod has its own independent PVC.

To revert the deployment back to 1 replica, execute the following command:

kubectl scale deploy pvc-demo-deploy -n longhorn-volume-demo --replicas=1
---

## Section Thirteen: Practical Exercise Eight: Basics of StatefulSets and volumeClaimTemplates

### 13.1 Why StatefulSets Are More Suitable for Stateful Applications

StatefulSets are ideal for applications that require:

- Stable network identities.
- Stable storage solutions.
-            storageClassName: longhorn
            resources:
              requests:
                storage: 1Gi
    EOF

Explanation:

    Each StatefulSet Pod automatically creates its own PVC.
    web-0 uses data-web-0.
    web-1 uses data-web-1.
    This is more reasonable than multiple Pods sharing one RWO PVC.

---

### 13.4 Using StatefulSets

Execute:

    kubectl apply -f 05-statefulset-volumeclaimtemplates.yaml

View:

    kubectl get sts -n longhorn-volume-demo
    kubectl get pod -n longhorn-volume-demo -o wide
    kubectl get pvc -n longhorn-volume-demo
    kubectl get pv

Expected PVCs:

    data-web-0
    data-web-1

---

### 13.5 Verifying That Each Pod Has Independent Data

View web-0:

    kubectl exec -n longhorn-volume-demo web-0 -- sh -c "hostname && tail -5 /data/app.log"

View web-1:

    kubectl exec -n longhorn-volume-demo web-1 -- sh -c "hostname && tail -5 /data/app.log"

Explanation:

    web-0 and web-1 each use their own PVC.
    They do not share the same RWO volume.
    This is the correct approach for most stateful, multi-replica applications.

---

### 13.6 Checking the Number of Longhorn Volumes

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io

You should see:

    The Volume corresponding to data-pvc
    The Volume corresponding to data-web-0
    The Volume corresponding to data-web-1

View Replicas:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

## Chapter XIV: Clarifying the Relationships Between PVCs, PVs, and Volumes

### 14.1 Checking PVCs

Execute:

    kubectl get pvc -n longhorn-volume-demo -o wide

---

### 14.2 Checking PVs

Execute:

    kubectl get pv

Find the corresponding PV based on the PVC:

    kubectl describe pvc data-pvc -n longhorn-volume-demo

View the Volume information:

    Volume:

Then execute:

    kubectl describe pv <pv-name>

---

### 14.3 Checking Longhorn Volumes

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io

Generally, the name of a Longhorn Volume corresponds to the volume handle of the PV/PVC.

View details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

---

### 14.4 Summarizing the Relationships

A typical relationship is:

    Namespace: longhorn-volume-demo
    PVC: data-pvc
      |
      v
    PV: pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
      |
      v
    Longhorn Volume: pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
      |
      v
    Engine + Replicas
      |
      v
    Data stored in /data/longhorn or /var/lib/longhorn

---

## Chapter XV: Troubleshooting Common Issues

### 15.1 PVC Remaining in Pending State

View:

    kubectl get pvc -n longhorn-volume-demo
    kubectl describe pvc data-pvc -n longhorn-volume-demo
    kubectl get sc
    kubectl describe sc longhorn
    kubectl get events -n longhorn-volume-demo --sort-by=.lastTimestamp

Check Longhorn:

    kubectl -n longhorn-system get pods
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200
    kubectl -n longhorn-system get nodes.longhorn.io

Common causes:

    The longhorn StorageClass does not exist.
    An exception occurred with the CSI Provisioner.
    An issue with the longhorn-manager.
    The node lacks schedulable disks.
    The number of replicas exceeds the available number of nodes.
    Insufficient disk space.
    Longhorn has not been successfully installed.

---

### 15.2 Pod Spending Too Much Time in ContainerCreating State

View:

    kubectl describe pod pvc-test-pod -n longhorn-volume-demo
    kubectl get events -n longhorn-volume-demo --sort-by=.lastTimestamp
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

Focus on checking for:

    FailedMount
    FailedAttachVolume
    Multi-Attach
    timeout
    iscsi
    mount

Check the node:

    systemctl status iscsid
    iscsiadm --version
    journalctl -u kubelet --since "30 minutes ago" | tail -100

Check Longhorn:

Check the Volume attach status in the Longhorn UI.
Do not forcibly delete the Volume.During production, it is important to note that deleting a Pod does not erase the data, but removing a PVC may trigger the deletion of both the PV and the Longhorn Volume, depending on the ReclaimPolicy setting. Therefore, before deleting a PVC in production, it is essential to confirm the business context, backup, and recovery plans. Additionally, Longhorn Replicas are not considered backups; critical data still requires dedicated Backup Targets and recovery procedures.

---

## Summary of This Article

This article covers the practical aspects of Longhorn dynamic volumes:

1. A PVC represents a user's request for persistent storage.
2. The StorageClass specifies how dynamic volumes should be provided.
3. Longhorn dynamically creates PVs through CSI.
4. Once a PVC is bound, it corresponds to a Longhorn Volume.
5. Pods can mount PVCs into container directories.
6. Data written to these mounted directories is stored in the Longhorn Volume.
7. Deleting a Pod does not remove the associated PVC or Longhorn Volume.
8. Rebuilding a Pod allows access to previous data.
9. RWO volumes are not suitable for concurrent read and write operations across multiple nodes.
10. When using a single RWO PVC in a Deployment, setting the replicas to 1 should be done with caution.
11. StatefulSet applications with multiple replicas benefit more from StatefulSet + volumeClaimTemplates.
12. Each StatefulSet replica can have its own independent PVC.
13. It is crucial to ensure that PVCs, PVs, Longhorn Volumes, and Replicas are properly aligned.
14. For issues with PVC Pending status, check StorageClass, CSI, Longhorn Manager, and node disks.
15. During Pod ContainerCreating, monitor FailedMount, iscsid, CSI, and Volume Attach for errors.
16. In cases of Volume Degraded, inspect Replicas, nodes, and disks.
17. Deleting a PVC is a high-risk operation; backup and business impact must be carefully considered in production.
18. Longhorn Replicas are not backups; additional backup strategies are necessary.
19. The next article will explore the Longhorn replica mechanism, including replica numbers, node distribution, and data availability.

---

## References

Longhorn Official Documentation:

    https://longhorn.io/docs/latest/

Longhorn Nodes and Volumes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn StorageClass Parameters:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Longhorn Volumes and Nodes:

    https://longhorn.io/docs/latest/nodes-and-volumes/volumes/

Longhorn Troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Longhorn Backup and Recovery:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes StatefulSet:

    https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

Kubernetes Deployment:

    https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

Kubernetes CSI Documentation:

    https://kubernetes-csi.github.io/docs/