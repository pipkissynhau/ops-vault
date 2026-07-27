# 02-kubeadm High-Availability Cluster Deployment Practice: External etcd + kube-vip + containerd (Ubuntu 22.04)

Recommended Path:

    04-Kubernetes/08-Operations/01-Cluster Deployment and High Availability/02-kubeadm High-Availability Cluster Deployment Practice: External etcd + kube-vip + containerd (Ubuntu 22.04).md

Tags:

    #Kubernetes
    #kubeadm
    #High-Availability Cluster
    #External-etcd
    #kube-vip
    #containerd
    #IPVS
    #Ubuntu2204
    #Domestic Environment
    #Cluster Deployment

---

## I. Document Purpose

This document outlines a process for deploying a highly available Kubernetes cluster using kubeadm.

This solution is designed for domestic environments and utilizes an External etcd architecture:

    Ubuntu Server 22.04
    containerd
    kubeadm / kubelet / kubectl
    Alibaba Cloud Docker CE repository
    Alibaba Cloud Kubernetes repository
    Alibaba Cloud google_containers image repository
    3 Master nodes for high availability control plane
    3 independent External etcd nodes
    kube-vip for providing APIServer VIP
    kube-proxy in IPVS mode
    containerd data directory set to /data/containerd
    etcd data directory set to /data/etcd

The goal of this document is to provide a step-by-step guide for deploying a Kubernetes cluster with External etcd high availability.

---

## II. Node Planning

### 2.1 Cluster Planning

| Item | Planning |
|---|---|
| Operating System | Ubuntu Server 22.04 |
| Kubernetes Version | v1.31.14 |
| Container Runtime | containerd |
| Deployment Tool | kubeadm |
| Control Plane | 3 Master nodes |
| etcd Mode | External etcd |
| etcd Nodes | 3 independent etcd nodes |
| APIServer High Availability | kube-vip |
| APIServer VIP | 10.0.0.30 |
| APIServer Domain Name | k8s-api-server |
| Pod IP Range | 10.244.0.0/16 |
| Service IP Range | 10.96.0.0/12 |
| kube-proxy Mode | IPVS |
| CNI Plugin | Calico |
| containerd Data Directory | /data/containerd |
| etcd Data Directory | /data/etcd |

---

### 2.2 Node IP Planning

| Role | Hostname | IP Address | Notes |
|---|---|---|---|
| Operations Tool Node | ops-server | 10.0.0.10 | Optional for GitLab, Jenkins, Harbor |
| Master 1 | k8s-master-01 | 10.0.0.20 | Control plane node |
| Master 2 | k8s-master-02 | 10.0.0.21 | Control plane node |
| Master 3 | k8s-master-03 | 10.0.0.22 | Control plane node |
| Worker 1 | k8s-worker-01 | 10.0.0.23 | Work node |
| Worker 2 | k8s-worker-02 | 10.0.0.24 | Work node |
| Worker 3 | k8s-worker-03 | 10.0.0.25 | Optional work node |
| APIServer VIP | k8s-api-server | 10.0.0.30 | Floating IP for kube-vip |
| etcd 1 | etcd-01 | 10.0.0.31 | External etcd node |
| etcd 2 | etcd-02 | 10.0.0.32 | External etcd node |
| etcd 3 | etcd-03 | 10.0.0.33 | External etcd node |

Notes:

    10.0.0.30 is not assigned to any physical server; it is used solely as a virtual IP for kube-vip.
    Both `kubeadm init` and `kubeadm join` use `k8s-api-server:6443` to access the control plane.
    etcd-01, etcd-02, and etcd-03 are not added as Kubernetes Worker nodes to the cluster.
    The External etcd nodes are managed by their local kubelets as static Pods.

---

### 2.3 Architecture Diagram

    kubectl / kubelet / Operations Tool
                  |
                  |
5. All nodes have access to domestic software repositories.
6. The IP address 10.0.0.30 is not occupied by any host.
7. It is recommended to mount a separate data disk or a large partition for the /data directory.
8. The Master node can access the etcd node on port 2379.
9. Etcd nodes can communicate with each other using ports 2379 and 2380.

