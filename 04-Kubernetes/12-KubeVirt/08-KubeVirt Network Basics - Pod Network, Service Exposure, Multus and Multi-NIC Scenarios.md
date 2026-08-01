# 08-KubeVirt Networking Basics: Pod Networking, Service Exposure, Multus, and Multi-NIC Scenarios

Recommended Path:

    04-Kubernetes/12-KubeVirt/08-KubeVirt Network basis:Pod Network,Service Exposure,Multus With Multinet scenes.md

Tags:

    #Kubernetes
    #KubeVirt
    #Network
    #PodNetwork
    #Service
    #NodePort
    #LoadBalancer
    #MetalLB
    #Multus
    #NetworkAttachmentDefinition
    #Multinet
    #VirtualMachineNetwork
    #CloudlandVirtualization

---

## I. Document Explanation

This document records the basic concepts and practical methods of networking in KubeVirt virtual machines.

Previous steps completed:

    1. KubeVirt Installation
    2. virtctl Installation
    3. containerDisk Creation of Test VM
    4. CDI / DataVolume Image Import
    5. PVC / Longhorn VM Disk Experiment

This document continues to learn KubeVirt networking basics.

Key focus areas:

    1. Understand how KubeVirt VMs default to access Pod networks
    2. Understand the masquerade networking mode
    3. Understand how VMs expose ports through Service
    4. Master ClusterIP / NodePort / LoadBalancer exposure methods
    5. Understand differences between VM networking and regular Pod networking
    6. Understand the role of Multus
    7. Learn about VM multi-NIC scenarios
    8. Complete a default Pod network VM experiment
    9. Complete an SSH exposure via Service experiment
    10. Optionally complete a Multus dual-NIC experiment
    11. Master common KubeVirt networking troubleshooting paths

Document positioning:

    From beginner to troubleshooting capability.
    Does not delve into SR-IOV, DPDK, NUMA, complex layer-2 networks, or high-performance virtualization networking.
    First establish foundational operational, verification, and troubleshooting capabilities.

---

## II. What to Understand About KubeVirt Networking

KubeVirt virtual machines ultimately run in virt-launcher Pods.

From a Kubernetes perspective:

    VM runs in a Pod

From a networking perspective:

    VM networking is first related to the virt-launcher Pod network

Basic chain:

    VM Guest OS
        |
        v
    virt-launcher Pod network namespace
        |
        v
    Kubernetes CNI
        |
        v
    Pod network / Service / NodePort / LoadBalancer

Therefore, KubeVirt networking troubleshooting cannot focus solely on the virtual machine internally.

Also check:

    1. VM / VMI network configuration
    2. virt-launcher Pod
    3. Pod IP
    4. CNI status
    5. Service and Endpoints
    6. NodePort / LoadBalancer
    7. VM Guest OS internal network card, IP, routing, firewall

---

## III. Common Networking Modes in KubeVirt

At the beginner stage, master three types:

    1. pod network + masquerade
    2. Service port exposure
    3. Multus multi-NIC

### 3.1 pod network + masquerade

This is the most common approach for beginners.

VM uses Kubernetes default Pod network.

Common VM YAML configuration:

    interfaces:
    - name: default
      masquerade: {}

    networks:
    - name: default
      pod: {}

Meaning:

    1. VM connects to default Pod network
    2. virt-launcher Pod handles network forwarding
    3. VM internally typically accesses external networks via NAT
    4. External access to VM ports usually requires Service

Suitable for:

    1. Beginner experiments
    2. Regular VM network connectivity testing
    3. Exposing SSH / HTTP via Service
    4. Scenarios where VM does not need to directly access traditional layer-2 networks

---

### 3.2 Service Port Exposure for VM

Since VMs run in virt-launcher Pods, they can expose ports like regular Pods via Service.

Common methods:

    ClusterIP
        Access VM within the cluster.

    NodePort
        Access VM from outside the cluster via NodeIP:NodePort.

    LoadBalancer
        If the cluster has MetalLB or cloud LB, access VM via LoadBalancer IP.

Common exposed ports:

    SSH 22
    HTTP 80
    HTTPS 443
    Custom business ports

Key point:

    Service selector must match the label of the virt-launcher Pod.

Common label:

    kubevirt.io/domain: <vm-name>

Example:

    kubevirt.io/domain: vm-network-demo

---

### 3.3 Multus Multi-NIC

Multus is a Kubernetes multi-network plugin.

By default, a Pod typically has only one main network.

Multus allows Pods to mount additional network interfaces.

For KubeVirt, Multus is commonly used for:

    1. VM multi-NIC
    2. VM access to traditional layer-2 networks
    3. VM distinguishing management, business, and storage networks
    4. Migrating VMs to traditional network architectures
    5. NFV / security gateway / network device VMs

Typical scenarios:

    eth0: Pod default network, used for cluster internal management
    eth1: Business network, accessed via Multus to a specific layer-2 network
    eth2: Storage or dedicated line network

At the beginner stage, first understand:

    Multus is not a built-in capability of KubeVirt.
    Multus is a Kubernetes multi-network capability.
    KubeVirt can use Multus to add additional NICs to VMs.

---

## IV. Differences Between KubeVirt Networking and Regular Pod Networking

| Comparison Item | Regular Pod | KubeVirt VM |
|---|---|---|
| Runtime Object | Container Process | Virtual Machine Guest OS |
| Network Location | Pod Network Namespace | VM Internal NIC + virt-launcher Pod Network |
| Access Method | Service -> PodIP:Port | Service -> virt-launcher Pod -> VM Port |
| Entry Method | kubectl exec | virtctl console / ssh |
| IP View | kubectl get pod -o wide | kubectl get vmi + VM internal ip addr |
| Multi-NIC | Requires Multus | Often combined with Multus |
| Port Listening | Container process listens | Guest OS internal process listens |

