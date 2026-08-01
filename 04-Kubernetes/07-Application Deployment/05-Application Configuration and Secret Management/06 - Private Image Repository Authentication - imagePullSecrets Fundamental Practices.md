# 06 - Private Image Registry Authentication: imagePullSecrets Basics Practice

## Document Notes
- Document Positioning: Private Image Registry Pull Authentication Introduction Practice
- Applicable Stage: After completing image building, pushing, and Deployment deployment basics, entering private registry authentication and image pull flow learning
- Recommended Path: `04-Kubernetes/07-Apply deployment/05-Apply Configuration and Key Management/06-Private mirror warehouse certification:imagePullSecrets Basic practice`

## Tags
#Kubernetes #Secret #imagePullSecrets #Harbor #PrivateMirrorWarehouse #MirrorPull #ErrImagePull #ImagePullBackOff #containerd #Docker #ConfigurationManagement #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why Must We Learn imagePullSecrets

Previously, we've completed these basic flows:

- Dockerfile building image
- Image naming and tag
- Push image to registry
- Deployment using image to start Pod

But there's a very practical problem here:

**If the image is in a private registry, why can the node pull it down?**

In local environments, many people tend to ignore this issue because the local machine might have already executed:

- `docker login`
- Or cached the image locally

However, Kubernetes nodes won't automatically inherit the local login status.  
For nodes, if they don't have appropriate registry authentication information, they might encounter:

- `ErrImagePull`
- `ImagePullBackOff`

These issues are very common in real environments and are one of the high-frequency entry points for application deployment troubleshooting.

Therefore, the significance of learning `imagePullSecrets` lies in:

**Enabling Kubernetes to have valid authentication credentials when pulling private images.**

---

## II. What Problem Does imagePullSecrets Solve

It solves:

**How a Pod can use the registry authentication information saved in Kubernetes to pull private images when starting.**

In other words, it's not a Secret for the application to use, but rather:

**A Secret for Kubernetes to use when pulling images.**

### You Can Understand It This Way

- Regular Secret: Read by business applications
- imagePullSecrets: Read by Kubernetes when pulling images

---

## III. Why It Belongs to the Secret System

Because registry authentication information is essentially sensitive data, such as:

- Registry username
- Registry password
- Access Token
- Docker config json

These contents obviously shouldn't be hard-coded in:

- Deployment YAML
- Image
- Plain text scripts

So Kubernetes also uses Secrets to save them.

In other words:

**The underlying of imagePullSecrets is still a Secret, but its purpose is image pull authentication.**

---

## IV. When Will imagePullSecrets Be Needed

The following scenarios typically require it:

### 1. Image is in a private Harbor

For example, enterprise internal Harbor projects that require login to pull.

### 2. Image is in a cloud vendor's private registry

For example:

- Alibaba Cloud ACR
- Tencent Cloud TCR
- Huawei Cloud SWR
- AWS ECR
- GCR private image

### 3. Nodes haven't pre-logged into the registry

Even if the local machine has logged in, the nodes themselves may not have logged in.

### 4. Cluster is a multi-node environment

Even if one node has pulled the image before, it doesn't mean other nodes have authentication information.

---

## V. When imagePullSecrets Are Usually Not Needed

### 1. Using public image registry

For example, directly pulling public ones:

- `nginx`
- `busybox`
- `redis`

### 2. Nodes already have available authentication

Some environments will pre-configure registry authentication at the node level, but this approach is less standardized than Kubernetes internal management.

### 3. Using local development single-node environment with pre-loaded image

For example, some local experimental environments have already imported the image into the node.

### Operation Understanding Focus

From a standard perspective, any formal private image usage scenario should prioritize:

**Let Kubernetes manage registry pull credentials through Secret formally.**

---

## VI. Understanding Two Common Pull Failure States

Before entering imagePullSecrets, first understand two common error states.

### 1. `ErrImagePull`

Indicates an error occurred when the node tried to pull the image.

### 2. `ImagePullBackOff`

Indicates the node entered a backoff retry state after failing to pull the image.

### Common Causes of These Two States Include

- Registry address written incorrectly
- Image name written incorrectly
- Tag doesn't exist
- Registry network unreachable
- Registry certificate anomaly
- No authentication information
- Authentication information is incorrect

### Operation Understanding Focus

When entering application deployment troubleshooting later, these two states will almost certainly be encountered.

---

## VII. What Is the Overall Working Logic of imagePullSecrets

First, establish a complete flow:

### 1. Pod defines a private image address

For example:

    registry.cn-hangzhou.aliyuncs.com/pri-syq/nginx-web:v1

Or:

    harbor.example.com/project/nginx-web:v1

### 2. Pod also declares `imagePullSecrets`

Informing Kubernetes:

- When pulling this image, use which Secret for authentication

### 3. Node receives scheduling task

kubelet prepares to pull the image.

### 4. kubelet reads the registry authentication information from the corresponding Secret

And accesses the image registry with these authentication credentials.

### 5. After authentication success, pull the image

Image pull succeeds, container continues to start.

---

## VIII. What Is the Most Common Secret Type

For image registry authentication, the most common Secret type is usually:

    kubernetes.io/dockerconfigjson

This type is specifically used to save Docker / OCI registry authentication information.

### Operation Understanding Focus

It's different from the ordinary `Opaque` Secret, with clearer semantics:

**This is a Secret for image registry authentication.**

---

## IX. What Is a Typical Image Pull Secret Structure

In actual use, people often don't manually write the full JSON content, but instead generate it through commands.  
But from a understanding perspective, we can first know:

This Secret usually saves information similar to Docker login after authentication.