Check node connectivity:

    ping -c 3 10.0.0.20
    ping -c 3 10.0.0.21
    ping -c 3 10.0.0.22
    ping -c 3 10.0.0.23
    ping -c 3 10.0.0.24
    ping -c 3 10.0.0.25
    ping -c 3 10.0.0.31
    ping -c 3 10.0.0.32
    ping -c 3 10.0.0.33

Check the /data directory:

    sudo mkdir -p /data
    df -h /data

Note:

    If /data is not mounted separately but is just a regular directory under the root partition, then containerd and etcd data will still occupy system disk space. In a production environment, it is recommended to mount /data on a separate data disk.

---

## Section 4: Configuring Hostnames and hosts Files for All Nodes

The following steps should be performed on all Master, Worker, and etcd nodes.

Node list:

    k8s-master-01
    k8s-master-02
    k8s-master-03
    k8s-worker-01
    k8s-worker-02
    k8s-worker-03
    etcd-01
    etcd-02
    etcd-03

---

### 4.1 Setting Hostnames

Perform this on k8s-master-01:

    sudo hostnamectl set-hostname k8s-master-01

Repeat for the other Master nodes and then for the Worker nodes and etcd nodes.

Check the configured hostnames:

    hostname

---

### 4.2 Configuring the /etc/hosts File

All nodes should perform the following steps:

    sudo cp /etc/hosts /etc/hosts.bak.$(date +%F-%H%M%S)

    Create a new /etc/hosts file with the following contents:

    10.0.0.10 ops-server
    10.0.0.20 k8s-master-01
    10.0.0.21 k8s-master-02
    10.0.0.22 k8s-master-03
    10.0.0.23 k8s-worker-01
    10.0.0.24 k8s-worker-02
    10.0.0.25 k8s-worker-03
    10.0.0.30 k8s-api-server
    10.0.0.31 etcd-01
    10.0.0.32 etcd-02
    10.0.0.33 etcd-03

    Check that the hosts are resolved correctly:

    getent hosts k8s-api-server
    getent hosts k8s-master-01
    getent hosts k8s-master-02
    getent hosts k8s-master-03
    getent hosts etcd-01
    getent hosts etcd-02
    getent hosts etcd-03

---

## Section 5: System Initialization for All Nodes

The following steps should be performed on all Master, Worker, and etcd nodes.

---

### 5.1 Configuring Clock Synchronization

Install chrony:

    sudo apt update
    sudo apt install -y chrony

Replace the NTP server configuration to use a domestic source:

    sudo sed -i.bak '/^pool ntp\.ubuntu\.com\|^pool [012]\.ubuntu\.pool\.ntp\.org/s/^/#/;/^#pool ntp\.ubuntu\.com/i\pool ntp.aliyun.com iburst\npool ntp.huaweicloud.com iburst\npool ntp.tencent.com iburst\npool time.windows.com iburst' /etc/chrony/chrony.conf

Start chrony and set it to start automatically at boot:

    sudo systemctl enable --The following operations are performed on all Master, Worker, and etcd nodes.

This document uses the Alibaba Cloud Docker CE repository to install containerd.io.

Note:

    Kubernetes nodes use containerd as the container runtime.
    This document only installs containerd.io; it does not install the full Docker Engine.
    The containerd data directory is uniformly set to /data/containerd.

---

### 6.1 Configuring the Alibaba Cloud Docker CE Repository

Create a keyrings directory:

    sudo install -m 0755 -d /etc/apt/keyrings

Add the Docker GPG key:

    curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | \
      sudo gpg --dearmor -o /etc/apt/ayrings/docker.gpg

Set permissions:

    sudo chmod a+r /etc/apt/ayrings/docker.gpg

Write the Docker CE software source:

    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/ayrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

Update the apt cache:

    sudo apt-get update

---

### 6.2 Installing containerd.io

Install:

    sudo apt-get install -y containerd.io

Check the version:

    containerd --version

---

### 6.3 Generating Default containerd Configurations

