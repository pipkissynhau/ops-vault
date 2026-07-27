# Interview Question 44: Notes on etcd Snapshots, Backups, and Restores

## Tags
#etcd #snapshots #backup_and_restore #Kubernetes #control-plane #interview-question #ops

## I. Interview Question

Interview question:  
What is an etcd snapshot? What is its purpose? How are backups and restores performed? What should be considered in a production environment?

---

## II. What is an etcd Snapshot

An etcd snapshot, essentially:

**creates a backup file of the data stored in etcd at a certain moment.**

It can be understood as:

- A complete backup of the etcd database
- An archive of the Kubernetes control plane status
- A reference for rolling back in case of failures

In a Kubernetes context, etcd typically stores the status of cluster objects, such as:

- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- Nodes
- Namespaces
- And other API objects

Therefore, **an etcd snapshot is not just a regular file backup; it is a backup of the core cluster state data.**

---

## III. What is the Purpose of an etcd Snapshot

## 1. For Backup

The most direct purpose is for backup.  
In cases where etcd data is corrupted, the disk fails, or there are node errors, snapshots can be used to restore the data.

## 2. For Restoration

When etcd experiences a failure or needs to be migrated, a snapshot can be used to recreate the data directory and restart the etcd service.

## 3. As a Backup Measure Before Upgrades

It is generally recommended to take an etcd snapshot before:

- Upgrading a Kubernetes cluster
- Upgrading etcd itself
- Making major changes to the control plane
- Migrating etcd nodes
- Adjusting the data directory

---

## IV. Snapshots and High Availability Are Not the Same Thing

This point is often asked during interviews.

### etcd Cluster High Availability
High availability focuses on ensuring that:

- Services can continue to provide functionality even if a node fails
- Multiple nodes can still maintain majority decision-making

### etcd Snapshots
Snapshots ensure that:

- Lost data can be restored
- The original data state can be recovered in case of upgrade failures

Therefore:

**A 3-node etcd cluster addresses high availability issues, while snapshots handle backup and restoration tasks.**

---

## V. Core Features of Snapshots

## 1. Snapshots Are Copies of Data at a Specific Moment

Snapshots are not real-time backups.  
For example:

- A snapshot is taken at 10:00
- A new Deployment is created at 10:30
- etcd fails at 11:00
- The system restores using the 10:00 snapshot

Data written after 10:00 will not be included in the restored state.

Thus, restoring from a snapshot essentially means:

**restoring data to the state it was in at the time of the snapshot.**

---

## VI. Basic Commands for Creating Snapshots

Using etcd v3 API, the common command is:

    export ETCDCTL_API=3

    etcdctl \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key \
      snapshot save /backup/etcd-snapshot.db

Explanation:

- --endpoints: Specifies the etcd access address
- --cacert: CA certificate
- --cert: Client or server certificate
- --key: Private key
- snapshot save: Creates a snapshot
- /backup/etcd-snapshot.db: Output snapshot file location

---

## VII. How to Understand This Command

You can think of it as:

“Connect to the current etcd node and export the data in the current keyspace as a snapshot file.”

In a Kubernetes context, this command is usually executed on the control plane node.

---

## VIII. How to Verify a Snapshot File

Currently, `etcdutl` is recommended for verifying snapshot status.

Example:

    etcdutl snapshot status /backup/etcd-snapshot.db -w table

This will display information such as:

- HASH
- REVISION
- TOTAL KEYS
- TOTAL SIZE

These details help confirm:

- Whether the snapshot file can be recognized
- That the data version and size are correct
- That the backup was successfully created

---

## IX. Basic Understanding of Restoration

Restoration is not simply “placing the snapshot file back”.  
The actual process involves:

**recreating a new etcd data directory from the snapshot file.**

In other words:

- The snapshot file serves as theBecause etcd stores the core status of the control plane, it provides a basis for recovery in case of an upgrade failure.

### 4. Why is etcdutl now recommended for recovery?
Because newer versions of etcd have gradually removed snapshot status and snapshot restore functions from etcdctl.

### 5. Once the snapshot file is generated, is everything safe?
Not necessarily. Other considerations include off-site storage, recovery drills, version compatibility, and certificate paths.

---

## Nineteen, Practical Memory Version

One-sentence memory:

**An etcd snapshot is a backup file of the etcd database at a specific moment in time.**

Another addition:

**High availability ensures that the system remains operational, while snapshots ensure that data can be recovered after an issue occurs.**

---

## Twenty, References and External Links

- Kubernetes Official: Operating etcd clusters for Kubernetes
- etcd Official: snapshot save / snapshot status / snapshot restore
- etcd Official Upgrade Instructions