In other words, it's essentially:

- Registry address
- Username
- Password or token
- Encoded authentication field

Encapsulated into a Kubernetes-recognized Secret object.

---

## X. Example 1: Using Alibaba Cloud Private Registry

Here's an example of a private image:

    registry.cn-hangzhou.aliyuncs.com/pri-syq/nginx-web:v1

Namespace is:

    test

If testing locally, you'd typically first execute: /think

docker login --username=syq1013 registry.cn-hangzhou.aliyuncs.com

Enter the password after execution, allowing the local machine to pull this private image.

Example:

    docker pull registry.cn-hangzhou.aliyuncs.com/pri-syq/nginx-web:v1

But here we must pay special attention:

**Executing `docker login` on the local machine does not automatically grant the Kubernetes cluster nodes this authentication capability.**

Therefore, in the cluster, it's common to save the repository authentication information as a Secret and then provide it to the Pod via `imagePullSecrets`.

---

## Eleven. Security Reminder: Do not write real repository passwords directly into command history or version control

In actual operations, the following approach may work but is not secure:

    kubectl create secret docker-registry aliyun-registry-secret \
      --namespace=test \
      --docker-server=registry.cn-hangzhou.aliyuncs.com \
      --docker-username=syq1013 \
      --docker-password='real password' \
      --docker-email=example@example.com

Main risks include:

- Real password may enter shell history
- Bastion host audit, session replay, screen recording may leave traces
- Copying commands, screenshots, or document organization may inadvertently include passwords
- If written into YAML and submitted to Git / Gitee, it creates long-term risks

### Operations Understanding Focus

In production environments, it's not recommended to:

- Directly write real passwords into command parameters
- Directly write real passwords into submittable Secret YAML
- Include Secret files with sensitive information in version control

Therefore, subsequent examples will uniformly use:

- `<REGISTRY_PASSWORD>`
- `<BASE64_OF_USERNAME_COLON_PASSWORD>`
- `<HARBOR_PASSWORD>`

These placeholders instead of real passwords.

---

## Twelve. Basic Writing Method for Creating imagePullSecrets via Command

First ensure the namespace exists:

    kubectl create namespace test

If `test` already exists, there's no need to recreate it.

Common commands for creating image pull Secret are as follows:

    kubectl create secret docker-registry aliyun-registry-secret \
      --namespace=test \
      --docker-server=registry.cn-hangzhou.aliyuncs.com \
      --docker-username=syq1013 \
      --docker-password='<REGISTRY_PASSWORD>' \
      --docker-email=example@example.com

### Parameter Explanation

- `docker-registry`
  Indicates creating a Secret specifically for image repository authentication

- `aliyun-registry-secret`
  Indicates the Secret name, to be referenced later in the Pod

- `--namespace=test`
  Indicates the namespace where this Secret is created

- `--docker-server=registry.cn-hangzhou.aliyuncs.com`
  Indicates the image repository address

- `--docker-username=syq1013`
  Indicates the repository username

- `--docker-password='<REGISTRY_PASSWORD>'`
  Indicates the repository password or access credential, placeholder used only for illustrative purposes

- `--docker-email=example@example.com`
  Is a compatibility field that can generally be retained

---

## Thirteen. More Secure Practice Approach

If you've already executed:

    docker login --username=syq1013 registry.cn-hangzhou.aliyuncs.com

The local machine typically generates:

    ~/.docker/config.json

At this point, consider creating a Secret based on existing Docker login credentials rather than re-entering the password in the command line.

Example:

    kubectl create secret generic aliyun-registry-secret \
      --namespace=test \
      --from-file=.dockerconfigjson=$HOME/.docker/config.json \
      --type=kubernetes.io/dockerconfigjson

### Operations Understanding Focus

The advantage of this approach is:

- No need to re-enter the password directly in the command line
- More aligned with real usage scenarios
- More suitable for scenarios where `docker login` has already been completed

But also note:

- `~/.docker/config.json` itself is also a sensitive file
- In multi-user jump hosts or shared environments, pay attention to permission controls
- Should not arbitrarily copy, submit, or long-term expose this file

---

## Fourteen. Overall Understanding of imagePullSecrets YAML Writing

In actual use, there are typically two parts of YAML:

- One part is the **Secret YAML**
- One part is the **Deployment / Pod YAML referencing `imagePullSecrets`**

In other words:

- `Secret` is responsible for saving repository authentication information
- `imagePullSecrets` is responsible for referencing this authentication information in the Pod

### You Can Understand It This Way

- `secret.yaml`: Defines "what the credentials are"
- `deployment.yaml`: Defines "who uses these credentials"

---

## Fifteen. Secret YAML Writing Method One: Using `stringData`

This method is more suitable for understanding and learning the structure of `.dockerconfigjson`.

File name example:

    secret.yaml

Content as follows:

apiVersion: v1
kind: Secret
metadata:
  name: aliyun-registry-secret
  namespace: test
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "registry.cn-hangzhou.aliyuncs.com": {
          "username": "your-registry-user",
          "password": "<REGISTRY_PASSWORD>",
          "auth": "<BASE64_OF_USERNAME_COLON_PASSWORD>"
        }
      }
    }

### Field Explanation

- `name: aliyun-registry-secret`
  Represents the name of this Secret

- `namespace: test`
  Represents the namespace where this Secret is located `test`

- `type: kubernetes.io/dockerconfigjson`
  Represents the type of Secret specifically for image registry authentication

- `.dockerconfigjson`
  Represents the content of the registry authentication information, formatted identically to Docker registry authentication structure

### Operations Understanding Focus

