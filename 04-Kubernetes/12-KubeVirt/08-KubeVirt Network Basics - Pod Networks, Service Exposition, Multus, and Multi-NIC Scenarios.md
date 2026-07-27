# 08-KubeVirt Network Basics: Pod Networks, Service Exposition, Multus, and Multi-NIC Scenarios

Recommended Path:

    04-Kubernetes/12-KubeVirt/08-KubeVirt Network Basics: Pod Networks, Service Exposition, Multus, and Multi-NIC Scenarios.md

Tags:

    #Kubernetes
    #KubeVirt
    #Network
    #Pod Network
    #Service
    #NodePort
    #LoadBalancer
    #MetalLB
    #Multus
    #NetworkAttachmentDefinition
    #Multi-NIC
    #Virtual Machine Network
    #Cloud-Native Virtualization

---

## I. Documentation Overview

This document outlines the fundamental concepts and practical methods of virtual machine networking in KubeVirt.

Previous steps have included:

    1. Installing KubeVirt
    2. Setting up virtctl
    3. Creating a test VM using containerDisk
    4. Importing images via CDI / DataVolume
    5. Experimenting with PVC / Longhorn virtual machine disks

This document continues the exploration of KubeVirt network basics.

Key Points:

    1. Understanding how KubeVirt VMs access the Pod network by default
    2. Comprehending the masquerade network mode
    3. Learning how VMs expose ports via Service
    4. Mastering exposure methods using ClusterIP, NodePort, and LoadBalancer
    5. Distinguishing between VM networks and regular Pod networks
    6. Grasping the role of Multus
    7. Exploring multi-NIC scenarios for VMs
    8. Completing an experiment with a default Pod network VM
    9. Performing an experiment to expose SSH via Service
    10. Optionally conducting an experiment with Multus and two NICs
    11. Familiarizing oneself with common troubleshooting approaches for KubeVirt networks

This document is designed to:

    Provide an entry-level understanding, with a focus on practical troubleshooting.
    It does not delve into advanced topics such as SR-IOV, DPDK, NUMA, complex Layer 2 networks, or high-performance virtualization networking.
    The goal is to build foundational skills that are easy to apply and verify.

---

## II. What to Understand First about KubeVirt Networks

KubeVirt virtual machines ultimately run within a virt-launcher Pod.

From the perspective of Kubernetes:

    The VM operates inside a Pod.

From a networking standpoint:

    The VM's network is first connected to the virt-launcher Pod's network.

Basic Components:

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

Therefore, when troubleshooting KubeVirt networks, one should not only consider the internal settings of the virtual machine but also:

    1. The network configuration of the VM/VMI
    2. The virt-launcher Pod
    3. The Pod's IP address
    4. The status of the CNI
    5. Services and Endpoints
    6. NodePort/LoadBalancer settings
    7. The internal NICs, IP addresses, routing, and firewall configurations of the VM Guest OS

---

## III. Common KubeVirt Network Modes

At the introductory level, focus on three main modes:

    1. pod network + masquerade
    2. Service for port exposure
    3. Multus for multiple NICs

### 3.1 pod network + masquerade

This is the most commonly used method at the beginning.

VMs use Kubernetes' default Pod network.

Common configuration in YAML:

    interfaces:
    - name: default
      masquerade: {}

    networks:
    - name: default
      pod: {}

Meaning:

    1. The VM connects to the default Pod network.
    2. The virt-launcher Pod handles network forwarding.
    3. Internal access to external resources within the VM typically uses NAT.
    4. External access to VM ports requires using Service.

Suitable for:

    1. Beginner experiments
    2. Testing basic network connectivity of VMs
    3. Exposing SSH/HTTP services via Service
    4. Scenarios where direct connection to traditional Layer 2 networks is not required

---

### 3.2 Service for Port Exposure

Since VMs run within virt-launcher Pods, they can expose ports just like regular Pods using Service.

Common methods:

    **ClusterIP**: Access the VM from within the cluster.
    **NodePort**: Access the VM from outside the cluster using NodeIP:NodePort.
    **LoadBalancer**:kubevirt-network-demo

Experimental VM:

    vm-network-demo

---

## VII. Pre-experiment Checks

### 7.1 Check KubeVirt

Execute:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

Requirements:

    KubeVirt must be available.
    virt-api, virt-controller, virt-handler, and virt-operator must all be running.

---

### 7.2 Check virtctl

Execute:

    virtctl version

