# Interview Question 43: Notes on K8s Cluster Upgrade (Based on Kubeadm Scenario)

## Tags
#Kubernetes #K8s #Cluster Upgrade #kubeadm #Interview Questions #Ops #Cloud-Native

## I. Interview Question

Question:
Please explain the overall approach, steps, and precautions for upgrading a Kubernetes cluster, as well as how to handle failure during the upgrade process.

---

## II. Essence of the Question

This question is not merely about knowing how to execute specific commands; it aims to assess the following aspects:

1. Understanding the principle of **upgrading the control plane first**
2. Familiarity with the step-by-step, batch-based, and rollback-capable upgrade approach
3. Consideration of **business impact, Pod eviction, compatibility, backup, and verification**
4. Knowledge of the **kubeadm upgrade process** and its coordination with kubelet/kubectl upgrades
5. Possession of **risk control awareness in a production environment**

---

## III. Standard Answer Structure

You can answer this question by following these steps:

1. Preparation before Upgrade
2. Upgrading Control Plane Nodes
3. Upgrading Worker Nodes
4. Post-Upgrade Verification
5. Risk Control and Rollback Strategies

If the interview time is limited, you can summarize it in one sentence:

**K8s cluster upgrades generally follow the principle of "evaluation first, backup, control plane first, worker nodes later, step-by-step upgrade, verification at each stage, and rollback in case of issues."**

---

## IV. Must-Confirm Items Before Upgrade

### 1. Verify Current and Target Versions

First, confirm the current cluster version, node versions, and kubeadm/kubelet/kubectl versions:

    kubectl get nodes -o wide
    kubectl version
    kubeadm version

Notes:

- It is generally not advisable to skip too many minor version updates in one go.
- Refer to the official version compatibility guidelines.
- There are specific version compatibility requirements between kubeadm, kubelet, and kubectl.

---

### 2. Read the Target Version's Release Notes

Focus on:

- Any deprecated APIs
- Changes in component parameters
- Potential impacts on Admission, Ingress, CSI, CNI, CoreDNS, etc.
- Compatibility with third-party components

For example, check:

- Whether any deprecated APIs are still in use.
- Whether Helm-deployed components support the new version.
- Compatibility of Calico/Flannel/Cilium.
- Changes to CoreDNS plugin configurations.

---

### 3. Perform Backup

Before upgrading in a production environment, consider backing up at least the following:

#### etcd Backup
If you have an independent etcd or local control plane etcd, back up its data:

    ETCDCTL_API=3 etcdctl \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key \
      snapshot save /backup/etcd-snapshot.db

#### Critical Configuration Backup
- `/etc/kubernetes/`
- Static Pod lists for kube-apiserver, controller-manager, scheduler
- kubelet configuration
- CNI configuration
- Important business YAML files/Helm values

---

### 4. Assess Business High Availability

Since node upgrades typically involve `cordon + drain`, verify the following:

- Whether your services have multiple replicas.
- The presence of PDB (PodDisruptionBudget).
- Any single-replica critical services.
- Special Pods such as DaemonSets, local storage, and emptyDir pods.
- Middleware Pods that cannot be easily evicted.

Failure to ensure high availability may lead to service interruptions during the upgrade.

---

### 5. Check Node and Component Status

Ensure the cluster is healthy before starting the upgrade:

    kubectl get nodes
    kubectl get pods -A
    kubectl get component statuses
    kubectl get events -A --sort-by=.lastTimestamp

Pay special attention to:

- The status of CoreDNS.
- The functionality of network plugins.
- The status of kube-proxy.
- The health of metrics-server and ingress-controller.
- The integrity of etcd.

---

## V. General Principles for Kubeadm Cluster Upgrades

### 1. Upgrade the Control Plane First, Then Worker Nodes

Reasons:

- The control plane determines the cluster's management capabilities.
- New versions of the control plane are usually compatible with older kubelet versions for a certain period.
- Starting with worker nodes first may increase the risk of version mismatches.

---

### 2. Upgrade Only One Minor Version at a Time

For example:

- `1.27 -> 1.28`
- `1.28 -> 1.29Some cluster upgrades fail not because of the upgrade command itself, but due to incompatibility between the upgraded business YAML and the new version. For example:

- API versions for Deployment, Ingress, CronJob, etc., have been deprecated.
- Some older versions of Helm Charts may still reference these deprecated APIs.

Therefore, it's advisable to check the following before upgrading:

    kubectl get ingress -A -o yaml
    kubectl get deploy -A -o yaml
    kubectl get cronjob -A -o yaml

To ensure that you're not still using outdated APIs.