Here:

    "auth": "<BASE64_OF_USERNAME_COLON_PASSWORD>"

is not the Base64 encoding of the password,  
but the entire following content encoded as Base64:

    your-registry-user:<REGISTRY_PASSWORD>

---

## Sixteen. Why `stringData` is More Suitable for Learning

Because `stringData` can directly write raw strings, Kubernetes automatically converts them to the underlying `data` format when creating Secrets.

This is more intuitive for understanding structure and less error-prone during the learning phase compared to manual Base64 encoding.

However, note:

**If real passwords are written here, this YAML file becomes a sensitive file.**

Therefore, it's recommended to always use placeholders like:

- `<REGISTRY_PASSWORD>`
- `<BASE64_OF_USERNAME_COLON_PASSWORD>`

in knowledge bases, notes, and Gitee examples, rather than writing actual passwords.

---

## Seventeen. Secret YAML Writing Method Two: Using `data`

This method is closer to the actual format Kubernetes stores internally.

File name example:

    secret.yaml

Content:

    apiVersion: v1
    kind: Secret
    metadata:
      name: aliyun-registry-secret
      namespace: test
    type: kubernetes.io/dockerconfigjson
    data:
      .dockerconfigjson: <BASE64_ENCODED_DOCKER_CONFIG_JSON>

### Field Explanation

Here:

    <BASE64_ENCODED_DOCKER_CONFIG_JSON>

represents the result of Base64 encoding the entire `.dockerconfigjson` content.

This method is more standard but less readable, making it less intuitive for learning compared to `stringData`.

### Operations Understanding Focus

- `stringData`: More suitable for understanding and manual writing
- `data`: Closer to the final stored format

---

## Eighteen. YAML Writing Method for imagePullSecrets in Deployment

After creating `Secret`, you need to explicitly reference it in the Pod template.

File name example:

    deployment.yaml

Content:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-web
      namespace: test
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx-web
      template:
        metadata:
          labels:
            app: nginx-web
        spec:
          imagePullSecrets:
            - name: aliyun-registry-secret
          containers:
            - name: nginx-web
              image: registry.cn-hangzhou.aliyuncs.com/pri-syq/nginx-web:v1
              imagePullPolicy: IfNotPresent
              ports:
                - containerPort: 80

### What Does This Section Mean

This segment:

    imagePullSecrets:
      - name: aliyun-registry-secret

means:

**The current Pod will use the Secret named `aliyun-registry-secret` as the registry authentication information when pulling images.**

---

## Nineteen. How to Apply Dual-File Version

If using YAML files, you can create them like this:

    kubectl apply -f secret.yaml
    kubectl apply -f deployment.yaml

After creation, you can check with these commands:

    kubectl get secret -n test
    kubectl get pod -n test
    kubectl describe pod -n test <pod-name>

---

## Twenty. Supplementary Understanding: Where Does `auth` Come From in `.dockerconfigjson`

In Secrets of type `kubernetes.io/dockerconfigjson`, the common authentication content typically looks like this:

{
  "auths": {
    "registry.cn-hangzhou.aliyuncs.com": {
      "username": "your-registry-user",
      "password": "<REGISTRY_PASSWORD>",
      "auth": "<BASE64_OF_USERNAME_COLON_PASSWORD>"
    }
  }
}

The most commonly misunderstood point here is:

**`auth` is not Base64 of the password alone, but Base64 of `username:password` as a whole.**

For example:

- Username: `your-registry-user`
- Password: `<REGISTRY_PASSWORD>`

The correct approach is to encode the entire following content:

    your-registry-user:<REGISTRY_PASSWORD>

---

## 21. Why You Can't Arbitrarily Use `echo` for Base64

Many people generate Base64 by directly executing:

    echo "example-password" | base64

Or:

    echo -e "example-password" | base64

This approach often includes the **newline character** in the encoding.  
In other words, the actual content being encoded is not:

    example-password

But:

    example-password\n

This leads to Base64 results that differ from expectations.  
If this error occurs in `auth` or Secret content, it may further cause image authentication failures.

### Operations Understanding Focus

In scenarios involving image repository authentication, Token, Secret, and certificate content processing, **an extra newline character can cause authentication failures**.

---

## 22. Correct Base64 Encoding Method

If you're simply verifying a string itself, use:

    echo -n 'example-password' | base64

Here, `-n` means **do not append a newline character at the end**.

---

## 23. Correct Generation Method for `auth` Field

If you need to generate the common `auth` field in Docker registry authentication, you should Base64 the entire:

    username:password

For example:

- Username: `your-registry-user`
- Password: `example-password`

The correct command should be:

    echo -n 'your-registry-user:example-password' | base64

The resulting output is suitable for placement in:

    "auth": "..."

### Operations Understanding Focus

Distinguish between two things:

#### 1. Base64 of Password Alone

    echo -n 'example-password' | base64

This is merely "Base64 of the password string itself".

#### 2. Base64 of `username:password`

    echo -n 'your-registry-user:example-password' | base64

This is the actual format required for the `.dockerconfigjson` field's `auth` field.

---

## 24. How to Reverse-Verify Base64

If you already have a Base64 string, you can decode it using:

    echo 'BASE64Contents' | base64 -d

Or:

    echo 'BASE64Contents' | base64 --decode

This can be used to check what the original content actually is.

---

## 25. How to Check if Decoded Result Contains Newline Characters

Sometimes, the decoded content appears fine but may actually have a hidden newline character at the end.  
In such cases, you can further use:

    echo 'BASE64Contents' | base64 -d | cat -A

Or:

    echo 'BASE64Contents' | base64 -d | xxd

