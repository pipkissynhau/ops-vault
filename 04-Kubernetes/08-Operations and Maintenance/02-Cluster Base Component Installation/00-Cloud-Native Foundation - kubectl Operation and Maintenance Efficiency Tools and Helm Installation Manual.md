# 00-Cloud-Native Foundation: kubectl Operation and Maintenance Efficiency Tools and Helm Installation Manual

Recommended Path:

    04-Kubernetes/08-Operation and Maintenance/02-Cluster Basic Components Installation/00-Cloud-Native Foundation: kubectl Operation and Maintenance Efficiency Tools and Helm Installation Manual.md

Tags:

    #Kubernetes
    #kubectl
    #Helm
    #K9s
    #Stern
    #kubectx
    #kubens
    #Operation and Maintenance Efficiency
    #Cloud-Native Tools

---

## I. Document Description

This document records the installation methods for commonly used operation and maintenance efficiency tools after a Kubernetes cluster has been deployed.

It mainly includes:

    1. kubectl auto-completion
    2. Setting kubectl alias as k
    3. Helm v3 installation
    4. Configuration of common Helm repositories
    5. K9s terminal cluster management tool
    6. Stern multi-Pod log aggregation tool
    7. kubectx / kubens multi-cluster and namespace switching tools

Target Nodes:

    Master node
    Operation and maintenance management machine
    Personal management terminal

Note:

    It is not necessary to install these tools on all Worker nodes.
    It is recommended to install them on nodes where kubectl is frequently used.

---

## II. Prerequisites

Confirm that the current node can execute kubectl normally:

    kubectl get nodes

If kubectl cannot access the cluster, you need to configure kubeconfig first.

On the Master node, you can usually perform the following steps:

    mkdir -p $HOME/.kube

    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

    sudo chown $(id -u):$(id -g) $HOME/.kube/config

Verification:

    kubectl get nodes

---

## III. kubectl Alias and Auto-Completion

### 3.1 Install bash-completion

Install the basic auto-completion package:

    sudo apt update

    sudo apt install -y bash-completion

Verification:

    ls -l /usr/share/bash-completion/bash_completion

---

### 3.2 Configure kubectl alias as k and enable auto-completion

To avoid rewriting ~/.bashrc repeatedly, it is recommended to use grep to check first and then append the necessary lines.

Execution:

    grep -qxF 'alias k=kubectl' ~/.bashrc || echo 'alias k=kubectl' >> ~/.bashrc

    grep -qXF 'source <(kubectl completion bash)' ~/.bashrc || echo 'source <(kubectl completion bash)' >> ~/.bashrc

    grep -qxF 'complete -o default -F __start_kubectl k' ~/.bashrc || echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

Reload the configuration:

    source ~/.bashrc

Note:

    alias k=kubectl
        Shortens kubectl to k.

    source <(kubectl completion bash)
        Enables Bash auto-completion for kubectl.

    complete -o default -F __start_kubectl k
        Allows the alias k to use kubectl's auto-completion logic as well.

---

### 3.3 Verify Auto-Completion

Verify kubectl:

    kubectl get nodes

Verify the k alias:

    k get nodes

Verify auto-completion:

    k g<Tab>

The expected completion should be:

    k get

Continue to verify resource completion:

    k get po<Tab>

The expected completion should be:

    k get pods

---

## IV. Helm v3 Installation

Helm is a commonly used package management tool in Kubernetes, often used to install components such as ingress-nginx, metrics-server, cert-manager, Prometheus, Grafana, Loki, and Argo CD.

Target Nodes:

    Master node
    Operation and maintenance management machine

---

### 4.1 Method 1: Official Script Installation

Download the installation script:

    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

Grant execution permissions:

    chmod 700 get_helm.sh

Execute the installation:

    ./get_helm.sh

Verification:

    helm version

Note:

    This method will automatically detect the system architecture and install Helm.
    If access to GitHub is slow in your current environment, you can use the manual installation method instead.

---

### 4.2 Method 2: Manual Installation using Domestic Mirrors

