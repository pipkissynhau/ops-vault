# 09-Harbor, containerd, and Kubernetes Private Image Pulling

#Docker #containerd #Harbor #PrivateWarehouse #Kubernetes #imagePullSecrets #nerdctl #ctr #crictl #HttpRepository #Transport #TheBarrier.

---

## Recommended Path

03-Container Technology/09-Harbor, containerd, and Kubernetes Private Image Pulling.md

---

## I. Document Description

This document organizes common operations for pulling private images from Harbor, Docker, containerd, and Kubernetes, with focus on:

- Harbor Login
- Image Tagging
- Image Push
- Image Pull
- Docker Trusting HTTP Harbor
- containerd Basic Configuration
- containerd Data Directory
- containerd Trusting HTTP Harbor
- `hosts.toml` Configuration
- `nerdctl` Common Usage
- `ctr` Common Usage
- `crictl` Common Usage
- containerd Namespace
- Kubernetes Defaults to Using `k8s.io` Namespace
- Why `nerdctl login` Is Not Equal to Pod Being Able to Pull Private Images
- Kubernetes Official Private Image Pulling Method
- HTTP Harbor Common Error Troubleshooting
- Private Repository Authentication Failure Troubleshooting
- Image Push Rejection Troubleshooting

Goals:

- Can log in to Harbor
- → Can tag/push/pull images
- → Can differentiate Docker and containerd registry configurations
- → Can make Docker trust HTTP Harbor
- → Can make containerd trust HTTP Harbor
- → Can use nerdctl/ctr/crictl for node-side verification
- → Can understand containerd namespace
- → Can troubleshoot Kubernetes private image pulling failures

---

## II. Harbor Login and Image Push/Pull

---

## Scenario 75: Logging into Harbor

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

`docker login` is used to log in to a Docker image registry.

The Harbor address here is:

```text
10.0.0.10:8090
```

After successful login, the local Docker client will save authentication information.

Common uses:

- Pull private images
- Push images to Harbor
- Verify username/password correctness
- Verify Harbor accessibility

---

## Scenario 76: Directly Using Username/Password in Command Line

### Command

```bash
docker login 10.0.0.10:8090 -u admin -p Harbor12345
```

### Explanation

- Can log in directly
- But plaintext passwords in command line are insecure
- In production environments, interactive input is recommended

### Security Note

Not recommended to use the following format long-term:

```bash
docker login 10.0.0.10:8090 -u admin -p Harbor12345
```

Reasons:

- Passwords may enter shell history
- Passwords may be visible in process lists
- Passwords may be captured by log systems
- Does not meet production security requirements

Preferred method: interactive input

```bash
docker login 10.0.0.10:8090
```

---

## Scenario 77: Tagging

### Command

```bash
docker tag nginx:latest 10.0.0.10:8090/project/nginx:latest
```

### Explanation

Before pushing images to Harbor, they usually need to be tagged in the Harbor registry format.

The format is generally:

```text
HarborAddress/Project name/Mirror Name:Label
```

Example:

```text
10.0.0.10:8090/project/nginx:latest
```

Meaning:

```text
Harbor Address:10.0.0.10:8090
Project name:project
Mirror Name:nginx
Label:latest
```

---

## Scenario 78: Pushing Images

### Command

```bash
docker push 10.0.0.10:8090/project/nginx:latest
```

### Explanation

Before pushing, ensure:

- Harbor address is correct
- Harbor project exists
- Image tag format is correct
- Current user is logged in
- Current user has push permissions

---

## Scenario 79: Pulling Images

### Command

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

### Explanation

Before pulling private images, ensure:

- Current machine can access Harbor
- Harbor address is correct
- Image path is correct
- Project permissions allow pulling
- If it's a private project, login is required beforehand

---

## Scenario 80: Login Verification

### Method 1: Try Pull

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

### Method 2: Check Local Authentication Info

```bash
cat ~/.docker/config.json
```

### Explanation

If `docker pull` can pull private images normally, it indicates Docker client login and access are basically normal.

Check local authentication info:

```bash
cat ~/.docker/config.json
```

Can confirm if Docker has saved authentication for the corresponding registry.

Note:

```text
~/.docker/config.json It may contain sensitive authentication information and should not be disclosed.
```

---

## III. Trusting HTTP Harbor (Docker)