### Understanding Method

- `cat -A` is suitable for quickly checking hidden characters
- `xxd` is suitable for confirming content at the hexadecimal level whether it contains newlines, spaces, or other invisible characters

This is very useful for troubleshooting authentication fields, Tokens, password strings, and certificate content.

---

## 26. Why imagePullSecrets Are Written in Pod Templates

Because the actual image pulling is a Pod-level behavior, not a Deployment-level behavior.

### What Deployment Manages

- Manages replicas
- Manages Pod templates
- Manages rolling updates

### What Pod Templates Manage

- Container images
- Environment variables
- Volume
- imagePullSecrets

### Operations Understanding Focus

Deployment itself does not pull images; the Pod on the node does.  
Therefore, repository authentication must appear in:

**The Pod template spec.**

---

## 27. Example 2: If It's an Enterprise Private Harbor, Are Commands and YAML Writing Methods the Same?

**Essentially, they are the same.**

Just replace these content:

- Replace `--docker-server` with the enterprise Harbor address
- Replace `--docker-username` with the Harbor username
- Replace `--docker-password` with the Harbor credential
- Replace `image` with the Harbor image path
- Secret name can be adjusted according to enterprise standards

For example, if the enterprise Harbor address is:

    harbor.example.com

The image path is:

    harbor.example.com/devops/nginx-web:v1

The namespace still uses:

    test

The local login command is usually:

    docker login harbor.example.com

Or explicitly write the username:

    docker login --username=harbor-user harbor.example.com

---

## 28. Harbor Example: Creating imagePullSecrets via Command

    kubectl create secret docker-registry harbor-registry-secret \
      --namespace=test \
      --docker-server=harbor.example.com \
      --docker-username=harbor-user \
      --docker-password='<HARBOR_PASSWORD>' \
      --docker-email=example@example.com

### Understanding Focus /think

This command has no fundamental difference from Aliyun's creation method.  
The difference lies only in the repository address and account system.

In other words:

- Aliyun uses private repository authentication
- Harbor also uses private repository authentication

Kubernetes sees only:

**"Give me a docker-registry Secret that can be used to pull private images."**

---

## 29. Harbor Example: Secret YAML Writing Method

    apiVersion: v1
    kind: Secret
    metadata:
      name: harbor-registry-secret
      namespace: test
    type: kubernetes.io/dockerconfigjson
    stringData:
      .dockerconfigjson: |
        {
          "auths": {
            "harbor.example.com": {
              "username": "harbor-user",
              "password": "<HARBOR_PASSWORD>",
              "auth": "<BASE64_OF_HARBOR_USERNAME_COLON_PASSWORD>"
            }
          }
        }

---

## 30. Harbor Example: Deployment YAML Writing Method

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-web
      namespace: test
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx-web
      template:
        metadata:
          labels:
            app: nginx-web
        spec:
          imagePullSecrets:
            - name: harbor-registry-secret
          containers:
            - name: nginx-web
              image: harbor.example.com/devops/nginx-web:v1
              imagePullPolicy: IfNotPresent
              ports:
                - containerPort: 80

---

## 31. What to Pay Special Attention to in Harbor Scenarios

If the enterprise Harbor uses a certificate signed by a regular CA, it can typically be handled the same way as cloud vendor repositories.

But if Harbor uses:

- A self-signed certificate
- An internal enterprise CA certificate
- A certificate not trusted by default nodes

Then even if `imagePullSecrets` is correct, image pulling may still fail, with common errors like:

    certificate signed by unknown authority

In such cases, you need to additionally handle:

**Let the container runtime on Kubernetes nodes trust the Harbor certificate.**

### Operations Understanding Focus

`imagePullSecrets` resolves:

- Authentication issues

It does not resolve:

- Issues with nodes not trusting the repository certificate

In other words, Harbor image pulling failures may involve two layers of issues:

1. Whether the authentication configuration is correct
2. Whether the nodes trust the Harbor certificate

---

## 32. Supplement: How to Handle if Harbor Uses an Internal HTTP Protocol

Here's a very important prerequisite first:

**`imagePullSecrets` can only resolve authentication issues and cannot automatically accept an HTTP repository.**

If Harbor is not using HTTPS but instead directly uses:

    http://harbor.example.com

This is considered by the container runtime on nodes as:

- A plaintext HTTP Registry
- An insecure repository (Insecure Registry)

Even if you have correctly configured:

- Harbor username and password
- `imagePullSecrets`
- The private image address in Deployment

The node may still fail to pull the image.

Because the issue is no longer about "whether authentication information exists," but rather:

**Whether the container runtime on the node allows accessing this repository via HTTP.**

### Core Understanding for This Scenario

If Harbor is an HTTP repository, the issue to handle is not the Pod YAML itself, but:

**The container runtime configuration on every Kubernetes node.**

In other words, you need to let:

- Docker
- containerd

Accept this Harbor as an "allowed HTTP repository" at the node level.

### Operations Understanding Focus

This is a completely different layer of issue from `imagePullSecrets`:

#### 1. What `imagePullSecrets` Resolves
- Whether the username/password is correct
- Whether the private image has authentication information

#### 2. What Node Trusting HTTP Harbor Resolves
- Whether the container runtime allows accessing the repository via HTTP
- Whether the repository is considered an Insecure Registry

So if Harbor is HTTP, you must at least satisfy both:

1. The node's container runtime allows access to this HTTP repository
2. The Pod has correct authentication information when pulling the private image

Otherwise, the image will still fail to pull.

---

## 33. How to Trust an Internal HTTP Harbor if the Node Uses Docker

