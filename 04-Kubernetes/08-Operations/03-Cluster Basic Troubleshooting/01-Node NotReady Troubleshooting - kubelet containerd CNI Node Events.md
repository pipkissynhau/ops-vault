# 01-Node NotReady Troubleshooting: kubelet, containerd, CNI, and Node Events

Recommended path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/01-Node NotReady Troubleshooting: kubelet, containerd, CNI, and Node Events.md

Tags:

    #Kubernetes
    #NodeNotReady
    #kubelet
    #containerd
    #CNI
    #Calico
    #NodePlatoon
    #ClusterInfrastructureBarriers

---

## I. Document Overview

This document records basic troubleshooting methods for Node NotReady in Kubernetes clusters.

Node NotReady is a very common issue in Kubernetes operations, typically indicating that a node cannot communicate normally with the control plane or that core runtime conditions on the node are not met.

Common impacts:

    1. New Pods cannot be scheduled to this node
    2. Existing Pods may fail to be managed normally
    3. Business on the node may be abnormal
    4. DaemonSet components may be abnormal
    5. CNI, kube-proxy, CSI, and other node components may be affected

Document objectives:

    1. Identify which node is NotReady
    2. Check Node Conditions
    3. Check Node Events
    4. Troubleshoot kubelet
    5. Troubleshoot containerd
    6. Troubleshoot CNI
    7. Troubleshoot network connectivity
    8. Troubleshoot disk, memory, and PID pressure
    9. Establish a standardized troubleshooting path

Applicable scenarios:

    1. kubeadm self-built cluster
    2. Private Kubernetes cluster
    3. Node suddenly becomes NotReady
    4. New node becomes NotReady after joining
    5. Node becomes NotReady after reboot
    6. CNI anomalies causing node NotReady

---

## II. What is Node NotReady

Check node status:

    kubectl get nodes -o wide

Example:

    NAME            STATUS     ROLES           AGE   VERSION
    k8s-master-01   Ready      control-plane   10d   v1.31.14
    k8s-worker-01   NotReady   <none>          10d   v1.31.14

NotReady indicates that the Kubernetes control plane considers the node currently unavailable.

Common causes:

    1. kubelet is not running
    2. kubelet cannot connect to APIServer
    3. containerd anomalies
    4. CNI anomalies
    5. Node network anomalies
    6. Disk pressure on the node
    7. Memory pressure on the node
    8. PID pressure on the node
    9. kubelet certificate anomalies
    10. Time synchronization issues on the node
    11. Components not recovered after node reboot

---

## III. Troubleshooting Overview

Do not restart nodes immediately when encountering Node NotReady.

Recommended order:

    1. First check kubectl get nodes
    2. Then check kubectl describe node
    3. Check Node Conditions
    4. Check Node Events
    5. Check kubelet on the node
    6. Check containerd
    7. Check CNI
    8. Check node network
    9. Check resource pressure
    10. Finally consider restarting components or the node

Troubleshooting flow:

    Control plane sees node NotReady
              |
              v
    describe node to check Conditions and Events
              |
              v
    Check kubelet on the node
              |
              v
    Check containerd on the node
              |
              v
    Check CNI/network/disk/memory
              |
              v
    Fix issues and wait for node to recover Ready

---

## IV. First Step: Confirming the Abnormal Node

Check all nodes:

    kubectl get nodes -o wide

Check labels and roles:

    kubectl get nodes --show-labels

Check only NotReady nodes:

    kubectl get nodes | grep NotReady

Record information:

    1. Node name
    2. Node IP
    3. Node role
    4. Kubernetes version
    5. Time when NotReady occurred
    6. Whether only one node is abnormal
    7. Whether multiple nodes are abnormal at the same time

Determine direction:

    Only one node is NotReady:
        Prioritize checking kubelet, containerd, CNI, disk, and network issues on the node.

    Multiple nodes are NotReady at the same time:
        Prioritize checking APIServer, network, certificate, time synchronization, and CNI global issues.

---