Download the Helm binary package from Huawei Cloud mirrors:

    cd /tmp

    sudo wget -c https://mirrors.huaweicloud.com/helm/v3.19.0/helm-v3.19.0-linux-amd64.tar.gz

Extract the package:

    tar -zxvf helm-v3.19.0-linux-amd64.tar.gz

The text after the 🔤 symbols contains technical information about various tools and their usage in Kubernetes operations. Since this part is not structured as markdown and does not contain any directives for translation, I will provide a general translation of the text without maintaining the original formatting or structure.

---

## VIII. Installation of kubectx and kubens

kubectx is used to quickly switch between Kubernetes contexts.

kubens is used to quickly switch between namespaces.

Use cases:

    1. Multi-cluster environments
    2. Operations across multiple namespaces
    3. Frequent switching between dev, test, and prod environments
    4. To avoid manually adding `-n namespace` at the end of each command

---

### 8.1 Install Git

Install dependencies:

    sudo apt update

    sudo apt install -y git

---

### 8.2 Install kubectx/kubens

Clone the repository:

    sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx

Create symbolic links:

    sudo ln -sf /opt/kubectx/kubectx /usr/local/bin/kubectx

    sudo ln -sf /opt/kubectx/kubens /usr/local/bin/kubens

Verify installation:

    kubectx

    kubens

---

### 8.3 Example Usage

View all cluster contexts:

    kubectx

Switch to a specified cluster context:

    kubectx <context-name>

View all namespaces:

    kubens

Switch to the default namespace:

    kubens kube-system

Verify the current context:

    kubectl config current-context

View the current namespace:

    kubectl config view --minify --output 'jsonpath={..namespace}'

If no output is displayed, it usually means the default namespace is "default".

---

## IX. Common kubectl Abbreviations

Common resource abbreviations:

| Full Resource | Abbreviation |
|---------------|-------------------|
| pods           | po               |
| services       | svc               |
| deployments    | deploy             |
| daemonsets      | ds                 |
| statefulsets   | sts                |
| namespaces     | ns                 |
| configmaps      | cm                 |
| secrets        | secret             |
| nodes          | no                 |
| ingress         | ing                 |
| persistentvolumeclaims | pvc                 |
| persistentvolumes  | pv                 |
| serviceaccounts | sa                 |

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

## X. Post-Installation Checklist

Run the following commands to ensure the tools are available:

    kubectl get nodes

    helm version

    helm repo list

    k9s version

    stern --version

    kubectx

    kubens

Verify autocompletion:

    k g<Tab>

It should complete with "k get".

---

## XI. Common Issues

### 11.1 k Commands Not Available

Check aliases:

    grep 'alias k=kubectl' ~/.bashrc

Reload the settings:

    source ~/.bashrc

---

### 11.2 k is Available but No Autocompletion

Check configurations:

    grep 'kubectl completion bash' ~/.bashrc

    grep '__start_kubectl k' ~/.bashrc

Reload the settings:

    source ~/.bashrc

If it still doesn't work, open a new Shell terminal.

---

### 11.3 helm repo update Is Slow

First, check if the network can access the repository:

    curl -I https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

    curl -I http://mirror.azure.cn/kubernetes/charts

If it's still slow, consider:

    1. Using an internal Helm repository
    2. Manually downloading chart packages
    3. Offline installation using `helm install ./chart.tgz`

---

### 11.4 k9s Cannot Connect to Clusters

First, verify kubectl:

    kubectl get nodes

Check the kubeconfig file:

    ls -l ~/.kube/config

    echo $KUBECONFIG

If kubectl fails, fix the kubeconfig file first.

---

### 11.5 stern Cannot Find Logs

First, confirm if the Pod exists:

    kubectl get pods -A | grep <keyword>

Check if the namespace is correct:

    kubectl get ns

Use the specified namespace:

    stern <keyword> -n <namespace>

---

## XII. Summary

This document covers the installation of several useful tools for Kubernetes operations, including:

