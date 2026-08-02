# Interview Question 43: Kubernetes Cluster Upgrade Interview Notes (Based on kubeadm Scenario)

## Tags
#Kubernetes #K8s #ClusterUpgrade #kubeadm #Interview #Transport #Clouds.

## One. Interview Question

Interview Question:  
Please explain the overall approach, upgrade steps, precautions, and handling procedures if the upgrade fails.

---

## Two. Essence of the Question

This question is not merely asking "how to execute commands", but it examines the following points:

1. Whether you understand the **principle of prioritizing control plane upgrades**  
2. Whether you know the **step-by-step, batch, and rollback-ready upgrade approach**  
3. Whether you consider **business impact, Pod eviction, compatibility, backup, and verification**  
4. Whether you understand the **kubeadm upgrade process** and **kubelet/kubectl compatibility upgrades**  
5. Whether you have **risk control awareness** in production environments

---

## Three. Standard Answer Structure

You can answer according to the following structure:

1. Pre-upgrade preparation  
2. Upgrade control plane nodes  
3. Upgrade worker nodes  
4. Post-upgrade verification  
5. Risk control and rollback strategy

If the interview time is short, you can summarize with one sentence:

**K8s cluster upgrades typically follow the principle of "first assessment, first backup, first control plane, then worker nodes, incremental upgrades, step-by-step verification, and rollback readiness."**

---

## Four. Must-Confirm Content Before Upgrade

### 1. Confirm Current Version and Target Version

First, confirm the current cluster version, node version, and kubeadm/kubelet/kubectl versions.

    kubectl get nodes -o wide
    kubectl version
    kubeadm version

Notes:

- Generally, you cannot jump multiple minor versions directly
- Refer to the official version compatibility strategy
- There are version compatibility requirements between kubeadm, kubelet, and kubectl

---

### 2. Read the Release Notes of the Target Version

Focus on:

- Whether APIs are deprecated
- Whether component parameters have changed
- Whether Admission, Ingress, CSI, CNI, CoreDNS, etc., are affected
- Whether third-party components are compatible with the target version

For example, pay attention to:

- Whether deprecated APIs are still in use
- Whether Helm-deployed components support the new version
- Whether Calico/Flannel/Cilium are compatible
- Whether CoreDNS plugin configurations have changed

---

### 3. Perform Backups

In production environments, at least consider the following backups before upgrading:

#### etcd Backup
If using an independent etcd or local etcd on control plane nodes, back up etcd data.

Example:

    ETCDCTL_API=3 etcdctl \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key \
      snapshot save /backup/etcd-snapshot.db

#### Key Configuration Backup

- `/etc/kubernetes/`
- kube-apiserver, controller-manager, scheduler static Pod manifests
- kubelet configuration
- CNI configuration
- Key business YAML/Helm values

---

### 4. Check Business High Availability

Upgrading nodes typically requires `cordon + drain`, so confirm:

- Whether the business is multi-replica
- Whether there are PDBs (PodDisruptionBudgets)
- Whether there are single-replica core services
- Whether there are DaemonSet, local storage, emptyDir, etc., special Pods
- Whether there are middleware Pods that cannot be evicted arbitrarily

Without high availability, the upgrade may cause business interruption.

---

### 5. Check Node and Component Status

Before upgrading, ensure the cluster is healthy; do not upgrade with existing issues.

Common checks:

    kubectl get nodes
    kubectl get pods -A
    kubectl get componentstatuses
    kubectl get events -A --sort-by=.lastTimestamp

Also focus on:

- Whether CoreDNS is normal
- Whether the network plugin is normal
- Whether kube-proxy is normal
- Whether metrics-server, ingress-controller is normal
- Whether etcd is healthy

---

## Five. General Principles for kubeadm Cluster Upgrade

### 1. Upgrade Control Plane First, Then Worker Nodes

Reason:

- The control plane determines cluster management capabilities
- New version control planes typically remain compatible with old kubelet for some time
- Upgrading worker nodes first may lead to higher version mismatch risks

---

### 2. Upgrade One Minor Version at a Time

Example:

- `1.27 -> 1.28`
- `1.28 -> 1.29`

