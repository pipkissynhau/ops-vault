# 01-Node NotReady Troubleshooting: kubelet, containerd, CNI, and Node Events

Recommended Path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/01-Node NotReady Troubleshooting: kubelet, containerd, CNI, and Node Events.md

Tags:

    #Kubernetes
    #NodeNotReady
    #kubelet
    #containerd
    #CNI
    #Calico
    #Node Troubleshooting
    #Cluster Basic Troubleshooting

---

## I. Document Description

This document records the basic troubleshooting methods for Node NotReady in a Kubernetes cluster.

Node NotReady is a very common issue in Kubernetes operations, typically indicating that a node is unable to report its status correctly to the control plane, or that the core operating conditions on the node are not met.

Common Impacts:

    1. New Pods cannot be scheduled to the node.
    2. Running Pods may not be manageable properly.
    3. Services running on the node may experience errors.
    4. DaemonSet components may malfunction.
    5. Node components such as CNI, kube-proxy, and CSI may be affected.

Objectives of This Document:

    1. Determine which node is experiencing NotReady.
    2. Check the Node Conditions.
    3. Examine the Node Events.
    4. Troubleshoot kubelet issues.
    5. Troubleshoot containerd issues.
    6. Troubleshoot CNI issues.
    7. Verify network connectivity.
    8. Check for disk, memory, and PID pressure issues.
    9. Establish a standardized troubleshooting process.

Applicable Scenarios:

    1. Self-built kubeadm clusters.
    2. Private Kubernetes clusters.
    3. Situations where a node suddenly becomes NotReady.
    4. Cases where a new node becomes NotReady after joining the cluster.
    5. Instances where a node becomes NotReady after restarting.
    6. Issues where CNI anomalies cause Node NotReady.

---

## II. What is Node NotReady

Viewing Node Status:

    kubectl get nodes -o wide

Example:

    NAME            STATUS     ROLES           AGE   VERSION
    k8s-master-01   Ready      control-plane   10d   v1.31.14
    k8s-worker-01   NotReady   <none>          10d   v1.31.14

NotReady indicates that the Kubernetes control plane considers the node currently unavailable.

Common Causes:

    1. kubelet is not running.
    2. kubelet cannot connect to the APIServer.
    3. containerd is experiencing issues.
    4. CNI is malfunctioning.
    5. Network problems on the node.
    6. High disk pressure on the node.
    7. High memory pressure on the node.
    8. High PID pressure on the node.
    9. Abnormalities with kubelet certificates.
    10. Out-of-sync node time.
    11. Components have not recovered after the node restarts.

---

## III. General Troubleshooting Approach

When troubleshooting Node NotReady, do not immediately restart the node.

Recommended Order:

    1. First, use `kubectl get nodes`.
    2. Then, use `kubectl describe node`.
    3. Check the Node Conditions.
    4. Examine the Node Events.
    5. Verify kubelet on the node.
    6. Check containerd.
    7. Check CNI.
    8. Verify network connectivity on the node.
    9. Assess resource usage.
    10. Only consider restarting components or the node as a last resort.

Main Troubleshooting Path:

    The control plane reports that the node is NotReady.
              |
              v
    Use `describe node` to check Conditions and Events.
              |
              v
    Verify kubelet locally on the node.
              |
              v
    Verify containerd locally on the node.
              |
              v
    Check CNI, network, disk, and memory.
              |
              v
    Fix the issues and wait for the node to return to a Ready state.

---

## IV. Step 1: Identify the Abnormal Node

View all nodes:

    kubectl get nodes -o wide

View labels and roles:

    kubectl get nodes --show-labels

Identify only NotReady nodes:

    kubectl get nodes | grep NotReady

Record relevant information:

    1. Node name.
    2. Node IP address.
    3. Node role.
    4. Kubernetes version.
    5. Time when NotReady occurred.
    6. Determine if## VII. Viewing Node Events

To view node events:

    kubectl describe node k8s-worker-01

You can also view events individually:

    kubectl get events -A --sort-by=.lastTimestamp

Filter by node:

    kubectl get events -A --sort-by=.lastTimestamp | grep k8s-worker-01