Create a configuration directory:

    sudo mkdir -p /etc/containerd

Generate default configurations:

    containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

Back up the configuration:

    sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak.$(date +%F-%H%M%S)

---

### 6.4 Changing the Containerd Image Storage Location

Create a directory:

    sudo mkdir -p /data/containerd

Modify the containerd root path:

    sudo sed -i.bak 's#^root = "/var/lib/containerd"#root = "/data/containerd"#g' /etc/containerd/config.toml

Check:

    grep -n '^root = ' /etc/containerd/config.toml

Expected result:

    root = "/data/containerd"

Explanation:

    The `root` directory is where containerd stores its persistent data.
    Images, snapshots, metadata, and other important data are stored in this directory.
    The `state` directory is typically `/run/containerd`, which is for runtime state data and generally does not need to be modified.
    It is best to complete this configuration before running `kubeadm init`.

---

### 6.5 Enabling SystemdCgroup

Modify the configuration:

    sudo sed -i.bak 's/SystemdCgroup = false(SystemdCgroup = true/g' /etc/containerd/config.toml

Check:

    grep "SystemdCgroup" /etc/containerd/config.toml

Expected result:

    SystemdCgroup = true

---

### 6.6 Replacing the Pause Image with an Alibaba Cloud Source

Replace the `pause` image in the containerd configuration with an Alibaba Cloud image:

    sudo sed -i.bak -E 's#registry.k8s.io/pause:[0-9.]+#registry.aliyuncs.com/google_containers/pause:3.10#g' /etc/containerd/config.toml

Verify the Sandbox image configuration:

    containerd config dump | grep -E "sandbox_image|pinned_helper_image"

If you still see `registry.k8s.io/pause`, check manually:

    grep -n "pause" /etc/containerd/config.toml

---

### 6.7 Restarting containerd

Restart:

    sudo systemctl restart containerd

Set to start automatically at boot:

    sudo systemctl enable containerd

Check the status:

    systemctl status containerd --no-pager

Check the data directory:

    sudo ls -ld /data/containerd

---

### 6.8 Configuring crictl

Write the crictl configuration:

    cat <<EOF | sudo tee /etc/crictl.yaml
    runtime-endpoint: unix:///run/containerd/containerd.sock
    image-endpoint: unix:///run/containerd/containerd.sock
    timeout: 10
    debug: false
    EOF

Check:

    sudo crictl info

View images:

    sudo crictl images

---

## Section 7: Installing nerdctl on All Nodes

nerdctl is a Docker-style command-line tool for containerd, useful for checking containerd images, containers, and namespaces.

The following operations are performed on all Master, Worker, and etcd nodes.

---

### 7.1 Installing the minimal nerdctl package

Set the version:

    NERDCTL_VERSION=2.2.2

Download:

    cdsudo apt-get update### 9.5 Generating Local etcd Certificates on Each etcd Node

Execute the following commands on etcd-01, etcd-02, and etcd-03:

    sudo kubeadm init phase certs etcd-server --config /root/kubeadm-etcd.yaml

    sudo kubeadm init phase certs etcd-peer --config /root/kubeadm-etcd.yaml

    sudo kubeadm init phase certs etcd-healthcheck-client --config /root/kubeadm-etcd.yaml

Check the results:

    sudo ls -l /etc/kubernetes/pki/etcd/

The output should include at least the following files:

    ca.crt
    ca.key
    server.crt
    server.key
    peer.crt
    peer.key
    healthcheck-client.crt
    healthcheck-client.key

---

### 9.6 Generating apiserver-etcd-client Certificates on etcd-01

Execute the following command on etcd-01:

    sudo kubeadm init phase certs apiserver-etcd-client --config /root/kubeadm-etcd.yaml

Check the results:

    sudo ls -l /etc/kubernetes/pki/

You should see the following files:

    apiserver-etcd-client.crt
    apiserver-etcd-client.key

---

### 9.7 Generating etcd Static Pods on Each etcd Node

Execute the following commands on etcd-01, etcd-02, and etcd-03:

    sudo kubeadm init phase etcd local --config /root/kubeadm-etcd.yaml