Not recommended to jump across multiple minor versions in production.

---

### 3. Upgrade Nodes One by One, Not All at Once

Correct approach:

- Upgrade control plane nodes one by one
- Upgrade worker nodes one by one or in batches
- Verify after each node upgrade

---

## Six. Control Plane Node Upgrade Process

The following example is for a kubeadm-managed cluster.

---

### Step 1: Check Upgrade Plan on the Target Control Plane Node

First upgrade the kubeadm package on the node, then check the upgrade plan.

Example to view upgrade targets:

    kubeadm upgrade plan

This command will tell you:

- Current cluster version
- Target version that can be upgraded
- Required versions of CoreDNS/kube-proxy and other components

---

### Step 2: Upgrade the First Control Plane Node

Execute on the first master node:

    kubeadm upgrade apply v1.xx.y

Purpose:

- Upgrade control plane component static Pods
- Update cluster configuration
- Handle certificates and related version information

Note:

- `apply` is typically used for the first control plane node
- If there are multiple control plane nodes, subsequent nodes commonly use `node`

---

### Step 3: Upgrade kubelet and kubectl

After installing the target version of kubelet and kubectl, restart kubelet.

Example approach:

    systemctl daemon-reload
    systemctl restart kubelet
    systemctl status kubelet

---

### Step 4: Verify the Status of the Control Plane Node

Verification content:

    kubectl get nodes
    kubectl get pods -A -o wide

Focus on observing:

- Is the current control plane node Ready  
- Have kube-apiserver / controller-manager / scheduler recovered  
- Is CoreDNS functioning normally  
- Is the cluster API accessible  

---

### Step 5: Upgrade Remaining Control Plane Nodes  

Other master nodes are typically handled as follows:  

1. Upgrade kubeadm  
2. Execute:  

       kubeadm upgrade node  

3. Upgrade kubelet / kubectl  
4. Restart kubelet  
5. Verify node recovery  

**Note:** In multi-master scenarios, upgrade one node at a time; do not perform concurrent upgrades.  

---

## VII. Worker Node Upgrade Process  

Worker node upgrades typically follow the flow of "block traffic -> evict Pods -> upgrade -> resume scheduling".  

---

### Step 1: Disable Scheduling  

    kubectl cordon <node-name>  

**Purpose:**  
- Mark the node as unschedulable  
- New Pods will no longer be scheduled to this node  

---

### Step 2: Evict Business Pods  

    kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data  

**Explanation:**  
- `--ignore-daemonsets`: Ignore DaemonSet Pods  
- `--delete-emptydir-data`: Allow deletion of emptyDir data  
- Some environments may also require `--force`, but this should be used cautiously in production  

Here, you can add a note during an interview:  

**The purpose of drain is to smoothly migrate business Pods to other nodes, minimizing the impact of upgrades on operations.**  

---

### Step 3: Upgrade kubeadm on the Node  

First upgrade kubeadm, then execute:  

    kubeadm upgrade node  

---

### Step 4: Upgrade kubelet and kubectl  

After the upgrade, restart kubelet:  

    systemctl daemon-reload  
    systemctl restart kubelet  
    systemctl status kubelet  

---

### Step 5: Resume Scheduling  

After confirming the node is Ready, execute:  

    kubectl uncordon <node-name>  

This allows new Pods to be scheduled back.  

---

## VIII. Post-Upgrade Verification Actions  

After the upgrade, do not only check if nodes are Ready; also perform dual verification at the business and component levels.  

### 1. Cluster-Level Verification  

    kubectl get nodes  
    kubectl get pods -A  
    kubectl version  

**Check:**  
- Are all nodes Ready  
- Are all system Pods Running / Completed  
- Are control plane and kubelet versions as expected  

---

### 2. Core Component Verification  

Focus on verifying:  

- CoreDNS  
- kube-proxy  
- CNI plugin  
- Ingress Controller  
- CSI storage plugin  
- metrics-server  
- Monitoring and log collection components  

---

### 3. Business-Level Verification  

At least verify the following:  