## V. Second Step: Check Node Details

Check details of the abnormal node:

    kubectl describe node k8s-worker-01

Focus on:

    1. Conditions
    2. Addresses
    3. Capacity
    4. Allocatable
    5. Non-terminated Pods
    6. Events

---

## VI. Check Node Conditions

In `kubectl describe node` output, focus on Conditions.

Common fields:

    Ready
    MemoryPressure
    DiskPressure
    PIDPressure
    NetworkUnavailable

Example:

Conditions:
  Type             Status    Reason
  ----             ------    ------
  MemoryPressure   False     KubeletHasSufficientMemory
  DiskPressure     False     KubeletHasNoDiskPressure
  PIDPressure      False     KubeletHasSufficientPID
  Ready            False     KubeletNotReady

### 6.1 Ready=False

Explanation:

  The node is currently unavailable.

Common Reasons:

  KubeletNotReady
  NodeStatusUnknown
  Kubelet stopped posting node status

Common Directions:

  1. kubelet anomaly
  2. kubelet unable to connect to APIServer
  3. container runtime anomaly
  4. CNI anomaly
  5. node network anomaly

---

### 6.2 NetworkUnavailable=True

Explanation:

  The node network is unavailable.

Common Directions:

  1. CNI not running normally
  2. Calico / Flannel anomaly
  3. PodCIDR allocation anomaly
  4. node routing anomaly
  5. missing CNI configuration file

Check:

  kubectl -n calico-system get pods -o wide

  kubectl -n kube-system get pods -o wide | grep -E "calico|flannel"

  ls -l /etc/cni/net.d/

---

### 6.3 DiskPressure=True

Explanation:

  The node has excessive disk pressure.

Common Impacts:

  1. Pods may be evicted
  2. kubelet may reject creating new Pods
  3. image pull failure
  4. containerd data directory filled

Check:

  df -h

  df -ih

  sudo du -sh /var/lib/containerd

  sudo du -sh /data/containerd

  sudo journalctl -u kubelet --since "1 hour ago"

---

### 6.4 MemoryPressure=True

Explanation:

  The node has excessive memory pressure.

Check:

  free -h

  top

  ps aux --sort=-%mem | head

  kubectl describe node k8s-worker-01

Common Causes:

  1. High node business load
  2. Pods without memory limits
  3. Memory leak in a process
  4. Insufficient system reservation

---

### 6.5 PIDPressure=True

Explanation:

  The node has excessive process count pressure.

Check:

  ps -eLf | wc -l

  cat /proc/sys/kernel/pid_max

  ps -eLf | awk '{print $1}' | sort | uniq -c | sort -nr | head

Common Causes:

  1. A process creatingMass threads
  2. Abnormal fork inside Pod
  3. Abnormal thread count in Java / Go programs
  4. Low node pid limit

---

## SevenI don't know.View Node Events

View node events:

  kubectl describe node k8s-worker-01

You can also view events separately:

  kubectl get events -A --sort-by=.lastTimestamp

Filter by node:

  kubectl get events -A --sort-by=.lastTimestamp | grep k8s-worker-01

Common Events:

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

Explanation:

  Events are important clues for troubleshooting.
  Seeing NetworkNotReady, prioritize checking CNI.
  Seeing InvalidDiskCapacity, prioritize checking disk and containerd.
  Seeing kubelet stopped posting node status, prioritize checking kubelet and APIServer connectivity.

---

## EightI don't know.Check kubelet on the faulty node

The following operations are executed on the faulty node.

Example:

  k8s-worker-01

---

### 8.1 Check kubelet status

Execute:

  systemctl status kubelet --no-pager

Normal status:

  active (running)

If it's not running, check the logs.

---

### 8.2 Check kubelet logs

Real-time view:

  sudo journalctl -u kubelet -f

View last 100 lines:

  sudo journalctl -u kubelet -n 100 --no-pager

View last 1 hour:

  sudo journalctl -u kubelet --since "1 hour ago" --no-pager