Core Differences:

    Applications in regular Pods directly listen on container network.
    Applications in KubeVirt VMs listen on virtual machine Guest OS NICs.
    When accessing VMs via Service, there's an additional layer of virt-launcher and VM network forwarding.

---

## Five. Experiment Plan

This article's experiments are divided into three groups.

### 5.1 Experiment One: Default Pod Network VM

Objective:

    Create a VM using pod network + masquerade.
    Enter console to check VM internal NICs and routing.

### 5.2 Experiment Two: Exposing VM SSH via Service

Objective:

    Understand VM exposure using ClusterIP, NodePort, LoadBalancer.

Notes:

    ClusterIP is mandatory
    NodePort is recommended
    LoadBalancer can be done if MetalLB is installed

### 5.3 Experiment Three: Multus Multi-NIC (Optional)

Objective:

    Install or check Multus.
    Create NetworkAttachmentDefinition.
    Create a VM with a second NIC.
    Check eth0 / eth1 inside VM.

Notes:

    Multus multi-NIC is closer to production complex networks.
    However, it requires CNI plugins, host NICs, and IP planning.
    If the current environment is unsuitable, you can just read and record without forced operation.

---

## Six. Experiment Environment

This article assumes the following environment:

    Kubernetes: v1.31.14
    KubeVirt: v1.4.0
    containerd
    Ubuntu 22.04
    CNI: Calico or other common CNI
    virtctl is installed
    KubeVirt is Available
    CDI is installed
    Longhorn is available
    MetalLB (optional)
    Multus (optional)

Cluster Node Example:

    k8s-master-01     10.0.0.20
    k8s-master-02     10.0.0.21
    k8s-master-03     10.0.0.22
    k8s-worker-01     10.0.0.23
    k8s-worker-02     10.0.0.24
    k8s-worker-03     10.0.0.25

Experiment Namespace:

    kubevirt-network-demo

Experiment VM:

    vm-network-demo

---

## Seven. Pre-Experiment Checks

### 7.1 Check KubeVirt

Execute:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

Requirements:

    KubeVirt Available
    virt-api Running
    virt-controller Running
    virt-handler Running
    virt-operator Running

---

### 7.2 Check virtctl

Execute:

    virtctl version

---

### 7.3 Check CNI

Calico Example:

    kubectl get pods -A -o wide | grep calico

Flannel Example:

    kubectl get pods -A -o wide | grep flannel

Requirements:

    CNI components Running.

---

### 7.4 Check CoreDNS

Execute:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system get svc kube-dns

Requirements:

    CoreDNS Running
    kube-dns Service exists

---

### 7.5 Check Base Network

Create a temporary Pod for testing:

    kubectl run net-precheck \
      --image=busybox:1.36 \
      --restart=Never \
      -it --rm -- sh

Execute inside Pod:

    nslookup kubernetes.default

    wget -qO- https://kubernetes.default.svc --no-check-certificate

If regular Pod network is abnormal, it's not recommended to proceed with VM network experiments.

---

## Eight. Create Experiment Namespace

Create namespace:

    kubectl create namespace kubevirt-network-demo --dry-run=client -o yaml | kubectl apply -f -

Check:

    kubectl get ns kubevirt-network-demo

Create directory:

    mkdir -p /root/k8s-yaml/kubevirt/network-demo

    cd /root/k8s-yaml/kubevirt/network-demo

---

## Nine. Experiment One: Create Default Pod Network VM

This article continues using CirrOS test VM.

Create VM: /think

```yaml
cat <<EOF > vm-network-demo.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-network-demo
  namespace: kubevirt-network-demo
  labels:
    app: vm-network-demo
spec:
  runStrategy: Manual
  template:
    metadata:
      labels:
        app: vm-network-demo
        kubevirt.io/domain: vm-network-demo
    spec:
      terminationGracePeriodSeconds: 0
      domain:
        resources:
          requests:
            memory: 512Mi
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
            ports:
            - name: ssh
              port: 2
            - name: http
              port: 80
      networks:
      - name: default
        pod: {}
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/cirros-container-disk-demo:latest
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            password: kubevirt
            chpasswd:
              expire: false
            ssh_pwauth: true
EOF
```

**Apply:**

```bash
kubectl apply -f vm-network-demo.yaml
```

**Check VM:**

```bash
kubectl -n kubevirt-network-demo get vm
```

**Start VM:**

```bash
virtctl start vm-network-demo -n kubevirt-network-demo
```

**Check:**

```bash
kubectl -n kubevirt-network-demo get vm

kubectl -n kubevirt-network-demo get vmi

kubectl -n kubevirt-network-demo get pods -o wide
```

**Expected:**

```
VM Running
VMI Running
virt-launcher Pod Running
```

---

## 10. Observing VMI and virt-launcher Network

**Check VMI:**

```bash
kubectl -n kubevirt-network-demo get vmi vm-network-demo -o wide
```

**Check VMI Details:**

```bash
kubectl -n kubevirt-network-demo describe vmi vm-network-demo
```

**Check VMI Network Fields:**

```bash
kubectl -n kubevirt-network-demo get vmi vm-network-demo -o yaml | grep -A30 interfaces
```

**Check virt-launcher Pod:**