If the node's container runtime is Docker, the common approach is to modify:

    /etc/docker/daemon.json

Example:

    {
      "insecure-registries": ["harbor.example.com"]
    }

If Harbor includes a port, you must also include it, for example:

    {
      "insecure-registries": ["harbor.example.com:8080"]
    }

After modification, restart Docker:

    systemctl daemon-reload
    systemctl restart docker

Then you can execute:

    docker info

To check if the output contains:

    Insecure Registries

If you run the following on the node itself:

    docker login harbor.example.com
    docker pull harbor.example.com/devops/nginx-web:v1

There are no issues, which indicates that the Docker on this node has already accepted the Harbor as an HTTP / insecure repository.

### Operations Understanding Focus

If Kubernetes nodes use Docker as the runtime, this step is a **node-level configuration**, not a cluster object configuration.

That is to say:

- Modifying a Deployment will not automatically make the node trust HTTP Harbor
- Modifying a Secret will not automatically make the node trust HTTP Harbor
- The configuration must be completed on each node that actually pulls images

---

## Thirty-Four, How to Trust Internal HTTP Harbor When Nodes Use containerd

In the containerd scenario, the core idea is typically:

- Create a registry host directory for the specified Harbor address
- Place `hosts.toml` in the directory
- Explicitly tell containerd: this Harbor allows access via HTTP

### First, explain two important points

#### 1. The main difference between containerd 1.x and 2.x is in `config.toml`
The common registry configuration path for containerd 1.x is usually:

    [plugins."io.containerd.grpc.v1.cri".registry]

The common registry configuration path for containerd 2.x is usually:

    [plugins.'io.containerd.cri.v1.images'.registry]

That is to say:

- 1.x and 2.x have different plugin paths in `config.toml`
- However, Harbor's HTTP trust and self-signed CA trust often still follow the main line of `certs.d/<registry>/hosts.toml`

#### 2. The following content is based on the current environment
The current node is using **containerd 2.x**, and `config.toml` has already clearly configured the registry configuration path, for example:

    version = 3

and:

    [plugins.'io.containerd.cri.v1.images'.registry]
      config_path = '/etc/containerd/certs.d:/etc/docker/certs.d'

This indicates that the current environment will read the registry host configuration from the following paths:

- `/etc/containerd/certs.d`
- `/etc/docker/certs.d`

However, from the perspective of standards and maintainability, it is still recommended to prioritize using:

    /etc/containerd/certs.d

### Recommended Practice for containerd 2.x

Assume the Harbor address is:

    harbor.example.com

First create the directory:

    mkdir -p /etc/containerd/certs.d/harbor.example.com

Then create:

    /etc/containerd/certs.d/harbor.example.com/hosts.toml

The content is as follows:

    server = "http://harbor.example.com"

    [host."http://harbor.example.com"]
      capabilities = ["pull", "resolve", "push"]

If the Harbor includes a port, for example:

    harbor.example.com:8080

The directory should be changed to:

    /etc/containerd/certs.d/harbor.example.com:8080

The corresponding `hosts.toml` content can be written as:

    server = "http://harbor.example.com:8080"

    [host."http://harbor.example.com:8080"]
      capabilities = ["pull", "resolve", "push"]

After making the changes, restart containerd:

    systemctl restart containerd

### How to Verify

It is recommended to verify on the node itself rather than just checking Pod status.

You can first try:

    crictl pull harbor.example.com/devops/nginx-web:v1

If your environment's `crictl` has not been configured yet, you can also further check on the node whether containerd has reloaded the directory configuration, or combine with Pod events for further troubleshooting.

### Operations Understanding Focus

In the containerd scenario, the core is not to modify Pods, but:

- Create a directory for the specific registry host
- Use `hosts.toml` to explicitly tell containerd: this Harbor should be accessed via HTTP

### Additional Reminder

Different Kubernetes distributions may encapsulate containerd registry configurations in different ways:

- Some directly use `/etc/containerd/certs.d/`
- Some will also combine with `config.toml`
- Some distributions may have their own encapsulation layer

However, if the current node has already correctly declared `config.toml` with `config_path`, then `certs.d/<registry>/hosts.toml` is often the most direct implementation method.

---

## Thirty-Five, Supplement: How to Handle Self-Signed Certificates if Harbor Uses Self-Signed Certificates

If Harbor is not HTTP but HTTPS, but the certificate is:

- A self-signed certificate
- A company internal CA certificate
- A certificate not in the node's default certificate trust chain

The node may report an error when pulling images:

    certificate signed by unknown authority

### What is the essence of this scenario

This is not about "lack of authentication information", but:

**The node does not trust the HTTPS certificate provided by Harbor.**

That is to say:

- The username and password may be correct
- `imagePullSecrets` may also be fine
- But the TLS handshake has already failed

### Operations Understanding Focus

Here, it is still necessary to distinguish between two layers:

#### 1. `imagePullSecrets`
Solving Harbor login authentication issues

#### 2. Node trusts self-signed certificate
Solving HTTPS certificate validation issues

These two things must be understood separately.

---

## Thirty-Six, How to Trust Self-Signed Harbor Certificate When Nodes Use Docker

If the Harbor address is:

    harbor.example.com

The common path is:

    /etc/docker/certs.d/harbor.example.com/ca.crt

If the Harbor includes a port, for example:

    harbor.example.com:8443

The directory is usually:

    /etc/docker/certs.d/harbor.example.com:8443/ca.crt

That is to say, the **CA certificate that issues the Harbor server certificate** should be placed in the corresponding directory.

After placing the certificate, restart Docker:

    systemctl restart docker

Then verify on the node itself: /think