Common errors: /think

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

### 8.3 kubelet is not running  

Try to start:  

    sudo systemctl start kubelet  

Set to start on boot:  

    sudo systemctl enable kubelet  

Check again:  

    systemctl status kubelet --no-pager  

If startup fails, focus on:  

    sudo journalctl -u kubelet -n 100 --no-pager  

Common causes:  

    1. Missing kubelet configuration file  
    2. Incorrect kubelet parameters  
    3. containerd is not running  
    4. Swap is not disabled  
    5. Certificate anomalies  
    6. Mismatched kubelet version  

---

## Nine. Check containerd  

kubelet relies on containerd to create and manage containers.  

If containerd is abnormal, the node is likely to be NotReady.  

---

### 9.1 Check containerd status  

Execute:  

    systemctl status containerd --no-pager  

Normal status:  

    active (running)  

If abnormal:  

    sudo systemctl restart containerd  

Check logs:  

    sudo journalctl -u containerd -n 100 --no-pager  

---

### 9.2 Use crictl to check runtime  

Check runtime information:  

    sudo crictl info  

Check containers:  

    sudo crictl ps  

Check all containers:  

    sudo crictl ps -a  

Check images:  

    sudo crictl images  

If crictl info fails, common causes:  

    1. containerd is not running  
    2. /etc/crictl.yaml configuration error  
    3. containerd socket path is incorrect  
    4. containerd configuration file anomaly  

Check crictl configuration:  

    cat /etc/crictl.yaml  

Expected:  

    runtime-endpoint: unix:///run/containerd/containerd.sock  
    image-endpoint: unix:///run/containerd/containerd.sock  

---

### 9.3 Check containerd data directory  

If the containerd data directory was changed to:  

    /data/containerd  

Check:  

    grep -n '^root = ' /etc/containerd/config.toml  

    df -h /data/containerd  

    sudo du -sh /data/containerd  

If the data directory's disk is full, it may cause:  

    1. Failed to pull images  
    2. Failed to create containers  
    3. kubelet reports DiskPressure  
    4. Node NotReady  

---

## Ten. Check CNI Network  

If kubelet logs show:  

    network plugin is not ready  
    cni config uninitialized  
    NetworkPluginNotReady  
    container runtime network not ready  

Prioritize checking CNI.  

---

### 10.1 Check CNI configuration file  

On the faulty node, execute:  

    ls -l /etc/cni/net.d/  

It should have CNI configuration files.  

Calico common files:  

    10-calico.conflist  

If the directory is empty, it indicates CNI configuration has not been deployed to this node.  

---

### 10.2 Check Calico Pod  

On the Master node, execute:  

    kubectl -n calico-system get pods -o wide  

Or:  

    kubectl get pods -A -o wide | grep calico  

Focus on whether there is a calico-node Pod on the faulty node.  

Check calico-node logs:  

    kubectl -n calico-system logs <calico-node-pod-name>  

If Calico is under kube-system:  

    kubectl -n kube-system logs <calico-node-pod-name>  

---

### 10.3 Check node network interfaces  

On the faulty node, execute:  

    ip addr  

    ip route  

Check for CNI-related interfaces, such as:  

    cali*  
    tunl0  
    vxlan.calico  
    cni0  

The interfaces vary by CNI mode, so determine based on the actual plugin.  

---

### 10.4 Common CNI issues  

Common causes:  

    1. Calico Pod is not running  
    2. CNI configuration not written to /etc/cni/net.d/  
    3. Node network is unreachable  
    4. Pod CIDR configuration error  
    5. BGP / VXLAN / IPIP mode anomalies  
    6. Node hostname / IP change  
    7. Firewall or security policy blocking  

---

## Eleven. Check connectivity from node to APIServer  

Node NotReady may also be due to kubelet being unable to connect to APIServer.  

---

### 11.1 Check APIServer VIP resolution  

On the faulty node, execute:  

    getent hosts k8s-api-server  

Expected resolution to:  

    10.0.0.30  

