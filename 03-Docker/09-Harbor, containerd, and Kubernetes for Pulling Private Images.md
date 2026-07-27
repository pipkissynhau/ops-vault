# 09-Harbor, containerd, and Kubernetes for Pulling Private Images

#Docker #containerd #Harbor #private repository #Kubernetes #imagePullSecrets #nerdctl #ctr #crictl #HTTP repository #ops #troubleshooting

---

## Recommended Path

03-Container Technology/09-Harbor, containerd, and Kubernetes for Pulling Private Images.md

---

## I. Document Overview

This document outlines common operational capabilities related to pulling private images using Harbor, Docker, containerd, and Kubernetes. Key topics include:

- Logging in to Harbor
- Tagging images
- Pushing images
- Pulling images
- Making Docker trust HTTP Harbor
- Basic configuration of containerd
- Containerd data directories
- Making containerd trust HTTP Harbor
- Configuration using `hosts.toml`
- Common usage of `nerdctl`, `ctr`, and `crictl`
- Containerd namespaces
- Kubernetes' default use of the `k8s.io` namespace
- Why `nerdctl login` does not guarantee that a Pod can pull private images
- Official methods for pulling private images in Kubernetes
- Troubleshooting common HTTP Harbor errors
- Troubleshooting authentication failures with private repositories
- Troubleshooting image push rejections

The goal is to ensure that users:

- Can log in to Harbor
- Can tag, push, and pull images
- Can distinguish between Docker and containerd repository configurations
- Can make Docker and containerd trust HTTP Harbor
- Can perform node-side verification using nerdctl/ctr/crictl
- Understand containerd namespaces
- Can troubleshoot Kubernetes issues when pulling private images

---

## II. Logging in to Harbor and Pushing/Pulling Images

---

## Scenario 75: Logging in to Harbor

### Command

```bash
docker login 10.0.0.10:8090
```

### Interactive Input

```text
Username
Password
```

### Explanation

`docker login` is used to log in to a Docker image repository.

The Harbor address here is:

```text
10.0.0.10:8090
```

After successful login, the local Docker client will save authentication information.

Common uses:

- Pulling private images
- Pushing images to Harbor
- Verifying account and password correctness
- Checking if Harbor is accessible

---

## Scenario 76: Using Account and Password Directly in the Command Line

### Command

```bash
docker login 10.0.0.10:8090 -u admin -p Harbor12345
```

### Explanation

- It allows direct login.
- However, using plain text passwords in the command line is not secure.
- In production environments, interactive input is recommended.

### Security Note

It is not advised to use the following format for long-term use:

```bash
docker login 10.0.0.10:8090 -u admin -p Harbor12345
```

Reasons:

- The password may be included in the shell history.
- It may be visible in the process list.
- It may be collected by logging systems.
- This does not meet production security requirements.

Interactive input is more secure:

```bash
docker login 10.0.0.10:8090
```

---

## Scenario 77: Tagging Images

### Command

```bash
docker tag nginx:latest 10.0.0.10:8090/project/nginx:latest
```

### Explanation

Before pushing an image to Harbor, it usually needs to be tagged in the format of the Harbor repository address.

The general format is:

```text
Harbor address/Project name/Image name:Tag
```

Example:

```text
10.0.0.10:8090/project/nginx:latest
```

Meaning:

```text
Harbor address: 10.0.0.10:8090
Project name: project
Image name: nginx
Tag: latest
```

---

## Scenario 78: Pushing Images

### Command

```bash
docker push 10.0.0.10:8090/project/nginx:latest
```

### Explanation

Before pushing, confirm the following:

- The Harbor address is correct.
- The Harbor project exists.
- The image tag format is correct.
- The current user is logged in.
- The current user has permission to push images.

---

## Scenario 79: Pulling Images

### Command

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

### Explanation

Before pulling a private image, confirm the following:

- The local- It is best to plan everything in advance before deploying in a Kubernetes cluster.