docker login harbor.example.com
docker pull harbor.example.com/devops/nginx-web:v1

If the pull is successful, it indicates that this node trusts the Harbor's self-signed certificate.

### Operations Understanding Focus

Here is placed:

**The CA certificate used to issue the Harbor server certificate**

Rather than just placing any arbitrary business certificate file.

If the certificate chain itself is incorrect, even if the path is correct, it may still fail verification.

---

## 37. How to Trust Self-Signed Harbor Certificates When Using containerd

In the containerd scenario, trusting the self-signed CA and HTTP Harbor follows the same main principle:

    /etc/containerd/certs.d/<registry>/

But you need to additionally place the CA certificate in the corresponding directory and explicitly specify it in `hosts.toml`.

### The following content in this section is based on containerd 2.x in the current environment

Assume the Harbor address is:

    harbor.example.com

First create the directory:

    mkdir -p /etc/containerd/certs.d/harbor.example.com

Place the CA certificate in:

    /etc/containerd/certs.d/harbor.example.com/ca.crt

Then create:

    /etc/containerd/certs.d/harbor.example.com/hosts.toml

Content as follows:

    server = "https://harbor.example.com"

    [host."https://harbor.example.com"]
      capabilities = ["pull", "resolve", "push"]
      ca = "/etc/containerd/certs.d/harbor.example.com/ca.crt"

If Harbor includes a port, for example:

    harbor.example.com:8443

The directory should be changed to:

    /etc/containerd/certs.d/harbor.example.com:8443

The example content can be written as:

    server = "https://harbor.example.com:8443"

    [host."https://harbor.example.com:8443"]
      capabilities = ["pull", "resolve", "push"]
      ca = "/etc/containerd/certs.d/harbor.example.com:8443/ca.crt"

After making the changes, restart containerd:

    systemctl restart containerd

### How to Verify

It is recommended to verify directly on the node:

    crictl pull harbor.example.com/devops/nginx-web:v1

If it can pull normally, it indicates:

- containerd has already trusted the Harbor's CA
- The Harbor certificate chain is at least usable for the current node

### Operations Understanding Focus

The key point for containerd here is:

- `server` is still `https://...`
- It is not disabling TLS
- It is explicitly telling containerd: "This Harbor's CA certificate is here"

This is different from treating "HTTP Harbor" as an insecure registry.

---

## 38. Can We Skip Certificate Validation Directly

Technically, in some scenarios, you can bypass TLS validation by configuration.

However, from a security perspective, this is typically only suitable for temporary testing and not recommended as a formal production solution.

### Why It's Not Advised for Long-Term Use

Because this effectively tells the node:

- No longer strictly validate certificate authenticity
- Just try to trust if it can connect

This reduces the security of the image supply chain.

### What Is a More Standard Approach

A more standard approach is usually:

1. Harbor uses HTTPS
2. Use an internal enterprise CA or a regular CA-signed certificate
3. Distribute the CA certificate to each node
4. Let Docker / containerd officially trust this CA
5. Continue using `imagePullSecrets` on the Pod side to resolve authentication issues

In other words:

**In formal environments, prioritize using HTTPS + correct CA trust chain, rather than long-term using HTTP or skipping validation.**

---

## 39. The Most Confusing Part

This section is most prone to confusing three things:

### 1. `imagePullSecrets`
Solving private repository account password authentication issues

### 2. HTTP Harbor Trust
Solving whether the node allows access to plaintext HTTP Registry

### 3. Self-Signed Certificate Trust
Solving whether the node trusts the Harbor's HTTPS certificate

### Must Clearly Distinguish

- Configuring `imagePullSecrets` does not mean the node trusts HTTP Harbor
- Configuring `imagePullSecrets` does not mean the node trusts the self-signed certificate
- The node trusting the Harbor certificate does not necessarily mean private repository authentication succeeds

These three are not alternatives but different levels of issues.

---

## 40. How to Answer Such Questions in Interviews or Troubleshooting

If an interviewer asks:

**Kubernetes fails to pull Harbor private images, besides imagePullSecrets, what else should be checked?**

You can answer according to this logic:

1. First distinguish between authentication failure, certificate failure, or HTTP/HTTPS access method issues
2. `imagePullSecrets` solves private repository authentication
3. If Harbor is HTTP, configure insecure registry or corresponding registry host settings on the node container runtime
4. If Harbor is HTTPS with self-signed certificate, ensure the node's Docker / containerd trusts the Harbor's CA
5. If running containerd 2.x, also confirm that `config.toml`'s registry's `config_path` has been correctly declared
6. Ultimately, verify at the node level, as the actual image pulling is done by kubelet working with the node container runtime

This answer is more complete than simply saying "just configure an imagePullSecrets".

---

## 41. Image Pull Failures Should Not Automatically Be Assumed to Be Authentication Issues

Many people immediately think it's an authentication issue when seeing:

- `ErrImagePull`
- `ImagePullBackOff`

But it's not necessarily the case.

### It Could Also Be These Reasons

#### 1. Image Address is Wrong

For example, the registry domain is misspelled.

#### 2. Image Name is Wrong

For example, the project name or repository path is incorrect.

#### 3. Tag Does Not Exist

For example, you wrote `v2`, but the repository actually doesn't have this tag.

#### 4. Node Cannot Reach the Registry

For example:

- DNS resolution issues
- Network policy restrictions
- Firewall not allowing traffic

#### 5. Registry Certificate Issues /think

Some HTTPS private repositories may also fail to pull if the certificate is not trusted.

### Operations Understanding Focus

Authentication issues are just one category of reasons for image pull failures, not the only cause.

---