Check the static Pod files:

    sudo ls -l /etc/kubernetes/manifests/

You should see the following file:

    etcd.yaml

Wait for the etcd container to start:

    sudo crictl ps | grep etcd

View the logs:

    ETCD_CONTAINER_ID=$(sudo crictl ps --name etcd -q)

    sudo crictl logs ${ETCD_CONTAINER_ID} | tail -n 50

---

### 9.8 Verifying the Health of the External etcd Cluster

Execute the following command on etcd-01:

    ETCD_CONTAINER_ID=$(sudo crictl ps --name etcd -q)

    sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
      --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      endpoint health --cluster

View the members:

    sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
      --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      member list

You should see 3 etcd members.

Note:

    The External etcd cluster will not appear in the output of `kubectl get pods -A` because it is not part of the Kubernetes cluster. You need to use `crictl` on the etcd nodes to check the static Pods.

---

### 9.9 Packaging Certificates Required for Master to Connect to etcd

Execute the following command on etcd-01:

    sudo tar -C /etc/kubernetes/pki -czf /root/master-etcd-client-pki.tar.gz \
      etcd/ca.crt \
      apiserver-etcd-client.crt \
      apiserver-etcd-client.key

Copy the certificate package to the 3 Master nodes:

    scp /root/master-etcd-client-pki.tar.gz root@k8s-master-01:/root/
    scp /root/master-etcd-client-pki.tar.gz root@k8s-master-02:/root/
    scp /root/master-etcd-client-pki.tar.gz root@k8s-master-03:/root/

On k8s-master-01, k8s-master-02, and k8s-master-03, execute the following commands:

    sudo mkdir -p /etc/kubernetes/pki
    sudo tar -C /etc/kubernetes/pki -xzf /root/master-etcd-client-pki.tar.gz

