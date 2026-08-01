# 00-Cloud Native Foundation: kubectl Operations Efficiency Tools and Helm Installation Manual

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/00-Cloud Native Foundation: kubectl Operations Efficiency Tools and Helm Installation Manual.md

Tags:

    #Kubernetes
    #kubectl
    #Helm
    #K9s
    #Stern
    #kubectx
    #kubens
    #TransportEfficiency
    #CloudtopTool

---

## I. Document Description

This document records the installation methods of commonly used operations efficiency tools after Kubernetes cluster deployment is complete.

Main contents include:

    1. kubectl auto-completion
    2. kubectl alias k
    3. Helm v3 installation
    4. Common Helm repository configuration
    5. K9s terminal cluster management tool
    6. Stern multi-Pod log aggregation tool
    7. kubectx / kubens multi-cluster and namespace switching tools

Execution nodes:

    Master node
    Operations management machine
    Personal management terminal

Notes:

    Not all Worker nodes need to install.
    It is recommended to install on nodes where kubectl is frequently executed.

---

## II. Prerequisites

Confirm that the current node can execute kubectl normally:

    kubectl get nodes

If kubectl cannot access the cluster, configure kubeconfig first.

On the Master node, typically execute:

    mkdir -p $HOME/.kube

    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

    sudo chown $(id -u):$(id -g) $HOME/.kube/config

Verification:

    kubectl get nodes

---

## III. kubectl Alias and Auto-Completion

### 3.1 Install bash-completion

Install the auto-completion base package:

    sudo apt update

    sudo apt install -y bash-completion

Check:

    ls -l /usr/share/bash-completion/bash_completion

---

### 3.2 Configure kubectl alias k and enable auto-completion

To avoid repeatedly writing to ~/.bashrc, it is recommended to append only if the line exists.

Execute:

    grep -qxF 'alias k=kubectl' ~/.bashrc || echo 'alias k=kubectl' >> ~/.bashrc

    grep -qxF 'source <(kubectl completion bash)' ~/.bashrc || echo 'source <(kubectl completion bash)' >> ~/.bashrc

    grep -qxF 'complete -o default -F __start_kubectl k' ~/.bashrc || echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

Reload configuration:

    source ~/.bashrc

Notes:

    alias k=kubectl
        Simplifies kubectl to k.

    source <(kubectl completion bash)
        Enables Bash auto-completion for kubectl.

    complete -o default -F __start_kubectl k
        Allows the k alias to use kubectl's auto-completion logic.

---

### 3.3 Verify Auto-Completion

Verify kubectl:

    kubectl get nodes

Verify k alias:

    k get nodes

Verify auto-completion:

    k g<Tab>

Expected completion:

    k get

Continue verifying resource completion:

    k get po<Tab>

Expected completion:

    k get pods

---

## IV. Helm v3 Installation

Helm is a commonly used package management tool for Kubernetes, often used to install components such as ingress-nginx, metrics-server, cert-manager, Prometheus, Grafana, Loki, Argo CD, etc.

Execution nodes:

    Master node
    Operations management machine

---

### 4.1 Method One: Official Script Installation

Download the installation script:

    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

Add executable permissions:

    chmod 700 get_helm.sh

Execute installation:

    ./get_helm.sh

Verification:

    helm version

Notes:

    This method automatically identifies the system architecture and installs Helm.
    If the current environment has slow access to GitHub, you can use manual installation method.

---

### 4.2 Method Two: Domestic Mirror Manual Installation

Download Helm binary package from Huawei Cloud mirror:

    cd /tmp

    sudo wget -c https://mirrors.huaweicloud.com/helm/v3.19.0/helm-v3.19.0-linux-amd64.tar.gz

Extract:

    tar -zxvf helm-v3.19.0-linux-amd64.tar.gz

Move to system PATH:

    sudo mv linux-amd64/helm /usr/local/bin/helm

Add executable permissions:

    sudo chmod +x /usr/local/bin/helm

Clean up temporary files:

    rm -rf linux-amd64 helm-v3.19.0-linux-amd64.tar.gz

Verification:

    helm version

---

## V. Common Helm Repository Configuration

Add common Helm repositories:

    helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

    helm repo add azure http://mirror.azure.cn/kubernetes/charts

Update repository index:

    helm repo update

View repositories:

    helm repo list

Search examples:

    helm search repo nginx

    helm search repo ingress

Notes:

    Some domestic Helm mirror repositories may have version lag or incomplete synchronization issues.
    In production environments, it is recommended to prioritize using company internal Helm repositories, or download and audit Chart packages in advance before installation.

---

## VI. K9s Installation

K9s is an interactive terminal Kubernetes cluster management tool.

Common Uses:

    1. View Pod / Deployment / Service
    2. View Pod logs
    3. Enter container
    4. View resource status
    5. Quickly switch namespace
    6. Assist in troubleshooting

---

### 6.1 Install K9s

Download:

    cd /tmp

    wget https://github.com/derailed/k9s/releases/download/v0.32.4/k9s_Linux_amd64.tar.gz

Extract:

    tar -zxvf k9s_Linux_amd64.tar.gz

Move to system PATH:

    sudo mv k9s /usr/local/bin/k9s

Add execute permission:

    sudo chmod +x /usr/local/bin/k9s

Verify:

    k9s version

Start:

    k9s

Notes:

    K9s depends on the current user's kubeconfig.
    If kubectl get nodes is not working properly, K9s will also fail to connect to the cluster.