Common events include:

    NodeNotReady
    NodeReady
    Starting kubelet
    InvalidDiskCapacity
    ContainerGCFailed
    ImageGCFailed
    EvictionThresholdMet
    FreeDiskSpaceFailed
    NetworkNotReady

Example:

    Warning  NetworkNotReady  kubelet  container runtime network not ready

Note:

    Events are important clues for troubleshooting. If you see "NetworkNotReady", check the CNI first. For "InvalidDiskCapacity", check the disk and containerd. If kubelet stops reporting node status, check its connectivity with the APIServer.

---

## VIII. Checking kubelet on Abnormal Nodes

The following steps should be performed on abnormal nodes. For example:

    k8s-worker-01

---

### 8.1 Viewing kubelet Status

Run:

    systemctl status kubelet --no-pager

Normal status:

    active (running)

If it's not running, check the logs.

---

### 8.2 Viewing kubelet Logs

View in real-time:

    sudo journalctl -u kubelet -f

View the last 100 lines:

    sudo journalctl -u kubelet -n 100 --no-pager

View the last hour:

    sudo journalctl -u kubelet --since "1 hour ago" --no-pager

Common errors include:

    container runtime is down
    network plugin is not ready
    cni config uninitialized
    failed to run Kubelet
    failed to get node
    node not found
    failed to contact API server
    x509 certificate has expired
    failed to load kubelet config file

---

### 8.3 If kubelet Is Not Running

Try starting it:

    sudo systemctl start kubelet

Set it to start automatically at boot:

    sudo systemctl enable kubelet

Check again:

    systemctl status kubelet --no-pager

If startup fails, focus on the following logs:

    sudo journalctl -u kubelet -n 100 --no-pager

Common reasons include:

    1. Missing kubelet configuration file
    2. Incorrect kubelet parameters
    3. containerd not running
    4. Swap memory not disabled
    5. Certificate issues
    6. Incompatible kubelet version

---

## IX. Checking containerd

kubelet relies on containerd to create and manage containers. If containerd is abnormal, the node may quickly become NotReady.

---

### 9.1 Viewing containerd Status

Run:

    systemctl status containerd --no-pager

Normal status:

    active (running)

If it's abnormal:

    sudo systemctl restart containerd

View logs:

    sudo journalctl -u containerd -n 100 --no-pager

---

### 9.2 Using crictl to Check the Runtime

View runtime information:

    sudo crictl info

List containers:

    sudo crictl ps

List all containers:

    sudo crictl ps -a

View images:

    sudo crictl images

If `crictl info` reports an error, common causes include:

    1. containerd not running
    2. Incorrect configuration in /etc/crictl.yaml
    3. Wrong path for the containerd socket
    4. Abnormal containerd configuration file

Check the crictl configuration:

    cat /etc/crictl.yaml

Expected content:

    runtime-endpoint: unix:///run/containerd/containerd.sock
    image-endpoint: unix:///run/containerd/containerd.sock

---

### 9.3 Checking containerd Data Directory

If the containerd data directory was changed during deployment to:

    /data/containerd

Check:

    grep -n '^root = ' /etc/containerd/config.toml

    df -h /data/containerd

    sudo du -sh /data/containerd

If the disk containing this directory is full, it may cause:

    1. Image retrieval failures
    2. Container creation failures
    3. kubelet reporting DiskPressure
    4. Node becoming NotReady

---

## X. Checking CNI Network

If you see the following in kubelet logs:

    network plugin is not ready
    cni config uninitialized
    NetworkPluginNotReady
    container runtime network not ready

Check the CNI configuration first.

---

### 10.1 Viewing CNI Configuration Files

On abnormal nodes, run:

    ls -l /etc/cni/net.d/

There should be/data
/data/containerd
/var/lib/kubelet

Common Issues:

1. Root partition is full.
2. Inode space is exhausted.
3. Containerd images are taking up too much disk space.
4. Logs are filling up the disk.
5. The kubelet pod directory is too large.

To view large directories:

    sudo du -xh /var | sort -h | tail -20

    sudo du -xh /data | sort -h | tail -20

---

### 12.2 Checking Memory Usage

Execute the following commands:

    free -h

    top

    ps aux --sort=-%mem | head