Check the results:

    sudo ls -```markdown
- name: vip_kubeconfig
  value: "/etc/kubernetes/super-admin.conf"
image: ghcr.io/kube-vip/kube-vip:v0.8.9
imagePullPolicy: IfNotPresent
name: kube-vip
resources: {}
securityContext:
  capabilities:
    add:
    - NET_ADMIN
    - NET_RAW
volumeMounts:
  - mountPath: /etc/kubernetes/super-admin.conf
    name: kubeconfig
  hostNetwork: true
volumes:
  - hostPath:
    path: /etc/kubernetes/super-admin.conf
    name: kubeconfig
status: {}
EOF

If the network card is not ens33, modify:

    sudo sed -i.bak 's/value: "ens33"/value: "actual network card name"/' /etc/kubernetes/manifests/kube-vip.yaml

Check:

    grep -n "vip_interface" /etc/kubernetes/manifests/kube-vip.yaml
    grep -n "vip_address" /etc/kubernetes/manifests/kube-vip.yaml
    grep -n "super-admin.conf" /etc/kubernetes/manifests/kube-vip.yaml

Note:

    The kube-vip image is from ghcr.io.
    If the current environment cannot access ghcr.io, it is recommended to synchronize the kube-vip image to an internal Harbor in advance and replace the image address in the YAML file.
    Before initializing master01, do not specify the type for hostPath to avoid creating an empty super-admin.conf prematurely.

---

## Section Eleven: Initializing the Kubernetes Cluster on Master-01

The following steps are performed on k8s-master-01.

In external etcd mode, it is recommended to use a configuration file when executing `kubeadm init`, as you need to specify the external etcd endpoints and certificate paths.

---

### 11.1 Creating a kubeadm Configuration File

Perform the following on k8s-master-01:

    cat <<EOF | sudo tee /root/kubeadm-config.yaml
    apiVersion: kubeadm.k8s.io/v1beta4
    kind: InitConfiguration
    localAPIEndpoint:
      advertiseAddress: "10.0.0.20"
      bindPort: 6443
    nodeRegistration:
      name: "k8s-master-01"
      criSocket: "unix:///run/containerd/containerd.sock"
    ---
    apiVersion: kubeadm.k8s.io/v1beta4
    kind: ClusterConfiguration
    kubernetesVersion: "v1.31.14"
    imageRepository: "registry.aliyuncs.com/google_containers"
    controlPlaneEndpoint: "k8s-api-server:6443"
    networking:
      podSubnet: "10.244.0.0/16"
      serviceSubnet: "10.96.0.0/12"
      dnsDomain: "cluster.local"
    apiServer:
      certSANs:
      - "k8s-api-server"
      - "10.0.0.30"
      - "10.0.0.20"
      - "10.0.0.21"
      - "10.0.0.22"
    etcd:
      external:
        endpoints:
        - "https://10.0.0.31:2379"
        - "https://10.0.0.32:2379"
        - "https://10.0.0.33:2379"
        caFile: "/etc/kubernetes/pki/etcd/ca.crt"
        certFile: "/etc/kubernetes/pki/apiserver-etcd-client.crt"
        keyFile: "/etc/kubernetes/pkiApiserver-etcd-client.key"
    ---
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    cgroupDriver: systemd
    EOF

Check:

    cat /root/kubeadm-config.yaml

---

### 11.2 Executing kubeadm init

Perform the following on k8s-master-01:

    sudo kubeadm init --config /root/kubeadm-config.yaml --upload-certs

After initialization is complete, save the `join` command from the output.

---

### 11.3 Switching kube-vip Back to admin.conf

After successful initialization with `kubeadm init`, switch kube-vip back from `super-admin.conf` to `admin.conf`:

    sudo cp /etc/kubernetes/manifests/kube-vip.yaml /etc/kubernetes/manifests/kube-vip.yaml.bak.$(date +%F-%H%M%S)

    sudocp /tmp/kube-proxy.yaml /tmp/kube-proxy.yaml.bak.$(date +%F-%H%M%S)

Modify the mode and scheduler:

    sed -i.bak -E \
      -e 's/^([[:space:]]*)mode:.*$/\1mode: "ipvs"/' \
      -e 's/^([[:space:]]*)scheduler:.*$/\1scheduler: "rr"/' \
      /tmp/kube-proxy.yaml

Check:

    grep -E "mode:|scheduler:" /tmp/kube-proxy.yaml

Apply the changes:

    kubectl replace -f /tmp/kube-proxy.yaml

---

### 12.3 Restart kube-proxy

Delete the kube-proxy Pod to allow the DaemonSet to rebuild automatically:

    kubectl -n kube-system delete pod -l k8s-app=kube-proxy

Check:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

---

### 12.4 Verify the Configuration

Check the configuration:

    kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config\.conf}' | grep -E "mode:|scheduler:"

The expected output is:

    mode: "ipvs"
    scheduler: "rr"

After creating a Service later on, you can use the following command to verify the IPVS rules:

    sudo ipvsadm -Ln

---

## Section Thirteen: Install the Calico Network Plugin

The following steps are performed on k8s-master-01.

The Pod network segment for this document is:

    10.244.0.0/16

The CIDR range specified in the Calico configuration must match the podSubnet defined in the kubeadm configuration.

---

### 13.1 Download Calico YAML Files

Create a directory:

    mkdir -p /root/k8s-yaml/calico
    cd /root/k8s-yaml/calico

Download the Calico Operator:

    curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

Download the custom-resources file:

    curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml

If downloading from GitHub is slow, you can download these files in advance on a machine with an internet connection and then copy them to this directory.

---

### 13.2 Modify the Pod CIDR Range

Edit the custom-resources.yaml file:

    sed -i.bak 's#cidr: 192.168.0.0/16#cidr: 10.244.0.0/16#' custom-resources.yaml

Check:

    grep -n "cidr:" custom-resources.yaml

---

### 13.3 Install Calico

Apply the Operator:

    kubectl create -f tigera-operator.yaml

Apply the custom resources:

    kubectl create -f custom-resources.yaml

Check the status:

    kubectl -n tigera-operator get pods -o wide
    kubectl -n calico-system get pods -o wide

Check the nodes:

    kubectl get nodes -o wide

Wait for k8s-master-01 to become Ready.

Note:

    If there is an issue with downloading the Calico images, you will need to synchronize the relevant images to the internal Harbor in advance, or set up a solution for accelerated image downloads.
    For clarity in this document, the image synchronization process is mentioned separately as a potential optimization step for production use.

---

## Section Fourteen: Add Master-02 and Master-03 to the Control Plane

The following steps are performed on k8s-master-02 and k8s-master-03 respectively.

Note:

    Do not apply the kube-vip.yaml file before executing the `kubeadm join --control-plane` command on master02 and master03.
    First, perform `kubeadm join --control-plane`.
    After successful joining, create the kube-vip static Pod later on.
    This ensures that the /etc/kubernetes/admin.conf file already exists.

---

### 14.1 Obtain the Control Plane Join Command

On k8s-master-01, check the join command generated by running `kubeadm init`.

If the token has expired, you can regenerate it:

    kubeadm token create --print-join-command

If the certificate-key is outdated, you can reupload the certificate:

    sudo kubeadm init phase upload-certs --upload-certs

The format of the control plane join command is as follows:

    sudo kubeadm join k8s-api-server:6443 \
      --token <token> \
      --discovery-token-ca-cert-hash sha25```markdown