Many self-hosted Harbors use HTTP in internal networks instead of HTTPS.

In such cases, Docker defaults to not trusting it, requiring manual configuration of insecure registry.

---

## Scenario 81: Docker Trusting HTTP Harbor

### Configuration File

```bash
/etc/docker/daemon.json
```

### Example

```json
{
  "insecure-registries": ["10.0.0.10:8090"]
}
```

### Effective Steps

Reload systemd:

```bash
systemctl daemon-reload
```

Restart Docker:

```bash
systemctl restart docker
```

### Explanation

If Harbor uses HTTP, and Docker defaults to HTTPS access, errors like:

```text
http: server gave HTTP response to HTTPS client
```

may occur.

In this case, configure:

```json
{
  "insecure-registries": ["10.0.0.10:8090"]
}
```

in Docker's `daemon.json`.

---

## Scenario 82: Verifying Docker Trusts Harbor

### Command

```bash
docker info
```

Check if the output contains:

```text
Insecure Registries:
  10.0.0.10:8090
```

### Explanation

If `docker info` shows Harbor address under `Insecure Registries`, it means Docker has recognized the HTTP registry.

After that, you can test login:

```bash
docker login 10.0.0.10:8090
```

Test pulling:

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

---

## IV. containerd Basics and Data Directory

---

## Scenario 83: Check containerd Version and Status

Check version:

```bash
containerd --version
```

Check service status:

```bash
systemctl status containerd
```

### Explanation

containerd is a common container runtime.

In Kubernetes, if nodes use containerd, kubelet will use containerd for pulling images and creating containers.

Therefore, when troubleshooting Kubernetes image pulling issues, you cannot only check Docker.

---

## Scenario 84: containerd Configuration File

Default configuration file:

```bash
/etc/containerd/config.toml
```

### Explanation

containerd's data directory, status directory, registry configuration, etc., are controlled here.

Commonly watched items include: /think

- `root`
- `state`
- registry configuration
- sandbox image
- runtime configuration
- registry `config_path`

---

## Scenario 85: containerd Default Data Directory

Common default values:

```toml
root  = "/var/lib/containerd"
state = "/run/containerd"
```

### Meaning

`root`

- Persistent data directory
- Images, snapshots, metadata, etc. are usually stored here

`state`

- Runtime state directory
- Typically located in `/run`

### Operations Understanding

`root` is similar to Docker's data directory, belonging to persistent directories.

`state` is more oriented toward runtime state, usually located in `/run`.

If the system disk is small, attention should also be paid to `/var/lib/containerd` disk usage.

---

## Scenario 86: Modifying containerd Data Directory

### Configuration Example

```toml
root = "/data/containerd"
state = "/run/containerd"
```

### Explanation

- Similar to Docker's `data-root`
- Suitable for migrating image and snapshot data to the data disk
- It's best to plan uniformly before Kubernetes cluster deployment

### Restart containerd After Modification

```bash
systemctl restart containerd
```

### Verify Status

```bash
systemctl status containerd
```

---

## Scenario 87: Why Optimize containerd Directories Before K8s Deployment

Reasons:

- Belongs to node baseline
- Avoid migrating image and container data after cluster is already running
- Avoid affecting kubelet / Pod operations
- More suitable for batch and template delivery

### Operations Understanding

If a Kubernetes cluster already runs many Pods, migrating containerd data directory later would carry higher risks.

Potential impacts:

- Node image data
- Container snapshot data
- Pod startup
- kubelet communication with containerd
- Node stability

Therefore, it's more recommended to plan during node initialization phase:

```toml
root = "/data/containerd"
```

---

## Section 5: Trust HTTP Harbor (containerd)

If Kubernetes nodes or `crictl / ctr / nerdctl` pull images via containerd, configuring only Docker login and insecure registry is often insufficient.

---

## Scenario 88: containerd Configuration for HTTP Harbor

### Directory

```bash
/etc/containerd/certs.d/10.0.0.10:8090/
```

### File

```bash
/etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

### Example Content

```toml
server = "http://10.0.0.10:8090"