If resolution fails, check:  

    cat /etc/hosts  

---

### 11.2 Check APIServer port  

Execute:  

    curl -k https://k8s-api-server:6443/livez  

Expected:  

    ok  

Or:  

    nc -vz k8s-api-server 6443

If it's not connected, check:

    1. Whether kube-vip is working properly
    2. Whether APIServer is working properly
    3. Whether network between nodes and Master is connected
    4. Whether hosts resolution is correct
    5. Whether firewall is blocking port 6443

---

### 11.3 Check kubelet kubeconfig

Check kubelet configuration:

    ls -l /etc/kubernetes/kubelet.conf

    ls -l /var/lib/kubelet/config.yaml

Check the APIServer address used by kubelet:

    grep server /etc/kubernetes/kubelet.conf

If the server address is incorrect, it may cause kubelet to fail reporting status.

---

## Twelve. Check node base resources

### 12.1 Check disk

Execute:

    df -h

    df -ih

Pay attention to:

    /
    /var
    /data
    /data/containerd
    /var/lib/kubelet

Common issues:

    1. Root partition full
    2. Inode full
    3. Containerd images occupy disk space
    4. Logs occupy disk space
    5. Kubelet pod directory too large

Check large directories:

    sudo du -xh /var | sort -h | tail -20

    sudo du -xh /data | sort -h | tail -20

---

### 12.2 Check memory

Execute:

    free -h

    top

    ps aux --sort=-%mem | head

If memory pressure is high, further determine whether it's caused by host processes or Pods.

---

### 12.3 Check CPU

Execute:

    top

    uptime

    ps aux --sort=-%cpu | head

If CPU is consistently maxed out, kubelet may also fail to report status timely.

---

### 12.4 Check system logs

Execute:

    dmesg -T | tail -100

    journalctl -xe --no-pager | tail -100

Pay attention to:

    OOM
    disk error
    filesystem error
    network error
    containerd error
    kubelet error

---

## Thirteen. Check node time synchronization

Time anomalies may cause certificate validation failure, kubelet communication issues with APIServer.

Execute:

    timedatectl

    chronyc sources -v

If time is out of sync:

    sudo systemctl restart chrony

    timedatectl

---

## Fourteen. New node join after NotReady

After a new node joins the cluster and is NotReady, common causes are more concentrated.

Troubleshooting order:

    1. Whether kubelet is running
    2. Whether containerd is running
    3. Whether CNI is properly deployed
    4. Whether /etc/cni/net.d has configuration
    5. Whether the node can access APIServer
    6. Whether the node can pull images
    7. Whether kubelet cgroup driver matches containerd
    8. Whether swap is disabled

Check swap:

    swapon --show

If there is output, disable it:

    sudo swapoff -a

    sudo sed -i.bak '/swap/s/^/#/' /etc/fstab

Check cgroup:

    grep "SystemdCgroup" /etc/containerd/config.toml

Expected:

    SystemdCgroup = true

---

## Fifteen. Verification after node recovery

After fixing the issue, check node status:

    kubectl get nodes -o wide

Wait for the node to recover Ready.

Check node details:

    kubectl describe node k8s-worker-01

Confirm:

    Ready=True
    MemoryPressure=False
    DiskPressure=False
    PIDPressure=False
    NetworkUnavailable=False

Check system Pods on the node:

    kubectl get pods -A -o wide | grep k8s-worker-01

Verify business Pods:

    kubectl get pods -A -o wide

---

## Sixteen. Whether to restart kubelet or containerd

### 16.1 Scenarios suitable for restarting kubelet

Recommended for:

    1. kubelet is frozen
    2. After modifying kubelet configuration
    3. kubelet fails to recover reporting status
    4. After certificate or configuration fixes

Command:

    sudo systemctl restart kubelet

---

### 16.2 Caution when restarting containerd

Restarting containerd may affect container management on the node, but generally won't directly delete containers.

Recommended for:

    1. containerd status is abnormal
    2. crictl info is unresponsive
    3. After modifying containerd configuration
    4. Obvious anomalies in images or runtime