---

## SevenI don't know.Stern Installation

Stern is a multi-Pod log aggregation tool.

Use Cases:

    1. A Deployment has multiple Pod replicas
    2. Need to track logs of multiple Pods simultaneously
    3. Want to automatically continue tracking new Pod logs after Pod recreation
    4. Troubleshoot rolling updates, restarts, and abnormal logs

---

### 7.1 Install Stern

Download:

    cd /tmp

    wget https://github.com/stern/stern/releases/download/v1.30.0/stern_1.30.0_linux_amd64.tar.gz

Extract:

    tar -zxvf stern_1.30.0_linux_amd64.tar.gz

Move to system PATH:

    sudo mv stern /usr/local/bin/stern

Add execute permission:

    sudo chmod +x /usr/local/bin/stern

Verify:

    stern --version

---

### 7.2 Stern Usage Examples

Track logs of Pods containing "nginx" in the default namespace:

    stern nginx -n default

Track coredns-related logs in the kube-system namespace:

    stern coredns -n kube-system

Track logs from a specific time onward:

    stern nginx -n default --since 10m

View logs of a specific container:

    stern nginx -n default -c nginx

Notes:

    stern can follow Pod names, Deployment names, or regular expressions.
    It is more efficient than manually running kubectl logs for troubleshooting multi-replica services.

---

## EightI don't know.kubectx and kubens Installation

kubectx is used for quickly switching Kubernetes contexts.

kubens is used for quickly switching namespaces.

Use Cases:

    1. Multi-cluster environments
    2. Multi-namespace operations
    3. Frequently switching between dev/test/prod environments
    4. Avoid manually specifying -n namespace in every command

---

### 8.1 Install git

Install dependencies:

    sudo apt update

    sudo apt install -y git

---

### 8.2 Install kubectx / kubens

Clone repository:

    sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx

Create symbolic links:

    sudo ln -sf /opt/kubectx/kubectx /usr/local/bin/kubectx

    sudo ln -sf /opt/kubectx/kubens /usr/local/bin/kubens

Verify:

    kubectx

    kubens

---

### 8.3 Usage Examples

View all cluster contexts:

    kubectx

Switch to a specified cluster context:

    kubectx <context-name>

View all namespaces:

    kubens

Switch to default namespace:

    kubens kube-system

Verify current context:

    kubectl config current-context

View current namespace:

    kubectl config view --minify --output 'jsonpath={..namespace}'

If there is no output, it usually indicates the current default namespace is default.

---

## NineI don't know.Common kubectl Abbreviations

Common resource abbreviations:

| Full Resource | Abbreviation |
|---|---|
| pods | po |
| services | svc |
| deployments | deploy |
| daemonsets | ds |
| statefulsets | sts |
| namespaces | ns |
| configmaps | cm |
| secrets | secret |
| nodes | no |
| ingress | ing |
| persistentvolumeclaims | pvc |
| persistentvolumes | pv |
| serviceaccounts | sa |

Common commands:

    k get no

    k get ns

    k get po -A

    k get svc -A

    k get deploy -A

    k -n kube-system get po

    k describe node k8s-worker-01

    k -n kube-system logs <pod-name>

    k -n kube-system describe pod <pod-name>

---

## TenI don't know.Post-Installation Checklist

Run the following commands to confirm tool availability:

    kubectl get nodes

    k get nodes

    helm version

    helm repo list

    k9s version

    stern --version

    kubectx

    kubens

Auto-completion verification:

    k g<Tab>

Expected to autocomplete to:

    k get

---

## ElevenI don't know.Common Issues

### 11.1 k Command Not Available

Check alias:

    grep 'alias k=kubectl' ~/.bashrc

Reload:

    source ~/.bashrc

---

### 11.2 k Is Available, But Auto-Completion Fails

Check configuration: /think

grep 'kubectl completion bash' ~/.bashrc

grep '__start_kubectl k' ~/.bashrc

Reload:

    source ~/.bashrc

If it still doesn't work, open a new Shell terminal.

---

### 11.3 helm repo update is slow

First confirm if the network can access the corresponding repositories:

    curl -I https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

    curl -I http://mirror.azure.cn/kubernetes/charts

If it's still slow, it's recommended to:

    1. Use the company's internal Helm repository
    2. Manually download the Chart package
    3. Use helm install ./chart.tgz for offline installation

---

### 11.4 k9s cannot connect to the cluster

First verify kubectl:

    kubectl get nodes

Check kubeconfig:

    ls -l ~/.kube/config

    echo $KUBECONFIG

If kubectl is not working, first fix the kubeconfig.

---

### 11.5 stern cannot find logs

First confirm if the Pod exists:

    kubectl get pods -A | grep <keyword>

Confirm if the namespace is correct:

    kubectl get ns

Use a specified namespace:

    stern <keyword> -n <namespace>

---

## TwelveI don't know.Summary

This article completes the installation of commonly used efficiency tools for Kubernetes operations:

    1. kubectl auto-completion
    2. k alias
    3. Helm v3
    4. Common Helm repositories
    5. K9s
    6. Stern
    7. kubectx
    8. kubens

These tools are not part of the Kubernetes cluster core components, but they are very suitable for daily operations, troubleshooting, and multi-cluster management.

It is recommended to install them on the following nodes:

    1. k8s-master-01
    2. Operations management machine
    3. Personal management terminal