```bash
kubectl -n kubevirt-network-demo get pods -o wide | grep virt-launcher
```

**Record:**

```
VMI IP
virt-launcher Pod IP
virt-launcher Node
```

**Note:**

```
When using masquerade, the VMI network display may not be a simple one-to-one relationship with the Pod network.
During the initial phase, focus on understanding:
    VM uses the default Pod network to access the cluster.
    External access to VM typically requires exposing ports via a Service.
```

---

## 11. Entering VM to View Network

**Enter Console:**

```bash
virtctl console vm-network-demo -n kubevirt-network-demo
```

**Login:**

```
Username: cirros
Password: kubevirt
```

**After Login, Execute:**

```bash
hostname

ip addr

ip route

cat /etc/resolv.conf

ping -c 3 10.96.0.1
```

**Check Kubernetes Service IP:**

```bash
kubectl get svc kubernetes -n default
```

If the Service IP is not 10.96.0.1, use the actual output.

**Exit Console:**

```
Ctrl + ]
```

**Note:**

```
Inside the VM, you see the Guest OS network.
kubectl shows the VMI and virt-launcher Pod network.
Both sides should be understood.
```

---

## 12. Experiment Two: Exposing VM SSH via ClusterIP

### 12.1 Create ClusterIP Service

**Create:** /think

```bash
cat <<EOF > svc-vm-network-ssh-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: vm-network-ssh
  namespace: kubevirt-network-demo
spec:
  type: ClusterIP
  selector:
    kubevirt.io/domain: vm-network-demo
  ports:
  - name: ssh
    protocol: TCP
    port: 22
    targetPort: 22
EOF
```

Apply:

```bash
kubectl apply -f svc-vm-network-ssh-clusterip.yaml
```

Check Service:

```bash
kubectl -n kubevirt-network-demo get svc vm-network-ssh
```

Check Endpoints:

```bash
kubectl -n kubevirt-network-demo get endpoints vm-network-ssh
```

Check Details:

```bash
kubectl -n kubevirt-network-demo describe svc vm-network-ssh
```

Expected:

Endpoints should not be empty.

If Endpoints is empty, check:

- Is the VM Running?
- Is the virt-launcher Pod Running?
- Is the Service selector correct?
- Does the Pod label include kubevirt.io/domain=vm-network-demo?

Check Pod label:

```bash
kubectl -n kubevirt-network-demo get pods --show-labels
```

---

### 12.2 SSH Port Testing Within Cluster

Create temporary test Pod:

```bash
kubectl run ssh-test \
  -n kubevirt-network-demo \
  --image=busybox:1.36 \
  --restart=Never \
  -it --rm -- sh
```

Execute inside test Pod:

```bash
nc -vz vm-network-ssh.kubevirt-network-demo.svc.cluster.local 22
```

If busybox does not have nc, try:

```bash
telnet vm-network-ssh.kubevirt-network-demo.svc.cluster.local 22
```

Note:

- This primarily verifies if TCP port 22 is reachable.
- Actual SSH login requires an ssh client.

---

## ThirteenI don't know.Experiment Three: Exposing VM SSH Using NodePort

### 13.1 Create NodePort Service

Create:

```bash
cat <<EOF > svc-vm-network-ssh-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: vm-network-ssh-nodeport
  namespace: kubevirt-network-demo
spec:
  type: NodePort
  selector:
    kubevirt.io/domain: vm-network-demo
  ports:
  - name: ssh
    protocol: TCP
    port: 22
    targetPort: 22
    nodePort: 30024
EOF
```

Apply:

```bash
kubectl apply -f svc-vm-network-ssh-nodeport.yaml
```

Check:

```bash
kubectl -n kubevirt-network-demo get svc
```

```bash
kubectl -n kubevirt-network-demo get endpoints vm-network-ssh-nodeport
```

---

### 13.2 SSH Access to VM from Outside Cluster

Execute from operations machine or any machine that can access node IP:

```bash
ssh cirros@10.0.0.23 -p 30024
```

Password:

```
kubevirt
```

If 10.0.0.23 is unreachable, try other Node IPs:

```bash
ssh cirros@10.0.0.24 -p 30024
```

```bash
ssh cirros@10.0.0.25 -p 30024
```

Note:

- NodePort exposes the port on all nodes by default.
- Whether it's accessible also depends on firewall, security group, routing, and kube-proxy status.

---

### 13.3 Troubleshooting NodePort Issues

Check Service:

```bash
kubectl -n kubevirt-network-demo describe svc vm-network-ssh-nodeport
```

Check Endpoints:

```bash
kubectl -n kubevirt-network-demo get endpoints vm-network-ssh-nodeport
```

Check kube-proxy:

```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
```

If using IPVS:

```bash
sudo ipvsadm -Ln | grep 30024
```

Check firewall:

```bash
sudo ufw status
```

Check sshd in VM:

```bash
virtctl console vm-network-demo -n kubevirt-network-demo
```

VM internal execution:

```bash
ps aux | grep ssh
```

```bash
netstat -lntp
```

Common causes:

1. VM is not Running
2. Service Endpoints is empty
3. SSH service is not started
4. NodePort is blocked by firewall
5. kube-proxy is abnormal
6. Accessed incorrect node IP or port
7. VM user password is not effective

---

## FourteenI don't know.Experiment Four: Exposing VM SSH Using LoadBalancer (Optional)

Prerequisite:

Cluster has MetalLB installed.  
IPAddressPool is configured.  
LoadBalancer Service can assign EXTERNAL-IP.