image: ghcr.io/kube-vip/kube-vip:v0.8.9
imagePullPolicy: IfNotPresent
name: kube-vip
resources: {}
securityContext:
  capabilities:
    add:
    - NET_ADMIN
    - NET_RAW
volumeMounts:
  - mountPath: /etc/kubernetes/admin.conf
    name: kubeconfig
hostNetwork: true
volumes:
  - hostPath:
    path: /etc/kubernetes/admin.conf
    type: File
    name: kubeconfig
status: {}
EOF

If the network card is not ens33, modify:

sudo sed -i.bak 's/value: "ens33"/value: "actual network card name"/' /etc/kubernetes/manifests/kube-vip.yaml

Check:

sudo crictl ps | grep kube-vip
ip addr | grep 10.0.0.30

---

### 14.4 Adding k8s-master-03 to the Control Plane

On k8s-master-03, execute:

sudo kubeadm join k8s-api-server:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key> \
  --apiserver-advertise-address "10.0.0.22" \
  --cri-socket "unix:///run/containerd/containerd.sock"

Configure kubectl:

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

---

### 14.5 Creating a kube-vip Static Pod on k8s-master-03

On k8s-master-03, execute:

sudo mkdir -p /etc/kubernetes/manifests

cat <<EOF | sudo tee /etc/kubernetes/manifests/kube-vip.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-vip
  namespace: kube-system
spec:
  containers:
  - args:
    - manager
    env:
    - name: vip_arp
      value: "true"
    - name: port
      value: "6443"
    - name: vip_interface
      value: "ens33"
    - name: vip_cidr
      value: "24"
    - name: cp_enable
      value: "true"
    - name: cp_namespace
      value: "kube-system"
    - name: vip_ddns
      value: "false"
    - name: vip_leaderelection
      value: "true"
    - name: vip_address
      value: "10.0.0.30"
    - name: vip_kubeconfig
      value: "/etc/kubernetes/admin.conf"
    image: ghcr.io/kube-vip/kube-vip:v0.8.9
    imagePullPolicy: IfNotPresent
    name: kube-vip
    resources: {}
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_RAW
    volumeMounts:
    - mountPath: /etc/kubernetes/admin.conf
      name: kubeconfig
status: {}
EOF

If the network card is not ens33, modify:

sudo sed -i.bak 's/value: "ens33"/value: "actual network card name"/' /etc/kubernetes/manifests/kube-vip.yaml

Check:

sudo crictl ps | grep kube-vip
ip addr | grep 10.0.0.30

---

### 14.6 Checking Master Status

On any Master node, execute:

kubectl get nodes -o wide

Check the control plane Pods:

kubectl -n kube-system get pods -o wide | grep -E "kube-apiserver|kube-controller|kube-scheduler|kube-vip"