---

### 7.3 Check CNI

For Calico example:

    kubectl get pods -A -o wide | grep calico

For Flannel example:

    kubectl get pods -A -o wide | grep flannel

Requirements:

    The CNI components must be running.

---

### 7.4 Check CoreDNS

Execute:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system get svc kube-dns

Requirements:

    CoreDNS must be running, and the kube-dns Service must exist.

---

### 7.5 Check Basic Networking

Create a temporary Pod for testing:

    kubectl run net-precheck \
      --image=busybox:1.36 \
      --restart=Never \
      -it --rm -- sh

Inside the Pod, execute:

    nslookup kubernetes.default

    wget -qO- https://kubernetes.default.svc --no-check-certificate

If the regular Pod's network is not functioning properly, it is not recommended to proceed with the VM networking experiments.

---

## VIII. Create an Experimental Namespace

Create a namespace:

    kubectl create namespace kubevirt-network-demo --dry-run=client -o yaml | kubectl apply -f -

Verify its creation:

    kubectl get ns kubevirt-network-demo

Create a directory:

    mkdir -p /root/k8s-yaml/kubevirt/network-demo

    cd /root/k8s-yaml/kubevirt/network-demo

---

## IX. Experiment 1: Create a Default Pod Network VM

This experiment continues to use the CirrOS test VM.

Create the VM configuration file:

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
                  port: 22
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

Apply the configuration:

    kubectl apply -f vm-network-demo.yaml

Check the created VM:

    kubectl -n kubevirt-network-demo get vm

Start the VM:

    virtctl start vm-network-demo -n kubevirt-network-demo

Verify its status:

    kubectl -n kubevirt-network-demo get vm

    kubectl -n kubevirt-network-demo get vmi

    kubectl -n kubevirt-network-demo get pods -o wide

Expected results:

    The VM must be running.
    The VMI must be running.
    The virt-launcher Pod must be running.

---

## X. Observe the Networks of the VMI and virt-launcher

Check the VMI's network settings:

    kubectl -n kubevirt-network-demo get vmi vm-network-demo -o wide

View detailed information about the VMI:

    kubectl -n kubevirt-network-demo describe vmi vm-network-demo

Examine the VMI's network fields:

    kubectl -n kubevirt-network-demo get vmi vm-network-demo -o yaml | grep -A30 interfaces

Check the virt-launcher Pod:

    kubectl -n kubevirt-network-demo get pods -o wide | grep virt-launcher

Record the following details:

    VMI IP address
    virt-launcher Pod IP      name: vm-network-ssh
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

Application:

    kubectl apply -f svc-vm-network-ssh-clusterip.yaml

View the Service:

    kubectl -n kubevirt-network-demo get svc vm-network-ssh

View the Endpoints:

    kubectl -n kubevirt-network-demo get endpoints vm-network-ssh

View details:

    kubectl -n kubevirt-network-demo describe svc vm-network-ssh

Expectation:

    The Endpoints should not be empty.

If the Endpoints are empty, check:

    Whether the VM is Running.
    Whether the virt-launcher Pod is Running.
    Whether the Service selector is correct.
    Whether the Pod label contains kubevirt.io/domain=vm-network-demo.

View the Pod labels:

    kubectl -n kubevirt-network-demo get pods --show-labels

---

### 12.2 Testing the SSH Port Within the Cluster

Create a temporary test Pod:

    kubectl run ssh-test \
      -n kubevirt-network-demo \
      --image=busybox:1.36 \
      --restart=Never \
      -it --rm -- sh

Execute inside the test Pod:

    nc -vz vm-network-ssh.kubevirt-network-demo.svc.cluster.local 22

If busybox does not have nc, you can try:

    telnet vm-network-ssh.kubevirt-network-demo.svc.cluster.local 22

Explanation:

    This mainly verifies whether the TCP port 22 is reachable.
    For actual SSH login, an ssh client is required.

---

## Section Thirteen: Experiment Three: Exposing VM SSH Using NodePort

### 13.1 Creating a NodePort Service

Create the configuration file:

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

Apply the configuration:

    kubectl apply -f svc-vm-network-ssh-nodeport.yaml

View the Service:

    kubectl -n kubevirt-network-demo get svc

    kubectl -n kubevirt-network-demo get endpoints vm-network-ssh-nodeport

---

### 13.2 Accessing the VM via SSH from Outside the Cluster