Create:

    cat <<EOF > svc-vm-network-ssh-lb.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: vm-network-ssh-lb
      namespace: kubevirt-network-demo
    spec:
      type: LoadBalancer
      selector:
        kubevirt.io/domain: vm-network-demo
      ports:
      - name: ssh
        protocol: TCP
        port: 22
        targetPort: 22
    EOF

Apply:

    kubectl apply -f svc-vm-network-ssh-lb.yaml

Check:

    kubectl -n kubevirt-network-demo get svc vm-network-ssh-lb

Expected:

    EXTERNAL-IP is assigned to MetalLB address pool IP.

Example:

    10.0.0.100

Access:

    ssh cirros@10.0.0.100 -p 22

Password:

    kubevirt

If EXTERNAL-IP remains Pending, first troubleshoot MetalLB:

    kubectl -n metallb-system get pods -o wide

    kubectl -n metallb-system get ipaddresspool

    kubectl -n metallb-system get l2advertisement

---

## FifteenI don't know.Experiment Five: Expose VM HTTP, Understand Methods

If HTTP service runs inside VM, expose port 80.

CirrOS may not have complete HTTP service by default.

This section only records methods, which are more suitable for verification when switching to Ubuntu/Rocky Linux VM later.

Service Example:

    cat <<EOF > svc-vm-network-http-nodeport.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: vm-network-http-nodeport
      namespace: kubevirt-network-demo
    spec:
      type: NodePort
      selector:
        kubevirt.io/domain: vm-network-demo
      ports:
      - name: http
        protocol: TCP
        port: 80
        targetPort: 80
        nodePort: 30080
    EOF

Apply:

    kubectl apply -f svc-vm-network-http-nodeport.yaml

Access:

    curl http://10.0.0.23:30080

Note:

    Service configuration does not guarantee VM's port 80 has service.
    If VM has no application listening on port 80, access will fail.
    Troubleshoot by entering VM to confirm port listening.

---

## SixteenI don't know.Core Logic of KubeVirt Service Exposure

Key to exposing VM is selector.

Example:

    selector:
      kubevirt.io/domain: vm-network-demo

It matches corresponding virt-launcher Pod.

Check:

    kubectl -n kubevirt-network-demo get pods --show-labels

If you see:

    kubevirt.io/domain=vm-network-demo

Service can find backend via this label.

Link:

    Client
      |
      v
    Service ClusterIP / NodePort / LoadBalancer
      |
      v
    virt-launcher Pod
      |
      v
    VM Guest OS Port

Troubleshoot by checking:

    Service
    Endpoints
    virt-launcher Pod
    VM internal port

---

## SeventeenI don't know.What is Multus

Multus is Kubernetes multi-network plugin.

Default Kubernetes Pod usually has only one main network:

    eth0

With Multus, Pod can mount multiple networks:

    eth0: default CNI network
    net1: second network interface
    net2: third network interface

For KubeVirt:

    VM can have multiple network interfaces via Multus.

Common uses:

    1. Separating management network and business network
    2. VM access traditional layer-2 network
    3. VM migration from traditional virtualization network model
    4. Security device VM multi-network interface forwarding
    5. Network function virtualization scenarios

---

## EighteenI don't know.Precautions Before Multus Practice

Multus practice should be cautious.

Reasons:

    1. Involves host CNI plugin
    2. Involves NetworkAttachmentDefinition
    3. Involves node network interface name
    4. Involves IP address planning
    5. May affect node network
    6. Different behaviors for macvlan/bridge/vlan modes
    7. Need network team confirmation in production environment

Experimental suggestions:

    1. First test in test cluster
    2. Do not directly operate production node main network interface
    3. IP address should not conflict with existing hosts
    4. First create isolated bridge network
    5. Then consider connecting to real layer-2 network
    6. Prepare rollback and cleanup

---

## NineteenI don't know.Check if Multus is Installed

Execute:

    kubectl get pods -A | grep -i multus

Check CRD:

    kubectl get crd | grep network-attachment

Check API resources:

    kubectl api-resources | grep NetworkAttachmentDefinition

If you see:

network-attachment-definitions.k8s.cni.cncf.io

Note that the Multus-related CRD already exists.

Check NAD:

    kubectl get network-attachment-definitions -A

Or shorthand:

    kubectl get net-attach-def -A

If the command is unavailable, it indicates Multus is not installed or the CRD does not exist.

---

## 20. Install Multus (Optional for Experimental Environments)

Note:

    Multus installation method and version must align with the cluster environment.
    Do not directly copy public manifests for production environments.
    Verify the version in a test environment first, then consolidate it to internal repositories.

Common approach for experimental environments:

    kubectl apply -f <multus-daemonset-yaml>

If your company or experimental environment already has a Multus installation package, prioritize using the internal version.

Post-installation checks:

    kubectl get pods -A | grep -i multus

    kubectl get crd | grep network-attachment

    kubectl get net-attach-def -A

Requirements:

    Multus DaemonSet Pod Running
    NetworkAttachmentDefinition CRD exists

---

## 21. Check CNI Plugin Binaries

Multus requires calling specific CNI plugins, such as:

    bridge
    macvlan
    host-local
    whereabouts
    vlan
    ipvlan

Check on nodes:

    ls -l /opt/cni/bin/

Focus on:

    bridge
    macvlan
    host-local

If any required plugins are missing, the corresponding NAD will fail.

Common errors:

    failed to find plugin "bridge" in path
    failed to find plugin "macvlan" in path
    failed to find plugin "host-local" in path

---