## 42. Common Mistakes When Dealing with This Issue

### 1. Secret Name Written Incorrectly

The Secret referenced in the Deployment does not exist.

### 2. Secret in the Wrong Namespace

Secrets must be in the same namespace as the Pod to be referenced.

### 3. Image Repository Address Mismatch with Secret Authentication Content

For example, a Secret is created for one repository address, but the image uses another repository address.

### 4. Assuming This Machine Can Pull, the Cluster Nodes Can Also Pull

This is a very typical misconception.

### 5. Ignoring Non-Authentication Issues Like Tag, Network, and Certificate

Leading to misjudgment.

### 6. Line Breaks Mixed in Base64 Generation

Causing `auth` content anomalies, further triggering authentication failure.

### 7. containerd Wrote `hosts.toml` but Did Not Confirm `config.toml` Declaration of `config_path`

Even if the directory and files exist, it may not take effect.

---

## 43. What to Check First When Troubleshooting Image Pull Issues

### 1. Check Pod Status First

Check for:

- `ErrImagePull`
- `ImagePullBackOff`

### 2. Then Check Event Information

Events usually show more specific reasons, such as:

- unauthorized
- not found
- connection refused
- certificate signed by unknown authority

### 3. Then Verify Image Name and Tag

Confirm the repository, project, image name, and tag are all correct.

### 4. Then Verify imagePullSecrets

Confirm:

- Whether it is declared
- Whether the name is correct
- Whether the namespace is correct

### 5. Then Check the Repository Itself

Confirm:

- Whether nodes can access the repository
- Whether DNS is correct
- Whether the certificate is normal
- Whether the repository actually has the image

### 6. Then Check Runtime Configuration

If the node uses Docker, check:

- `/etc/docker/daemon.json`
- `docker info`

If the node uses containerd, check:

- `/etc/containerd/config.toml`
- Whether `config_path` is correct
- `/etc/containerd/certs.d/<registry>/hosts.toml`
- Whether `ca.crt` is needed

### 7. Then Check Secret Content Structure

Confirm:

- Whether the Secret type is `kubernetes.io/dockerconfigjson`
- Whether the `.dockerconfigjson` structure is correct
- Whether `auth` is generated correctly according to `username:password`
- Whether Base64 accidentally includes line breaks

---

## 44. Recommended Verification Commands to Master

### 1. Check Whether Secret Exists

    kubectl get secret -n test

### 2. Check Secret Details

    kubectl describe secret aliyun-registry-secret -n test

### 3. Check Pod Status

    kubectl get pod -n test

### 4. Check Pod Details and Events

    kubectl describe pod -n test <pod-name>

### 5. Check Deployment Configuration

    kubectl get deploy nginx-web -n test -o yaml

### 6. Check Secret Original YAML

    kubectl get secret aliyun-registry-secret -n test -o yaml

### 7. Check containerd Configuration

    cat /etc/containerd/config.toml

### 8. Check Harbor Corresponding registry Directory

    ls -R /etc/containerd/certs.d

### 9. Use CRI Tool to Validate Image Pull

    crictl pull harbor.example.com/devops/nginx-web:v1

---

## 45. The Most Important Cognitive Points in This Article

### 1. Pulling Private Image Repositories Usually Requires Authentication
Nodes cannot assume they have inherent permissions by default.

### 2. Kubernetes Uses Secret to Store Repository Authentication Information
This is part of sensitive configuration management.

### 3. `imagePullSecrets` is used for Pod image pulling
It is not a Secret for business programs to read.

### 4. `imagePullSecrets` is not an independent object
It is merely a reference method for Secret in Pod / Deployment.

### 5. The actual authentication information is stored in `Secret`
And the Secret type is usually:

    kubernetes.io/dockerconfigjson

### 6. YAML writing typically has two parts
- `Secret YAML`
- `Deployment YAML`

### 7. `auth` is not password Base64
It is Base64 of `username:password`.

### 8. `echo` may default contain line breaks
If not adding `-n`, the encoding result may not meet expectations.

### 9. Harbor and cloud vendor private repositories are essentially consistent in creation
The main differences are in repository address, account system, and certificate environment.

### 10. Harbor may have three layers of issues
- Authentication issues
- HTTP repository trust issues
- HTTPS self-signed certificate trust issues

### 11. The main difference between containerd 1.x and 2.x is `config.toml`
- 1.x commonly is `plugins."io.containerd.grpc.v1.cri".registry`
- 2.x commonly is `plugins.'io.containerd.cri.v1.images'.registry`

### 12. If the current environment is containerd 2.x
Should focus on confirming:
- `version = 3`
- `config_path`
- `/etc/containerd/certs.d/<registry>/hosts.toml`

### 13. Knowledge base and examples should not write real passwords
Should uniformly use placeholders, such as:

- `<REGISTRY_PASSWORD>`
- `<BASE64_OF_USERNAME_COLON_PASSWORD>`
- `<HARBOR_PASSWORD>`

---

## 46. What Level Should You Reach to Learn This Article

At this stage, it is recommended to first reach the following level:

### 1. Understand why private repositories require imagePullSecrets  
### 2. Understand that imagePullSecrets fundamentally depends on Secret  
### 3. Be able to read the syntax of imagePullSecrets in Deployment  
### 4. Understand the difference between imagePullSecrets and regular business Secret  
### 5. Be able to make initial judgments on image pull failures  
### 6. Be able to create image pull Secret using commands  
### 7. Understand the relationship between Secret and Deployment in dual-file versions  
### 8. Understand the commonalities and differences in usage between Harbor and cloud vendor private repositories  
### 9. Understand the generation logic of `auth` and common Base64 pitfalls  
### 10. Understand that Harbor HTTP, HTTPS self-signed certificates and authentication issues are at different levels  
### 11. Understand the differences between Docker and containerd in Harbor trust mechanisms  
### 12. Understand that the main difference between containerd 1.x and 2.x is in the path of `config.toml`, while Harbor host trust typically still goes through `certs.d`  
### 13. Avoid writing real passwords directly into command history or version repositories  