### Restart containerd after modifications

```bash
systemctl restart containerd
```

### Verify the status

```bash
systemctl status containerd
```

---

## Scenario 87: Why optimize the containerd directory before deploying in K8s

Reasons:

- It is part of the node baseline configuration.
- Avoiding the need to migrate image and container data after the cluster has already started running can prevent disruptions.
- It ensures smoother operation of kubelet and Pods.
- It facilitates batch processing and template-based deployment.

### Operational understanding

If a Kubernetes cluster already has many Pods running, attempting to migrate the containerd data directory later on carries higher risks.

Possible impacts include:

- Affecting node image data.
- Disrupting container snapshot operations.
- Impeding Pod startup processes.
- Interrupting communication between kubelet and containerd.
- Reducing node stability.

Therefore, it is recommended to plan the configuration during the node initialization phase:

```toml
root = "/data/containerd"
```

---

## V. Trusting HTTP Harbor (containerd)

If a Kubernetes node or tools like `crictl`, `ctr`, or `nerdctl` use containerd for image pulling, relying solely on Docker login and an insecure registry is often insufficient.

---

## Scenario 88: Configuring HTTP Harbor for containerd

### Directory

```bash
/etc/containerd/certs.d/10.0.0.10:8090/
```

### File

```bash
/etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

### Example content

```toml
server = "http://10.0.0.10:8090"