## 22. Experiment 6: Create an Isolated Bridge Layer-2 Network (Optional)

This experiment creates a simple bridge network, suitable for beginners to understand Multus.

Features:

    1. Does not directly use the host's physical network interface
    2. Relatively low risk
    3. Suitable for observing the second network interface of a VM
    4. May not support cross-node communication, depending on the network plugin and topology

Create NetworkAttachmentDefinition:

    cat <<EOF > nad-bridge-net.yaml
    apiVersion: k8s.cni.cncf.io/v1
    kind: NetworkAttachmentDefinition
    metadata:
      name: bridge-net
      namespace: kubevirt-network-demo
    spec:
      config: '{
        "cniVersion": "0.3.1",
        "name": "bridge-net",
        "type": "bridge",
        "bridge": "br-kv-demo",
        "isGateway": true,
        "ipMasq": true,
        "ipam": {
          "type": "host-local",
          "subnet": "10.200.0.0/24",
          "rangeStart": "10.200.0.100",
          "rangeEnd": "10.200.0.200",
          "gateway": "10.200.0.1"
        }
      }'
    EOF

Apply:

    kubectl apply -f nad-bridge-net.yaml

Check:

    kubectl -n kubevirt-network-demo get net-attach-def

    kubectl -n kubevirt-network-demo describe net-attach-def bridge-net

Note:

    This is an experimental bridge network.
    Primarily used to give VMs an additional network interface and obtain a second IP.
    Not recommended for production use.

---

## 23. Experiment 7: Create a VM with a Second Network Interface (Optional)

Create VM: /think

```bash
cat <<EOF > vm-multus-bridge-demo.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-multus-bridge-demo
  namespace: kubevirt-network-demo
  labels:
    app: vm-multus-bridge-demo
spec:
  runStrategy: Manual
  template:
    metadata:
      labels:
        app: vm-multus-bridge-demo
        kubevirt.io/domain: vm-multus-bridge-demo
    spec:
      terminationGracePeriodSeconds: 0
      domain:
        resources:
          requests:
            memory: 512Mi
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
          - name: bridge-net
            bridge: {}
      networks:
      - name: default
        pod: {}
      - name: bridge-net
        multus:
          networkName: bridge-net
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/cirros-container-disk-demo:latest
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            password: kubevirt
            chpasswd:
              expire: false
            ssh_pwauth: true
EOF
```

**Apply:**

```bash
kubectl apply -f vm-multus-bridge-demo.yaml
```

**Start:**

```bash
virtctl start vm-multus-bridge-demo -n kubevirt-network-demo
```

**Check:**

```bash
kubectl -n kubevirt-network-demo get vm

kubectl -n kubevirt-network-demo get vmi

kubectl -n kubevirt-network-demo get pods -o wide
```

**Enter console:**

```bash
virtctl console vm-multus-bridge-demo -n kubevirt-network-demo
```

**Login:**

```
Username: cirros
Password: kubevirt
```

**VM internal check:**

```bash
ip addr

ip route
```

**Expected:**

- You can see the default network interface
- You may see the second network interface

If the second network interface does not automatically obtain an IP address, it may be due to issues with the Guest OS, DHCP, CNI IPAM, or image behavior.

You can further check the VMI:

```bash
kubectl -n kubevirt-network-demo describe vmi vm-multus-bridge-demo

kubectl -n kubevirt-network-demo get vmi vm-multus-bridge-demo -o yaml | grep -A50 interfaces
```

---

## 24. Experiment 8: macvlan Access to the Host's Layer 2 Network, Cautionary Optional

This experiment is closer to traditional VM access to physical networks.

**Risk warnings:**

1. Must confirm the node network interface name
2. Must confirm the IP address range is not in use
3. Must avoid conflicts with DHCP address pools
4. In production environments, must have the network team confirm
5. macvlan may default not be able to communicate directly with the host
6. Different switches and security policies may restrict MAC addresses

**Assumptions:**

- Host main network interface: ens33
- Node network segment: 10.0.0.0/24
- Gateway: 10.0.0.1
- Reserved IP for Multus VM: 10.0.0.150-10.0.0.160

First confirm the node network interface:

```bash
ip addr

ip route
```

After confirming that the master is ens33, create the NAD.

**Create:** /think

```bash
cat <<EOF > nad-macvlan-lan.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-lan
  namespace: kubevirt-network-demo
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "macvlan-lan",
    "type": "macvlan",
    "master": "ens33",
    "mode": "bridge",
    "ipam": {
      "type": "host-local",
      "subnet": "10.0.0.0/24",
      "rangeStart": "10.0.0.150",
      "rangeEnd": "10.0.0.160",
      "gateway": "10.0.0.1"
    }
  }'
EOF
```

**Apply:**

```bash
kubectl apply -f nad-macvlan-lan.yaml
```

**Check:**

```bash
kubectl -n kubevirt-network-demo get net-attach-def macvlan-lan
```

**Description:**

This method will connect the VM's second network interface to the 10.0.0.0/24 network.  
Ensure that 10.0.0.150-10.0.0.160 is not already in use by other devices.

**VM Creation Reference:**  
Refer to the bridge-net experiment, and change the second network to:

```yaml
interfaces:
- name: default
  masquerade: {}
- name: macvlan-lan
  bridge: {}

networks:
- name: default
  pod: {}
- name: macvlan-lan
  multus:
    networkName: macvlan-lan
```