Command:

    sudo systemctl restart containerd

Then restart kubelet:

    sudo systemctl restart kubelet

Production note:

    Confirm there are no critical business pods on the node before restarting containerd.
    If possible, cordone/drain the node first.

---

## Seventeen. Whether to cordon / drain

If the node is NotReady and requires long-term processing, it's recommended to isolate the node first.

### 17.1 Cordon node

Prevent new Pods from being scheduled to the node:

    kubectl cordon k8s-worker-01

Check:

    kubectl get nodes

The node will show:

    SchedulingDisabled

---

### 17.2 Drain node

Evict Pods from the node:

    kubectl drain k8s-worker-01 \
      --ignore-daemonsets \
      --delete-emptydir-data

Note: /data/containerd

drain affects business operations.
Production environments must confirm business replica count, PDB, data volumes, and maintenance window.

---

### 17.3 Recovery Scheduling

After issue resolution, resume node scheduling:

    kubectl uncordon k8s-worker-01

---

## Eighteen, Common Issues Quick Reference

### 18.1 kubelet is not running

Check:

    systemctl status kubelet --no-pager

Resolution:

    sudo systemctl restart kubelet

    sudo journalctl -u kubelet -n 100 --no-pager

---

### 18.2 containerd is not running

Check:

    systemctl status containerd --no-pager

Resolution:

    sudo systemctl restart containerd

    sudo crictl info

---

### 18.3 CNI is not ready

Check:

    ls -l /etc/cni/net.d/

    kubectl get pods -A -o wide | grep -E "calico|flannel"

Resolution:

    Check CNI Pod logs to confirm if CNI configuration has been deployed.

---

### 18.4 Node disk is full

Check:

    df -h

    df -ih

    sudo du -sh /data/containerd

Resolution:

    Clean up unused images, logs, or expand disk space.

---

### 18.5 Node cannot access APIServer

Check:

    getent hosts k8s-api-server

    curl -k https://k8s-api-server:6443/livez

Resolution:

    Check hosts, kube-vip, network, and firewall settings.

---

## Nineteen, Standard Troubleshooting Command List

### 19.1 Control Plane Execution

    kubectl get nodes -o wide

    kubectl describe node <node-name>

    kubectl get events -A --sort-by=.lastTimestamp

    kubectl get pods -A -o wide | grep <node-name>

    kubectl -n kube-system get pods -o wide

    kubectl -n calico-system get pods -o wide

---

### 19.2 Abnormal Node Execution

    hostname

    ip addr

    ip route

    df -h

    df -ih

    free -h

    top

    timedatectl

    chronyc sources -v

    systemctl status kubelet --no-pager

    journalctl -u kubelet -n 100 --no-pager

    systemctl status containerd --no-pager

    journalctl -u containerd -n 100 --no-pager

    crictl info

    crictl ps -a

    crictl images

    ls -l /etc/cni/net.d/

    getent hosts k8s-api-server

    curl -k https://k8s-api-server:6443/livez

---

## Twenty, Summary

The core of Node NotReady troubleshooting is not memorizing commands, but determining which layer the issue belongs to.

Standard layering:

    1. Control Plane Perspective:
        kubectl get nodes
        kubectl describe node
        Events
        Conditions

    2. Node Process Layer:
        kubelet
        containerd

    3. Network Layer:
        CNI
        Pod Network
        Node to APIServer

    4. Resource Layer:
        Disk
        inode
        Memory
        CPU
        PID

    5. Configuration Layer:
        kubelet.conf
        containerd config
        cgroup driver
        swap
        Time Synchronization

Common experience:

    1. Ready=False first check kubelet
    2. NetworkUnavailable=True first check CNI
    3. DiskPressure=True first check disk and inode
    4. NodeStatusUnknown first check node to APIServer connectivity
    5. New node NotReady focus on swap, containerd, CNI, kubelet
    6. Do not restart nodes immediately, first check describe and logs