---

## Forty-Seven, Common Follow-up Questions in Interviews  

This section commonly includes questions such as:  

- How does a private image repository pull images in Kubernetes  
- What is imagePullSecrets  
- What's the difference between imagePullSecrets and regular Secret  
- Why does Pod show `ImagePullBackOff`  
- How to troubleshoot image pull failures generally  
- Why is Secret also used for image pull authentication  
- Where should imagePullSecrets be written  
- Why can the local machine pull images but not the cluster  
- How to create docker-registry type Secret  
- Are the creation commands for Harbor and Alibaba Cloud private repositories the same  
- Why does Harbor still fail to pull images when using HTTP  
- Why does Harbor still fail to pull images when using self-signed certificates  
- How is `auth` generated in `.dockerconfigjson`  
- Why are the Base64 results of `echo -n` and regular `echo` different  
- What are the differences in Harbor trust mechanisms when nodes use Docker and containerd  
- What are the differences in registry configuration paths between containerd 1.x and 2.x  

---

## Forty-Eight, Stage Summary  

imagePullSecrets is a critical but often overlooked component in Kubernetes application deployment.  

The most important part of this article is not memorizing commands, but establishing these core understandings:  

- Private repositories typically require authentication to pull images  
- Kubernetes stores this type of authentication information in Secret  
- Pods reference these authentication details via imagePullSecrets  
- Authentication failure is a major cause of image pull failures  
- However, image pull failure does not necessarily mean it's an authentication issue  

In practical operations, you also need to master several common methods:  

- Creating `docker-registry` type Secret through commands  
- Understanding object relationships through `secret.yaml + deployment.yaml` dual-file  
- Understanding the structure of `.dockerconfigjson`  
- Understanding how `auth` is generated  
- Understanding why real passwords should not be written directly into command history or version repositories  

For enterprise Harbor scenarios, you also need to pay special attention to three issues:  

- Authentication issues  
- Node trust issues for HTTP Harbor  
- Node trust issues for HTTPS self-signed certificates Harbor  

If the node runtime is Docker, check:  

- `/etc/docker/daemon.json`  
- `/etc/docker/certs.d/`  

If the node runtime is containerd, you also need to further distinguish:  

- The registry path in `config.toml` differs between containerd 1.x and 2.x  
- However, Harbor host trust often still goes through `certs.d/<registry>/hosts.toml`  

As long as you establish these understandings, you won't just mechanically check statuses when facing image pull failure troubleshooting, but can more systematically judge the problem level.  

---

## Forty-Nine, Keyword Quick Notes  

- Private Image Repository: An image repository that requires authentication to pull  
- imagePullSecrets: The Secret reference used by Pods to pull private images  
- `kubernetes.io/dockerconfigjson`: Common image repository authentication Secret type  
- `ErrImagePull`: Image pull failure  
- `ImagePullBackOff`: Image pull failure retry status after failure  
- Namespace: Secret and Pod must be in the same namespace  
- Image Authentication: Credentials required for nodes to access private repositories  
- `kubectl create secret docker-registry`: Common command to create image repository authentication Secret  
- `.dockerconfigjson`: Docker / OCI repository authentication configuration structure  
- `auth`: Base64 of `username:password`  
- `echo -n`: Avoid introducing newline characters during encoding  
- HTTP Harbor: A plaintext repository requiring explicit node container runtime trust  
- Self-signed Certificate Harbor: An HTTPS repository requiring explicit CA trust from node container runtime  
- `hosts.toml`: containerd registry host configuration file  
- `config_path`: Entry point for containerd to read registry host configuration directory  
- `ca.crt`: CA file used to issue Harbor certificates  

---

## Fifty, Operational Extended Understanding  

From an operations perspective, imagePullSecrets is not just a "small configuration" for pulling images, but an important step in truly closing the image delivery chain.  

In actual environments, many teams focus only on:  

- Whether images can be built  
- Whether images can be pushed  

But the key to the cluster is:  

**Whether nodes can stably, legally, and controllably pull images.**  

This is why imagePullSecrets, although seemingly just a small configuration in the Pod, actually connects to:  

- Image repository permissions  
- Secret management  
- Node pull behavior  
- Application startup success or failure  
- Subsequent deployment troubleshooting  

Looking deeper, this topic also connects to several very practical issues:  

- How to securely store credentials  
- Whether passwords enter history  
- Whether Secrets are mistakenly committed to version repositories  
- Whether enterprise Harbor uses HTTP  
- Whether enterprise Harbor's certificate is trusted by nodes  
- Whether containerd's `config_path` has been correctly declared  
- Whether `hosts.toml` is written correctly  
- Whether encoding failures due to line breaks, spaces, or formatting issues cause authentication failures  

So this article may seem small, but it's actually a key step from "knowing how to deploy YAML" to "understanding the real image delivery chain."

## References
- Kubernetes Images:https://kubernetes.io/docs/concepts/containers/images/
- Pull an Image from a Private Registry:https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
- Kubernetes Secret:https://kubernetes.io/docs/concepts/configuration/secret/

---

## Next Day Suggestions
Next article recommendation:

[[01-livenessProbe readinessProbe Basic Practices]]