[host."http://10.0.0.10:8090"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
```

### Explanation

This configuration instructs containerd to use HTTP instead of HTTPS when accessing the specified address.

The directory name must match the repository URL, and the file name should be `hosts.toml`.

---

## Scenario 89: Checking the main containerd configuration

### Configuration file

```bash
/etc/containerd/config.toml
```

### Configuration items to check

```toml
[plugins."io.containerd.cri.v1.images".registry]
  config_path = "/etc/containerd/certs.d"
```

### Explanation

- The `config_path` must point to the `certs.d` directory; otherwise, the `hosts.toml` configuration will not take effect.

### Operational understanding

Simply creating the `hosts.toml` file in `/etc/containerd/certs.d` is not enough. You also need to ensure that `config_path` is set correctly, as containerd may fail to read the registry settings if it is missing.

---

## Scenario 90: Restarting containerd

### Command

```bash
systemctl restart containerd
```

### Explanation

After modifying containerd configuration, you need to restart it for the changes to take effect. It is recommended to check the status after restarting:

```bash
systemctl status containerd
```

If you are working with a Kubernetes node, you should also monitor the status of kubelet and Pods:

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

---

## VI. Nerdctl, Ctr, Crictl, and Namespaces

---

## Scenario 91: What is nerdctl?

`nerdctl` is a Docker-like CLI tool in the containerd ecosystem, with a syntax similar to Docker's.

### Common commands

- List images: `nerdctl images`
- List containers: `nerdctl ps -a`
- Login to a repository: `nerdctl login 10.0.0.10:8090`

### Explanation

`nerdctl` is user-friendly and has a command syntax similar to Docker's, making it useful for node-side debugging and verifying containerd's functionality.

It is often used for:

- Debugging on the node level.
- Checking if containerd can pull images successfully.
- Viewing images and containers managed by containerd.
- Logging into private repositories for temporary validation purposes.

---

## Scenario 92: Can nerdctl be used to login to a Harbor?

### Command

```bash
nerdctl login 10.0.0.10:8090
```

### Explanation

`nerdctl` can be used for node-side debugging and validation tasks. It indicates that the containerd client on the current node is capable of accessing the specified repository using those credentials.

However, it's important to note that:

```text
nerdctl loginNode-side debugging:

```bash
nerdctl login
```

```bash
ctr pull
```

```bash
crictl pull
```

For official Kubernetes usage:

```text
imagePullSecrets
```

### Operational understanding

A successful `nerdctl login` only indicates that the node-side tool has successfully logged into Harbor.

However, Kubernetes image pulling also involves the following components:

- kubelet
- containerd CRI plugin
- Pod specification
- imagePullSecrets
- Secrets in the namespace
- Harbor project permissions
- HTTP/HTTPS configuration

Therefore, it cannot be simply assumed that:

```text
A successful nerdctl login means that a Pod will definitely be able to pull images.
```

---

## Scenario 100: More recommended methods for officially pulling private images in Kubernetes

### Using imagePullSecrets

Explanation:

- This is the official and general method for authenticating to private repositories in Kubernetes.
- It is more standardized than relying solely on node-side login.

### Example of creating an imagePullSecret

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=10.0.0.10:8090 \
  --docker-username=admin \
  --docker-password='Harbor12345' \
  --docker-email=admin@example.com \
  -n default
```

### Example of referencing it in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-nginx
spec:
  imagePullSecrets:
    - name: harbor-secret
  containers:
    - name: nginx
      image: 10.0.0.10:8090/project/nginx:latest
```

### Example of referencing it in a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: private-nginx
  template:
    metadata:
      labels:
        app: private-nginx
    spec:
      imagePullSecrets:
        - name: harbor-secret
      containers:
        - name: nginx
          image: 10.0.0.10:8090/project/nginx:latest
```

### Note

The `imagePullSecret` must be in the same namespace as the Pod.

For example, if the Pod is in the `default` namespace, then the Secret should also be in the `default` namespace:

```bash
kubectl get secret -n default
```

---

## VIII. Common issues and troubleshooting

---

## Scenario 105: Docker login is successful, but Kubernetes image pulling fails

### Common causes

- Docker is logged in, but containerd is not configured.
- Harbor uses HTTP, and containerd does not trust it.
- The `hosts.toml` file is missing on the node.
- The `registry config_path` in `config.toml` is incorrect.
- The Pod lacks `imagePullSecrets`.

### Troubleshooting steps

```text
Check if kubelet/containerd reports any HTTP/HTTPS errors.

→ Check if `/etc/containerd/certs.d` exists.

→ Verify the contents of `hosts.toml`.

→ Check the `config_path` in `config.toml`.

→ Test using ctr/crictl/nerdctl.

→ Then check the Pod's `imagePullSecrets`.
```

### Specific commands

Check Pod events:

```bash
kubectl describe pod PodName -n Namespace
```

Check containerd configuration on the node:

```bash
cat /etc/containerd/config.toml
```

Check Harbor's `hosts.toml` file:

```bash
cat /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

Test image pulling using crictl:

```bash
crictl pull 10.0.0.10:8090/project/nginx:latest
```

Test image pulling using ctr:

```bash
ctr -n k8s.io images pull 10.0.0.10:8090/project/nginx:latest
```

Check the Secret:

```bash
kubectl get secret -n Namespace
```

Check Pod YAML:

```bash
kubectl get pod PodName -n Namespace -o yaml
```

---

## Scenario 106: "http: server gave HTTP response to HTTPS client"

### Cause

- The client defaults to accessing the repository via HTTPS.
- However, Harbor actually uses HTTP.

### Solution

For Docker:

- Configure `insecure-registries`.

For containerd:

- Configure `hosts.toml` and set `server` to `http`.

### Example of Docker configuration

```json
{
  "insecure-registries":```bash
cat /etc/containerd/config.toml
```→ Verify on the node side using crictl / ctr / nerdctl.  
→ Check the logs of kubelet and containerd.  

Production recommendations:  

→ In production environments for Harbor, HTTPS should be used preferentially.  
→ HTTP Harbor is only suitable for internal network testing or controlled environments.  
→ A successful Docker login does not necessarily mean that K8s can pull images successfully.  
→ When using containerd in K8s, focus on checking containerd rather than just Docker.  
→ For private image authentication, use imagePullSecrets preferentially.  
→ When troubleshooting containerd namespaces, explicitly specify k8s.io.