Execute from an operations machine or any machine that can access the node IP address:

    ssh cirros@10.0.0.23 -p 30024

Password:

    kubevirt

If 10.0.0.23 is unreachable, try another node IP:

    ssh cirros@10.0.0.24 -p 30024

    ssh cirros@10.0.0.25 -p 30024

Explanation:

    NodePort exposes the port on all nodes by default.
    Whether it is actually accessible depends on the firewall, security groups, routing settings, and the status of kube-proxy.

---

### 13.3 Troubleshooting if NodePort Is Unreachable

View the Service:

    kubectl -n kubevirt-network-demo describe svc vm-network-ssh-nodeport

View the Endpoints:

    kubectl -n kubevirt-network-demo get endpoints vm-network-ssh-nodeport

Check kube-proxy:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

If using IPVS:

    sudo ipvsadm -Ln | grep 30024

Check the firewall:

    sudo ufw status

Check sshd inside the VM:

    virtctl console vm-network-demo -n kubevirt-network-demo

Execute inside the VM:

    ps aux | grep ssh

    netstat -lntp

Common issues:

    1. The VM is not Running.
    2. The Service endpoints are empty.
    3. The SSH service is not started.
    4. The NodePort is blocked by the firewall.
    5. There is an issue with kube-proxy.
    6. The wrong node IP or port was accessed.
    7. The user password inside the VM is not valid.

---

## Section Fourteen: Experiment Four: Exposing VM SSH Using LoadBalancer (Optional)

Prerequisites:

    MetalLB must be installed in the cluster.
If no application inside the VM is listening on port 80, access will fail. When troubleshooting, you need to check whether the port is being monitored within the VM.bridge: {}
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

Application:

    kubectl apply -f vm-multus-bridge-demo.yaml

Start:

    virtctl start vm_multus-bridge-demo -n kubevirt-network-demo

View:

    kubectl -n kubevirt-network-demo get vm

    kubectl -n kubevirt-network-demo get vmi

    kubectl -n kubevirt-network-demo get pods -o wide

Enter console:

    virtctl console vm-multus-bridge-demo -n kubevirt-network-demo

Login:

    Username: cirros
    Password: kubevirt

View inside the VM:

    ip addr

    ip route

Expected results:

    The default network card should be visible.
    The second network card might also appear.

If the second network card does not automatically obtain an IP address, there could be issues with the Guest OS, DHCP, CNI IPAM, or the image configuration. You can further investigate by checking the VMI details:

    kubectl -n kubevirt-network-demo describe vmi vm-multus-bridge-demo

    kubectl -n kubevirt-network-demo get vmi vm_multus-bridge-demo -o yaml | grep -A50 interfaces

---

## Experiment 24: Macvlan Connecting to the Host's Layer 2 Network (Optional with Caution)

This experiment is more similar to how traditional VMs connect to physical networks.

Risk Notes:

    1. You must confirm the name of the node's network card.
    2. Ensure that the IP address range is not already in use.
    3. Avoid any conflicts with the DHCP address pool.
    4. In a production environment, get approval from the network team.
    5. By default, macvlan may not be able to communicate directly with the host.
    6. Different switches and security policies might restrict MAC addresses.

Assumptions:

    Host's primary network card: ens33
    Node IP range: 10.0.0.0/24
    Gateway: 10.0.0.1
    Reserved IP range for the Multus VM: 10.0.0.150-10.0.0.160

First, confirm the node's network card:

    ip addr

    ip route

After confirming that the master network card is ens33, create a NAD:

Create the configuration file:

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

Apply the configuration:

    kubectl apply -f nad-macvlan-lan.yaml

Check the configuration:

    kubectl -n kubevirt-network-demo get net-attach-def macvlan-lan

Note:

    This configuration will allow the VM's second network card to connect to the 10.0.0.0/24 network.
    Make sure that the reserved IP range (10.0.0.150-10.0.0.160) is not already in use by other devices.

To create the VM, refer to the bridge-net experiment and modify the interfaces section as follows:

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

Note:

    The macvlan experiment can be more susceptible to issues related to the physical network, virtualization platform, and switch      kubevirt.io/domain: vm-network-demo

Does the Pod label exist?

    kubevirt.io/domain=vm-network-demo

---

## Issue 27: Service has Endpoints but SSH access is unavailable