In External etcd mode, there should be no etcd Pods in the kube-system namespace on Master nodes.

Expected output:

3 kube-apiservers
3 kube-controller-managers
3 kube-schedulers
3 kube-vips

Check the VIP address:

ip addr | grep 10.0.0.30

---

## Section Fifteen: Adding Worker Nodes to the Cluster

The following steps are to be performed on k8s-worker-01, k8s-worker-02, and k8s-worker-03.

---

### 15.1 Obtaining the Worker Join Command

On k8s-master-01, execute:

kubeadm token create --print-join-command

The Worker join command is in the following format:

sudo kubeadm join k8s-api-server:644tigera-operator

These conditions should not persist for an extended period:

Pending
CrashLoopBackOff
ImagePullBackOff

In external etcd mode:

Running `kubectl get pods -A` will not display the etcd Pods. The etcd Pods need to be checked using `crictl` on the etcd nodes.

---

### 16.2 Verifying External etcd

Execute the following commands on etcd-01:

```bash
sudo crictl ps | grep etcd
```

Check the health status of etcd:

```bash
ETCD_CONTAINER_ID=$(sudo crictl ps --name etcd -q)
sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
  --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  endpoint health --cluster
```

Check the members:

```bash
sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
  --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  member list
```

Three etcd members should be displayed.

---

### 16.3 Verifying kube-vip

Execute the following commands on each of the three Master nodes:

```bash
ip addr | grep 10.0.0.30
```

Under normal circumstances, only one Master node holds the VIP at any given time.

Access the APIServer:

```bash
curl -k https://k8s-api-server:6443/livez
```

The response "ok" is expected.

Check the version:

```bash
curl -k https://k8s-api-server:6443/version
```

---

### 16.4 Verifying kube-proxy IPVS

Check the kube-proxy configuration:

```bash
kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config.conf}' | grep -E "mode:|scheduler:"
```

The expected result is:

```
mode: "ipvs"
scheduler: "rr"
```

View the IPVS rules:

```bash
sudo ipvsadm -Ln
```

If no service has been created yet, there may be few rules. More Virtual Server rules should appear after creating services.

---

### 16.5 Verifying CoreDNS

Create a test Pod:

```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never -- sleep 3600
```

Check the Pod:

```bash
kubectl get pod dns-test -o wide
```

Test resolution:

```bash
kubectl exec dns-test -- nslookup kubernetes.default.svc.cluster.local
```

The expected response is `10.96.0.1`.

Clean up:

```bash
kubectl delete pod dns-test
```

---

### 16.6 Verifying Pods and Services

Create a test Deployment:

```bash
kubectl create deployment nginx-test --image=nginx:1.25
```

Scale the Deployment:

```bash
kubectl scale deployment nginx-test --replicas=3
```

Check Pod distribution:

```bash
kubectl get pods -o wide
```

Create a Service:

```bash
kubectl expose deployment nginx-test --port=80 --target-port=80 --type=ClusterIP
```

Check the Service:

```bash
kubectl get svc nginx-test
```

Create a temporary test Pod:

```bash
kubectl run curl-test --image=busybox:1.36 --restart=Never -it --rm -- sh
```

Execute a command inside the container:

```bash
wget -qO- nginx-test.default.svc.cluster.local
```

If the nginx page is displayed, it indicates that:

- The Pod's network is functioning correctly.
- The Service is forwarding requests properly.
- CoreDNS is working correctly.
- kube-proxy is functioning normally.

Clean up:

```bash
kubectl delete svc nginx-test
kubectl delete deployment nginx-test
```

---

## Section Seventeen: Verifying High Availability of kube-vip

Note:

The following steps6. containerd is not running properly.
7. There are configuration errors in kubelet's static Pod management.

---

### 19.2 Failure of Kubeadm Init

Check on the Master node:

    journalctl -u kubelet -f
    systemctl status containerd --no-pager
    sudo crictl info
    sudo crictl images

Check the external etcd certificates:

    sudo ls -l /etc/kubernetes/pki/etcd/ca.crt
    sudo ls -l /etc/kubernetes/pki/apiserver-etcd-client.crt
    sudo ls -l /etc/kubernetes/pki apiserver-etcd-client.key