[host."http://10.0.0.10:8090"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
```

### Explanation

This configuration tells containerd:

```text
Visits 10.0.0.10:8090 use HTTP
```

instead of default HTTPS access.

Directory name needs to match the repository address:

```bash
/etc/containerd/certs.d/10.0.0.10:8090/
```

File name is:

```bash
hosts.toml
```

---

## Scenario 89: containerd Main Configuration Check

### Configuration File

```bash
/etc/containerd/config.toml
```

### Configuration to Pay Attention To

```toml
[plugins."io.containerd.cri.v1.images".registry]
  config_path = "/etc/containerd/certs.d"
```

### Explanation

- `config_path` should point to `certs.d` directory
- Otherwise `hosts.toml` won't take effect

### Operations Understanding

Just creating:

```bash
/etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

isn't enough.

You also need to confirm:

```toml
config_path = "/etc/containerd/certs.d"
```

Otherwise containerd may not read registry configurations in this directory.

---

## Scenario 90: Restart containerd

### Command

```bash
systemctl restart containerd
```

### Explanation

After modifying containerd configuration, restart containerd is required.

After restart, it's recommended to check status:

```bash
systemctl status containerd
```

If it's a Kubernetes node, also observe kubelet and Pod status:

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

---

## Section 6: nerdctl, ctr, crictl and namespace

---

## Scenario 91: What is nerdctl

`nerdctl` is a Docker-like CLI tool in containerd ecosystem, with syntax style similar to Docker.

### Common Commands

Check images:

```bash
nerdctl images
```

Check containers:

```bash
nerdctl ps -a
```

Login to registry:

```bash
nerdctl login 10.0.0.10:8090
```

### Explanation

`nerdctl` is more friendly to containerd users, with many commands similar to Docker.

Commonly used for:

- Node-side debugging
- Verifying if containerd can pull images
- Viewing images and containers managed by containerd
- Logging into private registry for temporary verification

---

## Scenario 92: Can nerdctl Log in to Harbor

### Command

```bash
nerdctl login 10.0.0.10:8090
```

### Explanation

- Can be used for node-side debugging and verification
- Indicates the containerd client environment on current node can try to access registry with this credential

But note:

```text
nerdctl login

≠ Kubernetes Pod I'm sure we can pull a private mirror directly.
```

### More Accurate Understanding

- `nerdctl login`: More oriented toward node-side debugging / verification
- `imagePullSecrets`: More oriented toward Kubernetes formal pull mechanism

---

## Scenario 93: ctr Common Commands

Check images:

```bash
ctr -n k8s.io images ls
```

Pull images:

```bash
ctr -n k8s.io images pull 10.0.0.10:8090/project/nginx:latest
```

### Explanation

`ctr` is containerd's built-in low-level CLI tool.

Features:

- Closer to containerd's underlying layer
- Parameters are less friendly than Docker
- Frequently used for troubleshooting containerd issues
- When Kubernetes uses containerd, typically need to pay attention to `k8s.io` namespace

---

## Scenario 94: crictl Common Commands

Check Pod sandbox:

```bash
crictl pods
```

Check containers:

```bash
crictl ps -a
```

Pull images:

```bash
crictl pull 10.0.0.10:8090/project/nginx:latest
```

Check images:

```bash
crictl images
```

### Explanation

`crictl` is a CRI-oriented debugging tool.

On Kubernetes nodes, `crictl` is commonly used for troubleshooting kubelet and container runtime issues.

Common uses:

- Check Pod sandbox
- Check containers
- Check images
- Manually test image pulling
- Assist in troubleshooting kubelet / containerd issues

---

## Scenario 95: containerd namespace Concept

containerd has namespace concept, common ones are:

```text
default

k8s.io
```

### Explanation

- When manually using `nerdctl / ctr` for testing, may be in `default`
- When Kubernetes uses containerd, default is:

```text
k8s.io
```

### Operations Understanding

containerd's namespace can be understood as logical isolation space.

Images and containers in different namespaces may have different views.

So seeing no image in one namespace doesn't mean it's missing in another.

---

## Scenario 96: Where are Kubernetes default images and containers located?

Conclusion:

```text
Kubernetes Use containerd I don't know.

Pod Pulling, managing mirrors and containers,

Defaults are in place. k8s.io namespace Lee.
```

### Common Commands

View Kubernetes images:

```bash
ctr -n k8s.io images ls
```

Or:

```bash
nerdctl --namespace k8s.io images ls
```

### Explanation

When troubleshooting Kubernetes node images, it's not recommended to only execute:

```bash
ctr images ls
```

Instead, it's better to explicitly specify:

```bash
ctr -n k8s.io images ls
```

---

## Scenario 97: Why can't you see images with `ctr images ls` even though the Pod is running?

Common reasons:

- You're looking at the wrong namespace
- You might be looking at `default`
- But Kubernetes actually uses `k8s.io`

### Correct way to view

```bash
ctr -n k8s.io images ls
```

Or:

```bash
nerdctl --namespace k8s.io images ls
```

---

## Scenario 98: Explicitly specifying namespace

### nerdctl

```bash
nerdctl --namespace k8s.io images ls
```

### ctr

```bash
ctr -n k8s.io images ls
```

### Explanation

- The safest way is to explicitly include `--namespace` or `-n`
- It's not advisable to overstate environment variable support in verbal descriptions
- Explicit parameters are clearest in practice

---

## VII. Supplementary Understanding of Kubernetes Private Image Pulling

---

## Scenario 99: Why does `nerdctl login` not guarantee a Pod can pull a private image?

Because Kubernetes officially pulling private images usually also considers:

- Whether containerd can access the registry
- Whether HTTP/HTTPS is trusted
- Whether the node network is reachable
- Whether the Pod has configured `imagePullSecrets`

### More accurate understanding

Node-side debugging:

```bash
nerdctl login
```

```bash
ctr pull
```

```bash
crictl pull
```

K8s official usage:

```text
imagePullSecrets
```

### Operations Understanding

`nerdctl login` success only indicates that the node-side tool has successfully logged in to Harbor.

But Kubernetes pulling images also involves:

- kubelet
- containerd CRI plugin
- Pod spec
- imagePullSecrets
- Secrets in the namespace
- Harbor project permissions
- HTTP / HTTPS configuration

Therefore, you cannot simply assume:

```text
nerdctl login Success = Pod I'm sure it'll work.
```

---

## Scenario 100: Recommended way for Kubernetes to officially pull private images

### Using imagePullSecrets

Explanation:

- This is the official and general private registry authentication method of Kubernetes
- It's more standardized than relying solely on node-side login

### Example of creating imagePullSecret

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=10.0.0.10:8090 \
  --docker-username=admin \
  --docker-password='Harbor12345' \
  --docker-email=admin@example.com \
  -n default
```

### Example of referencing in Pod

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

### Example of referencing in Deployment

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

`imagePullSecrets` must be in the same namespace as the Pod.

For example, if the Pod is in the `default` namespace, the Secret must also be in the `default` namespace:

```bash
kubectl get secret -n default
```

---

## VIII. Common Issues and Troubleshooting

---

## Scenario 105: Docker login succeeds, but Kubernetes image pull fails

### Common reasons

- Docker has logged in, but containerd is not configured
- Harbor is HTTP, containerd doesn't trust it
- Missing `hosts.toml` on the node
- `config.toml` registry `config_path` is misconfigured
- Pod lacks `imagePullSecrets`

### Troubleshooting approach

```text
Look. kubelet / containerd Report http/https Error

→ Look. /etc/containerd/certs.d Existence

→ Look. hosts.toml Is it correct?

→ Look. config.toml Yes. config_path

→ Use ctr / crictl / nerdctl Test

→ Look again. Pod Yes. imagePullSecrets
```

### Specific commands

Check Pod events:

```bash
kubectl describe pod PodName -n Namespace
```

Check node containerd configuration:

```bash
cat /etc/containerd/config.toml
```

Check Harbor hosts.toml:

```bash
cat /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

Test pulling with crictl:

```bash
crictl pull 10.0.0.10:8090/project/nginx:latest
```

Test pulling with ctr:

```bash
ctr -n k8s.io images pull 10.0.0.10:8090/project/nginx:latest
```

Check Secret:

```bash
kubectl get secret -n Namespace
```

Check Pod YAML:

```bash
kubectl get pod PodName -n Namespace -o yaml
```

---

## Scenario 106: `http: server gave HTTP response to HTTPS client`

### Reason

- Client defaults to HTTPS access to the registry
- But Harbor is actually HTTP

### Solution

Docker:

- Configure `insecure-registries`

containerd:

- Configure `hosts.toml`, write `server` as `http`

### Docker configuration example

```json
{
  "insecure-registries": ["10.0.0.10:8090"]
}
```

Restart Docker:

```bash
systemctl restart docker
```

### containerd configuration example

```toml
server = "http://10.0.0.10:8090"

[host."http://10.0.0.10:8090"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
```

Restart containerd:

```bash
systemctl restart containerd
```

---

## Scenario 107: `unauthorized: authentication required`

### Reason

- Not logged in
- Login account lacks permissions
- Project name is incorrect
- Harbor project is not public
- Pod lacks `imagePullSecrets`

### Troubleshooting

```text
docker login / nerdctl login

→ Confirm project path

→ Confirm account permissions

→ Confirm whether the project is private

→ Confirm. K8s secret Correct?
```

### Common commands

Docker login:

```bash
docker login 10.0.0.10:8090
```

nerdctl login:

```bash
nerdctl login 10.0.0.10:8090
```

Confirm image path:

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

Check K8s Secret:

```bash
kubectl get secret -n Namespace
```

Check if Pod references Secret:

```bash
kubectl get pod PodName -n Namespace -o yaml
```

---

## Scenario 108: Image push is rejected

### Common reasons

- Tag name doesn't match Harbor project path
- Project doesn't exist
- Login user lacks push permissions

### Correct format example

```text
10.0.0.10:8090/project/nginx:latest
```

### Troubleshooting commands

Check local images:

```bash
docker images
```

Redo tagging:

```bash
docker tag nginx:latest 10.0.0.10:8090/project/nginx:latest
```

Login to Harbor:

```bash
docker login 10.0.0.10:8090
```

Push image:

```bash
docker push 10.0.0.10:8090/project/nginx:latest
```

### Explanation

If push is rejected, you need to confirm:

- Whether the `project` project exists in Harbor
- Whether the current user has push permissions for the project
- Whether the image tag is written as the full Harbor path
- Whether Harbor has access control enabled
- Whether the Harbor project is read-only or has restricted permissions

---

## IX. Complete Troubleshooting Flow Example

---

## 1. Can Docker pull Harbor images?

Login to Harbor:

```bash
docker login 10.0.0.10:8090
```

Pull image:

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

If it fails, check whether Docker trusts HTTP Harbor: /think

```bash
docker info
```

Check for:

```text
Insecure Registries:
  10.0.0.10:8090
```

---

## 2. Can containerd Pull Harbor Images?

Check containerd configuration:

```bash
cat /etc/containerd/config.toml
```

Check `hosts.toml`:

```bash
cat /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

Test with `crictl`:

```bash
crictl pull 10.0.0.10:8090/project/nginx:latest
```

Test with `ctr`:

```bash
ctr -n k8s.io images pull 10.0.0.10:8090/project/nginx:latest
```

---

## 3. Does Kubernetes Pod Have imagePullSecrets Configured?

Check Secret:

```bash
kubectl get secret -n default
```

Check Pod YAML:

```bash
kubectl get pod PodName -n default -o yaml
```

Focus on:

```yaml
imagePullSecrets:
  - name: harbor-secret
```

Check Pod events:

```bash
kubectl describe pod PodName -n default
```

---

## 4. Check Images on K8s Nodes

Use `ctr`:

```bash
ctr -n k8s.io images ls
```

Use `nerdctl`:

```bash
nerdctl --namespace k8s.io images ls
```

Use `crictl`:

```bash
crictl images
```

---

## Ten. Common Commands Summary

---

## Harbor

Login:

```bash
docker login 10.0.0.10:8090
```

Login with credentials:

```bash
docker login 10.0.0.10:8090 -u admin -p Harbor12345
```

Tag:

```bash
docker tag nginx:latest 10.0.0.10:8090/project/nginx:latest
```

Push:

```bash
docker push 10.0.0.10:8090/project/nginx:latest
```

Pull:

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

Check Docker login info:

```bash
cat ~/.docker/config.json
```

---

## Docker Trusting HTTP Harbor

Config file:

```bash
/etc/docker/daemon.json
```

Config example:

```json
{
  "insecure-registries": ["10.0.0.10:8090"]
}
```

Reload systemd:

```bash
systemctl daemon-reload
```

Restart Docker:

```bash
systemctl restart docker
```

Verify:

```bash
docker info
```

---

## containerd

Check version:

```bash
containerd --version
```

Check status:

```bash
systemctl status containerd
```

Check configuration:

```bash
cat /etc/containerd/config.toml
```

Restart:

```bash
systemctl restart containerd
```

Check containerd data directory config:

```bash
grep -E 'root|state' /etc/containerd/config.toml
```

---

## containerd Trusting HTTP Harbor

Create directory:

```bash
mkdir -p /etc/containerd/certs.d/10.0.0.10:8090/
```

Edit hosts.toml:

```bash
vi /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

`hosts.toml` Example:

```toml
server = "http://10.0.0.10:8090"

[host."http://10.0.0.10:8090"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
```

Check `config_path`:

```bash
grep -n "config_path" /etc/containerd/config.toml
```

Need to pay attention to:

```toml
[plugins."io.containerd.cri.v1.images".registry]
  config_path = "/etc/containerd/certs.d"
```

Restart containerd:

```bash
systemctl restart containerd
```

---

## nerdctl / ctr / crictl

Login to Harbor:

```bash
nerdctl login 10.0.0.10:8090
```

Check nerdctl images:

```bash
nerdctl images
```

Check nerdctl containers:

```bash
nerdctl ps -a
```

Check K8s namespace images:

```bash
nerdctl --namespace k8s.io images ls
```

Check K8s namespace images (ctr):

```bash
ctr -n k8s.io images ls
```

Pull image (ctr):

```bash
ctr -n k8s.io images pull 10.0.0.10:8090/project/nginx:latest
```

Check Pod sandbox:

```bash
crictl pods
```

Check containers:

```bash
crictl ps -a
```

Pull image (crictl):

```bash
crictl pull 10.0.0.10:8090/project/nginx:latest
```

Check images (crictl):

```bash
crictl images
```

---

## Kubernetes imagePullSecrets

Create private registry Secret:

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=10.0.0.10:8090 \
  --docker-username=admin \
  --docker-password='Harbor12345' \
  --docker-email=admin@example.com \
  -n default
```

Check Secret:

```bash
kubectl get secret -n default
```

Check Pod YAML:

```bash
kubectl get pod PodName -n default -o yaml
```

Check Pod events:

```bash
kubectl describe pod PodName -n default
```

---

## Troubleshooting

Check nodes:

```bash
kubectl get nodes
```

Check all Pods:

```bash
kubectl get pods -A
```

Check Pod details:

```bash
kubectl describe pod PodName -n Namespace
```

Check kubelet logs:

```bash
journalctl -u kubelet -f
```

Check containerd logs:

```bash
journalctl -u containerd -f
```

---

## Eleven. One-Sentence Summary

The core relationship between Harbor, Docker, containerd, and Kubernetes private image pulling is:

```text
Harbor
→ Mirror repository

Docker
→ You can log in, call. tagI don't know.pushI don't know.pull

containerd
→ Kubernetes Runtime when nodes are actually used

nerdctl / ctr / crictl
→ Node Side Validation and Disable Tool

imagePullSecrets
→ Kubernetes Authentication for official extraction of private mirrors
```

HTTP Harbor needs to be viewed separately:

```text
Docker Use HTTP Harbor
→ Configure insecure-registries

containerd Use HTTP Harbor
→ Configure certs.d/Warehouse Address/hosts.toml
→ Confirm. config_path Point /etc/containerd/certs.d
```

containerd namespace needs to be remembered:

```text
The manual test is possible. default namespace

Kubernetes Use containerd Focus. k8s.io namespace
```

Common check commands:

```bash
ctr -n k8s.io images ls
```

```bash
nerdctl --namespace k8s.io images ls
```

Kubernetes private image pull failure troubleshooting path:

```text
kubectl describe pod Look at the events.
→ Make sure the mirror address is correct.
→ Confirm. Harbor Items and authority
→ Confirm. imagePullSecrets
→ Confirm nodal access Harbor
→ Confirm. containerd Trust? HTTP Harbor
→ Use crictl / ctr / nerdctl Verify by Node
→ View kubelet / containerd Log
```

Production recommendations:

```text
Harbor Priority use in the production environment HTTPS
HTTP Harbor Only for Intranet testing or controlled environments
Docker Login Success does not mean K8s I'm sure it'll work.
K8s Use containerd Focus on time. containerdNot just looking. Docker
Private mirror authentication priority imagePullSecrets
containerd namespace Visibility designation during Query k8s.io
```