Common causes:

    1. The sshd service inside the VM is not running.
    2. The firewall inside the VM is blocking the connection.
    3. Password login is not enabled in cloud-init.
    4. Incorrect username or password.
    5. Wrong value for Service's targetPort.
    6. NodePort is blocked by the firewall.
    7. Issues with kube-proxy.

Troubleshooting:

    Use virtctl console vm-network-demo -n kubevirt-network-demo to enter the VM.

Inside the VM, execute the following commands:

    ps aux | grep ssh
    netstat -lntp
    ip addr
    ip route

On the Kubernetes side:

    kubectl -n kubevirt-network-demo describe svc vm-network-ssh-nodeport
    kubectl -n kubevirt-network-demo get endpoints vm-network-ssh-nodeport

On the node side:

    sudo ipvsadm -Ln | grep 30024
    sudo ufw status

---

## Issue 28: DNS is not working inside the VM

Inside the VM, execute the following commands:

    cat /etc/resolv.conf
    nslookup kubernetes.default.svc.cluster.local

On the Kubernetes side, check:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
    kubectl -n kube-system get svc kube-dns
    kubectl -n kube-system get endpoints kube-dns

If the DNS is working for regular Pods but not inside the VM, continue to check:

    1. VMI network configuration.
    2. The /etc/resolv.conf file inside the VM.
    3. Whether the guest OS is correctly obtaining DNS information.
    4. Whether the masquerade network is functioning properly.
    5. If any NetworkPolicy rules are restricting access.
    6. Whether the network initialization of the VM image is correct.

---

## Issue 29: The VM can exit the cluster but cannot be accessed from outside

Common symptoms:

    Ping requests from inside the VM to external addresses work normally.
    However, SSH access from outside to the VM is blocked.

Note:

    Just because outgoing traffic is working does not mean incoming traffic will be successful.

By default, in KubeVirt's Pod network mode, external access to a VM usually requires the use of a Service.

Check the following:

    1. Whether a Service has been created.
    2. Whether the Service type is correct.
    3. Whether the Endpoints are functioning properly.
    4. Whether the port numbers are correct.
    5. Whether NodePort or LoadBalancer can be reached.
    6. Whether applications inside the VM are listening on the appropriate ports.

Do not assume that external access can be established directly using the internal IP address of the VM.

---

## Issue 30: The second network card of a Multus VM does not have an IP address

Common reasons:

    1. Incorrect NAD IPAM configuration.
    2. Exhaustion of the available IP address pool.
    3. Missing CNI plugins.
    4. The guest OS did not automatically configure interfaces.
    5. Weak network initialization in the VM image.
    6. Use of a static IP address with incomplete configuration.
    7. Incorrect multus annotation or networking name settings.
    8. Mismatch between the names of VM interfaces and networks.

Troubleshooting:

    kubectl -n kubevirt-network-demo describe vmi <vm-name>
    kubectl -n kubevirt-network-demo describe pod <virt-launcher-pod-name>
    kubectl -n kubevirt-network-demo get events --sort-by=.lastTimestamp
    kubectl -n kubevirt-network-demo get net-attach-def -o yaml

Inside the VM:

    ip addr
    ip link
    dmesg | grep -i virtio

Note:

    CirrOS is a minimal system, and its network debugging capabilities are limited.
    For more complex experiments with multiple network cards, it is recommended to use Ubuntu or Rocky Linux VMs.

---

## Issue 31: After connecting a macvlan interface to the LAN, external access is unavailable

Common reasons:

    1. Incorrect name for the master network card.
    2. Incorrect IP range and gateway settings.
    3. IP address conflicts with existing devices.
    4. Restrictions imposed by the underlying virtualization platform on MAC addresses.
    5. Security policies on the switch ports blocking access.
    6. By default, macvlan does not allow direct communication with the host machine.
    7. The VM is scheduled to a node that```bash
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system get endpoints kube-dns
kubectl get pods -A -o wide | grep -E "calico|flannel|cilium"
```

---

### 33.6 Multus

```bash
kubectl get pods -A | grep -i multus
kubectl get crd | grep network-attachment
kubectl get net-attach-def -A
kubectl -n kubevirt-network-demo describe net-attach-def bridge-net
kubectl -n kubevirt-network-demo describe net-attach-def macvlan-lan
```

---

### 33.7 VM 内部

```bash
ip addr
ip route
cat /etc/resolv.conf
ping -c 3 <gateway-ip>
ping -c 3 <service-ip>
ps aux | grep ssh
netstat -lntp
ss -lntp
```

---

## 三十四、生产环境网络设计建议