**Notes:**  
Macvlan experiments are more susceptible to physical network, virtualization platform, and switch policy restrictions.  
VMware Workstation / vSphere / cloud host environments may require additional policies allowing promiscuous mode, MAC address changes, and spoofing.  
If the network is unreachable, it doesn't necessarily mean KubeVirt has a problem—it could be due to underlying Layer 2 network policy restrictions.

---

## 25. Multus Multi-NIC Troubleshooting Path

If a VM with Multus fails to start or has issues with the second NIC, troubleshoot in this order:

1. Is Multus installed?  
2. Does the NetworkAttachmentDefinition exist?  
3. Is the NAD namespace correct?  
4. Is the VM's networks.multus.networkName correct?  
5. Do the VM's interfaces.name and networks.name match?  
6. Does the CNI plugin binary exist?  
7. Is IPAM able to assign addresses?  
8. Is the host's NIC name correct?  
9. Is the VM scheduled on a node with the corresponding NIC?  
10. virt-launcher Pod Events  
11. VMI Events  
12. Does the VM recognize the NIC internally?  
13. Is the underlying network allowing this Layer 2 access?

**Common Commands:**

```bash
kubectl get pods -A | grep -i multus
kubectl get crd | grep network-attachment
kubectl -n kubevirt-network-demo get net-attach-def
kubectl -n kubevirt-network-demo describe net-attach-def bridge-net
kubectl -n kubevirt-network-demo describe vmi vm-multus-bridge-demo
kubectl -n kubevirt-network-demo describe pod <virt-launcher-pod-name>
kubectl -n kubevirt-network-demo get events --sort-by=.lastTimestamp
```

**Node Checks:**

```bash
ls -l /opt/cni/bin/
ip link
ip addr
```

---

## 26. KubeVirt Network Common Issue: Service Endpoints Are Empty

**Phenomenon:**

```bash
kubectl get endpoints vm-network-ssh -n kubevirt-network-demo
```

**Output:**

```
<none>
```

**Common Causes:**

1. VM is not Running  
2. VMI does not exist  
3. virt-launcher Pod does not exist  
4. Service selector is incorrect  
5. Labels do not match  
6. Namespace is incorrect

**Troubleshooting:**

```bash
kubectl -n kubevirt-network-demo get vm
kubectl -n kubevirt-network-demo get vmi
kubectl -n kubevirt-network-demo get pods --show-labels
kubectl -n kubevirt-network-demo get svc vm-network-ssh -o yaml
```

**Key Check:**

```yaml
selector:
  kubevirt.io/domain: vm-network-demo
```

**Pod Label Existence:**

```yaml
kubevirt.io/domain=vm-network-demo
```

---

## 27. Common Issue Two: Service Has Endpoints But SSH Is Unreachable

**Common Causes:**

1. sshd is not running in the VM  
2. Firewall blocks access in the VM  
3. cloud-init is not configured for password login  
4. Incorrect username or password  
5. Service targetPort is incorrect  
6. NodePort is blocked by firewall  
7. kube-proxy is malfunctioning

**Troubleshooting:**

virtctl console vm-network-demo -n kubevirt-network-demo

VM Internal Execution:

    ps aux | grep ssh

    netstat -lntp

    ip addr

    ip route

Kubernetes Side:

    kubectl -n kubevirt-network-demo describe svc vm-network-ssh-nodeport

    kubectl -n kubevirt-network-demo get endpoints vm-network-ssh-nodeport

Node Side:

    sudo ipvsadm -Ln | grep 30024

    sudo ufw status

---

## 28. Common Issue Three: VM Internal DNS Not Working

VM Internal Execution:

    cat /etc/resolv.conf

    nslookup kubernetes.default.svc.cluster.local

Kubernetes Side Checks:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system get svc kube-dns

    kubectl -n kube-system get endpoints kube-dns

If regular Pod DNS is normal but VM internal DNS is abnormal, continue checking:

    1. VMI Network Configuration
    2. VM Internal /etc/resolv.conf
    3. Guest OS Correctly Obtains DNS
    4. Masquerade Network Normal
    5. NetworkPolicy Restrictions
    6. VM Image Network Initialization Normal

---

## 29. Common Issue Four: VM Can Exit Cluster but External Access Not Working

Common Symptoms:

    VM Internal Ping External Normal
    But External Access VM SSH Not Working

Judgment:

    Outbound Normal Does Not Mean Inbound Normal.

In KubeVirt Default Pod Network Mode, External Access to VM Typically Requires Service.

Checks:

    1. Service Created
    2. Service Type Correct
    3. Endpoints Normal
    4. Port Correct
    5. NodePort / LoadBalancer Reachable
    6. VM Internal Application Listening

Do Not Assume:

    External Can Directly Access VM Internal IP.

---

## 30. Common Issue Five: Multus VM Second NIC No IP

Common Causes:

    1. NAD IPAM Configuration Error
    2. IP Address Pool Exhausted
    3. CNI Plugin Missing
    4. Guest OS Not Automatically Configuring Interface
    5. VM Image Network Initialization Weak
    6. Static IP But Configuration Incomplete
    7. Multus Annotation or NetworkName Error
    8. VM Interfaces and Networks Name Mismatch

Troubleshoot:

    kubectl -n kubevirt-network-demo describe vmi <vm-name>

    kubectl -n kubevirt-network-demo describe pod <virt-launcher-pod-name>

    kubectl -n kubevirt-network-demo get events --sort-by=.lastTimestamp

    kubectl -n kubevirt-network-demo get net-attach-def -o yaml

VM Internal:

    ip addr

    ip link

    dmesg | grep -i virtio