- Are critical services accessible  
- Is service-to-Pod forwarding normal  
- Is DNS resolution normal  
- Are Ingress / LB services normal  
- Are mounted volumes normal  
- Are deployment, restart, and self-healing capabilities normal  

---

## IX. Key Precautions in Production Environments  

## 1. Do Not Ignore API Deprecation Issues  

Some cluster upgrades fail not because the upgrade command fails, but because post-upgrade business YAML is incompatible.  

Examples:  
- Deployment, Ingress, CronJob API versions are deprecated  
- Some old Helm Charts still reference deprecated APIs  

Therefore, it's best to check before upgrading:  

    kubectl get ingress -A -o yaml  
    kubectl get deploy -A -o yaml  
    kubectl get cronjob -A -o yaml  

Pay attention to whether old API versions are still in use.  

---

## 2. Be Aware of PDBs Causing Drain to Stall  

If strict PDBs are set, it may result in:  
- Pods unable to be evicted  
- Drain being blocked indefinitely  
- Upgrade window being extended  

At this point, assess business tolerance before deciding whether to temporarily adjust PDBs.  

---

## 3. Be Aware of DaemonSet, Static Pods, and Single-Replica Components  

These resources are common risks during upgrades:  
- DaemonSets are not automatically evicted by drain  
- Static Pods are directly managed by kubelet  
- Single-replica middleware may experience interruptions during migration  

---

## 4. Be Aware of Local Storage Workloads  

If Pods use:  
- hostPath  
- local PV  
- emptyDir  
- Local database files  

Evaluate data risks before upgrading nodes.  

---

## 5. Be Aware of Certificates and Time Synchronization  

Sometimes kubelet or apiserver anomalies after upgrades are not due to version issues, but may be:  
- Expired certificates  
- Certificates not being rotated correctly  
- Nodes having unsynchronized time  

---

## X. How to Handle Upgrade Failures  

This point is crucial for interviews, as it demonstrates production experience.  

### 1. First, Determine Where the Failure Occurred  

Common layers:  
- Package upgrade failure  
- kubeadm upgrade failure  
- kubelet startup failure  
- Static Pods of control plane failing to start  
- Network plugin anomalies  
- CoreDNS anomalies  
- Business Pods unable to recover  

---

### 2. Common Troubleshooting Commands  

Check kubelet:  

    systemctl status kubelet  
    journalctl -u kubelet -f  

Check static Pods:  

    crictl ps -a  
    crictl logs <container-id>  

Check nodes and Pods:  

    kubectl get nodes  
    kubectl get pods -A -o wide  
    kubectl describe pod <pod-name> -n <namespace>  

---

### 3. Rollback Strategy  

Note:  

**Kubernetes upgrades are not always reversible in any scenario.**  

Therefore, a more cautious statement for interviews is:  
- Take etcd snapshots and configuration backups before upgrades  
- Prioritize testing environment rehearsals  
- In production, reduce rollback probability through "small-step upgrades + node-by-node verification"  
- In case of severe failures, recover using etcd snapshots, node snapshots, and component configuration backups  
- In some cases, reinstall corresponding version components and restore control plane configurations  

A more professional expression:  

**Rather than emphasizing rollback after upgrades, emphasize pre-upgrade backups, rehearsals, canary releases, and node-by-node verification to minimize rollback needs.**  

---

## XI. Interview Answer Template (Suitable for Direct Verbal Response)  

If the interviewer asks, "Describe the Kubernetes cluster upgrade process," you can answer:

Kubernetes cluster upgrades I typically handle as a production change. First, I confirm the compatibility between the current version and target version, review the release notes of the target version, and focus on checking API deprecations, CNI, CoreDNS, CSI, and business component compatibility. Then I perform etcd and critical configuration backups in advance, confirming whether the business has multi-replica and Pod eviction readiness. During actual upgrades, I follow the principle of first upgrading the control plane, then the worker nodes, and upgrade node by node. For the control plane, I first upgrade kubeadm, then execute kubeadm upgrade apply or kubeadm upgrade node, followed by upgrading kubelet and kubectl, and restarting kubelet. For worker nodes, I first cordon, then drain to smoothly migrate business Pods, then upgrade kubeadm, kubelet, and kubectl, and finally uncordon to resume scheduling. Throughout the process, I validate node status, system component status, and critical business chains at every step. If an upgrade anomaly occurs, I troubleshoot using kubelet logs, static Pods, etcd, network plugins, and CoreDNS. The core idea in production environments is not to blindly pursue speed, but to perform small steps, verify node by node, ensure backup and recovery capabilities.