### 34.1 默认 Pod 网络适合基础管理

默认 pod network + masquerade 适合：

    1. 管理访问
    2. 简单 SSH
    3. 测试环境
    4. 内部服务访问
    5. 快速验证 VM 启动

但如果要承载传统业务网络，可能不够。

---

### 34.2 业务网络建议单独规划

传统 VM 迁移到 KubeVirt 时，经常需要：

    1. 管理网
    2. 业务网
    3. 存储网
    4. 备份网
    5. 安全检测网

这类场景通常需要 Multus。

---

### 34.3 IP 地址必须统一管理

不要随便在 NAD 里写 IP 地址段。

生产必须确认：

    1. IP 是否被占用
    2. 是否和 DHCP 冲突
    3. 网关是否正确
    4. 路由是否正确
    5. 安全策略是否放通
    6. DNS 是否配置
    7. 防火墙是否允许

---

### 34.4 多网卡 VM 要做好命名和文档

建议记录：

    eth0：管理网
    eth1：业务网
    eth2：存储网

同时记录：

    对应 NAD 名称
    对应网段
    对应 VLAN
    对应网关
    对应安全策略
    对应业务用途

---

### 34.5 不要把 NodePort 当长期生产入口

NodePort 适合：

    1. 实验
    2. 临时调试
    3. 小规模测试

生产更建议：

    1. LoadBalancer
    2. Ingress
    3. Gateway API
    4. 外部 LB
    5. 专用网络入口

---

## 三十五、面试回答：KubeVirt VM 网络怎么通

可以这样回答：

    KubeVirt 虚拟机通常运行在 virt-launcher Pod 中。
    默认情况下可以使用 Kubernetes 的 Pod 网络，例如通过 pod network + masquerade 让 VM 接入集群网络。
    如果需要从外部访问 VM，一般通过 Kubernetes Service 暴露 VM 的端口，比如 ClusterIP、NodePort 或 LoadBalancer。
    Service 会通过 selector 匹配 virt-launcher Pod，再转发到 VM 内部端口。
    如果虚拟机需要多网卡或接入传统二层网络，可以结合 Multus 和 NetworkAttachmentDefinition 给 VM 增加额外网络接口。

---

## 三十六、面试回答：KubeVirt 为什么需要 Multus

可以这样回答：

    Kubernetes 默认 Pod 通常只有一个主网络，而虚拟机在传统场景里经常需要多张网卡，例如管理网、业务网、存储网分离。
    Multus 可以让 Pod 或 KubeVirt VM 挂载多个网络接口。
    在 KubeVirt 中，可以通过 Multus 给 VM 添加第二张或多张网卡，使 VM 接入不同网络，例如二层业务网络、VLAN 网络或专用网络。
    所以 Multus 主要解决 KubeVirt 多网卡和传统网络接入的问题。

---

## 三十七、面试回答：KubeVirt VM 外部访问怎么做

可以这样回答：

    如果 VM 使用默认 Pod 网络，外部一般不能直接访问 VM 内部地址。
    常见做法是使用 Kubernetes Service 暴露 VM 端```bash
kubectl get vmi -n kubevirt-network-demo -o wide

kubectl get pods -n kubevirt-network-demo -o wide --show-labels

kubectl get svc -n kubevirt-network-demo

kubectl get endpoints -n kubevirt-network-demo

virtctl console vm-network-demo -n kubevirt-network-demo

kubectl get net-attach-def -A
```

**Experience and Judgment:**

1. Just because a VM is running does not necessarily mean that its business ports are accessible.
2. The presence of a ClusterIP for a Service does not guarantee that the corresponding Endpoints are functioning correctly.
3. If Endpoints are empty, it is necessary to check the selector fields and the labels of the virt-launcher Pod first.
4. If SSH access fails, you need to log into the VM to verify whether the sshd service is running.
5. If a NodePort is not accessible, examine kube-proxy settings, firewalls, and the node's network configuration.
6. For LoadBalancers in the "Pending" state, check the status of MetalLB.
7. In case of Multus-related issues, investigate NAD, CNI plugins, IPAM configurations, and the names of VM interfaces.
8. When using macvlan to connect to a physical Layer 2 network, be cautious of potential IP address conflicts and underlying switch limitations.

**Next Step: Continue Learning…**

09-KubeVirt Operation and Maintenance Troubleshooting: VM Startup Failures, Image Import Errors, Console and Event Analysis.md