Note:

    CirrOS Is a Minimal System, Network Debugging Capabilities Limited.
    Complex Multi-NIC Experiments Suggest Using Ubuntu / Rocky Linux VM.

---

## 31. Common Issue Six: macvlan Access LAN Then External Not Working

Common Causes:

    1. Master NIC Name Wrong
    2. IP Segment and Gateway Configuration Error
    3. IP Conflict with Existing Devices
    4. Upper Virtualization Platform MAC Address Restriction
    5. Switch Port Security Policy Restriction
    6. macvlan Default Cannot Communicate Directly with Host
    7. VM Scheduled to Node Without ens33
    8. Firewall Block

Troubleshoot:

    kubectl -n kubevirt-network-demo describe pod <virt-launcher-pod-name>

    kubectl -n kubevirt-network-demo describe vmi <vm-name>

Node:

    ip addr

    ip route

    ip link show ens33

VM Internal:

    ip addr

    ip route

    ping Gateway

External Client:

    ping VM Second NIC IP

Note:

    macvlan Experiment Highly Dependent on Underlying Network Environment.
    Do Not Judge Issues Only from Kubernetes Layer.

---

## 32. KubeVirt Network Troubleshooting Standard Path

When VM Network Not Working, Suggest Check in This Order:

    1. Check VM Running
    2. Check VMI Running
    3. Check virt-launcher Pod Running
    4. Check VMI Network Status
    5. Enter VM Check ip addr / ip route
    6. Check Service Existence
    7. Check Endpoints Not Empty
    8. Test ClusterIP
    9. Test NodePort
    10. Test LoadBalancer
    11. Check kube-proxy / IPVS
    12. Check CNI Status
    13. Check CoreDNS
    14. If Using Multus, Check NAD and Multus Pod
    15. If Accessing Layer 2 Network, Check Host NIC and Switch Policy

---

## 33. Standard Command List

### 33.1 KubeVirt Resources

    kubectl -n kubevirt-network-demo get vm

    kubectl -n kubevirt-network-demo describe vm vm-network-demo

kubectl -n kubevirt-network-demo get vmi

kubectl -n kubevirt-network-demo describe vmi vm-network-demo

kubectl -n kubevirt-network-demo get pods -o wide

kubectl -n kubevirt-network-demo logs <virt-launcher-pod-name> --tail=100

---

### 33.2 VM Operations

    virtctl start vm-network-demo -n kubevirt-network-demo

    virtctl stop vm-network-demo -n kubevirt-network-demo

    virtctl restart vm-network-demo -n kubevirt-network-demo

    virtctl console vm-network-demo -n kubevirt-network-demo

---

### 33.3 Service Exposure

    kubectl -n kubevirt-network-demo get svc

    kubectl -n kubevirt-network-demo describe svc vm-network-ssh

    kubectl -n kubevirt-network-demo get endpoints

    kubectl -n kubevirt-network-demo get pods --show-labels

---

### 33.4 NodePort / kube-proxy

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

    kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100

    sudo ipvsadm -Ln

    sudo ipvsadm -Ln --stats

    sudo ufw status

---

### 33.5 DNS / CNI

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system get svc kube-dns

    kubectl -n kube-system get endpoints kube-dns

    kubectl get pods -A -o wide | grep -E "calico|flannel|cilium"

---

### 33.6 Multus

    kubectl get pods -A | grep -i multus

    kubectl get crd | grep network-attachment

    kubectl get net-attach-def -A

    kubectl -n kubevirt-network-demo describe net-attach-def bridge-net

    kubectl -n kubevirt-network-demo describe net-attach-def macvlan-lan

---

### 33.7 VM Internal

    ip addr

    ip route

    cat /etc/resolv.conf

    ping -c 3 <gateway-ip>

    ping -c 3 <service-ip>

    ps aux | grep ssh

    netstat -lntp

    ss -lntp

---

## Thirty-Four, Production Network Design Recommendations

### 34.1 Default Pod Network is Suitable for Basic Management

Default pod network + masquerade is suitable for:

    1. Management access
    2. Simple SSH
    3. Test environments
    4. Internal service access
    5. Quick VM startup verification

However, it may not be sufficient for carrying traditional business networks.

---

### 34.2 Business Networks Should Be Planned Separately

When migrating traditional VMs to KubeVirt, it's often necessary:

    1. Management network
    2. Business network
    3. Storage network
    4. Backup network
    5. Security detection network

Such scenarios typically require Multus.

---

### 34.3 IP Addresses Must Be Uniformly Managed

Do not arbitrarily write IP address ranges in NADs.

Production must confirm:

    1. Whether IP is occupied
    2. Whether it conflicts with DHCP
    3. Whether the gateway is correct
    4. Whether the routing is correct
    5. Whether security policies are open
    6. Whether DNS is configured
    7. Whether firewall allows access

---

### 34.4 Multi-NIC VMs Need Proper Naming and Documentation

It's recommended to record:

    eth0: Management network
    eth1: Business network
    eth2: Storage network

Also record:

    Corresponding NAD name
    Corresponding subnet
    Corresponding VLAN
    Corresponding gateway
    Corresponding security policy
    Corresponding business purpose

---

### 34.5 Do Not Use NodePort as a Long-Term Production Entry Point

NodePort is suitable for:

    1. Experimentation
    2. Temporary debugging
    3. Small-scale testing

Production is more recommended:

    1. LoadBalancer
    2. Ingress
    3. Gateway API
    4. External LB
    5. Dedicated network entry point

---

## Thirty-Five, Interview Answer: How Does KubeVirt VM Network Work