Check etcd connectivity:

    telnet 10.0.0.31 2379
    telnet 10.0.0.32 2379
    telnet 10.0.0.33 2379

Common causes:

    1. The Master node does not have the external etcd client certificate.
    2. The etcd endpoints in kubeadm-config.yaml are incorrectly specified.
    3. The etcd port 2379 is unreachable.
    4. The etcd cluster itself is unhealthy.
    5. The imageRepository is not configured correctly.
    6. The kube-vip network interface name is incorrect.

---

### 19.3 kube-vip Is Not Effective

Check the kube-vip manifest:

    cat /etc/kubernetes/manifests/kube-vip.yaml

Check the network interfaces:

    ip addr

Check the kube-vip containers:

    sudo crictl ps | grep kube-vip
    sudo crictl ps -a | grep kube-vip

View the logs:

    sudo crictl logs <kube-vip-container-id>

Check the VIP address:

    ip addr | grep 10.0.0.30

Common causes:

    1. The vip_interface is incorrectly specified.
    2. The IP address 10.0.0.30 is already in use.
    3. Failed to pull the kube-vip image.
    4. The current network does not support ARP VIP migration.
    5. super-admin.conf was not used before initializing master01.
    6. After kubeadm init, kube-vip was not switched back to admin.conf.
    7. After master02/master03 joined the cluster, the kube-vip static Pod was not deployed.

---

### 19.4 Master Node Failed to Join the Cluster

Check if the VIP address is reachable:

    ping -c 3 k8s-api-server
    curl -k https://k8s-api-server:6443/livez

Check the token:

    kubeadm token list

Regenerate the token:

    kubeadm token create --print-join-command

Reupload the certificates:

    sudo kubeadm init phase upload-certs --upload-certs

Common causes:

    1. The token has expired.
    2. The certificate-key has expired.
    3. The VIP address is unreachable.
    4. There was an parsing error with k8s-api-server.
    5. The time is not synchronized.
    6. kube-vip.yaml was incorrectly placed before master02/master03 joined the cluster.

---

### 19.5 Services Are Not Accessible

Check the Services:

    kubectl get svc
    kubectl describe svc <svc-name>

Check the Endpoints:

    kubectl get endpoints <svc-name>

Check the Pod labels:

    kubectl get pods --show-labels

Check IPVS:

    sudo ipvsadm -Ln

Common causes:

    1. The Service selector is incorrectly specified.
    2. The Pod labels do not match.
    3. The endpoints are empty.
    4. The targetPort is incorrect.
    5. The service inside the container is not listening on the corresponding port.
    6. There is an issue with kube-proxy.

---

### 19.6 The containerd Data Directory Is Not Effective

Check the configuration:

    grep -n '^root = ' /etc/containerd/config.toml

Expected result:

    root = "/data/containerd"

Check the directory:

    sudo ls -ld /data/containerd
    sudo du -sh /data/containerd

Check the containerd status:

    systemctl status containerd --no-pager

Common causes:

    1. Containerd was not restarted after modifying the configuration.
    2. The sed command did not match the root configuration setting.
    3. The /data directory was not created.
    4. The /data directory is not mounted independently and still occupies space in the root partition.
    5. The directory was modified after the cluster11. After joining master02 and master03, deploy the kube-vip static Pod.
12. Use containerd as the container runtime.
13. Adjust the containerd data directory to /data/containerd.
14. Adjust the etcd data directory to /data/etcd.
15. Install containerd using the Alibaba Cloud Docker CE source.
16. Install kubeadm, kubelet, and kubectl using the Alibaba Cloud Kubernetes source.
17. Initialize the cluster using the Alibaba Cloud google_containers image repository.
18. Set kube-proxy to use the IPVS mode by default.
19. Use Calico for Pod networking.
20. When making configuration changes, always use sed -i.bak to facilitate rollbacks.

This note can serve as a basic Runbook for subsequently redeploying a highly available Kubernetes External etcd cluster.