If there is high memory pressure, it is necessary to determine whether it is caused by host processes or Pods.

---

### 12.3 Checking CPU Usage

Execute the following commands:

    top

    uptime

    ps aux --sort=-%cpu | head

If the CPU is constantly at full capacity, kubelet may not be able to report status in a timely manner.

---

### 12.4 Checking System Logs

Execute the following commands:

    dmesg -T | tail -100

    journalctl -xe --no-pager | tail -100

Pay attention to the following errors:

    OOM
    Disk error
    Filesystem error
    Network error
    Containerd error
    kubelet error

---

## Section 13: Checking Node Time Synchronization

Abnormal time settings can lead to certificate verification failures and communication issues between kubelet and the APIServer.

Execute the following commands:

    timedatectl

    chronyc sources -v

If the time is not synchronized:

    sudo systemctl restart chrony

    timedatectl

---

## Section 14: New Nodes Not Being Ready After Joining the Cluster

When a new node joins the cluster and remains in the NotReady state, the common causes are more concentrated. The troubleshooting sequence is as follows:

1. Check if kubelet is running.
2. Check if containerd is running.
3. Verify if CNI is being deployed correctly.
4. Confirm if there are any configuration files in /etc/cni/net.d.
5. Ensure that the node can access the APIServer.
6. Check if the node can pull images.
7. Verify if the kubelet cgroup driver is consistent with containerd.
8. Ensure that swap is disabled.

To check swap:

    swapon --show

If there is any output, disable swap:

    sudo swapoff -a

    sudo sed -i.bak '/swap/s/^/#/' /etc/fstab

Check cgroups:

    grep "SystemdCgroup" /etc/containerd/config.toml

The expected value is:

    SystemdCgroup = true

---

## Section 15: Verification After Restoring a Node

After repairs are completed, check the node status:

    kubectl get nodes -o wide

Wait for the node to become Ready.

View detailed node information:

    kubectl describe node k8s-worker-01

Confirm that:

    Ready=True
    MemoryPressure=False
    DiskPressure=False
    PIDPressure=False
    NetworkUnavailable=False

View system Pods on the node:

    kubectl get pods -A -o wide | grep k8s-worker-01

Verify business Pods:

    kubectl get pods -A -o wide

---

## Section 16: Whether to Restart kubelet or containerd

### 16.1 Scenarios Where Rebooting kubelet is Appropriate

Suitable for:

1. When kubelet freezes.
2. After modifying kubelet configuration.
3. When kubelet fails to report status.
4. After fixing certificate or configuration issues.

Command:

    sudo systemctl restart kubelet

---

### 16.2 Scenarios Where Rebooting containerd Should Be Done with Caution

Rebooting containerd may affect container management on the node, but it generally will not delete containers directly.

Suitable for:

1. When containerd is in an abnormal state.
2. When crictl info returns no response.
3. After modifying containerd configuration.
4. When there are obvious issues with images or runtime.

Command:

    sudo systemctl restart containerd

Then reboot kubelet:

    sudo systemctl restart kubelet

Production Note:

Before rebooting containerd, ensure that there are no important services running on the node. If possible, you can first cordon or drain the node.

---

## Section 17: Whether to Cordon or Drain a Node

If a NotReady node requires extensive troubleshooting, it is recommended to isolate the node first.

### 17.1 Cordoning a Node

Prevent new Pods from being scheduled to this node:

    kubectl cordon k8s-worker-01

View the status:

    kubectl get nodes

The node will display:

   5. Configuration Layer:
        kubelet.conf
        containerd config
        cgroup driver
        Swap memory management
        Time synchronization

Common troubleshooting tips:

    1. If "Ready=False" is reported, check the kubelet configuration first.
    2. If "NetworkUnavailable=True" is encountered, inspect the CNI settings first.
    3. When "DiskPressure=True" appears, examine the disk and inode status.
    4. For "NodeStatusUnknown", verify the node's connectivity to the APIServer.
    5. For new nodes that are not ready, focus on checking swap memory management, containerd configuration, CNI settings, and kubelet behavior.
    6. Avoid restarting nodes immediately; instead, start by using the "describe" command and reviewing logs for any issues.