---

## Twelve. Interviewer May Follow-up Questions

### 1. Why upgrade the control plane first?
Because the control plane manages the entire cluster, and new control plane versions typically have some compatibility with old kubelet versions. Upgrading master nodes first is more controllable in terms of risk.

### 2. Why drain nodes?
To smoothly migrate business Pods to other nodes, avoiding direct impact on business operations during node upgrades.

### 3. What is the most important preparation before upgrading?
Version compatibility check, etcd backup, business high availability confirmation, and test environment rehearsal.

### 4. What if draining a node gets stuck?
Typically, first check if it's caused by PDB, single-replica business, DaemonSet, or local storage Pods, then decide whether to adjust the strategy.

### 5. How to upgrade multiple masters?
Also upgrade node by node. First select one control plane node to execute `kubeadm upgrade apply`, and other control plane nodes to execute `kubeadm upgrade node`. Validate after each node upgrade.

### 6. What to do if business anomalies occur post-upgrade but nodes are Ready?
Prioritize checking CoreDNS, CNI, kube-proxy, Ingress, CSI, service forwarding, DNS resolution, mounted volumes, and business logs, rather than just node status.

---

## Thirteen. Common Weakness Points in Answers

### 1. Memorizing commands without understanding risk control
In interviews, you can't just say:

- cordon
- drain
- kubeadm upgrade

You should also explain:

- Why do this?
- What impact on business?
- How to mitigate risks?

---

### 2. Ignoring etcd backup
This is a typical deduction point.  
As long as you mention production upgrades, you should generally mention etcd snapshots and critical configuration backups.

---

### 3. Ignoring CNI / CoreDNS / CSI
Many candidates only say "node Ready is sufficient," but in real production, the most problematic areas after upgrades are often:

- Network
- DNS
- Storage
- Ingress

---

### 4. Describing upgrades as "massive simultaneous upgrades"
In production environments, this is a clearly immature statement.  
The correct approach is definitely **batched, node-by-node, and validated step-by-step**.

---

## Fourteen. Expressions That Add Value in Interviews

You can add a sentence:

**If it's a production environment, I would first rehearse the same version upgrade process in the test environment, and confirm that monitoring, logs, alerts, backups, and rollback plans are all prepared before the formal upgrade window.**

This sentence adds value as it demonstrates change awareness, not just command-line knowledge.

---

## Fifteen. Summary

The core of Kubernetes cluster upgrades is not the "upgrade command itself," but the following five things:

1. Version compatibility assessment
2. Backups and rehearsals
3. Upgrade control plane first, then worker nodes
4. Node-by-node upgrades with step-by-step validation
5. Troubleshootable and recoverable in case of anomalies

One-sentence summary:

**Kubernetes upgrades are a standardized change process, not a simple software installation process.**

---

## Sixteen. Operational Extension Understanding

In real production environments, Kubernetes upgrades are typically tied to the following work:

- Change approval
- Upgrade window scheduling
- Monitoring alert silence or observation
- Release freeze
- Business owner confirmation
- Post-upgrade inspection
- Fault trace-back and post-mortem analysis

Therefore, if you can combine "technical steps" with "production change awareness" in interviews, your overall answer will be more mature.

---

## Seventeen. Reference External Links

- Kubernetes official upgrade documentation  
  https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

- Kubernetes version compatibility strategy  
  https://kubernetes.io/releases/version-skew-policy/

- kubeadm official documentation  
  https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/

- drain command reference  
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/

- etcd snapshot backup documentation  
  https://etcd.io/docs/

## Eighteen. Interview Memorization Version

You can memorize it as a mnemonic:

**First assess, first backup; first master, then worker; first cordon, then drain; upgrade kubeadm first, then kubelet; validate each node after upgrading.**