You can answer:

    KubeVirt virtual machines typically run in virt-launcher Pods.
    By default, they can use Kubernetes' Pod network, such as through pod network + masquerade to connect to the cluster network.
    If external access to VMs is needed, it's generally done by exposing VM ports via Kubernetes Service, such as ClusterIP, NodePort, or LoadBalancer.
    Services match virt-launcher Pods via selector and forward traffic to the VM's internal port.
    If the VM needs multiple NICs or access to traditional Layer 2 networks, Multus and NetworkAttachmentDefinition can be combined to add additional network interfaces.

---

## Thirty-Six, Interview Answer: Why Does KubeVirt Need Multus

You can answer:

Kubernetes default Pods typically have a single main network, while virtual machines in traditional scenarios often require multiple network interfaces, such as management networks, business networks, and storage networks separated.
Multus can allow Pods or KubeVirt VMs to mount multiple network interfaces.
In KubeVirt, Multus can be used to add a second or multiple network interfaces to a VM, enabling the VM to access different networks, such as Layer 2 business networks, VLAN networks, or dedicated networks.
Thus, Multus mainly addresses the issue of multiple network interfaces and traditional network access for KubeVirt.

---

## 37. Interview Answer: How to Access KubeVirt VM from Outside

You can answer like this:

    If a VM uses the default Pod network, external access to the VM's internal address is generally not possible.
    A common approach is to expose the VM's port using a Kubernetes Service.
    For example, to expose SSH, create a Service with a selector matching kubevirt.io/domain=<vm-name>, forwarding port 22 to the VM.
    In an experimental environment, NodePort can be used; for bare-metal clusters, MetalLB can be combined with LoadBalancer, and HTTP applications can be integrated with Ingress or Gateway API.

---

## 38. Clean Up Experimental Resources

Stop VM:

    virtctl stop vm-network-demo -n kubevirt-network-demo

    virtctl stop vm-multus-bridge-demo -n kubevirt-network-demo

Delete VM:

    kubectl delete -f vm-network-demo.yaml --ignore-not-found

    kubectl delete -f vm-multus-bridge-demo.yaml --ignore-not-found

Delete Service:

    kubectl delete -f svc-vm-network-ssh-clusterip.yaml --ignore-not-found

    kubectl delete -f svc-vm-network-ssh-nodeport.yaml --ignore-not-found

    kubectl delete -f svc-vm-network-ssh-lb.yaml --ignore-not-found

    kubectl delete -f svc-vm-network-http-nodeport.yaml --ignore-not-found

Delete NAD:

    kubectl delete -f nad-bridge-net.yaml --ignore-not-found

    kubectl delete -f nad-macvlan-lan.yaml --ignore-not-found

Check confirmation:

    kubectl -n kubevirt-network-demo get vm

    kubectl -n kubevirt-network-demo get vmi

    kubectl -n kubevirt-network-demo get pods

    kubectl -n kubevirt-network-demo get svc

    kubectl -n kubevirt-network-demo get net-attach-def

Delete namespace:

    kubectl delete namespace kubevirt-network-demo

Note:

    If there are PVCs, DataVolumes, or other test resources in the namespace, deleting the namespace will clean them up together.
    Do not directly delete namespaces containing business VMs in production environments.

---

## 39. Summary of This Article

This article completes the basic learning and hands-on practice of KubeVirt networking.

Core content:

    1. KubeVirt VM runs in a virt-launcher Pod
    2. VMs can typically use Kubernetes Pod network by default
    3. pod network + masquerade is suitable for entry-level and basic management
    4. External access to VMs is usually achieved through Service exposure
    5. ClusterIP is suitable for cluster internal access
    6. NodePort is suitable for experiments and temporary debugging
    7. LoadBalancer is suitable for exposure combined with MetalLB or cloud LB
    8. Multus is used for multiple network interfaces and access to additional networks
    9. NetworkAttachmentDefinition is used to define additional networks
    10. In multi-network interface scenarios, pay attention to IP, gateway, host network interface, CNI plugin, and underlying network policies

Core chain:

    Client
      |
      v
    Service
      |
      v
    virt-launcher Pod
      |
      v
    VM Guest OS
      |
      v
    VM internal application port

Multus chain:

    VM
      |
      |-- Default Pod network
      |
      |-- Multus second network
              |
              v
          NetworkAttachmentDefinition
              |
              v
          bridge / macvlan / vlan / sriov etc. CNI

Most important commands:

    kubectl get vmi -n kubevirt-network-demo -o wide

    kubectl get pods -n kubevirt-network-demo -o wide --show-labels

    kubectl get svc -n kubevirt-network-demo

    kubectl get endpoints -n kubevirt-network-demo

    virtctl console vm-network-demo -n kubevirt-network-demo

    kubectl get net-attach-def -A

Experience judgment: /think

1. VM Running does not guarantee that business ports are definitely open  
2. Service having ClusterIP does not guarantee that Endpoints are definitely normal  
3. When Endpoints is empty, prioritize checking selector and virt-launcher Pod label  
4. If SSH is unreachable, enter the VM to check if sshd is listening  
5. If NodePort is unreachable, check kube-proxy, firewall, and node network  
6. For LoadBalancer Pending, check MetalLB  
7. For Multus anomalies, check NAD, CNI plugin, IPAM, and VM interface name  
8. When macvlan connects to real Layer 2 network, pay special attention to IP conflicts and underlying switch network limitations  

Next suggested learning:  

    09-KubeVirt Operations Troubleshooting: VM Startup Failure, Image Import Failure, Console and Event Analysis.md